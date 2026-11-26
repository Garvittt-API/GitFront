# API Architecture

**Status:** Architecture design (Phase 2)
**Last updated:** 2026-11-26

---

## 1. API approach

- **Next.js Route Handlers** under `app/api/**` (App Router).
- **Zod** validates every request and response; schemas are the single source for both validation and client typing.
- **Consistent JSON envelope** for success and error.
- A **typed client** generated/derived from Zod schemas is used by client islands.
- No separate API service/deployable.

## 2. Route map

```mermaid
flowchart LR
    subgraph Auth[/api/auth]
        A1[POST /session]
        A2[POST /logout]
    end
    subgraph Profile[/api/profile]
        P1[POST /]
        P2[GET /]
        P3[PATCH /]
        P4[POST /autosave]
        P5[GET /revisions]
        P6[GET /versions]
    end
    subgraph Render[/api/render]
        R1[POST /readme/preview]
        R2[POST /readme/export]
        R3[POST /portfolio/preview]
    end
    subgraph Portfolio[/api/portfolio]
        PF1[POST /publish]
        PF2[POST /unpublish]
    end
    subgraph GitHub[/api/github]
        G1[POST /preview]
        G2[POST /publications]
        G3[GET /publications/:id]
        G4[POST /webhooks]
    end
    subgraph Jobs[/api/jobs]
        J1[GET /:id]
    end
```

## 3. Consistent envelope

**Success:**
```json
{ "data": <payload> }
```

**Error:**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable, non-leaking message",
    "details": [ { "path": "identity.handle", "message": "invalid format" } ]
  }
}
```

## 4. Error model

| HTTP | Code | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Zod validation failure |
| 400 | `BAD_REQUEST` | malformed request |
| 401 | `UNAUTHENTICATED` | missing/invalid session |
| 403 | `FORBIDDEN` | authenticated but not owner |
| 404 | `NOT_FOUND` | resource missing |
| 409 | `REVISION_CONFLICT` | stale expected revision |
| 409 | `DUPLICATE` | idempotency/duplicate resource |
| 429 | `RATE_LIMITED` | rate limit exceeded |
| 502 | `UPSTREAM_ERROR` | GitHub/upstream failure |
| 503 | `SERVICE_UNAVAILABLE` | transient |
| 500 | `INTERNAL_ERROR` | unexpected (logged, not leaked) |

**Rules:**
- Never leak stack traces, SQL, or tokens in responses.
- Log full detail server-side; return safe messages client-side.
- GitHub errors are wrapped as `UPSTREAM_ERROR` with a retryable flag.

## 5. Authentication & authorization

- All `/api/**` routes except `auth` and `webhooks` require a session (`requireUser`).
- Object-level authorization via `requireOwnership` (user owns the profile/publication/job).
- Public portfolio delivery is served via **separate public routes** (`/portfolio/[slug]`), not `/api/**`, and only serves active releases.

## 6. Idempotency

- State-changing operations that must not duplicate (publish, import, export) accept an **`Idempotency-Key`** header or derive one server-side (e.g., publication idempotency key).
- Duplicate keys return the original result (`200`/`201`) rather than re-executing.
- Enforced via `idempotency_key` unique constraints.

## 7. Rate limiting

- Per-user and per-IP limits on expensive endpoints (import, publish, preview render).
- Use a server-side limiter (in-memory or Redis). **Redis provider: NEEDS VERIFICATION.**
- Return `429` with `Retry-After`.
- Public portfolio routes are read-only and cacheable; apply CDN/edge caching rather than per-user limits.

## 8. Status codes & async

- Long operations (import, render-heavy, publish) return `202 Accepted` with a `jobId`, and the client polls `GET /api/jobs/:id`.
- Synchronous operations return `200`/`201`.

## 9. Versioning

- API versioning via a prefix only if breaking changes are anticipated (e.g., `/api/v1/**`). **Recommendation:** start unversioned but keep the envelope stable; introduce a prefix at the first breaking change. **REQUIRES APPROVAL** on the versioning strategy.

## 10. Validation & typing

- Request and response schemas are Zod. Route Handlers parse with Zod, then call the service.
- The typed client is derived from the same schemas, eliminating drift.

## 11. Related

- `API_SPECIFICATION.md` — per-endpoint contracts.
- `SECURITY_ARCHITECTURE.md` — authz/threat coverage.
