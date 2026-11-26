---
name: spec
description: >
  Turn a settled idea into a written spec with tasks sized for one builder
  each, saved to docs/specs/. Consumes the Anchor Think Summary from the
  conversation or pasted in. Writes exactly one file. Ends by pointing to
  /anchor:build.
disable-model-invocation: true
---

# Spec

Turn the agreed idea into a durable spec file.

## Input

Look for an `## Anchor Think Summary` block in the conversation (live or pasted
from a previous session). If none exists, do NOT refuse — run one compressed
confirmation pass: state your best understanding of the seven think items,
get a confirm-or-correct, then proceed. Point the user at /anchor:think only
if the idea turns out to be genuinely unsettled.

## Output

Write `docs/specs/<YYYY-MM-DD>-<slug>.md` (create the directory if needed):

```markdown
# <Title>

Status: draft
Source: <one-line problem statement>

## Goal
## Non-goals
## Riskiest assumption

## Tasks

### T1: <short name>
- Type: ship              <!-- ship (code change) | scout (investigate → report) -->
- Status: todo            <!-- todo | building | checking | verified | failed | held | landed | abandoned -->
- Branch: —
- Escalation: none        <!-- ALWAYS present: none | required — <reason>.
                               Assess every task against the invariant
                               categories (see /anchor:build). -->
- Risk: normal            <!-- ALWAYS present on ship tasks: low | normal.
                               low = small, contained, behaviour-light
                               (copy, styles, docs): one checker + fast test
                               command. Verified mechanically at check time
                               from the diff; a mismatch upgrades to the
                               full contract. Scouts ignore this line. -->
- Acceptance criteria:
  - <criterion>
  - <criterion>
<!-- /anchor:build adds `- Repairs: 1` when a repair round is spent
     (/anchor:land deletes it on reopen) and a `- Checkers:` verdict line
     after each check; failure/flag detail lands in
     docs/reports/<spec-slug>-<task-id>-check.md. -->


## Holds
<!-- decision forks recorded by agents; user resolves at /anchor:land -->
- [ ] <YYYY-MM-DD> <task>: <decision needed> — recommendation: <one line>
```

## Task rules

- Each task sized for one builder agent in one session. Split rather than stuff.
- **Acceptance criteria must be self-contained.** A checker agent with zero
  project context — no spec, no conversation — must be able to verify the task
  from its criteria alone. Name exact files, commands, observable behaviours.
  "Works correctly" is not a criterion; "`npm test` passes and `GET /api/items`
  returns 200 with a JSON array" is.
- Prefer criteria a machine can check (command output, file existence, test
  results) over criteria requiring judgment.
- Order tasks by dependency; note blocking relationships inline.
- Scout tasks: acceptance criteria are the questions the report must answer —
  each answerable with evidence from the repo. Scouts never change code.
- The `Escalation:` line is mandatory on every task — `none` or
  `required — <reason>`. Assess against the invariant categories
  (destructive/irreversible ops, shared-DB schema changes, new dependencies,
  security/auth surface, deploys); `required` never auto-lands. Writing
  `none` is a claim you make deliberately, not a default to skip past.
- The `Risk:` line is the same kind of deliberate claim: `low` buys a lighter,
  faster check (one checker, fast test command) and is verified mechanically
  against the actual diff at check time — an oversized or sensitive diff
  upgrades to the full contract automatically. When unsure, write `normal`.

Show the user the spec, apply corrections, then end with:

```
Spec written: docs/specs/<file>

**Next:** run /anchor:build
```
