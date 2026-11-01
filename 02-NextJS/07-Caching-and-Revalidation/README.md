# 🗄️ Caching and Revalidation

> **TL;DR:** Next.js has **four caching layers**: Request Memoization (per render), Data Cache (fetch results across requests), Full Route Cache (prerendered HTML and RSC payload) and Router Cache (client-side). Revalidation refreshes them by time (`revalidate`) or on demand (`revalidatePath`, `revalidateTag`, `updateTag`, `refresh`). In **Next 16**, Cache Components (`'use cache'` + `cacheLife` + `cacheTag`) is the new explicit, opt-in model.

## 🧠 Concept

Caching = **storing the result of work so you don't redo it**. Revalidation = **deciding when that stored result is too old** and must be rebuilt.

Analogy: a newspaper office.

- **Request Memoization**: a reporter writing one article checks the same fact once, not ten times.
- **Data Cache**: the office archive of facts, reused for every article until someone marks them outdated.
- **Full Route Cache**: printed copies of today's paper, ready to hand out.
- **Router Cache**: the copy already in the reader's hands; flipping back doesn't need a new paper.

The big shift: **Next 13/14 cached aggressively by default** (lots of "why is my data stale?" confusion). **Next 15 made things uncached by default**, and **Next 16 makes caching explicit** with `'use cache'`.

## ⚙️ How it works / Deep dive

### The four layers

```text
 Browser                              Server
┌──────────────┐   navigation   ┌───────────────────────────────────────┐
│ Router Cache │ ─────────────► │ Full Route Cache (HTML + RSC payload) │
│ (RSC payload │                │        │ miss / dynamic route         │
│  in memory)  │                │        ▼                              │
└──────────────┘                │  React render                         │
                                │   ├─ Request Memoization (per request)│
                                │   │      │ miss                       │
                                │   │      ▼                            │
                                │   └─ Data Cache (persistent)          │
                                │          │ miss                       │
                                │          ▼                            │
                                │     Data source (API / DB)            │
                                └───────────────────────────────────────┘
```

| Layer | What | Where | Duration | Opt out / invalidate |
| --- | --- | --- | --- | --- |
| Request Memoization | Return values of identical GET `fetch` calls (or `React.cache` functions) | Server, during one render | One request | `AbortController` signal; not needed usually |
| Data Cache | `fetch` responses and `'use cache'` results | Server (persistent, survives deploys on many hosts) | Until revalidated or expired | `cache: 'no-store'`; `revalidate`; `revalidateTag` / `updateTag` |
| Full Route Cache | Rendered HTML and RSC payload of **static** routes | Server | Until revalidated or redeployed | Use dynamic APIs, `dynamic = 'force-dynamic'`, `revalidatePath` |
| Router Cache | RSC payload of visited and prefetched segments | Client memory | Session; time-limited per segment | `router.refresh()`, `refresh()`, `revalidatePath` / `updateTag` in a Server Action, `staleTimes` config |

Router Cache detail: since Next 15, **page segments aren't reused by default** (`staleTimes.dynamic = 0`), but layouts, `loading.tsx` and back/forward navigation still use it. In Next 14, dynamic pages were reused for 30 s and static pages for 5 min.

### Time-based revalidation (ISR)

```tsx
fetch(url, { next: { revalidate: 60 } }); // per fetch
export const revalidate = 60;             // per route segment (classic model)
```

Stale-while-revalidate: after 60 s the **next** request still gets stale data and triggers a background refresh; the one after gets fresh data.

### On-demand revalidation APIs (from `next/cache`)

| API | Where | Behaviour |
| --- | --- | --- |
| `revalidatePath('/blog')` | Server Action, Route Handler | Invalidates the data and route cache for a path. Optional type: `revalidatePath('/blog/[slug]', 'page' or 'layout')` |
| `revalidateTag('posts', 'max')` | Server Action, Route Handler | **Next 16 signature**: the second arg is a `cacheLife` profile (or `{ expire: n }`). Marks tagged entries stale, so the next visit gets stale-while-revalidate. Single-arg form is deprecated |
| `updateTag('posts')` | **Server Actions only** (new in 16) | Expires the tag **immediately**; the next read waits for fresh data. Gives **read-your-writes**, so the user sees their own change right after the mutation |
| `refresh()` | **Server Actions only** (new in 16) | Refreshes the client router so **uncached** dynamic data re-renders; doesn't touch cache tags |

Rule of thumb: **`updateTag`** when the user who mutated must see it instantly (edit profile, add a comment). **`revalidateTag(tag, 'max')`** for background freshness (a CMS webhook, where eventual consistency is fine).

### Cache Components (Next 16)

Enable with `cacheComponents: true` in `next.config.ts`. Then:

- **Nothing is cached unless you say so.** Data access that isn't cached must be inside `<Suspense>` (it streams as a dynamic hole), or the build errors.
- **`'use cache'`** can go at the top of a **file** (all exports cached), a **component**, or an **async function**. The output joins the **static shell** (Partial Prerendering).
- **Cache key** = build ID + function identity + **serialisable arguments** (and closed-over values). Different args mean different entries.
- **You can't call `cookies()` or `headers()` inside** a `'use cache'` scope. Read them outside and pass the values in as arguments. (Variants exist: `'use cache: private'` for per-user data and `'use cache: remote'` for a shared remote cache handler.)
- **`cacheLife(profile)`** sets how long. Profiles: `'seconds'`, `'minutes'`, `'hours'`, `'days'`, `'weeks'`, `'max'`, `'default'`, or a custom `{ stale, revalidate, expire }`.
  - `stale`: how long the client may use it without checking.
  - `revalidate`: after this, serve stale and refresh in the background.
  - `expire`: after this with no traffic, it must be regenerated (blocking).
