# Contributing

Thanks for your interest. This project is pre-alpha and is built mostly by AI agents working from
written task cards, so the process is deliberately strict.

## Before you start

1. Read [`AGENTS.md`](AGENTS.md), then [`docs/spec/SPEC.md`](docs/spec/SPEC.md).
2. Pick a ticket under `docs/plans/<milestone>/tickets/` whose `Status:` is `open` and whose
   `Blocked by:` tickets are all `merged` (rules: [`docs/agents/issue-tracker.md`](docs/agents/issue-tracker.md)).
   Open an issue first if you want to work on something without a ticket.
3. Read the contracts the ticket points to (`docs/spec/*.md`). **If a contract looks wrong, stop and
   open an issue.** Don't work around it in code.

## Rules

- Touch only the files your ticket owns. Don't add dependencies; the allowlist is in
  [`docs/harness/CONVENTIONS.md`](docs/harness/CONVENTIONS.md).
- One behaviour path: the CLI and MCP are thin adapters over the REST client. No business logic in
  adapters.
- Tests use synthetic data only ("Contact A", "Company X"). **Never put real people's data in tests,
  fixtures, docs or issues.**
- Never commit secrets, `.env*` files, databases or captures. A gitleaks pre-commit hook is
  recommended.
- `pnpm verify` must pass before you open a PR (available from M0-03).

## Pull requests

- Keep PRs to one ticket. Reference it in the title, e.g. `M1-05: jobs write side`.
- Describe what changed and paste the acceptance evidence from the ticket.
- Agent-written PRs start as drafts.

## Code of conduct

Participation is governed by [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md).
