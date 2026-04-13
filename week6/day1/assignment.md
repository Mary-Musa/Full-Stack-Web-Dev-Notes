# Week 6 - Day 1 Assignment

## Title
Callbacks, The Event Loop, and Callback Hell

## Overview
Week 6 is the async week. Today you meet the callback pattern -- the oldest and most basic form of asynchronous code in JavaScript. Your assignment is to build a small chain of `setTimeout` callbacks, feel the pain of "callback hell" for yourself, and then understand why Promises (tomorrow) and async/await (Wednesday) exist.

## Learning Objectives Assessed
- Explain what "asynchronous" means in JavaScript
- Use `setTimeout` with a callback
- Chain callbacks manually and observe the hell
- Write an error-first callback in the Node style

## Prerequisites
- Week 5 completed
- Day 1 reading: [callback pattern.md](./callback%20pattern.md)

## AI Usage Rules

**Ratio this week:** 65% manual / 35% AI
**Habit:** Read the docs before the prompt. Always open MDN first. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Explaining an async behaviour after you read the MDN page and ran your own code.
- **NOT ALLOWED FOR:** Generating callback chains. Explaining callback hell before you felt it.
- **AUDIT REQUIRED:** Yes. Every AI entry must include the MDN link you read first.

## Tasks

### Task 1: Five callbacks with setTimeout

**What to do:**
In `js-lab/week6-day1/main.js`, write code that prints the numbers 1 through 5, one per second, using nested `setTimeout` calls:

```javascript
setTimeout(() => {
  console.log(1);
  setTimeout(() => {
    console.log(2);
    setTimeout(() => {
      console.log(3);
      setTimeout(() => {
        console.log(4);
        setTimeout(() => {
          console.log(5);
        }, 1000);
      }, 1000);
    }, 1000);
  }, 1000);
}, 1000);
```

This is called "callback hell" or "the pyramid of doom". Type the whole thing yourself. Feel how ugly it gets.

**Expected output:**
Numbers print once per second in the terminal.

### Task 2: Reflect on the pain

**What to do:**
In `day1-notes.md`, write 4-5 sentences answering:
- What would happen if you needed to print 20 numbers this way?
- What makes this hard to read?
- What would you want from a better tool?

No AI. Your own frustration in your own words.

**Expected output:**
`day1-notes.md` committed.

### Task 3: Error-first callbacks

**What to do:**
Write a function `divide(a, b, callback)` that uses the Node-style error-first callback convention:

```javascript
function divide(a, b, callback) {
  if (b === 0) {
    callback(new Error("Cannot divide by zero"), null);
    return;
  }
  callback(null, a / b);
}

divide(10, 2, (err, result) => {
  if (err) {
    console.error("Error:", err.message);
    return;
  }
  console.log("Result:", result);
});

divide(10, 0, (err, result) => {
  if (err) {
    console.error("Error:", err.message);
    return;
  }
  console.log("Result:", result);
});
```

**Expected output:**
Both success and error cases work.

### Task 4: Event loop mental model

**What to do:**
In `day1-notes.md`, add a section called "The Event Loop". Draw (in text or photograph) a simple diagram showing:
- The call stack
- The task queue
- How setTimeout's callback gets there

In 4-5 sentences, explain what "synchronous" and "asynchronous" mean in your own words. Why does `setTimeout(..., 0)` not run immediately?

**Expected output:**
Event loop section in `day1-notes.md`.

### Task 5: The 15-minute rule + MDN

**What to do:**
When you get stuck today, open MDN before opening AI. For every AI interaction this week, the audit must include:
- The MDN page URL you read first
- What you tried before asking
- Your prompt

Create or update `AI_AUDIT.md` with at least one entry following this structure, even if the "what I tried" is "nothing, because I did not get stuck today".

**Expected output:**
Audit entry with the MDN link in place.

## Stretch Goals (Optional - Extra Credit)

- Rewrite the nested setTimeout chain as a loop using an index and self-invocation. (Hard but doable.)
- Write a simulated "file read" callback using `setTimeout(..., 500)` to fake async IO. Accept a filename and invoke the callback with `(null, "fake contents")`.
- Research `setImmediate` (Node) vs `setTimeout(fn, 0)` and write one sentence on the difference.

## Submission Requirements

- **What to submit:** Repo with `js-lab/week6-day1/`, `day1-notes.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Callback hell chain | 20 | Five nested setTimeouts printing 1-5 once per second. |
| Reflection on the pain | 10 | 4-5 sentences in student's own words. |
| Error-first callback | 20 | `divide` function working for both success and error. |
| Event loop mental model | 20 | Diagram plus explanation of sync vs async. setTimeout(0) question answered. |
| MDN-first discipline in audit | 20 | MDN URL included for every AI interaction. |
| Clean commits | 10 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Asking AI before opening MDN.** This week, MDN is the first stop. If your audit is missing MDN links, you lose points regardless of the code quality.
- **Assuming `setTimeout(fn, 1000)` runs in exactly 1 second.** It runs at the next event loop tick after 1000ms has passed. Busy main thread = longer delay.
- **Thinking the callback runs synchronously.** It does not. `console.log("before")` then `setTimeout(..., 0)` then `console.log("after")` -- the "after" always prints before the setTimeout callback.

## Resources

- Day 1 reading: [callback pattern.md](./callback%20pattern.md)
- Week 6 AI boundaries: [../ai.md](../ai.md)
- MDN setTimeout: https://developer.mozilla.org/en-US/docs/Web/API/setTimeout
- MDN Concurrency model and Event Loop: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop
