# M1-08 · Contacts module: list + board

Status: open
Blocked by: M1-06
Wave: 3
Owner: agent
Model: sonnet

**Wave** 3 · **Model** sonnet · **Depends on** M1-06 · **Parallel-safe with** M1-07, M1-09, M1-10

**Goal.** REQ-CON-02, -06 (`GET /contacts`, `GET /contacts/board`).

**Owns.** `apps/server/src/modules/contacts/**` (after M1-06 merged). Adds only list/board routes,
their repo queries and tests.

**Contract.** Filters per `ContactListQuery`. `followUpDue=YYYY-MM-DD` means `next_follow_up_at ≤
end of that day (owner TZ)` and status not in (completed, closed). `sort=followUp` orders by
`next_follow_up_at` ascending, nulls last.

**Acceptance.** Tests:
- The `followUpDue` filter respects the TZ day boundary.
- Board counts match SQL.
- Keyset paging is unique across pages.
- The `connectionState=invite_pending` filter returns only those contacts.

## Comments
