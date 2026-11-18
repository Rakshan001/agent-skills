# Core setup, schema, Drizzle, hooks, errors, client

Source docs are Better Auth v1.7 (checked 2026-11-02). Base: `https://better-auth.com`.

## Install and environment

- `npm install better-auth`. If client and server are separate projects, install it in both.
  — /docs/installation
- `BETTER_AUTH_SECRET`: "at least 32 characters and generated with high entropy"
  (`openssl rand -base64 32`, or `npx auth@latest secret`). — /docs/installation, /docs/concepts/cli
- Secret lookup order: `secret` option, then `BETTER_AUTH_SECRET`, then `AUTH_SECRET`. If none is
  set it falls back to a fixed dev string and **throws in production**. — /docs/reference/options#secret
  (Gotcha: an app coming from Auth.js v5 usually already has `AUTH_SECRET`; Better Auth will pick
  it up silently if `BETTER_AUTH_SECRET` is missing.)
- `BETTER_AUTH_URL` = base URL of the app. If neither `baseURL` nor the env var is set, it is
  inferred from the request — the docs call that "not recommended". — /docs/reference/options#baseurl
- `basePath` defaults to `/api/auth`. — /docs/reference/options#basepath
- Auth file: `auth.ts` in the project root, `lib/` or `utils/` (optionally under `src/`, `app/`,
  `server/`), exporting `auth` or a default export. The CLI looks there. — /docs/installation

```ts
// src/lib/auth.ts
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "@better-auth/drizzle-adapter"; // see note below
import { db } from "@/db";

export const auth = betterAuth({
  database: drizzleAdapter(db, { provider: "pg" }),
  emailAndPassword: { enabled: true },
  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID as string,
      clientSecret: process.env.GITHUB_CLIENT_SECRET as string,
    },
  },
});
```

**Drizzle adapter import path differs between pages of the same docs version.** The Drizzle page
says `npm install @better-auth/drizzle-adapter` and `import { drizzleAdapter } from
"@better-auth/drizzle-adapter"` (Relations v2: `@better-auth/drizzle-adapter/relations-v2`);
the installation and performance pages still show `better-auth/adapters/drizzle`. Use whichever
your installed version exports. — /docs/adapters/drizzle, /docs/installation

With an ORM adapter you can import `betterAuth` from `better-auth/minimal` (no Kysely; no built-in
migrations). — /docs/guides/optimizing-for-performance#bundle-size-optimization

## CLI (v1.7 name: `auth`)

| Command | What it does |
|---|---|
| `npx auth@latest generate` | Writes an ORM schema (Drizzle/Prisma) or SQL (Kysely). Loads your config, so plugins, `modelName`, `fields`, `additionalFields` are reflected. `--config`, `--output`, `--adapter drizzle --dialect postgresql` to run without a DB connection. |
| `npx auth@latest migrate` | Applies schema directly — **built-in Kysely adapter only**. |
| `npx auth@latest check schema --config ./src/lib/auth.ts` | Verifies the configured schema can hold what Better Auth writes. For Drizzle it reads the schema object, **not** the live DB, so it cannot tell you a migration was not applied. |
| `npx auth@latest create-admin --email ... --name ... --role admin` | First admin (admin plugin). |
| `npx auth@latest upgrade`, `info`, `init`, `secret` | Upgrade helper, diagnostics, scaffold, secret. |

— /docs/concepts/cli. Older docs and blog posts use `npx @better-auth/cli generate`; the v1.7 docs
use `npx auth`. *(Whether the old package name still works on 1.7: unverified.)*

With Drizzle: `npx auth@latest generate` → `npx drizzle-kit generate` → `npx drizzle-kit migrate`.
— /docs/adapters/drizzle

