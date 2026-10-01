# M1-09 · Capture module + safe-fetch

Status: open
Blocked by: M1-01, M1-05
Wave: 3
Owner: agent
Model: sonnet

**Wave** 3 · **Model** sonnet · **Depends on** M1-01, M1-05 · **Parallel-safe with** M1-07, M1-08, M1-10

**Goal.** REQ-JOB-02 and SEC-13 (`POST /capture/job`).

**Owns.** `apps/server/src/modules/capture/**`, `apps/server/src/shared/safe-fetch.ts`,
`apps/server/src/shared/safe-fetch.test.ts`.

**Contract.**
- `safeFetch(url, {timeoutMs=10000, maxBytes=2_000_000, maxRedirects=3, resolver?, fetchImpl?}) → Promise<{finalUrl: string; contentType: string; body: string}>`.
  - `resolver: (hostname: string) => Promise<Array<{address: string; family: 4 | 6}>>`. The default
    wraps `lookup(hostname, { all: true })` from `node:dns/promises` (allowed here because this is
    server code, not core).
  - `fetchImpl` defaults to global `fetch`, and tests inject a fake.
  - **Size rule:** if `Content-Length` > `maxBytes`, abort before reading. Otherwise read the body
    stream and abort as soon as the received bytes exceed `maxBytes`. Both cases throw
    `AppError('unprocessable', …, {reason: 'content_too_large'})`. Headers don't count.
  - Other failures throw `AppError('unprocessable', …, {reason})` with reason `blocked_address` |
    `bad_scheme` | `bad_port` | `too_many_redirects` | `bad_content_type` | `timeout` |
    `fetch_failed`.
  It rejects non-http(s) and ports other than 80/443. It rejects any resolved address that is
  loopback, private (10/8, 172.16/12, 192.168/16), link-local (169.254/16, fe80::/10), CGNAT
  (100.64/10), unique-local (fc00::/7), multicast or unspecified. It follows redirects manually
  (`redirect: 'manual'`) and re-validates each hop. Content-type must be `text/html` or
  `application/(ld+)json`. It stops reading past `maxBytes`.
- Capture decision table:

| Input | Behaviour |
|---|---|
| `url` on `*.linkedin.com`, no html/text | 422 `unprocessable`, `details.reason='linkedin_requires_client_capture'` |
| `url` only | `safeFetch` → html → steps below |
| `html` (± url) | no fetch. Parse the given html |
| `text` only | no fetch. Title = the `title` input (required, else 400). Description = text |
| html has JSON-LD JobPosting | `method:'jsonld'`, map ParsedJob → `JobCreateInput` |
| no JSON-LD | `method:'html'`: title per the **title rule** below, description = readable text, `warnings:['no_structured_data']` |
| parsed job has no `companyName` (the input has no company field) | company name = `registrableDomain(url)` when a url exists, else `"Unknown company"`, and `warnings += 'company_unknown'` |

**Title rule** (html method): take the `<title>` text and split it on ` - `, ` | `, ` – `, ` — `.
While there's more than one part, drop the **last** part if (case-insensitively, trimmed) it equals
the company name, the first label of the registrable domain, or one of `careers`, `jobs`,
`greenhouse`, `lever`, `ashby`, `workday`, `linkedin`, `indeed`. Join the remaining parts with ` - `.
An empty `<title>` → the `title` input, or 422 `unprocessable` with reason `title_missing`.

  Then `createJob(db, input, ctx, {source: ctx.actor === 'extension' ? 'extension' : 'url'})`, passing
  `track`, `status` and `tags` through unchanged.

**Acceptance.** Tests (no real network; `safeFetch` with a fake resolver and a fake fetch):
- `http://127.0.0.1/x`, `http://10.0.0.5`, `http://169.254.169.254/latest`, `http://[::1]/` and a
  public host that redirects to `http://192.168.1.1` → all rejected.
- Port 8080 → rejected.
- A JSON-LD fixture HTML (from M1-01) posted as `html` creates the job with the parsed fields.
- A LinkedIn URL alone → 422 with that reason.
- Capturing the same URL twice → `created:false`.
- `Content-Length: 3000000` → `content_too_large` without reading the body. A streamed body past
  2 MB → the same error.
- Title rule: `Senior Engineer - Example Corp` (company `Example Corp`) → `Senior Engineer`, and
  `Engineer | Careers | Example` → `Engineer`.
- Text-only capture without `title` → 400.

**Don'ts.** No real network in tests. No fetches that bypass `safeFetch`.

## Comments
