# Week 6 - Day 3 Assignment

## Title
Async/Await and Try/Catch

## Overview
Today you meet the final form of async JavaScript: `async`/`await`. Same promise machinery as yesterday, cleaner shape. Your assignment is to rewrite every Promise example from Day 2 using `async`/`await`, then wrap the fragile parts in `try`/`catch` blocks.

## Learning Objectives Assessed
- Declare `async` functions
- Use `await` to pause execution until a promise resolves
- Handle errors with `try`/`catch` blocks
- Understand that `async` functions always return a Promise
- Know when to use promises directly vs async/await

## Prerequisites
- Days 1-2 completed

## AI Usage Rules

**Ratio:** 65/35. **Habit:** MDN first. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Explaining the difference between `await` inside a for loop vs Promise.all.
- **NOT ALLOWED FOR:** Generating async rewrites.
- **AUDIT REQUIRED:** Yes, with MDN links.

## Tasks

### Task 1: Rewrite the countdown with async/await

**What to do:**
In `js-lab/week6-day3/main.js`, rewrite Day 2's Promise chain using `async`/`await`:

```javascript
function wait(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function countdown() {
  for (let i = 1; i <= 5; i++) {
    await wait(1000);
    console.log(i);
  }
}

countdown();
```

Compare to Day 2's chain in `day3-notes.md`. Which is clearer?

**Expected output:**
Numbers print once per second. Notes include the comparison.

### Task 2: try/catch with async

**What to do:**
Use the `divide` function from Day 2 (or rewrite it) and handle the error with try/catch:

```javascript
async function safeDivide(a, b) {
  try {
    const result = await divide(a, b);
    console.log("Result:", result);
  } catch (err) {
    console.error("Error:", err.message);
  }
}

safeDivide(10, 2);
safeDivide(10, 0);
```

**Expected output:**
Both cases work. Error is caught, not thrown.

### Task 3: Sequential vs parallel awaits

**What to do:**
Write two functions, one that awaits three things sequentially and one that awaits them in parallel:

```javascript
async function sequential() {
  const start = Date.now();
  const a = await wait(1000).then(() => "a");
  const b = await wait(1000).then(() => "b");
  const c = await wait(1000).then(() => "c");
  console.log(`Sequential: ${Date.now() - start}ms`, [a, b, c]);
}

async function parallel() {
  const start = Date.now();
  const [a, b, c] = await Promise.all([
    wait(1000).then(() => "a"),
    wait(1000).then(() => "b"),
    wait(1000).then(() => "c"),
  ]);
  console.log(`Parallel: ${Date.now() - start}ms`, [a, b, c]);
}

sequential();
parallel();
```

In `day3-notes.md`, record the times. The sequential version should take about 3 seconds; the parallel one about 1. Explain why.

**Expected output:**
Both functions run. Notes compare timings.

### Task 4: async function returns a Promise

**What to do:**
Prove that an `async` function always returns a Promise:

```javascript
async function getValue() {
  return 42;
}

const result = getValue();
console.log(result);              // Promise { 42 }
console.log(result instanceof Promise); // true
getValue().then((v) => console.log(v)); // 42
```

Add a comment explaining what this means in practice.

**Expected output:**
Three lines of output showing the Promise nature.

### Task 5: When to use what

**What to do:**
In `day3-notes.md`, write 4-6 sentences answering: "When should I use `.then` chains vs `async`/`await`?" Your own words. Think about readability, error handling, and loops.

**Expected output:**
Notes include the comparison.

## Stretch Goals (Optional - Extra Credit)

- Write a function that uses `for await...of` to iterate over an async iterable.
- Use `try`/`catch` with a nested await inside a loop and handle per-iteration errors without breaking the loop.
- Investigate top-level await (ES2022) and write one example in a module.

## Submission Requirements

- **What to submit:** Repo with Day 3 files, `day3-notes.md`, updated `AI_AUDIT.md`.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Countdown rewrite with async/await | 20 | Working loop using await. Notes compare to Day 2. |
| try/catch error handling | 20 | safeDivide handles both cases. No unhandled rejections. |
| Sequential vs parallel comparison | 25 | Both functions run. Timings recorded. Reason explained. |
| async returns Promise proof | 10 | Three lines demonstrating Promise behaviour. |
| "When to use what" notes | 15 | Real answer in student's words. |
| MDN-first audit | 5 | MDN links as needed. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Forgetting `await`.** Calling an async function without `await` returns a Promise, not the value. `const result = getUser()` is wrong; `const result = await getUser()` is right.
- **Awaiting in a loop when you meant parallel.** `for (const url of urls) { await fetch(url); }` waits sequentially. `await Promise.all(urls.map(fetch))` is the parallel version.
- **try/catch catching the wrong thing.** If you throw outside the `await`, the catch still catches it -- but if you call a function that is not async, the catch does nothing. Be intentional.

## Resources

- Day 3 reading: [Async/Await & Error Handling.md](./Async/Await%20&%20Error%20Handling.md)
- Week 6 AI boundaries: [../ai.md](../ai.md)
- MDN async function: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function
- MDN await: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await
