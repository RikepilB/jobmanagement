# Domain docs: how skills should read this repo's domain knowledge

Single-context repo, adapted layout (no `CONTEXT.md` or `docs/adr/` directory):

| Upstream expects | This repo uses |
|---|---|
| `CONTEXT.md` (glossary, domain language) | `docs/spec/SPEC.md` §3 Glossary + §4 requirements |
| `docs/adr/NNNN-*.md` | `docs/decisions.md` (one append-only ADR log, `ADR-001`…) |
| contracts | `docs/spec/data-model.md`, `api-rest.md`, `mcp.md`, `cli.md`, `security.md` (each is the single source for its contract) |

Before exploring code for a ticket, read the ticket's **Read first** list, then the SPEC glossary
if a term is unclear. Record new decisions as a new ADR entry in `docs/decisions.md`, never as
separate ADR files. Don't create `CONTEXT.md` or `docs/adr/`.
