# Decisions (ADR log)

Append-only. Status is **Proposed** until Richard reviews it. Context lives in
[`blueprint.md`](blueprint.md) and [`pathpeer-teardown.md`](pathpeer-teardown.md).

## 2026-09-29: ADR-001 Candidate-side, single-user-first, self-hostable OSS
**Status:** Proposed. **Decision:** Build a personal job-search system, not a coaching
platform. Drop PathPeer's organization/coach/client tenancy. Every table carries `user_id`, so
multi-user needs no schema rewrite. **Why:** Richard is the user, and self-hosting keeps
third-party PII under his control.

## 2026-09-29: ADR-002 Fresh scaffold, lift ScoutLane modules, don't fork
**Status:** Proposed. **Update 2026-09-29:** the JobSync (MIT) fork option was spiked at the
code level (`landscape.md` §5). **Recommendation stands: build fresh**, now also lifting
JobSync's MIT modules with attribution. Forking would mean reworking its `applied`-boolean
status coupling (132 references), its time-tracking `Activity`, and its single `matchScore`
(57 references), all while upstream is actively migrating that same model. Richard decides.
**Decision:** Start clean and copy specific modules (kanban,
extractText, openrouter client, job runner, storage tiers). **Why:** ScoutLane is employer-side
(Organization, Applicant, per-job PipelineStage, careers board), and its status is derived from
stage names (unknown names silently become REVIEWING). This product needs fixed 7-value enums
for both jobs and contacts.

## 2026-09-29: ADR-003 Stack: Next.js 16 + Postgres + Prisma 7 + pg-boss, pnpm monorepo
**Status:** **Superseded by ADR-015 (2026-09-30).** **Decision:** Modular monolith. `apps/web`,
`apps/extension` and `apps/mcp` share `packages/schemas` (Zod). **Why:** The same stack as the
main reuse donor. Postgres is the only stateful dependency. **Alternative:** Drizzle (findleads)
is lighter and serverless-friendly. Revisit before P0 schema work.

## 2026-09-29: ADR-004 Services are the single source; REST v1 for non-UI clients
**Status:** Proposed. **Decision:** UI uses server actions. The extension, MCP and importer use
`/api/v1` with personal access tokens. Both call the same services. **Why:** An extension can't
call server actions, and parity by construction avoids ScoutLane's documented API parity gap.

## 2026-09-29: ADR-005 Identity vs sighting for jobs; CRM-preserving upserts
**Status:** Proposed. **Decision:** `jobs` (identity, fingerprint-unique) + `job_sightings`
(every source hit) + `applications` (tracking state). Contact upserts never overwrite status,
rating, notes or outreach fields. **Why:** Re-scans and multi-source duplicates must never wipe
tracking state (findleads DATA-01 lesson).

## 2026-09-29: ADR-006 Two different scores
**Status:** Proposed. **Decision:** *Job fit* = job × resume version (`job_analyses`, stale on
resume change). *Resume quality* ("ATS score") = resume version against a rubric
(`resume_versions.quality_score`, via ResumeScorer/hiring-agent). **Why:** PathPeer's
"compatibility %" is job fit. Mixing the two makes both meaningless.

## 2026-09-29: ADR-007 Clipper extension is its own trust boundary
**Status:** Proposed. **Decision:** MV3 with `activeTab` + `scripting` + `sidePanel`. The only
host permission is the user's API origin. Capture happens on user click, current tab only.
JSON-LD first, then an LLM normalizes visible text. No LinkedIn automation, no cookies. Not
built inside agent-bridge. **Why:** LinkedIn ToS, Voidscape's "never reads browser secrets"
contract, and agent-bridge's separate ADR. **Conflict to resolve:** Career OS says "never
inject DOM scripts on LinkedIn". The Clipper needs a scoped exception for a read-only
`innerText` extract on user click (no writes, no synthetic events). This needs Richard's
confirmation. The fallback is pasting profile text.

## 2026-09-29: ADR-008 Email resolution chain with honest confidence
**Status:** Proposed. **Decision:** manual → domain-pattern learning → MX check → optional
paid providers. SMTP RCPT probing is off by default. Store provider, method, confidence and
checked_at. **Why:** Reproduce PathPeer's "verified work email" feature without the
deliverability and ethics problems of probing.

## 2026-09-29: ADR-009 Draft-only outreach with a hook gate
**Status:** Proposed. **Decision:** The system drafts (F1–F6 from `richard-outreach`) and the
human sends. No draft without a verifiable hook. Daily caps from Career OS are shown as a
ledger. **Why:** Account safety and message quality. PathPeer's generic intro messages had no
hook.

## 2026-09-29: ADR-010 Drop Roadmap, Feedback, coach features; defer Careers
**Status:** Proposed. **Why:** They were coach-driven or PathPeer-specific. Careers
(target-role narratives) is optional in P5.

