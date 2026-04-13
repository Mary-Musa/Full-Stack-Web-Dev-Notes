# Week 6 - Day 4 Assignment

## Title
Fetch API, Public APIs, and Pre-Weekend Data Dashboard Setup

## Overview
Day 4 is pre-weekend polish day. Today you use the Fetch API to call a real public API, parse its JSON response, handle loading and error states, and begin the weekend Data Dashboard project. By the end of today you should have a page that fetches live data, handles three states (loading, success, error), and displays the result.

## Learning Objectives Assessed
- Use `fetch()` to call a public API
- Parse JSON responses with `.json()`
- Handle all three UI states: loading, success, error
- Use `async`/`await` in real code
- Begin the weekend Data Dashboard project

## Prerequisites
- Days 1-3 completed

## AI Usage Rules

**Ratio:** 65/35. **Habit:** MDN first. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Explaining a CORS error after you read the browser console and searched for it.
- **NOT ALLOWED FOR:** Generating fetch code for you. Choosing which API to call.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: First fetch to a public API

**What to do:**
Pick any public API that does not require authentication. Suggestions:
- `https://api.exchangerate-api.com/v4/latest/KES` (exchange rates)
- `https://api.github.com/users/YOUR_USERNAME` (your GitHub profile)
- `https://dog.ceo/api/breeds/image/random` (random dog image)
- `https://api.open-meteo.com/v1/forecast?latitude=-1.28&longitude=36.82&current=temperature_2m` (Nairobi weather)

In `js-lab/week6-day4/main.js`, write:

```javascript
async function loadData() {
  const response = await fetch("YOUR_API_URL");
  const data = await response.json();
  console.log(data);
}

loadData();
```

Type it yourself. Run it. Inspect the shape of `data` in the console.

**Expected output:**
JSON data visible in the console.

### Task 2: Handle errors properly

**What to do:**
Wrap the fetch in a try/catch and also check the response status:

```javascript
async function loadData() {
  try {
    const response = await fetch("YOUR_API_URL");
    if (!response.ok) {
      throw new Error(`API returned ${response.status}`);
    }
    const data = await response.json();
    return data;
  } catch (err) {
    console.error("Failed to load data:", err.message);
    return null;
  }
}
```

Test by deliberately breaking the URL -- add a typo. Observe that your error handler catches it.

**Expected output:**
Both success and error paths work. No unhandled rejection.

### Task 3: Build a three-state UI

**What to do:**
Create `index.html` with:
- A `<div id="status">` for the status (loading/error/empty)
- A `<div id="content">` for the rendered data

In `main.js`, write code that:
1. Sets `#status` to "Loading..." immediately on page load
2. Fetches the data
3. On success: clears `#status`, renders the data in `#content`
4. On error: sets `#status` to "Error: ..." and leaves `#content` empty

```javascript
const statusEl = document.getElementById("status");
const contentEl = document.getElementById("content");

async function render() {
  statusEl.textContent = "Loading...";
  try {
    const data = await loadData();
    statusEl.textContent = "";
    contentEl.innerHTML = `<pre>${JSON.stringify(data, null, 2)}</pre>`;
  } catch (err) {
    statusEl.textContent = `Error: ${err.message}`;
  }
}

render();
```

**Expected output:**
Opening the page shows "Loading..." briefly, then the data appears.

### Task 4: Data Dashboard project kickoff

**What to do:**
Create `dashboard.html` and `dashboard.js`. This is the skeleton for your weekend project. Today build the minimum viable version:

1. Pick a theme (weather, crypto prices, GitHub stats, whatever interests you).
2. Fetch at least ONE data point from a public API.
3. Display it in an HTML element with reasonable styling.
4. Show loading and error states.

The weekend project will expand this into a multi-widget dashboard. Today only one widget is required.

**Expected output:**
Working one-widget dashboard. Screenshot `day4-dashboard.png` in `assets/`.

### Task 5: Pre-weekend checklist

**What to do:**
Create `CHECKLIST.md`:

```markdown
## Week 6 Day 4 Pre-Weekend Checklist

- [ ] Days 1-3 files all committed and working
- [ ] First fetch returns parsed JSON in the console
- [ ] Error handling catches a broken URL
- [ ] Three-state UI (loading/success/error) works on index.html
- [ ] dashboard.html shows at least one widget from a real API
- [ ] AI_AUDIT.md has MDN links for every entry this week
- [ ] Repo pushed and clean
```

Tick every box honestly.

**Expected output:**
Checklist committed.

## Stretch Goals (Optional - Extra Credit)

- Use the AbortController API to cancel a fetch if the user navigates away.
- Add a "refresh" button that re-fetches the data.
- Cache the last successful response in `localStorage` so the dashboard shows last-known-good data during an error.

## Submission Requirements

- **What to submit:** Repo with all Day 4 files, screenshot, `CHECKLIST.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| First fetch working | 15 | Real API data in the console. |
| Error handling | 15 | Broken URL triggers the error path. No crashes. |
| Three-state UI | 20 | Loading, success, error all visible in the browser. |
| Dashboard MVP | 25 | One widget fetches, shows loading, renders data, handles errors. |
| Pre-weekend checklist | 10 | All boxes honest. |
| MDN-first audit | 10 | MDN URLs in entries. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Not checking `response.ok`.** `fetch` resolves on any HTTP response -- including 404 and 500. You must check `response.ok` (or `response.status`) to distinguish real success from HTTP errors.
- **CORS surprises.** Some APIs block requests from browsers unless they have the right headers. If you get a CORS error, try a different API -- do not fight it.
- **Hardcoding secrets in JavaScript.** Never put an API key in client-side JS. Anyone can see it. If an API needs a key, use a public API instead or proxy through a server (later in the course).

## Resources

- Day 4 reading: [Fetch API & Working with JSON.md](./Fetch%20API%20&%20Working%20with%20JSON.md)
- Day 5: [Data Dashboard Project.md](../day5/Data%20Dashboard%20Project.md)
- Week 6 AI boundaries: [../ai.md](../ai.md)
- MDN fetch: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch
- Public APIs list: https://github.com/public-apis/public-apis
