# M0-04 · Environment verification spike (Windows)

Status: open
Blocked by: M0-03
Wave: 3
Owner: agent
Model: sonnet

**Wave** 3 · **Model** sonnet · **Depends on** M0-03 · **serialised**

**Goal.** Evidence that the chosen drivers work on this machine, before any schema work.

**Owns.** `scripts/env-check.ts`, `docs/harness/env-verification.md`.

**Contract.** `pnpm tsx scripts/env-check.ts` prints PASS/FAIL for:
1. `@libsql/client` opens `file:<tmp>/check.db`.
2. `PRAGMA journal_mode=WAL` returns `wal`.
3. `PRAGMA foreign_keys=ON` enforced (inserting an orphan row fails).
4. `CREATE VIRTUAL TABLE t USING fts5(x)` + a `MATCH` query returns the row.
5. `@node-rs/argon2` hash + verify round-trip.
6. Node version printed.

`env-verification.md` records the output, the date and the machine, and a **decision line**:
"libsql OK", or "FTS5 missing in libsql → switch to better-sqlite3 13.0.3 (orchestrator adds it via
a serialised card; schema unaffected)".

**Acceptance.** The script exits 0 and prints 6 PASS lines, **or** the decision line names the
fallback and the orchestrator applies it before wave 4.

**Don'ts.** Don't change the schema plan. Don't add packages yourself.

## Comments
