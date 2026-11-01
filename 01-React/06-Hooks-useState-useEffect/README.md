# 🪝 Hooks: useState and useEffect

> **TL;DR:** Hooks let function components hold state and run side effects. `useState` gives a value that survives re-renders plus a setter that schedules a re-render (batched, with a **snapshot** per render). `useEffect` syncs your component with something **outside** React after render, with a dependency array controlling when it re-runs and a cleanup that undoes it.

## 🧠 Concept

### What hooks are and why they exist

Hooks (React 16.8) are functions starting with `use` that "hook into" React features from function components. They replaced classes because classes had real problems:

- **Logic reuse was awkward**: HOCs and render props caused "wrapper hell".
- **Related logic was split** across lifecycles (subscribe in `componentDidMount`, unsubscribe in `componentWillUnmount`), while unrelated logic got mixed in one method.
- **`this` confusion**: binding handlers, stale `this.props`.
- Classes are harder for tools to optimise and minify.

With hooks, one feature's logic lives together and can be extracted into a **custom hook**.

### Rules of Hooks and WHY

1. Call hooks only at the **top level**: not inside `if`, loops, or nested functions.
2. Call hooks only from **React function components or custom hooks**.

Why: React does not store hooks by name. It keeps a **linked list** of hook slots per component and matches them by **call order**. Render 1: slot 1 is `useState(name)`, slot 2 is `useEffect`. If a condition skips a hook on render 2, every later hook reads the wrong slot.

```text
Render 1:  useState(name) -> slot 1   useState(age) -> slot 2   useEffect -> slot 3
Render 2:  (if skipped)               useState(age) -> slot 1 ❌ gets "name" value
```

The ESLint plugin `eslint-plugin-react-hooks` enforces these rules. (React 19's `use()` is the one exception: it may be called conditionally.)

## ⚙️ How it works / Deep dive

### useState essentials

```jsx
const [count, setCount] = useState(0);   // initial value used only on first render

// Lazy init: pass a FUNCTION so the expensive work runs only once
const [todos, setTodos] = useState(() => JSON.parse(localStorage.getItem('todos')) ?? []);
// ❌ useState(JSON.parse(...)) would re-parse on every render (result ignored)
```

### Updates are async and batched

`setX` does not change the variable immediately. It **queues** an update and React re-renders later. Since **React 18, automatic batching** groups multiple updates into one render everywhere: event handlers, `setTimeout`, promises, native listeners. (Before 18, only React event handlers were batched.)

### State is a snapshot per render

Each render sees its own fixed value of state, like a photo. Inside a handler, `count` is the value from **that** render, even after you call `setCount`.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function addThree() {
    setCount(count + 1); // count is 0 -> queue "set to 1"
    setCount(count + 1); // count is still 0 -> "set to 1"
    setCount(count + 1); // still 0 -> "set to 1"
    console.log(count);  // 0 (snapshot!)
  } // Result: 1, not 3 ❌

  function addThreeFixed() {
    setCount((c) => c + 1); // functional update: uses the latest queued value
    setCount((c) => c + 1);
    setCount((c) => c + 1);
  } // Result: 3 ✅

  return <button onClick={addThreeFixed}>{count}</button>;
}
```

Rule of thumb: if the new state depends on the old state, use the **functional form** `setX(prev => ...)`.

### Updating objects and arrays immutably

```jsx
setUser((u) => ({ ...u, address: { ...u.address, city: 'Pune' } })); // nested copy
setItems((xs) => [...xs, item]);                 // add
setItems((xs) => xs.filter((x) => x.id !== id)); // remove
setItems((xs) => xs.map((x) => (x.id === id ? { ...x, qty: x.qty + 1 } : x))); // edit
```

Setting the **same** value (by `Object.is`) lets React bail out of re-rendering.

### useEffect: what a side effect is

Anything that reaches **outside** the render calculation: fetching data, subscriptions, timers, manual DOM changes, analytics, `localStorage`. Render must stay pure, so these go in effects, which run **after React commits** to the DOM and usually after the browser paints (so they don't block the screen update).

### Dependency array

| Form | Runs when |
| --- | --- |
| `useEffect(fn)` (no array) | After **every** render |
| `useEffect(fn, [])` | Once after mount (cleanup on unmount) |
| `useEffect(fn, [a, b])` | After mount and whenever `a` or `b` changes (`Object.is`) |

### Cleanup and when it runs

Return a function from the effect to undo it. Cleanup runs **before the effect re-runs** (with the old values) and **on unmount**.

```jsx
useEffect(() => {
  const id = setInterval(() => setSeconds((s) => s + 1), 1000);
  return () => clearInterval(id); // stop the old timer
}, []);
```

### Fetching data with cleanup

```jsx
function User({ userId }) {
  const [user, setUser] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();      // cancel the request
    setUser(null);

    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(res.statusText);
        return res.json();
      })
      .then(setUser)
      .catch((err) => {
        if (err.name !== 'AbortError') setError(err); // ignore our own aborts
      });

    return () => controller.abort(); // userId changed or unmounted: drop stale request
  }, [userId]);

  if (error) return <p>Error!</p>;
  return user ? <h2>{user.name}</h2> : <p>Loading...</p>;
}
```

Without the abort (or an `ignore` flag set to `true` in cleanup), a slow response for an **old** `userId` can arrive last and overwrite the correct data: a **race condition**. In real apps prefer a data library (TanStack Query, SWR) or framework data fetching (Next.js Server Components).

### StrictMode double run in development

In dev, `<StrictMode>` mounts each component, **immediately unmounts it, then mounts again**, so effects run setup, cleanup, setup. It exists to surface missing cleanups (duplicate subscriptions, leaking timers). It does **not** happen in production. The fix is a correct cleanup, not removing StrictMode.

## 💻 Code examples

### You might not need an effect

```jsx
// ❌ Derived state stored via an effect: extra render, can go out of sync
const [fullName, setFullName] = useState('');
useEffect(() => setFullName(`${first} ${last}`), [first, last]);

