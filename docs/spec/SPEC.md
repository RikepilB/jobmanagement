# jobmanagement: product specification

> **Read this first** if you are building anything. It says *what* the product does. The contracts
> say *exactly how*: [`data-model.md`](data-model.md) · [`api-rest.md`](api-rest.md) · [`mcp.md`](mcp.md) ·
> [`cli.md`](cli.md) · [`security.md`](security.md) · [`optimization.md`](optimization.md) ·
> [`architecture.md`](architecture.md). When this file and a contract disagree, **the contract wins**:
> stop and report the conflict to the orchestrator.

## 1. Product in one paragraph
A self-hosted, MIT-licensed, **agent-first** job-search workspace for one person. It tracks jobs
through a 7-column pipeline and people through a 7-column networking pipeline, links the two, logs
outreach, and tells you who to follow up with today. Richard's AI agents run it through a **CLI
(`jm`)** and **MCP**. A deliberately simple web UI shows the same data. It replaces PathPeer (see
[`../pathpeer-teardown.md`](../pathpeer-teardown.md)) and is the system of record for AutoApply and
ResumeScorer ([`../suite-map.md`](../suite-map.md)).

## 2. Users and surfaces
| User | Surface | Typical job |
|---|---|---|
| Richard's agents (Claude Code / Codex) | MCP tools, `jm … --json` | "Process today's follow-ups", "save this job", "log that I sent an F1" |
| Richard | Web UI (boards, tables, detail pages) | Scan the pipeline, drag cards, read history |
| Richard | `jm` in a terminal | Quick adds, imports, backups |
| Clipper extension (M3) | REST with a `capture` token | Save the job/person on the current tab |
| AutoApply (later) | REST | Read fit and job list, write apply outcomes |

## 3. Glossary
- **Job**: one posting (identity). **Tracking**: Richard's state for a job (status, rating, dates).
  A job without tracking is "untracked" (discovered or imported, not yet saved).
- **Contact**: a person. **Networking status**: where the relationship is (`saved` … `closed`).
- **Connection state / thread state**: LinkedIn invite status and conversation status. These are
  what PathPeer couldn't show (Career OS "RULE ZERO").
- **Outreach message**: a message Richard **sent** (M1) or a **draft** (M2), in format F1–F6
  (`richard-outreach` skill).
- **Activity**: one immutable history line.
- **Follow-up due**: a contact whose `next_follow_up_at` is on or before today and whose status is
  not completed/closed, **or** a due reminder.

## 4. Requirements (M1 unless marked)
Every requirement is available through **REST + CLI + MCP** (REQ-AGT-01) and, where noted, the web
UI.

### Jobs
| ID | Requirement | Acceptance (summary) |
|---|---|---|
| REQ-JOB-01 | Add a job manually with a company by id or name | `POST /jobs` creates the company if missing (dedupe by domain, then name) and tracking `saved` unless `track:false` |
| REQ-JOB-02 | Capture a job from a URL (non-LinkedIn) | JSON-LD first, readable text fallback, SSRF-safe (SEC-13). LinkedIn URL without content → 422 |
| REQ-JOB-03 | Never create duplicates silently | Exact fingerprint → existing job, `created:false`. Same company + title → `possibleDuplicates`, never merged |
| REQ-JOB-04 | List, filter, search and paginate jobs | All filters in api-rest §3. Keyset pagination. FTS for `q` |
| REQ-JOB-05 | Move a job between the 7 statuses | Status side-effect rules (data-model §5). One activity per move |
| REQ-JOB-06 | Rate 0–5, flag, set next action date, tag, note | PATCH + tags + notes. Every change logged |
| REQ-JOB-07 | Board view | 7 columns in enum order, counts, ≤100 cards each. Web drag-and-drop calls move |
| REQ-JOB-08 | Job detail | Company, tracking, linked contacts with roles, tags, notes, sightings, history |
| REQ-JOB-09 | Look up by URL | `GET /lookup?url=` returns the job/contact or null |

### Companies
| REQ-CMP-01 | Company list with job/contact counts, tier, links | api-rest §2 |
|---|---|---|
| REQ-CMP-02 | Company detail rolls up jobs and contacts | |
| REQ-CMP-03 | Dedupe by domain, then normalised name. Delete blocked while jobs exist | 409 |

