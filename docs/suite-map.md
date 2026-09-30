# Suite map: jobmanagement × AutoApply × ResumeScorer

Three repos, one job search. This doc fixes **who owns what**, so the three never grow
competing ledgers or competing scores. Sources: `AutoApply/docs/2026-07-19-autoapply-harness-design.md`
(approved design, no code yet) and `ResumeScorer/PROJECT.md` + `scripts/job_finder.py`
(working, 197-job ledger).

## 1. Roles

| Repo | Role | Posture | Owns |
|---|---|---|---|
| **jobmanagement** | **System of record** plus the human workspace: jobs, contacts, companies, outreach, activity, analytics | **Draft-only**, never submits or sends anything | The database, the job-fit score, statuses, activity log, facts profile, the API and MCP |
| **AutoApply** | **Apply engine**: fills and submits ATS forms under a consent gate | **Acts on Richard's behalf** within its permission gate (calibration mode first) | Form fillers (Greenhouse/Lever/Ashby Playwright), the AI-agent fallback, screening-answer drafting, and submission |
| **ResumeScorer** | **Resume-quality lab**: rubric score + honest edit plan (hiring-agent) | Advise-only, Richard edits | The resume-quality ("ATS") score, the rubric knowledge, the `score-resume` skill and `resume-optimizer` agent |

The blast radius stays split, as both existing designs already argue. jobmanagement never
holds site credentials and never POSTs an application. AutoApply is a separate client with its
own consent gate. If AutoApply breaks, the tracker is unaffected.

```
                 ┌──────────────── jobmanagement (system of record) ────────────────┐
 ResumeScorer    │ jobs · job_sightings · applications · contacts · outreach ·      │   Clipper / UI / MCP
 job_finder.py ──┼──► import as job_sightings (P5)                                  │◄── Richard
 (Richard's PC:  │                                                                  │
  hiring-agent) ──┼──► POST /api/v1/resume-versions/:id/quality (report ingest, P6) │
                 │ job_analyses (THE fit score)  ──── GET /api/v1/jobs?min_fit=… ───┼──► AutoApply
                 │ applications.status/outcome   ◄─── PATCH /api/v1/applications ───┼─── (submits, logs
                 │ documents (cover letter, tailored resume) ◄── POST /documents ───┼────  answers + outcome)
                 └──────────────────────────────────────────────────────────────────┘
```

## 2. Integration contracts

### ResumeScorer → jobmanagement
| What | Contract | Phase |
|---|---|---|
| `data/jobs/jobs.json` ledger (`{jobs: {<sha1>: {id, source, company, title, location, url, posted, snippet, first_seen, last_seen}}, last_run}`) | Import each record as a **`job_sighting`** (`source`, `external_id = id`, `raw = record`), never directly as a `job`. Its id hashes `source\|company\|title\|location`, so it is **per-source**, and the same role from HN and Greenhouse gives two ids. Sightings dedupe into `jobs` by fingerprint (ADR-005) | P1 importer, then P5 |
| `job_finder.py` sources (HN Algolia, jobright-ai GitHub lists, Ashby/Greenhouse/Lever by slug) + `config.json` (`ats_boards, role_keywords, location_keywords, exclude_keywords, github_lists`) | Port to `packages/sources` adapters. `config.json` becomes the discovery settings (same keys). Keep the Python script as the reference implementation until parity | P5 |
| hiring-agent score report | **Push, not pull.** hiring-agent is a single-machine tool (hardcoded Windows paths, PowerShell 5.1, venv in `C:\Users\a2021\dev\hiring-agent`), so a Docker or Vercel worker can't run it. A small local script on Richard's PC (or the `score-resume` skill) POSTs the report to `/api/v1/resume-versions/:id/quality`, or the report file is uploaded in the UI. The app never touches ResumeScorer's filesystem. The score stays **separate from job fit** (ADR-006) | P6 |

### AutoApply ↔ jobmanagement
| AutoApply design item | Maps to | Change to AutoApply's design |
|---|---|---|
| §5 its **own fit-score prompt** with a 75% gate | `job_analyses.score` from jobmanagement (one prompt, versioned, per resume version) | **Drop the separate fit prompt.** AutoApply reads `GET /api/v1/jobs?min_fit=75&status=SAVED`. The 75% threshold becomes AutoApply config. One fit score for the whole suite (ADR-012) |
| §5 "Important" signal (strong category + experience match) | A field on the job-fit analysis (`priority: important\|normal`) or a user tag | Computed once in jobmanagement |
| §3 calibration mode (every job takes the "ask" branch) | A jobmanagement **review queue** (applications with `autoapply_state = proposed`) | AutoApply proposes, and Richard approves in the jobmanagement UI |
| §10 ledger fields `applied_at, resume_used, answers_given, outcome` | `applications.applied_at`, `applications.resume_version_id`, `documents` (answers as a document, kind `application_answers`), `activities` (outcome) | AutoApply writes over the API. **No second `jobs.json` ledger** |
| §6 cover letter (important tier only) | `documents` (kind `cover_letter`) linked to the application | Drafted by `richard-cover-letter`, stored in jobmanagement |
| §7 persona/facts file (not written yet) | The **single facts source** (§3) | Shared, not AutoApply-private |
| §4 Tier A discovery boards (Built In, HiringCafe, Job Bank, WWR, RemoteOK, Himalayas, Remotive…) | `packages/sources` adapters | Discovery lives in jobmanagement, and AutoApply consumes it |
| §11 non-goals: no LinkedIn, no credentials, no Tier B | Same as jobmanagement ADR-007 | No change |
| Reference project ApplyPilot | **AGPL**: ideas only (landscape §1) | Don't copy code into either MIT repo |

**Status field for AutoApply:** add `applications.autoapply_state` (`none | proposed | approved |
submitted | failed | skipped_below_threshold`), plus an activity for every transition, so
calibration dry-runs are distinguishable from real submissions (AutoApply §10's goal).

## 3. One facts source (the honesty anchor)

Today the same facts live in four places: `richard-outreach` proof points and identity anchor,
AutoApply's planned persona/facts file (§7), career-ops-style `cv-facts.json`, and the resumes
themselves. **Proposal:** one versioned `profile/facts.yml` owned by jobmanagement (identity,
work authorization, compensation floor, proof points with metrics, forbidden claims), exported
read-only to the skills and to AutoApply. Where it physically lives, in the app DB or in the
Second Brain vault, is an **open decision** (blueprint §13).

## 4. Build order across the suite

1. jobmanagement P0–P1 (system of record + importer) comes **before** AutoApply writes any code,
   so AutoApply is born as an API client and not as a ledger owner.
2. ResumeScorer keeps working as-is. Its ledger gets imported, and nothing breaks.
3. AutoApply starts after jobmanagement P2 (fit analysis exists), in calibration mode.
4. P6: hiring-agent adapter and the facts export to skills.

## 5. Edits needed in the other repos (not made; propose to Richard)

- `AutoApply/docs/2026-07-19-autoapply-harness-design.md` §5/§10: replace its own fit prompt
  and `jobs.json` ledger with the jobmanagement API (this doc §2).
- `ResumeScorer/docs/job-finder.md`: note that jobmanagement imports the ledger as sightings,
  and that the ledger remains the source for ResumeScorer's own use.
