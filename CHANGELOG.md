# Changelog

All notable changes to the AOD Standard and its public website are recorded in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased] — FEAT-0013 Phase D

Feature: **FEAT-0013 — Media Pipeline & Publishing Assets**
Status: Publish pipeline projects hero, hero srcset, social image, and gallery
(with alt text, captions, credits, and licensing) into `projects_public`.

### Added — Phase D

- `publish_cms_project` now resolves ready derivatives for the draft's Hero,
  Social, and Gallery media and projects them atomically into
  `projects_public` (`primary_image_url`, `primary_image_alt`, `hero_srcset`,
  `hero_alt`, `hero_decorative`, `social_image_url`, `social_image_alt`,
  `gallery` jsonb). Immutable revisions record `gallery_media_ids`, and
  `project_media_references` rows are written for each Hero/Social/Gallery
  media so referenced originals cannot be deleted or downgraded while a
  published revision uses them.
- Publish refuses if any referenced Hero/Social/Gallery media is not in
  `ready` state — no dangling public references possible.
- `unpublish_cms_project` / `archive_cms_project` continue to hide the row
  via `is_public_visible = false`; media references are preserved so the
  published revision stays reproducible.
- `public.storage_config` singleton + `public.public_media_url(path)` helper
  used server-side to build absolute public URLs without hardcoding.
- Gallery ordering — Move Up / Move Down controls in the Media tab details
  panel. Order persists via `gallery_media_ids` array position and is
  respected on publish.
- Retry processing action for media in the `failed` state (Media tab details
  panel) — reuses the existing private original, no re-upload needed. Now
  callable by editors as well as admins.

## [Prior] — FEAT-0013 Phase C

Status: Media tab editor UI (upload, browse, describe, assign roles, archive).

### Added — Phase C

- Media tab in the project editor: drag-and-drop uploads, grid/list browser, processing-state indicators, and details side panel with alt text, decorative flag, caption, credit, license, and role assignment (Hero / Social / Gallery).
- `listProjectMedia`, `updateMediaMetadata`, `archiveMedia`, `restoreArchivedMedia`, and `listMediaUsage` server functions (`src/lib/admin/media.functions.ts`).
- `gallery_media_ids` added to editable `ProjectContent` — role toggles autosave via the existing draft autosave.
- RLS policies: editors and admins can read and update descriptive metadata on `project_media`. Checksum immutability and referenced-ready-state protection triggers remain in force.

## [Prior] — FEAT-0013 Phase B

Status: Runtime pipeline (upload → finalize → derivatives). Editor UI and publish integration deferred to later phases.


### Added

- `project-media-public` flipped to a truly public bucket with world-readable SELECT policy and no client write policies (service role only, immutable checksum-versioned paths).
- Storage RLS on `project-media-private` restricting reads and writes to editors/admins via `can_edit_projects`.
- Admin-callable RPCs (SECURITY DEFINER, EXECUTE revoked from `anon`/`public`):
  `mint_media_upload_ticket`, `finalize_media_upload`, `mark_media_ready`, `mark_media_failed`.
- Backend-only RPC `record_media_derivative` (service role EXECUTE only).
- Immutable-checksum trigger on `project_media` — once set, `checksum_sha256` cannot change.
- Supabase edge function `process-media` (Deno + ImageScript) that verifies checksum, sniffs magic bytes, and generates JPEG variants (`thumb 320w`, `card 640w`, `hero 1440w`, `hero2x 2560w`) at immutable path `derivatives/{media_id}/{variant}-{checksum12}.jpg` with `Cache-Control: 31536000, immutable`. Timing-safe service-role bearer auth.
- Server functions `createUploadTicket`, `finalizeMediaUpload`, `retryMediaProcessing`, `getMediaPipelineHealth` (`src/lib/admin/media.functions.ts`).
- Hidden admin verification surface at `/admin/media/health` (not linked from nav).
- Idempotent dedupe on finalize: same `(project_id, sha256)` archives the new pending row and returns the existing ready `media_id`.

### Security

- Public bucket writes require the service role; RLS on `storage.objects` denies anon/authenticated writes to `project-media-public`.
- All new SECURITY DEFINER functions perform explicit `can_edit_projects` (or admin) checks and revoke EXECUTE from `PUBLIC`/`anon`; `record_media_derivative` is service-role-only.
- Edge function requires `Authorization: Bearer <service_role>` via timing-safe compare; rejects otherwise with 401.
- Google Authentication remains the primary supported provider; `trgriffith@brambletech.com` is seeded into `admin_allowlist` (idempotent). Bootstrap admin flow untouched.

