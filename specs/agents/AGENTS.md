# AGENTS.md — Rules of engagement for coding agents in this repository

You are implementing GrantBrain, an AI knowledge platform for nonprofit grant teams. The architecture, data model, and all binding decisions live in the `specs` repo (mirrored summaries below). **Your job is implementation within decided boundaries. You do not make architectural decisions.**

## The prime rule

If completing your ticket requires a choice that is not answered by (a) the ticket, (b) this file, or (c) the specs — **stop and ask in the ticket/PR thread. A question is a successful outcome. A guessed decision is a failed one.**

Architectural choices you must never make unilaterally: adding dependencies, adding packages/modules or moving module boundaries, changing database schema beyond what the ticket specifies, adding external services, changing API contracts in `packages/shared/contracts/`, changing prompts without version bump, adding events/crons, changing auth or RLS behavior.

## Repository map

```
apps/web        Next.js UI — imports ONLY @grantbrain/api and @grantbrain/shared
apps/worker     Inngest handlers — thin wrappers, delegate to modules
packages/api    tRPC routers — the only business-logic entry from web
packages/ingestion | core | reasoning | workflow   domain modules
packages/llm    ALL model calls + versioned prompts. No provider SDKs elsewhere.
packages/evals  Braintrust suites + golden datasets
packages/shared types, zod schemas, contracts, errors
```

## Hard rules (violations = PR rejected)

1. **Tenancy:** every query on tenant tables filters by `org_id` derived from the server session — never from client input. New tenant tables get `org_id` + RLS policy in the same migration.
2. **Provenance:** creating a `facts` row with `status:'verified'` requires ≥1 `fact_sources` row in the same transaction. Draft transitions to `accepted` must pass the claims-attribution check. Never bypass, including in scripts.
3. **Approvals:** any code path that sends/export-finalizes externally must require a granted `approvals` row, enforced server-side.
4. **LLM discipline:** model calls only via `packages/llm`. Prompt edits = new version file + eval run noted in PR. No parsing model output with regex; use `llm/structured.ts` helpers.
5. **Async discipline:** anything that must not be lost goes through Inngest with an idempotency key. No fire-and-forget promises for side effects.
6. **Types:** TypeScript strict. No `any`, no `@ts-ignore`. Zod-validate every external boundary (HTTP, LLM output, file parsing, webhooks).
7. **Determinism:** deadlines, amounts, statuses, commitments are read from the database, never inferred by an LLM at runtime.
8. **Logs:** no user document content or PII in logs or error messages. Use IDs.

## Conventions

- pnpm; conventional commits; one ticket = one PR = one concern.
- Tests: Vitest colocated (`*.test.ts`); every ticket's acceptance criteria map to named tests; Playwright for flows the ticket marks E2E.
- Migrations: Drizzle only; migration files reviewed as part of PR; never edit an applied migration.
- Naming: table and column names exactly as in `specs/architecture/02-data-model.md`. Do not "improve" names.
- UI: implement from the written design spec in the ticket; server components by default; design tokens only — no ad-hoc colors/spacing.
- Errors: typed error codes from `shared/errors.ts` (`PROVENANCE_VIOLATION`, `APPROVAL_REQUIRED`, ...). Never throw bare strings.

## Definition of done (self-check before opening PR)

- [ ] All acceptance criteria have a passing named test
- [ ] `pnpm typecheck && pnpm lint && pnpm test` green locally
- [ ] No new dependencies (or ticket explicitly authorized them; justified in PR body)
- [ ] No schema/contract/prompt changes beyond ticket scope
- [ ] PR description: what changed, how tested, any questions raised, which specs sections you relied on
- [ ] Out-of-scope discoveries listed at the bottom of the PR as `FOLLOW-UP:` items — do NOT fix them in this PR

## When tests or CI fail

Fix forward within ticket scope. If the failure reveals a spec gap or a conflict between specs and reality, stop and flag with the exact file/line of the conflict. Do not reinterpret the spec to make tests pass.

## Session hygiene

Work only the assigned ticket. Do not refactor adjacent code "while you're there." Do not update dependencies. Do not modify this file, anything under `specs/`, CI workflows, or eval gates — those changes come from the human only.
