# Architecture: stack options, recommendation, runtime shape

> Source of truth for **how the system is built**. Product behaviour lives in
> [`SPEC.md`](SPEC.md). Contracts: [`data-model.md`](data-model.md), [`api-rest.md`](api-rest.md),
> [`mcp.md`](mcp.md), [`cli.md`](cli.md). Status: **the recommendation is provisional until Richard
> confirms it** (ADR-015). Stack-specific detail lives only in this file, `docs/plans/M0-foundations.md`
> and `docs/harness/CONVENTIONS.md`, so a stack switch touches only those three.

## 1. What the architecture must serve

1. **Agents are first-class clients.** Richard's agents (Claude Code, Codex, Claude Desktop) drive the
   product through a **CLI** (`jm`) and **MCP**. The web UI is just another client. It is deliberately
   simple.
2. **Single owner, self-hosted, MIT, open source.** It runs on Richard's Windows 11 machine first, and
   on Linux/macOS/Docker for other users.
3. **Buildable by simpler models in parallel.** Explicit layers, few framework conventions, contracts
   in one place, tests that run in seconds.
4. **Cheap at any scale.** No platform whose bill grows with traffic by default (Nelson Lee, "cost
   shape" argument [article 019]).
5. **Defensible end to end.** Every major choice answers the six portfolio questions (§8). This repo is
   also a public portfolio piece.

## 2. Three stack options

| | **Option 1: Next.js full-stack** | **Option 2: API-first TypeScript core** ✅ | **Option 3: Go single binary** |
|---|---|---|---|
| Server | Next.js 16 App Router, route handlers + server actions | **Hono 4** on Node (`@hono/node-server`) | Go 1.2x, chi or echo |
| DB | PostgreSQL (Docker or Neon) + Drizzle/Prisma | **SQLite file** (libSQL driver) + Drizzle ORM | SQLite (pure-Go `modernc.org/sqlite`, no CGO) + sqlc |
| Jobs/queue | pg-boss or `after()` | In-process runner on a `tasks` table | In-process runner or River |
| Web UI | React Server Components | **React 19 + Vite SPA** served by the same process | Embedded Vite SPA, or templ + htmx |
| CLI | Separate Node package | **`jm` Node CLI** sharing the typed client | Same binary (cobra) |
| MCP | Route handler (JobSync pattern) | **`/mcp` Streamable HTTP + `jm mcp` stdio**, one tool module | mcp-go in the same binary |
| Extension (M3) | TS | TS, shares Zod contracts | TS, needs JSON-Schema codegen for contracts |
| Deploy | Docker / Vercel | `jm serve` (no Docker needed) / Docker / later Cloudflare Workers + D1 (**to verify**) | Download one `.exe` |

### Scoring (1–5, weighted)

| Criterion (weight) | Opt 1 | Opt 2 | Opt 3 | Why |
|---|---|---|---|---|
| Self-host on Windows / Docker (×3) | 3 | **5** | 5 | Opt 1 needs a Postgres server. Opt 2 is one process + one file. Opt 3 is one exe |
| CLI + MCP connectivity (×3) | 4 | **5** | 5 | Opt 2: CLI, MCP and web all use one typed client over one REST API |
| Buildable by a simpler model (×3) | 3 | **5** | 3 | RSC, server actions and caching are classic traps. Go + a TS SPA + TS extension means two languages and duplicated contracts |
| Reuse of MIT and own code (×2) | 5 | 4 | 1 | Opt 2 *ports* the React kanban, pure-TS parsers, the JobSync scrapers and career-ops logic |
| Cost shape at scale (×1) | 3 | 5 | 5 | Opt 2 runs anywhere, including edge later |
| Performance (×1) | 3 | 4 | 5 | Single-user workloads are tiny, so nobody loses here |
| Portfolio signal (×1) | 3 | 4 | 5 | Opt 3 signals Go. Opt 2 signals API-first design + a defensible schema |
| **Weighted total** | **49** | **66** | **56** | |

**Recommendation: Option 2.** Choose Option 3 instead only if the priority becomes "non-developers
download one file and run it". Option 1 loses on buildability and self-host simplicity even though it
has the most reuse.

### Pinned versions (observed on npm 2026-09-29; M0-03 installs exactly these)

| Package | Pin | Note |
|---|---|---|
| Node.js | **24 LTS** recommended (`engines: >=24`); 25.6.1 is installed here and works | `node:sqlite` works on 25 but is experimental, so it's not used |
| pnpm | 10.30.3 | `packageManager` field |
| typescript | **6.0.3** | 7.0 exists, but typescript-eslint and other tooling cap at <6.1 |
| @biomejs/biome | 2.5.14 | Lint + format in one tool, no ESLint |
| hono / @hono/node-server / @hono/zod-validator | 4.13.11 / 2.1.3 / 0.9.1 | |
| drizzle-orm / drizzle-kit | 0.45.3 / 0.31.11 | Stable 0.45 has **no** `node-sqlite` driver (1.0 RC only) |
| @libsql/client | 0.18.0 | N-API prebuilds incl. `@libsql/win32-x64-msvc`. **FTS5 availability is verified in M0-04**; fallback is better-sqlite3 13.0.3 |
| zod | 4.6.5 | Shared contracts |
| @modelcontextprotocol/sdk | **1.31.0 (v1.x stable)** | v2 (`@modelcontextprotocol/server`) is alpha. Don't use it |
| commander | 15.0.0 | CLI |
| @node-rs/argon2 | 2.2.1 | N-API, no node-gyp on Windows |
| react / react-dom | 19.3.0 | |
| react-router | **7.18.4** | v8 exists but isn't in current docs. Pin 7.x |
| @tanstack/react-query | 5.104.0 | |
| @dnd-kit/core | 6.3.1 | Column moves only (no in-column sorting) |
| vite / @vitejs/plugin-react | 8.3.1 / 6.1.1 | |
| tailwindcss / @tailwindcss/vite | 4.3.3 | Plain utility classes, no component library |
| vitest | 5.0.2 | Peer `vite ^8` OK |
| tsx / tsup | 4.23.15 / 8.5.1 | dev runner / build for server + CLI |
| ulid | 3.0.2 | Sortable ids |
| openai (M2) | 7.25.0 | OpenAI-compatible client for OpenRouter / Ollama / OpenAI |

## 3. Architecture style

| Style | Verdict | Reason |
|---|---|---|
| Plain monolith (no boundaries) | ✗ | Parallel agents collide in shared files, and nothing stops cross-module reach-ins |
| **Modular monolith** | ✅ **chosen** | One process and one DB, with hard module boundaries (`modules/<name>/`). Pre-registered module stubs let parallel agents own disjoint files |
| Microservices | ✗ | One user, one machine. Network boundaries add auth, retries and distributed consistency that no one needs, and that a simpler model can't reason about |
| Serverless / edge-only | ✗ for v1 | Long LLM calls, PDF parsing and local files fit a process better. Kept as a *deploy target* for later (Cloudflare, to verify) |
| **Hybrid** | ✅ for the edges | Core = modular monolith. Docs/landing = static Astro site on Cloudflare (M7). The worker can run as a second role of the same binary (`jm serve --role worker`) if LLM jobs ever block requests |

**Split rule:** don't create a new service. First try a module; then a second *role* of the same
process. Only a measured problem justifies a separate deployable.

## 4. Runtime topology

```
            Richard's agents                                  Richard
   ┌─────────────────────────────────┐              ┌────────────────────────┐
   │ Claude Code / Codex / Desktop   │              │ Browser (SPA)  Chrome  │
   │   ├─ MCP over HTTP ─────────────┼──┐           │   │ cookie      Clipper│(M3)
   │   └─ MCP stdio: `jm mcp` ───┐   │  │           └───┼──────────────┼─────┘
   │ Shell: `jm jobs list --json`│   │  │               │              │ bearer token
   └─────────────────────────────┼───┘  │               │              │
                                 ▼      ▼               ▼              ▼
                        ┌───────────────────────────────────────────────────────┐
                        │ `jm serve`  (one Node process, 127.0.0.1:4747)          │
                        │  Hono app                                               │
                        │   /api/v1/*  REST (Zod-validated)  ◄── the only behaviour│
                        │   /mcp       MCP Streamable HTTP (stateless) ─┐          │
                        │   /healthz   /*  static SPA                    │ in-process│
                        │   middleware: request-id · auth (cookie|bearer) · csrf ·│
                        │               rate-limit · error → problem JSON        │
                        │  modules/<name>/{routes,service,repo}.ts               │
                        │  task runner (tasks table, lease + retry)              │
                        │  Drizzle ─► SQLite file  <dataDir>/jm.db (WAL, FTS5)   │
                        └───────────────────────────────────────────────────────┘
```

- **One behaviour path.** MCP tools and CLI commands are thin adapters over the **typed REST client**
  (`packages/client`). Inside the server, `/mcp` builds its client on `app.request()`
  (in-process, no network hop). `jm mcp` builds the same MCP server on a real HTTP client. So there is
  one implementation of every rule: the REST route and its service.
- **Data dir:** `JM_DATA_DIR` (default `~/.jm`, which is `C:\Users\<you>\.jm` on Windows) holds `jm.db`,
  `backups/`, `files/` (M2 resumes). **Keep it out of OneDrive**, because sync corrupts live SQLite files.
- **Bind 127.0.0.1** by default. Exposing it beyond localhost needs an explicit `--host 0.0.0.0` and
  HTTPS (SEC-07).

## 5. Repository layout (pnpm workspace)

Code lives at **`C:\Users\a2021\dev\jobmanagement`**, outside OneDrive (ADR-019).

```
jobmanagement/
├─ AGENTS.md                  agent contract + "For agents" (read first)
├─ apps/
│  ├─ server/                 Hono app + task runner + migrations on boot
│  │  └─ src/
│  │     ├─ main.ts           entry: config → db → app → serve
│  │     ├─ app.ts            createApp(deps): middleware + module registry + /mcp + static
│  │     ├─ config.ts         env → typed config (zod)
│  │     ├─ auth/             sessions, tokens, password, middleware, rate-limit
│  │     ├─ shared/           activity.ts (recordActivity), errors.ts, pagination.ts, ssrf.ts
│  │     ├─ tasks/            runner.ts + handlers registry
│  │     └─ modules/<name>/   routes.ts · service.ts · repo.ts · *.test.ts   (one folder per module)
│  ├─ web/                    Vite + React SPA  (src/pages/<page>/, src/lib/api.ts)
│  ├─ cli/                    `jm` (commander): src/commands/<module>.ts, src/mcp-stdio.ts
│  └─ extension/              MV3 Clipper (M3)
├─ packages/
│  ├─ core/                   zod contracts (entities, DTOs, enums), pure functions
│  │                          (fingerprint, jsonld, url-normalize, status-rules) + fixtures
│  ├─ db/                     drizzle schema.ts, migrations/, createDb(), test helpers
│  ├─ client/                 typed REST client: src/<module>.ts + createClient({baseUrl, token, fetch})
│  └─ mcp/                    createMcpServer(client): src/tools/<module>.ts
└─ docs/                      spec/, plans/, harness/, decisions.md, handoff/
```

**Module list (pre-stubbed in M0 so no card edits a shared registry):** `system`, `auth`,
`companies`, `jobs`, `capture`, `contacts`, `links`, `outreach`, `followups`, `activities`, `tags`,
`notes`, `search`, `imports`, `exports`, `settings`, `summary`.

**Boundary rules**
- A module imports another module only through that module's `service.ts` exports. It never imports
  another module's `repo.ts`.
- Every mutation calls `recordActivity()` inside the same DB transaction.
- `packages/core` has no I/O and no dependencies beyond zod, so it's pure and fully unit-tested.
- Web, CLI and MCP never touch the DB. They go through `packages/client`.

## 6. Request flow (example: move a job to "applied")

```
jm jobs move 01J… applied --json
  → client.jobs.move(id, 'applied')  POST /api/v1/applications/01J…/move {status:'applied'}
  → authMiddleware (bearer, scope write) → zValidator(MoveApplicationInput)
  → jobs.service.moveApplication(db, id, 'applied', actor='cli')
       tx: update applications set status, applied_at=coalesce(applied_at, now)
           recordActivity(tx, {entity:'application', type:'status_changed', from:'saved', to:'applied', actor:'cli'})
  → 200 {data: Application}
```

## 7. Deployment modes

| Mode | For | How |
|---|---|---|
| **Local (default)** | Richard, any self-hoster | `pnpm build && jm init && jm serve`. Data in `~/.jm` |
| Docker | Linux servers / NAS | One image, volume at `/data`, `--host 0.0.0.0` behind a TLS proxy (Caddy) |
| Cloudflare (later, **to verify**) | Hosted / demo | Hono on Workers + D1 (SQLite dialect). Must first verify FTS5 on D1, Node compatibility for PDF parsing, and long LLM calls |
| Docs/landing (M7) | Public | Astro Starlight, static, Cloudflare Pages (static-first [article 019]) |

## 8. Defend it: the six questions [article 020]

1. **Why this database and schema?** One owner means one SQLite file. That gives zero ops, a one-file
   backup (`VACUUM INTO`), FTS5 search built in, and the same dialect as Cloudflare D1. The schema
   splits **identity** (`jobs`, `companies`, `contacts`) from **tracking** (`applications`, statuses)
   and **history** (`activities`), so re-imports never wipe state (findleads DATA-01 lesson).
2. **Normalised by design?** Yes, to 3NF for entities. The deliberate denormalisations are:
   `activities.from_value/to_value` (history must survive entity edits) and the FTS tables (a derived
   index kept by triggers). Tags use a polymorphic `taggings` table, a documented trade (no FK) for
   one simple tag system across entity types.
3. **What breaks first at 100×?** 700 contacts becomes 70k, and 300 jobs becomes 30k plus
   discovery. Unindexed filters and offset pagination break first, so M1 ships keyset pagination +
   indexes on every filter column. Next is the single writer: SQLite WAL gives one writer. Imports
   batch in transactions, and the task runner serialises writes. Multi-user at 100× means one DB
   file per owner (ADR-016), not a shared DB.
4. **Edge cases handled / skipped:**
   - Handled: duplicate captures, re-imports (idempotent via `external_refs`), LinkedIn URLs refused
     server-side, SSRF, and clock/timezone (store UTC ms, render in owner TZ).
   - Skipped on purpose: multi-user sharing, real-time sync, and offline web UI.
5. **What did decisions cost?** SQLite: no concurrent multi-writer and no Postgres extensions, but
   ops cost is $0. SPA instead of SSR: no SEO, which isn't needed for an app behind a login. Hono
   instead of Next: less reuse (ports instead of lifts), but a much smaller surface for simpler
   models.
6. **What would change on a rebuild?** Revisit this after M1 dogfooding. Record the answers in
   `docs/decisions.md`.

## 9. Optimisation and security

Budgets and techniques: [`optimization.md`](optimization.md). Security requirements (SEC-01 to
SEC-18) derived from the 12-check audit [article 006] plus agent-specific risks:
[`security.md`](security.md).
