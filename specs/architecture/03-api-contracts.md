# 03 — API Contracts Between Layers

**Status:** Accepted (v1 surface) · **Last updated:** 2026-08-13

Internal API is tRPC; all input/output shapes are zod schemas exported from `packages/shared/contracts/`. This document is the human-readable index; the zod files are the machine truth. **Devin: if this doc and the zod schemas disagree, stop and flag — do not pick one.**

Conventions: all procedures scoped by authenticated org; errors are typed (`NOT_FOUND`, `CONFLICT`, `VALIDATION`, `PROVENANCE_VIOLATION`, `APPROVAL_REQUIRED`); long-running work returns a `jobId` and progress streams over SSE.

## ingestion.*

- `ingestion.createUpload(filename, mime, kind?) → { uploadUrl, documentId }` — presigned URL flow.
- `ingestion.getDocument(documentId) → Document`
- `ingestion.listConfirmations() → ConfirmationItem[]` — candidate facts needing review, grouped by subject; includes conflicts.
  - `ConfirmationItem = { factId, subject, predicate, value, confidence, sources: SourceRef[], conflictWith?: factId }`
- `ingestion.resolveConfirmation(factId, action: 'verify'|'reject'|'edit', editedValue?) → Fact`
- `ingestion.resolveConflict(conflictId, resolution) → void`

## core.*

- `core.getOrgProfile() → { organization, programs[], outcomes[], budgetLines[] }` — verified facts only.
- `core.getFact(factId) → Fact & { sources: SourceRef[] }` — powers provenance popovers.
- `core.search(query, filters?) → { chunks: ChunkHit[], facts: FactHit[] }` — hybrid retrieval (vector + keyword + fact lookup). Used by reasoning AND by the UI's "ask the brain" box. Same ranking code path for both — no divergence.
- `core.funders.list/get/upsert`, `core.interactions.log(...)` — funder graph CRUD.

## reasoning.* (all stateless; all return artifacts referencing fact IDs)

- `reasoning.assessFit(funderId | funderProfileText) → FitAssessment`
  - `FitAssessment = { score: 0-100, rationale: AttributedText[], risks: string[], comparables?: string[] }`
- `reasoning.draft.start({ opportunityId, questions: QuestionSpec[], kind }) → { jobId, draftId }`
  - `QuestionSpec = { id, prompt, wordLimit?, charLimit?, notes? }`
  - Job emits progress; result is a `drafts` row with `draft_claims`.
- `reasoning.draft.regenerateBlock(draftId, blockRef, guidance?) → Block`
- `reasoning.translateBudget({ applicationId, template: FunderBudgetTemplate }) → BudgetMapping`
  - `BudgetMapping = { rows: { templateCell, budgetLineIds[], amount, formulaNote }[], unmapped: [], reconciliation: { sourceTotal, mappedTotal, delta } }`
  - Contract: `delta must equal 0` for the mapping to be acceptable; non-zero delta returns the mapping flagged `unreconciled`, and export is blocked.
- `reasoning.triage(opportunityId) → { effortEstimateHours, winProbability, evNote, dataQuality: 'thin'|'ok'|'rich' }` — Phase 5; returns `dataQuality:'thin'` honestly rather than fabricating confidence.

## workflow.*

- `workflow.pipeline.list(filters) → Opportunity[]` · `workflow.pipeline.move(opportunityId, status)`
- `workflow.tasks.*` — CRUD + assign.
- `workflow.approvals.request(subjectType, subjectId) → Approval(pending)` · `workflow.approvals.grant(approvalId)`
  - Export/send procedures REQUIRE a granted approval id; they fail with `APPROVAL_REQUIRED` otherwise. This is checked server-side.
- `workflow.nudges.preview() → Nudge[]` — what tomorrow's scan would send; used for the settings/trust UI.

## experience-layer obligations (not an API, but contractual)

- Every rendered claim from a draft shows its provenance affordance; clicking resolves via `core.getFact`.
- Accepting a draft (`status: review → accepted`) triggers the claim-attribution check server-side; UI must handle `PROVENANCE_VIOLATION` by highlighting the offending blocks.

## Versioning

Breaking a schema in `shared/contracts` requires: (a) an ADR if it changes a core concept, or (b) a ticket explicitly titled `contract-change:` otherwise. Additive changes are fine in normal tickets.
