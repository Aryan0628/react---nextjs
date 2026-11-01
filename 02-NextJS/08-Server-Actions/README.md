# ⚡ Server Actions

> **TL;DR:** Server Actions are **async functions marked `'use server'` that run on the server but can be called from the client**: from a `<form action>`, a button, or an event handler. Under the hood Next.js turns each one into a **POST endpoint**. They handle mutations (create, update, delete), work without JS (progressive enhancement), and pair with `useActionState`, `useFormStatus`, `useOptimistic` and revalidation. They are **public endpoints**, so always authenticate and validate inside them.

## 🧠 Concept

Before Server Actions, a mutation meant: write an API route, `fetch('/api/todos', { method: 'POST', body })` from the client, handle loading and errors, and then refetch the data.

With Server Actions you **write a function and pass it to the form**. Next.js generates a hidden endpoint with an ID, wires the client call to a `POST` request, runs your function on the server, and can send the **updated UI back in the same round trip** (after `revalidatePath` or `updateTag`).

Analogy: instead of writing a letter, addressing it and posting it (API route plus fetch), you press an intercom button labelled "add todo" and the server just does it.

React calls these **Server Functions**. When used for form or mutation work they are called **Server Actions**.

## ⚙️ How it works / Deep dive

### Defining them: inline vs file-level

- **File-level** `'use server'` at the top of a file: **every export** is a Server Action. This is the only way to use them in Client Components (import them from that file).
- **Inline** `'use server'` as the first line of an async function inside a **Server Component**. It can close over variables from the component (Next.js **encrypts** closed-over values sent to the client).

### Forms, `FormData` and progressive enhancement

Pass the action to `<form action={fn}>`. On submit React calls it with a **`FormData`** object (read fields with `formData.get('name')`). If the form is rendered by a Server Component, it **works even before JS loads, or with JS disabled**: it falls back to a normal HTML form POST. That is **progressive enhancement**. Once hydrated, React submits without a full reload.

### Calling from event handlers

In Client Components you can call an imported action like a normal async function (`await likePost(id)`), often inside `startTransition` to get a pending state.

### React 19 hooks

| Hook | Package | Purpose |
| --- | --- | --- |
| `useActionState(action, initialState)` | `react` | Returns `[state, formAction, isPending]`. The action receives `(prevState, formData)`, and what it returns becomes `state` (field errors, success message) |
| `useFormStatus()` | `react-dom` | `{ pending, data, method }` of the **parent** `<form>`. Must be used in a **child component** of the form (for example, a submit button) |
| `useOptimistic(state, updateFn)` | `react` | Shows an instant optimistic UI while the action runs; it reverts automatically if the action fails |

### After the mutation: revalidate and redirect

- `revalidatePath('/todos')`, `updateTag('todos')` (read-your-writes, Next 16), `revalidateTag('todos', 'max')`, or `refresh()` to update the UI.
- `redirect('/todos/123')` from `next/navigation` navigates after success. It throws internally, so call it **outside `try/catch`** and after revalidating.
- You can also set cookies with `(await cookies()).set(...)`.

### Validation with Zod

