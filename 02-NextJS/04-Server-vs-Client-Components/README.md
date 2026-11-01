# ⚛️ Server Components vs Client Components

> **TL;DR:** In the App Router every component is a **React Server Component (RSC)** by default. It runs only on the server, can read databases and secrets directly, and ships **zero JS**. Add `'use client'` only where you need interactivity (state, effects, event handlers, browser APIs). That directive marks a **boundary**: the file and everything it imports become client code.

## 🧠 Concept

### Why Server Components exist

In classic React (even with SSR), **every component's JS is sent to the browser** and hydrated, including components that only show static text or data. RSC splits the tree into two kinds:

- **Server Components**: render on the server (at build or request time). Their code **never reaches the browser**, only their rendered output does.
- **Client Components**: the familiar React components. They are pre-rendered to HTML on the server too, **then** their JS is sent to the browser and hydrated.

Analogy: a restaurant. Server Components are the **kitchen**: it has the ingredients (database, secrets) and sends out finished dishes. Client Components are the **waiter at your table**: they react to what you do (clicks, typing). You don't need to ship the kitchen to the customer.

### Benefits

| Benefit | Explanation |
| --- | --- |
| Less JS | Server Component code and its heavy dependencies (markdown parsers, date libraries) stay on the server |
| Direct data access | `await db.query()` inside the component, with no API layer or `useEffect` |
| Security | API keys, tokens and DB credentials never reach the client |
| Faster first load | HTML streams early; less hydration work |
| Caching | Rendered output can be cached and reused across users |

### "Client Component" does not mean "client-only rendered"

Client Components **are still rendered to HTML on the server** for the first load. "Client" means "this code is also shipped to and runs in the browser".

## ⚙️ How it works / Deep dive

### What `'use client'` really means

`'use client'` goes at the **top of a file** and declares a **boundary / entry point** from the server module graph into the client module graph.

- You don't need it on every interactive file, only on the **entry** file that a Server Component imports.
- Everything that file **imports** is pulled into the client bundle automatically, even files without the directive.
- Props passing across the boundary (server to client) must be **serialisable**.

```text
page.tsx (server)
 └── imports Counter.tsx ('use client')   ← boundary
       └── imports Button.tsx            ← client automatically
       └── imports utils.ts              ← bundled for the client
```

### Can / can't table

| Need | Server Component | Client Component |
| --- | --- | --- |
| `async`/`await` data fetching in the component | ✅ | ❌ (use `use()` or a library) |
| Access DB, file system, secrets | ✅ | ❌ |
| `useState`, `useReducer`, `useEffect`, `useRef` | ❌ | ✅ |
| `onClick`, `onChange`, event handlers | ❌ | ✅ |
| Browser APIs (`window`, `localStorage`, geolocation) | ❌ | ✅ (in effects or handlers) |
| Context (`createContext` / `useContext`) | ❌ | ✅ |
| Keep large deps off the bundle | ✅ | ❌ |
| Custom hooks that use state/effects | ❌ | ✅ |

### Composition rules

1. **Server can import Client.** This is the normal case: a server page renders `<LikeButton />`.
2. **Client cannot import Server.** Anything a client file imports *becomes* client code, so a server-only module would either break or leak.
3. **But Client can *render* Server Components passed as `children` or other props.** The server renders them first and passes the **result** (already-rendered output) through the slot. That is the "donut" pattern: a client shell with a server-rendered hole.

### Serialisable props

Props from Server to Client Components travel over the network inside the RSC payload, so they must be serialisable: strings, numbers, booleans, `null`, plain objects and arrays, `Date`, `Map`, `Set`, Promises, JSX, and **Server Actions**. **Not allowed:** ordinary functions, class instances, Symbols. (Passing `onClick={() => ...}` from a server page fails.)

### The RSC Payload

When the server renders, it produces a compact, streamable **RSC Payload**: a serialised description of the rendered tree. It contains:

- The rendered output of Server Components (as HTML-like element descriptions).
- **Placeholders plus JS file references** for Client Components, along with their props.
- Data passed from server to client.

### How it relates to hydration

On the **first load**, Next.js uses the payload to produce **HTML** (fast paint). The browser then uses the payload to reconcile the tree and **hydrate only the Client Components**. Server Components are never hydrated; there is no JS for them. On **client navigations**, only the RSC payload is fetched (no full HTML) and React merges it into the existing tree, keeping client state.

### `server-only` package

Some modules (DB access, secret keys) must never be imported into client code by accident. Add `import 'server-only'` at the top: the build **fails** if a Client Component imports it. (`client-only` is the opposite.) Also note that only `NEXT_PUBLIC_*` env vars are inlined into client bundles; others become empty strings there.

### `'use client'` vs `'use server'`: different things

