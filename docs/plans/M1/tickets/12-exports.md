# M1-12 · Exports

Status: open
Blocked by: M1-07, M1-08, M1-09, M1-10
Wave: 4
Owner: agent
Model: sonnet

**Wave** 4 · **Model** sonnet · **Depends on** wave 3 · **Parallel-safe with** M1-11, M1-13, M1-14

**Goal.** REQ-EXP-01 and SEC-21 (api-rest §12 exports).

**Owns.** `apps/server/src/modules/exports/**`.

**Contract.** `GET /exports/json` streams `{version, exportedAt, tables: {<name>: rows[]}}` for every
table except `sessions` and `api_tokens`, with `owner` without `password_hash`. `GET /exports/csv?entity=`
for jobs (job + company + tracking) and contacts (contact + company + primary email). CSV quoting
per RFC 4180, and cells starting with `= + - @ \t \r` are prefixed with `'` (formula-injection guard).
Records an `exported` activity.

**Acceptance.** Tests:
- The JSON has no `password_hash`, `token_hash` or `sessions` key.
- A CSV cell `=HYPERLINK(...)` is exported as `'=HYPERLINK(...)`.
- A cell with a comma and a quote is escaped correctly.
- The row counts match the DB.

## Comments
