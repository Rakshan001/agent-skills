# Migrating to Better Auth from Auth.js / NextAuth (and others)

Better Auth v1.7 docs, checked 2026-11-02. Primary source:
`https://better-auth.com/docs/guides/next-auth-migration-guide`. Lines marked *(inference)* are
reasoned from the docs, not stated by them.

## What changes, in one table

| Auth.js v5 | Better Auth |
|---|---|
| `NextAuth({ providers: [GitHub] })` → `{ handlers, auth, signIn, signOut }` | `betterAuth({ socialProviders: { github: { clientId, clientSecret } } })` → `auth` |
| `app/api/auth/[...nextauth]/route.ts` → `export const { GET, POST } = handlers` | rename to `[...all]`; `export const { GET, POST } = toNextJsHandler(auth)` |
| `signIn("github")` (client) | `authClient.signIn.social({ provider: "github" })` |
| `signOut()` | `authClient.signOut()` / `auth.api.signOut({ headers: await headers() })` |
| `useSession()` → `{ data, status, update }` | `authClient.useSession()` → `{ data, error, isPending, isRefetching, refetch }` |
| `await auth()` (server) | `await auth.api.getSession({ headers: await headers() })` |
| `await signIn("github")` (server) | `await auth.api.signInSocial({ body: { provider: "github" } })` → `{ redirect, url }` |
| `AUTH_SECRET` | `BETTER_AUTH_SECRET` (Better Auth also falls back to `AUTH_SECRET`) |
| `AUTH_URL` / `NEXTAUTH_URL` | `BETTER_AUTH_URL` or `baseURL` |
| middleware `auth` wrapper | `getSessionCookie` (presence only) + `auth.api.getSession` in each page/route |

— migration guide, /docs/reference/options#secret

## Sessions: everyone signs in again

- Auth.js **JWT strategy** keeps no session rows, so there is nothing to migrate; Better Auth uses
  its own cookie (`better-auth.session_token`) and will not read Auth.js's JWT cookie, so every
  user must sign in once after cutover. *(inference — the guide does not say it outright; it only
  says "If you were using the database session strategy ... you will also need to migrate your
  database".)*
- Auth.js **database strategy**: you could map `Session.sessionToken → session.token` and
  `expires → expiresAt`, but Better Auth validates signed cookies, so old cookies still will not
  match *(inference)*. Plan for a forced re-sign-in either way and tell users.
- Users with OAuth links keep working after re-sign-in because Better Auth recognises a returning
  provider identity by (`providerId`, `accountId`) — as long as you migrated the account rows.

## Table mapping (from the guide)

| Auth.js | Better Auth | Notes |
|---|---|---|
| `User.id` | `user.id` | keep it (see IDs below) |
| `User.name` (optional) | `user.name` (**required**) | backfill `""` or a derived name for nulls |
| `User.email` (optional) | `user.email` (**required, unique**) | users without email need a placeholder |
| `User.emailVerified` (**Date**) | `user.emailVerified` (**boolean**) | `emailVerified = (old IS NOT NULL)` |
| `User.image` | `user.image` | |
| — | `user.createdAt`, `user.updatedAt` | required timestamps |
| `Account.provider` | `account.providerId` | |
| `Account.providerAccountId` | `account.accountId` | |
| `Account.userId` | `account.userId` | |
| `refresh_token` / `access_token` / `id_token` / `scope` | `refreshToken` / `accessToken` / `idToken` / `scope` | camelCase |
| `expires_at` (number, seconds) | `accessTokenExpiresAt` (Date) | convert |
| `type`, `token_type`, `session_state` | — | dropped |
| — | `account.password` | credential accounts only |
| `VerificationToken(identifier, token, expires)` | `verification(id, identifier, value, expiresAt, ...)` | do not migrate pending tokens; let them re-request |

Passwords: "In Better Auth, passwords must be stored in the `Account` table ... you **must** create
a record in the `account` table for each user with `providerId: "credential"` and `accountId` set to
the Better Auth user `id`. Without this record, password-based sign-ins will fail." If the old hash
is not scrypt, plug the old algorithm in:

```ts
emailAndPassword: {
  enabled: true,
  password: {
    hash: async (password) => bcrypt.hash(password, 10),
    verify: async ({ hash, password }) => bcrypt.compare(password, hash),
  },
},
```

— /docs/guides/clerk-migration-guide (bcrypt example), /docs/reference/security#password-hashing

## Keeping existing user ids

- Import script pattern (from the Clerk guide): `const ctx = await auth.$context;` then
  `ctx.adapter.create({ model: "user", data: { id, email, emailVerified, name, image, createdAt,
  updatedAt }, forceAllowId: true })` and `ctx.adapter.create({ model: "account", data: {
  providerId, accountId, userId, ... } })`. Or plain SQL `INSERT ... SELECT` from the old tables —
  ids are just strings to Better Auth.
- New ids after cutover must match the column type: uuid columns → `advanced.database.generateId:
  "uuid"` (or `() => crypto.randomUUID()`); integer ids → `"serial"`, or a function returning
  `false` for `user` only (mixed id types). — /docs/concepts/database#id-generation
- `ctx.adapter` writes bypass `databaseHooks`; `ctx.context.internalAdapter` (in hooks) runs them.
  — /docs/concepts/hooks#adapter *(that `auth.$context.adapter` skips hooks: inference from the
  hooks page's description of `adapter` vs `internalAdapter`)*

## When the app already has tables named `account` / `session`

Typical case: the app's own `account` is a handle/workspace namespace and `session` is something
else. Better Auth's core models are literally `user`, `session`, `account`, `verification`, so:

```ts
// db/auth-schema.ts — Better Auth tables under non-clashing names
export const authSession = pgTable("auth_session", { /* generated columns */ });
export const authAccount = pgTable("auth_account", { /* generated columns */ });
export const verification = pgTable("verification", { /* ... */ });

// lib/auth.ts
import { users } from "@/db/schema";            // the app's existing user table
import { authSession, authAccount, verification } from "@/db/auth-schema";

export const auth = betterAuth({
  database: drizzleAdapter(db, {
    provider: "pg",
    // explicit map; keys = the modelName of each Better Auth model below
    schema: { users, authSession, authAccount, verification },
  }),
  user: { modelName: "users" },
  session: { modelName: "authSession" },   // also makes `npx auth generate` emit these names
  account: { modelName: "authAccount" },
  // per-model `fields: { userId: "..." }` only where a Drizzle property name differs
});
```

- The Drizzle page documents two equivalent ways: re-key the adapter `schema` map
  (`{ ...schema, user: schema.users }`) **or** keep the schema and set `user: { modelName: "users" }`.
  Pick one and keep schema-map keys equal to `modelName`; a `modelName` that is not a key in the map
  is a combination the docs do not show. — /docs/adapters/drizzle#modifying-table-names
- Options used: `user|session|account|verification.modelName` and `.fields`.
  — /docs/concepts/database#custom-table-names
  Field mapping on Drizzle is by **property name**: `email: varchar("email_address")` needs no
  `fields` entry. — /docs/adapters/drizzle#modifying-field-names
- Never pass the app's whole schema object unmodified when it contains an `account` or `session`
  export — Better Auth would read and write your handle table. Pass an explicit map.
- Set the `modelName`s **before** running `npx auth@latest generate`; it loads your config, so the
  generated file will use them. Diff the output against your existing `user` table instead of
  applying it blindly. — /docs/concepts/cli
- Code reading the session/user objects still uses Better Auth's names (`session.userId`,
  `user.emailVerified`) — "Type inference ... will still use the original field names".

