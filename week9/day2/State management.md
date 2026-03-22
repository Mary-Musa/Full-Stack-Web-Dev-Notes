# State Management - Detailed Reading Notes (React)

## 1. What is State Management?

In React, **state** refers to data that changes over time and affects what the user sees on the screen.

Examples of state:

- Logged in user
- Theme (dark/light)
- Shopping cart items
- Form inputs
- Notifications
- API data

**State management** is the process of:

- Storing data
- Updating data
- Sharing data between components
- Keeping UI in sync with data

React state can exist at different levels:

1. Local state (inside one component)
2. Shared state (between a few components)
3. Global state (used across the whole app)

Beginners must **master local state first before global state**.

---

# 2. Local State First (Most Important Concept)

React documentation emphasizes that most state should be **kept as local as possible**.

Use:

- `useState`
- `useReducer` (for complex local state)

Example:

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>Increase</button>
    </div>
  );
}
```

Key ideas:

- State lives in a component
- When state changes → component re-renders
- State updates are asynchronous
- Never mutate state directly

Wrong:

```js
count = count + 1;
```

Correct:

```js
setCount(count + 1);
```

### Updating State Based on Previous State

When the new state depends on the previous state, always use the **functional update form**:

```js
// Wrong - may use stale value
setCount(count + 1);

// Correct - always uses latest value
setCount((prev) => prev + 1);
```

This matters most inside event handlers that fire rapidly or inside `useEffect`.

### State with Objects and Arrays

When state is an object or array, always return a **new copy** - never mutate.

```js
// Wrong
state.name = "Alice";

// Correct - spread to copy, then override
setState({ ...state, name: "Alice" });

// Arrays - add item
setItems([...items, newItem]);

// Arrays - remove item
setItems(items.filter((item) => item.id !== id));

// Arrays - update item
setItems(
  items.map((item) => (item.id === id ? { ...item, done: true } : item)),
);
```

---

# 3. useReducer - Complex Local State

`useReducer` is an alternative to `useState` for managing **complex state logic** inside a single component.

Think of it like a mini state machine:

- You dispatch an **action** (what happened)
- A **reducer function** decides how state changes

### Syntax

```js
const [state, dispatch] = useReducer(reducer, initialState);
```

### Example - Counter with useReducer

```jsx
import { useReducer } from "react";

const initialState = { count: 0 };

function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    case "decrement":
      return { count: state.count - 1 };
    case "reset":
      return initialState;
    default:
      throw new Error("Unknown action: " + action.type);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </div>
  );
}
```

### When to prefer useReducer over useState

| Situation                             | Use        |
| ------------------------------------- | ---------- |
| Simple value (toggle, number, string) | useState   |
| Multiple related state values         | useReducer |
| Next state depends on previous        | useReducer |
| Complex update logic                  | useReducer |
| You want to move logic out of the JSX | useReducer |

---

# 4. Lifting State Up

This is **the most important state management concept in React**.

When two components need the same data:

- Move the state to their **closest common parent**
- Pass data down via props

This is called **lifting state up**.

### Example Structure

```
App
 ├── ProductList
 └── Cart
```

Both need cart data → Move state to `App`.

```jsx
function App() {
  const [cart, setCart] = useState([]);

  return (
    <>
      <ProductList cart={cart} setCart={setCart} />
      <Cart cart={cart} />
    </>
  );
}
```

Concept:

- Parent owns state
- Children receive state via props
- Children update state via functions passed as props

This is **how most React apps actually work**.

### State Colocation (The Opposite Rule)

While lifting state up is important, the inverse rule is equally important: **colocate state as close as possible to where it's used**.

If only one component needs a piece of state, do not put it in a parent. Move it down (or keep it local).

> "Put state as close to where it's needed as possible." - React team

This keeps components self-contained and avoids unnecessary re-renders of parent/sibling components.

---

# 5. Prop Drilling

**Prop drilling** happens when you pass props through many components just to reach a deeply nested component.

Example:

```
App
 └── Layout
      └── Sidebar
           └── UserProfile
```

If `UserProfile` needs `user`, you might pass:

```
App → Layout → Sidebar → UserProfile
```

This becomes messy in large apps.

Problems with prop drilling:

- Intermediate components receive props they don't use
- Harder to refactor
- Code becomes cluttered

This is when we introduce **Context API**.

---

# 6. Context API (Global State for Small/Medium Apps)

React Context allows you to **share state globally without prop drilling**.

Used for:

- Auth user
- Theme
- Language
- Notifications
- App settings

### Steps to use Context

## Step 1 - Create Context

```jsx
import { createContext } from "react";

