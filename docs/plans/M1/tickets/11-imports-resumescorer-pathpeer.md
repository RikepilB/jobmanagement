# M1-11 · Imports: ResumeScorer + PathPeer

Status: open
Blocked by: M0-01, M1-03, M1-05, M1-06, M1-07, M1-08, M1-09, M1-10
Wave: 4
Owner: agent
Model: sonnet

**Wave** 4 · **Model** sonnet · **Depends on** M0-01 (PathPeer shape), M1-03, M1-05, M1-06, M1-07, M1-08, M1-09, M1-10 · **Parallel-safe with** M1-12, M1-13, M1-14

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

## Comments
