---
name: context
description: >
  Write or refresh CONTEXT.md — the per-project file every anchor command
  reads: stack, test command, terms, dated decisions. Run once per repo,
  refresh on major change. Reads the repo; writes only CONTEXT.md.
disable-model-invocation: true
---

# Context

Write `CONTEXT.md` at the repo root. Explore the repo first (manifests, lockfiles,
CI config, existing docs); ask the user only what the repo cannot answer.

## Required structure

```markdown
# Context: <project>

## Stack
- <language, framework, hosting — one line each>
- Test command: `<command>`     <!-- REQUIRED. /anchor:build refuses without it. -->
- Fast test command: `<command>`<!-- OPTIONAL. Cheap verification (lint +
                                     typecheck + targeted tests) used instead
                                     of the full test command for `Risk: low`
                                     tasks. Absent = low-risk tasks run the
                                     full test command. -->
- Mode: auto-land               <!-- auto-land | gate. Absent = auto-land.
                                     auto-land: verified unescalated tasks merge
                                     without approval; gate: everything waits. -->
- Escalation paths: <globs>     <!-- OPTIONAL. Per-project additions to the
                                     mechanical tripwire in /anchor:land's
                                     merge rules. -->

## Terms
- <project word>: <meaning>

## Decisions
- <YYYY-MM-DD>: <decision + one-line why>
```

## Rules

- `Test command:` is mandatory. If the project has no tests, record the closest
  verification command (build, typecheck, lint) and say so explicitly — checkers
  need SOMETHING runnable. Know the trade-off: the test command is the safety
  floor of auto-land — with a typecheck-only command, anything that typechecks
  can land. Accept that, or set `Mode: gate`.
- `Fast test command:` lowers that floor further for `Risk: low` tasks — a
  deliberate speed trade, backstopped by the mechanical tier verification in
  /anchor:check (a mistagged task upgrades to the full contract). Only record
  one the user actually wants.
- Decisions are append-mostly but curated: when a decision is reversed, rewrite
  the entry to record the reversal — don't leave both truths standing.
- Keep the whole file under a page. It is read by every builder and every
  anchor command; every line is a recurring token cost.
- Do not duplicate what PREFERENCES.md already says globally — CONTEXT.md holds
  only what is specific to THIS project.
