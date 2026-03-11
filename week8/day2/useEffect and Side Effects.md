# useEffect and Side Effects in React

## Learning Goals

By the end of this topic, you should be able to:

- Explain what a "side effect" is and why React separates it from rendering
- Identify when `useEffect` is needed - and when it is not
- Write `useEffect` with the correct dependency array
- Fetch data from an API inside an effect and handle loading, error, and empty states
- Write cleanup logic for effects that need it
- Recognize and avoid common beginner mistakes like infinite loops and stale state

---

## 1. What "Side Effects" Are (In React Terms)

When React renders a component, it produces JSX - a description of what the UI should look like. This process is meant to be **pure**: given the same props and state, your component should return the same output every time. No surprises.

A **side effect** is anything that steps outside of this purely descriptive rendering to interact with the world beyond React. For example:

- Fetching data from an API
- Starting or stopping a timer (`setInterval`, `setTimeout`)
- Subscribing or unsubscribing to something (a WebSocket, a browser event listener)
- Reading or writing to `localStorage`
- Directly manipulating browser APIs (e.g., changing `document.title`)
- Syncing with a non-React library or third-party widget

React calls these **external systems** - things React does not control and cannot track during rendering.

> The React team's mental model: **Effects are for synchronizing your component with an external system.** If there is no external system involved, you probably do not need an Effect.

### Why rendering must stay pure

If your render function had side effects - like making an API call every time React re-rendered - you would get unpredictable behavior, duplicate requests, and hard-to-find bugs. Keeping rendering pure makes React fast and predictable. Side effects go in `useEffect`, which React runs *after* rendering is complete.

---

## 2. When `useEffect` Is Needed - And When It Is Not

This is the most important judgment call with `useEffect`. Getting it wrong is one of the most common sources of bugs for beginners.

### You DO need an Effect when:

You need to **synchronize** with something outside React:

| Situation | Example |
|---|---|
| Fetching data on load or when a value changes | Load user profile when `userId` changes |
| Connecting to a subscription | Open a WebSocket connection |
| Managing a timer | Start a countdown when a game begins |
| Adding a browser event listener | Listen for `window.resize` or `keydown` |
| Syncing with a non-React library | Initialize a chart library after the DOM renders |

### You do NOT need an Effect when:

You are just **deriving UI from existing state or props**. All of the following can be done directly in the render function - no `useEffect` needed:

| Situation | Wrong approach | Correct approach |
|---|---|---|
| Filtering a list | `useEffect` to compute filtered list into state | Compute directly in render: `items.filter(...)` |
| Computing a total | `useEffect` to sum cart amounts into state | Compute directly: `cart.reduce(...)` |
| Showing/hiding UI | `useEffect` to toggle a visibility state | Conditional rendering based on existing state |
| Validation messages | `useEffect` watching input to set error state | Derive from form state with a `validate()` function |

**Rule of thumb:** If a value can be calculated from existing state or props, calculate it during render. Do not store it in a separate state variable controlled by an Effect.

