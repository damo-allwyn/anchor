---
name: think
description: >
  (anchor) Pressure-test an idea through focused interrogation until the plan is
  settled, then emit a durable summary block that survives session loss or
  compaction. Proportional: small ideas get one confirmation pass, not the full
  interview. Writes nothing to disk. Ends by pointing to /anchor:spec.
disable-model-invocation: true
---

# Think

Interrogate this idea with the user until you both understand it the same way.
Address the user in second person throughout.

## Before asking anything

1. If `CONTEXT.md` exists in the repo, read it. Never ask a question it already
   answers (stack, past decisions, terms).
2. Read `PREFERENCES.md` from the anchor plugin directory
   (`${CLAUDE_PLUGIN_ROOT}/PREFERENCES.md`; when working inside the anchor repo
   itself, `./PREFERENCES.md`). Apply stated stack and implementation
   preferences as defaults instead of asking about them.
3. If a question can be answered by exploring the codebase, explore the
   codebase instead of asking.

## Proportionality

First, silently size the idea:

- **Small** — one behaviour, no new systems, no data model changes: skip the
  interview. State your understanding of all seven checklist items in one recap
  and ask for a single confirm-or-correct.
- **Everything else** — full interrogation: one question at a time, each with
  your recommended answer, until every checklist item is settled. Chase new
  questions that answers open up.

## What to resolve

- The exact problem, from the user's perspective. Why this, why now?
- Who the user of the thing is. For personal projects: which version of them, doing what?
- The single most important behaviour.
- What "done" looks like — how it will be verified.
- What is explicitly NOT being built.
- The riskiest assumption in the plan.
- What existing code or systems this touches.

## Rules

- Treat vague answers ("roughly", "something like that") as unresolved. Ask again.
- Do not propose implementations unless asked. "What it touches" names systems,
  not designs.

## Closing

When everything is settled, recap ALL seven items back in plain terms and ask
whether that matches. If corrected: settle the new question, recap again.

On confirmation, print this block verbatim-filled — it is the durable handoff
that survives session loss (the user can paste it into a fresh session):

```
## Anchor Think Summary: <slug>
- Problem: ...
- For: ...
- Key behaviour: ...
- Done means: ...
- Not building: ...
- Riskiest assumption: ...
- Touches: ...
```

Then end with:

```
Settled.

**Next:** run /anchor:spec
```
