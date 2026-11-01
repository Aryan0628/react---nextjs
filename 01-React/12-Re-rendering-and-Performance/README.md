# 🚀 Re-rendering & Performance

> **TL;DR:** A component re-renders when its state changes, its parent re-renders, or a context it reads changes. A re-render is just calling your function again — React only touches the DOM for what actually changed. Optimise by measuring first, then: colocate state, pass JSX as `children`, memoize (or let the React Compiler do it), code-split, and virtualise long lists.

## 🧠 Concept

### What triggers a re-render

1. **Its own state changes** (`setState` with a new value — `Object.is` comparison).
2. **Its parent re-renders** — by default, all children re-render too, recursively.
3. **A context it consumes changes.**

### The big misconception

"A component re-renders because its props changed" — **not quite**. Props can only change because the **parent re-rendered**. And when the parent re-renders, the child re-renders **even if its props are identical**. Props comparison only matters when you opt in with `React.memo`.

### Render ≠ DOM update

Rendering = React calls your component to get new JSX (cheap-ish JS work). Then React **diffs** (reconciles) the new tree against the old one and **commits** only the differences to the real DOM. So a re-render that produces the same output costs CPU time but causes no DOM change. Re-renders become a problem only when they're expensive or extremely frequent.

Analogy: re-rendering is a chef re-reading the recipe; committing is actually re-plating the dish. Re-reading is cheap unless the recipe is 300 pages long.

## ⚙️ How it works / Deep dive

### React.memo and shallow comparison

`React.memo(Component)` skips re-rendering when every prop is `Object.is`-equal to last time. Primitives compare by value; **objects, arrays and functions compare by reference**. So this breaks memo:

```jsx
const Chart = React.memo(function Chart({ options, onSelect }) { /* heavy */ });

function Dashboard() {
  const [count, setCount] = useState(0);
  // ❌ new object and new function every render → memo never skips
  return <Chart options={{ type: "bar" }} onSelect={() => console.log("hi")} />;
}
```

Fix by keeping references stable:

```jsx
function Dashboard() {
  const [count, setCount] = useState(0);
  const options = useMemo(() => ({ type: "bar" }), []);          // same object each render
  const onSelect = useCallback((id) => console.log(id), []);     // same function each render
  return <Chart options={options} onSelect={onSelect} />;        // ✅ memo can skip now
}
```

`useMemo`/`useCallback` alone do nothing for performance unless the receiver is memoized (or the value is an effect dependency, or the computation is genuinely expensive).

### Structural fixes (often better than memo)

- **State colocation / move state down:** if only a small widget uses some state, put the state inside that widget so typing in it doesn't re-render the whole page.
- **Lift content up (children as props):** a component that owns state re-renders its own JSX, but `children` was created by the parent, so it's the same element reference and React skips it.

```jsx
// ❌ Every scroll re-renders ExpensiveTree
function Page() {
  const [y, setY] = useState(0);
  return <div onScroll={(e) => setY(e.target.scrollTop)}><ExpensiveTree /></div>;
}

// ✅ ScrollTracker owns the state; ExpensiveTree is passed in as children and not re-rendered
function ScrollTracker({ children }) {
  const [y, setY] = useState(0);
  return <div onScroll={(e) => setY(e.target.scrollTop)}>{children}</div>;
}
function Page() {
  return <ScrollTracker><ExpensiveTree /></ScrollTracker>;
}
```

### React Compiler (auto memoization)

The React Compiler (stable v1.0 since late 2025) is a build-time Babel/SWC plugin that analyses components and hooks and **automatically inserts memoization** — effectively `useMemo`, `useCallback` and `React.memo` everywhere it's safe. It requires your code to follow the Rules of React (pure render, no mutation of props/state). Enable in Next.js 16 with `reactCompiler: true` in `next.config`. With it, manual memo becomes mostly unnecessary.

### Other techniques

| Technique | What it does |
| --- | --- |
| Code splitting (`React.lazy` + `Suspense`) | Load a component's JS only when it's rendered (routes, modals, charts). |
| List virtualisation (react-window, TanStack Virtual) | Render only the ~20 visible rows of a 10,000-row list. |
| Debounce / throttle | Debounce: run after the user stops typing (search). Throttle: at most once per N ms (scroll, resize). |
| `useDeferredValue` / `useTransition` | Keep input responsive while a slow render happens in the background. |
| Avoid unnecessary effects | Derive values during render instead of `useEffect` + `setState` (which causes an extra render). |
| Stable, unique keys | Use IDs, not indexes, so React reuses the right DOM nodes and preserves state. |
| Images | Correct sizes, modern formats (WebP/AVIF), `loading="lazy"`, explicit width/height to avoid layout shift. |

