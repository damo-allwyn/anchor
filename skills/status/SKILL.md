---
name: status
description: >
  (anchor) Read-only pipeline snapshot: specs and their task states, live
  anchor branches and worktrees, running agents, and what to do next. Writes
  nothing. Safe to run anytime.
disable-model-invocation: true
---

# Status

Full snapshot, never a delta. Writes nothing.

Gather:

1. Specs in `docs/specs/` — per spec: header status, task counts by state
   (todo / building / verified / failed / landed).
2. `git worktree list` and `git branch --list 'anchor/*'` — live anchor
   worktrees and branches, including orphans (branch with no matching task in
   any spec, or task marked landed with branch still alive).
3. Any builder or checker agents still running this session.

## Render exactly four sections, each present even when empty

- **Needs you** — the drain queue: unresolved holds, escalated or gated tasks
  awaiting approval at /anchor:land, unread scout reports, failed tasks, split
  verdicts, orphaned branches.
- **In flight** — tasks building or checking right now.
- **Recently landed** — landed tasks from the most recent spec.
- **Up next** — todo tasks whose dependencies are met, and the single
  recommended next command.
