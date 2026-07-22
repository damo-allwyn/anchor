---
name: build
description: >
  (anchor) Build tasks from the current spec with parallel agents, max 3 per
  run. Ship tasks get isolated git worktrees; scouts investigate read-only and
  report. Verified unescalated tasks auto-land when the project Mode allows;
  everything needing a human queues at /anchor:land. Decision forks become
  holds, never blockers.
disable-model-invocation: true
---

# Build

Orchestrate agents over the spec's open tasks. Autonomy by default, control by
policy: invariant escalations and clean evidence decide what lands without the
user — not per-task permission.

## Preconditions — halt if any fail

1. A spec file exists in `docs/specs/` with tasks in `todo` (ask which spec if several).
2. `CONTEXT.md` contains a `Test command:` line. No test command, no build —
   builders and checkers both need it. Point to /anchor:context.
3. No uncommitted source changes on the main branch. Anchor state files
   (`docs/specs/`, `docs/reports/`, `CONTEXT.md`) may be dirty — agents never
   commit on main; the user does.

## Mode

Read `Mode:` from `CONTEXT.md`: `auto-land` or `gate`. Absent = `auto-land`.

## Invariant escalations — regardless of mode

A task is `Escalation: required` if it touches ANY of: destructive or
irreversible operations, schema changes to a shared database, new dependencies,
security/auth surface, deploys. Escalated tasks build normally but NEVER
auto-land — they always wait for the user at /anchor:land.

If a builder discovers mid-task it needs an escalation-category action not in
its criteria, it stops and reports — record that as a hold.

## Task selection

Take up to **3** `todo` tasks whose dependencies are landed or verified. More
than 3 open: pick the 3 the user confirms.

## Dispatch

**Ship tasks** — one worktree each, engine-independent:

```
git worktree add ../<repo>-anchor-<task-id> -b anchor/<spec-slug>-<task-id>
```

Builders work ONLY in their assigned worktree, never the main checkout.
Brief: acceptance criteria verbatim, worktree path + branch, CONTEXT.md stack /
test command / decisions, PREFERENCES.md implementation preferences.

**Scout tasks** — no worktree, no branch. Spawn `anchor-scout` with the task's
questions; the report goes to `docs/reports/<spec-slug>-<task-id>.md` (left
uncommitted, like all anchor state on main).

Engine for parallel dispatch: agent teams when enabled this session, otherwise
background subagents. The pipeline must behave identically under either —
the engine is a detail, the worktree contract is not.

Update the task's `Status: building` and `Branch:` in the spec file on dispatch
— edit only, never `git commit` on the main branch.

## On each agent finishing

Ships → /anchor:check (two zero-context checkers). Scouts → one zero-context
checker verifying the report answers every question with evidence.

- **Verified + auto-land + not escalated** → merge immediately under
  /anchor:land's merge rules (--no-ff, test main after, red main halts all
  further merges, merge commits only). `Status: landed`. Don't wait for
  sibling tasks.
- **Verified + (gate mode or escalated)** → `Status: verified`, queues for
  /anchor:land.
- **Checker failure** → one repair round with the failure reasons, re-check.
  Second failure → `Status: failed`. Never loop further.
- **Genuine decision fork** (builder blocked on a choice only the user can
  make) → record it under the spec's `## Holds` with a recommendation,
  `Status: held`, and move on. Holds are an async queue for the user, not a
  reason to stop the run.

## End state

Report per task: status, branch, checker verdicts, test output tail, holds
recorded. Then:

```
<a> landed, <b> awaiting you at /anchor:land, <c> held, <d> failed.
```
