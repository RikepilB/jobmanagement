# M1-04 · Search, lookup, settings, summary

Status: open
Blocked by: M1-00
Wave: 1
Owner: agent
Model: sonnet

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

## Comments
