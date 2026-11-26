# Canonical ProfileDocument Specification

**Status:** Architecture design (Phase 2) — formal spec for review
**Last updated:** 2026-11-26

The **ProfileDocument** is the single canonical, versioned, structured developer profile. It is the source of truth for both README Studio (Markdown renderer) and Portfolio Studio (Web renderer). This document defines its formal TypeScript + Zod specification.

> TypeScript/Zod examples inside this Markdown specification are permitted (architecture design only — not application code).

---

## 1. Design goals

1. **Single source of truth.** Both studios read and write this one document.
2. **Versioned.** `schemaVersion` enables backward-compatible evolution.
3. **Stable identifiers.** Every repeatable item has a stable `id` so ordering, references, and provenance survive edits.
4. **Provenance.** Imported vs authored is explicit at the field level.
5. **Renderer-agnostic.** Shared fields are common; renderer-specific customization is namespaced (`readme`, `portfolio`).
6. **Validated.** Zod enforces the schema; invalid documents are rejected at the service boundary.
7. **Deterministic rendering.** The document is a pure input to renderers.

## 2. Document envelope

```ts
const ProfileDocument = z.object({
  schemaVersion: z.literal(1),          // current
  id: z.string().uuid(),               // document instance id (stable)
  identity: z.object({ ... }).describe('Who the developer is'),
  headline: z.string().max(200).optional().default(''),
  biography: z.string().max(5000).optional().default(''),
  socialLinks: z.array(SocialLink).default([]),
  skills: z.array(Skill).default([]),
  projects: z.array(ProjectEntry).default([]),
  experience: z.array(ExperienceEntry).default([]),
  education: z.array(EducationEntry).default([]),
  certifications: z.array(CertificationEntry).default([]),
  customSections: z.array(CustomSection).default([]),
  readme: z.object({ ... }).default({}),    // README renderer config
  portfolio: z.object({ ... }).default({}), // Portfolio renderer config
  theme: z.object({ ... }).default({}),     // shared theme settings
  sectionOrder: z.array(z.string()).default([]), // ordered section keys
  visibility: z.object({ ... }).default({}), // per-section visibility
});
```

## 3. Field-by-field specification

### 3.1 Identity (required)
Who the developer is. Shared by both renderers.

```ts
const Identity = z.object({
  displayName: z.string().min(1).max(100),
  handle: z.string().min(1).max(64).regex(/^[a-zA-Z0-9._-]+$/),
  email: z.string().email().optional().or(z.literal('')), // optional, may be private
  location: z.string().max(120).optional().default(''),
  pronouns: z.string().max(60).optional().default(''),
  avatarUrl: z.string().url().max(500).optional().or(z.literal('')),
  website: z.string().url().max(500).optional().or(z.literal('')),
});
```

- **Required:** `displayName`, `handle`.
- **Optional:** `email`, `location`, `pronouns`, `avatarUrl`, `website`.
- **Validation:** `handle` restricted to a safe charset; URLs must be valid `http(s)` (see security: malicious URLs).

### 3.2 Headline & Biography
- `headline`: short tagline (optional).
- `biography`: longer text (optional). **Validation:** length caps; renderers must sanitize as Markdown/plain text per output.

### 3.3 Social links
```ts
const SocialLink = z.object({
  id: z.string().min(1).max(64),         // stable id, e.g. 'github'
  type: z.enum(['github','linkedin','twitter','x','devto','website','mastodon','bluesky','other']),
  url: z.string().url().max(500),
  label: z.string().max(60).optional(),
  visibility: z.enum(['public','private']).default('public'),
});
```
- **Required:** `id`, `type`, `url`. `id` unique within the array.
- `visibility: 'private'` fields must never appear in public portfolio responses (see security).

### 3.4 Skills
```ts
const Skill = z.object({
  id: z.string().min(1).max(64),
  name: z.string().min(1).max(100),
  level: z.enum(['beginner','intermediate','advanced','expert']).optional(),
  category: z.string().max(60).optional(),
});
```

### 3.5 Projects
```ts
const ProjectEntry = z.object({
  id: z.string().min(1).max(64),
  title: z.string().min(1).max(200),
  description: z.string().max(2000).optional().default(''),
  url: z.string().url().max(500).optional().or(z.literal('')),
  repoFullName: z.string().max(200).optional(), // e.g. 'owner/repo'
  technologies: z.array(z.string().max(60)).default([]),
  highlight: z.boolean().default(false),
  source: z.enum(['authored','imported']).default('authored'),
  visibility: z.enum(['public','private']).default('public'),
});
```
- `source` distinguishes imported repo facts from authored descriptions.
- Linked to `project_entries` table; `repoFullName` optionally references `github_repositories.full_name`.

### 3.6 Experience
```ts
const ExperienceEntry = z.object({
  id: z.string().min(1).max(64),
  role: z.string().min(1).max(120),
  organization: z.string().min(1).max(120),
  startDate: z.string().regex(/^\d{4}-\d{2}$/).optional(), // YYYY-MM
  endDate: z.string().regex(/^\d{4}-\d{2}$/).optional(),
  current: z.boolean().default(false),
  summary: z.string().max(2000).optional().default(''),
  visibility: z.enum(['public','private']).default('public'),
});
```

