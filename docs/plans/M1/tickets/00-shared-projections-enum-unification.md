# M1-00 · Shared projections + enum unification

Status: open
Blocked by: M0-15
Wave: 0
Owner: agent
Model: sonnet

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

## Comments
