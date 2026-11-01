# 🧱 Layouts, Templates, Loading and Error UI

> **TL;DR:** Special files in each route segment wrap your page automatically. `layout.tsx` is shared UI that **persists** across navigations, `template.tsx` is a layout that **remounts** on every navigation, `loading.tsx` is an automatic Suspense fallback, `error.tsx` is an automatic error boundary (Client Component), and `not-found.tsx` handles 404s.

## 🧠 Concept

You don't hand-write `<Suspense>` or error boundaries for every page. Next.js reads the special files in a folder and **builds the wrapper tree for you**. Think of a page as a gift inside nested boxes: each folder adds its own boxes (layout, error boundary, loading state) around the content.

### Root layout (required)

`app/layout.tsx` is the top-level wrapper for the whole app. It **must** render `<html>` and `<body>`, because Next.js doesn't generate them for you. It is a Server Component by default and is the place for global CSS, fonts, providers and `metadata`.

### Nested layouts

Each folder can have its own `layout.tsx`. Layouts **nest**: the parent layout's `children` is the child layout, whose `children` is the page.

```text
URL: /dashboard/settings

app/layout.tsx              RootLayout
└── app/dashboard/layout.tsx       DashboardLayout (sidebar)
    └── app/dashboard/settings/page.tsx   SettingsPage

Rendered:
<RootLayout>
  <DashboardLayout>
    <SettingsPage />
  </DashboardLayout>
</RootLayout>
```

### Layouts persist (partial rendering)

When you navigate from `/dashboard/settings` to `/dashboard/billing`, only the **page** changes. `DashboardLayout` **is not re-rendered or remounted**, so its state (an open sidebar, a search input, a scroll position) survives. That is called **partial rendering**, and it makes navigation fast because the server only sends the changed segment.

Consequences:

- Layouts **don't receive `searchParams`** and don't re-render on navigation, so they can't show the current path. Use a Client Component with `usePathname()` for active links.
- A layout can't pass data down to its `children` as props. Fetch the data in the page (duplicate `fetch` calls are memoised, so that is cheap).

## ⚙️ How it works / Deep dive

### `template.tsx`: a layout that remounts

Same shape as a layout (`children` prop), but it gets a **new instance on every navigation**: state resets, effects re-run, DOM is recreated.

| | `layout.tsx` | `template.tsx` |
| --- | --- | --- |
| On navigation between children | Persists, keeps state | Remounts, resets state |
| `useEffect` | Runs once | Runs on every navigation |
| Use for | Navbars, sidebars, shells | Enter/exit animations, per-page analytics, resetting forms |

### `loading.tsx`: instant loading state

`loading.tsx` wraps the segment's `page` (and its children) in a **React `<Suspense>` boundary** automatically. While the async page is fetching data, the fallback shows **immediately**, because it is prefetched and part of the static shell. Navigation feels instant, the layout stays interactive, and the page content **streams in** when it is ready.

```text
<Layout>
  <Suspense fallback={<Loading />}>
    <Page />        ← async, awaiting data
  </Suspense>
</Layout>
```

Use your own `<Suspense>` boundaries inside the page for finer-grained streaming (for example, one per widget).

### `error.tsx`: error boundary per segment

- **Must be a Client Component** (`'use client'`), because React error boundaries rely on client-side state and the `reset` function.
- Receives `error` (with an optional `digest`, a hash matching server logs; production server errors hide the real message for safety) and `reset()`, which re-renders the segment to try again.
- Catches errors in its **segment's children**, **not** in the `layout.tsx` of the same folder (the layout sits outside the boundary). To catch layout errors, put `error.tsx` in the **parent** folder.
- The rest of the app (parent layouts, sibling slots) keeps working.

### `global-error.tsx`

Catches errors in the **root layout** itself. It **replaces** the root layout when it is active, so it must render its own `<html>` and `<body>`. It is active in production only (dev shows the error overlay).

### `not-found.tsx` and `notFound()`

- Calling `notFound()` (from `next/navigation`) throws a special error that renders the closest `not-found.tsx` and returns a **404 status**.
- `app/not-found.tsx` also handles **any unmatched URL** in the app.
- Use it when a dynamic resource doesn't exist (`/posts/999`).

### Component hierarchy order

Within one segment, Next.js composes the files in this order (outermost first):

