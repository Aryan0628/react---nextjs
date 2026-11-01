# 🧬 Lifecycle & Class Components

> **TL;DR:** Before hooks (2019), stateful React components were ES6 classes with `this.state`, `this.setState()` and lifecycle methods like `componentDidMount`. Hooks replaced them for new code, but interviews still ask about lifecycle order, and **error boundaries still require a class**.

## 🧠 Concept

A component has a "life": it is **born** (mounted into the DOM), **changes** (re-renders when props/state change), and **dies** (removed from the DOM). Class components expose a method for each moment so you can run code at the right time — like hooks into a timeline.

Analogy: think of a shop. **Mounting** = opening the shop (set up shelves, turn on lights → fetch data, add listeners). **Updating** = rearranging stock when deliveries arrive. **Unmounting** = closing the shop (turn off lights, lock doors → clear timers, remove listeners).

Function components with hooks express the same ideas, but grouped **by purpose** (one `useEffect` per concern) instead of **by time** (all setup crammed into `componentDidMount`).

## ⚙️ How it works / Deep dive

### Anatomy of a class component

- Extends `React.Component` and **must** implement `render()`.
- `constructor(props)` — call `super(props)` first, then initialise `this.state`.
- `this.setState(partial)` **shallow-merges** the object into state (unlike `useState`, which replaces). It is async/batched, so use the updater form `setState(prev => ...)` when the new value depends on the old one.
- Methods are not auto-bound: passing `this.handleClick` as a callback loses `this`. Fix with a constructor `bind`, or a class-field arrow function.

### Lifecycle phases and method order

```text
 MOUNTING                    UPDATING (new props / setState / forceUpdate)     UNMOUNTING
 ────────                    ─────────────────────────────────────────────     ──────────
 constructor()                                                                 componentWillUnmount()
      │
 static getDerivedStateFromProps()  ◄────── also runs before every update
      │                                     │
      │                              shouldComponentUpdate()  → false? stop here
      │                                     │
 render()  ───────── "render phase" (pure, may be paused/restarted) ─────────
      │                                     │
      │                              getSnapshotBeforeUpdate()  (read DOM before change)
      ▼                                     ▼
 ─────────────── "commit phase": React updates the real DOM ────────────────
      │                                     │
 componentDidMount()               componentDidUpdate(prevProps, prevState, snapshot)
```

| Method | Phase | Use it for |
| --- | --- | --- |
| `constructor` | Mount | Init state, bind methods. No side effects. |
| `static getDerivedStateFromProps(props, state)` | Mount + Update | Rare: derive state from props. Returns object or `null`. |
| `render` | Mount + Update | Return JSX. Must be pure. |
| `componentDidMount` | Mount | Fetch data, subscriptions, DOM measurements. |
| `shouldComponentUpdate(nextProps, nextState)` | Update | Return `false` to skip re-render (perf). |
| `getSnapshotBeforeUpdate(prevProps, prevState)` | Update | Capture DOM info (e.g. scroll position) just before commit. |
| `componentDidUpdate(prevProps, prevState, snapshot)` | Update | React to changes; compare with prev props before fetching. |
| `componentWillUnmount` | Unmount | Cleanup: timers, listeners, sockets. |
| `static getDerivedStateFromError` / `componentDidCatch` | Error | Error boundaries (see topic 14). |

Legacy methods `componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate` are prefixed `UNSAFE_` because they break under concurrent rendering. Avoid them.

### Class ↔ Hooks mapping

| Class | Hooks equivalent |
| --- | --- |
| `this.state` + `setState` | `useState` / `useReducer` |
| `componentDidMount` | `useEffect(() => {...}, [])` |
| `componentDidUpdate` | `useEffect(() => {...}, [dep])` |
| `componentWillUnmount` | cleanup function returned from `useEffect` |
| `shouldComponentUpdate` / `PureComponent` | `React.memo` (or the React Compiler) |
| `getSnapshotBeforeUpdate` | `useLayoutEffect` (approximate) |
| `getDerivedStateFromProps` | compute during render, or `key` to reset |
| `this.myRef = createRef()` | `useRef` |
| `static contextType` | `useContext` / `use` |
| Error boundary | **No hook** — still a class |

### PureComponent

`React.PureComponent` implements `shouldComponentUpdate` with a **shallow comparison** of props and state. If nothing changed by reference, render is skipped. It's the class version of `React.memo`. Mutating an object/array in place breaks it (reference is the same → no update).

