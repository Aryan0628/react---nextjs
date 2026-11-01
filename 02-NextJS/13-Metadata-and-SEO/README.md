# 🔎 Metadata & SEO

> **TL;DR:** Next.js renders HTML on the server, so crawlers see real content. In the App Router you describe the `<head>` with a `metadata` object or an async `generateMetadata()` function, exported from `layout.tsx`/`page.tsx` (Server Components only). Special files such as `opengraph-image.tsx`, `sitemap.ts`, `robots.ts` and `manifest.ts` generate the rest.

## 🧠 Concept

Search engines and social previews (WhatsApp, X, LinkedIn, Slack) read your HTML: the `<title>`, `<meta>` tags, Open Graph tags and structured data. With a plain client-side React app the first HTML is an empty `<div id="root">`. Google *can* run JS, but it's slower and less reliable, and most social scrapers don't run JS at all.

Next.js fixes this two ways:

1. **Server rendering / static generation** puts the content in the initial HTML.
2. **The Metadata API**: you declare metadata as data, and Next renders the right tags, merges them across nested layouts and removes duplicates.

## ⚙️ How it works / Deep dive

### Static vs dynamic metadata

- `export const metadata: Metadata = {...}` is for fixed values (home, about).
- `export async function generateMetadata({ params, searchParams }, parent)` is for values that depend on data (a blog post title). `params` is a **Promise** in Next 15+.
- You can only export them from **Server Components** (`layout`/`page`). A `'use client'` file can't export metadata.
- **Fetch dedupe:** the page usually needs the same data as `generateMetadata`. Identical `fetch` GET calls in one render are memoized automatically. For ORM/DB calls, wrap the function in React `cache()` so it runs once.
- **Streaming metadata (15.2+):** for real browsers `generateMetadata` no longer blocks the UI; tags stream in. Bots (detected by user agent) still get metadata in the initial `<head>`.

### Merging and title templates

Metadata is merged **from the root layout down to the page**. Deeper segments override keys from higher ones (shallow merge per top-level key, so a page's `openGraph` replaces the layout's `openGraph` completely).

`title.template` in a layout (for example `'%s | Acme'`) wraps child titles. `title.default` is used when a child sets no title.

### Key fields

| Field | Produces |
| --- | --- |
| `metadataBase` | Base URL so relative OG/canonical URLs become absolute |
| `title`, `description`, `keywords` | `<title>`, `<meta name="description">` |
| `openGraph` | `og:title`, `og:image`, `og:type`… (Facebook, LinkedIn, WhatsApp) |
| `twitter` | `twitter:card`, `twitter:image`… |
| `alternates.canonical` / `alternates.languages` | `<link rel="canonical">`, `hreflang` links |
| `robots` | `<meta name="robots" content="noindex">` |
| `icons` | Favicons and apple-touch icons |

`viewport` and `themeColor` live in a separate `export const viewport` (since Next 14).

### File-based metadata (special files in `app/`)

| File | Result |
| --- | --- |
| `favicon.ico`, `icon.png`, `apple-icon.png` | Icon `<link>` tags |
| `opengraph-image.png` or `.tsx`, `twitter-image.*` | OG image for that segment; `.tsx` generates it with `ImageResponse` |
| `sitemap.ts` | `/sitemap.xml` |
| `robots.ts` | `/robots.txt` |
| `manifest.ts` | `/manifest.webmanifest` (PWA) |

### Canonical URLs

If the same content is reachable at several URLs (`?ref=twitter`, trailing slash, `/products/1` and `/shoes/1`), tell Google which one is the original with `alternates.canonical`. That keeps duplicate-content penalties away and puts the ranking signals on one URL.

### JSON-LD structured data

This gives Google machine-readable facts (Product price, Article author, Breadcrumbs) that can turn into rich results. In Next you render a `<script type="application/ld+json">` in the page itself. There's no metadata key for it.

## 💻 Code examples

### Root layout: base, template, defaults

```tsx
// app/layout.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  metadataBase: new URL('https://acme.com'),
  title: { default: 'Acme Store', template: '%s | Acme Store' },
  description: 'Quality gear, shipped fast.',
  openGraph: { siteName: 'Acme Store', type: 'website', locale: 'en_US' },
  twitter: { card: 'summary_large_image', site: '@acme' },
};
```

### Dynamic page metadata (with dedupe)

