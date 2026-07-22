---
name: build
description: >
  Build tasks from the current spec with parallel agents, max 3 per run. Ship
  tasks get isolated git worktrees; scouts investigate read-only and report.
  Verified unescalated tasks auto-land when the project Mode allows; everything
  needing a human queues at /anchor:land. Decision forks become holds, never
  blockers.
disable-model-invocation: true
---

# Build

Orchestrate agents over the spec's open tasks. Autonomy by default, control by
policy: invariant escalations and clean evidence decide what lands without the
user — not per-task permission.

## Companion contracts — read the files, don't improvise

The check and land skills are hidden from model invocation; you cannot load
them as skills mid-run. Read the files instead:

- Before dispatching checkers: `${CLAUDE_PLUGIN_ROOT}/skills/check/SKILL.md`
- Before ANY merge (auto-land included): `${CLAUDE_PLUGIN_ROOT}/skills/land/SKILL.md`
  — its merge rules are canonical. Never merge from memory of them.

## Preconditions — halt if any fail

1. A spec file exists in `docs/specs/` with open tasks (ask which spec if several).
2. `CONTEXT.md` contains a `Test command:` line. No test command, no build —
   builders and checkers both need it. Point to /anchor:context.
3. The main checkout is on the main branch, with no uncommitted source
   changes. Anchor state files (`docs/specs/`, `docs/reports/`, `CONTEXT.md`)
   may be dirty — agents never commit on main; the user does.

## Mode

Read `Mode:` from `CONTEXT.md`: `auto-land` or `gate`. Absent = `auto-land`.

## Invariant escalations — regardless of mode

A task is `Escalation: required` if it touches ANY of: destructive or
irreversible operations, schema changes to a shared database, new dependencies,
security/auth surface, deploys. Escalated tasks build normally but NEVER
auto-land — they always wait for the user at /anchor:land. The mechanical
tripwire in land's merge rules backstops this tagging: an untagged hit still
queues for the human.

If a builder discovers mid-task it needs an escalation-category action not in
its criteria, it stops and reports — record that as a hold.

## Task selection

Take up to **3** tasks whose dependencies are landed or verified:

- `todo` tasks, plus
- **stale reclaims** — tasks stuck in `building` or `checking` with no live
  agent in this session (a crashed or compacted earlier run). Confirm with
  the user before reclaiming, then: ship + `building` → re-dispatch a builder
  into the existing worktree, briefed "worktree may contain partial work —
  assess and continue"; ship + `checking` → skip straight to checkers;
  scout + `building` → re-dispatch anchor-scout with the same report path.

More than 3 candidates: pick the 3 the user confirms.

## Dispatch

**Ship tasks** — one worktree each, engine-independent:

```
git worktree add ../<repo>-anchor-<spec-slug>-<task-id> -b anchor/<spec-slug>-<task-id> main
```

Branch from `main` explicitly — never from whatever happens to be checked out.
Reopened tasks: if the branch already exists, `git worktree add <path> <branch>`
(no `-b`); if the worktree path already exists, reuse it as-is. Never delete
existing work at dispatch.

Builders work ONLY in their assigned worktree, never the main checkout.
Brief: acceptance criteria verbatim, worktree path + branch, CONTEXT.md stack /
test command / decisions, PREFERENCES.md implementation preferences, and the
bootstrap contract:

- Install dependencies from the lockfile first (detect the package manager
  from the lockfile present). Installing from an existing lockfile is NOT the
  "new dependencies" escalation; adding packages is.
- Symlink the repo root's gitignored `.env*` files into the worktree (absolute
  paths) — worktrees don't carry ignored files.
- No long-running dev servers; if one is unavoidable, use a port unique to
  the task.

**Scout tasks** — no worktree, no branch. Spawn `anchor-scout` with the task's
questions; the report goes to `docs/reports/<spec-slug>-<task-id>.md` (left
uncommitted, like all anchor state on main).

Engine for parallel dispatch: agent teams when enabled this session, otherwise
background subagents. The pipeline must behave identically under either —
the engine is a detail, the worktree contract is not.

Update the task's `Status: building` and `Branch:` in the spec file on dispatch
— edit only, never `git commit` on the main branch.

## On each agent finishing

Ships → the /anchor:check contract (read the file): two zero-context checkers
on distinct lenses. Set `Status: checking` when checkers are dispatched.
Scouts → one zero-context `anchor-report-checker` verifying the report answers
every question with evidence.

- **Verified + auto-land + not escalated** → apply land's canonical merge
  rules (read the file): pre-flight, tripwire, freshness re-test, --no-ff
  merge, test main, rollback on red, cleanup only after green. `Status:
  landed`. Don't wait for sibling tasks. A tripwire hit or a dirty main
  checkout at merge time → don't merge; `Status: verified`, queues at
  /anchor:land.
- **Verified + (gate mode or escalated)** → `Status: verified`, queues for
  /anchor:land.
- **Checker failure** → if the task block has no `Repairs:` line, add
  `Repairs: 1`, dispatch one repair builder in the same worktree with the
  failure reasons, re-check. Failure with `Repairs: 1` already present →
  `Status: failed`. Never loop further — the counter lives in the spec file,
  so a crashed session cannot forget it.
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
