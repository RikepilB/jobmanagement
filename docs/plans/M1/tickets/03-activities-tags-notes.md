# M1-03 · Activities, tags, notes

Status: open
Blocked by: M1-00
Wave: 1
Owner: agent
Model: sonnet

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

## Comments
