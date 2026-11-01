# ⚛️ React Basics and JSX

> **TL;DR:** React is a JavaScript **library** for building UIs out of **components**. You describe *what* the UI should look like for a given state, and React updates the DOM for you. JSX is just syntax sugar that compiles to plain function calls (`jsx()` / `React.createElement()`).

## 🧠 Concept

### What React is (library vs framework)

React only handles the **view layer**: rendering UI and keeping it in sync with data. It does not ship routing, data fetching or a build system. That makes it a *library*: you call it and choose the other pieces yourself. A *framework* (Next.js, Angular) calls your code and decides structure for you. Next.js is a framework built **on top of** React.

### Declarative vs imperative

- **Imperative** = you give step-by-step instructions ("find the element, change its text, add a class").
- **Declarative** = you describe the end result ("when count is 3, show 3") and React works out the steps.

Analogy: imperative is giving a taxi driver every turn; declarative is just giving the address.

```js
// Imperative: plain DOM, you manage every change yourself
const btn = document.querySelector('#btn');
const label = document.querySelector('#label');
let count = 0;
btn.addEventListener('click', () => {
  count++;
  label.textContent = `Clicked ${count} times`; // manual DOM update
});
```

```jsx
// Declarative: React, you describe UI as a function of state
function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount(count + 1)}>
      Clicked {count} times
    </button>
  );
}
```

### Component-based

A UI is split into small, reusable, self-contained pieces (`Navbar`, `Card`, `Button`). Each component owns its markup, logic and (optionally) state. Components nest inside each other to form a **tree**.

### One-way (unidirectional) data flow

Data flows **down** from parent to child through props. A child cannot modify its parent's data directly; it can only call a function the parent passed down. This makes data changes predictable and easy to trace.

### SPA (Single Page Application)

The browser loads one HTML page once. After that, JavaScript swaps content in and out as you navigate, without a full page reload. Fast after the first load, but plain client-side SPAs have slower first paint and weaker SEO. Frameworks like Next.js fix this with server rendering.

## ⚙️ How it works / Deep dive

### What JSX compiles to

Browsers do not understand JSX. A compiler (Babel, SWC, esbuild) turns it into function calls that return plain JS objects called **React elements**.

```jsx
// You write
const el = <h1 className="title">Hello</h1>;

// New JSX transform (React 17+), auto-imported, no need to import React
import { jsx as _jsx } from 'react/jsx-runtime';
const el = _jsx('h1', { className: 'title', children: 'Hello' });

// Classic transform (old): this is why files used to need `import React`
const el = React.createElement('h1', { className: 'title' }, 'Hello');
```

The result is just an object like `{ type: 'h1', props: { className: 'title', children: 'Hello' } }`. React later turns these objects into real DOM nodes.

### JSX rules

| Rule | Why / Example |
| --- | --- |
| `className` not `class` | `class` is a reserved JS keyword |
| `htmlFor` not `for` | `for` is a reserved JS keyword |
| One parent element | A function can return only one value, so wrap siblings |
| Use Fragments `<>...</>` | Groups siblings without adding an extra `div` |
| Self-close empty tags | `<img />`, `<input />`, `<MyComp />` |
| camelCase attributes | `onClick`, `tabIndex`, `strokeWidth` (except `aria-*` and `data-*`) |
| `{}` holds **expressions** | `{a + b}`, `{fn()}`, `{ok ? 'Y' : 'N'}` work; `if`, `for` do not |
| Components start with a capital | `<card>` is treated as an HTML tag, `<Card>` as your component |

### Style objects

Inline styles take a **JS object**, not a string. Keys are camelCase, numbers default to `px`.

```jsx
<div style={{ backgroundColor: 'teal', fontSize: 18, marginTop: '1rem' }}>
  Styled box
</div>
// Outer {} = "JS expression here", inner {} = the object literal
```

### How a React app boots

1. `index.html` has an empty container: `<div id="root"></div>`.
2. The bundler loads `main.jsx`.
3. `createRoot` attaches React to that div and `render` mounts your top component.

```jsx
// main.jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

`ReactDOM.render` (the old API) was **removed in React 19**. Use `createRoot` (or `hydrateRoot` for server-rendered HTML).

### Vite vs Create React App

- **Create React App (CRA)** was officially **deprecated in Feb 2025**. It used Webpack and was slow to start and rebuild.
- **Vite** is the go-to for a plain React SPA: native ES modules in dev, near-instant start and fast HMR.
- For full apps, the React docs recommend a **framework** such as Next.js or React Router (framework mode).

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app && npm install && npm run dev
```

## 💻 Code examples

```jsx
// Fragment + expressions + list + conditional, all in one
function Profile({ user }) {
  const fullName = `${user.first} ${user.last}`; // logic lives above the return

  return (
    <>
      <h2 className="name">{fullName}</h2>
      <label htmlFor="bio">Bio</label>
      <textarea id="bio" defaultValue={user.bio} />
      {user.isAdmin ? <span>Admin</span> : null}
      <img src={user.avatar} alt={fullName} />
    </>
  );
}
```

## ⚠️ Common mistakes & gotchas

- Writing `if` / `for` inside `{}`: use a ternary, `&&`, or `.map()` instead.
- `style="color: red"`: must be an object, `style={{ color: 'red' }}`.
- Forgetting to close tags: `<br>` is an error, write `<br />`.
- Lower-case component names: `<myButton />` renders an unknown HTML tag.
- Returning two siblings without a wrapper: wrap them in a Fragment.
- Thinking JSX is HTML: it is JavaScript, so attributes are JS property names.

## 🎤 Say this in the interview

> React is a declarative, component-based UI library: I describe the UI as a function of state, and React efficiently syncs the DOM. JSX is syntactic sugar that compiles to `jsx()` calls returning plain element objects, which React then renders through `createRoot`.

## ❓ Likely interview questions

**Is React a library or a framework?**

A library: it only handles the view layer. Frameworks like Next.js add routing, data fetching and rendering strategies on top of it.

**What does declarative mean in React?**

You describe what the UI should look like for a given state and React figures out the DOM operations needed to get there.

**What does JSX compile to?**

Calls to `jsx()` from `react/jsx-runtime` (React 17+), or `React.createElement()` in the classic transform. Both return plain JS objects (React elements).

**Why `className` instead of `class`?**

JSX is JavaScript, and `class` is a reserved keyword. Similarly `for` becomes `htmlFor`.

**Why must JSX return a single parent?**

A component is a function and can return only one value. Use a Fragment `<>...</>` to avoid an extra DOM node.

**What is one-way data flow?**

Data passes downward via props. Children notify parents only through callback functions, which keeps changes predictable.

**Why is CRA no longer recommended?**

It was deprecated in 2025 because it was slow and unmaintained. Use Vite for an SPA or a framework like Next.js for full apps.

**What does `createRoot` do?**

It creates a React root attached to a DOM container, enabling concurrent features; `root.render(<App />)` mounts the tree.

---

⬅️ Prev: [React Index](../README.md) | Next ➡️: [Virtual DOM and Reconciliation](../02-Virtual-DOM-and-Reconciliation/README.md)
