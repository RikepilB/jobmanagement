# REST API v1: single source of truth

> The **only behaviour path**. CLI commands ([`cli.md`](cli.md)) and MCP tools ([`mcp.md`](mcp.md))
> are thin adapters over these routes through `packages/client`. Zod schemas named below live in
> `packages/core/src/contracts/<module>.ts`. Blueprint §6 is superseded by this file.

## 0. Conventions

| Topic | Rule |
|---|---|
| Base | `http://127.0.0.1:4747/api/v1` (port = `JM_PORT`, default 4747) |
| Format | JSON in and out, UTF-8. Request bodies are validated by `@hono/zod-validator`. Unknown fields → 400 (`.strict()`) |
| Success | Single: `200 {"data": T}` · Created: `201 {"data": T, "created": true}` · Existing returned by dedupe: `200 {"data": T, "created": false}` · List: `200 {"data": T[], "nextCursor": string \| null}` |
| Errors | `{"error": {"code": ErrorCode, "message": string, "details"?: unknown}}` |
| Error codes | `bad_request` 400 · `unauthorized` 401 · `forbidden` 403 · `not_found` 404 · `conflict` 409 · `unprocessable` 422 · `rate_limited` 429 (with `Retry-After`) · `internal` 500 |
| Auth | Cookie `jm_session` (web) **or** `Authorization: Bearer jm_…` (CLI, MCP, extension, AutoApply). See scopes below |
| Actor | Optional header `X-JM-Actor: web \| cli \| mcp \| extension`. Default is `web` for cookie auth and `cli` for bearer. Stored in `activities.actor` |
| Pagination | `?limit=1..200` (default 50) + `?cursor=<opaque>`. **Keyset** on `(sortKey, id)`, never OFFSET |
| Time | All timestamps are epoch **ms** (number). Date-only query params are `YYYY-MM-DD`, interpreted in the owner timezone |
| Ids | ULID strings. Every `:id` path param also accepts a **unique prefix ≥ 6 chars**, resolved server-side (`resolveEntityId`). Unknown → 404. Ambiguous → 409 `conflict` with `details.candidates: {id, label}[]` (max 10) |
| Size limits | 1 MB per body. `/imports/*` allows 50 MB (admin) |
| CSRF | Cookie-authenticated non-GET requests must pass Hono `csrf()` (Origin check). Bearer requests are exempt |

