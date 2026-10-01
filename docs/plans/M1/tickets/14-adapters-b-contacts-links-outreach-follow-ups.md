# M1-14 · Adapters B: contacts, links, outreach, follow-ups

Status: open
Blocked by: M1-06, M1-08, M1-10
Wave: 4
Owner: agent
Model: sonnet

**Wave** 4 · **Model** sonnet · **Depends on** M1-06, M1-08, M1-10 · **Parallel-safe with** M1-11, M1-12, M1-13

**Goal.** REQ-AGT-01/02/03 for contacts, links, outreach and follow-ups.

**Read first.** `docs/spec/mcp.md` §1–§2 · `docs/spec/cli.md` §0, §2 · api-rest §0, §5–§8 · M0-09, M0-10,
M0-11 (same helpers as M1-13).

**Owns.** `packages/client/src/{contacts,links,outreach,followups}.ts`,
`packages/mcp/src/tools/{contacts,links,outreach,followups}.ts`,
`apps/cli/src/commands/{contacts,links,outreach,followups}.ts` + tests.

**Contract.** mcp.md §2 tools `contacts_*`, `link_contact_job`, `outreach_*`, `followups_due`,
`contacts_snooze`. cli.md §2 commands `jm contacts …`, `jm link/unlink`, `jm outreach …`,
`jm followups`, `jm remind`. `about`, `headline` and outreach `body` are wrapped untrusted.

**Acceptance.** Tests: MCP `outreach_log` with f1 moves a saved contact to contacted (via
`contacts_get`) · `followups_due` returns that contact after the fake clock advances past
`followUpDays` · CLI `jm outreach log … --body-file` reads the file locally and posts its content ·
the snooze CLI prints the new date.

## Comments