React has an entire guide on this called ["You Might Not Need an Effect"](https://react.dev/learn/you-might-not-need-an-effect) - it is worth reading once you are comfortable with the basics.

---

## 3. `useEffect` - How It Works

### The signature

```js
useEffect(setup, dependencies?)
```

- `setup` - a function containing your effect logic. It optionally returns a **cleanup function**.
- `dependencies` - an optional array of values. React watches this array to decide when to re-run the effect.

### The render cycle with Effects

React processes your component in this exact order:

```
1. Your component function runs (produces JSX)
        |
2. React commits JSX to the DOM (the page updates)
        |
3. React runs your useEffect (syncs with the outside world)
        |
4. [Later, when deps change or component unmounts]
        |
5. React runs your cleanup function
        |
6. React runs the effect again with new values
```

The key insight: **Effects always run after the component renders, never during.** This guarantees the DOM is ready and React has committed all changes before your effect does anything.

### A minimal example

```jsx
import { useState, useEffect } from "react";

export default function PageTitle() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    // Runs after every render where count changed
    document.title = `You clicked ${count} times`;
  }, [count]); // Re-run only when count changes

  return (
    <button onClick={() => setCount(count + 1)}>
      Click me ({count})
    </button>
  );
}
```

Every time `count` changes, React re-renders the component, then runs the effect, which updates the browser tab title.

---

## 4. The Dependency Array

The dependency array is the second argument to `useEffect`. It controls *when* the effect re-runs. This is where most beginners get confused, so it is worth understanding precisely.

```js
useEffect(() => { /* effect */ }, [dep1, dep2]);
```

After every render, React compares the new value of each dependency against its previous value (using `Object.is` comparison). If anything changed, React runs cleanup first, then runs the effect again.

### The three patterns

#### Pattern 1: No dependency array

```js
useEffect(() => {
  console.log("This runs after every single render");
});
```

Runs after **every render** - when the component first mounts, and every time it re-renders for any reason. This is rarely what you want and can easily cause infinite loops if the effect updates state.

---

#### Pattern 2: Empty dependency array `[]`

```js
useEffect(() => {
  console.log("This runs once, when the component first mounts");
}, []);
```

Runs **exactly once** after the component mounts. The cleanup function (if you return one) runs when the component unmounts. This is used for "do something once" patterns, like establishing a connection or fetching initial data.

---

#### Pattern 3: Specific dependencies

```js
useEffect(() => {
  console.log("This runs when userId or query changes");
}, [userId, query]);
```

Runs after the initial mount, and again any time `userId` or `query` changes. This is the most common and most powerful pattern - it lets you re-fetch data, re-subscribe, or re-sync whenever specific values update.

---

### The rule: include every value the effect uses

If your effect reads a variable from props or state, that variable must be in the dependency array. If it is missing, the effect will use a **stale** (outdated) copy of that value and produce bugs that are hard to diagnose.

Modern React projects use the `eslint-plugin-react-hooks` linter rule (`exhaustive-deps`) which warns you when you are missing a dependency. Learn to trust this warning - it is almost always right.

```js
// This is buggy - userId is used but not listed as a dependency
useEffect(() => {
  fetch(`/api/users/${userId}`); // stale userId on re-renders
}, []); // Wrong

// This is correct
useEffect(() => {
  fetch(`/api/users/${userId}`);
}, [userId]); // Correct: re-runs when userId changes
```

---

## 5. Fetching Data from APIs

Fetching data is the most common reason beginners reach for `useEffect`, and it is the right call - an API is an external system.

### The three pieces of state you always need

```jsx
const [data, setData] = useState(null);
const [isLoading, setIsLoading] = useState(true);
const [error, setError] = useState(null);
```

Every fetch effect follows the same structure:

1. Set loading to `true`
2. Make the fetch request
3. On success: store the data, set loading to `false`
4. On error: store the error, set loading to `false`

### Example: fetching a list of posts

```jsx
import { useState, useEffect } from "react";

export default function PostList() {
  const [posts, setPosts] = useState([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setIsLoading(true);
    setError(null);

    fetch("https://jsonplaceholder.typicode.com/posts")
      .then((res) => {
        if (!res.ok) {
          throw new Error("Failed to fetch posts.");
        }
        return res.json();
      })
      .then((data) => {
        setPosts(data);
        setIsLoading(false);
      })
      .catch((err) => {
        setError(err.message);
        setIsLoading(false);
      });
  }, []); // Empty array: fetch once on mount

  if (isLoading) return <p>Loading posts...</p>;
  if (error) return <p>Error: {error}</p>;
  if (posts.length === 0) return <p>No posts found.</p>;

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### Fetching based on a changing value

When the data to fetch depends on a prop or state (like a `userId` or a search `query`), put that value in the dependency array:

```jsx
useEffect(() => {
  setIsLoading(true);

  fetch(`https://jsonplaceholder.typicode.com/users/${userId}`)
    .then((res) => res.json())
    .then((data) => {
      setUser(data);
      setIsLoading(false);
    })
    .catch((err) => {
      setError(err.message);
      setIsLoading(false);
    });
}, [userId]); // Re-fetch every time userId changes
```

---

## 6. Loading, Error, and Empty States

Every real data-fetching component needs to handle four possible UI states. Skipping any of them creates a poor user experience.

| State | What to render | When it applies |
|---|---|---|
| Loading | "Loading..." or a spinner | Waiting for the fetch to complete |
| Error | An error message with some context | The fetch failed (network error, bad status) |
| Empty | "No results found" or similar | Fetch succeeded but returned no data |
| Success | The actual data - a list, cards, etc. | Fetch succeeded and data exists |

This is a checklist to go through every time you fetch data. The pattern of checking each state in sequence before rendering the main content looks like this:

```jsx
if (isLoading) return <p>Loading...</p>;
if (error) return <p>Error: {error}</p>;
if (data.length === 0) return <p>No results found.</p>;

// If we reach here, we have data
return <ul>{data.map(...)}</ul>;
```

This "early return" style keeps the main render clean and easy to read.

---

## 7. Cleanup - What It Is and Why It Exists

Some effects set up a resource that needs to be torn down when the component is done with it. If you leave these running when the component unmounts, you get memory leaks, duplicate listeners, and state updates on unmounted components (which React warns you about).

To handle this, `useEffect` lets you **return a cleanup function**:

```jsx
useEffect(() => {
  // --- Setup ---
  // Start something (a timer, a subscription, an event listener)

  return () => {
    // --- Cleanup ---
    // Stop or undo what you started
  };
}, [deps]);
```

Cleanup runs in two situations:
1. **Before the effect re-runs** - when a dependency changed and React needs to reset
2. **When the component unmounts** - when it is removed from the page entirely

### Examples of effects that need cleanup

**Interval timer:**

```jsx
useEffect(() => {
  const intervalId = setInterval(() => {
    setSeconds((prev) => prev + 1);
  }, 1000);

  return () => {
    clearInterval(intervalId); // Stop the timer when done
  };
}, []);
```

**Browser event listener:**

```jsx
useEffect(() => {
  function handleKeyDown(e) {
    if (e.key === "Escape") setMenuOpen(false);
  }

  window.addEventListener("keydown", handleKeyDown);

  return () => {
    window.removeEventListener("keydown", handleKeyDown); // Remove listener when done
  };
}, []); // Attach once on mount, remove on unmount
```

**Cancelling a fetch with AbortController:**

When a user changes an input quickly, multiple fetch requests can be in flight at once. The responses can arrive out of order, so an old request might overwrite the result of a newer one - this is called a **race condition**.

`AbortController` solves this by cancelling the previous request in the cleanup:

```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch(`https://api.example.com/search?q=${query}`, {
    signal: controller.signal, // Pass the abort signal to fetch
  })
    .then((res) => res.json())
    .then((data) => setResults(data))
    .catch((err) => {
      if (err.name !== "AbortError") {
        // Only handle real errors - ignore intentional aborts
        setError(err.message);
      }
    });

  return () => {
    controller.abort(); // Cancel the request if query changes or component unmounts
  };
}, [query]); // Re-run (and cancel previous) whenever query changes
```

The rule: **if an effect starts something, the cleanup must stop it.**

---

## 8. Common Beginner Mistakes

### Mistake 1: Infinite loops

An infinite loop happens when your effect updates a state variable that is also listed as a dependency. React sees the state change, re-runs the effect, which changes the state again - forever.

```jsx
// This loops forever
useEffect(() => {
  setCount(count + 1); // Updates count...
}, [count]);            // ...which triggers the effect again
```

**Fix:** Think about whether the effect really needs to respond to that value changing. Often the solution is to derive the value during render instead, or to restructure the logic.

---

### Mistake 2: Using `useEffect` for event handling

Effects are not the right tool for responding to user interactions like button clicks or form submissions. Those are **events** - they should be handled in event handlers (`onClick`, `onSubmit`), not effects.

```jsx
// Wrong - using an effect to react to a button click
useEffect(() => {
  if (submitted) {
    sendFormData(form);
    setSubmitted(false);
  }
}, [submitted]);

