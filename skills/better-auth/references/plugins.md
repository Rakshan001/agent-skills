# Plugins: schema, setup, pitfalls

Better Auth v1.7 docs, checked 2026-11-02. URL pattern: `https://better-auth.com/docs/plugins/<slug>`.
Schema columns below were read from the rendered docs tables.

General rules:
- Server plugins import from `better-auth/plugins` unless noted; client plugins from
  `better-auth/client/plugins`. Add the client plugin whenever the server one has one, or the
  `authClient.<namespace>` methods are missing.
- A plugin that adds schema needs `npx auth@latest generate` + your ORM migration afterwards.
- Server form of any method: `auth.api.<name>({ body | query, headers: await headers() })`.
  Server names sometimes differ from client names — they are listed where they do.

## admin — `/docs/plugins/admin`

`admin()` / `adminClient()`. First admin: `npx auth@latest create-admin --email ... --name ... --role admin`.

- **Schema:** `user.role` string, `user.banned` boolean, `user.banReason` string,
  `user.banExpires` date; `session.impersonatedBy` string.
- **Options:** `defaultRole` (`"user"`), `adminRoles` (`["admin"]`), `adminUserIds` (`[]`, full
  access), `impersonationSessionDuration` (seconds, 1 hour), `defaultBanReason`,
  `defaultBanExpiresIn` (undefined = never), `bannedUserMessage` (string or `(user) => string`),
  `ac` + `roles` (custom access control, also passed to `adminClient`).
- **Methods** (client `authClient.admin.*`): `createUser({ email, password, name, role?, data? })`,
  `listUsers({ query: { searchValue, searchField, searchOperator, limit, offset, sortBy,
  sortDirection, filterField, filterValue, filterOperator } })` → `{ users, total, limit, offset }`,
  `getUser`, `setRole({ userId, role })`, `setUserPassword`, `updateUser({ userId, data })`
  (server: **`adminUpdateUser`**), `banUser({ userId, banReason?, banExpiresIn? })` (seconds),
  `unbanUser`, `listUserSessions({ userId })`, `revokeUserSession({ sessionToken })`,
  `revokeUserSessions({ userId })`, `impersonateUser({ userId })`, `stopImpersonating()`,
  `removeUser` (hard delete), `hasPermission` (server: **`userHasPermission`**),
  `checkRolePermission` (client-only, sync, does not check the signed-in user).
- **Banning:** "Bans a user, preventing them from signing in and revokes all of their existing
  sessions." Sign-in returns `bannedUserMessage`. With cookie cache on, other devices may keep a
  cached session until `maxAge` (see sessions file). What a banned user sees on *social* sign-in is
  not spelled out — *unverified*.
- **Impersonation:** lasts until browser close or `impersonationSessionDuration`; admins cannot
  impersonate admins unless granted `impersonate-admins`.
- **Roles:** built-in `admin`, `user`; multiple roles stored comma-separated. Custom AC:
  `createAccessControl` from `better-auth/plugins/access`, defaults from
  `better-auth/plugins/admin/access` (`defaultStatements`, `adminAc`). Custom roles **replace** the
  built-in permissions for that role name — spread `adminAc.statements` to keep them.
- **Pitfall:** with email-enumeration protection on (`requireEmailVerification` or
  `autoSignIn: false`) add the admin fields to `emailAndPassword.customSyntheticUser`.

## organization — `/docs/plugins/organization`

`organization()` / `organizationClient()`.

- **Schema:** `organization` (id, name, slug, logo, metadata, createdAt), `member` (id, userId,
  organizationId, role, createdAt), `invitation` (id, email, inviterId, organizationId, role,
  status, createdAt, expiresAt; + teamId with teams), `session.activeOrganizationId`,
  `session.activeTeamId`; teams add `team` (id, name, memberCount, organizationId, createdAt,
  updatedAt) and `teamMember` (id, teamId, userId, membershipKey, createdAt); dynamic AC adds
  `organizationRole`. Rename via `schema: { organization: { modelName, fields, additionalFields } }`.
