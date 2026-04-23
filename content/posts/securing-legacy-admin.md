+++
date = '2026-04-23T11:52:24+02:00'
title = 'Securing a legacy server administration webpage'
series = ["Securing legacy admin page"]
series_order = 1
+++

# Porting a legacy setuid-root admin helper to modern Linux

Here is the line that used to sit in `/etc/sudoers.d/legacy-admin` on the appliance we're working on :

```
webapp ALL=(root) NOPASSWD:SETENV: /opt/appliance/bin/admin_root.php
```

The said appliance ships a server administration page — the kind geared toward non-tech-savvy users who need to set a hostname, configure a network, mount or format a disk, or reboot. Under the hood, every one of those operations requires root. For the past twenty years, "requires root" meant one line in /etc/sudoers.d/ that made a single PHP script runnable as root, without a password, by whatever user the web UI runs as.
This series is about replacing that line with something that has, in 2026, no real excuse not to exist: a small web service that runs as an unprivileged system user, delegates most of what it used to do as root to system daemons over D-Bus, authorizes each call through polkit, and funnels the residue through a single narrow root helper — all of it inside a hardened systemd unit whose writable paths fit on one line.


## The one-line hook

Back to our sudoers one-liner, three things are wrong with it :

- **`NOPASSWD`** — no re-authentication when the web UI becomes the operating system.
- **`SETENV`** — the caller's environment survives the `sudo`. The callee inherits `PATH`, `LD_PRELOAD`, `LD_LIBRARY_PATH`, everything.
- **One script dispatching roughly forty privileged operations** — hostname, network, services, reboot, disks, RAID, OS upgrade, licence install, password reset, backup, restore, factory reset. The blast radius is whatever the union of all those do.

If you squint, you can see the attack already. One path-controllable binary invocation anywhere inside that script, plus a `PATH=/tmp/pwned` from the caller, plus `SETENV`, equals root. It turns out there is one.


## The history of how we got here

Before we criticise the helper, credit where it's due: I don't know the exact and full story of this code, but it was written around 2000, on FreeBSD, almost certainly as a plain setuid binary. It was ported to Linux, starting around 2008, and the `sudo NOPASSWD: SETENV:` rule was the glue added to make it work on the new OS.

Polkit — the piece we'll use to replace a lot of what the helper did — was released in 2008. Systemd's `Protect*` / `Private*` hardening directives landed after 2010. `systemd-run` as a usable transient-unit spawner came later still. *Almost none of the machinery that replaces this helper existed when it was written.*

So the story that follows is not "we/they were idiots." It's "the tools got better, and an architectural pattern — one big privileged script behind a big privileged sudo rule — outlived the era it made sense in." The one thing I will criticise by name is the `SETENV` tag on the Linux port, because that is the specific thing that turned a tired design into a ladder to root.


## What the helper did, and the one attack that explains the rest

Forty-ish operations, one script. To save us scrolling through an inventory, here is the single worst line — the one that made the end-to-end compromise trivial once any upstream bug existed:

```php
// admin_root.php — invoked as root via sudo on the Linux port
function setOwner(string $path, string $owner): void {
    // 'chown' is unqualified — $PATH decides which binary runs.
    xexec("chown $owner:$owner " . escapeshellarg($path));
}
```

`chown` has no leading slash. On FreeBSD, with a clean setuid environment, that was fine — `execvp`-style PATH lookup resolved to the real `/usr/sbin/chown`. On Linux, once `SETENV` was on the sudo rule, the caller chose `PATH`.

End-to-end, the chain reads like this:

1. A factory-default admin password that nobody rotated in the field.
2. An authenticated shell injection in an unrelated PHP page — the kind of thing an `escapeshellarg` omission introduces in a single commit. `RCE as webapp`.
3. A local pivot from `webapp` to any account that can run `sudo` — often `webapp` itself, sometimes one hop away.
4. `PATH=/tmp/pwned sudo /opt/appliance/bin/admin_root.php …` — `SETENV` survives.
5. The script calls `chown` unqualified. `/tmp/pwned/chown` runs as root.
6. Game over.

And this was not the only chain that existed at that time, we found other easy ones too.

The helper was never the security boundary. The boundary was a hope that the web front-end above it would never misuse it. That hope was more defensible when the helper was written, when shell injection was less well understood and when the PHP stack it ran behind had a smaller surface. It doesn't hold up now.


## The design principles of the replacement

Four rules :

1. The web service does **not** run as root.
2. Every privileged operation is **individually** authorized.
3. Where the OS already ships a privileged daemon for the operation — systemd, NetworkManager, UDisks2, hostnamed, timedated, logind — we talk to it over D-Bus rather than re-implementing the operation ourselves as root.
4. For the residue that cannot go through D-Bus (editing `/etc/systemd/resolved.conf`, running `chpasswd`, installing a file on behalf of a specific user), we first decided to funnel everything through **one** narrow, statically typed helper that reads one JSON command on stdin and writes one JSON response on stdout. No shell, ever. Same sandbox as the caller. Later, when tightening systemd sandbox security for our service, we changed our mind and replaced our helper with another system service. More on that later.


