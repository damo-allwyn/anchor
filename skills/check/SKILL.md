---
name: check
description: >
  Verify one built task with zero-context checker agents — two on distinct
  lenses, or one for mechanically-verified `Risk: low` tasks. Checkers receive
  ONLY the acceptance criteria, branch, and test command — no spec, no
  conversation, no think summary. All dispatched checkers must pass.
  Read-only over the code; writes only task status and check reports.
disable-model-invocation: true
---

# Check

Adversarial verification of one task. The value of this gate is exactly the
checkers' ignorance — protect it.

Ship tasks get two checkers (below) — or one, when the tier check passes.
Scout tasks get ONE `anchor-report-checker` under the same zero-context
contract: only the questions, the report path, and the repo — verdict is
whether every question is answered with real evidence.

## Tier — proportional checking

Read the task's `Risk:` line (absent = `normal`). `Risk: low` is a claim,
never trusted at face value — verify it mechanically from the diff
(`git diff main...<branch>`) before dispatching:

1. Diffstat ≤ 5 files and ≤ 100 changed lines.
2. No touches on escalation-tripwire paths (the list in land's merge rules,
   plus CONTEXT.md `Escalation paths:`).
3. No touches on the test command's own config or scripts (package.json
   scripts, test configs, CI).

All three hold → dispatch **one** `anchor-checker`, no lens (the agent covers
both emphases when unassigned), no model override. Its test command is
CONTEXT.md's `Fast test command:` when defined, else the test command. It
still reports flags, and flags still block auto-land — the tier changes how
much checking runs, never what blocks a merge. The deliberate trade: low-tier
tasks lose the two-checker verdict decorrelation.

Any condition fails → upgrade to the full contract below (two checkers, both
lenses, model split, full test command) and note the upgrade on the
`- Checkers:` line (e.g. `— tier upgraded: diff 14 files`). A mistagged task
costs nothing in safety.

`Risk: normal` (or absent) → the full contract below.

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
  Verify the worktree is clean: uncommitted tracked changes at handoff are a
  FAIL — the merge takes only what was committed, so behaviour that depends
  on uncommitted files is a lie.
- **Checker B — audit**: lead with the diff — scope creep (reported under
  out-of-scope changes; the pipeline treats it as a flag); test-config or
  test-script tampering (a FAIL); touches on escalation-category files and
  semantic red flags (auth/permission logic edits, destructive operations,
  fetch-and-execute patterns, vendored blobs, install hooks) — reported as
  flags, never a failure by themselves.

Both checkers verdict EVERY criterion — the lens directs how they look, not
what they cover. Dispatch checker B with a `model` override different from
the session's model (session on opus or fable → `sonnet`; session on sonnet →
`opus`; never haiku) — same-model checkers share blind spots.

## Verdict

- **All dispatched checkers PASS** → task verified. Update spec:
  `Status: verified`.
- **Any FAIL** → task not verified. Collect each failing checker's
  criterion-by-criterion reasons and evidence for the repair round.
- **Split verdict** (one pass, one fail) → treat as FAIL, but mark it
  `split` — splits often mean an ambiguous acceptance criterion, which is a
  spec bug worth fixing too.

Persist every outcome (build owns the writes): a `- Checkers:` line in the
task block, and on any FAIL, split, or flag, the detail in
`docs/reports/<spec-slug>-<task-id>-check.md`. Verdicts, flags, and repair
reasons must survive the session — never leave them only in conversation.

A PASS with flags is still a PASS — but flags block auto-land; the task
queues at /anchor:land with the flags on display.

## Rules

- Checkers never fix anything. Failure reasons flow back through /anchor:build's
  repair round, to the builder (or a fresh scout, for report failures).
- Never re-run a checker hoping for a different answer. A flaky verdict is
  itself evidence — report it.
- If a criterion is unverifiable as written (checker cannot tell what would
  satisfy it), report that as a spec defect, not a pass.