- **Options:** `allowUserToCreateOrganization` (true), `organizationLimit`, `creatorRole`
  (`"owner"`), `membershipLimit` (100), `sendInvitationEmail` (**required for invitations**),
  `invitationExpiresIn` (48h), `invitationLimit` (100), `cancelPendingInvitationsOnReInvite`,
  `requireEmailVerificationOnInvitation`, `disableOrganizationDeletion`, `organizationHooks`,
  `teams: { enabled, maximumTeams, maximumMembersPerTeam }` (also pass `teams` to the client),
  `dynamicAccessControl: { enabled }` (needs `ac`; also on the client).
- **Methods** (`authClient.organization.*`): `create`, `checkSlug`, `list` / `useListOrganizations()`,
  `setActive({ organizationId | null })`, `useActiveOrganization()`, `getFullOrganization`,
  `update`, `delete`, `inviteMember` (server **`createInvitation`**), `acceptInvitation`,
  `rejectInvitation`, `cancelInvitation`, `listMembers`, `removeMember`, `updateMemberRole`,
  `getActiveMember`, `leave`, `hasPermission`; server-only `auth.api.addMember` (no invitation).
- **Pitfalls:** active organization is `null` after sign-in — set it in
  `databaseHooks.session.create.before` by returning `activeOrganizationId`. Call
  `acceptInvitation` only after the user is signed in. With predictable ids (`serial`, custom)
  accepting requires a verified matching email. Creating an org for another user: call server-side
  **without** session headers (with headers, `userId` is ignored). `checkRolePermission` ignores
  dynamic roles.

## multiSession — `/docs/plugins/multi-session`

`multiSession({ maximumSessions: 5 })` / `multiSessionClient()`. No schema (extra cookie per
signed-in account). Methods: `listDeviceSessions()`, `setActive({ sessionToken })` (server
**`setActiveSession`**), `revoke({ sessionToken })` (server **`revokeDeviceSession`**). `signOut`
revokes all of them. `customSession` does not change `listDeviceSessions` unless its third argument
is `{ shouldMutateListDeviceSessionsEndpoint: true }`.

## twoFactor — `/docs/plugins/2fa`

`twoFactor()` (issuer = `appName`) / `twoFactorClient({ onTwoFactorRedirect })`.

- **Schema:** `user.twoFactorEnabled`; table `twoFactor` (id, userId, secret, backupCodes,
  verified, failedVerificationCount, lockedUntil). *(The database concept page lists different
  user columns — `twoFactorSecret`, `twoFactorBackupCodes`; trust the plugin page.)*
- **Flow:** `twoFactor.enable({ password, method?: "totp" | "otp" })`; TOTP returns `totpURI` +
  `backupCodes`, and `twoFactorEnabled` stays false until a code is verified; OTP needs
  `otpOptions.sendOTP`. Sign-in answers `twoFactorRedirect: true` + `twoFactorMethods`; then
  `verifyTotp` / `verifyOtp` / `verifyBackupCode` (`trustDevice` = 30 days).
- **Pitfalls:** enabling requires a password — OAuth/passkey/magic-link users need
  `allowPasswordless: true`. Only `/sign-in/email`, `/sign-in/username`, `/sign-in/phone-number`
  are gated; social, magic link, email OTP and passkey sign-ins are **not** gated by default. In
  after hooks `ctx.context.newSession` is `null` during the challenge. Lockout is on by default
  (429 `ACCOUNT_TEMPORARILY_LOCKED`).

## passkey — `/docs/plugins/passkey`

Separate package: `npm install @better-auth/passkey`; `passkey()` from `@better-auth/passkey`,
`passkeyClient()` from `@better-auth/passkey/client`.

- **Schema:** table `passkey` (id, name, publicKey, userId, credentialID, counter, deviceType,
  backedUp, transports, createdAt, aaguid).
- **Options:** `rpID` (registrable domain; `localhost` in dev), `rpName`, `origin` (no trailing
  `/`), `authenticatorSelection`, `registration.requireSession` (true).
- **Methods:** `authClient.passkey.addPasskey()` (user must be signed in),
  `authClient.signIn.passkey({ autoFill })`, `listUserPasskeys` (server `listPasskeys`),
  `deletePasskey`, `updatePasskey`. Conditional UI needs `autocomplete="... webauthn"`.

