# 💧 Hydration & Common Errors

> **TL;DR:** Hydration is React "attaching" to HTML the server already rendered, by re-running the components in the browser and wiring up event handlers. If the browser's first render differs from the server HTML, you get a **hydration mismatch**. Most other Next.js errors come from mixing up the server/client boundary.

## 🧠 Concept

Picture the server shipping a **printed photo** of your UI (HTML). It shows up instantly, but you can't click anything. Hydration is React laying a **transparent touch-screen** over the photo. To line up correctly, React renders the same tree in the browser and expects it to **match exactly**. If the photo says "10:00:01" and the browser renders "10:00:03", React can't tell which is right. It warns, and it may throw away the server HTML for that subtree and re-render on the client, which is slower and makes the screen flicker.

### Hydration step by step

1. The server renders components to HTML (plus the RSC payload) and streams it.
2. The browser paints the HTML right away (fast FCP/LCP), and it isn't interactive yet.
3. The JS bundles for **Client Components** download.
4. React calls `hydrateRoot`, renders the client tree in memory and compares it with the existing DOM.
5. It attaches event listeners and makes the page interactive. Effects (`useEffect`) run **after** hydration.

Only Client Components hydrate. Server Components have no JS, so nothing needs to match on the client for them.

## ⚙️ How it works / Deep dive

### Why mismatches happen

| Cause | Example | Why it differs |
| --- | --- | --- |
| Time/randomness | `new Date().toLocaleTimeString()`, `Math.random()`, `uuid()` | Different values on server and client |
| Browser-only APIs | `typeof window !== 'undefined' ? A : B`, `localStorage.getItem` | The server always takes the "no window" branch |
| Locale/timezone formatting | `toLocaleDateString()`, `Intl.NumberFormat()` | Server TZ/locale ≠ user's |
| Invalid HTML nesting | `<p><div/></p>`, `<a>` inside `<a>`, `<tr>` without `<tbody>` | The browser "fixes" the DOM, so it no longer matches |
| Browser extensions | Grammarly or password managers inject attributes/elements | The DOM is changed before React hydrates |
| Inconsistent data | The client re-fetches different data on first render | Different props |

React 19 gives much better error messages (a diff of the mismatched text) and tolerates extension-injected nodes in `<head>`/`<body>` better.

### Fixes

1. **Render browser-only values after mount**: `useEffect` + a `mounted` flag, or `useSyncExternalStore` with a server snapshot.
2. **Skip SSR for that component**: `dynamic(() => import('./Map'), { ssr: false })` (inside a Client Component).
3. **`suppressHydrationWarning`** on a single element whose text legitimately differs (a timestamp, or the `<html>` tag when a theme script sets `class`). It only works **one level deep** and doesn't fix logic bugs.
4. **Make data consistent**: compute the value on the server and pass it as a prop, or format dates with a fixed `timeZone`/locale.
5. **Fix the HTML nesting.**

## 💻 Code examples

### Mounted flag

```tsx
'use client';
import { useEffect, useState } from 'react';

export function LocalTime() {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []); // runs only in the browser, after hydration

  if (!mounted) return <span>--:--</span>; // same output on server and first client render
  return <span>{new Date().toLocaleTimeString()}</span>;
}
```

### Reading localStorage safely

```tsx
'use client';
import { useEffect, useState } from 'react';

export function useStoredTheme() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light'); // server-safe default
  useEffect(() => {
    const saved = localStorage.getItem('theme');
    if (saved === 'dark' || saved === 'light') setTheme(saved);
  }, []);
  return [theme, setTheme] as const;
}
```

### Legit difference

```tsx
// next-themes style: a script sets class="dark" on <html> before React hydrates
<html lang="en" suppressHydrationWarning>
  <body>{children}</body>
</html>
```

### Consistent date formatting

```tsx
const fmt = new Intl.DateTimeFormat('en-US', { dateStyle: 'medium', timeZone: 'UTC' });
<time dateTime={post.date}>{fmt.format(new Date(post.date))}</time>;
```

## 🧯 Other common Next.js errors

### Hooks in a Server Component

```text
You're importing a component that needs useState. This React Hook only works
in a Client Component. To fix, mark the file (or its parent) with the "use client" directive.
```

