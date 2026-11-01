# 🎯 React Interview Q&A — Rapid Fire

> **TL;DR:** 50 high-frequency React questions with short, interview-ready answers, plus "what will this print?" snippets on batching, stale closures, effect order, keys and rendering quirks. Read the answer out loud — if you can explain it in 1–3 sentences, you're ready.

## 📘 Basics

**1. What is React and why use it?**

A JavaScript library for building UIs from reusable components. You describe UI declaratively as a function of state, and React efficiently updates the DOM when state changes.

**2. What is JSX?**

Syntax sugar that compiles to `jsx()` function calls (via `react/jsx-runtime`) producing plain objects (React elements). It's not HTML: use `className`, `htmlFor`, camelCase events, and `{}` for JS expressions.

**3. What is the Virtual DOM?**

A lightweight in-memory tree of React elements. On each render React builds a new tree, diffs it with the previous one, and applies only the minimal changes to the real DOM.

**4. What is reconciliation?**

The diffing process. Heuristics: elements of a different type rebuild the whole subtree; same type keeps the DOM node and updates attributes; list children are matched by `key`.

**5. What is React Fiber?**

The reconciler rewrite (React 16) that splits rendering into small units of work, allowing React to pause, prioritise and resume — the foundation for concurrent features.

**6. Props vs state?**

Props are inputs passed from the parent and are read-only. State is data owned by the component that changes over time and triggers re-renders.

**7. Why do lists need keys, and why not use the index?**

Keys identify items across renders so React can reuse the right DOM nodes and state. Indexes break when items are inserted, removed or reordered — state attaches to the wrong row.

**8. Controlled vs uncontrolled inputs?**

Controlled: value lives in React state (`value` + `onChange`). Uncontrolled: the DOM holds the value (`defaultValue`, read via ref or `FormData`).

**9. Function vs class components?**

Functions with hooks are the modern standard: less boilerplate, no `this`, logic reuse via custom hooks. Classes are legacy, except for error boundaries.

**10. What does `<StrictMode>` do?**

Dev-only checks: double-invokes render and effects (mount → unmount → mount) to expose impure code and missing cleanups, and warns about deprecated APIs. No effect in production.

**11. What are synthetic events?**

React's cross-browser wrapper around native events. Since React 17 listeners are attached to the root container, not `document`.

## 🪝 Hooks

**12. What are the Rules of Hooks?**

Call hooks only at the top level (not in loops, conditions or nested functions) and only from React functions or custom hooks — React relies on call order to match state. (`use()` is the exception that can be conditional.)

**13. `useState` vs `useReducer`?**

`useState` for simple independent values; `useReducer` when the next state depends on complex logic or several values change together — logic moves into a pure reducer.

**14. How do `useEffect` dependencies work?**

No array: runs after every render. `[]`: after mount only (cleanup on unmount). `[a, b]`: after mount and whenever `a` or `b` change (`Object.is`).

**15. `useEffect` vs `useLayoutEffect`?**

`useEffect` runs after the browser paints (non-blocking). `useLayoutEffect` runs synchronously after DOM mutation but before paint — use for measuring layout to avoid flicker.

**16. `useRef` vs `useState`?**

Both persist across renders, but changing `ref.current` does not re-render. Use refs for DOM nodes and mutable values not shown in UI.

**17. `useMemo` vs `useCallback`?**

`useMemo` caches a computed value; `useCallback` caches a function. Useful for expensive work, stable deps, or props to memoized children.

**18. What is a stale closure?**

A function captured values from an old render (e.g. an interval created with `[]` deps keeps seeing the initial state). Fix with functional updates, correct deps, or a ref / `useEffectEvent`.

**19. What is a custom hook?**

A function starting with `use` that composes other hooks to reuse stateful logic. Each component calling it gets its own independent state.

**20. Why does my effect run twice in development?**

Strict Mode deliberately mounts, unmounts and remounts to check that effects clean up properly. It doesn't happen in production.

**21. How do you fetch data safely in `useEffect`?**

Use a cleanup with an `AbortController` or an `ignore` flag to avoid race conditions and setting state after unmount. In real apps prefer TanStack Query or framework data fetching.

## ⚡ Rendering & Performance

**22. What causes a component to re-render?**

Its state changes, its parent re-renders, or a context it consumes changes. Props "changing" is only a result of the parent rendering.

**23. What does `React.memo` do?**

Skips re-rendering a component if its props are shallowly equal to last render. Inline objects/functions defeat it unless memoized.

**24. Is `setState` synchronous?**

No. Updates are queued and batched; the new value is visible on the next render, not on the next line.

**25. What is automatic batching?**

Since React 18, multiple state updates anywhere (events, promises, timeouts) in the same tick produce a single re-render.

**26. What is the React Compiler?**

A build-time tool (v1.0 stable) that automatically memoizes components and values, making most manual `memo`/`useMemo`/`useCallback` unnecessary.

**27. How do you code-split in React?**

`React.lazy(() => import("./X"))` wrapped in `<Suspense fallback>`; frameworks also split per route automatically.

**28. How do you render huge lists efficiently?**

Virtualisation (react-window, TanStack Virtual) renders only visible rows; also pagination or infinite scroll.

**29. Why not define a component inside another component?**

It becomes a new component type on every render, so React unmounts and remounts it, losing state and focus.

## 🗃️ State

**30. What is "lifting state up"?**

Moving shared state to the closest common parent and passing it down via props so siblings stay in sync.

**31. What is prop drilling and how do you avoid it?**

Passing props through components that don't need them. Fix with composition (`children`), Context, or a store.

