# M0-02 · Repo bootstrap outside OneDrive

Status: merged
Blocked by: none
Wave: 1
Owner: orchestrator
Model: -

**Wave** 1 · **Owner** orchestrator · **Depends on** Richard's confirmation of ADR-015 (stack) and ADR-019 (location) · **Parallel-safe with** M0-01

**Goal.** The code repo exists at `C:\Users\a2021\dev\jobmanagement` with the planning docs,
licence, ignore rules and harness agents, in one initial commit.

**Owns.** Everything at the new repo root: `docs/**` (copied from the OneDrive planning folder),
`AGENTS.md` (copied), `README.md`, `LICENSE`, `THIRD_PARTY_NOTICES.md`, `.gitignore`,
`.gitattributes`, `.editorconfig`, `.node-version`, `.claude/agents/jm-implementer.md`,
`jm-verifier.md`, `jm-security-auditor.md` (copied verbatim from `docs/harness/prompts/`),
`handoff.md`.

**Contract.**
- `git init -b main`. `LICENSE` = MIT, "Copyright (c) 2026 Richard Pillaca".
- `.gitignore`: `node_modules/`, `dist/`, `coverage/`, `.env*` + `!.env.example`, `*.db`,
  `*.db-shm`, `*.db-wal`, `data/`, `.jm/`, `*.har`, `captures.jsonl`.
- `.gitattributes`: `* text=auto eol=lf`. `.node-version`: `24`.
- `README.md` sections: what it is (2 lines) · status · quick start (`pnpm install`,
  `pnpm build`, `jm init`, `jm serve`) · **For agents** (link to AGENTS.md) · **Privacy** (what
  leaves the machine: nothing in M1) · honest data-at-rest note (SEC-08) · licence.
- The OneDrive planning folder gets a `MOVED.md` pointing to the new path. **Nothing is deleted
  there.**

**Acceptance.** `git -C C:\Users\a2021\dev\jobmanagement log --oneline` shows exactly one commit ·
`git status` clean · the three `.claude/agents/jm-*.md` files parse (frontmatter has `name`,
`description`, `tools`, `model`).

**Don'ts.** Don't push anywhere. Don't create a GitHub repo (that's Richard's call). Don't move or
delete the OneDrive folder.

## Comments

- 2026-09-30: repo created at `C:/Users/a2021/dev/jobmanagement` and docs committed (`3d23246`). Remaining: LICENSE, README, .gitignore, .gitattributes, .node-version, `.claude/agents/jm-*.md`.
- 2026-10-03: orchestrator: done. Added LICENSE, THIRD_PARTY_NOTICES.md (Matt Pocock skills MIT text), README, .gitignore (also ignores handoff transcripts/snapshots), .gitattributes, .editorconfig, .node-version, `.claude/agents/jm-*.md`, plus OSS files (CONTRIBUTING, CODE_OF_CONDUCT, SECURITY, `.github/` templates). Deviations, at Richard's explicit request: a public GitHub repo `RikepilB/jobmanagement` was created and pushed (overrides the "don't push" line); history has 4 commits instead of one.
