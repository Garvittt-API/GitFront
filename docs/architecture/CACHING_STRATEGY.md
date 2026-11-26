# Caching Strategy

**Status:** Architecture design (Phase 2)
**Last updated:** 2026-11-26

---

## 1. Caching goals

1. **Conserve GitHub API rate limits** (the primary driver).
2. **Speed up repeated renders** (preview, published pages).
3. **Reduce DB load** for hot reads (profiles, public portfolios).

## 2. Two-tier model

```mermaid
flowchart LR
    REQ[Request] --> HOT{hot cache\n(Redis / in-memory)}
    HOT -->|miss| DUR[durable cache\n(Postgres tables)]
    DUR -->|miss| SOURCE[source: GitHub API / DB]
    SOURCE --> DUR
    DUR --> HOT
    HOT --> REQ
```

- **Tier 1 — durable cache (Postgres):** always available, survives restarts. Stores GitHub snapshots/ETags and rendered artifacts.
- **Tier 2 — hot cache (Redis or in-memory):** optional, short TTL, for rate-limit counters and hot reads. Added only if profiling justifies it.

**NEEDS VERIFICATION:** Redis provider/cost/India availability if adopted. Recommend starting **without** Redis and adding it only on evidence.

## 3. What to cache

| Data | Cache tier | TTL / invalidation |
|---|---|---|
| GitHub profile snapshot | durable (`github_snapshots`) + ETag | refresh on TTL or explicit |
| GitHub repo list | durable (`github_repositories`) | refresh on TTL/webhook |
| GitHub API responses | durable + conditional requests | ETag / `If-None-Match` |
| Rendered README (per revision+template) | durable (derived) | immutable per revision → cacheable long |
| Rendered portfolio (per release) | durable + edge | immutable per release → cacheable long |
| Public portfolio HTML | edge/CDN | long, invalidate on unpublish |
| Rate-limit counters | hot (Redis/in-memory) | short window |

## 4. GitHub-specific caching

- Use **conditional requests** (`If-None-Match` with stored `etag`). `304 Not Modified` conserves quota.
- Honor `X-RateLimit-Remaining`; when low, rely on cache and queue refreshes.
- Never cache a GitHub token or auth-dependent private data in a shared cache.

## 5. Rendered-artifact caching

- Rendered output is a **pure function** of (document revision, template, options) → cacheable by a content hash key.
- Cache key: `render:{kind}:{revisionId}:{templateId}:{optionsHash}`.
- Immutable inputs → long TTL / no invalidation needed.
- Invalidate on template change/version.

## 6. Public portfolio delivery caching

- Public pages are **read-only and immutable per release** → cache at the edge (CDN) with long TTL.
- Invalidate/purge on publish, unpublish, or release replacement.
- **Security:** ensure private content is never cached publicly; private fields are excluded before caching.

## 7. Cache invalidation

- **Event-driven:** publish/unpublish, template change, import refresh trigger invalidation/purge.
- **TTL-based:** GitHub snapshots refresh on TTL.
- **Content-hash:** rendered artifacts keyed by content hash are naturally invalidate-free.

## 8. Failure behavior

- If the hot cache is unavailable, fall back to the durable cache / source (caches are an optimization, not a correctness requirement).
- Cache stampede protection: single-flight/lock for expensive regeneration.
- Never serve stale **authorization** decisions from cache; auth is checked per request.

## 9. Related

- `GITHUB_INTEGRATION.md` §4-6 — rate-limit handling and snapshots.
- `DEPLOYMENT_ARCHITECTURE.md` — edge/CDN configuration.
