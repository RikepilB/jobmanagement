# M1-10 · Links, outreach log, follow-ups & reminders

Status: open
Blocked by: M1-05, M1-06
Wave: 3
Owner: agent
Model: sonnet

**Wave** 3 · **Model** sonnet · **Depends on** M1-05, M1-06 · **Parallel-safe with** M1-07, M1-08, M1-09

**Goal.** REQ-LNK-01, REQ-OUT-01..03, REQ-FUP-01..02 reminders (api-rest §6, §7, §8).

**Owns.** `apps/server/src/modules/links/**`, `modules/outreach/**`, `modules/followups/**`.

**Contract.**
- Links: idempotent insert/delete on `job_contacts`, recording `linked`/`unlinked` activities on
  **both** the job and the contact.
- `logOutreach`: insert with status `sent` and `sent_at` (default now), set the contact's
  `last_outbound_at` (through a contacts service function `touchOutbound(tx, contactId, sentAt, ctx)` that
  **M1-06 must also export**). If format `f1` and the contact is `saved` → call `moveContact(…,'contacted')`
  (M1-06). If format `f6` → clear `next_follow_up_at`. Records
  `outreach_logged`.
- `PATCH /outreach/:id` replied/withdrawn per api-rest §7: replied → `touchInbound`. Withdrawn invite
  → `patchContact({connectionState:'invite_withdrawn'})`. F6 logged → `patchContact({nextFollowUpAt:null})`.
  Never write `contacts` directly.
- `GET /outreach/today`: counts by channel where `sent_at` is today in the owner TZ, plus the caps
  from settings.
- `GET /followups?due=`: contacts due + reminders due and not done + tracked jobs with a due
  `next_action_at`. `POST /reminders`, `POST /reminders/:id/done`.

**Acceptance.** Tests:
- Linking twice → one row.
- Logging an F1 on a saved contact moves it to `contacted` with `next_follow_up_at` set.
- Marking replied sets `thread_state=replied` + `last_inbound_at`.
- Today counts across a TZ day boundary.
- Follow-ups for a date include a snoozed-into-that-day contact and exclude completed ones.

## Comments
