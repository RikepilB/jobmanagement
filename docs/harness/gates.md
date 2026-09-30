# Gates

> A gate reports **PASS / FAIL / NOT FOUND**, with evidence (command output tail or file:line).
> Never PASS without evidence [006]. The commands are created in M0-03 and M0-14.

## 1. Commands

| Command | Runs | Created by |
|---|---|---|
| `pnpm verify` | `biome check .` → `tsc -b` → `vitest run` (all packages) | M0-03 |
| `pnpm verify:full` | `pnpm verify` → `pnpm -r build` → `node scripts/smoke.mjs` (boots the server on a temp data dir and a random port, checks `/healthz`, `jm status --json`, MCP `tools/list`) → `node scripts/gates.mjs` (grep gates + secret scan + web bundle size) | M0-14 |
| `pnpm --filter <pkg> test -- <file>` | one package / one file while iterating | M0-03 |

`scripts/gates.mjs` grep gates (each must return 0 hits):
- `sql.raw` outside `packages/db/migrations/**` (SEC-09)
- `child_process` in `apps/server/src/**` (SEC-09)
- `dangerouslySetInnerHTML` in `apps/web/src/**` (SEC-09)
- `localStorage` / `sessionStorage` in `apps/web/src/**` (SEC-06)
- `Date.now()` in `apps/server/src/modules/**/service.ts` (CONVENTIONS §3: use `ctx.now()`)
- `toMatchSnapshot` / `toMatchInlineSnapshot` anywhere (CONVENTIONS §5)
- secret patterns in tracked files (SEC-05)
- web `dist` main bundle > 300 KB gzip (optimization §1)

## 2. Gate levels

| Gate | When | Who | Pass condition |
|---|---|---|---|
| **G0 card-local** | before the implementer reports DONE | implementer | the card's acceptance commands + `pnpm verify` green |
| **G1 verify** | after DONE | verifier (haiku) | independent re-run of G0 on the card branch. `git diff --name-only main...HEAD` ⊆ the card's Owns. No `package.json`/lockfile diff unless the card owns them. No test breaks CONVENTIONS §5 |
| **G2 merge** | after G1 PASS | orchestrator | `git merge --no-ff` then `pnpm verify` on `main` is green, else revert |
| **G3 wave** | after every card in a wave is merged | orchestrator | `pnpm verify:full` green. Wave report written |
| **G4 milestone** | end of a milestone | orchestrator + security auditor (opus) | exit criteria in the milestone plan each have evidence. Security audit has no CRITICAL/HIGH open (or Richard accepted each explicitly). `handoff.md` + `docs/handoff/` updated. Drift rules reviewed |

## 3. Verifier report format
```
CARD <ID> · branch card/<ID>-<slug> · commit <sha>
| Check | Status | Evidence |
|---|---|---|
| acceptance: <behaviour/test name> | PASS/FAIL/NOT FOUND | <output line or file:line> |
| pnpm verify | PASS/FAIL | <last 5 lines> |
| files ⊆ Owns | PASS/FAIL | <offending paths> |
| no dependency changes | PASS/FAIL | |
| test rules (no snapshot / was-called-only) | PASS/FAIL | <file:line> |
VERDICT: PASS | FAIL (<one line why>)
```

## 4. When a gate fails (debug-evidence-loop protocol)
1. **Reproduce** with the exact failing command, and save the minimal output.
2. **Isolate**: the smallest test or input that fails.
3. **One hypothesis at a time**: state it, test it, record the result.
4. Fix only once the cause is evidenced, in the card's branch, within its owned files. If the cause
   is outside the card, the card is BLOCKED and the orchestrator owns the fix.
5. Never "fix" a gate by weakening it (deleting a test, adding a lint ignore, widening Owns) without
   a CONVENTIONS rule explaining why.

## 5. Recorded timings
| Date | `pnpm verify` | `pnpm verify:full` | Machine |
|---|---|---|---|
| (filled by M0-14) | | | |
