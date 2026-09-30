# Goal

Replace PathPeer (account archived, read-only) with an **MIT** open-source, self-hosted,
**agent-first** job-search system: jobs kanban, networking CRM, outreach log, follow-ups, a `jm` CLI
and MCP for Richard's agents, and a minimal web UI. It is the system of record for AutoApply and
ResumeScorer. This session turned the plan into a buildable spec, plans and a multi-agent harness,
based on the four attached skills and all 20 nelsonxlee articles. **No code yet.**

## Current state

- **Spec (source of truth):** `docs/spec/`
  - `SPEC.md`: requirements (REQ ids), non-goals, agent workflows
  - `architecture.md`: 3 stack options scored (Option 2 = Hono + SQLite/Drizzle + Vite SPA + `jm`
    CLI + MCP scores 66, Go 56, Next.js 49), versions verified on npm and context7 on 2026-09-29,
    style comparison, topology, layout, the six "defend it" answers
  - `data-model.md`: SQLite, one DB per owner
  - `api-rest.md`, `mcp.md`, `cli.md`: one behaviour path, with CLI and MCP as adapters
  - `security.md`: SEC-01 to SEC-21, derived from the article 006 audit
  - `optimization.md`
- **Plans:** `docs/plans/ROADMAP.md` (M0–M7), plus `M0-foundations.md` (15 full cards) and
  `M1-mvp-core.md` (21 full cards, waves ≤4 parallel, shared service signatures). M2–M7 are titles
  only, expanded just in time.
- **Harness:** `docs/harness/`
  - `README.md`: learnings → rules table, roles/models, loop, merges
  - `CONVENTIONS.md`: the Drift Loop target, dependency allowlist, module pattern, read-any/write-own rule
  - `task-card-template.md`, `gates.md` (G0–G4), `routines.md` (post-v1)
  - `prompts/`: implementer (sonnet), verifier (haiku), security-audit (opus), orchestrator
  - Root `AGENTS.md` with the "For agents" section
- **Decisions:** ADR-015 (stack, supersedes ADR-003), ADR-016 (DB per owner), ADR-017 (agents in the
  MVP), ADR-018 (harness replaces GSD here), ADR-019 (code outside OneDrive). ADR-014 amended. All
  Proposed; Richard confirms 015 and 019 before M0-02.
- **Blueprint:** now marked as the vision doc. §5/§6 point to the spec, and the §10 reuse map
  relabels ScoutLane/JobSync items as ports.
- **Verified facts:**
  - Node here is v25.6.1. `node:sqlite` works with FTS5.
  - Drizzle stable 0.45.3 has **no** node-sqlite driver, hence libsql (with `@libsql/win32-x64-msvc`).
  - MCP SDK v2 is alpha, so pin v1.31.
  - typescript-eslint supports TS <6.1, so pin TS 6.0.3 and use Biome.
  - React Router v8 isn't in the docs, so pin 7.18.4.
- A comprehension check (3 Haiku subagents on M1-05, M1-09, M1-13) found **6 blocker, 8 major and
  9 minor** issues in Run 1. A cold Run 2 on M1-05 confirmed **none recurred**, and found 1
  false-positive blocker, 5 finer majors and 3 minors. Those were addressed but not re-verified
  (no Run 3). The Run 1 fixes were:
  - `createJob` source opts
  - server-side id-prefix resolution (`resolveEntityId`)
  - the capture title/size/resolver rules
  - tags in capture
  - adapter ownership
  - `jm companies` CLI

  Evidence is in `docs/harness/comprehension-check-log.md`. The template gained "Imports you'll use"
  and "Inputs that aren't obvious" fields.
- **The repo now exists at `C:\Users\a2021\dev\jobmanagement`** (git, branch `main`), and the first
  commit holds the planning docs. The OneDrive planning folder keeps its copies plus a `MOVED.md`
  pointer. From here on, **this repo is the source of truth**. Still no code. The rest of M0-02
  (LICENSE, README, .gitignore, `.claude/agents/jm-*.md`) hasn't been done yet.

## Files in flight

None once the comprehension-check fixes are applied (see the log).

## Changed (this session)

- Added `docs/spec/*` (8 files), `docs/plans/*` (3), `docs/harness/*` (README, CONVENTIONS,
  task-card-template, gates, routines, prompts ×4, comprehension-check-log), and root `AGENTS.md`.
- Edited `docs/decisions.md` (ADR-003 superseded, ADR-014 amended, ADR-015–019 added) and
  `docs/blueprint.md` (banner, §5/§6 pointers, §10 note).

## Failed attempts

- The Firecrawl MCP key is still invalid. The GitHub MCP didn't connect this session (earlier it
  gave "Bad credentials"). `gh` CLI + WebSearch + context7 worked.
- Python heredocs containing Windows paths (`C:\Users`) fail with a unicode-escape error; use raw
  strings.
- The first M1 draft had moves in the read-side cards, which created a circular wave dependency.
  Fixed: moves and snooze live in the write-side cards M1-05/M1-06.

# Next steps

1. **URGENT:** M0-01 PathPeer export (blueprint §0 / M0-01 card). Store it outside OneDrive.
2. Richard confirms ADR-015 (stack) and ADR-019 (repo location), and answers the open decisions
   (blueprint §13 #2–#10; #2 ORM and #3 hosting are largely settled by ADR-015).
3. Build: open Claude Code (Opus) **in this repo**, paste `docs/harness/prompts/orchestrator.md`, and
   finish M0-02: the repo and the docs commit already exist, so it only needs LICENSE, README,
   .gitignore, .gitattributes, .node-version and `.claude/agents/`. Then continue with M0-03.
4. After the M1 exit: dogfood for a week, then expand M2 cards.
