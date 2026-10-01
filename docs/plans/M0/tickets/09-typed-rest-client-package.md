# M0-09 · Typed REST client package

Status: open
Blocked by: M0-06, M0-07
Wave: 6
Owner: agent
Model: sonnet

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

## Comments
