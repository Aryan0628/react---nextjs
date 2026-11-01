# 🧭 React Router

> **TL;DR:** React Router maps URLs to components **in the browser** without full page reloads, using the History API. v7 (current) ships as one package, `react-router`, with three modes: **declarative** (`<BrowserRouter>` + `<Routes>`), **data** (`createBrowserRouter` + loaders/actions) and **framework** (the successor of Remix, with SSR and file-based route config).

## 🧠 Concept

### Server routing vs client-side routing

Traditional sites: every link click asks the server for a new HTML page → full reload, white flash, all JS state lost.

A **SPA with client-side routing** loads one HTML page once. When you click a link, the router:

1. Calls `history.pushState()` to change the URL **without** a request.
2. Re-renders the React tree to show the component matching the new URL.
3. Listens to the `popstate` event so back/forward buttons work.

Analogy: a theatre that changes the set between scenes instead of making the audience leave and buy new tickets.

### Versions at a glance

- **v6** (2021): `<Routes>` replaced `<Switch>`, `element={<X />}`, nested routes + `<Outlet>`, hooks API. v6.4 added data routers (loaders/actions).
- **v7** (late 2024): non-breaking upgrade from v6; `react-router-dom` merged into `react-router`; Remix merged in as **framework mode**.

## ⚙️ How it works / Deep dive

### Navigation building blocks

| Tool | Use |
| --- | --- |
| `<a href>` | Full page reload — only for external sites |
| `<Link to="/about">` | Client-side navigation, no reload |
| `<NavLink to="/about">` | `Link` that knows if it's active (`isActive`) — for nav menus |
| `useNavigate()` | Navigate in code: after form submit, logout, redirect |
| `<Navigate to="/login" />` | Redirect while rendering (e.g. route guards) |

### Reading the URL

| Hook | Returns | Example |
| --- | --- | --- |
| `useParams()` | Dynamic segments | `/users/:id` → `{ id: "42" }` |
| `useSearchParams()` | `[searchParams, setSearchParams]` for the query string | `?sort=price&page=2` |
| `useLocation()` | `{ pathname, search, hash, state }` | Analytics, "redirect back after login" |

### Nested routes, Outlet, index & 404

- **Nested routes** share a layout: the parent renders `<Outlet />` where the matched child should appear.
- **Index route** (`index`) is the default child shown at the parent's exact path.
- **Splat route** `path="*"` matches anything unmatched → 404 page.

## 💻 Code examples

### Declarative mode (classic v6 style, still valid in v7)

```jsx
import { BrowserRouter, Routes, Route, Link, NavLink, Outlet } from "react-router"; // v6: "react-router-dom"

function Layout() {
  return (
    <>
      <nav>
        <NavLink to="/" end className={({ isActive }) => (isActive ? "active" : "")}>Home</NavLink>
        <NavLink to="/users">Users</NavLink>
      </nav>
      <main><Outlet /></main> {/* child route renders here */}
    </>
  );
}

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route index element={<Home />} />              {/* "/" */}
          <Route path="users" element={<Users />} />      {/* "/users" */}
          <Route path="users/:id" element={<UserDetail />} />
          <Route element={<ProtectedRoute />}>            {/* pathless guard */}
            <Route path="dashboard" element={<Dashboard />} />
          </Route>
          <Route path="*" element={<NotFound />} />       {/* 404 */}
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

### Params, search params, navigate

```jsx
import { useParams, useSearchParams, useNavigate, useLocation } from "react-router";

function UserDetail() {
  const { id } = useParams();                       // always a string
  const [searchParams, setSearchParams] = useSearchParams();
  const tab = searchParams.get("tab") ?? "profile"; // /users/42?tab=posts
  const navigate = useNavigate();
  const location = useLocation();

  return (
    <>
      <h1>User {id} — {tab}</h1>
      <button onClick={() => setSearchParams({ tab: "posts" })}>Posts</button>
      <button onClick={() => navigate(-1)}>Back</button>
      <button onClick={() => navigate("/users", { replace: true })}>All users</button>
      <small>{location.pathname}</small>
    </>
  );
}
```

### Protected route

```jsx
import { Navigate, Outlet, useLocation } from "react-router";

function ProtectedRoute() {
  const { user } = useAuth();
  const location = useLocation();
  if (!user) {
    // remember where they wanted to go; replace so "back" doesn't loop
    return <Navigate to="/login" replace state={{ from: location }} />;
  }
  return <Outlet />; // render the protected child routes
}

