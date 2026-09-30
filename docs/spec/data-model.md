# Data model (SQLite, Drizzle): single source of truth

> The **only** place tables are defined. `packages/db/src/schema.ts` implements exactly this.
> A change here needs an ADR line in `docs/decisions.md` plus a new migration. Shipped
> migrations are never edited. Blueprint §5 is superseded by this file.

## 0. Conventions

| Rule | Detail |
|---|---|
| Tenancy | **One database file per owner** (ADR-016). There is no `user_id` column. Authorization = authenticated + token scope |
| IDs | `text` primary key, **ULID** generated in app code (`ulid()`), so ids sort by creation time |
| Timestamps | `integer` **epoch milliseconds UTC** (`integer({ mode: 'number' })`). Names end in `_at`. The UI renders in the owner timezone |
| Booleans | `integer({ mode: 'boolean' })` |
| JSON | `text({ mode: 'json' }).$type<T>()`, validated by the matching zod schema in `packages/core` |
| Enums | `text` + app-level zod enum + SQL `CHECK` in the migration. Values are lowercase snake_case |
| Deletes | Hard delete with `ON DELETE CASCADE` on children. The deletion itself is kept in `activities` |
| Pragmas (set on every connection) | `journal_mode=WAL`, `foreign_keys=ON`, `busy_timeout=5000`, `synchronous=NORMAL` |
| Money | Integer minor units (`salary_min` in whole currency units per period; no cents needed) |
| Normalised strings | `*_norm` columns = lowercase, trimmed, collapsed whitespace, punctuation stripped (`normalizeName()` in core) |

Milestone column: **M0** = created by the M0 schema migration (everything M1 needs). Later tables are
added by a serialised schema card at the start of their milestone.

## 1. Enums (packages/core/src/enums.ts)

| Enum | Values |
|---|---|
| `ApplicationStatus` | `saved`, `applied`, `networking`, `interview`, `offer`, `rejected`, `withdrawn` |
| `ContactStatus` | `unsaved`, `saved`, `contacted`, `messaging`, `stuck`, `completed`, `closed` |
| `LocationType` | `remote`, `hybrid`, `onsite`, `unknown` |
| `Sponsorship` | `yes`, `no`, `unknown` |
| `Compatibility` | `strong`, `moderate`, `weak` |
| `ConnectionState` | `none`, `invite_pending`, `connected`, `invite_withdrawn` |
| `ThreadState` | `none`, `awaiting_reply`, `replied` |
| `JobContactRole` | `recruiter`, `hiring_manager`, `teammate`, `referrer`, `other` |
| `OutreachFormat` | `f1`, `f2`, `f2q`, `f3`, `f4`, `f5`, `f6`, `other` |
| `OutreachChannel` | `linkedin_invite`, `linkedin_dm`, `email`, `other` |
| `OutreachStatus` | `draft`, `sent`, `replied`, `withdrawn` |
| `EmailType` | `work`, `personal`, `unknown` |
| `EmailStatus` | `verified`, `guessed`, `unknown`, `bounced` |
| `EntityType` | `job`, `application`, `company`, `contact`, `outreach`, `note`, `system` |
| `Actor` | `web`, `cli`, `mcp`, `extension`, `import`, `system` |
| `Source` | `manual`, `url`, `extension`, `import_pathpeer`, `import_resumescorer`, `discovery` |
| `TaskStatus` | `queued`, `running`, `done`, `failed` |
| `CompanyTier` | `a`, `b`, `c` |
| `AutoApplyState` | `none`, `proposed`, `approved`, `submitted`, `failed`, `skipped` |
| `AtsProvider` | `greenhouse`, `lever`, `ashby`, `workday`, `smartrecruiters`, `other` |

## 2. Tables

### owner (M0): exactly one row, `id = 'owner'`
| column | type | notes |
|---|---|---|
| id | text PK | always `'owner'` |
| display_name | text not null | |
| password_hash | text not null | argon2id (SEC-03) |
| settings | text json not null default `'{}'` | `OwnerSettings` (§3) |
| created_at / updated_at | integer not null | |

### sessions (M0)
| column | type | notes |
|---|---|---|
| id | text PK | **sha256 hex of the cookie secret**. The raw secret is never stored |
| created_at, expires_at | integer not null | lifetime 14 days, sliding on use |
| revoked_at | integer null | logout / revoke-all |
| user_agent | text null | |

