# M1-17 · Web: contacts + companies pages

Status: open
Blocked by: M1-13, M1-14
Wave: 5
Owner: agent
Model: sonnet

**Wave** 5 · **Model** sonnet · **Depends on** M1-13 (companies client), M1-14 · **Parallel-safe with** M1-15, M1-16

**Owns.** `apps/web/src/pages/{contacts*,companies*}/**`, `apps/web/src/components/{contacts,companies}/**`.

**Contract.**
- **Contacts board and list:** same patterns as jobs, with filters for status, follow-up due,
  connection state, q and tag.
- **Contact detail:** profile fields, a status select, connection/thread state selects, a follow-up
  date with snooze buttons (3/7/14 days), emails (add, primary), linked jobs, an outreach log (list
  + "Log sent message" form with format/channel/body/hook), tags, notes and history.
- **Companies:** a table (name, tier, #jobs, #contacts, links) and a detail page (jobs and contacts
  tables).

**Acceptance.** Build passes. Component tests: the contact detail renders `about` as plain text, and
the outreach form's format select lists exactly the `OutreachFormat` enum values.

## Comments
