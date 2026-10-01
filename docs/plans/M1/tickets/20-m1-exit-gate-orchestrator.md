# M1-20 · M1 exit gate (orchestrator)

Status: open
Blocked by: M1-19
Wave: -
Owner: orchestrator
Model: -

**Goal.** Gate G4 for M1, then hand the product to Richard for dogfooding. The criteria to evidence are the **M1 exit criteria** in `docs/plans/M1-mvp-core.md` (end of file).

**Does.**
1. `pnpm verify:full` green, with evidence.
2. `jm-security-auditor` full run (SEC-01…21). Fix or get explicit acceptance for every
   CRITICAL/HIGH.
3. The README gets the "For agents" section (AGENTS.md summary + connect commands) and the privacy
   + data-at-rest statements.
4. Runbook for Richard: `jm import pathpeer <export-dir> --dry-run`, then the real run, and compare
   counts with `counts.md`.
5. Drift review of CONVENTIONS, then handoff (`handoff.md` + a `docs/handoff/` session entry).
6. Propose enabling the routines (`docs/harness/routines.md`) once a GitHub remote exists (Richard
   decides).

## Comments
