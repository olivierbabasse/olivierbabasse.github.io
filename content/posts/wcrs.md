+++
date = '2026-04-08T09:51:22+02:00'
draft = false
title = 'Optimizing a simple word count tool (WC-style) implementation'
+++


# Context

System `wc` is the kind of tool you assume is already fast because it's been shipping on Unix for 50 years, but it turns out that on modern hardware a careful implementation can beat it by an order of magnitude.
I started with a naive rust implementation of the default mode of the `wc` challenge (see https://codingchallenges.fyi/challenges/challenge-wc) and tried to improve performance, benchmarking against a naive implementation and the system `wc` command. I started in ASCII mode only to keep things simple and focused, counting bytes, words and lines.
Benchmarking will be done using `hyperfine` with 3 warmups and 10 runs on a 1GB test file to have meaningful execution times.

Spoiler : I reached an **~56x speedup over the naive Rust implementation** (about 92× over `/usr/bin/wc` on the same machine). All the speedup figures throughout the article are reported against the naive Rust baseline, since that's what reflects the optimization effort ; the final progression table also includes the system `wc` row for context. 


# Reference implementation

I did my tests on an i7-9700k with linux kernel 6.8.0 :
reference implementation `/usr/bin/wc` from GNU coreutils : 3.879s

For "fairness" I ran my tests with `LC_ALL=C` to avoid having wc paying a cost to UTF-8 decoding, but it didn't change anything.


# Naive implementation

Reading the whole file into a buffer, I got the total size of the buffer then did two passes to count lines (looking for `\n`) and words (looking for spaces)

```rust
use std::env;
use std::fs;

fn main() {
    let filename = env::args().nth(1).expect("Usage: wcrs <filename>");
    let data = fs::read(&filename).expect("Could not read file");

    let bytes = data.len();
    let lines = data.iter().filter(|&&b| b == b'\n').count();
    let words = data
        .split(|b| b.is_ascii_whitespace())
        .filter(|w| !w.is_empty())
        .count();

    println!("  {lines}  {words} {bytes} {filename}");
}
```

Run time : 2.372s, which is already faster than the system wc.

That a naive two-pass Rust implementation already beats `/usr/bin/wc` was unexpected and worth dwelling on. Two reasons :

- `is_ascii_whitespace()` is `matches!(*self, b'\t' | b'\n' | b'\x0C' | b'\r' | b' ')` — it inlines into a branchless five-byte compare sequence with no function call.
- LLVM auto-vectorizes simple iterator chains like `data.iter().filter(|&&b| b == b'\n').count()` into a SIMD `memchr`-style loop. With `target-cpu=native` (added in the next step) that becomes AVX2 on this machine.

Meanwhile system `wc` is doing per-byte work with an `iswspace`-equivalent function call per character and no SIMD. So we're effectively comparing a simple loop against compiler-generated SIMD code. Even before any optimization effort, Rust + LLVM hands most of the win for free.

One thing I want to point out before going further : at every step from here on, I checked output against `/usr/bin/wc` to make sure I hadn't broken anything.


# Usual release optimization options and CPU native code

Adding the usual release options to Cargo.toml brings almost no gain alone :

```toml
[profile.release]
codegen-units = 1
lto = "fat"
panic = "abort"
debug = true
strip = false
```

Note that I keep debug info in the binary to have readable flamegraph/samply info.

The next step is to target the CPU native instruction set via .cargo/config.toml :

```toml
[build]
rustflags = ["-Ctarget-cpu=native"]
```

This is way more effective. Run time : 2.055s


# Buffered IO and single pass

Profiling pointed at the two scan passes as the bulk of the cost : 77% for word counting, 6% for line counting and 17% for reading + allocating.

Two changes felt obviously worth doing together : read the file in 64 KB chunks instead of slurping the whole gigabyte into a `Vec` (cuts the 17%), and merge the two passes into one loop (collapses the 6% + 77% into a single sweep).

Why 64 KB ? I tried smaller (8, 16, 32) and larger (128 KB up to 1 MB). The problem with small values is too many syscalls. With too large values we exceed CPU cache size. The i7-9700K has 32 KB L1d and 256 KB L2 per core, so when `read()` returns, the freshly-written 64 KB buffer is still warm in L2 and streams into L1d sequentially as we scan. Anything bigger has to come back through L3.

User time/system time split is quite interesting : a bit more user time but 4x less system time.
The combined inner loop is slightly slower per byte than the naive two-pass form because LLVM auto-vectorizes `data.iter().filter(|&&b| b == b'\n').count()` into a `memchr`-style SIMD scan, but our combined loop with branching word-state stays fully scalar. We lost that free vectorization on the line count.

Run-time : 1.822s


# Improving newline counting with memchr

An easy first win : the `memchr` crate provides a `memchr`-like function using a SIMD implementation working on multiple bytes at once (32 with AVX2)

Run-time : 1.627s


# Branchless word counting

Now we work on word counting, the most CPU consuming part.

```rust
if b.is_ascii_whitespace() {
    in_word = false;
} else if !in_word {
    in_word = true;
    words += 1;
}
```

I tried to replace the "if" with branchless code because when branch prediction is not good, the CPU can waste a lot of time recovering from mispredicts :

```rust
for &b in chunk {
    let is_ws = b.is_ascii_whitespace();
    words += (!is_ws && !in_word) as u64;
    in_word = !is_ws;
}
```

It didn't work, in fact it was slightly slower.

`perf stat -r 5 -e instructions,cycles,branches,branch-misses` gave me around 15% branch-misses for both versions, but 4% more instructions and cycles for the branchless version.

Looking at the code with `objdump`, we always have branching in `is_ascii_whitespace()` :

```
movzbl (%rax),%edx
cmp    $0x20,%rdx
ja     <non_ws>      ; fast path : byte > 0x20 -> not whitespace
bt     %rdx,%rbx
jae    <non_ws>      ; bit-test against the WS mask in %rbx
```

Same quantity of branching, more instructions in the "branchless" version, it's a loss.


# SIMD word counting

Now things get interesting, big improvement potential here.

The idea here is to try to parallelize the checks done in the above loop. Using AVX2, we'd like to work on 32 chars at a time instead of 1.

First we extract the word counting loop into its own function :
```rust
fn count_words_scalar(chunk: &[u8], in_word: &mut bool) -> u64 {
    let mut words: u64 = 0;
    let mut iw = *in_word;
    for &b in chunk {
        if b.is_ascii_whitespace() {
            iw = false;
        } else if !iw {
            iw = true;
            words += 1;
        }
    }
    *in_word = iw;
    words
}
```

Then we start to work on a SIMD implementation. The code above counts "word starts" sequentially (in_word value going from false to true), keeping "in_word status" between chunks. We want to do the same thing on 32 bytes at a time.

Rust's `is_ascii_whitespace()` matches 5 byte values :
```rust
pub const fn is_ascii_whitespace(&self) -> bool {
    matches!(*self, b'\t' | b'\n' | b'\x0C' | b'\r' | b' ')
}
```

For the SIMD path I'll match 6 byte values, adding `\x0B` (vertical tab) to align with POSIX `isspace()` and what `/usr/bin/wc` consider whitespace. I should have done it before but I hadn't noticed this 6th value until this point.

Using `_mm256_set1_epi8`, we create our whitespace masks :
```rust
let space = _mm256_set1_epi8(b' ' as i8);
etc...
```

We'll then loop over 32 bytes chunks, loading them one after the other with `_mm256_loadu_si256` :
```rust
let data = _mm256_loadu_si256(chunk.as_ptr().add(i) as *const __m256i);
```

Using `_mm256_cmpeq_epi8`, we can find which of our 32-bytes equal a specific value, we'll repeat this operation 6 times with all 6 separators and OR the results to get all separators in our 32 bytes :
```rust
let ws = _mm256_or_si256(
    _mm256_or_si256(
        _mm256_or_si256(
            _mm256_cmpeq_epi8(data, space),
            _mm256_cmpeq_epi8(data, tab),
        ),
        _mm256_or_si256(
            _mm256_cmpeq_epi8(data, newline),
            _mm256_cmpeq_epi8(data, cr),
        ),
    ),
    _mm256_or_si256(
        _mm256_cmpeq_epi8(data, vt),
        _mm256_cmpeq_epi8(data, ff),
    ),
);
```

I wrote it as a balanced tree on purpose — the front-end will reorder ops to extract ILP either way, but a balanced tree lines up better visually and makes the dependency depth explicit (3 OR layers instead of 5 in a left-leaning chain).

We get something like this :
```
bytes: [H, e, l, l, o, ' ', w, o, r, l, d, \n, T, h, i, s, ' ', i, s, ' ', a, ' ', t, e, s, t, \n, ' ', ' ', f, o, o]
ws:     0  0  0  0  0   1   0  0  0  0  0   1  0  0  0  0   1   0  0   1   0   1   0  0  0  0   1   1    1   0  0  0
```
(1 meaning 0xff)

Now the trick is to find all word starts, ie the bytes which are NOT separators, but preceded by a separator :

```rust
word_start = not_ws & (ws << 1)
```

We create the ws 32-bit value from our 32 bytes with `_mm256_movemask_epi8` and it's just some classic bit manipulation :

```
ws:         0 0 0 0 0 1 0 0 0 0 0 1 0 0 0 0 1 0 0 1 0 1 0 0 0 0 1 1 1 0 0 0
not_ws:     1 1 1 1 1 0 1 1 1 1 1 0 1 1 1 1 0 1 1 0 1 0 1 1 1 1 0 0 0 1 1 1
prev_ws:    ? 0 0 0 0 0 1 0 0 0 0 0 1 0 0 0 0 1 0 0 1 0 1 0 0 0 0 1 1 1 0 0   (ws shifted right by 1, the leading ? comes from the previous block)
word_start: ? 0 0 0 0 0 1 0 0 0 0 0 1 0 0 0 0 1 0 0 1 0 1 0 0 0 0 0 0 1 0 0   (not_ws AND prev_ws)
```

Last, we just have to count the 1-bits with `count_ones()` and forward the `in_word` state to the next block (it's the last bit of the block).

The remaining data (<32 bytes) goes through the sequential implementation

Our SIMD implementation now looks like this :
```rust
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
unsafe fn count_words_avx2(chunk: &[u8], in_word: &mut bool) -> u64 { unsafe {
    let mut words: u64 = 0;
    // Invariant : prev_ws_bit is 1 iff the byte just before this block was whitespace.
    // We feed it into bit 0 of `prev_was_ws` so the very first byte of the block
    // can correctly be detected as a word start when needed.
    let mut prev_ws_bit: u32 = if *in_word { 0 } else { 1 };

    let space = _mm256_set1_epi8(b' ' as i8);
    let tab = _mm256_set1_epi8(b'\t' as i8);
    let newline = _mm256_set1_epi8(b'\n' as i8);
    let cr = _mm256_set1_epi8(b'\r' as i8);
    let vt = _mm256_set1_epi8(0x0B as i8);
    let ff = _mm256_set1_epi8(0x0C as i8);

    let mut i = 0;
    let len = chunk.len();

    while i + 32 <= len {
        let data = _mm256_loadu_si256(chunk.as_ptr().add(i) as *const __m256i);

        let ws = _mm256_or_si256(
            _mm256_or_si256(
                _mm256_or_si256(
                    _mm256_cmpeq_epi8(data, space),
                    _mm256_cmpeq_epi8(data, tab),
                ),
                _mm256_or_si256(
                    _mm256_cmpeq_epi8(data, newline),
                    _mm256_cmpeq_epi8(data, cr),
                ),
            ),
            _mm256_or_si256(
                _mm256_cmpeq_epi8(data, vt),
                _mm256_cmpeq_epi8(data, ff),
            ),
        );

        let ws_mask = _mm256_movemask_epi8(ws) as u32;
        let not_ws_mask = !ws_mask;
        let prev_was_ws = (ws_mask << 1) | prev_ws_bit;
        let word_starts = not_ws_mask & prev_was_ws;
        words += word_starts.count_ones() as u64;
        prev_ws_bit = (ws_mask >> 31) & 1;

        i += 32;
    }

    // prev_ws_bit was just updated to (last bit of the last block we processed) ;
    // if that bit is 0 the last byte was non-whitespace, i.e. we end inside a word.
    *in_word = prev_ws_bit == 0;
    words + count_words_scalar(&chunk[i..], in_word)
}}
```

and to have our code support non AVX2 platforms :
```rust
fn count_words(chunk: &[u8], in_word: &mut bool) -> u64 {
    #[cfg(target_arch = "x86_64")]
    {
        if is_x86_feature_detected!("avx2") {
            return unsafe { count_words_avx2(chunk, in_word) };
        }
    }
    count_words_scalar(chunk, in_word)
}
```

The result is impressive with a run time of 157ms, almost **10 times faster** compared to the previous step !


# Integrating line counting within word counting loop

Instead of reading memory twice (once with `memchr`, once with our SIMD loop), I did it all in the SIMD loop.
I reused the newline vector to directly count the number of newlines in the input (`_mm256_cmpeq_epi8`/`_mm256_movemask_epi8`/`count_ones`).

```rust
let nl_cmp = _mm256_cmpeq_epi8(data, newline);
let nl_mask = _mm256_movemask_epi8(nl_cmp) as u32;
lines += nl_mask.count_ones() as u64;
```

The gain is very small this time, with a run time of 154ms (within the error margin according to `hyperfine`, so I may even have gained nothing).
Testing with a larger test file might help here.


# Replacing buffered reading with memory mapping

Quite straightforward using the `mmap` crate :

```rust
let file = File::open(&filename)?;
let mmap = unsafe { Mmap::map(&file)? };
let data: &[u8] = &mmap;
let bytes = data.len() as u64;
...
```

On Linux I get a small gain — run-time is now 137 ms. No combination of `mmap()` advice (`madvise(MADV_SEQUENTIAL)`, `madvise(MADV_WILLNEED)`) helped beyond that. `MADV_WILLNEED` in particular, which prefetches pages upfront, gave worse results : we're no longer interleaving the page-mapping cost with the SIMD work.

> #### Sidebar : the same code on a 2015 MacBook
>
> While writing this article I was running the same benchmarks on an early-2015 MacBook in parallel. From this step on, the two platforms diverge enough that I'm folding the macOS thread into this sidebar rather than carrying it through the rest of the article.
>
> On the MacBook, `mmap()` instead of `read()` made things almost **2× slower**. Comparing my word counter to `cat` with `/usr/bin/time` showed dramatically more system time, despite `mmap()` supposedly being cheaper than `read()` because it doesn't copy.
>
> The likely culprit is soft page faults. Unlike `read()`, `mmap()` doesn't populate page tables eagerly — the first time each thread touches a page, the CPU traps into the kernel to install the PTE. 1 GB / 4 KB = 262 K pages, and at a few hundred nanoseconds per fault (more under contention) you lose 100-150 ms — exactly the gap I measured.
>
> I tried everything I could without sudo : `madvise(MADV_SEQUENTIAL)`, `madvise(MADV_WILLNEED)`, a sequential prefault walking the `mmap` region one byte per page to force PTEs in up front, and a multithreaded version of the same. None of them moved the needle on macOS.
>
> ```rust
> let mut sink: u8 = 0;
> let mut p = 0;
> while p < data.len() {
>     sink ^= unsafe { *data.as_ptr().add(p) };
>     p += 4096;
> }
> std::hint::black_box(sink);
> ```
>
> Probably a combination of macOS's `mmap()` implementation, the old hardware (small TLB, 4 KB-only pages), and contention I never instrumented properly. The Linux numbers above and everything that follows are on the i7-9700K — the macOS path is dead-ended for the rest of this article.


# Where do I go now ?

Time to use tool **Top-down Microarchitectural Analysis (TMA)** : it classifies every CPU cycle into four buckets : retiring (useful work), backend bound, frontend bound, bad speculation, then drills down. On Linux it's available directly via `perf stat`.

`perf stat -M TopdownL1 ./target/release/wcrs test_large.txt` gives :

| Bucket          | %     | Verdict                                  |
| --------------- | ----- | ---------------------------------------- |
| Retiring        | 47.4% | Useful work                              |
| Backend Bound   | 45.1% | Pipeline blocked on execution resources  |
| Frontend Bound  | 6.5%  | Fine                                     |
| Bad Speculation | 1.0%  | Fine                                     |

About half the cycles are doing real work, the other half is mostly stalled in the backend. Branch mispredicts and frontend (instruction supply) are negligible. The interesting thing is what's inside Backend Bound — `-M TopdownL2` splits it :

| Sub-bucket       | %     |
| ---------------- | ----- |
| Memory Bound     | 29.7% |
| Core Bound       | 15.1% |

Memory stalls are about twice port-utilization stalls. That gives an idea which optimisation strategy to opt for. Investigating "memory bound" with `-M TopdownL3` tells me 20% of all cycles are waiting on DRAM :

| Sub-bucket   | %     |
| ------------ | ----- |
| DRAM Bound   | 20.1% |
| L2 Bound     | 7.3%  |
| L1 Bound     | 4.3%  |
| L3 Bound     | 3.6%  |
| Store Bound  | 0.0%  |

Two optimization paths :
- parallelization
- improving SIMD code


# A two-shuffle whitespace classifier

Following the SIMD path first. The dominant part of the inner loop is whitespace detection : 6 `_mm256_cmpeq_epi8` + 5 `_mm256_or_si256` = 11 SIMD operations.

I use a trick found in `simdjson` and similar high-throughput byte scanners : a two-shuffle nibble classifier. Idea : every byte is fully determined by its two nibbles (4 bits each), and `_mm256_shuffle_epi8` does a 16-entry table lookup on the low nibble of each byte in parallel for 32 bytes at once. So we encode "does any whitespace byte have this low nibble?" in one table, "does any whitespace byte have this high nibble?" in another, and a byte is whitespace if and only if both lookups intersect on the same byte.

To make the intersection meaningful, each of the 6 whitespace bytes gets a unique bit :

| byte | bit | mask |
| ---- | --- | ---- |
| 0x09 | 0   | 0x01 |
| 0x0A | 1   | 0x02 |
| 0x0B | 2   | 0x04 |
| 0x0C | 3   | 0x08 |
| 0x0D | 4   | 0x10 |
| 0x20 | 5   | 0x20 |

Then build two 16-entry tables — one keyed by low nibble, one by high :

```
lo_table[low_nibble] = OR of bits whose whitespace byte has this low nibble
  idx :  0    1  2  3  4  5  6  7  8  9    A    B    C    D    E  F
  val :  0x20 0  0  0  0  0  0  0  0  0x01 0x02 0x04 0x08 0x10 0  0
                                            └──────── 0x09..0x0D ──────┘
         └─ 0x20

hi_table[high_nibble] = OR of bits whose whitespace byte has this high nibble
  idx :  0    1  2    3  4  5  6  7  8  9  A  B  C  D  E  F
  val :  0x1F 0  0x20 0  0  0  0  0  0  0  0  0  0  0  0  0
         │       └─ 0x20 (only byte with high nibble 2)
         └─ all of 0x09..0x0D (high nibble 0)
```

The classification is then :

```
is_whitespace(b) = (lo_table[b & 0xF] & hi_table[b >> 4]) != 0
```

Worked example, byte by byte :

```
b = 0x0A :  lo_table[0xA] & hi_table[0x0]  =  0x02 & 0x1F  =  0x02   ->  whitespace
b = 0x20 :  lo_table[0x0] & hi_table[0x2]  =  0x20 & 0x20  =  0x20   ->  whitespace
b = 0x29 :  lo_table[0x9] & hi_table[0x2]  =  0x01 & 0x20  =  0x00   ->  not whitespace
b = 0x00 :  lo_table[0x0] & hi_table[0x0]  =  0x20 & 0x1F  =  0x00   ->  not whitespace
```

The unique-bit assignment is what guarantees that intersection is non-zero only when the same whitespace byte's bit is set in both tables. Without it, byte 0x00 would falsely match (both nibbles map to "something whitespace lives here", but it'd be a different something in each).

In Rust, with the table duplicated across both 128-bit lanes (because `_mm256_shuffle_epi8` operates lane-locally) :

```rust
let lo_table = _mm256_setr_epi8(
    0x20, 0,    0,    0,    0,    0,    0,    0,
    0,    0x01, 0x02, 0x04, 0x08, 0x10, 0,    0,
    0x20, 0,    0,    0,    0,    0,    0,    0,
    0,    0x01, 0x02, 0x04, 0x08, 0x10, 0,    0,
);
let hi_table = _mm256_setr_epi8(
    0x1F, 0,    0x20, 0,    0,    0,    0,    0,
    0,    0,    0,    0,    0,    0,    0,    0,
    0x1F, 0,    0x20, 0,    0,    0,    0,    0,
    0,    0,    0,    0,    0,    0,    0,    0,
);
let nibble_mask = _mm256_set1_epi8(0x0F);
let zero        = _mm256_setzero_si256();

// In the inner loop :
let lo_nib    = _mm256_and_si256(data, nibble_mask);
let hi_nib    = _mm256_and_si256(_mm256_srli_epi16(data, 4), nibble_mask);
let lo_match  = _mm256_shuffle_epi8(lo_table, lo_nib);
let hi_match  = _mm256_shuffle_epi8(hi_table, hi_nib);
let intersect = _mm256_and_si256(lo_match, hi_match);
let is_not_ws = _mm256_cmpeq_epi8(intersect, zero);
let ws_mask   = !(_mm256_movemask_epi8(is_not_ws) as u32);
```

7 SIMD ops for the WS classifier (down from 11). The downstream word-boundary code (`(ws_mask << 1) | prev_ws_bit`, popcount, etc.) is unchanged.

The improvement is almost non-existent, well within error-margin of hyperfine... run-time is 135ms


# Single-shuffle classifier trick

Looking at `lo_table` again :

```
idx :  0    1  2  3  4  5  6  7  8  9    A    B    C    D    E  F
val :  0x20 0  0  0  0  0  0  0  0  0x01 0x02 0x04 0x08 0x10 0  0
```

We notice something : the **non-zero positions are exactly the low nibbles of the whitespace bytes** (positions 9..D for 0x09..0x0D, position 0 for 0x20). The values themselves are arbitrary "unique bits" we picked for the AND-trick to work — they could be anything distinct enough.

What if instead of "unique bits", we put **the whitespace byte's own value** at each non-zero position?

```
ws_table[low_nibble] = the whitespace byte whose low nibble matches, or 0
  idx :  0    1  2  3  4  5  6  7  8  9    A    B    C    D    E  F
  val :  0x20 0  0  0  0  0  0  0  0  0x09 0x0A 0x0B 0x0C 0x0D 0  0
```

Now `_mm256_shuffle_epi8(ws_table, data)` returns, per byte :

```
result_byte = ws_table[data_byte & 0x0F]
```

For each whitespace byte the lookup returns... the byte's own value :

- `0x09 & 0xF = 0x9` -> `ws_table[9] = 0x09` -> equals input
- `0x0A & 0xF = 0xA` -> `ws_table[A] = 0x0A` -> equals input
- `0x20 & 0xF = 0x0` -> `ws_table[0] = 0x20` -> equals input

For non-whitespace bytes, the lookup returns either zero or some whitespace byte that *doesn't* equal the input :

- `0x49 ('I')` -> `ws_table[9] = 0x09 ≠ 0x49`
- `0x40 ('@')` -> `ws_table[0] = 0x20 ≠ 0x40`
- `0x80` (UTF-8 cont. byte) -> high bit set, `_mm256_shuffle_epi8` returns 0 ≠ 0x80
- `0xFF` -> high bit set, returns 0 ≠ 0xFF

So, comparing the lookup result against the original byte directly produces the WS mask. One `_mm256_shuffle_epi8` and one `_mm256_cmpeq_epi8` do everything :

```
data:        [..., 'H',  'e',  'l',  'l',  'o',  ' ',  'w',  'o',  'r',  'l',  'd', '\n', ...]
                    ↓     ↓     ↓     ↓     ↓     ↓     ↓     ↓     ↓     ↓     ↓     ↓
ws_lookup:   [..., 0x20,  ?,    ?,    ?,    ?,   0x20,  ?,    ?,    ?,    ?,    ?,  0x0A, ...]
             (each byte = ws_table[input & 0xF])

cmpeq with data, byte by byte :
                  'H'==0x20  'e'==?  ...  ' '==0x20  ...  '\n'==0x0A
                     no        no            yes              yes
ws_mask:     [..., 0,         0,    0,  0, 0, 1,    0, 0, 0, 0, 0, 1, ...]
```

In code :

```rust
let ws_table = _mm256_setr_epi8(
    0x20, 0,    0,    0,    0,    0,    0,    0,
    0,    0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0,    0,
    0x20, 0,    0,    0,    0,    0,    0,    0,
    0,    0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0,    0,
);

// In the inner loop :
let ws_lookup = _mm256_shuffle_epi8(ws_table, data);
let is_ws     = _mm256_cmpeq_epi8(ws_lookup, data);
let ws_mask   = _mm256_movemask_epi8(is_ws) as u32;
```

Two SIMD operations for the entire whitespace classifier (plus the `_mm256_movemask_epi8` to extract the bitmask, same as before). And `ws_mask` here directly means "is whitespace" — no inversion needed, unlike the two-shuffle version where we had to NOT the cmpeq-against-zero result.

This works because of a property the general two-table classifier doesn't need : in our set, every whitespace byte has a unique low nibble (positions 0, 9, A, B, C, D — no collisions). If two whitespace bytes shared a low nibble, the table couldn't represent both at once and we'd be forced back to the two-table approach. ASCII whitespace happens to be perfectly arranged for this : `0x09-0x0D` walk through low nibbles 9-D, and `0x20` lives alone at position 0.

The result is impressive, run-time is down to 124ms


# Counting without leaving SIMD : the SAD trick

I got this one from fastlwc (https://github.com/expr-fi/fastlwc) : there is a way to convert the `_mm256_movemask_epi8` + `popcnt` chain to SIMD code.

## The trick

The observation : `_mm256_cmpeq_epi8` returns `0xFF` for matches and `0x00` for non-matches. As a *signed* byte, `0xFF` is `-1`. So if we subtract the cmpeq result from a per-byte counter, every match contributes `-(-1) = +1` and every non-match contributes `-0 = 0`.

That gives us 32 independent 8-bit counters living inside a single YMM register. Each iteration just does one `_mm256_sub_epi8` to update them all in parallel.

Walk-through, on an 8-byte mini-version (the real one is 32-byte) :

```
input bytes   :  [ a,  b, \n,  c,  d, \n, \n,  e]

cmpeq with '\n' (signed view) :  [ 0,  0, -1,  0,  0, -1, -1,  0]
                  (hex view)  :  [00, 00, FF, 00, 00, FF, FF, 00]

acc           :  [ 0,  0,  0,  0,  0,  0,  0,  0]   (starts at zero)
acc -= cmpeq  :  [ 0,  0,  1,  0,  0,  1,  1,  0]
```

Each byte lane is its own little 8-bit counter. After another iteration on different data :

```
input bytes   :  [\n,  x,  y,  z, \n,  q,  r,  s]
cmpeq with \n :  [-1,  0,  0,  0, -1,  0,  0,  0]

acc (carried) :  [ 0,  0,  1,  0,  0,  1,  1,  0]
acc -= cmpeq  :  [ 1,  0,  1,  0,  1,  1,  1,  0]
```

Each byte lane is a `u8`, max value 255. So we can run up to 255 iterations before any lane could overflow.

## Flushing : `_mm256_sad_epu8`

After 255 iterations, lane `i` holds "how many newlines fell at position `i` across the last 255 blocks of 32 bytes". To finalise, we need to add all 32 lane counts into one scalar `lines` total.

`_mm256_sad_epu8(acc, zero)` (Sum of Absolute Differences against zero) is built for exactly this. For each 8-byte sub-vector it computes `byte0 + byte1 + ... + byte7` and returns the sum in a 64-bit lane. The 32-byte accumulator collapses to four 64-bit partial sums in one op :

```
acc                       :  [b0, b1, b2, b3, b4, b5, b6, b7,  b8 ... b15,  b16 ... b23,  b24 ... b31]

_mm256_sad_epu8(acc, 0)   :  [b0+b1+...+b7   |   b8+...+b15   |   b16+...+b23   |   b24+...+b31]
                                64 bits           64 bits           64 bits           64 bits
```

Add those four `u64`s together, add to the running `lines` total, reset `acc` to zero, off we go for the next batch. That last step costs ~5 instructions every 8160 bytes — basically free on average.

## In code

```rust
const BATCH:        usize = 255;
const BATCH_BYTES:  usize = BATCH * 32;
let zero = _mm256_setzero_si256();

while i + BATCH_BYTES <= len {
    let mut nl_acc = zero;                 // fresh per-byte counter
    let batch_end = i + BATCH_BYTES;

    while i < batch_end {
        let data   = _mm256_loadu_si256(chunk.as_ptr().add(i) as *const __m256i);
        let nl_cmp = _mm256_cmpeq_epi8(data, newline);  // 0xFF (= -1) per match
        nl_acc     = _mm256_sub_epi8(nl_acc, nl_cmp);   // subtract -> +1 per match
        // ... (word counting unchanged)
        i += 32;
    }

    lines += sum_u64_lanes(_mm256_sad_epu8(nl_acc, zero));
}
```

with the helper :

```rust
unsafe fn sum_u64_lanes(v: __m256i) -> u64 {
    let mut tmp = [0u64; 4];
    _mm256_storeu_si256(tmp.as_mut_ptr() as *mut __m256i, v);
    tmp[0] + tmp[1] + tmp[2] + tmp[3]
}
```

Run-time is now 120ms


# SAD on the word counting path

The same trick can be applied to word counting, with a twist : the word-boundary calculation `(ws_mask << 1) | prev_ws_bit` currently lives in scalar registers, which is why we still need `_mm256_movemask_epi8` + `popcnt` on the word side. To unlock SAD-style counting for words, the carry chain has to move into the YMM domain, using `_mm256_permute2x128_si256` to bridge the two 128-bit lanes and `_mm256_alignr_epi8::<15>` to do the 1-byte left shift, then `_mm256_andnot_si256` to compute `word_starts` directly as a vector. That vector then feeds the same SAD accumulator pattern as the line-counting path.

The code becomes quite complex but run-time is now down to 116ms


# Software prefetching

After the SAD tricks, TMA was showing memory bound at 38.6%, core bound at 16.8%. So memory was now twice the bottleneck of compute and it became obvious I should try software prefetching.

The change is a simple change in the inner loop :

```rust
const PREFETCH_DIST: usize = 768;

while i < batch_end {
    _mm_prefetch::<_MM_HINT_T0>(chunk.as_ptr().add(i + PREFETCH_DIST) as *const i8);

    let data = _mm256_loadu_si256(chunk.as_ptr().add(i) as *const __m256i);
    // ... rest of the loop, unchanged
}
```

`_MM_HINT_T0` brings the line into L1 directly. `_mm_prefetch` is a hint and never faults, so prefetching a few bytes past the end of the `mmap` region is harmless, no buffer overrun.

To tune `PREFETCH_DIST`, I tried different values from 128-2048. 768 and 1024 gave the best results. Below seems to be not enough to hide L2->L1 latency and more may be polluting cache lines that we're not going to use.

The most interesting TMA stat is "L2 bound" lowering from 14% to 5% and an impressive run-time of 104ms


# Going parallel with rayon

At this point, I don't know what to do to improve single-threaded speed. Memory bound is dominating TMA stats, so the only way to get more memory bandwidth is to go multi-thread !

The algorithm splits cleanly. Each `count_avx2` call is independent of the others, it processes a slice and returns `(words, lines)` with a small bit of internal state for the word-boundary carry. To parallelise, give each thread its own slice, run `count_avx2` on it with a fresh `in_word`, and merge the results.

The only subtlety is the cross-chunk word-boundary fix-up. If chunk N ends in non-whitespace and chunk N+1 starts in non-whitespace, the word that spans the boundary gets counted twice, once as "chunk N's last word ending" and once as "chunk N+1's first word starting". Each chunk reports two extra booleans (does it start / end in non-whitespace), and the merge subtracts 1 for every boundary that falls inside a word.

```rust
struct ChunkResult {
    words: u64,
    lines: u64,
    starts_nonws: bool,
    ends_nonws: bool,
}

fn count_chunk(chunk: &[u8]) -> ChunkResult {
    if chunk.is_empty() {
        return ChunkResult { words: 0, lines: 0, starts_nonws: false, ends_nonws: false };
    }
    let mut in_word = false;
    let (words, lines) = count(chunk, &mut in_word);
    ChunkResult {
        words,
        lines,
        starts_nonws: !chunk[0].is_ascii_whitespace(),
        ends_nonws:   !chunk[chunk.len() - 1].is_ascii_whitespace(),
    }
}
```

In `main`, one chunk per thread :

```rust
let n_threads = rayon::current_num_threads().max(1);
let chunk_size = data.len().div_ceil(n_threads).max(1);

let results: Vec<ChunkResult> = data
    .par_chunks(chunk_size)
    .map(count_chunk)
    .collect();

let mut lines: u64 = 0;
let mut words: u64 = 0;
for r in &results { lines += r.lines; words += r.words; }
for i in 1..results.len() {
    if results[i - 1].ends_nonws && results[i].starts_nonws {
        words -= 1;  // boundary inside a word — was counted twice
    }
}
```


## How it scales

`RAYON_NUM_THREADS=N` to sweep :

| Threads | Wall (ms) | User (ms) | System (ms) | Speedup vs 1T |
| ------- | --------- | --------- | ----------- | ------------- |
| 1       | 110.0     | 60.5      | 49.5        | 1.00×         |
| 2       | 77.2      | 78.4      | 52.6        | 1.42×         |
| 3       | 67.4      | 104.8     | 52.7        | 1.63×         |
| 4       | 62.9      | 130.4     | 57.0        | 1.75×         |
| 5       | 59.2      | 153.0     | 59.1        | 1.86×         |
| 6       | 59.1      | 182.9     | 63.2        | 1.86×         |
| 7       | 59.0      | 206.7     | 68.8        | 1.86×         |
| 8       | 60.3      | 215.6     | 72.9        | 1.82×         |

So : ~1.86× peak speedup with 8 cores, with a plateau at 5 threads. Far from linear.

Two things to read out of the table :

- User time scales almost linearly with thread count (60 -> 215 ms), but most of that added user time is threads stalled on memory. The threads are running, but they're spending most of their cycles waiting on DRAM.
- System time climbs from 49 -> 73 ms : more threads = more parallel page faults to handle. We'll see in the pread chapter that this is straight kernel fault-handling cost, not lock contention as I initially assumed.


## TMA after parallelisation

| Bucket          | Single-thread | Multithreaded (8T) |
| --------------- | ------------- | ------------------ |
| **Memory Bound** | 28.9%        | **70.2%**          |
| Core Bound      | 20.8%         | 9.5%               |
| Retiring        | 39.5%         | 14.5%              |
| IPC             | 1.62          | **0.60**           |
| CPUs utilised   | ~1.0          | 4.83 / 8           |

Memory Bound jumped from 29% to 70% of total cycles : two-thirds of all cycles are now spent waiting on memory. Per-core IPC dropped from 1.62 to 0.60 because the cores are mostly stalled. The CPUs-utilised number says that on average ~5 of the 8 cores are doing real work at any given moment.
We're doing 1 GB / 59ms = 17GB/s

Using `mbw` to get practical memory speed on my system :

| Method                                  | Single-thread copy bandwidth |
| --------------------------------------- | ---------------------------- |
| MEMCPY (libc memcpy)                    | 7.6 GB/s                     |
| DUMB (`b[i] = a[i]` plain loop)         | 12.9 GB/s                    |
| MCBLOCK (optimised block memcpy)        | **18.2 GB/s**                |

`mbw` reports *copy* bandwidth — each byte is read and written, so the underlying memory traffic for MCBLOCK is ~36 GB/s of reads + writes. The memory subsystem on this machine is clearly capable of much more than 17 GB/s in absolute terms. Note also that our scanner is *read-only* : there are no stores contending for memory-controller cycles, so the pure-read bandwidth a userspace process can extract is in principle higher than mbw's per-direction figure of 18 GB/s. We'll see this confirmed in the THP chapter below, where we hit ~27 GB/s of pure reads.

I tried a bench by removing the SIMD part of the code (compute), keeping only data reading :

| Threads | wcrs (full work) | Pure-read (no compute) |
| ------- | ---------------- | ---------------------- |
| 1       | 110.0 ms         | 108.5 ms               |
| 2       | 77.2 ms          | 77.6 ms                |
| 4       | 62.9 ms          | 66.3 ms                |
| 8       | 59.6 ms          | 59.8 ms                |

At every thread count, doing nothing per byte is barely faster than doing the full SIMD scan. Total user time across cores is essentially identical in both versions. The compute part is hidden entirely by memory stalls. So we're not at max memory read speed but we are bottlenecked by `mmap()`.


## Beating `cat`

`cat test_large.txt > /dev/null` runs in 100 ms on this machine and is essentially the I/O floor for sequential single-threaded reads. Even our 60 ms `mmap`+rayon version is **significantly below cat** despite doing real work, and the `pread` version below pushes that to 42 ms. Two reasons `mmap`+rayon already beats `cat` :

1. `cat` does one `read()` syscall after another, copying 1 GB through the kernel into a userspace buffer. That copy is its own ~99 ms of system time.
2. We use `mmap()`, which has no copy at all — only the page faults. And we have 8 threads faulting / scanning in parallel, where `cat` has one.

So the "I/O floor" is only a floor for the single-threaded design. Parallelism alone blasts through it, which is why we initially didn't reach for the more involved `pread` rewrite. Then we benchmarked fastlwc and saw it at 38 ms. Time to look closer.


# Another strategy for multi-thread version : replacing `mmap` with `pread`

Out of curiosity I benchmarked [`fastlwc-mt`](https://github.com/expr-fi/fastlwc) on the same warm-cached ext4 file : it does the same algorithmic work (single-shuffle WS classifier, SAD-based counting, the tricks we adapted from it) but uses `pread()` into a 128 KB per-thread buffer instead of `mmap()`. Result : 42ms. But its single-threaded version is 15% slower. Why ?

`mmap()` does less total kernel work per byte : installing PTEs is cheap, way cheaper than memcpy'ing the data. So 1-thread `mmap()` is faster than 1-thread `pread()`. But `mmap()`+rayon does not scale cleanly past 4-5 threads.

My first guess was `mmap_lock` contention. I checked it with `perf record --call-graph dwarf` (kernel symbols resolved via `kptr_restrict=0`). The kernel-side hot path is :

| Kernel category               | % of total samples |
| ----------------------------- | ------------------ |
| Page-fault / page-cache work (`filemap_map_pages`, `next_uptodate_folio`, `xas_load`, `set_pte_range`) | **10.4%** |
| Memcg accounting              | 1.7%               |
| **Lock / sync (all forms)**   | **1.0%**           |
| IRQ / syscall dispatch        | 0.9%               |
| Counters                      | 0.7%               |
| Scheduler                     | 0.7%               |

So the lock-contention hypothesis is wrong : ~1% of total samples is in any lock-related code. The real kernel cost is page-cache traversal during fault handling.

Why does `mmap`+rayon plateau then ? At 8 threads, user-mode cycles grow 3.7x compared to 1 thread (273 M → 1014 M) while wall time only drops 1.86x (102 → 60 ms). The cores are stalled on memory in the SIMD scan side, not waiting in the kernel.

`pread()` ends up faster mostly because the kernel's `copy_to_user` streams into a per-thread 128 KB buffer that fits in L2, giving the SIMD scan a tighter, cache-friendly working set than the page-cache -> page-table -> SIMD-load chain that `mmap()` produces.

The THP chapter below breaks the same ceiling from the other side : keep `mmap()`, switch the source file to huge pages, and the SIMD scan inherits a much cache- and TLB-friendlier access pattern without rewriting the I/O path.

Once the asymmetry was clear, I rewrote the I/O path to match. `count_avx2` and the SIMD inner loop are untouched; only `main` and the per-thread function change. Each rayon worker now :
- Pre-allocates one shared anonymous huge-page-backed buffer pool (one big `mmap()` up-front, split into 2 MB-aligned per-worker slots) instead of one `mmap()` per task. One up-front allocation instead of N in parallel, and each worker gets a TLB-cheap, L2-friendly destination buffer for `copy_to_user`.
- Loops `pread(fd, slot, 128 KB, offset)` -> SIMD scan -> next chunk through its assigned byte range.
- Returns the same `ChunkResult` (words, lines, plus first/last non-whitespace booleans), and the cross-thread word-boundary fix-up is identical to before.

Run-time is 42ms on my setup, exact same time as fastlwc-mt.

`mmap()` was a simpler approach but `pread()` is the right choice here since THP is a bit of a pain to use.


# Improving the `mmap` version using Transparent Huge Pages

Out of curiosity I pushed past the "`mmap()` version is done" line into system configuration. The 50 ms of system time we kept seeing was 16 K page-fault batches at 4 KB granularity. With 2 MB transparent huge pages that becomes ~512 fault entries, ~30x fewer kernel trips, with the same data delivered.

It took some setup and I had several surprises. THP has to be activated and file-backed THP is not supported by ext4, so I had to run my tests on a filesystem supporting it.

Once set, the results were impressive : run-time down to 37 ms.

To make sure I understood where the gain came from, I measured the same binary at 1 thread on the 4 KB-paged ext4 file vs the 2 MB-paged tmpfs file, with `perf stat` capturing fault count, TLB-miss count and page-walker activity :

| Metric                          |      (4 KB)    |             (2 MB) | Δ            |
| ------------------------------- | -------------- | ------------------ | ------------ |
| `minor-faults`                  | 16 086         | **586**            | -27×         |
| `dTLB-load-misses`              | 830 539        | 18 733             | -44×         |
| `dtlb_load_misses.walk_active`  | 21.3 M cycles  | 0.7 M cycles       | -30×         |

Two findings :
- 80 % of the THP gain at 1 T comes from the kernel side : fewer faults to install. Per-fault kernel time is similar in both cases (~2.5 µs), so the savings come from doing 27x fewer of them, not from each one being faster.
- 20 % of the gain is on the user side : TLB pressure collapses (44x fewer dTLB misses, 30× fewer page-walker cycles), the SIMD scan stops paying TLB miss penalties, and the file's contiguous 2 MB physical chunks give better DRAM row-buffer locality. Real but secondary.


# Final progression and conclusion

| Step                                                         | Wall      | vs naive Rust |
| ------------------------------------------------------------ | --------- | ------------- |
| `/usr/bin/wc`                                                | 3.879 s   | -             |
| Naive Rust                                                   | 2.372 s   | 1.00×         |
| + release flags + target-cpu=native                          | 2.055 s   | 1.15×         |
| + buffered IO + single pass                                  | 1.822 s   | 1.30×         |
| + memchr line counting                                       | 1.627 s   | 1.46×         |
| + branchless word counting (lost)                            | 1.627 s   | 1.46×         |
| + SIMD word counting                                         | 157 ms    | 15×           |
| + SIMD line counting (combined loop)                         | 154 ms    | 15×           |
| + `mmap()` (Linux)                                           | 137 ms    | 17×           |
| + two-shuffle WS classifier                                  | 135 ms    | 18×           |
| + single-shuffle WS classifier                               | 124 ms    | 19×           |
| + SAD line counting                                          | 120 ms    | 20×           |
| + SAD word counting (vector carry)                           | 116 ms    | 20×           |
| + software prefetch                                          | 104 ms    | 23×           |
| + rayon                                                      | 57 ms     | 42×           |
| **+ `pread()` + huge-page user buffer**                      | **42 ms** | **56×**       |
| + tmpfs + huge pages *(requires THP support)*                | 37 ms     | 64×           |

**~56× speedup over the naive Rust baseline, ~92× over `/usr/bin/wc`** on the same machine, entirely from Rust-level work, on plain ext4, with no system tuning.

56x speedup through a chain of small wins : LLVM auto-vectorisation, AVX2 intrinsics, a single-shuffle classifier, the SAD counting trick, software prefetch, rayon, and `pread()` replacing `mmap()` as the I/O path that actually scales past 4 cores. 
The work was mostly picking the right next step, and TMA (`perf stat -M TopdownL1/L2/L3`) was the tool to use. 
Several of my own hypotheses turned out wrong (branchless word counting, `mmap()`-is-strictly-faster, single-thread compute being the bottleneck) ; only the measurements caught them.
The final binary ties the C reference (`fastlwc-mt`) on plain ext4 with ~10% less total CPU, which is a reasonable place to stop.

Why bother making `wc` faster ? It's an interesting exercise on something small enough to keep it in your head, it touches every layer of the stack (compiler auto-vectorisation, SIMD intrinsics, cache hierarchy, branch prediction, kernel I/O paths, memory controller, parallelism...), and the same kind of recipes can apply to other streaming tasks like base64 decode, JSON scanning, CSV field counting...

Full source code on GitHub : https://github.com/olivierbabasse/wcrs