```text
<Layout>
  <Template>
    <ErrorBoundary fallback={<Error />}>          ← error.tsx
      <Suspense fallback={<Loading />}>          ← loading.tsx
        <NotFoundBoundary fallback={<NotFound />}>  ← not-found.tsx
          <Page />
        </NotFoundBoundary>
      </Suspense>
    </ErrorBoundary>
  </Template>
</Layout>
```

Memory hook: **"Lazy Turtles Eat Lettuce, Not Pizza"**: Layout, Template, Error, Loading, Not-found, Page.

Because `error.tsx` is outside `loading.tsx`, an error thrown while loading data is caught. Because the layout is outside everything, layout errors bubble to the parent's `error.tsx`.

## 💻 Code examples

### Root layout and nested layout

```tsx
// app/layout.tsx: required, must include <html> and <body>
import './globals.css';
import type { Metadata } from 'next';

export const metadata: Metadata = { title: 'My App', description: 'Demo' };

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

```tsx
// app/dashboard/layout.tsx: persists while moving between dashboard pages
import { Sidebar } from './_components/sidebar';

export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex">
      <Sidebar />           {/* state survives navigation */}
      <main>{children}</main>
    </div>
  );
}
```

### Loading, error, not-found

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <div className="skeleton">Loading dashboard…</div>;
}
```

```tsx
// app/dashboard/error.tsx: must be a Client Component
'use client';
import { useEffect } from 'react';

export default function Error({
  error, reset,
}: { error: Error & { digest?: string }; reset: () => void }) {
  useEffect(() => { console.error(error); }, [error]); // log to Sentry etc.
  return (
    <div>
      <h2>Something went wrong</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

```tsx
// app/posts/[id]/page.tsx
import { notFound } from 'next/navigation';

export default async function Post({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const post = await db.post.findUnique({ where: { id } });
  if (!post) notFound();               // renders closest not-found.tsx with 404
  return <h1>{post.title}</h1>;
}
```

```tsx
// app/global-error.tsx: replaces the root layout, needs html/body
'use client';
export default function GlobalError({ reset }: { error: Error; reset: () => void }) {
  return (
    <html><body>
      <h2>App crashed</h2>
      <button onClick={() => reset()}>Reload</button>
    </body></html>
  );
}
```

## ⚠️ Common mistakes & gotchas

- Forgetting `'use client'` in `error.tsx`, which causes a build error.
- Expecting `error.tsx` to catch errors in the **same folder's** layout. It can't; put it one level up.
- Omitting `<html>` and `<body>` in the root layout or in `global-error.tsx`.
- Trying to read `searchParams` or the pathname in a layout and expecting it to update. Layouts don't re-render on navigation.
- Using `template.tsx` when you only need a layout, which throws away state on every navigation.
- One `loading.tsx` for a whole page with independent slow parts. Add nested `<Suspense>` boundaries so fast parts aren't blocked.
- Throwing errors for **expected** cases (validation failures). Return them as values; error boundaries are for unexpected errors.

## 🎤 Say this in the interview

> "Layouts are shared UI that persist and don't re-render on navigation, so state survives. Templates remount every time. `loading.tsx` is an automatic Suspense boundary that gives instant loading states and streaming, and `error.tsx` is an automatic client-side error boundary with `reset`. The nesting order is layout, template, error, loading, not-found, page."

## ❓ Likely interview questions

**Why must the root layout include `<html>` and `<body>`?**

Next.js doesn't generate them. The root layout is the document shell for every page.

**Layout vs template?**

A layout persists across navigations and keeps state. A template creates a new instance on every navigation, which is useful for animations or resetting state.

**How does `loading.tsx` work?**

It wraps the page in a `<Suspense>` boundary. The fallback shows immediately (it is prefetched) while server data loads, then the content streams in.

**Why is `error.tsx` a Client Component?**

Error boundaries are a client-side React feature that needs state and the `reset` callback to recover without a full reload.

**Why doesn't `error.tsx` catch errors from the layout in the same folder?**

The layout wraps the error boundary, so it sits outside it. Use the parent segment's `error.tsx` or `global-error.tsx`.

**How do you return a 404 for a missing record?**

Call `notFound()` from `next/navigation`. It renders the nearest `not-found.tsx` with a 404 status.

**Can a layout pass data to its page?**

Not through props. Fetch in both places; request memoisation and `React.cache` dedupe the calls.

---

⬅️ Prev: [App Router and Routing](../02-App-Router-and-Routing/README.md) | Next ➡️: [Server vs Client Components](../04-Server-vs-Client-Components/README.md)
