# 🗃️ State Management: Redux, Zustand & TanStack Query

> **TL;DR:** Not all state is the same. Keep **local** state in components, put truly **global client** state in a store (Redux Toolkit or Zustand), let **server** state live in a cache (TanStack Query / RTK Query), and keep shareable UI state in the **URL**.

## 🧠 Concept

### Four kinds of state

| Kind | Example | Where it should live |
| --- | --- | --- |
| Local / UI | Modal open, input text, hover | `useState` / `useReducer` in the component |
| Global client | Cart, theme, auth user, multi-step wizard | Context, Zustand, Redux Toolkit |
| Server (remote) | Products list, user profile from API | TanStack Query, RTK Query, SWR (a cache, not a store) |
| URL | Search query, filters, page number, tab | Query string / route params (`useSearchParams`) |

The most common mistake is treating **server data** as global client state (fetch in `useEffect`, dump into Redux). Server data is a **cache** of something owned elsewhere — it goes stale, needs refetching, deduping and invalidation. Dedicated libraries do that for you.

### Redux in plain words

Redux is a **bank**. The **store** is the vault holding all state. You can't walk in and change balances — you fill a **deposit slip** (an **action**: `{ type: "cart/add", payload: item }`) and hand it to the teller (**dispatch**). The teller follows strict rules (**reducer**: a pure function `(state, action) => newState`). Anyone can read balances through a window (**selector**).

```text
  ┌────────────┐  dispatch(action)  ┌──────────────┐  (state, action)  ┌───────────┐
  │     UI     │ ─────────────────► │  Middleware  │ ────────────────► │  Reducer  │
  │ (component)│                    │ (thunk, log) │                   │  (pure)   │
  └────────────┘                    └──────────────┘                   └─────┬─────┘
        ▲                                                                    │ new state
        │      useSelector re-renders only if the selected value changed     ▼
        └─────────────────────────────────────────────────────────────── ┌───────────┐
                                                                         │   Store   │
                                                                         └───────────┘
```

### The three principles of Redux

1. **Single source of truth** — the whole app state is one object tree in one store.
2. **State is read-only** — the only way to change it is to dispatch an action.
3. **Changes are made with pure functions** — reducers return new state without side effects or mutation.

## ⚙️ How it works / Deep dive

### Redux Toolkit (RTK) — the official way to write Redux

Classic Redux needed action types, action creators, switch statements and spread-everything immutability. RTK removes the boilerplate:

- `configureStore` — creates the store with thunk middleware and DevTools pre-wired.
- `createSlice` — defines state + reducers + auto-generated action creators in one place. Uses **Immer**, so you can write "mutating" code (`state.value += 1`) and Immer produces a new immutable object behind the scenes.
- `createAsyncThunk` — handles async logic and dispatches `pending` / `fulfilled` / `rejected` actions automatically.
- `useSelector` / `useDispatch` (from `react-redux`) — read state and dispatch from components.

### Middleware & thunks

Reducers must be pure, so where does async code go? **Middleware** sits between `dispatch` and the reducer and can intercept actions. The **thunk** middleware lets you dispatch a _function_ instead of an object; that function receives `dispatch` and `getState`, so it can fetch data and dispatch real actions when done.

### RTK Query (brief)

Built into RTK. You define an API with `createApi` and endpoints; it generates hooks like `useGetTodosQuery()` with caching, deduping, loading states and tag-based invalidation — Redux's answer to TanStack Query.

## 💻 Code examples

### Redux Toolkit todo + counter

```jsx
// store.js
import { configureStore, createSlice, createAsyncThunk } from "@reduxjs/toolkit";

export const fetchTodos = createAsyncThunk("todos/fetch", async () => {
  const res = await fetch("/api/todos");
  return res.json(); // becomes action.payload of todos/fetch/fulfilled
});

const todosSlice = createSlice({
  name: "todos",
  initialState: { items: [], status: "idle", count: 0 },
  reducers: {
    added(state, action) {
      state.items.push({ id: Date.now(), text: action.payload, done: false }); // Immer makes this safe
    },
    toggled(state, action) {
      const todo = state.items.find((t) => t.id === action.payload);
      if (todo) todo.done = !todo.done;
    },
    incremented(state) {
      state.count += 1;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchTodos.pending, (state) => { state.status = "loading"; })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.status = "idle";
        state.items = action.payload;
      })
      .addCase(fetchTodos.rejected, (state) => { state.status = "error"; });
  },
});

export const { added, toggled, incremented } = todosSlice.actions;
export const store = configureStore({ reducer: { todos: todosSlice.reducer } });
```

```jsx
// main.jsx — wrap the app
<Provider store={store}>
  <App />
</Provider>

// TodoList.jsx
import { useSelector, useDispatch } from "react-redux";

function TodoList() {
  const items = useSelector((state) => state.todos.items); // re-renders only when items change
  const dispatch = useDispatch();

  useEffect(() => { dispatch(fetchTodos()); }, [dispatch]);

  return (
    <>
      <button onClick={() => dispatch(added("Learn RTK"))}>Add</button>
      {items.map((t) => (
        <li key={t.id} onClick={() => dispatch(toggled(t.id))}>{t.done ? "✅" : "⬜"} {t.text}</li>
      ))}
    </>
  );
}
```

