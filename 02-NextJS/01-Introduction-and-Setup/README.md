# 🚀 Introduction to Next.js and Setup

> **TL;DR:** Next.js is a React **framework**. React gives you components; Next.js adds routing, server rendering, data fetching, a backend (API + Server Actions), bundling and optimisation, so you can ship a full-stack app with no extra setup. Current version: **Next.js 16** (App Router, Turbopack, React 19).

## 🧠 Concept

React on its own is a **UI library**. It renders components and stops there. A plain React app (Vite or the old Create React App) is a **Single Page Application (SPA)**: the server sends an almost empty `index.html` plus a large JS bundle, and the browser builds the whole page.

Think of React as an **engine** and Next.js as the **whole car**: engine, steering (routing), fuel system (data fetching), dashboard (dev tooling) and safety features (optimisations).

### Problems with plain React (CSR SPA) that Next.js solves

| Problem in plain React | Why it hurts | Next.js solution |
| --- | --- | --- |
| Empty HTML on first load | Crawlers and link previews see `<div id="root">` with no content, so SEO is poor | HTML is rendered on the server (SSR/SSG) |
| Slow first paint | The browser must download, parse and run JS before showing anything | HTML arrives ready to show; JS hydrates it afterwards |
| No built-in routing | You install and configure `react-router` yourself | File-system routing (`app/blog/page.tsx` serves `/blog`) |
| No backend | You need a separate Express server for APIs and secrets | Route Handlers, Server Actions, Server Components |
| Lots of config | Webpack/Babel, code splitting, image optimisation, env vars | Zero-config: Turbopack, automatic code splitting, `next/image`, `next/font` |
| Data-fetching waterfalls | `useEffect` runs fetches only after render, on the client | Fetch on the server, close to the database |

### Key features overview

- **App Router** (`app/`): file-based routing with layouts, loading and error states, and nested routes.
- **React Server Components (RSC)**: components that run only on the server and send zero JS to the browser.
- **Rendering options**: static (SSG), dynamic (SSR), ISR, streaming, and Partial Prerendering via Cache Components.
- **Data and caching**: `fetch` in async components, the `'use cache'` directive, and revalidation APIs.
- **Server Actions**: call server functions straight from forms and buttons.
- **Route Handlers**: build REST endpoints in `app/api/.../route.ts`.
- **Proxy** (`proxy.ts`, called `middleware.ts` before v16): runs code before a request is handled.
- **Optimisations**: `next/image`, `next/font`, `next/script`, Metadata API for SEO.

### App Router vs Pages Router (high level)

| | App Router (`app/`) | Pages Router (`pages/`) |
| --- | --- | --- |
| Status | Default and recommended since Next 13.4 | Legacy, still supported |
| Components | Server Components by default | Client components only |
| Data fetching | `async` components + `fetch` / DB calls | `getServerSideProps`, `getStaticProps` |
| Layouts | Nested `layout.tsx` that keep state | `_app.tsx` / `_document.tsx` |
| Mutations | Server Actions | API routes only |
| Streaming | Built in (`loading.tsx`, Suspense) | Limited |

Both routers can live in one project, which helps when migrating, but a URL must not be defined in both.

## ⚙️ How it works / Deep dive

### Request flow: server render, then HTML, then hydration

```text
Browser requests /products
        │
        ▼
Next.js server: renders Server Components and fetches data
        │  produces (1) HTML  (2) RSC Payload (serialised component tree)
        ▼
Browser shows HTML immediately  ← fast first paint, good SEO
        │
        ▼
Browser downloads JS for Client Components only
        │
        ▼
Hydration: React attaches event handlers to the existing HTML
        │
        ▼
Page is interactive. Later navigations are client-side (only the RSC payload is fetched)
```

**Hydration** means React "wakes up" server-rendered HTML. It does not rebuild the DOM. It walks the HTML and attaches state and event listeners. If server and client output differ, you get a **hydration mismatch** warning.

### Turbopack

- A Rust-based bundler and Webpack's successor, written by the Next.js team.
- **Next 15**: stable for `next dev`.
- **Next 16**: the **default** for both `next dev` and `next build`. Fast refresh and builds are much faster.
- To fall back to Webpack (for example, for a custom webpack config), run `next build --webpack`.

