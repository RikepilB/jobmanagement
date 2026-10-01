# M0-12 · Web skeleton

Status: open
Blocked by: M0-07, M0-08, M0-09
Wave: 7
Owner: agent
Model: sonnet

**Wave** 7 · **Model** sonnet · **Depends on** M0-07, M0-08 (login route), M0-09 · **Parallel-safe with** M0-10, M0-11

**Goal.** A Vite React SPA with login, layout, a router with **all M1 pages pre-stubbed**, and the
API hooks layer. It builds into `apps/web/dist` for the server to serve.

**Read first.** SPEC §4 REQ-UI-01 · CONVENTIONS §7 · `docs/spec/api-rest.md` §1.

**Owns.** `apps/web/**`.

**Contract.**
- Routes (react-router 7, `createBrowserRouter`): `/login`, `/` (Today), `/jobs/board`, `/jobs`,
  `/jobs/:id`, `/contacts/board`, `/contacts`, `/contacts/:id`, `/companies`, `/companies/:id`,
  `/activity`, `/search`, `/settings`. Each page is `src/pages/<name>/index.tsx` exporting a stub
  `<h1>`. The route table is **written once here**.
- `src/lib/api.ts`: a `createClient({ baseUrl: '/', fetch: window.fetch })` singleton (cookie auth)
  plus a QueryClient. On 401, redirect to `/login`.
- `src/components/`: `Layout` (nav links to every page), `Table`, `Empty`, `ErrorBox`, `Loading`,
  `StatusBadge`. Plain Tailwind.
- Login page posts `auth.login` and redirects to `/`.
- `vite.config.ts`: dev proxy `/api` and `/mcp` → `http://127.0.0.1:4747`.

**Acceptance.** `pnpm --filter @jm/web build` succeeds · the gzip size of the main chunk is reported
and is < 300 KB · a unit test renders `Layout` (react-dom/server `renderToString`, so no jsdom
needed) and asserts that every nav link exists · a grep gate shows no `localStorage`.

**Don'ts.** No component library, no CSS framework besides Tailwind, no auth token storage.

## Comments
