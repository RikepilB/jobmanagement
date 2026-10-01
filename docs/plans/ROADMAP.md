# ROADMAP

> One milestone at a time [005]. Tickets are **one file each** under `docs/plans/<M>/tickets/` (local-markdown
> tracker, `docs/agents/issue-tracker.md`). Parallel waves run in sandcastle sandboxes (Mode B,
> `docs/harness/sandcastle.md`) from M0 wave 4. M2+ tickets are generated with the `to-tickets` skill.
> **Full cards exist only for M0 and M1.** For M2–M7 the orchestrator
> expands the titles below into full cards (`docs/harness/task-card-template.md`) at the start of
> that milestone, from `docs/spec/*`, because pre-written cards go stale. Every milestone ends with
> gate G4 (`docs/harness/gates.md`).

## Status

| Milestone | Theme | Status | Plan |
|---|---|---|---|
| M0 | Foundations: repo, toolchain, schema, contracts, server/client/MCP/CLI/web skeletons, gates | **ready to build** (needs ADR-015/019 confirmed) | [`M0-foundations.md`](M0-foundations.md) |
| M1 | Agent-first MVP: jobs, contacts, companies, links, outreach log, follow-ups, search, imports, exports via REST + CLI + MCP + simple web | ready after M0 | [`M1-mvp-core.md`](M1-mvp-core.md) |
| M2 | AI layer | titles only | below |
| M3 | Clipper extension | titles only | below |
| M4 | Networking engine | titles only | below |
| M5 | Discovery + analytics | titles only | below |
| M6 | Suite integration | titles only | below |
| M7 | OSS release | titles only | below |

