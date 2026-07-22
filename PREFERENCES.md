# Preferences

Global stack and implementation preferences. Read by /anchor:think (as defaults
instead of questions) and included in every builder brief. Per-project overrides
live in that project's CONTEXT.md and win.

## Stack defaults

- Web: Next.js (App Router) / TypeScript / Tailwind / Vercel; shadcn/ui
  components; Framer Motion for transitions and micro-interactions
- Data & auth: Supabase (PostgreSQL + Auth)
- Scripting & backend: Python, bash
- Infra: AWS / Terraform
- AI: OpenRouter (LLMs), fal.ai (media), vector embeddings for similarity
- Automation: n8n webhooks
- Design: Figma

## PoC mindset

Most work is prototypes validating ideas, not enterprise software.

- If something takes >30 minutes to implement, find a simpler approach.
- Copy-paste is fine; extract later if needed. No abstraction layers "for later".
- Mock data and placeholder content encouraged — fake it till you make it.
- TypeScript: fix errors, don't over-type. `any` is acceptable in PoCs.
- No formal test suites in PoCs. Verification = bash-first validation (below)
  plus the closest runnable command (build / typecheck / lint) recorded as the
  `Test command:` in CONTEXT.md.

## Bash-first validation

New integrations and non-trivial logic get proven in isolation BEFORE app code.

- Apply to: API integrations, unfamiliar libraries or services, data
  transformations and parsing, multi-step data flows, algorithms.
- Skip for: UI-only changes, simple CRUD on established patterns, copy/config
  changes, new pages following existing patterns.
- Flow: break into testable hypotheses → minimal disposable scripts in
  `scripts/scratch/` (gitignored) → run, inspect, iterate → only then write app
  code, mirroring the proven prototype.
- Secrets come from `.env.local` env vars — never hardcoded.
- If validation fails after reasonable effort: report findings and stop. Don't
  push an unproven approach into app code.

## LLM prompt work

- New prompts and major revisions get iterative testing in a scratch script
  before implementation: many runs against a fast cheap model, checkpoint
  analysis every few iterations, keep the winning prompt with its score.

## Conventions

- Files `kebab-case`, components `PascalCase`, functions/variables `camelCase`.
- SQL migrations: sequential numbered files in `supabase/migrations/`
  (`###_descriptive_name.sql`), additive-only against shared databases (new
  tables, nullable columns).
- Next.js: server components by default (`'use client'` only when needed),
  server actions for mutations, route handlers for APIs, flat component trees.

## What NOT to do (PoCs)

- No comprehensive error handling; no loading skeletons for every state.
- No comments explaining obvious code.
- No analytics/logging/monitoring beyond explicitly requested product events.
- No time on SEO or meta tags.

## Git

- Never author commits on the main branch. The only main-branch commits agents
  create are merges of verified (auto-land) or user-approved (gate) task
  branches. Everything else touching main — spec updates, reports, context
  edits, cleanups — stays uncommitted for the user to commit themselves.

## When stuck

- Search the error (web, Stack Overflow), check latest official docs, then
  simplify — a PoC needs a working solution, not the "right" one. After a
  reasonable failed attempt: stop, state what's broken and why, propose a new
  direction. Never run in circles.

## Maintaining this file

- Curated, not append-forever: rewrite or prune entries rather than piling on.
- Only durable preferences — no project specifics (those go in CONTEXT.md),
  no one-off decisions.
- Update flow: edit in the anchor repo, commit, push; other machines pick it
  up on plugin update.
