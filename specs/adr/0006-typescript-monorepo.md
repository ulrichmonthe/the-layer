# ADR-0006 — TypeScript monorepo (Turborepo), Next.js, tRPC, modular monolith

**Status:** Accepted · **Date:** 2026-08-13 · **Deciders:** Founder

## Context
One human orchestrating agents. Agents perform best in a single-language, single-repo, strongly-typed codebase with end-to-end type inference — every type error is a free quality gate.

## Decision
Turborepo monorepo, TypeScript strict everywhere, Next.js (App Router) for web, tRPC for internal API, Drizzle for DB, zod for all boundaries, Vitest + Playwright for tests. Deploy: Vercel (web) + Fly.io or Railway (worker + LiteLLM). Structure per 01-services.md.

## Alternatives considered
Python for the AI pipeline + TS front (two toolchains doubles agent context and CI complexity; TS AI tooling is now sufficient); Nx (heavier than needed).

## Consequences
Easier: one lint/test/typecheck pipeline gating everything; shared zod contracts. Harder: if we later need heavy Python-ecosystem ML, we add a sidecar service behind an API (clean seam). Committing to: strict mode stays on; no `any` escapes.

## Devin-facing rules
- `pnpm` only. No new deps without ticket-level approval; justify in PR body.
- No `any`, no `@ts-ignore` (use `@ts-expect-error` with a comment + linked issue if truly needed).
- Server components by default; client components only with a reason comment.
