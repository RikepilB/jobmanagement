# jobmanagement

A self-hosted, agent-first job-search workspace for one person: a jobs kanban, a networking CRM,
an outreach log and follow-ups, driven by your AI agents through a `jm` CLI and MCP, with a simple
web UI on top. MIT-licensed. Built as an open replacement for PathPeer.

## Status

**Pre-alpha. Planning is complete; the app is not built yet.** There is nothing to install or run
today. What exists is the full specification, the milestone plans and the multi-agent build harness.

| Milestone | What it delivers | Status |
|---|---|---|
| M0 | Repo, toolchain, schema, server/client/MCP/CLI/web skeletons, gates | ready to build |
| M1 | MVP: jobs, contacts, companies, outreach log, follow-ups, search, imports and exports over REST + CLI + MCP + web | planned |
| M2–M7 | AI layer, Clipper extension, networking engine, discovery, suite integration, release | titles only |

See [`docs/plans/ROADMAP.md`](docs/plans/ROADMAP.md).

## Quick start (once M0 lands)

```bash
pnpm install
pnpm build
jm init     # creates the data dir (default ~/.jm) and the owner account
jm serve    # http://127.0.0.1:4747
```

Then connect an agent over MCP, for example `claude mcp add jm -- jm mcp`.

## For agents

Start with [`AGENTS.md`](AGENTS.md). It covers setting the app up for a user, building this repo
card by card, and operating the product through MCP or `jm --json`.

## Privacy

These are design requirements from the spec; they become testable as the milestones land.

- Single owner, local first. The server binds to `127.0.0.1` and keeps everything in one SQLite
  file in your data dir.
- **Nothing leaves your machine in M1.** From M2, the LLM provider you configure receives only the
  job and profile text you choose to analyse.
- The server refuses to fetch LinkedIn URLs, and there are no delete tools over MCP.
- **Data at rest, honestly:** tokens and sessions are hashed, and from M2 provider API keys are
  encrypted. Contact names and emails are stored **in plaintext** in v1. Disk encryption on the
  host is your responsibility and is not counted as a control here
  ([`docs/spec/security.md`](docs/spec/security.md), SEC-08).

## Documentation

| Path | Contents |
|---|---|
| [`docs/spec/`](docs/spec/) | Product spec and contracts: data model, REST, MCP, CLI, security, optimization, architecture |
| [`docs/plans/`](docs/plans/) | Roadmap and one ticket file per task |
| [`docs/harness/`](docs/harness/) | How agents build this repo: roles, conventions, gates, prompts |
| [`docs/decisions.md`](docs/decisions.md) | Architecture decision records |
| [`docs/landscape.md`](docs/landscape.md) | Prior art and the licence gate |
| [`docs/pathpeer-teardown.md`](docs/pathpeer-teardown.md) | Feature map of the product this replaces |

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). Security reports: [`SECURITY.md`](SECURITY.md).

## Licence

[MIT](LICENSE) © 2026 Richard Pillaca. Third-party material and its licences:
[`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md).
