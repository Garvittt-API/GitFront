# Repository Structure

**Status:** Architecture design (Phase 2)
**Last updated:** 2026-11-26

A single repository (modular monolith). Structure favors clear boundaries without excessive package fragmentation. All paths are under the repo root.

---

## 1. Top-level layout

```
.
├── app/                      # Next.js App Router: routes, pages, route handlers
├── components/               # Shared UI components (presentational + islands)
├── src/
│   ├── modules/              # Domain modules (one folder per module)
│   ├── lib/                  # Shared leaf utilities (no domain imports)
│   ├── db/                   # Drizzle schema, migrations, connection
│   ├── schema/               # Canonical ProfileDocument + shared Zod schemas
│   ├── auth/                 # Auth.js config + session primitives
│   └── ...
├── tests/                    # Test suites (unit/integration/e2e/security)
├── scripts/                  # Dev/CI helper scripts
├── .github/workflows/        # CI/CD
├── drizzle/                  # Generated migrations
├── docs/
│   ├── architecture/         # This documentation set
│   └── audit/                # Documentation audit
├── package.json
├── tsconfig.json
├── drizzle.config.ts
└── next.config.ts
```

## 2. `src/modules/` — one folder per module

Each module folder follows a consistent internal shape:

```
src/modules/<module>/
├── index.ts            # Public service interface (what other modules may import)
├── service.ts          # Business logic
├── repository.ts       # Data access (Drizzle) — own tables only
├── schema.ts           # Module-local Zod schemas/types (if any)
└── *.test.ts           # Unit tests colocated or under tests/
```

```
src/modules/
├── auth/                # AuthService, session, ownership primitives
├── profile/             # Canonical ProfileDocument lifecycle
├── templates/           # Template registry + compatibility
├── rendering/
│   ├── markdown/        # Markdown renderer
│   ├── portfolio/       # Portfolio renderer
│   └── index.ts         # RenderService dispatch
├── versioning/          # Revisions, releases, rollback, exports
├── github/              # GitHubService: OAuth, App, import, PR, webhooks
├── jobs/                # JobService + worker + handler registry
└── delivery/            # Public portfolio serving
```

## 3. `src/schema/` — canonical profile (single source of truth)

```
src/schema/
├── profile-document.ts  # The versioned ProfileDocument Zod schema + types
├── profile-version.ts   # schema_version enum + migration helpers
├── errors.ts            # Shared typed errors
└── ids.ts               # ID types
```

This is the **only** place the canonical `ProfileDocument` is defined. Both renderers and both studios import from here.

## 4. `src/db/` — database layer

```
src/db/
├── index.ts             # client + transaction helpers
├── schema/              # Drizzle table definitions (one file per table group)
│   ├── users.ts
│   ├── accounts.ts
│   ├── sessions.ts
│   ├── profiles.ts
│   ├── profile-revisions.ts
│   ├── profile-versions.ts
│   ├── github.ts
│   ├── projects.ts
│   ├── templates.ts
│   ├── publications.ts
│   ├── jobs.ts
│   └── webhooks.ts
└── migrations/          # Drizzle migrations (generated)
```

## 5. `app/` — routes

```
app/
├── (studio)/            # Authenticated studio area
│   ├── readme/          # README Studio pages
│   └── portfolio/       # Portfolio Studio pages
├── portfolio/[slug]/    # Public portfolio route (delivery)
├── api/
│   ├── auth/            # session, logout
│   ├── profile/         # profile CRUD, autosave, revisions, versions
│   ├── github/          # import, repos, publications, webhooks
│   ├── render/          # preview, export
│   ├── portfolio/       # publish, unpublish, preview
│   └── jobs/            # background operation status
└── layout.tsx / globals.css
```

## 6. `components/` — UI

```
components/
├── ui/                  # Generic primitives (button, card, input, ...)
├── studio/              # Studio-specific components
│   ├── readme/          # README editor, preview, template picker
│   └── portfolio/       # Portfolio editor, section ordering, preview
└── shared/              # Cross-studio shared presentational components
```

**Rule:** components are presentational/interactive only. Business logic lives in `src/modules/*`. No DB/auth/secrets in components.

## 7. `src/lib/` — shared utilities (leaf)

```
src/lib/
├── logger.ts            # Structured logging
├── env.ts               # Server-side env validation
├── rate-limit.ts        # Rate-limit helpers
├── idempotency.ts       # Idempotency-key helpers
└── text/                # Sanitization/util helpers (markdown, urls)
```

## 8. Tests

```
tests/
├── unit/                # Pure function tests (renderers, validation, import resolution)
├── integration/         # Service + DB tests (revisions, publishing, jobs)
├── api/                 # Route Handler tests (authz, status codes, envelopes)
├── e2e/                 # Critical user workflows
├── security/            # Threat-model verification tests
├── a11y/                # Accessibility tests
└── fixtures/            # Profile documents, templates, snapshots
```

See `TESTING_ARCHITECTURE.md`.

## 9. Configuration & docs

- `.env.example` — documented server-only env vars (no secrets committed).
- `docs/architecture/` — this set.
- `drizzle/` — migrations (reviewed via PR).

## 10. Dependency direction reminder

The physical layout mirrors `MODULE_BOUNDARIES.md`. Enforce with a lint/CI rule that:
- `src/schema` and `src/lib` import no domain module.
- `src/modules/*` import only allowed modules (per the dependency graph).
- `components/` and `app/` import domain services, not DB internals or secrets.