| | `'use client'` | `'use server'` |
| --- | --- | --- |
| Marks | A **component boundary** (module goes to the client bundle) | **Server Functions / Server Actions** callable from the client |
| Placement | Top of a file | Top of a file (all exports become actions) or top of an async function body |
| Not for | Marking Server Components (that's the default; no directive) | Marking Server Components |

Common trap: `'use server'` does **not** mean "this is a Server Component".

## 💻 Code examples

### Server page using a client leaf

```tsx
// app/products/[id]/page.tsx: Server Component (no directive)
import { db } from '@/lib/db';
import { AddToCart } from './add-to-cart';

export default async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const product = await db.product.findUnique({ where: { id } }); // direct DB access
  return (
    <div>
      <h1>{product?.name}</h1>
      <AddToCart productId={id} />   {/* only this ships JS */}
    </div>
  );
}
```

```tsx
// app/products/[id]/add-to-cart.tsx
'use client';
import { useState } from 'react';

export function AddToCart({ productId }: { productId: string }) {
  const [qty, setQty] = useState(1);
  return <button onClick={() => setQty(q => q + 1)}>Add {qty} of {productId}</button>;
}
```

### Passing a Server Component into a Client Component (children pattern)

```tsx
// modal.tsx
'use client';
import { useState } from 'react';

export function Modal({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return <>{open && <div className="modal">{children}</div>}<button onClick={() => setOpen(true)}>Open</button></>;
}

// page.tsx (server)
import { Modal } from './modal';
import { Cart } from './cart'; // Server Component that queries the DB

export default function Page() {
  return <Modal><Cart /></Modal>; // Cart rendered on server, passed as a slot
}
```

### Context provider wrapped in a Client Component

```tsx
// app/providers.tsx
'use client';
import { ThemeProvider } from 'next-themes';
export function Providers({ children }: { children: React.ReactNode }) {
  return <ThemeProvider attribute="class">{children}</ThemeProvider>;
}
// app/layout.tsx (server) → <body><Providers>{children}</Providers></body>
// children stay Server Components
```

### Third-party library without `'use client'`

```tsx
// app/_components/carousel.tsx: re-export it behind a client boundary
'use client';
export { Carousel } from 'acme-carousel'; // uses useState internally but has no directive
```

### Guarding server code

```ts
// lib/db.ts
import 'server-only';                 // build error if imported by a Client Component
export const db = createClient(process.env.DATABASE_URL!);
```

## ⚠️ Common mistakes & gotchas

- Adding `'use client'` to the root layout or page "to make it work", which turns the whole tree into client code.
- Importing a Server Component inside a Client Component file. Pass it as `children` instead.
- Passing functions or class instances as props from server to client.
- Using `useState` or `onClick` in a file without `'use client'`, which causes a build error.
- Using `'use server'` thinking it marks a Server Component.
- Reading `window` during render in a Client Component (it still pre-renders on the server). Use `useEffect` or `dynamic(() => import(...), { ssr: false })` inside a Client Component.
- Putting secrets in a module that a client file imports. Use `server-only`.

## 🎤 Say this in the interview

> "Server Components are the default. They run only on the server, can fetch data and use secrets directly, and send no JS, just their rendered output in the RSC payload. `'use client'` marks a boundary where the module and its imports join the client bundle. I keep Client Components at the leaves, pass server content into them as `children`, and wrap context providers in a small client component."

## ❓ Likely interview questions

**What is a React Server Component?**

A component that renders only on the server. Its code and dependencies are never sent to the browser; only the serialised output (RSC payload) is.

**Are Client Components rendered on the server?**

Yes. They are pre-rendered to HTML on first load, then hydrated in the browser.

**Can a Client Component render a Server Component?**

It can't import one, but it can receive one as `children` or another prop from a Server Component parent.

**What is the RSC payload?**

A compact, streamable serialisation of the rendered server tree, with placeholders and JS references for Client Components and their props. It is used to build the DOM, hydrate, and update on navigation.

**Difference between `'use client'` and `'use server'`?**

`'use client'` marks a client module boundary. `'use server'` marks Server Functions (actions) callable from the client. Server Components need no directive.

**How do you use Context in the App Router?**

Create the provider in a `'use client'` file and wrap `{children}` with it in the layout. Children can still be Server Components.

**What does the `server-only` package do?**

It makes the build fail if that module is imported into client code, which prevents leaking secrets or DB code.

**Where should `'use client'` go?**

As low in the tree as possible: on small interactive leaves, not on pages or layouts.

---

⬅️ Prev: [Layouts, Templates, Loading and Error](../03-Layouts-Templates-Loading-Error/README.md) | Next ➡️: [Rendering Strategies](../05-Rendering-Strategies/README.md)
