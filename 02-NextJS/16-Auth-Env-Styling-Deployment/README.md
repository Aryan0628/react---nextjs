# 🔐 Auth, Env Vars, Styling & Deployment

> **TL;DR:** Do **optimistic** auth redirects in `proxy.ts`, but do the **real** checks next to the data (a Data Access Layer used by Server Components, Server Actions and Route Handlers). Store sessions in `httpOnly` cookies. Env vars are server-only unless prefixed `NEXT_PUBLIC_`, and those are **inlined into the JS at build time**. Style with Tailwind or CSS Modules. Deploy on Vercel, or self-host with `next start` / Docker `output: 'standalone'`.

## 🧠 Concept

These four topics are where "it works on my machine" turns into "it works in production". Interviewers like them because they reveal whether you understand **where code runs**: on the server (secrets and DB allowed) or in the browser (everything is public).

## ⚙️ How it works / Deep dive

### 1. Authentication: where to check

Think of layers of defence:

| Layer | Check type | Why |
| --- | --- | --- |
| `proxy.ts` | **Optimistic**: does a session cookie exist? Then redirect | Fast UX and no flash of protected UI. Not a security boundary (see CVE-2025-29927) |
| **Data Access Layer (DAL)** | **Real**: verify the session, then check the role/ownership | Every read goes through it, so you can't forget |
| Server Actions | **Real**: re-check inside every action | Actions are public POST endpoints that anyone can call |
| Route Handlers | **Real**: verify inside the handler | Same, they're public URLs |
| Layouts | ❌ Don't rely on them alone | Layouts don't re-render on every navigation (partial rendering) |

#### Session vs JWT

- **Database session**: the cookie holds a random ID, and the server looks it up in the DB/Redis. Easy to revoke and always current, at the cost of one lookup per request.
- **Stateless JWT session**: the cookie holds a signed (and ideally encrypted) token with the user ID and expiry. No DB lookup, but it's hard to revoke before it expires, so keep expiry short.
- Either way, store it in an **`httpOnly`, `secure`, `sameSite=lax` cookie**. JavaScript can't read it (XSS-safe) and the browser sends it automatically. Avoid `localStorage` for tokens.

**Libraries:** Auth.js / NextAuth v5 (an `auth()` helper that works everywhere), **Better Auth** (TypeScript-first, self-hosted), **Clerk** (hosted, drop-in UI), **Supabase Auth** (with `@supabase/ssr`), plus Firebase, WorkOS and Kinde.

### 2. Environment variables

**Load order** (first match wins):

```text
1. process.env (real environment, e.g. set in Vercel/Docker)
2. .env.$(NODE_ENV).local   e.g. .env.production.local
3. .env.local               (skipped when NODE_ENV=test)
4. .env.$(NODE_ENV)         e.g. .env.development
5. .env
```

- Commit `.env` / `.env.development` (non-secret defaults). **Git-ignore `.env*.local`.**
- **Server-only** (`DATABASE_URL`, `STRIPE_SECRET_KEY`): available in Server Components, Actions, Route Handlers and the proxy. In the browser they're `undefined`.
- **`NEXT_PUBLIC_*`**: the bundler **replaces** `process.env.NEXT_PUBLIC_X` with the literal string **at build time**. That means:
  - The value is **visible to anyone** in the JS bundle, so never put secrets there.
  - Changing it requires a **rebuild**. The same Docker image can't switch values between staging and prod.
  - Dynamic lookups like `process.env[name]` are **not** inlined.
- Need a runtime-configurable public value? Read the server-only var in a Server Component and pass it down as a prop. (Next 16 removed `publicRuntimeConfig`/`serverRuntimeConfig`.)
- Add `import 'server-only'` to modules that touch secrets.

### 3. Styling

- **Tailwind CSS** is the default in `create-next-app` (v4, with `@import "tailwindcss"` in `globals.css`). It's zero-runtime and works perfectly with Server Components.
- **CSS Modules** (`Button.module.css`) give you locally scoped class names, zero runtime and RSC-friendly.
- **Global CSS** is imported once in the **root layout** (`app/layout.tsx`). In the Pages Router it could only be imported in `_app`.
- **Sass** is supported once you install `sass`.
- **CSS-in-JS** (styled-components, Emotion) needs a runtime and React context, so it **only works in Client Components** and needs a style registry using `useServerInsertedHTML` for SSR. Prefer zero-runtime options (Tailwind, CSS Modules, vanilla-extract, Panda CSS, StyleX).

### 4. Deployment

