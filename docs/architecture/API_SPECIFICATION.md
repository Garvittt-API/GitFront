# API Specification

**Status:** Architecture design (Phase 2) — contracts for review
**Last updated:** 2026-11-26

All endpoints use the envelope defined in `API_ARCHITECTURE.md`. Auth is via httpOnly session cookie. All `data` responses are Zod-validated.

**Common error responses** (see `API_ARCHITECTURE.md` §4):
```json
{ "error": { "code": "VALIDATION_ERROR", "message": "...", "details": [...] } }
```

---

## 1. Authentication

### 1.1 Create session (GitHub sign-in)
- **Method:** `POST`
- **Route:** `/api/auth/session`
- **Auth:** none
- **Request:** `{ "provider": "github" }`
- **Response:** `200 { "data": { "redirect": "/api/auth/github" } }` (or sets cookie directly via OAuth callback)
- **Errors:** 400 `BAD_REQUEST`
- **Rate limit:** per-IP
- **Idempotency:** n/a

### 1.2 Logout
- **Method:** `POST`
- **Route:** `/api/auth/logout`
- **Auth:** required
- **Response:** `200 { "data": { "revoked": true } }`
- **Errors:** 401 `UNAUTHENTICATED`
- **Idempotency:** safe to repeat

---

## 2. GitHub preview (import preview)

### 2.1 GitHub import preview
- **Method:** `POST`
- **Route:** `/api/github/preview`
- **Auth:** required
- **Authorization:** user must own their profile; imports into the user's own profile only
- **Request:** `{ "githubLogin": string }`
- **Response:** `200 { "data": { "profile": ProfileDocument, "repositories": RepoSummary[], "conflicts": Conflict[] } }`
- **Behavior:** fetches GitHub data (server-side), builds an import proposal, applies import resolution (never overwrites authored fields). Does **not** persist a revision until the user confirms.
- **Errors:** 400 `VALIDATION_ERROR`, 429 `RATE_LIMITED`, 502 `UPSTREAM_ERROR`
- **Rate limit:** per-user (expensive)
- **Idempotency:** optional `Idempotency-Key`

---

## 3. Profile

### 3.1 Create profile
- **Method:** `POST`
- **Route:** `/api/profile`
- **Auth:** required
- **Authorization:** creates a profile for the authenticated user (one per user)
- **Request:** `{ "document": ProfileDocument, "fromImport": boolean }`
- **Response:** `201 { "data": { "profileId": uuid, "revisionId": uuid, "revisionNumber": int, "schemaVersion": int } }`
- **Errors:** 400 `VALIDATION_ERROR`, 409 `DUPLICATE` (profile already exists)
- **Rate limit:** per-user
- **Idempotency:** optional `Idempotency-Key`

### 3.2 Retrieve profile
- **Method:** `GET`
- **Route:** `/api/profile`
- **Auth:** required
- **Authorization:** user must own the profile
- **Response:** `200 { "data": { "profileId", "schemaVersion", "document": ProfileDocument, "currentRevisionId", "currentRevisionNumber", "publishedRevisionId", "status" } }`
- **Errors:** 401, 403 `FORBIDDEN`, 404
- **Rate limit:** per-user
- **Idempotency:** n/a (read)

### 3.3 Update profile (manual save)
- **Method:** `PATCH`
- **Route:** `/api/profile`
- **Auth:** required
- **Authorization:** owner
- **Request:** `{ "document": ProfileDocument, "expectedRevision": int, "changeSummary": string }`
- **Response:** `200 { "data": { "revisionId", "revisionNumber" } }`
- **Errors:** 400 `VALIDATION_ERROR`, 409 `REVISION_CONFLICT` (stale), 403, 404
- **Rate limit:** per-user
- **Idempotency:** optional `Idempotency-Key`

