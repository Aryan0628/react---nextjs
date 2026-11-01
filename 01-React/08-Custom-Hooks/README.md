# 🛠️ Custom Hooks

> **TL;DR:** A custom hook is a function whose name starts with `use` and that calls other hooks. It lets you **reuse stateful logic** (not the state itself) across components: every component that calls it gets its own independent state. Typical examples are `useFetch`, `useLocalStorage`, `useDebounce`, `useToggle` and `useOnlineStatus`.

## 🧠 Concept

### What and why

When two components need the same *behaviour* (fetch with loading/error, sync to `localStorage`, listen to window size), you'd otherwise copy the same `useState` + `useEffect` code into both. A custom hook extracts that code into one function.

Analogy: a custom hook is a **recipe**, not a dish. Two kitchens using the same recipe each cook their own meal; they don't share a plate.

Benefits:

- **DRY**: logic written once, used everywhere.
- **Readable components**: `const { data, loading } = useFetch(url)` hides the plumbing.
- **Separation of concerns**: UI in the component, behaviour in the hook.
- **Testable** in isolation.

### Naming with `use`

The `use` prefix is a convention React's lint rules rely on: it tells the linter (and humans) that the function calls hooks, so the **Rules of Hooks** apply to it and its callers. A function that doesn't call any hooks should **not** be named `useX`; make it a normal utility.

### Each call has independent state

```jsx
function Panel() {
  const a = useToggle(); // own state
  const b = useToggle(); // separate, unrelated state
}
```

Custom hooks share **logic**, never **state**. To share actual state between components, lift it up, use Context, or a store (Zustand, Redux).

## ⚙️ How it works / Deep dive

### How React sees a custom hook

React doesn't know custom hooks exist. At runtime the hook's body runs **inline** inside the calling component, so its `useState`/`useEffect` calls simply become that component's hook slots. That's why the same rules (top level, same order) apply and why every caller is independent.

### Design tips

- Return what callers need: a value (`useDebounce`), a tuple like `useState` (`[value, toggle]`), or an object for many fields (`{ data, error, loading }`).
- Accept options rather than hardcoding (`delay`, `initialValue`).
- Clean up subscriptions, timers and requests in effect cleanups.
- Keep hooks focused on one job; compose small hooks into bigger ones.

## 💻 Code examples

### useFetch (loading, error, abort)

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    setError(null);

    fetch(url, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(setData)
      .catch((err) => { if (err.name !== 'AbortError') setError(err); })
      .finally(() => { if (!controller.signal.aborted) setLoading(false); });

    return () => controller.abort(); // url changed / unmounted
  }, [url]);

  return { data, error, loading };
}

// Usage
const { data: posts, loading, error } = useFetch('/api/posts');
```

In production, prefer TanStack Query or SWR (caching, dedupe, retries), but writing `useFetch` is a classic interview task.

### useLocalStorage

```jsx
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {   // lazy init: read storage once
    try {
      const stored = localStorage.getItem(key);
      return stored !== null ? JSON.parse(stored) : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value)); // sync on change
  }, [key, value]);

  return [value, setValue]; // same shape as useState
}

const [theme, setTheme] = useLocalStorage('theme', 'light');
```

In SSR frameworks (Next.js), `localStorage` doesn't exist on the server, so guard with `typeof window !== 'undefined'` or read it in an effect.

### useDebounce

```jsx
function useDebounce(value, delay = 500) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id); // new keystroke cancels the old timer
  }, [value, delay]);

  return debounced;
}

// Only fetch once the user stops typing for 500 ms
const debouncedQuery = useDebounce(query);
const { data } = useFetch(`/api/search?q=${debouncedQuery}`);
```

### useToggle

```jsx
function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn((v) => !v), []); // stable function
  return [on, toggle];
}

const [isOpen, toggleOpen] = useToggle();
```

### useWindowSize and useOnlineStatus

```jsx
function useWindowSize() {
  const [size, setSize] = useState({ width: window.innerWidth, height: window.innerHeight });

  useEffect(() => {
    const onResize = () => setSize({ width: window.innerWidth, height: window.innerHeight });
    window.addEventListener('resize', onResize);
    return () => window.removeEventListener('resize', onResize); // avoid leaks
  }, []);

  return size;
}

// Subscribing to an external store: useSyncExternalStore is the recommended tool
function subscribe(callback) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function useOnlineStatus() {
  return useSyncExternalStore(
    subscribe,
    () => navigator.onLine, // client snapshot
    () => true              // server snapshot (SSR)
  );
}
```

### Testing custom hooks

Use `renderHook` from **React Testing Library** and wrap updates in `act`.

```jsx
import { renderHook, act } from '@testing-library/react';

test('useToggle flips the value', () => {
  const { result } = renderHook(() => useToggle());
  act(() => result.current[1]());
  expect(result.current[0]).toBe(true);
});
```

### Custom hook vs utility function vs HOC

| | Custom hook | Utility function | HOC |
| --- | --- | --- | --- |
| Calls hooks? | Yes | No | Yes (inside the wrapper) |
| Holds state / effects | Yes | No (pure logic) | Yes |
| Example | `useFetch(url)` | `formatDate(d)` | `withAuth(Component)` |
| Output | Values / functions | Any value | A new component |
| Downsides | Must follow Rules of Hooks | None | Wrapper hell, prop collisions, unclear data source |
| Today | Preferred for logic reuse | For non-React logic | Mostly legacy |

## ⚠️ Common mistakes & gotchas

- Naming a plain function `useSomething` when it calls no hooks (or a hook-calling function without `use`, which hides Rules of Hooks violations).
- Expecting two components using the same hook to share state.
- Calling a custom hook conditionally or in a loop.
- Forgetting cleanup (listeners, timers, aborts) inside the hook.
- Returning new object/function references every render, causing consumers' effects to re-run; memoise when consumers depend on them.
- Accessing `window` / `localStorage` during render in SSR apps.

## 🎤 Say this in the interview

> A custom hook is a `use`-prefixed function that composes built-in hooks to reuse stateful logic; each component calling it gets its own isolated state. It replaced HOCs and render props as the main reuse pattern because it's just a function call, with no extra wrappers.

## ❓ Likely interview questions

**What is a custom hook?**

A function starting with `use` that calls other hooks to encapsulate reusable stateful logic.

**Do components using the same custom hook share state?**

No. Each call runs inside its component and creates independent state.

**Why must custom hooks start with `use`?**

So lint tooling and React conventions can apply the Rules of Hooks to them and their callers.

**Write a `useDebounce` hook.**

Store the value in state, set a timeout in an effect to update it after `delay`, and clear the timeout in the cleanup.

**Custom hook vs HOC?**

Hooks share logic via a function call with no extra component layers; HOCs wrap components, causing nesting and prop-name collisions.

**How do you test a custom hook?**

With `renderHook` from React Testing Library, using `act` for state updates.

**When would you use `useSyncExternalStore` in a custom hook?**

When subscribing to an external data source (browser APIs, stores) so React reads it consistently, including during concurrent rendering and SSR.

---

⬅️ Prev: [Advanced Hooks](../07-Hooks-Advanced/README.md) | Next ➡️: [Lifecycle and Class Components](../09-Lifecycle-and-Class-Components/README.md)
