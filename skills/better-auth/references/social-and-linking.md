# Social providers and account linking

Better Auth v1.7 docs, checked 2026-11-02. Base `https://better-auth.com`.

## Provider config (`socialProviders.<id>`)

```ts
socialProviders: {
  google: {
    clientId: process.env.GOOGLE_CLIENT_ID!,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    prompt: "select_account",
    // accessType: "offline", prompt: "select_account consent"  -> always get a refresh token
    // hd: "company.com"  -> require a Google Workspace hosted domain
  },
  github: {
    clientId: process.env.GITHUB_CLIENT_ID!,
    clientSecret: process.env.GITHUB_CLIENT_SECRET!,
  },
}
```

Options (all optional except client id/secret): `redirectURI`, `scope`, `disableDefaultScope`,
`mapProfileToUser`, `disableSignUp`, `disableImplicitSignUp`, `overrideUserInfoOnSignIn`,
`requireEmailVerification`, `prompt`, `responseMode`, `getUserInfo`, `refreshAccessToken`,
`verifyIdToken`, `disableIdTokenSignIn`, `authorizationEndpoint`, `clientKey` (TikTok).
— /docs/reference/options#socialproviders, /docs/concepts/oauth#provider-options

- **Callback URL** default: `/api/auth/callback/<provider>` on the base URL. Register exactly that
  (dev + prod) in the provider console. Google: `redirect_uri_mismatch` means `baseURL` is wrong
  or unset ("the callback URL may default to `localhost`"). — /docs/authentication/google
- `clientId` may be an array for Google/Apple/Microsoft/Facebook/Cognito (ID-token audience for
  web + iOS + Android); the code flow uses the first entry. — /docs/concepts/oauth#clientid
- **GitHub:** `user:email` scope is requested by default; a GitHub *App* needs "Email addresses:
  Read-only" permission, otherwise `email_not_found`. Private primary email → `/user` returns
  `email: null`. No refresh tokens for OAuth apps. — /docs/authentication/github
- **Google:** refresh token only on first consent unless `accessType: "offline"` +
  `prompt: "select_account consent"`. `hd: "*"` = any Workspace domain; tokens without `hd` are
  rejected when `hd` is set; a custom `getUserInfo` replaces the built-in `hd` check.
  — /docs/authentication/google
- Client: `authClient.signIn.social({ provider, callbackURL, errorCallbackURL, newUserCallbackURL,
  disableRedirect })`; server: `auth.api.signInSocial({ body: { provider } })`. — /docs/basic-usage
- `disableImplicitSignUp: true` → new users are created only when sign-in is called with
  `requestSignUp: true`. `disableSignUp` → no new users from this provider. — /docs/concepts/oauth

## Users without email

Better Auth requires an email on every user. Providers that may omit it fail with
`error=email_not_found`. Fallback with a placeholder on a reserved domain:

```ts
discord: { clientId, clientSecret,
  mapProfileToUser: (profile) => ({ email: profile.email ?? `${profile.id}@discord.placeholder.invalid` }) }
```

Placeholder addresses cannot receive reset/magic-link/invite mail. Apple only sends `email` on first
consent (persist it then); Microsoft Entra `email` is tenant-mutable and unverified — anchor on
`profile.oid`; treat Facebook emails as unverified. — /docs/concepts/oauth#handling-providers-without-email

## `mapProfileToUser` and server-owned fields

- Its return value is treated as **provider input**: only `user.additionalFields` with
  `input: true` are stored; `input: false` fields are ignored (their `defaultValue` still applies).
- Do **not** flip a role/ban/org field to `input: true` just so the mapper can write it — that also
  lets sign-up and update-user requests set it. Store verified claims with your own DB write.
- To gate sign-in on a provider claim, enforce it *before* the session is issued (e.g. Google `hd`,
  `user.validateUserInfo`, or a custom `getUserInfo` returning `null`).
— /docs/concepts/oauth#mapprofiletouser, #server-owned-fields-and-authorization-claims

## Email verification on social sign-in

`socialProviders.<p>.requireEmailVerification: true` (default `false`): when the provider reports
the email unverified, the user/account are still created or linked but **no session** is issued;
callback redirects with `?error=email_not_verified`, ID-token sign-in returns 403
`EMAIL_NOT_VERIFIED`. Independent of `emailAndPassword.requireEmailVerification`. Only enable for
providers with a trustworthy `email_verified`. — /docs/concepts/oauth#requireemailverification

## Account linking (`account.accountLinking`)

| Option | Default | Effect |
|---|---|---|
| `enabled` | `true` | Allow linking at all. |
| `trustedProviders` | — | Array or `async (request) => [...]`. Same-email sign-ins from these link **even if the provider does not confirm verification** — docs: "may increase the risk of account takeover." |
| `disableImplicitLinking` | `false` | Same-email OAuth sign-in for an existing user is rejected with `account_not_linked`; users link from settings via `linkSocial()`. New users can still sign up. |
| `allowDifferentEmails` | `false` | Allow linking a provider whose email differs. |
| `allowUnlinkingAll` | `false` | Allow removing the last account. |
| `updateUserInfoOnLink` | `false` | Copy name/image/mapped fields on link; never changes `email`/`emailVerified`. |

— /docs/reference/options#accountlinking, /docs/concepts/users-accounts#account-linking

- Implicit linking happens when the email matches an existing user **and** the provider verified
  it or is trusted. If you insert user rows yourself (migration), set `emailVerified: true` only
  where it was really verified — `account_not_linked` docs say the matching user must have
  `emailVerified: true` for linking. — /docs/reference/errors/account_not_linked
- Explicit link: `authClient.linkSocial({ provider, callbackURL, scopes? })` /
  `auth.api.linkSocialAccount({ body, headers })`; extra scopes later via `linkSocial` again (scopes
  merge into `account.scope`). Unlink: `authClient.unlinkAccount({ accountId: account.id })` — the
  local account **row id**, from `listAccounts()`.
- Adding a password to an OAuth-only user: `auth.api.setPassword({ body: { newPassword }, headers })`
  (server-only) or the forgot-password flow.
- Provider tokens are stored **unencrypted** by default; `account.encryptOAuthTokens: true` or
  encrypt in `databaseHooks.account.create.before`. `account.updateAccountOnSignIn` refreshes
  stored tokens. — /docs/reference/options#account, /docs/concepts/users-accounts#token-encryption
- `getAccessToken` needs the account row `id` as `accountId` (or `useAccountCookie: true`); a
  `providerId` is not a selector. Expired tokens are refreshed first. — /docs/concepts/oauth#get-access-token
