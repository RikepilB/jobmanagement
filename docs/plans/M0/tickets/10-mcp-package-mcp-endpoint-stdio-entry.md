# M0-10 · MCP package + `/mcp` endpoint + stdio entry

Status: open
Blocked by: M0-08, M0-09
Wave: 7
Owner: agent
Model: sonnet

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

## Comments
