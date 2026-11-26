# Architecture Audit

**Status:** Architecture design (Phase 2) — verification report
**Last updated:** 2026-11-26

Verification of the architecture against the required checks, plus an honest accounting of unresolved decisions, risks, and gaps.

---

## 1. Verification checklist

| # | Check | Result |
|---|-------|--------|
| 1 | Every required document exists | ✅ All 25 created under `docs/architecture/` |
| 2 | Database relationships | ✅ ER diagram + FKs documented; one profile per user; append-only revisions |
| 3 | API consistency with data model | ✅ Endpoints reference `profiles`, `profile_revisions`, `profile_versions`, `github_publications`, `portfolio_releases`, `background_operations` |
| 4 | Module dependency direction | ✅ Acyclic; `shared` is a leaf; documented allowed edges |
| 5 | Authentication & authorization boundaries | ✅ Server-side sessions; `requireUser`/`requireOwnership`; object-level authz |
| 6 | GitHub permission separation | ✅ Sign-in ≠ repo write; App installation for publishing |
| 7 | Profile schema supports both studios | ✅ Shared document + separate `readme`/`portfolio` config namespaces |
| 8 | Versioning & publishing consistency | ✅ Append-only revisions; immutable releases; optimistic concurrency |
| 9 | MVP requirements map to architecture modules | ✅ Mapped (see §8) |
| 10 | Testing covers critical requirements | ✅ All 10 critical scenarios defined + security tests |
| 11 | No undocumented external dependencies | ✅ Listed; provider specifics marked NEEDS VERIFICATION |
| 12 | Remaining NEEDS VERIFICATION items | ⚠️ Consolidated in §3 |

## 2. Completed architecture areas

- Application architecture (Next.js App Router, modular monolith, server/client boundaries, API, jobs, packages).
- Database (Postgres, Drizzle, relational + JSONB, ownership, indexes, migrations, retention).
- Authentication (GitHub OAuth + DB sessions, provider model, linking, separation from repo permission).
- GitHub integration (lookup, discovery, rate limits, caching, snapshots, refresh, OAuth, App installs, ownership verification, PR publishing, webhooks, token security).
- Rendering (canonical document, validation, import resolution, Markdown + portfolio generation, template registration/compatibility, preview, published rendering).
- Versioning (drafts, autosave, optimistic concurrency, immutable snapshots, exports, releases, rollback, conflict resolution).
- Infrastructure (hosting, DB, storage, jobs, caching, email, monitoring, logging, errors, CI/CD).
- Security (full threat model with mitigations + verification tests).
- Repository structure, module boundaries, testing, observability, disaster recovery.

## 3. NEEDS VERIFICATION (consolidated)

These are **not** assumptions of fact — they require confirmation before implementation:

1. **Hosting provider** (Vercel): pricing, free-tier, India-region availability, data residency.
2. **Managed PostgreSQL provider:** pricing, free-tier, India-region availability, PITR/backup capabilities.
3. **Object storage provider:** choice, pricing, free-tier, India availability.
4. **Redis (optional):** only if profiling justifies; provider/cost/India.
5. **Email provider (Resend/SES):** pricing, free-tier, India deliverability.
6. **Sentry:** free-tier/quota, India availability.
7. **GitHub:** OAuth app creation, GitHub App creation/billing/limits, unauthenticated vs authenticated rate limits, pagination, webhook reliability.
8. **GitHub App installation UX** and revocation flow.
9. **GDPR/India DPDP Act** legal specifics for data retention/erasure and observability PII.
10. **Metrics/alerting** provider and integration.
11. **API versioning strategy** (`/api/v1` prefix vs unversioned) — REQUIRES APPROVAL.
12. **Static export** of portfolios at launch vs runtime delivery — REQUIRES APPROVAL.

## 4. Unresolved decisions (REQUIRES APPROVAL)

| ID | Decision | Proposal | Risk if deferred |
|----|----------|----------|------------------|
| D1 | Auth library | Auth.js (NextAuth v5) + Drizzle adapter | Alternative: bespoke session service |
| D2 | GitHub publishing mechanism | GitHub App installation | Requires external App creation; broad `repo` OAuth rejected |
| D3 | API versioning | Start unversioned; add prefix at first breaking change | Breaking-change discipline |
| D4 | Portfolio delivery | Runtime App Router routes (static export optional) | Static export is a future optimization |
| D5 | Billing | No billing tables in MVP | Only if monetization is decided |
| D6 | Product documentation provenance | The spec files in `docs/` were absent; architecture derived from inline product context | The inline context should be committed as the product spec before implementation |