### Measuring: React DevTools

- **Profiler tab:** record an interaction, see which components rendered, how long each took, and "why did this render?" (enable in settings).
- **"Highlight updates when components render":** flashes a border around components as they re-render — great for spotting unexpected cascades.
- Rule: **measure, then optimise.** Most re-renders are fine.

### Web Vitals (brief)

| Metric | Measures | Good |
| --- | --- | --- |
| LCP — Largest Contentful Paint | Loading: when the main content appears | ≤ 2.5 s |
| INP — Interaction to Next Paint | Responsiveness to clicks/typing (replaced FID in 2024) | ≤ 200 ms |
| CLS — Cumulative Layout Shift | Visual stability: content jumping around | ≤ 0.1 |

## 💻 Code examples

```jsx
// Code splitting
const AdminPanel = lazy(() => import("./AdminPanel")); // separate JS chunk

function App({ isAdmin }) {
  return (
    <Suspense fallback={<Spinner />}>
      {isAdmin && <AdminPanel />}
    </Suspense>
  );
}
```

```jsx
// Virtualised list (react-window v1 API; v2 renamed it to <List rowComponent rowCount rowHeight>)
import { FixedSizeList } from "react-window";

function BigList({ items }) {
  return (
    <FixedSizeList height={400} itemCount={items.length} itemSize={35} width="100%">
      {({ index, style }) => <div style={style}>{items[index].name}</div>}
    </FixedSizeList>
  );
}
```

```jsx
// Debounced search with a custom hook
function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id); // reset timer on every keystroke
  }, [value, delay]);
  return debounced;
}

function Search() {
  const [q, setQ] = useState("");
  const debouncedQ = useDebounce(q);
  useEffect(() => { if (debouncedQ) fetch(`/api/search?q=${debouncedQ}`); }, [debouncedQ]);
  return <input value={q} onChange={(e) => setQ(e.target.value)} />;
}
```

```jsx
// ❌ Unnecessary effect: extra render + out-of-sync risk
const [fullName, setFullName] = useState("");
useEffect(() => setFullName(`${first} ${last}`), [first, last]);

// ✅ Derive during render
const fullName = `${first} ${last}`;
```

## ⚠️ Common mistakes & gotchas

- Wrapping everything in `useMemo`/`useCallback` "just in case" — adds cost and noise without a memoized consumer.
- `React.memo` on a component that receives `children` or inline objects — always re-renders anyway.
- Using array index as `key` in lists that reorder/insert → wrong state attached to wrong row.
- Defining a component **inside** another component → it's a new type each render, so it remounts and loses state.
- Optimising in dev mode — Strict Mode double renders and dev builds are slower; profile production builds.
- Putting fast-changing values in a widely consumed Context.

## 🎤 Say this in the interview

> A component re-renders when its state changes, its parent renders, or its context changes — props changing is just a consequence of the parent rendering. Rendering isn't a DOM update; React diffs and commits only what changed. I measure with the Profiler first, then fix structurally by colocating state and passing children, memoize where it helps, and in new projects let the React Compiler handle memoization.

## ❓ Likely interview questions

**Does a child re-render if its props didn't change?**

Yes, if its parent re-rendered — unless the child is wrapped in `React.memo` and all props are shallow-equal.

**What's the difference between `useMemo` and `useCallback`?**

`useMemo` caches a computed value; `useCallback` caches a function reference. `useCallback(fn, deps)` equals `useMemo(() => fn, deps)`.

**Why doesn't `React.memo` work with inline objects or arrow functions?**

They're recreated each render, so the shallow comparison sees a new reference every time.

**What is the React Compiler?**

A build-time tool that automatically memoizes components and values, removing most manual `memo`/`useMemo`/`useCallback`. It relies on code following the Rules of React.

**How do you render a list of 10,000 items efficiently?**

Virtualise it (react-window / TanStack Virtual) so only visible rows are in the DOM; also paginate or use infinite scroll.

**Debounce vs throttle?**

Debounce waits until activity stops (search input). Throttle runs at most once per interval (scroll handlers).

**How do you find what's causing slow renders?**

React DevTools Profiler (flamegraph, render durations, why-did-this-render) and "highlight updates"; the browser Performance panel for long tasks.

**What are LCP, INP and CLS?**

Core Web Vitals: loading speed of main content, responsiveness to interactions, and visual stability.

⬅️ Prev: [11 State Management](../11-State-Management-Redux-Zustand/README.md) | Next ➡️: [13 Design Patterns](../13-Design-Patterns/README.md)
