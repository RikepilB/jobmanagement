# Comprehension-check log

> Evidence that simpler models can execute the cards. Method: harness README §5. Each checker is a
> `haiku` subagent, **read-only**, given one card plus only the files it lists (it may also read
> M0 cards). It reports ambiguities, missing info, contradictions, unknown terms and out-of-scope
> files, each with a severity.

## Run 1: 2026-09-30 (planning session, before any code)

| Card | BLOCKER | MAJOR | MINOR | Status after fixes |
|---|---|---|---|---|
| M1-05 Jobs write side | 3 | 3 | 5 | all BLOCKER/MAJOR fixed. MINOR items fixed where cheap (sort order, tag loop, detail assembly, ATS enum) |
| M1-09 Capture + safe-fetch | 1 (really a MAJOR contradiction) | 3 | 2 | all fixed |
| M1-13 Adapters A | 2 | 2 | 2 | all fixed |

### Findings → fixes (what changed in the docs)

**M1-05**
- *`source` isn't in `JobCreateInput` but the card used it* → `createJob` gains
  `opts: {source, sightingExternalId, sightingRaw}`. Routes pass `manual`, capture `url`/`extension`,
  imports `import_*`. REST stays unchanged (M1 signature table).
- *No import path for `normalizeUrl` etc.* → the card now has "Imports you'll use" (`@jm/core`,
  `@jm/core/parsers`, the server `shared/*`, other modules' services).
- *Read-first missed data-model §4/§5* → added §1, §2, §4, §5, plus M0-06, M1-00, M1-02, M1-03.
- *"Add a sighting if source differs" was ambiguous* → the exact rule (same `source` + same
  `source_url`, null-safe; the external-id unique index for imports).
- *Activity multiplicity unclear* → the exact list (job `created`, application `status_changed`,
  tags and company record their own).
- *Untrack activity type unclear* → `status_changed` from status to null with a fixed summary.
- Also: `possibleDuplicates` (≤10, `updated_at desc`), a `getJobDetail` assembly recipe, and a new
  `AtsProvider` enum in data-model §1.

**M1-09**
- *Tags not in `CaptureJobInput`* → `track`, `status` and `tags` added to `CaptureJobInput` (api-rest §4)
  and `jobs_capture_url` (mcp.md), passed through to `createJob`.
- *The title-suffix rule was vague* → an exact **title rule** algorithm plus two test cases.
- *`maxBytes` behaviour undefined* → Content-Length pre-check + a streaming cutoff, both
  `content_too_large`. A full list of `reason` codes.
- *Resolver signature missing* → the exact type plus the default (`dns/promises.lookup` with `all:true`),
  and a `fetchImpl` injection point.
- *When `company_unknown` fires* → an exact trigger, plus `"Unknown company"` for text-only captures.

**M1-13**
- *The capture CLI file vs the `jm jobs capture` subcommand* → Owns now lists
  `commands/{jobs,companies}.ts` and `tools/{jobs,capture}.ts`. The unused stubs stay empty, with
  the reason stated.
- *Id-prefix resolution unspecified* → moved **server-side**: `shared/resolve-id.ts`
  (`resolveEntityId`, M0-07 contract + tests), api-rest §0 and cli.md §0 updated, and a CONVENTIONS
  rule added (every `:id` goes through it). Ambiguous → 409 → CLI exit 4.
- *Client/MCP/CLI helpers not referenced* → Read-first for M1-13, M1-14 and M1-15 now cite M0-09,
  M0-10 and M0-11 by function name.
- *No `companies_*` MCP tools* → an explicit decision (reachable via `search` and details), and
  `jm companies …` added to cli.md §2.
- *Summary text format vague* → an example table in mcp.md §1.

## Run 2: 2026-09-30 (fresh cold Haiku checker, M1-05 only, after the Run 1 fixes)

| Card | BLOCKER | MAJOR | MINOR |
|---|---|---|---|
| M1-05 | 1 (false positive) | 5 | 3 |

- **None of the Run 1 findings recurred.** The Run 1 fixes held.
- BLOCKER "contracts don't exist yet" is a false positive: they come from M0-06, and M1's
  precondition is the M0 exit. The card now says "created by M0-06, merged before M1 starts".
- New, finer-grained MAJORs, all fixed:
  - `JobPatchInput` fields weren't enumerated → an exact patchable/non-patchable list in api-rest §3
    (`.strict()` → 400).
  - ATS pairing rule → both fields or neither (400). `detectAts` applies only when both are absent.
  - Sighting precedence → two mutually exclusive branches (a)/(b).
  - `recordActivity` call shape → an example with `ctx.actor`/`ctx.now()`.
  - Fingerprint order → after ATS resolution.
- MINORs fixed: the 409 error code is named, and `resolveCompany` never updates an existing company.
- **Not re-verified by a Run 3.** The Run 2 fixes are addressed but unconfirmed.

### Lesson (added to the harness)
Each run surfaces finer-grained gaps, so this is a loop, not a one-off. Run 1's most common failures
were an *implicit input* (`source`) and an *implicit helper location*, which led to the template
fields "Imports you'll use" and "Inputs that aren't obvious". Run 2's were *unenumerated field
lists* and *precedence between two rules*. So spec field lists must be explicit, and rules with two
branches must say which one wins. **The orchestrator runs this check on every card in a wave before
dispatch, and dispatches only when BLOCKER = 0 and MAJOR = 0** (false positives noted in the log).

## Next runs
Before each wave is dispatched, record: date, cards, counts, fixes.
