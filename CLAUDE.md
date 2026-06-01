# brood-terminal — guidance for Claude

A tabbed, full-screen terminal-style app — a **Brood mini-shell**, not a bash
emulator. Built on the `ui-run` display seam (`docs/brood-for-claude.md` →
*Interactive apps*).

## Shell model (eshell-style) & why it's not real bash

Brood is the shell language. `commands/run-command` dispatches:
- `(brood form)` → evaluated as Brood, the value shown;
- a built-in name with no shell metacharacters → the fast in-process built-in;
- everything else (or a `*cmd` prefix, or a line with `| > < ; & $ \` * ?`) →
  handed to `/bin/sh -c` in the tab's cwd, output captured. POSIX
  compatibility comes from delegating to `sh`, not reimplementing it.

A *faithful* emulator (a live `bash` prompt, `vim`, `htop`) needs a **PTY**, which
the runtime lacks: `run-process` inherits stdio and blocks, so external commands
are captured only **after** they exit — no streaming, no interactive/full-screen
programs. Lifting that is **runtime** work, planned in `ROADMAP.md` (Tier 1: PTY
builtins + line output; Tier 2: a `vte` cell grid for full-screen apps). The
input-delivery seam already exists — `gui.rs` posts to a process mailbox and
`std/ui.blsp`'s `with-events` folds mailbox messages into `ui-run` as input.

## Layout

- `src/commands.blsp` — the command interpreter. `(run-command input ctx)` →
  a result map `{:lines :cwd? :clear? :exit?}`. Pure over `ctx`
  (`{:cwd :cols :history}`); the filesystem builtins are its only side channel.
  Output is **span-lines**: a list of `[string face]` spans (`face` a `display`
  style map or nil). Commands: ls/ll, cd, pwd, cat, head, tail, wc, grep, echo,
  mkdir, touch, rm, history, clear, whoami, date, help, exit.
- `src/term.blsp` — the `ui-run` app: model/`view`/`update`, tabs, line editing,
  ↑/↓ history, Tab path-completion, scrollback, mouse-wheel + PgUp/PgDn scroll.
  `(start)` launches it on `*term-display*`.
- `src/main.blsp` — entry point; `(main)` calls `(term/start)`.

`view`/`update` are pure (model + size → frame / model + input → model), so the
whole UI is testable without a terminal — see `tests/term_test.blsp`, which
drives `update` and asserts on the model + that `view` returns a frame.

## Running & testing

- `nest test` — the suite (commands + UI reducer), no TTY needed.
- `nest run` — launch the terminal (needs a real TTY).
- `nest run --for 1s` — bounded run for CI/verification. The harness shell has
  no TTY, so wrap it in a PTY: `script -qec "nest run --for 1s" /dev/null`.

## Original scaffold notes

## Running

- `nest test`   — run the test suite (each test runs in its own green process).
- `nest run`    — invoke the entry point. Defaults to the `main` function in
  the `main` module; override in `project.blsp` with `:main` (bare symbols,
  the manifest is data — `:main app` runs `app/main`; `:main (app start)`
  runs `app/start`; never quote them).
  Each module is a namespace (ADR-065): a file's `defmodule name` makes its
  `def`/`defn` define `name/foo`; a bare reference resolves in the current
  namespace, then through `(:use …)` imports, then root/prelude. So the same
  short name in two modules is fine (`a/parse` vs `b/parse`) — only earmuffed
  `*foo*` vars are ambient/bare and must stay unique.
- `nest run --for 2s` — run a loop / full-screen TUI for a bounded time, then
  exit cleanly (`2s`, `500ms`, or a bare integer of ms). The way to exercise
  a never-returning program end-to-end or in CI.
- `nest format` — format Brood source.

## Writing Brood

`docs/brood-for-claude.md` is the language reference geared for AI assistants
— syntax, idioms, and the patterns that aren't shared with other Lisps. Read
it before generating Brood code. The `.claude/skills/writing-brood` skill
carries the short version and auto-loads when Claude Code edits `.blsp` files.

Brood ships randomness (`rand-int`/`rand-float`/`shuffle`/`sample` — pure and
seedable, thread the seed), bitwise ops (`bit-and`/`bit-or`/`bit-xor`/...),
and discovery (`apropos`, `all-globals`, `doc-search`) — use the last three to
find what exists instead of guessing names.

## MCP integration

`.mcp.json` points Claude Code at this project's `nest mcp` server, so `cd brood-terminal && claude`
auto-attaches an agent that can `eval`, `load`, `lookup`, `macroexpand`, `format`,
and discover the image with `apropos` / `all-globals` / `doc-search`, against the
live image (ADR-036, `docs/mcp.md` upstream).
