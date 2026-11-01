# 🧩 Components, Props and State

> **TL;DR:** Components are functions that return UI. **Props** are inputs passed from the parent (read-only). **State** is a component's own memory that, when updated via its setter, triggers a re-render. Share state by **lifting it up** to the closest common parent, and always update it **immutably**.

## 🧠 Concept

### Functional vs class components

| | Function component | Class component |
| --- | --- | --- |
| Syntax | Plain function returning JSX | `class X extends React.Component` with `render()` |
| State | `useState`, `useReducer` hooks | `this.state` + `this.setState` |
| Lifecycle | `useEffect` and friends | `componentDidMount`, etc. |
| `this` binding | None | Must bind handlers or use arrow fields |
| Status | **The standard today** | Legacy; still needed only for error boundaries |

### Naming: capital letter

JSX treats lowercase tags (`<div>`) as HTML and capitalised tags (`<UserCard>`) as components. A component named `userCard` will silently render an unknown `<usercard>` element.

### Props: the component's inputs

Think of a component as a function and props as its **arguments**. The parent decides them; the child must **never mutate** them. Same props should give the same UI (purity).

### State: the component's memory

State is data that **changes over time** and should update the screen when it does (form input, toggle, fetched data). A plain `let` variable does not work because:

1. It is **re-created** on every render (the function runs again from scratch), so the value resets.
2. Changing it does **not tell React** to re-render.

`useState` solves both: React stores the value outside the function and re-renders when you call the setter.

```jsx
function Broken() {
  let count = 0;                        // resets to 0 on every render
  return <button onClick={() => count++}>{count}</button>; // never updates
}

function Works() {
  const [count, setCount] = useState(0); // persisted by React
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

## ⚙️ How it works / Deep dive

### Props in practice

```jsx
// Destructuring + default values (the modern way)
function Button({ label = 'Click me', variant = 'primary', onClick }) {
  return <button className={variant} onClick={onClick}>{label}</button>;
}
```

`Component.defaultProps` was **removed for function components in React 19**; use default parameters as above.

### The `children` prop

Whatever you put **between** a component's tags arrives as `props.children`. Great for wrappers like cards, modals and layouts.

```jsx
function Card({ title, children }) {
  return (
    <section className="card">
      <h3>{title}</h3>
      {children}
    </section>
  );
}

<Card title="Profile"><p>Any JSX goes here</p></Card>;
```

### Child to parent: pass a function down

Data only flows down, so a child "talks up" by calling a callback the parent provided.

```jsx
function Parent() {
  const [msg, setMsg] = useState('');
  return (
    <>
      <Child onSend={setMsg} />
      <p>Child said: {msg}</p>
    </>
  );
}

function Child({ onSend }) {
  return <button onClick={() => onSend('Hello!')}>Send</button>;
}
```

### Props vs state

| | Props | State |
| --- | --- | --- |
| Owned by | Parent | The component itself |
| Mutable by the component? | No (read-only) | Yes, via the setter only |
| Triggers re-render when changed? | Yes (parent re-renders with new props) | Yes |
| Purpose | Configure / pass data down | Track data that changes over time |
| Analogy | Function arguments | Variables a function remembers |

### Lifting state up

When **two siblings** need the same data, move the state to their **closest common parent** and pass it down as props plus a setter.

```jsx
// Two inputs that must stay in sync: Celsius and Fahrenheit
function Converter() {
  const [celsius, setCelsius] = useState(0); // single source of truth

  return (
    <>
      <TempInput label="°C" value={celsius} onChange={setCelsius} />
      <TempInput
        label="°F"
        value={(celsius * 9) / 5 + 32}
        onChange={(f) => setCelsius(((f - 32) * 5) / 9)}
      />
    </>
  );
}

function TempInput({ label, value, onChange }) {
  return (
    <label>
      {label}
      <input type="number" value={value} onChange={(e) => onChange(Number(e.target.value))} />
    </label>
  );
}
```

### Prop drilling

Passing props through several middle components that don't use them, just to reach a deep child (`App > Layout > Sidebar > Menu > Avatar`). It works but is noisy and brittle.

Fixes, in rough order of preference:

1. **Composition**: pass components as `children` so intermediates don't need to know the data.
2. **Context API** (`createContext` + `useContext`) for app-wide data like theme, auth, locale.
3. **State library** (Zustand, Redux Toolkit, Jotai) for complex, frequently changing global state.

### Immutability and why it matters

React decides whether state changed using `Object.is`, i.e. a **reference comparison**, not a deep compare. If you mutate an object and pass the same reference back, React sees "same object" and may skip the re-render. Memoised children (`React.memo`) and effect deps rely on the same check.

```jsx
// ❌ Mutation: same reference, React may not re-render
user.name = 'Ana';
setUser(user);

// ✅ New object: new reference, React re-renders
setUser({ ...user, name: 'Ana' });
setTodos([...todos, newTodo]);             // add
setTodos(todos.filter(t => t.id !== id));  // remove
setTodos(todos.map(t => t.id === id ? { ...t, done: true } : t)); // update
```

## 💻 Code examples

```jsx
// Composition fixes prop drilling: Layout never touches `user`
function App() {
  const user = { name: 'Riya' };
  return (
    <Layout>
      <Avatar user={user} />
    </Layout>
  );
}

function Layout({ children }) {
  return <main className="layout">{children}</main>;
}
```

## ⚠️ Common mistakes & gotchas

- Mutating props (`props.title = 'x'`): props are read-only.
- Mutating state (`arr.push(x)`) then calling the setter with the same array.
- Copying props into state (`useState(props.value)`): it won't update when the prop changes; derive instead.
- Duplicating the same state in two siblings instead of lifting it up.
- Spreads are **shallow**: nested objects must be copied at every level you change.
- Calling the handler in JSX: `onSend={setMsg('x')}` runs during render; wrap it in an arrow.

## 🎤 Say this in the interview

> Props are read-only inputs from the parent; state is the component's own memory, and updating it through the setter triggers a re-render. When siblings need the same data I lift state to their common parent, and I always update immutably because React detects changes by reference.

## ❓ Likely interview questions

**Difference between props and state?**

Props are passed in by the parent and read-only; state is owned and updated by the component itself. Both cause re-renders when they change.

**Why can't I use a normal variable instead of state?**

It resets on every render and changing it doesn't trigger a re-render.

**How does a child send data to its parent?**

The parent passes a callback function as a prop, and the child calls it with the data.

**What is lifting state up?**

Moving shared state to the closest common ancestor so siblings stay in sync via props.

**What is prop drilling and how do you avoid it?**

Passing props through many layers that don't need them. Fix with composition, Context, or a state library.

**Why should state be immutable?**

React compares by reference (`Object.is`); mutating keeps the same reference, so updates can be skipped and memoisation breaks.

**What is the `children` prop?**

The JSX nested between a component's opening and closing tags, used to build wrappers and layouts.

---

⬅️ Prev: [Virtual DOM and Reconciliation](../02-Virtual-DOM-and-Reconciliation/README.md) | Next ➡️: [Lists, Keys and Conditional Rendering](../04-Lists-Keys-Conditional-Rendering/README.md)
