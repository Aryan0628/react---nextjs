# 🎯 Next.js Interview Q&A (Rapid Fire)

> **How to use:** cover the answer, say yours out loud, then compare. Answers are short on purpose. Expand with an example in the real interview. Versions: **React 19, Next.js 16** (App Router).

## 🟢 Basics

**1. What is Next.js and why use it over plain React?**

A React framework with routing, server rendering (SSR/SSG/streaming), Server Components, data fetching, caching, image/font optimization and a bundler built in. React alone is just the UI library.

**2. What is the App Router vs the Pages Router?**

App Router (`app/`, Next 13+) is built on React Server Components, nested layouts and streaming, and it's the default. Pages Router (`pages/`) is the older model with `getStaticProps`/`getServerSideProps`, still supported.

**3. What's new in Next.js 16?**

Turbopack is the default bundler, `middleware.ts` became `proxy.ts`, Cache Components (`'use cache'`, `cacheLife`, `cacheTag`) deliver PPR, there are new `updateTag()`/`refresh()` and `revalidateTag(tag, profile)` APIs, React Compiler support is stable, sync `params`/`cookies()` access was removed, and `next lint` was removed.

**4. What did Next.js 15 change that people still trip on?**

`params`, `searchParams`, `cookies()` and `headers()` became async, and `fetch` and GET Route Handlers are no longer cached by default.

**5. What special files exist in an app route segment?**

`page`, `layout`, `template`, `loading`, `error`, `not-found`, `route`, `default`, plus `global-error` at the root.

## 🗺️ Routing

**6. How do you create dynamic routes?**

Use folders like `[id]`, `[...slug]` (catch-all) or `[[...slug]]` (optional catch-all). Read them via the `params` Promise on the server or `useParams()` on the client.

**7. What are route groups?**

`(folder)` organizes routes or gives them different layouts without affecting the URL.

**8. What are parallel and intercepting routes?**

`@slot` folders render several pages in one layout at once (dashboards). `(.)photo` intercepts a route to show it in a modal while keeping a shareable URL.

**9. layout vs template?**

A layout persists across navigations and keeps its state. A template re-mounts on every navigation.

**10. How do you navigate from code?**

`useRouter()` from `next/navigation` (`push`, `replace`, `refresh`, `back`), or `redirect()` on the server.

**11. Redirect vs rewrite?**

A redirect changes the URL with a 3xx. A rewrite serves other content under the same URL.

**12. What does proxy.ts do?**

It runs before routing for redirects, rewrites, headers and optimistic auth. Keep it light, and never make it the only auth check.

## 🧩 Server vs Client Components

**13. What are React Server Components?**

Components that render only on the server, can be `async`, access the DB and secrets directly, and send zero JS to the browser.

**14. When do you need 'use client'?**

For state, effects, event handlers, browser APIs, or libraries that use context or hooks.

**15. Does 'use client' mean the component isn't server-rendered?**

No. It's still SSR'd to HTML and then hydrated. The directive marks the boundary where client JS begins.

**16. Can a Client Component import a Server Component?**

Not directly. Pass the Server Component as `children` or another prop from a Server parent instead.

**17. What props can cross the server-to-client boundary?**

Serializable values: primitives, plain objects/arrays, Dates, Maps/Sets, Promises, JSX and Server Actions. Not functions or class instances.

**18. How do you share context like a theme or query client?**

Make a `'use client'` `Providers` component and wrap `children` with it in the root layout.

**19. What is hydration and what causes mismatches?**

React attaches to the server HTML by re-rendering in the browser. Mismatches come from Date/random values, `window`/`localStorage`, locale formatting, invalid HTML or extensions.

## 🔄 Rendering & Data

**20. Explain SSG, SSR, ISR, CSR and PPR.**

SSG is built once. SSR renders per request. ISR is static and regenerated in the background. CSR renders in the browser. PPR is a static shell with dynamic holes streamed in.

**21. How do you fetch data in the App Router?**

Use `await` directly in an `async` Server Component (fetch, ORM or SDK). There's no `useEffect` and no API hop.

**22. What makes a route dynamic?**

