# anchor

Personal software development framework for Claude Code. Idea → spec → parallel
agent build → zero-context verification → human-gated landing.

Influences: [muster](https://github.com/markstent/muster) (skill pipeline shape),
[firstmate](https://github.com/kunchenguid/firstmate) (evidence discipline,
memory routing, gate philosophy) — rebuilt lightweight on native Claude Code
primitives.

## Pipeline

```
/anchor:think    interrogate the idea (proportional), emit durable summary
/anchor:spec     write docs/specs/<date>-<slug>.md — ship + scout tasks with
                 self-contained acceptance criteria
/anchor:build    ≤3 tasks in parallel; ships get worktree + branch (based on
                 main, bootstrapped: deps + .env symlinks), scouts investigate
                 read-only; verified unescalated tasks auto-land; stale tasks
                 from crashed runs get reclaimed
/anchor:check    2 blind checkers per ship on distinct lenses (behavioral /
                 audit, different models); 1 report-checker per scout; all
                 must pass; verdicts + flags persist to the spec and
                 docs/reports/*-check.md
/anchor:land     drain the human queue: escalated/gated/tripwired tasks,
                 decision holds, scout reports, failures; owns the canonical
                 merge rules (tripwire → freshness re-test → --no-ff → test
                 main → rollback on red → cleanup after green)
```

Utilities: `/anchor:context` (per-repo CONTEXT.md, once), `/anchor:status`
(read-only snapshot).

## Principles

- **Autonomy by default, control by policy.** The human is the bottleneck —
  protect their attention. Verified unescalated work lands itself
  (`Mode: auto-land`, the default); control lives in invariant escalations,
  blind verification, and evidence trails — not per-task permission. Repos
  that want approval on everything set `Mode: gate` in CONTEXT.md.
- **Invariant escalations, always.** Destructive/irreversible ops, shared-DB
  schema changes, new dependencies, security/auth surface, deploys — never
  auto-land, regardless of mode. Tagging is backstopped twice: a mechanical
  path tripwire at merge time (lockfiles, dependency manifests, migrations,
  workflows, deploy files, `.env*`) catches what paths can catch; the audit
  checker's flags (auth-logic edits, destructive code, fetch-and-execute,
  vendored blobs, install hooks) catch the semantic rest. Either one blocks
  auto-land. Agent conduct itself is instruction-guarded, not
  sandbox-guarded — these backstops exist to catch what instructions miss.
- **Decisions are a queue, not a blocker.** An agent hitting a genuine fork
  records a hold with a recommendation and parks that task (`held`, partial
  work preserved in its worktree); the run keeps working on other tasks. You
  drain holds in batch at /anchor:land.
- **A report is evidence, not authorization.** Scouts investigate read-only;
  implementing a recommendation requires a promoted ship task.
- **Main is yours.** Agents never author commits on the main branch — the only
  main-branch commits anchor creates are --no-ff merges of verified (auto-land)
  or approved (gate) task branches. A merge that turns main red is rolled back
  (`git reset --keep ORIG_HEAD`) and halts further landing. Spec, report, and
  context edits stay uncommitted for you to commit.
- **Checkers are blind on purpose.** They get acceptance criteria, a branch,
  and a test command — nothing else. They verify what was asked, not what was
  meant.
- **State is plain files.** Per-project: `CONTEXT.md` + `docs/specs/*.md`
  (task status, repair counters — everything a crashed session must not
  forget — lives in the spec). Global: `PREFERENCES.md` in this repo.
- **The engine is a seam.** Worktree isolation is done with plain `git worktree`;
  agent teams vs. background subagents is an implementation detail of
  /anchor:build.

## Install (per machine)

```
/plugin marketplace add <github-user>/anchor
/plugin install anchor@anchor
```

Update everywhere: push to this repo, then `/plugin marketplace update` on each
machine.

Optional: agent teams engine requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
in the environment. Without it, /anchor:build falls back to background
subagents automatically.

## Layout

```
.claude-plugin/   plugin.json + marketplace.json (repo is its own marketplace)
skills/           think, spec, build, check, land, context, status
agents/           anchor-builder, anchor-checker, anchor-report-checker,
                  anchor-scout
PREFERENCES.md    global stack + implementation preferences
```

## Per-project requirements

- `CONTEXT.md` with a `Test command:` line — /anchor:build refuses to run
  without one (checkers need something runnable; the test command is also the
  safety floor of auto-land). Optional `Mode:` line (`auto-land` | `gate`,
  absent = auto-land) and `Escalation paths:` line (per-project tripwire
  additions).
- `docs/specs/` — created by /anchor:spec; `docs/reports/` — scout reports.
