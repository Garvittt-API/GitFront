# Database Architecture

**Status:** Architecture design (Phase 2)
**Last updated:** 2026-11-26

---

## 1. Database choice & hosting

- **Engine:** PostgreSQL (single system of record).
- **Access:** Drizzle ORM with a thin connection/transaction layer.
- **Hosting:** Managed PostgreSQL provider. **Provider, pricing, free-tier, and India-region availability: NEEDS VERIFICATION.**

**Rationale:** relational integrity for ownership/versions/publications, JSONB for the versioned document and snapshots, mature tooling, transactional job queue, and broad managed availability.

## 2. Design principles

1. **Relational core, JSONB document.** Identity, ownership, versions, publications, jobs, and GitHub metadata are relational. The canonical `ProfileDocument` is JSONB inside an immutable revision row, validated by Zod.
2. **Ownership via foreign keys.** Every user-owned row has `user_id` FK → `users`, and cross-account access is impossible by construction (queries always scope by `user_id`).
3. **UUIDs for public-facing IDs.** Use UUIDs for `users`, `profiles`, `revisions`, `publications`, etc. Avoid sequential integers for anything exposed externally (prevents enumeration).
4. **Immutable revisions.** `profile_revisions` is append-only; updates are new rows.
5. **Index for real queries only.** Index foreign keys, unique constraints, and query patterns (e.g., `(user_id, created_at)` for history). Avoid speculative indexes.
6. **Soft delete for user data.** `deleted_at` for user-owned content; hard delete only via a retention purge job.
7. **No billing tables** in the MVP (no approved monetization requirement).

## 3. Connection & transaction management

- A single pooled Postgres client per process; Drizzle configured with the pool.
- Repository functions accept an optional `tx` (transaction) so multi-table operations (e.g., create revision + release) are atomic.
- Long-running jobs use short transactions; no long-held locks.
- Connection string from server-side env only.

## 4. Migration strategy

- **Drizzle migrations**, forward-only, generated locally and reviewed via PR.
- Baseline migration for the initial schema.
- Deploy gate: migrations run as a distinct, reversible-by-deploy step before/with app rollout.
- Rule: no destructive migration (drop/rename that loses data) without a documented, tested rollback and a product-owner-approved plan.
- `schema_version` on `profiles` tracks the document schema; document-schema changes are separate from DB migrations.

## 5. Data modeling: relational vs JSONB boundaries

| Data | Storage | Why |
|---|---|---|
| Users, accounts, sessions | relational | identity, integrity, revocation |
| Profile metadata + current/published revision pointers | relational | queryable, FK integrity |
| `ProfileDocument` content | JSONB (in revisions) | schema-evolving, versioned, custom sections |
| GitHub snapshots/repo metadata | relational + JSONB payload | queryable repo lists + raw cache |
| Templates | relational + JSONB config | registry + versioned config |
| Publications/releases | relational | status, FK to revision, idempotency |
| Jobs | relational | durable queue, idempotency keys |
| Webhook events | relational + JSONB payload | idempotency, audit |

## 6. Retention & deletion

- **Soft delete:** user-owned rows get `deleted_at`. Deleted users/profiles are hidden from all queries immediately.
- **Retention purge job:** after a configured retention window (e.g., 30 days, configurable), a job hard-deletes or anonymizes soft-deleted user data (profiles, revisions, snapshots, sessions, publications), keeping only what is legally required.
- **Sessions:** expire by `expires_at`; revoked on logout.
- **Webhook events & job history:** retained for a configurable window (observability), then purged; publication/release records retained as needed for audit but anonymized if the user is purged.
- **GitHub snapshots:** refreshable caches; expired/stale snapshots purged on a schedule; never treated as the source of truth for user content.
- **GDPR/erasure:** a user-deletion routine cascades (via FK `ON DELETE` / application logic) to remove or anonymize all user data. *Confirm applicable jurisdictions (incl. India DPDP Act) — NEEDS VERIFICATION for legal specifics.*

## 7. Backups & durability

- Managed Postgres continuous backup + point-in-time recovery (PITR) where the provider supports it. **Provider capability: NEEDS VERIFICATION.**
- RPO/RTO targets defined in `DISASTER_RECOVERY.md`.
- Object storage (exports) has its own versioning/backup per provider.

## 8. Security considerations (database)

- Least-privilege DB role for the app (no superuser); read/write only to its schema.
- No secrets in DB columns (sessions store token **hashes**, not raw tokens).
- Encryption at rest via provider; TLS in transit.
- SQL injection prevented by Drizzle's parameterized queries (never raw string interpolation).
- Row-level scoping by `user_id` enforced in repository code; authorization checked at the service layer.

## 9. Performance notes

- The document is read/written frequently (autosave). Store as JSONB; revisions are append-only so writes are inserts.
- `profiles.current_revision_id` avoids scanning revisions for "latest".
- Index `(profile_id, revision_number)` and `(profile_id, created_at)` for history.
- GitHub snapshot reads are cache-friendly; avoid hot-looping GitHub API in request paths (use jobs).

## 10. Related

- `DATABASE_SCHEMA.md` — full table designs.
- `PROFILE_DOCUMENT_SPEC.md` — document schema stored in JSONB.