- **`cacheTag('posts')`** labels the entry for `revalidateTag` or `updateTag`.
- Non-serialisable values like `children` can **pass through** a cached component without being part of the key.

### `unstable_cache` (legacy)

Before `'use cache'`, non-fetch data (DB queries) was cached with `unstable_cache(fn, keyParts, { revalidate, tags })`. It still works, but in Next 16 `'use cache'` replaces it.

### Version differences

| Behaviour | Next 14 | Next 15 | Next 16 |
| --- | --- | --- | --- |
| `fetch` default | Cached (`force-cache`) | Uncached | Uncached; opt in with `'use cache'` / `force-cache` |
| GET Route Handlers | Static or cached by default | Uncached by default | Uncached by default |
| Router Cache for pages | 30 s dynamic / 5 min static | 0 s for dynamic pages | Same as 15 |
| Non-fetch caching | `unstable_cache` | `unstable_cache` (`'use cache'` experimental) | `'use cache'` + `cacheLife` + `cacheTag` (stable, opt-in) |
| PPR | — | `experimental.ppr` | Via `cacheComponents: true` |
| `revalidateTag` | `revalidateTag(tag)` | `revalidateTag(tag)` | `revalidateTag(tag, profile)`; plus `updateTag`, `refresh` |

## 💻 Code examples

### Cached function with tag and lifetime

```ts
// lib/posts.ts
import { cacheLife, cacheTag } from 'next/cache';

export async function getPosts() {
  'use cache';
  cacheLife('hours');
  cacheTag('posts');
  return db.post.findMany({ orderBy: { createdAt: 'desc' } });
}
```

### Cached component vs dynamic part

```tsx
import { Suspense } from 'react';
import { cookies } from 'next/headers';

export default function Page() {
  return (
    <>
      <PostList />                                   {/* in static shell */}
      <Suspense fallback={<p>…</p>}><Greeting /></Suspense> {/* dynamic hole */}
    </>
  );
}

async function PostList() {
  'use cache';
  const posts = await getPosts();
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}

async function Greeting() {
  const name = (await cookies()).get('name')?.value ?? 'guest'; // can't be inside 'use cache'
  return <p>Hi {name}</p>;
}
```

### Mutation with read-your-writes

```ts
// app/actions.ts
'use server';
import { updateTag, revalidatePath, refresh } from 'next/cache';

export async function createPost(formData: FormData) {
  await db.post.create({ data: { title: String(formData.get('title')) } });
  updateTag('posts');          // author sees the new post immediately
  // revalidatePath('/posts'); // alternative: path-based
  // refresh();                // re-render uncached data on the current page
}
```

### CMS webhook (eventual consistency)

```ts
// app/api/revalidate/route.ts
import { revalidateTag } from 'next/cache';

export async function POST(req: Request) {
  if (req.headers.get('x-secret') !== process.env.REVALIDATE_SECRET) {
    return new Response('Unauthorized', { status: 401 });
  }
  revalidateTag('posts', 'max'); // Next 16: stale-while-revalidate with the 'max' profile
  return Response.json({ revalidated: true });
}
```

## ⚠️ Common mistakes & gotchas

- "My data is stale" in Next 14 (cached `fetch` by default) vs "my page is slow" in Next 15+ (nothing cached). Know your version.
- Calling `revalidateTag('x')` with one arg in Next 16 (deprecated). Pass a profile.
- Using `updateTag` in a Route Handler. It's **Server Actions only**.
- Reading `cookies()` inside a `'use cache'` function. Pass values as args.
- Passing non-serialisable args to a cached function (class instances, functions).
- Forgetting the client Router Cache: after a mutation, revalidate in the Server Action, or the UI may show old data.
- Using `unstable_cache` in new Next 16 code instead of `'use cache'`.

## 🎤 Say this in the interview

> "Next.js has four caches: request memoisation per render, the persistent Data Cache, the Full Route Cache for static routes, and the client Router Cache. Since Next 15 nothing is cached by default. In Next 16 I opt in with Cache Components: `'use cache'` on functions or components, `cacheLife` for duration and `cacheTag` for invalidation. After a mutation I use `updateTag` for read-your-writes, or `revalidateTag(tag, 'max')` for background refresh."

## ❓ Likely interview questions

**Name the four caching layers.**

Request Memoization (per request), Data Cache (persistent fetch and `'use cache'` results), Full Route Cache (static HTML and RSC payload), Router Cache (client).

**`revalidatePath` vs `revalidateTag`?**

Path invalidates everything for a URL. Tag invalidates every cache entry labelled with that tag, across many pages; it is more precise.

**What's new about `revalidateTag` in Next 16?**

It takes a cacheLife profile as its second argument (for example `'max'`) and uses stale-while-revalidate semantics.

**`updateTag` vs `revalidateTag`?**

`updateTag` (Server Actions only) expires immediately, so the next read is fresh: read-your-writes. `revalidateTag` marks entries stale, so they refresh in the background.

**What does `'use cache'` do and where can it go?**

It caches the return value of a file's exports, a component or a function, keyed by its arguments. With `cacheComponents` it becomes part of the static shell.

**How do you cache a DB query (not `fetch`)?**

`'use cache'` (Next 16) or `unstable_cache` (legacy). Use React `cache()` for per-request dedupe only.

**What changed in caching defaults from 14 to 15?**

`fetch`, GET Route Handlers and client page navigation stopped being cached by default.

---

⬅️ Prev: [Data Fetching](../06-Data-Fetching/README.md) | Next ➡️: [Server Actions](../08-Server-Actions/README.md)