> **D6 is important:** the architecture was authored from the product context provided in the tasking message because the `docs/` product specification files were not present (see `docs/audit/DOCUMENTATION_AUDIT.md`). This must not be treated as silently approved documentation; the product owner should commit the spec so the architecture has a durable source of truth.

## 5. Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Provider pricing/free-tier assumptions wrong | High | Verify all providers before implementation; design to be portable (managed Postgres, S3-compatible, standard Node) |
| GitHub rate limits constrain import/refresh UX | Medium | Caching, conditional requests, backoff, async jobs, clear UX |
| GitHub App setup is non-trivial | Medium | Document setup; separate sign-in from publishing; start with import-only MVP if App is delayed |
| Boundary erosion in the monolith | Medium | Enforce dependency rules in CI; module interfaces; code review |
| Private-field leakage in public output | High | Visibility enforced at service + delivery; tests; sanitization |
| Token exposure | High | Server-side only; secret scanning; no tokens in client/logs |
| Over-engineering | Medium | Explicit "no service without justification"; modular monolith; optional Redis |
| Schema drift between document spec and DB | Medium | Single Zod spec; `schema_version`; migration tests |

## 6. Inconsistencies / gaps identified

- **No contradictions found within the architecture**, but a **provenance gap** exists: the product spec files were missing, so the architecture is derived from inline context (flagged in D6).
- **No billing tables** — consistent with "add only if justified"; no monetization requirement exists.
- **AI features** — explicitly out of MVP; threat-modeled for future (prompt injection) but not implemented.
- **Custom domains** — future; architecture anticipates routing but does not implement.

## 7. Missing specifications (to close before implementation)

- The committed **product specification** in `docs/` (vision, MVP scope, acceptance criteria, decision log). *Required to make the architecture's source of truth durable.*
- **Provider selection** confirmations (hosting, DB, storage, email, monitoring).
- **Acceptance criteria** for each studio workflow (to drive E2E tests).
- **Brand/identity** guidance (theme/visual system) — deferred; template system is data-driven.

## 8. Security concerns (summary)

- Full threat model in `SECURITY_ARCHITECTURE.md`.
- Highest-priority controls: object-level authz, private-field exclusion, token security, webhook signature/idempotency, Markdown/URL sanitization, optimistic concurrency.
- All threats have verification tests in `TESTING_ARCHITECTURE.md`.

## 9. Implementation dependencies

1. **Product spec committed** (D6) → validates the architecture's source of truth.
2. **Provider selections verified** (hosting, DB, storage, email, monitoring).
3. **GitHub OAuth app + (later) GitHub App** provisioned (external setup).
4. **Auth library decision approved** (D1).
5. **Schema + migration baseline** (from `DATABASE_SCHEMA.md`).
6. **ProfileDocument Zod schema** (from `PROFILE_DOCUMENT_SPEC.md`) as the first shared package.
7. **Renderers + golden tests** (determinism).
8. **Versioning + publishing** flows.
9. **GitHub integration** (import first, publishing later).

## 10. Recommended implementation sequence

> **Not starting implementation now — awaiting approval.** Sequence for after approval:

1. **Foundation:** repo scaffold, config, DB connection, Drizzle schema + baseline migration, `shared` schema (ProfileDocument Zod).
2. **Auth:** Auth.js + sessions + `requireUser`/`requireOwnership`.
3. **Profile:** create/load/edit/autosave with optimistic concurrency + revisions.
4. **Rendering:** Markdown renderer + golden tests; then portfolio renderer.
5. **Templates:** registry + compatibility + built-in templates.
6. **Import:** GitHub import + resolution (import-first, no overwrite).
7. **Versioning/publishing:** versions, portfolio publish/unpublish.
8. **GitHub publishing:** App install + PR flow (after import is solid).
9. **Jobs/observability/deploy:** worker, Sentry, logging, CI/CD, deploy.
10. **Security & accessibility:** security test suite, a11y, performance baselines.

## 11. Conclusion

The architecture is **coherent, complete, and implementation-ready at a design level**, contingent on: (a) the product spec being committed, (b) provider verifications, and (c) approval of the decisions in §4. No implementation should begin until those are resolved and explicitly approved.
