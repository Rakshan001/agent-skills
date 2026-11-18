---
name: better-auth
description: >-
  Reference for Better Auth, the open-source TypeScript auth library (better-auth.com, v1.7 docs):
  real option names, schema, and the edge cases that break integrations. Use when adding auth to a
  TypeScript app with Better Auth, or when the user says "better auth", "better-auth",
  "betterAuth()", "auth.api.getSession", "toNextJsHandler", "nextCookies", "getSessionCookie",
  "createAuthClient", "drizzleAdapter", "npx auth generate", "databaseHooks", "cookieCache",
  "trustedOrigins", "BETTER_AUTH_SECRET", "BETTER_AUTH_URL", or names a Better Auth plugin (admin,
  organization, multi-session, two-factor, passkey, magic link, email OTP, username, api-key,
  bearer, jwt, device authorization, one tap, captcha, have-i-been-pwned, customSession, openAPI).
  Also for migrating to Better Auth from Auth.js / NextAuth, Clerk or custom auth (table
  remapping, keeping user ids, re-sign-in). NOT for work on other auth libraries that is not
  moving to Better Auth, and not a general security review.
---

# Better Auth — reference for integrating it without the usual breakage

Facts an agent otherwise guesses: option names, table shapes, which call sets cookies where, and
what each knob actually does. Every non-obvious claim in `references/` carries its docs URL.

> **Freshness.** Verified against the Better Auth **v1.7** docs on 2026-11-02
> (`https://better-auth.com/docs/llms.txt`; 1.8 is in beta). Anything the docs did not confirm is
> marked *unverified*. Match the docs version to the installed `better-auth` version before you
> trust a name: the docs index lists one per release line.

