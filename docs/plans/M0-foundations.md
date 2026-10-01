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
| 3 | M0-04 environment verification spike · M0-16 sandcastle agent farm (orchestrator + Richard) | parallel (disjoint files) |
| 4 | M0-05 DB schema v1 · M0-06 core contracts + pure functions | parallel (disjoint packages). **First wave that can run in Mode B (sandcastle)** |
| 5 | M0-07 server skeleton + registries | serialised |
| 6 | M0-08 auth · M0-09 client · M0-13 task runner | parallel |
| 7 | M0-10 MCP · M0-11 CLI · M0-12 web skeleton | parallel |
| 8 | M0-14 full gates · then M0-15 exit | serialised |

## Tickets

One file per ticket (local-markdown tracker: see `docs/agents/issue-tracker.md`). The ticket file is the source of truth for its contract and `Status:`.

| ID | Title | Wave | Blocked by | Owner | File |
|---|---|---|---|---|---|
| M0-01 | Export PathPeer data (user-run runbook) | any | none | human | [`M0/tickets/01-export-pathpeer-data-user-run-runbook.md`](M0/tickets/01-export-pathpeer-data-user-run-runbook.md) |
| M0-02 | Repo bootstrap outside OneDrive | 1 | none | orchestrator | [`M0/tickets/02-repo-bootstrap-outside-onedrive.md`](M0/tickets/02-repo-bootstrap-outside-onedrive.md) |
| M0-03 | Workspace, toolchain and the full dependency set | 2 | M0-02 | agent | [`M0/tickets/03-workspace-toolchain-and-the-full-dependency-set.md`](M0/tickets/03-workspace-toolchain-and-the-full-dependency-set.md) |
| M0-04 | Environment verification spike (Windows) | 3 | M0-03 | agent | [`M0/tickets/04-environment-verification-spike-windows.md`](M0/tickets/04-environment-verification-spike-windows.md) |
| M0-05 | DB schema v1, migrations, db factory, test helper | 4 | M0-04 | agent | [`M0/tickets/05-db-schema-v1-migrations-db-factory-test-helper.md`](M0/tickets/05-db-schema-v1-migrations-db-factory-test-helper.md) |
| M0-06 | Core contracts, enums and pure functions | 4 | M0-03 | agent | [`M0/tickets/06-core-contracts-enums-and-pure-functions.md`](M0/tickets/06-core-contracts-enums-and-pure-functions.md) |
| M0-07 | Server skeleton, shared helpers, pre-registered module stubs | 5 | M0-05, M0-06 | agent | [`M0/tickets/07-server-skeleton-shared-helpers-pre-registered-mo.md`](M0/tickets/07-server-skeleton-shared-helpers-pre-registered-mo.md) |
| M0-08 | Auth: password, sessions, tokens, scopes, CSRF, rate limits | 6 | M0-07 | agent | [`M0/tickets/08-auth-password-sessions-tokens-scopes-csrf-rate-l.md`](M0/tickets/08-auth-password-sessions-tokens-scopes-csrf-rate-l.md) |
| M0-09 | Typed REST client package | 6 | M0-06, M0-07 | agent | [`M0/tickets/09-typed-rest-client-package.md`](M0/tickets/09-typed-rest-client-package.md) |
| M0-10 | MCP package + `/mcp` endpoint + stdio entry | 7 | M0-08, M0-09 | agent | [`M0/tickets/10-mcp-package-mcp-endpoint-stdio-entry.md`](M0/tickets/10-mcp-package-mcp-endpoint-stdio-entry.md) |
| M0-11 | CLI skeleton | 7 | M0-08, M0-09 | agent | [`M0/tickets/11-cli-skeleton.md`](M0/tickets/11-cli-skeleton.md) |
| M0-12 | Web skeleton | 7 | M0-07, M0-08, M0-09 | agent | [`M0/tickets/12-web-skeleton.md`](M0/tickets/12-web-skeleton.md) |
| M0-13 | Task runner | 6 | M0-07 | agent | [`M0/tickets/13-task-runner.md`](M0/tickets/13-task-runner.md) |
| M0-14 | Full gates, smoke test, CI | 8 | M0-02, M0-03, M0-04, M0-05, M0-06, M0-07, M0-08, M0-09, M0-10, M0-11, M0-12, M0-13 | agent | [`M0/tickets/14-full-gates-smoke-test-ci.md`](M0/tickets/14-full-gates-smoke-test-ci.md) |
| M0-15 | M0 exit (orchestrator) | - | M0-14 | orchestrator | [`M0/tickets/15-m0-exit-orchestrator.md`](M0/tickets/15-m0-exit-orchestrator.md) |
| M0-16 | Sandcastle agent farm (Mode B orchestration) | 3 | M0-03 | orchestrator | [`M0/tickets/16-sandcastle-agent-farm.md`](M0/tickets/16-sandcastle-agent-farm.md) |

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