The cause is that files in `app/` are Server Components by default. Fix it by adding `'use client'` at the top of the **smallest interactive component**, not the whole page.

### Event handlers passed to Client Component props

```text
Error: Event handlers cannot be passed to Client Component props.
  <button onClick={function onClick} ...>
```

A Server Component tried to pass a function (`onClick`) across the boundary. Functions aren't serializable. Either move the handler into a Client Component, or pass a **Server Action** (a function marked `'use server'`), which *is* allowed.

### Non-serializable props

Props from Server to Client Components must be serializable: primitives, plain objects/arrays, `Date`, `Map`/`Set`, Promises, JSX and Server Actions. **Class instances**, functions, Symbols and ORM objects with methods fail. Map them to plain objects first.

### `window is not defined` / `document is not defined`

Client Components **also render on the server** (SSR). Touching `window` at module top level or during render crashes there. Move it into `useEffect` or an event handler, or load the library with `dynamic(..., { ssr: false })`.

### Async params errors after upgrading to 15/16

```text
Error: Route "/blog/[slug]" used `params.slug`. `params` should be awaited before using its properties.
```

In Next 15, `params`, `searchParams`, `cookies()`, `headers()` and `draftMode()` became **async**. Sync access was a warning in 15 and is **removed in 16** (it errors). Fix:

```tsx
// Server Component
export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const cookieStore = await cookies();
  return <h1>{slug}</h1>;
}

// client-page.tsx (separate file): Client Component receiving params as a prop
'use client';
import { use } from 'react';
export function ClientPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = use(params);
  return <p>{slug}</p>;
}
```

Codemod: `npx @next/codemod@latest next-async-request-api .`

### `useSearchParams()` should be wrapped in a suspense boundary

On static pages, `useSearchParams` makes the build fail ("Missing Suspense boundary with useSearchParams"). Wrap the component that uses it in `<Suspense fallback={...}>`.

### Importing server code into the client

If a Client Component imports a file that uses DB secrets, it can leak secrets or break the build. Add `import 'server-only'` to server modules so this fails loudly at build time.

## ⚠️ Common mistakes & gotchas

- Fixing every mismatch with `suppressHydrationWarning`. It hides bugs and only covers one level.
- Putting `'use client'` on the root layout to "fix" errors, which throws away all the Server Component benefits.
- Assuming `'use client'` means "not rendered on the server". It still SSRs.
- `if (typeof window !== 'undefined')` inside render, which is a guaranteed mismatch. Use an effect instead.
- Debugging mismatches without first disabling browser extensions (try an incognito window).

## 🎤 Say this in the interview

> "Hydration is React rendering the client tree in the browser, matching it against the server HTML and attaching event handlers. Mismatches come from anything non-deterministic between server and client: dates, random values, `window` or `localStorage` checks, locale formatting, invalid HTML or extensions. I fix them by deferring browser-only values to `useEffect`, using `ssr: false` for browser-only widgets, keeping data consistent, and using `suppressHydrationWarning` only for legitimate one-off differences."

## ❓ Likely interview questions

**What is hydration?**

Making server-rendered HTML interactive. React re-renders the Client Components in the browser, reconciles them with the existing DOM and attaches listeners.

**What causes hydration mismatch errors?**

Output that differs between server and first client render, such as `Date`/`random`, browser APIs, locale/TZ, invalid nesting or extensions.

**How do you render something that depends on localStorage?**

Render a neutral default first, then read `localStorage` in `useEffect` (or use `ssr: false`).

**Does 'use client' disable SSR?**

No. Client Components are still pre-rendered to HTML on the server and then hydrated.

**Why can't I pass onClick from a Server Component?**

Functions aren't serializable across the RSC boundary. Only Server Actions can be passed.

**What's the fix for "params should be awaited"?**

Make the component `async` and `await params` (or `use(params)` in a Client Component). It's required in Next 16.

**What does suppressHydrationWarning do?**

It silences mismatch warnings for one element's attributes/text. It's meant for unavoidable differences like timestamps or theme classes.

---

⬅️ Prev: [14 Pages Router (Legacy)](../14-Pages-Router-Legacy/README.md) | Next ➡️: [16 Auth, Env, Styling & Deployment](../16-Auth-Env-Styling-Deployment/README.md)
