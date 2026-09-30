# PathPeer teardown: what we are replacing

Observed 2026-09-29 on Richard's own PathPeer portal (account **archived**, so it is read-only:
"You can access the modules but cannot make any changes"). Everything was read through the UI
and the tRPC calls the app makes on its own. We never read tokens, headers or cookies.
Contact names, emails and profile URLs are left out on purpose. This repo is meant to go public.

## 1. Stack fingerprint

| Signal | Observed |
|---|---|
| Framework | Next.js (Sentry `sentry.javascript.nextjs` 8.55), Chakra UI tabs + Tailwind classes |
| API | tRPC over `/api/trpc/<router>.<procedure>`, batched GETs, superjson `{json, meta}` inputs |
| Auth | `/api/auth/session` (NextAuth), and tRPC also needs a non-cookie credential. A direct in-page `fetch` returned **401** |
| IDs | 24-hex Mongo ObjectIds, so the backend is MongoDB |
| Storage | Avatars and logos on GCS (`storage.googleapis.com/shapeshifter-avatar/...`) |
| Tenancy | B2B coaching model. An `organization` has coaches, and a `client` is the job seeker. The public URL is `/public/<clientId>?view=<module>` |
| Extension | "Career Clipper" (the Career OS harness calls it that) clips LinkedIn profiles and jobs into the board |

## 2. Modules (sidebar) and what each one does

| Module | `view=` | Features observed | tRPC procedures |
|---|---|---|---|
| Summary | `summary` | Dashboard: weekly execution %, overall % since start, job-app target (x/10 per week), contacts target (x/50 per week), job pipeline counts, networking pipeline counts, recent notes, goal (locations, work modes, salary floor), target careers, roadmap %, documents | `documents.get` + the ones below |
| Analytics | `analytics` | Avg overall score, avg job-application score, avg networking score, interview-to-application rate, messaging-to-contacted rate, weekly execution timeline chart | `analytics.getClientSummary`, `analytics.getClientAnalytics` |
| Activity | `activity` | Global feed of status transitions ("Contact X Saved → Contacted"), filter by type | `clients.getActivities` |
| Execution | `execution` | Overall execution %, weekly % split into Job Applications and Networking Contacts | (reuses analytics) |
| Resume | `resume` | PDF viewer of the uploaded resume + **"Additional Skills" bank** (≈126 free-text skills). It feeds the "Have these missing skills?" action | `clients.getResume`, `documents.getAll`, `documents.getUrl` |
| Roadmap | `roadmap` | Coach-created milestones ("Ask your coach to create a roadmap") | `clients.hasOrganizationMilestones` |
| Careers | `careers` | Target roles, each with an AI narrative, fit %, and five competency dimensions (Technical Expertise, Project Experience, Problem-Solving, Collaboration, Adaptability) | `clients.getSelectedTransitions` |
| **Jobs** | `jobs` | Discovery feed of auto-scanned jobs: score, salary, skill chips, age, source (Greenhouse, LinkedIn, careers page), location. Filters and sort | `jobs.getAnalyzedJobs`, `jobs.scanJobs` (POST) |
| **Jobs › Tracker** | `jobs&tracker=true` | Kanban with 7 columns (below). Card: logo, title, company, age, location-type badge, location, score ring, star rating, tag icon, note badges ("Introduced me to a professional"), `EXPIRED` flag | `jobs.getTrackedJobs` (×7, one per status), `tags.getApplicationsByEntityType`, `tags.getAllTagDefinitions` |
| Companies | `companies` | Table: company, tier, #jobs, #contacts, created, website + LinkedIn links | `companies.list` |
| **Networking** | `networking` | Contacts table: contact, company + title, compatibility (Strong/Moderate/…), status, created, actions (star, tag, LinkedIn). Filters: company, compatibility, time, tags, rating, status | `clients.getContacts` |
| **Networking › Tracker** | `networking&tracker=true` | Kanban with 7 contact statuses. Card: name, title, company, age, compatibility tier | `clients.getTrackedContacts` |
| Notes | `notes` | Free notes ("keep track of progress") | `clients.getNotes` |
| Documents | `documents` | Uploaded files (resume PDFs) | `documents.getAll` |
| Feedback | — | Product feedback to PathPeer | `util.getMyFeedback` |
| (global) | — | Reminders, release notes, session | `clients.getReminders`, `util.getNewReleaseNotes`, `clientSession.me` |

## 3. Enums (exact)

**Job tracker status:** `SAVED`, `APPLIED`, `NETWORKING`, `INTERVIEW`, `OFFER`, `REJECTED` (UI label "Not Selected"), `WITHDRAWN`.

**Contact status:** `Unsaved`, `Saved`, `Contacted`, `Messaging`, `Stuck`, `Completed`, `Closed`.

**Contact compatibility tier:** `Strong`, `Moderate` (plus a lower tier, presumably `Weak`).

**Qualification match state:** met (green), partial (amber), missing (red).

