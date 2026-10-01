# M0-13 · Task runner

Status: open
Blocked by: M0-07
Wave: 6
Owner: agent
Model: sonnet

**Wave** 6 · **Model** sonnet · **Depends on** M0-07 · **Parallel-safe with** M0-08, M0-09

**Goal.** A reliable in-process background runner on the `tasks` table.

**Read first.** data-model §2 `tasks` · optimization §3.

**Owns.** `apps/server/src/tasks/**`.

**Contract.**
- `enqueue(db, {type, payload, runAt?, maxAttempts?}) → taskId`.
- `registerHandler(type, fn)`. A handler registry file `tasks/handlers.ts` imports
  `modules/imports/tasks.ts` (a stub `export const handlers = {}` **created by M0-07**, later owned
  by M1-11) and registers each entry. **Written once here.**
- `startRunner({db, now, pollMs=2000, concurrency=2, leaseMs=300000}) → {stop()}`: claims due tasks
  atomically (`UPDATE … WHERE status='queued' AND run_at<=? OR (status='running' AND locked_until<?)
  RETURNING`), runs the handler, and writes `result` or `last_error`. Retries use exponential
  backoff (`run_at = now + 2^attempts * 10s`) until `max_attempts`, then `failed`.
- `getTask(db, id)`.

**Acceptance.** Tests with a fake clock:
- A queued task runs once and ends `done` with its result.
- A throwing handler retries, then ends `failed` after `max_attempts` with `last_error`.
- A stale lease is re-claimed.
- Two runners never run the same task (simulate with two claim calls).

**Don'ts.** No external queue libraries. No `setInterval` without `stop()`.

## Comments
