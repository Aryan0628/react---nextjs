# 🛡️ Error Boundaries, Refs & Portals

> **TL;DR:** **Error boundaries** catch render errors in a subtree and show a fallback instead of a blank page (still class-based, or use `react-error-boundary`). **Refs** give direct access to DOM nodes or mutable values; in React 19 `ref` is a normal prop and ref callbacks can return cleanup. **Portals** render children into a different DOM node (modals, tooltips) while staying in the same React tree.

## 🧠 Concept

### Error boundaries

Since React 16, an uncaught error during rendering **unmounts the entire React tree** — the user sees a white screen. React's reasoning: showing broken/corrupted UI is worse than showing nothing. An error boundary is a **try/catch for components**: it wraps a subtree, and if anything inside throws while rendering, it shows a fallback UI instead and the rest of the app keeps working.

### Refs

A ref is a **box that survives re-renders** (`ref.current`) and **doesn't trigger a render when changed**. Two uses: grab a DOM node (focus, scroll, measure, integrate a non-React library) or store mutable values (timer IDs, previous value).

### Portals

Sometimes a component must _visually_ escape its parent — a modal inside a card with `overflow: hidden` or a low `z-index` stacking context gets clipped. A portal renders the element into another DOM node (e.g. `document.body`) while it **logically** stays where you wrote it: it still gets context and its events bubble to React parents.

## ⚙️ How it works / Deep dive

### Writing an error boundary (class)

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  // 1. Render phase: update state so next render shows the fallback
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  // 2. Commit phase: side effects such as logging
  componentDidCatch(error, info) {
    logToSentry(error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? <h2>Something went wrong.</h2>;
    }
    return this.props.children;
  }
}

<ErrorBoundary fallback={<p>Chart failed to load</p>}>
  <RevenueChart />
</ErrorBoundary>
```

### What error boundaries do NOT catch

| Not caught | Why | How to handle |
| --- | --- | --- |
| Event handlers | Run outside rendering | Regular `try/catch`, set error state, or `showBoundary(error)` |
| Async code (`setTimeout`, promises, `fetch`) | Happens after render finished | `try/catch` / `.catch()`, then set state or `showBoundary` |
| Server-side rendering errors | Boundary runs on the client | Framework error pages (Next.js `error.tsx`) |
| Errors in the boundary itself | It can't catch its own crash | Put another boundary above it |

Tip: rethrowing inside a `setState` updater (`setState(() => { throw err; })`) or using `showBoundary` forwards async errors into the nearest boundary. React 19 also adds root-level hooks `createRoot(el, { onCaughtError, onUncaughtError })` for global logging.

### react-error-boundary (what people actually use)

```jsx
import { ErrorBoundary, useErrorBoundary } from "react-error-boundary";

