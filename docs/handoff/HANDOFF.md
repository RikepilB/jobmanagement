# HANDOFF (father): jobmanagement

**Read this first.** It's the whole-project handoff: freshest state on top, then an append-only index
of every session. Session folders are immutable. Only `## Current state` is replaced each session.

```
docs/handoff/
  HANDOFF.md            ← this file: current state + session index
  .current-session      ← active session folder name (used by the hooks)
  _meta/                ← TEMPLATE.md · LATER-TASKS.md
  <YYYY-MM-DD>-<name>/  ← one folder per session: HANDOFF.md (+ transcript.md, snapshots)
```

## Current state (2026-09-30)
- **Phase:** planning complete, **no app code yet**. Repo `C:\Users\a2021\dev\jobmanagement`:
  - `main` @ `3d23246` (docs)
  - branch `docs/sandcastle-mattpocock-harness` @ `b4d7696` (sandcastle + Matt Pocock skills + per-ticket tracker), not merged
  - no remote
- **Source of truth:**
  - `docs/spec/` (product + contracts)
  - `docs/plans/` (ROADMAP; M0 has 16 tickets, M1 has 21, in `docs/plans/<M>/tickets/`)
  - `docs/harness/` (orchestrator, Sonnet implementers, Haiku verifier; Mode A worktrees / Mode B sandcastle)
  - `docs/decisions.md` (ADR-001..020)
- **Stack (ADR-015, proposed):** Hono + SQLite (Drizzle/libSQL) + Vite React SPA + `jm` CLI + MCP (SDK v1).
  MIT licence (ADR-011, accepted).
- **Blocking / urgent:**
  - The PathPeer export (M0-01): the account is archived and read-only.
  - Richard to confirm ADR-015/020.
- **Pending (interrupted request):** GitHub repo, Vercel deploy, bulk CSV/PDF/docs import, PathPeer data.
  Triaged in `_meta/LATER-TASKS.md`. Not started.
- **Next move:** M0-01 export, then build M0 with `docs/harness/prompts/orchestrator.md`.

## Session index (append-only, newest first)
- 2026-09-29 → 2026-09-30 · [`2026-09-29-pathpeer-replacement-planning`](2026-09-29-pathpeer-replacement-planning/HANDOFF.md): PathPeer teardown, landscape, full spec + plans + lean harness, sandcastle + Matt Pocock skills (commits `3d23246`, `b4d7696`).

<!-- compact-handoff:auto-snapshot -->