### Scopes
| Scope | Grants |
|---|---|
| `read` | every `GET` except exports |
| `write` | every mutation except admin routes |
| `capture` | `POST /capture/job`, `POST /contacts` (upsert), `GET /lookup` only (the Clipper's token) |
| `admin` | `/tokens*`, `/imports/*`, `/exports/*`, `PATCH /settings`, every `DELETE` |
A cookie session has all scopes. Missing scope → 403 `forbidden`.

## 1. System & auth (module `system`, `auth`)

| Method & path | Auth | Body / query | Response `data` |
|---|---|---|---|
| `GET /healthz` (no `/api/v1` prefix) | none | – | `{ok: true, version}` |
| `GET /system/status` | read | – | `SystemStatus {version, dbSizeBytes, counts: {jobs, contacts, companies}, now}` |
| `POST /auth/login` | none (rate-limited, SEC-02) | `LoginInput {password}` | `{ok: true}` + `Set-Cookie: jm_session=…; HttpOnly; SameSite=Lax; Path=/; Secure (when https)` |
| `POST /auth/logout` | cookie | – | `{ok: true}` (session revoked) |
| `GET /auth/me` | any | – | `{authType: 'session' \| 'token', scopes, tokenName?}` |
| `GET /tokens` | admin | – | `ApiTokenInfo[]` (never the hash) |
| `POST /tokens` | admin | `TokenCreateInput {name, scopes: Scope[], expiresInDays?: 1..3650}` | `ApiTokenCreated {info, token}`. **`token` is shown once** |
| `DELETE /tokens/:id` | admin | – | `{revoked: true}` |

Owner creation is **not** an HTTP route. `jm init` writes the owner row directly (no unauthenticated
setup endpoint, SEC-10).

## 2. Companies (`companies`)

| Method & path | Scope | Input | Output |
|---|---|---|---|
| `GET /companies` | read | `CompanyListQuery {q?, tier?, sort?: 'name'\|'updated', limit, cursor}` | `CompanyListItem[] {…Company, jobCount, contactCount}` |
| `POST /companies` | write | `CompanyCreateInput {name, domain?, websiteUrl?, careersUrl?, linkedinUrl?, tier?}` | `Company`. Dedupe (data-model §2) returns the existing row with `created:false` |
| `GET /companies/:id` | read | – | `CompanyDetail {company, jobs: JobCard[], contacts: ContactCard[], noteCount}` |
| `PATCH /companies/:id` | write | `CompanyPatchInput` (all optional) | `Company` |
| `DELETE /companies/:id` | admin | – | 409 `conflict` if jobs exist, else `{deleted: true}` |

## 3. Jobs & tracking (`jobs`)
The API is **job-centric**: tracking fields (status, rating, flagged, nextActionAt) are read and
written through the job. The `applications` table stays an internal detail.

| Method & path | Scope | Input | Output |
|---|---|---|---|
| `GET /jobs` | read | `JobListQuery {status?: comma list of ApplicationStatus or 'untracked', companyId?, locationType?, ratingMin?: 0..5, tag?, q?, postedWithinDays?, flagged?, sort?: 'updated'\|'posted'\|'rating'\|'created', limit, cursor}` | `JobListItem[]` = `JobCard` + `{postedAt, locationText, salary?, tagNames[], contactCount}` |
| `GET /jobs/board` | read | `{tag?, q?}` | `{columns: {status, count, items: JobCard[] (max 100, sort updated desc)}[]}` in enum order |
| `POST /jobs` | write | `JobCreateInput {title, company: {id} \| {name, domain?}, url?, applyUrl?, locationText?, locationType?, salaryMin?, salaryMax?, salaryCurrency?, salaryPeriod?, postedAt?, descriptionMd?, sponsorship?, atsProvider?, atsJobId?, track?: boolean (default true), status?: ApplicationStatus (default 'saved'), tags?: string[]}` | `{data: JobDetail, created, possibleDuplicates: JobCard[]}` |
| `GET /jobs/:id` | read | – | `JobDetail {job, company, tracking: Tracking \| null, contacts: {contact: ContactCard, role}[], tags: string[], noteCount, sightings: Sighting[]}` |
| `PATCH /jobs/:id` | write | `JobPatchInput` = optional `{title, applyUrl, locationText, locationType, salaryMin, salaryMax, salaryCurrency, salaryPeriod, postedAt, descriptionMd, sponsorship, seniority, isExpired}` (job fields) + optional `{rating, flagged, nextActionAt: number \| null}` (tracking fields). **Not patchable:** company, url/canonicalUrl, fingerprint, atsProvider/atsJobId, source (create a new job instead) | `JobDetail` |
| `POST /jobs/:id/move` | write | `MoveInput {status: ApplicationStatus}` | `JobDetail`. Creates tracking if missing and applies status rules (data-model §5) |
| `POST /jobs/:id/untrack` | write | – | `JobDetail` (tracking row deleted; activity kept) |
| `DELETE /jobs/:id` | admin | – | `{deleted: true}` |
| `GET /lookup` (implemented in module `search`) | read or capture | `{url}` | `{job: JobCard \| null, contact: ContactCard \| null}` matched by normalised URL (job `canonical_url`/`apply_url`, contact `linkedin_url`) |

`JobCard = {id, title, companyId, companyName, status: ApplicationStatus \| null, rating, flagged, locationType, updatedAt}`.

## 4. Capture (`capture`)

| Method & path | Scope | Input | Output |
|---|---|---|---|
| `POST /capture/job` | capture or write | `CaptureJobInput {url?, html?, text?, title?, track?: boolean (default true), status?: ApplicationStatus (default 'saved'), tags?: string[]}`. At least one of url/html/text. `track`/`status`/`tags` pass straight through to `createJob` | `{data: JobDetail, created, possibleDuplicates, extraction: {method: 'jsonld'\|'html'\|'text', warnings: string[]}}` |

Rules:
- **URL only:** the server fetches it through `safeFetch` (SEC-13: http/https only, ports 80/443,
  public IPs only (resolved and re-checked on every redirect), ≤3 redirects, ≤2 MB, 10 s timeout,
  `text/html` only).
- **LinkedIn URLs** (`*.linkedin.com`) with no `html`/`text` → 422 `unprocessable` with
  `details.reason = 'linkedin_requires_client_capture'` (ADR-007: no server fetches of LinkedIn).
- Extraction order: JSON-LD `JobPosting` (`parseJobPostingJsonLd` in core), then the `<title>` +
  visible `main`/`article` text (`extractReadableText` in core). If the company can't be determined,
  `warnings` includes `company_unknown` and the job goes under a company named from the URL host.
- Dedupe exactly as `POST /jobs`.

## 5. Contacts (`contacts`)

| Method & path | Scope | Input | Output |
|---|---|---|---|
| `GET /contacts` | read | `ContactListQuery {status?: comma list, companyId?, compatibility?, connectionState?, threadState?, followUpDue?: 'YYYY-MM-DD', tag?, q?, sort?: 'updated'\|'created'\|'name'\|'followUp', limit, cursor}` | `ContactListItem[]` = `ContactCard` + `{headline, linkedinUrl, nextFollowUpAt, lastOutboundAt, tagNames[]}` |
| `GET /contacts/board` | read | `{tag?, q?}` | `{columns: {status, count, items: ContactCard[] (max 100)}[]}` |
| `POST /contacts` | write or capture | `ContactUpsertInput {fullName, linkedinUrl?, company?: {id} \| {name, domain?}, headline?, title?, locationText?, about?, avatarUrl?, status?: ContactStatus (default 'saved'), compatibility?, source?}` | `{data: ContactDetail, created}`. Upsert by normalised `linkedinUrl` **preserving CRM fields** (data-model §2) |
| `GET /contacts/:id` | read | – | `ContactDetail {contact, company, emails, jobs: {job: JobCard, role}[], outreach: OutreachMessage[] (last 20), tags, noteCount}` |
| `PATCH /contacts/:id` | write | `ContactPatchInput` (profile fields + `rating, compatibility, connectionState, threadState, nextFollowUpAt, lastInboundAt`) | `ContactDetail` |
| `POST /contacts/:id/move` | write | `{status: ContactStatus}` | `ContactDetail` (status rules §5) |
| `POST /contacts/:id/snooze` | write | `{days: 1..60}` | `ContactDetail` (`next_follow_up_at = today + days`) |
| `POST /contacts/:id/emails` | write | `EmailInput {email, type?, status?, isPrimary?}` | `ContactEmail` (provider `manual`) |
| `DELETE /contacts/:id/emails/:emailId` | write | – | `{deleted: true}` |
| `DELETE /contacts/:id` | admin | – | `{deleted: true}` |

`ContactCard = {id, fullName, title, companyId, companyName, status, rating, compatibility, connectionState, threadState, updatedAt}`.

## 6. Links (`links`)

| Method & path | Scope | Input | Output |
|---|---|---|---|
| `POST /links` | write | `LinkInput {jobId, contactId, role}` | `{linked: true}` (idempotent) |
| `DELETE /links` | write | `LinkInput` | `{unlinked: true}` |

## 7. Outreach log (`outreach`)

| Method & path | Scope | Input | Output |
|---|---|---|---|
| `POST /contacts/:id/outreach` | write | `OutreachLogInput {format, channel, body, subject?, jobId?, hookUsed?, sentAt?: number (default now)}` | `OutreachMessage`. Side effects: `last_outbound_at = sentAt`, and if `format ∈ {f1}` and the contact is `saved` → move to `contacted` (status rules). If `format='f6'` → `next_follow_up_at = null` |
| `GET /contacts/:id/outreach` | read | – | `OutreachMessage[]` |
| `PATCH /outreach/:id` | write | `{status: 'replied' \| 'withdrawn'}` | `OutreachMessage`. `replied` → contact `thread_state='replied'`, `last_inbound_at=now`; `withdrawn` with `channel='linkedin_invite'` → `connection_state='invite_withdrawn'` |
| `GET /outreach/today` | read | – | `{invites: n, dms: n, emails: n, caps: {invites, dms}}` counted from `sent_at` today (owner TZ) |

Drafting (F1–F6 generation with the hook gate) is M2 and lives in `docs/plans/ROADMAP.md`.

## 8. Follow-ups & reminders (`followups`)

| Method & path | Scope | Input | Output |
|---|---|---|---|
| `GET /followups` | read | `{due?: 'YYYY-MM-DD' (default today)}` | `{contacts: ContactListItem[] (next_follow_up_at ≤ end of due day, status not completed/closed), reminders: Reminder[] (due, not done), jobs: JobCard[] (next_action_at due)}` |
| `POST /reminders` | write | `ReminderInput {entityType, entityId, dueAt, kind, note?}` | `Reminder` |
| `POST /reminders/:id/done` | write | – | `Reminder` |

## 9. Activities (`activities`)

| Method & path | Scope | Input | Output |
|---|---|---|---|
| `GET /activities` | read | `ActivityQuery {entityType?, entityId?, type?, actor?, since?: number, limit, cursor}` (sorted newest first) | `Activity[]` |

## 10. Tags & notes (`tags`, `notes`)

| Method & path | Scope | Input | Output |
|---|---|---|---|
| `GET /tags` | read | – | `{id, name, color, usage: number}[]` |
| `POST /tags` | write | `{name, color?}` | `Tag` (idempotent by name_norm) |
| `POST /tags/apply` | write | `{tag: string (name, created if missing), entityType: 'job'\|'contact'\|'company', entityId}` | `{applied: true}` |
| `POST /tags/remove` | write | same | `{removed: true}` |
| `GET /notes` | read | `{entityType?, entityId?, limit, cursor}` | `Note[]` |
| `POST /notes` | write | `NoteInput {entityType?, entityId?, bodyMd}` | `Note` |
| `PATCH /notes/:id` | write | `{bodyMd}` | `Note` |
| `DELETE /notes/:id` | admin | – | `{deleted: true}` |

## 11. Search (`search`)

| `GET /search` | read | `{q (min 2 chars), types?: 'job,contact,company', limit ≤ 50}` | `{type, id, title, subtitle, snippet}[]` (FTS5 `bm25` rank for jobs/contacts, `LIKE` on name_norm for companies) |
|---|---|---|---|

## 12. Imports, exports, tasks (`imports`, `exports`)

| Method & path | Scope | Input | Output |
|---|---|---|---|
| `POST /imports/resumescorer` | admin | `{ledger: ResumeScorerLedger}` (the parsed jobs.json) | `202 {data: {taskId}}` |
| `POST /imports/pathpeer` | admin | `{bundle: PathPeerBundle, dryRun?: boolean}`. Shape fixed after the M0-01 export | `202 {data: {taskId}}` |
| `GET /tasks/:id` | read | – | `Task {id, type, status, attempts, lastError, result}`. Import result = `{created, updated, skipped, errors[]}` per entity |
| `GET /exports/json` | admin | – | `application/json` attachment: every table except `sessions`, `api_tokens`, `owner.password_hash` |
| `GET /exports/csv` | admin | `{entity: 'jobs' \| 'contacts'}` | `text/csv` attachment. Cells sanitised against formula injection (findleads `sanitizeCsvCell`) |

The server **never reads a filesystem path taken from a request**. The CLI reads files locally and
posts their content (SEC-12).

## 13. Settings & summary (`settings`, `summary`)

| Method & path | Scope | Input | Output |
|---|---|---|---|
| `GET /settings` | read | – | `OwnerSettings` |
| `PATCH /settings` | admin | `Partial<OwnerSettings>` | `OwnerSettings` |
| `GET /summary` | read | – | `Summary {jobs: Record<ApplicationStatus, n>, untrackedJobs, contacts: Record<ContactStatus, n>, followUpsDueToday: n, week: {applications: n, contactsContacted: n, targets}, today: {invites, dms, emails, caps}}` |

## 14. Rate limits (in-process token buckets, SEC-02)
| Route group | Limit |
|---|---|
| `POST /auth/login` | 5 per minute per IP, then 429. 10 failures in an hour lock login for 15 min |
| `/capture/*` | 30 per minute per token |
| `/imports/*` | 2 per minute |
| Everything else | 600 per minute per token/session |
