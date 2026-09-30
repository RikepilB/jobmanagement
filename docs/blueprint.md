# jobmanagement: architecture and blueprint

> **Status 2026-09-30:** this is the **vision document**. For implementation, the source of truth is
> [`spec/`](spec) (SPEC, architecture, data model, REST, MCP, CLI, security, optimisation), the plans in
> [`plans/`](plans), and the build harness in [`harness/`](harness). ADR-015 to ADR-019 supersede this
> file's stack (§3: Next.js/Postgres → Hono/SQLite), data model (§5), API (§6), MCP timing (§9: now
> M1) and roadmap (§12 → `plans/ROADMAP.md`). The principles (§1), risks (§11) and PathPeer export (§0)
> still apply.


> An open-source, self-hostable, candidate-side job search system to replace PathPeer.
> Core: a **jobs kanban**, a **networking CRM** (contacts + kanban + outreach) and a
> **clipper extension**. AI analysis and discovery are layered on top. Status: **plan, nothing
> built**. Source evidence: [`pathpeer-teardown.md`](pathpeer-teardown.md). Decisions:
> [`decisions.md`](decisions.md). Skills: [`skills-audit.md`](skills-audit.md). Other OSS
> projects and what to borrow: [`landscape.md`](landscape.md). How AutoApply and ResumeScorer
> plug in: [`suite-map.md`](suite-map.md). **License: MIT** (ADR-011).

---

## 0. Do this first: export PathPeer data (urgent)

The PathPeer account is **archived and read-only**, and access can end at any time. It holds
about 306 tracked jobs, 700+ contacts (571 saved, 71 contacted, 21 messaging, 20 stuck,
37 completed), analyses, intro messages, resolved emails, the activity history, a 126-item
skill
bank and the resume PDF. Export before building anything else.

