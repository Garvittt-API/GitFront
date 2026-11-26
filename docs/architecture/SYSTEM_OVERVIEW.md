# System Overview — Developer Identity Platform

**Status:** Architecture design (Phase 2) — not yet implemented
**Last updated:** 2026-11-26
**Source of truth:** Product context supplied in the architecture tasking (README Studio + Portfolio Studio scope, canonical profile principle). *Provenance note: the product spec files in `docs/` were not present at the time of authoring; the product context was provided inline in the tasking message. See `ARCHITECTURE_AUDIT.md` → "Unresolved decisions".*

---

## 1. Purpose

The Developer Identity Platform lets a developer create **one canonical, structured developer profile** and render it into two distinct, first-class outputs:

1. **README Studio** → a GitHub profile `README.md` (Markdown renderer).
2. **Portfolio Studio** → a public portfolio website (Web renderer).

The platform owns the canonical structured data and the rendering pipelines. GitHub is both a **data source** (profile + repository import) and a **publishing target** (README via reviewed pull requests).

## 2. Guiding principles

1. **One source of truth.** A single canonical `ProfileDocument` per developer. Both studios render from it. There are no two separate profile copies.
2. **Two independent renderers.** Markdown renderer and Web renderer are separate, deterministic, independently testable pipelines.
3. **Shared data stays consistent; output customization stays independent.** Fields used by both renderers are shared. README-only and portfolio-only presentation settings live in separate config namespaces and never bleed across.
4. **Imported facts never silently overwrite user-authored content.** GitHub import populates fields and tags provenance; any field already authored by the user is preserved and flagged.
5. **Published versions are immutable and distinguishable from drafts.** Drafts are mutable and versioned; published artifacts are immutable snapshots with explicit release identity.
6. **Least privilege, secrets server-side only.** GitHub tokens, installation credentials, and session tokens never reach the browser.
7. **Affordable and simple first.** A modular monolith on a single PostgreSQL database. No service is introduced without a justified requirement.
8. **Security by design, testable, accessible, production-oriented.**

## 3. Product surface

### 3.1 README Studio
- Public GitHub profile import.
- Repository discovery and selection.
- Structured developer profile creation.
- Professional README templates.
- Editable content and project descriptions.
- Live Markdown preview.
- Copy and download.
- Version history.
- GitHub publishing through reviewed pull requests (future/phased).

### 3.2 Portfolio Studio
- Professional developer portfolio creation.
- Shared canonical developer profile (same profile as README Studio).
- Independent portfolio presentation settings.
- Multiple responsive templates.
- Section customization and ordering.
- Live preview.
- Public portfolio publishing.
- Website export.
- Custom domains (future).

## 4. High-level capability map

| Capability | Owning module | Notes |
|---|---|---|
| Authentication & account | `auth` | GitHub login + session management |
| Canonical profile storage | `profile` | Single `ProfileDocument` per user |
| GitHub data import | `github-integration` | Public lookup, repo discovery, snapshots |
| Profile editing & autosave | `profile` | Draft revisions, optimistic concurrency |
| Markdown rendering | `rendering/markdown` | Deterministic README output |
| Portfolio rendering | `rendering/portfolio` | Deterministic web output |
| Template system | `templates` | README + portfolio template registry |
| Versioning & publishing | `versioning` | Revisions, snapshots, releases, rollback |
| GitHub publishing | `github-integration` | PR creation via GitHub App |
| Background jobs | `jobs` | Async import, render, publish, webhooks |
| Public portfolio delivery | `delivery` | Public read-only portfolio endpoints |

## 5. System context

```mermaid
flowchart LR
    DEV[Developer / User]
    BROWSER[Browser / Next.js App]
    APP[Developer Identity Platform\n(Next.js monolith)]
    DB[(PostgreSQL)]
    GH[GitHub API / Webhooks]
    STORE[(Object Storage\noptional)]
    CDN[Public Portfolio Delivery]

    DEV --> BROWSER
    BROWSER -->|HTTPS| APP
    APP -->|SQL| DB
    APP -->|OAuth / API / Webhooks| GH
    APP -->|blobs| STORE
    APP -->|rendered public site| CDN
    GH -->|webhooks| APP
```

## 6. Core concepts

- **ProfileDocument** — the versioned, validated canonical structured profile (see `PROFILE_DOCUMENT_SPEC.md`).
- **Revision** — an immutable, append-only snapshot of a `ProfileDocument` at a point in time (draft autosave, snapshot, export, or published).
- **Release** — a published, immutable artifact derived from a specific revision (a README publication or a portfolio release).
- **Template** — a registered, versioned presentation definition for a renderer.
- **Snapshot (GitHub)** — cached, point-in-time GitHub data (profile + repos), stored separately from user-authored data.
- **Background operation** — a durable, idempotent unit of asynchronous work.

## 7. Non-goals (explicit, to avoid scope creep)

- No separate profile stores per studio.
- No general-purpose CMS or blogging platform.
- No AI generation in the MVP rendering pipeline (see security: prompt injection).
- No custom-domain DNS management in the MVP (future).
- No billing/subscription systems unless the approved roadmap justifies them (none justified at this time → no billing tables).
- No microservices in the MVP.

## 8. Related documents

See `docs/architecture/` index. Key references: `HIGH_LEVEL_ARCHITECTURE.md`, `MODULE_BOUNDARIES.md`, `DATABASE_ARCHITECTURE.md`, `PROFILE_DOCUMENT_SPEC.md`, `SECURITY_ARCHITECTURE.md`.