### 3.7 Education
```ts
const EducationEntry = z.object({
  id: z.string().min(1).max(64),
  institution: z.string().min(1).max(160),
  degree: z.string().max(120).optional(),
  field: z.string().max(120).optional(),
  startDate: z.string().regex(/^\d{4}-\d{2}$/).optional(),
  endDate: z.string().regex(/^\d{4}-\d{2}$/).optional(),
  visibility: z.enum(['public','private']).default('public'),
});
```

### 3.8 Certifications
```ts
const CertificationEntry = z.object({
  id: z.string().min(1).max(64),
  name: z.string().min(1).max(160),
  issuer: z.string().max(160),
  year: z.string().regex(/^\d{4}$/).optional(),
  credentialUrl: z.string().url().max(500).optional().or(z.literal('')),
  visibility: z.enum(['public','private']).default('public'),
});
```

### 3.9 Custom sections
```ts
const CustomSection = z.object({
  id: z.string().min(1).max(64),
  key: z.string().min(1).max(64).regex(/^[a-z0-9-]+$/), // stable, kebab
  title: z.string().min(1).max(120),
  kind: z.enum(['heading','paragraph','list','links','media']).default('paragraph'),
  content: z.union([
    z.string().max(10000),
    z.array(z.string().max(2000)),
    z.array(z.object({ label: z.string().max(120), url: z.string().url().max(500) })),
  ]),
  visibility: z.enum(['public','private']).default('public'),
});
```

### 3.10 README configuration (renderer-specific)
```ts
const ReadmeConfig = z.object({
  templateId: z.string().uuid().optional(),     // resolves to templates(kind='readme')
  includeBadges: z.boolean().default(true),
  includeStars: z.boolean().default(true),
  includeTopLanguages: z.boolean().default(true),
  layout: z.enum(['classic','compact','minimal']).default('classic'),
  showSocialLinks: z.boolean().default(true),
  showProjects: z.boolean().default(true),
  showExperience: z.boolean().default(true),
  showEducation: z.boolean().default(true),
  showCertifications: z.boolean().default(true),
  footer: z.string().max(500).optional().default(''),
});
```
- **Independent** from portfolio config. Changing README config must not affect portfolio output.

### 3.11 Portfolio configuration (renderer-specific)
```ts
const PortfolioConfig = z.object({
  templateId: z.string().uuid().optional(),     // resolves to templates(kind='portfolio')
  layout: z.enum(['classic','sidebar','grid','minimal','full']).default('classic'),
  heroEnabled: z.boolean().default(true),
  showSkills: z.boolean().default(true),
  showProjects: z.boolean().default(true),
  showExperience: z.boolean().default(true),
  showEducation: z.boolean().default(true),
  showCertifications: z.boolean().default(true),
  showContact: z.boolean().default(true),
  navOrder: z.array(z.string()).default([]),
});
```
- **Independent** from README config.

### 3.12 Theme settings (shared)
```ts
const Theme = z.object({
  primaryColor: z.string().max(20).default('#0f172a'),
  accentColor: z.string().max(20).default('#2563eb'),
  fontFamily: z.string().max(60).default('system-ui'),
  darkMode: z.enum(['light','dark','system']).default('system'),
});
```
- Shared across renderers; renderers map theme to their own output format.

### 3.13 Section ordering
```ts
const sectionOrder: z.array(z.string()).default([])
```
- Array of section keys (`'headline' | 'biography' | 'skills' | 'projects' | 'experience' | 'education' | 'certifications' | custom section keys`).
- Drives renderer output order. Unknown keys are rejected by the renderer (template compatibility).

### 3.14 Visibility settings
- Per-section/per-item `visibility: 'public' | 'private'`.
- **Public portfolio responses must omit private items.** README export is user-controlled (the user explicitly exports), but private items should be flagged/omitted unless the user explicitly includes them — see security.

## 4. Stable identifiers

- `id` on every repeatable item is **stable** across revisions unless the user deletes and re-adds.
- `identity` has no `id` (singleton).
- Custom section `key` is stable and kebab-cased; used in `sectionOrder`.
- Document `id` is stable for the profile's document instance.

## 5. Validation rules (summary)

| Rule | Where enforced |
|---|---|
| Required identity fields present | Zod |
| `handle` charset | Zod regex |
| URL fields valid `http(s)` | Zod + sanitizer |
| Date formats `YYYY-MM` / `YYYY` | Zod regex |
| Item `id` unique within array | Zod refine |
| `sectionOrder` keys known | renderer/template compatibility |
| Private items never in public output | service + delivery layer |
| Length caps on all text | Zod `.max()` |
| `schemaVersion` supported | service |

## 6. Backward compatibility & schema migrations

