# M1-16 · Web: jobs pages

Status: open
Blocked by: M1-13
Wave: 5
Owner: agent
Model: sonnet

**Wave** 5 · **Model** sonnet · **Depends on** M1-13 · **Parallel-safe with** M1-15, M1-17

**Owns.** `apps/web/src/pages/jobs*/**`, `apps/web/src/components/jobs/**`.

**Contract.** REQ-UI-01 for jobs.
- **Board:** 7 columns, cards (title, company, rating stars, location type). Drag and drop
  (`@dnd-kit/core`, the column-move pattern from ScoutLane `KanbanBoard.tsx`, ported) calls
  `jobs.move` with an optimistic update and rollback on error.
- **List:** a table with filters (status multi-select, q, tag, rating ≥) mirrored in the URL query
  string, and a "Load more" button (cursor).
- **Detail:** fields, a status select, a rating control, a flag toggle, linked contacts (link via a
  contact search box using `/search?types=contact`, with a role select), tags (add/remove), notes
  (add), and history (`/activities?entityType=job`). The description renders as **plain text**
  (SEC-09).
- An "Add job" dialog (manual form) and a "Capture URL" input.

**Acceptance.** `pnpm --filter @jm/web build` passes. A component test (renderToString with
mocked query data) shows that the board renders 7 columns with counts, and that the detail page
renders the description inside a `<pre>`, not as HTML. The page works in the smoke run (M1-19).

**Don'ts.** No new UI libraries. No `dangerouslySetInnerHTML`. No local persistence.

## Comments
