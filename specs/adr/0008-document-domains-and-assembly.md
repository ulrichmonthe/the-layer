# ADR-0008 — Document domains as a facet, and grant document assembly

**Status:** Accepted · **Date:** 2026-08-13 · **Deciders:** Founder (+ Architect review)

## Context
`00-overview.md` lists **document assembly** as one of the seven lifecycle stages the brain projects into, but no service, contract, or table was ever written for it — `01`, `02` and `03` cover match, draft, translate and triage only. Separately, the corpus has no way to hold the operational documents funders demand as attachments: the 501(c)(3) determination letter, audited financials, board roster, insurance certificates, W-9, bylaws. Today they land in `documents.kind:'other'` and dissolve. Chasing those files, and noticing when one has expired, is real recurring work for a two-person team. The question raised was whether this means departments become a first-class concept in the product.

## Decision
Departments are **a label on a document, not a boundary**. Add a `domain` facet to `documents` (`governance · legal · finance · hr · insurance · program · grant`) alongside the existing `kind`, plus document validity dates. Build assembly as deterministic workflow state: `requirements` per opportunity, `attachments` binding documents to them, and a **derived** gap list of what is missing or expiring. One corpus, one retrieval path, one tenancy key. No change to ADR-0001 or ADR-0002.

## Alternatives considered
- **Departments as first-class (a `departments` table, per-department permissions)** — reopens RLS (ADR-0001), authorization (ADR-0002) and extraction (ADR-0003, since contracts need different extraction than proposals), and serves an org larger than the 1–2 person team this product targets. It is a different product, and adopting it by drift would be the worst way to get there.
- **Do nothing; let `kind:'other'` absorb operational files** — cheapest, but leaves the document-assembly promise in `00-overview` permanently broken and the recurring expiry problem unsolved.
- **Assembly as a reasoning engine** — rejected. "Missing" and "expired" are date maths and null checks. Product constraint 4 says deterministic state is never an LLM inference, and a fabricated "you have everything" is exactly the failure this product cannot afford.

## Consequences
Easier: operational documents become first-class without new infrastructure; the gap list is cheap, testable and explainable; `domain` gives retrieval a useful filter. Harder: someone must set `domain` — extraction guesses it as a candidate, a human confirms it, same loop as any other fact. Committing to: maintaining the domain enum as a closed list, and keeping the gap list derived rather than materialised.

Exit cost if departments must later become a real permission boundary: low. `documents.domain` is already the label a policy would key off, so the work is adding an RLS policy and a role model — not re-filing the corpus.

## Devin-facing rules
- `documents.domain` is a label, **not** a permission boundary. RLS stays `org_id`-only. Do not add a `departments` table, a domain-scoped role, or a second tenancy key.
- One corpus, one retrieval path. `core.search` gains an optional `domain` filter; nothing else changes. No separate index, no per-domain embedding space.
- The gap list is derived and deterministic — computed from `attachments` and `documents.valid_until`. Never compute "missing" or "expired" with a model.
- `requirements` and `attachments` are workflow tables. `reasoning` may read them; it never writes them.
- Missing required attachments **flag**, they do not block export in v1. Adding a blocking check requires a new ADR.
