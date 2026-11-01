# 📋 Lists, Keys and Conditional Rendering

> **TL;DR:** Render lists with `.map()` and give every item a **stable, unique `key`** so React can track identity between renders. Using the array index as key breaks state when items are reordered, inserted or deleted. Show or hide UI with ternaries, `&&`, early returns or `null`, and watch out for `0 &&` rendering a literal `0`.

## 🧠 Concept

### Rendering lists

JSX can render an **array of elements**. `.map()` turns an array of data into an array of JSX.

```jsx
const fruits = ['Apple', 'Mango', 'Kiwi'];

<ul>
  {fruits.map((fruit) => <li key={fruit}>{fruit}</li>)}
</ul>;
```

### Why keys exist

When a list changes, React has to answer: "which old item corresponds to which new item?" Without keys it matches by **position** (1st with 1st, 2nd with 2nd). A `key` is the item's **identity**, like a name tag at a party: even if people swap seats, you still know who is who.

With good keys React can **move** existing DOM nodes and keep each component's state attached to the right item, instead of re-creating or mislabelling them.

## ⚙️ How it works / Deep dive

### The index-as-key bug (concrete example)

Each row has an uncontrolled `<input>` the user types into. The input's internal value lives in the DOM node / component that React reuses based on the key.

```jsx
function TodoList() {
  const [todos, setTodos] = useState([
    { id: 'a', text: 'Buy milk' },
    { id: 'b', text: 'Walk dog' },
  ]);

  const addToTop = () =>
    setTodos([{ id: crypto.randomUUID(), text: 'New task' }, ...todos]);

  return (
    <>
      <button onClick={addToTop}>Add to top</button>
      {todos.map((todo, index) => (
        // ❌ key={index}
        <div key={index}>
          {todo.text} <input placeholder="notes" />
        </div>
      ))}
    </>
  );
}
```

What happens:

1. User types "2 litres" in the notes input next to **Buy milk** (key `0`).
2. Click "Add to top". The new array is `[New task, Buy milk, Walk dog]`.
3. With index keys, `New task` now has key `0`. React thinks key `0` is the **same item** as before, so it reuses that row and its input.
4. Result: "2 litres" now appears next to **New task**. The text labels updated, but the input state stayed with the position.

With `key={todo.id}` React sees key `a` moved to position 1, moves that DOM node, and the note stays with **Buy milk**.

### When index keys are acceptable

All three must be true: the list is **static** (never reordered/filtered), items are never inserted or removed except at the end, and items have no state (no inputs, no local state).

### Choosing good keys

| Key source | Verdict |
| --- | --- |
| Database / API `id` | ✅ Best |
| Generated once when the item is created (`crypto.randomUUID()` stored in data) | ✅ Good |
| Unique field like email or slug | ✅ Fine |
| Array index | ⚠️ Only for static, stateless lists |
| `Math.random()` / `uuid()` **inside** render | ❌ New key every render, so every item remounts |

Rules: keys must be unique **among siblings** (not globally), stable across renders, and they are **not passed as a prop** (`props.key` is undefined; pass `id` separately if the child needs it).

### Keys on Fragments

Short syntax `<>` can't take a key. Use the long form when mapping to multiple elements.

```jsx
import { Fragment } from 'react';

{terms.map((t) => (
  <Fragment key={t.id}>
    <dt>{t.word}</dt>
    <dd>{t.meaning}</dd>
  </Fragment>
))}
```

### Conditional rendering patterns

| Pattern | Use when |
| --- | --- |
| Ternary `cond ? <A /> : <B />` | Choose between two things |
| `cond && <A />` | Show something or nothing |
| Early `return` | Whole component depends on a condition (loading, error) |
| `return null` | Render nothing at all |
| Variable / object lookup | Many branches (status maps to component) |

```jsx
function Dashboard({ user, isLoading, error, notifications }) {
  if (isLoading) return <Spinner />;          // early return
  if (error) return <ErrorMsg error={error} />;
  if (!user) return null;                     // render nothing

  return (
    <>
      <h1>{user.isAdmin ? 'Admin panel' : 'Your dashboard'}</h1>
      {notifications.length > 0 && <Badge count={notifications.length} />}
    </>
  );
}

// Lookup object instead of nested ternaries
const STATUS_VIEW = { idle: <Idle />, loading: <Spinner />, done: <Done /> };
return STATUS_VIEW[status] ?? null;
```

### The `0 &&` gotcha

`&&` returns the **left value** when it's falsy. React renders numbers, so `0` shows up on screen. (`false`, `null`, `undefined` and `true` render nothing.)

```jsx
{items.length && <List items={items} />}      // ❌ renders "0" when empty
{items.length > 0 && <List items={items} />}  // ✅ boolean on the left
{!!items.length && <List items={items} />}    // ✅ also fine
```

`NaN` has the same problem, and an empty string `''` renders nothing visible but is still a text node.

### Rendering nothing

Return `null` (or `false`) from a component, or put `null` in JSX. Note the component **still runs** its hooks; only its output is empty. To fully skip a component, don't render it from the parent.

## 💻 Code examples

```jsx
// Filter + map, with stable ids and an empty state
function ProductList({ products, query }) {
  const visible = products.filter((p) =>
    p.name.toLowerCase().includes(query.toLowerCase())
  );

  if (visible.length === 0) return <p>No products found.</p>;

  return (
    <ul>
      {visible.map((p) => (
        <ProductRow key={p.id} product={p} /> // key goes on the outermost mapped element
      ))}
    </ul>
  );
}
```

## ⚠️ Common mistakes & gotchas

- Missing key: console warning plus inefficient or buggy updates.
- Key on the wrong element: it must go on the **outermost** element returned from `map`, not on a child inside it.
- Generating keys during render with `Math.random()`: every item remounts every render.
- Index as key on sortable, filterable or editable lists.
- `count && <X />` rendering `0`.
- Deeply nested ternaries: extract into variables, early returns or a small component.

## 🎤 Say this in the interview

> Keys give list items a stable identity so React can match them across renders. Index keys tie state to position, so when you insert or reorder items, things like input values end up on the wrong row; I use stable IDs from the data instead.

## ❓ Likely interview questions

**Why does React need keys?**

To identify which item is which across renders so it can reorder, add and remove efficiently and keep each item's state attached correctly.

**What's wrong with using the index as a key?**

When items are inserted, deleted or reordered, indexes shift, so React reuses the wrong component instances and their state (e.g. input text) ends up on the wrong item.

**Do keys need to be globally unique?**

No, only unique among siblings in the same list.

**Can a child read its `key` prop?**

No. `key` is consumed by React; pass the value under another prop name if needed.

**Why does `{count && <X />}` sometimes show `0`?**

`&&` returns `0` when count is zero, and React renders numbers. Use `count > 0 &&`.

**How do you render nothing from a component?**

Return `null`. The component still mounts and runs its hooks; it just outputs nothing.

**What happens if keys change every render?**

Every item is unmounted and remounted, losing state and hurting performance.

---

⬅️ Prev: [Components, Props and State](../03-Components-Props-State/README.md) | Next ➡️: [Events and Forms](../05-Events-and-Forms/README.md)