## magicLink / emailOTP — `/docs/plugins/magic-link`, `/docs/plugins/email-otp`

- `magicLink({ sendMagicLink: async ({ email, token, url, metadata }, ctx) => {} })` /
  `magicLinkClient()`; `authClient.signIn.magicLink({ email, callbackURL })`. Options `expiresIn`
  (300s), `disableSignUp`, `storeToken`. Token is single-use.
- `emailOTP({ sendVerificationOTP: async ({ email, otp, type }) => {} })` / `emailOTPClient()`;
  types `"sign-in" | "email-verification" | "forget-password"`; `otpLength` 6, `expiresIn` 300,
  `allowedAttempts` 3, `overrideDefaultEmailVerification`. `authClient.emailOtp.sendVerificationOtp`,
  `authClient.signIn.emailOtp({ email, otp })`. Docs: do not await the send (timing); use
  `waitUntil` on serverless.
- Both sign users **up** unless `disableSignUp: true`, and both use the `verification` table (no
  new schema). Since 1.7, signing in this way to an account whose email was never verified
  **removes its password and other linked accounts and revokes its sessions**.

## username — `/docs/plugins/username`

`username()` / `usernameClient()`; needs `emailAndPassword.enabled`. Schema: `user.username`,
`user.displayUsername`. Options: `minUsernameLength` 3, `maxUsernameLength` 30,
`usernameValidator`, normalization lowercases by default. `authClient.signIn.username({ username,
password })`, `isUsernameAvailable`. To stop username enumeration add
`disabledPaths: ["/is-username-available"]`.

## apiKey — `/docs/plugins/api-key`

Separate package: `@better-auth/api-key` (`apiKey()`), client `@better-auth/api-key/client`
(`apiKeyClient()`).

- **Schema:** table `apikey` (id, configId, name, start, prefix, key, referenceId, refillInterval,
  refillAmount, lastRefillAt, enabled, rateLimitEnabled, rateLimitTimeWindow, rateLimitMax,
  requestCount, remaining, lastRequest, expiresAt, createdAt, updatedAt, permissions, metadata).
  `userId` was replaced by `referenceId`.
- **Use:** `authClient.apiKey.create({ name, ... })` returns the plaintext `key` once (get/list
  never return it; keys are hashed by default). Server verify:
  `auth.api.verifyApiKey({ body: { key, permissions? } })` → `{ valid, error, key }`. Header
  default `x-api-key` (`apiKeyHeaders`).
- **Pitfalls:** `enableSessionForAPIKeys` lets `getSession` accept a key — docs: "A leaked api key
  can be used to impersonate a user." Calling `verifyApiKey` then `getSession` with the same key
  counts twice against its rate limit. `get`/`update`/`delete` must pass the key's `configId`.

## bearer — `/docs/plugins/bearer`

`bearer()`; no client plugin — client sends `fetchOptions: { auth: { type: "Bearer", token: () =>
... } }`. Token comes back in the `set-auth-token` header after sign-in; `auth.api.getSession({
headers })` then works with `Authorization: Bearer`. Docs: "intended only for APIs that don't
support cookies ... Improper implementation could easily lead to security vulnerabilities."

## jwt — `/docs/plugins/jwt`

`jwt()` / `jwtClient()`. Schema: `jwks` (id, publicKey, privateKey, alg, crv, createdAt,
expiresAt). Not a session replacement. Token: `authClient.token()` or `GET /api/auth/token`, or the
`set-auth-jwt` header on get-session. Verify with `/api/auth/jwks` (e.g. `jose`
`createRemoteJWKSet`). Defaults: EdDSA, `jwt.expirationTime` 15 min, issuer/audience = base URL,
payload = whole user (`jwt.definePayload` to trim). With OIDC/MCP provider plugins disable
`/token` (`disabledPaths`).

## deviceAuthorization (RFC 8628) — `/docs/plugins/device-authorization`

`deviceAuthorization({ verificationUri: "/device" })` / `deviceAuthorizationClient()`.

