# M0-08 · Auth: password, sessions, tokens, scopes, CSRF, rate limits

Status: open
Blocked by: M0-07
Wave: 6
Owner: agent
Model: sonnet

**Wave** 6 · **Model** sonnet · **Depends on** M0-07 · **Parallel-safe with** M0-09, M0-12, M0-13

**Goal.** SEC-01/02/03/04/17/19 implemented, plus the `auth` module routes.

**Read first.** `docs/spec/security.md` · `docs/spec/api-rest.md` §0, §1, §14 · data-model §2 (owner, sessions, api_tokens).

**Owns.** `apps/server/src/auth/**`, `apps/server/src/modules/auth/**`.

**Contract.**
- `auth/password.ts`: `hashPassword`, `verifyPassword` (argon2id via `@node-rs/argon2`). Minimum
  12 characters.
- `auth/owner.ts`: `createOwner(db, {displayName, password})` (fails if one exists),
  `setOwnerPassword(db, password)` (revokes all sessions). Used by the CLI's `jm init` and `jm reset-password`.
- `auth/tokens.ts`: `createToken(db, {name, scopes, expiresInDays?}) → {info, token}` where
  `token = 'jm_' + base64url(32 random bytes)` and only the sha256 is stored. Also `verifyToken`,
  `revokeToken`, `listTokens`, and `last_used_at` updates (at most once a minute).
- `auth/sessions.ts`: create (cookie secret, 14-day sliding), verify, revoke, revokeAll.
- `auth/middleware.ts`: resolves cookie **or** bearer, sets `ctx.scopes`/`ctx.actor`, and enforces
  the route's declared scope (403/401). Cookie sessions get all scopes. The `X-JM-Actor` header is
  accepted only from the allowed enum.
- `auth/rate-limit.ts`: in-memory token buckets per api-rest §14, plus the login lockout.
- CSRF: Hono `csrf()` for cookie-authenticated non-GET requests.
- `modules/auth/routes.ts`: `POST /auth/login`, `POST /auth/logout`, `GET /auth/me`, `GET/POST
  /tokens`, `DELETE /tokens/:id`, each writing activities per SEC-19.

**Acceptance.** Tests:
- The stored hash starts with `$argon2id$`.
- Wrong password → 401. The sixth attempt within a minute → 429 with `Retry-After`.
- Login sets an HttpOnly, SameSite=Lax cookie.
- A cookie POST with a foreign `Origin` → 403.
- A token with `read` scope can GET and gets 403 on POST.
- A revoked token → 401. After logout, the old cookie → 401.
- The token plaintext is returned exactly once (`GET /tokens` never contains it).
- Activities exist for login and for token create/revoke.

**Don'ts.** Never log passwords or tokens. No JWTs (server-side revocable sessions only). No
unauthenticated owner-creation route.

## Comments