### api_tokens (M0)
| column | type | notes |
|---|---|---|
| id | text PK | ULID |
| name | text not null unique | e.g. `claude-code`, `clipper` |
| prefix | text not null | first 8 chars after `jm_`, shown in lists |
| token_hash | text not null unique | sha256 hex of the full token (`jm_` + 32 random bytes base64url) |
| scopes | text json not null | array of `read` \| `write` \| `capture` \| `admin` |
| created_at | integer not null | |
| expires_at | integer null | null = no expiry (CLI default 365 days) |
| last_used_at, revoked_at | integer null | |

### companies (M0)
| column | type | notes |
|---|---|---|
| id | text PK | |
| name | text not null | |
| name_norm | text not null | index |
| domain | text null | lowercase registrable domain. **Unique where not null** |
| website_url, careers_url, linkedin_url, logo_url | text null | |
| tier | text null | `CompanyTier` |
| created_at, updated_at | integer not null | |

Dedupe order: `domain` exact, then `name_norm` exact. Never fuzzy-merge.

### jobs (M0): identity of a posting
| column | type | notes |
|---|---|---|
| id | text PK | |
| company_id | text not null → companies.id | cascade delete: **restrict** (delete jobs first) |
| title | text not null | |
| title_norm | text not null | |
| canonical_url | text null | normalised by `normalizeUrl()`. **Unique where not null** |
| apply_url | text null | |
| location_text | text null | |
| location_type | text not null default `'unknown'` | `LocationType` |
| salary_min, salary_max | integer null | |
| salary_currency | text null | ISO 4217 |
| salary_period | text null | `year` \| `month` \| `hour` |
| posted_at | integer null | |
| description_md | text null | **untrusted text** (SEC-14) |
| sponsorship | text not null default `'unknown'` | |
| seniority | text null | free text in M1 |
| source | text not null | `Source` |
| ats_provider | text null | `AtsProvider` |
| ats_job_id | text null | unique together with ats_provider where both not null |
| fingerprint | text not null unique | see §4 |
| is_expired | integer bool not null default 0 | |
| created_at, updated_at | integer not null | |

Indexes: `(company_id)`, `(posted_at)`, `(title_norm)`.

### job_sightings (M0): every time a source saw the posting
| column | type | notes |
|---|---|---|
| id | text PK | |
| job_id | text not null → jobs.id cascade | |
| source | text not null | `Source` or a discovery source name |
| source_url | text null | |
| external_id | text null | source's own id (e.g. ResumeScorer sha1) |
| raw | text json null | original record, for audit |
| seen_at | integer not null | |

Unique `(source, external_id)` where external_id not null.

### applications (M0): tracking state (0 or 1 per job)
| column | type | notes |
|---|---|---|
| id | text PK | |
| job_id | text not null unique → jobs.id cascade | |
| status | text not null default `'saved'` | `ApplicationStatus` |
| rating | integer not null default 0 | 0–5 |
| flagged | integer bool not null default 0 | |
| applied_at, interview_at, offer_at, closed_at | integer null | set by status rules (§5) |
| next_action_at | integer null | reminder date |
| autoapply_state | text not null default `'none'` | `AutoApplyState` (suite-map) |
| created_at, updated_at | integer not null | |

Indexes: `(status, updated_at)`, `(next_action_at)`.

### contacts (M0)
| column | type | notes |
|---|---|---|
| id | text PK | |
| company_id | text null → companies.id set null | |
| full_name | text not null | |
| full_name_norm | text not null | |
| headline, title, location_text, avatar_url | text null | |
| about | text null | **untrusted text** |
| linkedin_url | text null | normalised `https://www.linkedin.com/in/<slug>`. **Unique where not null** |
| status | text not null default `'saved'` | `ContactStatus` |
| rating | integer not null default 0 | 0–5 |
| compatibility | text null | `Compatibility` |
| connection_state | text not null default `'none'` | |
| thread_state | text not null default `'none'` | |
| last_outbound_at, last_inbound_at, next_follow_up_at | integer null | |
| source | text not null | `Source` |
| created_at, updated_at | integer not null | |

Indexes: `(status, updated_at)`, `(company_id)`, `(next_follow_up_at)`, `(connection_state)`.
**CRM-preserving upsert:** re-capturing a contact may update the profile fields (headline, title,
about, avatar, location, company) and **must never** change status, rating, compatibility,
connection/thread state, `*_at` outreach fields or notes.

### contact_emails (M0)
| column | type | notes |
|---|---|---|
| id | text PK | |
| contact_id | text not null → contacts.id cascade | |
| email | text not null | lowercase |
| type | text not null default `'unknown'` | `EmailType` |
| status | text not null default `'unknown'` | `EmailStatus` |
| provider | text not null default `'manual'` | `manual`, `pattern`, provider name (M4) |
| method | text null | e.g. `pattern:first.last` |
| confidence | real null | 0–1 |
| is_primary | integer bool not null default 0 | |
| checked_at, created_at | integer | |

