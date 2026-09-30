# Landscape: open-source job-search projects to learn from and borrow

Researched 2026-09-29 with the `gh` CLI (GitHub API) and web search. The Firecrawl key was
rejected and the GitHub MCP had bad credentials. Star counts and push dates are as of that
day. jobmanagement is **MIT** (ADR-011). The license column below decides what we may copy.

## 1. License gate (read before borrowing anything)

| Class | Rule | Projects |
|---|---|---|
| **MIT / Apache-2.0** | Code may be copied. **Keep the original copyright notice** in the copied file and list it in `THIRD_PARTY_NOTICES.md`. For Apache-2.0, also keep its LICENSE/NOTICE text | JobSync, career-ops, Reactive Resume, JobSpy, Muatasim job-tracker, jobseeker-analytics, Feashliaa job-board-aggregator, JobNavigator, OpenApply (MIT) · Resume-Matcher (Apache-2.0) |
| **AGPL-3.0** | Ideas only. Never vendor the code into this repo | ApplyPilot, Monica, OpenResume, pingcrm, JobCtrl, offerPilot, Twenty (mostly AGPLv3, some `@license Enterprise` files under a commercial license, and a few MIT packages; verified in its LICENSE) |
| **AGPL / commercial, as a service** | May run as a **separate self-hosted service** behind our provider interface, never vendored | Reacher `check-if-email-exists` (email verification) |
| **Commons Clause** (source-available, *not* OSI open source) | Ideas only | job-ops |
| **No license file** | Not reusable, even when a README badge claims MIT | JobMatchAI (the badge says MIT, but the API reports no license) |

**Security read before merge:** every lifted file gets reviewed before it lands (ADR-013). JobSync has a history here: **open #68** (path traversal + IDOR in resume download) and recently closed IDOR, stored-XSS and credentials-in-URL issues. Its resume-download route is **not** lifted. Backup import is checked for **zip-slip** and for per-row ownership.