---


## [Unreleased] — FEAT-0012 Phase 3C

Feature: **FEAT-0012 — Project Administration, Authentication & Content Management**
Status: Publishing pipeline & public-site integration

### Added

- Atomic CMS publishing projector (`publish_cms_project`, `unpublish_cms_project`, `archive_cms_project`) that mints an immutable revision, projects it to `projects_public`, updates editorial status, and writes an audit event as a single transaction. Public read model reads exclusively from the immutable revision content, never from the mutable draft.
- Public projection metadata: `is_public_visible`, `origin_type` (`cms` vs future community mirror), `projected_at`, and `cms_revision_id` so drift is computed against the exact projected revision.
- Database-enforced cross-table slug uniqueness between community submissions and CMS projects (triggers on both `projects` and `projects_public`).
- "Live on site" link in the editor action bar for published projects.
- CMS-published projects intermix with community submissions in the `Built with AOD` grid, detail routes, sitemap, and slug listings, ordered by `published_at`.

### Changed

- Unpublish and archive preserve the last published revision and hide the public read-model row via `is_public_visible = false` instead of deleting it. Re-publishing restores visibility.
- Public projection SELECT policy filters on `is_public_visible = true`; editors/admins retain full read access for admin surfaces.



## [Website 1.1.0] — 2026-07-16

Tag: `website-v1.1.0`
Feature: **FEAT-0010 — Built with AOD Community Showcase**
Status: Production Ready · Release Candidate APPROVED

### Added

- **Built with AOD** community showcase at `/built-with-aod` with anonymous project submission, moderation workflow, and public detail pages.
- Anonymous submission form with Turnstile challenge, per-image alt text capture, conditional validation for private projects, and friendly user-facing error messages.
- Moderation dashboard (`/built-with-aod/moderation`) with Pending → Approved → Published lifecycle, project verification status (internal), and reviewer notes.
- Dual-bucket media pipeline: `project-media-private` for pending submissions, `project-media-public` mirror on publish.
- Server-rendered detail pages with hero imagery, gallery, project highlights, and "The Story" section.
- Dynamic sitemap entries for published showcase projects.
- Analytics events: `Gallery Viewed` (landing impression), `Gallery Media Opened` (detail interaction), submission funnel events.

### Security

- Anonymous Data API access restricted to explicit column-level `GRANT`s on `projects` and `project_media`. Submitter PII, IP hash, moderator notes, review instructions, terms acceptance, verification status, and private storage paths are unreachable from the public Data API.
- Row-scoped SELECT policies for `anon` and `authenticated` (`status = 'published'`); moderator and service_role paths unchanged.
- Private media paths never included in any public server function projection.

### Changed

- Verification status remains internal during MVP — not exposed in UI, API, JSON, SEO, or structured data.
- Published imagery always ships with meaningful alt text (submitter-provided or project name fallback).

### Stage Gate

- Security Review: PASS
- Accessibility Review: PASS
- Regression Review: PASS
- TypeScript: zero errors
- Public Data API surface verified via direct probe

---

## [Website 1.0.0] — 2026-07

Tag: `website-v1.0.0`

### Added

- Public AOD Standard website with dark-only "Modern Engineering Minimalism" design system, IBM Plex Sans + JetBrains Mono typography, and signature blinking green cursor.
- Static-first content pipeline sourcing `/content/**/*.md` through the build-time registry.
- Command palette search, intent-first "Start Here" onboarding, and clustered navigation.
- Machine-readable endpoints: `/llms.txt`, `/llms-full.txt`, `/manifest.json`, `/raw/<path>`, `/sitemap.xml`.
- SEO metadata and JSON-LD schemas across all routes.
- Plausible Analytics with custom event tracking, isolated in `src/lib/analytics.ts`.
- "Copy Bootstrap Prompt" flow on the Start Here page.

---

## [Standard 1.0.0] — 2026-07

Tag: `standard-v1.0.0`

Initial public release of the AOD Standard. See `content/VERSION.md` for the full inclusion list covering methodology, lifecycle workflows, governance artifact specifications, and operational documentation.
