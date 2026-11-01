# ⚔️ React vs Next.js

> **TL;DR:** React is a **UI library**. It gives you components, state and rendering, and nothing else. Next.js is a **framework built on React** that adds routing, server rendering, data fetching, caching, a backend layer and performance optimizations. Every Next.js app is a React app, but a React app is not automatically a Next.js app.

## 🧠 Concept

Think of it like building a car:

- **React** is the **engine**. It's powerful, but you still have to choose and assemble the chassis, wheels and steering yourself: a router, a bundler, a data-fetching approach, SSR, an API server.
- **Next.js** is the **complete car**. It uses React as the engine and ships with everything else already assembled, with sensible defaults (conventions).

The **core technical difference** is **where the HTML is produced**:

- A plain React app (Vite) is **client-side rendered (CSR)**. The server sends an almost empty HTML file plus a JS bundle, and the browser builds the page.
- Next.js renders on the **server**: at request time (SSR), at build time (SSG), or a mix (ISR, streaming, PPR). The browser receives ready-made HTML, then React **hydrates** it to make it interactive.

## ⚙️ How a page loads

### React (CSR)

```text
1. Browser requests /products
2. Server returns:  <html><body><div id="root"></div><script src="bundle.js"></script></body></html>
3. Browser downloads + parses + runs bundle.js       ⏳ (user sees blank page / spinner)
4. React renders components, useEffect fires fetch("/api/products")   ⏳
5. Data arrives → re-render → user finally sees products
```

### Next.js (SSR / SSG)

```text
1. Browser requests /products
2. Server runs Server Components, fetches data, renders full HTML
3. Browser receives HTML WITH products already in it   ✅ (content visible immediately, SEO-friendly)
4. Small JS bundle loads → hydration → buttons become interactive
```

This is why Next.js wins on **SEO** (crawlers see real content) and **first-load performance** (better LCP / FCP).

## 📊 Feature Comparison

| Feature | React (with Vite) | Next.js |
| --- | --- | --- |
| Type | Library (UI layer only) | Full-stack React framework |
| Default rendering | CSR | Server-rendered (static by default), with SSR / ISR / streaming |
| Rendering options | CSR (SSR only if you build it yourself) | CSR, SSR, SSG, ISR, Streaming, PPR |
| Routing | External library (React Router, TanStack Router) | Built-in **file-based** routing |
| Server Components | Need a framework to use them | Default in App Router |
| SEO | Weak by default (empty initial HTML) | Strong (full HTML + Metadata API) |
| Data fetching | `useEffect`, TanStack Query, SWR | `async` Server Components + `fetch` caching, plus client libs |
| Backend / API | Separate server (Express, Nest, etc.) | Route Handlers + Server Actions |
| Mutations | Call your own API | Server Actions (`'use server'`) |
| Middleware | ❌ | ✅ `proxy.ts` (formerly `middleware.ts`) |
| Code splitting | Manual with `React.lazy` | Automatic per route |
| Images / fonts | Manual or libraries | `next/image`, `next/font` built-in |
| Caching | You build it | Multi-layer caching + revalidation |
| Bundler | Vite (esbuild/Rollup) | Turbopack (default in v16) |
| Env vars exposed to browser | `VITE_` prefix | `NEXT_PUBLIC_` prefix |
| Hosting | Any static host (S3, Netlify, GitHub Pages) | Node server / Vercel / Docker, or static export |
| Flexibility | Very high (you choose everything) | Opinionated (conventions) |
| Learning curve | Lower | Higher (server/client split, caching) |

## 💻 Code Comparison

### Routing

```jsx
// React + React Router: routes are declared in code
<BrowserRouter>
  <Routes>
    <Route path="/" element={<Home />} />
    <Route path="/blog/:id" element={<Post />} />
  </Routes>
</BrowserRouter>

// inside Post
const { id } = useParams();
```

```text
Next.js: routes come from the folder structure
app/page.tsx            →  /
app/blog/[id]/page.tsx  →  /blog/:id
```

```tsx
// app/blog/[id]/page.tsx
export default async function Post({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  return <h1>Post {id}</h1>;
}
```

### Data fetching

```jsx
// React: fetch on the client after the first render
function Products() {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/products')
      .then((r) => r.json())
      .then((data) => { setProducts(data); setLoading(false); });
  }, []);

  if (loading) return <Spinner />;
  return <ProductList products={products} />;
}
```

```tsx
// Next.js: fetch on the server, no useEffect, no loading state, no client JS for this part
export default async function Products() {
  const products = await db.product.findMany(); // direct DB access is safe here
  return <ProductList products={products} />;
}
```