## The picture

The architecture as we designed it first :

```
  browser ──HTTPS──▶ serveradmin (user: serveradmin, sandboxed systemd unit)
                        │
                        ├─ D-Bus ──▶ systemd / logind / hostnamed /
                        │            timedated / NetworkManager / UDisks2
                        │           (polkit gates each action per uid)
                        │
                        └─ sudo ──▶ serveradmin-helper (user: root,
                                    inherits parent's mount namespace;
                                    JSON-on-stdin, JSON-on-stdout)
```

Two paths. Most of the privileged operations the appliance needs are served by the top one — `serveradmin` makes a D-Bus call, the target daemon asks polkit "is uid `serveradmin` allowed this action?", polkit says yes, the operation happens. The word "root" does not appear in our code path at all. The remaining go through the helper, which runs as root, yes, but inside the same sandboxed filesystem view as its parent, and with argument parsing that is statically typed by a Rust tagged enum instead of by the shell.

That is the whole architecture. The rest of this post is a tour of what each piece does and why. Each section will be the subject of a separate article.


## A quick tour of each mechanism

### 1. D-Bus to system daemons

Instead of shelling out to `systemctl`, `hostnamectl`, `nmcli`, `timedatectl`, or `mount`, the admin service calls the real daemon directly on the system bus. Restarting a unit becomes this:

```rust
proxy.restart_unit(unit_name, "replace").await?;
```

Arguments are typed. No shell, no `escapeshellarg`, no argv parsing. Errors come back as structured D-Bus faults (`NoSuchUnit`, `UnitMasked`, …) instead of as stderr you have to parse. And — crucially — the target daemon asks polkit to authorize the call, which means authorization is owned by a single dedicated daemon rather than scattered across an `exec` pattern in forty places.

Six daemons account for almost everything an appliance admin UI actually does: `org.freedesktop.systemd1` (services), `login1` (reboot/poweroff), `hostname1` (hostname), `timedate1` (clock and NTP), `NetworkManager` (everything IP), `UDisks2` (mount / unmount / SMART).


### 2. Polkit

Every one of those system daemons publishes a list of *action ids* — small strings like `org.freedesktop.systemd1.manage-units` or `org.freedesktop.login1.reboot`. Polkit is the system-wide daemon that answers, for each action id, "is caller X allowed to do this?".

Our rule file is short enough to quote in one paragraph. It says: the user `serveradmin` may do these specific action ids, and only these:

```javascript
// /etc/polkit-1/rules.d/50-serveradmin.rules
polkit.addRule(function(action, subject) {
    if (subject.user !== "serveradmin") return;

    var allowed = [
        "org.freedesktop.hostname1.set-static-hostname",
        "org.freedesktop.systemd1.manage-units",
        "org.freedesktop.login1.reboot",
        "org.freedesktop.login1.reboot-multiple-sessions",
        "org.freedesktop.login1.reboot-ignore-inhibit",
        "org.freedesktop.NetworkManager.settings.modify.system",
        "org.freedesktop.timedate1.set-ntp",
        "org.freedesktop.udisks2.filesystem-mount-system",
        // …
    ];
    if (allowed.indexOf(action.id) !== -1) return polkit.Result.YES;
});
```

Compare with the legacy sudoers line at the top of this post: that rule said "run this entire script as root, with the caller's environment." This rule says "this specific user may do these specific operations, and nothing else." Adding a new privileged operation means adding a new action id to the list, not extending what a single binary is allowed to do.


### 3. Systemd unit hardening

The admin service runs as an unprivileged user under a sandboxed unit. The relevant directives are short:

```ini
ProtectSystem=strict
ReadWritePaths=/opt/serveradmin/log /opt/serveradmin/etc/tls /etc /var/log
ProtectHome=true
PrivateTmp=true
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictNamespaces=mnt
LockPersonality=yes
RestrictRealtime=yes
MemoryDenyWriteExecute=yes
RestrictSUIDSGID=yes
```

`ProtectSystem=strict` makes the entire filesystem read-only for this service, except for the paths listed in `ReadWritePaths`. Easy to audit.

The piece worth internalising — the thing that makes the whole architecture work — is that **the sandbox is inherited by child processes, including ones spawned via `sudo`**. When the admin service invokes the root helper, the helper runs as real uid 0, with full capabilities, but it sees the same read-only `/usr` its parent sees. A lot of post-exploit persistence (dropping a binary in `/usr/local/sbin`, swapping out `/usr/bin/ssh`, editing an init script) doesn't land, because those paths are not writable no matter what your uid is.


