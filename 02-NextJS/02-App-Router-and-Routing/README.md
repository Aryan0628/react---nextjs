# 🗺️ App Router and Routing

> **TL;DR:** In the App Router, **folders are URL segments** and a `page.tsx` file makes a folder publicly reachable. Square brackets make dynamic segments, parentheses group folders without changing the URL, `@slot` folders render parallel UI, and `(.)` folders intercept routes (for example, modals).

## 🧠 Concept

**File-system routing** means you never write a route table. The folder tree inside `app/` *is* the route table.

- A **folder** = a URL segment (`app/blog` → `/blog`).
- A **`page.tsx`** inside it = the UI for that URL. Without a `page.tsx`, the folder is **not** a route. That lets you safely keep components, utils and tests next to routes (**colocation**).
- Other special files (`layout.tsx`, `loading.tsx`, `error.tsx`, `route.ts`) add behaviour to the segment.

Analogy: the `app/` folder is a building. Folders are rooms, and `page.tsx` is the door that opens a room to visitors.

### Folder to URL cheat-sheet

| Folder / file | URL it matches | Notes |
| --- | --- | --- |
| `app/page.tsx` | `/` | Home |
| `app/about/page.tsx` | `/about` | Static route |
| `app/blog/[slug]/page.tsx` | `/blog/hello` | Dynamic, `params.slug = 'hello'` |
| `app/shop/[...slug]/page.tsx` | `/shop/a`, `/shop/a/b` (not `/shop`) | Catch-all, `slug = ['a','b']` |
| `app/docs/[[...slug]]/page.tsx` | `/docs`, `/docs/a/b` | Optional catch-all, `slug` may be `undefined` |
| `app/(marketing)/pricing/page.tsx` | `/pricing` | Route group, not in URL |
| `app/_lib/utils.ts` | — | Private folder, never routable |
| `app/dashboard/@analytics/page.tsx` | `/dashboard` | Parallel slot, passed as a prop to the layout |
| `app/feed/(..)photo/[id]/page.tsx` | `/photo/1` (intercepted) | Intercepting route |
| `app/api/users/route.ts` | `/api/users` | API endpoint (Route Handler) |

## ⚙️ How it works / Deep dive

### Dynamic segments and async `params`

`[id]` captures one segment. The value arrives in the `params` prop. **Since Next 15, `params` and `searchParams` are Promises**, and Next 16 removed the temporary synchronous access, so you **must `await`** them in Server Components (or unwrap them with React's `use()` in Client Components). Why: it lets Next.js start rendering before request-specific data is known, which enables static shells and streaming.

### `searchParams`

`searchParams` holds the query string (`?page=2&sort=asc`). It exists **only on `page.tsx`**, not on layouts. Reading it makes the page **dynamic**, because the value is only known at request time. In Client Components, use the `useSearchParams()` hook.

### Route groups `(group)`

Wrapping a folder name in parentheses organises routes **without affecting the URL**. Uses:

- Separate sections with different layouts, for example `(marketing)` vs `(app)`, each with its own `layout.tsx`.
- Multiple root layouts: remove the top-level `app/layout.tsx` and give each group its own `<html>`.
- Avoid having two groups resolve to the same URL (`(a)/about` and `(b)/about` is a build error).

### Private folders `_folder`

An underscore prefix opts the folder and all its children out of routing. Use it for `_components`, `_lib` and similar folders to show clearly that they are internal.

### Parallel routes `@slot`

Render **several pages in the same layout at once**, each loading, streaming and erroring on its own. Every `@folder` becomes a prop on the parent layout (`children` is an implicit slot).

- Slots **do not** affect the URL.
- **`default.tsx`** is the fallback a slot renders when Next.js can't work out its active state after a **hard refresh** (for example, the URL matches a sub-page that one slot doesn't have). **Next 16 requires a `default.tsx` for every slot**, or the build fails. Return `null` or call `notFound()` in it.
- Good for dashboards, split views, and conditional UI (for example, render `@admin` vs `@user` based on role).

### Intercepting routes `(.)`, `(..)`, `(..)(..)`, `(...)`

Show a route **inside the current layout** during client-side navigation (soft navigation), while a direct visit or refresh (hard navigation) shows the full page.

| Convention | Matches segment |
| --- | --- |
| `(.)` | Same level |
| `(..)` | One level up |
| `(..)(..)` | Two levels up |
| `(...)` | From the `app` root |

The levels are based on **route segments, not folders**, so `@slot` folders don't count. The classic use is the **photo modal**: click a photo in a feed and it opens in a modal with a shareable URL; refresh and you get the full photo page.

## 💻 Code examples

### Dynamic route with async params and searchParams

