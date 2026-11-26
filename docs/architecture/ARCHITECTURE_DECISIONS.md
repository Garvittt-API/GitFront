# Architecture Decisions (ADRs)

**Status:** Proposed architecture decisions for review. Each decision records context, decision, consequences, and alternatives. Decisions marked **REQUIRES APPROVAL** must be confirmed by the product owner before implementation.

**Last updated:** 2026-11-26

---

## Decision index

| ID | Area | Decision | Status |
|----|------|----------|--------|
| ADR-001 | Application framework | Next.js App Router | Proposed |
| ADR-002 | Deployment model | Modular monolith | Proposed |
| ADR-003 | Server/client boundaries | Server Components + thin client islands | Proposed |
| ADR-004 | API architecture | Next.js Route Handlers + Zod, typed client | Proposed |
| ADR-005 | ORM | Drizzle ORM | Proposed |
| ADR-006 | Database | PostgreSQL | Proposed |
| ADR-007 | Data modeling | Relational core + JSONB for the document | Proposed |
| ADR-008 | Migrations | Drizzle migrations, forward-only, baselined | Proposed |
| ADR-009 | Authentication | GitHub OAuth (Auth.js) + DB sessions | Proposed |
| ADR-010 | Session security | Opaque httpOnly session cookies | Proposed |
| ADR-011 | GitHub publish permission | GitHub App installation (not OAuth `repo`) | Proposed |
| ADR-012 | GitHub data access | Server-side only, minimal scopes | Proposed |
| ADR-013 | Rendering | Deterministic server-side renderers | Proposed |
| ADR-014 | Template system | Registry + versioned compatibility | Proposed |
| ADR-015 | Versioning | Append-only revisions + immutable releases | Proposed |
| ADR-016 | Concurrency | Optimistic concurrency (revision number / ETag) | Proposed |
| ADR-017 | Hosting | Vercel (app) + managed PostgreSQL | Proposed, pricing NEEDS VERIFICATION |
| ADR-018 | Background jobs | Postgres-backed job queue + worker | Proposed |
| ADR-019 | Caching | Redis (optional) / in-memory + DB cache | Proposed |
| ADR-020 | Object storage | S3-compatible (per deployment) | Proposed, provider NEEDS VERIFICATION |
| ADR-021 | Email | Resend or SES | Proposed, NEEDS VERIFICATION |
| ADR-022 | Monitoring/errors | Sentry + structured logs | Proposed |
| ADR-023 | CI/CD | GitHub Actions | Proposed |
| ADR-024 | Public portfolio delivery | App Router public routes (or static export) | Proposed |
| ADR-025 | Billing | None in MVP | Proposed (no tables) |

---

## ADR-001 — Next.js App Router

**Context.** Two studios with shared data and independent presentation; live preview; server-side GitHub calls; future custom domains and public portfolio delivery. Need SSR/SSG for portfolio pages and a single codebase.

**Decision.** Use **Next.js (App Router)** as the application framework.

**Consequences (+):**
- Colocation of UI, server actions/route handlers, and rendering in one codebase.
- App Router gives Server Components, streaming, and route-level public/private separation needed for the public portfolio.
- Strong ecosystem and long-term maintenance.
- Deployable to multiple hosts (Vercel, self-hosted).

**Consequences (−):**
- App Router conventions and caching semantics require discipline.
- Heavier framework than a minimal renderer; acceptable given the two-studio product.

**Alternatives considered.**
- *Vite + React SPA + separate API* — rejected: loses SSR/SSG for public portfolio, adds a second deployable, more infra.
- *Remix/React Router* — viable, but Next.js ecosystem fit (auth libs, hosting) is stronger here.

**Status:** Proposed.

---

## ADR-002 — Modular monolith

**Context.** Two studios share one canonical profile; shared logic must not duplicate; initial affordability is required.

**Decision.** Build a **modular monolith**: one deployment, clear internal module boundaries, no network calls between modules. Modules communicate via typed in-process interfaces, never via direct cross-module DB access outside their own repositories.

**Consequences (+):** Simple deploy, atomic transactions across profile + publishing, low latency, low cost, easy to test end-to-end.

