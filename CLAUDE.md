# CLAUDE.md — PortScout

Handoff context for Claude Code. Read this first, then `SPEC.md` and
`IMPLEMENTATION_PLAN.md`. This file is the operating manual; the other two are the
what and the how.

---

## What this is

A local-first Linux desktop utility (Tauri 2.x) that lists active network ports with
their owning processes and lets the user kill them in one click. Ships as a `.deb`
for Ubuntu (x86_64) and Raspberry Pi (aarch64). The headline workflow: type a port
number, see what's holding it, free it.

**If you only remember one thing:** the privilege model is the crux. The app runs as
the invoking user, never as root. Own processes are killed directly; other-owned /
root processes route through `pkexec`. Do not shortcut this by suggesting the app run
elevated.

---

## Stack

- **Backend:** Rust. `tokio` (async commands), `nix` (signals + uid), `serde`.
  No `sysinfo` — enumeration is `ss` + `/proc`, which is lighter and more precise.
- **Frontend:** React + TypeScript + Vite + Tailwind.
- **Shell:** Tauri 2.x. Commands are the Rust↔JS boundary (see below).
- **Targets:** Ubuntu 22.04+/24.04+ x86_64, Raspberry Pi OS aarch64 (64-bit only).

---

## Architecture at a glance

```
ss -tulpnaH ──▶ src-tauri/src/ports.rs ──▶ list_ports() ──▶ React PortTable
                (parse + resolve uid)        (Tauri command)

Kill clicked ──▶ owned_by_me?
                 ├─ yes ──▶ kill.rs: SIGTERM (nix) ──▶ 2s grace ──▶ SIGKILL
                 └─ no  ──▶ kill.rs: pkexec kill (polkit prompt)
```

---

## Tauri command surface (the contract)

Keep this boundary narrow and typed. The frontend never passes a shell string — it
passes a PID and a signal enum. Rust validates.

```rust
list_ports()                      -> Vec<PortEntry>
resolve_port(port: u16)           -> Vec<PortEntry>
kill_process(pid: u32, sig: Sig)  -> KillResult   // own process, direct
kill_process_elevated(pid: u32)   -> KillResult   // via pkexec
```

`Sig` is a closed enum (`Term | Kill`) — never an arbitrary int/string from JS.
`KillResult { ok: bool, escalated: bool, message: String }`. All commands async;
enumeration runs on a blocking thread pool so the UI never stalls.

`PortEntry` fields: `protocol, local_addr, local_port, state, pid, process,
exe_path, user, owned_by_me`. See SPEC §3 for exact types.

---

## Build order

Follow `IMPLEMENTATION_PLAN.md` phases 0→6. Each phase ends runnable — don't jump
ahead. The critical path is: enumerate correctly (Phase 1) before any UI, and get
the owned-process kill solid (Phase 3) before touching pkexec (Phase 4).

Start each phase by confirming the previous phase's exit criterion still holds.

---

## Conventions

- **Aesthetic:** dark terminal / cyberpunk. Monospace throughout. Phosphor-green or
  teal accents on near-black. Clinical, disciplined spacing — no rounded-friendly
  fluff. This is a sysadmin tool; it should look like one.
- **Frontend structure:** typed `invoke()` wrappers live in `src/lib/api.ts` — no
  raw `invoke` calls scattered in components. Components stay presentational.
- **Errors are toasts, not crashes.** Permission-denied, process-already-gone,
  pkexec-cancelled, `ss` missing → non-blocking toast. The app never dies on a
  failed operation.
- **Rust:** parser lives in `ports.rs` and is unit-tested against captured `ss`
  fixtures in `tests/fixtures/ss/`. A line that won't parse is skipped with a logged
  warning — never a panic.

---

## Hard rules (do not violate)

1. **Never run the whole app as root.** Elevation is per-kill, via `pkexec`, only
   when `owned_by_me` is false.
2. **No shell interpolation, ever.** `pkexec` is invoked with fixed argv
   (`pkexec kill -TERM <pid>`), PID validated numeric first. Same for any command.
3. **Re-validate before killing.** Between enumerate and kill a PID can be recycled.
   Before signalling, confirm the process name still matches the row's `process`;
   abort if it changed. This prevents killing an innocent recycled PID.
4. **Confirm destructive kills.** Inline confirm showing name + pid + port. No
   one-click kill without a confirm step in the current design.
5. **Enumeration is read-only.** Listing ports never mutates system state.

---

## Gotchas / watch-items

- **`ss` format drift** across iproute2 versions is the main fragility. The `-p`
  field is `users:(("node",pid=4821,fd=23))`. Fixture-test the parser; fail soft.
- **pkexec absent** in a minimal env — detect at startup, disable elevated kills
  with a tooltip rather than failing at click time.
- **Hidden PIDs without root:** `ss -p` may show a socket but not its owning process
  for sockets you don't own. Those rows render with pid/process null. Rescan-with-
  elevation is v1.1, not v1 — don't build it into the happy path yet.
- **Pi build is slow.** Rust on aarch64 takes a while; consider `sccache`. Build
  natively on the Pi — don't attempt x86→aarch64 cross-compile for v1.
- **`.deb` Depends:** declare `iproute2` (ss) and `policykit-1` (pkexec) in the
  Tauri Debian config so a clean install resolves them.

---

## Out of scope for v1

Remote host scanning, historical/time-series tracking, firewall/iptables
management, packet inspection, kill-process-group, auto-refresh diffing. See
SPEC §1 non-goals and §10 open questions. Don't gold-plate — v1 is a tight,
correct, good-looking local port killer.

---

## Repo layout

```
portscout/
├── CLAUDE.md   ← you are here
├── SPEC.md
├── IMPLEMENTATION_PLAN.md
├── src/                       # React frontend
│   ├── components/PortTable.tsx
│   ├── components/ResolveBar.tsx
│   ├── lib/api.ts             # typed invoke() wrappers
│   └── theme.css
├── src-tauri/
│   ├── src/main.rs
│   ├── src/ports.rs           # enumeration + parser
│   ├── src/kill.rs            # signal + pkexec paths
│   └── tauri.conf.json        # allowlist, deb Depends, icons
└── tests/fixtures/ss/*.txt    # captured ss output for parser tests
```
