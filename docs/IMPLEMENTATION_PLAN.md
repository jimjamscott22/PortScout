# IMPLEMENTATION_PLAN.md — PortScout

Phased build for the Tauri 2.x port scanner/killer. Each phase ends in something
runnable. Assumes: Rust stable, Node 18+, Tauri CLI 2.x, `ss`/`pkexec` present on
target (both standard on Ubuntu + Pi OS).

---

## Phase 0 — Scaffold & toolchain (½ day)

- `npm create tauri-app@latest portscout` → React + TypeScript + Vite template.
- Add Tailwind; drop in the dark/monospace base theme (phosphor-green/teal tokens)
  so the aesthetic is settled before any real UI.
- Confirm `tauri dev` opens a window on the dev box.
- Rust deps: `nix` (signals/uid), `serde`/`serde_json` (already via Tauri),
  `tokio` (async commands). No `sysinfo` needed — `ss` + `/proc` is lighter and
  more precise for this.

**Exit:** blank themed window launches via `tauri dev`.

---

## Phase 1 — Enumeration backend (1 day)

- Implement `list_ports()` as a Tauri command.
  - Spawn `ss -tulpnaH` via `tokio::process::Command`.
  - Parse each line into `PortEntry`. The `-p` field format is
    `users:(("node",pid=4821,fd=23))` — parse process name + pid with a small
    hand-rolled parser (regex optional, but the shape is fixed enough to split).
  - Resolve `user` and `owned_by_me` from `/proc/<pid>/status` (`Uid:` line) vs
    the app's own uid (`nix::unistd::getuid`).
  - Resolve `exe_path` from `/proc/<pid>/exe` (readlink; may EACCES — tolerate).
- Unit-test the parser against captured `ss` fixture lines (IPv4, IPv6, udp,
  no-pid kernel socket, multi-process socket).

**Exit:** `list_ports()` returns correct structured data, verified against
`ss -tulpn` by eye on all three machines.

---

## Phase 2 — Read-only UI (1 day)

- Port table component: virtualized if the row count warrants (usually < 100, so
  plain render is fine for v1).
- Wire **Refresh** → `list_ports()`; render loading + error states.
- Status bar (counts + last-scan timestamp).
- Rows without a PID render muted; rows not `owned_by_me` show the lock affordance.
- Filter box (client-side, matches port/process/pid/addr) + "listening only" toggle.

**Exit:** live, filterable, refreshable read-only view on real data.

---

## Phase 3 — Kill path, owned processes (1 day)

- `kill_process(pid, signal)` command using `nix::sys::signal::kill`.
  - Validate PID still exists before signalling.
  - SIGTERM first; poll `/proc/<pid>` for up to ~2s; if alive, surface the
    "still running — force kill?" affordance → SIGKILL.
- Inline per-row confirmation UI (name + pid + port) before firing.
- Toast feedback: killed / already gone / failed.

**Exit:** can free a port owned by the current user, end to end, with confirm +
graceful-then-force escalation.

---

## Phase 4 — Privilege escalation (1 day)

- `kill_process_elevated(pid)` → fixed argv `pkexec kill -TERM <pid>` (validated
  numeric PID, no shell).
- Rows not `owned_by_me` route their Kill action here; polkit dialog handles auth.
- Handle pkexec exit codes: 126 (dismissed), 127 (auth failed) → friendly toasts,
  not scary errors.

**Exit:** killing a root-owned process (e.g. a stray `postgres`/`pihole-FTL` in a
test scenario) works via a polkit prompt, app never runs as root.

---

## Phase 5 — Resolve-port workflow + polish (1 day)

- `resolve_port(port)` command + the "resolve port" box as the headline flow:
  type `3000`, see the offender, one-click kill.
- Auto-refresh toggle (2/5/10s/off), off by default.
- Keyboard-first niceties (your usual): `/` focus filter, `r` refresh, `Esc`
  cancel a pending confirm.
- Empty states, error toasts, final aesthetic pass.

**Exit:** headline conflict-resolution flow feels instant; app is pleasant.

---

## Phase 6 — Packaging & multi-arch (1 day)

- `tauri build` on Ubuntu x86_64 → `.deb`. Verify install/uninstall + launcher
  entry + icon.
- Declare runtime deps in the Tauri Debian config: `pkexec` lives in
  `policykit-1` (Ubuntu) / present on Pi OS; `ss` in `iproute2`. Add both to the
  `.deb` `Depends` so a clean install doesn't miss them.
- Pi (aarch64) build: simplest path is building **natively on the Pi** (clone,
  `tauri build`). Cross-compilation from x86_64 is possible but the webkit2gtk /
  sysroot setup is fiddly — native-on-Pi is the pragmatic v1 route given you have
  the hardware.
- Smoke-test the `.deb` on both Ubuntu boxes and the Pi.

**Exit:** installable `.deb` on all three machines, deps resolved automatically.

---

## Phase 7 — (v1.1 backlog, not blocking)

- Rescan-with-elevation to fill hidden PIDs (SPEC §6).
- Kill process group for dev servers that fork workers (SPEC §10.2).
- Auto-refresh diffing with new/departed highlight (SPEC §10.1).
- Remember window size/position, last filter state.

---

## Risk / watch-items

- **`ss` output format drift** across iproute2 versions — the parser is the
  fragile part. Fixture-test it and fail soft (a row that won't parse is skipped
  with a logged warning, not a crash).
- **pkexec absent** in a minimal environment — detect at startup, disable elevated
  kills with an explanatory tooltip rather than failing at click time.
- **PID reuse race** — between enumerate and kill, a PID could be recycled. Mitigate
  by re-validating process name matches the row's `process` immediately before
  signalling; abort if it changed.
- **Pi build time** — Rust compiles are slow on a Pi. Budget the first build
  generously or use `sccache`.

---

## Suggested repo layout

```
portscout/
├── CLAUDE.md                  # handoff context for Claude Code
├── SPEC.md
├── IMPLEMENTATION_PLAN.md
├── src/                       # React frontend
│   ├── components/PortTable.tsx
│   ├── components/ResolveBar.tsx
│   ├── lib/api.ts             # typed wrappers over invoke()
│   └── theme.css
├── src-tauri/
│   ├── src/main.rs
│   ├── src/ports.rs           # enumeration + parser
│   ├── src/kill.rs            # signal + pkexec paths
│   └── tauri.conf.json        # allowlist, deb Depends, icons
└── tests/fixtures/ss/*.txt    # captured ss output for parser tests
```
