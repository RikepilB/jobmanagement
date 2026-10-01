# M1-07 · Jobs module: list + board

Status: open
Blocked by: M1-05
Wave: 3
Owner: agent
Model: sonnet

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

## Comments
