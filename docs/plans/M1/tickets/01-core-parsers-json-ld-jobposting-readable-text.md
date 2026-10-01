# M1-01 · Core parsers: JSON-LD JobPosting + readable text

Status: open
Blocked by: M0-15
Wave: 1
Owner: agent
Model: sonnet

**Wave** 1 · **Model** sonnet · **Depends on** M0 · **Parallel-safe with** M1-02, M1-03, M1-04

**Goal.** Pure functions that turn a job page's HTML into a normalised job draft.

**Read first.** api-rest §4 · data-model §2 `jobs` · schema.org `JobPosting` fields (title,
hiringOrganization.name/sameAs, jobLocation.address, jobLocationType, baseSalary, datePosted,
validThrough, description, identifier, url, applicantLocationRequirements).

**Owns.** `packages/core/src/parsers/**`, `packages/core/test-fixtures/**`.

**Contract.**
- `parseJobPostingJsonLd(html: string): ParsedJob | null`. Finds every
  `<script type="application/ld+json">`, tolerates arrays, `@graph`, and invalid JSON blocks (skip
  them), and picks the first `@type` `JobPosting`. It maps into `ParsedJob {title, companyName?,
  companyUrl?, locationText?, locationType, salaryMin?, salaryMax?, salaryCurrency?, salaryPeriod?,
  postedAt?, descriptionText?, atsJobId?, url?}`. HTML in `description` is converted to plain text
  (block tags → newlines, entities decoded).
- `extractReadableText(html): {title: string | null; text: string}`. It drops
  `script/style/noscript/nav/header/footer/svg` blocks, prefers `<main>` or `<article>` content,
  strips tags, decodes common entities (`&amp; &lt; &gt; &quot; &#39; &nbsp;` + numeric), collapses
  whitespace, and caps the result at 50,000 characters.
- `detectAts(url): {provider, jobId} | null` for `boards.greenhouse.io`/`job-boards.greenhouse.io`,
  `jobs.lever.co`, `jobs.ashbyhq.com`, `*.myworkdayjobs.com`.
- No dependencies beyond zod (regex/string based; the limitation is noted in a code comment).

**Decision table: `jobLocationType` / salary mapping**
| JSON-LD | Ours |
|---|---|
| `jobLocationType: "TELECOMMUTE"` | `remote` |
| no `jobLocationType`, has `jobLocation` | `onsite` (`hybrid` only if the text contains "hybrid") |
| `baseSalary.value.minValue/maxValue` + `unitText: YEAR/MONTH/HOUR` | min/max + period `year/month/hour` |
| `baseSalary.value.value` (single number) | min = max = value |
| anything missing | field omitted (never invented) |

**Acceptance.** `pnpm --filter @jm/core test -- parsers`. **Synthetic** fixtures (write them yourself;
companies `Example Corp`, domains `example.com`), shaped like Greenhouse, Lever and Ashby pages with
JSON-LD, one `@graph` page, one page with a broken JSON-LD block plus a valid one, and one page with
no JSON-LD. Assert exact extracted fields for each. `detectAts` table test.

**Don'ts.** No network. No copying real job pages as fixtures. No HTML parser libraries.

## Comments
