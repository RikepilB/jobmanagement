# The build harness: how agents build jobmanagement

> A deliberately **lean** harness: one orchestrator, simpler-model implementers working in parallel
> in isolated worktrees, a read-only verifier, and hard gates. **It replaces GSD for this repo**, so
> there aren't two planning systems competing. Files: this README, [`CONVENTIONS.md`](CONVENTIONS.md)
> (the drift-loop target), [`task-card-template.md`](task-card-template.md), [`gates.md`](gates.md),
> [`prompts/`](prompts) (orchestrator, implementer, verifier, security-audit) and, after v1,
> [`routines.md`](routines.md).

## 1. Where each rule comes from
Sources: Nelson Lee's articles (vault `06_Media/Articles/nelsonxlee`, cited as [NNN]) and Richard's
skills.

| Lesson | Source | Harness rule |
|---|---|---|
| State the goal and the constraint, then stop. Long procedures make current models worse | [012] Boris Cherny | Task cards contain **goal, constraints, contract, owned files, acceptance, don'ts**, with **no step-by-step procedure**. Precision comes from contracts and tests, not from instructions |
| Draft PRs always. Nothing merges unattended | [012] | Implementers commit on a card branch (and open a **draft** PR if a remote exists). Only the orchestrator merges, and only after the verifier reports PASS |
| Tests assert real outputs. No snapshots, no "was it called" tests, coverage % is not a target | [012] | CONVENTIONS §Testing. The verifier rejects tests that break these rules |
| Every deviation is a missing rule (the Drift Loop) | [008] | When review finds a deviation, the orchestrator **appends a rule to CONVENTIONS.md**, not just a fix. The file compounds |
| Update section by section. Never regenerate everything | [008], [013] | One card = one vertical slice with owned files. No card rewrites another module. The orchestrator reviews each wave before starting the next |
| Write the source-of-truth doc before opening tools | [013] ICP doc | `docs/spec/*` exists before any code. Each contract has one file. Cards cite spec sections |
| Study existing products for shape | [013] Jakob's law | `docs/landscape.md` + `docs/pathpeer-teardown.md`. The UI copies familiar board/table/drawer shapes |
| Three-state evidence: PASS / FAIL / NOT FOUND with file:line. Mark what was inferred. Audits don't write code | [006], [007], [009], [017] | The verifier and security-audit prompts use exactly this contract |
| Stop and ask for the missing asset instead of inventing it | [017] | An implementer who lacks a contract detail, fixture, dependency or file outside its ownership reports **BLOCKED** with the exact need |
| Docs have two audiences: add a "For agents" section. Skills are the repo's public interface. Features stay disabled until their preconditions exist. State privacy plainly | [016] | `AGENTS.md` + README "For agents". M6 ships `.agents/skills/`. LLM features are disabled until a key exists (M2). The README says what leaves the machine |
| Don't juggle six sessions: one thing at a time | [005] | **One orchestrator, one milestone at a time.** Parallelism only inside a wave, over disjoint files, at most 4 implementers |
| Step back before diving in | [011] | M0 settles decisions (stack, repo location, drivers) before any feature card |
| Report at the cadence of the work's volatility | [015] | The orchestrator reports per **wave**, not per card |
| Count only delivered and collected | [002] | "Done" = merged to main **and** gates green. Written-but-unmerged doesn't count |
| Six questions every project must survive | [020] | `architecture.md` §8. Every ADR states its cost |
| Static first, cost shape | [019] | Local process first. Edge deploy later, verified |
| Anti-pattern lists and decision tables do most of a prompt's work | [001], [004] | Cards carry a **Don'ts** list and, where inputs vary, a decision table |
| Webhooks: verify the signature, dedupe by event id, return 200 fast | [009] | CONVENTIONS rule for any future inbound webhook (AutoApply callbacks) |
| Reproduce, gather minimal evidence, test one hypothesis at a time | skill `debug-evidence-loop` | Failing-gate protocol (gates.md §4) |
| Local evidence first; confirm before registry calls; never change deps during a scan | skill `dependency-security-scan` | Implementers never add packages. The dependency allowlist in CONVENTIONS |
| Safety net, read-only audit, move-only, one file per commit, verify against the baseline | skill `clean-vibe-codebase` | Protocol for porting code (ScoutLane, JobSync, career-ops) and for later cleanups |
| Never invent facts. Unknown → `[VERIFY: …]` | skill `score-resume` | Every generated artifact (fixtures, docs, M2 prompts) follows this |

