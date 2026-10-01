# M1-15 · Adapters C: activities, tags, notes, search, settings, summary, imports, exports

Status: open
Blocked by: M1-03, M1-04, M1-11, M1-12
Wave: 5
Owner: agent
Model: sonnet

**Wave** 5 · **Model** sonnet · **Depends on** M1-03, M1-04, M1-11, M1-12 · **Parallel-safe with** M1-16, M1-17

**Read first.** `docs/spec/mcp.md` §1–§2 · `docs/spec/cli.md` §0, §2 · api-rest §0, §9–§13 · M0-09, M0-10,
M0-11.

**Owns.** `packages/client/src/{activities,tags,notes,search,settings,summary,imports,exports}.ts`,
`packages/mcp/src/tools/{activities,tags,notes,search,summary}.ts`,
`apps/cli/src/commands/{activities,tags,notes,search,settings,summary,imports,exports}.ts` + tests.

**Contract.** mcp.md tools `activity_recent`, `notes_add`, `tags_apply`, `search`, `pipeline_summary`.
cli.md commands `jm activity`, `jm note`, `jm tag/untag`, `jm search`, `jm settings get/set`,
`jm import resumescorer|pathpeer` (reads files locally, posts content, polls `/tasks/:id` every 1 s
until done, prints the report), `jm export json|csv`, and `jm help agents` (prints the AGENTS.md
"operating" subsection).

**Acceptance.** Tests: the import CLI with the synthetic ResumeScorer fixture prints the created
count, and a re-run prints 0 created · `pipeline_summary` returns counts matching the REST
`/summary` · `jm settings set followUpDays=5` persists · `jm export csv jobs --out <tmp>` writes a file.

## Comments
