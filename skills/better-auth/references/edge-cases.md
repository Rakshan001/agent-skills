# Edge cases and gotchas

Better Auth v1.7 docs, checked 2026-11-02. Each row: symptom → cause → fix → source.
*(inference)* marks reasoning from documented behaviour rather than a stated fact.

## Schema and data

| Symptom | Cause | Fix | Source |
|---|---|---|---|
| Generated schema collides with the app's own `account` / `session` / `user` table | Core model names are fixed defaults | Set `modelName` per model and pass an explicit adapter `schema` map whose keys equal those names; run `generate` after | /docs/concepts/database#custom-table-names, /docs/adapters/drizzle |
| `unable_to_create_user` on OAuth sign-up; insert fails on NOT NULL | App column Better Auth never fills; or a hook threw | Declare in `user.additionalFields` (`input: false`), fill in `databaseHooks.user.create.before` → `{ data }`; `npx auth@latest check schema` | /docs/concepts/database#database-hooks, /docs/reference/errors/unable_to_create_user |
| Hook created a side row but the user insert failed → orphan | Before hook runs before the insert; own writes not in Better Auth's transaction (undocumented — assume separate) | Idempotent creation + cleanup, or move to the after hook when no FK from `user` requires it | /docs/concepts/database#database-hooks |
| After hook ran although sign-up failed | Adapter without transactions | Make after-hook side effects idempotent | /docs/concepts/database#2-after-hook |
| `role`/flag set by a client at sign-up | `additionalFields.input` defaults to `true` | `input: false` on server-owned fields | /docs/concepts/typescript#the-input-property |
| `defaultValue` not present in DB rows written elsewhere | `defaultValue` is JS-layer only; DB column optional | Add a DB default too if other writers exist | /docs/concepts/database#extending-core-schema |
| uuid column rejects ids like `"Xyz..."` | Default id is a random base62 string | `advanced.database.generateId: "uuid"` (or a function) | /docs/concepts/database#id-generation |
| Numeric ids come back as strings | `"serial"` keeps ids typed as string | Expected; pass strings to endpoints | same |
| Imported users get new ids | Adapter generates ids | `ctx.adapter.create({ ..., forceAllowId: true })` | /docs/guides/clerk-migration-guide |
| Password users cannot sign in after import | Passwords live in `account`, not `user` | One `account` row: `providerId: "credential"`, `accountId: user.id`, `password: hash`; custom `password.verify` for non-scrypt hashes | /docs/guides/next-auth-migration-guide |
| `emailVerified` type error on import | Auth.js uses a timestamp; Better Auth a boolean | `emailVerified = old IS NOT NULL` | same |
| Dates shifted by the server's offset | String-mode columns without zone are read as UTC | Use `timestamp()` (default) or `timestamp({ mode: "string", withTimezone: true })` | /docs/adapters/drizzle#date-columns |
| `check schema` passes but prod errors | Drizzle check reads the schema object, not the DB | Confirm the migration was applied to each DB | /docs/concepts/cli#schema |
| `getMigrations` / `npx auth migrate` does nothing with Drizzle | Kysely-only | `generate` + `drizzle-kit` | /docs/concepts/database#programmatic-migrations |
| Joins enabled but queries fail | Drizzle relations missing / v1 vs v2 | Regenerate schema; pass relations; `@better-auth/drizzle-adapter/relations-v2` for Relations v2 | /docs/adapters/drizzle#joins |

## Sessions and Next.js

| Symptom | Cause | Fix | Source |
|---|---|---|---|
| Revoked/banned user still has access for a few minutes | `cookieCache` serves a signed copy until `maxAge` | Shorter `maxAge`, disable, or `query: { disableCookieCache: true }` on sensitive reads | /docs/concepts/session-management#cookie-cache |
| Custom session fields always recomputed | `customSession` output is not cached | Keep it cheap or move data elsewhere | same, caveats |
| Protected page reachable with a forged cookie | Middleware used `getSessionCookie` only | `auth.api.getSession` in the page/route/action | /docs/integrations/next |
| Middleware redirects logged-in users | Custom `cookiePrefix`/name not passed to `getSessionCookie` | `getSessionCookie(request, { cookieName, cookiePrefix })` | same |
| Server Action sign-in "works" but no cookie | Server Actions need Next's `cookies()` | `nextCookies()` last in `plugins` | same |
| Session not extended / cache stale on RSC-only pages | RSCs cannot set cookies | Refresh via a Route Handler / Server Action / client call | same |
| `authClient.getSession()` null in an RSC | Client can't read cookies server-side | `auth.api.getSession({ headers: await headers() })` | /docs/reference/faq |
| Read replica errors on `/get-session` | GET refreshes sessions by writing | `session.deferSessionRefresh: true` | /docs/concepts/session-management |
| Users logged out after deploy | Secret swapped in place, or `cookieCache.version` bumped | Rotate with versioned `secrets` | /docs/reference/options#secrets *(inference for in-place swap)* |
| Edge middleware can't call `auth.api` | Next ≤15.1 middleware is Edge-only | Cookie check, or Node runtime (15.2+ experimental, 16 `proxy.ts`) | /docs/integrations/next |
| Safari loses the session cross-domain | ITP blocks third-party cookies | Proxy `/api/auth/*` through the frontend domain or share a parent domain | /docs/concepts/cookies |

