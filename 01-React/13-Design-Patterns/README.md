# 🧩 React Design Patterns

> **TL;DR:** React favours **composition over inheritance**: build big things by nesting small components and passing JSX through `children` or props. Classic patterns (HOCs, render props, container/presentational) solved logic reuse before hooks; today **custom hooks** + **compound components** + **providers** cover most needs.

## 🧠 Concept

A "pattern" is a reusable answer to a recurring problem. In React the two recurring problems are:

1. **Reusing UI structure** → composition (children, slots, compound components).
2. **Reusing stateful logic** → historically HOCs and render props, now custom hooks.

Analogy: composition is LEGO — you don't "inherit" from a red brick to make a wall, you snap bricks together. React's docs explicitly say they've never needed component inheritance hierarchies.

## ⚙️ How it works / Deep dive

### 1. Composition (children & slots)

`children` is whatever you put between the opening and closing tags. For multiple "holes", pass JSX through named props — **slots**.

```jsx
function Card({ title, actions, children }) {
  return (
    <div className="card">
      <header>{title}{actions}</header>   {/* slots */}
      <section>{children}</section>       {/* default slot */}
    </div>
  );
}

<Card title={<h2>Profile</h2>} actions={<button>Edit</button>}>
  <p>Bio goes here</p>
</Card>
```

When: always — it's the default way to build flexible layouts (modals, cards, layouts) without prop explosions.

### 2. Container / Presentational

- **Container** — knows _how things work_: fetches data, holds state, passes it down.
- **Presentational** — knows _how things look_: pure props → JSX, easy to test and reuse in Storybook.

```jsx
function UserListContainer() {
  const { data: users = [], isPending } = useUsers(); // logic
  return <UserList users={users} loading={isPending} />;
}
function UserList({ users, loading }) {               // pure UI
  if (loading) return <Spinner />;
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

When: useful mental model for separation; a custom hook often replaces the container. In Next.js, Server Components naturally play the "container" role.

### 3. Higher-Order Component (HOC)

A function that **takes a component and returns a new, enhanced component**: `const Enhanced = withX(Component)`.

```jsx
function withAuth(Component) {
  return function WithAuth(props) {
    const { user } = useAuth();
    if (!user) return <Navigate to="/login" />;
    return <Component {...props} user={user} />; // inject extra prop
  };
}

const Dashboard = withAuth(function Dashboard({ user }) {
  return <h1>Welcome {user.name}</h1>;
});
```

| Pros | Cons |
| --- | --- |
| Reuse cross-cutting logic (auth, logging, theming) | "Wrapper hell" in DevTools when stacked |
| Works with class components | Prop name collisions, implicit "magic" props |
| Wraps without modifying the original | Must forward refs/static methods manually; harder to type |

When: legacy code, libraries (`connect` from react-redux, `withRouter`), or wrapping a whole component (auth guard, error boundary).

### 4. Render props

A component takes a **function as a prop** (often `children`) and calls it with its internal state, letting the caller decide what to render.

```jsx
function MouseTracker({ children }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return <div onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })}>{children(pos)}</div>;
}

<MouseTracker>{({ x, y }) => <p>Mouse at {x}, {y}</p>}</MouseTracker>
```

Why hooks replaced it: `const pos = useMouse()` gives the same reuse without nesting callbacks ("callback pyramid"). Still seen in libraries (Formik, react-window rows, headless UI components).

### 5. Compound components

A group of components that **work together and share implicit state** through context — like `<select>` and `<option>`. The parent owns state; children read it.

```jsx
const TabsContext = createContext(null);

function Tabs({ defaultValue, children }) {
  const [active, setActive] = useState(defaultValue);
  return <TabsContext value={{ active, setActive }}>{children}</TabsContext>;
}

function TabList({ children }) {
  return <div role="tablist">{children}</div>;
}

function Tab({ value, children }) {
  const { active, setActive } = useContext(TabsContext);
  return (
    <button role="tab" aria-selected={active === value} onClick={() => setActive(value)}>
      {children}
    </button>
  );
}

function TabPanel({ value, children }) {
  const { active } = useContext(TabsContext);
  return active === value ? <div role="tabpanel">{children}</div> : null;
}

Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Usage — flexible markup, no prop drilling, no config arrays
<Tabs defaultValue="a">
  <Tabs.List>
    <Tabs.Tab value="a">Account</Tabs.Tab>
    <Tabs.Tab value="b">Billing</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="a">Account settings</Tabs.Panel>
  <Tabs.Panel value="b">Billing settings</Tabs.Panel>
