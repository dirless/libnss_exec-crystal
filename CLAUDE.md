# libnss_exec-crystal

A glibc NSS module written in Crystal that delegates passwd, group, and shadow lookups to an external script (`/sbin/nss_exec`). Crystal port of [tests-always-included/libnss_exec](https://github.com/tests-always-included/libnss_exec).

## Critical architecture note: no GC, no Crystal runtime

v3.0.0 is a complete rewrite using **only raw C library calls**. No Crystal GC, no stdlib, no hidden allocations. This is required because glibc loads NSS modules via `dlopen()` — Crystal's runtime initializers (GC, fiber scheduler, signal handlers) never run, causing segfaults on the first string allocation if you use them.

The entire module is a **single file** (`src/libnss_exec.cr`) that:
- Parses colon-delimited NSS entries via `strtoul`/`strtol` and manual field splitting
- Writes all output into glibc's caller-provided buffer (zero heap allocations)
- Executes `/sbin/nss_exec` via `fork`/`execve` (no shell, no injection risk)
- Uses wrapping arithmetic (`&+`, `&-`) throughout to prevent overflow traps
- Exports 14 standard NSS entry points (`_nss_exec_getpwnam_r`, etc.)

**Never add Crystal stdlib usage to this file.**

## Language / stack

- Crystal >= 1.0.0 (used only as a C-like language — no stdlib)
- Linux with glibc only
- Compiles to `libnss_exec.so.2`

## Key entry points

| File | Purpose |
|------|---------|
| `src/libnss_exec.cr` | Entire module — all 14 NSS entry points |
| `examples/nss_exec.sh` | Example lookup script for `/sbin/nss_exec` |
| `test/generate_test_data.sh` | Generates randomized test data (users, groups) |
| `test/stress_test.sh` | Comprehensive stress test (with and without root) |
| `TEST_PLAN.md` | Full installation and test walkthrough |

## Build & install

```sh
make                    # → libnss_exec.so.2
make symbols            # verify exported NSS entry points
sudo make install       # installs to /lib/
sudo cp examples/nss_exec.sh /sbin/nss_exec && sudo chmod 755 /sbin/nss_exec
# Add 'exec' after 'files' in /etc/nsswitch.conf
getent passwd testuser  # verify
```

## Script interface

`/sbin/nss_exec` is called with a command and optional arg:

```
/sbin/nss_exec getpwnam <username>
/sbin/nss_exec getpwuid <uid>
/sbin/nss_exec getgrnam <groupname>
/sbin/nss_exec getgrgid <gid>
/sbin/nss_exec getpwent <index>   # enumeration
```

Exit codes: 0=found, 1=not found, 2=try again, other=unavailable.

## Used by

`dirless-agent` — the agent's `nss_exec.cr` implements the `/sbin/nss_exec` script interface, reading from the local SQLite database.
