# CLI `jm`: single source of truth

> The shell interface for Richard and his agents. Every command maps to REST routes
> ([`api-rest.md`](api-rest.md)) through `packages/client`. Built with **commander 15**, bundled by
> tsup into `apps/cli/dist/jm.js`, exposed as the `jm` bin (`pnpm --filter @jm/cli link --global` in
> dev).

## 0. Global behaviour

| Topic | Rule |
|---|---|
| Config | `~/.jm/config.json` `{url, token}`. Env overrides: `JM_URL`, `JM_TOKEN`. File written with owner-only permissions where the OS supports it. The token is never printed after creation |
| Output | Human table by default. **`--json`** prints exactly the REST response body (`{data, nextCursor?}`) on stdout, so agents use `--json`. Errors go to stderr |
| Exit codes | `0` ok · `1` usage/validation · `2` not found · `3` auth (401/403) · `4` conflict · `5` server/network · `6` rate limited |
| Ids | Full ULIDs or any unique prefix ≥ 6 chars, passed straight to the API, which resolves them (api-rest §0). Not found → exit 2. Ambiguous (409) → exit 4, and the candidates are printed |
| Actor | Sends `X-JM-Actor: cli`. `jm mcp` sends `mcp` |
| Confirmation | Destructive commands (`delete`, `tokens revoke`) require `--yes`. Without it they print what would happen and exit 1 |
| Help | Every command has `--help` with one example. `jm help agents` prints the "For agents" quick reference |

## 1. Local server commands (run on the machine that holds the data)

| Command | Does |
|---|---|
| `jm init [--data-dir <path>] [--name <display>]` | Creates the data dir + DB, runs migrations, prompts for the owner password (twice, no echo), creates an `admin` token named `cli`, and writes `~/.jm/config.json`. Refuses if an owner already exists |
| `jm serve [--port 4747] [--host 127.0.0.1] [--data-dir]` | Starts the server (API + /mcp + web + task runner). Warns loudly if host ≠ 127.0.0.1 without `JM_TLS_OK=1` |
| `jm migrate` | Applies pending migrations and exits |
| `jm backup [--out <file>]` | `VACUUM INTO` a timestamped copy under `<dataDir>/backups/` |
| `jm reset-password` | Local only. Sets a new owner password and revokes all sessions |

## 2. Client commands (talk to the server over HTTP)

| Command | REST |
|---|---|
| `jm login --url <url> --token <token>` | `GET /auth/me` to validate, then writes config |
| `jm status` | `GET /system/status` + `GET /summary` |
| `jm tokens list` · `jm tokens create <name> --scopes read,write [--days 365]` · `jm tokens revoke <id> --yes` | `/tokens` |
| `jm jobs list [--status saved,applied] [--q text] [--tag t] [--company id] [--rating-min n] [--limit n] [--all]` | `GET /jobs` (`--all` follows cursors) |
| `jm jobs board` | `GET /jobs/board` |
| `jm jobs get <id>` | `GET /jobs/:id` |
| `jm jobs add --title T --company C [--url U] [--location L] [--type remote] [--status saved] [--tag t]…` | `POST /jobs` |
| `jm jobs capture <url> [--no-track]` | `POST /capture/job` |
| `jm jobs move <id> <status>` | `POST /jobs/:id/move` |
| `jm jobs rate <id> <0-5>` · `jm jobs flag <id> [--off]` | `PATCH /jobs/:id` |
| `jm jobs delete <id> --yes` | `DELETE /jobs/:id` |
| `jm companies list [--q text] [--tier a] [--limit n] [--all]` · `jm companies get <id>` · `jm companies add --name N [--domain D] [--tier a\|b\|c]` · `jm companies set <id> [--tier] [--website]` | `/companies` |
| `jm contacts list [--status …] [--q] [--follow-up-due today\|YYYY-MM-DD] [--connection invite_pending] [--limit] [--all]` | `GET /contacts` |
| `jm contacts board` | `GET /contacts/board` |
| `jm contacts get <id>` | `GET /contacts/:id` |
| `jm contacts add --name N [--linkedin URL] [--company C] [--title T] [--headline H] [--status saved]` | `POST /contacts` (upsert) |
| `jm contacts move <id> <status>` | `POST /contacts/:id/move` |
| `jm contacts set <id> [--connection connected] [--thread replied] [--follow-up YYYY-MM-DD] [--rating n] [--compat strong]` | `PATCH /contacts/:id` |
| `jm contacts snooze <id> <days>` | `POST /contacts/:id/snooze` |
| `jm contacts email <id> <email> [--type work] [--status verified]` | `POST /contacts/:id/emails` |
| `jm link <contactId> <jobId> --role recruiter` · `jm unlink …` | `POST/DELETE /links` |
| `jm outreach log <contactId> --format f1 --channel linkedin_invite (--body "…" \| --body-file f) [--job id] [--hook "…"]` | `POST /contacts/:id/outreach` |
| `jm outreach mark <outreachId> replied\|withdrawn` | `PATCH /outreach/:id` |
| `jm outreach today` | `GET /outreach/today` |
| `jm followups [--date YYYY-MM-DD]` | `GET /followups` |
| `jm remind <entityType> <id> --due YYYY-MM-DD [--note …]` | `POST /reminders` |
| `jm activity [--entity job:<id>] [--limit 20]` | `GET /activities` |
| `jm note <entityType:id \| global> "text"` | `POST /notes` |
| `jm tag <entityType:id> <tag>` · `jm untag …` | `POST /tags/apply` / `remove` |
| `jm search "<q>" [--types job,contact]` | `GET /search` |
| `jm import resumescorer <path-to-jobs.json>` | reads the file locally, `POST /imports/resumescorer`, then polls `GET /tasks/:id` until done |
| `jm import pathpeer <export-dir> [--dry-run]` | reads the files locally, builds the bundle, `POST /imports/pathpeer`, polls |
| `jm export json [--out file]` · `jm export csv jobs\|contacts [--out file]` | `/exports/*` |
| `jm settings get` · `jm settings set followUpDays=5 dailyCaps.invites=15` | `/settings` |
| `jm mcp` | Runs the MCP stdio server ([`mcp.md`](mcp.md)) |

## 3. Examples an agent will run
```bash
jm followups --json                      # what to do today
jm contacts list --status saved --limit 5 --json
jm outreach log 01J9Z3 --format f1 --channel linkedin_invite --body-file note.txt --hook "posted about X"
jm jobs capture https://boards.greenhouse.io/acme/jobs/123 --json
jm jobs move 01J9Y8 applied
```
