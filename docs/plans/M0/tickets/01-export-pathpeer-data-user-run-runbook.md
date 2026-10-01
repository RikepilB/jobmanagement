# M0-01 · Export PathPeer data (user-run runbook)

Status: open
Blocked by: none
Wave: any
Owner: human
Model: -

**Wave** any · **Owner** Richard with Claude in Chrome (Opus main session) · **Depends on** none · **Blocks** M1-11 only

**Goal.** A complete, verified dump of Richard's own PathPeer data, stored outside OneDrive and
outside the repo, plus a PII-free *shape* file the importer can be built against.

**Read first.** `docs/blueprint.md` §0 · `docs/pathpeer-teardown.md` §2–§7.

**Produces.**
- `C:\Users\a2021\dev\jobmanagement-private\pathpeer-export-<YYYY-MM-DD>\`
  - `captures.jsonl`: one line per tRPC response `{procedure, input, capturedAt, body}` (response
    bodies only, **no request headers**), **or** `pathpeer.har` (sanitised export) as fallback
  - `resume.pdf` (downloaded through the UI)
  - `counts.md`: per-status counts read from the UI **at export time**
- In the repo (after M0-02): `docs/spec/pathpeer-export-shape.md`, listing for each captured
  procedure the JSON **key paths and value types only** (e.g. `result.data.json[].trackerStatus: string`).
  **No values.**

**Method.** Blueprint §0. First spike: install the fetch-capture hook, open one job drawer, and
confirm at least one body was captured. If nothing is captured, use the HAR fallback. Navigate with
sidebar clicks only (a full page load drops the hook).

**Acceptance.**
- Counts in the captures equal `counts.md` for every job status and contact status.
- `Select-String -Path <dir>\* -Pattern 'authorization|cookie' -SimpleMatch` → 0 matches.
- `pathpeer-export-shape.md` contains no email addresses, names or URLs (grep `@`, `linkedin.com/in`).

**Don'ts.** No token hunting. Never read storage or cookies. Nothing goes into OneDrive, and no
values go into the repo.

## Comments
