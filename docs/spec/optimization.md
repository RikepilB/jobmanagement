# Optimisation budgets and techniques

> Small single-user workloads, so the goal is **predictable latency, low LLM cost and fast tests**,
> not scale. Numbers are budgets that tests or gates check where noted.

## 1. Budgets

| Area | Budget | How it's checked |
|---|---|---|
| API list endpoints (1k jobs / 10k contacts, local) | p95 < 100 ms | `apps/server/test/perf.test.ts` seeds synthetic rows and times 20 list calls (M1-19) |
| Board endpoints | p95 < 150 ms | same |
| Search (FTS5) | p95 < 50 ms | same |
| Import 1,000 contacts | < 10 s, one transaction per 200 rows | import test with a synthetic bundle |
| `pnpm verify` (lint + typecheck + unit) | < 90 s on Richard's machine | measured in M0-14 and recorded in `gates.md` |
| Web initial JS | < 300 KB gzip | `vite build` report check in `verify:full` |
| Cold start `jm serve` | < 2 s to `/healthz` 200 | smoke script |
| LLM (M2) | Default daily budget $1. Cached re-runs cost $0 | `llm_usage` + cache hit test |

## 2. Database
- Pragmas on every connection: WAL, `synchronous=NORMAL`, `foreign_keys=ON`, `busy_timeout=5000`.
- An index for **every filter and sort column** in the list queries (data-model §2). New filters
  must add an index in the same schema card.
- **Keyset pagination** on `(sortKey, id)`. Offset pagination is banned (CONVENTIONS rule).
- **No N+1:** list endpoints fetch cards with one joined query plus one batched query for tags
  (`WHERE entity_id IN (…)`). Code review rejects per-row queries inside loops.
- FTS5 via triggers, so search never scans base tables. Use `bm25()` ranking + `snippet()`.
- Writes go through short transactions. Imports batch 200 rows per transaction. Single writer (WAL)
  is plenty for one owner.
- `jm backup` uses `VACUUM INTO` (consistent snapshot while the server runs).

## 3. Server
- Hono `compress()` for JSON > 1 KB. Static assets are served with `Cache-Control: immutable` for
  hashed files and `no-cache` for `index.html`.
- The task runner polls every 2 s, runs at most 2 tasks at once, and leases for 5 min. LLM tasks
  (M2) run at concurrency 1 by default.
- Outbound fetches (capture, discovery) use a 10 s timeout, ≤2 MB, and `If-None-Match`/ETag when a
  source provides it (M5).

## 4. LLM cost (M2+)
- **Analyze on save, not on sighting.** Discovery results are pre-filtered by keywords and location
  before any model call [career-ops liveness idea: skip dead postings].
- Cache key = `sha256(prompt_version + model + normalised input)`. An identical input returns the
  stored analysis.
- Default to a cheap model with a fallback chain (ScoutLane pattern). The model is configured in
  Settings, never hard-coded.
- The prompt shape is **goal, constraint, output schema, stop** (no long procedures), which is Boris
  Cherny's subtraction rule [article 012]. Short prompts are cheaper and, on current models, better.

## 5. Web
- TanStack Query cache with `staleTime` of 30 s for lists. Moves update optimistically, with
  rollback on error.
- Boards load at most 100 cards per column. "Load more" uses the list endpoint.
- No virtualisation until a list actually exceeds 1,000 rendered rows.

## 6. Build and agent throughput
- `pnpm verify` stays fast: Vitest runs in parallel threads, and the typecheck uses project
  references (`tsc -b`).
- Every card's acceptance names **specific test files**, so an implementer runs only those while
  iterating and runs `pnpm verify` once at the end.