- `next build` creates `.next/` with the server bundles, static assets and prerendered HTML/RSC payloads. It prints a route table showing which routes are static (○), partially prerendered (◐) or dynamic (ƒ). In Next 16 it uses Turbopack and no longer runs lint (`next lint` was removed; use ESLint/Biome directly).
- `next start` runs the production Node server (Node 20.9+ required for Next 16).
- **Vercel**: zero config, with ISR/cache, image optimization, edge network and preview deploys built in.
- **Docker / self-host**: set `output: 'standalone'`, which traces only the needed `node_modules` into `.next/standalone/` with a minimal `server.js`. Copy `public/` and `.next/static/` in yourself.
- **Static export** `output: 'export'` produces plain HTML/CSS/JS for any static host (S3, GitHub Pages). **Limitations:** no Server Actions, no proxy/middleware, no ISR/revalidation, no `cookies()`/`headers()`, no rewrites/redirects/headers config, dynamic routes need `generateStaticParams`, the default image optimizer doesn't work (use `unoptimized` or a custom loader), and Route Handlers are limited to static GET.
- **Runtimes:** **Node.js** (default; full Node APIs, npm packages, DB drivers) vs **Edge** (`export const runtime = 'edge'`; V8 isolates, fast cold starts, close to users, but limited APIs and package support). Next 16's `proxy.ts` runs on Node.
- **Self-hosting caching considerations:**
  - By default the ISR/data cache lives on the **local filesystem/memory**. With several instances behind a load balancer they diverge, so configure a shared `cacheHandler` (and `cacheHandlers` for `'use cache'`) backed by Redis or similar.
  - `revalidateTag`/`updateTag` only purge the instance's own cache unless that cache is shared.
  - Set a stable `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` and `deploymentId` across instances to avoid version-skew errors.
  - Put a CDN in front, but respect Next's `Cache-Control` headers. Image optimization uses `sharp` on your server, which costs CPU.

## 💻 Code examples

### Data Access Layer + httpOnly cookie session

```ts
// lib/dal.ts
import 'server-only';
import { cache } from 'react';
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
import { decrypt } from '@/lib/session'; // e.g. jose jwtVerify

export const verifySession = cache(async () => {
  const token = (await cookies()).get('session')?.value; // cookies() is async (15+)
  const session = await decrypt(token);
  if (!session?.userId) redirect('/login');
  return { userId: session.userId as string, role: session.role as string };
});

export async function getInvoices() {
  const { userId } = await verifySession(); // every data read is guarded
  return db.invoice.findMany({ where: { userId } });
}
```

```ts
// app/actions/auth.ts
'use server';
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

export async function login(formData: FormData) {
  const user = await checkCredentials(formData.get('email'), formData.get('password'));
  if (!user) return { error: 'Invalid credentials' };
  const token = await encrypt({ userId: user.id, role: user.role });
  (await cookies()).set('session', token, {
    httpOnly: true, // not readable by JS (XSS-safe)
    secure: true,
    sameSite: 'lax', // CSRF mitigation
    path: '/',
    maxAge: 60 * 60 * 24 * 7,
  });
  redirect('/dashboard');
}
```

You can only **set** cookies in Server Actions, Route Handlers and the proxy, not while a Server Component is rendering.

### Standalone Docker build

```ts
// next.config.ts
const config = { output: 'standalone' as const };
export default config;
```

```dockerfile
# Dockerfile (abridged multi-stage)
FROM node:22-alpine AS build
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM node:22-alpine AS run
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

## ⚠️ Common mistakes & gotchas

- Protecting pages only in the proxy or a layout. Server Actions and Route Handlers stay callable directly.
- Storing JWTs in `localStorage`, which any XSS can steal.
- Putting a secret in a `NEXT_PUBLIC_` var, which ships it to every visitor.
- Expecting `NEXT_PUBLIC_` values to change without a rebuild.
- Using styled-components in Server Components.
- Running multiple self-hosted instances with no shared cache, which serves stale or inconsistent pages.
- Using `output: 'export'` and then adding Server Actions or the proxy, which fails at build.

## 🎤 Say this in the interview

> "I treat the proxy as an optimistic redirect only. Real authorization lives in a Data Access Layer that every Server Component, Server Action and Route Handler goes through, with sessions in httpOnly secure cookies. Secrets stay in server-only env vars, because `NEXT_PUBLIC_` values are inlined into the client bundle at build time. For deployment I use Vercel, or Docker with `output: 'standalone'` plus a shared cache handler when there are several instances."

## ❓ Likely interview questions

**Where should you check authentication in the App Router?**

Optimistically in the proxy, and authoritatively in a Data Access Layer used by Server Components, Server Actions and Route Handlers.

**Why not rely on a layout for auth?**

Layouts don't re-render on client navigations between child pages, and they don't protect Actions or Route Handlers.

**Session vs JWT?**

DB sessions are revocable but need a lookup. JWTs are stateless and fast but hard to revoke. Store either one in httpOnly cookies.

**How do NEXT_PUBLIC env vars work?**

They're inlined as string literals at build time. They're public, and changing them needs a rebuild.

**What does output standalone do?**

It produces a minimal self-contained server (`server.js` plus traced dependencies), which is ideal for Docker.

**Limitations of output export?**

No server features: Actions, proxy, ISR, cookies/headers, default image optimization, rewrites/redirects.

**Edge vs Node runtime?**

Edge has fast cold starts and runs close to users but has limited APIs. Node has full compatibility and is the default (proxy runs on Node in Next 16).

**Can you use styled-components with RSC?**

Only in Client Components, with a registry for SSR. Zero-runtime CSS is preferred.

---

⬅️ Prev: [15 Hydration & Common Errors](../15-Hydration-and-Common-Errors/README.md) | Next ➡️: [17 Interview Q&A](../17-Interview-QA/README.md)