Never trust `FormData`: the client controls it. Parse it with a schema on the server and **return** errors as values (don't throw), so `useActionState` can display them.

### 🔐 Security: they are public HTTP endpoints

Anyone can call an action's endpoint with any payload (curl, a modified client), even if the button is hidden in your UI.

- **Authenticate and authorise inside every action**: check the session and check that the user owns the resource. Don't rely on the page having checked it, or on the proxy alone.
- **Validate all input** (Zod), including IDs and hidden inputs.
- **Closures**: values captured by inline actions are encrypted, but **don't treat that as a security boundary**. Still avoid capturing secrets, and re-check permissions.
- **Action IDs** are non-deterministic, encrypted references that are recomputed per build, and unused actions are removed from the bundle (Next 15+). That reduces the attack surface but **doesn't replace auth**.
- Built-in CSRF protection: POST only, plus an `Origin` vs `Host` check (configure `serverActions.allowedOrigins` behind proxies).
- Return only what the client needs (no full DB rows with password hashes).
- Body size limit defaults to 1 MB (`serverActions.bodySizeLimit`).

## 💻 Code examples

### Actions file with validation, auth and revalidation

```ts
// app/todos/actions.ts
'use server';
import { z } from 'zod';
import { updateTag } from 'next/cache';
import { redirect } from 'next/navigation';
import { auth } from '@/lib/auth';

const TodoSchema = z.object({ title: z.string().min(1, 'Title required').max(100) });

export type FormState = { errors?: { title?: string[] }; message?: string };

export async function createTodo(prev: FormState, formData: FormData): Promise<FormState> {
  const session = await auth();                       // 1. authenticate
  if (!session) return { message: 'Not signed in' };

  const parsed = TodoSchema.safeParse({ title: formData.get('title') }); // 2. validate
  if (!parsed.success) return { errors: parsed.error.flatten().fieldErrors };

  await db.todo.create({ data: { title: parsed.data.title, userId: session.user.id } });
  updateTag('todos');                                 // 3. read-your-writes
  redirect('/todos');                                 // 4. outside try/catch
}

export async function deleteTodo(id: string) {
  const session = await auth();
  const todo = await db.todo.findUnique({ where: { id } });
  if (!session || todo?.userId !== session.user.id) throw new Error('Forbidden'); // authorise
  await db.todo.delete({ where: { id } });
  updateTag('todos');
}
```

### Form with `useActionState` and `useFormStatus`

```tsx
// app/todos/new-todo-form.tsx
'use client';
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';
import { createTodo, type FormState } from './actions';

function SubmitButton() {
  const { pending } = useFormStatus();               // reads the parent form's status
  return <button disabled={pending}>{pending ? 'Saving…' : 'Add'}</button>;
}

export function NewTodoForm() {
  const [state, formAction, isPending] = useActionState<FormState, FormData>(createTodo, {});
  return (
    <form action={formAction}>
      <input name="title" />
      {state.errors?.title && <p className="error">{state.errors.title[0]}</p>}
      {state.message && <p>{state.message}</p>}
      <SubmitButton />
    </form>
  );
}
```

### Inline action in a Server Component (progressive enhancement)

```tsx
// app/newsletter/page.tsx
export default function Newsletter() {
  async function subscribe(formData: FormData) {
    'use server';
    await db.subscriber.create({ data: { email: String(formData.get('email')) } });
  }
  return (
    <form action={subscribe}>       {/* works without JS */}
      <input type="email" name="email" required />
      <button>Subscribe</button>
    </form>
  );
}
```

### Event handler plus `useOptimistic`

```tsx
'use client';
import { useOptimistic, startTransition } from 'react';
import { likePost } from './actions';

export function LikeButton({ postId, likes }: { postId: string; likes: number }) {
  const [optimisticLikes, addLike] = useOptimistic(likes, (curr, inc: number) => curr + inc);
  return (
    <button onClick={() => startTransition(async () => {
      addLike(1);              // instant UI
      await likePost(postId);  // server call (POST); reverts if it throws
    })}>
      ♥ {optimisticLikes}
    </button>
  );
}
```

## 🆚 Server Actions vs Route Handlers

| | Server Actions | Route Handlers |
| --- | --- | --- |
| Purpose | Mutations from **your own** UI | Public or external APIs, webhooks, non-React clients |
| HTTP | POST only (auto-generated endpoint) | Any method (GET, POST, PUT, DELETE…) with your own URL |
| Call style | Function call or `<form action>` | `fetch('/api/...')` |
| Progressive enhancement | ✅ Forms work without JS | ❌ |
| Type safety | End-to-end (shared TS types) | Manual |
| Revalidate and return UI in one round trip | ✅ | ❌ (the client must refetch) |
| Caching | Never cached | GET can be cached (opt-in) |
| Good for | Forms, likes, CRUD inside the app | Mobile apps, third-party integrations, webhooks, streaming |

## ⚠️ Common mistakes & gotchas

- Assuming actions are private because the button is only shown to admins. **Always check auth inside.**
- Calling `redirect()` inside `try/catch`: it throws, so the catch swallows the redirect.
- Using `useFormStatus` in the same component that renders the `<form>`. It must be in a child.
- Defining inline `'use server'` functions inside Client Components (not allowed). Import from a `'use server'` file.
- Using Server Actions for data **fetching**. They run sequentially and use POST; fetch in Server Components instead.
- Throwing for validation errors instead of returning them as state.
- Forgetting to revalidate, so the UI shows stale data after the mutation.

## 🎤 Say this in the interview

> "Server Actions are `'use server'` async functions that Next.js exposes as POST endpoints, so I can call them from forms or event handlers without writing an API. They support progressive enhancement, integrate with `useActionState`, `useFormStatus` and `useOptimistic`, and can revalidate and return fresh UI in one round trip. Since they're public endpoints, I always authenticate, authorise and validate with Zod inside every action."

## ❓ Likely interview questions

**What is a Server Action under the hood?**

An async server function with an encrypted ID. The client invokes it through a POST request to the page, and Next.js runs it and returns the result (plus an updated RSC payload if you revalidated).

**Inline vs file-level `'use server'`?**

Inline goes inside a Server Component's function and can capture variables. File-level makes every export an action and is required for importing into Client Components.

**How do you show validation errors and pending state?**

`useActionState` gives the returned state plus `isPending`. `useFormStatus` gives `pending` in a child of the form.

**Are Server Actions secure by default?**

No. They are reachable public endpoints. Next.js adds CSRF origin checks, encrypted closures and unguessable IDs, but you must handle auth, authorisation and input validation.

**What is progressive enhancement here?**

Forms using Server Actions submit as normal HTML POSTs before JS loads, then upgrade to client-side submission after hydration.

**How do you update the UI after a mutation?**

Call `updateTag`, `revalidateTag`, `revalidatePath` or `refresh()` in the action, optionally `redirect()`, and use `useOptimistic` for instant feedback.

**When would you use a Route Handler instead?**

For external consumers (mobile apps, webhooks, third parties), non-POST methods, custom URLs, or cached GET APIs.

---

⬅️ Prev: [Caching and Revalidation](../07-Caching-and-Revalidation/README.md) | Next ➡️: [Route Handlers (API)](../09-Route-Handlers-API/README.md)
