# ⚡ Built-in Optimizations

> **TL;DR:** Next.js ships components that fix the usual performance problems for you. `next/image` handles responsive, lazy, modern-format images without layout shift. `next/font` self-hosts fonts with zero layout shift. `next/script` controls when third-party scripts load. `next/dynamic` lazy-loads client code. On top of that you get automatic code splitting and prefetching. Together they target the Core Web Vitals: **LCP, INP and CLS**.

## 🧠 Concept

Most slow sites fail in the same few ways. Images are huge, fonts swap late and make the text jump, third-party scripts block the main thread, and one giant JS bundle ships everything up front. Instead of making you hand-tune each of these, Next gives you components with good defaults. Think of them as performance guard-rails: use the component and you get the best practice for free.

## ⚙️ How it works / Deep dive

### `next/image`

What it does automatically:

- **Resizes on demand** and serves the right size per device via a generated `srcset`.
- **Converts formats** to WebP by default (add AVIF via `images.formats`).
- **Lazy-loads** by default (`loading="lazy"`), so offscreen images don't compete with the LCP.
- **Prevents CLS**: the required `width`/`height` (or `fill`) reserve space before the image arrives.
- **Caches** optimized images on the server (in Next 16 `minimumCacheTTL` defaults to 4 hours).

Two sizing modes:

| Mode | When | What you pass |
| --- | --- | --- |
| Fixed `width` + `height` | You know the aspect ratio | Intrinsic size (used for the ratio, CSS can still scale it) |
| `fill` | Size depends on the parent (cards, heroes) | Parent needs `position: relative` plus a size, and you pass `sizes` |

**`sizes` explained:** it tells the browser how wide the image will *display* (for example `"(max-width: 768px) 100vw, 33vw"`), so it can pick the smallest `srcset` file that fits. Without `sizes`, a `fill` image is assumed to be `100vw`, so you download a desktop-sized file on mobile.

**LCP image:** mark the hero image so it loads eagerly with high priority. Next 16 deprecates the `priority` prop in favour of **`preload`** (or use `loading="eager"` / `fetchPriority="high"`). Older code uses `priority`.

**Remote images** need an allow-list in `images.remotePatterns`. `images.domains` is deprecated in Next 16. Other Next 16 changes: local images with query strings need `images.localPatterns`, `qualities` defaults to `[75]`, and optimizing images from local IPs is blocked by default.

### `next/font`

- Downloads font files **at build time** and **self-hosts** them with your assets. The browser never calls Google, which helps privacy and saves a DNS lookup and connection.
- Generates a **size-adjusted fallback font**, so swapping from fallback to real font causes no layout shift (CLS ≈ 0).
- `next/font/google` covers any Google Font. `next/font/local` takes your own files. Variable fonts are recommended.
- Expose it as a **CSS variable** to use it with Tailwind.

### `next/script` strategies

| Strategy | Loads | Use for |
| --- | --- | --- |
| `beforeInteractive` | Before hydration, injected into the initial HTML (put it in the root layout) | Bot detection, consent managers that must run first |
| `afterInteractive` (default) | Right after hydration | Analytics, tag managers |
| `lazyOnload` | When the browser is idle | Chat widgets, social embeds |
| `worker` (experimental) | In a web worker via Partytown (Pages Router only) | Heavy third-party scripts off the main thread |

`onLoad`/`onReady`/`onError` callbacks need a Client Component.

### `next/dynamic` and code splitting

- Next **automatically code-splits per route**. Each page loads only its own JS.
- **Server Components ship zero JS**. Only `'use client'` components end up in the bundle.
- `next/dynamic` (`React.lazy` + `Suspense` under the hood) loads a client component **only when it renders**. That's useful for modals, charts, editors and maps.
- `ssr: false` (skip server rendering, for browser-only libraries) is **only allowed inside Client Components**. Using it in a Server Component is a build error.
- `<Link>` **prefetches** routes in the viewport, so the next page's code and data are often ready before the click.

### Other knobs

- **Turbopack** is the default bundler for dev and build in Next 16. Use `--webpack` to opt out.
- **React Compiler** support is stable (`reactCompiler: true`). It memoizes components automatically, which cuts down manual `useMemo`/`useCallback`.
- `experimental.optimizePackageImports` tree-shakes barrel-file libraries such as icon packs.
- **Bundle analysis:** `@next/bundle-analyzer` (webpack, so run `ANALYZE=true next build --webpack`). Newer Next versions also ship an experimental Turbopack analyzer.

### Core Web Vitals and how Next helps

| Metric | Measures | "Good" | How Next helps |
| --- | --- | --- | --- |
| **LCP** (Largest Contentful Paint) | Loading speed of the main content | ≤ 2.5 s | SSR/static HTML, streaming, `next/image` preload + AVIF/WebP, `next/font` |
| **INP** (Interaction to Next Paint, replaced FID in 2024) | Responsiveness | ≤ 200 ms | Less client JS (RSC), code splitting, `next/script` lazy strategies, React Compiler |
| **CLS** (Cumulative Layout Shift) | Visual stability | ≤ 0.1 | Image dimensions, font fallback metrics, Suspense skeletons |

