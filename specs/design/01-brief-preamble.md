# 01 — Design Brief Preamble

**Status:** Draft · **Last updated:** 2026-08-14

Paste this at the top of every Claude Design session, followed by one surface brief from `02-briefs-phase-1.md`. It exists because the first design round contradicted the accepted specs in two places — a design that disagrees with the data model costs a reconciliation pass before any of it can be built.

---

## The product

GrantBrain is AI knowledge infrastructure for nonprofit grant teams. The user is one or two non-technical, severely time-constrained people who are the whole grant function of a small nonprofit. They are not power users and will not be trained. Assume they are doing this at 9pm after their real job.

The system ingests their documents, builds a provenance-backed knowledge core, and projects it into the grant lifecycle. Trust is the product: one fabricated statistic sent to a funder ends the relationship.

## Design system

Use the **Broadsheet** design system. Take every colour, font, spacing, radius and shadow from its tokens (`var(--color-*)`, `var(--font-*)`, `var(--space-*)`, `var(--radius-*)`, `var(--shadow-*)`). Never hard-code a hex, font name, or px value the tokens already carry. Left-aligned asymmetric layouts, hierarchy from the serif scale and whitespace rather than boxes and dividers.

## Five constraints that outrank convenience

1. **Zero-setup onboarding.** The brain builds itself from uploaded documents. Never design a form the user must fill before getting value.
2. **Provenance is existential.** Every asserted fact carries a pointer to its source. Every claim rendered on screen needs a provenance affordance that resolves to the source quote, document and page.
3. **Human approval gates.** Nothing leaves the organisation without explicit sign-off. Design the gate as a real ceremony, not a toast.
4. **Deterministic state under probabilistic AI.** Deadlines, amounts, statuses and commitments are stored records. Never design a surface implying the AI computed a number it should have read.
5. **Compounding memory.** Every submission and reply flows back in.

## Rules that constrain the interface

- **Use the exact enum values from `02-data-model.md`.** Do not invent statuses. `documents.status` is `uploaded | extracting | extracted | failed`. `facts.status` is `candidate | verified | rejected | superseded`. `drafts.status` is `generating | review | accepted | exported`. If a state you need does not exist, say so in a visible TODO rather than inventing one.
- **`candidate` and `verified` are different and the difference is load-bearing.** ADR-0003 exists to stop unconfirmed extractions being treated as truth. Do not design affordances that blur the distinction or make blanket promotion effortless without deliberate friction.
- **A fact with no source cannot exist.** Never mock a value with no document behind it.
- **Conflicts surface, never auto-resolve.** Recency may be a visual suggestion; it is never an automatic choice.
- **Typed errors get real states**: `PROVENANCE_VIOLATION`, `APPROVAL_REQUIRED`, `CONFLICT`, `VALIDATION`, `NOT_FOUND`.
- **Long-running work is a job, not a spinner.** Extraction and drafting run in a worker and can take minutes. Design for leaving and coming back.

## Two conflicts that are open — do not silently resolve them

The first design round decided both of these without flagging it. If a surface touches them, show the option and mark it **OPEN**.

**A. What the export gate tests.** ADR-0004 and `02-data-model` say a draft cannot reach `accepted` "while any claim has empty `fact_ids`" — the test is *presence of a source*. The first design instead blocked on `status !== 'verified'`, which is stricter: a claim can have a quote, document and page while its fact is still `candidate`. Both are defensible. Neither is decided.

**B. Approval routing.** The first design offered "Send to Dana for review" and an approval that goes onward for signature. The `approvals` table has no requester, assignee, status or chain, and `drafts.status` has no state for it. Design it if the surface needs it, but mark it OPEN — it requires a schema change that has not been agreed.

## When something is not specified

Leave a visible TODO naming what is missing. Do not invent a mechanism. An unanswered question in a design is cheap; an invented mechanism that reaches the data model is not.
