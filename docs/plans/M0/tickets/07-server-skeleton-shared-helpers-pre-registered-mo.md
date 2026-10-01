# M0-07 · Server skeleton, shared helpers, pre-registered module stubs

Status: open
Blocked by: M0-05, M0-06
Wave: 5
Owner: agent
Model: sonnet

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

## Comments
