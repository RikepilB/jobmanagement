# M0-16 · Sandcastle agent farm (Mode B orchestration)

Status: open
Blocked by: M0-03
Wave: 3
Owner: orchestrator
Model: opus

**Wave** 3 (after M0-03; M0-04 may run in parallel because the files are disjoint) · **Model** orchestrator (Opus) with Richard for two human steps · **Depends on** M0-03 · **Parallel-safe with** M0-04

**Goal.** From M0 wave 4 onward, parallel waves run AFK in sandcastle Docker sandboxes from a WSL
agent-farm clone, following every restriction in `docs/harness/sandcastle.md` §2.

**Read first.** `docs/harness/sandcastle.md` (all) · `docs/harness/intake/sandcastle.yaml` ·
`docs/harness/prompts/implementer.md` and `verifier.md` · `docs/agents/issue-tracker.md` ·
`/c/Users/a2021/dev/refs/sandcastle/README.md` (the pinned v0.12.0 source, read-only).

**Inputs that aren't obvious.**
- `CLAUDE_CODE_OAUTH_TOKEN`: **Richard** runs `claude setup-token` in WSL and pastes the value into
  `~/jm-farm/.sandcastle/.env` himself. No agent ever reads, prints or copies it (SC-5).
- WSL facts (2026-09-30): Ubuntu, UID/GID 1000, Docker reachable from WSL, **no Node in WSL yet**.

**Owns.** `.sandcastle/Dockerfile`, `.sandcastle/.env.example`, `.sandcastle/.gitignore`,
`.sandcastle/run-wave.ts`, `.sandcastle/schemas.ts`, `.sandcastle/implement-prompt.md`,
`.sandcastle/verify-prompt.md`, `.sandcastle/README.md`, `scripts/farm-sync.sh`, and a row in
`docs/harness/gates.md` §5.

**Must not touch.** `package.json`/lockfile (M0-03 already pinned `@ai-hero/sandcastle@0.12.0` +
`tsx`), `docs/plans/**` statuses (the orchestrator updates those in the canonical repo), and anything
in `apps/` or `packages/`.

**Contract.**
1. **WSL prerequisites** (commands in `.sandcastle/README.md`): Node 24 via nvm, `corepack enable`,
   pnpm 10.30.3; `docker version` works from WSL.
2. **Farm:** first `git config --global --add safe.directory /mnt/c/Users/a2021/dev/jobmanagement`
   (otherwise git in WSL refuses the Windows-owned repo with "dubious ownership" on clone and push), then
   `git clone /mnt/c/Users/a2021/dev/jobmanagement ~/jm-farm` (ext4, **not** `/mnt/c`), then
   `pnpm install --frozen-lockfile`. `scripts/farm-sync.sh`: `git fetch origin && git checkout main
   && git reset --hard origin/main && pnpm install --frozen-lockfile` (the farm is disposable, SC-1).
3. **Scaffold:** in the farm, run `npx @ai-hero/sandcastle init --template blank`, **review every
   generated file**, then keep only what this contract lists (commit the kept files from the
   canonical repo, never from the farm).
4. **Dockerfile (SC-4):** start from the scaffolded one. Replace the `curl … install.sh | bash` line
   with `npm install -g @anthropic-ai/claude-code@2.1.285` (re-check the version and record it),
   add `corepack enable && corepack prepare pnpm@10.30.3 --activate`, keep the non-root `agent`
   user with UID/GID 1000, and base on Node 24. Build with `npx @ai-hero/sandcastle docker
   build-image` (image `jm-sandcastle:local`).
5. **`.sandcastle/.gitignore`** includes `.env`, `logs/`, `worktrees/`, `reports/`. Proof: `git
   check-ignore .sandcastle/.env` prints the path.
6. **`run-wave.ts` + `schemas.ts`** exactly per `docs/harness/sandcastle.md` §4. **Before writing
   them, verify every API name** (`createSandbox`, `sandbox.run`, `sandbox.exec`, `Output.object`,
   `claudeCode`, `docker({imageName, mounts})`, `result.commits`, `result.iterations`) against the
   installed `node_modules/@ai-hero/sandcastle` type definitions. Any mismatch → adapt to the real
   API and record it in `.sandcastle/README.md` → "API notes". If behaviour is missing, report BLOCKED.
7. **Prompts** per sandcastle.md §5, with a comment at the top of each naming its source harness prompt.
   The implement prompt **must** tell the agent to read `{{MILESTONE_INDEX}}` before `{{TICKET_PATH}}`.
8. **Dry run:** `npx tsx .sandcastle/run-wave.ts --milestone M0 --dry-run` lists the correct frontier
   (after M0-03/M0-16 are merged, the frontier is exactly **M0-04 and M0-06**; M0-05 waits for M0-04).
9. **Live smoke:** a throwaway ticket file `docs/plans/M0/tickets/99-sandcastle-smoke.md` (Owner
   agent, goal: "add `scripts/sandcastle-smoke.txt` containing `ok`"; delete it after) runs end to
   end: a commit on `card/M0-99-sandcastle-smoke`, the gate exec runs, and the verifier report
   parses as `VerifierReport`. The smoke run **must prove the dependency strategy**: the read-only
   pnpm-store mount may fail (EROFS, or no hard links across mounts). If it does, switch to upstream's own
   pattern, `copyToWorktree: ["node_modules"]` from the Linux-native farm, drop the store mount, and
   record which strategy works in `.sandcastle/README.md`. The branch is pushed to the canonical repo, then deleted after
   inspection.

**Acceptance.**
- Evidence for items 1, 5, 8 and 9 in the ticket report (command output tails).
- The written API notes list every checked name as confirmed or adapted.
- `docker ps` is empty after the smoke run (no stray containers).
- Checked by grep on `.sandcastle/`: no `noSandbox`, `devices`, `groups`, `network`, or
  `/var/run/docker.sock` in any runner code or config (SC-2/SC-3).

**Don'ts.**
- Never run sandcastle against `C:\Users\a2021\dev\jobmanagement` or any `/mnt/c` path.
- Never print or commit the token.
- Never use `merge-to-head`/`head`.
- Never let the runner edit `docs/plans/**`.
- Never upgrade sandcastle beyond 0.12.0 here.

## Comments
