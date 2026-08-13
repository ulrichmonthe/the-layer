# GrantBrain Specs Repo

This repository is the **single source of truth for what we are building and why**. No code lives here. Every architectural decision, data model, and contract that the coding agent (Devin) must respect is written down here first.

## Why this repo exists

We are building with an autonomous coding agent. Agents implement well; they must never decide. This repo is where decisions live. If it isn't written here, it isn't decided — and the agent must ask, not assume.

## Structure

```
specs/
├── README.md                        ← you are here
├── architecture/
│   ├── 00-overview.md               ← system context, the five layers, design principles
│   ├── 01-services.md               ← service decomposition per layer
│   ├── 02-data-model.md             ← knowledge core schema (the heart of the product)
│   └── 03-api-contracts.md          ← contracts between layers/services
├── adr/
│   ├── 0000-template.md             ← copy this for every new decision
│   ├── 0001-postgres-pgvector.md
│   ├── 0002-auth-clerk.md
│   ├── 0003-extraction-behind-interface.md
│   ├── 0004-provenance-model.md
│   ├── 0005-llm-gateway.md
│   ├── 0006-typescript-monorepo.md
│   ├── 0007-background-jobs-inngest.md
│   └── 0008-document-domains-and-assembly.md
└── agents/
    ├── AGENTS.md                    ← copy to the ROOT of the code repo
    └── architect-instructions.md    ← paste into the Claude "Architect" Project
```

## Operating workflow

1. **Set up the Architect.** Create a Claude Project named "GrantBrain Architect". Paste `agents/architect-instructions.md` into its custom instructions. Upload every file in this repo as project knowledge. Re-upload whenever a doc changes (or connect the repo via an integration).
2. **Before each build phase:** consult the Architect with the phase goal. It reviews against these docs, flags gaps, and drafts any new ADRs needed. You approve; you commit.
3. **New irreversible choice → new ADR.** Copy `adr/0000-template.md`. One page max. An ADR is required for anything expensive to reverse: datastore, vendor, auth, core schema shape, protocol.
4. **Changing a decision:** never edit an accepted ADR's decision. Write a new ADR that supersedes it and link both ways.
5. **Weekly drift review:** paste the week's merged-PR summary into the Architect and ask: "Does anything here violate or silently extend the architecture?" File fix-up tickets for drift.
6. **AGENTS.md is a copy, not a link.** The canonical version lives here; a copy sits at the root of the code repo. When you change it here, copy it over in the same sitting.

## Status conventions

Every architecture doc carries a status header: `Draft` → `Accepted` → `Superseded`. Devin may only rely on `Accepted` documents. Anything `Draft` is not yet a decision.

## The one rule

**If the coding agent needs to make a choice that this repo doesn't answer, the correct output is a question, not code.** This rule is restated in AGENTS.md and in every ticket's boilerplate.
