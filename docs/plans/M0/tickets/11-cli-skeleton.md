# M0-11 · CLI skeleton

Status: open
Blocked by: M0-08, M0-09
Wave: 7
Owner: agent
Model: sonnet

**Wave** 7 · **Model** sonnet · **Depends on** M0-08, M0-09 · **Parallel-safe with** M0-10

**Goal.** A `jm` binary with local commands (`init`, `serve`, `migrate`, `backup`,
`reset-password`), client commands (`login`, `status`, `tokens …`, `mcp`), output/exit-code
helpers, and pre-stubbed command modules.

**Read first.** `docs/spec/cli.md` (all).

**Owns.** `apps/cli/**`.

**Contract.**
- `src/main.ts`: a commander program registering `src/commands/<module>.ts` for all 17 modules
  (**written once here**), plus `local.ts` (init/serve/migrate/backup/reset-password) and `mcp.ts`.
- `src/config.ts`: read/write `~/.jm/config.json` (`os.homedir()`), with `JM_URL`/`JM_TOKEN`
  overrides.
- `src/output.ts`: `printResult(result, {json})` (json → `JSON.stringify(body)`, else a table) and
  `exitFor(error)` using the codes in cli.md §0.
- `jm init`: prompts for the password twice without echo (use `node:readline` with muted output),
  calls `createOwner`, `createToken({name:'cli', scopes:['read','write','capture','admin']})`, and
  writes the config. `jm serve` calls `startServer`. `jm backup` runs `VACUUM INTO`.
- `jm mcp`: `runStdio(createClient(config))`.
- `tsup.config.ts` builds `dist/jm.js` with a `#!/usr/bin/env node` banner. `package.json`
  `bin: { jm: 'dist/jm.js' }`.

**Acceptance.** Tests (commander invoked in-process with a mock client and a temp HOME dir):
- `jm status --json` prints the exact REST body.
- An unknown id → exit 2. A 401 → exit 3.
- `jm tokens revoke x` without `--yes` → exit 1 and nothing called.
- `jm init` on an existing owner → refuses.
- After `pnpm --filter @jm/cli build`, `node dist/jm.js --help` exits 0.

**Don'ts.** Never print the token after `init`/`tokens create` except that single creation output.
No interactive prompts in client commands (agents run them).

## Comments
