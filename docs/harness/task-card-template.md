# Task card template

> One card = one agent session = one branch. Target **≤ ~400 changed lines** (tests included). If
> it's bigger, split it. A card never contains a step-by-step procedure: precision comes from the
> contract and the acceptance tests [012].

```markdown
### <ID> · <short title>
**Wave** <n> · **Model** sonnet|haiku|opus · **Depends on** <IDs> · **Parallel-safe with** <IDs or "none (serialised)">

**Goal.** <1–2 sentences describing the outcome, not the steps.>

**Read first.** <exact files/sections: e.g. docs/spec/api-rest.md §3, docs/spec/data-model.md §2 jobs,
apps/server/src/modules/companies/service.ts (for the company resolve function)>

**Imports you'll use.** <exact functions/types and where they come from, e.g. `normalizeUrl` from
`@jm/core`, `resolveCompany` from `modules/companies/service`. Required: implicit helpers caused
most BLOCKERs in comprehension-check run 1>

**Inputs that aren't obvious.** <any value the code needs that isn't in the request body (source,
actor, settings) and where it comes from>

**Owns** (may create/edit only these):
- <path or glob>

**Must not touch.** package.json files, lockfile, packages/db/**, packages/core/src/contracts/**,
any registry file, any other module's folder. <plus card-specific>

**Contract.** <what must exist when done, by name: routes, exported functions with signatures,
MCP tools, CLI commands, UI pages; cite REQ-/SEC- ids>

**Acceptance.**
- `pnpm --filter <pkg> test -- <file>` passes, with these behaviours covered:
  - <behaviour 1 (observable output)>
  - <behaviour 2>
- `pnpm verify` passes.

**Don'ts.**
- <anti-pattern specific to this card>
- (always) no new dependencies, no edits outside Owns, no snapshot tests, no invented fields.

**Decision table** (only if the inputs vary in kind):
| Input shape | Rule |
|---|---|

**Report.** STATUS: DONE | BLOCKED · files changed · acceptance results (PASS/FAIL + output tail) ·
Noticed (bugs outside scope) · Deviations (should be "none")
```
