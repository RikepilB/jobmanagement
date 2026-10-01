# Issue tracker: local markdown (tickets under `docs/plans/`)

Read by the Matt Pocock engineering skills (`to-tickets`, `triage`, `to-spec`, `wayfinder`) and by
the sandcastle wave runner (`.sandcastle/run-wave.ts`, M0-16). It adapts the upstream
local-markdown convention to this repo's milestone layout. **Tickets are the task cards.**

## Layout
- One milestone per directory: `docs/plans/<M>/` (e.g. `docs/plans/M1/`).
- Milestone index: `docs/plans/<M>-*.md` (goal, waves table, shared signatures, ticket table,
  exit criteria). It links to the tickets and never duplicates their contract.
- **One file per ticket:** `docs/plans/<M>/tickets/<NN>-<slug>.md`, numbered from `00`/`01`,
  never a combined tickets file. Its body follows `docs/harness/task-card-template.md`.
- The spec for a milestone is `docs/spec/*` (shared, not per milestone).

## Header lines (machine-read; the first lines after the `# <ID> · <title>` heading)
```
Status: open | claimed | in-review | merged | needs-human | blocked
Blocked by: none | M1-02, M1-03
Wave: <n>
Owner: agent | orchestrator | human
Model: sonnet | haiku | opus
```
- **`Blocked by:` is authoritative** for dependencies. The `**Depends on**` prose in a ticket body is
  informational. If they disagree, the header wins and the prose gets fixed.
- **Frontier** = `Status: open`, `Owner: agent`, and every ticket in `Blocked by:` is `merged`.
  Lowest number first.
- **Claim:** the orchestrator sets `Status: claimed` when it dispatches the ticket (Mode A) or starts the
  wave that includes it (Mode B). **The orchestrator is the only writer of ticket status.** The sandcastle
  runner never edits `docs/plans/**`.
- **In review:** set by the orchestrator when the implementer reports DONE, or when a Mode B report
  shows gate + verifier PASS.
- **Merged:** set by the orchestrator after gate G2, with the merge sha appended under `## Comments`.
- **needs-human / blocked:** set with a one-line reason under `## Comments` (BLOCKED-NEED text
  from the implementer).

## When a skill says "publish to the issue tracker"
Create the next-numbered file under `docs/plans/<M>/tickets/` using the task-card template and the
header lines above. `to-tickets` output = one file per tracer-bullet ticket, with `Blocked by:`
from its blocking edges. Also add a row to the milestone index table.

## When a skill says "fetch the relevant ticket"
Read `docs/plans/<M>/tickets/<NN>-*.md`. People pass the ID (`M1-05`) or the path.

## Comments and history
Append to the bottom under `## Comments`: `- YYYY-MM-DD: <who>: <what>` (status changes,
blocked needs, verifier verdicts, merge shas).
