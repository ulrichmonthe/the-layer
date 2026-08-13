# 00 — Architecture Overview

**Status:** Accepted · **Owner:** Founder · **Last updated:** 2026-08-13

## What we are building

An AI knowledge infrastructure ("the brain") for nonprofit grant teams — typically 1–2 people, non-technical, severely time-constrained. The system ingests an organization's existing documents, builds a persistent, provenance-backed knowledge core, and projects that core into every stage of the grant lifecycle: funder discovery, proposal drafting, budget translation, document assembly, pipeline management, reporting, and go/no-go triage.

## Product-defining constraints (these shape every technical choice)

1. **Zero-setup onboarding.** The brain builds itself from uploaded documents. We never ask users to populate forms/databases before delivering value. First session must end with the brain knowing their organization.
2. **Provenance is existential.** Every fact the system asserts carries a pointer to its source (document, page, span). A generated draft may not contain claims that lack provenance. One fabricated statistic sent to a funder destroys the product.
3. **Human approval gates.** Nothing leaves the organization (email, submission, export marked final) without explicit human sign-off. AI drafts; humans decide. Structurally enforced, not just claimed.
4. **Deterministic state under probabilistic AI.** Deadlines, numbers, commitments are stored records, never LLM inferences. The AI reasons over the record; it is never the record.
5. **Compounding memory.** Every submitted application, funder reply, and report flows back into the core. The system gets smarter with use; institutional knowledge survives staff turnover.

## The five layers

```
┌─────────────────────────────────────────────────────┐
│ 5. EXPERIENCE      conversation in, artifacts out,  │
│                    review surfaces, approval gates  │
├─────────────────────────────────────────────────────┤
│ 4. WORKFLOW        pipeline state, deadlines,       │
│                    tasks, proactive nudges (push)   │
├─────────────────────────────────────────────────────┤
│ 3. REASONING       projections of the core:         │
│                    match · draft · translate · triage│
├─────────────────────────────────────────────────────┤
│ 2. KNOWLEDGE CORE  org profile (structured) +       │
│                    document corpus (indexed) +      │
│                    funder graph (relational)        │
├─────────────────────────────────────────────────────┤
│ 1. INGESTION       upload, connectors, extraction,  │
│                    confirmation loop, ongoing capture│
└─────────────────────────────────────────────────────┘
```

### Layer responsibilities and non-responsibilities

**1. Ingestion** — Accepts documents (PDF, DOCX, XLSX) and connector data (email, drive, finance — later phases). Runs extraction (vendor behind interface, see ADR-0003), producing candidate facts with confidence scores. Surfaces low-confidence and conflicting extractions to the user for confirmation. Writes only *candidate* facts; promotion to *verified* happens via user confirmation or high-confidence auto-promotion rules. Does NOT: interpret, draft, or decide.

**2. Knowledge Core** — Three stores, one truth (see 02-data-model.md): structured org profile, indexed document corpus with chunk embeddings, and the funder relationship graph. The corpus holds the organisation's operational documents too — governance, legal, finance, HR, insurance — tagged by `domain` as a facet, not partitioned into departments (ADR-0008). Owns provenance. Owns conflict records (two sources disagree → conflict surfaced, never silently resolved). Does NOT: call LLMs for generation; the core is a system of record.

**3. Reasoning** — Stateless projection engines that read the core and produce artifacts: **Match** (org profile × funder data → fit assessment), **Draft** (question set × corpus → attributed narrative), **Translate** (org budget × funder template → mapped budget with reconciliation), **Triage** (history × pipeline → expected-value recommendation). Every output claim carries fact IDs. Does NOT: write to the core directly; outputs go to review, and only accepted artifacts produce new core records.

**4. Workflow** — Pipeline entities (opportunities, applications, awards, reports), deadlines, tasks, approval gates, document assembly (funder requirement checklists, attachments, and a derived gap list of what is missing or expiring — ADR-0008), and the nudge engine (durable scheduled jobs → email). Push, not pull: the system initiates. Does NOT: auto-send anything externally without an approval record; does NOT infer assembly gaps with a model — they are computed.

**5. Experience** — Next.js app. Conversational command surface for input/orchestration; document-style artifact surfaces for review (drafts with claim-level provenance links, budget side-by-sides). Renders approval gates. Does NOT: contain business logic; it calls layer APIs.

## Trust boundaries

- LLM calls go through the gateway (ADR-0005) — never direct from the app.
- User documents and extracted facts are tenant-isolated at the database level (org_id on every row; RLS on).
- No user content in logs. Trace payloads to the eval/observability platform are scrubbed of PII per the scrubbing spec (to be written before Phase 1 exit).

## What is explicitly out of scope for v1

- Funder portal auto-submission (no applicant-side APIs exist; we optimize copy-ready output instead)
- Multi-org consultancies (one org per workspace in v1)
- Non-English documents
- Fine-tuned models (retrieval + prompting only until evals prove a gap)