### Why hooks replaced classes

- **Logic reuse:** classes needed HOCs/render props (wrapper hell); hooks share logic via custom hooks.
- **Related code split apart:** a subscription was set up in `componentDidMount` and torn down in `componentWillUnmount`; `useEffect` keeps setup + cleanup together.
- **`this` confusion** and binding boilerplate disappear.
- Functions are easier to minify, type, and optimise (the React Compiler only targets function components and hooks).

## 💻 Code examples

```jsx
import React from "react";

class Counter extends React.Component {
  constructor(props) {
    super(props); // must come first, otherwise `this` is undefined
    this.state = { count: 0, label: "Clicks" };
    this.increment = this.increment.bind(this); // option 1: bind
  }

  // option 2: class-field arrow function — `this` is lexically bound
  reset = () => this.setState({ count: 0 }); // merges: `label` is kept

  increment() {
    // updater form: safe when called several times in one event
    this.setState((prev) => ({ count: prev.count + 1 }));
  }

  componentDidMount() {
    this.timer = setInterval(() => console.log("tick"), 1000);
  }

  componentDidUpdate(prevProps, prevState) {
    if (prevState.count !== this.state.count) {
      document.title = `Count: ${this.state.count}`;
    }
  }

  componentWillUnmount() {
    clearInterval(this.timer); // avoid memory leaks
  }

  render() {
    return (
      <button onClick={this.increment} onDoubleClick={this.reset}>
        {this.state.label}: {this.state.count}
      </button>
    );
  }
}
```

The same with hooks:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => console.log("tick"), 1000);
    return () => clearInterval(timer); // = componentWillUnmount
  }, []); // = componentDidMount

  useEffect(() => {
    document.title = `Count: ${count}`; // = componentDidUpdate on count
  }, [count]);

  return <button onClick={() => setCount((c) => c + 1)}>Clicks: {count}</button>;
}
```

## ⚠️ Common mistakes & gotchas

- Forgetting `super(props)` → `this` is not initialised.
- Calling `setState` in `componentDidUpdate` without a condition → infinite loop.
- Reading `this.state` right after `setState` → still the old value (updates are batched).
- Passing `this.handleClick` without binding → `this` is `undefined` in the handler.
- Mutating state (`this.state.items.push(x)`) instead of creating a new array.
- Fetching in the constructor — side effects belong in `componentDidMount`.
- React 19 removed string refs (`ref="input"`) and legacy context (`contextTypes`); old class code using them must migrate.

## 🎤 Say this in the interview

> Class components have three lifecycle phases — mount, update, unmount — with methods like `componentDidMount`, `componentDidUpdate` and `componentWillUnmount`. Hooks map onto them (`useEffect` with deps and cleanup), but group code by concern instead of by time. I write new code with hooks; the one thing that still needs a class is an error boundary.

## ❓ Likely interview questions

**What is the order of lifecycle methods on mount?**

`constructor` → `getDerivedStateFromProps` → `render` → (DOM updated) → `componentDidMount`.

**And on update?**

`getDerivedStateFromProps` → `shouldComponentUpdate` → `render` → `getSnapshotBeforeUpdate` → (DOM updated) → `componentDidUpdate`.

**How is `setState` different from the `useState` setter?**

`setState` shallow-merges objects into state; the `useState` setter replaces the value entirely. Both are batched.

**What's the difference between render phase and commit phase?**

Render phase calls `render` and computes the new tree — it must be pure and may be interrupted. Commit phase applies DOM changes and runs `componentDidMount`/`componentDidUpdate` (and effects).

**What is `PureComponent`?**

A class component with a built-in shallow-compare `shouldComponentUpdate`; the class equivalent of `React.memo`.

**How do you replicate `componentWillUnmount` with hooks?**

Return a cleanup function from `useEffect`. It runs on unmount and before the effect re-runs.

**Is `useEffect(fn, [])` exactly `componentDidMount`?**

Close, but not identical: `componentDidMount` runs synchronously after DOM commit (like `useLayoutEffect`), while `useEffect` runs after the browser paints. Also, in Strict Mode dev, effects mount → unmount → mount to surface missing cleanup.

**Why do we still need class components?**

Error boundaries (`getDerivedStateFromError` / `componentDidCatch`) have no hook equivalent yet. Otherwise only for legacy code.

⬅️ Prev: [08 Custom Hooks](../08-Custom-Hooks/README.md) | Next ➡️: [10 Context API](../10-Context-API/README.md)
