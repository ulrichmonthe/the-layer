# ADR-0003 — Document extraction vendor behind an internal interface

**Status:** Accepted · **Date:** 2026-08-13 · **Deciders:** Founder

## Context
Extraction quality on messy nonprofit PDFs is our hardest dependency and the vendor market (Reducto, LlamaParse, Unstructured, Azure DI) is moving fast. We must be able to benchmark and swap without touching product code.

## Decision
Define `Extractor` interface in `packages/ingestion/extractor/`: `extract(document) → { blocks[], tables[], candidateFacts[], confidence }`. Ship v1 with **Reducto** as the implementation, chosen pending our own benchmark on 20 labeled real documents (golden set). The benchmark harness lives in `packages/evals` and is a Phase 1 exit requirement.

## Alternatives considered
Building parsing ourselves — months of runway on a solved-enough problem. Committing hard to one vendor with direct SDK calls throughout — swap cost becomes a rewrite.

## Consequences
Easier: A/B vendors on the same golden set; per-document-kind routing later (e.g., different engine for budget spreadsheets). Harder: interface is a lowest-common-denominator; vendor-specific goodies need explicit interface extension. Committing to: maintaining the golden set as pilot documents arrive.

## Devin-facing rules
- No vendor SDK imports outside `extractor/impl-*`. Product code sees only the interface.
- Candidate facts from extraction are ALWAYS `status:'candidate'` with a confidence score and ≥1 source span. Auto-promotion rules live in one file: `ingestion/promotion-rules.ts`.
- Extraction runs only in the worker (retries, timeouts per ADR-0007), never in a request handler.