### Version notes (Next 16)

- Node.js **20.9+** is required.
- `next lint` was **removed**; run ESLint (or Biome) directly.
- React Compiler support is stable: set `reactCompiler: true` in `next.config.ts`.
- `middleware.ts` is renamed to `proxy.ts`.

## 💻 Code examples

### Create a project

```bash
npx create-next-app@latest my-app
# Prompts: TypeScript? ESLint/Biome? Tailwind? src/ dir? App Router? Turbopack? import alias?
cd my-app
npm run dev        # http://localhost:3000
```

### Project structure

```text
my-app/
├── app/                  # App Router: routes, layouts, pages
│   ├── layout.tsx        # Root layout (required, contains <html> and <body>)
│   ├── page.tsx          # Route "/"
│   ├── globals.css
│   └── favicon.ico
├── public/               # Static files served from "/" (e.g. /logo.png)
├── next.config.ts        # Next.js configuration
├── tsconfig.json         # TS config (with "@/*" path alias)
├── package.json
├── proxy.ts              # (optional) runs before requests, formerly middleware.ts
├── .env.local            # Secrets (not committed). Only NEXT_PUBLIC_* reaches the browser
└── eslint.config.mjs
```

### Scripts

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint ."
  }
}
```

| Command | What it does |
| --- | --- |
| `next dev` | Dev server with hot reload (Turbopack) |
| `next build` | Production build; pre-renders static pages and prints a route summary (○ static, ƒ dynamic) |
| `next start` | Serves the production build (run `build` first) |

### Minimal config and first page

```ts
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  reactCompiler: true,     // auto-memoisation (stable in 16)
  cacheComponents: true,   // opt in to 'use cache' / Partial Prerendering (Next 16)
};

export default nextConfig;
```

```tsx
// app/page.tsx: a Server Component by default
export default async function HomePage() {
  const res = await fetch('https://api.example.com/stats');
  const stats = await res.json();
  return <h1>Users: {stats.users}</h1>; // HTML comes from the server
}
```

## ⚠️ Common mistakes & gotchas

- Calling Next.js "a library": it is a **framework** built on top of the React library.
- Putting secrets in `NEXT_PUBLIC_*` env vars. These are inlined into the browser bundle.
- Thinking SSR means no JS. Client Components still ship JS and hydrate.
- Following Next 13/14 tutorials blindly: they show `middleware.ts`, sync `params`, and cached-by-default `fetch`.
- Running `next start` without running `next build` first.

## 🎤 Say this in the interview

> "Next.js is a full-stack React framework. It adds file-system routing, server rendering (SSR, SSG, ISR, streaming), React Server Components, Server Actions and built-in optimisations. That fixes plain React's SEO and first-load issues and removes the setup work. In Next 16 the App Router and Turbopack are the defaults, and caching is opt-in through Cache Components."

## ❓ Likely interview questions

**What is the difference between React and Next.js?**

React is a UI library for building components. Next.js is a framework on top of React that adds routing, server-side rendering, data fetching, backend capabilities and build tooling.

**Why is a plain React SPA bad for SEO?**

The initial HTML is basically an empty `div`. Content only appears after JS runs, so crawlers and social previews may see nothing.

**What is hydration?**

React attaches event listeners and state to HTML that the server already rendered, which makes it interactive without re-creating the DOM.

**App Router or Pages Router: which and why?**

App Router. It is the default, supports Server Components, nested layouts, streaming and Server Actions. Pages Router is legacy.

**What is Turbopack?**

A Rust-based incremental bundler. It has been the default for dev and build since Next 16, and replaces Webpack.

**What changed in Next 16 at a glance?**

Turbopack by default, `proxy.ts` replacing `middleware.ts`, Cache Components (`'use cache'`), new caching APIs (`updateTag`, `refresh`, and `revalidateTag` with a profile), stable React Compiler support, async-only request APIs, and `next lint` removed.

---

⬅️ Prev: [Next.js Overview](../README.md) | Next ➡️: [App Router and Routing](../02-App-Router-and-Routing/README.md)
