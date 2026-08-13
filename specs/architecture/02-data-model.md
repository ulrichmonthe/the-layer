# 02 — Knowledge Core Data Model

**Status:** Accepted (v1 scope) · **Last updated:** 2026-08-13

Postgres + pgvector (ADR-0001). Every table carries `org_id` (tenant isolation, RLS), `created_at`, `updated_at`. IDs are UUIDv7. Names below are canonical — Devin must not rename.

## A. Identity & tenancy

**organizations** — the customer workspace.
`id, name, ein, fiscal_year_start, settings jsonb`

**users** — Clerk-backed (ADR-0002).
`id, org_id, clerk_id, role enum('admin','editor','reviewer')`

## B. Document corpus

**documents** — every ingested file.
`id, org_id, kind enum('proposal','report','budget','financial_statement','990','letter','other'), domain enum('governance','legal','finance','hr','insurance','program','grant'), title, source enum('upload','email','drive'), storage_key, mime, status enum('uploaded','extracting','extracted','failed'), fiscal_year int null, funder_id null, valid_from date null, valid_until date null`

`kind` is what the document *is*; `domain` is which part of the organisation it belongs to (ADR-0008). Both are facets on one corpus — `domain` is a label, never a permission boundary, and never a second tenancy key. Extraction proposes `domain`; a human confirms it, like any other candidate. `valid_until` is set only for documents that expire (insurance certificates, registrations, audits) and drives the assembly gap list.

**document_chunks** — retrieval units.
`id, document_id, seq, text, page_start, page_end, embedding vector(1536), token_count`
Chunking strategy is versioned in `llm/`; changing it triggers `corpus.reindex`.

## C. Facts & provenance (the load-bearing tables)

**facts** — atomic, typed assertions extracted from documents or confirmed by users.
```
id, org_id,
subject_type enum('organization','program','outcome','budget_line','funder','person'),
subject_id uuid,
predicate text,            -- e.g. 'annual_served_count', 'founded_year', 'program_description'
value jsonb,               -- typed value: {kind:'number', n:1200} | {kind:'text', t:'...'} | {kind:'money', amount, currency, fy}
status enum('candidate','verified','rejected','superseded'),
confidence numeric,        -- extractor confidence at creation
verified_by uuid null,     -- user who confirmed, null if auto-promoted
superseded_by uuid null
```

**fact_sources** — provenance. Every fact has ≥1 row here. **A fact with no source row is a bug.**
`id, fact_id, document_id, page int null, char_start int null, char_end int null, quote text`

**fact_conflicts** — two facts, same subject+predicate, incompatible values.
`id, org_id, fact_a, fact_b, status enum('open','resolved'), resolution enum('chose_a','chose_b','merged','both_true') null, resolved_by null`
Rule: conflicts are surfaced to users, never silently resolved. Recency is a *suggestion* in the UI, not an auto-resolution.

## D. Org profile (structured views over facts + first-class entities)

**programs** — `id, org_id, name, status enum('active','ended'), description_fact_id null`
**outcomes** — `id, org_id, program_id, metric_name, period (daterange), value_fact_id`
**budget_lines** — org chart of accounts. `id, org_id, code, name, category enum('personnel','fringe','travel','equipment','supplies','contractual','indirect','other'), parent_id null`

Principle: entities give stable identity; facts give attributed values. UI displays come from verified facts; entity rows never store unattributed numbers.

## E. Funder graph

**funders** — `id, org_id, name, ein null, kind enum('private_foundation','community_foundation','corporate','government','other'), website null, notes`
**funder_contacts** — `id, funder_id, name, email null, role null`
**interactions** — every touchpoint. `id, org_id, funder_id, kind enum('email','call','meeting','submission','award','rejection','report'), occurred_at, summary, document_id null`

## F. Grant lifecycle (workflow layer's tables, listed here because reasoning reads them)

**opportunities** — a fundable opening. `id, org_id, funder_id, title, deadline date null, amount_min, amount_max, status enum('identified','evaluating','pursuing','declined_by_us','applied','awarded','rejected'), fit_assessment jsonb null, effort_estimate_hours int null`
**applications** — `id, opportunity_id, submitted_at null, requested_amount, program_ids uuid[]`
**commitments** — what we promised. Written when an application is marked submitted. `id, application_id, kind enum('outcome','budget','activity','report_due'), description, due date null, fact_id null`
**awards** — `id, application_id, amount, period daterange, restrictions text`
**reports** — `id, award_id, due date, status enum('upcoming','drafting','review','submitted'), document_id null`
**tasks / approvals** — `tasks(id, org_id, title, due, assignee_id, related_type, related_id, status)` · `approvals(id, org_id, subject_type enum('draft_export','outbound_email','report_submission'), subject_id, approved_by, approved_at)` — an outbound action without an approval row must be impossible at the code level, not just the UI level.

### Document assembly (ADR-0008)

**requirements** — a document the funder asks to be attached to an application.
`id, org_id, opportunity_id, label, doc_domain null, doc_kind null, required bool default true, notes text null`
`doc_domain`/`doc_kind` are hints used to suggest candidate documents; they do not constrain what may be attached.

**attachments** — a document satisfying a requirement.
`id, org_id, requirement_id, document_id, attached_by, attached_at`

The gap list is **derived, never stored**: a required requirement with no attachment row is *missing*; an attached document whose `valid_until` has passed or falls inside the org's warning window is *expiring*. This is date arithmetic and null checks — never an LLM inference (constraint 4 in 00-overview). In v1 gaps **flag**; they do not block export. Blocking requires a new ADR.

## G. Artifacts (reasoning outputs)

**drafts** — `id, org_id, opportunity_id, kind enum('narrative','budget','report','loi'), content jsonb (block-structured), status enum('generating','review','accepted','exported'), prompt_version, model`
**draft_claims** — per-claim attribution. `id, draft_id, block_ref, text, fact_ids uuid[]`
Rule (enforced by eval gate AND a DB check in the export path): a draft cannot reach `accepted` while any claim has empty `fact_ids`.

## Open questions (Draft — resolve before Phase 2)

1. Do outcome metrics need standardized taxonomy mapping (e.g., to IRIS+) or free-form v1? Lean: free-form v1.
2. Funder records: global shared table with org-level overlay vs per-org? v1: per-org (simpler, no cross-tenant leakage risk); revisit when we add a funder database.
3. Budget template mapping storage (funder template ↔ budget_lines mapping memory) — schema TBD in Phase 4 planning.
4. Reusable requirement templates: most funders ask for the same handful of attachments. Per-opportunity `requirements` rows only (today) vs an org-level template a new opportunity is seeded from. Lean: add templates once we have seen real funder checklists, not before. (ADR-0008)
5. How long is the expiry warning window, and is it per-org or global? Lean: org-level setting in `organizations.settings`, defaulting to 60 days. (ADR-0008)
