# M1: Agent-first MVP (full task cards)

> **Goal:** every requirement in `docs/spec/SPEC.md` §4 works through **REST + CLI + MCP**, with a
> minimal web UI. Richard can run his pipeline for a week without PathPeer.
> **Precondition:** the M0 exit gate has passed. Every card inherits the template's default
> "Must not touch" list (`package.json`/lockfile, `packages/db/**`, `packages/core/src/contracts/**`,
> registries, other modules' folders) and the always-on Don'ts (no new deps, no snapshot tests, no
> invented fields).

## Waves (≤ 4 implementers at once)

| Wave | Cards | Mode |
|---|---|---|
| 0 | M1-00 shared projections + enum unification | serialised |
| 1 | M1-01 core parsers · M1-02 companies · M1-03 activities/tags/notes · M1-04 search/lookup/settings/summary | parallel |
| 2 | M1-05 jobs (write side) · M1-06 contacts (write side) | parallel |
| 3 | M1-07 jobs (list + board) · M1-08 contacts (list + board) · M1-09 capture · M1-10 links + outreach + follow-ups | parallel |
| 4 | M1-11 imports · M1-12 exports · M1-13 adapters A (jobs/companies/capture) · M1-14 adapters B (contacts/links/outreach/followups) | parallel |
| 5 | M1-15 adapters C (activities/tags/notes/search/settings/summary/imports/exports) · M1-16 web: jobs · M1-17 web: contacts + companies | parallel |
| 6 | M1-18 web: today/activity/search/settings | single |
| 7 | M1-19 golden path, parity, perf, seed | serialised |
| 8 | M1-20 M1 exit gate | orchestrator |

**Shared service signatures** (every card may rely on these; they're built in the named card):

| Function | Built in | Signature |
|---|---|---|
| `resolveCompany` | M1-02 | `(tx, ref: {id: string} \| {name: string; domain?: string}, ctx) => Promise<Company>` (creates if missing; dedupe: domain then name_norm) |
| `recordActivity` | M0-07 | see M0-07 |
| `applyTag` / `removeTag` | M1-03 | `(tx, {tag, entityType, entityId}, ctx) => Promise<void>` |
| `addNote` | M1-03 | `(tx, {entityType?, entityId?, bodyMd}, ctx) => Promise<Note>` |
| `createJob` | M1-05 | `(db, input: JobCreateInput, ctx, opts?: {source?: Source; sightingExternalId?: string; sightingRaw?: unknown}) => Promise<{job: JobDetail, created: boolean, possibleDuplicates: JobCard[]}>`. `source` is **not** a REST field: routes pass `manual`, capture passes `url`/`extension`, imports pass `import_*` |
| `moveJob` | M1-05 | `(db, id, status, ctx) => Promise<JobDetail>` |
| `upsertContact` | M1-06 | `(db, input: ContactUpsertInput, ctx) => Promise<{contact: ContactDetail, created: boolean}>` |
| `moveContact` | M1-06 | `(db, id, status, ctx) => Promise<ContactDetail>` |
| `patchContact` | M1-06 | `(db, id, patch: ContactPatchInput, ctx) => Promise<ContactDetail>` |
| `touchOutbound` / `touchInbound` | M1-06 | `(tx, contactId, at: number, ctx) => Promise<void>` (sets `last_outbound_at` / `last_inbound_at` + `thread_state='replied'` for inbound; records an `updated` activity) |
| `linkContactJob` | M1-10 | `(tx, {jobId, contactId, role}, ctx) => Promise<void>` (idempotent) |
| `logOutreach` | M1-10 | `(db, contactId, input: OutreachLogInput, ctx) => Promise<OutreachMessage>` |
| `getSettings` | M1-04 | `(db) => Promise<OwnerSettings>` (defaults merged) |

`ctx` = `{ now: () => number; actor: Actor; settings?: OwnerSettings }` from `shared/context.ts`.

## Tickets

One file per ticket (local-markdown tracker: see `docs/agents/issue-tracker.md`). The ticket file is the source of truth for its contract and `Status:`.

| ID | Title | Wave | Blocked by | Owner | File |
|---|---|---|---|---|---|
| M1-00 | Shared projections + enum unification | 0 | M0-15 | agent | [`M1/tickets/00-shared-projections-enum-unification.md`](M1/tickets/00-shared-projections-enum-unification.md) |
| M1-01 | Core parsers: JSON-LD JobPosting + readable text | 1 | M0-15 | agent | [`M1/tickets/01-core-parsers-json-ld-jobposting-readable-text.md`](M1/tickets/01-core-parsers-json-ld-jobposting-readable-text.md) |
| M1-02 | Companies module | 1 | M1-00 | agent | [`M1/tickets/02-companies-module.md`](M1/tickets/02-companies-module.md) |
| M1-03 | Activities, tags, notes | 1 | M1-00 | agent | [`M1/tickets/03-activities-tags-notes.md`](M1/tickets/03-activities-tags-notes.md) |
| M1-04 | Search, lookup, settings, summary | 1 | M1-00 | agent | [`M1/tickets/04-search-lookup-settings-summary.md`](M1/tickets/04-search-lookup-settings-summary.md) |
| M1-05 | Jobs module: write side | 2 | M1-01, M1-02, M1-03 | agent | [`M1/tickets/05-jobs-module-write-side.md`](M1/tickets/05-jobs-module-write-side.md) |
| M1-06 | Contacts module: write side | 2 | M1-02, M1-03, M1-04 | agent | [`M1/tickets/06-contacts-module-write-side.md`](M1/tickets/06-contacts-module-write-side.md) |
| M1-07 | Jobs module: list + board | 3 | M1-05 | agent | [`M1/tickets/07-jobs-module-list-board.md`](M1/tickets/07-jobs-module-list-board.md) |
| M1-08 | Contacts module: list + board | 3 | M1-06 | agent | [`M1/tickets/08-contacts-module-list-board.md`](M1/tickets/08-contacts-module-list-board.md) |
| M1-09 | Capture module + safe-fetch | 3 | M1-01, M1-05 | agent | [`M1/tickets/09-capture-module-safe-fetch.md`](M1/tickets/09-capture-module-safe-fetch.md) |
| M1-10 | Links, outreach log, follow-ups & reminders | 3 | M1-05, M1-06 | agent | [`M1/tickets/10-links-outreach-log-follow-ups-reminders.md`](M1/tickets/10-links-outreach-log-follow-ups-reminders.md) |
| M1-11 | Imports: ResumeScorer + PathPeer | 4 | M0-01, M1-03, M1-05, M1-06, M1-07, M1-08, M1-09, M1-10 | agent | [`M1/tickets/11-imports-resumescorer-pathpeer.md`](M1/tickets/11-imports-resumescorer-pathpeer.md) |
| M1-12 | Exports | 4 | M1-07, M1-08, M1-09, M1-10 | agent | [`M1/tickets/12-exports.md`](M1/tickets/12-exports.md) |
| M1-13 | Adapters A: jobs, companies, capture (client + MCP + CLI) | 4 | M1-02, M1-05, M1-07, M1-09 | agent | [`M1/tickets/13-adapters-a-jobs-companies-capture-client-mcp-cli.md`](M1/tickets/13-adapters-a-jobs-companies-capture-client-mcp-cli.md) |
| M1-14 | Adapters B: contacts, links, outreach, follow-ups | 4 | M1-06, M1-08, M1-10 | agent | [`M1/tickets/14-adapters-b-contacts-links-outreach-follow-ups.md`](M1/tickets/14-adapters-b-contacts-links-outreach-follow-ups.md) |
| M1-15 | Adapters C: activities, tags, notes, search, settings, summary, imports, exports | 5 | M1-03, M1-04, M1-11, M1-12 | agent | [`M1/tickets/15-adapters-c-activities-tags-notes-search-settings.md`](M1/tickets/15-adapters-c-activities-tags-notes-search-settings.md) |
| M1-16 | Web: jobs pages | 5 | M1-13 | agent | [`M1/tickets/16-web-jobs-pages.md`](M1/tickets/16-web-jobs-pages.md) |
| M1-17 | Web: contacts + companies pages | 5 | M1-13, M1-14 | agent | [`M1/tickets/17-web-contacts-companies-pages.md`](M1/tickets/17-web-contacts-companies-pages.md) |
| M1-18 | Web: Today, Activity, Search, Settings | 6 | M1-15, M1-16, M1-17 | agent | [`M1/tickets/18-web-today-activity-search-settings.md`](M1/tickets/18-web-today-activity-search-settings.md) |
| M1-19 | Golden path, parity, performance, seed | 7 | M1-00, M1-01, M1-02, M1-03, M1-04, M1-05, M1-06, M1-07, M1-08, M1-09, M1-10, M1-11, M1-12, M1-13, M1-14, M1-15, M1-16, M1-17, M1-18 | agent | [`M1/tickets/19-golden-path-parity-performance-seed.md`](M1/tickets/19-golden-path-parity-performance-seed.md) |
| M1-20 | M1 exit gate (orchestrator) | - | M1-19 | orchestrator | [`M1/tickets/20-m1-exit-gate-orchestrator.md`](M1/tickets/20-m1-exit-gate-orchestrator.md) |

## M1 exit criteria (each needs evidence)
1. Every REQ in SPEC §4 has at least one passing test, listed in a REQ → test table in the exit
   report.
2. The parity test and golden path pass. The perf budgets are met.
3. Richard's real PathPeer export imports with counts equal to `counts.md` (or an explicit deferral).
4. A real Claude Code session completes SPEC §6 workflows 1–4 over MCP.
5. The security audit has no open CRITICAL/HIGH.
6. Richard confirms a week of use without opening PathPeer, or lists what blocked it (which becomes
   the M2 wave-0 input).
