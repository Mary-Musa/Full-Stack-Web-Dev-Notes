# Week 6 - Day 2 Assignment

## Title
Promises -- Resolved, Rejected, and Chained

## Overview
Yesterday you met callback hell. Today you meet the first cure: Promises. Your assignment is to rewrite yesterday's nested setTimeout chain using Promise chaining, handle errors with `.catch`, and use `Promise.all` to run multiple async operations in parallel.

## Learning Objectives Assessed
- Create a Promise manually with `new Promise((resolve, reject) => {})`
- Chain `.then(...).then(...).then(...)` as a cleaner alternative to nested callbacks
- Handle errors with `.catch`
- Use `Promise.all` and `Promise.race` on arrays of promises

## Prerequisites
- Day 1 completed

## AI Usage Rules

**Ratio:** 65/35. **Habit:** MDN first. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Explaining `Promise.all` vs `Promise.allSettled` after you tried both.
- **NOT ALLOWED FOR:** Generating promise chains for you.
- **AUDIT REQUIRED:** Yes, with MDN links.

## Tasks

### Task 1: Wrap setTimeout in a Promise

**What to do:**
Create a `wait(ms)` utility function that returns a Promise which resolves after `ms` milliseconds:

```javascript
function wait(ms) {
  return new Promise((resolve) => {
    setTimeout(resolve, ms);
  });
}

wait(1000).then(() => console.log("1 second passed"));
```

**Expected output:**
"1 second passed" prints after 1 second.

### Task 2: Rewrite the callback hell chain

**What to do:**
Use your `wait` function to rewrite Day 1's nested setTimeout chain as a clean `.then` chain:

```javascript
wait(1000)
  .then(() => { console.log(1); return wait(1000); })
  .then(() => { console.log(2); return wait(1000); })
  .then(() => { console.log(3); return wait(1000); })
  .then(() => { console.log(4); return wait(1000); })
  .then(() => { console.log(5); });
```

Compare to Day 1's version in `day2-notes.md`. Which is more readable?

**Expected output:**
Promise chain works, notes explain the improvement.

### Task 3: Error handling with .catch

**What to do:**
Write a Promise-returning function `divide(a, b)` that rejects on division by zero:

```javascript
function divide(a, b) {
  return new Promise((resolve, reject) => {
    if (b === 0) {
      reject(new Error("Cannot divide by zero"));
      return;
    }
    resolve(a / b);
  });
}

divide(10, 2)
  .then((result) => console.log("Result:", result))
  .catch((err) => console.error("Error:", err.message));

divide(10, 0)
  .then((result) => console.log("Result:", result))
  .catch((err) => console.error("Error:", err.message));
```

**Expected output:**
Both success and error cases handled. Error does not crash the program.

### Task 4: Promise.all and Promise.race

**What to do:**
Create three Promises that resolve at different times:

```javascript
const p1 = wait(1000).then(() => "one");
const p2 = wait(2000).then(() => "two");
const p3 = wait(3000).then(() => "three");

Promise.all([p1, p2, p3]).then((results) => {
  console.log("All done:", results);
});

Promise.race([p1, p2, p3]).then((first) => {
  console.log("First done:", first);
});
```

Observe: `all` waits for everything; `race` resolves as soon as the first one does.

**Expected output:**
Both behaviours visible in the console.

### Task 5: Promise.allSettled for mixed results

**What to do:**
Create an array that mixes resolving and rejecting promises. Use `Promise.allSettled` and inspect the output:

```javascript
const mixed = [
  divide(10, 2),
  divide(10, 0),
  divide(8, 4),
];

Promise.allSettled(mixed).then((results) => {
  results.forEach((r, i) => console.log(i, r.status, r.value ?? r.reason.message));
});
```

Explain in `day2-notes.md` why `allSettled` is preferable to `all` when you do not want one failure to nuke the whole batch.

**Expected output:**
`allSettled` working. Notes include the trade-off.

## Stretch Goals (Optional - Extra Credit)

- Use `Promise.any` instead of `Promise.race` and explain the difference.
- Chain errors through multiple `.then`s and catch them all with one `.catch` at the end.
- Write a retry helper: `retry(fn, times)` that retries a rejecting function up to N times before giving up.

## Submission Requirements

- **What to submit:** Repo with `js-lab/week6-day2/`, `day2-notes.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| wait() utility | 10 | Correctly wraps setTimeout in a Promise. |
| Promise chain rewrite | 20 | Cleanly rewrites Day 1's chain. Notes compare readability. |
| .catch error handling | 20 | Both success and error handled cleanly. |
| Promise.all and Promise.race | 20 | Both working with three test promises. |
| Promise.allSettled | 15 | Mixed results inspected. Notes on when to prefer it. |
| MDN-first audit | 10 | MDN links in audit entries. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Forgetting to `return` inside a `.then`.** If you do not return the next promise, the next `.then` runs immediately instead of waiting for it.
- **Swallowing errors.** A `.then` without a `.catch` means rejections are silent. Always end chains with `.catch`.
- **Creating promises when the function already returns one.** `new Promise((resolve) => { fetch(url).then(resolve); })` is an anti-pattern. Just return the fetch.

## Resources

- Day 2 reading: [Promises API.md](./Promises%20API.md)
- Week 6 AI boundaries: [../ai.md](../ai.md)
- MDN Promise: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
