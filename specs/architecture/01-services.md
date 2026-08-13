# 01 — Service Decomposition

**Status:** Accepted · **Last updated:** 2026-08-13

We run a **modular monolith** in a TypeScript monorepo (ADR-0006): one deployable app + one worker process, with hard internal module boundaries. We are not doing microservices at this scale; we ARE keeping module seams clean enough that any module could be split out later.

```
apps/
  web/            Next.js app (Experience layer) — UI only, calls packages/api
  worker/         Inngest worker — all background jobs
packages/
  api/            tRPC (or REST) routers — the ONLY entry to business logic from web
  ingestion/      upload handling, extraction orchestration, confirmation queue
  core/           knowledge core: schema, repositories, fact/provenance/conflict logic
  reasoning/      match, draft, translate, triage engines (stateless)
  workflow/       pipeline, deadlines, tasks, approvals, assembly, nudge scheduling
  llm/            gateway client, prompt registry, retrieval utilities
  evals/          Braintrust suites, golden datasets, scorers
  shared/         types, zod schemas, errors, config
```

## Module rules (enforced; also stated in AGENTS.md)

1. `web` imports only from `api` and `shared`. Never from `core`, `reasoning`, etc.
2. `reasoning` reads the core via `core`'s repository interfaces; it never writes to the database.
3. All LLM calls go through `llm`. No direct model-provider SDK imports anywhere else. Every prompt lives in `llm/prompts/` with a version string.
4. All background work goes through `worker` via Inngest events (ADR-0007). No fire-and-forget promises for anything that must not be lost.
5. Cross-module calls use exported interfaces, not deep imports. `import ... from '@grantbrain/core/src/internal/...'` is forbidden.
6. Every module exposes its zod schemas from `shared` so API contracts (03) stay in one place.

## Job inventory (worker)

| Event | Handler module | Notes |
|---|---|---|
| `document.uploaded` | ingestion | store → enqueue extraction |
| `document.extract` | ingestion | vendor call, retries, confidence scoring |
| `facts.candidates_ready` | ingestion | conflict detection vs existing facts; build confirmation queue |
| `draft.requested` | reasoning | long-running; streams progress to UI |
| `nudge.scan` (cron) | workflow | daily: find due deadlines/stale tasks, plus expiring documents and open assembly gaps (ADR-0008) → compose nudges |
| `nudge.send` | workflow | requires approval record if outbound to third party |
| `corpus.reindex` | core | re-embed on chunking/prompt-version change |

## What Devin may and may not touch

- May: implement inside modules per tickets; add tests; add migrations **when the ticket explicitly says so**.
- May not: add packages/modules, change module boundaries, add external dependencies, or alter events/schemas without a ticket that cites an ADR or an Architect sign-off note.
