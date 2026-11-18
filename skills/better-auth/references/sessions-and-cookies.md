# Sessions and cookies

Better Auth v1.7 docs, checked 2026-11-02. Base `https://better-auth.com`.

## Model

- Cookie-based sessions. The `session_token` cookie is "an opaque, secret-signed session
  identifier"; the server looks the session up on each request. — /docs/concepts/session-management,
  /docs/concepts/cookies
- With `session.cookieCache` on, a second `session_data` cookie carries a short-lived signed copy
  of the session so most reads skip the database. Since 1.7 the cached session "is now tied to the
  `session_token` cookie". — /docs/concepts/session-management#cookie-cache, /docs/guides/1-7-upgrade-guide
- Without any database configured, Better Auth runs **stateless** (session only in the cookie);
  most plugins need a database. — /docs/concepts/session-management#stateless-session-management

## Lifetime options (`session.*`)

| Option | Default | Effect |
|---|---|---|
| `expiresIn` | `604800` (7 days) | Session lifetime, seconds. |
| `updateAge` | `86400` (1 day) | When a session is used and this age is reached, `expiresAt` is pushed to now + `expiresIn` (sliding). |
| `disableSessionRefresh` | `false` | Never slide. |
| `deferSessionRefresh` | `false` | `GET /get-session` becomes read-only and returns `needsRefresh: true`; the client POSTs to refresh. For read-replica setups — by default GET `/get-session` **writes**. |
| `freshAge` | `86400` | A session is "fresh" if `createdAt` is within this; some endpoints (e.g. delete user without password, 2FA `viewBackupCodes`) require fresh. `0` disables. |
| `cookieCache` | off | `{ enabled, maxAge, strategy: "compact" \| "jwt" \| "jwe", refreshCache?, version? }` |
| `storeSessionInDatabase` / `preserveSessionInDatabase` | `false` | With `secondaryStorage`: also write to DB / keep revoked rows (marked ended) in DB. |
| `modelName`, `fields`, `additionalFields` | — | Rename / extend the table. |

— /docs/concepts/session-management, /docs/reference/options#session

## Cookie cache — the staleness trade-off

```ts
session: { cookieCache: { enabled: true, maxAge: 5 * 60, strategy: "compact" } }
```

Docs, verbatim: "When `cookieCache` is enabled, revoked sessions may remain active on other devices
until the cookie cache expires (`maxAge`)." Fixes: disable it, shorten `maxAge` (e.g. 60s), or
force a DB read for sensitive operations:

```ts
await auth.api.getSession({ query: { disableCookieCache: true }, headers: await headers() });
await authClient.getSession({ query: { disableCookieCache: true } });
```

— /docs/concepts/session-management#cookie-cache

- Strategies: `compact` (default; base64url + HMAC-SHA256, smallest), `jwt` (HS256, readable,
  tamper-evident; asymmetric via the JWT plugin's `sessionCookieCache: true`), `jwe` (encrypted).
- Changing `cookieCache.version` and redeploying invalidates all cached sessions.
- Fields added by `customSession` are **not** cached — your function runs on every fetch.
- In Next.js, Server Components cannot set cookies, so the cache is not refreshed there (see
  `nextjs.md`).

## Revocation API

| Client | Server | Effect |
|---|---|---|
| `authClient.listSessions()` | `auth.api.listSessions({ headers })` | Active sessions of the current user |
| `authClient.revokeSession({ token })` | `auth.api.revokeSession` | End one session by token |
| `authClient.revokeOtherSessions()` | `auth.api.revokeOtherSessions` | All but the current |
| `authClient.revokeSessions()` | `auth.api.revokeSessions` | All, including current |
| `authClient.changePassword({ newPassword, currentPassword, revokeOtherSessions: true })` | | Revoke on password change |
| `emailAndPassword.revokeSessionsOnPasswordReset: true` | | Revoke on reset (default `false`) |
| admin: `authClient.admin.revokeUserSession({ sessionToken })` / `revokeUserSessions({ userId })` | `auth.api.revokeUserSessions` | Another user's sessions |

— /docs/concepts/session-management, /docs/reference/options#emailandpassword, /docs/plugins/admin.
*(Server names `listSessions` / `revokeSession(s)` / `revokeOtherSessions` follow the docs' naming
pattern but are only shown client-side on that page — unverified.)*

- `authClient.updateSession({ ... })` / `auth.api.updateSession({ body, headers })` updates only
  custom `additionalFields`, never core fields.
- With secondary storage, revocation removes the entry after the DB commit; a storage failure is
  logged and those sessions "can stay valid until they expire".

## Secondary storage (Redis/KV)

```ts
import { Redis } from "ioredis";
import { redisStorage } from "@better-auth/redis-storage";
betterAuth({ secondaryStorage: redisStorage({ client: new Redis(process.env.REDIS_URL!), keyPrefix: "better-auth:" }) });
```

- Interface: `get`, `getAndDelete`, `increment(key, ttl)`, `set(key, value, ttl?)`, `delete` —
  `increment` and `getAndDelete` became **required** in 1.7. — /docs/concepts/database#secondary-storage,
  /docs/guides/1-7-upgrade-guide
- When provided, sessions go to secondary storage **instead of** the DB unless
  `storeSessionInDatabase: true`; verification records and (with `rateLimit.storage:
  "secondary-storage"`) rate-limit counters go there too. — /docs/concepts/session-management,
  /docs/reference/options#verification

## Cookies

| Setting | Detail |
|---|---|
| Name format | `${prefix}.${cookie_name}`, default prefix `better-auth` → `better-auth.session_token`. Change with `advanced.cookiePrefix`. |
| Default cookies | `session_token`, `session_data` (cookie cache), `dont_remember` (`rememberMe: false`); plugins add more (2FA: `two_factor`). |
| Per-cookie override | `advanced.cookies.session_token = { name, attributes }` |
| All cookies | `advanced.defaultCookieAttributes` |
| Secure | `httpOnly` + `secure` in production; `advanced.useSecureCookies: true` forces `Secure` everywhere. With dynamic `baseURL`, `protocol: "https"` → secure. |
| SameSite | `Lax` by default (security page). |
| Subdomains | `advanced.crossSubDomainCookies: { enabled: true, domain: "example.com", additionalCookies? }` — list the other subdomains in `trustedOrigins`; keep `domain` as specific as possible. |

— /docs/concepts/cookies, /docs/reference/security#cookies, /docs/reference/options#advanced

- Whether the production cookie name gains a `__Secure-` prefix: *unverified in the v1.7 docs* —
  read the actual `Set-Cookie` header once and use that name in `getSessionCookie` / tests.
- Safari ITP blocks cookies when the auth API is on another site than the frontend. Proxy the auth
  routes through the frontend's domain (Vercel `rewrites`, Netlify redirects) or share a parent
  domain with `crossSubDomainCookies`. — /docs/concepts/cookies#safari-itp-and-cross-domain-setups
- Hook cookie helpers: `ctx.setCookie`, `ctx.setSignedCookie(name, value, ctx.context.secret, opts)`;
  read the session cookie name from `ctx.context.authCookies.sessionToken.name`. — /docs/concepts/hooks
