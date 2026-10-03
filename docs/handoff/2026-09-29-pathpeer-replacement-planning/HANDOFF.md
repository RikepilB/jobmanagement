# Session: 2026-09-29-pathpeer-replacement-planning

## Goal
Plan an MIT, open-source, agent-first replacement for PathPeer (jobs kanban + networking CRM + `jm`
CLI/MCP), with specs, plans and a multi-agent build harness simple enough for smaller models.
Spanned 2026-09-29 → 2026-09-30. Planning only, no app code.

## What was done
- Read-only PathPeer teardown via Claude in Chrome → `docs/pathpeer-teardown.md` (modules, enums, tRPC map; no third-party PII).
- Blueprint and vision → `docs/blueprint.md`. Skills audit → `docs/skills-audit.md` (`richard-outreach` hook gate tested: PASS).
- OSS landscape (career-ops, job-ops, JobSync, etc.) + licence gate + JobSync code spike (verdict: build fresh, port MIT pieces) → `docs/landscape.md`.
- Suite contract with AutoApply/ResumeScorer (one fit score, one system of record) → `docs/suite-map.md`.
- Read all 20 nelsonxlee articles + 4 skills. Lessons → harness rules (`docs/harness/README.md` §1).
- Spec set → `docs/spec/` (SPEC, architecture with 3 stack options scored 66/56/49 → Hono + SQLite/Drizzle + Vite SPA + `jm` CLI + MCP, data-model, api-rest, mcp, cli, security SEC-01..22, optimization). Library pins verified on npm + context7.
- Plans → `docs/plans/ROADMAP.md` + M0 (16 tickets) + M1 (21 tickets), one file per ticket under `docs/plans/<M>/tickets/`.
- Lean harness → `docs/harness/` (roles, CONVENTIONS + drift loop, gates G0–G4, prompts, routines, comprehension-check log). Haiku comprehension checks: Run 1 6B/8M/9m fixed, Run 2 0 recurrences.
- Repo created outside OneDrive → `C:\Users\a2021\dev\jobmanagement`, commit `3d23246` on `main`.
- Sandcastle + Matt Pocock skills adopted (ADR-020): SkillSpector intake (raw CRITICAL → APPROVE_WITH_RESTRICTIONS, `docs/harness/intake/`), `docs/harness/sandcastle.md`, ticket M0-16, 5 pinned skills in `.claude/skills` + `.agents/skills` → commit `b4d7696` on branch `docs/sandcastle-mattpocock-harness`.
- ADR-001..020 in `docs/decisions.md`. Memory note `jobmanagement-project.md` kept current.

## Files changed
- Everything under `docs/`, plus `AGENTS.md`, `handoff.md`, `.claude/skills/*`, `.agents/skills/*` (repo).
- OneDrive planning folder: `MOVED.md` (pointer). The rest there is now stale.
- Outside repos: `~/.claude/projects/C--Users-a2021-OneDrive/memory/jobmanagement-project.md`;
  vault `00_Inbox/2026-09-30_jobmanagement-pathpeer-replacement.md`;
  read-only reference clones in `C:\Users\a2021\dev\refs\{jobsync,sandcastle}`.

## Failed attempts
- PathPeer tRPC direct `fetch` → 401 (auth isn't cookie-only). Not pursued (no token hunting).
- The Firecrawl MCP key is invalid. The GitHub MCP gave bad credentials or timed out. Used the `gh` CLI + WebSearch + context7.
- Bash heredocs collapse `\\` → Python unicode-escape errors on `C:\Users` paths. Write scripts with the Write tool, or use forward slashes.
- The session header claimed "git repository: true" for the OneDrive folder. It isn't one; the repo is at `C:\Users\a2021\dev\jobmanagement`.

## Next steps
1. **URGENT:** M0-01 PathPeer export (Richard + Claude in Chrome), stored in `C:\Users\a2021\dev\jobmanagement-private\`.
2. Richard confirms ADR-015 (stack) and ADR-020, and the open blueprint §13 decisions. Merge `docs/sandcastle-mattpocock-harness` → `main` (fast-forward).
3. Triage the interrupted request in `docs/handoff/_meta/LATER-TASKS.md` (GitHub repo, Vercel, bulk CSV/PDF import, PathPeer data). Each outward-facing step needs explicit confirmation.
4. Build: open Claude Code (Opus) in the repo, paste `docs/harness/prompts/orchestrator.md`, finish M0-02, then M0-03 → M0-16 (sandcastle farm: Richard installs Node in WSL + `claude setup-token`).

## Files in this folder
- `HANDOFF.md`: this digest
- `transcript.md`: not captured yet. Richard: `/export C:\Users\a2021\dev\jobmanagement\docs\handoff\2026-09-29-pathpeer-replacement-planning\transcript.md`
