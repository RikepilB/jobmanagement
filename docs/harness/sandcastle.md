# Sandcastle: orchestration Mode B (sandboxed, AFK, parallel)

> [`mattpocock/sandcastle`](https://github.com/mattpocock/sandcastle) (`@ai-hero/sandcastle`, MIT)
> runs a coding agent inside a Docker sandbox on its own git branch and returns commits.
> This harness uses it to run whole **waves** of tickets unattended, with at most 4 in parallel,
> each implemented, gated and verified inside its own container. Sandcastle's own repo uses the same
> shape (planner → parallel implement + review, `MAX_PARALLEL = 4`, `.sandcastle/run.ts`), but here
> the planner is replaced by the ticket tracker's `Blocked by:` edges.
> **Intake:** [`intake/sandcastle.yaml`](intake/sandcastle.yaml). Verdict **APPROVE_WITH_RESTRICTIONS**,
> pinned **v0.12.0 @ `e99f832`**. The restrictions below are mandatory.

## 1. Mode A vs Mode B

| | Mode A: Agent-tool worktrees | **Mode B: sandcastle** |
|---|---|---|
| Isolation | git worktree on the host. The agent can read the whole disk | **Docker container.** The agent sees only the repo worktree + declared env |
| Runs while you're away | only while the Claude Code session is open | yes (`npx tsx .sandcastle/run-wave.ts`) |
| Implement → gate → verify | three separate subagent calls | one warm sandbox: `run` → `exec('pnpm verify')` → `run` (verifier, structured output) |
| Verifier output | text table | `Output.object` with a zod schema, so it's machine-comparable |
| Other agents | Claude only | `claudeCode()`, `codex()`, `opencode()`, `pi()`… |
| Needs | nothing extra | WSL2 + Docker, the pinned sandcastle devDependency, and a `claude setup-token` token |
| Use for | M0 waves 1–3 (before the farm exists), serialised/orchestrator tickets, no-Docker days | **all parallel agent waves from M0 wave 4 on** (after M0-16) |

## 2. Mandatory restrictions (from the intake)

| # | Rule | Why |
|---|---|---|
| SC-1 | Run sandcastle **only from the WSL agent-farm clone** (`~/jm-farm`, ext4), never against the canonical repo `C:\Users\a2021\dev\jobmanagement` | Open bugs: #854 (bind-mount worktrees corrupt host worktree git metadata, Linux/macOS too), #859 (Windows host `node_modules` clobbered by Linux installs; WSL-mount installs time out), plus #876/#924/#940/#996 (Windows host failures). A disposable Linux-native clone avoids all of them. If metadata breaks: `git worktree prune`, or delete and re-clone the farm |
| SC-2 | Provider **`docker()` only** for AFK runs. Never `noSandbox()` for implementer or verifier runs | `noSandbox` runs the agent on the host: no isolation |
| SC-3 | Never set `devices`, `groups`, `network`, or any mount of the Docker socket. The only allowed mount is the pnpm store, **read-only** (if that breaks installs, use `copyToWorktree: ["node_modules"]` instead; M0-16 decides) | Those are the scanner's "privileged container / container escape" findings. They're opt-in, so we never opt in |
| SC-4 | Build our **own** Dockerfile from `sandcastle init --template blank`, replacing `curl … claude.ai/install.sh \| bash` with `npm install -g @anthropic-ai/claude-code@<pinned>` (2.1.285 observed on 2026-09-30) and pinning pnpm via corepack | Removes unpinned remote-script execution at image build ("external script fetching" finding) |
| SC-5 | `.sandcastle/.env` holds **only** `CLAUDE_CODE_OAUTH_TOKEN`, a dedicated token Richard creates with `claude setup-token` and pastes himself. No other secrets, ever. It's gitignored. Verify with `git check-ignore .sandcastle/.env` before any push | The token is injected into the container env ("credential access", a confirmed capability). The agent can read it, so keep it revocable and single-purpose |
| SC-6 | `branchStrategy: { type: "branch", branch: "card/<ID>-<slug>" }`. **One distinct branch per ticket.** Never `head`. Never `merge-to-head`: **only the orchestrator merges** (gate G2) | Draft-PR-style review [article 012]. Two runs sharing a branch collide |
| SC-7 | `captureSessions` left at its default (true): transcripts are copied to the **WSL** `~/.claude/projects` for review. Nothing from the Windows `~/.claude` is mounted | Transcripts are audit evidence. The host config never enters containers |
| SC-8 | Prompts never interpolate untrusted text into `` !`command` `` shell expansion. `promptArgs` carry only ticket ids, paths and branch names | Shell expansion runs inside the sandbox |
| SC-9 | Upgrading sandcastle = a new SkillSpector scan + intake record + a serialised card. Never `^` ranges | CLAUDE.md supply-chain rule |

## 3. Topology

```
Windows (canonical)                         WSL2 Ubuntu (agent farm, disposable)
C:\Users\a2021\dev\jobmanagement   ◄─push── ~/jm-farm  (git clone /mnt/c/Users/a2021/dev/jobmanagement)
  orchestrator session (Opus)       card/*    │  pnpm install (Linux binaries)
  reviews · merges (G2) · updates             │  npx tsx .sandcastle/run-wave.ts --milestone M1 --wave 3
  ticket Status (single writer)               ▼
                                     docker: sandbox per ticket (max 4)
                                       implement (sonnet) → exec pnpm verify → verify (haiku, Output.object)
                                       commits on card/<ID>-<slug> → report .sandcastle/reports/<ID>.json
```

- **The farm never edits ticket files.** It writes reports and pushes branches. The orchestrator is
  the **single writer** of `Status:` and `## Comments` in the canonical repo.
- Start of each wave, in the farm: `git fetch origin && git checkout main && git reset --hard origin/main`
  (the farm is disposable and has no local work by rule), then `pnpm install --frozen-lockfile`.
- End of each wave, in the farm: `git push origin 'card/*'`. Then, in the canonical repo, the
  orchestrator reviews each report + diff, merges passing branches (`resolving-merge-conflicts` if
  needed), updates statuses and runs G2/G3.

## 4. Wave runner contract (`.sandcastle/run-wave.ts`, built in M0-16)

Command: `npx tsx .sandcastle/run-wave.ts --milestone M1 [--wave 3] [--max 4] [--only M1-07,M1-08] [--dry-run]`

1. Parse every `docs/plans/<M>/tickets/*.md` header. **Frontier** = `Status: open` ∧ `Owner: agent`
   ∧ every `Blocked by:` ticket is `merged` (∧ `Wave` matches if `--wave` is given). `--dry-run`
   prints the frontier and exits.
2. For each frontier ticket, with a semaphore of `--max` (default 4, never more):
   ```ts
   await using sandbox = await createSandbox({
     sandbox: docker({ imageName: "jm-sandcastle:local",
                       mounts: [{ hostPath: "~/.local/share/pnpm/store", sandboxPath: "/home/agent/.local/share/pnpm/store", readonly: true }] }),
     branch: `card/${id}-${slug}`,
     hooks: { sandbox: { onSandboxReady: [{ command: "pnpm install --frozen-lockfile" }] } },
   });
   const impl = await sandbox.run({ name: `Implementer ${id}`, agent: claudeCode("claude-sonnet-5"),
     promptFile: ".sandcastle/implement-prompt.md",
     promptArgs: { TICKET_ID: id, TICKET_PATH: path, BRANCH: branch, MILESTONE_INDEX: indexPath },
     maxIterations: 3, idleTimeoutSeconds: 900 });
   const gate = await sandbox.exec("pnpm verify");            // non-zero exitCode is returned, not thrown
   // if gate fails and impl made commits: one more sandbox.run with promptArgs.FAILURE_TAIL (last 60 lines)
   const verdict = await sandbox.run({ name: `Verifier ${id}`, agent: claudeCode("claude-haiku-4-5-20251001"),
     promptFile: ".sandcastle/verify-prompt.md", promptArgs: { TICKET_ID: id, TICKET_PATH: path, BRANCH: branch },
     output: Output.object({ tag: "report", schema: VerifierReport }) });
   ```
   API names come from the v0.12.0 README (`createSandbox`, `sandbox.run`, `sandbox.exec`,
   `Output.object`, `claudeCode(model, {effort})`). **M0-16 verifies each against the installed
   package before relying on it**, and reports BLOCKED on any mismatch.
3. `VerifierReport` (zod, in `.sandcastle/schemas.ts`):
   `{ ticket, branch, checks: {name, status: "PASS"|"FAIL"|"NOT_FOUND", evidence}[], filesOutsideOwns: string[], dependencyChanges: boolean, verdict: "PASS"|"FAIL", reason }`.
4. Write `.sandcastle/reports/<ID>.json` (gitignored) = `{ implementer stdout tail, gate exitCode + tail, report, commits }`.
   Print a wave table. Push the branches.
5. Never merge, never edit `docs/plans/**`, never push to `main`.

## 5. Prompts (`.sandcastle/*.md`, created in M0-16 from the harness prompts)
- `implement-prompt.md` = `docs/harness/prompts/implementer.md` body. Drop the worktree/branch
  lines (sandcastle is already on the branch and the hook ran the install). Add: "Read the milestone
  index `{{MILESTONE_INDEX}}` first (the header rules and Shared service signatures apply to your
  ticket), then your ticket `{{TICKET_PATH}}`. When done and `pnpm verify` passes, print
  `<promise>COMPLETE</promise>`." The runner passes `MILESTONE_INDEX` (e.g. `docs/plans/M1-mvp-core.md`).
- `verify-prompt.md` = `docs/harness/prompts/verifier.md` without the worktree setup/teardown (the
  sandbox *is* the throwaway checkout). It ends by emitting the `<report>` JSON matching
  `VerifierReport`.
- **Drift rule:** the harness prompts are the source of truth. When one changes, update the
  `.sandcastle/` copy in the same commit (gates.mjs checks that the shared paragraphs match; M0-16).

## 6. Cost and safety valves
- At most 4 sandboxes at once, `maxIterations: 3` for implementers, and `idleTimeoutSeconds` 900.
  A wave stops starting new tickets after 2 consecutive BLOCKED/FAIL results and reports.
- Runs consume Richard's Claude plan through the setup-token. The wave report lists iterations
  per ticket (`result.iterations`) to watch usage.
- Kill switch: Ctrl+C aborts the runner, and worktrees with uncommitted changes are preserved
  (`preservedWorktreePath` is printed). Then `docker ps`, and remove stray containers.
