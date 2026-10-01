# M0-03 · Workspace, toolchain and the full dependency set

Status: open
Blocked by: M0-02
Wave: 2
Owner: agent
Model: sonnet (or orchestrator)

**Wave** 2 · **Model** sonnet (or orchestrator) · **Depends on** M0-02 · **serialised**

**Goal.** A pnpm workspace with seven packages, strict TypeScript, Biome, Vitest projects, and
**every M1 dependency installed once at the pinned versions**, with `pnpm verify` green on trivial
tests.

**Read first.** `docs/spec/architecture.md` §2 (pins), §5 (layout) · `docs/harness/CONVENTIONS.md` §1–§2.

**Owns.** Root: `package.json`, `pnpm-workspace.yaml`, `pnpm-lock.yaml`, `tsconfig.base.json`,
`tsconfig.json`, `biome.json`, `vitest.config.ts`, `.env.example`. Per package
(`apps/server`, `apps/web`, `apps/cli`, `packages/core`, `packages/db`, `packages/client`,
`packages/mcp`): `package.json`, `tsconfig.json`, `vitest.config.ts`, `src/index.ts`,
`src/smoke.test.ts`. Web also owns `index.html`, `vite.config.ts`, `src/main.tsx`.

**Contract.**
- Root `package.json`: `"private": true`, `"type": "module"`, `"packageManager": "pnpm@10.30.3"`,
  `"engines": {"node": ">=24"}`, scripts `verify` = `biome check . && tsc -b && vitest run`,
  `verify:full` = `pnpm verify && echo "verify:full lands in M0-14"`, `build` = `pnpm -r build`,
  `test` = `vitest run`.
- Package names: `@jm/core`, `@jm/db`, `@jm/client`, `@jm/mcp`, `@jm/server`, `@jm/cli`, `@jm/web`.
  Internal deps use `workspace:*`.
- Dependencies (exact versions from architecture §2, no `^`):

  | Package | deps | devDeps |
  |---|---|---|
  | core | zod, ulid | – |
  | db | @jm/core, drizzle-orm, @libsql/client | drizzle-kit |
  | client | @jm/core | – |
  | mcp | @jm/core, @jm/client, @modelcontextprotocol/sdk, zod | – |
  | server | @jm/core, @jm/db, @jm/client, @jm/mcp, hono, @hono/node-server, @hono/zod-validator, @node-rs/argon2, drizzle-orm, zod | tsup, tsx |
  | cli | @jm/core, @jm/client, @jm/mcp, @jm/server, @jm/db, commander | tsup |
  | web | @jm/core, @jm/client, react, react-dom, react-router, @tanstack/react-query, @dnd-kit/core | vite, @vitejs/plugin-react, tailwindcss, @tailwindcss/vite, @types/react, @types/react-dom |
  | root | – | typescript 6.0.3, @biomejs/biome 2.5.14, vitest 5.0.2, @types/node, **@ai-hero/sandcastle 0.12.0**, **tsx 4.23.15** (for the Mode B wave runner; intake: `docs/harness/intake/sandcastle.yaml`) |
- `tsconfig.base.json`: `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`,
  `module`/`moduleResolution` = `NodeNext` (web: `Bundler`), `verbatimModuleSyntax`, `composite`
  per package for `tsc -b`.
- Root `vitest.config.ts`: `test.projects: ['./apps/*/vitest.config.ts', './packages/*/vitest.config.ts']`.
  Each package config uses `defineProject` with a `name`. Web uses the `jsdom` environment
  **only if** a jsdom-dependent test exists (none in M0, so node).
- `.env.example`: `JM_PORT=4747`, `JM_HOST=127.0.0.1`, `JM_DATA_DIR=`, `JM_PUBLIC_URL=`,
  `JM_CORS_ORIGINS=`, `JM_INSECURE_OK=`.

**Acceptance.** `pnpm install` completes on Windows with no native build errors · `pnpm verify`
green (7 smoke tests) · `pnpm ls -r --depth 0` shows exactly the pinned versions.

**Don'ts.** No ESLint/Prettier/Jest. No `^` ranges. No packages outside the allowlist. If a pinned
version fails to install, **stop and report** the error text. Don't pick another version.

## Comments
