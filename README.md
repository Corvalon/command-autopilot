# /autopilot

A [Claude Code](https://claude.com/claude-code) slash command that works a backlog autonomously: it plans before it codes, writes tests first (TDD), skips gated items instead of stopping, and checkpoints after every deliverable. It runs item after item until the backlog is clear or you stop it.

**Requires the [`/save`](https://github.com/Corvalon/command-save) command.** Autopilot delegates all checkpointing and committing to `/save` — it is the single commit owner. Install `/save` first.

## Behavior highlights

- **Plan-first** — every item gets a plan with explicit tests and runnable audit checkpoints before code is written.
- **Skip, don't stop** — an item blocked on an operator decision, missing infra, or design ambiguity is skipped (logged), and autopilot moves to the next priority. The loop only stops when the backlog is clear or you intervene.
- **Local boundary only** — autopilot ships to the local-commit boundary; promotion to deployed/verified is the operator's call.
- **Toast on end or blocked** — you get a notification when the run ends or needs you, not per item.

## Install

```sh
cp autopilot.md ~/.claude/commands/autopilot.md
# install the /save dependency too:
#   https://github.com/Corvalon/command-save
```

Then run `/autopilot <plan-or-task>` in Claude Code.