// Correct - handle it directly in the event handler
function handleSubmit(e) {
  e.preventDefault();
  sendFormData(form);
}
```

Effects synchronize with external systems. Event handlers respond to user actions. Keep these separate.

---

### Mistake 3: "Why does my effect run twice in development?"

If you are using React 18 with **Strict Mode** (which Create React App and Next.js enable by default), you may see your effects running twice in development. This is intentional - React deliberately mounts, unmounts, and remounts components to help you catch effects that do not clean up properly.

This behavior only happens in development. In production, effects run once as expected.

**The lesson:** write your effects so they can safely run more than once. Always write the cleanup function. If your effect causes a problem when run twice, that is a signal your cleanup is incomplete.

---

### Mistake 4: Missing dependencies (stale closures)

If your effect uses a value from props or state but that value is not in the dependency array, the effect will capture an old, stale version of that value and never see updates.

```jsx
const [name, setName] = useState("Alice");

// This effect will always print "Alice" - it captured the initial value
useEffect(() => {
  console.log("Name is:", name); // Stale - never updates
}, []); // Wrong: name is missing from deps

// This is correct
useEffect(() => {
  console.log("Name is:", name);
}, [name]); // Correct: re-runs whenever name changes
```

---

## 9. Practice Projects

Build these projects in order. Each one adds a new layer of complexity on top of the last.

---

### A) Random User Card

**Goal:** Fetch a random user from [randomuser.me](https://randomuser.me/api/) and display their photo, name, and email.

**What it teaches:**
- Basic `useEffect` with `[]` (fetch once on mount)
- Loading and error states
- Rendering fetched data

**Requirements:**
- Show "Loading..." while fetching
- Show the user's photo, name, email, and location
- Add a "Get another user" button that fetches a new random user on click
- Stretch: store the last 3 users you fetched and let the user scroll through them

---

### B) Weather App

**Goal:** Let the user type a city name and see the current weather for that city (use [OpenWeatherMap](https://openweathermap.org/api) or [wttr.in](https://wttr.in/) for a free option).

**What it teaches:**
- Controlled input + form submission triggering a fetch
- Dependency-based effect (fetch when city changes)
- Loading/error/empty state for each search

**Requirements:**
- Form with a city input field and a search button
- Show loading while fetching
- Show an error if the city is not found
- Show current temperature, weather condition, and location name
- Stretch: automatically search when the user stops typing (debounced effect)

---

### C) Country Explorer

**Goal:** Fetch the full list of countries from the [REST Countries API](https://restcountries.com/) and display them in a searchable, filterable grid.

**What it teaches:**
- Fetching a large dataset once on mount
- Filtering data in render (without `useEffect`) based on a search input
- Handling an empty filtered result gracefully

**Requirements:**
- Fetch all countries on mount, show loading while doing so
- Display each country as a card (flag, name, population, region)
- Search input filters the list in real time (no additional fetches needed - filter the existing data in render)
- Region filter dropdown narrows the list further
- Show "No countries matched your search" if the filtered list is empty
- Stretch: clicking a country card shows more details (capital, currencies, languages)

---

### D) Blog Posts Viewer

**Goal:** Fetch a list of blog posts and show the full content of whichever post the user selects.

**What it teaches:**
- Two dependent fetches: posts list on mount, then comments when a post is selected
- `useEffect` with a dependency (`selectedPostId`)
- Conditional rendering of a detail panel

**Requirements:**
- Fetch all posts from [JSONPlaceholder](https://jsonplaceholder.typicode.com/posts) on mount
- Render a list of post titles on the left
- Clicking a title shows the post body on the right
- When a post is selected, also fetch and show its comments (another `useEffect` dependent on `selectedPostId`)
- Show loading and error states for both fetches
- Stretch: paginate the post list (show 10 at a time)

---

## 10. End-of-Topic Summary

| Concept | Key takeaway |
|---|---|
| Side effects | Code that reaches outside React's rendering: fetches, timers, listeners |
| Pure rendering | Render should be a pure function - effects do the side-effectful work after render |
| When to use `useEffect` | Only when synchronizing with an external system |
| When NOT to use it | When you can compute the result directly from state/props during render |
| Dependency array | Controls when the effect re-runs; must include every reactive value the effect reads |
| `[]` (empty array) | Run once on mount, cleanup on unmount |
| `[dep1, dep2]` | Run on mount and whenever deps change |
| No array | Run after every render (rarely intentional) |
| Cleanup function | Return a function from `useEffect` to stop timers, remove listeners, cancel fetches |
| Race conditions | Use `AbortController` to cancel stale fetch requests when dependencies change |
| Infinite loops | Happen when an effect updates a state that is one of its own dependencies |
| Effects running twice | Strict Mode intentionally mounts/unmounts in development - always write cleanup |

---

## Official React Documentation References

- [useEffect reference](https://react.dev/reference/react/useEffect)
- [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects) - includes data fetching and cleanup patterns
- [Lifecycle of Reactive Effects](https://react.dev/learn/lifecycle-of-reactive-effects) - how dependencies drive re-syncing
- [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) - prevents overuse
- [Removing Effect Dependencies](https://react.dev/learn/removing-effect-dependencies) - how to structure effects to minimize dependencies