</Tabs>
```

When: UI kits — Tabs, Accordion, Menu, Select, Modal (Radix UI and shadcn/ui use this heavily).

### 6. Controlled vs uncontrolled components (as a pattern)

- **Controlled:** the parent owns the value and passes `value` + `onChange`. Single source of truth, easy validation.
- **Uncontrolled:** the component keeps its own internal state (or the DOM does, read via `ref`/`FormData`); parent passes only `defaultValue`.
- Good reusable components support **both**: if `value` prop is provided, use it; otherwise fall back to internal state.

```jsx
function Toggle({ on: controlledOn, defaultOn = false, onChange }) {
  const [internalOn, setInternalOn] = useState(defaultOn);
  const isControlled = controlledOn !== undefined;
  const on = isControlled ? controlledOn : internalOn;

  function toggle() {
    if (!isControlled) setInternalOn(!on);
    onChange?.(!on);
  }
  return <button onClick={toggle}>{on ? "ON" : "OFF"}</button>;
}
```

### 7. Custom hooks — the modern default

Extract stateful logic into a `useSomething` function. Each call gets **its own** state; only logic is shared, not data.

```jsx
function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn((v) => !v), []);
  return [on, toggle];
}
```

When: any time you'd reach for an HOC or render prop to share logic.

### 8. Provider pattern

Wrap a subtree in a Provider that exposes shared data/services via context, plus a custom hook to read it (`<AuthProvider>` + `useAuth()`, `<QueryClientProvider>`). Covered in topic 10.

When: app-wide dependencies — auth, theme, i18n, query client, feature flags.

### Quick "when to use" summary

| Pattern | Use when |
| --- | --- |
| Composition / slots | Flexible layout components |
| Container / Presentational | Separating data logic from pure UI |
| HOC | Wrapping whole components; legacy/library APIs |
| Render props | Caller must control rendering with internal state (rare now) |
| Compound components | Related components sharing implicit state (Tabs, Accordion) |
| Controlled / uncontrolled | Reusable inputs that parents may or may not control |
| Custom hooks | Reusing stateful logic — first choice |
| Provider | Shared app-wide data or services |

## ⚠️ Common mistakes & gotchas

- Creating an HOC **inside** render (`const X = withAuth(Y)` in a component body) → new component type every render → remounts, state lost.
- Expecting custom hooks to share state between components — they share logic; use context/store for shared state.
- Compound components used outside their parent → context is `null` (throw a helpful error).
- Switching an input from uncontrolled to controlled (`value` from `undefined` to a string) → React warning.
- Over-abstracting: a pattern for a component used once adds complexity without benefit.

## 🎤 Say this in the interview

> React favours composition — I build with `children` and slot props rather than inheritance. For logic reuse I use custom hooks, which replaced most HOCs and render props, and for component families like Tabs I use the compound component pattern with context so consumers get flexible markup without prop drilling.

## ❓ Likely interview questions

**What is a Higher-Order Component?**

A function that takes a component and returns a new component with extra behaviour or props, e.g. `withAuth(Dashboard)`.

**Why do hooks replace HOCs and render props?**

They share logic without extra wrapper components, avoid prop collisions and nesting, and are easier to read and type.

**What are compound components?**

A set of components that share implicit state via context and are used together, like `<Tabs>`, `<Tabs.Tab>`, `<Tabs.Panel>`.

**What does "composition over inheritance" mean in React?**

Build complex UIs by nesting and passing components as props/children instead of extending component classes.

**What's the render props pattern?**

Passing a function prop that the component calls with its data to decide what to render.

**Controlled vs uncontrolled component?**

Controlled: value lives in parent state via `value`/`onChange`. Uncontrolled: value lives in the component/DOM, set initially via `defaultValue` and read via ref or form data.

**When would you still write an HOC today?**

To wrap entire components with cross-cutting concerns (auth guards, analytics, error boundaries) or when working with class components or libraries that expose HOCs.

⬅️ Prev: [12 Re-rendering & Performance](../12-Re-rendering-and-Performance/README.md) | Next ➡️: [14 Error Boundaries, Refs & Portals](../14-Error-Boundaries-Refs-Portals/README.md)
