# M0-06 · Core contracts, enums and pure functions

Status: open
Blocked by: M0-03
Wave: 4
Owner: agent
Model: sonnet

**Wave** 4 · **Model** sonnet · **Depends on** M0-03 · **Parallel-safe with** M0-05

**Goal.** `packages/core` holds every zod contract named in `docs/spec/api-rest.md`, the enums
from data-model §1, and the pure helpers M1 needs (signatures + tests). No I/O.

**Read first.** `docs/spec/api-rest.md` (all) · `docs/spec/data-model.md` §1, §3, §4, §5.

**Owns.** `packages/core/**`.

**Contract.**
- `src/enums.ts`: const arrays + zod enums for every enum in data-model §1.
- `src/contracts/<module>.ts` for `common` (Paginated, ErrorBody, ErrorCode, Scope, Actor, Cursor),
  `auth`, `companies`, `jobs`, `capture`, `contacts`, `links`, `outreach`, `followups`,
  `activities`, `tags`, `notes`, `search`, `imports`, `exports`, `settings`, `summary`,
  `system`. Every input/output schema **named in api-rest** exists with that exact name, and input
  objects are `.strict()`.
- `src/ids.ts`: `newId()` (ULID). `src/time.ts`: `startOfDayInTz(ms, tz)`, `addDays(ms, n)`,
  `parseLocalDate('YYYY-MM-DD', tz)`.
- `src/normalize.ts`: `normalizeName`, `normalizeUrl` (data-model §4 rules),
  `normalizeLinkedinProfileUrl` (→ `https://www.linkedin.com/in/<slug>` or null), `registrableDomain`
  (simple: host minus `www.`; no public-suffix list in M1, recorded as a known limitation).
- `src/fingerprint.ts`: `jobFingerprint({atsProvider, atsJobId, canonicalUrl})` per data-model §4
  (sha256 via `node:crypto`, the only allowed Node import in core).
- `src/status-rules.ts`: `applicationTransition(prev, next, now)` and
  `contactTransition(prev, next, now, settings)` returning patches (data-model §5).
- `src/cursor.ts`: `encodeCursor({k, id})` / `decodeCursor(s)` (base64url JSON, validated).

**Acceptance.** `pnpm --filter @jm/core test` covers:
- `normalizeUrl` strips `utm_*`, `gh_src`, fragment, `www.` and the trailing slash, and sorts query params.
- `jobFingerprint` priority ats > url > manual.
- **Every row** of data-model §5 (both tables).
- `contactTransition` into `contacted` with existing `next_follow_up_at` keeps it.
- `startOfDayInTz` across a DST change in `America/Toronto`.
- Cursor round-trip and rejection of a tampered cursor.
- Each contract module exports every schema name listed in api-rest (one test iterates a name list).

**Decision table: `normalizeLinkedinProfileUrl`**
| Input | Output |
|---|---|
| `https://ca.linkedin.com/in/some-slug/?trk=x` | `https://www.linkedin.com/in/some-slug` |
| `linkedin.com/in/Some-Slug` | `https://www.linkedin.com/in/some-slug` |
| `https://www.linkedin.com/company/acme` | `null` (not a profile) |
| empty / non-URL | `null` |

**Don'ts.** No DB, fetch or fs imports. No default exports. Don't invent fields that api-rest doesn't
list. If a shape is ambiguous, report BLOCKED with the exact question.

## Comments
