# ADR-0004 — Fact-level provenance with claim attribution on outputs

**Status:** Accepted · **Date:** 2026-08-13 · **Deciders:** Founder

## Context
Trust is the product. Users must be able to verify any assertion in one click; generated drafts must be structurally incapable of containing unattributed claims. "The model usually doesn't hallucinate" is not a mechanism.

## Decision
Three-part mechanism: (1) **facts + fact_sources** as defined in 02-data-model — no verified fact without a source span; (2) **draft_claims** — drafting decomposes output into claims, each carrying `fact_ids[]`; (3) **hard gates** — a draft cannot transition to `accepted` with any empty-`fact_ids` claim (server-side check + blocking eval in CI measuring attribution faithfulness on the golden question set).

Narrative *style* text (transitions, framing) is exempt; anything checkable (numbers, names, dates, program descriptions, outcome statements) is a claim. The claim-segmentation prompt is versioned; changing it requires eval run.

## Alternatives considered
- **Document-level citations** (like chat citations) — too coarse; can't block a single bad number.
- **Post-hoc fact-checking pass** — checker and generator can be wrong together; we want retrieval-constrained generation with attribution recorded at creation time.

## Consequences
Easier: review UX (click any claim → source quote), audit story, funder-AI-policy compliance narrative. Harder: drafting pipeline is more complex than naive generation; some latency cost. Committing to: attribution evals as permanent blocking CI.

## Devin-facing rules
- Any code path creating a verified fact must create fact_sources in the same transaction.
- Never bypass the accepted-transition check, including in admin/debug tooling.
- New reasoning outputs (future engines) must ship with a claims-attribution design or an Architect note explaining why exempt.