- **Schema:** `deviceCode` (id, deviceCode, userCode, userId, clientId, scope, status, expiresAt,
  lastPolledAt, pollingInterval); `deviceCode`/`userCode` unique.
- **Options:** `expiresIn` "30m", `interval` "5s", `userCodeLength` 8, `deviceCodeLength` 40,
  `validateClient(clientId)`, `onDeviceAuthRequest`.
- **Flow:** device `authClient.device.code({ client_id })` → user opens `verificationUri` while
  **signed in**, `authClient.device.approve({ userCode })` → device polls
  `authClient.device.token({ grant_type: "urn:ietf:params:oauth:grant-type:device_code",
  device_code, client_id })`, handling `authorization_pending` and `slow_down`.
- **Pitfalls:** the returned `access_token` is a Better Auth **session token**, not an OAuth access
  token; use it with the `bearer` plugin. The approval page is a security boundary — show client +
  scopes and require an explicit approve. Real OAuth tokens need `@better-auth/oauth-provider`
  (`oauthDeviceAuthorization`) + `jwt()`.

## oneTap — `/docs/plugins/one-tap`

`oneTap()` (client id from `socialProviders.google.clientId` or its own `clientId`) /
`oneTapClient({ clientId })`; `await authClient.oneTap()`. Option spelled `disableSignUp`. Set
Authorized JavaScript origins in Google Cloud. Uses FedCM (cannot be disabled).

## lastLoginMethod — `/docs/plugins/last-login-method`

`lastLoginMethod()` / `lastLoginMethodClient()`; cookie-only by default (`httpOnly: false`,
non-essential cookie). `storeInDatabase: true` adds `user.lastLoginMethod`.
`authClient.getLastUsedLoginMethod()`, `isLastUsedLoginMethod("google")`. With
`crossSubDomainCookies` set `domain` on the client too.

## captcha — `/docs/plugins/captcha`

`captcha({ provider: "cloudflare-turnstile" | "google-recaptcha" | "hcaptcha" | "captchafox" |
"vercel-botid", secretKey })`; no client plugin. Client sends the token in header
`x-captcha-response`. Default `endpoints`: `/sign-up/email`, `/sign-in/email`,
`/request-password-reset` — setting `endpoints` **replaces** them, so list magic-link/OTP routes too.

## haveIBeenPwned — `/docs/plugins/have-i-been-pwned`

`haveIBeenPwned({ customPasswordCompromisedMessage? })`; on sign-up / password change a breached
password fails with `PASSWORD_COMPROMISED`. Sends only a 5-char SHA-1 prefix. No schema.

## customSession — `/docs/concepts/session-management#customizing-session-response`

```ts
import { customSession } from "better-auth/plugins";
plugins: [customSession(async ({ user, session }, ctx) => ({ user: { ...user, plan: "pro" }, session }))]
// client: customSessionClient<typeof auth>() from "better-auth/client/plugins"
```

Caveats: the callback's `session` does not infer plugin fields (pass options as the 2nd arg);
inference fails when client and server are separate repos; **custom fields are not cached** — the
function runs on every session fetch.

## openAPI — `/docs/plugins/open-api`

`openAPI()`: Scalar UI at `/api/auth/reference`, JSON via `auth.api.generateOpenAPISchema()`.
"Still in the early stages of development". Consider not exposing it in production.

## oAuthProxy — `/docs/plugins/oauth-proxy`

`oAuthProxy({ productionURL, secret })` on production **and** every preview; register only the
production callback; preview origins in `trustedOrigins`; same secret everywhere or you get
`state_mismatch`. Docs: keep the secret "out of preview deployments that run untrusted contributor
code."

## testUtils — `/docs/plugins/test-utils`

`testUtils({ captureOTP? })` in a test-only auth config; `const test = (await auth.$context).test`;
`test.createUser`, `test.saveUser`, `test.login({ userId })` → `{ headers, cookies, ... }`,
`test.getAuthHeaders({ userId })`, `test.getCookies({ userId })` (Playwright), `test.getOTP(email)`.
Docs: "Prefer keeping it out of production auth configs."