function Fallback({ error, resetErrorBoundary }) {
  return (
    <div role="alert">
      <p>Error: {error.message}</p>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

<ErrorBoundary
  FallbackComponent={Fallback}
  onReset={() => queryClient.resetQueries()} // clear whatever caused the error
  resetKeys={[userId]}                        // auto-reset when userId changes
>
  <UserProfile userId={userId} />
</ErrorBoundary>

// Forward async/event errors into the boundary
function SaveButton() {
  const { showBoundary } = useErrorBoundary();
  return <button onClick={() => save().catch(showBoundary)}>Save</button>;
}
```

### Placement strategy

- **Top-level** boundary: last-resort "Oops, reload" page.
- **Per route/section**: a failing sidebar doesn't kill the main content.
- **Per risky widget**: charts, third-party embeds, user-generated content.
- Pair with `<Suspense>`: Suspense handles _loading_, error boundary handles _failure_.

### Refs in depth

- `useRef(initial)` → `{ current }`. Attach with `<input ref={inputRef} />`; React sets `.current` after commit and resets it on unmount.
- **forwardRef (pre-19):** function components couldn't receive `ref` as a prop, so you wrapped them: `forwardRef((props, ref) => ...)`.
- **React 19:** `ref` is a regular prop for function components. `forwardRef` still works but is no longer needed and will be deprecated.
- **Callback refs:** pass a function instead of an object; React calls it with the node. Useful for measuring, or refs inside lists.
- **React 19 ref cleanup:** a ref callback may return a cleanup function, called when the node is removed (instead of calling the callback with `null`).
- `useImperativeHandle(ref, () => ({ focus, clear }))` exposes a limited custom API instead of the raw DOM node.

### Portals in depth

- `createPortal(children, domNode)` from `react-dom`.
- **Event bubbling follows the React tree, not the DOM tree**: a click inside a portal-rendered modal bubbles to the React parent's `onClick`, even though in the DOM the modal is under `body`.
- Context, state and refs work normally across the portal.

### Fragments (brief)

`<>...</>` or `<Fragment key={id}>` groups children without adding a wrapper `div` — keeps DOM clean and avoids breaking table/flex/grid layouts. Use the long form when you need a `key` in a list.

## 💻 Code examples

```jsx
// React 19: ref as a plain prop — no forwardRef
function TextInput({ ref, ...props }) {
  return <input ref={ref} {...props} />;
}

function Form() {
  const inputRef = useRef(null);
  return (
    <>
      <TextInput ref={inputRef} placeholder="Name" />
      <button onClick={() => inputRef.current.focus()}>Focus</button>
    </>
  );
}

// Pre-19 equivalent
const OldTextInput = forwardRef(function OldTextInput(props, ref) {
  return <input ref={ref} {...props} />;
});
```

```jsx
// Callback ref with React 19 cleanup
function Measured() {
  return (
    <div
      ref={(node) => {
        const observer = new ResizeObserver(([entry]) => console.log(entry.contentRect.width));
        observer.observe(node);
        return () => observer.disconnect(); // runs when the node is removed
      }}
    >
      Resize me
    </div>
  );
}
```

```jsx
// Modal with a portal
import { createPortal } from "react-dom";

function Modal({ open, onClose, children }) {
  if (!open) return null;
  return createPortal(
    <div className="backdrop" onClick={onClose}>
      <div className="modal" role="dialog" aria-modal="true" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.body // rendered here, escaping overflow/z-index of parents
  );
}
```

## ⚠️ Common mistakes & gotchas

- Expecting error boundaries to catch `onClick` or `fetch` errors — they don't.
- One boundary around the whole app only → any widget failure blanks everything.
- Fallback with no reset path → user stuck; provide "Try again" and `resetKeys`.
- Reading `ref.current` during render — it's `null` on first render and mutating it doesn't re-render.
- Using refs for values the UI displays → use state instead.
- Portal event surprise: clicks inside the modal trigger parent `onClick` handlers via React bubbling.
- Portals and SSR: `document` doesn't exist on the server — render portals only after mount.

## 🎤 Say this in the interview

> An error boundary is a class component using `getDerivedStateFromError` and `componentDidCatch` that catches render errors in its subtree and shows a fallback; it doesn't catch event-handler or async errors, which I handle with try/catch or `showBoundary` from `react-error-boundary`. In React 19, `ref` is a normal prop so `forwardRef` isn't needed, and portals let modals escape overflow and z-index while events still bubble through the React tree.

## ❓ Likely interview questions

**Why do we need error boundaries?**

Without one, a render error unmounts the whole app. Boundaries isolate failures and show a fallback.

**Can you write an error boundary with hooks?**

No — it needs `getDerivedStateFromError`/`componentDidCatch`, which only exist on classes. Use a class or `react-error-boundary`.

**What don't error boundaries catch?**

Event handlers, async code, SSR errors, and errors thrown in the boundary itself.

**What changed about refs in React 19?**

`ref` is passed as a regular prop to function components (no `forwardRef`), and ref callbacks can return a cleanup function.

**What is a callback ref?**

A function passed to `ref` that React calls with the DOM node — useful for measuring or setting up observers.

**What is a portal and when do you use it?**

`createPortal` renders children into another DOM node — for modals, tooltips, dropdowns that must escape `overflow: hidden` or stacking contexts.

**How do events behave inside a portal?**

They bubble according to the React component tree, so a parent component's handlers still fire.

⬅️ Prev: [13 Design Patterns](../13-Design-Patterns/README.md) | Next ➡️: [15 Suspense, Lazy & Concurrent React](../15-Suspense-Lazy-Concurrent-React/README.md)
