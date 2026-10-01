# M1-06 · Contacts module: write side

Status: open
Blocked by: M1-02, M1-03, M1-04
Wave: 2
Owner: agent
Model: sonnet

**Wave** 2 · **Model** sonnet · **Depends on** M1-02, M1-03, M1-04 · **Parallel-safe with** M1-05

**Goal.** REQ-CON-01, -03, -04, -05, REQ-FUP-02 snooze (upsert, get, patch, emails, delete, **move, snooze**) per api-rest §5.

**Owns.** `apps/server/src/modules/contacts/**`.

**Contract.** `POST /contacts` (upsert), `GET /contacts/:id`, `PATCH /contacts/:id`,
`POST/DELETE /contacts/:id/emails…`, `DELETE /contacts/:id`, `POST /contacts/:id/move`,
`POST /contacts/:id/snooze`. `moveContact` applies `contactTransition(prev, next, now, settings)`
(settings via `getSettings`, M1-04). Snooze sets `next_follow_up_at = startOfDay(today + days)` in the
owner TZ. Also exports `patchContact`, `touchOutbound`, `touchInbound` (see the **Shared service signatures** table in `docs/plans/M1-mvp-core.md`),
which the outreach module uses. `upsertContact`: normalise
`linkedinUrl`. If it exists, update **only profile fields** (full_name, headline, title,
location_text, about, avatar_url, company) and never CRM fields (data-model §2). Else insert with
`status` (default `saved`). Emails are lowercased, unique per contact, and `isPrimary` true clears
the others' primary flag.

**Acceptance.** Tests:
- Upsert twice with the same LinkedIn URL in different forms (`ca.linkedin.com/...?trk=`) → one
  row, `created:false` on the second.
- **The second upsert with `status:'closed'` doesn't change the status**, while the headline does
  update. This is the CRM-preserving test.
- PATCH `connectionState: 'connected'` records an `updated` activity with from/to.
- Adding the same email twice → one row. Setting primary flips the others.
- `saved→contacted` sets `last_outbound_at`, `next_follow_up_at = +followUpDays` and
  `connection_state=invite_pending`.
- `contacted→messaging` sets `thread_state=replied` and `connection_state=connected`.
- `completed` clears `next_follow_up_at`.
- Snooze 3 days lands on the start of that local day.

**Don'ts.** Don't implement list/board (M1-08).

## Comments
