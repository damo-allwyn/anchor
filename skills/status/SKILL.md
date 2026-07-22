---
name: status
description: >
  Read-only pipeline snapshot: specs and their task states, live anchor
  branches and worktrees, running agents, and what to do next. Writes nothing.
  Safe to run anytime.
disable-model-invocation: true
---

# Status

Full snapshot, never a delta. Writes nothing.

Gather:

1. Specs in `docs/specs/` — per spec: header status, task counts by state
   (todo / building / checking / verified / failed / held / landed /
   abandoned).
2. `git worktree list` and `git branch --list 'anchor/*'` — live anchor
   worktrees (`../<repo>-anchor-<spec-slug>-<task-id>`) and branches,
   including orphans (branch with no matching task in any spec, or task
   marked landed with branch still alive).
3. Any builder or checker agents still running this session.
4. **Stale tasks** — `building` or `checking` with no live agent in this
   session: a crashed or compacted earlier run left them behind.

## Render exactly four sections, each present even when empty

- **Needs you** — the drain queue: unresolved holds, ALL verified tasks
  awaiting merge at /anchor:land (gate, escalated, tripwired, flagged, or
  stranded auto-land leftovers), unread scout reports, failed tasks, split
  verdicts and flags (read from each task's `- Checkers:` line and
  `docs/reports/*-check.md`), orphaned branches, and stale tasks (recommend
  reclaiming via /anchor:build).
- **In flight** — tasks building or checking right now.
- **Recently landed** — landed tasks from the most recent spec.
- **Up next** — todo tasks whose dependencies are met, and the single
  recommended next command.
