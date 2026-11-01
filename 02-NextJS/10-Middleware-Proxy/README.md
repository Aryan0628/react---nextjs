# 🚦 Middleware → Proxy

> **TL;DR:** A proxy (called middleware before Next.js 16) is a single function that runs **before** a request reaches a route. It can redirect, rewrite, change headers/cookies or answer straight away. In Next 16 the file is `proxy.ts` and it exports `proxy()`. Keep it light, and never make it your only auth check.

## 🧠 Concept

Picture a security guard at the building entrance. Every visitor (request) walks past the guard before getting to an office (route). The guard can:

- let them through (`NextResponse.next()`),
- send them to another entrance (**redirect**: the browser sees the new URL),
- quietly walk them to a different office while their ticket still says the old one (**rewrite**: the URL stays the same),
- add a badge (set headers or cookies),
- turn them away (return a 401/403 response).

The guard does **not** do the office work. That means no rendering pages and no heavy database queries. Those belong in Server Components, Server Actions and Route Handlers.

### Typical use cases

- **Auth redirect**: no session cookie means a redirect to `/login`.
- **i18n**: read `Accept-Language` and redirect/rewrite `/about` to `/en/about`.
- **Rewrites**: multi-tenant `acme.app.com` maps to `/tenants/acme`.
- **A/B testing**: assign a bucket cookie, then rewrite to `/home-a` or `/home-b`.
- **Headers**: security headers (CSP nonce), request IDs, CORS.
- **Geo**: block or redirect by country (taken from your host's headers).
- **Bot protection / rate limiting**: check user agents or call a fast KV store.

## ⚙️ How it works / Deep dive

### File location and naming

| Version | File | Export | Runtime |
| --- | --- | --- | --- |
| Next 12–15 | `middleware.ts` | `middleware()` | Edge by default (Node opt-in, stable in 15.5) |
| Next 16+ | `proxy.ts` | `proxy()` (or default export) | **Node.js** |
| Next 16 (legacy) | `middleware.ts` | `middleware()` | Still works but **deprecated**; keep it only if you need Edge |

- The file sits at the **project root**, or in `src/` if you use one. It goes next to `app/`, not inside it.
- You get **one file per project**. Split the logic into imported helpers.
- Why the rename? "Middleware" got confused with Express middleware. "Proxy" says what it is: a network boundary in front of your app.
- Codemod: `npx @next/codemod@canary middleware-to-proxy .`

### Where it runs in the request pipeline

```text
1. headers()  from next.config
2. redirects() from next.config
3. proxy.ts   ← you are here
4. beforeFiles rewrites → filesystem routes (public/, _next) → afterFiles rewrites
5. dynamic routes (app/[slug]) → fallback rewrites
```

### The API in one glance

- `NextRequest` extends the Web `Request` with `nextUrl` (a parsed URL with `pathname` and `searchParams`) and `cookies`.
- `NextResponse.next()` continues to the route. You can pass modified request headers.
- `NextResponse.redirect(url, 307 | 308)` sends a 3xx, and the browser makes a **new request**.
- `NextResponse.rewrite(url)` serves a different route under the **same URL**.
- `NextResponse.json()` / `new NextResponse(body, { status })` answers right away.

### Redirect vs rewrite

| | Redirect | Rewrite |
| --- | --- | --- |
| URL in address bar | Changes | Stays the same |
| Extra round trip | Yes (browser re-requests) | No (handled on the server) |
| SEO | Search engines see the new URL | Search engines see the original URL |
| Use for | Login gates, moved pages, locale prefixes | A/B tests, multi-tenancy, proxying to another app |

### Matcher

By default the proxy runs on **every** request, including JS chunks and images. The `matcher` limits it to the paths you care about. Its values must be **static constants**, because they are read at build time.

## 💻 Code examples

### Auth guard (optimistic check)

```ts
// proxy.ts (Next 16). In Next ≤15 name the file middleware.ts and the function middleware
import { NextResponse, type NextRequest } from 'next/server';

const protectedRoutes = ['/dashboard', '/settings'];

export function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const session = request.cookies.get('session')?.value; // read a cookie

  const isProtected = protectedRoutes.some((p) => pathname.startsWith(p));

  if (isProtected && !session) {
    const loginUrl = new URL('/login', request.url);
    loginUrl.searchParams.set('from', pathname); // come back here after login
    return NextResponse.redirect(loginUrl);
  }

  if (pathname === '/login' && session) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }

  return NextResponse.next();
}

export const config = {
  // Skip API routes, static assets, image optimizer and files with extensions
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico|.*\\..*).*)'],
};
```

### Rewrite for A/B testing, plus headers and cookies

```ts
import { NextResponse, type NextRequest } from 'next/server';

export function proxy(request: NextRequest) {
  // 1. Pick a sticky bucket
  const bucket =
    request.cookies.get('ab-bucket')?.value ?? (Math.random() < 0.5 ? 'a' : 'b');

  // 2. Rewrite: user sees "/", server renders "/home-a" or "/home-b"
  const res = NextResponse.rewrite(new URL(`/home-${bucket}`, request.url));

  // 3. Persist the bucket and add a response header
  res.cookies.set('ab-bucket', bucket, { path: '/', maxAge: 60 * 60 * 24 * 30 });
  res.headers.set('x-ab-bucket', bucket);
  return res;
}

export const config = { matcher: '/' };
```

### Pass data to Server Components via request headers

```ts
export function proxy(request: NextRequest) {
  const headers = new Headers(request.headers);
  headers.set('x-request-id', crypto.randomUUID());
  // Read later in a Server Component with: (await headers()).get('x-request-id')
  return NextResponse.next({ request: { headers } });
}
```

### Advanced matcher (object form)

```ts
export const config = {
  matcher: [
    {
      source: '/dashboard/:path*',
      missing: [{ type: 'header', key: 'next-router-prefetch' }], // skip prefetches
    },
  ],
};
```

## ⚠️ Common mistakes & gotchas

- **Treating the proxy as your only auth check.** In March 2025, **CVE-2025-29927** let attackers skip middleware completely by sending a crafted `x-middleware-subrequest` header. Every app that relied *only* on middleware for auth was exposed. It was patched in 15.2.3 / 14.2.25 / 13.5.9 / 12.3.5. The lesson: check again in the Data Access Layer, Server Actions and Route Handlers.
- **Heavy work in the proxy.** It runs on every matched request, prefetches included. A DB query there slows down every navigation. Stick to cookie reads or JWT signature checks.
- **Forgetting the matcher.** Without one, the proxy also runs for `/_next/static/*` and images, which wastes time and can cause redirect loops.
- **Redirect loops.** Redirecting `/login` to `/login` because the login page itself sits behind the guard.
- **Dynamic matcher values.** `matcher: getRoutes()` won't work, because it is analysed at build time.
- **`request.geo` / `request.ip`** were removed in Next 15. Use your host's headers (for example `x-vercel-ip-country`) or `@vercel/functions`.
- **Edge-only code in `proxy.ts`.** Proxy runs on Node in Next 16. If you must stay on Edge, keep `middleware.ts` for now.
- Proxy can't be used with `output: 'export'` (static export has no server).

## 🎤 Say this in the interview

> "Proxy, which was middleware before Next 16, runs before a request is matched to a route, so it's good for redirects, rewrites, headers and optimistic auth checks based on a cookie. I keep it fast and limit it with a matcher. Real authorization happens close to the data in a Data Access Layer, because the 2025 middleware-bypass CVE showed that the proxy alone is not a security boundary."

## ❓ Likely interview questions

**What is middleware/proxy in Next.js and when does it run?**

A single function in `proxy.ts` (formerly `middleware.ts`). It runs on the server before routing, after `next.config` headers/redirects and before filesystem routes and rewrites.

**What changed in Next.js 16?**

`middleware.ts` became `proxy.ts`, the exported function is now `proxy`, and it runs on the Node.js runtime. `middleware.ts` still works but is deprecated and kept for Edge use cases.

**What is the difference between redirect and rewrite?**

A redirect sends a 3xx so the browser loads a new URL that the user can see. A rewrite serves different content under the same URL, entirely on the server.

**Why use a matcher?**

To run the proxy only on the paths you need and skip static assets, images and APIs. That improves performance and avoids loops.

**Should you do database lookups in the proxy?**

Generally no. It runs on every matched request. Do cheap optimistic checks (cookie present, JWT valid) and leave the real checks to server code.

**Is middleware enough to protect routes?**

No. CVE-2025-29927 showed it can be bypassed. Always verify the session again where the data is read or mutated.

**How do you pass information from the proxy to a page?**

Set request headers with `NextResponse.next({ request: { headers } })` and read them with `headers()` in a Server Component. You can also set cookies.

**Can you have multiple middleware files?**

No, there is one per project. Compose the logic by importing helper functions and branching on `pathname`.

---

⬅️ Prev: [09 Route Handlers & API](../09-Route-Handlers-API/README.md) | Next ➡️: [11 Navigation & Linking](../11-Navigation-and-Linking/README.md)
