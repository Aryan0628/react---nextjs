# 🖱️ Events and Forms

> **TL;DR:** React wraps native events in cross-browser **SyntheticEvents**, attaches handlers with camelCase props, and delegates listening to the root container. Inputs are **controlled** (React state is the source of truth) or **uncontrolled** (the DOM holds the value, you read it when needed). React 19 adds **form Actions**: `<form action={fn}>`, `useActionState`, `useFormStatus`.

## 🧠 Concept

### Synthetic events

Your handler receives a `SyntheticEvent`: a React wrapper with the same interface as the native event (`target`, `preventDefault()`, `stopPropagation()`), but consistent across browsers. The original is available as `e.nativeEvent`. (Event pooling, where React reused event objects, was removed in React 17, so you can safely use `e` asynchronously.)

### Handlers are camelCase props that take a function

```jsx
<button onClick={handleClick}>OK</button>   // ✅ pass the function
<button onClick={handleClick()}>OK</button> // ❌ calls it during render
<button onClick={() => remove(id)}>X</button> // ✅ arrow wrapper to pass args
```

Think of it as handing React a phone number to call later, not making the call right now.

### Controlled vs uncontrolled inputs

- **Controlled**: the input's value comes from React state (`value={name}`) and every keystroke goes through `onChange` into state. React is the single source of truth.
- **Uncontrolled**: the DOM keeps the value. You give an initial `defaultValue` and read the current value via a `ref` or `FormData` on submit.

Analogy: controlled is a puppet (React moves every string); uncontrolled is a free actor you only ask "what did you say?" at the end.

## ⚙️ How it works / Deep dive

### preventDefault, bubbling and stopPropagation

Events **bubble** from the clicked element up through its ancestors, and React's synthetic events bubble through the **React tree** the same way (even through portals).

```jsx
function Card() {
  return (
    <div onClick={() => console.log('card clicked')}>
      <button
        onClick={(e) => {
          e.stopPropagation();        // card handler will NOT run
          console.log('button clicked');
        }}
      >
        Like
      </button>
      <a href="/x" onClick={(e) => e.preventDefault()}>No navigation</a>
    </div>
  );
}
```

- `preventDefault()` stops the browser's default action (form submit reload, link navigation).
- `stopPropagation()` stops parents' handlers from firing.
- Capture phase: use `onClickCapture` to run on the way down.
- `return false` does **not** prevent default in React (it did in old jQuery/HTML handlers).

### Event delegation at the root (React 17+)

React does not attach a listener to every button. It attaches one listener per event type to the **root container** (`#root`) and dispatches to the right component handler itself. Before React 17 this was on `document`, which clashed with other libraries and multiple React versions on one page; React 17 moved it to the root.

### Controlled inputs by type

```jsx
function Controls() {
  const [name, setName] = useState('');
  const [agree, setAgree] = useState(false);
  const [city, setCity] = useState('delhi');

  return (
    <>
      <input value={name} onChange={(e) => setName(e.target.value)} />

      {/* checkbox uses `checked`, not `value` */}
      <input type="checkbox" checked={agree} onChange={(e) => setAgree(e.target.checked)} />

      {/* select: put `value` on <select>, not `selected` on <option> */}
      <select value={city} onChange={(e) => setCity(e.target.value)}>
        <option value="delhi">Delhi</option>
        <option value="mumbai">Mumbai</option>
      </select>
    </>
  );
}
```

### Multi-field form with one handler

Use the input's `name` attribute as the key in a single state object.

```jsx
function SignupForm() {
  const [form, setForm] = useState({ email: '', password: '', remember: false });

  function handleChange(e) {
    const { name, value, type, checked } = e.target;
    setForm((prev) => ({ ...prev, [name]: type === 'checkbox' ? checked : value }));
  }

  function handleSubmit(e) {
    e.preventDefault(); // stop full page reload
    console.log(form);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" value={form.email} onChange={handleChange} />
      <input name="password" type="password" value={form.password} onChange={handleChange} />
      <input name="remember" type="checkbox" checked={form.remember} onChange={handleChange} />
      <button type="submit">Sign up</button>
    </form>
  );
}
```

### Uncontrolled input

```jsx
function SearchBox() {
  const inputRef = useRef(null);
  const handleSubmit = (e) => {
    e.preventDefault();
    console.log(inputRef.current.value);              // via ref
    console.log(new FormData(e.currentTarget).get('q')); // or via FormData
  };
  return (
    <form onSubmit={handleSubmit}>
      <input name="q" ref={inputRef} defaultValue="react" />
    </form>
  );
}
```

