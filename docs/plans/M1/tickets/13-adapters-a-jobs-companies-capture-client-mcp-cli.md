# M1-13 · Adapters A: jobs, companies, capture (client + MCP + CLI)

Status: open
Blocked by: M1-02, M1-05, M1-07, M1-09
Wave: 4
Owner: agent
Model: sonnet

**Wave** 4 · **Model** sonnet · **Depends on** M1-02, M1-05, M1-07, M1-09 · **Parallel-safe with** M1-11, M1-12, M1-14

**Goal.** REQ-AGT-01/02/03 for jobs, companies and capture.

**Read first.** `docs/spec/mcp.md` §1–§2 · `docs/spec/cli.md` §0, §2 · api-rest §0, §2–§4 · M0-09 (client
factory: `createClient`, per-module `(http) => ({…})` files, `JmApiError`) · M0-10 (`format.ts`: `ok`,
`fail`, `untrusted`) · M0-11 (`output.ts`: `printResult`, `exitFor`; command registration).

**Owns.** `packages/client/src/{jobs,companies,capture}.ts`, `packages/mcp/src/tools/{jobs,capture}.ts`,
`apps/cli/src/commands/{jobs,companies}.ts`, plus their test files. (`jm jobs capture` is a subcommand in
`commands/jobs.ts`. `commands/capture.ts` and `tools/companies.ts` stay as the empty M0 stubs, because M1
has no top-level `jm capture` command and no `companies_*` MCP tools: companies are reachable through
`search` and job/contact details.)

**Contract.**
- Client methods mirror every route (`jobs.list(query)`, `jobs.board()`, `jobs.get(id)`,
  `jobs.create(input)`, `jobs.patch(id, p)`, `jobs.move(id, s)`, `jobs.untrack(id)`,
  `jobs.remove(id)`, `companies.*`, `capture.job(input)`).
- MCP tools exactly as in mcp.md §2 for `jobs_*` (in `tools/jobs.ts`) and `jobs_capture_url` (in
  `tools/capture.ts`). `descriptionMd` is wrapped with `untrusted()` from `packages/mcp/src/format.ts`.
  The summary format follows mcp.md §1 (id/title/company/status).
- CLI `jm companies list|get|add|set` exactly as in cli.md §2.
- CLI commands exactly as in cli.md §2 for `jm jobs …`. The `capture` subcommand lives in `jobs`
  per cli.md (`jm jobs capture`).

**Acceptance.** Tests: each client method against a real app in-process · MCP `tools/call jobs_move`
changes the status (verified via a client `get`) · `jobs_get` output marks the description untrusted ·
CLI `jm jobs list --status applied --json` prints the REST body · `jm jobs move <6-char-prefix> applied`
works (the prefix is resolved by the API) · an ambiguous prefix → exit 4 with the candidates printed.

## Comments