### Zustand — tiny, hook-based store

```jsx
import { create } from "zustand";
import { useShallow } from "zustand/react/shallow";

const useCartStore = create((set) => ({
  items: [],
  add: (item) => set((state) => ({ items: [...state.items, item] })), // set merges top-level
  clear: () => set({ items: [] }),
}));

function CartCount() {
  const count = useCartStore((s) => s.items.length); // selector: re-renders only when length changes
  return <span>{count}</span>;
}

function CartActions() {
  // selecting several values as an object → wrap with useShallow (required in Zustand v5)
  const { add, clear } = useCartStore(useShallow((s) => ({ add: s.add, clear: s.clear })));
  return <button onClick={clear}>Clear</button>;
}
```

Why it's popular: no Provider, almost no boilerplate, selectors built in, works outside React (`useCartStore.getState()`), ~1 KB.

### TanStack Query — server state

```jsx
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

function Todos() {
  const queryClient = useQueryClient();

  const { data, isPending, error } = useQuery({
    queryKey: ["todos"],                       // cache key
    queryFn: () => fetch("/api/todos").then((r) => r.json()),
    staleTime: 60_000,                         // fresh for 1 min → no refetch
  });

  const addTodo = useMutation({
    mutationFn: (text) => fetch("/api/todos", { method: "POST", body: JSON.stringify({ text }) }),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ["todos"] }), // refetch list
  });

  if (isPending) return <p>Loading…</p>;
  if (error) return <p>Error!</p>;
  return (
    <>
      <button onClick={() => addTodo.mutate("New")}>Add</button>
      {data.map((t) => <li key={t.id}>{t.text}</li>)}
    </>
  );
}
```

**Stale-while-revalidate:** cached data is shown instantly (even if stale), while a background refetch updates it. You also get deduping, retries, refetch on window focus, pagination and garbage collection (`gcTime`).

### Decision table

| Situation | Use |
| --- | --- |
| State used by one component or a small subtree | `useState` / `useReducer` + lift state up |
| Rarely changing global values (theme, user, locale) | Context |
| Shared client state, small/medium app, minimal boilerplate | Zustand |
| Large app, many devs, strict patterns, DevTools, middleware | Redux Toolkit |
| Data from an API | TanStack Query (or RTK Query if already on Redux) |
| Filters, search, pagination that should be shareable/bookmarkable | URL search params |
| Next.js App Router data | Fetch in Server Components first; client cache only for interactive data |

## ⚠️ Common mistakes & gotchas

- Storing server data in Redux/Zustand and hand-rolling loading/error/caching.
- Mutating state in plain reducers (outside `createSlice`/Immer) → UI doesn't update.
- Selecting the whole store (`useSelector((s) => s)`) → re-renders on every change.
- Returning a new object/array from a selector each time → extra re-renders (use `useShallow` in Zustand, memoized selectors with `createSelector` in Redux).
- Putting non-serializable values (class instances, functions, Promises) in Redux state.
- In Next.js, a module-level store is shared across requests on the server — create the store per request inside a client provider.

## 🎤 Say this in the interview

> I first classify state: local, global client, server, or URL. Local stays in components, server data goes in TanStack Query for caching and revalidation, and only genuinely shared client state goes into a store — Zustand for simplicity or Redux Toolkit for large teams that need strict patterns and DevTools.

## ❓ Likely interview questions

**What are the core parts of Redux?**

Store (holds state), actions (plain objects describing what happened), reducers (pure functions computing new state), dispatch (sends actions), selectors (read slices of state).

**Why is Redux Toolkit recommended over plain Redux?**

It removes boilerplate: `createSlice` generates actions, Immer allows mutable-style updates, and `configureStore` sets up thunk and DevTools by default.

**How does Immer let you "mutate" state?**

It gives you a draft proxy, records your changes, and produces a new immutable object with structural sharing.

**Where do async calls go in Redux?**

In middleware — usually thunks via `createAsyncThunk`, or RTK Query for data fetching. Never inside reducers.

**Why choose Zustand over Redux?**

Less boilerplate, no Provider, built-in selectors, tiny bundle. Redux wins for very large apps needing conventions, middleware and time-travel debugging.

**What is server state and why treat it differently?**

Data owned by the backend that can change without you knowing. It needs caching, deduping, background refetching and invalidation — what TanStack Query provides.

**What's `staleTime` vs `gcTime` in TanStack Query?**

`staleTime`: how long data counts as fresh (no refetch). `gcTime`: how long unused cached data is kept in memory before garbage collection.

⬅️ Prev: [10 Context API](../10-Context-API/README.md) | Next ➡️: [12 Re-rendering & Performance](../12-Re-rendering-and-Performance/README.md)
