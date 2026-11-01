# 🔌 Route Handlers (API Routes)

> **TL;DR:** A `route.ts` file inside `app/` creates an HTTP endpoint. Export functions named after HTTP methods (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`…) that take a Web `Request` (or `NextRequest`) and return a `Response` (or `NextResponse`). Use them for public APIs, webhooks and non-React clients. GET handlers are **uncached by default since Next 15**.

## 🧠 Concept

Route Handlers are the App Router's version of the Pages Router's `pages/api/*`. They are built on **Web standard** `Request` and `Response` APIs (the same as in browsers, Deno and Cloudflare Workers) instead of Node's `req`/`res` objects.

Analogy: if `page.tsx` is a **shop window** (it shows UI to people), `route.ts` is the **delivery hatch at the back**: it exchanges raw data (JSON, files, streams) with other systems.

When to use them:

- A mobile app, third-party service or another frontend needs a JSON API.
- **Webhooks** (Stripe, GitHub, CMS) that POST to your app.
- Non-HTML responses: RSS, sitemaps, files, OG images, proxies, streaming.
- **Not** for your own UI's mutations (use Server Actions) or your own Server Components' data (call the function directly).

## ⚙️ How it works / Deep dive

### File convention

| File | URL |
| --- | --- |
| `app/api/users/route.ts` | `/api/users` |
| `app/api/users/[id]/route.ts` | `/api/users/123` |
| `app/feed.xml/route.ts` | `/feed.xml` |

- `api/` is only a convention; any folder works.
- A folder **cannot contain both `route.ts` and `page.tsx`** at the same segment level, because both would claim the same URL.
- Supported exports: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`. Unsupported methods return **405**. If `OPTIONS` isn't defined, Next.js implements it automatically.

### Reading the request

| Data | How |
| --- | --- |
| JSON body | `await request.json()` |
| Form body | `await request.formData()` |
| Raw text (webhook signatures) | `await request.text()` |
| Query string | `request.nextUrl.searchParams.get('q')` (`NextRequest`) or `new URL(request.url).searchParams` |
| Dynamic params | Second argument `{ params }`, which is a **Promise**: `const { id } = await params` |
| Headers | `request.headers.get('authorization')` or `(await headers()).get(...)` |
| Cookies | `request.cookies.get('token')` or `(await cookies()).get('token')` |

### Responding

- `NextResponse.json(data, { status: 201 })` or the standard `Response.json(data)`.
- `new Response(body, { status, headers })` for custom content types.
- `NextResponse.redirect(new URL('/login', request.url))`.
- Set cookies with `response.cookies.set(...)` or `(await cookies()).set(...)`.

Common status codes: **200** OK, **201** Created, **204** No Content, **400** Bad Request (validation), **401** Unauthenticated, **403** Forbidden, **404** Not Found, **405** Method Not Allowed, **500** Server Error.

### Caching behaviour

| Version | GET default |
| --- | --- |
| Next 13 / 14 | **Static or cached** if no dynamic APIs were used |
| Next 15 / 16 | **Uncached** (runs on every request) |

Opt in to a static GET with `export const dynamic = 'force-static'` (classic model). With Cache Components in Next 16, put `'use cache'` in a helper function the handler calls. Other methods (`POST` and so on) are never cached.

### CORS

Browsers block cross-origin calls unless the response includes `Access-Control-Allow-*` headers. Return them from each handler and from an `OPTIONS` handler (preflight). For many routes, set them centrally in `proxy.ts` (Next 16) or with `headers()` in `next.config.ts`.

### Streaming responses

Return a `ReadableStream` (for example, AI token streaming or Server-Sent Events). The client receives the chunks as they're produced.

### Pages Router equivalent

```ts
// pages/api/users/[id].ts (Pages Router, legacy)
import type { NextApiRequest, NextApiResponse } from 'next';

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === 'GET') return res.status(200).json({ id: req.query.id });
  res.setHeader('Allow', ['GET']);
  res.status(405).end();
}
```

Differences: one default export with a manual `req.method` switch, Node-style `req`/`res`, and `req.query` is sync. App Router uses named method exports, Web `Request`/`Response`, and async `params`.

## 💻 Code examples

### Collection: GET and POST

```ts
// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';

export async function GET(request: NextRequest) {
  const page = Number(request.nextUrl.searchParams.get('page') ?? 1);
  const posts = await db.post.findMany({ skip: (page - 1) * 10, take: 10 });
  return NextResponse.json(posts); // 200
}

const PostSchema = z.object({ title: z.string().min(1) });

export async function POST(request: NextRequest) {
  const token = request.headers.get('authorization');
  if (!token) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const parsed = PostSchema.safeParse(await request.json());
  if (!parsed.success) return NextResponse.json(parsed.error.flatten(), { status: 400 });

  const post = await db.post.create({ data: parsed.data });
  return NextResponse.json(post, { status: 201 });
}
```

### Dynamic handler with async params

```ts
// app/api/posts/[id]/route.ts
import { NextResponse } from 'next/server';

type Ctx = { params: Promise<{ id: string }> }; // or RouteContext<'/api/posts/[id]'> (15.5+)

export async function GET(_req: Request, { params }: Ctx) {
  const { id } = await params;                     // Promise since Next 15
  const post = await db.post.findUnique({ where: { id } });
  if (!post) return NextResponse.json({ error: 'Not found' }, { status: 404 });
  return NextResponse.json(post);
}

export async function DELETE(_req: Request, { params }: Ctx) {
  const { id } = await params;
  await db.post.delete({ where: { id } });
  return new Response(null, { status: 204 });
}
```

### Cookies and CORS

```ts
// app/api/public/route.ts
import { cookies } from 'next/headers';

const cors = {
  'Access-Control-Allow-Origin': 'https://partner.com',
  'Access-Control-Allow-Methods': 'GET, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization',
};

export async function OPTIONS() {
  return new Response(null, { status: 204, headers: cors }); // preflight
}

export async function GET() {
  const theme = (await cookies()).get('theme')?.value ?? 'light';
  return Response.json({ theme }, { headers: cors });
}
```

### Webhook (raw body plus signature check)

```ts
// app/api/webhooks/stripe/route.ts
import Stripe from 'stripe';
import { revalidateTag } from 'next/cache';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(request: Request) {
  const body = await request.text();                    // raw body needed for signature
  const sig = request.headers.get('stripe-signature')!;
  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch {
    return new Response('Invalid signature', { status: 400 });
  }
  if (event.type === 'checkout.session.completed') {
    await markOrderPaid(event.data.object.id);
    revalidateTag('orders', 'max');                     // Next 16 signature
  }
  return Response.json({ received: true });
}
```

### Streaming response

```ts
// app/api/stream/route.ts
export async function GET() {
  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      for (const word of ['Hello', ' ', 'world']) {
        controller.enqueue(encoder.encode(word));
        await new Promise(r => setTimeout(r, 300));
      }
      controller.close();
    },
  });
  return new Response(stream, { headers: { 'Content-Type': 'text/plain; charset=utf-8' } });
}
```

## ⚠️ Common mistakes & gotchas

- Putting `route.ts` and `page.tsx` in the same folder, which causes a conflict error.
- Using a default export or `req.method` switching (Pages Router habit). Use named method exports.
- Reading `params.id` synchronously. `await params` (required in Next 16).
- Expecting GET to be cached (Next 14 behaviour). It is uncached since Next 15.
- Calling `request.json()` before verifying a webhook signature. Use the **raw** `text()` body.
- Calling your own Route Handlers from Server Components. Call the shared function directly.
- Forgetting the `OPTIONS` preflight or CORS headers for cross-origin clients.
- Skipping auth and validation: these are public endpoints, just like Server Actions.

## 🎤 Say this in the interview

> "Route Handlers are `route.ts` files that export HTTP-method functions using the Web Request and Response APIs. I use them for public APIs, webhooks, streaming and non-React clients, and Server Actions for my own UI's mutations. Since Next 15, GET handlers aren't cached by default, and dynamic `params` arrive as a Promise."

## ❓ Likely interview questions

**How do you create an API endpoint in the App Router?**

Add `app/api/<name>/route.ts` and export async functions like `GET` or `POST` that return a `Response` or `NextResponse`.

**How do you read dynamic params and query strings?**

Params come from the second argument (`await params`). The query string comes from `request.nextUrl.searchParams`.

**Are GET Route Handlers cached?**

Not by default since Next 15. In Next 14 they were static by default. Opt in with `dynamic = 'force-static'` or `'use cache'` helpers.

**Can `route.ts` and `page.tsx` live in the same folder?**

No. Both would handle the same URL, so Next.js reports a conflict.

**`NextRequest`/`NextResponse` vs `Request`/`Response`?**

The Next versions extend the Web APIs with helpers such as `nextUrl`, `cookies`, `NextResponse.json` and `redirect`.

**Route Handler vs Server Action?**

Route Handlers serve external clients with any HTTP method and URL. Server Actions are POST-only functions for your own UI's mutations, with progressive enhancement.

**How do you handle CORS?**

Return `Access-Control-Allow-*` headers and an `OPTIONS` preflight handler, or set them centrally in `proxy.ts` or `next.config.ts` headers.

---

⬅️ Prev: [Server Actions](../08-Server-Actions/README.md) | Next ➡️: [Middleware / Proxy](../10-Middleware-Proxy/README.md)