## 2. Roles and models

| Role | Model (Claude Code Agent tool) | Tools | Writes code? |
|---|---|---|---|
| **Orchestrator** | Opus (main session) | all | Only integration fixes, merges, CONVENTIONS/plan edits, and the serialised "shared" cards when it chooses |
| **Implementer** | `model: "sonnet"`, `isolation: "worktree"` | Read, Edit, Write, Bash, Grep, Glob | Yes, only the card's owned files |
| **Verifier** | `model: "haiku"`, in its own detached worktree (`git worktree add --detach`) | Read, Bash, Grep, Glob | **Never** |
| **Security auditor** | `model: "opus"` | Read, Bash, Grep, Glob | **Never** |
| **Comprehension checker** | `model: "haiku"` | Read | Never. It reads a card cold and lists ambiguities before dispatch (§5) |

Prompts live in [`prompts/`](prompts). M0-02 installs them as `.claude/agents/jm-*.md` so they can be
invoked by name. Codex users read the same content through `AGENTS.md`.

## 3. The loop (per milestone)

```
orchestrator
 ├─ read: docs/plans/ROADMAP.md → current milestone plan → handoff.md → CONVENTIONS.md
 ├─ (M2+) expand the next milestone's cards just-in-time from docs/spec using the template
 ├─ for each wave in order:
 │    ├─ comprehension check (haiku) on each card in the wave → fix the ambiguities it finds
 │    ├─ dispatch ≤4 implementers in parallel (sonnet, worktree), one card each
 │    ├─ on DONE  → verifier (haiku) on that branch → PASS? merge to main + re-run `pnpm verify`
 │    │           FAIL? send the report back to the same implementer once → still FAIL? orchestrator
 │    │           runs the debug-evidence-loop protocol or re-scopes the card
 │    ├─ on BLOCKED → resolve the need (contract clarification, fixture, file ownership, dependency)
 │    │              and add the resolution to the card and, if general, to CONVENTIONS
 │    ├─ Drift Loop: every deviation found → a new CONVENTIONS rule
 │    └─ wave gate G3 (`pnpm verify:full`) → wave report (one paragraph + table)
 └─ milestone gate G4: security audit + exit criteria evidence + handoff → only then the next milestone
```

**Serialised cards** (run alone, before a parallel wave): anything touching the schema/migrations,
module registries, root configs, `package.json`/lockfile, or `packages/core/src/contracts/*`
shared types. M0 pre-creates every registry entry and stub, so M1 feature cards never edit a shared
index.

## 4. Branches, commits, merges
- Branch `card/<ID>-<slug>` from the current `main`, one card per branch.
- Conventional commits (`feat(jobs): …`, `test(auth): …`). The body ends with `Card: <ID>`.
- If a GitHub remote exists, the implementer runs `gh pr create --draft`. The orchestrator marks it
  ready and merges it.
- The orchestrator merges with `git merge --no-ff`, then re-runs `pnpm verify` on `main`. If that
  fails, it reverts the merge immediately and the card goes back.
- Never force-push, never use `--no-verify` (global CLAUDE.md).

## 5. Comprehension check (before dispatching a wave)
Give each card, alone, to a Haiku subagent with this instruction: *"You will implement this card
with no other context except the files it lists. List every ambiguity, every file you would need
that isn't listed, and every term you don't understand."* Fix the card and re-check until
**BLOCKER = 0 and MAJOR = 0** (log false positives). Only then dispatch it. The first run of this check (2026-09-30) and its fixes
are recorded in [`comprehension-check-log.md`](comprehension-check-log.md).

## 6. What the orchestrator reports to Richard
Per wave: cards merged, cards blocked (with the need), gate results, and new CONVENTIONS rules.
Per milestone: exit criteria with evidence, security audit summary, and the next milestone's first
wave.

## 7. After v1: routines
See [`routines.md`](routines.md): four scheduled maintenance passes ([012], adapted). They always
end in **draft PRs**, never merges.
