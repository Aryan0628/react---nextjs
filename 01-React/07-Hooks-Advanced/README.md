# 🧰 Advanced Hooks

> **TL;DR:** `useRef` stores a mutable value (or DOM node) without re-rendering. `useMemo` caches a **value**, `useCallback` caches a **function**, both to keep references stable or skip expensive work. `useReducer` centralises complex state logic. `useTransition` / `useDeferredValue` mark updates as non-urgent. React 19 adds `use`, `useActionState`, `useFormStatus`, `useOptimistic`, and the React Compiler can memoise automatically.

## 🧠 Concept

### Memoisation and the referential equality problem

Memoisation = **remember a result, reuse it while inputs are the same**. In React this matters because every render creates **new** objects and functions:

```jsx
function Parent() {
  const options = { theme: 'dark' };   // new object every render
  const onSave = () => save();         // new function every render
  return <Child options={options} onSave={onSave} />;
}
// {} !== {} and fn !== fn, so React.memo(Child) sees "changed props" every time,
// and any effect with [options] as a dep re-runs every render.
```

`useMemo` / `useCallback` return the **same reference** until dependencies change, which makes `React.memo` and dependency arrays actually work.

## ⚙️ How it works / Deep dive

### useRef

`useRef(initial)` returns a box `{ current }` that persists across renders. Changing `.current` does **not** trigger a re-render.

```jsx
// 1. DOM access
function Search() {
  const inputRef = useRef(null);
  return (
    <>
      <input ref={inputRef} />
      <button onClick={() => inputRef.current.focus()}>Focus</button>
    </>
  );
}

// 2. Mutable value with no re-render (timer id, render count, latest value)
const timerRef = useRef(null);
timerRef.current = setTimeout(doThing, 500);
clearTimeout(timerRef.current);

// 3. Previous value pattern
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => { ref.current = value; }, [value]); // updates after render
  return ref.current;                                  // value from last render
}
```

| | `useRef` | `useState` |
| --- | --- | --- |
| Re-render on change | No | Yes |
| Update timing | Immediate (synchronous mutation) | Next render (snapshot) |
| Use for | DOM nodes, timers, values not shown in UI | Anything displayed on screen |
| Read during render? | Avoid (except lazy init) | Yes |

### useMemo vs useCallback

```jsx
const sorted = useMemo(() => bigList.toSorted(compare), [bigList]); // caches a VALUE
const handleClick = useCallback(() => select(id), [id]);          // caches a FUNCTION
// useCallback(fn, deps) is exactly useMemo(() => fn, deps)
```

When it helps:

- **Expensive calculations** (filtering/sorting thousands of items) that would otherwise re-run on unrelated renders.
- Props passed to a **`React.memo` child**, so it can skip re-rendering.
- Values used as **dependencies** of other hooks (`useEffect`, another `useMemo`).

When it's useless (or worse, overhead):

- Cheap calculations (`a + b`, small arrays).
- Passing callbacks to plain DOM elements (`<button onClick>`), which don't compare props.
- Child isn't memoised, so it re-renders anyway.
- Deps change every render, so the cache never hits.

```jsx
const Row = React.memo(function Row({ item, onSelect }) {
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});

function List({ items }) {
  const [selected, setSelected] = useState(null);
  const onSelect = useCallback((id) => setSelected(id), []); // stable ref
  return items.map((it) => <Row key={it.id} item={it} onSelect={onSelect} />);
}
```

### React Compiler (auto-memoisation)

The **React Compiler** (stable v1.0 since late 2025) is a build-time plugin that analyses components and inserts memoisation automatically, so most manual `useMemo`, `useCallback` and `React.memo` become unnecessary. It assumes you follow the Rules of React (pure render, no mutation). Next.js 16 supports it via the `reactCompiler` config option. Still know the manual hooks for interviews and legacy code.

### useReducer

Prefer it over `useState` when state has **multiple related fields**, the next state depends on complex logic, or many event types update it. A **reducer** is a **pure** function `(state, action) => newState`: no side effects, no mutation, same input same output.

```jsx
const initial = { items: [], status: 'idle', error: null };

function cartReducer(state, action) {
  switch (action.type) {
    case 'added':
      return { ...state, items: [...state.items, action.item] };
    case 'removed':
      return { ...state, items: state.items.filter((i) => i.id !== action.id) };
    case 'cleared':
      return initial;
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function Cart() {
  const [state, dispatch] = useReducer(cartReducer, initial);
  return (
    <button onClick={() => dispatch({ type: 'added', item: { id: 1, name: 'Pen' } })}>
      Add ({state.items.length})
    </button>
  );
}
```

`dispatch` is stable (never changes), so it's safe to pass down without `useCallback`.

### useContext

`useContext(MyContext)` reads the nearest provider's value, avoiding prop drilling. Every consumer re-renders when the value changes. In React 19 you can render `<MyContext value={...}>` directly instead of `<MyContext.Provider>`. Details in the Context topic.

### useLayoutEffect vs useEffect

```text
render -> commit DOM -> useLayoutEffect (sync, blocks paint) -> PAINT -> useEffect (async)
```

- `useEffect`: default choice; doesn't block the screen.
- `useLayoutEffect`: when you must **measure the DOM** (size, position) and update **before** the user sees it, avoiding a flicker (tooltips, scroll position). Overuse hurts performance; it doesn't run on the server.

