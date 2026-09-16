# Keep Claude Code out of the terminal's alternate screen

## Context

Scrolling up in a dev3 agent pane showed one screen and nothing more. Measured on the
live `dev3` tmux socket: 13 of 17 panes reported `alternate_on=1` and `history_size=0`
against a configured `history-limit` of 250 000. `capture-pane -S -` and `-S -200`
returned the same rows — the lines were never kept, so there was nothing deeper to
fetch.

## Investigation

The alternate screen was Claude Code's doing, not tmux's and not dev3's. Probed on a
private tmux socket (`tmux -L scrollprobe`, 120x30, same worktree, same input):

| Launch | `alternate_on` | `history_size` | `capture-pane -S -` |
|---|---|---|---|
| `claude` | 1 | 0 | 30 rows (one screen) |
| `CLAUDE_CODE_NO_FLICKER=1 claude` | 1 | 0 | 30 rows |
| `CLAUDE_CODE_DISABLE_ALTERNATE_SCREEN=1 claude` | 0 | 116 | 146 rows, back to the startup banner |

`CLAUDE_CODE_NO_FLICKER` is not the cause — the alternate screen is on with or without
it. The recovered history is clean: 121 distinct output lines, no duplicated repaint
frames.

Claude Code 2.1.273 gates the behaviour on
`CLAUDE_CODE_NO_FLICKER === false || CLAUDE_CODE_DISABLE_ALTERNATE_SCREEN`, and lists
the flag beside `CLAUDE_CODE_DISABLE_MOUSE` / `CLAUDE_CODE_ACCESSIBILITY`. Neither name
appears in Claude Code's public documentation — both were read out of the shipped
binary.

Codex, Gemini CLI and cursor-agent were probed the same way and already run in the
normal buffer with live history, so the problem and the fix are Claude-only.

## Decision

`CLAUDE_DEFAULT_ENV` in `src/bun/agents.ts` now injects
`CLAUDE_CODE_DISABLE_ALTERNATE_SCREEN: "1"` for every Claude-based preset. A user who
wants Claude's in-app scroll back overrides `envVars` on their agent configuration.

## Risks

The flag is undocumented and read out of the binary, so a future Claude Code release may
rename or drop it. The failure mode is benign — an unknown env var is ignored and the
pane goes back to holding no history, which is today's behaviour. Claude also stops
owning the screen on exit, so its transcript stays in the scrollback instead of being
wiped, and its own virtual scroll gives way to the terminal's.

Only panes created after the change are affected; the app must be restarted and existing
panes keep no history.

## Alternatives considered

`set -g alternate-screen off` in `src/bun/tmux/config.ts` — rejected: it forces the
behaviour on every program in every pane (`vim`, `less`, `htop` would all dump repaint
frames into the scrollback and never restore the screen), and it does nothing for the
native terminal backend, which has no tmux at all.

`CLAUDE_CODE_NO_FLICKER=0` reaches the same code path, but it overloads a flag that is
there for a different reason and reads as a rendering regression rather than an intent.
