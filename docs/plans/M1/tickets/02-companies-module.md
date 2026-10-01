# M1-02 · Companies module

Status: open
Blocked by: M1-00
Wave: 1
Owner: agent
Model: sonnet

**Wave** 1 · **Model** sonnet · **Depends on** M1-00 · **Parallel-safe with** M1-01, M1-03, M1-04

**Goal.** REQ-CMP-01..03 via api-rest §2, plus `resolveCompany` for other modules.

**Owns.** `apps/server/src/modules/companies/**`.

**Contract.** Routes exactly as api-rest §2 (scopes as listed). Service exports `resolveCompany`
(signature in the **Shared service signatures** table in `docs/plans/M1-mvp-core.md`), `createCompany`, `getCompanyDetail`, `listCompanies`, `patchCompany`,
`deleteCompany`. Every mutation records an activity (`entity_type 'company'`).

**Acceptance.** HTTP tests:
- Create → 201. Same domain again → 200 `created:false` and the same id.
- Same name with different case/punctuation and no domain → the existing row is returned.
- List keyset pagination: 3 pages of 2 cover 5 rows without duplicates.
- Detail counts jobs and contacts (insert rows directly for the counts).
- Delete with jobs → 409.
- `resolveCompany` creates exactly once across two calls.

## Comments
