# M1: Agent-first MVP (full task cards)

> **Goal:** every requirement in `docs/spec/SPEC.md` §4 works through **REST + CLI + MCP**, with a
> minimal web UI. Richard can run his pipeline for a week without PathPeer.
> **Precondition:** the M0 exit gate has passed. Every card inherits the template's default
> "Must not touch" list (`package.json`/lockfile, `packages/db/**`, `packages/core/src/contracts/**`,
> registries, other modules' folders) and the always-on Don'ts (no new deps, no snapshot tests, no
> invented fields).

## Waves (≤ 4 implementers at once)

| Wave | Cards | Mode |
|---|---|---|
| 0 | M1-00 shared projections + enum unification | serialised |
| 1 | M1-01 core parsers · M1-02 companies · M1-03 activities/tags/notes · M1-04 search/lookup/settings/summary | parallel |
| 2 | M1-05 jobs (write side) · M1-06 contacts (write side) | parallel |
| 3 | M1-07 jobs (list + board) · M1-08 contacts (list + board) · M1-09 capture · M1-10 links + outreach + follow-ups | parallel |
| 4 | M1-11 imports · M1-12 exports · M1-13 adapters A (jobs/companies/capture) · M1-14 adapters B (contacts/links/outreach/followups) | parallel |
| 5 | M1-15 adapters C (activities/tags/notes/search/settings/summary/imports/exports) · M1-16 web: jobs · M1-17 web: contacts + companies | parallel |
| 6 | M1-18 web: today/activity/search/settings | single |
| 7 | M1-19 golden path, parity, perf, seed | serialised |
| 8 | M1-20 M1 exit gate | orchestrator |

**Shared service signatures** (every card may rely on these; they're built in the named card):

| Function | Built in | Signature |
|---|---|---|
| `resolveCompany` | M1-02 | `(tx, ref: {id: string} \| {name: string; domain?: string}, ctx) => Promise<Company>` (creates if missing; dedupe: domain then name_norm) |
| `recordActivity` | M0-07 | see M0-07 |
| `applyTag` / `removeTag` | M1-03 | `(tx, {tag, entityType, entityId}, ctx) => Promise<void>` |
| `addNote` | M1-03 | `(tx, {entityType?, entityId?, bodyMd}, ctx) => Promise<Note>` |
| `createJob` | M1-05 | `(db, input: JobCreateInput, ctx, opts?: {source?: Source; sightingExternalId?: string; sightingRaw?: unknown}) => Promise<{job: JobDetail, created: boolean, possibleDuplicates: JobCard[]}>`. `source` is **not** a REST field: routes pass `manual`, capture passes `url`/`extension`, imports pass `import_*` |
| `moveJob` | M1-05 | `(db, id, status, ctx) => Promise<JobDetail>` |
| `upsertContact` | M1-06 | `(db, input: ContactUpsertInput, ctx) => Promise<{contact: ContactDetail, created: boolean}>` |
| `moveContact` | M1-06 | `(db, id, status, ctx) => Promise<ContactDetail>` |
| `patchContact` | M1-06 | `(db, id, patch: ContactPatchInput, ctx) => Promise<ContactDetail>` |
| `touchOutbound` / `touchInbound` | M1-06 | `(tx, contactId, at: number, ctx) => Promise<void>` (sets `last_outbound_at` / `last_inbound_at` + `thread_state='replied'` for inbound; records an `updated` activity) |
| `linkContactJob` | M1-10 | `(tx, {jobId, contactId, role}, ctx) => Promise<void>` (idempotent) |
| `logOutreach` | M1-10 | `(db, contactId, input: OutreachLogInput, ctx) => Promise<OutreachMessage>` |
| `getSettings` | M1-04 | `(db) => Promise<OwnerSettings>` (defaults merged) |

`ctx` = `{ now: () => number; actor: Actor; settings?: OwnerSettings }` from `shared/context.ts`.

---

### M1-00 · Shared projections + enum unification
**Wave** 0 · **Model** sonnet · **Depends on** M0 exit · **serialised**

**Goal.** One place for card/detail read projections, and the DB schema importing enums from
`@jm/core` instead of local copies.

**Read first.** data-model §1–§2 · api-rest `JobCard`/`ContactCard` definitions · CONVENTIONS §3.

**Owns.** `apps/server/src/shared/projections.ts`, `apps/server/src/shared/projections.test.ts`,
`packages/db/src/schema.ts` (**import lines only**).

**Contract.**
- `selectJobCards(db, where: SQL | undefined, opts: {orderBy, limit})`: returns `JobCard[]` (one
  joined query: jobs ⨝ companies ⟕ applications).
- `selectContactCards(db, where, opts)`: `ContactCard[]` (contacts ⟕ companies).
- `tagNamesFor(db, entityType, ids: string[]) → Map<id, string[]>`: one batched query.
- `jobCardById`, `contactCardById`.
- Schema: replace local enum arrays with imports from `@jm/core/enums`. `drizzle-kit generate`
  must report **no changes** (no new migration).

**Acceptance.** Tests with `createTestDb` + inserted synthetic rows: a job card has `companyName`,
`status: null` when untracked and the status when tracked · `tagNamesFor` returns names for 3 ids
in one call · the `drizzle-kit generate` output contains "No schema changes" (paste it in the report).

**Don'ts.** No business rules here, only reads. No new migration.

---

### M1-01 · Core parsers: JSON-LD JobPosting + readable text
**Wave** 1 · **Model** sonnet · **Depends on** M0 · **Parallel-safe with** M1-02, M1-03, M1-04

**Goal.** Pure functions that turn a job page's HTML into a normalised job draft.

**Read first.** api-rest §4 · data-model §2 `jobs` · schema.org `JobPosting` fields (title,
hiringOrganization.name/sameAs, jobLocation.address, jobLocationType, baseSalary, datePosted,
validThrough, description, identifier, url, applicantLocationRequirements).

**Owns.** `packages/core/src/parsers/**`, `packages/core/test-fixtures/**`.

**Contract.**
- `parseJobPostingJsonLd(html: string): ParsedJob | null`. Finds every
  `<script type="application/ld+json">`, tolerates arrays, `@graph`, and invalid JSON blocks (skip
  them), and picks the first `@type` `JobPosting`. It maps into `ParsedJob {title, companyName?,
  companyUrl?, locationText?, locationType, salaryMin?, salaryMax?, salaryCurrency?, salaryPeriod?,
  postedAt?, descriptionText?, atsJobId?, url?}`. HTML in `description` is converted to plain text
  (block tags → newlines, entities decoded).
- `extractReadableText(html): {title: string | null; text: string}`. It drops
  `script/style/noscript/nav/header/footer/svg` blocks, prefers `<main>` or `<article>` content,
  strips tags, decodes common entities (`&amp; &lt; &gt; &quot; &#39; &nbsp;` + numeric), collapses
  whitespace, and caps the result at 50,000 characters.
- `detectAts(url): {provider, jobId} | null` for `boards.greenhouse.io`/`job-boards.greenhouse.io`,
  `jobs.lever.co`, `jobs.ashbyhq.com`, `*.myworkdayjobs.com`.
- No dependencies beyond zod (regex/string based; the limitation is noted in a code comment).

**Decision table: `jobLocationType` / salary mapping**
| JSON-LD | Ours |
|---|---|
| `jobLocationType: "TELECOMMUTE"` | `remote` |
| no `jobLocationType`, has `jobLocation` | `onsite` (`hybrid` only if the text contains "hybrid") |
| `baseSalary.value.minValue/maxValue` + `unitText: YEAR/MONTH/HOUR` | min/max + period `year/month/hour` |
| `baseSalary.value.value` (single number) | min = max = value |
| anything missing | field omitted (never invented) |

**Acceptance.** `pnpm --filter @jm/core test -- parsers`. **Synthetic** fixtures (write them yourself;
companies `Example Corp`, domains `example.com`), shaped like Greenhouse, Lever and Ashby pages with
JSON-LD, one `@graph` page, one page with a broken JSON-LD block plus a valid one, and one page with
no JSON-LD. Assert exact extracted fields for each. `detectAts` table test.

**Don'ts.** No network. No copying real job pages as fixtures. No HTML parser libraries.

---

### M1-02 · Companies module
**Wave** 1 · **Model** sonnet · **Depends on** M1-00 · **Parallel-safe with** M1-01, M1-03, M1-04

**Goal.** REQ-CMP-01..03 via api-rest §2, plus `resolveCompany` for other modules.

**Owns.** `apps/server/src/modules/companies/**`.

**Contract.** Routes exactly as api-rest §2 (scopes as listed). Service exports `resolveCompany`
(signature above), `createCompany`, `getCompanyDetail`, `listCompanies`, `patchCompany`,
`deleteCompany`. Every mutation records an activity (`entity_type 'company'`).

**Acceptance.** HTTP tests:
- Create → 201. Same domain again → 200 `created:false` and the same id.
- Same name with different case/punctuation and no domain → the existing row is returned.
- List keyset pagination: 3 pages of 2 cover 5 rows without duplicates.
- Detail counts jobs and contacts (insert rows directly for the counts).
- Delete with jobs → 409.
- `resolveCompany` creates exactly once across two calls.

---

### M1-03 · Activities, tags, notes
**Wave** 1 · **Model** sonnet · **Depends on** M1-00 · **Parallel-safe with** M1-01, M1-02, M1-04

**Goal.** REQ-ACT-02, REQ-TAG-01, REQ-NOTE-01 (api-rest §9–§10), plus the `applyTag`/`addNote`
services used by imports.

**Owns.** `apps/server/src/modules/activities/**`, `modules/tags/**`, `modules/notes/**`,
`apps/server/src/shared/entity-exists.ts` (new: `entityExists(db, entityType, id) → boolean`).

**Contract.** Routes per api-rest §9–§10. `applyTag` creates a missing tag (by `name_norm`) and is
idempotent. Tag or note on a non-existent entity → 404. Notes support global (no entity). The
activity list is keyset newest-first with filters.

**Acceptance.** Tests:
- Applying the same tag twice → one tagging, one `tagged` activity.
- `Remote` and `remote ` map to the same tag.
- A note on a missing job → 404.
- The activity feed filtered by `entityType+entityId` returns only that entity's rows in time order.
- `GET /tags` usage counts are correct.

---

### M1-04 · Search, lookup, settings, summary
**Wave** 1 · **Model** sonnet · **Depends on** M1-00 · **Parallel-safe with** M1-01, M1-02, M1-03

**Goal.** REQ-SRC-01, REQ-JOB-09, REQ-SET-01, REQ-SUM-01 (api-rest §3 lookup, §11, §13).

**Owns.** `apps/server/src/modules/search/**`, `modules/settings/**`, `modules/summary/**`.

**Contract.**
- `GET /search`: FTS5 `MATCH` on `jobs_fts`/`contacts_fts` with `bm25()` order and `snippet()`
  (≤ 120 chars). Companies by `name_norm LIKE`. The user query is sanitised into FTS tokens: each
  word is double-quoted, and any `"` inside is removed, so user input can't inject FTS syntax.
- `GET /lookup?url=`: `normalizeUrl` → match `jobs.canonical_url`/`apply_url`.
  `normalizeLinkedinProfileUrl` → match `contacts.linkedin_url`.
- `GET/PATCH /settings`: `getSettings` merges stored JSON over the defaults (data-model §3).
  `PATCH` validates `Partial<OwnerSettings>` and records an activity.
- `GET /summary`: counts by status (jobs with tracking, untracked count, contacts), follow-ups
  due today (same definition as SPEC §3), week counts (applications moved to `applied` since the
  week start in the owner TZ, contacts moved to `contacted`, both from `activities`), today's
  outreach by channel vs caps.

**Acceptance.** Tests:
- The search query `c++ "dev"` doesn't throw and returns matches for "dev".
- Lookup finds a job saved with `?utm_source=x` when queried without it.
- Settings `PATCH {followUpDays: 5}` persists and the other defaults stay.
- Summary week counts use the owner TZ boundary (fake clock across a Monday 00:00 America/Toronto).

---

### M1-05 · Jobs module: write side
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
- `createJob` (signature above), in one transaction:
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

---

### M1-06 · Contacts module: write side
**Wave** 2 · **Model** sonnet · **Depends on** M1-02, M1-03, M1-04 · **Parallel-safe with** M1-05

**Goal.** REQ-CON-01, -03, -04, -05, REQ-FUP-02 snooze (upsert, get, patch, emails, delete, **move, snooze**) per api-rest §5.

**Owns.** `apps/server/src/modules/contacts/**`.

**Contract.** `POST /contacts` (upsert), `GET /contacts/:id`, `PATCH /contacts/:id`,
`POST/DELETE /contacts/:id/emails…`, `DELETE /contacts/:id`, `POST /contacts/:id/move`,
`POST /contacts/:id/snooze`. `moveContact` applies `contactTransition(prev, next, now, settings)`
(settings via `getSettings`, M1-04). Snooze sets `next_follow_up_at = startOfDay(today + days)` in the
owner TZ. Also exports `patchContact`, `touchOutbound`, `touchInbound` (signatures table above),
which the outreach module uses. `upsertContact`: normalise
`linkedinUrl`. If it exists, update **only profile fields** (full_name, headline, title,
location_text, about, avatar_url, company) and never CRM fields (data-model §2). Else insert with
`status` (default `saved`). Emails are lowercased, unique per contact, and `isPrimary` true clears
the others' primary flag.

**Acceptance.** Tests:
- Upsert twice with the same LinkedIn URL in different forms (`ca.linkedin.com/...?trk=`) → one
  row, `created:false` on the second.
- **The second upsert with `status:'closed'` doesn't change the status**, while the headline does
  update. This is the CRM-preserving test.
- PATCH `connectionState: 'connected'` records an `updated` activity with from/to.
- Adding the same email twice → one row. Setting primary flips the others.
- `saved→contacted` sets `last_outbound_at`, `next_follow_up_at = +followUpDays` and
  `connection_state=invite_pending`.
- `contacted→messaging` sets `thread_state=replied` and `connection_state=connected`.
- `completed` clears `next_follow_up_at`.
- Snooze 3 days lands on the start of that local day.

**Don'ts.** Don't implement list/board (M1-08).

---

### M1-07 · Jobs module: list + board
**Wave** 3 · **Model** sonnet · **Depends on** M1-05 · **Parallel-safe with** M1-08, M1-09, M1-10

**Goal.** REQ-JOB-04, -07 (`GET /jobs`, `GET /jobs/board`).

**Owns.** `apps/server/src/modules/jobs/**` (after M1-05 merged). Same folder, sequenced by wave.
Adds only list/board routes, their repo queries and tests.

**Contract.**
- List filters exactly as `JobListQuery`, with keyset pagination per sort. `q` joins `jobs_fts`.
  `status=untracked` means no application row.
- Board: 7 columns in enum order, each `{status, count, items ≤100 by updated_at desc}`.

**Acceptance.** Tests:
- The board's counts equal a direct SQL count per status.
- List `status=applied,interview&ratingMin=3&q=engineer` returns the expected ids.
- Paging with limit 2 over 5 rows returns 5 unique ids.
- `status=untracked` returns only jobs without tracking.
- Every sort option returns a stable order when timestamps tie (tie-break on id).

---

### M1-08 · Contacts module: list + board
**Wave** 3 · **Model** sonnet · **Depends on** M1-06 · **Parallel-safe with** M1-07, M1-09, M1-10

**Goal.** REQ-CON-02, -06 (`GET /contacts`, `GET /contacts/board`).

**Owns.** `apps/server/src/modules/contacts/**` (after M1-06 merged). Adds only list/board routes,
their repo queries and tests.

**Contract.** Filters per `ContactListQuery`. `followUpDue=YYYY-MM-DD` means `next_follow_up_at ≤
end of that day (owner TZ)` and status not in (completed, closed). `sort=followUp` orders by
`next_follow_up_at` ascending, nulls last.

**Acceptance.** Tests:
- The `followUpDue` filter respects the TZ day boundary.
- Board counts match SQL.
- Keyset paging is unique across pages.
- The `connectionState=invite_pending` filter returns only those contacts.

---

### M1-09 · Capture module + safe-fetch
**Wave** 3 · **Model** sonnet · **Depends on** M1-01, M1-05 · **Parallel-safe with** M1-07, M1-08, M1-10

**Goal.** REQ-JOB-02 and SEC-13 (`POST /capture/job`).

**Owns.** `apps/server/src/modules/capture/**`, `apps/server/src/shared/safe-fetch.ts`,
`apps/server/src/shared/safe-fetch.test.ts`.

**Contract.**
- `safeFetch(url, {timeoutMs=10000, maxBytes=2_000_000, maxRedirects=3, resolver?, fetchImpl?}) → Promise<{finalUrl: string; contentType: string; body: string}>`.
  - `resolver: (hostname: string) => Promise<Array<{address: string; family: 4 | 6}>>`. The default
    wraps `lookup(hostname, { all: true })` from `node:dns/promises` (allowed here because this is
    server code, not core).
  - `fetchImpl` defaults to global `fetch`, and tests inject a fake.
  - **Size rule:** if `Content-Length` > `maxBytes`, abort before reading. Otherwise read the body
    stream and abort as soon as the received bytes exceed `maxBytes`. Both cases throw
    `AppError('unprocessable', …, {reason: 'content_too_large'})`. Headers don't count.
  - Other failures throw `AppError('unprocessable', …, {reason})` with reason `blocked_address` |
    `bad_scheme` | `bad_port` | `too_many_redirects` | `bad_content_type` | `timeout` |
    `fetch_failed`.
  It rejects non-http(s) and ports other than 80/443. It rejects any resolved address that is
  loopback, private (10/8, 172.16/12, 192.168/16), link-local (169.254/16, fe80::/10), CGNAT
  (100.64/10), unique-local (fc00::/7), multicast or unspecified. It follows redirects manually
  (`redirect: 'manual'`) and re-validates each hop. Content-type must be `text/html` or
  `application/(ld+)json`. It stops reading past `maxBytes`.
- Capture decision table:

| Input | Behaviour |
|---|---|
| `url` on `*.linkedin.com`, no html/text | 422 `unprocessable`, `details.reason='linkedin_requires_client_capture'` |
| `url` only | `safeFetch` → html → steps below |
| `html` (± url) | no fetch. Parse the given html |
| `text` only | no fetch. Title = the `title` input (required, else 400). Description = text |
| html has JSON-LD JobPosting | `method:'jsonld'`, map ParsedJob → `JobCreateInput` |
| no JSON-LD | `method:'html'`: title per the **title rule** below, description = readable text, `warnings:['no_structured_data']` |
| parsed job has no `companyName` (the input has no company field) | company name = `registrableDomain(url)` when a url exists, else `"Unknown company"`, and `warnings += 'company_unknown'` |

**Title rule** (html method): take the `<title>` text and split it on ` - `, ` | `, ` – `, ` — `.
While there's more than one part, drop the **last** part if (case-insensitively, trimmed) it equals
the company name, the first label of the registrable domain, or one of `careers`, `jobs`,
`greenhouse`, `lever`, `ashby`, `workday`, `linkedin`, `indeed`. Join the remaining parts with ` - `.
An empty `<title>` → the `title` input, or 422 `unprocessable` with reason `title_missing`.

  Then `createJob(db, input, ctx, {source: ctx.actor === 'extension' ? 'extension' : 'url'})`, passing
  `track`, `status` and `tags` through unchanged.

**Acceptance.** Tests (no real network; `safeFetch` with a fake resolver and a fake fetch):
- `http://127.0.0.1/x`, `http://10.0.0.5`, `http://169.254.169.254/latest`, `http://[::1]/` and a
  public host that redirects to `http://192.168.1.1` → all rejected.
- Port 8080 → rejected.
- A JSON-LD fixture HTML (from M1-01) posted as `html` creates the job with the parsed fields.
- A LinkedIn URL alone → 422 with that reason.
- Capturing the same URL twice → `created:false`.
- `Content-Length: 3000000` → `content_too_large` without reading the body. A streamed body past
  2 MB → the same error.
- Title rule: `Senior Engineer - Example Corp` (company `Example Corp`) → `Senior Engineer`, and
  `Engineer | Careers | Example` → `Engineer`.
- Text-only capture without `title` → 400.

**Don'ts.** No real network in tests. No fetches that bypass `safeFetch`.

---

### M1-10 · Links, outreach log, follow-ups & reminders
**Wave** 3 · **Model** sonnet · **Depends on** M1-05, M1-06 · **Parallel-safe with** M1-07, M1-08, M1-09

**Goal.** REQ-LNK-01, REQ-OUT-01..03, REQ-FUP-01..02 reminders (api-rest §6, §7, §8).

**Owns.** `apps/server/src/modules/links/**`, `modules/outreach/**`, `modules/followups/**`.

**Contract.**
- Links: idempotent insert/delete on `job_contacts`, recording `linked`/`unlinked` activities on
  **both** the job and the contact.
- `logOutreach`: insert with status `sent` and `sent_at` (default now), set the contact's
  `last_outbound_at` (through a contacts service function `touchOutbound(tx, contactId, sentAt, ctx)` that
  **M1-06 must also export**). If format `f1` and the contact is `saved` → call `moveContact(…,'contacted')`
  (M1-06). If format `f6` → clear `next_follow_up_at`. Records
  `outreach_logged`.
- `PATCH /outreach/:id` replied/withdrawn per api-rest §7: replied → `touchInbound`. Withdrawn invite
  → `patchContact({connectionState:'invite_withdrawn'})`. F6 logged → `patchContact({nextFollowUpAt:null})`.
  Never write `contacts` directly.
- `GET /outreach/today`: counts by channel where `sent_at` is today in the owner TZ, plus the caps
  from settings.
- `GET /followups?due=`: contacts due + reminders due and not done + tracked jobs with a due
  `next_action_at`. `POST /reminders`, `POST /reminders/:id/done`.

**Acceptance.** Tests:
- Linking twice → one row.
- Logging an F1 on a saved contact moves it to `contacted` with `next_follow_up_at` set.
- Marking replied sets `thread_state=replied` + `last_inbound_at`.
- Today counts across a TZ day boundary.
- Follow-ups for a date include a snoozed-into-that-day contact and exclude completed ones.

---

### M1-11 · Imports: ResumeScorer + PathPeer
**Wave** 4 · **Model** sonnet · **Depends on** M1-05..M1-10, M0-01 (PathPeer shape) · **Parallel-safe with** M1-12, M1-13, M1-14

**Goal.** REQ-IMP-01, REQ-IMP-02 (api-rest §12) as idempotent background tasks.

**Read first.** data-model §6 · `docs/spec/pathpeer-export-shape.md` (from M0-01).

**Owns.** `apps/server/src/modules/imports/**` (including `tasks.ts` handlers),
`apps/server/src/modules/imports/fixtures/**` (**synthetic** bundles only).

**Contract.**
- `POST /imports/resumescorer {ledger}` and `POST /imports/pathpeer {bundle, dryRun?}` validate
  the shape with zod, enqueue task `import.resumescorer` / `import.pathpeer`, and return 202 with
  `taskId`.
- The handlers process in batches of 200 per transaction. Idempotency via `external_refs(source,
  external_id)`: an existing ref → update only allowed fields, never create. Result:
  `{created, updated, skipped, errors[]}` per entity. `dryRun` computes the same report and writes
  nothing.
- ResumeScorer: data-model §6 (untracked jobs + sightings).
- PathPeer: the data-model §6 mapping table, applied to the **real shape** from
  `pathpeer-export-shape.md`. **If that file doesn't exist, report BLOCKED. Don't guess the
  shape.** Activities imported with the original timestamps and `actor='import'`.

**Acceptance.** Tests with synthetic fixtures:
- Importing ResumeScorer twice → the second run creates 0.
- PathPeer dry-run reports the counts and the DB is unchanged.
- A real run → per-status counts match the fixture.
- Re-running → 0 created.
- A contact already present by LinkedIn URL is matched, not duplicated, and keeps its local status
  (CRM-preserving).

---

### M1-12 · Exports
**Wave** 4 · **Model** sonnet · **Depends on** wave 3 · **Parallel-safe with** M1-11, M1-13, M1-14

**Goal.** REQ-EXP-01 and SEC-21 (api-rest §12 exports).

**Owns.** `apps/server/src/modules/exports/**`.

**Contract.** `GET /exports/json` streams `{version, exportedAt, tables: {<name>: rows[]}}` for every
table except `sessions` and `api_tokens`, with `owner` without `password_hash`. `GET /exports/csv?entity=`
for jobs (job + company + tracking) and contacts (contact + company + primary email). CSV quoting
per RFC 4180, and cells starting with `= + - @ \t \r` are prefixed with `'` (formula-injection guard).
Records an `exported` activity.

**Acceptance.** Tests:
- The JSON has no `password_hash`, `token_hash` or `sessions` key.
- A CSV cell `=HYPERLINK(...)` is exported as `'=HYPERLINK(...)`.
- A cell with a comma and a quote is escaped correctly.
- The row counts match the DB.

---

### M1-13 · Adapters A: jobs, companies, capture (client + MCP + CLI)
**Wave** 4 · **Model** sonnet · **Depends on** M1-02, M1-05, M1-07, M1-09 · **Parallel-safe with** M1-11, M1-12, M1-14

**Goal.** REQ-AGT-01/02/03 for jobs, companies and capture.

**Read first.** `docs/spec/mcp.md` §1–§2 · `docs/spec/cli.md` §0, §2 · api-rest §0, §2–§4 · M0-09 (client
factory: `createClient`, per-module `(http) => ({…})` files, `JmApiError`) · M0-10 (`format.ts`: `ok`,
`fail`, `untrusted`) · M0-11 (`output.ts`: `printResult`, `exitFor`; command registration).

**Owns.** `packages/client/src/{jobs,companies,capture}.ts`, `packages/mcp/src/tools/{jobs,capture}.ts`,
`apps/cli/src/commands/{jobs,companies}.ts`, plus their test files. (`jm jobs capture` is a subcommand in
`commands/jobs.ts`. `commands/capture.ts` and `tools/companies.ts` stay as the empty M0 stubs, because M1
has no top-level `jm capture` command and no `companies_*` MCP tools: companies are reachable through
`search` and job/contact details.)

**Contract.**
- Client methods mirror every route (`jobs.list(query)`, `jobs.board()`, `jobs.get(id)`,
  `jobs.create(input)`, `jobs.patch(id, p)`, `jobs.move(id, s)`, `jobs.untrack(id)`,
  `jobs.remove(id)`, `companies.*`, `capture.job(input)`).
- MCP tools exactly as in mcp.md §2 for `jobs_*` (in `tools/jobs.ts`) and `jobs_capture_url` (in
  `tools/capture.ts`). `descriptionMd` is wrapped with `untrusted()` from `packages/mcp/src/format.ts`.
  The summary format follows mcp.md §1 (id/title/company/status).
- CLI `jm companies list|get|add|set` exactly as in cli.md §2.
- CLI commands exactly as in cli.md §2 for `jm jobs …`. The `capture` subcommand lives in `jobs`
  per cli.md (`jm jobs capture`).

**Acceptance.** Tests: each client method against a real app in-process · MCP `tools/call jobs_move`
changes the status (verified via a client `get`) · `jobs_get` output marks the description untrusted ·
CLI `jm jobs list --status applied --json` prints the REST body · `jm jobs move <6-char-prefix> applied`
works (the prefix is resolved by the API) · an ambiguous prefix → exit 4 with the candidates printed.

---

### M1-14 · Adapters B: contacts, links, outreach, follow-ups
**Wave** 4 · **Model** sonnet · **Depends on** M1-06, M1-08, M1-10 · **Parallel-safe with** M1-11, M1-12, M1-13

**Goal.** REQ-AGT-01/02/03 for contacts, links, outreach and follow-ups.

**Read first.** `docs/spec/mcp.md` §1–§2 · `docs/spec/cli.md` §0, §2 · api-rest §0, §5–§8 · M0-09, M0-10,
M0-11 (same helpers as M1-13).

**Owns.** `packages/client/src/{contacts,links,outreach,followups}.ts`,
`packages/mcp/src/tools/{contacts,links,outreach,followups}.ts`,
`apps/cli/src/commands/{contacts,links,outreach,followups}.ts` + tests.

**Contract.** mcp.md §2 tools `contacts_*`, `link_contact_job`, `outreach_*`, `followups_due`,
`contacts_snooze`. cli.md §2 commands `jm contacts …`, `jm link/unlink`, `jm outreach …`,
`jm followups`, `jm remind`. `about`, `headline` and outreach `body` are wrapped untrusted.

**Acceptance.** Tests: MCP `outreach_log` with f1 moves a saved contact to contacted (via
`contacts_get`) · `followups_due` returns that contact after the fake clock advances past
`followUpDays` · CLI `jm outreach log … --body-file` reads the file locally and posts its content ·
the snooze CLI prints the new date.

---

### M1-15 · Adapters C: activities, tags, notes, search, settings, summary, imports, exports
**Wave** 5 · **Model** sonnet · **Depends on** M1-03, M1-04, M1-11, M1-12 · **Parallel-safe with** M1-16, M1-17

**Read first.** `docs/spec/mcp.md` §1–§2 · `docs/spec/cli.md` §0, §2 · api-rest §0, §9–§13 · M0-09, M0-10,
M0-11.

**Owns.** `packages/client/src/{activities,tags,notes,search,settings,summary,imports,exports}.ts`,
`packages/mcp/src/tools/{activities,tags,notes,search,summary}.ts`,
`apps/cli/src/commands/{activities,tags,notes,search,settings,summary,imports,exports}.ts` + tests.

**Contract.** mcp.md tools `activity_recent`, `notes_add`, `tags_apply`, `search`, `pipeline_summary`.
cli.md commands `jm activity`, `jm note`, `jm tag/untag`, `jm search`, `jm settings get/set`,
`jm import resumescorer|pathpeer` (reads files locally, posts content, polls `/tasks/:id` every 1 s
until done, prints the report), `jm export json|csv`, and `jm help agents` (prints the AGENTS.md
"operating" subsection).

**Acceptance.** Tests: the import CLI with the synthetic ResumeScorer fixture prints the created
count, and a re-run prints 0 created · `pipeline_summary` returns counts matching the REST
`/summary` · `jm settings set followUpDays=5` persists · `jm export csv jobs --out <tmp>` writes a file.

---

### M1-16 · Web: jobs pages
**Wave** 5 · **Model** sonnet · **Depends on** M1-13 · **Parallel-safe with** M1-15, M1-17

**Owns.** `apps/web/src/pages/jobs*/**`, `apps/web/src/components/jobs/**`.

**Contract.** REQ-UI-01 for jobs.
- **Board:** 7 columns, cards (title, company, rating stars, location type). Drag and drop
  (`@dnd-kit/core`, the column-move pattern from ScoutLane `KanbanBoard.tsx`, ported) calls
  `jobs.move` with an optimistic update and rollback on error.
- **List:** a table with filters (status multi-select, q, tag, rating ≥) mirrored in the URL query
  string, and a "Load more" button (cursor).
- **Detail:** fields, a status select, a rating control, a flag toggle, linked contacts (link via a
  contact search box using `/search?types=contact`, with a role select), tags (add/remove), notes
  (add), and history (`/activities?entityType=job`). The description renders as **plain text**
  (SEC-09).
- An "Add job" dialog (manual form) and a "Capture URL" input.

**Acceptance.** `pnpm --filter @jm/web build` passes. A component test (renderToString with
mocked query data) shows that the board renders 7 columns with counts, and that the detail page
renders the description inside a `<pre>`, not as HTML. The page works in the smoke run (M1-19).

**Don'ts.** No new UI libraries. No `dangerouslySetInnerHTML`. No local persistence.

---

### M1-17 · Web: contacts + companies pages
**Wave** 5 · **Model** sonnet · **Depends on** M1-13 (companies client), M1-14 · **Parallel-safe with** M1-15, M1-16

**Owns.** `apps/web/src/pages/{contacts*,companies*}/**`, `apps/web/src/components/{contacts,companies}/**`.

**Contract.**
- **Contacts board and list:** same patterns as jobs, with filters for status, follow-up due,
  connection state, q and tag.
- **Contact detail:** profile fields, a status select, connection/thread state selects, a follow-up
  date with snooze buttons (3/7/14 days), emails (add, primary), linked jobs, an outreach log (list
  + "Log sent message" form with format/channel/body/hook), tags, notes and history.
- **Companies:** a table (name, tier, #jobs, #contacts, links) and a detail page (jobs and contacts
  tables).

**Acceptance.** Build passes. Component tests: the contact detail renders `about` as plain text, and
the outreach form's format select lists exactly the `OutreachFormat` enum values.

---

### M1-18 · Web: Today, Activity, Search, Settings
**Wave** 6 · **Model** sonnet · **Depends on** M1-15, M1-16, M1-17

**Owns.** `apps/web/src/pages/{today,activity,search,settings}/**`.

**Contract.**
- **Today (`/`):** summary tiles (pipeline counts, week vs targets, today's invites/DMs vs caps with
  a warning at ≥ cap) + follow-ups due (contacts with snooze/open actions, reminders with done,
  jobs with next actions).
- **Activity:** a feed with entity-type/actor filters.
- **Search:** a query box with grouped results.
- **Settings:** tokens (list, create with scopes and one-time display plus a copy button, revoke
  with confirmation) and the settings form (timezone, followUpDays, weekly targets, daily caps).

**Acceptance.** Build passes. A component test confirms that the token-create result is shown once
and cleared when the dialog closes (no storage APIs used).

---

### M1-19 · Golden path, parity, performance, seed
**Wave** 7 · **Model** sonnet · **Depends on** all previous M1 cards · **serialised**

**Owns.** `scripts/seed.ts`, `apps/server/test/golden-path.test.ts`, `apps/server/test/parity.test.ts`,
`apps/server/test/perf.test.ts`, `scripts/smoke.mjs` (extend it only).

**Contract.**
- `scripts/seed.ts`: synthetic demo data (20 companies, 150 jobs across statuses, 300 contacts,
  links, outreach, tags). Deterministic via a fixed seed.
- **Golden path** (through `@jm/client` in-process): create job → capture html → link contact → log
  F1 → follow-ups after +7 days → move to messaging → mark replied → summary numbers match.
- **Parity:** a table listing every M1 REST route and the CLI command + MCP tool covering it. The
  test asserts that every non-delete route has both, and that the MCP `tools/list` names equal the
  mcp.md §2 list exactly.
- **Perf:** seed 1k jobs / 10k contacts in a temp DB. p95 of 20 calls each is below the budgets in
  optimization §1 (list, board, search).
- `smoke.mjs` additionally runs `jm jobs add` and `jm followups --json` against the booted server.

**Acceptance.** `pnpm verify:full` is green including these tests. Perf numbers go in the card report
and in `docs/harness/gates.md` §5.

---

### M1-20 · M1 exit gate (orchestrator)
**Goal.** Gate G4 for M1, then hand the product to Richard for dogfooding.

**Does.**
1. `pnpm verify:full` green, with evidence.
2. `jm-security-auditor` full run (SEC-01…21). Fix or get explicit acceptance for every
   CRITICAL/HIGH.
3. The README gets the "For agents" section (AGENTS.md summary + connect commands) and the privacy
   + data-at-rest statements.
4. Runbook for Richard: `jm import pathpeer <export-dir> --dry-run`, then the real run, and compare
   counts with `counts.md`.
5. Drift review of CONVENTIONS, then handoff (`handoff.md` + a `docs/handoff/` session entry).
6. Propose enabling the routines (`docs/harness/routines.md`) once a GitHub remote exists (Richard
   decides).

## M1 exit criteria (each needs evidence)
1. Every REQ in SPEC §4 has at least one passing test, listed in a REQ → test table in the exit
   report.
2. The parity test and golden path pass. The perf budgets are met.
3. Richard's real PathPeer export imports with counts equal to `counts.md` (or an explicit deferral).
4. A real Claude Code session completes SPEC §6 workflows 1–4 over MCP.
5. The security audit has no open CRITICAL/HIGH.
6. Richard confirms a week of use without opening PathPeer, or lists what blocked it (which becomes
   the M2 wave-0 input).
