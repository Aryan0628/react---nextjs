# 📡 Data Fetching

> **TL;DR:** In the App Router you fetch data **directly inside async Server Components**, with `fetch`, an ORM, or any server SDK. There's no `getServerSideProps` and no `useEffect`. Control caching with `fetch` options (or `'use cache'` in Next 16), avoid waterfalls with `Promise.all` and `<Suspense>`, dedupe with request memoisation or React `cache()`, and use client-side libraries only for interactive, client-driven data.

## 🧠 Concept

In old React, data fetching looked like this: render, then `useEffect`, then fetch, then setState, then re-render (with loading spinners everywhere and API keys exposed). In Next.js Pages Router you used special functions (`getServerSideProps`, `getStaticProps`).

In the App Router, a **component is just an async function on the server**: it `await`s data and returns JSX. The data is fetched **next to where it is used**, close to the database, and only the resulting HTML or RSC payload goes to the browser.

Analogy: instead of the customer (browser) phoning the warehouse for each item, the shop assistant (server) grabs everything from the back room and hands over a packed bag.

## ⚙️ How it works / Deep dive

### `fetch` options

Next.js extends the native Web `fetch`:

| Option | Meaning |
| --- | --- |
| `cache: 'force-cache'` | Use the **Data Cache**: fetch once, reuse across requests (static) |
| `cache: 'no-store'` | Always fetch fresh at request time (dynamic) |
| `next: { revalidate: 60 }` | Cache, but refresh at most every 60 s (ISR for data) |
| `next: { revalidate: false }` | Cache indefinitely |
| `next: { tags: ['posts'] }` | Label the cache entry so `revalidateTag('posts', ...)` or `updateTag('posts')` can invalidate it |

### Default caching per version

| Version | `fetch` default |
| --- | --- |
| Next 13 / 14 | **Cached** (`force-cache`) unless opted out |
| Next 15 | **Not cached** (`no-store`-like). The route can still be statically prerendered if nothing dynamic is used |
| Next 16 (with `cacheComponents`) | Not cached. Wrap work in `'use cache'` to cache it; uncached fetches must sit inside `<Suspense>` |

### Fetching from a DB or ORM directly

Server Components can import Prisma, Drizzle or any SDK. No API route is needed. Because these aren't `fetch`, they aren't in the Data Cache automatically: wrap them in `'use cache'` (Next 16) or the legacy `unstable_cache`, and dedupe per request with React `cache()`.

### Sequential vs parallel fetching (waterfalls)

A **waterfall** happens when request B only starts after request A finishes, even though B doesn't need A's result.

```text
Sequential:  [--- user 1s ---][--- posts 1s ---]           = 2s
Parallel:    [--- user 1s ---]
             [--- posts 1s ---]                             = 1s
```

- **Sequential is fine** when B depends on A (you need `user.id` to get their posts).
- Otherwise **start the promises first and `await` them together** (`Promise.all` or `Promise.allSettled`).
- Better still, **split independent parts into components wrapped in `<Suspense>`** so each streams as soon as it is ready and the page doesn't wait for the slowest one.

### Request memoisation and React `cache()`

- During **one render pass**, identical `fetch` calls (same URL and options, GET) are **automatically deduped**. A layout and a page can both call `getUser()`, and only one network request runs.
- This only applies to `fetch` inside the React tree (not Route Handlers), and it is cleared after the request.
- For **non-fetch** work (DB queries), wrap the function in React's `cache()` to get the same per-request dedupe.

### Passing promises to Client Components and `use()`

You can start a fetch in a Server Component **without awaiting it**, pass the **promise** as a prop, and unwrap it in a Client Component with React 19's `use()`. The server doesn't block, and the Client Component suspends until the data streams in.

### Client-side fetching (SWR / TanStack Query)

Use it when data is **driven by client interaction or must stay live**: infinite scroll, polling, search-as-you-type, optimistic UIs, data after login in a heavily interactive widget. These libraries add caching, revalidation on focus, retries and mutations on the client. Default to server fetching for initial page data.

### `loading.tsx` with data fetching

While an async page awaits data, the nearest `loading.tsx` (an automatic Suspense boundary) shows instantly, then the page streams in. Use component-level `<Suspense>` for finer control.

### Don't call your own Route Handlers from Server Components

`fetch('/api/products')` from a Server Component is an anti-pattern:

1. **Extra network hop**: the server makes an HTTP request to itself (serialise, network, parse) when it could just call the function.
2. **Relative URLs don't work on the server**; you'd need an absolute URL and env config.
3. **At build time no server is running**, so static generation fails.
4. You lose type safety and memoisation benefits.

