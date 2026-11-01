# 🖼️ Rendering Strategies (CSR, SSR, SSG, ISR, Streaming, PPR)

> **TL;DR:** "Rendering strategy" means **when and where HTML is generated**: in the browser (CSR), on every request (SSR), at build time (SSG), at build time and refreshed periodically (ISR), progressively in chunks (Streaming), or as a static shell with dynamic holes (PPR). In the App Router you don't pick a function. Next.js decides **static vs dynamic per route** based on what your code uses. In Next 16, **Cache Components** make PPR the model.

## 🧠 Concept

Analogy: a bakery.

- **SSG**: bake all the bread at 5 a.m. (build). Instant to serve, but it may get stale.
- **ISR**: bake in advance, but re-bake a loaf in the background once it's older than X minutes.
- **SSR**: bake each loaf when the customer orders. Always fresh, but they wait.
- **CSR**: hand the customer flour and a recipe (JS) and let them bake at home.
- **Streaming**: hand over the sandwich piece by piece as each part is ready.
- **PPR**: the box and bread are pre-made (static shell); the custom filling (dynamic part) is added per order.

## ⚙️ How it works / Deep dive

### CSR: Client-Side Rendering

```text
Request → empty HTML + JS bundle → browser runs JS → fetches data → renders UI
```

- ✅ Cheap hosting, rich interactivity after load.
- ❌ Slow first paint, poor SEO, client-side data waterfalls.
- Use for: highly interactive, auth-only widgets (dashboards behind login), or parts of a page via Client Components plus SWR or TanStack Query.

### SSR: Server-Side Rendering (dynamic rendering)

```text
Request → server fetches data → renders HTML → sends HTML → hydrate
```

- ✅ Always fresh, per-user content, SEO-friendly.
- ❌ Slower TTFB (time to first byte), because the server works on every request; costs more.
- Use for: personalised pages, pages depending on cookies, headers or search params.

### SSG: Static Site Generation

```text
Build → render HTML once → deploy to CDN → every request gets the same file instantly
```

- ✅ Fastest, cheapest, very scalable.
- ❌ Data is frozen at build time; long builds for many pages.
- Use for: marketing pages, docs, blogs. This is the **default** in the App Router when nothing dynamic is used.

### ISR: Incremental Static Regeneration

```text
Build → static HTML (cached)
Request after `revalidate` seconds → serve STALE page instantly, regenerate in background
Next request → fresh page
```

- Called **stale-while-revalidate**. Can also be triggered **on demand** with `revalidatePath` or `revalidateTag`.
- ✅ Static speed with fresh-enough data.
- ❌ One visitor sees stale content after expiry.
- Use for: product catalogues, news, CMS content.

### Streaming

```text
Request → server sends the shell + loading fallbacks immediately
        → each <Suspense> section streams in as its data resolves
```

- Built on React Suspense; `loading.tsx` is a route-level Suspense boundary.
- ✅ Better TTFB and FCP; slow data doesn't block the whole page.
- Use for: pages mixing fast and slow data (a dashboard with a slow "recommendations" widget).

### PPR: Partial Prerendering (via Cache Components in Next 16)

```text
Build → prerender a STATIC SHELL (nav, layout, cached content, Suspense fallbacks)
Request → shell served instantly from CDN, in the same response the
          dynamic "holes" (inside <Suspense>) are rendered and streamed in
```

- Combines SSG speed with SSR freshness **on the same page**.
- Next 14/15 had it as experimental (`experimental.ppr`). **Next 16 delivers it through `cacheComponents: true`**, and the old `ppr` flag is gone.

### Static vs dynamic rendering in the App Router

By default Next.js tries to render each route **statically** at build time. A route becomes **dynamic** (rendered per request) when it uses:

| Trigger | Why |
| --- | --- |
| `cookies()`, `headers()`, `draftMode()` | Request-specific data |
| `searchParams` prop in a page | Query string known only at request time |
| `connection()` from `next/server` | Explicitly "wait for a real request" |
| Uncached data: `fetch(url, { cache: 'no-store' })` | Must be fresh each time |
| `export const dynamic = 'force-dynamic'` | Manual opt-in |
| Dynamic `params` not covered by `generateStaticParams` | Rendered on demand (then cached) |

`next build` prints `○ (Static)` or `ƒ (Dynamic)` per route, so check it.

**Version note:** since Next 15 `fetch` isn't cached by default, but a route with a plain `fetch` and no request APIs can **still be prerendered statically** at build time. In Next 13/14 `fetch` was cached by default, so more things were static implicitly.

### Route segment config (classic model)

```tsx
export const dynamic = 'auto';   // 'auto' | 'force-dynamic' | 'force-static' | 'error'
export const revalidate = 60;    // ISR: seconds; false = cache forever; 0 = always dynamic
export const dynamicParams = true; // false → 404 for params not in generateStaticParams
```

With `cacheComponents: true` (Next 16), `dynamic` and `revalidate` segment configs are **not used**. You express caching with `'use cache'` and `cacheLife` instead (see the caching topic).

### How Cache Components changes the model (Next 16)

Instead of "the whole route is static **or** dynamic", it's **per component**:

