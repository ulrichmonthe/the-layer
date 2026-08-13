# ADR-0005 — All model calls through a gateway + versioned prompt registry

**Status:** Accepted · **Date:** 2026-08-13 · **Deciders:** Founder

## Context
We need cost tracking, caching, provider failover, and the ability to change models without hunting call sites. Evals (Braintrust) need every call traced with prompt version metadata.

## Decision
LiteLLM (self-hosted proxy) as the gateway; `packages/llm` is the only module that talks to it. Prompts live in `packages/llm/prompts/<name>/<version>.ts` with typed input/output (zod). Every call logs: prompt name+version, model, token counts, latency, trace id → Braintrust.

## Alternatives considered
Portkey (managed; fine, slightly less control), direct SDKs with a thin wrapper (wrapper always erodes), Helicone (gateway in maintenance mode under new ownership).

## Consequences
Easier: model swaps, spend caps per org, cached retrieval calls, complete eval traces. Harder: one more running service (acceptable: it's stateless). Committing to: prompt changes ALWAYS bump version and run the eval suite.

## Devin-facing rules
- No `openai`/`anthropic` SDK imports outside `packages/llm`.
- Never edit a prompt in place — copy to new version file, update the caller, note in PR description; CI runs evals against the new version.
- No streaming responses parsed with regex; use the structured-output helpers in `llm/structured.ts`.
