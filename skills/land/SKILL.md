---
name: land
description: >
  (anchor) Drain the human queue: gated or escalated verified tasks, failed
  tasks, decision holds, and scout reports. Merging is mechanical after a yes.
  Owns the merge rules — including auto-land merges performed during
  /anchor:build. Merge commits are the only commits anchor makes on main.
disable-model-invocation: true
---

# Land

The user's drain queue. Everything autonomous already happened; what's here
genuinely needs a human.

## Queue — present in this order

1. **Escalated or gated verified tasks** — evidence table:

   | Task | Branch | Checkers | Tests | Diffstat | Why it waits |

   Plus per task: criteria vs. checker findings one line each, split verdicts
   and spec defects flagged. Explicit yes per task (or "all") merges it.
   Silence, "looks fine", "probably ok" — not approval; ask again plainly.
2. **Holds** — each decision fork with the recording agent's recommendation.
   Write the resolution and date into the spec's `## Holds`; a resolved hold
   reopens its task (`Status: todo`, criteria amended if the resolution
   changes them).
3. **Scout reports** — one-paragraph summary each, link to the report file.
   A report is evidence, not authorization: "implement it" means adding a new
   ship task to the spec referencing the report — never acting on the report
   directly.
4. **Failed tasks** — never landable. Reopen (`Status: todo`, criteria
   optionally amended) or drop (delete branch and worktree, mark abandoned).

## Merge rules — canonical, shared by auto-land during /anchor:build

For each task being landed:

```
git merge --no-ff anchor/<spec-slug>-<task-id>
git worktree remove ../<repo>-anchor-<task-id>
git branch -d anchor/<spec-slug>-<task-id>
```

These merge commits are the ONLY commits anchor ever creates on the main
branch — whether merged here after approval or automatically in auto-land
mode. Never `git commit` anything else there: spec updates, reports, and
cleanup edits stay uncommitted, for the user to commit themselves.

Run the test command on main after each merge; a red main halts landing
immediately — report, do not continue with remaining merges.

Update spec: `Status: landed`. When all tasks are landed or resolved, set the
spec's header `Status: done`.

## Report

Branches merged, main test result, holds resolved, spec state, anything left open.