## NOT NULL app columns on `user` (e.g. `personal_account_id`)

Better Auth inserts users from sign-up, OAuth, magic link, admin `createUser`, etc., and will not
fill columns it does not know. Fill them in a database hook and declare them so the adapter writes
them and type inference sees them:

```ts
import { APIError } from "better-auth/api";

betterAuth({
  user: {
    additionalFields: {
      personalAccountId: { type: "string", required: false, input: false }, // server-owned
    },
  },
  databaseHooks: {
    user: {
      create: {
        before: async (user) => {
          const personalAccountId = await createPersonalAccountRow(user); // your own db write
          if (!personalAccountId) throw new APIError("BAD_REQUEST", { message: "Could not create account" });
          return { data: { ...user, personalAccountId } };               // Better Auth field names
        },
      },
    },
  },
});
```

- `input: false` keeps clients from supplying the id at sign-up. — /docs/concepts/typescript#the-input-property
- `required: false` here only affects Better Auth's input validation; the DB constraint stays NOT
  NULL and the hook guarantees the value. *(How `required: true` + `input: false` behaves without a
  `defaultValue` is not documented — unverified, so this example avoids it.)*
- Verify with `npx auth@latest check schema` — it reports "required columns it never fills".
- **Ordering / foreign keys:** the before hook runs before Better Auth's insert; the after hook runs
  only after commit. So if `user.personal_account_id → account.id`, create that row in the before
  hook. Whether your own write in the before hook shares Better Auth's transaction is not documented
  — treat it as separate: a later failure (duplicate email, DB error) can leave an orphan row, so
  make the creation idempotent or sweep orphans. A circular FK (account row also points back at
  `user.id`) cannot be satisfied this way unless the constraint is deferrable *and* both inserts are
  in one transaction — prefer a nullable back-reference filled in the after hook.
- If the extra row only depends on the user (no FK from `user`), create it in
  `databaseHooks.user.create.after` instead — it runs post-commit and is skipped on rollback.
- A throw from the hook during OAuth surfaces as `unable_to_create_user`.

## Behaviour to re-check after cutover

- `getSession` hits the DB unless `session.cookieCache` is enabled; with it, revocation lags up to
  `maxAge`. Sliding expiry: 7 days, extended when the session is ≥1 day old (`expiresIn`,
  `updateAge`). — `sessions-and-cookies.md`
- Server Actions that sign users in/out need `nextCookies()` last in `plugins`.
- Implicit account linking is on by default (verified email or `trustedProviders`). Imported users
  need truthful `emailVerified` values. — `social-and-linking.md`
- Auth.js middleware that decoded the JWT for claims has no equivalent: middleware can only see the
  cookie; read claims in the page via `getSession` (or `customSession`).
- Run the migration in development first, keep the old auth installed until verified, then remove it.
  — /docs/guides/clerk-migration-guide