### useId

Generates a unique, **SSR-stable** ID for accessibility attributes. Not for list keys.

```jsx
const id = useId();
<><label htmlFor={id}>Email</label><input id={id} /></>;
```

### useTransition and useDeferredValue

Concurrent rendering lets React treat updates as **urgent** (typing, clicking: must feel instant) or **non-urgent** (re-rendering a huge filtered list). Non-urgent work can be interrupted by urgent work.

```jsx
// useTransition: you own the state setter, mark the slow update
function Search({ items }) {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState(items);
  const [isPending, startTransition] = useTransition();

  function onChange(e) {
    setQuery(e.target.value);                        // urgent: input stays snappy
    startTransition(() => setResults(filter(items, e.target.value))); // non-urgent
  }
  return <><input value={query} onChange={onChange} />{isPending && '...'}<List data={results} /></>;
}

// useDeferredValue: you only have the value (e.g. from props), defer a copy of it
function Results({ query, items }) {
  const deferredQuery = useDeferredValue(query);  // lags behind during heavy renders
  const list = useMemo(() => filter(items, deferredQuery), [deferredQuery]);
  return <List data={list} />;
}
```

In React 19, `startTransition` accepts **async** functions (these are "Actions").

### useImperativeHandle and refs to components

Lets a child expose a **custom, limited API** to a parent's ref instead of the raw DOM node. In **React 19, `ref` is a regular prop** for function components, so `forwardRef` is no longer needed (it still works but is slated for deprecation).

```jsx
// React 19
function FancyInput({ ref }) {
  const inputRef = useRef(null);
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    clear: () => { inputRef.current.value = ''; },
  }));
  return <input ref={inputRef} />;
}
// Older: const FancyInput = forwardRef((props, ref) => { ... });

// Parent: const r = useRef(); <FancyInput ref={r} />; r.current.focus();
```

### React 19 hooks

```jsx
// use(): read a promise (suspends until resolved) or a context. Can be called conditionally.
function Comments({ commentsPromise }) {
  const comments = use(commentsPromise);          // wrap in <Suspense fallback>
  return comments.map((c) => <p key={c.id}>{c.text}</p>);
}

// useActionState: action result + pending state for a form
const [state, formAction, isPending] = useActionState(async (prev, formData) => {
  return await saveName(formData.get('name'));
}, null);

// useFormStatus (react-dom): child of <form> reads its pending status
const { pending } = useFormStatus();

// useOptimistic: show the expected result instantly, auto-reverts if the action fails
const [optimisticTodos, addOptimistic] = useOptimistic(
  todos,
  (current, newTodo) => [...current, { ...newTodo, sending: true }]
);
async function addTodo(formData) {
  addOptimistic({ id: Date.now(), text: formData.get('text') }); // instant UI
  await createTodo(formData);                                     // real request
}
```

## 💻 Code examples

See the snippets above; each subsection is a self-contained example.

## ⚠️ Common mistakes & gotchas

- Reading or writing `ref.current` during render: makes output unpredictable; do it in effects or handlers.
- Expecting a ref change to update the UI: it won't; use state.
- Wrapping everything in `useMemo`/`useCallback`: adds cost and noise with no benefit.
- `useCallback` without `React.memo` on the child: no skipped renders.
- Mutating state inside a reducer, or doing side effects (fetch) in it.
- Using `useId` for list keys.
- `useLayoutEffect` for data fetching: blocks paint for no reason.

## 🎤 Say this in the interview

> `useRef` holds mutable values or DOM nodes without re-rendering; `useMemo` and `useCallback` preserve referential equality so memoised children and effect deps don't fire needlessly, though the React Compiler now handles most of that automatically. For complex state I use `useReducer`, and for heavy UI updates I mark them non-urgent with `useTransition` or `useDeferredValue`.

## ❓ Likely interview questions

**Difference between `useMemo` and `useCallback`?**

`useMemo` caches the result of a function; `useCallback` caches the function itself. `useCallback(fn, d)` equals `useMemo(() => fn, d)`.

**When would you use `useRef` instead of `useState`?**

For values that shouldn't trigger re-renders: DOM nodes, timer IDs, previous values, flags.

**When is `useReducer` better than `useState`?**

Multiple related state fields, complex transitions, or many action types; it also makes logic testable as a pure function.

**`useEffect` vs `useLayoutEffect`?**

`useEffect` runs after paint; `useLayoutEffect` runs synchronously after DOM mutation but before paint, for measurements that prevent flicker.

**`useTransition` vs `useDeferredValue`?**

Both mark work as non-urgent. Use `useTransition` when you control the state update; use `useDeferredValue` when you only receive the value.

**What changed with refs in React 19?**

`ref` is a normal prop for function components, so `forwardRef` isn't needed; ref callbacks can also return a cleanup function.

**What does `useOptimistic` do?**

Shows an optimistic version of state during an async action and automatically reverts to the real state when the action ends.

**Does the React Compiler replace `useMemo`?**

Mostly, yes: it auto-memoises components and values at build time, but manual hooks remain valid and are still needed without the compiler.

---

⬅️ Prev: [Hooks: useState and useEffect](../06-Hooks-useState-useEffect/README.md) | Next ➡️: [Custom Hooks](../08-Custom-Hooks/README.md)