### Contacts & networking
| REQ-CON-01 | Upsert a contact by LinkedIn URL, **preserving CRM fields** | Re-upsert changes profile fields only (data-model §2) |
|---|---|---|
| REQ-CON-02 | List, filter, search and paginate contacts | incl. `followUpDue`, `connectionState` |
| REQ-CON-03 | Move between the 7 networking statuses with side effects | `contacted` sets last_outbound and next follow-up (+`followUpDays`), etc. |
| REQ-CON-04 | Set connection state, thread state, follow-up date, rating, compatibility | PATCH |
| REQ-CON-05 | Store emails manually with type and status | provider `manual`. Automatic resolution is M4 |
| REQ-CON-06 | Board view | same as REQ-JOB-07 |
| REQ-LNK-01 | Link a contact to a job with a role | Idempotent. The job shows its network count (Career OS goal: 3–5 per job) |
| REQ-OUT-01 | Log a sent outreach message (format, channel, body, optional hook, job) | Updates `last_outbound_at`. F1 on a `saved` contact moves it to `contacted` |
| REQ-OUT-02 | Mark an outreach message replied or withdrawn | Updates thread/connection state |
| REQ-OUT-03 | Today's outreach counts vs daily caps | `GET /outreach/today`. Caps come from settings (15 invites / 30 DMs) |
| REQ-FUP-01 | Follow-ups due for a date | Contacts + reminders + jobs with a due `next_action_at` |
| REQ-FUP-02 | Snooze a contact N days. Add or complete reminders | |

### Cross-cutting
| REQ-ACT-01 | Every mutation writes exactly one activity (per entity touched) with actor | Tests in every service card |
|---|---|---|
| REQ-ACT-02 | Activity feed: global and per entity, newest first | |
| REQ-TAG-01 | Tags on jobs, contacts and companies. Created on first use | |
| REQ-NOTE-01 | Markdown notes, global or per entity | |
| REQ-SRC-01 | Global search across jobs, contacts and companies | FTS5 bm25 + snippets |
| REQ-SUM-01 | Pipeline summary | Counts by status, follow-ups due today, week progress vs targets, today vs caps |
| REQ-SET-01 | Owner settings (timezone, followUpDays, weeklyTargets, dailyCaps) | |
| REQ-IMP-01 | Import the ResumeScorer `jobs.json` as untracked jobs + sightings, idempotently | Re-running creates 0 new rows |
| REQ-IMP-02 | Import the PathPeer export idempotently, with dry-run and a per-entity report | Counts match the export's status counts (M0-01) |
| REQ-EXP-01 | Export everything as JSON and jobs/contacts as CSV | Secrets excluded (SEC-21) |
| REQ-OPS-01 | `jm init` / `jm serve` / `jm migrate` / `jm backup` | cli.md §1 |

### Agents
| REQ-AGT-01 | CLI and MCP parity for every M1 capability except deletes over MCP | Parity table test in M1-19 |
|---|---|---|
| REQ-AGT-02 | Machine output: `jm --json` prints the REST body. MCP returns `structuredContent` | |
| REQ-AGT-03 | Untrusted third-party text is marked in MCP output | SEC-14 |
| REQ-AGT-04 | README "For agents" section + `jm help agents` quick reference | [article 016] |

### Auth
| REQ-AUTH-01 | Owner password login (web) → HttpOnly session cookie. Logout | |
|---|---|---|
| REQ-AUTH-02 | API tokens with scopes, shown once, revocable, with last-used time | |
| REQ-AUTH-03 | Rate limits, CSRF, host allowlist, security headers | SEC-02/17/18/07 |

### Web UI (functional, minimal design)
| REQ-UI-01 | Pages: Login, Today (follow-ups + summary), Jobs board, Jobs list, Job detail, Contacts board, Contacts list, Contact detail, Companies list/detail, Activity, Search, Settings (tokens, targets, caps) | Every page works with keyboard and mouse. Plain Tailwind, no design system |
|---|---|---|
| REQ-UI-02 | Third-party text rendered as plain text | SEC-09 |

## 5. Non-goals (all milestones unless revisited)
- No multi-user accounts or sharing (a second user means a second DB, ADR-016).
- **No auto-sending** of any message or application. Drafts only. AutoApply is a separate product
  with its own consent gate.
- No LinkedIn scraping, automation or background fetches (ADR-007).
- No SMTP probing for email verification (ADR-008).
- No coach/org features, roadmap or feedback modules (ADR-010).
- No in-app chat "Copilot" (dropped: the user's own agents connect over MCP, so the agent *is* the
  copilot).

## 6. Key agent workflows (must work end to end in M1)
1. **Morning routine:** `pipeline_summary` → `followups_due` → for each contact, the agent drafts in
   chat with `richard-outreach` (outside this app in M1) → Richard sends → `outreach_log`.
2. **Save a job from a link:** `jobs_capture_url` → if `possibleDuplicates` is non-empty, ask Richard
   → `tags_apply`.
3. **Source contacts for a job** (Career OS source-contacts): `jobs_get` → Richard opens LinkedIn
   → `contacts_upsert` (from what Richard pastes or clips) → `link_contact_job`.
4. **Weekly sweep** (Career OS sweep): `contacts_search {status:['contacted'], connectionState:'invite_pending'}`
   → stale >7 days → Richard decides withdraw → `outreach_mark withdrawn`.
5. **Migration day:** `jm import pathpeer <dir> --dry-run` → review report → real run → `jm status`.

## 7. Milestones (detail in [`../plans/ROADMAP.md`](../plans/ROADMAP.md))
M0 foundations · **M1 agent-first MVP** (this spec) · M2 AI layer · M3 Clipper · M4 networking
engine · M5 discovery + analytics · M6 suite integration · M7 OSS release.