**Method** (Richard's own logged-in session, driven by Claude in Chrome, no credential handling):

1. **Primary: capture responses in the page.** Install a small in-page hook that wraps
   `window.fetch` and records the **response bodies** of `/api/trpc/*` calls the app already
   makes. Record the procedure name plus JSON. **Never** record request headers. Then walk the
   UI: Jobs Tracker (7 status lists), Networking list (scroll to load all pages), Companies,
   Notes, Resume, Activity. The list procedures `jobs.getTrackedJobs` (×7), `clients.getContacts`,
   `clients.getTrackedContacts`, `companies.list`, `clients.getActivities` and `clients.getNotes`
   should carry the bulk. A 30-minute spike should confirm the payload fields.
2. **Per-entity detail** is available only when a drawer is opened (`jobs.getAnalyzedJob`,
   `clients.getContactById`, `clients.getContactEmail`). Prioritise it: every job in Applied,
   Networking, Interviewing and Offered, and every contact in Contacted, Messaging and Completed
   (≈250 drawer opens). Saved contacts get list-level data only.
   **Caveats (unverified until the spike):** a wrapper installed after page load sees nothing
   if the tRPC client captured a `fetch` reference at startup. A full page `navigate` also
   destroys the hook, so move only with in-app sidebar clicks. **The spike's first check:**
   install the hook, open one job drawer, and confirm at least one captured body.
3. **Fallback A: HAR export, run by the user.** DevTools → Network → preserve log → walk the
   views → "Export HAR (sanitized)". Before saving it outside OneDrive, confirm no
   `Authorization`/`Cookie` headers remain. This captures response bodies with no script
   injection at all.
4. **Fallback B: DOM extraction** from `innerText` where neither route captures a list.
5. Download the resume PDF(s) through the UI.
6. Save everything to `C:\Users\a2021\dev\jobmanagement-private\pathpeer-export-<date>\`, which is
   **outside OneDrive and outside the repo**. It is PII.
7. **Verify:** the per-status counts match the PathPeer UI **at export time**. Teardown §7 is
   only a reference, because counts shifted between page loads (Applied 79 vs 78).

The importer (Phase 1) maps these dumps onto the schema in §5. It is idempotent and keyed by
`external_refs(source='pathpeer', external_id)`.

---

## 1. Product principles

1. **Candidate-side and single-user first.** No org or coach model. Every table carries
   `user_id` from day one, so multi-user can come later without a migration rewrite.
2. **Local-first, self-hostable, open.** `docker compose up` gives a working app. Full JSON
   export always works. No vendor lock-in.
3. **Draft, never send.** The system prepares messages and moves. The human clicks send on
   LinkedIn or in email. This matches the Career OS hard rule.
4. **Capture on user action only.** The extension reads the current tab when clicked. It never
   crawls or automates LinkedIn, and never reads cookies or credentials.
5. **Every AI output is reproducible.** Store the prompt version, model, input hash and resume
   version. Stale analyses are marked stale, never silently kept.
6. **Agent-operable.** A REST API plus an MCP server let Claude Code run the Career OS pipeline
   against this app instead of scraping PathPeer.

## 2. Scope by phase (summary)

| Phase | Theme | Ships |
|---|---|---|
| **P0** | Rescue + foundations | PathPeer export, ADRs, scaffold, schema, CI |
| **P1** | **MVP core (manual)** | Jobs list + kanban, contacts table + kanban, companies, activity log, tags, notes, reminders, URL capture (JSON-LD), PathPeer importer, JSON/CSV export |
| **P2** | AI layer | Resume versions + parsing + skill bank, job-fit analysis, contact analysis, outreach drafts (F1–F6) with hook gate |
| **P3** | Clipper extension | MV3 side panel: capture job/person, known-URL lookup, link contact↔job, status/tag edits |
| **P4** | Networking engine | Outreach log + connection state, follow-up rules, daily caps ledger, email resolution chain, people suggestions |
| **P5** | Discovery + execution | Job source adapters, scheduled scan + prefilter + scoring, weekly targets, dashboard, analytics, digest |
| **P6** | Agent layer | MCP server, Career OS repointed, entity Copilot, Voidscape transcripts, resume-quality (ATS) report ingest |
| **P7** | OSS release | Self-host packaging, synthetic seed data, docs, security review, production-readiness |

Dropped: Roadmap (coach-driven), Feedback, release notes, org/coach tenancy. Careers
(target-role narratives) is an optional P5 item.

---

## 3. Architecture

**Pattern: modular monolith** in a pnpm workspace. It is one deployable web app plus two thin
clients (extension, MCP) that share Zod schemas. There are no microservices. Postgres is the
only stateful dependency, and the queue lives inside it too.

```
 ┌──────────────── Clients ────────────────┐
 │ Web UI (Next.js RSC + client islands)   │──server actions──┐
 │ Chrome MV3 Clipper (side panel)         │──REST /api/v1 ───┤  Bearer token
 │ MCP clients (Claude Code, Desktop)      │──HTTP /api/mcp ──┤  Bearer token
 │ AutoApply (apply engine, suite-map.md)  │──REST /api/v1 ───┤  Bearer token
 │ CLI importer (PathPeer, CSV)            │──REST /api/v1 ───┤
 └─────────────────────────────────────────┘                  ▼
                               ┌──────────── apps/web (Next.js 16) ─────────────┐
                               │ route handlers + server actions (thin, Zod)    │
                               │            ▼                                   │
                               │ server/services/*  ← single source of truth    │
                               │  jobs · contacts · companies · activities ·    │
                               │  tags · notes · reminders · documents ·        │
                               │  resume · analysis · outreach · discovery ·    │
                               │  enrichment · goals · analytics · export       │
                               │            ▼                                   │
                               │ packages/db (Prisma) ──► PostgreSQL 16         │
                               │ jobs runner ──► pg-boss (same Postgres)        │
                               └───────┬──────────────┬───────────────┬─────────┘
                                       ▼              ▼               ▼
                             packages/ai        packages/sources  packages/enrich
                             (OpenRouter /      (Greenhouse, Lever, (email resolvers,
                              Ollama, prompts,   Ashby, Workday,    search providers,
                              Zod outputs,       JSON-LD, HN,       LinkedIn shortcut
                              evals)             curated lists)     URL builder)
                                       │
                                       ▼
                             worker process (pg-boss): analyze.job, analyze.contact,
                             resume.parse, discovery.scan (cron), enrich.email,
                             reminders.digest (cron), export.build
```

**Job runner:** lift ScoutLane's `server/jobs/runner.ts` + `dispatch.ts` abstraction. In
`worker` mode it enqueues to pg-boss and a long-running worker picks jobs up (the Docker
self-host default). In `inline` mode it runs via `after()`, which is the Vercel fallback. For
long discovery scans, add findleads' **checkpoint/cursor** pattern (`lib/jobs/checkpoint.ts`)
so a killed invocation resumes cleanly.

### 3.1 Stack

| Layer | Choice | Why |
|---|---|---|
| Web | Next.js 16 App Router, React 19, TypeScript strict | Same stack as ScoutLane and findleads, so modules lift cleanly |
| UI | Tailwind v4 + shadcn/ui, dnd-kit (kanban), TanStack Table (contacts/jobs tables), Recharts | ScoutLane already uses dnd-kit and shadcn |
| DB | PostgreSQL 16 + Prisma 7 (`@prisma/adapter-pg`) | ScoutLane is the largest reuse donor. Swapping to Drizzle is an open option ([`decisions.md`](decisions.md) ADR-003) |
| Queue | pg-boss 12 | No Redis. The queue lives in the same Postgres |
| Validation | Zod 4 in `packages/schemas`, shared by web, extension and MCP | One contract for every client |
| AI | `openai` SDK → OpenRouter (fallback chain), optional Ollama for private runs | Lifted from ScoutLane `lib/llm/openrouter.ts`. ResumeScorer already proved Ollama for PII work |
| Auth | Single owner account (Auth.js credentials or magic link) + **hashed API tokens** (one model for Clipper, MCP, AutoApply; ADR-014) | Server actions can't serve an extension. Tokens are revocable, scoped and expiring |
| Files | Storage tiers from ScoutLane `lib/storage/upload.ts` (local disk → S3-compatible → DB blob) | Self-host works with zero config |
| Extension | MV3, Vite + TypeScript, `chrome.sidePanel` | See §7 |
| MCP | `@modelcontextprotocol/sdk`, **Streamable HTTP at `/api/mcp` inside the web app** (JobSync pattern, ADR-014) | See §9 |
| Secrets | Provider API keys (LLM, email finder, search) encrypted at rest with `ENCRYPTION_KEY` (JobSync `lib/encryption.ts`) | Self-hosters paste keys in Settings |
| Tests | Vitest + Playwright, AI evals as fixtures | ScoutLane pattern |

### 3.2 Repository layout

```
jobmanagement/
├─ apps/
│  ├─ web/                    Next.js app (UI, /api/v1, server actions, worker entry)
│  │  └─ src/
│  │     ├─ app/(app)/        dashboard, jobs, jobs/board, networking, networking/board,
│  │     │                    companies, notes, documents, resume, analytics, settings
│  │     ├─ app/api/v1/       REST for extension, AutoApply, importer (hashed-token auth)
│  │     ├─ app/api/mcp/      MCP over Streamable HTTP (same tokens, ADR-014)
│  │     ├─ server/services/  domain logic (one folder per module, *-impl.ts testable)
│  │     ├─ server/jobs/      runner + dispatch (lifted), queue definitions
│  │     └─ components/       kanban (lifted), drawers, tables, filters
│  └─ extension/              MV3 clipper
├─ packages/
│  ├─ schemas/                Zod: entities, enums, API DTOs, AI output schemas
│  ├─ db/                     Prisma schema, migrations, seed (synthetic only)
│  ├─ ai/                     llm client, prompts/*.md (versioned), analyzers, evals/
│  ├─ sources/                job source adapters + JSON-LD JobPosting parser
│  └─ enrich/                 email resolvers, people-search providers, LinkedIn URL builder
├─ tools/importers/pathpeer/  maps export dumps → /api/v1 (idempotent)
├─ docker-compose.yml         postgres + web + worker
└─ docs/                      blueprint, teardown, decisions, skills audit, handoff tree
```

---

## 4. Modules (feature spec)

### 4.1 Jobs (P1)
- **Tracker kanban**: 7 fixed columns (`SAVED, APPLIED, NETWORKING, INTERVIEW, OFFER, REJECTED,
  WITHDRAWN`). Dragging writes an `activity` and sets `applied_at`, `interview_at` and so on.
  Columns are **fixed enums**, never derived from display names.
- **List view** with PathPeer's filter contract (teardown §4): text, position, location,
  location type, posted-within, min fit, source, sponsorship, open/expired, tracker status,
  tags, min rating. Sort by posted, saved, fit or updated.
- **Card**: logo, title, company, age, location type, location, fit ring, rating, tags,
  follow-up badge, expired flag.
- **Drawer tabs**: Details (JD markdown + apply link) · Analysis (P2) · Networking (linked
  contacts, suggestions in P4, LinkedIn shortcuts) · Updates (activity log) · Copilot (P6).
- **Add job**: paste a URL. The server fetches it, tries JSON-LD `JobPosting` first, falls back
  to LLM normalization of the page text (P2), then dedupes. Manual form is always available.

### 4.2 Networking / contacts (P1 core, P4 engine)
- **Table** with filters (company, compatibility, created, tags, rating, status, connection
  state, follow-up due) and a **kanban** with 7 columns: `SAVED, CONTACTED, MESSAGING, STUCK,
  COMPLETED, CLOSED` plus `UNSAVED` (suggestion inbox).
- **Drawer tabs**: Analysis (P2) · Jobs (roles at their company plus explicitly linked jobs) ·
  **Outreach** (new: message log, connection state, next follow-up, drafts by F-type) · Email
  (P4) · Updates · Copilot (P6).
- **Connection state** (fixes PathPeer's blind spot): `none | invite_pending | connected |
  invite_withdrawn`, plus `last_outbound_at`, `last_inbound_at` and `thread_state` (`none |
  awaiting_reply | replied`). The user sets these by hand or the extension captures them
  (§7).
- **Job ↔ contact link** with a role: `recruiter | hiring_manager | teammate | referrer | other`.
  Career OS "3–5 contacts per job" becomes a visible counter on the job card.

### 4.3 Companies (P1)
Table with name, domain, tier (user-set `A/B/C` or dream/target/backup), #jobs, #contacts, and
website/LinkedIn links. Auto-created on job or contact capture and deduped by domain, then by
normalized name. The company page rolls up jobs, contacts and notes.

### 4.4 Activity log (P1)
An append-only `activities` table (entity type, id, type, from→to, payload, actor
user/extension/mcp/system). It powers the Updates tabs, the Activity feed and **all analytics**.
Nothing else is the source of truth for history.

### 4.5 Tags, notes, reminders, documents (P1)
Tags work across entity types, as in PathPeer's `tags.getApplicationsByEntityType`. Notes can
be global or attached to an entity. Reminders have a due date, an entity and an optional rule
source (P4 follow-ups). Documents hold resumes, cover letters and exports in storage tiers.

### 4.6 Resume (P2)
Versioned uploads. Text is extracted with ScoutLane's `resume/extractText.ts` (PDF/DOCX/TXT),
parsed by the LLM into structured JSON, and joined by the **skill bank**, the user's own claimed
skills. "Have these missing skills?" on a job adds a skill and rescores. Setting the active
resume version marks older analyses `stale`.

### 4.7 Job-fit analysis (P2)
Input: normalized job + active resume version + skill bank + the **facts source**
(suite-map §3), matching AutoApply §5's inputs so AutoApply can drop its own prompt. Output (Zod): `score 0–100`,
`summary`, `qualifications[{text, kind: required|preferred, status: met|partial|missing,
evidence}]`, `keywords[]`, `sponsorship_signal`, `seniority`. Keyed by `(job_id,
resume_version_id, prompt_version)`. Adapted from ScoutLane `lib/match/scoreApplicant.ts` with
the perspective **flipped** (candidate evaluating a job).

This is **not** the resume-quality ("ATS") score. That one lives separately on
`resume_versions` (P6: ResumeScorer / hiring-agent **pushes** its report to the API. The app never runs it). See ADR-006.

### 4.8 Contact analysis + outreach drafts (P2)
- Contact analysis output: `positive_indicators[]`, `negative_indicators[]`, `commonalities[]`,
  **`hooks[]`** (specific, verifiable observations, each with its source: post, project, bio,
  profile view, mutual), `compatibility: strong|moderate|weak`, `recommended_format`.
- **Outreach drafts** use the `richard-outreach` formats as the "message type" selector: F1
  connection note (≤300 chars) · F2 first DM · F2-Q question DM · F3 cold email · F4 thank-you
  · F5 loop-closer · F6 follow-up. **Hook gate:** no verifiable hook means no draft (the UI asks
  for one). The column→format mapping comes from Career OS (Saved→F1, Contacted→wait or F6,
  Messaging→F2/F2-Q, Completed→F4/F5).
- Drafts are stored (`outreach_messages.status = draft`). The user copies and sends by hand,
  then marks it sent, which logs the activity and schedules follow-ups.

### 4.9 Email resolution (P4)
A provider chain, first hit wins. Every result stores `provider, method, confidence, checked_at`:
1. Manual entry or an email already known.
2. **Domain pattern learning.** Infer the pattern (`first.last@`, `flast@`…) from other
   confirmed emails at the same domain in the user's own data. Mark it `guessed`.
3. **MX check** (DNS only), which validates that the domain accepts mail at all.
4. **Optional paid providers** via user API keys (Hunter, Prospeo, Apollo…). Can return
   `verified`.
5. SMTP `RCPT TO` probing is **off by default** (see risks §11).

The UI shows the badge (`verified | guessed | unknown`), the type (`work | personal`) and the
provenance line, which PathPeer only implied.

### 4.10 People suggestions (P4)
- **LinkedIn shortcut links** (zero risk): prebuilt people-search URLs for Recruiters, Hiring
  Managers and Teammates at the company, using title keywords from the job.
- **Search-API suggestions** (opt-in, user key): Brave / Google Programmable Search / SerpAPI
  query `site:linkedin.com/in "<Company>" (recruiter OR "talent acquisition" OR "<role>")`, then
  LLM ranking, then a card with snippet and **+Add**, which creates an `UNSAVED` contact stub.
  Full details arrive when the user opens the profile and clips it (§7).

### 4.11 Discovery (P5)
Source adapters port ResumeScorer `scripts/job_finder.py`: Greenhouse, Lever and Ashby public
board APIs by company slug; HN "Who is hiring" (Algolia API); curated new-grad GitHub lists.
Workday and SmartRecruiters come later. A scheduled `discovery.scan` writes **sightings** and
dedupes them into jobs (ADR-005). A cheap keyword/location prefilter and a zero-token
**liveness check** (career-ops pattern) then run **before** any LLM call, and only the top N
live postings go to fit analysis. JobSync's MIT Greenhouse/Lever/Ashby scrapers are the
starting code. ResumeScorer's `jobs.json` is imported as sightings (suite-map §2). The feed is the Jobs list filtered to
`tracker_status = null` (not yet saved).

### 4.12 Execution, dashboard, analytics (P5)
- **Weekly targets**: applications/week, contacts/week, plus Career OS caps (invites/day ≤15
  safe/20 max, DMs/day ≤30, weekdays 08:00–18:00 ET).
- **Dashboard**: this week vs targets, pipeline counts, follow-ups due today, today's outreach
  ledger vs caps, stale analyses, recent activity.
- **Analytics** (from `activities` only): applied→interview rate, contacted→messaging rate
  (invite acceptance proxy), avg fit of applied jobs, response time, and weekly execution
  timeline. The acceptance-rate guard (<40% pause, <25% halt) shows as a banner.
- **Digest**: optional daily email or in-app summary (Resend, lifted from ScoutLane `lib/email`).

### 4.13 Copilot (P6)
A chat thread per entity (job, contact, company). The context bundle is the entity, its
analysis, linked entities, notes and the resume. Suggested prompts: "Intro message", "How do I
ask for a referral?", "Prep me for this interview". The same bundle is served over MCP (§9).

---

## 5. Data model

**Superseded by [`spec/data-model.md`](spec/data-model.md)** (SQLite dialect, one DB per owner, ADR-015/016). That file is the single source of truth. Don't implement from this section.

## 6. API surface

**Superseded by [`spec/api-rest.md`](spec/api-rest.md), [`spec/mcp.md`](spec/mcp.md) and [`spec/cli.md`](spec/cli.md).** One behaviour path (REST), with CLI and MCP as thin adapters (ADR-015/017).

## 7. Chrome extension: "Clipper" (P3)

**Trust boundary:** this is its own extension and its own repo folder. It is **not** built
inside `agent-bridge`. Voidscape and agent-bridge keep their separate contracts (agent-bridge
README, 2026-09-06 ADR). What we reuse from agent-bridge is the *posture*: MV3, `activeTab`,
a strict extension CSP, and user-opened action only.

| Aspect | Design |
|---|---|
| Permissions | `activeTab`, `scripting`, `sidePanel`, `storage`. **Host permission only for the user's API origin** (configurable). No `cookies`, no `webRequest`, no persistent LinkedIn host access |
| Trigger | User opens the side panel or clicks "Clip". Then, and only then, `chrome.scripting.executeScript` runs on the current tab |
| LinkedIn rule conflict | Career OS says "never inject DOM scripts on LinkedIn". This Clipper runs `executeScript` on LinkedIn tabs, limited to a **read-only `innerText` extract on user click**: no DOM writes, no synthetic events, no clicks. That is the same class of access Career Clipper had. **Richard must confirm this exception** (§13) |
| Job pages | 1) `<script type="application/ld+json">` `JobPosting`. *Hypothesis for the P3 fixture spike:* Greenhouse, Lever, Ashby, Workday and many career pages emit it. 2) Site adapters for LinkedIn Jobs and Indeed that read visible text blocks. 3) Generic: `main`/`article` `innerText` |
| Profile pages | Visible profile text (name, headline, current role, about, experience, education) is sent as text. The server normalizes it with an LLM into the contact schema. **No CSS-class selectors** on LinkedIn: they churn, and text + LLM holds up better |
| Messaging pages | Optional "Update status from this thread": reads the visible thread header and last message direction and date, and sets `thread_state` / `last_inbound_at`. The user clicks every time |
| Panel UX | Known-URL badge ("Saved · Applied · 87%"), quick status/rating/tags, link to job, "Draft F1" (P2), a captured-fields preview **before** save |
| Auth | API token pasted once, stored in `chrome.storage.local` |
| Distribution | Unpacked for self-hosters first. Chrome Web Store listing in P7 |

## 8. AI pipeline

- `packages/ai/prompts/<name>@v<N>.md`: versioned prompt files. Outputs are validated against
  Zod schemas, JSON mode first then a plain retry (the ScoutLane pattern).
- Analyzers: `normalizeJob`, `normalizeContact`, `analyzeJobFit`, `analyzeContact`,
  `draftOutreach(format)`, `rankSuggestions`, `parseResume`.
- **Caching**: skip the call when `input_hash + prompt_version + resume_version` is unchanged.
- **Cost control**: prefilter before fit analysis, analyze on save (not on sighting) by default,
  per-day token budget in settings, and a free/cheap default model with fallback chain
  (OpenRouter).
- **Privacy**: an Ollama provider for users who want contact PII to stay local.
- **Evals** (`packages/ai/evals/`): synthetic jobs, resumes and profiles with expected ranges.
  CI checks schema validity and score stability (±5 across 3 runs). The draft gate test asserts
  no draft without a hook and none over 300 chars for F1.

## 9. MCP server (P6)

Served at `/api/mcp` (Streamable HTTP) from the web app, with the same tokens as the Clipper
(ADR-014). JobSync's `src/app/api/mcp/route.ts` is the MIT reference. Tools: `list_applications(status)`, `move_application`, `get_job(id|url)`, `capture_job(url)`,
`list_contacts(status, followup_due)`, `get_contact`, `move_contact`, `link_contact_job`,
`log_outreach(contact, format, body, sent_at)`, `get_followups_due`, `get_ledger_today`,
`get_context_bundle(entity)`. The Career OS shortcuts (source-contacts, process-contact
v6 "saved batch" and "sweep" modes) call these instead of reading PathPeer through Chrome.
`richard-outreach` still owns the words. The MCP tool only stores them.

## 10. Reuse map from existing projects

> **Under ADR-015 (Hono + SQLite), every ScoutLane and JobSync row below is a *port*, not a lift.**
> The logic is reused, but the Next.js/Prisma glue is rewritten, following the `clean-vibe-codebase`
> protocol (baseline first, one file per commit). External MIT code keeps its notices (ADR-013).
> Pure-TS pieces (parsers, `extractText`, the React kanban using `@dnd-kit/core`) port with the
> fewest changes.

| Need | Lift from | Path |
|---|---|---|
| Kanban board | ScoutLane | `src/components/pipeline/KanbanBoard.tsx`, `KanbanColumn.tsx` (dnd-kit) |
| Resume text extraction | ScoutLane | `src/lib/resume/extractText.ts` (+ `__fixtures__`) |
| LLM client + fallback chain | ScoutLane | `src/lib/llm/openrouter.ts` |
| Fit scoring (flip perspective) | ScoutLane | `src/lib/match/scoreApplicant.ts` |
| Stage-email drafting pattern | ScoutLane | `src/lib/llm/draftStageEmail.ts` |
| Job runner inline/worker | ScoutLane | `src/server/jobs/runner.ts`, `dispatch.ts`, `src/server/queues/*` |
| Storage tiers | ScoutLane | `src/lib/storage/upload.ts`, `upload-limits.ts` |
| Rate limiting | ScoutLane | `src/lib/rate-limit.ts` |
| Identity vs sighting + CRM-preserving upsert | findleads | `lib/db/schema.ts`, `lib/db/businesses.ts` (`upsertBusiness`) |
| Checkpointed long job | findleads | `lib/jobs/runScrapeJob.ts`, `lib/jobs/checkpoint.ts` |
| CSV export sanitizing | findleads | `lib/csv/` |
| Job sources | ResumeScorer | `scripts/job_finder.py` (Ashby/Greenhouse/Lever/HN/GitHub lists) |
| Resume-quality score | ResumeScorer | `hiring-agent` pipeline via `skills/score-resume` (external venv) |
| Transcripts of chats/interviews | void-scape | `voidscape inspect → preview → read`, manifest cited `[MM:SS]` |
| MV3 security posture | agent-bridge | `extension/manifest.json` (posture only, not code sharing) |
| Outreach voice + formats | skill | `richard-outreach` (F1–F6, hook gate, banned words) |
| Pipeline rules + caps | Second Brain | `03_Career_OS/CLAUDE.md` (column framework, safety caps) |
| MCP route, token model, key encryption, ATS scrapers, cron scheduler, backup/restore | JobSync (MIT, external) | `src/app/api/mcp/route.ts`, `src/lib/mcp/auth`, `src/lib/encryption.ts`, `src/lib/scraper/*`, `src/lib/scheduler`, `src/app/api/backup/*`. Keep headers, list in `THIRD_PARTY_NOTICES.md` (ADR-013) |
| Liveness, SimHash repost detection, follow-up cadence, fact allowlist, eval harness | career-ops (MIT, external) | `check-liveness.mjs`, `fingerprint-core.mjs`, `detect-reposts.mjs`, `followup-cadence.mjs`, `verify-cv-facts.mjs`, `eval-golden.mjs` |
| Fit score consumer, apply engine | AutoApply (own) | Reads `job_analyses` over `/api/v1`, writes `autoapply_state` (suite-map §2) |

Full list of external borrows with licenses: [`landscape.md`](landscape.md) §4.

**Don't fork ScoutLane** (ADR-002). It is employer-side (Organization, Applicant, per-job
`PipelineStage`, public careers board), and its stage-name-derived status silently maps
unknown names to `REVIEWING`.

## 11. Risks and constraints

| Risk | Mitigation |
|---|---|
| **LinkedIn User Agreement** prohibits scraping and automated collection | Capture only on user click, current tab only. No background fetches of LinkedIn, no automated connects/DMs, no stored LinkedIn session. Text-only extraction. Documented in the README |
| **SMTP probing** to "verify" emails is blocked from residential IPs, unreliable on catch-all domains, and ethically grey | Off by default. Use pattern + MX + opt-in paid providers with honest confidence labels |
| **PII of third parties** (contacts, emails) | Self-hosted data only. Seeds and fixtures are synthetic. Per-contact delete and export. No telemetry. Ollama option |
| **PathPeer data loss** | P0 export before anything else (§0) |
| **LLM cost and drift** | Prefilter, caching by hash, prompt versions, eval gate in CI |
| **ATS HTML/API changes** | JSON-LD first. Adapters have fixtures and contract tests. Failures degrade to "paste text" |
| **Scope creep** (Careers, Copilot, discovery before the core works) | Phase exit criteria (§12). P1 must replace daily PathPeer use by itself |

## 12. Roadmap with exit criteria

| Phase | Build | Exit criteria (verified, not asserted) |
|---|---|---|
| **P0** | PathPeer export (§0) · ADR review · `project-scaffold --archetype modular-monolith` (dry-run first) · pnpm workspace · Prisma schema · docker-compose (postgres) · CI (typecheck, lint, test) | Export counts match the PathPeer UI at export time. `docker compose up` + `pnpm dev` boots an empty app. CI is green |
| **P1** | Services + UI for jobs (list, kanban, drawer: Details/Updates), contacts (table, kanban, drawer: Jobs/Updates/Outreach log), companies, activities, tags, notes, reminders · URL capture with JSON-LD · PathPeer importer · export | Importer reproduces all counts. A drag writes an activity. URL capture works on 3 ATS fixtures. Richard runs a week of the pipeline without opening PathPeer |
| **P2** | Resume versions + parse + skill bank · job-fit analysis · contact analysis with hooks · F1–F6 drafts with hook gate · LLM normalize fallback for capture | Eval suite passes (schema-valid, ±5 stability). A draft is refused without a hook. Stale flag fires on resume change |
| **P3** | Clipper MV3 (job + profile + known-URL badge + link to job + thread-state capture) | Clips a job on Greenhouse, Lever, Ashby, Workday and LinkedIn Jobs, and clips a LinkedIn profile. Only the API origin is a host permission. It replaces Career Clipper in source-contacts |
| **P4** | Outreach log + connection state + follow-up rules (F6 at 5–7 business days) + caps ledger · email resolution chain · people suggestions + LinkedIn shortcuts | The "sweep" view reproduces process-contact v6 sweep output on imported data. Caps banner fires in a synthetic test |
| **P5** | Source adapters + scheduled scan + prefilter · weekly goals · dashboard · analytics · digest | A daily scan produces deduped jobs with scores. Dashboard numbers reconcile with raw `activities` queries |
| **P6** | MCP server · Career OS shortcuts repointed · Copilot · Voidscape transcript attach · resume-quality report ingest (pushed from Richard's PC) · AutoApply API contract (suite-map) · optional Gmail reply-watch (job-ops / jobseeker-analytics idea) | A Claude Code session runs "saved batch" end to end over MCP with zero PathPeer/Chrome reads |
| **P7** | Self-host docs, synthetic seed, `security-review`, `production-readiness`, license, **scrub local paths / vault references / personal ids from docs**, public repo | A fresh clone reaches a working demo in under 10 minutes by following the README |

## 13. Open decisions (need Richard)

1. ~~**License**~~: **resolved, MIT** (ADR-011, 2026-09-29).
2. **ORM**: Prisma (reuse ScoutLane and JobSync) or Drizzle (findleads, lighter). The plan
   assumes Prisma.
3. **Hosting target for personal use**: Docker on the local machine or Vercel + Neon.
   For single-user Docker, **SQLite + an in-process `node-cron` scheduler** (proven by JobSync)
   is simpler than Postgres + pg-boss. Vercel needs Postgres (Neon) with `after()`/pg-boss.
   This choice also settles the DB engine.
4. **Paid providers**: which email-finder and search API keys you're willing to pay for (all
   optional, and the chain works without them). The free self-hosted option is to run Reacher
   (AGPL) as a separate service.
5. **Name/brand** for the public repo (`jobmanagement` is a working name). Run the
   `design-intent` pass before any UI work.
6. **LinkedIn script exception**: allow the Clipper's read-only, user-click `innerText`
   extract on LinkedIn, despite the Career OS "never inject DOM scripts on LinkedIn" rule
   (ADR-007)? If not, LinkedIn capture becomes "paste the profile text".
7. **Build fresh or fork JobSync** (ADR-002)? The recommendation is fresh, lifting its MIT
   modules. The spike evidence is in [`landscape.md`](landscape.md) §5.
8. **Where the single facts source lives** (identity, work authorization, proof points,
   forbidden claims), shared by the app, `richard-outreach` and AutoApply: in the app DB
   (exported to the skills) or in the Second Brain vault (read by the app).
   See [`suite-map.md`](suite-map.md) §3.
9. **AutoApply contract**: accept suite-map §2 (AutoApply drops its own fit prompt and
   `jobs.json` ledger and becomes an API client)? That requires editing AutoApply's approved
   design doc.
10. **Interim tracker while P1 is built**: PathPeer is read-only, so Career OS's "move the
    PathPeer card after every send" step already fails. Options: run JobSync locally (MIT,
    `docker compose up`; SkillSpector scan first if its MCP is enabled in Claude), use
    career-ops' markdown tracker, or keep the Career OS markdown logs only. If a tool is
    chosen, the P1 importer also reads its export.
