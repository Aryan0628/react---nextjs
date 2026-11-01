# 📜 Pages Router (Legacy)

> **TL;DR:** Before the App Router (Next 13, 2022), every Next.js app used the `pages/` folder. Data was fetched with `getStaticProps` (SSG), `revalidate` (ISR), `getStaticPaths` and `getServerSideProps` (SSR), and the app was wrapped by `_app` and `_document`. It's still supported in Next 16, still common in production code, and still asked about in interviews. Both routers can live side by side.

## 🧠 Concept

In the Pages Router, **every file in `pages/` is a route** and **every page is a Client Component** (all of its JS ships to the browser). Data loading happens in special **exported functions that run only on the server**. Next calls them, then passes the result to your page as `props`.

The rendering mode is picked **per page** by which function you export:

| Export | Mode | Runs when |
| --- | --- | --- |
| Nothing | Static (automatic static optimization) | Build time |
| `getStaticProps` | SSG | Build time |
| `getStaticProps` + `revalidate` | ISR | Build time, then in the background after N seconds |
| `getServerSideProps` | SSR | On every request |

## ⚙️ How it works / Deep dive

### File-system routing

```text
pages/index.tsx            → /
pages/about.tsx            → /about
pages/blog/[slug].tsx      → /blog/:slug
pages/docs/[...path].tsx   → /docs/a/b/c   (catch-all)
pages/shop/[[...f]].tsx    → /shop and /shop/x (optional catch-all)
pages/api/users.ts         → /api/users    (API route)
pages/_app.tsx             → wraps every page
pages/_document.tsx        → custom <html>/<body> (server only)
pages/404.tsx, 500.tsx     → error pages
```

### `_app.tsx` vs `_document.tsx`

- **`_app`** wraps every page *on both server and client*. It's where global CSS imports, providers (theme, Redux, React Query) and persistent layouts go.
- **`_document`** renders on the **server only**, once. It customizes `<Html>`, `<Head>` and `<body>`, for example a `lang` attribute or font preconnects. No state and no event handlers.

### `getStaticPaths` and fallback

For dynamic SSG routes like `[slug]`, Next must know **which** slugs to pre-build. `getStaticPaths` returns them along with a `fallback` mode for any slug **not** in the list:

| `fallback` | Unknown path at request time | Use when |
| --- | --- | --- |
| `false` | 404 | Small, fixed set of pages (docs) |
| `true` | Serves a **fallback shell** right away (`router.isFallback === true`), builds the page in the background, then swaps in the data | Huge catalogs where a loading skeleton is fine |
| `'blocking'` | Waits on the server, renders full HTML (SSR-like), caches it for later | Huge catalogs where you want no loading state (better for SEO) |

### `getServerSideProps` context

It receives `{ params, query, req, res, resolvedUrl, locale, draftMode }` and returns `{ props }`, `{ redirect: { destination, permanent } }` or `{ notFound: true }`. Because it runs per request, it's slower (no CDN caching by default), but the data is always fresh and you can read cookies from `req`.

### API routes

`pages/api/*.ts` export a Node-style `(req, res) => {}` handler. They're the equivalent of App Router Route Handlers (`app/api/*/route.ts`), which use Web `Request`/`Response` instead.

### Client-side data fetching

For user-specific or frequently changing data that doesn't need SEO, render a static page and fetch in the browser, usually with **SWR** or React Query (caching, revalidate on focus, dedupe).

## 💻 Code examples

### SSG + ISR + getStaticPaths

```tsx
// pages/blog/[slug].tsx
import type { GetStaticPaths, GetStaticProps } from 'next';
import { useRouter } from 'next/router';
import Head from 'next/head';

type Props = { post: { title: string; body: string } };

export const getStaticPaths: GetStaticPaths = async () => {
  const top = await getTopSlugs(); // pre-build only the popular ones
  return { paths: top.map((slug) => ({ params: { slug } })), fallback: 'blocking' };
};

export const getStaticProps: GetStaticProps<Props> = async ({ params }) => {
  const post = await getPost(params!.slug as string);
  if (!post) return { notFound: true };
  return { props: { post }, revalidate: 60 }; // ISR: regenerate at most once a minute
};

export default function Post({ post }: Props) {
  const router = useRouter(); // from next/router in pages/
  if (router.isFallback) return <p>Loading…</p>; // only with fallback: true
  return (
    <>
      <Head><title>{post.title}</title></Head>
      <article>{post.body}</article>
    </>
  );
}
```

### SSR with getServerSideProps

```tsx
// pages/dashboard.tsx
import type { GetServerSideProps } from 'next';

export const getServerSideProps: GetServerSideProps = async ({ req }) => {
  const session = await getSession(req.cookies.session);
  if (!session) return { redirect: { destination: '/login', permanent: false } };
  const stats = await getStats(session.userId);
  return { props: { stats } };
};

export default function Dashboard({ stats }: { stats: { visits: number } }) {
  return <p>Visits: {stats.visits}</p>;
}
```

