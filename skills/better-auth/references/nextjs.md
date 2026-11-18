# Next.js (App Router) integration

Better Auth v1.7 docs, checked 2026-11-02. Main source: `https://better-auth.com/docs/integrations/next`.

## Route handler

```ts
// app/api/auth/[...all]/route.ts
import { auth } from "@/lib/auth";
import { toNextJsHandler } from "better-auth/next-js";

export const { GET, POST } = toNextJsHandler(auth);
```

Keep the path `/api/auth/[...all]` (recommended). Pages Router uses `toNodeHandler(auth.handler)`
from `better-auth/node` with `export const config = { api: { bodyParser: false } }`.

## Client

```ts
// lib/auth-client.ts
import { createAuthClient } from "better-auth/react";
export const authClient = createAuthClient();
```

## Server Components, Server Actions, Route Handlers

```tsx
import { auth } from "@/lib/auth";
import { headers } from "next/headers";
import { redirect } from "next/navigation";

export default async function DashboardPage() {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) redirect("/sign-in");
  return <h1>Welcome {session.user.name}</h1>;
}
```

- `getSession` returns `{ session, user }` or `null`. It reads the DB (or secondary storage)
  unless the cookie cache answers. Pass `query: { disableCookieCache: true }` to force a fresh
  read. — /docs/concepts/session-management
- **RSCs cannot set cookies.** Docs: the cookie cache "will not be refreshed until the server is
  interacted with from the client via Server Actions or Route Handlers." Sliding expiry
  (`updateAge`) likewise cannot write a refreshed cookie from an RSC. *(The second sentence is an
  inference from the same constraint.)*
- **Server Actions that set cookies** (`auth.api.signInEmail`, `signUpEmail`, ...) need the
  `nextCookies` plugin, last in the array:

```ts
import { nextCookies } from "better-auth/next-js";
export const auth = betterAuth({ /* ... */ plugins: [/* others */, nextCookies()] });
```

- Need the Set-Cookie header yourself? `auth.api.<x>({ ..., returnHeaders: true })` →
  `headers.getSetCookie()`. — /docs/concepts/api
- Avoid the client loading flash: fetch the session in the RSC and call
  `authClient.hydrateSession(initialSession)` in the client component before `useSession()`.
  — /docs/concepts/client#ssr-hydration
- `authClient.getSession()` does not work inside an RSC (no cookies); use `auth.api.getSession`.
  Do not use `useSession` in `layout.tsx`. — /docs/reference/faq

## Proxy / middleware — optimistic only

| Next.js | Option |
|---|---|
| 16+ | `proxy.ts` exporting `proxy()`; Node runtime, so `auth.api.getSession({ headers: await headers() })` works. |
| 15.2+ | `middleware.ts` with `export const config = { runtime: "nodejs", matcher: [...] }` for full validation — "experimental in Next.js versions before 16". |
| 13 – 15.1.x | Edge runtime, no DB access: cookie presence check, or `betterFetch` to `/api/auth/get-session` forwarding the cookie header. |

Cookie-only check (all versions):

```ts
import { NextRequest, NextResponse } from "next/server";
import { getSessionCookie } from "better-auth/cookies";

export async function middleware(request: NextRequest) {
  const sessionCookie = getSessionCookie(request);
  if (!sessionCookie) return NextResponse.redirect(new URL("/sign-in", request.url));
  return NextResponse.next();
}
export const config = { matcher: ["/dashboard"] };
```

- Docs warning: `getSessionCookie` "only checks for the existence of a session cookie; it does
  **not** validate it ... anyone can manually create a cookie to bypass it. You must always
  validate the session on your server for any protected actions or pages."
- It "does not automatically reference the auth config": with a custom `cookiePrefix` / cookie
  name pass `getSessionCookie(request, { cookieName, cookiePrefix })`.
- `getCookieCache(request)` from `better-auth/cookies` returns the session from the cookie cache
  (needs `cookieCache` enabled; subject to the same staleness window).
- The Auth.js migration guide and Next.js docs both recommend auth checks **in each page/route**,
  not in proxy or layouts. — /docs/guides/next-auth-migration-guide#protecting-resources

## Deployment notes

- Vercel previews: set `baseURL: { allowedHosts: ["myapp.com", "*.vercel.app"], protocol: "https" }`
  (allowed hosts are added to `trustedOrigins` automatically), or `oAuthProxy` for OAuth callbacks
  on changing preview URLs. See `security.md`. — /docs/guides/dynamic-base-url
- Serverless: in-memory rate limiting resets per instance; use DB or secondary storage. Background
  work: `advanced.backgroundTasks.handler: waitUntil` from `@vercel/functions` (introduces eventual
  consistency). — /docs/concepts/rate-limit#storage, /docs/reference/options#backgroundtasks
- Edge runtime: middleware on Next ≤15.1 cannot call the DB, so it cannot call `auth.api`; keep
  `auth` imports out of Edge bundles there.
- "Dual module hazard" errors (two copies of `better-auth` loaded) have a FAQ entry with
  per-bundler fixes. — /docs/reference/faq
