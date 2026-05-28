# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A macOS Seatbelt (sandbox-exec) profile that constrains Claude Code's filesystem access to a single working directory, blocking sensitive `HOME` paths like `~/.ssh`, `~/.aws`, `~/.gnupg`, and `~/.kube`.

Two files:
- `claude-code.sb` — Seatbelt profile in Scheme-like policy language; parameterized with `HOME` and `WORKING_DIR`
- `raincoat` — Bash wrapper that calls `sandbox-exec -f` with those parameters and forwards all args to `~/.local/bin/claude`

## Install

```bash
cp claude-code.sb ~/.local/share/claude/claude-code.sb
cp raincoat ~/.local/bin/raincoat
chmod +x ~/.local/bin/raincoat
```

## Testing sandbox changes

Run a command under the profile to verify a rule:

```bash
sandbox-exec -f claude-code.sb \
  -D HOME="$HOME" \
  -D WORKING_DIR="$PWD" \
  -D TMPDIR="$(cd "${TMPDIR:-/tmp}" && pwd -P)" \
  bash -c "<command to test>"
```

All three params (`HOME`, `WORKING_DIR`, `TMPDIR`) are required. If `TMPDIR` is omitted, `(param "TMPDIR")` evaluates to `""` and `(subpath "")` matches all paths — the file-write* and file-read-xattr rules become unbounded, making the test sandbox silently far more permissive than production.

Watch deny events in real time (separate terminal):

```bash
log stream --predicate 'eventMessage contains "deny"' --style compact
```

Note: some denies are silent (no log entry). Silent failures are common with atomic write patterns — if the lock or temp file write is denied, the operation hangs or returns EPERM without appearing in the deny log. Debug by binary-searching rules, not by reading the log.

## Profile architecture

The profile is allowlist-based (`deny default`). Rule order matters: a later `deny` overrides an earlier `allow` for the same path.

### Broad-allow + narrow-deny pattern

The write-hardening block exploits rule ordering to claw back specific paths inside `WORKING_DIR` that would otherwise be writable:

```scheme
;; Broad allow
(allow file-write* (subpath (param "WORKING_DIR")))

;; Then narrow denies on top
(deny file-write*
    (subpath (string-append (param "WORKING_DIR") "/.git/hooks"))
    (literal (string-append (param "WORKING_DIR") "/.git/config"))
    ...)
```

This pattern (credit: CJHwong/agent-seatbelt) prevents a sandboxed agent from installing git hooks that execute outside the sandbox.

### Credential-store lockdown

A `deny file-read* (with no-log)` block explicitly blocks `~/.ssh`, `~/.aws`, `~/.gnupg`, and `~/.kube`. Using `file-read*` (rather than `file-read-data`) means the deny overrides the unconditional `(allow file-read-metadata)` that appears earlier — so even `stat(2)` on those directories returns EPERM. This is intentional: subprocesses should not be able to probe whether credential dirs exist.

### Atomic write patterns

Several files use a lock + temp-then-rename pattern for safe writes. The profile must allow both the lock file and the temp file (via regex), not just the final target. This applies to:
- `~/.claude.json` — lock file + `.tmp.<pid>.<hash>` temps
- `~/Library/Keychains/login.keychain-db` — `.sb-*` atomic temps (securityd commits)

Silent EPERM (no deny log entry) is the failure mode when the temp/lock write is missing — see the Testing section below.

## Key constraints

- **Network**: Seatbelt `network-outbound` only accepts `*` or `localhost` as host values — arbitrary IPs and CIDRs fail at compile time. The profile uses `(remote tcp)` (unrestricted). Use `pf(4)` if per-host restriction is needed.
- **`sandbox-exec` is deprecated** by Apple but functional on macOS 14+. Tested on macOS 15 arm64.
- **TTY ioctls** are scoped to terminal nodes only (`/dev/tty`, `/dev/ptmx`, `/dev/pts`, `ttys[0-9]+`), not `(subpath "/dev")`, to avoid granting unnecessary disk/BPF ioctls.
- **Required params**: `HOME`, `WORKING_DIR`, and `TMPDIR` must all be supplied to `sandbox-exec -D`. Missing `TMPDIR` makes `(subpath "")` match all paths — the sandbox is silently unbounded. The `raincoat` wrapper resolves and passes all three correctly.
