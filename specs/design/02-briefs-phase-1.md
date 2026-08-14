# 02 — Phase 1 Design Briefs

**Status:** Draft · **Last updated:** 2026-08-14

Eight surfaces. Use one brief per Claude Design session, always after `01-brief-preamble.md`. Briefs 1–4 are the onboarding spine and are worth designing as one continuous flow.

---

## 1 · First run — the empty workspace

**The moment.** A grant manager has just signed up. They have no idea what this is yet and are one click from closing the tab. Constraint 1 says the brain builds itself from documents — so the entire screen is one invitation to drop files, with enough context that dropping feels safe rather than reckless.

**On screen.** Organisation name (from Clerk). A drop target that dominates. Plain language about what happens to the files and what comes back. Suggestions of what to drop, phrased in their vocabulary — last year's proposal, your 990, the audited financials, the board roster, your insurance certificate. Accepted formats are PDF, DOCX, XLSX.

**Actions.** Drop or browse → `ingestion.createUpload(filename, mime, kind?)`. Multi-file.

**Must not.** No setup wizard, no profile form, no "tell us about your organisation" step. If this screen asks the user to type facts about their org, it has failed the product's first constraint.

---

## 2 · Upload and extraction progress

**The moment.** Twelve files are uploading. Extraction runs in a worker (ADR-0003) and takes minutes. The user will leave and come back.

**On screen.** Per-document rows carrying real `documents.status` values: `uploaded → extracting → extracted → failed`. Title, kind, size. An aggregate sense of how far along the whole batch is. Files that failed need a reason a non-technical person can act on and a retry.

**States.** All queued · partially extracted · all done · some failed · empty. Design the leave-and-return case explicitly: this page is not a modal and progress is not lost.

**Contract gap — mark TODO.** There is no procedure in `03-api-contracts.md` that lists documents by status or streams extraction progress. Design what the surface needs and flag it.

**Must not.** No blocking spinner. Nothing that implies the user must sit and wait.

---

## 3 · The brain panel

**The moment.** This is the payoff for constraint 1 — the first session ends with the brain knowing the organisation. This panel is the proof. It has to feel like something was built *for them* out of *their* documents, not like a dashboard.

**On screen.** Documents ingested. Verified facts. Open conflicts, which are the one number that is a call to action rather than a statistic. Consider a glimpse of what it now knows — a few recently verified facts in plain sentences, each with its provenance affordance.

**Contract gap — mark TODO.** No procedure exists. It needs roughly `core.stats() → { documentCount, verifiedFactCount, openConflictCount }`.

**Must not.** Do not present candidate facts as things the brain knows. Only `verified` facts count here.

---

## 4 · Confirmation queue

**The moment.** Extraction produced candidate facts with confidence scores. This is where the brain becomes trustworthy, and it is the highest-stakes screen in Phase 1. Get it wrong and users either rubber-stamp everything or abandon it as tedious.

**On screen.** `ingestion.listConfirmations() → ConfirmationItem[]`, grouped by subject. Each item: the subject, the predicate in plain language, the extracted value, extractor confidence, and the source spans — quote, document, page. Items that conflict with an existing fact are marked and route to brief 5.

**Actions.** Per item: verify, reject, or edit then verify — `ingestion.resolveConfirmation(factId, action, editedValue?)`.

**The open question this surface owns.** Should bulk confirmation exist at all? The first design shipped a one-click "Verify remaining 2". ADR-0003 built the candidate/verified distinction precisely to stop unexamined promotion, and high-confidence items already auto-promote via `promotion-rules.ts` before reaching this queue — so everything the user sees here is, by definition, something the system was unsure about. Design the queue so that confirming each item is *fast* rather than *bulk*. If you propose bulk, show what the user gives up.

**Must not.** No default-verified state. No affordance that makes accepting everything easier than reading it.

---

## 5 · Conflict resolution

**The moment.** Two documents disagree. The 2024–25 program report says 88% tutor retention; the June board packet says 84%. Both are true of different denominators. This is a judgment only the user can make, and the interface's job is to make the judgment easy and the stakes legible.

**On screen.** The two facts side by side with equal weight, each with its full source quote, document and page. Where they came from and when. A plain-language read of why they might differ.

**Actions.** `ingestion.resolveConflict(conflictId, resolution)` where resolution is `chose_a | chose_b | merged | both_true`. All four need a home — `both_true` in particular, since it is the honest answer surprisingly often.

**Must not.** Do not pre-select the more recent one. Recency may be labelled; it is never the default choice. Do not bury `both_true`.

---

## 6 · Ask the brain

**The moment.** "What did we say about tutor retention last year?" This is the same `core.search` path reasoning uses — hybrid retrieval over chunks and facts.

**On screen.** A question input that does not look like a chatbot toy. Answers composed of retrieved chunks and facts, every assertion carrying its provenance affordance. Show what was searched.

**States.** Nothing found — say so plainly rather than generating something. Thin results — say the corpus may not cover it yet.

**Must not.** Never render an answer without sources. "I don't know" is a correct and valuable answer here.

---

## 7 · Document shelf

**The moment.** "Where's our current insurance certificate?" — ADR-0008. Also the surface where a user notices something expired before a funder does.

**On screen.** Documents filterable by `domain` (`governance · legal · finance · hr · insurance · program · grant`) and `kind`. Title, domain, kind, fiscal year, upload date, and `valid_until` where set. Expired and expiring-soon documents are visually distinct — this is the whole reason validity dates exist.

**Actions.** `core.documents.list(filters)`. Filter, open, re-upload a replacement for an expiring document.

**Design note.** `domain` is a facet, not a folder tree and not a permission boundary. Do not design departmental sections, per-department access, or anything implying legal documents are walled off from ops.

---

## 8 · Fact detail and provenance affordance

**The moment.** The user clicks any asserted value anywhere in the product and asks "says who?" This must feel instant and identical everywhere — the same component in a draft, a search answer, and the brain panel.

**On screen.** `core.getFact(factId) → Fact & { sources: SourceRef[] }`. The claim, the source quote, document, page or cell reference. Fact status. Extractor confidence where relevant. Where else this fact is used.

**States.** `verified` · `candidate` (not yet confirmed — offer confirmation inline) · `conflicted` (route to brief 5) · `superseded` (show what replaced it).

**Already designed** in `GrantBrain Review Flow.dc.html` as the source panel. Reuse it rather than redesigning; extend it to cover `superseded`, which the existing design does not handle.

---

## After these eight

Surfaces 9–29 are in `00-surface-inventory.md`. Do not design them yet — the Phase 1 build will teach you things about the confirmation loop that should inform every later surface.