```tsx
// app/products/[slug]/page.tsx
import type { Metadata } from 'next';
import { cache } from 'react';
import { notFound } from 'next/navigation';

const getProduct = cache(async (slug: string) => db.product.findUnique({ where: { slug } }));

type Props = { params: Promise<{ slug: string }> };

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  const product = await getProduct(slug); // runs once, shared with the page
  if (!product) return { title: 'Not found' };
  return {
    title: product.name, // becomes "Shoe X | Acme Store"
    description: product.summary,
    alternates: { canonical: `/products/${slug}` },
    openGraph: { images: [{ url: product.image, width: 1200, height: 630 }] },
  };
}

export default async function Page({ params }: Props) {
  const { slug } = await params;
  const product = await getProduct(slug);
  if (!product) notFound();

  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: product.name,
    image: product.image,
    offers: { '@type': 'Offer', price: product.price, priceCurrency: 'USD' },
  };

  return (
    <>
      <script
        type="application/ld+json"
        // escape "<" to prevent XSS via injected </script>
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd).replace(/</g, '\\u003c') }}
      />
      <h1>{product.name}</h1>
    </>
  );
}
```

### Generated OG image

```tsx
// app/products/[slug]/opengraph-image.tsx
import { ImageResponse } from 'next/og';

export const size = { width: 1200, height: 630 };
export const contentType = 'image/png';

export default async function OgImage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  return new ImageResponse(
    <div style={{ display: 'flex', fontSize: 72, background: '#111', color: '#fff', width: '100%', height: '100%', alignItems: 'center', justifyContent: 'center' }}>
      {slug}
    </div>,
    size,
  );
}
```

### Sitemap and robots

```ts
// app/sitemap.ts
import type { MetadataRoute } from 'next';

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const products = await db.product.findMany({ select: { slug: true, updatedAt: true } });
  return [
    { url: 'https://acme.com', lastModified: new Date(), priority: 1 },
    ...products.map((p) => ({ url: `https://acme.com/products/${p.slug}`, lastModified: p.updatedAt })),
  ];
}
```

```ts
// app/robots.ts
import type { MetadataRoute } from 'next';

export default function robots(): MetadataRoute.Robots {
  return {
    rules: { userAgent: '*', allow: '/', disallow: ['/admin', '/api'] },
    sitemap: 'https://acme.com/sitemap.xml',
  };
}
```

### Pages Router comparison

```tsx
// pages/products/[slug].tsx (legacy)
import Head from 'next/head';

export default function Product({ product }: { product: { name: string } }) {
  return (
    <>
      <Head>
        <title>{`${product.name} | Acme`}</title>
        <meta name="description" content="..." />
      </Head>
      <h1>{product.name}</h1>
    </>
  );
}
```

`next/head` is imperative JSX scattered in components, with duplicates fixed via `key`. The App Router's Metadata API is declarative, typed, merged across layouts, and `next/head` does nothing in `app/`.

## ⚠️ Common mistakes & gotchas

- Exporting `metadata` from a `'use client'` file (error). Keep the page as a Server Component and move the interactivity into a child.
- Missing `metadataBase`, so OG image URLs stay relative and social cards break.
- Expecting nested `openGraph` objects to deep-merge. They don't, so repeat the fields you need.
- Fetching data twice for metadata and page without `cache()` (non-fetch data sources).
- Using `next/head` inside the App Router.
- Leaving `robots: { index: false }` on in production after staging.
- Special files like `sitemap.ts`/`robots.ts` must **default-export** their function.

## 🎤 Say this in the interview

> "Next helps SEO by rendering real HTML on the server. In the App Router I declare a static `metadata` object or an async `generateMetadata` in Server Component layouts and pages. Next merges it down the tree, with title templates for consistency. I add canonical URLs, Open Graph images (often generated with `ImageResponse`), `sitemap.ts`, `robots.ts` and JSON-LD for rich results."

## ❓ Likely interview questions

**How does Next.js improve SEO over a CRA/Vite SPA?**

Crawlers and social scrapers get fully rendered HTML with meta tags, instead of an empty shell that needs JS.

**metadata vs generateMetadata?**

A static object vs an async function for data-dependent values. Both work only in Server Component `layout`/`page` files.

**Won't generateMetadata and the page fetch the same data twice?**

No. `fetch` is memoized per request, and for DB calls you wrap them in React `cache()`.

**How do title templates work?**

The layout sets `title: { template: '%s | Site', default: 'Site' }`, and child pages set a plain `title` that gets inserted into `%s`.

**How do you create dynamic OG images?**

Add an `opengraph-image.tsx` that returns `new ImageResponse(<JSX/>)` from `next/og`.

**What is a canonical URL and why does it matter?**

The preferred URL for duplicate or near-duplicate content. It concentrates ranking signals on one URL and avoids duplicate-content problems.

**How do you add structured data?**

Render a `<script type="application/ld+json">` with a schema.org object in the page, escaping `<`.

---

⬅️ Prev: [12 Built-in Optimizations](../12-Built-in-Optimizations/README.md) | Next ➡️: [14 Pages Router (Legacy)](../14-Pages-Router-Legacy/README.md)
