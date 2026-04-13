# Week 8 - Day 2 Assignment

## Title
useEffect and Side Effects

## Overview
Today you meet `useEffect`, the hook that lets components do things outside of rendering -- fetch data, subscribe to events, set timers, read localStorage. Your assignment is a series of focused exercises that teach you when effects run, how to clean them up, and how to avoid the classic stale-closure bug.

## Learning Objectives Assessed
- Use `useEffect` with an empty dependency array (mount only)
- Use `useEffect` with a dependency (runs when that value changes)
- Return a cleanup function from an effect
- Read and write localStorage from an effect

## Prerequisites
- Day 1 completed

## AI Usage Rules

**Ratio:** 55/45. **Habit:** AI as pair partner. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Debugging an effect that runs too many times after you read the dependency array.
- **NOT ALLOWED FOR:** Writing your first effects.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Mount-only effect

**What to do:**
Add a `useEffect` to a component that runs once on mount. Log a message to the console:

```jsx
import { useEffect } from "react";

function App() {
  useEffect(() => {
    console.log("Mounted");
  }, []);

  return <h1>Hello</h1>;
}
```

Reload the page. You should see "Mounted" once in the console. (In strict mode dev, you may see it twice -- that is intentional and only in dev.)

**Expected output:**
"Mounted" log visible.

### Task 2: Effect with a dependency

**What to do:**
Add a counter from last week. Add an effect that runs whenever the count changes:

```jsx
const [count, setCount] = useState(0);

useEffect(() => {
  console.log("Count changed:", count);
}, [count]);
```

Click the increment button. See the log fire.

**Expected output:**
Log fires on every count change, not on every render.

### Task 3: Effect with cleanup

**What to do:**
Create a timer component that logs a message once per second, and cleans up the timer when unmounted:

```jsx
function Timer() {
  useEffect(() => {
    const id = setInterval(() => console.log("tick"), 1000);
    return () => {
      console.log("Cleanup");
      clearInterval(id);
    };
  }, []);
  return <p>Timer running. Check the console.</p>;
}
```

In `App.jsx`, toggle the Timer on and off with a button. Observe "Cleanup" firing when you hide the component.

**Expected output:**
"tick" every second while mounted. "Cleanup" fires when unmounted.

### Task 4: localStorage with useEffect

**What to do:**
Take your Week 7 Task Manager and persist tasks to localStorage. Two effects:

```jsx
// Load on mount
useEffect(() => {
  const saved = localStorage.getItem("tasks");
  if (saved) setTasks(JSON.parse(saved));
}, []);

// Save on every change
useEffect(() => {
  localStorage.setItem("tasks", JSON.stringify(tasks));
}, [tasks]);
```

Reload the page. Tasks persist.

**Expected output:**
Adding tasks, refreshing the page, tasks still there.

### Task 5: Classic stale closure debug

**What to do:**
Write this deliberately buggy code and explain the bug in `day2-notes.md`:

```jsx
function StaleCounter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // BUG: stale closure
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <p>{count}</p>;
}
```

Run it. Notice that it increments once and then stops. Why? Because the effect's closure captured `count = 0` from the first render and never re-runs (empty deps). Fix it two ways and note both in `day2-notes.md`:

1. Add `count` to the dependency array (but this restarts the interval every second -- bad)
2. Use a functional setter: `setCount((c) => c + 1)` (correct fix)

**Expected output:**
Both fixes in notes, explained in student's own words.

## Stretch Goals (Optional - Extra Credit)

- Use `AbortController` in an effect that fetches data -- cancel the fetch if the component unmounts before the response arrives.
- Write a custom hook `useLocalStorage(key, initial)` that wraps the pattern from Task 4.
- Subscribe to `window.resize` in an effect and clean up on unmount. Display the current window width in the UI.

## Submission Requirements

- **What to submit:** Repo with Day 2 files, `day2-notes.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Mount-only effect | 10 | One log on mount. |
| Effect with dependency | 15 | Log fires on count change, not on unrelated renders. |
| Effect with cleanup | 20 | Timer ticks, cleanup fires on unmount. |
| localStorage persistence | 20 | Tasks survive page reload. |
| Stale closure debug | 25 | Bug reproduced, both fixes explained in student's own words. |
| Clean commits | 5 | Conventional messages. |
| AI Audit | 5 | Pair conversation log if used. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Omitting the dependency array.** An effect without a second argument runs on every render. This is almost never what you want.
- **Lying about the dependencies.** If your effect uses `count` but you leave it out of the array, the ESLint plugin will warn and your closure will be stale.
- **Forgetting cleanup on subscriptions.** Memory leaks and "can't update state on unmounted" warnings come from missing cleanups.

## Resources

- Day 2 reading: [useEffect and Side Effects.md](./useEffect%20and%20Side%20Effects.md)
- Week 8 AI boundaries: [../ai.md](../ai.md)
- react.dev useEffect: https://react.dev/reference/react/useEffect
- react.dev You Might Not Need an Effect: https://react.dev/learn/you-might-not-need-an-effect