**Consequences (−):** Modules must be kept clean; risk of boundary erosion if discipline is lost.

**Alternatives considered.**
- *Microservices* — rejected: premature; no scaling or organizational driver; adds infra and network failure modes.
- *Multiple deployables (web + api)* — rejected: duplicates deployment/CI, adds complexity without benefit.

**Status:** Proposed.

---

## ADR-003 — Server/client component boundaries

**Decision.** Default to **Server Components**. Use `"use client"` only for interactive islands (editors, preview controls, drag-and-drop section ordering, live preview toggles). All data fetching and secret handling stay server-side.

**Rationale.** Keeps secrets out of the bundle, minimizes client JS, and centralizes authorization in server code. Interactive editors are the only necessary client islands.

**Status:** Proposed.

---

## ADR-004 — API architecture

**Decision.** **Next.js Route Handlers** under `app/api/**`, each validated with **Zod**, returning a consistent JSON envelope. A thin typed client (generated from Zod schemas) is used by client components.

**Rationale.** Route Handlers avoid a separate API service and deployable. Zod gives one schema source used for both validation and typing. Consistent envelope (see `API_ARCHITECTURE.md`) keeps error handling uniform.

**Alternatives considered.**
- *tRPC* — rejected for MVP: strong but adds a dependency and deviates from plain HTTP for the public/portfolio surface.
- *GraphQL* — rejected: no query flexibility need; more surface to secure.

**Status:** Proposed.

---

## ADR-005 — Drizzle ORM

**Decision.** **Drizzle ORM** over PostgreSQL.

**Rationale.** Lightweight, explicit SQL, first-class Postgres types, migration tooling, no heavyweight runtime, and good control over indexes/constraints. Fits "affordable and simple."

**Alternatives considered.**
- *Prisma* — viable; heavier runtime/client and generated client in production. Drizzle chosen for lower overhead and SQL transparency.
- *Raw SQL* — rejected for MVP: too repetitive, no shared schema typing.

**Status:** Proposed.

---

## ADR-006 — PostgreSQL

**Decision.** **PostgreSQL** as the single system of record.

**Rationale.** Relational integrity, JSONB (for the document + snapshots), transactional job queue, mature tooling, and free-tier availability across managed providers.

**Alternatives considered.**
- *SQLite* — rejected: no concurrent multi-writer production semantics; not suitable for a multi-user SaaS.
- *MySQL* — viable but JSON/tooling/ergonomics favor Postgres here.

**Hosting note.** Managed Postgres (e.g., Neon, Supabase, RDS). **Provider pricing, free-tier limits, and India-region availability: NEEDS VERIFICATION.**

**Status:** Proposed.

---

## ADR-007 — Relational core + JSONB document

**Decision.** Relational tables for **identity, ownership, versions, publications, jobs, GitHub metadata**; store the canonical `ProfileDocument` as **JSONB** within a versioned revision row, with the schema enforced by Zod at the application boundary and a `schema_version` column.

**Rationale.** The document is a structured, frequently-edited, schema-evolving payload best stored as JSONB (avoids dozens of nullable columns and makes revision storage trivial). Relational tables preserve referential integrity, ownership, and queryable metadata. This is a deliberate boundary: JSONB content is validated by Zod, not trusted raw.

**Alternatives considered.**
- *Fully normalized (one column per field)* — rejected: schema churn, complex revisions, poor fit for custom sections.
- *Fully document store* — rejected: we still need relational ownership/versions/publications.

**Status:** Proposed.

---

## ADR-008 — Migration strategy

**Decision.** **Drizzle migrations**, forward-only, applied in CI and gated in deployment. A baseline migration for initial schema. Explicit, reviewed migration PRs for every schema change. No destructive migration without a documented rollback plan.

**Rationale.** Predictable, reviewable, reproducible. Forward-only avoids brittle down-migrations in a young product.

**Status:** Proposed.

---

## ADR-009 — Authentication: GitHub OAuth + DB sessions