```tsx
// app/blog/[slug]/page.tsx
type Props = {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
};

export default async function BlogPost({ params, searchParams }: Props) {
  const { slug } = await params;           // must await (Next 15+, required in 16)
  const { lang = 'en' } = await searchParams;
  const post = await getPost(slug, String(lang));
  return <article><h1>{post.title}</h1></article>;
}
// Next 15.5+ also generates global helpers: PageProps<'/blog/[slug]'>
```

### Catch-all and Client Component access

```tsx
// app/docs/[[...slug]]/page.tsx: matches /docs and /docs/a/b/c
export default async function Docs({ params }: { params: Promise<{ slug?: string[] }> }) {
  const { slug = [] } = await params;
  return <p>Path: {slug.join(' / ') || 'index'}</p>;
}
```

```tsx
'use client';
import { useParams, useSearchParams, usePathname, useRouter } from 'next/navigation';

export function Filters() {
  const { slug } = useParams<{ slug: string }>();
  const searchParams = useSearchParams();
  const router = useRouter();
  const pathname = usePathname();
  const setSort = (s: string) => router.push(`${pathname}?sort=${s}`);
  return <button onClick={() => setSort('asc')}>Sort ({searchParams.get('sort')})</button>;
}
```

### Parallel routes: dashboard

```text
app/dashboard/
├── layout.tsx
├── page.tsx            → children
├── @analytics/
│   ├── page.tsx
│   └── default.tsx     (required in Next 16)
└── @team/
    ├── page.tsx
    └── default.tsx
```

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children, analytics, team,
}: { children: React.ReactNode; analytics: React.ReactNode; team: React.ReactNode }) {
  return (
    <div className="grid grid-cols-2">
      <main>{children}</main>
      <section>{analytics}</section>  {/* streams and errors independently */}
      <section>{team}</section>
    </div>
  );
}
// app/dashboard/@analytics/default.tsx → export default function Default() { return null; }
```

### Intercepting routes: photo modal

```text
app/
├── layout.tsx            // renders {children} and {modal}
├── @modal/
│   ├── default.tsx       // return null (no modal by default)
│   └── (.)photo/[id]/page.tsx   // modal version (soft navigation)
├── photo/[id]/page.tsx   // full page (hard navigation / refresh)
└── page.tsx              // feed with <Link href="/photo/1">
```

```tsx
// app/@modal/(.)photo/[id]/page.tsx
import { Modal } from '@/app/_components/modal'; // client component using router.back()

export default async function PhotoModal({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  return <Modal><img src={`/photos/${id}.jpg`} alt="" /></Modal>;
}
```

## ⚠️ Common mistakes & gotchas

- Forgetting `page.tsx`: the folder exists but returns a 404.
- Destructuring `params` synchronously (`params.id`). In Next 16 this is an error; `await` it.
- Expecting `searchParams` in a `layout.tsx`. It is only passed to pages.
- `[...slug]` does **not** match the base path; use `[[...slug]]` for that.
- Missing `default.tsx` in parallel slots, which causes build errors in Next 16 and 404s on refresh in older versions.
- Counting `@slot` folders when picking `(..)` levels.
- Using `<a>` instead of `<Link>`: you lose client-side navigation and prefetching.

## 🎤 Say this in the interview

> "The App Router uses file-system routing: folders are segments and `page.tsx` makes a segment public. That gives safe colocation. I use `[id]` for dynamic segments (awaiting `params`, since it's a Promise from Next 15), route groups to share layouts without touching the URL, and parallel plus intercepting routes for dashboards and URL-addressable modals."

## ❓ Likely interview questions

**What makes a folder a route?**

A `page.tsx` (or `route.ts` for APIs) inside it. Other files in the folder are not publicly routable.

**Difference between `[...slug]` and `[[...slug]]`?**

Catch-all requires at least one segment. Optional catch-all also matches the parent path, with `slug` undefined.

**Why are `params` and `searchParams` Promises now?**

Since Next 15 they are async, so Next.js can prepare and stream work before request data is needed. Next 16 removed sync access. Use `await` or `use()`.

**What are route groups used for?**

Organising routes and giving sections different layouts (even multiple root layouts) without adding a URL segment.

**What are parallel routes and why `default.tsx`?**

`@slot` folders render multiple pages in one layout, each with its own loading and error states. `default.tsx` is the fallback after a hard refresh when a slot has no matching segment; it is mandatory in Next 16.

**Explain intercepting routes with an example.**

Clicking a photo in a feed opens `/photo/1` in a modal (intercepted with `(.)photo`). Refreshing or sharing the URL renders the full photo page.

**How do you read route params in a Client Component?**

Use `useParams()`, `useSearchParams()` and `usePathname()` from `next/navigation`, or unwrap a passed `params` promise with `use()`.

---

⬅️ Prev: [Introduction and Setup](../01-Introduction-and-Setup/README.md) | Next ➡️: [Layouts, Templates, Loading and Error](../03-Layouts-Templates-Loading-Error/README.md)
