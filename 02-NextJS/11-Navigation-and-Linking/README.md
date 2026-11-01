# 🧭 Navigation & Linking

> **TL;DR:** Use `<Link>` for normal navigation. It navigates on the client and prefetches. Use `useRouter()` from **`next/navigation`** for navigation from code. Hooks like `usePathname`, `useSearchParams` and `useParams` read the URL in Client Components. On the server, use `redirect()`, `permanentRedirect()` and `notFound()`.

## 🧠 Concept

A normal `<a href>` makes the browser throw the page away and download a new HTML document. That's a **hard navigation**: every bit of state is lost and the screen flashes.

Next.js turns navigation into a **soft navigation**. When you click a `<Link>`, the router fetches only the **RSC payload** for the segments that changed, swaps them in, and keeps the shared layouts (and their state) mounted. It feels like a SPA, yet every page is still server-rendered on first load.

Two more tricks make it feel instant:

- **Prefetching**: when a `<Link>` enters the viewport (in production), Next fetches the next route in the background.
- **Router Cache**: an in-memory cache of the segments you have visited, so back/forward is instant.

## ⚙️ How it works / Deep dive

### `<Link>` props worth knowing

| Prop | What it does |
| --- | --- |
| `href` | String or object `{ pathname: '/shop', query: { page: 2 } }` |
| `replace` | Replaces the current history entry instead of pushing a new one |
| `scroll` | Default `true`: scrolls to the new page's top if it isn't visible. `false` keeps the scroll position |
| `prefetch` | Default (`null`/"auto"): static routes are fully prefetched; dynamic routes only up to the nearest `loading.tsx`. `true` means full, `false` means none |

Prefetching happens **only in production** (`next build && next start`), never in `next dev`.

### Soft vs hard navigation

| Soft (client-side) | Hard (full reload) |
| --- | --- |
| `<Link>`, `router.push` | `<a>`, `window.location`, external URLs |
| Only the changed segments re-render | The whole document reloads |
| Layout state, scroll and inputs are kept | All client state is lost |
| — | Also happens when moving between **different root layouts** |

### Router Cache and back/forward

- Visited segments and prefetched data live in the client **Router Cache**.
- **Back/forward always restores from this cache**, which is why it's instant and keeps scroll position.
- Since **Next 15**, page segments are not reused on a fresh `<Link>` navigation by default (`staleTimes.dynamic = 0`), so you see fresh data. Layouts and `loading.tsx` are still reused. You can tune this with `experimental.staleTimes`.
- `router.refresh()`, `revalidatePath`/`revalidateTag`/`updateTag` in a Server Action, and `cookies().set` all invalidate it.

### `useRouter` (App Router)

```ts
import { useRouter } from 'next/navigation'; // NOT 'next/router'
```

| Method | Purpose |
| --- | --- |
| `push(href, { scroll })` | Navigate and add a history entry |
| `replace(href)` | Navigate without adding a history entry |
| `refresh()` | Re-fetch Server Components for the current route. Client state (useState) is kept |
| `back()` / `forward()` | Browser history |
| `prefetch(href)` | Manually prefetch a route |

⚠️ **`next/router` is the Pages Router API.** Importing it in `app/` throws "NextRouter was not mounted". The App Router has no `router.query`, `router.pathname` or `router.events`. Use the hooks below instead.

### URL hooks (Client Components only)

- `usePathname()` returns `'/blog/hello'` (no query string).
- `useSearchParams()` returns a read-only `URLSearchParams`. On a **statically rendered** route, the component that calls it must be wrapped in `<Suspense>`. Otherwise the build fails with "Missing Suspense boundary with useSearchParams", because search params are only known in the browser.
- `useParams()` returns `{ slug: 'hello' }` for dynamic segments. It's synchronous on the client.
- `useSelectedLayoutSegment(s)()` gives the active child segment(s), which is handy for tabs.

In **Server Components** you read the URL from props instead: `params` and `searchParams` are **Promises** (async since Next 15, and sync access was removed in 16).

### Server-side navigation helpers (`next/navigation`)

| Function | Status | Use |
| --- | --- | --- |
| `redirect(url)` | 307 (303 in a Server Action) | After a mutation, or when an auth check fails |
| `permanentRedirect(url)` | 308 | A resource moved for good (SEO) |
| `notFound()` | 404 | Renders the nearest `not-found.tsx` |

They work by **throwing** a special error. Call them outside `try/catch`, or rethrow.

### Redirects in `next.config.ts`

