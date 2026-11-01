# 🌳 Virtual DOM and Reconciliation

> **TL;DR:** The Virtual DOM is a lightweight JS-object copy of the UI. On every state change React builds a new tree, **diffs** it against the previous one (reconciliation), and applies only the minimal changes to the real DOM. **Fiber** is the engine that does this work in small, interruptible, prioritised chunks.

## 🧠 Concept

### Why direct DOM updates are slow

The DOM itself is not "slow" to touch once, but each change can trigger **style recalculation, layout (reflow) and paint**. If code updates many nodes one by one, or reads layout in between writes, the browser does that expensive work repeatedly. Hand-writing the minimal set of updates for a complex UI is also error-prone.

### What the Virtual DOM is

A **plain JavaScript object tree** that describes what the UI should look like. Creating and comparing JS objects is cheap. React compares the old and new trees and then does one batched set of real DOM writes.

Analogy: an architect edits the **blueprint** (VDOM) as much as needed, then the builders change only the walls that actually moved (real DOM).

```jsx
// <div className="box"><p>Hi</p></div> is roughly this object:
{
  type: 'div',
  props: {
    className: 'box',
    children: { type: 'p', props: { children: 'Hi' } }
  }
}
```

## ⚙️ How it works / Deep dive

### What happens on a state change (step by step)

1. You call `setState` (e.g. `setCount(1)`).
2. React schedules a re-render of that component (updates are batched).
3. **Render phase:** React calls the component function again, producing a new element tree for it and its children.
4. **Reconciliation:** React diffs the new tree against the previous one and records what changed.
5. **Commit phase:** React applies those changes to the real DOM in one go.
6. Browser paints, then `useEffect` callbacks run.

```text
 state change
      |
      v
+-------------+    diff     +-------------+
|  Old VDOM   | <---------> |  New VDOM   |
|  <ul>       |             |  <ul>       |
|   <li>A     |             |   <li>A     |
|   <li>B     |             |   <li>B*    |  * text changed
+-------------+             |   <li>C+    |  + new node
                            +-------------+
      |
      v
 Commit: update text of li#2, insert li#3 (nothing else touched)
      |
      v
   Real DOM
```

### Diffing heuristics (why it's O(n), not O(n³))

A perfect tree diff is O(n³). React makes two assumptions to get roughly O(n):

| Situation | What React does |
| --- | --- |
| Element **type changes** (`<div>` to `<span>`, `<A>` to `<B>`) | Destroys the whole old subtree (state lost) and builds a new one |
| **Same type** DOM element | Keeps the node, updates only changed attributes/styles |
| **Same type** component | Keeps the instance and its state, re-renders with new props |
| **Lists** of children | Matches items by `key`; without keys, matches by position |

So a stable **type** and a stable **key** at the same position are what preserve state.

```jsx
// Toggling this destroys and recreates <Counter>, its state resets,
// because the parent element type changes from div to section.
{isFancy ? <div><Counter /></div> : <section><Counter /></section>}

// Changing a key forces a fresh instance on purpose (handy to reset a form)
<ProfileForm key={userId} />
```

### Fiber (React 16+)

Before Fiber, React's "stack reconciler" walked the tree recursively in one go. A large update blocked the main thread, so typing and animations stuttered.

**Fiber** rewrote the reconciler so that each component becomes a *fiber*: a unit of work (a JS object with links to its parent, child and sibling). Because work is a linked list of units, React can:

- **Split work** into chunks and yield back to the browser between them.
- **Pause, resume or throw away** work in progress.
- **Prioritise** updates: typing in an input (urgent) beats re-rendering a big list (can wait).
- Keep two trees: the **current** tree on screen and a **work-in-progress** tree being built.

That ability to interrupt rendering is exactly what enables **concurrent features** like `useTransition`, `useDeferredValue` and Suspense.

### Render phase vs commit phase

| | Render phase | Commit phase |
| --- | --- | --- |
| What happens | Call components, build new tree, diff | Apply DOM mutations, set refs, run layout effects |
| Interruptible? | Yes (in concurrent rendering) | No, always synchronous |
| Side effects allowed? | No, must be **pure** | Yes |
| May run more than once? | Yes (StrictMode, discarded work) | Once per update |

This is why component bodies must be pure: React may call them and discard the result.

### Virtual DOM vs Shadow DOM

| | Virtual DOM | Shadow DOM |
| --- | --- | --- |
| What | JS object copy of the UI | Browser-native encapsulated DOM subtree |
| Purpose | Efficient updates via diffing | Style and markup **encapsulation** |
| Who provides it | Libraries (React, Vue) | The browser (Web Components) |
| Related? | No, they solve different problems | |

## 💻 Code examples

```jsx
function App() {
  const [count, setCount] = useState(0);
  return (
    <div className="app">          {/* same type, kept as-is */}
      <h1>Static title</h1>        {/* unchanged, no DOM write */}
      <p>Count: {count}</p>        {/* only this text node is updated */}
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}
// Clicking re-runs App(), but the commit touches a single text node.
```

## ⚠️ Common mistakes & gotchas

- "VDOM makes React faster than vanilla JS": not true. Hand-optimised DOM code can be faster; the VDOM makes *good-enough* updates automatic and declarative.
- "Re-render means the DOM is rebuilt": no. Re-render means the function runs again; the DOM changes only where the diff found differences.
- Defining a component **inside** another component creates a new type every render, so React unmounts and remounts it (state lost, inputs lose focus).
- Using array index as `key` in reorderable lists confuses the diff (see topic 04).
- Side effects in the render body break because render can run multiple times.

## 🎤 Say this in the interview

> On a state change React re-runs the component to get a new virtual tree, diffs it against the previous one using type and key heuristics, and commits only the minimal DOM changes. Fiber splits that render work into interruptible, prioritised units, which is what makes concurrent features like transitions possible.

## ❓ Likely interview questions

**What is the Virtual DOM?**

An in-memory JS object representation of the UI that React diffs between renders to compute minimal real-DOM updates.

**What is reconciliation?**

The process of comparing the new element tree with the previous one to decide what to update, keep or remove.

**What happens if an element's type changes?**

React tears down the old subtree, including component state, and mounts a new one.

**Why are keys important in reconciliation?**

They let React match list items across renders by identity instead of position, so it can reorder instead of re-creating and preserve the right state.

**What is React Fiber?**

The reconciliation engine since React 16, which represents work as units (fibers) that can be paused, prioritised and resumed, enabling concurrent rendering.

**Render phase vs commit phase?**

Render is pure and interruptible (compute changes); commit is synchronous and applies DOM changes, refs and layout effects.

**Is Virtual DOM the same as Shadow DOM?**

No. VDOM is a diffing strategy in JS; Shadow DOM is a browser feature for encapsulating styles and markup.

---

⬅️ Prev: [Basics and JSX](../01-Basics-and-JSX/README.md) | Next ➡️: [Components, Props and State](../03-Components-Props-State/README.md)
