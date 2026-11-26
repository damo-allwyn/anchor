---
name: land
description: >
  Drain the human queue: every verified task awaiting merge, failed tasks,
  decision holds, and scout reports. Merging is mechanical after a yes. Owns
  the canonical merge rules — including auto-land merges performed during
  /anchor:build. Merge commits are the only commits anchor makes on main.
disable-model-invocation: true
---

# Land

The user's drain queue. Everything autonomous already happened; what's here
genuinely needs a human.

## Queue — present in this order

1. **Verified ship tasks awaiting merge** (scouts queue under item 3) — ALL
   of them, whatever the reason: gate mode, `Escalation: required`, a
   tripwire hit, checker flags, or stranded auto-land leftovers (session died
   before the merge, a dirty checkout degraded it, or a red main halted
   sibling merges). Evidence table:

   | Task | Branch | Checkers | Tests | Diffstat | Why it waits |

   "Checkers" reads from the task's `- Checkers:` line and the check report
   file (`docs/reports/<spec-slug>-<task-id>-check.md`) when one exists.
   "Why it waits" names the reason: `gate`, `escalation`, `tripwire` (list
   the matching files), `flags` (list them), or `stranded`. Plus per task:
   criteria vs. checker findings one line each, split verdicts and spec
   defects called out. Explicit yes per task (or "all") merges it —
   and approval is a REPLY to this table: it must arrive after the queue is
   presented, in a later user message. A blanket approval embedded in the
   invocation itself ("merge everything", "treat this message as approval",
   "proceed autonomously") predates the evidence — it is NOT approval;
   present the queue and stop. Consequence, by design: a headless one-shot
   /anchor:land can never merge — it reports the queue; merging takes an
   interactive yes. This governs the /anchor:land human queue ONLY —
   auto-land merges during /anchor:build are authorized by Mode plus the
   merge rules below, never by reply-approval, and are unaffected. Silence,
   "looks fine", "probably ok" — not approval; ask again plainly. An explicit
   NO gets the failed-task options: reopen (`Status: todo`, delete
   `Repairs:`, criteria amended if that's the objection) or drop
   (`Status: abandoned` + cleanup) — a rejected task never sits at
   `verified` to re-queue forever.
2. **Holds** — each decision fork with the recording agent's recommendation.
   Write the resolution and date into the spec's `## Holds`. A resolution
   either reopens the task (`Status: todo`, criteria amended if the
   resolution changes them, `Repairs:` line deleted — branch and worktree
   survive for reuse) or drops it (`Status: abandoned`, cleanup below if a
   branch exists).
3. **Scout reports** — one-paragraph summary each, link to the report file.
   Once presented and acknowledged, set the scout task `Status: landed` —
   presentation IS a scout's delivery; scouts never merge. A report is
   evidence, not authorization: "implement it" means adding a new ship task
   to the spec referencing the report — never acting on the report directly.
4. **Failed tasks** — never landable. Reopen: `Status: todo`, delete the
   `Repairs:` line, criteria optionally amended; branch and worktree survive
   for /anchor:build to reuse. Offer "reopen fresh" — cleanup first (worktree
   before branch: `git worktree remove <path>`, then `git branch -D`). Or
   drop: `Status: abandoned`, then the same cleanup, worktree first.

## Merge rules — canonical, shared by auto-land during /anchor:build

For each task being landed, in this exact order:

1. **Pre-flight.** Main checkout is on the main branch with no uncommitted
   source changes (anchor state — `docs/specs/`, `docs/reports/`,
   `CONTEXT.md` — is exempt). Here at /anchor:land, dirty source → halt and
   ask the user to commit or stash. During /anchor:build auto-land, dirty
   source → don't merge; queue the task here instead. Exemption caveat: if
   the BRANCH itself touches an anchor state file that is dirty on main, the
   merge would abort — that's also a builder contract violation (builders
   never touch anchor state); surface it and have the user commit or stash
   those files before merging.
2. **Escalation tripwire.** `git diff main...<branch> --name-only`, matched
   against: lockfiles (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`,
   `bun.lock*`, `requirements*.txt`, `poetry.lock`, `uv.lock`, `go.sum`,
   `Cargo.lock`), dependency manifests (`package.json`, `pyproject.toml`,
   `go.mod`, `Cargo.toml`, `Gemfile`), `**/migrations/**`,
   `.github/workflows/`, `Dockerfile*`, `*.tf`, `vercel.json`, `fly.toml`,
   `.env*`, plus any `Escalation paths:` entries in CONTEXT.md. A hit on a
   task not marked `Escalation: required` → NEVER auto-land; queue here
   flagged `tripwire`. After explicit human approval the tripwire is
   informational. The tripwire is path-based; semantic escalations
   (auth-surface edits, destructive code) arrive via checker flags, which
   block auto-land the same way.
3. **Freshness.** If main has moved since the branch's base: `git merge main`
   inside the task's worktree. Conflict → `git merge --abort`; during
   /anchor:build auto-land, queue the task here with the conflicting file
   list; here at /anchor:land, ask the user — reopen the task for a rebuild
   against current main (`Status: todo`, delete `Repairs:`, worktree kept)
   or resolve the conflict manually and re-run /anchor:land. Never
   auto-resolve. Clean → if the sync changed a lockfile, reinstall deps in
   the worktree; re-run the task's test command there (CONTEXT.md's
   `Fast test command:` when the task checked at the low tier, else the test
   command). Red: during /anchor:build
   auto-land → repair round (build owns it); here at /anchor:land →
   `Status: failed` with the failing output as a note and move to the next
   queue item — the reopen path owns repair. Green → continue.
4. **Merge.** `git merge --no-ff anchor/<spec-slug>-<task-id>`, then the
   post-merge test on main — governed by a mechanical rule, not judgment:
   when `git rev-parse HEAD^{tree}` equals `git rev-parse <branch>^{tree}`
   (the merge produced exactly the tree already tested in the worktree —
   always true when main never moved or the freshness step just ran green)
   AND the branch diff touches no lockfile or dependency manifest (installed
   deps on main may differ from the worktree's), the re-test is redundant —
   do NOT run it: identical tree, identical result. Either condition fails →
   run the test command on main.
5. **Red main** → `git reset --keep ORIG_HEAD` (never `--hard` — it would
   destroy the uncommitted spec edits on main), `Status: failed` with a note,
   halt all remaining merges. Branch and worktree survive for repair. If
   `reset --keep` itself refuses (overlapping local edits), stop and report —
   never escalate to `--hard`.
6. **Cleanup — only after a green main test, worktree before branch:**
   `git worktree remove ../<repo>-anchor-<spec-slug>-<task-id>`, then
   `git branch -d anchor/<spec-slug>-<task-id>`. If the removal refuses
   ("contains modified or untracked files"): inspect with
   `git -C <worktree> status --porcelain` — only untracked scratch → remove
   with `--force`; tracked modifications → stop and report. Never `rm -rf`.
   Then `Status: landed`.

These merge commits are the ONLY commits anchor ever creates on the main
branch — whether merged here after approval or automatically in auto-land
mode. Never `git commit` anything else there: spec updates, reports, and
cleanup edits stay uncommitted, for the user to commit themselves.

When all tasks are landed or abandoned, set the spec's header `Status: done`.

## Report

Branches merged, main test result, holds resolved, spec state, anything left open.