### 3.4 Autosave
- **Method:** `POST`
- **Route:** `/api/profile/autosave`
- **Auth:** required
- **Authorization:** owner
- **Request:** `{ "document": ProfileDocument, "expectedRevision": int }`
- **Response (success):** `200 { "data": { "revisionId", "revisionNumber" } }`
- **Response (conflict):** `409 { "error": { "code": "REVISION_CONFLICT", "details": [ { "currentRevision": int } ] } }`
- **Behavior:** non-destructive; a stale autosave never mutates state.
- **Errors:** 400, 409 `REVISION_CONFLICT`, 403, 404
- **Rate limit:** per-user (frequent, so higher than manual save)
- **Idempotency:** the `expectedRevision` check is the primary guard; optional `Idempotency-Key`

### 3.5 Revision conflict handling
- Handled by `expectedRevision` on update/autosave.
- On `409 REVISION_CONFLICT`, the client must re-fetch the profile and merge before retrying.
- No separate endpoint; conflict is a response condition. (Documented here per the required list.)

### 3.6 Version history
- **Method:** `GET`
- **Route:** `/api/profile/revisions`
- **Auth:** required
- **Authorization:** owner
- **Query:** `?limit=20&before=<cursor>`
- **Response:** `200 { "data": { "items": [ { "revisionId", "revisionNumber", "revisionKind", "source", "changeSummary", "createdAt" } ], "nextCursor": string | null } }`
- **Errors:** 401, 403, 404
- **Rate limit:** per-user
- **Idempotency:** n/a

### 3.7 Profile versions (named snapshots)
- **Method:** `GET`
- **Route:** `/api/profile/versions`
- **Auth:** required
- **Authorization:** owner
- **Response:** `200 { "data": { "items": [ { "versionId", "versionLabel", "revisionId", "createdAt" } ] } }`
- **Errors:** 401, 403, 404
- **Idempotency:** n/a

---

## 4. Rendering & export

### 4.1 README preview
- **Method:** `POST`
- **Route:** `/api/render/readme/preview`
- **Auth:** required
- **Authorization:** owner of the document/revision
- **Request:** `{ "revisionId": uuid | null, "templateId": uuid | null }` (`revisionId` null = current draft)
- **Response:** `200 { "data": { "markdown": string, "templateId": uuid, "schemaVersion": int } }`
- **Errors:** 400, 403, 404, 409 `INCOMPATIBLE_TEMPLATE` (or 400 with code `TEMPLATE_INCOMPATIBLE`)
- **Rate limit:** per-user
- **Idempotency:** n/a (pure render)

### 4.2 README export
- **Method:** `POST`
- **Route:** `/api/render/readme/export`
- **Auth:** required
- **Authorization:** owner
- **Request:** `{ "revisionId": uuid | null, "templateId": uuid | null }`
- **Response:** `202 { "data": { "jobId": uuid } }` (async if large) OR `200 { "data": { "markdown": string, "downloadUrl": string | null } }`
- **Behavior:** exports the selected revision; the exported markdown must match the selected revision exactly.
- **Errors:** 400, 403, 404
- **Rate limit:** per-user
- **Idempotency:** optional `Idempotency-Key`

### 4.3 Portfolio preview
- **Method:** `POST`
- **Route:** `/api/render/portfolio/preview`
- **Auth:** required
- **Authorization:** owner
- **Request:** `{ "revisionId": uuid | null, "templateId": uuid | null }`
- **Response:** `200 { "data": { "html": string, "templateId": uuid, "schemaVersion": int } }`
- **Errors:** 400, 403, 404, 400 `TEMPLATE_INCOMPATIBLE`
- **Rate limit:** per-user
- **Idempotency:** n/a

---

## 5. Portfolio publishing

