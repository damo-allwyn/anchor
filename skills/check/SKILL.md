---
name: check
description: >
  (anchor) Verify one built task with two independent zero-context checker
  agents. Checkers receive ONLY the acceptance criteria, branch, and test
  command — no spec, no conversation, no think summary. Both must pass.
  Read-only over the code; writes only the task status in the spec file.
disable-model-invocation: true
---

# Check

Adversarial verification of one task. The value of this gate is exactly the
checkers' ignorance — protect it.

Ship tasks get two checkers (below). Scout tasks get ONE checker under the same
zero-context contract: only the questions, the report path, and the repo —
verdict is whether every question is answered with real evidence.

## Zero-context contract

Spawn **two** `anchor-checker` subagents in parallel. Each prompt contains ONLY:

1. The task's acceptance criteria, verbatim.
2. The worktree path and branch name.
3. The test command.
4. The base branch to diff against (main).

Nothing else. Not the spec, not the goal, not why the change was requested,
not what the other checker is doing. If you catch yourself adding context "to
help the checker" — stop; that is the failure mode this gate exists to prevent.

## Verdict

- **Both PASS** → task verified. Update spec: `Status: verified`.
- **Any FAIL** → task not verified. Collect each failing checker's
  criterion-by-criterion reasons and evidence for the repair round.
- **Split verdict** (one pass, one fail) → treat as FAIL, but mark it
  `verdict: split` in the report — splits often mean an ambiguous acceptance
  criterion, which is a spec bug worth fixing too.

## Rules

- Checkers never fix anything. Failure reasons flow back through /anchor:build's
  repair round, to the builder.
- Never re-run a checker hoping for a different answer. A flaky verdict is
  itself evidence — report it.
- If a criterion is unverifiable as written (checker cannot tell what would
  satisfy it), report that as a spec defect, not a pass.