Use these for static, known-ahead redirects such as an old URL scheme. They run **before** the proxy.

## 💻 Code examples

### Active nav link

```tsx
'use client';
import Link from 'next/link';
import { usePathname } from 'next/navigation';

const links = [
  { href: '/', label: 'Home' },
  { href: '/blog', label: 'Blog' },
];

export function NavLinks() {
  const pathname = usePathname();
  return (
    <nav>
      {links.map(({ href, label }) => {
        const active = href === '/' ? pathname === '/' : pathname.startsWith(href);
        return (
          <Link key={href} href={href} className={active ? 'font-bold underline' : ''}
            aria-current={active ? 'page' : undefined}>
            {label}
          </Link>
        );
      })}
    </nav>
  );
}
```

### Updating search params (filters) with Suspense

```tsx
'use client';
import { useRouter, usePathname, useSearchParams } from 'next/navigation';

export function SortSelect() {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  function onChange(value: string) {
    const params = new URLSearchParams(searchParams.toString()); // copy (read-only original)
    params.set('sort', value);
    router.replace(`${pathname}?${params}`, { scroll: false });
  }

  return (
    <select defaultValue={searchParams.get('sort') ?? 'new'} onChange={(e) => onChange(e.target.value)}>
      <option value="new">Newest</option>
      <option value="price">Price</option>
    </select>
  );
}

// page.tsx (Server Component)
// <Suspense fallback={<p>Loading…</p>}><SortSelect /></Suspense>
```

### Server-side redirect and notFound

```tsx
// app/posts/[id]/page.tsx
import { notFound, redirect } from 'next/navigation';

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params; // async in Next 15/16
  const post = await getPost(id);
  if (!post) notFound();
  if (post.movedTo) redirect(`/posts/${post.movedTo}`);
  return <h1>{post.title}</h1>;
}
```

### Config redirects

```ts
// next.config.ts
import type { NextConfig } from 'next';

const config: NextConfig = {
  async redirects() {
    return [
      { source: '/old-blog/:slug', destination: '/blog/:slug', permanent: true }, // 308
    ];
  },
};
export default config;
```

## ⚠️ Common mistakes & gotchas

- Importing `useRouter` from `next/router` in the App Router.
- Using `<a>` for internal links, which causes full reloads and loses state.
- Wrapping `redirect()` in `try/catch`, which swallows the redirect.
- Calling `redirect()` inside a client event handler. Use `router.push` there, or call a Server Action that redirects.
- Forgetting `<Suspense>` around `useSearchParams` consumers, which makes the build fail or turns the whole page client-rendered.
- Testing prefetch behaviour in `next dev`. It only runs in production builds.
- Expecting `router.refresh()` to reset client state. It doesn't; it only re-fetches server data.

## 🎤 Say this in the interview

> "In the App Router I use `<Link>` for navigation. It does soft, client-side navigation, re-renders only the changed segments and prefetches routes in the viewport. For navigation from code I use `useRouter` from `next/navigation`, and on the server `redirect`, `permanentRedirect` or `notFound`. Back/forward is instant because visited segments live in the client Router Cache."

## ❓ Likely interview questions

**Link vs a tag?**

`<Link>` does client-side navigation with prefetching and keeps layout state. `<a>` triggers a full document reload.

**next/router vs next/navigation?**

`next/router` belongs to the Pages Router. `next/navigation` is the App Router package with `useRouter`, `usePathname`, `useSearchParams`, `useParams`, `redirect` and `notFound`.

**What does router.refresh() do?**

It re-requests the current route's Server Components and merges the result without losing client state or scroll position.

**Why does useSearchParams need Suspense?**

On static routes the query string is unknown at build time. Suspense lets Next prerender everything else and render that part on the client.

**redirect vs permanentRedirect?**

307 (temporary, 303 inside a Server Action) vs 308 (permanent, so search engines move ranking to the new URL).

**How does prefetching work?**

In production, links in the viewport are prefetched. Static routes are prefetched fully, dynamic routes up to the nearest `loading.tsx`. You control it with the `prefetch` prop.

**What is soft navigation?**

Client-side transitions where only the changed route segments are fetched and rendered, while shared layouts stay mounted.

**How do you listen to route changes in the App Router?**

There's no `router.events`. Run a `useEffect` that depends on `usePathname()` and `useSearchParams()`.

---

⬅️ Prev: [10 Middleware → Proxy](../10-Middleware-Proxy/README.md) | Next ➡️: [12 Built-in Optimizations](../12-Built-in-Optimizations/README.md)