## M2: AI layer (drafts, analysis, resume)
**Exit:** fit analysis and outreach drafts work over REST, CLI and MCP, and an eval suite passes.
Nothing is sent anywhere without a configured key (features stay disabled until one exists [016]).
Budget enforced (SEC-11).
- M2-00 Schema: `provider_keys`, `resume_versions`, `skills`, `job_analyses`, `contact_analyses`, `llm_usage` (serialised)
- M2-01 Allow the `openai` dep + LLM provider module (OpenRouter/Ollama/OpenAI-compatible, fallback chain; port of ScoutLane `lib/llm/openrouter.ts`)
- M2-02 Encrypted provider keys (AES-256-GCM, `JM_ENCRYPTION_KEY`) + settings UI/CLI (`jm keys set`)
- M2-03 LLM usage ledger + daily budget + cost-preview endpoint for batch runs
- M2-04 Resume versions: upload (SEC-09 upload rules), text extraction (port ScoutLane `extractText.ts`; the PDF dependency needs orchestrator approval), parse to JSON, skill bank
- M2-05 Facts source (`profile/facts` in the app, exported for skills; suite-map §3; decision #8)
- M2-06 Job-fit analysis (score, summary, tri-state qualifications, "add missing skill", stale-on-resume-change). Prompt = goal/constraint/schema [012]
- M2-07 Contact analysis (indicators, commonalities, **verifiable hooks with sources**)
- M2-08 Outreach drafts F1–F6 with the **hook gate** (no hook → no draft), ported from the `richard-outreach` rules
- M2-09 Capture fallback: LLM normalisation of plain text → `JobCreateInput`
- M2-10 Adapters (client/MCP/CLI) + web panels for analysis and drafts
- M2-11 Eval suite: synthetic JDs/profiles, schema validity, ±5 score stability, prompt-injection case (SEC-14)

## M3: Clipper (MV3 extension)
**Exit:** clip a job on Greenhouse/Lever/Ashby/Workday/LinkedIn Jobs and a LinkedIn profile on
click; the known-URL badge works; the only host permission is the API origin; it replaces Career
Clipper in the source-contacts workflow. Needs Richard's decision #6 (LinkedIn read-only exception).
- M3-01 Extension scaffold (Vite + TS, `activeTab`, `scripting`, `sidePanel`, `storage`), token setup screen
- M3-02 Job capture (JSON-LD first, visible text fallback) → `POST /capture/job` with `html`
- M3-03 Profile capture (visible text → `POST /contacts` via M2-09 normalisation or a structured form)
- M3-04 Known-URL badge via `GET /lookup`, quick status/rating/tag edits, link to job
- M3-05 Thread-state capture on LinkedIn messaging pages (user click → `PATCH /contacts/:id`)
- M3-06 CORS allowlist for the extension origin (SEC-15) + e2e with an unpacked extension on synthetic pages

## M4: Networking engine
**Exit:** process-contact "sweep" and "saved batch" modes reproduce from app data; caps ledger
enforced as warnings; email resolution with honest provenance.
- M4-01 Sweep view + tool (pending invites > N days, connected-no-reply, stale messaging)
- M4-02 Caps ledger warnings (Career OS caps, acceptance-rate guard <40% pause / <25% halt)
- M4-03 Email resolution chain: pattern learning from the owner's verified emails + MX check (ADR-008)
- M4-04 Optional paid email providers (behind the provider-key interface; Apollo/Hunter; Reacher as a separate service)
- M4-05 People suggestions: LinkedIn search shortcut URLs + optional search-API suggestions (`people_suggestions` table)
- M4-06 Follow-up cadence rules (port career-ops `followup-cadence.mjs` ideas: business days, F6 once)

## M5: Discovery + analytics
**Exit:** a scheduled scan produces deduped, liveness-checked, pre-filtered jobs; the dashboard
reconciles with raw activity queries.
- M5-00 Schema: `discovery_sources`, `discovery_runs`, `goals` (serialised)
- M5-01 Source adapters: Greenhouse/Lever/Ashby (port JobSync MIT scrapers), HN Who's Hiring, curated GitHub lists (port ResumeScorer `job_finder.py`)
- M5-02 Scheduled scan via the task runner, ETag caching, keyword/location prefilter
- M5-03 Liveness check + SimHash repost detection (career-ops ideas), suggested-duplicate review queue
- M5-04 Analytics from `activities` (conversion rates, response time, weekly execution)
- M5-05 Web dashboard + discovery feed page

## M6: Suite integration
**Exit:** AutoApply reads fit and writes outcomes over the API; ResumeScorer pushes quality scores;
Career OS shortcuts run over MCP with zero PathPeer reads.
- M6-01 AutoApply contract endpoints (`autoapply_state` transitions, answers as documents) per `docs/suite-map.md` §2
- M6-02 Resume-quality report ingest (`POST /resume-versions/:id/quality`) + a local push script for `score-resume`
- M6-03 Ship `.agents/skills/` in the repo (`jm-daily`, `jm-sweep`, `jm-capture`) as its public agent interface [016]
- M6-04 Repoint the Career OS harness + `richard-outreach` header to jm MCP tools (edits in the vault/skills-lab, with Richard's approval)
- M6-05 Voidscape transcript attach (a notes/document link with `[MM:SS]` citations; Voidscape keeps its own gates)
- M6-06 Optional Gmail reply-watch (ideas from job-ops/jobseeker-analytics), classification → suggested status changes

## M7: OSS release
**Exit:** a fresh clone reaches a working demo in under 10 minutes; the security audit is clean;
public repo.
- M7-01 `THIRD_PARTY_NOTICES.md` complete (every ported file; ADR-013)
- M7-02 Synthetic demo seed + screenshots + a short demo video script
- M7-03 Docs site: Astro Starlight, static, Cloudflare Pages (static-first [019])
- M7-04 Docker image + compose (`/data` volume, Caddy TLS example)
- M7-05 `production-readiness` + `landing-audit` skills on the docs site; head-metadata audit [017]
- M7-06 Full security audit + optional field-level encryption for contact PII (closes the SEC-08 gap)
- M7-07 Enable the four routines (`docs/harness/routines.md`) on the public repo
- M7-08 Launch write-up: the architecture thread (six questions) for X/LinkedIn [020]

## Cross-cutting timeline rule
Don't start a milestone until the previous G4 has passed **and** Richard has read the milestone
report. Report per wave, not per card [015].
