# Security-relevant configuration: what each setting does

Better Auth v1.7 docs, checked 2026-11-02. Main source: `https://better-auth.com/docs/reference/security`.
Describe these as behaviours ("rejects", "limits", "hashes"). None of them makes an app "safe".

## CSRF and `trustedOrigins`

What Better Auth does by default (security page):
1. Prefers non-simple requests (JSON `Content-Type` / custom header) for state changes; email
   sign-in/up also accept form posts and rely on the checks below.
2. Validates the `Origin` header against `baseURL` + `trustedOrigins`; rejects others.
3. `SameSite=Lax` session cookies (override via `defaultCookieAttributes`).
4. Fetch Metadata (`Sec-Fetch-*`) blocks cross-site navigations to sign-in/up without cookies
   (login CSRF). Requests with no Origin/Referer and no Fetch Metadata (servers, native apps) pass;
   non-browser clients that *do* send `Origin` must use a trusted one.
5. GET mutations (OAuth callbacks) are protected by `state`/`nonce`; state + PKCE stored in the
   `verification` table (or an encrypted cookie with `account.storeStateStrategy: "cookie"`).

```ts
trustedOrigins: ["https://app.example.com", "https://*.example.com", "myapp://", "chrome-extension://ID"],
// or: trustedOrigins: async (request) => request ? [...] : ["https://app.example.com"]
```

- `request` is `undefined` at init and for `auth.api` calls — always return a default list.
  The function runs on every request. — /docs/reference/options#trustedorigins
- Patterns: `?` one char, `*` no `/`, `**` anything; protocol-less `*.example.com` accepts any
  protocol. Custom schemes match host exactly when a host is given.
- Relative callback URLs must start with a single `/`; `//`, backslashes and encoded separators
  are rejected. — /docs/reference/security#trusted-origins
- "Do not leave the localhost origin in a trusted origins list of a production auth instance."
- `advanced.disableCSRFCheck: true` turns off origin + Fetch Metadata checks.
  `advanced.disableOriginCheck: true` turns off callback/redirect URL validation **and** CSRF
  checks (backward compatibility). Fix CORS/origin errors by listing the origin instead.

## Base URL behind proxies and on preview deployments

- Resolution order (security page): static `baseURL` → env (`BETTER_AUTH_URL`,
  `NEXT_PUBLIC_BETTER_AUTH_URL`, ...) → `X-Forwarded-Host`/`-Proto` only when
  `advanced.trustedProxyHeaders: true` → request URL.
- Dynamic form: `baseURL: { allowedHosts: ["myapp.com", "*.vercel.app"], protocol: "https",
  fallback?: "https://myapp.com" }`. Hosts are validated against the allowlist and added to
  `trustedOrigins`; unknown hosts **throw** unless `fallback` is set (fallback "can hide
  deployment or proxy misconfiguration"). — /docs/reference/options#baseurl
- Since 1.7, forwarded headers are ignored with `allowedHosts` unless `trustedProxyHeaders: true`;
  nginx/Vercel/Cloudflare/Netlify rewrite `Host` and need no change. Direct `auth.api` calls must
  resolve to an allowed host or use `fallback`. — /docs/guides/1-7-upgrade-guide#behind-a-proxy
  (The dynamic-base-url guide still says `x-forwarded-host` is read first; the options page and
  upgrade guide say otherwise — follow the upgrade guide.)
- Wrong base URL symptoms: OAuth `redirect_uri_mismatch`, callbacks to `localhost`, cookies set
  for the wrong host, `state_mismatch`.
- Vercel preview OAuth: either allow `*.vercel.app` (each preview URL must also be a registered
  redirect URI at the provider) or use `oAuthProxy` (register only the production callback).

## Secrets

- `BETTER_AUTH_SECRET` (≥32 chars) signs cookies and encrypts tokens; required in production.
- Rotation: `secrets: [{ version: 2, value: "new" }, { version: 1, value: "old" }]` or
  `BETTER_AUTH_SECRETS=2:new,1:old`. First entry encrypts; the rest only decrypt; legacy data falls
  back to `secret`. "Ordinary key rotation requires no database migration or downtime."
  — /docs/reference/options#secrets, /docs/reference/security#secret-rotation
- Swapping `BETTER_AUTH_SECRET` in place invalidates everything signed/encrypted with it (sessions,
  encrypted tokens) *(inference — the reason versioned secrets exist)*.
- Upgrades that change key derivation break in-flight OAuth/SAML state: upgrade all nodes together
  and expect users mid-sign-in to restart.

## Rate limiting

```ts
rateLimit: {
  enabled: true,                // default: on in production, off in development
  window: 10, max: 100,         // defaults (seconds / requests)
  storage: "database",          // "memory" (default) | "database" | "secondary-storage"
  customRules: { "/sign-in/email": { window: 10, max: 3 }, "/get-session": false },
}
```

- Built-in stricter rules: `/sign-in/email` 3 per 10s; `/two-factor/*` 3 per 10s.
- "Server-side requests made using `auth.api` aren't affected by rate limiting."
- Memory storage "may not be suitable ... particularly in serverless" — each instance has its own
  counters and loses them on cold start. `storage: "database"` needs the `rateLimit` table (id, key,
  count, lastRequest) — generate + migrate. If `secondaryStorage` is configured it is used.
- Custom store: `customStorage: { consume: async (key, rule) => ({ allowed, retryAfter }) }` —
  must check-and-increment atomically (1.7 removed `get`/`set`).
- Over-limit responses are 429 with `X-Retry-After` (seconds). — /docs/concepts/rate-limit

## Client IP

- Default header `x-forwarded-for`; comma-separated chains are **not** trusted by default.
- Behind a proxy set one trusted header — `advanced.ipAddress.ipAddressHeaders: ["x-real-ip"]` or
  `["cf-connecting-ip"]` — or list proxies in `advanced.ipAddress.trustedProxies` (walks the chain
  right to left). Make sure clients cannot set that header directly.
- `disableIpTracking` stops recording IPs. IPv6 is normalised; dev falls back to `127.0.0.1`.
  — /docs/concepts/rate-limit#connecting-ip-address, /docs/reference/security#ip-address-headers

## Passwords and email enumeration

- Default hash: scrypt; override with `emailAndPassword.password.{hash, verify}`.
- Defaults: `minPasswordLength` 8, `maxPasswordLength` 128, `requireEmailVerification` off,
  `revokeSessionsOnPasswordReset` false, reset token 3600s.
- Sign-up returns the same 200 for existing emails **only** when `requireEmailVerification: true`
  or `autoSignIn: false`; otherwise it returns 422 for an existing email. Plugins that add user
  columns then need `customSyntheticUser`. — /docs/authentication/email-password#email-enumeration-protection
- Account deletion: `user.deleteUser.enabled` (off by default); requires password, a fresh session
  (`freshAge`), or the `sendDeleteAccountVerification` email flow; `beforeDelete` can throw
  `APIError` to stop it. — /docs/concepts/users-accounts#delete-user

## Other switches worth knowing

- `account.encryptOAuthTokens` (default `false`) — provider tokens stored plaintext otherwise.
- Outbound OAuth/OIDC requests refuse HTTP redirects (SSRF hardening). — /docs/reference/security
- `telemetry.enabled` defaults to `false`. — /docs/reference/options#telemetry
- Report vulnerabilities via GitHub Security Advisories on `better-auth/better-auth`.