> **Wording rule.** Describe what a setting does ("rejects requests whose `Origin` is not in
> `trustedOrigins`"). Never tell the user a setup is "safe" or "secure"; that is a claim no config
> option can make.

## When to use / when not

- **Use:** new Better Auth setup, adding a plugin, wiring Next.js, debugging sessions/cookies/
  OAuth redirects, or moving an existing app onto Better Auth.
- **Not:** apps staying on another auth library; generic OWASP review; designing your own
  crypto. Better Auth has its own official agent skills (`/docs/ai-resources/skills`); this one
  is the edge-case and migration layer.

## Which reference to open

| You need | Open |
|---|---|
| Install, env vars, core tables, Drizzle adapter, renaming tables/columns, extra user columns, IDs, `databaseHooks`, `hooks`, errors, the client | `references/core-and-schema.md` |
| Session expiry, freshness, cookie cache, revocation, cookie names/prefix/domain, secondary storage | `references/sessions-and-cookies.md` |
| Next.js App Router: route handler, RSC/server actions, `nextCookies()`, proxy/middleware checks | `references/nextjs.md` |
| Google / GitHub setup, scopes, `mapProfileToUser`, account linking, `trustedProviders`, unverified emails | `references/social-and-linking.md` |
| Any plugin: what it adds to the schema, server + client setup, pitfalls | `references/plugins.md` |
| Moving from Auth.js / NextAuth (or Clerk/custom): table mapping, keeping ids, passwords, re-sign-in | `references/migration-from-authjs.md` |
| CSRF / `trustedOrigins`, base URL behind proxies, secrets + rotation, rate limiting, IP headers | `references/security.md` |
| The long list of gotchas with fixes | `references/edge-cases.md` |

## Decision flow

1. **Greenfield or migration?**
   - *Greenfield* → install, let `npx auth@latest generate` write the schema, apply it with your
     ORM, done.
   - *Migrating from Auth.js / NextAuth* → there is no session migration for JWT strategy
     (users sign in again); map the user/account tables, convert `emailVerified` from a
     timestamp to a boolean, decide the ID strategy. Read `migration-from-authjs.md` first.
   - *From Clerk / Auth0 / Supabase* → the docs ship a guide each (`/docs/guides/*-migration-guide`);
     the shared pattern is a script that writes `user` + `account` rows through
     `ctx.adapter.create({ ..., forceAllowId: true })` and plugs the old password hasher into
     `emailAndPassword.password.{hash,verify}`.
   - *Custom auth* → same as above: keep ids, create a `credential` account row per password user.
2. **Does the app already have tables called `user`, `session`, `account` or `verification`?**
   Remap Better Auth's models (`session: { modelName }`, adapter `schema` map) **before** running
   `generate`. Never let the generator emit a second `account` table over yours.
3. **Pick plugins by need** (each adds schema; re-run `generate` after adding one):

   | Need | Plugin |
   |---|---|
   | Roles, ban/unban, impersonation, list/revoke a user's sessions | `admin` |
   | Teams / workspaces / invitations / per-org roles | `organization` |
   | Several signed-in accounts in one browser | `multiSession` |
   | TOTP / OTP second factor, backup codes | `twoFactor` |
   | WebAuthn | `passkey` (package `@better-auth/passkey`) |
   | Passwordless email | `magicLink` or `emailOTP` |
   | Username sign-in | `username` |
   | Machine credentials for your API | `apiKey` (package `@better-auth/api-key`) |
   | Non-cookie clients (mobile, CLI) | `bearer`; JWTs for other services: `jwt` |
   | CLI / TV sign-in (RFC 8628) | `deviceAuthorization` (+ `bearer` to use the token) |
   | Extra data on every session response | `customSession` |
   | Bot friction / breached passwords | `captcha`, `haveIBeenPwned` |
   | OAuth on Vercel preview URLs | `oAuthProxy` |
   | Calling sign-in/up from Server Actions | `nextCookies()` — **last** in `plugins` |

## Setup checklist

1. `npm install better-auth` (+ the adapter package your docs version names).
2. Env: `BETTER_AUTH_SECRET` (≥32 chars, high entropy; `npx auth@latest secret` or
   `openssl rand -base64 32`) and `BETTER_AUTH_URL`. Note Better Auth also reads `AUTH_SECRET`.
3. `lib/auth.ts` exporting `auth = betterAuth({ database, emailAndPassword?, socialProviders?, plugins })`.
4. Remap models/fields if names clash; declare app-owned user columns in `user.additionalFields`.
5. `npx auth@latest generate` → review → apply with your ORM (`drizzle-kit generate` +
   `drizzle-kit migrate`). `npx auth@latest migrate` only works with the built-in Kysely adapter.
6. `npx auth@latest check schema --config ./src/lib/auth.ts`.
7. Mount `app/api/auth/[...all]/route.ts` with `toNextJsHandler(auth)`.
8. Client: `createAuthClient()` from `better-auth/react` (+ client plugins mirroring server plugins).
9. Server checks: `auth.api.getSession({ headers: await headers() })` in each page/route/action.
10. Production: `baseURL`/`BETTER_AUTH_URL` set, `trustedOrigins` listing every real frontend
    origin, rate-limit storage that survives restarts, no localhost origins.

## Top edge cases (rules — details and sources in `references/edge-cases.md`)

1. **Name clash:** if the app owns `account`/`session` tables, rename Better Auth's models
   (`account: { modelName: ... }`, `session: { modelName: ... }`) and map the Drizzle tables in
   `drizzleAdapter(db, { schema: {...} })`. Type inference still uses the original names.
2. **NOT NULL app columns on `user`:** Better Auth will not fill them. Declare them in
   `user.additionalFields` (`input: false`) and set them in `databaseHooks.user.create.before`
   by returning `{ data: { ...user, field } }`. Run `npx auth@latest check schema`.
3. **Social sign-up creates users with whatever the provider returns** — `name`, `email`,
   `emailVerified`, `image`. Your hook runs for OAuth sign-ups too.
4. **Implicit account linking** joins a same-email OAuth sign-in to an existing user when the
   provider says the email is verified or the provider is in `trustedProviders`. Set
   `account.accountLinking.disableImplicitLinking: true` to make linking explicit.
5. **Unverified emails:** `socialProviders.<p>.requireEmailVerification` is opt-in per provider
   and independent of `emailAndPassword.requireEmailVerification`.
6. **Cookie cache vs revocation:** with `session.cookieCache` on, a revoked or banned session can
   stay usable on other devices until `maxAge`; pass `query: { disableCookieCache: true }` for
   sensitive reads.
7. **Proxy/middleware sees cookies, not validity:** `getSessionCookie()` only checks the cookie
   exists. Validate with `auth.api.getSession` in the page, route or action.
8. **RSCs cannot set cookies:** the cookie cache is not refreshed from a Server Component; calls
   that set cookies from Server Actions need `nextCookies()` as the last plugin.
9. **Base URL:** set `baseURL` or `BETTER_AUTH_URL`. For preview deployments use
   `baseURL: { allowedHosts: [...] }`; unknown hosts throw unless `fallback` is set.
10. **Rate limiting is off in development and in-memory by default** — memory resets per
    serverless instance; use `storage: "database"` or `"secondary-storage"`. `auth.api` calls are
    never rate limited.
11. **IDs:** default is a random string. For uuid columns set
    `advanced.database.generateId: "uuid"`; to keep imported ids use `forceAllowId: true`.
12. **Banning revokes sessions** (admin plugin); combine with cookie-cache rule 6.
13. **Secrets:** rotate with `secrets: [{ version, value }]` / `BETTER_AUTH_SECRETS`, not by
    swapping `BETTER_AUTH_SECRET` in place.
14. **Never disable `advanced.disableCSRFCheck` / `disableOriginCheck` to fix a CORS error** —
    add the origin to `trustedOrigins`. `disableOriginCheck` also turns CSRF checks off.
15. **Date columns:** the generator emits `timestamp()`; string-mode columns are written in UTC,
    and values without a zone are read as UTC.

## Definition of done for an integration

- [ ] `npx auth@latest check schema` passes; the ORM migration is generated, reviewed and applied
      to every environment's database (including any clash remapping).
- [ ] Sign-up/sign-in works for every enabled method; a new user row has every NOT NULL app
      column filled (hook or `additionalFields`).
- [ ] Sign-out clears the session cookie and the session row/storage entry.
- [ ] Revocation: `revokeSession` / `revokeOtherSessions` (or admin `revokeUserSessions`) ends
      access within the documented window (immediately without cookie cache, ≤ `maxAge` with it).
- [ ] Ban (if admin plugin): banned user cannot sign in; existing sessions revoked.
- [ ] Rate limit returns 429 with `X-Retry-After` in a production-mode run, from a store that
      survives restarts.
- [ ] A cross-origin POST from an origin not in `trustedOrigins` is rejected; every real frontend
      origin is listed; no localhost origin in production.
- [ ] Protected pages/routes/actions call `auth.api.getSession` server-side, not only middleware.
- [ ] Migration (if any): existing users keep their ids; OAuth links map to
      `providerId`/`accountId`; password users have a `credential` account row; users were told
      they must sign in again.
- [ ] Tests cover sign-in, a protected route without a session (redirect/401), revocation and
      ban. The `testUtils()` plugin provides `test.login()` / `test.getAuthHeaders()` helpers;
      keep it out of the production config.