Reading request-time data (`cookies()`, `headers()`, `searchParams`, `connection()`) or uncached data. With Cache Components, uncached data must sit inside `<Suspense>` or be marked `'use cache'`.

**23. What is streaming?**

Sending HTML in chunks. `loading.tsx` or `<Suspense>` shows a fallback while slow parts resolve, which improves TTFB and LCP.

**24. How do you avoid request waterfalls?**

Start the fetches in parallel (`Promise.all`), or preload them and pass Promises down, and split slow parts into separate Suspense boundaries.

**25. What does generateStaticParams do?**

It pre-renders dynamic routes at build time (it replaced `getStaticPaths`).

## 🗄️ Caching

**26. What caching layers exist?**

Request memoization (dedupes within one render), the Data Cache / `'use cache'` (persists across requests), Full Route Cache (prerendered output) and the client Router Cache.

**27. What is 'use cache'?**

A directive (with `cacheComponents: true`) that caches a function's or component's output. Set lifetime with `cacheLife('hours')` and label it for invalidation with `cacheTag('posts')`.

**28. revalidateTag vs updateTag vs refresh?**

`revalidateTag(tag, 'max')` marks data stale (stale-while-revalidate). `updateTag(tag)` (Server Actions only) expires it right away for read-your-writes. `refresh()` re-renders uncached data on the client.

**29. revalidatePath vs revalidateTag?**

Invalidate by route vs by data label. Tags are more precise when many pages share the same data.

**30. How did fetch caching change in Next 15?**

`fetch` became uncached by default (before that it was `force-cache`). Opt in with `cache: 'force-cache'`, `next: { revalidate }` or `'use cache'`.

## 📨 Server Actions & APIs

**31. What are Server Actions?**

Async functions marked `'use server'` that you call from forms or Client Components. Next turns them into POST endpoints, and they work without JS when used in a `<form action>`.

**32. Are Server Actions secure by default?**

No. They are public endpoints. Validate input (Zod) and check auth and authorization inside every action.

**33. Which hooks pair with Server Actions?**

`useActionState` (state, pending, errors), `useFormStatus` (pending in a submit button), `useOptimistic` (instant UI).

**34. Route Handlers vs Server Actions?**

Route Handlers (`route.ts`) are HTTP APIs for webhooks, mobile clients and third parties. Server Actions handle mutations from your own UI.

**35. How do you show an error or redirect after a mutation?**

Return an error object for `useActionState`. On success call `updateTag`/`revalidatePath`, then `redirect()` outside `try/catch`.

## 🚀 Optimization & SEO

**36. Why use next/image?**

Automatic resizing and srcset, WebP/AVIF, lazy loading, no CLS, and caching. Use `preload` (formerly `priority`) for the LCP image and allow remote hosts via `remotePatterns`.

**37. How does next/font help?**

It self-hosts fonts at build time and uses size-adjusted fallbacks, so there's zero layout shift and no requests to Google.

**38. How do you lazy-load a heavy component?**

`next/dynamic(() => import('./Chart'))`, with `ssr: false` only from a Client Component.

**39. How do you add SEO metadata?**

Export `metadata` or `generateMetadata` from Server Component layouts and pages, plus `sitemap.ts`, `robots.ts`, `opengraph-image.tsx` and JSON-LD.

**40. What are the Core Web Vitals?**

LCP ≤ 2.5s, INP ≤ 200ms, CLS ≤ 0.1. Next helps with SSR/streaming, image and font optimization, and less client JS.

## 🛡️ Auth, Env & Deployment

**41. Where do you check auth?**

Optimistic redirects in the proxy, real checks in a Data Access Layer used by pages, Server Actions and Route Handlers. Keep sessions in httpOnly cookies.

**42. What's special about NEXT_PUBLIC_ variables?**

They're inlined into the client bundle at build time, so they're public and need a rebuild to change.

**43. How do you self-host Next.js?**

`next build && next start`, or Docker with `output: 'standalone'`. With several instances, configure a shared cache handler.

**44. What are the limits of output export?**

No server: no Server Actions, proxy, ISR, cookies/headers or default image optimization.

## 🏗️ Scenario & design questions

### Scenario 1: E-commerce product page

**45. How would you build a fast, SEO-friendly product page?**

