# M0: Foundations (full task cards)

> **Goal:** an empty but complete, testable skeleton. Every package, registry, stub, gate and
> contract exists, so M1 cards only *fill in* their own files. Stack-specific: see
> `docs/spec/architecture.md` §2 (pins) and §5 (layout).
> **Exit criteria** (gate G4): see the end of this file.

## Waves

| Wave | Cards | Mode |
|---|---|---|
| any time | M0-01 PathPeer export (Richard + Claude in Chrome) | independent, **most time-sensitive** |
| 1 | M0-02 repo bootstrap | serialised (orchestrator) |
| 2 | M0-03 workspace + toolchain + all dependencies | serialised |
| 3 | M0-04 environment verification spike | serialised |
| 4 | M0-05 DB schema v1 · M0-06 core contracts + pure functions | parallel (disjoint packages) |
| 5 | M0-07 server skeleton + registries | serialised |
| 6 | M0-08 auth · M0-09 client · M0-13 task runner | parallel |
| 7 | M0-10 MCP · M0-11 CLI · M0-12 web skeleton | parallel |
| 8 | M0-14 full gates · then M0-15 exit | serialised |

---

### M0-01 · Export PathPeer data (user-run runbook)
**Wave** any · **Owner** Richard with Claude in Chrome (Opus main session) · **Depends on** none · **Blocks** M1-11 only

**Goal.** A complete, verified dump of Richard's own PathPeer data, stored outside OneDrive and
outside the repo, plus a PII-free *shape* file the importer can be built against.

**Read first.** `docs/blueprint.md` §0 · `docs/pathpeer-teardown.md` §2–§7.