### Backend endpoint

```js
// React: needs a separate Express server
app.get('/api/products', (req, res) => res.json(products));
```

```ts
// Next.js: lives in the same project
// app/api/products/route.ts
export async function GET() {
  return Response.json(products);
}
```

## 🎯 When to choose which

### Choose plain React (Vite) when

- The app sits **behind a login** (admin panels, internal dashboards), so SEO doesn't matter
- It's a **highly interactive SPA**: editors, design tools, games, chat apps
- You already have a **separate backend** (Java, Go, Django, Node API)
- You want **full control** over architecture and minimal abstractions
- You want **cheap static hosting** with no server
- It's an **embeddable widget** or micro-frontend

### Choose Next.js when

- **SEO is important**: e-commerce, blogs, news, marketing and landing pages
- You need a **fast first load** and good Core Web Vitals
- You want **frontend + backend in one codebase**
- Content is mostly static but changes sometimes (**ISR**)
- It's a big team or app that benefits from **conventions**
- You want built-in image, font and script optimization

## ✅ ❌ Pros & Cons

| | React (plain) | Next.js |
| --- | --- | --- |
| Pros | Simple mental model, flexible, lightweight, huge ecosystem, static hosting | SEO, performance, routing, full-stack, caching, optimizations out of the box |
| Cons | Poor SEO by default, slower first load, you configure everything | More concepts (server/client boundary, caching), needs a server for SSR, opinionated, some hosting lock-in concerns |

## ⚠️ Common misconceptions

- **"Next.js replaces React."** No. It *uses* React. You still write React components and hooks.
- **"Server Components are a Next.js feature."** They're a **React** feature. Next.js App Router is the most popular framework that implements them.
- **"React can't do SSR."** It can, with `renderToPipeableStream` + `hydrateRoot`. But you'd have to build routing, data loading, code splitting and hydration yourself, and that's exactly what Next.js does for you.
- **"Next.js is always faster."** Not for everything. A client-heavy app behind a login gains little from SSR, and it adds server cost and complexity.
- **"Create React App is the standard way to start."** CRA is **deprecated** (2025). The React docs recommend a framework (Next.js, React Router framework mode) or Vite.

## 🎤 Say this in the interview

> "React is a JavaScript library for building component-based UIs. It renders on the client by default and leaves routing, data fetching and server rendering up to you. Next.js is a framework on top of React. It adds file-based routing and several rendering strategies (SSR, SSG, ISR and streaming), defaults to React Server Components, and includes Server Actions and API routes for backend logic, plus image and font optimization. I'd use plain React for SPAs or dashboards where SEO doesn't matter, and Next.js for SEO-critical, performance-sensitive or full-stack apps."

## ❓ Likely interview questions

**Q: Why would you use Next.js over React?**
SEO (server-rendered HTML), faster first load, built-in routing, full-stack features (Route Handlers, Server Actions), automatic code splitting, and image/font optimization without extra setup.

**Q: Is Next.js a frontend or a backend framework?**
Both. It's a full-stack framework: React on the client, plus server rendering, API endpoints and Server Actions running on Node.js or Edge.

**Q: Can you build SSR in plain React?**
Yes, using `react-dom/server` APIs with an Express server and `hydrateRoot` on the client. But you'd have to manage routing, data loading, code splitting and hydration yourself.

**Q: What's the difference in how routing works?**
React uses a library where routes are declared in JSX/config. Next.js derives routes from the `app/` folder structure (`page.tsx` files, `[param]` folders).

**Q: How does data fetching differ?**
In React you usually fetch on the client after render (`useEffect` / TanStack Query). In Next.js you fetch on the server inside `async` Server Components, before the HTML is sent, with built-in caching and revalidation.

**Q: Does the official React team recommend using a framework?**
Yes. react.dev recommends starting with a framework such as Next.js or React Router (framework mode), or Vite if you want to build from scratch. Create React App is deprecated.

**Q: Which one has better performance?**
Next.js usually gives a better *first load* (LCP, SEO) because of server rendering and less client JS. After the page has loaded, both are React and perform about the same. It depends on the use case.

**Q: What are alternatives to Next.js?**
React Router v7 framework mode (formerly Remix), TanStack Start, Astro (content sites), and Gatsby (SSG, declining).

---

⬅️ Prev: [Next.js Interview Q&A](../02-NextJS/17-Interview-QA/README.md) | 🏠 [Home](../README.md)