export const ThemeContext = createContext();
```

You can provide a default value:

```jsx
export const ThemeContext = createContext("light");
```

The default is used only when a component is rendered outside of the Provider.

## Step 2 - Create Provider

```jsx
import { useState } from "react";
import { ThemeContext } from "./ThemeContext";

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

## Step 3 - Wrap App

```jsx
<ThemeProvider>
  <App />
</ThemeProvider>
```

## Step 4 - Use Context

```jsx
import { useContext } from "react";
import { ThemeContext } from "./ThemeContext";

function Button() {
  const { theme, setTheme } = useContext(ThemeContext);

  return (
    <button onClick={() => setTheme("dark")}>Current theme: {theme}</button>
  );
}
```

### Best Practice - Custom Hook for Context

Instead of importing and calling `useContext` everywhere, wrap it in a custom hook:

```jsx
// ThemeContext.jsx
import { createContext, useContext, useState } from "react";

const ThemeContext = createContext();

export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// Custom hook - cleaner to import and use
export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) throw new Error("useTheme must be used inside ThemeProvider");
  return context;
}
```

Usage:

```jsx
import { useTheme } from "./ThemeContext";

function Button() {
  const { theme, setTheme } = useTheme();
  // ...
}
```

### Context Performance Warning

When the context value changes, **every component that consumes that context re-renders**, even if they only use part of the value.

To avoid this:

- Split contexts by concern (e.g. `AuthContext`, `ThemeContext` separately)
- Only put what's truly global in context
- Use Zustand when performance matters

### When to use Context

Use Context when:

- Many components need the same data
- Auth user
- Theme
- Notifications
- App settings

Do NOT use Context for:

- Forms
- Temporary UI state
- Component-specific state
- Frequently changing values (causes too many re-renders)

---

# 7. Zustand (Cleaner Global State for Medium Apps)

After Context, the next step should be **Zustand**, not Redux.

Zustand is:

- Simple
- Minimal boilerplate
- Easy global state
- Very popular in modern React apps
- No Provider needed

### Install

```
npm install zustand
```

### Create Store

```jsx
import { create } from "zustand";

export const useStore = create((set) => ({
  cart: [],
  addToCart: (item) => set((state) => ({ cart: [...state.cart, item] })),
  removeFromCart: (id) =>
    set((state) => ({ cart: state.cart.filter((i) => i.id !== id) })),
  clearCart: () => set({ cart: [] }),
}));
```

### Use Store

```jsx
import { useStore } from "./store";

function Product() {
  const addToCart = useStore((state) => state.addToCart);

  return (
    <button onClick={() => addToCart({ id: 1, name: "Phone" })}>
      Add to cart
    </button>
  );
}

function Cart() {
  const cart = useStore((state) => state.cart);

  return (
    <ul>
      {cart.map((item) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

No providers, no wrapping app - very simple.

### Selective Subscription (Performance Tip)

Only subscribe to what you need. This prevents unnecessary re-renders:

```jsx
// Only re-renders when cart changes, not when other state changes
const cart = useStore((state) => state.cart);

// Only re-renders when count changes
const count = useStore((state) => state.count);
```

### Zustand with Persistence (localStorage)

Zustand has built-in middleware to persist state across page refreshes:

```jsx
import { create } from "zustand";
import { persist } from "zustand/middleware";

export const useStore = create(
  persist(
    (set) => ({
      cart: [],
      addToCart: (item) => set((state) => ({ cart: [...state.cart, item] })),
    }),
    { name: "cart-storage" }, // key in localStorage
  ),
);
```

### When to use Zustand

Use Zustand when:

- App is medium size
- Many components share state
- Context becomes messy
- You need cleaner global state
- You want persistence without extra setup
- Performance is a concern (fine-grained subscriptions)

---

# 8. Redux Toolkit (Large/Enterprise Apps)

Redux is the oldest and most established state management solution. **Redux Toolkit (RTK)** is the modern way to write Redux - it removes most of the old boilerplate.

Use Redux when:

- Very large app with many developers
- Complex state with many actions
- Need powerful devtools (time travel debugging)
- Team already uses Redux

### Install

```
npm install @reduxjs/toolkit react-redux
```

### Create a Slice

A **slice** is a piece of Redux state with its own reducers and actions.

```jsx
// features/cartSlice.js
import { createSlice } from "@reduxjs/toolkit";

const cartSlice = createSlice({
  name: "cart",
  initialState: { items: [] },
  reducers: {
    addItem: (state, action) => {
      state.items.push(action.payload); // RTK uses Immer - direct mutation is OK here
    },
    removeItem: (state, action) => {
      state.items = state.items.filter((i) => i.id !== action.payload);
    },
  },
});