1. Everything that doesn't touch request data or uncached I/O is prerendered into the **static shell**.
2. Data you want cached goes in a `'use cache'` function or component and **joins the shell**.
3. Anything dynamic (`cookies()`, uncached fetch, `await params` for unknown values) **must be inside `<Suspense>`**. It becomes a streamed **dynamic hole**. If you forget the Suspense boundary, the build errors ("uncached data accessed outside of `<Suspense>`").

## 💻 Code examples

### SSG for dynamic routes with `generateStaticParams`

```tsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetch('https://cms.example.com/posts').then(r => r.json());
  return posts.map((p: { slug: string }) => ({ slug: p.slug })); // pre-built at build time
}

export const dynamicParams = true; // unknown slugs are rendered on demand, then cached
export const revalidate = 3600;    // ISR: regenerate at most hourly

export default async function Post({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await fetch(`https://cms.example.com/posts/${slug}`).then(r => r.json());
  return <article>{post.title}</article>;
}
```

### Streaming with Suspense

```tsx
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <>
      <h1>Dashboard</h1>                       {/* sent immediately */}
      <Suspense fallback={<p>Loading revenue…</p>}>
        <Revenue />                            {/* streamed when ready */}
      </Suspense>
    </>
  );
}

async function Revenue() {
  const data = await fetch('https://api.example.com/revenue', { cache: 'no-store' }).then(r => r.json());
  return <p>${data.total}</p>;
}
```

### PPR with Cache Components (Next 16)

```tsx
// next.config.ts → { cacheComponents: true }
import { Suspense } from 'react';
import { cookies } from 'next/headers';
import { cacheLife } from 'next/cache';

export default function ProductPage() {
  return (
    <>
      <ProductInfo />                              {/* cached → static shell */}
      <Suspense fallback={<p>Loading cart…</p>}>
        <Cart />                                   {/* dynamic hole */}
      </Suspense>
    </>
  );
}

async function ProductInfo() {
  'use cache';
  cacheLife('hours');
  const p = await db.product.findFirst();
  return <h1>{p?.name}</h1>;
}

async function Cart() {
  const session = (await cookies()).get('session')?.value; // request-time data
  const cart = await getCart(session);
  return <p>{cart.items.length} items</p>;
}
```

## 📊 Big comparison table

| Strategy | HTML generated | Freshness | TTFB | SEO | Typical use |
| --- | --- | --- | --- | --- | --- |
| CSR | Browser, after JS | Live | Fast (empty) but slow content | ❌ Weak | Private, interactive widgets |
| SSR | Server, per request | Always fresh | Slower | ✅ | Personalised or request-based pages |
| SSG | Build time | Stale until rebuild | Fastest | ✅ | Marketing, docs |
| ISR | Build, then background regen | Fresh within N seconds or on demand | Fastest | ✅ | Catalogues, CMS, blogs |
| Streaming | Server, progressive chunks | Fresh | Fast shell | ✅ | Mixed fast and slow data |
| PPR / Cache Components | Static shell at build plus dynamic holes per request | Per component | Fastest shell | ✅ | E-commerce, dashboards, most apps |

## ⚠️ Common mistakes & gotchas

- Calling `cookies()` in a root layout, which makes **every** route dynamic.
- Expecting `revalidate` to update instantly. ISR serves stale once, then regenerates.
- Confusing SSR with streaming: streaming is SSR sent in chunks, not a separate data model.
- Forgetting `generateStaticParams`, so dynamic pages render on first request instead of at build.
- With `cacheComponents`, accessing dynamic data outside `<Suspense>` causes a build error.
- Assuming Next 14 behaviour (cached `fetch`) in Next 15/16 projects.

## 🎤 Say this in the interview

> "The App Router renders routes statically by default and switches to dynamic rendering when you use request APIs like `cookies()`, `headers()` or `searchParams`, or uncached data. ISR adds time-based or on-demand revalidation, and streaming with Suspense sends the shell first. In Next 16, Cache Components deliver Partial Prerendering: a static shell made of cached parts, with dynamic parts streamed inside Suspense boundaries."

## ❓ Likely interview questions

**SSR vs SSG?**

SSR renders HTML on every request (fresh, slower). SSG renders once at build (fast, possibly stale).

**What is ISR and how does it work?**

Static pages that regenerate in the background after a `revalidate` interval or on demand, using stale-while-revalidate.

**What makes an App Router route dynamic?**

`cookies()`, `headers()`, `searchParams`, `connection()`, uncached data, or `dynamic = 'force-dynamic'`.

**What does `generateStaticParams` do?**

It returns the list of params to pre-render at build time for dynamic routes. `dynamicParams = false` returns 404 for anything else.

**What is streaming and why use it?**

Sending HTML in chunks as Suspense boundaries resolve, so slow data doesn't block the page. It improves TTFB and FCP.

**What is Partial Prerendering?**

A static shell served instantly, plus dynamic holes rendered at request time in the same response. In Next 16 it is enabled through `cacheComponents: true`.

**How do you check if a route is static or dynamic?**

Look at the `next build` output (○ static vs ƒ dynamic).

---

⬅️ Prev: [Server vs Client Components](../04-Server-vs-Client-Components/README.md) | Next ➡️: [Data Fetching](../06-Data-Fetching/README.md)