1. **Route:** `app/products/[slug]/page.tsx` with `generateStaticParams` for the top sellers. The rest render on demand and get cached.
2. **Static shell (cached):** product name, images and description via a `'use cache'` function with `` cacheTag(`product-${slug}`) `` and `cacheLife('hours')`.
3. **Dynamic holes (streamed):** price/stock and personalized recommendations inside `<Suspense>`. With Cache Components this is PPR: an instant shell plus streamed data.
4. **Interactivity:** small Client Components for the image gallery, variant picker and "Add to cart" (a Server Action plus `useOptimistic` for the cart count).
5. **Invalidation:** the CMS/admin webhook calls a Route Handler that runs `revalidateTag('product-123', 'max')`. Admin edits in a Server Action use `updateTag`.
6. **SEO:** `generateMetadata` (deduped with React `cache`), canonical URL, `opengraph-image.tsx`, Product JSON-LD, `sitemap.ts`.
7. **Performance:** `next/image` with `preload` on the hero and correct `sizes`, `next/font`, lazy-loaded reviews via `next/dynamic` or Suspense.

### Scenario 2: Dashboard with authentication

**46. How would you structure an authenticated dashboard?**

1. **Routes:** `app/(auth)/login` and `app/(dashboard)/layout.tsx` with a sidebar. Use parallel routes (`@stats`, `@activity`) so each panel streams independently with its own `loading`/`error`.
2. **Auth:** the proxy redirects requests without a session cookie to `/login`. A `verifySession()` DAL (wrapped in React `cache`, `server-only`) runs in every data function, Server Action and Route Handler. Roles are checked per resource.
3. **Data:** per-user data is dynamic (it reads cookies), so don't share-cache it. Fetch in parallel and stream each widget with Suspense.
4. **Mutations:** Server Actions with Zod validation, `useActionState` for errors, and `refresh()`/`updateTag` so the UI updates.
5. **Client state:** keep it minimal. Store filters in the URL (`useSearchParams` plus `router.replace`) so views are shareable. Use React Query only for real-time polling.
6. **Security:** httpOnly secure cookies, CSRF covered by Server Action origin checks, rate-limit login, and never expose secrets via `NEXT_PUBLIC_`.

### Scenario 3: Blog or marketing site with a CMS

**47. How would you build a content site that editors update often?**

Prerender everything with `generateStaticParams`, cache CMS fetches with `'use cache'` + `cacheTag('posts')`, and have a CMS webhook Route Handler call `revalidateTag('posts', 'max')`. Add `draftMode()` for previews, the Metadata API plus sitemap for SEO, and host on a CDN.

### Scenario 4: The page is slow, what do you check

**48. A Next.js page has a poor LCP and a big bundle. How do you debug it?**

1. Check the `next build` route table: is the page dynamic when it should be static?
2. Look for waterfalls (sequential `await`s), then parallelize and add Suspense.
3. Run the bundle analyzer and look for oversized `'use client'` trees, barrel imports and heavy libraries. Push `'use client'` down to the leaves and lazy-load.
4. Make sure the LCP image uses `next/image` with `preload` and correct `sizes`, and fonts come from `next/font`.
5. Defer third-party scripts with `next/script` `lazyOnload`.
6. Measure with `useReportWebVitals` or Speed Insights before and after.

### Scenario 5: Migrating from the Pages Router

**49. How would you migrate a large Pages Router app?**

Incrementally. Add `app/` next to `pages/`, create the root layout and providers, and move low-risk routes first. Map `getStaticProps` to async Server Components with caching, `getServerSideProps` to dynamic Server Components, and API routes to Route Handlers or Server Actions. Swap `next/router` for `next/navigation` and `next/head` for metadata, and run the async-API codemods.

**50. What would you say is the single biggest mental shift in the App Router?**

The default is the server. Fetch data where it's rendered, send as little JS as possible, and treat caching as explicit (`'use cache'`) instead of implicit.

---

⬅️ Prev: [16 Auth, Env, Styling & Deployment](../16-Auth-Env-Styling-Deployment/README.md) | Next ➡️: [React vs Next.js](../../03-React-vs-NextJS/README.md)