export const { addItem, removeItem } = cartSlice.actions;
export default cartSlice.reducer;
```

### Create the Store

```jsx
// store.js
import { configureStore } from "@reduxjs/toolkit";
import cartReducer from "./features/cartSlice";

export const store = configureStore({
  reducer: {
    cart: cartReducer,
  },
});
```

### Wrap App with Provider

```jsx
import { Provider } from "react-redux";
import { store } from "./store";

<Provider store={store}>
  <App />
</Provider>;
```

### Use in Components

```jsx
import { useSelector, useDispatch } from "react-redux";
import { addItem, removeItem } from "./features/cartSlice";

function Cart() {
  const items = useSelector((state) => state.cart.items);
  const dispatch = useDispatch();

  return (
    <div>
      {items.map((item) => (
        <div key={item.id}>
          {item.name}
          <button onClick={() => dispatch(removeItem(item.id))}>Remove</button>
        </div>
      ))}
    </div>
  );
}
```

### Redux vs Zustand - Quick Comparison

| Feature           | Zustand     | Redux Toolkit           |
| ----------------- | ----------- | ----------------------- |
| Boilerplate       | Very little | Some                    |
| Setup             | Simple      | More involved           |
| Provider required | No          | Yes                     |
| DevTools          | Basic       | Excellent (time travel) |
| Best for          | Medium apps | Large/enterprise apps   |
| Learning curve    | Low         | Medium                  |

---

# 9. When to Use What (Very Important Table)

| Situation                 | Use           |
| ------------------------- | ------------- |
| Form input                | useState      |
| Toggle modal              | useState      |
| Counter                   | useState      |
| Complex local state logic | useReducer    |
| Two components share data | Lift state up |
| Avoid prop drilling       | Context       |
| App-wide state            | Context       |
| Medium app global state   | Zustand       |
| Complex enterprise app    | Redux / RTK   |

---

# 10. State Management Mental Model (Very Important)

Teach learners this decision process:

**Ask: Who needs this state?**

1. Only one component → useState
2. Complex logic in one component → useReducer
3. Parent + child → Lift state up
4. Many components → Context
5. Large app or performance needs → Zustand
6. Very large enterprise → Redux

This mental model is extremely important.

### Common Mistakes to Avoid

| Mistake                              | Fix                                                   |
| ------------------------------------ | ----------------------------------------------------- |
| Putting everything in global state   | Default to local state, only lift when needed         |
| Mutating state directly              | Always create a new copy with spread/filter/map       |
| Using stale state in updates         | Use functional update form: `setState(prev => …)`     |
| One giant context for everything     | Split contexts by concern                             |
| Reaching for Redux before needing it | Start with useState → Context → Zustand → Redux       |
| Ignoring re-render performance       | Use selective Zustand subscriptions or split contexts |

---

# 11. Practice Projects for State Management

## Project 1 - Theme Manager

Features:

- Light/Dark mode
- Context API
- Toggle button
- Store theme globally
- Apply class/style based on theme

What you practice: `createContext`, `useContext`, Provider pattern, custom hook.

## Project 2 - Shopping Cart

Features:

- Add/remove items
- Cart total (computed from state)
- Item count badge
- Global cart state
- Zustand or Context

What you practice: Array state, derived state, lifting state, Zustand store.

## Project 3 - Auth UI State

Features:

- Login/logout button
- Store user info
- Protected navbar (show/hide links based on auth)
- Context API

What you practice: Boolean flags in state, conditional rendering, Context with user object.

## Project 4 - Global Notification System

Features:

- Show success/error messages
- Trigger notification from any component
- Auto-dismiss after 3 seconds
- Global state (Context or Zustand)

What you practice: Array of notifications, `useEffect` for auto-dismiss, global state patterns.

## Project 5 - Todo List with useReducer

Features:

- Add/remove/toggle todos
- Filter by status (all/active/done)
- All logic in a reducer function

What you practice: `useReducer`, action types, derived filtered state.

---

# 12. Final Learning Outcome

By the end of this section, learners should be able to:

- Use `useState` properly (including object/array state)
- Use functional update form for state that depends on previous
- Use `useReducer` for complex local state
- Lift state up
- Understand and recognize prop drilling
- Use Context API with a custom hook wrapper
- Use `useContext`
- Use Zustand for global state
- Know when local vs global state is needed
- Structure state properly in real apps
- Understand basic Redux Toolkit concepts

---

# Recommended Teaching Order (Very Important)

Teach in this exact order:

1. useState (simple values)
2. useState (objects and arrays)
3. Lifting state up
4. Prop drilling (problem)
5. Context API (solution)
6. useContext + custom hook
7. useReducer
8. Zustand
9. When to use global vs local state
10. Redux Toolkit (overview only for beginners)