## 2026-09-29: ADR-011 License: MIT
**Status:** Accepted (Richard, 2026-09-29). **Decision:** jobmanagement is MIT-licensed open
source. **Consequences:** Code may be copied only from MIT/Apache-2.0 projects, keeping their
notices (ADR-013). AGPL, Commons Clause and unlicensed projects are ideas only.

## 2026-09-29: ADR-012 One fit score and one system of record for the suite
**Status:** Proposed. **Decision:** jobmanagement owns the only job-fit score (`job_analyses`)
and the only application ledger. AutoApply reads fit over `/api/v1` and writes outcomes back
(`applications.autoapply_state`). ResumeScorer's `jobs.json` is imported as `job_sightings`.
**Why:** AutoApply's design had its own fit prompt and ledger. Two diverging scores and two
ledgers would contradict each other. Details: [`suite-map.md`](suite-map.md).

## 2026-09-29: ADR-013 Third-party code gate
**Status:** Proposed. **Decision:** Every copied file keeps its original copyright header and
is listed in `THIRD_PARTY_NOTICES.md` (source repo, commit, license). Apache-2.0 files also
keep the NOTICE text. AGPL tools (Reacher) may only run as separate services behind a
provider interface. **Every lifted file gets a security read before merge.** JobSync's
resume-download route is not lifted (open #68: path traversal + IDOR), and backup import is
checked for zip-slip and ownership. **Why:** The repo must remain cleanly MIT. The license table is in
[`landscape.md`](landscape.md) §1.

## 2026-09-29: ADR-014 MCP served inside the web app; one token model
**Status:** Proposed. **Amended 2026-09-30 by ADR-015:** "web app" now means the Hono server (`/mcp`, stateless Streamable HTTP, SDK v1), plus a stdio entry `jm mcp`. Both come from one `createMcpServer(client)`. **Decision:** Serve MCP over Streamable HTTP at `/api/mcp` from the
Next.js app (the JobSync pattern). Do not ship a separate `apps/mcp` process. The Clipper,
MCP clients and AutoApply share one hashed-token model (prefix, scopes, expiry,
last_used). Stored provider API keys are encrypted with `ENCRYPTION_KEY`. **Why:** One
process and one auth path, and it is proven in an MIT project. This supersedes the PAT and
`apps/mcp` details in ADR-004 and blueprint §3.2 and §9.

## 2026-09-30: ADR-015 Stack: API-first TypeScript core (Hono + SQLite/Drizzle + Vite SPA + `jm` CLI + MCP)
**Status:** Proposed. **Richard confirms.** **Decision:** Option 2 in `docs/spec/architecture.md`
(weighted score 66 vs Next.js 49 vs Go 56). Pins are in architecture §2: TS 6.0, Hono 4.13, Drizzle
0.45 + @libsql/client 0.18, MCP SDK 1.31 (v1), react-router 7, Vite 8, Vitest 5, Biome. **Why:**
agents are first-class clients (one REST path adapted by CLI and MCP), zero-ops self-host on Windows,
buildable by simpler models (no RSC or server-action magic), and portable to Cloudflare D1 later
(to verify). **Cost:** code from ScoutLane and JobSync becomes *ports*, not lifts. No Postgres
features. **Supersedes** ADR-003.

## 2026-09-30: ADR-016 Tenancy: one SQLite database per owner
**Status:** Proposed. **Decision:** no `user_id` columns. A second user means a second database file.
**Why:** removes the most common authorization bug class (missing per-row owner checks), keeps
queries simple for simpler models, and matches the DB-per-tenant shape of libSQL/D1. **Amends**
ADR-001 (which required `user_id` everywhere).

## 2026-09-30: ADR-017 Agents are first-class from the MVP
**Status:** Proposed. **Decision:** CLI (`jm`) and MCP ship in M1 with parity to REST (except deletes
over MCP). The in-app "Copilot" is dropped: Richard's own agents connect over MCP. **Why:** Richard's
stated core requirement ("connect my agents through CLI or MCP"). **Changes** the blueprint roadmap
(MCP was P6).

## 2026-09-30: ADR-018 The lean harness replaces GSD for this repo
**Status:** Proposed. **Decision:** `docs/harness/` (one orchestrator, Sonnet implementers in
worktrees, Haiku verifier, Opus security auditor, gates G0–G4, Drift Loop into CONVENTIONS) is the
only planning and execution system here. **Why:** the lessons from the Nelson Lee articles
(subtraction, draft PRs, one thing at a time) plus two competing planning systems would confuse
simpler models.

## 2026-09-30: ADR-019 Code repo lives outside OneDrive
**Status:** Proposed. **Richard confirms.** **Decision:** code at `C:\Users\a2021\dev\jobmanagement`,
data dir `~/.jm`, PathPeer export in `C:\Users\a2021\dev\jobmanagement-private`. **Why:** OneDrive
corrupts venvs, git objects and live SQLite files on this machine (ResumeScorer PROJECT.md). Parallel
worktrees and `node_modules` make that worse.