- `schemaVersion` is a **major** version integer. Minor/backward-compatible additions increment a separate `minor` concept handled by additive Zod fields with defaults.
- **Additive changes** (new optional fields, new enum values with defaults) are backward compatible: old documents validate against new schema.
- **Breaking changes** (removing/renaming fields, changing types, removing enum values) require a **new major `schemaVersion`** and a migration:
  1. Add a migration function that transforms `schemaVersion = N` documents to `schemaVersion = N+1`.
  2. Keep both versions valid until all stored revisions are migrated or safely dropped.
  3. Renderers declare a supported `schema_version` range; out-of-range documents are rejected with `INCOMPATIBLE_SCHEMA`.
- Migrations run server-side over revision rows (a background job), and are idempotent.
- `profiles.schema_version` tracks the **current** document schema; revisions keep their own `schemaVersion` inside the document.

```ts
interface SchemaMigration {
  from: number;
  to: number;
  transform: (doc: unknown) => unknown;
}
// migrations: SchemaMigration[] applied in order, idempotently.
```

## 7. Imported-data resolution

Imported GitHub facts are resolved against user-curated data with these rules:

1. **Never overwrite authored fields.** If a field is already authored (non-empty and not flagged `source: 'imported'`), keep it.
2. **Tag provenance.** Imported fields carry `source: 'imported'` (or a provenance map).
3. **Surface conflicts.** When imported value differs from an authored value, present both and let the user choose; do not silently pick.
4. **Repo facts populate projects.** Discovered repos can seed `projects` entries with `source: 'imported'` and `repoFullName`; the user's authored `description`/`title` are preserved.
5. **Snapshots are caches.** `github_snapshots` hold raw data; resolution produces a **draft revision** only after user review (or explicitly on auto-import).
6. **Refresh behavior.** A refresh re-fetches the snapshot and re-runs resolution, but again never overwrites authored content; only imported fields update.

```ts
interface ImportResolution {
  document: ProfileDocument;      // resulting document
  conflicts: Array<{ path: string; authored: unknown; imported: unknown }>;
  applied: string[];              // paths filled from import
}
```

## 8. Examples

### 8.1 Valid minimal document
```json
{
  "schemaVersion": 1,
  "id": "0f8c1e2a-1234-4abc-9def-abcdef012345",
  "identity": { "displayName": "Ada Lovelace", "handle": "ada" },
  "headline": "Full-stack engineer",
  "biography": "",
  "socialLinks": [],
  "skills": [],
  "projects": [],
  "experience": [],
  "education": [],
  "certifications": [],
  "customSections": [],
  "readme": { "templateId": null, "layout": "classic" },
  "portfolio": { "templateId": null, "layout": "classic" },
  "theme": { "primaryColor": "#0f172a" },
  "sectionOrder": ["headline", "skills", "projects"],
  "visibility": {}
}
```

### 8.2 Valid richer document (excerpt)
```json
{
  "schemaVersion": 1,
  "id": "0f8c1e2a-1234-4abc-9def-abcdef012345",
  "identity": {
    "displayName": "Ada Lovelace",
    "handle": "ada",
    "location": "Bengaluru, IN",
    "website": "https://example.com"
  },
  "headline": "Full-stack engineer & open-source contributor",
  "biography": "Builds developer tooling.",
  "socialLinks": [
    { "id": "github", "type": "github", "url": "https://github.com/ada", "visibility": "public" }
  ],
  "skills": [
    { "id": "s1", "name": "TypeScript", "level": "expert" }
  ],
  "projects": [
    { "id": "p1", "title": "GitFront", "repoFullName": "ada/gitfront", "source": "imported", "visibility": "public" }
  ],
  "experience": [],
  "education": [],
  "certifications": [],
  "customSections": [
    { "id": "c1", "key": "talks", "title": "Talks", "kind": "list", "content": ["DEVcon 2025"], "visibility": "public" }
  ],
  "readme": { "templateId": null, "layout": "compact", "showExperience": true },
  "portfolio": { "templateId": null, "layout": "sidebar", "showExperience": true },
  "theme": { "primaryColor": "#0f172a", "darkMode": "system" },
  "sectionOrder": ["headline", "biography", "skills", "projects", "experience", "talks"],
  "visibility": {}
}
```

### 8.3 Invalid examples (must be rejected)

```json
// missing required identity
{ "schemaVersion": 1, "id": "...", "identity": {} }
// invalid handle charset
{ "identity": { "displayName": "Ada", "handle": "ada name!" } }
// bad URL
{ "socialLinks": [{ "id": "github", "type": "github", "url": "javascript:alert(1)" }] }
// bad date format
{ "experience": [{ "id": "e1", "role": "Eng", "organization": "X", "startDate": "2025/13" }] }
// duplicate item id
{ "skills": [ { "id": "s1", "name": "A" }, { "id": "s1", "name": "B" } ] }
// unknown schema version
{ "schemaVersion": 99, "id": "...", "identity": { "displayName": "A", "handle": "a" } }
```

## 9. Consistency with the data model

- The JSONB `document` in `profile_revisions` conforms to this spec.
- `profiles.schema_version` reflects the current schema version.
- `project_entries` maps to `projects` (authored rows have `source='authored'`, imported have `source='imported'`).
- `readme.templateId` / `portfolio.templateId` resolve to `templates` rows by `kind`.
- `sectionOrder` and `visibility` are consumed by renderers; private visibility enforced at the delivery boundary.