Runtime schema validation: at init Better Auth compares the schema with what it writes and
"reports missing tables, missing columns, and required columns it never fills". Disable with
`advanced.database.validateSchema: false`. (The database page says it is on "including in
production"; the options page says "default: `true` outside production" — the two pages disagree.)
— /docs/concepts/database#schema-validation, /docs/reference/options#advanced

## Core schema (four tables)

Field names are Better Auth's logical names; the generator maps them to your ORM.
— /docs/concepts/database#core-schema (rendered table), /docs/guides/next-auth-migration-guide

| Table | Fields |
|---|---|
| `user` | `id` string PK · `name` string (required) · `email` string unique (required) · `emailVerified` **boolean** (required) · `image` string? · `createdAt` Date · `updatedAt` Date |
| `session` | `id` · `userId` FK→user (cascade) · `token` unique · `expiresAt` · `ipAddress`? · `userAgent`? · `createdAt` · `updatedAt` |
| `account` | `id` · `userId` FK→user (cascade) · `accountId` (provider's id for the user) · `providerId` · `accessToken`? · `refreshToken`? · `accessTokenExpiresAt`? · `refreshTokenExpiresAt`? · `scope`? · `idToken`? · `password`? · `createdAt` · `updatedAt` |
| `verification` | `id` · `identifier` · `value` · `expiresAt` · `createdAt` · `updatedAt` |

- An `account` row is **one sign-in method** for a user, identified by the pair
  (`providerId`, `accountId`). Email/password lives here as `providerId: "credential"`,
  `accountId: <user.id>`, `password: <hash>` — not on `user`. — /docs/concepts/database#account
- `name`, `image` and `email` cannot be removed from `user`. — /docs/reference/faq
- Plugins add tables/columns; re-run `generate` after adding one. — /docs/concepts/database#plugins-schema
- Recommended indexes: `user.email`, `account.userId`, `session.userId`, `session.token`,
  `verification.identifier` (+ plugin ones). — /docs/guides/optimizing-for-performance

## Renaming tables and columns

Config side (works with every adapter; the CLI honours it):

```ts
betterAuth({
  user:    { modelName: "users", fields: { name: "full_name", email: "email_address" } },
  session: { modelName: "user_sessions", fields: { userId: "user_id" } },
  account: { modelName: "accounts", fields: { userId: "user_id" } },
  verification: { modelName: "verifications" },
});
```

"Type inference in your code will still use the original field names (e.g., `user.name`, not
`user.full_name`)." Plugin tables/columns are renamed through the plugin's own `schema` option.
— /docs/concepts/database#custom-table-names, /docs/reference/options#user

Drizzle side — the adapter looks tables up by **key in the schema object** and maps fields by the
Drizzle **property name**:

```ts
import * as schema from "./schema";
drizzleAdapter(db, {
  provider: "pg",
  schema: { ...schema, user: schema.users }, // Better Auth model "user" -> your `users` table
});
// or keep `schema` and set `user: { modelName: "users" }` in the auth config

export const user = pgTable("user", {
  email: varchar("email_address", { length: 255 }).notNull().unique(), // prop `email`, column `email_address`
});
```

- `usePlural: true` if all your tables are plural. `schemaName: "auth"` puts generated tables in a
  Postgres schema (`pgSchema("auth")`). — /docs/adapters/drizzle
- **Clash rule:** if the app already exports its own `account` / `session` table in that schema
  object, do **not** pass it unmodified. Pass an explicit map whose keys equal Better Auth's
  `modelName`s (e.g. `schema: { users, authSession, authAccount, verification }` with
  `session: { modelName: "authSession" }`, `account: { modelName: "authAccount" }`), so
  `generate` also emits non-clashing names. *(That `modelName` is looked up as a schema-object key
  is inferred from the docs' two equivalent examples.)* Full example: `migration-from-authjs.md`.

## Extra columns: `additionalFields`

```ts
betterAuth({
  user: {
    additionalFields: {
      role: { type: ["user", "admin"], required: false, defaultValue: "user", input: false },
      lang: { type: "string", required: false, defaultValue: "en" },
    },
  },
});
```

- `type`, `required`, `defaultValue` ("only applies in the JavaScript layer; in the database, the
  field will be optional"), `input` (default `true`: sign-up/update/`mapProfileToUser` may set it),
  `returned` (default `true`). — /docs/concepts/database#extending-core-schema
- **Set `input: false` on anything server-owned** (roles, flags, foreign keys) — with the default
  `input: true` a client can send it on sign-up. — /docs/concepts/typescript#the-input-property
- Client typing: `inferAdditionalFields<typeof auth>()` from `better-auth/client/plugins`, or
  pass the field map when client and server are separate projects. — /docs/concepts/typescript

## ID generation

`advanced.database.generateId`: default a random base62 string; `"uuid"`; `"serial"` (numeric,
still typed as string, values arrive as `"123"`); `false` (DB generates every id); or a function
`({ model, size }) => string | false | undefined` (return `false`/`undefined` to let the DB generate
that model's id). "By default, Better-Auth will generate UUIDs for the `id` field for all tables,
except adapters that use `PostgreSQL` where we allow the database to generate the UUID
automatically" (under the `"uuid"` setting). — /docs/concepts/database#id-generation,
/docs/reference/options#advanced

Keeping existing ids during an import: `ctx.adapter.create({ model: "user", data: { id, ... },
forceAllowId: true })` where `ctx = await auth.$context`. — /docs/guides/clerk-migration-guide

## `databaseHooks` (user, session, account, verification)

```ts
import { APIError } from "better-auth/api";

betterAuth({
  databaseHooks: {
    user: {
      create: {
        before: async (user, ctx) => {
          if (blocked(user.email)) return false;                 // abort the insert
          return { data: { ...user, firstName: user.name.split(" ")[0] } }; // replace payload
        },
        after: async (user) => { /* side effects: welcome email, billing customer */ },
      },
      update: { before: async (data, ctx) => ({ data }) },       // ctx.context.session available
      delete: { before: async (user) => true },
    },
    session: { create: { before: async (session) => ({ data: { ...session } }) } },
  },
});
```

- Before hook: return `false` → abort; return `{ data }` → replaces the payload ("return
  Better-Auth named fields, not the original field names in your database"); throw
  `new APIError("BAD_REQUEST", { message })` to fail with a message. — /docs/concepts/database#database-hooks
- After hook: "When the operation is part of a transaction, the hook runs after it commits and is
  skipped if it rolls back. If transactions aren't enabled for your database adapter, the hook also
  runs when a later step of the same request fails." Put emails / external calls in after hooks.
  — /docs/concepts/database#2-after-hook
- "Additional fields are supported, however full type inference for these fields isn't yet
  supported." — same page
- Whether DB writes you make *inside* a before hook (with your own `db`) share Better Auth's
  transaction: **not documented — assume they do not** and make them idempotent / clean them up.
- Policy gate that runs before user creation, account linking and (for OAuth/SSO) every
  re-sign-in: `user.validateUserInfo: ({ user, source }) => ({ error, errorDescription }) | void`.
  `databaseHooks.user.create.before` still runs afterwards. — /docs/concepts/users-accounts#callbacks
- A throwing hook during OAuth sign-up surfaces as the `unable_to_create_user` error page.
  — /docs/reference/errors/unable_to_create_user

## Request `hooks` (before/after middleware)

```ts
import { createAuthMiddleware, APIError } from "better-auth/api";

betterAuth({
  hooks: {
    before: createAuthMiddleware(async (ctx) => {
      if (ctx.path !== "/sign-up/email") return;
      if (!ctx.body?.email.endsWith("@example.com"))
        throw new APIError("BAD_REQUEST", { message: "Email must end with @example.com" });
    }),
    after: createAuthMiddleware(async (ctx) => {
      if (ctx.path.startsWith("/sign-up") && ctx.context.newSession) { /* ... */ }
    }),
  },
});
```

- One function per `before`/`after` — branch on `ctx.path`. — /docs/concepts/hooks
- `ctx`: `path`, `body`, `headers`, `request` (may be absent for `auth.api` calls), `query`,
  `context` (`newSession` in after hooks, `returned`, `responseHeaders`, `authCookies`, `secret`,
  `password.hash/verify`, `adapter`, `internalAdapter`, `generateId`, `runInBackground`,
  `runInBackgroundOrAwait`). Helpers: `ctx.json`, `throw ctx.redirect(url)`, `ctx.setCookie`,
  `ctx.setSignedCookie`, `ctx.getCookie`, `ctx.getSignedCookie`. — /docs/concepts/hooks
- Prefer `ctx.context.internalAdapter` over raw `db` writes when you want `databaseHooks` and
  secondary storage honoured. — /docs/concepts/hooks#internal-adapter

## Server API and errors

- Every endpoint (core + plugins) is `auth.api.<name>({ body, query, headers })`. Pass
  `returnHeaders: true` to get `{ headers, response }` (read `headers.getSetCookie()`), or
  `asResponse: true` for a `Response`. — /docs/concepts/api
- Failures throw `APIError`; check with `isAPIError(error)` from `better-auth/api`
  (`error.message`, `error.status`). — /docs/concepts/api#error-handling
- `auth.api` calls bypass rate limiting. — /docs/concepts/rate-limit
- Browser-flow errors redirect to `onAPIError.errorURL` (default `/api/auth/error`); also
  `onAPIError.throw`, `onAPIError.onError(error, ctx)`, `customizeDefaultErrorPage`.
  — /docs/reference/options#onapierror. Error codes listed at /docs/reference/errors
  (`state_mismatch`, `account_not_linked`, `unable_to_create_user`, `email_not_found`,
  `signup_disabled`, ...).
- `disabledPaths: ["/sign-up/email"]` turns endpoints off. — /docs/reference/options#disabledpaths

## Client

```ts
import { createAuthClient } from "better-auth/react"; // or /vue /svelte /solid /client
export const authClient = createAuthClient({ /* baseURL only if on another origin */ });

await authClient.signIn.social({ provider: "github", callbackURL: "/dashboard",
  errorCallbackURL: "/error", newUserCallbackURL: "/welcome" });
await authClient.signIn.email({ email, password });
await authClient.signOut({ fetchOptions: { onSuccess: () => router.push("/login") } });
const { data, error, isPending, refetch } = authClient.useSession();
```

- If the auth routes are not at `/api/auth`, pass the full URL including the path as `baseURL`.
  — /docs/installation
- Calls return `{ data, error }`; `error.code` keys are on `authClient.$ERROR_CODES`.
  — /docs/concepts/client#error-codes
- Types: `typeof authClient.$Infer.Session` / `typeof auth.$Infer.Session` (`{ session, user }`).
  — /docs/concepts/typescript
- `authClient.getSession` cannot read cookies in a Server Component; use `auth.api.getSession`.
  Do not put `useSession` in `layout.tsx`. — /docs/reference/faq
- TypeScript: `strict: true` (or at least `strictNullChecks`). — /docs/reference/faq
