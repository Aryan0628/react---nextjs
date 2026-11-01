# ⏳ Suspense, Lazy & Concurrent React (React 18 → 19)

> **TL;DR:** `<Suspense>` shows a fallback while something inside is "not ready" — lazy-loaded code or data. **Concurrent rendering** (React 18) makes rendering interruptible so urgent updates (typing) aren't blocked by slow ones (filtering a big list). React 19 builds on it with Actions, `use()`, `useOptimistic`, `useActionState`, Server Components and the React Compiler.

## 🧠 Concept

### Suspense

Suspense is a **declarative loading boundary**. Instead of every component juggling `isLoading` flags, a component can "suspend" (say "I'm not ready"), and the nearest `<Suspense fallback={...}>` above it shows the fallback until it's ready.

What can suspend:

- `React.lazy` components whose JS chunk is still downloading.
- Data read with `use(promise)` or via Suspense-enabled frameworks/libraries (Next.js Server Components, TanStack Query's `useSuspenseQuery`, Relay).
- Plain `useEffect` fetching does **not** trigger Suspense.

### Concurrent rendering

Before React 18, rendering was **synchronous and blocking**: once React started rendering an update, it couldn't stop until done. A slow render froze typing.

Concurrent React can **pause, interrupt, resume or throw away** a render. Analogy: a chef cooking a big slow stew (low-priority render) who stops mid-chop to hand a customer their coffee (urgent keystroke), then goes back to the stew. If the order changes, the half-made stew is tossed and restarted with the new order — the customer never sees a half-cooked dish, because React only commits complete results.

You opt in **per update** using transitions; nothing is concurrent unless you mark it.

## ⚙️ How it works / Deep dive

### React 18 features

| Feature | What it does |
| --- | --- |
| `createRoot` | New root API (`ReactDOM.render` removed in 19). Enables concurrent features. |
| Automatic batching | Multiple `setState` calls are batched into one render **everywhere** — in timeouts, promises, native listeners — not just React event handlers. |
| `useTransition` / `startTransition` | Mark an update as non-urgent. Urgent updates interrupt it; `isPending` tells you it's in progress. |
| `useDeferredValue` | Get a "lagging" copy of a value; React renders with the old value first, then the new one in the background. |
| Streaming SSR | Server sends HTML in chunks; Suspense boundaries stream in as their data resolves. |
| Selective hydration | Parts of the page hydrate independently; React prioritises hydrating what the user interacts with. |
| `useId` | Stable unique IDs that match between server and client (for `aria`/`htmlFor`). |

### Nested Suspense boundaries

Boundaries can be nested to reveal UI progressively: the page shell shows first, then sections pop in as they're ready. Each boundary is independent — a slow comments section doesn't block the article.

### `useTransition` vs `useDeferredValue`

- `useTransition` — you **own the state setter** and wrap the update: `startTransition(() => setTab("posts"))`.
- `useDeferredValue` — you **receive a value** (e.g. a prop) and want an expensive child to lag behind it.
- Transitions also stop already-visible content from being replaced by a Suspense fallback; React keeps the old UI until the new one is ready.

### React 19 features summary

| Feature | One-liner |
| --- | --- |
| Actions | Async functions passed to transitions or `<form action={fn}>`; React manages pending state, errors and optimistic updates. |
| `useActionState(action, initialState)` | Returns `[state, formAction, isPending]` — form result + pending flag for an Action. |
| `useFormStatus()` (`react-dom`) | Inside a `<form>`, read `{ pending, data, method }` of the parent form — e.g. disable submit button. |
| `useOptimistic(state, updateFn)` | Show the expected result immediately while the Action runs; reverts automatically on failure. |
| `use(resource)` | Read a promise (suspends until resolved) or a context; can be called conditionally. |
| `ref` as a prop | No more `forwardRef` for function components; ref callbacks may return cleanup. |
| `<Context value>` as provider | Render the context directly instead of `<Context.Provider>`. |
| Document metadata | `<title>`, `<meta>`, `<link>` inside any component are hoisted into `<head>`. |
| Stylesheet & resource APIs | `<link rel="stylesheet" precedence>`, `preload`, `preinit` for resource loading. |
| Server Components | Components that run only on the server (zero client JS), can be `async` and fetch data directly. Used through frameworks like Next.js. |
| Server Functions (Actions) | Functions marked `"use server"` that the client can call like normal functions; the framework does the network request. |
| React Compiler | Build-time automatic memoization (v1.0 stable, separate from React itself). |
| React 19.2 additions | `<Activity mode="hidden">` (keep hidden UI mounted with state) and `useEffectEvent` (non-reactive logic in effects). |

## 💻 Code examples

### Lazy loading with nested Suspense

```jsx
import { lazy, Suspense } from "react";

const Settings = lazy(() => import("./Settings")); // must default-export a component

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>          {/* outer: whole page */}
      <Header />
      <Suspense fallback={<p>Loading settings…</p>}> {/* inner: just this section */}
        <Settings />
      </Suspense>
    </Suspense>
  );
}
```

### Data with `use()`

```jsx
import { use, Suspense } from "react";

// The promise must be created OUTSIDE render (cached, from a parent/server, or a library),
// otherwise a new promise is created every render and it suspends forever.
function Comments({ commentsPromise }) {
  const comments = use(commentsPromise); // suspends until resolved; errors go to error boundary
  return comments.map((c) => <p key={c.id}>{c.text}</p>);
}

function Post({ id }) {
  const commentsPromise = useMemo(() => fetchComments(id), [id]);
  return (
    <Suspense fallback={<p>Loading comments…</p>}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  );
}
```

### Transitions keep typing responsive

```jsx
function SearchPage() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState(allItems);
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    setQuery(e.target.value);                // urgent: input updates instantly
    startTransition(() => {
      setResults(filterHuge(e.target.value)); // non-urgent: can be interrupted
    });
  }

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <span>Updating…</span>}
      <ResultList items={results} />
    </>
  );
}

// Same idea with useDeferredValue when you only receive the value
function Results({ query }) {
  const deferredQuery = useDeferredValue(query);
  return <SlowList query={deferredQuery} />; // wrap SlowList in memo so it can skip
}
```

### React 19 form Action

```jsx
import { useActionState, useOptimistic } from "react";
import { useFormStatus } from "react-dom";

function SubmitButton() {
  const { pending } = useFormStatus(); // reads the parent <form>
  return <button disabled={pending}>{pending ? "Saving…" : "Save"}</button>;
}

function NameForm({ currentName }) {
  const [optimisticName, setOptimisticName] = useOptimistic(currentName);
  const [error, formAction] = useActionState(async (prevError, formData) => {
    const name = formData.get("name");
    setOptimisticName(name);           // show new name immediately
    const res = await updateName(name);
    return res.ok ? null : "Failed to save";
  }, null);

  return (
    <form action={formAction}>
      <title>{`Profile — ${optimisticName}`}</title> {/* hoisted to <head> */}
      <p>Hello, {optimisticName}</p>
      <input name="name" />
      <SubmitButton />
      {error && <p role="alert">{error}</p>}
    </form>
  );
}
```

## ⚠️ Common mistakes & gotchas

- `React.lazy` inside a component body → new lazy component each render, remounts. Declare at module top level.
- No `<Suspense>` above a suspending component → React throws an error.
- Creating the promise for `use()` during render → infinite suspend loop.
- Wrapping controlled-input updates in `startTransition` → input lags; only wrap the expensive update.
- Thinking transitions make code faster — they make it **feel** responsive by prioritising.
- `useFormStatus` must be in a component rendered **inside** the `<form>`, not the one rendering it.

## 🎤 Say this in the interview

> Suspense is a declarative loading boundary for lazy code and Suspense-enabled data. Concurrent rendering, from React 18, makes rendering interruptible so I can mark slow updates as transitions and keep input responsive. React 19 adds Actions with `useActionState`, `useOptimistic` and `useFormStatus`, the `use` API, ref as a prop, document metadata, and Server Components that frameworks like Next.js build on.

## ❓ Likely interview questions

**What is concurrent rendering?**

React can interrupt, pause and resume rendering, prioritising urgent updates over transitions. It only commits complete trees, so users never see half-rendered UI.

**What is automatic batching?**

Since React 18, multiple state updates in the same tick — even inside promises or timeouts — are combined into a single re-render.

**`useTransition` vs `useDeferredValue`?**

`useTransition` wraps a state update you control; `useDeferredValue` defers a value you receive. Both mark work as non-urgent.

**How does `use()` differ from other hooks?**

It can be called conditionally and in loops, and it reads promises (suspending) or context.

**What does `useOptimistic` do?**

Shows an assumed result immediately while an async Action runs, and reverts to real state when it finishes or fails.

**What are streaming SSR and selective hydration?**

The server streams HTML per Suspense boundary as data resolves; the client hydrates boundaries independently, prioritising ones the user interacts with.

**Are Server Components a React or Next.js feature?**

A React feature (architecture + `"use server"`/`"use client"` directives); frameworks like Next.js provide the bundler and server integration to use them.

⬅️ Prev: [14 Error Boundaries, Refs & Portals](../14-Error-Boundaries-Refs-Portals/README.md) | Next ➡️: [16 React Router](../16-React-Router/README.md)