### 5.1 Publish portfolio
- **Method:** `POST`
- **Route:** `/api/portfolio/publish`
- **Auth:** required
- **Authorization:** owner
- **Request:** `{ "revisionId": uuid, "templateId": uuid | null, "slug": string }`
- **Response:** `201 { "data": { "releaseId": uuid, "slug": string, "publishedAt": timestamptz, "isActive": true } }`
- **Behavior:** creates an immutable `portfolio_releases` row from the revision; makes it active. Private items are excluded from public output.
- **Errors:** 400, 403, 404, 409 `DUPLICATE`
- **Rate limit:** per-user
- **Idempotency:** optional `Idempotency-Key` (idempotent by `(slug, revisionId)`)

### 5.2 Unpublish portfolio
- **Method:** `POST`
- **Route:** `/api/portfolio/unpublish`
- **Auth:** required
- **Authorization:** owner
- **Request:** `{ "releaseId": uuid }`
- **Response:** `200 { "data": { "releaseId", "isActive": false, "unpublishedAt": timestamptz } }`
- **Behavior:** deactivates the release immediately; the public route stops serving it.
- **Errors:** 400, 403, 404
- **Rate limit:** per-user
- **Idempotency:** safe to repeat (setting inactive is idempotent)

---

## 6. GitHub publishing

### 6.1 GitHub publishing approval
- **Method:** `POST`
- **Route:** `/api/github/publications`
- **Auth:** required
- **Authorization:** owner + must have a GitHub App installation with write access to the target repo
- **Request:** `{ "revisionId": uuid, "repoOwner": string, "repoName": string, "branch": string }`
- **Response:** `202 { "data": { "publicationId": uuid, "status": "pending" } }`
- **Behavior:** creates an immutable release + publication record (idempotency key), enqueues the publish job. Requires explicit user approval.
- **Errors:** 400, 403 (no installation/permission), 404, 409 `DUPLICATE`
- **Rate limit:** per-user
- **Idempotency:** **required** — idempotency key `publication:{profileId}:{revisionId}:{repo}` prevents duplicate PRs.

### 6.2 GitHub publication status
- **Method:** `GET`
- **Route:** `/api/github/publications/:id`
- **Auth:** required
- **Authorization:** owner
- **Response:** `200 { "data": { "publicationId", "revisionId", "repoOwner", "repoName", "branch", "prNumber", "prUrl", "status", "attemptCount", "updatedAt" } }`
- **Errors:** 401, 403, 404
- **Rate limit:** per-user
- **Idempotency:** n/a

### 6.3 Webhook (inbound, not a user API)
- **Method:** `POST`
- **Route:** `/api/github/webhooks`
- **Auth:** signature verification (GitHub `X-Hub-Signature-256`)
- **Response:** `200 { "data": { "accepted": true } }` or `401` on invalid signature
- **Rate limit:** per-IP
- **Idempotency:** event-id idempotency in `webhook_events`

---

## 7. Background operations

### 7.1 Background operation status
- **Method:** `GET`
- **Route:** `/api/jobs/:id`
- **Auth:** required
- **Authorization:** owner (or system)
- **Response:** `200 { "data": { "jobId", "type", "status", "attempts", "maxAttempts", "createdAt", "completedAt", "error" | null } }`
- **Errors:** 401, 403, 404
- **Rate limit:** per-user
- **Idempotency:** n/a

---

## 8. Public delivery (non-`/api`)

- **Route:** `/portfolio/[slug]`
- **Auth:** none (public)
- **Authorization:** serves only **active** `portfolio_releases`; private fields excluded.
- **Response:** server-rendered HTML (200), or 404 if not published/active.
- **Rate limit:** edge/CDN cached; no per-user limit.
- **Idempotency:** n/a

## 9. Consistency notes

- Every `data` payload is Zod-validated.
- Error envelope is identical across all endpoints.
- Object-level authorization (`owner`) is enforced on every owner-scoped endpoint.
- Idempotency is enforced where duplication is harmful (publish, import, export, publication).
- Async endpoints return `202` + `jobId`; status via `/api/jobs/:id`.
