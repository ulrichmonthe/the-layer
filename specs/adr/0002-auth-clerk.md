# ADR-0002 — Clerk for authentication

**Status:** Accepted · **Date:** 2026-08-13 · **Deciders:** Founder

## Context
Non-technical users; we need email/password + Google login, org/workspace support, and eventually SSO for larger customers. Zero appetite for building auth.

## Decision
Clerk, using Clerk Organizations mapped 1:1 to our `organizations` table. Roles (`admin`/`editor`/`reviewer`) live in OUR database, not Clerk metadata — authorization is ours, authentication is Clerk's.

## Alternatives considered
- **Auth0** — heavier, pricier at seed stage. — **Roll our own** — no. — **WorkOS AuthKit** — revisit when SSO deals appear; WorkOS can sit alongside Clerk later.

## Consequences
Easier: session handling, org invitations, MFA for free. Harder: SSO/SAML later may need WorkOS addition (acceptable, known path). Committing to: webhook sync Clerk→our users table as the only way user rows are created.

## Devin-facing rules
- Never trust client-provided org_id; derive from the Clerk session server-side in one middleware, everywhere.
- Authorization checks call our `can(user, action, resource)` helper — never inline role string comparisons.
- No Clerk SDK imports outside `packages/api/auth/`.