### 4. `systemd-run` as an escape hatch

One operation that can't fit inside the sandbox: system upgrade. If everything is inherited, how does `apt-get dist-upgrade` ever work? It needs to write `/usr`, `/lib`, `/boot`, `/var/lib/dpkg` — all blocked by `ProtectSystem=strict`. And `sudo` doesn't help: the mount namespace that enforces this sandbox follows the *process*, not the uid.

The escape hatch is `systemd-run`:

```rust
Command::new("/usr/bin/systemd-run")
    .args(["--wait", "--pipe", "--collect", "--service-type=exec"])
    .arg("/opt/serveradmin/libexec/upgrade_os.sh").status()?;
```

`systemd-run` asks PID 1 to spawn a *new*, transient sibling unit with default (unrestricted) properties. The child is not a descendant of our sandbox — it's a descendant of PID 1 — so our namespace doesn't propagate. `--wait --pipe` streams its stdout back to the caller so we can show upgrade progress to the user. `--collect` cleans up the transient unit when it exits, even on failure.

This is an *intentional* hole in the sandbox, narrowly scoped to the one operation that genuinely needs it. The other 99% of the time, the sandbox holds.


### 5. Linux namespaces (with `mount` as the worked example)

Linux has eight namespace types: `mnt`, `uts`, `ipc`, `pid`, `net`, `user`, `cgroup`, `time`. Systemd's `Protect*` and `Private*` directives quietly set several of them up on your behalf. In our case, the one that affects day-to-day operations is `mnt`.

The admin service runs in its own mount namespace. If the root helper — which inherits that namespace — just calls `mount /dev/vdb1 /mnt/recording`, the mount succeeds inside the namespace, but the rest of the system never sees it, and it vanishes when the helper exits. The fix is to enter PID 1's mount namespace for the syscall:

```bash
nsenter -t 1 -m mount -o noatime /dev/vdb1 /mnt/recording
```

`RestrictNamespaces=mnt` in the unit file is what permits this: we may *join* the mount namespace of another process, but we may not create a fresh one of our own, or join any namespace of any other type. Minimum necessary.


### 6. The narrow helper

One sudoers line:

```
serveradmin ALL=(root) NOPASSWD: /opt/serveradmin/sbin/serveradmin-helper
```

One binary. No `SETENV` tag — so `sudo` scrubs the environment down to its default whitelist. No wildcards. No script — the binary is a native Rust executable, so there's no interpreter to subvert via `PHPRC` or `PYTHONSTARTUP` or `PERL5OPT`.

Inside the binary, the protocol is one JSON value on stdin, one JSON value on stdout, then exit. The request type is a Rust tagged enum, which means argument parsing *is* argument validation:

```rust
#[derive(serde::Deserialize)]
#[serde(tag = "cmd", rename_all = "snake_case")]
enum HelperCmd {
    InstallLicence { path: PathBuf },
    SetResolvedDns { servers: Vec<IpAddr> },
    ChangePassword { user: String, password: String },
    Upgrade,
    // …
}
```

`IpAddr` parses or errors out. `PathBuf` goes through `canonicalize()` and an allowlist before it's used. A new helper command means a new enum variant; the compiler flags every site that has to handle it.

Crucially, the helper is not always root for the whole of its work. For appliance licence specifically, we have to let the user install a signed file. The helper enters as root, `fork()`s, the child `setuid()`s to an unprivileged user, *then* reads and signature-verifies the licence blob. Only the parent — which never touched the attacker-influenced bytes — does the final `rename()` into `/opt/appliance/etc/`. Root is used for the syscalls that require root, and never for parsing.


### 7. Web-layer auth

Even setting aside the sudoers line, the legacy web auth was its own set of problems. `$_SESSION['UserPass'] = $password;` — the plaintext password kept for the lifetime of the session, replayed as HTTP Basic (`CURLOPT_USERPWD`) on every backend call, rendered into a page in one place so that any XSS would exfiltrate it. No CSRF token. No rate limit on login. Cookie flags whatever the PHP defaults gave you.

The rewrite authenticates once via PAM, zeroises the password on drop, uses an `HttpOnly` / `SameSite=Strict` / `Secure` session cookie, requires a per-session CSRF token on every mutating request (compared in constant time), rate-limits logins per IP with a sliding window, tracks session timeout with a monotonic clock so NTP steps and `date` commands can't extend or shorten a session, and emits a structured audit-log line per mutating request.

The gap between the two designs is not one subtle improvement; it's seven independent improvements that happen to travel together on appliances from this era.


## Alternatives considered and rejected

Nothing in the architecture above is especially new. Most of what's interesting is what we *didn't* do. Nine alternatives deserve a paragraph each.