// In Login, after success:
// navigate(location.state?.from?.pathname ?? "/dashboard", { replace: true });
```

Note: client-side guards are UX only — the API must still check authorization.

### Data mode: loaders, actions, lazy routes

```jsx
import { createBrowserRouter, useLoaderData, Form, redirect } from "react-router";
import { RouterProvider } from "react-router/dom";

const router = createBrowserRouter([
  {
    path: "/",
    Component: Layout,
    errorElement: <ErrorPage />,                  // route-level error boundary
    children: [
      { index: true, Component: Home },
      {
        path: "users/:id",
        loader: async ({ params }) => fetch(`/api/users/${params.id}`).then((r) => r.json()),
        action: async ({ request, params }) => {
          const form = await request.formData();
          await updateUser(params.id, form.get("name"));
          return redirect(`/users/${params.id}`);
        },
        Component: UserPage,
      },
      { path: "admin", lazy: () => import("./routes/admin") }, // module exports Component, loader…
    ],
  },
]);

function UserPage() {
  const user = useLoaderData();          // data loaded BEFORE render — no useEffect waterfall
  return (
    <Form method="post">                 {/* submits to the route action */}
      <input name="name" defaultValue={user.name} />
      <button>Save</button>
    </Form>
  );
}

createRoot(document.getElementById("root")).render(<RouterProvider router={router} />);
```

Loaders run in parallel for all matched routes when navigation starts, and after an action React Router automatically revalidates loaders. Other helpers: `useNavigation()` (pending state), `useFetcher()` (mutations without navigation).

In declarative mode you can lazy-load with `React.lazy`: `const Admin = lazy(() => import("./Admin"))` + `<Suspense>` around `<Routes>`.

### Framework mode (Remix successor)

```jsx
// app/routes.ts — route config
import { index, route } from "@react-router/dev/routes";
export default [index("routes/home.tsx"), route("users/:id", "routes/user.tsx")];

// app/routes/user.tsx — a route module
export async function loader({ params }) {
  return getUser(params.id);         // runs on the server (SSR)
}
export default function User({ loaderData }) {
  return <h1>{loaderData.name}</h1>;
}
```

Framework mode adds a Vite plugin, SSR/SSG/SPA rendering, code splitting per route, and type-safe route modules — a full-stack framework comparable to Next.js.

## ⚠️ Common mistakes & gotchas

- Using `<a href>` for internal links → full reload and lost state.
- Forgetting `<Outlet />` in a parent layout → child routes never render.
- Forgetting `end` on a "/" `NavLink` → Home stays active on every page.
- `useParams` values are strings — convert with `Number(id)`.
- Hosting an SPA without a fallback → refreshing `/users/42` 404s on the server; configure all paths to serve `index.html`.
- Calling `navigate()` during render instead of using `<Navigate />` or an effect/event.
- Mixing v5 APIs (`Switch`, `component=`, `useHistory`) with v6/v7.

## 🎤 Say this in the interview

> React Router does client-side routing with the History API, so navigation swaps components without a page reload. I use nested routes with `<Outlet>` for layouts, `useParams`/`useSearchParams` for URL state, a guard component that returns `<Navigate>` for protected routes, and in data mode loaders and actions to fetch before render. v7 merged Remix in as framework mode for SSR.

## ❓ Likely interview questions

**How does client-side routing work?**

The router intercepts link clicks, updates the URL with `history.pushState`, and renders the matching component; `popstate` handles back/forward.

**`Link` vs `NavLink` vs `<a>`?**

`Link` navigates without reload; `NavLink` additionally exposes active state for styling; `<a>` triggers a full page load.

**What is `<Outlet />`?**

The placeholder in a parent route's element where the matched child route renders.

**How do you implement a protected route?**

A wrapper route component that checks auth and returns `<Navigate to="/login" replace />` or `<Outlet />`.

**What are loaders and actions?**

Route-level functions in data/framework mode: loaders fetch data before the route renders; actions handle form submissions/mutations, then loaders revalidate.

**`useNavigate` vs `<Navigate>`?**

`useNavigate` returns a function for imperative navigation in events/effects; `<Navigate>` redirects declaratively when rendered.

**What is React Router v7 framework mode?**

The evolution of Remix: file/config-based routes, server loaders/actions, SSR and a Vite plugin — a full-stack React framework.

⬅️ Prev: [15 Suspense, Lazy & Concurrent React](../15-Suspense-Lazy-Concurrent-React/README.md) | Next ➡️: [17 Interview Q&A](../17-Interview-QA/README.md)