Unique `(contact_id, email)`.

### job_contacts (M0)
`job_id` → jobs cascade, `contact_id` → contacts cascade, `role` (`JobContactRole`), `created_at`.
PK `(job_id, contact_id, role)`. Index `(contact_id)`.

### outreach_messages (M0)
| column | type | notes |
|---|---|---|
| id | text PK | |
| contact_id | text not null → contacts cascade | |
| job_id | text null → jobs set null | |
| format | text not null | `OutreachFormat` |
| channel | text not null | `OutreachChannel` |
| subject | text null | |
| body | text not null | |
| hook_used | text null | the verifiable hook (M2 gate) |
| status | text not null default `'sent'` | M1 logs sent messages. Drafts come in M2 |
| sent_at | integer null | |
| created_at, updated_at | integer not null | |

Index `(contact_id, created_at)`, `(sent_at)`.

### activities (M0): append-only history
| column | type | notes |
|---|---|---|
| id | text PK | ULID (time-ordered) |
| entity_type | text not null | `EntityType` |
| entity_id | text not null | no FK on purpose, so history survives deletes |
| type | text not null | `created`, `updated`, `status_changed`, `linked`, `unlinked`, `outreach_logged`, `note_added`, `tagged`, `untagged`, `deleted`, `imported`, `exported`, `login`, `token_created`, `token_revoked` |
| from_value, to_value | text null | |
| summary | text not null | human line, e.g. `Job "Full Stack Dev" saved → applied` |
| payload | text json null | |
| actor | text not null | `Actor` |
| occurred_at | integer not null | |

Indexes: `(entity_type, entity_id, occurred_at)`, `(occurred_at)`, `(type, occurred_at)`.
Never updated or deleted by the app.

### reminders (M0)
`id`, `entity_type`, `entity_id`, `due_at` (not null), `kind` (`follow_up` \| `apply_deadline` \|
`custom`), `note`, `done_at`, `created_at`. Index `(done_at, due_at)`.

### tags (M0) / taggings (M0)
`tags`: `id`, `name` (unique, case-insensitive via `name_norm` unique), `color`.
`taggings`: `tag_id` → tags cascade, `entity_type`, `entity_id`. PK all three. Index
`(entity_type, entity_id)`.

### notes (M0)
`id`, `entity_type` null, `entity_id` null (both null = global note), `body_md` not null,
`created_at`, `updated_at`. Index `(entity_type, entity_id)`.

### external_refs (M0)
`entity_type`, `entity_id`, `source` (e.g. `pathpeer`), `external_id`. PK `(source, external_id)`.
Index `(entity_type, entity_id)`. Makes every import idempotent.

### tasks (M0): background work queue
| column | type | notes |
|---|---|---|
| id | text PK | |
| type | text not null | handler name, e.g. `import.pathpeer` |
| payload | text json not null | |
| status | text not null default `'queued'` | `TaskStatus` |
| run_at | integer not null | |
| attempts | integer not null default 0 | |
| max_attempts | integer not null default 3 | |
| locked_until | integer null | lease; stale leases are re-claimable |
| last_error | text null | |
| result | text json null | |
| created_at, updated_at | integer not null | |

Index `(status, run_at)`.

### Full-text search (M0 custom migration)
- `jobs_fts` = FTS5 with columns `(title, company_name, location_text, description_md)`, `tokenize='unicode61 remove_diacritics 2'`, `rowid` mapped via a `jobs_fts_map(job_id text unique, rowid integer)` table.
- `contacts_fts` = FTS5 with `(full_name, headline, title, company_name, about)`, same mapping pattern.
- Kept in sync by **SQL triggers** on insert/update/delete of `jobs`/`contacts` (and on `companies.name` update). Triggers live in a hand-written migration (`drizzle-kit generate --custom`).
- If M0-04 finds that FTS5 is unavailable in the chosen driver, switch the driver (better-sqlite3), never the feature.

### Later milestones (added by a serialised schema card)
| Table | Milestone | Purpose |
|---|---|---|
| `provider_keys` (provider, encrypted_key, last_verified_at) | M2 | LLM / email / search keys, AES-GCM with `JM_ENCRYPTION_KEY` (SEC-08) |
| `resume_versions` (document path, extracted_text, parsed json, is_active, quality_score, quality_report json) | M2 | |
| `skills` (name, name_norm unique, source) | M2 | skill bank |
| `job_analyses` (job_id, resume_version_id, prompt_version, model, input_hash, score, summary, qualifications json, stale) | M2 | unique `(job_id, resume_version_id, prompt_version)` |
| `contact_analyses` (contact_id, prompt_version, model, input_hash, positives, negatives, commonalities, hooks json, compatibility, recommended_format) | M2 | |
| `llm_usage` (day, provider, model, input_tokens, output_tokens, cost_estimate) | M2 | daily budget (SEC-11) |
| `people_suggestions` (job_id, name, title, linkedin_url, snippet, source, status) | M4 | |
| `discovery_sources`, `discovery_runs` | M5 | |
| `goals` (week_start, targets json) | M5 | |