File inputs (`<input type="file">`) are **always uncontrolled**.

### When to use which

| Controlled | Uncontrolled |
| --- | --- |
| Live validation, formatting, masking | Simple forms read only on submit |
| Disable submit until valid | Integrating non-React libraries |
| Value depends on other state | File inputs |
| More re-renders (every keystroke) | Fewer re-renders, less code |

### Form libraries: React Hook Form + Zod

For big forms, **React Hook Form** uses uncontrolled inputs under the hood (few re-renders) and **Zod** defines a schema used for both validation and TypeScript types.

```tsx
const schema = z.object({ email: z.string().email(), age: z.coerce.number().min(18) });
type FormData = z.infer<typeof schema>;

const { register, handleSubmit, formState: { errors } } =
  useForm<FormData>({ resolver: zodResolver(schema) });

<form onSubmit={handleSubmit((data) => save(data))}>
  <input {...register('email')} />
  {errors.email && <p>{errors.email.message}</p>}
</form>;
```

### React 19 form Actions (brief)

- `<form action={fn}>`: React calls `fn(formData)` on submit, no `preventDefault` needed. Uncontrolled fields reset after a successful action. In Next.js `fn` can be a **Server Action**.
- `useActionState(action, initialState)` returns `[state, formAction, isPending]`; your action receives `(prevState, formData)` and returns the new state (great for errors / success messages).
- `useFormStatus()` (from `react-dom`) gives `{ pending, data }` of the **parent** form; it must be called in a component rendered **inside** the `<form>`.

```jsx
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';

async function subscribe(prev, formData) {
  const email = formData.get('email');
  if (!email.includes('@')) return { error: 'Invalid email' };
  await api.subscribe(email);
  return { ok: true };
}

function SubmitButton() {
  const { pending } = useFormStatus(); // reads the enclosing form
  return <button disabled={pending}>{pending ? 'Saving...' : 'Subscribe'}</button>;
}

function Newsletter() {
  const [state, formAction] = useActionState(subscribe, {});
  return (
    <form action={formAction}>
      <input name="email" />
      <SubmitButton />
      {state.error && <p>{state.error}</p>}
      {state.ok && <p>Subscribed!</p>}
    </form>
  );
}
```

## ⚠️ Common mistakes & gotchas

- `onClick={fn()}`: runs on render (and may cause an infinite loop if it sets state).
- Forgetting `e.preventDefault()` in `onSubmit`: page reloads.
- `value` without `onChange`: read-only input plus a warning. Use `defaultValue` or add a handler.
- Switching between controlled and uncontrolled (`value={undefined}` then a string): initialise with `''`.
- Using `value` on checkboxes instead of `checked`.
- Calling `useFormStatus` in the same component that renders the `<form>`: it returns nothing useful.

## 🎤 Say this in the interview

> React uses synthetic events delegated at the root, so handlers get a consistent cross-browser event. For forms, controlled inputs keep the value in state for live validation, uncontrolled inputs leave it in the DOM and read it on submit; for large forms I use React Hook Form with Zod, and React 19 Actions with `useActionState` for submit-and-pending flows.

## ❓ Likely interview questions

**What is a SyntheticEvent?**

React's cross-browser wrapper around the native event with the same API; the native one is at `e.nativeEvent`.

**Controlled vs uncontrolled component?**

Controlled: value lives in React state and updates through `onChange`. Uncontrolled: value lives in the DOM and is read via ref or FormData.

**How do you pass an argument to an event handler?**

Wrap it in an arrow function: `onClick={() => handleDelete(id)}`.

**How does event delegation work in React?**

React attaches one listener per event type at the root container (since React 17) and dispatches to component handlers internally.

**How do you handle many inputs with one handler?**

Give each input a `name` and update state with a computed key: `[e.target.name]: e.target.value`.

**What do `useActionState` and `useFormStatus` do?**

`useActionState` wires an action to a form and returns its latest result and pending flag; `useFormStatus` lets a child of a form read whether that form is submitting.

**Why use React Hook Form?**

It minimises re-renders by using uncontrolled inputs and integrates schema validation (Zod) with typed results.

---

⬅️ Prev: [Lists, Keys and Conditional Rendering](../04-Lists-Keys-Conditional-Rendering/README.md) | Next ➡️: [Hooks: useState and useEffect](../06-Hooks-useState-useEffect/README.md)
