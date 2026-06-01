# brood-terminal — roadmap

## Where we are now (v0 — userland, no runtime changes)

A native-window (`--features gui`) **eshell-style shell**: Brood is the language,
built-ins are Brood functions, and anything that isn't a built-in is delegated to
a real POSIX shell.

- **GUI by default** (`term/start` → `(gui-display)`) — no host terminal needed.
  `term/start-term` runs it inside an existing terminal instead. One pure
  `view`/`update` drives both frontends (ADR-046 display seam).
- **Dispatch** (`commands/run-command`):
  - `(brood form)` → evaluated as Brood, the value shown.
  - a built-in name with no shell metacharacters → the fast in-process built-in
    (`ls` with colour, `cd`, `cat`, `grep`, …).
  - anything else, or a `*cmd` prefix, or a line using `| > < ; & $ \` * ?` →
    handed to `/bin/sh -c` (run in the tab's cwd, output captured). **POSIX
    compatibility for free** — sh does the pipes/globs/redirection/quoting; we
    don't reimplement any of it.
- Tabs, scrollback, ↑/↓ history, Tab path-completion, mouse-wheel / PgUp-PgDn
  scroll.

**The v0 ceiling:** external commands are run with `run-process` (blocking,
output redirected to a temp file and slurped back **after** the command exits).
So there is **no live streaming** and **no interactive / full-screen program**
(a `bash` prompt, `vim`, `htop`, `less`, anything that reads keys or repaints).
That ceiling is a *runtime* capability gap — the host has no PTY — not something
userland Brood can lift. The two tiers below lift it.

---

## Tier 1 — real interactive processes, line-oriented (PTY in the runtime)

**Goal:** host a real `bash` (and any line-oriented program — a REPL, `python`,
`git log` paging off) in a tab, with live output as it arrives and keystrokes fed
to the child.

**Runtime work** (`crates/lisp`, behind a feature like the existing `gui`):

- Add `portable-pty` (the wezterm crate) — spawn a child on a PTY master, get an
  async reader + writer + a resize handle. ~30 lines.
- New builtins, mirroring how `gui.rs` already delivers window input to a process
  mailbox via `process::deliver(Message)` (ADR-058):
  - `(pty-spawn cmd args opts)` → a handle; a reader thread posts
    `[:pty-data handle bytes]` to the **calling** process's mailbox, and
    `[:pty-exit handle code]` on child exit.
  - `(pty-write handle string)` — feed stdin / keystrokes.
  - `(pty-resize handle cols rows)` — `TIOCSWINSZ` on SIGWINCH / window resize.
  - `(pty-kill handle)`.

**Brood work** (small — the seam already exists):

- `std/ui.blsp` already has `with-events`, which makes `ui-run`'s `:poll` drain
  mailbox messages and feed them to `update` as loop input. Wrap the display with
  the same trick so `[:pty-data …]` / `[:pty-exit …]` arrive as `update` inputs.
- Each tab gains a `:pty` handle. `update`:
  - a printable key / `:enter` / control key on a pty-backed tab → `pty-write`
    (instead of editing a local input string).
  - `[:pty-data h bytes]` → append to that tab's scrollback, handling `\r`, `\n`
    and a small set of SGR colour escapes (reuse the span-line model — colour is
    already a face).
  - `[:pty-exit h code]` → mark the tab done / show the exit.
- `cd` stays a shell built-in (a child can't change our cwd); a new tab spawns
  `bash` in the current cwd.

**Lands:** real `bash`, `git`, `make`, `python` etc. with live output.
**Still out:** full-screen apps that drive the cursor/alt-screen (vim, htop) —
those need a real cell grid, which is Tier 2.

---

## Tier 2 — full terminal emulator (VT grid)

**Goal:** xterm-class behaviour — cursor addressing, scroll regions, the
alternate screen, full SGR. `vim`, `htop`, `less`, `tmux` all work.

**Runtime work:**

- Parse the PTY byte stream with `vte` (the alacritty parser) or `termwiz`,
  maintaining a **cell grid** on the Rust side: each cell `{glyph fg bg attrs}`,
  plus cursor position, scroll region, and the alt-screen toggle.
- Expose the grid to Brood: `(pty-cells handle)` → rows of cells (or a diff since
  last read), `(pty-cursor handle)` → `[row col visible]`.

**Brood work (small, and it fits the display seam perfectly):**

- The display frame is *already* cell-based (`[:text row col s face]` +
  `[:cursor r c]`). So a terminal tab's `view` just mirrors the grid into those
  ops — colour/attrs map straight onto a face. No new rendering concepts.
- The emulator complexity lives in Rust (where `vte`/`termwiz` already exist);
  Brood stays a thin painter, which is the whole point of ADR-046.

**Lands:** a genuine terminal emulator written in Brood, running in a native
window, with the v0 eshell as a per-tab alternative shell.

---

## Why this ordering

Tier 1 reuses *everything* already built (tabs, scrollback span-lines, input,
the `with-events` seam) and the input-delivery pattern is a copy of `gui.rs`, so
it's the cheap, high-value step. Tier 2 is mostly Rust (the VT parser) because
the Brood side is already cell-shaped. The v0 userland shell is the fallback that
needs no runtime build and keeps working throughout.
