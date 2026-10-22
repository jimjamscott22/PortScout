# SPEC.md — PortScout

A local-first Linux desktop utility for inspecting active network ports and killing
the processes bound to them. Tauri 2.x, Rust backend, React/Vite/Tailwind frontend.
Packaged as a `.deb` for Ubuntu (x86_64) and Raspberry Pi (aarch64).

---

## 1. Problem & Goals

**Problem.** Port conflicts (`EADDRINUSE`, "address already in use") are a routine
dev annoyance. Resolving them means remembering `ss`/`lsof` incantations, grepping
for the PID, then `kill`-ing it — often with a `sudo` round-trip when the offender
is root-owned.

**Goal.** A single-window app that shows every listening/bound port with its owning
process, lets you filter to a port or PID, and kills the process in one click —
handling privilege escalation cleanly when required.

**Non-goals (v1).**
- No remote host scanning. Local machine only.
- No historical tracking / time-series (that's a PySizer-shaped problem, not this one).
- No firewall/iptables management.
- No packet inspection.

---

## 2. Target Platforms

| Platform            | Arch     | Notes                                    |
|---------------------|----------|------------------------------------------|
| Ubuntu 22.04+       | x86_64   | Primary dev target                       |
| Ubuntu 24.04+       | x86_64   | Secondary                                |
| Raspberry Pi OS     | aarch64  | 64-bit only; armhf out of scope for v1   |

Distribution: `.deb` per arch. Built via `tauri build`; Pi build produced either
natively on the Pi or via cross-compilation (see Implementation Plan §6).

---

## 3. Core Data Model

The single domain object is a **PortEntry**:

```
PortEntry {
  protocol:    "tcp" | "tcp6" | "udp" | "udp6"
  local_addr:  string   // e.g. "0.0.0.0", "127.0.0.1", "[::]"
  local_port:  u16
  state:       string    // "LISTEN", "ESTAB", etc. (tcp only; udp is "")
  pid:         u32 | null
  process:     string | null   // comm name, e.g. "node", "python3"
  exe_path:    string | null   // full path when resolvable
  user:        string | null   // owning username
  owned_by_me: bool            // current uid == process uid
}
```

Source of truth: `ss -tulpnaH` parsed by the Rust backend. `-H` suppresses the
header (stable parsing); `-p` gives `users:(("proc",pid=NNN,fd=N))`; `-a` includes
non-listening; `-n` keeps ports numeric. The `owned_by_me` flag is computed by
comparing the process's uid (from `/proc/<pid>/status`) against the app's own uid.

---

## 4. Functional Requirements

### FR-1 — Enumerate ports
On launch and on manual refresh, list all TCP/UDP ports with the fields above.
Ports with no resolvable PID (kernel-owned, or permission-hidden) still appear,
with `process`/`pid` null and a muted visual treatment.

### FR-2 — Live-ish refresh
A **Refresh** button re-runs enumeration. Optional auto-refresh toggle with a
selectable interval (2s / 5s / 10s / off). Off by default — this is a utility, not
a monitor, and polling `ss` every 2s wastes cycles.

### FR-3 — Filter & search
Free-text filter matching against port, process name, PID, or address. A quick
"listening only" toggle (hides ESTAB/TIME_WAIT churn — the common case for
resolving a conflict).

### FR-4 — Kill a process
Each row with a PID has a **Kill** action.
- If `owned_by_me`: send `SIGTERM` directly via the Rust `nix`/`libc` kill, then
  offer `SIGKILL` escalation if the process survives a short grace period (~2s).
- If **not** owned by me (root or another user): route through `pkexec` so the user
  gets a polkit auth dialog rather than the app demanding blanket root. Never run
  the whole app as root.
- Always confirm destructive kills behind a small inline confirmation, showing
  process name + PID + port so the user can't fat-finger the wrong row.

### FR-5 — Bulk / conflict resolution helper
Given a port number (typed into a "resolve port" box), find the process holding it
and surface a one-click kill. This is the headline workflow: "port 3000 is stuck,
free it."

### FR-6 — Feedback & errors
Surface: permission-denied on kill, process-already-gone, pkexec-cancelled, and
enumeration failures (e.g. `ss` missing) as non-blocking toasts.

---

## 5. Non-Functional Requirements

- **Local-first & offline.** Zero network calls. No telemetry.
- **Fast cold start.** Window interactive < 500ms; first port list < 300ms after.
- **Small footprint.** Target binary + webview assets under ~12MB installed.
- **Least privilege.** App runs as the invoking user. Elevation only per-kill,
  only when required, via polkit.
- **Read-only until acted on.** Enumeration never mutates system state.

---

## 6. Privilege Model (the crux)

| Action                          | Privilege needed        | Mechanism            |
|---------------------------------|-------------------------|----------------------|
| List own ports/PIDs             | none                    | `ss`, `/proc`        |
| List *others'* PID for a port   | none (names may hide*)  | `ss -p`              |
| Kill own process                | none                    | `kill(2)` via `nix`  |
| Kill other user's / root process| root                    | `pkexec kill`        |

*Without root, `ss -p` may show the socket but not the owning process for sockets
you don't own. Those rows render with PID/process null and a "details need
elevation" hint; an optional **"Rescan with elevation"** button re-runs `ss` under
`pkexec` to fill them in. Deferred to v1.1 if it complicates the happy path.

---

## 7. UI Surface (one window)

```
┌─ PortScout ──────────────────────────────── [⟳ Refresh] [auto ▾] ─┐
│ [ resolve port: ____ ]  [ filter: __________ ]  ☑ listening only  │
├──────────────────────────────────────────────────────────────────┤
│ PROTO  LOCAL ADDRESS:PORT   STATE    PID    PROCESS     USER   ⚡  │
│ tcp    0.0.0.0:3000         LISTEN   4821   node        jamie  ⚡  │
│ tcp6   [::]:8000            LISTEN   5190   python3     jamie  ⚡  │
│ tcp    127.0.0.1:5432       LISTEN   988    postgres    root   ⚡  │
│ udp    0.0.0.0:53           —        712    pihole-FTL  root   ⚡  │
├──────────────────────────────────────────────────────────────────┤
│ 24 ports · 4 listening · scanned 12:04:31                         │
└──────────────────────────────────────────────────────────────────┘
```

Aesthetic: dark terminal, monospace, phosphor-green/teal accents, clinical spacing.
Kill glyph (⚡) is the per-row action; rows not owned by the user get a subtle
lock affordance to signal the pkexec prompt that's coming.

---

## 8. Tauri Command Surface (Rust ↔ JS boundary)

```
list_ports()                 -> Vec<PortEntry>
resolve_port(port: u16)      -> Vec<PortEntry>   // entries on that port
kill_process(pid, signal)    -> KillResult       // own process, direct
kill_process_elevated(pid)   -> KillResult       // via pkexec
```

`KillResult { ok: bool, escalated: bool, message: string }`. All commands are
async; enumeration runs on a blocking thread pool so the UI never stalls.

---

## 9. Security Notes

- No arbitrary command execution from the frontend. The frontend passes a **PID and
  a signal enum** — never a shell string. Rust validates the PID exists and maps the
  signal from a closed set.
- `pkexec` is invoked with a fixed argv (`pkexec kill -TERM <pid>`), no shell
  interpolation, PID validated as numeric before the call.
- Tauri allowlist scoped to only the four commands above; no fs/shell plugins
  exposed to the webview beyond what's needed.

---

## 10. Open Questions

1. Should auto-refresh diff the list (highlight new/departed ports) or just replace?
   — Lean: diff, subtle flash on change. Deferred to v1.1.
2. Kill by exe path vs PID when a process has forked children — do we offer
   "kill process group"? Real for `node`/dev servers that spawn workers. Flag for v1.1.
3. Rescan-with-elevation (§6) in v1 or v1.1? — Lean v1.1 to keep the happy path clean.
