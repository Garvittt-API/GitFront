# Data Flow

**Status:** Architecture design (Phase 2)
**Last updated:** 2026-11-26

End-to-end data flows for the platform's critical workflows.

---

## 1. Authentication flow

```mermaid
sequenceDiagram
    participant U as User
    participant B as Browser
    participant A as Auth Module
    participant GH as GitHub
    participant DB as Postgres

    U->>B: Click "Sign in with GitHub"
    B->>A: GET /api/auth/github
    A->>GH: Redirect to GitHub OAuth authorize
    GH->>U: Authorize (minimal scopes)
    U->>GH: Approve
    GH->>A: Redirect with code
    A->>GH: Exchange code for access_token (server-side)
    A->>GH: GET /user (read:user)
    A->>DB: find/create user + account row
    A->>DB: create session row (token hash)
    A->>B: Set httpOnly session cookie
    B->>A: Authenticated requests carry cookie
```

- The GitHub access token from sign-in is **never** sent to the browser and is **not** used for repo modification.
- Session cookie is opaque, httpOnly, Secure, SameSite=Lax.

## 2. GitHub import flow

```mermaid
flowchart TD
    A[User: Import from GitHub] --> B[GitHubService fetch public profile]
    B --> C[GitHubService list repositories]
    C --> D[Store github_snapshot + github_repositories]
    D --> E[Build import proposal]
    E --> F[ProfileService.resolveImport]
    F --> G{field authored?}
    G -->|Yes| H[Keep authored value, flag conflict]
    G -->|No| I[Fill from snapshot, source=github]
    H --> J[Create draft revision with provenance]
    I --> J
    J --> K[User reviews/edits in studio]
```

**Invariant:** imported data never overwrites authored content. Conflicts are shown to the user.

## 3. Profile editing & autosave flow

```mermaid
sequenceDiagram
    participant E as Editor (client island)
    participant H as /api/profile/autosave
    participant P as ProfileService
    participant DB as Postgres

    E->>H: POST { document, expectedRevision }
    H->>P: saveRevision(userId, document, expectedRevision)
    P->>DB: read profile.current_revision
    alt matches
        P->>DB: INSERT profile_revisions (revision_number+1)
        P->>DB: update profiles.current_revision
        P-->>H: new revision
        H-->>E: 200 { revisionNumber, revisionId }
    else mismatch
        P-->>H: conflict
        H-->>E: 409 { currentRevision }
        E->>E: reload & merge
    end
```

## 4. README rendering flow

```mermaid
flowchart LR
    D[ProfileDocument\n(revision)] --> V[Zod validate]
    V --> T[TemplateService.resolve\ntemplate + schema_version]
    T --> M[MarkdownRenderer]
    M --> OUT[README.md string]
    OUT --> PREVIEW[Live preview]
    OUT --> COPY[Copy / Download]
    OUT --> EXPORT[Export revision]
```

Deterministic: same (document, template, options) → same output. Golden tests assert equality.

## 5. Portfolio rendering flow

```mermaid
flowchart LR
    D[ProfileDocument\n(revision)] --> V[Zod validate]
    V --> T[TemplateService.resolve\nportfolio template]
    T --> P[PortfolioRenderer]
    P --> HTML[HTML / ReactNode]
    HTML --> PREVIEW[Live preview]
    HTML --> PUBLISH[Create portfolio release]
    PUBLISH --> PUBLIC[Public delivery route]
```

## 6. GitHub pull-request publishing flow

```mermaid
sequenceDiagram
    participant U as User
    participant H as /api/github/publications
    participant V as VersioningService
    participant G as GitHubService
    participant J as JobService
    participant GH as GitHub

    U->>H: POST publish { revisionId, repo, branch }
    H->>V: create immutable release for revision
    H->>G: create github_publications row (pending)
    H->>J: enqueue publish job (idempotency key)
    H-->>U: 202 { publicationId }
    J->>G: create PR via App installation token
    G->>GH: POST pulls
    GH-->>G: PR number/url
    G->>J: status=open
    loop status polling / webhook
        J->>GH: GET PR
        GH-->>J: state
    end
    J->>G: final status (merged/closed)
```

Idempotency key prevents duplicate PRs on retry. See `GITHUB_INTEGRATION.md`.

## 7. Webhook flow

```mermaid
flowchart TD
    A[GitHub webhook] --> B[Verify signature + event]
    B --> C{valid?}
    C -->|No| D[Reject 401]
    C -->|Yes| E[Store webhook_events with event_id unique]
    E --> F{duplicate?}
    F -->|Yes| G[Skip (idempotent)]
    F -->|No| H[Enqueue job]
    H --> I[Process: update publication/repo state]
```

## 8. Published vs draft data flow

```mermaid
flowchart LR
    DRAFT[Draft revisions\nmutable, autosaved] --> SNAP[Snapshot/Export revision\nimmutable]
    SNAP --> REL[Release\nimmutable, typed]
    REL --> R1[README publication\ngithub_publications]
    REL --> R2[Portfolio release\nportfolio_releases]
    R2 --> PUB[Public delivery]
```

- Drafts are the editing surface.
- Releases are immutable and reference a specific revision.
- Public delivery only ever serves releases, never drafts.

## 9. Background operation data flow

```mermaid
flowchart LR
    REQ[Request] --> ENQ[JobService.enqueue]
    ENQ --> TBL[(background_operations)]
    TBL --> WR[Worker claim]
    WR --> RUN[Execute handler]
    RUN --> OK[succeeded]
    RUN --> RETRY[retry w/ backoff]
    RETRY --> DLQ[failed/dead-letter]
```

Status is exposed via `/api/jobs/:id`. The UI polls or subscribes.

## 10. Data flow invariants summary

| Invariant | Enforced at |
|---|---|
| One canonical profile per user | `profiles.user_id` unique |
| Imported ≠ authored | `PROFILE_DOCUMENT_SPEC.md` provenance + `resolveImport` |
| Revisions immutable | append-only `profile_revisions` |
| Published immutable & revision-linked | `*_releases` / `github_publications` reference revisions |
| Public serves releases only | `delivery` module reads releases only |
| Secrets server-side | auth/github modules never export tokens to client |
| Deterministic render | pure renderer functions + golden tests |