**Email metadata:** verification (`Verified`), type (`work`/`personal`), source ("resolved via third-party lookup").

## 4. Job-list query contract (`jobs.getAnalyzedJobs`)

```
{ clientId, text, positions[], locationTypes, locations, limit: 10, offset: 0,
  sortBy: "postedAt", postedTime, minScore, publisher, sponsorship, status,
  trackerStatuses, tagIds, minUserRating }
```
Maps to our filters: free text, position, location, location type (Remote/Hybrid/Onsite),
posted-within, min fit score, source/publisher, visa sponsorship, job status (open/expired),
tracker status, tags, min star rating.

## 5. Job detail drawer

Header: title, company, salary range, location, posted date, location-type badge, star rating
(0–5), flag toggle, "Add Tag".

| Tab | Content | Procedures |
|---|---|---|
| Details | Full JD rendered as sections (summary, responsibilities, qualifications…), apply link | `jobs.getJob`, `jobs.getCompanyById` |
| Analysis | **Compatibility score %** (ring), one-paragraph summary, Required qualifications and Preferred qualifications, each tri-state met/partial/missing, plus a "Have these missing skills?" action (adds them to the skill bank and rescores) | `jobs.getAnalyzedJob` |
| Networking | **AI Suggestions (beta)**: "AI Networking Assistant — N candidates found, looking for more…", each with name, title, a quoted search-result-style snippet, a LinkedIn link, and **+Add**. **LinkedIn Shortcuts**: Recruiters / Hiring Managers / Teammates (prebuilt LinkedIn people-search links). **Your Network**: contacts linked to this job | `clients.getContacts({companyId, analysisId})`, `jobs.getTeamForJob` |
| Updates | Per-job audit log (Saved, Saved → Interview, …) with actor avatar and relative time | `clients.getActivitiesByReference` |
| (Copilot) | Chat thread bound to the job analysis | `copilot.getMessages({referenceId: analysisId})` |

The suggestion snippets read like web-search results, so they probably come from a
`site:linkedin.com/in` search API rather than LinkedIn scraping. This is inferred, not confirmed.

## 6. Contact detail drawer

Header: avatar, name, status pill, title, company logo, star rating, "Add Tag".

| Tab | Content | Procedures |
|---|---|---|
| Analysis | **Positive indicators** (has photo, 500+ connections, recently active, involved in hiring, tenure, relevant domain…), **Negative indicators** (not a hiring manager, inactive, location mismatch…), **Commonalities** (2–3 paragraphs), **Intro Message** (generated, Copy button) | `clients.getContactById` |
| Jobs | Jobs at the contact's company, with fit % | `jobs.getCompanyJobs` |
| Email | "Contact's Email": address, **Verified** badge, **work** tag, copy button. Note: "Resolved via third-party lookup. Verified emails are confirmed deliverable. Work emails tend to be more reliable for outreach." Plus **Draft Email** (AI-generated outreach, "Email type" selector) | `clients.getContactEmail`, `clients.getContactEmailMessages` |
| Updates | Contact audit log (Added, Saved → Contacted) | `clients.getActivitiesByReference` |
| Copilot | Chat with prompts like "Help me with an intro message", "How to ask for a referral?" | `copilot.getMessages` |

## 7. Volume in the account (import checklist targets)

Jobs tracker: Saved 152 · Applied 78 · Networking 18 · Interviewing 1 · Offered 1 · Not Selected 36 · Withdrawn 20 (≈306).
Contacts (Summary widget): Saved 571 · Contacted 71 · Messaging 21 · Stuck 20 · Completed 37 (Closed not shown).
Resume skill bank: ≈126 skills. Documents: at least 1 resume PDF.

## 8. Verdict: keep, improve, drop

| Keep (core) | Improve | Drop / defer |
|---|---|---|
| Jobs kanban (7 statuses) + list with filters | **Message state is invisible** (Career OS RULE ZERO: cards don't show whether an invite was accepted or a thread exists). Add connection state, last outbound/inbound, and follow-up due dates | Coach/organization tenancy |
| Contacts table + networking kanban | Intro messages are generic (no verifiable hook). Gate drafts on a real hook, as `richard-outreach` does | Roadmap (coach-driven) |
| Job analysis (score + tri-state qualifications + skill bank) | Scores don't say which resume version they used. Tie each analysis to a resume version and mark it stale on change | Feedback, release notes |
| Contact analysis + email lookup + draft by type | Email "verified" has no provenance. Store provider, method, checked-at and confidence | Careers narratives (defer to a late phase) |
| Per-entity activity log + global feed | No outreach safety caps. Add a daily ledger (invites/DMs vs caps) from Career OS rules | |
| Companies roll-up, tags, notes, documents, reminders | Export/API lock-in: ship a REST API, an MCP server and full JSON export from day 1 | |
| Execution targets + analytics | Analytics computed from the activity log, so they can be reproduced | |
| Clipper extension | Page-aware side panel ("already saved: Applied, 87%") and JSON-LD first parsing | |