## OAuth and linking

| Symptom | Cause | Fix | Source |
|---|---|---|---|
| `redirect_uri_mismatch`, callback to localhost | `baseURL` / `BETTER_AUTH_URL` unset or wrong | Set it; register `/api/auth/callback/<id>` per environment | /docs/authentication/google |
| `state_mismatch` | Verification storage not shared across instances; state cookie dropped (Safari ITP, `.vercel.app` previews, proxy stripping `Cookie`); >10 min or clock skew; oAuthProxy without shared `secret` | Shared DB/Redis, stable custom domain, NTP, same proxy secret everywhere | /docs/reference/errors/state_mismatch, /docs/plugins/oauth-proxy |
| `state_invalid` | `BETTER_AUTH_SECRET` rotated mid-flow, or cookie mangled by a proxy | Rotate in quiet windows; versioned `secrets` | /docs/reference/errors/state_mismatch |
| `email_not_found` from GitHub | GitHub App without "Email addresses" permission, or private email | Grant permission; placeholder via `mapProfileToUser` | /docs/authentication/github |
| Attacker-controlled provider account joins a victim's user | Implicit linking with `trustedProviders` that don't verify emails | Only trust providers that verify; or `disableImplicitLinking: true` | /docs/concepts/users-accounts#forced-linking |
| `account_not_linked` | Implicit linking off/untrusted, or imported user has `emailVerified: false` | Explicit `linkSocial`, or correct `emailVerified` | /docs/reference/errors/account_not_linked |
| Unverified provider email still gets a session | `requireEmailVerification` is per provider and off | `socialProviders.<p>.requireEmailVerification: true` (only for trustworthy providers) | /docs/concepts/oauth#requireemailverification |
| Mapped `role` from the IdP ignored | `mapProfileToUser` respects `input: false` | Write verified claims with your own DB call | /docs/concepts/oauth#server-owned-fields-and-authorization-claims |
| No Google refresh token after first login | Google issues it on first consent only | `accessType: "offline"`, `prompt: "select_account consent"` | /docs/authentication/google |
| Magic-link/OTP sign-in wiped a user's password | 1.7: proving the mailbox clears unproven credentials on never-verified accounts | Tell users to reset the password | /docs/guides/1-7-upgrade-guide |
| 2FA bypassed via social / magic link / passkey | 2FA gates only email/username/phone sign-in | Custom hook to gate other methods | /docs/plugins/2fa |

## Deployment and abuse controls

| Symptom | Cause | Fix | Source |
|---|---|---|---|
| Rate limit never triggers locally | Disabled in development | `rateLimit.enabled: true` to test | /docs/concepts/rate-limit |
| Rate limit ineffective on serverless | Memory store per instance | `storage: "database"` or `"secondary-storage"` | same |
| Everyone shares one rate-limit bucket / limit trivially bypassed | Wrong IP header behind a proxy | `ipAddressHeaders` or `trustedProxies` | same |
| CORS / "invalid origin" from a second frontend | Origin not trusted | Add to `trustedOrigins` (not `disableCSRFCheck`) | /docs/reference/security |
| Preview deployments throw on unknown host | `allowedHosts` without a match | Add the pattern; `fallback` only deliberately | /docs/reference/options#baseurl |
| Forwarded host ignored behind a custom proxy | 1.7 ignores forwarded headers by default | `advanced.trustedProxyHeaders: true` if the proxy sets them | /docs/guides/1-7-upgrade-guide |
| Secondary storage errors after upgrade | 1.7 requires `increment` + `getAndDelete` | Implement both | same |
| Emails delayed or timing differences on serverless | Sends awaited / killed after response | `advanced.backgroundTasks.handler: waitUntil` | /docs/reference/options#backgroundtasks |
| Deleting a user leaves app data | Hard delete covers auth rows; app tables are yours | `user.deleteUser.beforeDelete/afterDelete`, or FKs with cascade | /docs/concepts/users-accounts#delete-user |
| `deleteUser` refused for OAuth user | Needs password, fresh session, or email verification flow | `sendDeleteAccountVerification` | same |