Instead, **extract the logic into a shared function** (`lib/products.ts`) and call it from both the Server Component and the Route Handler.

## 💻 Code examples

### Basic fetch with caching options

```tsx
// app/posts/page.tsx
export default async function PostsPage() {
  const posts: Post[] = await fetch('https://api.example.com/posts', {
    next: { revalidate: 3600, tags: ['posts'] }, // cache for 1h, tag for invalidation
  }).then(r => r.json());

  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

### Direct DB plus React `cache()` dedupe

```ts
// lib/data.ts
import 'server-only';
import { cache } from 'react';
import { db } from './db';

export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } }); // runs once per request even if called many times
});
```

### Parallel fetching

```tsx
export default async function Profile({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const userPromise = getUser(id);        // start both requests…
  const postsPromise = getPosts(id);
  const [user, posts] = await Promise.all([userPromise, postsPromise]); // …await together
  return <><h1>{user?.name}</h1><p>{posts.length} posts</p></>;
}
```

### Streaming independent parts

```tsx
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <>
      <Suspense fallback={<p>Loading stats…</p>}><Stats /></Suspense>
      <Suspense fallback={<p>Loading feed…</p>}><Feed /></Suspense>  {/* doesn't wait for Stats */}
    </>
  );
}
```

### Promise to Client Component with `use()`

```tsx
// app/page.tsx (server)
import { Suspense } from 'react';
import { Comments } from './comments';

export default function Page() {
  const commentsPromise = getComments(); // NOT awaited
  return (
    <Suspense fallback={<p>Loading comments…</p>}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  );
}
```

```tsx
// app/comments.tsx
'use client';
import { use } from 'react';

export function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise); // suspends until resolved
  return <ul>{comments.map(c => <li key={c.id}>{c.text}</li>)}</ul>;
}
```

### Client-side fetching with SWR

```tsx
'use client';
import useSWR from 'swr';
const fetcher = (url: string) => fetch(url).then(r => r.json());

export function LivePrice({ symbol }: { symbol: string }) {
  const { data, error, isLoading } = useSWR(`/api/price/${symbol}`, fetcher, { refreshInterval: 5000 });
  if (isLoading) return <span>…</span>;
  if (error) return <span>Error</span>;
  return <span>{data.price}</span>;
}
```

## ⚠️ Common mistakes & gotchas

- Using `useEffect` plus fetch for initial page data when a Server Component could fetch it.
- Accidental waterfalls: `await a(); await b();` when they're independent.
- `fetch('/api/...')` from a Server Component. Call the function directly.
- Assuming `fetch` is cached (Next 14 habit) or not cached (Next 15 habit) without checking the version.
- Expecting ORM queries to be cached like `fetch`. They aren't; use `'use cache'` or `cache()`.
- Putting `async` on a Client Component. Not supported; use `use()` or a data library.
- Leaking secrets by fetching with API keys inside Client Components.

## 🎤 Say this in the interview

> "In the App Router I fetch data in async Server Components, directly from the DB or with `fetch`, so there's no client waterfall and no exposed secrets. I parallelise independent requests with `Promise.all` or separate Suspense boundaries, rely on request memoisation or React `cache()` for dedupe, and only use SWR or TanStack Query for client-driven live data. Since Next 15 `fetch` is uncached by default, so I opt in with `force-cache`, `revalidate` or `'use cache'`."

## ❓ Likely interview questions

**How do you fetch data in the App Router?**

Make the Server Component `async` and `await` a fetch or DB call inside it. `getServerSideProps` and `getStaticProps` are Pages Router only.

**Is `fetch` cached by default?**

In Next 13/14 yes; since Next 15 no. Opt in with `cache: 'force-cache'`, `next.revalidate`, or `'use cache'` (Next 16).

**What is a request waterfall and how do you avoid it?**

Sequential requests that could run concurrently. Avoid them with `Promise.all`, or by splitting into components wrapped in their own `<Suspense>`.

**What is request memoisation?**

Automatic dedupe of identical GET `fetch` calls within one render pass. Use React `cache()` for non-fetch functions.

**Why shouldn't a Server Component call your own Route Handler?**

It adds an unnecessary HTTP round trip, needs absolute URLs, and breaks at build time. Call the shared server function directly.

**When would you still fetch on the client?**

For interactive or real-time data: polling, infinite scroll, search-as-you-type, user-triggered refreshes.

**How do you pass server data to a Client Component without blocking?**

Pass an un-awaited promise as a prop and unwrap it with `use()` inside `<Suspense>`.

---

⬅️ Prev: [Rendering Strategies](../05-Rendering-Strategies/README.md) | Next ➡️: [Caching and Revalidation](../07-Caching-and-Revalidation/README.md)
