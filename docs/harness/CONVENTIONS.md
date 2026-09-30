# CONVENTIONS: rules every agent follows (the Drift Loop target)

> Read before every card. **Append-only:** when review finds a deviation, the orchestrator adds a
> numbered rule under §9 with the date and the card that exposed it. Rules are hard ("never X",
> "always Y"), never "try to keep it clean" [008].

## 1. Stack (pinned: see `docs/spec/architecture.md` §2)
- TypeScript 6.0 strict (`strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`). ESM
  only. Node ≥ 24.
- Lint and format: **Biome only**. Don't add ESLint or Prettier.
- Server: Hono 4 on `@hono/node-server`. DB: Drizzle 0.45 + `@libsql/client` (or the M0-04
  fallback). Validation: zod 4. MCP: `@modelcontextprotocol/sdk` 1.31 (v1 API: `McpServer`,
  `registerTool(name, {title, description, inputSchema: {…zod shape…}}, handler)`).
- Web: React 19 + Vite 8 + react-router 7 + TanStack Query 5 + Tailwind 4 + `@dnd-kit/core`.
  No component library.
- CLI: commander 15. Build: tsup. Tests: Vitest 5.

## 2. Dependency allowlist (SEC-16)
Only the packages pinned in `architecture.md` §2 plus their `@types/*`. **Implementers never edit
any `package.json` or the lockfile.** If a card seems to need a package, report BLOCKED with the
package name, the reason, and an alternative that uses existing deps. The orchestrator decides and
adds it in a serialised card after a local review (maintainer, licence, install scripts). A registry
audit needs Richard's confirmation.

## 3. Server module pattern
```
apps/server/src/modules/<name>/
  routes.ts    export const routes = defineRoutes('<name>', [defineRoute({ method, path, scope, input?, handler })])
  service.ts   export functions (db, input, ctx) => result      ← the only cross-module entry point
  repo.ts      Drizzle queries for this module's tables only
  <name>.test.ts  tests through app.request() (HTTP level) + service-level tests
```
- A handler: validate (zod from `@jm/core/contracts/<name>`) → call a service → `c.json({ data })`.
  No SQL in handlers.
- A service: pure business rules plus `db.transaction(async tx => …)`. **Every mutation calls
  `recordActivity(tx, …)` inside the same transaction.**
- **Reads may join any table. Writes only touch your own module's tables.** A repo may *read* any
  table (joins for cards and details). To *write* another module's table, call that module's
  `service.ts`. Never import another module's `repo.ts`.
- Shared read projections (`JobCard`, `ContactCard`, tag names per entity) live in
  `apps/server/src/shared/projections.ts` (M1-00). Use them instead of re-writing the joins.
- Table ownership for writes: companies→`companies` · jobs→`jobs`, `applications`,
  `job_sightings` · contacts→`contacts`, `contact_emails` · links→`job_contacts` ·
  outreach→`outreach_messages` · followups→`reminders` · tags→`tags`, `taggings` · notes→`notes` ·
  settings→`owner.settings` · auth→`owner`, `sessions`, `api_tokens` · imports→`external_refs` (plus
  the others through their services) · shared→`activities` (`recordActivity`), `tasks` (runner).
- Errors: throw `AppError(code, message, details?)` from `shared/errors.ts`. Never return ad-hoc
  error JSON.
- Every `:id` path param goes through `resolveEntityId(db, table, param)` (`shared/resolve-id.ts`)
  before use, so id prefixes work everywhere (api-rest §0).
- Time: `ctx.now()` (injected clock), never `Date.now()` inside services, so tests control time.
- IDs: `newId()` from `@jm/core` (ULID). Never `Math.random`.

## 4. Contracts
- Request and response types come **only** from `packages/core/src/contracts/*`. Web, CLI, client
  and MCP import them from there and never redeclare them.
- If a contract seems wrong or missing a field: **stop and report BLOCKED**. Don't edit
  `packages/core/src/contracts/*` from a feature card (it's serialised).

## 5. Testing
- Test behaviour through the public surface: HTTP via `app.request()` for routes, exported functions
  for services and core.
- Assert **actual output values** (status codes, bodies, DB rows). **No snapshot tests. No tests that
  only check that a function was called. Coverage % is not a goal** [012].
- Every test gets a **fresh in-memory or temp-file DB** from `@jm/db/testing` (`createTestDb()`), with
  migrations applied. No shared state between tests.
- Fixtures are **synthetic** ("Contact A", "Company X", `example.com`). Never real people's data.
- Name tests after behaviour: `it('moving to applied sets applied_at once')`.

## 6. Security coding rules (from `docs/spec/security.md`)
- Never `sql.raw` outside `packages/db/migrations`. Never `child_process` in `apps/server/src`.
  Never `dangerouslySetInnerHTML` in `apps/web/src`.
- Never read a filesystem path taken from request input.
- Outbound HTTP from the server goes **only** through `shared/safe-fetch.ts`.
- Never log secrets, tokens, passwords, cookies or full request bodies. Log ids and counts.
- The web app never touches `localStorage`/`sessionStorage` for auth.
- Any future inbound webhook: verify the signature, dedupe on the event id, respond 200 fast and do
  the work in a task [009].

## 7. Web UI rules (design is not a priority)
- Plain semantic HTML + Tailwind utilities. One neutral palette. No custom animation.
- Every page has an empty state, a loading state and an error state (the error message comes from
  the API error `message`).
- Third-party text (descriptions, about, bodies) renders in a `<pre class="whitespace-pre-wrap">` or
  plain paragraphs, never as HTML.
- Data access only through `src/lib/api.ts` (TanStack Query hooks wrapping `@jm/client`).

## 8. Working rules for implementers
- Touch only the files your card **owns**. Need another file? Report BLOCKED.
- Don't refactor, rename or "improve" anything outside your card. If you spot a bug elsewhere, write
  it under "Noticed" in your report and don't fix it.
- Don't invent missing requirements, fixtures, env vars or API fields. Report BLOCKED [017].
- Don't write step-by-step comments. Comments explain *why*, not *what*.
- Finish with the card's acceptance commands, then `pnpm verify`, and paste the tail of the output
  into your report.

## 9. Drift-loop rules (append below; newest last)
<!-- Format: R<n> (YYYY-MM-DD, card <ID>): <hard rule>. Why: <one line>. -->
R1 (2026-09-30, planning): Routes are registered only via `defineRoute` with a scope. Why: SEC-01
inventory test.
R2 (2026-09-30, planning): Offset pagination is banned. Use keyset `(sortKey, id)`. Why: 100× load
(architecture §8).
