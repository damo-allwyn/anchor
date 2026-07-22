---
name: land
description: >
  Drain the human queue: gated, escalated, or tripwired verified tasks, failed
  tasks, decision holds, and scout reports. Merging is mechanical after a yes.
  Owns the canonical merge rules — including auto-land merges performed during
  /anchor:build. Merge commits are the only commits anchor makes on main.
disable-model-invocation: true
---

# Land

The user's drain queue. Everything autonomous already happened; what's here
genuinely needs a human.

## Queue — present in this order

1. **Escalated, gated, or tripwired verified tasks** — evidence table:

   | Task | Branch | Checkers | Tests | Diffstat | Why it waits |

   "Why it waits" names the reason: gate mode, `Escalation: required`, or
   `tripwire` (mechanical hit — list the matching files). Plus per task:
   criteria vs. checker findings one line each, split verdicts and spec
   defects flagged. Explicit yes per task (or "all") merges it. Silence,
   "looks fine", "probably ok" — not approval; ask again plainly.
2. **Holds** — each decision fork with the recording agent's recommendation.
   Write the resolution and date into the spec's `## Holds`; a resolved hold
   reopens its task (`Status: todo`, criteria amended if the resolution
   changes them, `Repairs:` line deleted).
3. **Scout reports** — one-paragraph summary each, link to the report file.
   A report is evidence, not authorization: "implement it" means adding a new
   ship task to the spec referencing the report — never acting on the report
   directly.
4. **Failed tasks** — never landable. Reopen: `Status: todo`, delete the
   `Repairs:` line, criteria optionally amended; branch and worktree survive
   for /anchor:build to reuse. Offer "reopen fresh" — delete both first
   (`git branch -D`, `git worktree remove`). Or drop: `Status: abandoned`,
   `git branch -D`, `git worktree remove`.

## Merge rules — canonical, shared by auto-land during /anchor:build

For each task being landed, in this exact order:

1. **Pre-flight.** Main checkout is on the main branch with no uncommitted
   source changes (anchor state — `docs/specs/`, `docs/reports/`,
   `CONTEXT.md` — is exempt). Here at /anchor:land, dirty source → halt and
   ask the user to commit or stash. During /anchor:build auto-land, dirty
   source → don't merge; queue the task here instead.
2. **Escalation tripwire.** `git diff main...<branch> --name-only`, matched
   against: lockfiles (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`,
   `bun.lock*`, `requirements*.txt`, `poetry.lock`, `uv.lock`, `go.sum`,
   `Cargo.lock`), dependency manifests (`package.json`, `pyproject.toml`,
   `go.mod`, `Cargo.toml`, `Gemfile`), `**/migrations/**`,
   `.github/workflows/`, `Dockerfile*`, `*.tf`, `vercel.json`, `fly.toml`,
   `.env*`, plus any `Escalation paths:` entries in CONTEXT.md. A hit on a
   task not marked `Escalation: required` → NEVER auto-land; queue here
   flagged `tripwire`. After explicit human approval the tripwire is
   informational.
3. **Freshness.** If main has moved since the branch's base: `git merge main`
   inside the task's worktree. Conflict → `git merge --abort`, queue the task
   here with the conflicting file list — never auto-resolve. Clean → if the
   sync changed a lockfile, reinstall deps in the worktree; re-run the test
   command there. Red → treat as a checker failure (repair round rules in
   /anchor:build). Green → continue.
4. **Merge.** `git merge --no-ff anchor/<spec-slug>-<task-id>`, then run the
   test command on main.
5. **Red main** → `git reset --keep ORIG_HEAD` (never `--hard` — it would
   destroy the uncommitted spec edits on main), `Status: failed` with a note,
   halt all remaining merges. Branch and worktree survive for repair.
6. **Cleanup — only after a green main test:**
   `git worktree remove ../<repo>-anchor-<spec-slug>-<task-id>` and
   `git branch -d anchor/<spec-slug>-<task-id>`. Then `Status: landed`.

These merge commits are the ONLY commits anchor ever creates on the main
branch — whether merged here after approval or automatically in auto-land
mode. Never `git commit` anything else there: spec updates, reports, and
cleanup edits stay uncommitted, for the user to commit themselves.

When all tasks are landed or resolved, set the spec's header `Status: done`.

## Report

Branches merged, main test result, holds resolved, spec state, anything left open.