**Just tighten the sudoers and keep PHP.** The `SETENV` and the wildcard are the surface, not the disease. The underlying problem is that PHP plus shell interpolation plus blanket root is an architectural mismatch — every new feature reintroduces injection risk, and `escapeshellarg` is only correct if you remember to call it on every interpolation. A tighter sudoers leaves both the host language and the `exec` pattern in place; you'd be playing whack-a-mole for years.

**Run the whole service as root and skip polkit/D-Bus.** Simpler, larger blast radius. Every handler becomes security-critical. The whole point of the rewrite is that most operations are things systemd / NetworkManager / UDisks2 / hostnamed already know how to do safely; delegating is cheaper than owning the logic as root ourselves, and the authorization conversation with polkit is also the audit conversation.

**Use a setuid binary — classic Unix.** Setuid has no notion of per-action authorization (the binary runs with all the target uid's power from the first syscall) and no native audit trail. It is still the right tool for the one narrow helper, with `sudo` playing the role. It is not the right tool for forty unrelated privileged features.

**Grant Linux capabilities (`CAP_NET_ADMIN`, `CAP_SYS_ADMIN`) instead of root.** For a *narrow* daemon — one that binds a privileged port and nothing else — `CAP_NET_BIND_SERVICE` is beautiful. For a generalist admin surface, the union of capabilities required is effectively `CAP_SYS_ADMIN`, which is "root-equivalent for anything interesting." No real reduction. No polkit audit trail.

**Container / LXC / Docker isolation.** The appliance is a single-host product. Adding a container would buy filesystem and pid isolation we already get from systemd's `Protect*` / `Private*` family, at the cost of a whole new deployment model. Overkill here.

**Use Cockpit.** Cockpit solves an identically-shaped problem. It should be the first option you reach for on greenfield. It wasn't chosen here because the admin surface is product-specific (licence install, domain-specific RAID tooling, service orchestration) and lives alongside existing in-house Rust services. A small custom service sharing the existing ecosystem was less integration work than shoehorning Cockpit plugins into it. If you're starting today with no ecosystem constraint, read the Cockpit docs first.

**SELinux or AppArmor policies on top of the legacy PHP.** MAC policies can restrict what the PHP helper *can* do if it's exploited, but writing a policy that's both tight enough to matter and loose enough not to break legitimate flows is a full-time job, and the underlying injection paths still exist. Mitigation, not elimination. Still a reasonable temporary hardening step if you can't rewrite tomorrow.

**Go or Python instead of Rust for the helper.** Rust is now the house language, phasing out C and PHP code. Consistency won. For a privileged helper specifically, memory safety, no implicit exceptions, and strong typing of the JSON protocol are genuinely useful — parsing attacker-influenced JSON into a tagged enum *is* most of the validation story. Go would have been fine. Python drags in a large stdlib attack surface for a minimal-surface binary.

**Keep the legacy PHP UI and rewire only the privileged helpers.** This is a legitimate halfway house, and if you have a large PHP surface you can't afford to rewrite in one go, it's probably the right move. Keep the PHP pages, keep the API endpoints, but replace the privileged orchestration behind them with calls to a new Rust service over HTTPS. The only reason the admin UI here went all the way to a new React SPA is that the admin surface was also gaining new features that were awkward to bolt onto the old page model. Don't let a want for new UI features drag a necessary security rewrite further than it has to go.


## A template for your own appliance

If you're reading this because you have a sudoers file that looks like ours, here's a checklist you can run against your own codebase this afternoon :

1. Audit carefully your `/etc/sudoers.d/` files.
2. Inventory what each such helper actually does. For each operation, check whether a system D-Bus daemon already does it. Most of what appliance admins do is already owned by systemd, NetworkManager, UDisks2, hostnamed, timedated, or logind.
3. Move the front-end to an unprivileged system user. Put a systemd unit around it with `ProtectSystem=strict` and an explicit `ReadWritePaths=`.
4. Take the residue — the operations that genuinely need to edit a config file as root — and funnel them through one typed helper with a JSON allowlist or move them to a new D-Bus daemon.
5. Authorize the D-Bus side with polkit per action id, not per binary.
6. Add middleware-level audit logging from day one.
7. While you're in there: audit the web auth. Plaintext passwords in the session, missing CSRF, missing rate limit, loose cookie flags — these travel with sudoers `SETENV` for the same "historical reasons".

Two commands to run right now :

```
systemd-analyze security <your-web-admin>.service
pkaction --verbose
```

The first tells you what directives you're missing in your unit file. 
The second enumerates the polkit action ids your system already publishes : evaluate how much of what your admin helper does is a reinvention.


## The rest of this series

**D-Bus IPC**
**Polkit authorizations**
**systemd security**
**systemd-run**
**linux namespaces**
**helper**
**web auth**
**evolving our helper into a service**
**what's left to add or try**

The opener stands alone; the rest will land as I write them.
