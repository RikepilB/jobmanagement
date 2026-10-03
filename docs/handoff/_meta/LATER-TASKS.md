# Later tasks (deferred, not started)

## 2026-09-30: interrupted request from Richard (not started; needs confirmation per item)
> "make a repo on github and then a vercel deployment and then test the project management and add
> bulk for csv files and pdfs and docs and so on. also add all my data from path peer in this new one"

| Item | Notes / gate before starting |
|---|---|
| ~~GitHub repo~~ **DONE 2026-10-03** | Richard asked explicitly: public `RikepilB/jobmanagement`, MIT, OSS files added (M0-02). Branch fast-forwarded into `main` and pushed. Local paths with the Windows username remain in docs (low sensitivity, not rewritten) |
| Vercel deployment | There's **no app code yet** (M0 not built). ADR-015 targets local `jm serve` + SQLite. Vercel needs a hosted DB (e.g. Turso/libSQL, Neon) plus a serverless adapter, which conflicts with the single-owner local design and SEC-07/SEC-18. Decide hosting (blueprint §13 #3) first. Possibly a static docs site only (Astro, M7) |
| "Test the project management" | Means building M0 → M1 (the harness is ready). Start with the M0-01 export + finishing M0-02 |
| Bulk import: CSV files, PDFs, docs | New scope. CSV import of jobs/contacts fits M1-11 (imports) as a new ticket. PDF/doc ingestion (resumes, JDs) fits M2 (resume parsing; the PDF dependency needs intake). Write the tickets with `to-tickets` |
| Import all PathPeer data | M0-01 export (user-run in Chrome, stored outside OneDrive) → M1-11 importer. **The most time-sensitive item** (the account is archived) |
