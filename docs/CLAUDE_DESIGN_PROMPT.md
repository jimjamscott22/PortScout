# Claude Design prompt — PortScout UI exploration

Paste the block below into Claude Design. It's self-contained. Everything after the
`---` is the prompt; the notes above it are just for you.

**How to use:** feed it as-is for a first pass. Then iterate with follow-ups like
"tighten the density," "show the pkexec confirm state," "try a variant where the
resolve-port box is the primary hero element." The prompt asks for a few states up
front so you have something to react to.

---

Design the UI for **PortScout**, a local-first Linux desktop utility that lists
active network ports and lets the user kill the process bound to each one. It's a
single-window desktop app (Tauri) aimed at developers resolving port conflicts —
the "port 3000 is already in use" problem.

**Aesthetic direction:** dark terminal / cyberpunk. Near-black background,
monospace typography throughout, phosphor-green or teal accents used sparingly for
emphasis and interactive elements. Clinical, disciplined spacing — this is a
sysadmin instrument, not a consumer app. Think a well-made TUI reimagined as a
crisp GUI: dense but legible, every pixel earning its place. No gradients, no
glow-for-glow's-sake, no rounded playful softness. Restraint is the whole point.

**The one window contains:**

1. **Top bar** — app name, a Refresh button, and an auto-refresh interval control
   (off / 2s / 5s / 10s), off by default.

2. **A "resolve port" input** — the headline feature. The user types a port number
   (e.g. `3000`); the app surfaces the process holding it with a one-click kill.
   This is the primary workflow — give it visual priority.

3. **A filter/search input** and a "listening only" toggle.

4. **The port table** — the core surface. Columns: protocol (tcp/tcp6/udp/udp6),
   local address:port, state (LISTEN/ESTAB/etc.), PID, process name, user, and a
   per-row kill action. Design for ~10–40 rows typically visible.
   - Rows the current user owns get a direct kill affordance.
   - Rows owned by root or another user show a subtle lock cue — killing them
     triggers a system auth (polkit) prompt, so hint at that coming.
   - Rows with no resolvable process (kernel-owned) render muted / de-emphasized.

5. **A status bar** — total port count, listening count, last-scan timestamp.

**Key interaction states to show (at least a few of these):**

- The default populated table.
- The inline kill confirmation: when a kill is triggered, show a compact confirm
  affordance displaying process name + PID + port so the user can't fat-finger the
  wrong row.
- The "resolve port" flow mid-use — a port typed in, the matching process surfaced.
- The distinction between an owned-process row and a root-owned (lock) row.

**Constraints & feel:**

- Everything monospace. Numbers (ports, PIDs) should align cleanly in columns —
  tabular figures.
- Dark theme only — this app has no light mode.
- Density matters: a developer scanning for one stuck port shouldn't have to
  scroll past airy padding. Compact rows, clear column rhythm.
- Accent color (phosphor-green or teal) is for interactive/emphasis only — kill
  actions, the active resolve match, focused inputs. Don't paint the whole UI in it.
- The kill action is destructive; it should read as deliberate, not a casual tap.

Give me a couple of directions to react to — e.g. one that leans into a classic
terminal-emulator look (scanline-adjacent, high-contrast green-on-black) and one
that's more of a modern clinical dark dashboard (teal accents, softer neutrals,
more whitespace discipline than glow). I'll pick a direction and we'll refine.