**Decision.** Primary login via **GitHub OAuth** using **Auth.js (NextAuth v5)** with a **Drizzle adapter**, storing accounts/sessions in PostgreSQL. Sessions are **database-backed** (not stateless JWT) for revocability.

**Rationale.** GitHub is the target user's identity source; DB sessions allow logout/revocation and reduce token theft impact. Auth.js handles the OAuth flow and CSRF protections.

**Critical distinction.** GitHub **sign-in** (identity) is separate from permission to **modify repositories** (publishing). Sign-in grants no repo access. Publishing requires a separate, explicit GitHub App installation grant.

**Future providers.** Email/magic-link and Google can be added as additional `accounts` rows behind the same session service.

**Alternatives considered.**
- *Bespoke OAuth + session service* — rejected: re-implements well-known hardened flows.
- *Stateless JWT sessions* — rejected: cannot revoke; worse for account-takeover mitigation.

**Status:** Proposed. **REQUIRES APPROVAL** on whether Auth.js is the chosen auth library vs. a bespoke session service.

---

## ADR-010 — Session security

**Decision.** **Opaque session tokens** (random, high-entropy) stored hashed in `sessions`; delivered via `httpOnly`, `Secure`, `SameSite=Lax` cookies; short access lifetime with refresh rotation; revocable on logout/password change.

**Rationale.** Opaque server-side tokens are revocable and do not leak payload. `httpOnly` prevents theft via XSS. `SameSite` mitigates CSRF.

**Status:** Proposed.

---

## ADR-011 — GitHub publish permission via GitHub App

**Decision.** README publishing uses a **GitHub App installation** (user-installed, per owner/repo), NOT the sign-in OAuth token with `repo` scope. The App generates short-lived **installation tokens** server-side for PR creation.

**Rationale.** Least privilege: an App installation grants scoped, revocable, per-installation access without long-lived personal tokens. No personal access tokens are stored. Installation tokens are minted on demand and never persisted or sent to the browser.

**Alternatives considered.**
- *OAuth with `repo` scope* — rejected: broad, long-lived, and conflates sign-in with repo modification.
- *Personal Access Tokens* — rejected: high-risk, broad, and hard to revoke per repo.

**Status:** Proposed. **REQUIRES APPROVAL** (GitHub App creation is an external setup action; also depends on GitHub App billing/limits — NEEDS VERIFICATION).

---

## ADR-012 — GitHub data access (server-side only, minimal scopes)

**Decision.** All GitHub API access happens **server-side**. Sign-in OAuth uses minimal scopes (`read:user`, `user:email`). Public profile lookup and repo discovery use the **unauthenticated/public** API or a minimal token; rate limits handled via caching + ETag + backoff.

**Rationale.** Minimizes the blast radius of any token. Public data needs no repo permissions.

**Status:** Proposed.

---

## ADR-013 — Deterministic server-side renderers

**Decision.** Both renderers run **server-side** and produce **deterministic, testable** output from a `ProfileDocument` + template. Markdown renderer produces a string; portfolio renderer produces React/HTML. No client-side randomness or uncontrolled external data in output.

**Rationale.** Determinism enables golden-file tests and safe publishing. Server-side keeps templates/secrets off the client.

**Status:** Proposed.

---

## ADR-014 — Template registry + versioned compatibility

**Decision.** A **template registry** with each template carrying a `schema_version` and a declared set of required/optional document features. Compatibility is checked at render time; incompatible templates are rejected with a clear error.

**Rationale.** Supports multiple templates per studio while keeping renderers stable as the document schema evolves.

**Status:** Proposed.

---

## ADR-015 — Append-only revisions + immutable releases

**Decision.** Every save (autosave, manual, import, export, publish, rollback) creates an **immutable `profile_revisions` row**. Published artifacts (README publication, portfolio release) are **immutable releases** referencing a specific revision. Drafts are mutable only in the sense that new revisions are appended; existing revisions never change.

**Rationale.** Gives full history, safe rollback, and clean draft/published separation.

**Status:** Proposed.

---

## ADR-016 — Optimistic concurrency

**Decision.** **Optimistic concurrency control** using a monotonic `revision_number` per profile (and HTTP `ETag`/`If-Match` on updates). A write carrying a stale revision is rejected with a conflict error; the client re-reads and retries.