**32. Context vs Redux?**

Context is a transport mechanism; all consumers re-render on value change. Redux/Zustand are state managers with selectors, middleware and DevTools for large, frequently changing state.

**33. Describe the Redux data flow.**

UI dispatches an action → middleware → reducer computes new state → store updates → subscribed components whose selected data changed re-render.

**34. Server state vs client state?**

Server state is a cache of remote data (needs fetching, caching, revalidation — TanStack Query). Client state is UI-owned (modals, form inputs, theme).

**35. Why must state be immutable?**

React compares references (`Object.is`) to detect changes. Mutating keeps the same reference, so React may skip the update; immutability also enables cheap memo checks.

## 🧪 Advanced

**36. What is an error boundary?**

A class component with `getDerivedStateFromError`/`componentDidCatch` that catches render errors below it and shows a fallback. Doesn't catch event handler or async errors.

**37. What is a portal?**

`createPortal(child, domNode)` renders into a different DOM node (modals, tooltips) while keeping React context and event bubbling through the React tree.

**38. What is Suspense?**

A boundary that shows a fallback while children are loading code (`lazy`) or data (`use`, framework/library integrations).

**39. What does `useTransition` do?**

Marks a state update as non-urgent so urgent updates (typing) can interrupt it; `isPending` lets you show a loading hint.

**40. Server Components vs Client Components?**

Server Components run only on the server, can be async and access data directly, and ship zero JS. Client Components (`"use client"`) run in the browser and can use state, effects and event handlers.

**41. What is hydration and what causes hydration mismatches?**

Hydration attaches React to server-rendered HTML. Mismatches happen when the client's first render differs from the server HTML — e.g. `Date.now()`, `Math.random()`, `window` checks during render.

**42. Name the new hooks in React 19.**

`use`, `useActionState`, `useOptimistic`, and `useFormStatus` (from `react-dom`). React 19.2 added `useEffectEvent`.

**43. Do we still need `forwardRef`?**

Not in React 19 — `ref` is a regular prop for function components. `forwardRef` still works but will be deprecated.

**44. HOC vs custom hook?**

Both reuse logic; HOCs wrap components (extra tree layers, prop collisions), custom hooks share logic directly inside a component — preferred today.

## 🧩 Output-based / tricky snippets

### 45. Batching — what shows after one click, and what is logged?

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  function handleClick() {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
    console.log(count);
  }
  return <button onClick={handleClick}>{count}</button>;
}
```

Logs `0`, button shows `1`. Each call uses the same snapshot `count = 0`, so all three queue "set to 1". Batching means one re-render. Use `setCount((c) => c + 1)` three times to get `3`.

### 46. Stale closure — what does this display over 5 seconds?

```jsx
function Timer() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => setCount(count + 1), 1000);
    return () => clearInterval(id);
  }, []);
  return <p>{count}</p>;
}
```

It shows `1` and stays at `1`. The interval closure captured `count = 0` from the first render, so it keeps setting `0 + 1`. Fix: `setCount((c) => c + 1)`.

### 47. Effect order — what is the log order (no Strict Mode)?

```jsx
function Child() {
  console.log("Child render");
  useEffect(() => console.log("Child effect"), []);
  return null;
}

function Parent() {
  console.log("Parent render");
  useLayoutEffect(() => console.log("Parent layout effect"), []);
  useEffect(() => console.log("Parent effect"), []);
  return <Child />;
}
```

```text
Parent render
Child render
Parent layout effect
Child effect
Parent effect
```

Render goes top-down; effects run bottom-up (children before parents) after commit, and all layout effects run before any passive `useEffect`. In Strict Mode dev you'd also see double renders and an extra cleanup + re-run of each effect.

### 48. Keys and state — does the count reset when toggling?

```jsx
function App() {
  const [isFancy, setIsFancy] = useState(false);
  return (
    <>
      {isFancy ? <Counter color="pink" /> : <Counter color="blue" />}
      <button onClick={() => setIsFancy((f) => !f)}>Toggle</button>
    </>
  );
}
```

No — the count is preserved. Same component type at the same position = same instance; only the `color` prop changes. To reset, give different keys: `<Counter key={isFancy ? "fancy" : "plain"} />`. Changing `key` is the idiomatic way to reset a component's state.

### 49. What renders when `count` is 0?

```jsx
function Messages({ count }) {
  return <div>{count && <p>You have {count} messages</p>}</div>;
}
```

It renders the text `0`. `0 && ...` evaluates to `0`, and React renders numbers (it skips only `false`, `null`, `undefined` and `true`). Fix: `{count > 0 && ...}` or a ternary.

### 50. Mutation — does the UI update?

```jsx
function Profile() {
  const [user, setUser] = useState({ name: "Aryan" });
  function rename() {
    user.name = "Gupta";
    setUser(user);
  }
  return <button onClick={rename}>{user.name}</button>;
}
```

No re-render — the button still shows "Aryan". `setUser` receives the same object reference, `Object.is(old, new)` is `true`, so React bails out. Fix: `setUser({ ...user, name: "Gupta" })`.

## 🎤 Final interview tips

- Start every answer with the **one-line definition**, then the **why**, then a tiny example.
- Mention trade-offs ("memo has a cost", "context re-renders all consumers") — it signals real experience.
- Know what's new: React 19 Actions, `use`, `useOptimistic`, ref as prop, Server Components, React Compiler.
- If unsure, reason out loud from first principles: "state change → render → diff → commit".

⬅️ Prev: [16 React Router](../16-React-Router/README.md) | Next ➡️: [Next.js](../../02-NextJS/README.md)
