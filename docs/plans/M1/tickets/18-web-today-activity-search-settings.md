# M1-18 · Web: Today, Activity, Search, Settings

Status: open
Blocked by: M1-15, M1-16, M1-17
Wave: 6
Owner: agent
Model: sonnet

**Wave** 6 · **Model** sonnet · **Depends on** M1-15, M1-16, M1-17

**Owns.** `apps/web/src/pages/{today,activity,search,settings}/**`.

**Contract.**
- **Today (`/`):** summary tiles (pipeline counts, week vs targets, today's invites/DMs vs caps with
  a warning at ≥ cap) + follow-ups due (contacts with snooze/open actions, reminders with done,
  jobs with next actions).
- **Activity:** a feed with entity-type/actor filters.
- **Search:** a query box with grouped results.
- **Settings:** tokens (list, create with scopes and one-time display plus a copy button, revoke
  with confirmation) and the settings form (timezone, followUpDays, weekly targets, daily caps).

**Acceptance.** Build passes. A component test confirms that the token-create result is shown once
and cleared when the dialog closes (no storage APIs used).

## Comments
