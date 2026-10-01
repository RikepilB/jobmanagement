# M0-14 · Full gates, smoke test, CI

Status: open
Blocked by: M0-02, M0-03, M0-04, M0-05, M0-06, M0-07, M0-08, M0-09, M0-10, M0-11, M0-12, M0-13
Wave: 8
Owner: agent
Model: sonnet

**Wave** 8 · **Model** sonnet · **Depends on** all M0 cards · **serialised**

**Goal.** `pnpm verify:full` exists and is green. The grep, secret and bundle gates from
`docs/harness/gates.md` run. CI is ready for when a remote exists.

**Owns.** `scripts/gates.mjs`, `scripts/secret-scan.mjs`, `scripts/smoke.mjs`, root
`package.json` (`verify:full` script only), `.github/workflows/ci.yml`,
`docs/harness/gates.md` §5 (timings row).

**Contract.** gates.md §1 exactly, including the two harness-integrity gates: `.claude/skills/*` is byte-identical to `.agents/skills/*`, and the shared paragraphs of `.sandcastle/*-prompt.md` match `docs/harness/prompts/*` (these apply once M0-16 is merged). `smoke.mjs`: temp data dir → programmatic `createOwner` +
`createToken` → `startServer` on a random port → `/healthz` 200 → `node apps/cli/dist/jm.js status
--json` with `JM_URL`/`JM_TOKEN` → MCP `tools/list` via fetch → stop → cleanup. CI: ubuntu-latest
and windows-latest, Node 24, pnpm, `pnpm install --frozen-lockfile`, `pnpm verify:full`, plus a
separate `pnpm audit --prod` job (the registry call runs in CI only, SEC-16).

**Acceptance.** `pnpm verify:full` green locally. Deliberately adding `localStorage` to a web file
makes it fail (checked, then reverted, and the check is recorded in the card report). Timings are
recorded in gates.md §5.

**Don'ts.** Don't weaken a gate to make it pass.

## Comments
