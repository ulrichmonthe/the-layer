# 00 — Surface Inventory

**Status:** Draft · **Last updated:** 2026-08-14

Every surface the end-to-end experience needs, what governs it, and whether it has been designed. This is a map, not a decision — the phase column follows the roadmap in `specs/diagrams/roadmap.excalidraw`, which is itself unapproved.

Design status: **done** · **partial** — exists but incomplete · **none**.

## Phase 1 — Foundation

The phase you build first, and the phase with almost no design.

| # | Surface | Governing spec | Contract | Design |
|---|---|---|---|---|
| 1 | First run / empty workspace | 00 constraint 1 (zero setup) | — | none |
| 2 | Upload + extraction progress | 01 job inventory, ADR-0003 | `ingestion.createUpload`, `documents.status` | none |
| 3 | The brain panel | 00 constraint 1 | **no contract — gap** | partial (nav in review flow) |
| 4 | Confirmation queue | ADR-0003, 02 §C | `ingestion.listConfirmations`, `resolveConfirmation` | none |
| 5 | Conflict resolution | 02 §C `fact_conflicts` | `ingestion.resolveConflict` | partial (panel only, no queue) |
| 6 | Ask the brain | 03 `core.search` | `core.search` | none |
| 7 | Document shelf | ADR-0008 | `core.documents.list` | none |
| 8 | Fact detail / provenance | ADR-0004 | `core.getFact` | done (source panel) |

## Phase 2 — Funder fit

| # | Surface | Governing spec | Contract | Design |
|---|---|---|---|---|
| 9 | Funder list | 02 §E | `core.funders.list` | none |
| 10 | Funder detail + interaction timeline | 02 §E | `core.funders.get`, `core.interactions.log` | none |
| 11 | Opportunity intake | 02 §F | `workflow.pipeline.*` | none |
| 12 | Fit assessment | 03 `reasoning.assessFit` | `FitAssessment` | none |
| 13 | Pipeline board | 02 §F status enum | `workflow.pipeline.list/move` | none |

## Phase 3 — Drafting

| # | Surface | Governing spec | Contract | Design |
|---|---|---|---|---|
| 14 | Question set intake | 03 `QuestionSpec` | `reasoning.draft.start` | none |
| 15 | Draft generation progress | 01 `draft.requested`, SSE | job progress | none |
| 16 | Draft review | ADR-0004 | `drafts`, `draft_claims` | done |
| 17 | Block regenerate | 03 | `reasoning.draft.regenerateBlock` | none |
| 18 | Approval ceremony | 02 §F `approvals` | `workflow.approvals.*` | done |

## Phase 4 — Assemble + export

| # | Surface | Governing spec | Contract | Design |
|---|---|---|---|---|
| 19 | Budget translate side-by-side | 03 `BudgetMapping` | `reasoning.translateBudget` | none |
| 20 | Requirements checklist + gap list | ADR-0008 | `workflow.assembly.gaps` | none |
| 21 | Attach document to requirement | ADR-0008 | `workflow.assembly.attach/suggest` | none |
| 22 | Export package | 00 constraint 3 | approval-gated export | none |

## Phase 5 — Pipeline + triage

| # | Surface | Governing spec | Contract | Design |
|---|---|---|---|---|
| 23 | Nudge preview / trust settings | 03 `workflow.nudges.preview` | same | none |
| 24 | Task list | 02 §F `tasks` | `workflow.tasks.*` | none |
| 25 | Reports | 02 §F `reports` status enum | — | none |
| 26 | Triage recommendation | 03 `reasoning.triage` | same | none |

## Cross-cutting

| # | Surface | Governing spec | Design |
|---|---|---|---|
| 27 | Sign-in / org creation | ADR-0002 | none |
| 28 | Settings — roles, expiry window | ADR-0002, ADR-0008 open Q5 | none |
| 29 | Typed error states | 03 conventions | none |

Error states worth designing once and reusing: `PROVENANCE_VIOLATION` (highlight offending blocks), `APPROVAL_REQUIRED`, `CONFLICT`, `VALIDATION`, `NOT_FOUND`.

## Contract gaps this inventory exposes

1. **The brain panel has no procedure.** Surface 3 needs `core.stats() → { documentCount, verifiedFactCount, openConflictCount }`. Not in `03-api-contracts.md`. It is the Phase 1 payoff moment, so it cannot stay unspecified.
2. **Extraction progress has no read path.** `documents.status` exists, but nothing in `03` lists documents by status or streams progress for surface 2.
3. **Approval routing** — see `01-brief-preamble.md`, open conflict B.

## Sequencing note

Design Phase 1 (surfaces 1–8) before anything else. That is what gets built first, and AGENTS.md requires a written design spec in the ticket before implementation. Surfaces 9–29 can wait; designing them now risks locking in decisions the Phase 1 build will inform.
