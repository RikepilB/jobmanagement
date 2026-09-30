# Skills audit for jobmanagement

Which of Richard's installed skills this project can **use**, which need **improving**, and which
to **create**. Each skill gets a test prompt and pass criteria, so a test can be replayed.

**Status legend:** `TESTED` (run in a session, evidence noted) · `NOT RUN` (the test is defined
but hasn't been executed). Only one skill was actually run on 2026-09-29.

**Out of scope for this project:** the Space Apps harness (11 skills: hackathon-specific), the
box legal workflows, figma-* (unless a Figma pass is chosen for UI), motivation-letter,
linkedin-post, worksmith (Jira/Confluence), the productivity plugin, and the pdf-viewer skills.

---

## 1. Use as-is

| Skill | Where it plugs in | Phase | Test prompt (synthetic data only) | Pass criteria | Status |
|---|---|---|---|---|---|
| `richard-outreach` | Outreach drafts. F1–F6 **is** the message-type selector PathPeer calls "Email type". The hook gate becomes a product rule | P2, P4 | "F1 note + F3 email to Contact A, Talent Acquisition at Company X; commonality: both in tech hiring, both international; goal: referral path" | With **no verifiable hook** it must refuse to draft and output WARNING. It must not put a referral ask in a first message | **TESTED 2026-09-29, PASS.** The skill was loaded and executed on this prompt. Output, verbatim: `WARNING — No strong connection found. Add a specific detail before writing.` No draft was produced. The referral goal was declined per Golden Rule 2 ("Never a referral in a first message"). **Not yet tested:** draft quality with a real hook (test 1b) |
| `richard-outreach` (1b) | same | P2 | Same contact + hook: "posted last week about hiring juniors who ship side projects" | F1 ≤300 chars, no links, one ask, zero banned words, notes block present | NOT RUN |
| `richard-cover-letter` | "Generate cover letter" on a job (saved to `documents`) | P2 | Synthetic JD (full-stack, React/Node, Toronto) | 4-part canonical structure, only verified proof points, [VERIFY] for gaps | NOT RUN |
| `score-resume` + `resume-optimizer` agent | Resume-quality ("ATS") score on `resume_versions` (ADR-006), kept apart from job fit | P6 | Score the current resume PDF, then run the optimizer on the report | Report + ranked honest edits, nothing invented | NOT RUN: it uses paid OpenRouter keys through the external `dev/hiring-agent` venv. Run it deliberately |
| `voidscape` | Coffee chat or interview recording → transcript evidence attached to a contact/job | P6 | `voidscape inspect` on a local sample recording | inspect → preview → read gates shown, `[MM:SS]` citations, no cloud call without approval | NOT RUN |
| `engineering:system-design`, `engineering:architecture` | ADR hygiene in `docs/decisions.md` | P0 | "Evaluate ADR-003 Prisma vs Drizzle for this blueprint" | Trade-off table, a recommendation, and consequences | NOT RUN |
| `grill-with-docs` | Sharpen P1 scope before building (writes `docs/decisions.md`) | P0 | "Grill me on the P1 scope in docs/blueprint.md" | One question at a time, each with a recommended answer. Decisions get appended | NOT RUN |
| `disposable-prototype` | Spikes: PathPeer export hook, extension JSON-LD capture, kanban feel | P0, P3 | "Prototype the JSON-LD JobPosting capture on 3 saved ATS HTML fixtures" | Answer captured, code discarded | NOT RUN |
| `design-intent` → `frontend-design` → `anti-slop-review` | UI direction, build, review | P1 | "Design intent for a dense, calm job-search CRM (tables + kanban)" | Brief exists before UI code. Review runs on rendered screenshots | NOT RUN |
| `production-readiness`, `security-review`, `code-review` | Pre-release gates | P7 | Standard | Advisory list. No blocking claims without evidence | NOT RUN |
| `handoff-context`, `catch-up` | Session continuity | all | — | — | in use |
| `find-note` | Pull Career OS context (playbooks, audience matrix) on demand | P2, P4 | "Where is the connection note audience matrix?" | Returns the `03_Career_OS/Networking/...` path | NOT RUN |
| `apollo:enrich-lead` | Optional **email provider** in the resolution chain | P4 | "Enrich: Contact A, Company X" (synthetic) | Returns email + confidence, **shows credit cost** before spend | NOT RUN: **Apollo MCP needs authorization first** (claude.ai connector settings) |

## 2. Improve

| Skill / asset | Problem found | Change |
|---|---|---|
| `richard-outreach` | The header says pipeline logic "lives in the Career OS harness… PathPeer board moves". That breaks when PathPeer goes away. It only emits prose | (a) Repoint to jobmanagement statuses. (b) Add a **machine-readable output mode** (`{format, channel, body, hook_used, char_count, notes}`) so the app/MCP can store drafts. (c) Add a status→format table matching blueprint §4.8. (d) Make "hook" a required input field |
| Career OS harness (`03_Career_OS/CLAUDE.md`, `source-contacts_shortcut_v3.md`, `process-contact_shortcut_v4/v5/v6.md`) | Built on PathPeer URLs, "Career Clipper", and reading the board through Chrome. RULE ZERO exists only because PathPeer hides message state | After P3/P6: replace PathPeer URLs with MCP tools. Replace Career Clipper with the Clipper extension. Drop RULE ZERO once `connection_state`/`thread_state` exist. Keep the caps and audience matrix |
| ResumeScorer `scripts/job_finder.py` | Python, writes a JSON ledger, no dedupe with a tracker | Port to `packages/sources` adapters (TS). Keep the Python script as the reference implementation until parity |
| `score-resume` | Prose report only | Add a JSON summary block (`score, dimensions, top_fixes`) so the P6 adapter can store it on `resume_versions` |
| `linkedin-triage` (Voidscape project skill) | Overlaps with contact capture | Keep it inside Voidscape's boundary. Don't merge it with the Clipper. Revisit when P3 ships |

## 3. Create

New personal skills go into `skills-lab` (CATALOG row + sync script entry + `sync pull`), per
the global CLAUDE.md rule.

| New skill | Purpose | Inputs → outputs | Phase |
|---|---|---|---|
| `job-intake` | Paste a URL or JD text and get a normalized job (Zod schema from `packages/schemas`) plus a quick fit read. Useful before the extension exists, and as the MCP-side helper after | URL/text → job JSON + fit summary | P1–P2 |
| `contact-research` | Turn user-supplied profile text into indicators, commonalities and **verifiable hooks with sources**. Feeds the `richard-outreach` hook gate | profile text → contact analysis JSON | P2 |
| `pipeline-sweep` | Weekly review: stale applications, follow-ups due, pending invites >7 days, caps usage, stuck contacts → action list. Replaces process-contact "sweep" mode, backed by MCP | MCP data → ranked action list | P4–P6 |
| `interview-prep` | Job + company + linked contacts + Voidscape transcripts → prep brief with likely questions and stories from proof points | entity bundle → brief | P6 |
| (not a skill) `jobmanagement-mcp` | The MCP server in blueprint §9 | — | P6 |

## 4. How to run a test (repeatable)

1. Use synthetic people and companies only ("Contact A", "Company X").
2. Run the skill with the test prompt above in a fresh session.
3. Compare against the pass criteria and record `TESTED <date>: pass/fail + one-line evidence`
   in the Status column.
4. A failure goes into the skill's own repo (skills-lab) as an issue or fix, not into this project.