## 3. OwnerSettings (json in `owner.settings`)
```ts
{
  timezone: string;              // IANA, default 'America/Toronto'
  followUpDays: number;          // default 7  → next_follow_up_at when a contact becomes 'contacted'
  weeklyTargets: { applications: number; contacts: number };   // default 10 / 50
  dailyCaps: { invites: number; dms: number };                 // default 15 / 30 (Career OS)
  llmDailyBudgetUsd?: number;    // M2
}
```

## 4. Fingerprint and dedupe (packages/core/src/fingerprint.ts)
1. `ats_provider` + `ats_job_id` known → `fingerprint = "ats:<provider>:<id>"`.
2. Else `canonical_url` known → `"url:" + sha256(normalizeUrl(url)).slice(0,32)`.
3. Else → `"manual:" + ulid()` (never collides; manual entries are distinct by definition).

Exact fingerprint match → return the existing job (`created: false`). Separately, **possible
duplicates** = same `company_id` + same `title_norm` (+ same `location_text` normalised if both
present). These are returned to the caller as `possibleDuplicates[]` and **never merged
automatically** (landscape §4, Muatasim rule). SimHash comes in M5.

`normalizeUrl`: lowercase scheme and host, drop `www.`, drop fragment, drop tracking params
(`utm_*`, `gh_src`, `ref`, `source`, `lever-source`, `trk`), sort remaining query params, strip
trailing slash.

## 5. Status side-effect rules (packages/core/src/status-rules.ts)
Pure function `applicationTransition(prev, next, now)` → patch:

| to | sets (only if currently null) | also |
|---|---|---|
| `applied` | `applied_at = now` | |
| `interview` | `interview_at = now`; `applied_at = now` if null | |
| `offer` | `offer_at = now` | |
| `rejected`, `withdrawn` | `closed_at = now` | |
| back to an open status (`saved`, `applied`, `networking`, `interview`, `offer`) | `closed_at = null` | |

`contactTransition(prev, next, now, settings)` → patch:

| to | sets |
|---|---|
| `contacted` | `last_outbound_at = now` if null; `next_follow_up_at = now + followUpDays` if null; `connection_state = 'invite_pending'` if `'none'` |
| `messaging` | `thread_state = 'replied'` if `'none'`/`'awaiting_reply'`; `connection_state = 'connected'` if `'invite_pending'` |
| `completed`, `closed` | `next_follow_up_at = null` |

Any status → any status is allowed (the user is in charge). Every transition writes one
`status_changed` activity.

## 6. Import mappings

### ResumeScorer `data/jobs/jobs.json` → jobs + job_sightings (no application)
Shape: `{ jobs: { [sha1]: { id, source, company, title, location, url, posted, snippet, first_seen, last_seen } }, last_run }`.
- sighting: `source='import_resumescorer'`, `external_id=id`, `raw=record`, `seen_at=last_seen`.
- job: dedupe by fingerprint (URL). Create company by name. `description_md = snippet`,
  `posted_at = parse(posted)` if parseable. **No application row** (discovered, not tracked).

### PathPeer export → all entities (**confirm field names against the real export, M0-01**)
| PathPeer | Ours |
|---|---|
| tracker `SAVED/APPLIED/NETWORKING/INTERVIEW/OFFER/REJECTED/WITHDRAWN` | `ApplicationStatus` lowercase (`INTERVIEW`→`interview`, `OFFER`→`offer`) |
| contact status `Unsaved/Saved/Contacted/Messaging/Stuck/Completed/Closed` | `ContactStatus` lowercase |
| compatibility `Strong/Moderate/…` | `Compatibility` lowercase |
| job id / contact id / company id (24-hex) | `external_refs(source='pathpeer', external_id)` |
| job fields (title, company, salary, location, posted date, url, description) | `jobs.*` |
| rating (stars) | `applications.rating` / `contacts.rating` |
| tags | `tags` + `taggings` |
| activities ("X Saved → Contacted", time) | `activities` with `actor='import'` and the original timestamp |
| resolved email (+ Verified, work) | `contact_emails(status='verified', type='work', provider='pathpeer')` |
| intro message / analysis | `notes` on the contact (body = analysis text) until M2 tables exist |
| notes | `notes` |