// ✅ Just compute it during render
const fullName = `${first} ${last}`;

// ❌ Reacting to a user action inside an effect
useEffect(() => { if (submitted) sendAnalytics(); }, [submitted]);

// ✅ Do it in the event handler that caused it
function handleSubmit() { save(); sendAnalytics(); }
```

Effects are for **syncing with external systems**, not for transforming data or responding to events.

## ⚠️ Common mistakes & gotchas

- **Infinite loop**: setting state inside an effect with no deps array (or with that state as a dep).
- **Missing deps**: the effect uses stale props/state. Trust the `exhaustive-deps` lint rule.
- **Object/array/function deps**: a new reference every render makes the effect run every render. Move it inside the effect, use primitives as deps, or memoise.
- **Stale closures**: an interval created once with `[]` captures the first `count` forever. Use the functional updater `setCount(c => c + 1)` or a ref.
- `async` effect callback: `useEffect(async () => ...)` returns a promise, not a cleanup. Define an async function inside and call it.
- Reading state right after setting it and expecting the new value.

## 🎤 Say this in the interview

> `useState` gives each render a snapshot of state; updates are queued and batched, so when the next value depends on the previous one I use the functional updater. `useEffect` is for syncing with external systems after render: dependencies decide when it re-runs, and cleanup runs before the next run and on unmount, which also prevents race conditions in data fetching.

## ❓ Likely interview questions

**Why were hooks introduced?**

To reuse stateful logic without HOCs/render props, keep related logic together, and avoid class and `this` complexity.

**Why can't hooks be called conditionally?**

React identifies hooks by call order in a per-component list; skipping one shifts every later hook to the wrong slot.

**Why does calling `setCount(count + 1)` three times only add 1?**

Each call reads the same render snapshot of `count`. Use `setCount(c => c + 1)`.

**What is automatic batching?**

Since React 18, multiple state updates anywhere (handlers, timeouts, promises) are grouped into a single re-render.

**When does the cleanup function run?**

Before the effect runs again with new deps, and when the component unmounts.

**Why does my effect run twice in development?**

StrictMode intentionally mounts, unmounts and remounts to expose missing cleanups. It doesn't happen in production.

**How do you avoid race conditions when fetching in an effect?**

Abort the request with `AbortController` or set an `ignore` flag in cleanup so stale responses are discarded.

**What is lazy initial state?**

Passing a function to `useState` so the expensive initial computation runs only on the first render.

---

⬅️ Prev: [Events and Forms](../05-Events-and-Forms/README.md) | Next ➡️: [Advanced Hooks](../07-Hooks-Advanced/README.md)
