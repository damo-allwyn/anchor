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
auto-land — they always wait for the user at /anchor:land. Two mechanical
backstops catch untagged escalations: the path tripwire in land's merge rules,
and the audit checker's flags — either one forces the queue.

If a builder discovers mid-task it needs an escalation-category action not in
its criteria, it stops and reports — record that as a hold.

## Task selection

Take up to **3** tasks whose dependencies are landed or verified:

- `todo` tasks, plus
- **stale reclaims** — tasks stuck in `building` or `checking` with no live
  agent in this session (a crashed or compacted earlier run). Confirm with
  the user before reclaiming — and don't reclaim if another session might be
  live. Then: ship + `building` → re-dispatch a builder into the existing
  worktree, briefed "worktree may contain partial work — assess and
  continue" plus the check report file's failure reasons when one exists
  (a mid-repair crash); ship + `checking` → skip straight to checkers; scout +
  `building` → re-dispatch anchor-scout with the same report path; scout +
  `checking` → re-dispatch the report checker on the existing report.

More than 3 candidates: pick the 3 the user confirms. Stranded `verified`
tasks are not build's problem — they queue at /anchor:land.

## Dispatch

**Ship tasks** — one worktree each, engine-independent. Run
`git worktree prune` first (clears stale registrations), then:

- Worktree path exists → reuse it as-is; never delete existing work.
- Else, branch exists (reopened task) → `git worktree add <path> <branch>`.
- Else → fresh:

```
git worktree add ../<repo>-anchor-<spec-slug>-<task-id> -b anchor/<spec-slug>-<task-id> main
```

Branch from `main` explicitly — never from whatever happens to be checked out.
If the spec names a branch that no longer exists, stop and surface the
inconsistency — don't silently recreate it.

Builders work ONLY in their assigned worktree, never the main checkout.
Brief: acceptance criteria verbatim, worktree path + branch, CONTEXT.md stack /
test command / decisions, PREFERENCES.md implementation preferences, and the
bootstrap contract below. For `Risk: low` tasks, brief the `Fast test command:`
from CONTEXT.md (when defined) as the test command — the check contract
mechanically verifies the tier claim against the diff and re-runs the full
command if the claim doesn't hold.

- Install dependencies from the lockfile first (detect the package manager
  from the lockfile present) — but clone before installing: if the repo root
  has an installed dependency directory (`node_modules`, `.venv`) and its
  lockfile is byte-identical to the worktree's, `cp -Rc` it into the worktree
  (APFS copy-on-write, near-instant; flag unsupported → fall back to a normal
  install; worktree already has the directory — a reused worktree — skip
  both, cloning onto it would nest it). Installing from an existing lockfile
  is NOT the "new dependencies" escalation; adding packages is.
- Symlink the repo root's gitignored `.env*` files into the worktree (absolute
  paths) — worktrees don't carry ignored files.
- No long-running dev servers; if one is unavoidable, use the port assigned
  in the brief (assign each dispatched task a unique one).
- Commit everything meant to land; hand off a clean worktree (uncommitted
  tracked changes fail the check). Scratch goes to gitignored paths.
- Never touch anchor state files (`docs/specs/`, `docs/reports/`,
  `CONTEXT.md`) in any checkout — a needed CONTEXT.md change is a hold, not
  an edit.

**Scout tasks** — no worktree, no branch. Spawn `anchor-scout` with the task's
questions; the report goes to `docs/reports/<spec-slug>-<task-id>.md` (left
uncommitted, like all anchor state on main).

Engine for parallel dispatch: agent teams when enabled this session, otherwise
background subagents. The pipeline must behave identically under either —
the engine is a detail, the worktree contract is not.

Update the task's `Status: building` and `Branch:` in the spec file on dispatch
— edit only, never `git commit` on the main branch.

## On each agent finishing

First, integrity: re-read the task's spec block and compare against what you
last wrote (status, `Repairs:`). An edit you didn't make is a finding — record
a hold, don't trust it.

Ships → the /anchor:check contract (read the file): zero-context checkers,
tier-resolved — two on distinct lenses, or one for mechanically-verified
`Risk: low` tasks. Scouts → one zero-context `anchor-report-checker`
verifying the report answers every question with evidence. Either way, set
`Status: checking` when a task's checkers are dispatched, and when verdicts
return, persist them: a `- Checkers:` line in the task block (e.g.
`- Checkers: behavioral PASS / audit PASS — flags: none`), plus, on any FAIL,
split, or flag, the detail in `docs/reports/<spec-slug>-<task-id>-check.md` —
verdicts and repair reasons must survive the session.

- **Verified + auto-land + not escalated + no checker flags** → apply land's
  canonical merge rules (read the file): pre-flight, tripwire, freshness
  re-test, --no-ff merge, test main, rollback on red, cleanup only after
  green. `Status: landed`. Don't wait for sibling tasks. A tripwire hit,
  checker flags (out-of-scope changes, escalation-category touches, semantic
  red flags), or a dirty main checkout at merge time → don't merge;
  `Status: verified`, queues at /anchor:land.
- **Verified + (gate mode or escalated)** → `Status: verified`, queues for
  /anchor:land.
- **Verified scout** → `Status: verified`; the report queues for the user at
  /anchor:land (scouts never merge; land marks them `landed` once the report
  is presented).
- **Checker failure** (including a freshness re-test gone red during
  auto-land — write that failing test output into the check report file
  first; repair briefs read from disk, not memory) → if the task block has
  no `Repairs:` line, add `Repairs: 1`, set `Status: building`, and dispatch
  one repair agent — a builder in the same worktree for ships, a fresh scout
  on the same report for scouts — briefed with the failure reasons from the
  check report file. Then re-check with the complete contract for the task's
  type — ships: fresh checkers with the tier re-resolved from the post-repair
  diff (repairs can grow a diff past the low bar); scouts: one fresh
  report-checker — never a partial re-run of just the failed criteria.
  Failure with `Repairs: 1` already present → `Status: failed`. Never loop
  further — the counter lives in the spec file, so a crashed session cannot
  forget it.
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
