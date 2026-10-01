# M1-05 · Jobs module: write side

Status: open
Blocked by: M1-01, M1-02, M1-03
Wave: 2
Owner: agent
Model: sonnet

**Wave** 2 · **Model** sonnet · **Depends on** M1-01 (parsers aren't needed, but `detectAts` is), M1-02, M1-03 · **Parallel-safe with** M1-06

**Goal.** REQ-JOB-01, -03, -05, -06, -08 (create, dedupe, get, patch, delete, **move, untrack**) per api-rest §3.

**Read first.** data-model §1, §2 (`jobs`, `applications`, `job_sightings`), §4 (fingerprint + normalizeUrl), §5 (status rules) · api-rest §0, §3 · M0-06 (what `@jm/core` exports) · M1-00 (projections) · M1-02 (`resolveCompany`) · M1-03 (`applyTag`).

**Imports you'll use.** From `@jm/core` (all created by M0-06, which is merged before M1 starts): `normalizeUrl`, `normalizeName`, `jobFingerprint`, `applicationTransition`, `newId`, contracts `JobCreateInput`/`JobPatchInput`/`MoveInput`/`JobDetail`/`JobCard`. From `@jm/core/parsers`: `detectAts` (M1-01). From server: `shared/projections`, `shared/activity`, `shared/errors`, `modules/companies/service` (`resolveCompany`), `modules/tags/service` (`applyTag`).

**Owns.** `apps/server/src/modules/jobs/**`.

**Contract.**
- `POST /jobs`, `GET /jobs/:id`, `PATCH /jobs/:id`, `DELETE /jobs/:id`, `POST /jobs/:id/move`, `POST /jobs/:id/untrack`.
- `moveJob`: create the application if missing, apply `applicationTransition(prev, next, now)`, record
  `status_changed` (entity `application`) with the summary `Job "<title>" <from> → <to>`, and return the detail.
  Untrack deletes the application row and records `status_changed` with from=`<status>`, to=`null`, summary
  `Job "<title>" untracked`.
- `createJob` (signature in the **Shared service signatures** table in `docs/plans/M1-mvp-core.md`), in one transaction:
  1. `resolveCompany`.
  2. Compute, in this order:
     - `canonical_url = normalizeUrl(url)`.
     - **ATS:** if the input has **both** `atsProvider` and `atsJobId`, use them. If it has exactly
       one of them → 400 `bad_request`. If it has neither → `detectAts(url)` (may be null).
     - `fingerprint = jobFingerprint({atsProvider, atsJobId, canonicalUrl})` **after** the ATS
       resolution.
  3. If the fingerprint exists → return the existing job with `created:false`, and maybe add a
     sighting. **Sighting rule, exactly one branch applies:**
     - (a) `opts.sightingExternalId` given → `INSERT … ON CONFLICT (source, external_id) DO NOTHING`.
     - (b) otherwise → insert only if no sighting of that job has the same `source` and the same
       `source_url` (null-safe comparison).
  4. Else insert the job (`source = opts.source ?? 'manual'`) + one sighting (same source, `source_url = url`,
     `external_id = opts.sightingExternalId`, `raw = opts.sightingRaw`).
  5. If `track` (default true), insert the application with the `status` (default `saved`) and apply
     `applicationTransition(null, status)`.
  6. For each tag in `input.tags`: `applyTag(tx, {tag, entityType:'job', entityId}, ctx)` (it records its own
     `tagged` activity).
  7. Activities: one `created` (entity `job`, summary `Job "<title>" created`). If tracked, one
     `status_changed` (entity `application`, from `null`, to the status). A company created by
     `resolveCompany` records its own (and an existing company is returned unchanged, because
     `resolveCompany` never updates existing company fields). Call shape:
     `recordActivity(tx, {entityType:'job', entityId: job.id, type:'created', summary, actor: ctx.actor,
     occurredAt: ctx.now()})`.
  8. `possibleDuplicates` = up to 10 other jobs with the same `company_id` + `title_norm` (excluding
     itself), ordered `updated_at desc`, as `JobCard`s.
- `PATCH` accepts exactly the fields listed for `JobPatchInput` in api-rest §3 (the zod schema is
  `.strict()`, so a non-patchable field → 400). Job fields → `jobs`, tracking fields (`rating`,
  `flagged`, `nextActionAt`) → `applications`. Tracking fields on an untracked job →
  `AppError('conflict', 'job is not tracked; move it first')` (409).
- `getJobDetail(db, id)` builds `JobDetail` from: the job row, its company row, the application row or
  null (`tracking`), the linked contacts (`job_contacts` joined with `selectContactCards`, each with its
  `role`), `tagNamesFor(db,'job',[id])`, a notes count, and sightings (`seen_at desc`). A missing
  id → `AppError('not_found')`.
- `atsProvider` in input is validated against the `AtsProvider` enum (data-model §1).

**Acceptance.** HTTP tests:
- Create → 201 with tracking `saved` and one activity per entity.
- The same URL with tracking params → 200 `created:false`, same id, sightings count 2 when the source differs.
- Same company + title, different URL → a new job **and** `possibleDuplicates` contains the first.
- `track:false` → `tracking: null`.
- PATCH rating on an untracked job → 409.
- Delete removes the job and application (cascade) and records a `deleted` activity.
- Moving `saved→applied` sets `applied_at`. Moving `applied→interview→applied` keeps the original
  `applied_at`.
- `rejected` sets `closed_at`, and moving back to `applied` clears it.
- Moving an untracked job creates tracking.
- `atsProvider` without `atsJobId` → 400. A Greenhouse URL with no ATS input → `ats_provider='greenhouse'`
  (via `detectAts`), and the fingerprint starts `ats:`.
- PATCH with `canonicalUrl` (not patchable) → 400.

**Don'ts.** Never merge on `possibleDuplicates`. Don't implement list/board (M1-07).

## Comments