### `_app.tsx` and an API route

```tsx
// pages/_app.tsx
import type { AppProps } from 'next/app';
import '@/styles/globals.css';

export default function App({ Component, pageProps }: AppProps) {
  return (
    <ThemeProvider>
      <Component {...pageProps} />
    </ThemeProvider>
  );
}
```

```ts
// pages/api/hello.ts
import type { NextApiRequest, NextApiResponse } from 'next';

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'GET') return res.status(405).end();
  res.status(200).json({ message: 'Hello' });
}
```

### Client-side fetching with SWR

```tsx
import useSWR from 'swr';
const fetcher = (url: string) => fetch(url).then((r) => r.json());

export function Profile() {
  const { data, error, isLoading } = useSWR('/api/me', fetcher);
  if (isLoading) return <p>Loading…</p>;
  if (error) return <p>Failed</p>;
  return <p>Hi {data.name}</p>;
}
```

## 🔁 Pages Router → App Router mapping

| Pages Router | App Router |
| --- | --- |
| `pages/about.tsx` | `app/about/page.tsx` |
| `_app.tsx` + `_document.tsx` | Root `app/layout.tsx` (with `<html>`/`<body>`) |
| Per-page layout pattern (`getLayout`) | Nested `layout.tsx` files |
| `getStaticProps` | `async` Server Component (with `'use cache'` or static by default) |
| `revalidate: 60` | `cacheLife` / `export const revalidate = 60` / `fetch(..., { next: { revalidate } })` |
| `getStaticPaths` + `fallback` | `generateStaticParams` + `dynamicParams` |
| `getServerSideProps` | `async` Server Component reading `cookies()`/`headers()` (dynamic) |
| `pages/api/*.ts` | `app/**/route.ts` Route Handlers (or Server Actions for mutations) |
| `next/router` `useRouter` | `next/navigation` `useRouter`, `usePathname`, `useSearchParams` |
| `next/head` | `metadata` / `generateMetadata` |
| `pages/404.tsx`, `_error.tsx` | `not-found.tsx`, `error.tsx`, `global-error.tsx` |
| `router.isFallback` | `loading.tsx` / `<Suspense>` |

## 🧳 Migration notes

- **Incremental adoption:** `app/` and `pages/` can coexist. Move one route at a time. If both define the same path, that's a build error.
- Navigating **between** the two routers is a hard navigation (full reload).
- Start by creating the root layout. Move providers into a `'use client'` `Providers` component that the layout renders.
- Replace data functions with `async` components. Move interactivity into small Client Components.
- Watch out: `next/router` hooks don't work in `app/`. `useRouter().query` becomes `useSearchParams` + `useParams`.
- Next 16 removed AMP support and deprecated `next/legacy/image`. Check the upgrade guide for Pages-specific removals.

## ⚠️ Common mistakes & gotchas

- Using `getStaticProps` in a non-page component. These functions only work in files under `pages/`.
- Passing non-serializable values (Dates, class instances) in `props`. They must be JSON-serializable.
- Using `fallback: true` without handling `router.isFallback`, which crashes on undefined props.
- Calling your own `/api` route from `getServerSideProps`. Call the DB/service directly instead.
- Thinking `getServerSideProps` pages are cached. They're not unless you set `Cache-Control` on `res`.

## 🎤 Say this in the interview

> "In the Pages Router the rendering mode is chosen per page: `getStaticProps` for SSG, adding `revalidate` for ISR, `getStaticPaths` with a fallback mode for dynamic static routes, and `getServerSideProps` for SSR. The App Router replaces those functions with async Server Components and caching directives, and both routers can coexist, so you migrate route by route."

## ❓ Likely interview questions

**What is the difference between getStaticProps and getServerSideProps?**

Build time (plus optional ISR) vs every request. SSG is cached on the CDN and fast; SSR is always fresh but slower.

**Explain fallback false vs true vs blocking.**

Unknown paths 404 vs a loading shell first and then data vs the server waiting and returning full HTML. Both `true` and `blocking` cache the result.

**What is ISR?**

Static pages regenerated in the background after `revalidate` seconds. Users get the stale page while the new one builds (stale-while-revalidate).

**_app vs _document?**

`_app` wraps every page on client and server (providers, global CSS). `_document` is server-only HTML-shell customization.

**Can you use the App and Pages Routers together?**

Yes. Incremental migration is supported. Routes can't conflict, and moving between them is a full page load.

**What replaces getStaticPaths in the App Router?**

`generateStaticParams`, with `dynamicParams = false` to 404 unknown params.

**When would you fetch data on the client with SWR?**

For private, frequently changing, non-SEO data such as dashboards and user widgets.

---

⬅️ Prev: [13 Metadata & SEO](../13-Metadata-and-SEO/README.md) | Next ➡️: [15 Hydration & Common Errors](../15-Hydration-and-Common-Errors/README.md)