**Using a tool vs copying code:** installing or enabling any of these as a tool (career-ops ships `.claude/`, `.claude-plugin/`, `.codex-plugin/` and installs via `npx`; JobSync's MCP server in Claude) needs a **SkillSpector scan first**, per the global CLAUDE.md supply-chain rule.

**Principle conflict:** JobSpy is MIT but scrapes LinkedIn, which breaks our capture-on-click
rule (ADR-007). Use it only for non-LinkedIn boards, or skip it.

## 2. The strong ones (job trackers)

| Project | ★ | License | Stack | What it is |
|---|---|---|---|---|
| [career-ops-hq/career-ops](https://github.com/career-ops-hq/career-ops) | 73.0k | MIT | Node scripts + markdown prompt "modes", run inside any AI CLI (Claude Code, Codex…); Go TUI; optional web | Agent-first pipeline: scan portals → A–H evaluation (1–5 score) → tailored CV/cover letter PDF → markdown tracker. Never submits |
| [DaKheera47/job-ops](https://github.com/DaKheera47/job-ops) | 4.0k | Commons Clause | TypeScript, Docker, orchestrator + extractors | Multi-board search (LinkedIn/Indeed/Glassdoor + 10), 0–100 fit, CV tailoring, visa-sponsor check, **Gmail post-application tracking**. Paid cloud tier |
| [Gsync/jobsync](https://github.com/Gsync/jobsync) | 1.3k | MIT | Next.js 15, React 19, shadcn, Prisma 6 + **SQLite**, Auth.js v5, Vercel AI SDK, MCP SDK | Self-hosted tracker: jobs, resumes (+PDF export), contacts ↔ jobs with roles, company pages, docked AI chat, **MCP server**, scheduled discovery automations, question bank, backup/restore |

**Honourable mentions:** [Muatasim-Aswad/job-tracker](https://github.com/Muatasim-Aswad/job-tracker)
(MIT, 10★: tiny, but it has the best capture model, see §4) ·
[JustAJobApp/jobseeker-analytics](https://github.com/JustAJobApp/jobseeker-analytics) (MIT,
202★: Gmail-driven status tracking) · [vesaias/JobNavigator](https://github.com/vesaias/JobNavigator)
(MIT, 132★) · [SergeyKhval/OpenApply](https://github.com/SergeyKhval/OpenApply) (MIT, 31★).

## 3. Adjacent building blocks

| Need | Project | ★ | License | Use |
|---|---|---|---|---|
| Resume tailoring vs JD | [srbhr/Resume-Matcher](https://github.com/srbhr/Resume-Matcher) | 28.5k | Apache-2.0 | Tailoring/cover-letter prompts, and local-LLM setup |
| Resume builder + JSON schema + PDF | [reactive-resume/reactive-resume](https://github.com/reactive-resume/reactive-resume) | 43.5k | MIT | Resume data model and export target (job-ops exports to it) |
| Resume parser | [xitanggg/open-resume](https://github.com/xitanggg/open-resume) | 8.9k | AGPL | Ideas only (its PDF parsing heuristics) |
| Job scraping | [speedyapply/JobSpy](https://github.com/speedyapply/JobSpy) | 4.4k | MIT | Indeed/Glassdoor/Google/ZipRecruiter adapters. Its LinkedIn scraping is **excluded** |
| ATS aggregation | [Feashliaa/job-board-aggregator](https://github.com/Feashliaa/job-board-aggregator) | 160 | MIT | Company-slug lists for Greenhouse/Lever/Ashby discovery |
| Auto-apply | [Pickle-Pixel/ApplyPilot](https://github.com/Pickle-Pixel/ApplyPilot) | 1.7k | AGPL | Ideas only. It is already the AutoApply design reference |
| Networking CRM | [twentyhq/twenty](https://github.com/twentyhq/twenty) | 57.7k | AGPLv3 + commercial EE files | Ideas: kanban-over-any-object views, record timeline |
| Personal CRM | [monicahq/monica](https://github.com/monicahq/monica) | 25.4k | AGPL | Ideas: relationship reminders, "last talked" |
| Personal networking CRM | [sneg55/pingcrm](https://github.com/sneg55/pingcrm) | 124 | AGPL | Ideas: inbox sync for follow-up state |
| Email verification | [reacherhq/check-if-email-exists](https://github.com/reacherhq/check-if-email-exists) | 10.0k | AGPL/commercial | Optional **separate service** behind the ADR-008 email-resolver interface |

## 4. What to borrow (concrete)

| From | Borrow | Where it lands |
|---|---|---|
| career-ops (MIT) | **Liveness check** before paying for analysis (`check-liveness.mjs`, `liveness-*.mjs`) | P5 discovery, P2 analyze-on-save |
| career-ops (MIT) | **SimHash JD fingerprint** for cross-listing and repost detection (`fingerprint-core.mjs`, `detect-reposts.mjs`), stronger than company+title+location | `jobs.fingerprint` (blueprint §5) |
| career-ops (MIT) | Posting-legitimacy block (ghost/scam signals), **work-auth hard blocker** when a JD says "no sponsorship" | Job-fit analysis output |
| career-ops (MIT) | `followup-cadence.mjs` / `followup-seed.mjs` cadence rules; `reply-watch` classification of employer replies | P4 follow-ups, P6 reply-watch |
| career-ops (MIT) | `cv-facts.json` allowlist and a fact-check pass (`verify-cv-facts.mjs`); **system vs user data contract** | Single facts source (suite map §3), export/update safety |
| career-ops (MIT) | Standalone evaluators against cheap/local models (`openai-eval`, `ollama-eval`) + a golden eval set (`eval-golden.mjs`) | `packages/ai/evals` |
| JobSync (MIT) | **MCP over Streamable HTTP inside the web app** (`src/app/api/mcp/route.ts`), hashed tokens with prefix + scopes + expiry (`McpAccessToken`), token resolver `lib/mcp/auth` | Replaces the separate `apps/mcp`. One token model for MCP **and** the Clipper |
| JobSync (MIT) | `lib/encryption.ts` + `ENCRYPTION_KEY` for stored provider API keys | Settings: email, search and LLM keys |
| JobSync (MIT) | ATS scrapers `lib/scraper/{greenhouse,lever,ashby}` + `mapper.ts`; in-process `node-cron` scheduler via `instrumentation.ts` | `packages/sources`. Scheduler option for single-user Docker |
| JobSync (MIT) | Backup/restore zip with preflight + rollback (`api/backup/*`) | Export/import (P1) |
| JobSync (MIT) | Docked AI chat panel that keeps the page usable | Copilot (P6) |
| Muatasim job-tracker (MIT) | **Never auto-merge fuzzy duplicates.** Suggest the match, the user confirms, and only exact ATS id matches auto-merge. It also has an adapter model for capture sources | Dedupe policy (ADR-005), Clipper adapters |
| job-ops / jobseeker-analytics | **Gmail reply-watch**: classify recruiter emails into status suggestions (ideas; jobseeker-analytics code is MIT) | Optional late phase (P6+) |
| Reactive Resume (MIT) | Resume JSON schema and export | Resume versions (P2), optional export |

## 5. JobSync spike: fork it or build fresh? (code-level, 2026-09-29)

Shallow clone at `C:\Users\a2021\dev\refs\jobsync` (commit `ef3d7ee`, 1,104 files). Only
read, never installed or run.

| Check | Finding | Cost for our plan |
|---|---|---|
| Kanban | **None.** No dnd library. The `sortable` matches are table headers | Build it anyway |
| Job status model | `JobStatus` table seeded `draft, applied, interview, offer, rejected, expired, archived` (+ offer-accepted/declined). There's also a separate `applied` boolean + `appliedDate` (**132 references, 20+ files**) and a `JobStage` timeline. **Changing upstream right now:** stage timeline migration on 2026-09-19, `applied` backfill on 2026-09-26 | Adding `NETWORKING`/`WITHDRAWN` and fixed columns means reworking a model upstream is still changing, so merge conflicts are guaranteed |
| Event log | `Activity` = **time tracking** (start/end, duration, breaks), used in dashboards (40 files) | Our audit log needs a new table, so a naming and semantic clash |
| Scoring | One `matchScore`/`matchData` per job (**57 references**), set by an automation's resume | Per-resume-version analyses need a UI rework across ~8 components |
| Contacts | `Contact` has no status/pipeline, no connection state, no outreach. `JobContact` with role exists | The networking CRM is almost all new, but it could land as new tables |
| API/MCP | `/api/mcp` (9 tools: add/find/update job, batch add, save match results, review resume, questions). REST is limited to AI, backup, export and automations | Good pattern to borrow. The Clipper needs new capture endpoints either way |
| Scheduler | `node-cron` in-process + Greenhouse/Lever/Ashby scrapers | Good. Worth borrowing |
| Upstream appetite | No issues requesting kanban, outreach or networking. **Open security issue #68** (path traversal + IDOR in resume download), and recently closed IDOR/XSS issues | No sign upstream wants the CRM. A fork inherits the security backlog |

**Recommendation: build fresh and lift JobSync's MIT modules with attribution** (MCP route and
token model, encryption, ATS scrapers, scheduler pattern, backup format). Forking means
rewriting the status, activity and score core that upstream is still changing. Reconsider
contributing *generic* pieces upstream (for example a kanban view) once ours is stable. This
is Richard's call; ADR-002 records it as Proposed.

## 6. What none of them do (our differentiators)

1. **A networking CRM as a first-class pipeline**: contact statuses, **connection and thread
   state**, a follow-up engine, and daily **safety caps**. At best the trackers above store a
   contact list.
2. **An F1–F6 outreach format system with a hook gate** (from `richard-outreach`). career-ops
   drafts ≤300-char notes, but with no hook gate and no format ladder.
3. **Email resolution with honest provenance** (ADR-008).
4. **A PathPeer importer**, plus the Clipper covering **people**, not only jobs.
5. One system of record for a **suite**: jobmanagement + AutoApply + ResumeScorer
   ([`suite-map.md`](suite-map.md)).

## Sources

- Web search: [pmitchell-dev/JobBoard](https://github.com/pmitchell-dev/JobBoard) ·
  [GitHub topic: application-tracker](https://github.com/topics/application-tracker) ·
  [Muatasim-Aswad/job-tracker](https://github.com/Muatasim-Aswad/job-tracker) ·
  [Dixievaseshaped907/job-tracker](https://github.com/Dixievaseshaped907/job-tracker) (⚠ likely clone of Muatasim's repo: created 2026-07-30, 0★, same description. Don't use it) ·
  [plankanban/planka](https://github.com/plankanban/planka) ·
  [GitHub topic: job-tracker](https://github.com/topics/job-tracker?o=desc&s=stars) ·
  [GitHub topic: jobtracker](https://github.com/topics/jobtracker) ·
  [GitHub topic: job-application-tracker](https://github.com/topics/job-application-tracker) ·
  [Vikunja](https://vikunja.io/)
- GitHub API (`gh api search/repositories`, `repos/*`, `readme`, `contents`) for every repo
  linked in §2–§3.
