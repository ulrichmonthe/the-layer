# ADR-0001 — Postgres + pgvector as the only datastore (v1)

**Status:** Accepted · **Date:** 2026-08-13 · **Deciders:** Founder

## Context
The knowledge core needs relational integrity (facts ↔ sources ↔ conflicts), a vector index for retrieval, and strict tenant isolation. Team is one human + agents; every additional datastore multiplies operational surface and agent confusion.

## Decision
One Postgres database (Neon) for everything, pgvector for embeddings, Drizzle ORM for schema + migrations. Row-Level Security on with `org_id` policies on every tenant table.

## Alternatives considered
- **Dedicated vector DB (Pinecone/Turbopuffer)** — better at 10M+ vectors; we'll have thousands per org. Premature; adds a consistency boundary between facts and embeddings.
- **Supabase** — viable; Neon chosen for branch-per-preview-deploy workflow which our gate stack depends on.

## Consequences
Easier: transactions across facts/chunks/provenance; one backup story; preview-branch databases per PR. Harder: if retrieval scale explodes, we migrate vectors out (clean seam: embeddings only touched via `core` repositories). Committing to: RLS discipline, migration review on every PR.

## Devin-facing rules
- All schema changes via Drizzle migrations; never raw SQL DDL in app code.
- Every new tenant table gets `org_id` + RLS policy in the same migration. A table without RLS fails review.
- No new datastores, caches, or queues. Redis et al. require a new ADR.
