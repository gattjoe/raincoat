# Claude Code Seatbelt Sandbox

macOS Seatbelt profile for running Claude Code with minimal filesystem permissions.

---

```
  Trust Boundary Map

  ┌─────────────────────────────────────────────────────────────┐
  │  User shell (UNSANDBOXED)                                   │
  │                                                             │
  │   $ raincoat [args]                                         │
  │       │                                                     │
  │       ▼ captures $PWD as WORKING_DIR (frozen here)          │
  │       │                                                     │
  │       ▼ exec sandbox-exec ──────────────────────────────┐   │
  └────────────────────────────────────────────────────────┐│   │
                                                           ││   │
    ┌──────────────────────────────────────────────────────▼▼─┐ │
    │  SANDBOX BOUNDARY                                       │ │
    │                                                         │ │
    │  Claude Code binary (~/.local/bin/claude)               │ │
    │  └─ Node.js / V8                                        │ │
    │     └─ bash / git / npm subprocesses (inherit sandbox)  │ │
    │                                                         │ │
    │  CAN READ:  system paths, ~/.claude/, WORKING_DIR,      │ │
    │             ~/.nvm, shell configs, keychains (file)     │ │
    │                                                         │ │
    │  CAN WRITE: ~/.claude/, ~/.claude.json, WORKING_DIR     │ │
    │             (except .git/hooks, .git/config, .mcp.json) │ │
    │                                                         │ │
    │  CANNOT READ: ~/.ssh/, ~/.aws/, ~/.gnupg/, ~/Desktop/   │ │
    │                                                         │ │
    │  NETWORK: unrestricted outbound TCP                     │ │
    │  EXEC: unrestricted                                     │ │
    │  MACH: unrestricted (all XPC services reachable)        │ │
    └─────────────────────────────────────────────────────────┘ │
                                                                │
    Git hooks, IDE tasks execute HERE (outside sandbox) ◄───────┘
```

---

## Files

- `claude-code.sb` — Seatbelt profile (parameters: `HOME`, `WORKING_DIR`)
- `raincoat` — drop-in wrapper that invokes `sandbox-exec` with the correct parameters

## Install

```bash
cp claude-code.sb ~/.local/share/claude/claude-code.sb
cp raincoat ~/.local/bin/raincoat
chmod +x ~/.local/bin/raincoat
```

## Usage

```bash
# From any project directory, use instead of 'claude':
raincoat

# Pass any normal claude flags:
raincoat -p "run git log --oneline -3"
raincoat --continue
```

## What is blocked

| Path | Reason |
|------|--------|
| `~/.ssh/` | SSH private keys |
| `~/.gnupg/` | GPG keys |
| `~/.aws/` | AWS credentials |
| `~/Desktop/`, `~/Downloads/`, `~/Documents/` | Personal files |
| Any project outside `WORKING_DIR` | Lateral movement |

## What is allowed

| Path | Access |
|------|--------|
| `~/.claude/` | Read + write (config, history, memory, plugins) |
| `WORKING_DIR` (`$PWD`) | Read + write |
| `~/.local/share/claude/` | Read (Claude binary + versions) |
| `~/.nvm/` | Read (Node.js runtime for spawned tools) |
| `~/Library/Keychains/` | Read (OAuth session; securityd still enforces per-item ACLs) |
| System paths, Homebrew, Xcode | Read + exec |
| Outbound TCP | All (Anthropic API, GitHub, npm, OTLP telemetry) |

## Known limitations

- **Network**: outbound TCP is unrestricted. Tightening to specific hosts breaks `npm install`, `git fetch`, etc. Seatbelt network filters only accept `*` or `localhost` as the host value — arbitrary IPs and CIDR ranges are rejected at profile compile time, so LAN-to-specific-IP restriction is not possible here. Use `pf(4)` if LAN isolation is required.
- **`sandbox-exec` is deprecated** by Apple but remains functional on macOS 26+.
- Profile was tested on macOS 26 (arm64). Additional paths may be needed on other versions.

## Non-obvious requirements discovered during testing

| Path | Why needed |
|------|-----------|
| `/dev/ptmx` (write) | Node.js opens this to allocate a pty when spawning bash subprocesses; blocking it causes a silent hang in the event loop |
| `/dev/dtracehelper` (write) | Written by the security and Core Foundation frameworks at startup |
| `~/.CFUserTextEncoding` (read) | Read by Core Foundation during process initialization |
| `~/.claude.json` (read) | Legacy flat config path checked on startup for migration |
| `~/Library/Keychains/` (read) | OAuth session (`Claude Code-credentials` keychain item) |
| `/Applications/Ghostty.app/Contents/Resources/` (read) | Ghostty stores its terminfo here (not in `/usr/share/terminfo`); blocking it causes a blank-cursor hang when the TUI tries to look up `xterm-ghostty` |
| `file-ioctl` on `/dev/tty`, `/dev/ptmx`, `/dev/pts`, `/dev/ttys` | `tcsetattr()`/`tcgetattr()` and `TIOCGWINSZ` require ioctl access on the controlling tty. Without it, `TIOCGETD` returns `EPERM` (not captured by the sandbox deny log), the terminal stays in cooked mode, and Ink's terminal capability queries are echoed back as raw escape sequences. Scoped to terminal nodes only — `(subpath "/dev")` would also cover `/dev/disk*` and `/dev/bpf*`, enabling unnecessary disk probing ioctls. |
| `~/.claude.json` and `~/.claude.json.backup` (write) | Claude Code writes to the legacy flat config on startup during settings migration. Blocking these writes causes a silent hang before the TUI renders — no deny log entry appears because the operation fails in a code path that doesn't propagate the error. |
