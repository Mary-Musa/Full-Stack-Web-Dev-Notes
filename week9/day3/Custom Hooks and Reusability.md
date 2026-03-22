# Custom Hooks and Reusability - Reading Notes (React)

## 1. What Are Custom Hooks?

A **custom hook** is a JavaScript function that:

- Starts with the word `use` (required by React convention)
- Can call other React hooks inside it (`useState`, `useEffect`, `useRef`, etc.)
- Extracts and reuses stateful logic between components

Custom hooks are **not a new React feature** - they are a pattern built on top of existing hooks.

### Why They Exist

Without custom hooks, if two components need the same stateful logic, you are forced to copy and paste that logic into both. Custom hooks let you extract that logic into one place and share it.

> "Custom hooks let you extract component logic into reusable functions." - React docs

### The Core Rule

A custom hook **must start with `use`**. This is not just convention - React relies on it to enforce the rules of hooks (hooks can't be called conditionally, must be called at the top level, etc.).

```js
// Valid custom hook name
function useFetch() { ... }
function useToggle() { ... }
function useLocalStorage() { ... }

// Invalid - React won't enforce hook rules
function fetchData() { ... }
```

---

## 2. Extracting Logic From Components

The most common reason to write a custom hook is that a component is doing **too much** - it has both UI rendering and complex logic mixed together.

Custom hooks let you **separate UI from logic**.

### Before - Logic Mixed Into Component

```jsx
function UserProfile() {
  const [isOpen, setIsOpen] = useState(false);

  function toggle() {
    setIsOpen((prev) => !prev);
  }

  return (
    <div>
      <button onClick={toggle}>Toggle</button>
      {isOpen && <p>Profile details here</p>}
    </div>
  );
}
```

### After - Logic Extracted Into a Hook

```jsx
// useToggle.js
import { useState } from "react";

function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);

  function toggle() {
    setValue((prev) => !prev);
  }

  return [value, toggle];
}

export default useToggle;
```

```jsx
// UserProfile.jsx
import useToggle from "./useToggle";

function UserProfile() {
  const [isOpen, toggle] = useToggle();

  return (
    <div>
      <button onClick={toggle}>Toggle</button>
      {isOpen && <p>Profile details here</p>}
    </div>
  );
}
```

The component is now only concerned with **rendering**. The logic lives in the hook.

---

## 3. useToggle - Your First Custom Hook

`useToggle` is the simplest and most common custom hook to write. It manages a boolean value that flips between `true` and `false`.

```js
// hooks/useToggle.js
import { useState } from "react";

function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);

  function toggle() {
    setValue((prev) => !prev);
  }

  return [value, toggle];
}

export default useToggle;
```

### Usage Examples

```jsx
// Modal
const [isOpen, toggleModal] = useToggle();

// Dark mode
const [isDark, toggleTheme] = useToggle();

// Show/hide password
const [showPassword, togglePassword] = useToggle();
```

### What You Learn From This Hook

- How to wrap `useState` in a function
- How to return multiple values from a hook (array destructuring)
- The naming convention `use...`
- How the same hook can power many different UI patterns

---

## 4. useFetch - Data Fetching Hook

Almost every React app fetches data. Without a custom hook, every component that fetches data has to manage the same three pieces of state: `data`, `loading`, and `error`.

### Before - Repeated in Every Component

```jsx
function Users() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch("/api/users")
      .then((res) => res.json())
      .then((data) => {
        setData(data);
        setLoading(false);
      })
      .catch((err) => {
        setError(err.message);
        setLoading(false);
      });
  }, []);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  return (
    <ul>
      {data.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

This pattern is repeated for every component that fetches - `Users`, `Posts`, `Products`, etc.

### After - useFetch Hook

```js
// hooks/useFetch.js
import { useState, useEffect } from "react";

function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;

    setLoading(true);
    setError(null);

    fetch(url)
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP error: ${res.status}`);
        return res.json();
      })
      .then((data) => {
        if (!cancelled) {
          setData(data);
          setLoading(false);
        }
      })
      .catch((err) => {
        if (!cancelled) {
          setError(err.message);
          setLoading(false);
        }
      });

    return () => {
      cancelled = true; // cleanup: prevent state update after unmount
    };
  }, [url]);

  return { data, loading, error };
}

export default useFetch;
```

### Usage

```jsx
import useFetch from "./hooks/useFetch";

function Users() {
  const { data, loading, error } = useFetch("/api/users");

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  return (
    <ul>
      {data.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}

function Posts() {
  const { data: posts, loading, error } = useFetch("/api/posts");

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  return (
    <ul>
      {posts.map((p) => (
        <li key={p.id}>{p.title}</li>
      ))}
    </ul>
  );
}
```

### Key Concepts in useFetch

| Concept                  | Explanation                                                                |
| ------------------------ | -------------------------------------------------------------------------- |
| `cancelled` flag         | Prevents setting state after a component unmounts (memory leak prevention) |
| `!res.ok` check          | `fetch` does NOT throw on 4xx/5xx - you must check manually                |
| Re-fetches on URL change | `url` in the dependency array means it re-runs when the URL changes        |
| Object return            | Returns `{ data, loading, error }` - named for clarity                     |

---

## 5. useLocalStorage - Persistent State

`useLocalStorage` works like `useState` but persists the value in `localStorage` so it survives page refreshes.

```js
// hooks/useLocalStorage.js
import { useState } from "react";

function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  function setValue(value) {
    try {
      const valueToStore =
        value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error("useLocalStorage error:", error);
    }
  }

  return [storedValue, setValue];
}

export default useLocalStorage;
```

### Usage

```jsx
import useLocalStorage from "./hooks/useLocalStorage";

function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage("theme", "light");

  return (
    <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
      Current: {theme}
    </button>
  );
}
```

After a page refresh, `theme` is still `"dark"` - it reads from localStorage on mount.

### Key Concepts in useLocalStorage

| Concept                         | Explanation                                               |
| ------------------------------- | --------------------------------------------------------- |
| Lazy initializer                | `useState(() => ...)` - reads from localStorage only once |
| `JSON.parse` / `JSON.stringify` | localStorage only stores strings                          |
| `try/catch`                     | localStorage can throw in private browsers or when full   |
| Function as value               | Supports functional update form like `setState`           |

---

## 6. useForm - Form State Management

Managing form state by hand requires a separate `useState` for every field. `useForm` centralizes this into one hook.

```js
// hooks/useForm.js
import { useState } from "react";

function useForm(initialValues) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});

  function handleChange(e) {
    const { name, value } = e.target;
    setValues((prev) => ({ ...prev, [name]: value }));
  }

  function reset() {
    setValues(initialValues);
    setErrors({});
  }

  return { values, errors, setErrors, handleChange, reset };
}

export default useForm;
```

### Usage

```jsx
import useForm from "./hooks/useForm";

function RegisterForm() {
  const { values, handleChange, reset } = useForm({
    name: "",
    email: "",
    password: "",
  });

  function handleSubmit(e) {
    e.preventDefault();
    console.log(values);
    reset();
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="name"
        value={values.name}
        onChange={handleChange}
        placeholder="Name"
      />
      <input
        name="email"
        value={values.email}
        onChange={handleChange}
        placeholder="Email"
      />
      <input
        name="password"
        value={values.password}
        onChange={handleChange}
        type="password"
      />
      <button type="submit">Register</button>
    </form>
  );
}
```

A single `handleChange` handles all fields because it uses `e.target.name` to know which field to update.

### Key Concepts in useForm

| Concept         | Explanation                                            |
| --------------- | ------------------------------------------------------ |
| `[name]: value` | Computed property name - updates any field dynamically |
| Single handler  | One `handleChange` for all inputs via `name` attribute |
| `reset()`       | Restores to `initialValues`                            |
| Extensible      | Can add validation, `touched`, `dirty` tracking on top |

---

## 7. Separation of UI and Logic

Custom hooks enforce the key architectural principle: **components should describe what to render, not how to manage data**.

### The Mental Model

| Concern        | Where it lives |
| -------------- | -------------- |
| State          | Custom hook    |
| Side effects   | Custom hook    |
| Event handlers | Custom hook    |
| JSX / Layout   | Component      |
| Styling        | Component      |

A well-written component using custom hooks often looks like this:

```jsx
function ProductPage() {
  const { data: product, loading, error } = useFetch(`/api/products/${id}`);
  const [inCart, toggleCart] = useToggle();
  const [quantity, setQuantity] = useLocalStorage("qty", 1);

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.price}</p>
      <button onClick={toggleCart}>{inCart ? "Remove" : "Add to Cart"}</button>
    </div>
  );
}
```

All logic is delegated. The component only renders.

---

## 8. Reuse Patterns

### Pattern 1 - Return an Array (like useState)

Use when the values are simple and the user will commonly rename them:

```js
return [value, setValue];

// Usage
const [isOpen, toggle] = useToggle();
const [theme, setTheme] = useLocalStorage("theme", "light");
```

### Pattern 2 - Return an Object (like useFetch)

Use when there are multiple named values - easier to read and pick from:

```js
return { data, loading, error };

// Usage
const { data, loading, error } = useFetch(url);
const { data: users } = useFetch("/api/users"); // rename with aliasing
```

### Pattern 3 - Accept Configuration Options

```js
function useFetch(url, options = {}) { ... }
function useLocalStorage(key, initialValue) { ... }
function useForm(initialValues, validationSchema) { ... }
```

### Pattern 4 - Compose Hooks Inside Hooks

Custom hooks can use other custom hooks:

```js
function useAuthUser() {
  const { data: user, loading } = useFetch("/api/me");
  const [token, setToken] = useLocalStorage("token", null);

  function logout() {
    setToken(null);
  }

  return { user, loading, token, logout };
}
```

---

## 9. Rules of Custom Hooks

Custom hooks follow the same rules as built-in hooks:

1. **Only call hooks at the top level** - not inside loops, conditions, or nested functions
2. **Only call hooks from React functions** - components or other custom hooks, never plain JS functions
3. **Name must start with `use`** - React and linters depend on this

```js
// Wrong - hook called inside condition
function useWrong(enabled) {
  if (enabled) {
    const [value, setValue] = useState(0); // breaks rules of hooks
  }
}

// Correct - condition is inside the hook, not around the hook call
function useCorrect(enabled) {
  const [value, setValue] = useState(0);
  if (!enabled) return null;
  return [value, setValue];
}
```

---

## 10. Where to Put Custom Hooks

Organize hooks in a dedicated folder:

```
src/
 ├── components/
 ├── pages/
 └── hooks/
      ├── useToggle.js
      ├── useFetch.js
      ├── useLocalStorage.js
      └── useForm.js
```

Each hook lives in its own file and is imported where needed.

---

## 11. When to Write a Custom Hook

Ask yourself:

1. **Am I copying stateful logic into multiple components?** → Extract it into a hook
2. **Is my component over 80-100 lines?** → Look for logic to extract
3. **Does this component do more than one thing?** → Split concerns with hooks
4. **Could this logic be useful elsewhere?** → Write a hook

### Signs You Need a Custom Hook

| Sign                                              | Hook to write     |
| ------------------------------------------------- | ----------------- |
| Many components fetch data with loading/error     | `useFetch`        |
| Multiple booleans toggled in various places       | `useToggle`       |
| Forms with similar field handling patterns        | `useForm`         |
| State that needs to survive a page refresh        | `useLocalStorage` |
| Window resize/scroll listeners in multiple places | `useWindowSize`   |
| Debouncing user input in search fields            | `useDebounce`     |

---

## 12. Common Mistakes to Avoid

| Mistake                                | Fix                                                    |
| -------------------------------------- | ------------------------------------------------------ |
| Not starting the name with `use`       | Always name hooks `useSomething`                       |
| Putting hooks inside conditionals      | Call all hooks unconditionally at the top level        |
| Making the hook do too much            | One hook = one responsibility                          |
| Not cleaning up effects (memory leaks) | Return a cleanup function from `useEffect`             |
| Returning too much from the hook       | Only return what the component actually needs          |
| Creating a hook for one-time-use logic | Only extract to a hook if it will be reused or complex |

---

## 13. Practice Project - Refactor With Custom Hooks

### Goal

Take any of the previous projects (Theme Manager, Shopping Cart, Auth UI, Notification System, Todo List) and refactor them to use custom hooks.

### Refactoring Checklist

For each component:

- [ ] Does it manage its own boolean toggle? → Extract `useToggle`
- [ ] Does it fetch data with loading/error? → Extract `useFetch`
- [ ] Does it read/write to localStorage? → Extract `useLocalStorage`
- [ ] Does it manage a form? → Extract `useForm`
- [ ] Is the component over 60 lines of logic? → Look for what to extract

### Example - Before and After

**Before (Shopping Cart with logic in component):**

```jsx
function ShoppingCart() {
  const [cart, setCart] = useState(() => {
    const saved = localStorage.getItem("cart");
    return saved ? JSON.parse(saved) : [];
  });

  function addItem(item) {
    const updated = [...cart, item];
    setCart(updated);
    localStorage.setItem("cart", JSON.stringify(updated));
  }

  function removeItem(id) {
    const updated = cart.filter((i) => i.id !== id);
    setCart(updated);
    localStorage.setItem("cart", JSON.stringify(updated));
  }

  const total = cart.reduce((sum, item) => sum + item.price, 0);

  return ( /* JSX */ );
}
```

**After (logic extracted into hooks):**

```jsx
// hooks/useCart.js
import useLocalStorage from "./useLocalStorage";

function useCart() {
  const [cart, setCart] = useLocalStorage("cart", []);

  function addItem(item) {
    setCart((prev) => [...prev, item]);
  }

  function removeItem(id) {
    setCart((prev) => prev.filter((i) => i.id !== id));
  }

  const total = cart.reduce((sum, item) => sum + item.price, 0);

  return { cart, addItem, removeItem, total };
}

export default useCart;
```

```jsx
// ShoppingCart.jsx - now just renders
import useCart from "./hooks/useCart";

function ShoppingCart() {
  const { cart, addItem, removeItem, total } = useCart();

  return ( /* JSX only */ );
}
```

The component shrank from ~30 lines of logic to pure rendering. The hook is reusable in a Navbar badge, a checkout page, or anywhere else that needs cart data.

---

## 14. Final Learning Outcome

By the end of this section, learners should be able to:

- Explain what a custom hook is and why it exists
- Identify when logic should be extracted into a hook
- Write `useToggle`, `useFetch`, `useLocalStorage`, and `useForm` from scratch
- Compose hooks inside other hooks
- Separate UI from logic across their components
- Organize hooks in a `hooks/` folder
- Refactor existing components to use custom hooks
- Think in terms of reusable, single-responsibility logic units

---

## Recommended Teaching Order

1. Show a component with repeated/complex logic (the problem)
2. Extract it into a custom hook step-by-step (the solution)
3. Teach `useToggle` - simplest hook
4. Teach `useFetch` - most practical hook
5. Teach `useLocalStorage` - shows hooks can wrap browser APIs
6. Teach `useForm` - shows hooks simplify complex patterns
7. Show composing hooks inside hooks
8. Refactor practice projects using custom hooks
