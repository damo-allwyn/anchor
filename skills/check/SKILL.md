---
name: check
description: >
  Verify one built task with two independent zero-context checker agents on
  distinct lenses. Checkers receive ONLY the acceptance criteria, branch, and
  test command — no spec, no conversation, no think summary. Both must pass.
  Read-only over the code; writes only the task status in the spec file.
disable-model-invocation: true
---

# Check

Adversarial verification of one task. The value of this gate is exactly the
checkers' ignorance — protect it.

Ship tasks get two checkers (below). Scout tasks get ONE `anchor-report-checker`
under the same zero-context contract: only the questions, the report path, and
the repo — verdict is whether every question is answered with real evidence.

## Zero-context contract

Spawn **two** `anchor-checker` subagents in parallel. Each prompt contains ONLY:

1. The task's acceptance criteria, verbatim.
2. The worktree path and branch name.
3. The test command.
4. The base branch to diff against (main).
5. Its lens assignment (below) — method emphasis, not extra context.

Nothing else. Not the spec, not the goal, not why the change was requested,
not what the other checker is doing. If you catch yourself adding context "to
help the checker" — stop; that is the failure mode this gate exists to prevent.

## Lenses — decorrelate the two verdicts

- **Checker A — behavioral**: lead with execution — run the test command,
  execute every command named in the criteria, exercise observable behaviour.
- **Checker B — audit**: lead with the diff — scope creep, test-config or
  test-script tampering, touches on escalation-category files (reported as
  flags).

Both checkers verdict EVERY criterion — the lens directs how they look, not
what they cover. Dispatch checker B with a `model` override different from
the session's model (session on opus or fable → `sonnet`; session on sonnet →
`opus`; never haiku) — same-model checkers share blind spots.

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