**Produces.**
- `C:\Users\a2021\dev\jobmanagement-private\pathpeer-export-<YYYY-MM-DD>\`
  - `captures.jsonl`: one line per tRPC response `{procedure, input, capturedAt, body}` (response
    bodies only, **no request headers**), **or** `pathpeer.har` (sanitised export) as fallback
  - `resume.pdf` (downloaded through the UI)
  - `counts.md`: per-status counts read from the UI **at export time**
- In the repo (after M0-02): `docs/spec/pathpeer-export-shape.md`, listing for each captured
  procedure the JSON **key paths and value types only** (e.g. `result.data.json[].trackerStatus: string`).
  **No values.**

**Method.** Blueprint §0. First spike: install the fetch-capture hook, open one job drawer, and
confirm at least one body was captured. If nothing is captured, use the HAR fallback. Navigate with
sidebar clicks only (a full page load drops the hook).

**Acceptance.**
- Counts in the captures equal `counts.md` for every job status and contact status.
- `Select-String -Path <dir>\* -Pattern 'authorization|cookie' -SimpleMatch` → 0 matches.
- `pathpeer-export-shape.md` contains no email addresses, names or URLs (grep `@`, `linkedin.com/in`).

**Don'ts.** No token hunting. Never read storage or cookies. Nothing goes into OneDrive, and no
values go into the repo.

---

### M0-02 · Repo bootstrap outside OneDrive
**Wave** 1 · **Owner** orchestrator · **Depends on** Richard's confirmation of ADR-015 (stack) and ADR-019 (location) · **Parallel-safe with** M0-01

**Goal.** The code repo exists at `C:\Users\a2021\dev\jobmanagement` with the planning docs,
licence, ignore rules and harness agents, in one initial commit.

**Owns.** Everything at the new repo root: `docs/**` (copied from the OneDrive planning folder),
`AGENTS.md` (copied), `README.md`, `LICENSE`, `THIRD_PARTY_NOTICES.md`, `.gitignore`,
`.gitattributes`, `.editorconfig`, `.node-version`, `.claude/agents/jm-implementer.md`,
`jm-verifier.md`, `jm-security-auditor.md` (copied verbatim from `docs/harness/prompts/`),
`handoff.md`.

**Contract.**
- `git init -b main`. `LICENSE` = MIT, "Copyright (c) 2026 Richard Pillaca".
- `.gitignore`: `node_modules/`, `dist/`, `coverage/`, `.env*` + `!.env.example`, `*.db`,
  `*.db-shm`, `*.db-wal`, `data/`, `.jm/`, `*.har`, `captures.jsonl`.
- `.gitattributes`: `* text=auto eol=lf`. `.node-version`: `24`.
- `README.md` sections: what it is (2 lines) · status · quick start (`pnpm install`,
  `pnpm build`, `jm init`, `jm serve`) · **For agents** (link to AGENTS.md) · **Privacy** (what
  leaves the machine: nothing in M1) · honest data-at-rest note (SEC-08) · licence.
- The OneDrive planning folder gets a `MOVED.md` pointing to the new path. **Nothing is deleted
  there.**

**Acceptance.** `git -C C:\Users\a2021\dev\jobmanagement log --oneline` shows exactly one commit ·
`git status` clean · the three `.claude/agents/jm-*.md` files parse (frontmatter has `name`,
`description`, `tools`, `model`).

**Don'ts.** Don't push anywhere. Don't create a GitHub repo (that's Richard's call). Don't move or
delete the OneDrive folder.

---

### M0-03 · Workspace, toolchain and the full dependency set
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
  | root | – | typescript 6.0.3, @biomejs/biome 2.5.14, vitest 5.0.2, @types/node |
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

---

### M0-04 · Environment verification spike (Windows)
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

---

### M0-05 · DB schema v1, migrations, db factory, test helper
**Wave** 4 · **Model** sonnet · **Depends on** M0-04 · **Parallel-safe with** M0-06

**Goal.** `packages/db` implements `docs/spec/data-model.md` §2 (all tables marked M0) exactly,
generates migrations, and exposes a db factory with pragmas plus a test helper.

**Read first.** `docs/spec/data-model.md` (all) · `docs/harness/env-verification.md`.

**Owns.** `packages/db/**`.

**Contract.**
- `src/schema.ts`: every M0 table, column, default, unique/partial index from data-model §2.
  Enums use the **string literal unions from data-model §1**, declared locally as const arrays
  (`@jm/core` may not exist yet, so M1 later swaps these to imports in a serialised card).
- `drizzle.config.ts` (`dialect: 'sqlite'`, `schema`, `out: './migrations'`).
  `migrations/` generated by drizzle-kit **plus** one custom migration (`--custom --name fts`) with:
  CHECK constraints for enum columns, `jobs_fts`/`contacts_fts` FTS5 tables, map tables and triggers
  (data-model §2 "Full-text search").
- `src/client.ts`: `createDb({ url }): Db` (libsql or the M0-04 fallback) that sets the 4 pragmas on
  open. `migrateDb(db)` applies `migrations/` via the driver's migrator.
- `src/testing.ts`: `createTestDb(): Promise<Db>` (temp file or `:memory:`, migrated). Exported as
  `@jm/db/testing`.
- `src/index.ts` exports `schema`, `createDb`, `migrateDb`, type `Db`.

**Acceptance.** `pnpm --filter @jm/db test` covers:
- A fresh DB migrates, and every table exists (`sqlite_master` query).
- Inserting a job with an unknown `company_id` fails (FK).
- Inserting an invalid `applications.status` fails (CHECK).
- Inserting a job then `SELECT … FROM jobs_fts WHERE jobs_fts MATCH 'title'` finds it, and after an
  update of the title the old term no longer matches.
- `canonical_url` partial unique: two NULLs allowed, two equal URLs rejected.

`pnpm verify` green.

**Don'ts.** No tables from later milestones. No `user_id` columns (ADR-016). Never edit a generated
migration after committing it.

---

### M0-06 · Core contracts, enums and pure functions
**Wave** 4 · **Model** sonnet · **Depends on** M0-03 · **Parallel-safe with** M0-05

**Goal.** `packages/core` holds every zod contract named in `docs/spec/api-rest.md`, the enums
from data-model §1, and the pure helpers M1 needs (signatures + tests). No I/O.

**Read first.** `docs/spec/api-rest.md` (all) · `docs/spec/data-model.md` §1, §3, §4, §5.

**Owns.** `packages/core/**`.

**Contract.**
- `src/enums.ts`: const arrays + zod enums for every enum in data-model §1.
- `src/contracts/<module>.ts` for `common` (Paginated, ErrorBody, ErrorCode, Scope, Actor, Cursor),
  `auth`, `companies`, `jobs`, `capture`, `contacts`, `links`, `outreach`, `followups`,
  `activities`, `tags`, `notes`, `search`, `imports`, `exports`, `settings`, `summary`,
  `system`. Every input/output schema **named in api-rest** exists with that exact name, and input
  objects are `.strict()`.
- `src/ids.ts`: `newId()` (ULID). `src/time.ts`: `startOfDayInTz(ms, tz)`, `addDays(ms, n)`,
  `parseLocalDate('YYYY-MM-DD', tz)`.
- `src/normalize.ts`: `normalizeName`, `normalizeUrl` (data-model §4 rules),
  `normalizeLinkedinProfileUrl` (→ `https://www.linkedin.com/in/<slug>` or null), `registrableDomain`
  (simple: host minus `www.`; no public-suffix list in M1, recorded as a known limitation).
- `src/fingerprint.ts`: `jobFingerprint({atsProvider, atsJobId, canonicalUrl})` per data-model §4
  (sha256 via `node:crypto`, the only allowed Node import in core).
- `src/status-rules.ts`: `applicationTransition(prev, next, now)` and
  `contactTransition(prev, next, now, settings)` returning patches (data-model §5).
- `src/cursor.ts`: `encodeCursor({k, id})` / `decodeCursor(s)` (base64url JSON, validated).

**Acceptance.** `pnpm --filter @jm/core test` covers:
- `normalizeUrl` strips `utm_*`, `gh_src`, fragment, `www.` and the trailing slash, and sorts query params.
- `jobFingerprint` priority ats > url > manual.
- **Every row** of data-model §5 (both tables).
- `contactTransition` into `contacted` with existing `next_follow_up_at` keeps it.
- `startOfDayInTz` across a DST change in `America/Toronto`.
- Cursor round-trip and rejection of a tampered cursor.
- Each contract module exports every schema name listed in api-rest (one test iterates a name list).

**Decision table: `normalizeLinkedinProfileUrl`**
| Input | Output |
|---|---|
| `https://ca.linkedin.com/in/some-slug/?trk=x` | `https://www.linkedin.com/in/some-slug` |
| `linkedin.com/in/Some-Slug` | `https://www.linkedin.com/in/some-slug` |
| `https://www.linkedin.com/company/acme` | `null` (not a profile) |
| empty / non-URL | `null` |

**Don'ts.** No DB, fetch or fs imports. No default exports. Don't invent fields that api-rest doesn't
list. If a shape is ambiguous, report BLOCKED with the exact question.

---

### M0-07 · Server skeleton, shared helpers, pre-registered module stubs
**Wave** 5 · **Model** sonnet · **Depends on** M0-05, M0-06 · **serialised**

**Goal.** A bootable Hono server with every cross-cutting piece and **all 17 module folders
pre-stubbed and registered**, so M1 cards never edit a registry.

**Read first.** `docs/spec/architecture.md` §4–§6 · `docs/spec/api-rest.md` §0 ·
`docs/spec/security.md` SEC-01, 07, 10, 15, 18 · `docs/harness/CONVENTIONS.md` §3.

**Owns.** `apps/server/src/**` and `apps/server/tsup.config.ts`. This card creates the stub files in
`auth/` and `tasks/`. After it merges, ownership of those two directories transfers to M0-08 and
M0-13, and every `modules/<name>/` folder transfers to its M1 card.

**Contract.**
- `config.ts`: zod-parsed env → `Config {port, host, dataDir, publicUrl?, corsOrigins[], insecureOk}`.
  The default host is `127.0.0.1`.
- `app.ts`: `createApp(deps: {db, config, now?: () => number}) → Hono`. Middleware order: request-id →
  host allowlist (SEC-18) → `secureHeaders` (SEC-07) → CORS only for `corsOrigins` (SEC-15) →
  error handler (`AppError` → `{error}` JSON, zod errors → 400 `bad_request`) → auth (from M0-08
  stub: currently rejects everything except public routes) → module routers at `/api/v1` →
  `/mcp` mount point (stub route returning 501 until M0-10) → `/healthz` → static SPA from
  `apps/web/dist` with `index.html` fallback for non-API GETs.
- `shared/define-route.ts`: `defineRoute({method, path, scope: Scope \| 'public', handler, input?})`,
  `defineRoutes(module, routes[])`, and a registry of route metadata used by the SEC-01 test.
- `shared/errors.ts` (`AppError`, `ErrorCode`), `shared/pagination.ts` (keyset helpers using core
  cursor), `shared/activity.ts` (`recordActivity(tx, {entityType, entityId, type, fromValue?,
  toValue?, summary, payload?, actor, occurredAt})`), `shared/resolve-id.ts`
  (`resolveEntityId(db, table, idOrPrefix) → string`: a full ULID is returned as-is if it exists, a prefix ≥ 6
  chars does `WHERE id LIKE ? || '%' LIMIT 11`. 0 rows → `AppError('not_found')`, >1 → `AppError('conflict',
  …, {candidates})`, <6 chars → `bad_request`), `shared/context.ts` (per-request ctx: `now()`,
  `actor`, `scopes`), `shared/safe-fetch.ts` (**stub** exporting the signature
  `safeFetch(url, opts)` that throws `not implemented`; owned by M1-09).
- `modules/<name>/routes.ts` for all 17 modules (architecture §5): each exports
  `routes = defineRoutes('<name>', [])`, except `system`, which implements `GET /system/status`.
  `modules/index.ts` imports all 17 (**the only registry, written once here**).
  Empty `service.ts` and `repo.ts` stubs per module (`export {}`), plus
  `modules/imports/tasks.ts` = `export const handlers: Record<string, TaskHandler> = {}`. Also
  stub files `auth/middleware.ts` (rejects everything except `public` routes) and `tasks/index.ts`
  (`export {}`). From wave 6 those two directories are owned by M0-08 and M0-13.
- `main.ts`: `startServer(config)` → `createDb` → `migrateDb` → `createApp` → `serve({fetch, port,
  hostname})`, with graceful shutdown on SIGINT/SIGTERM. Also exported for the CLI (`jm serve`).

**Acceptance.** `pnpm --filter @jm/server test`:
- `GET /healthz` → 200 `{ok:true}`.
- `GET /api/v1/system/status` without auth → 401 `unauthorized` JSON shape.
- Request with `Host: evil.example` → rejected (SEC-18).
- Security headers present on `/healthz`.
- A thrown `AppError('conflict')` inside a test route → 409 with the error body.
- `recordActivity` inserts a row with the given fields (service-level test with `createTestDb`).
- `resolveEntityId`: a full id, a unique 6-char prefix, an ambiguous prefix (409 with candidates) and a
  5-char prefix (400).
- **`route-inventory.test.ts`** (SEC-01): every `app.routes` entry under `/api/v1` has registry
  metadata with a scope.

`pnpm verify` green.

**Don'ts.** No business logic in module stubs. No auth logic here (M0-08). Don't serve the SPA for
`/api/*` or `/mcp`.

---

### M0-08 · Auth: password, sessions, tokens, scopes, CSRF, rate limits
**Wave** 6 · **Model** sonnet · **Depends on** M0-07 · **Parallel-safe with** M0-09, M0-12, M0-13

**Goal.** SEC-01/02/03/04/17/19 implemented, plus the `auth` module routes.

**Read first.** `docs/spec/security.md` · `docs/spec/api-rest.md` §0, §1, §14 · data-model §2 (owner, sessions, api_tokens).

**Owns.** `apps/server/src/auth/**`, `apps/server/src/modules/auth/**`.

**Contract.**
- `auth/password.ts`: `hashPassword`, `verifyPassword` (argon2id via `@node-rs/argon2`). Minimum
  12 characters.
- `auth/owner.ts`: `createOwner(db, {displayName, password})` (fails if one exists),
  `setOwnerPassword(db, password)` (revokes all sessions). Used by the CLI's `jm init` and `jm reset-password`.
- `auth/tokens.ts`: `createToken(db, {name, scopes, expiresInDays?}) → {info, token}` where
  `token = 'jm_' + base64url(32 random bytes)` and only the sha256 is stored. Also `verifyToken`,
  `revokeToken`, `listTokens`, and `last_used_at` updates (at most once a minute).
- `auth/sessions.ts`: create (cookie secret, 14-day sliding), verify, revoke, revokeAll.
- `auth/middleware.ts`: resolves cookie **or** bearer, sets `ctx.scopes`/`ctx.actor`, and enforces
  the route's declared scope (403/401). Cookie sessions get all scopes. The `X-JM-Actor` header is
  accepted only from the allowed enum.
- `auth/rate-limit.ts`: in-memory token buckets per api-rest §14, plus the login lockout.
- CSRF: Hono `csrf()` for cookie-authenticated non-GET requests.
- `modules/auth/routes.ts`: `POST /auth/login`, `POST /auth/logout`, `GET /auth/me`, `GET/POST
  /tokens`, `DELETE /tokens/:id`, each writing activities per SEC-19.

**Acceptance.** Tests:
- The stored hash starts with `$argon2id$`.
- Wrong password → 401. The sixth attempt within a minute → 429 with `Retry-After`.
- Login sets an HttpOnly, SameSite=Lax cookie.
- A cookie POST with a foreign `Origin` → 403.
- A token with `read` scope can GET and gets 403 on POST.
- A revoked token → 401. After logout, the old cookie → 401.
- The token plaintext is returned exactly once (`GET /tokens` never contains it).
- Activities exist for login and for token create/revoke.

**Don'ts.** Never log passwords or tokens. No JWTs (server-side revocable sessions only). No
unauthenticated owner-creation route.

---

### M0-09 · Typed REST client package
**Wave** 6 · **Model** sonnet · **Depends on** M0-06, M0-07 · **Parallel-safe with** M0-08, M0-12, M0-13

**Goal.** `@jm/client`: one typed client used by the CLI, MCP and web. Module files are pre-stubbed.

**Owns.** `packages/client/**`.

**Contract.**
- `createClient({ baseUrl, token?, fetch?, actor? }) → JmClient`. `fetch` is injectable, so tests
  and `/mcp` pass `app.request`-backed fetch. It sends `Authorization` when a token is given,
  `X-JM-Actor`, and `credentials: 'include'` for the web.
- Errors: non-2xx → throws `JmApiError {status, code, message, details}` parsed from the error
  body. Network errors → `JmApiError {status: 0, code: 'network'}`.
- `src/<module>.ts` for the same 17 modules, each exporting a factory `(http) => ({ … })`. In M0,
  only `system.status()` and `auth.me()`, `auth.login()`, `auth.logout()`, `tokens.list/create/revoke`
  are implemented. The rest are empty objects typed with `{}`, which M1 adapter cards fill in.
  `src/index.ts` assembles them (**written once here**).
- `paginateAll(fetchPage)` async iterator helper for `--all`.

**Acceptance.** Tests against a real `createApp` + `createTestDb` via an injected fetch
(`(url, init) => app.request(url, init)`): `system.status()` with a valid token returns data. A bad
token throws `JmApiError` with `status 401`, `code 'unauthorized'`. `paginateAll` follows cursors
(tested with a fake page function).

**Don'ts.** No business logic, no caching, no retries except none. Don't redeclare contract types;
import them from `@jm/core`.

---

### M0-10 · MCP package + `/mcp` endpoint + stdio entry
**Wave** 7 · **Model** sonnet · **Depends on** M0-08, M0-09 · **Parallel-safe with** M0-11

**Goal.** `createMcpServer(client)` with pre-stubbed tool modules, served statelessly at `/mcp`
(bearer only) and runnable over stdio.

**Read first.** `docs/spec/mcp.md` §0–§1 · SEC-14/18/20.

**Owns.** `packages/mcp/**`, `apps/server/src/mcp-route.ts` (replacing the 501 stub; the stub file
path is fixed by M0-07).

**Contract.**
- `packages/mcp/src/index.ts`: `createMcpServer(client: JmClient): McpServer` (SDK v1:
  `import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js'`). Registers tools from
  `src/tools/<module>.ts` (17 files, each exporting `register(server, client)`; empty except
  `system`). **Written once here.**
- `src/tools/system.ts`: `system_status` only. (`pipeline_summary` lives in `tools/summary.ts`,
  filled by M1-15.)
- `src/format.ts`: helpers `ok(data, summaryText)`, `fail(err)`, and
  `untrusted(text) → {content_trust:'untrusted', text}` (SEC-14).
- `apps/server/src/mcp-route.ts`: `POST /mcp` requires a bearer token (a cookie is **not** accepted).
  Per request: an in-process client (`fetch` = `app.request` with the caller's token), then
  `createMcpServer`, then `new WebStandardStreamableHTTPServerTransport({ sessionIdGenerator:
  undefined })` from `@modelcontextprotocol/sdk/server/webStandardStreamableHttp.js` (**verify
  this import path in the installed SDK; if it differs, report BLOCKED with the path you found**),
  then `transport.handleRequest(c.req.raw)`. GET/DELETE → 405.
- `packages/mcp/src/stdio.ts`: `runStdio(client)` using `StdioServerTransport`.

**Acceptance.** Test: POST a JSON-RPC `initialize` then `tools/list` to `/mcp` with a token → the
list contains `system_status`. Without a token → 401. `tools/call system_status` returns
`structuredContent.version`. No tool name matches `/delete|remove_all/` (SEC-20).

**Don'ts.** No sessions (stateless). No business logic in tools. Don't use SDK v2 packages.

---

### M0-11 · CLI skeleton
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

---

### M0-12 · Web skeleton
**Wave** 7 · **Model** sonnet · **Depends on** M0-07, M0-08 (login route), M0-09 · **Parallel-safe with** M0-10, M0-11

**Goal.** A Vite React SPA with login, layout, a router with **all M1 pages pre-stubbed**, and the
API hooks layer. It builds into `apps/web/dist` for the server to serve.

**Read first.** SPEC §4 REQ-UI-01 · CONVENTIONS §7 · `docs/spec/api-rest.md` §1.

**Owns.** `apps/web/**`.

**Contract.**
- Routes (react-router 7, `createBrowserRouter`): `/login`, `/` (Today), `/jobs/board`, `/jobs`,
  `/jobs/:id`, `/contacts/board`, `/contacts`, `/contacts/:id`, `/companies`, `/companies/:id`,
  `/activity`, `/search`, `/settings`. Each page is `src/pages/<name>/index.tsx` exporting a stub
  `<h1>`. The route table is **written once here**.
- `src/lib/api.ts`: a `createClient({ baseUrl: '/', fetch: window.fetch })` singleton (cookie auth)
  plus a QueryClient. On 401, redirect to `/login`.
- `src/components/`: `Layout` (nav links to every page), `Table`, `Empty`, `ErrorBox`, `Loading`,
  `StatusBadge`. Plain Tailwind.
- Login page posts `auth.login` and redirects to `/`.
- `vite.config.ts`: dev proxy `/api` and `/mcp` → `http://127.0.0.1:4747`.

**Acceptance.** `pnpm --filter @jm/web build` succeeds · the gzip size of the main chunk is reported
and is < 300 KB · a unit test renders `Layout` (react-dom/server `renderToString`, so no jsdom
needed) and asserts that every nav link exists · a grep gate shows no `localStorage`.

**Don'ts.** No component library, no CSS framework besides Tailwind, no auth token storage.

---

### M0-13 · Task runner
**Wave** 6 · **Model** sonnet · **Depends on** M0-07 · **Parallel-safe with** M0-08, M0-09

**Goal.** A reliable in-process background runner on the `tasks` table.

**Read first.** data-model §2 `tasks` · optimization §3.

**Owns.** `apps/server/src/tasks/**`.

**Contract.**
- `enqueue(db, {type, payload, runAt?, maxAttempts?}) → taskId`.
- `registerHandler(type, fn)`. A handler registry file `tasks/handlers.ts` imports
  `modules/imports/tasks.ts` (a stub `export const handlers = {}` **created by M0-07**, later owned
  by M1-11) and registers each entry. **Written once here.**
- `startRunner({db, now, pollMs=2000, concurrency=2, leaseMs=300000}) → {stop()}`: claims due tasks
  atomically (`UPDATE … WHERE status='queued' AND run_at<=? OR (status='running' AND locked_until<?)
  RETURNING`), runs the handler, and writes `result` or `last_error`. Retries use exponential
  backoff (`run_at = now + 2^attempts * 10s`) until `max_attempts`, then `failed`.
- `getTask(db, id)`.

**Acceptance.** Tests with a fake clock:
- A queued task runs once and ends `done` with its result.
- A throwing handler retries, then ends `failed` after `max_attempts` with `last_error`.
- A stale lease is re-claimed.
- Two runners never run the same task (simulate with two claim calls).

**Don'ts.** No external queue libraries. No `setInterval` without `stop()`.

---

### M0-14 · Full gates, smoke test, CI
**Wave** 8 · **Model** sonnet · **Depends on** all M0 cards · **serialised**

**Goal.** `pnpm verify:full` exists and is green. The grep, secret and bundle gates from
`docs/harness/gates.md` run. CI is ready for when a remote exists.

**Owns.** `scripts/gates.mjs`, `scripts/secret-scan.mjs`, `scripts/smoke.mjs`, root
`package.json` (`verify:full` script only), `.github/workflows/ci.yml`,
`docs/harness/gates.md` §5 (timings row).

**Contract.** gates.md §1 exactly. `smoke.mjs`: temp data dir → programmatic `createOwner` +
`createToken` → `startServer` on a random port → `/healthz` 200 → `node apps/cli/dist/jm.js status
--json` with `JM_URL`/`JM_TOKEN` → MCP `tools/list` via fetch → stop → cleanup. CI: ubuntu-latest
and windows-latest, Node 24, pnpm, `pnpm install --frozen-lockfile`, `pnpm verify:full`, plus a
separate `pnpm audit --prod` job (the registry call runs in CI only, SEC-16).

**Acceptance.** `pnpm verify:full` green locally. Deliberately adding `localStorage` to a web file
makes it fail (checked, then reverted, and the check is recorded in the card report). Timings are
recorded in gates.md §5.

**Don'ts.** Don't weaken a gate to make it pass.

---

### M0-15 · M0 exit (orchestrator)
**Goal.** G4 for M0: verify the exit criteria below with evidence, run the security auditor
(partial: SEC-01/02/03/04/05/06/07/15/17/18/20 apply now), update the handoff, and report to
Richard.

## M0 exit criteria (each needs evidence)
1. The repo is at `C:\Users\a2021\dev\jobmanagement`, `git log` is clean, and the MIT licence is present.
2. `pnpm verify:full` is green on Windows, with timings recorded.
3. `jm init` → `jm serve` → web login works → `jm status --json` works → `claude mcp add jm -- jm mcp`
   lists `system_status` in a real Claude Code session.
4. All 17 modules are stubbed in the server, client, MCP, CLI and web registries (a list diff shows
   17/17 in each).
5. `docs/spec/pathpeer-export-shape.md` exists (M0-01), **or** M0-01 is explicitly deferred by
   Richard.
6. The security audit shows no open CRITICAL/HIGH for the SEC items applicable in M0.
