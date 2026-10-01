# M1-19 · Golden path, parity, performance, seed

Status: open
Blocked by: M1-00, M1-01, M1-02, M1-03, M1-04, M1-05, M1-06, M1-07, M1-08, M1-09, M1-10, M1-11, M1-12, M1-13, M1-14, M1-15, M1-16, M1-17, M1-18
Wave: 7
Owner: agent
Model: sonnet

**Wave** 7 · **Model** sonnet · **Depends on** all previous M1 cards · **serialised**

**Owns.** `scripts/seed.ts`, `apps/server/test/golden-path.test.ts`, `apps/server/test/parity.test.ts`,
`apps/server/test/perf.test.ts`, `scripts/smoke.mjs` (extend it only).

**Contract.**
- `scripts/seed.ts`: synthetic demo data (20 companies, 150 jobs across statuses, 300 contacts,
  links, outreach, tags). Deterministic via a fixed seed.
- **Golden path** (through `@jm/client` in-process): create job → capture html → link contact → log
  F1 → follow-ups after +7 days → move to messaging → mark replied → summary numbers match.
- **Parity:** a table listing every M1 REST route and the CLI command + MCP tool covering it. The
  test asserts that every non-delete route has both, and that the MCP `tools/list` names equal the
  mcp.md §2 list exactly.
- **Perf:** seed 1k jobs / 10k contacts in a temp DB. p95 of 20 calls each is below the budgets in
  optimization §1 (list, board, search).
- `smoke.mjs` additionally runs `jm jobs add` and `jm followups --json` against the booted server.

**Acceptance.** `pnpm verify:full` is green including these tests. Perf numbers go in the card report
and in `docs/harness/gates.md` §5.

## Comments