Measure them with `useReportWebVitals` from `next/web-vitals`, Lighthouse, or Vercel Speed Insights.

## 💻 Code examples

### Images

```tsx
import Image from 'next/image';
import hero from '@/public/hero.jpg'; // static import gives width/height + blur placeholder

export function Hero() {
  return <Image src={hero} alt="Mountains" placeholder="blur" preload />; // `priority` in ≤15
}

export function ProductCard({ src }: { src: string }) {
  return (
    <div className="relative aspect-square w-full">
      <Image
        src={src}
        alt="Product"
        fill
        sizes="(max-width: 768px) 100vw, 33vw" // 3-column grid on desktop
        className="object-cover"
      />
    </div>
  );
}
```

```ts
// next.config.ts
const config = {
  images: {
    formats: ['image/avif', 'image/webp'],
    remotePatterns: [{ protocol: 'https', hostname: 'cdn.example.com', pathname: '/products/**' }],
  },
};
export default config;
```

### Fonts with Tailwind (v4)

```tsx
// app/layout.tsx
import { Inter } from 'next/font/google';
import localFont from 'next/font/local';

const inter = Inter({ subsets: ['latin'], variable: '--font-inter', display: 'swap' });
const brand = localFont({ src: './fonts/Brand.woff2', variable: '--font-brand' });

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${inter.variable} ${brand.variable}`}>
      <body className="font-sans">{children}</body>
    </html>
  );
}
```

```css
/* app/globals.css */
@import "tailwindcss";
@theme inline {
  --font-sans: var(--font-inter);
  --font-brand: var(--font-brand);
}
```

### Scripts and dynamic imports

```tsx
import Script from 'next/script';

export function ThirdParty() {
  return (
    <>
      <Script src="https://www.googletagmanager.com/gtag/js?id=G-XXX" strategy="afterInteractive" />
      <Script src="https://widget.chat.com/load.js" strategy="lazyOnload" />
    </>
  );
}
```

```tsx
'use client'; // ssr:false requires a Client Component
import dynamic from 'next/dynamic';

const Chart = dynamic(() => import('./Chart'), {
  ssr: false, // uses window/canvas
  loading: () => <p>Loading chart…</p>,
});

export function Dashboard() {
  return <Chart />;
}
```

## ⚠️ Common mistakes & gotchas

- Using `fill` without `sizes`, which means oversized downloads on mobile.
- Using `fill` without a positioned parent, so the image collapses or covers the page.
- Lazy-loading the hero/LCP image. Mark it `preload` (or `priority` in older versions).
- Forgetting `remotePatterns`, which gives "hostname is not configured" errors.
- Importing Google Fonts via `<link>` instead of `next/font`, which brings layout shift and an external request.
- `ssr: false` inside a Server Component (error). Move it into a `'use client'` wrapper.
- Putting everything behind `'use client'`, which kills the zero-JS benefit of Server Components.

## 🎤 Say this in the interview

> "Next optimizes the Core Web Vitals through its components. `next/image` handles responsive sizes, modern formats, lazy loading and reserved space, which helps LCP and CLS. `next/font` self-hosts fonts with metric-matched fallbacks for zero CLS. `next/script` and `next/dynamic` keep third-party code and heavy client components off the critical path, which helps INP. Add per-route code splitting, prefetching and Server Components that ship no JS."

## ❓ Likely interview questions

**Why use next/image over img?**

Automatic resizing and srcset, WebP/AVIF, lazy loading, reserved space against CLS, and server-side caching.

**What does the sizes prop do?**

It tells the browser the rendered width at each breakpoint, so it picks the smallest adequate `srcset` candidate.

**How do you optimize the LCP image?**

Use `preload` (Next 16) or `priority` (older), correct `sizes`, a modern format, and render it server-side.

**How does next/font avoid layout shift?**

It self-hosts the font and generates a fallback font with adjusted metrics (`size-adjust`), so the swap doesn't move text.

**Explain next/script strategies.**

`beforeInteractive` loads before hydration, `afterInteractive` (default) right after hydration, `lazyOnload` when idle, and `worker` (experimental) in a web worker.

**When would you use next/dynamic with ssr false?**

For browser-only libraries (charts, maps, editors) that touch `window`. Only from a Client Component.

**What are the Core Web Vitals?**

LCP (loading, ≤2.5s), INP (responsiveness, ≤200ms, replaced FID), CLS (stability, ≤0.1).

**How do you find what bloats your bundle?**

Use `@next/bundle-analyzer` or the Turbopack analyzer, then look for big client components and barrel imports.

---

⬅️ Prev: [11 Navigation & Linking](../11-Navigation-and-Linking/README.md) | Next ➡️: [13 Metadata & SEO](../13-Metadata-and-SEO/README.md)
