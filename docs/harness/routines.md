# Routines: scheduled maintenance (enable after v1 = end of M1)

> The idea comes from Boris Cherny's four daily passes, via Nelson Lee [article 012]: work nobody
> would ticket for themselves, run on a schedule, **always ending in a draft PR, never a merge**.
> Create them with `/schedule` in Claude Code (runs on Claude Code's web infrastructure, so the repo
> needs a GitHub remote). The prompts below are rewritten for this repo. Each is goal + constraint
> + output, nothing more.

| Routine | Cadence | Prompt |
|---|---|---|
| **Maintenance** | nightly | "Maintenance pass on this repo. Report dependencies with published advisories (don't upgrade), config/env vars the code no longer reads or reads but `.env.example` lacks, TODO/FIXME older than 90 days, and docs in `docs/spec` that name routes, tools, tables or commands that no longer exist. Change nothing but docs and `.env.example`. One draft PR per category. Say 'nothing' and skip empty categories." |
| **Dead code** | weekly | "Find exports, routes, MCP tools, CLI commands, env vars and components nothing references (count test-only references as unreferenced). Check dynamic imports, string references and the module registries before calling anything dead. Rank by confidence. One draft PR deleting only high-confidence items. List the rest in the PR body." |
| **Risk-first tests** | weekly | "Find untested paths where a failure costs something real here: auth/tokens, status side-effect rules, imports (data loss), CRM-preserving upserts, safe-fetch, exports (PII). Ignore coverage %. Write tests for the top 3 gaps asserting real outputs, with no snapshots and no was-called-only tests. One draft PR." |
| **Abstraction police** | weekly | "Find logic written more than twice in different shapes (date formatting, status labels, client error mapping, pagination) and abstractions with a single caller. Show the variants side by side with the single replacement. One draft PR for the top 2 only. Checklist the rest in the PR body. Don't touch `docs/spec`." |

Plan caps (from the source article, verify against current plan limits): Pro 5 routine runs/day,
Max 15. Four routines fit in Pro if they don't all run daily.

Every routine PR is reviewed by the orchestrator exactly like a card. Deviations it reveals become
CONVENTIONS rules (Drift Loop).
