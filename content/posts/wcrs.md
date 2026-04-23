+++
date = '2026-04-08T09:51:22+02:00'
draft = false
title = 'Optimizing a simple word count tool (WC-style) implementation in rust'
+++

# Context

System `wc` is the kind of tool you assume is already fast because it's been shipping on Unix for 50 years, but it turns out that on modern hardware a careful implementation can beat it by an order of magnitude.
I started with a naive rust implementation of the default mode of the `wc` challenge (see https://codingchallenges.fyi/challenges/challenge-wc) and tried to improve performance, benchmarking against a naive implementation and the system `wc` command.
I decided to work on ASCII mode only to keep things simple and focused, counting chars, words and lines.
Benchmarking will be done using `hyperfine` with 3 warmups and 10 runs on a 1GB test file to have meaningful execution times.

Spoiler : I reached an **11x speedup** on old hardware and **40x** on a more modern one, the limiting factor being **available memory bandwidth**.


# Reference implementation

I did my tests on my old MacBook Pro early 2015 (Broadwell i5-5287U, macOS 12.7.6) :
reference implementation `/usr/bin/wc` : 3.645s

In the last steps I also used a more modern Linux machine (i7-9700k, linux kernel 5.15) :
reference implementation : 4.806s (yes, it's slower than on my old MacBook, unbelievable !)


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

Run time : 3.233s, which is already a bit faster than the macOS reference implementation


# Usual release optimization options and CPU native code

Adding the usual release options to Cargo.toml brings almost no gain :

```toml
[profile.release]
codegen-units = 1
lto = "fat"
panic = "abort"
debug = true
strip = false
```

I don't gain any visible performance benefit yet.
Note that I keep debug info in the binary to have readable flamegraph/samply info.
The next step is to target the CPU native instruction set via .cargo/config.toml :

```toml
[build]
rustflags = ["-Ctarget-cpu=native"]
```

| Machine | Run time |
| ------- | -------- |
| MBP     | 3.002s   |
| Linux   | 2.319s   |


# Buffered IO and single pass

Adding some timing info shows allocating memory + reading is around 21%, line counting pass 7% and word counting pass 72%.
We'll start by reading data in 64KB chunks (avoiding the large alloc) and counting lines and words in a single pass.

Why 64 KB ? I tried with less (8, 16, 32), I tried with more (128... 1MB) and 64KB seems like a sweet spot (it's a tiny bit faster). It's certainly related to hardware, in particular the cache size.

Way less memory use and a 22% improvement mostly due to less allocating and doing one pass instead of two.
MBP run time : 2.347s


# Improving newline counting with memchr

An easy first win : the `memchr` crate provides a `memchr`-like function using a SIMD implementation working on multiple bytes at once (32 with AVX2)

MBP run time goes down to 2.015s, a 14% improvement


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

I tried to replace the "if" with branchless code because when branch prediction is not good, the CPU can waste a lot of time, but it did not improve things, even a bit slower on MBP : 2.129s

```rust
for &b in chunk {
    let is_ws = b.is_ascii_whitespace();
    words += (!is_ws && !in_word) as u64;
    in_word = !is_ws;
}
```

The explanation might be that we now have an additional eval/add sequence in every iteration and also because branch prediction is very accurate in this case (most bytes are not word separators).


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

Looking at the `is_ascii_whitespace()` implementation, we can see we have 6 possible word separators :
```rust
pub const fn is_ascii_whitespace(&self) -> bool {
    matches!(*self, b'\t' | b'\n' | b'\x0C' | b'\r' | b' ')
}
```

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
ws:           0 0 0 0 0 1 0 0 0 0 0 1 0 0 0 0 1 0 0 1 0 1 0 0 0 0 1 1 1 0 0 0
not_ws:       1 1 1 1 1 0 1 1 1 1 1 0 1 1 1 1 0 1 1 0 1 0 1 1 1 1 0 0 0 1 1 1
prev_ws:    ? 0 0 0 0 0 1 0 0 0 0 0 1 0 0 0 0 1 0 0 1 0 1 0 0 0 0 1 1 1 0 0 0  (ws shifted right by 1)
word_start:   ? 0 0 0 0 0 1 0 0 0 0 0 1 0 0 0 0 1 0 0 1 0 1 0 0 0 0 0 0 0 1 0 0  (not_ws AND prev_ws)
```

Last, we just have to count the 1-bits with `count_ones()` and forward the `in_word` state to the next block (it's the last bit of the block).

The remaining data (<32 bytes) goes through the sequential implementation

Our SIMD implementation now looks like this :
```rust
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
unsafe fn count_words_avx2(chunk: &[u8], in_word: &mut bool) -> u64 { unsafe {
    let mut words: u64 = 0;
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

The result is impressive with a run time of **273ms**, almost **6 times faster** with this step alone !


# Integrating line counting within word counting loop

Instead of reading memory twice (once with `memchr`, once with our SIMD loop), I did it all in the SIMD loop.
I reused the newline vector to directly count the number of newlines in the input (`_mm256_cmpeq_epi8`/`_mm256_movemask_epi8`/`count_ones`).

```rust
let nl_cmp = _mm256_cmpeq_epi8(data, newline);
let nl_mask = _mm256_movemask_epi8(nl_cmp) as u32;
lines += nl_mask.count_ones() as u64;
```

The gain is very small this time, with a run time of 260ms (with a 10ms error margin according to `hyperfine`, so I may even have gained nothing).
Testing with a larger test file might help here.

| Machine | Run time |
| ------- | -------- |
| MBP     | 273ms    |
| Linux   | 168ms    |


# Replacing buffered reading with memory mapping

Quite straightforward using the `mmap` crate :

```rust
let file = File::open(&filename)?;
let mmap = unsafe { Mmap::map(&file)? };
let data: &[u8] = &mmap;
let bytes = data.len() as u64;
...
```

Contrary to what I expected, using `mmap()` instead of `read()` made it almost 2 times slower on my MacBook
It might be due to the hardware (old and small TLB, small 4K memory pages) and to the macOS `mmap()` implementation.

I tried the same thing on my more "modern" linux machine and I could see some small improvements, going from 168ms to 156ms. I then tried to improve things using `madvise()` but the results did not improve.

| Machine | Run time |
| ------- | -------- |
| MBP     | ~500ms   |
| Linux   | 156ms    |


# Parallel processing with rayon

Even though `mmap()` makes things way slower at least on my MacBook, it's easier for a first implementation of parallel processing, so I'll use it to map the file in memory.

Then I divided the data into chunks (one per available thread) and applied counting functions to them :

```rust
fn count_chunk(chunk: &[u8]) -> ChunkResult {
    if chunk.is_empty() {
        return ChunkResult { words: 0, lines: 0, first_is_nonws: false, last_is_nonws: false };
    }
    let mut in_word = false;
    let (words, lines) = count(chunk, &mut in_word);
    ChunkResult {
        words,
        lines,
        first_is_nonws: !chunk[0].is_ascii_whitespace(),
        last_is_nonws: !chunk[chunk.len() - 1].is_ascii_whitespace(),
    }
}
```

When merging results, I had to pay attention to words which were split apart (chunk n ends with non-whitespace and chunk n+1 starts with non-whitespace) :

```rust
let results: Vec<ChunkResult> = data
    .par_chunks(chunk_size)
    .map(count_chunk)
    .collect();

let mut lines: u64 = 0;
let mut words: u64 = 0;
for r in &results {
    lines += r.lines;
    words += r.words;
}
for i in 1..results.len() {
    if results[i - 1].last_is_nonws && results[i].first_is_nonws {
        words -= 1;
    }
}
```

I got a run-time of around 256ms on the MacBook, almost the same thing as before mmap + rayon.
Using the `mbw` tool, I can see a max memory bandwidth of 4.8GB/s for my computer and we're actually using 4GB/s

On linux, it's running almost 3 times faster than before (58ms), but we have 8 real cores on this machine, so it's not scaling linearly. In fact, we likely are already **reaching the memory bandwidth limit** : I calculated an effective bandwidth of 17.6GB/s during the test, for a maximum of 20GB/s according to the `mbw` tool.

TODO : test it on the same hardware with linux kernel 6.0 for large page cache support, it could improve things (less page fault overhead maybe ?)

| Machine | Run time |
| ------- | -------- |
| MBP     | 256ms    |
| Linux   | 58ms     |


# Conclusion

Having reached almost **80-85% of memory bandwidth** for both machines, it's a bit hard to go further without kernel tweaking.
Implementing UTF-8 support could make an interesting follow-up, putting more pressure on the CPU.

Progression across all steps :

| Step                            | MBP     | Linux   |
| ------------------------------- | ------- | ------- |
| `/usr/bin/wc`                   | 3.645 s | 4.806 s |
| Naive                           | 3.233 s | —       |
| + release flags, target-cpu     | 3.002 s | 2.319 s |
| + buffered IO, single pass      | 2.347 s | —       |
| + `memchr` for newline counting | 2.015 s | —       |
| + SIMD word counting            | 273 ms  | —       |
| + SIMD line counting            | 273 ms  | 168 ms  |
| + `mmap`                        | ~500 ms | 156 ms  |
| + `rayon` parallelisation       | 256 ms  | 58 ms   |

Let's just celebrate the current gains compared to the naive implementation :

| Machine          | Naive   | Final  | Speedup |
| ---------------- | ------- | ------ | ------- |
| 2015 MacBook Pro | 3.00 s  | 256 ms | 11.72×  |
| i7-9700K Linux   | 2.32 s  | 58 ms  | 40.25×  |

Full source code on GitHub : https://github.com/olivierbabasse/wcrs
