# ADR-0007 — Inngest for durable background jobs

**Status:** Accepted · **Date:** 2026-08-13 · **Deciders:** Founder

## Context
Extraction, drafting, reindexing, and the nudge engine are long-running and must never be silently lost. The nudge engine additionally needs reliable cron + scheduled sends. We do not want to operate our own queue infra.

## Decision
Inngest for all async work: event-driven functions, step retries, cron. Job inventory lives in 01-services.md and is the authoritative list. Failures page via Sentry alerting.

## Alternatives considered
Temporal (more powerful, much heavier to operate/learn), BullMQ+Redis (adds Redis, violates ADR-0001's one-datastore stance), Vercel cron + fire-and-forget (unacceptable durability).

## Consequences
Easier: retries/backoff/idempotency semantics handled; local dev server. Harder: vendor coupling for orchestration (exit: functions are thin wrappers around module logic, so migration is re-wiring, not rewrite). Committing to: idempotency keys on every event.

## Devin-facing rules
- Every event has a zod schema in shared/contracts/events.ts and an idempotency key.
- Handler bodies delegate to module functions; no business logic inline in Inngest functions.
- Any new event/cron requires updating the job inventory table in 01-services.md in the same PR.
