# 📡 Context API

> **TL;DR:** Context lets a parent make a value available to **any** descendant without passing props through every level. Create it with `createContext`, provide it with `<MyContext value={...}>` (React 19), read it with `useContext` or `use`. Every consumer re-renders when the value changes, so memoize and split contexts.

## 🧠 Concept

### The problem: prop drilling

When deeply nested components need the same data (theme, logged-in user, locale), you end up passing props through components that don't use them — they just forward it. This is **prop drilling**: noisy, fragile, and painful to refactor.

```jsx
// ❌ Prop drilling: Layout and Header don't care about `user`, they just pass it on
function App() {
  const [user, setUser] = useState({ name: "Aryan" });
  return <Layout user={user} />;
}
function Layout({ user }) {
  return <Header user={user} />;
}
function Header({ user }) {
  return <Avatar user={user} />;
}
function Avatar({ user }) {
  return <span>{user.name}</span>;
}
```

### The fix: context

Context is like a **radio broadcast**: the Provider is the station, and any component inside it can "tune in" with `useContext` — no wires (props) through the middle.

```jsx
// ✅ Context: only the producer and the consumer know about `user`
const UserContext = createContext(null);

function App() {
  const [user, setUser] = useState({ name: "Aryan" });
  return (
    <UserContext value={user}>
      <Layout />
    </UserContext>
  );
}
function Layout() {
  return <Header />;
}
function Header() {
  return <Avatar />;
}
function Avatar() {
  const user = useContext(UserContext);
  return <span>{user.name}</span>;
}
```

## ⚙️ How it works / Deep dive

### The three pieces

- `createContext(defaultValue)` — creates the context object.
- **Provider** — in React 19 you render the context itself: `<ThemeContext value="dark">`. The older `<ThemeContext.Provider value="dark">` still works but will be deprecated.
- **Consumer** — `useContext(ThemeContext)` in function components. React 19's `use(ThemeContext)` does the same but can be called conditionally (inside `if`). Legacy: `<ThemeContext.Consumer>` / `static contextType`.

### What the default value means

The default is used **only when there is no matching Provider above the consumer** in the tree. It is not an "initial value" — if a Provider passes `value={undefined}`, consumers get `undefined`, not the default. Tip: use `null` as default and throw in a custom hook so missing providers fail loudly.

### Lookup rule

`useContext` finds the **nearest** Provider above it. Nested providers override outer ones, so you can theme a sub-tree differently.

### Performance model

- When the Provider's `value` changes (compared with `Object.is`), **every component calling `useContext` for it re-renders**, even if it only uses one field. `React.memo` on an intermediate component does **not** block this.
- Passing a new object literal each render (`value={{ user, login }}`) means a "change" every time the provider's parent renders → memoize with `useMemo`/`useCallback`.
- **Split contexts** by how often they change: e.g. `ThemeContext` and `AuthContext` separately, or a `StateContext` + `DispatchContext` pair — `dispatch` is stable, so components that only dispatch never re-render.
- There are no built-in selectors. If you need fine-grained subscriptions to a big, fast-changing store, use Zustand/Redux.

## 💻 Code examples

### ThemeContext (simple)

```jsx
const ThemeContext = createContext("light"); // default if no provider

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");
  const toggle = useCallback(() => setTheme((t) => (t === "light" ? "dark" : "light")), []);
  const value = useMemo(() => ({ theme, toggle }), [theme, toggle]); // stable reference
  return <ThemeContext value={value}>{children}</ThemeContext>;
}

function ThemeButton() {
  const { theme, toggle } = useContext(ThemeContext);
  return <button onClick={toggle}>Theme: {theme}</button>;
}
```

### AuthContext with a safe custom hook

```jsx
const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const login = useCallback(async (email, password) => {
    const res = await fetch("/api/login", { method: "POST", body: JSON.stringify({ email, password }) });
    setUser(await res.json());
  }, []);
  const logout = useCallback(() => setUser(null), []);

  const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);
  return <AuthContext value={value}>{children}</AuthContext>;
}

// Custom hook: nicer API + fails loudly if used outside the provider
export function useAuth() {
  const ctx = useContext(AuthContext);
  if (ctx === null) {
    throw new Error("useAuth must be used inside <AuthProvider>");
  }
  return ctx;
}

// Usage anywhere below <AuthProvider>
function Navbar() {
  const { user, logout } = useAuth();
  return user ? <button onClick={logout}>Logout {user.name}</button> : <a href="/login">Login</a>;
}
```

### State + dispatch split (with useReducer)

```jsx
const TodosStateContext = createContext(null);
const TodosDispatchContext = createContext(null);

function TodosProvider({ children }) {
  const [todos, dispatch] = useReducer(todosReducer, []);
  return (
    <TodosStateContext value={todos}>
      <TodosDispatchContext value={dispatch}>{children}</TodosDispatchContext>
    </TodosStateContext>
  );
}

// AddTodo only needs dispatch → it does NOT re-render when todos change
function AddTodo() {
  const dispatch = useContext(TodosDispatchContext);
  return <button onClick={() => dispatch({ type: "add", text: "New" })}>Add</button>;
}
```

### Context vs Redux

| | Context | Redux / Zustand |
| --- | --- | --- |
| What it is | A **transport** (dependency injection) mechanism | A **state management** library |
| Holds state? | No — you pair it with `useState`/`useReducer` | Yes, in an external store |
| Re-render granularity | All consumers on any value change | Only components whose selected slice changed |
| DevTools / middleware | None | Time-travel, logging, thunks |
| Best for | Theme, auth user, locale, low-frequency values | Large, frequently updated, shared app state |

## ⚠️ Common mistakes & gotchas

- Inline object as `value` → every consumer re-renders on each parent render.
- One giant "AppContext" with everything → any change re-renders the whole app.
- Expecting the default value to show up when a Provider exists with `undefined`.
- Forgetting to wrap the tree in the Provider → silent `null` errors (fix: throwing custom hook).
- Using context for rapidly changing values (mouse position, form keystrokes) across many consumers.
- Context does not work in Server Components — in Next.js, providers must be Client Components (`"use client"`).

## 🎤 Say this in the interview

> Context solves prop drilling by letting any descendant read a value from the nearest Provider. It's a transport mechanism, not a state manager — I pair it with `useState` or `useReducer`. Because every consumer re-renders when the value changes, I memoize the value, split contexts by concern, and reach for Zustand or Redux when I need selectors on frequently changing state.

## ❓ Likely interview questions

**What problem does Context solve?**

Prop drilling — passing data through many layers of components that don't need it.

**When is the default value of `createContext` used?**

Only when a consumer has no matching Provider above it in the tree.

**What changed with Context in React 19?**

You can render `<MyContext value={...}>` directly as the provider instead of `<MyContext.Provider>`, and read it with `use(MyContext)`, which can be called conditionally.

**Does `React.memo` stop context-triggered re-renders?**

No. A component that consumes the context re-renders when the value changes, regardless of memo on it or its parents.

**How do you optimise a Context?**

Memoize the `value`, split into multiple contexts (state vs dispatch, or by domain), keep providers low in the tree, or move to a store with selectors.

**Can Context replace Redux?**

For small apps with low-frequency global data, yes. For large, frequently updated state needing selectors, middleware and DevTools, a store is better.

**Why make a custom hook like `useAuth`?**

It hides the context object, gives a clean API, and throws a clear error when used outside the Provider.

⬅️ Prev: [09 Lifecycle & Class Components](../09-Lifecycle-and-Class-Components/README.md) | Next ➡️: [11 State Management](../11-State-Management-Redux-Zustand/README.md)