**Rationale.** Prevents stale autosaves from overwriting newer revisions without heavy locking.

**Alternatives considered.**
- *Pessimistic locking* — rejected: unnecessary contention for an editor-per-user model.
- *Last-write-wins* — rejected: violates the "stale autosave must not overwrite newer revision" requirement.

**Status:** Proposed.

---

## ADR-017 — Hosting

**Decision.** **Vercel** for the Next.js application; **managed PostgreSQL** for the database.

**Rationale.** Minimal ops for a small team; Next.js-native. Keeps initial cost low.

**NEEDS VERIFICATION:** Vercel pricing/free-tier, managed Postgres provider pricing/free-tier, and **India-region availability and data-residency** for both. If India residency is required, evaluate regional providers.

**Alternatives considered.** Self-hosted Node + container on a VPS — viable, more ops; deferred unless cost/residency requires it.

**Status:** Proposed, **NEEDS VERIFICATION**.

---

## ADR-018 — Background jobs

**Decision.** A **Postgres-backed durable job queue** (`background_operations` table) with a worker process, idempotency keys, retry with backoff, and dead-letter state. No external queue service for the MVP.

**Rationale.** Reuses the existing database (no new infra), transactions integrate with the data, idempotency prevents duplicate PRs/webhook double-processing. Adequate for MVP volumes.

**Alternatives considered.**
- *QStash / Inngest / BullMQ+Redis* — rejected for MVP: a justified external service isn't yet required; can be introduced later if volume/latency demands it.

**Status:** Proposed.

---

## ADR-019 — Caching

**Decision.** **Two tiers:** (1) a durable cache for GitHub responses and rendered artifacts stored in Postgres; (2) an optional **Redis** (or in-memory) hot cache for rate-limit counters and short-TTL reads, added only if profiling shows need.

**Rationale.** GitHub rate limits and repeated renders are the main cache drivers; a DB-backed cache suffices initially. Redis is explicitly optional to avoid unjustified infra.

**NEEDS VERIFICATION:** Redis provider/cost/India availability if adopted.

**Status:** Proposed.

---

## ADR-020 — Object storage

**Decision.** **S3-compatible object storage** for exported website bundles and any large snapshots, accessed server-side via short-lived presigned URLs (never exposing credentials).

**Rationale.** Exports and large snapshots don't belong in the DB row. Presigned URLs keep storage off the client.

**NEEDS VERIFICATION:** provider (Supabase storage / R2 / S3), pricing, free-tier, India availability.

**Status:** Proposed, **NEEDS VERIFICATION**.

---

## ADR-021 — Email

**Decision.** Use **Resend or AWS SES** for transactional email (verification, publish notifications), configured server-side.

**NEEDS VERIFICATION:** pricing, free tier, India deliverability/sender reputation.

**Status:** Proposed, **NEEDS VERIFICATION**.

---

## ADR-022 — Monitoring & error tracking

**Decision.** **Sentry** for error tracking + performance; **structured JSON logging** from the app; a simple health endpoint.

**Rationale.** Minimal, sufficient for MVP observability.

**NEEDS VERIFICATION:** Sentry free-tier/quota and India/region availability.

**Status:** Proposed.

---

## ADR-023 — CI/CD

**Decision.** **GitHub Actions**: lint/typecheck/test on PR; deploy on merge to main; migrations applied as a gated deploy step.

**Status:** Proposed.

---

## ADR-024 — Public portfolio delivery

**Decision.** Deliver public portfolios via **App Router public routes** (server-rendered, read-only, cacheable). For high-traffic/SEO, an optional **static export** of published releases to object storage + CDN later.

**Rationale.** Simplest correct MVP; static export is a future optimization, not a requirement.

**Status:** Proposed. **REQUIRES APPROVAL** if static export is desired at launch.

---

## ADR-025 — Billing

**Decision.** **No billing tables in the MVP.** The approved product scope does not justify billing/subscription. Add only if a monetization decision is made and documented.

**Status:** Proposed (no tables).
