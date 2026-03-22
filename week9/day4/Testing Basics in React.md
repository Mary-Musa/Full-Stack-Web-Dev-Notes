# Testing Basics in React - Reading Notes

---

## 1. Why Testing Matters

Every application you build will eventually break - either from a bug you introduce, a feature you add, or a dependency that changes. Testing is how you **catch those breaks before your users do**.

Testing ensures that your application:

- Works correctly across different scenarios
- Does not break when you add new features (regression prevention)
- Handles user interactions and edge cases properly
- Is reliable, maintainable, and safe to change

In real-world software development, **testing is not optional** - it's a standard part of professional development. Untested code is considered unfinished code in most engineering teams.

### What Testing Gives You as a Developer

| Without Tests                | With Tests                          |
| ---------------------------- | ----------------------------------- |
| Fear of changing code        | Confidence to refactor              |
| Bugs discovered by users     | Bugs caught locally before shipping |
| Manual click-through testing | Automated regression checks         |
| Slow, risky deployments      | Fast, reliable CI/CD pipelines      |

### What We Test in React Applications

1. **Components render correctly** - does the UI appear as expected?
2. **User interactions work** - do buttons, clicks, and inputs behave correctly?
3. **Forms submit correctly** - does data get validated and submitted?
4. **Important user flows work end-to-end** - can a user complete the whole journey?

---

## 2. The Testing Pyramid

The **Testing Pyramid** is a mental model that helps you understand how many tests of each type to write.

```
        /\
       /  \
      / E2E \         ← Few, slow, expensive
     /--------\
    /  Integration \  ← Some
   /--------------\
  /   Unit Tests   \  ← Many, fast, cheap
 /------------------\
```

- **Unit Tests** - test small, isolated pieces of logic. Fast and cheap.
- **Integration/Component Tests** - test how components work together. Medium cost.
- **End-to-End (E2E) Tests** - test the full app in a real browser. Slow and expensive.

> **Rule of thumb:** Write many unit tests, some component tests, and a few critical E2E tests.

---

## 3. Types of Testing in React

There are **three main types of testing** you need to know:

| Type                 | What It Tests                         | When to Use                         | Tool                  |
| -------------------- | ------------------------------------- | ----------------------------------- | --------------------- |
| Unit Testing         | Pure functions, helpers, utilities    | Logic and calculations              | Vitest                |
| Component/UI Testing | React components and interactions     | UI behavior and rendering           | React Testing Library |
| End-to-End Testing   | Complete user flows in a real browser | Critical paths like login, checkout | Playwright            |

### Recommended Learning Order

1. **Unit Testing** with Vitest - test logic
2. **Component Testing** with React Testing Library - test UI
3. **End-to-End Testing** with Playwright - test full flows

---

## 4. Unit & Component Testing with Vitest

### What is Vitest?

Vitest is a modern, fast testing framework built specifically for Vite-based projects. It is the natural testing tool for React apps created with `npm create vite`.

- Similar to Jest in API (`describe`, `it`, `expect`)
- Much faster because it runs inside Vite (no extra build step)
- Works natively with TypeScript and ES Modules

### Installation

```bash
npm install -D vitest
```

Add a test script to `package.json`:

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run"
  }
}
```

> `vitest` runs in watch mode (re-runs on file save).
> `vitest run` runs once and exits - useful in CI.

### Basic Test Structure

```js
import { describe, it, expect } from "vitest";

// The function you want to test
function add(a, b) {
  return a + b;
}

// Test suite - groups related tests
describe("add function", () => {
  // Individual test case
  it("adds two positive numbers", () => {
    expect(add(2, 3)).toBe(5);
  });

  it("adds a positive and negative number", () => {
    expect(add(10, -3)).toBe(7);
  });

  it("adds two zeros", () => {
    expect(add(0, 0)).toBe(0);
  });
});
```

**Key terms:**

- `describe()` - groups related tests into a suite
- `it()` or `test()` - defines a single test case
- `expect()` - the assertion that checks a value
- `toBe()` - a matcher that checks strict equality

### Common Matchers

| Matcher              | What it checks                 |
| -------------------- | ------------------------------ |
| `toBe(value)`        | Strict equality (`===`)        |
| `toEqual(value)`     | Deep equality (objects/arrays) |
| `toBeTruthy()`       | Value is truthy                |
| `toBeFalsy()`        | Value is falsy                 |
| `toContain(item)`    | Array or string contains item  |
| `toHaveLength(n)`    | Array/string has length n      |
| `toThrow()`          | Function throws an error       |
| `toBeGreaterThan(n)` | Number is greater than n       |

### Real-World Unit Test Example

```js
// utils/cartUtils.js
export function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

export function applyDiscount(total, discountPercent) {
  return total - (total * discountPercent) / 100;
}
```

```js
// utils/cartUtils.test.js
import { describe, it, expect } from "vitest";
import { calculateTotal, applyDiscount } from "./cartUtils";

describe("calculateTotal", () => {
  it("returns 0 for an empty cart", () => {
    expect(calculateTotal([])).toBe(0);
  });

  it("calculates total for single item", () => {
    const items = [{ price: 50, quantity: 2 }];
    expect(calculateTotal(items)).toBe(100);
  });

  it("calculates total for multiple items", () => {
    const items = [
      { price: 30, quantity: 1 },
      { price: 20, quantity: 3 },
    ];
    expect(calculateTotal(items)).toBe(90);
  });
});

describe("applyDiscount", () => {
  it("applies 10% discount correctly", () => {
    expect(applyDiscount(100, 10)).toBe(90);
  });

  it("returns full price for 0% discount", () => {
    expect(applyDiscount(200, 0)).toBe(200);
  });
});
```

### What to Test with Vitest

- Utility/helper functions
- Data formatting functions (dates, prices, strings)
- Calculation logic (totals, discounts, scores)
- Form validation functions
- API response transformation functions
- Business logic

---

## 5. UI/Component Testing with React Testing Library

### What is React Testing Library?

React Testing Library (RTL) is a library that lets you test React components the way a **real user** would interact with them.

The core philosophy is:

> _"Test the way the user uses the app, not the way the code is written."_

This means you should NOT test:

- Internal state
- Private methods
- Implementation details

You SHOULD test:

- What appears on screen
- What happens when you click
- What happens when you type
- What the user actually sees and does

### Installation

```bash
npm install -D @testing-library/react @testing-library/user-event @testing-library/jest-dom
```

Also configure Vitest to use jsdom (browser environment simulation):

```js
// vite.config.js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: "./src/setupTests.js",
  },
});
```

```js
// src/setupTests.js
import "@testing-library/jest-dom";
```

### Core API: `render` and `screen`

```js
import { render, screen } from "@testing-library/react";
import Button from "./Button";

test("renders the button with correct text", () => {
  render(<Button label="Submit" />);

  const button = screen.getByText("Submit");
  expect(button).toBeInTheDocument();
});
```

- `render()` - mounts the component into a virtual DOM
- `screen` - lets you query what's currently on screen
- `getByText()` - finds an element by its visible text

### Querying Elements

RTL provides several methods to find elements on screen:

| Method                                 | When to Use                                       |
| -------------------------------------- | ------------------------------------------------- |
| `getByText("...")`                     | Find by visible text                              |
| `getByRole("button", { name: "..." })` | Find by ARIA role (preferred)                     |
| `getByLabelText("...")`                | Find form inputs by label                         |
| `getByPlaceholderText("...")`          | Find inputs by placeholder                        |
| `getByTestId("...")`                   | Find by `data-testid` attribute (last resort)     |
| `queryByText("...")`                   | Like getBy but returns `null` instead of throwing |
| `findByText("...")`                    | Async - waits for element to appear               |

> **Best practice:** Prefer `getByRole` - it tests accessibility too.

### Common `jest-dom` Matchers

| Matcher                    | What it checks                 |
| -------------------------- | ------------------------------ |
| `toBeInTheDocument()`      | Element exists in the DOM      |
| `toBeVisible()`            | Element is visible to the user |
| `toBeDisabled()`           | Button/input is disabled       |
| `toHaveValue("...")`       | Input has the given value      |
| `toHaveTextContent("...")` | Element contains the text      |
| `toHaveClass("...")`       | Element has the CSS class      |

---

## 6. Testing User Interactions

### Click Events

```js
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import Counter from "./Counter";

test("increments counter when button is clicked", async () => {
  render(<Counter />);

  // Initial state
  expect(screen.getByText("Count: 0")).toBeInTheDocument();

  // Simulate click
  const button = screen.getByRole("button", { name: "Increment" });
  await userEvent.click(button);

  // Updated state
  expect(screen.getByText("Count: 1")).toBeInTheDocument();
});
```

> Always use `await` with `userEvent` - it fires real browser events asynchronously.

### Toggle / Conditional Rendering

```js
test("shows and hides content when toggle is clicked", async () => {
  render(<Accordion title="FAQ" content="Here is the answer." />);

  // Content hidden by default
  expect(screen.queryByText("Here is the answer.")).not.toBeInTheDocument();

  // Click to expand
  await userEvent.click(screen.getByText("FAQ"));

  // Content visible
  expect(screen.getByText("Here is the answer.")).toBeInTheDocument();
});
```

---

## 7. Testing Forms

Forms are one of the most critical things to test in any application. A broken form means lost users and lost data.

### What to Test in Forms

- Input values are captured correctly
- Validation errors appear for invalid input
- Successful submission shows the correct response
- Submit button is disabled when form is invalid (if applicable)

### Complete Form Test Example

```js
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import LoginForm from "./LoginForm";

describe("LoginForm", () => {
  test("shows error when email is empty and form is submitted", async () => {
    render(<LoginForm />);

    await userEvent.click(screen.getByRole("button", { name: "Login" }));

    expect(screen.getByText("Email is required")).toBeInTheDocument();
  });

  test("shows error for invalid email format", async () => {
    render(<LoginForm />);

    await userEvent.type(screen.getByLabelText("Email"), "notanemail");
    await userEvent.click(screen.getByRole("button", { name: "Login" }));

    expect(screen.getByText("Enter a valid email")).toBeInTheDocument();
  });

  test("submits successfully with valid credentials", async () => {
    render(<LoginForm />);

    await userEvent.type(screen.getByLabelText("Email"), "user@example.com");
    await userEvent.type(screen.getByLabelText("Password"), "securepass");
    await userEvent.click(screen.getByRole("button", { name: "Login" }));

    expect(screen.getByText("Login successful!")).toBeInTheDocument();
  });
});
```

---

## 8. Testing API Data & Async Components

When your component fetches data, you need to test the **loading state**, the **success state**, and the **error state**.

### Mocking `fetch` with Vitest

```js
import { render, screen } from "@testing-library/react";
import { vi } from "vitest";
import PostList from "./PostList";

// Mock the global fetch
beforeEach(() => {
  global.fetch = vi.fn();
});

test("displays posts after fetching", async () => {
  // Mock a successful API response
  fetch.mockResolvedValueOnce({
    ok: true,
    json: async () => [
      { id: 1, title: "First Post" },
      { id: 2, title: "Second Post" },
    ],
  });

  render(<PostList />);

  // Loading state
  expect(screen.getByText("Loading...")).toBeInTheDocument();

  // Wait for data to appear
  expect(await screen.findByText("First Post")).toBeInTheDocument();
  expect(screen.getByText("Second Post")).toBeInTheDocument();
});

test("shows error message when fetch fails", async () => {
  fetch.mockRejectedValueOnce(new Error("Network error"));

  render(<PostList />);

  expect(await screen.findByText("Failed to load posts.")).toBeInTheDocument();
});
```

> `findByText()` is the async version of `getByText`. Use it when waiting for data to load.

---

## 9. End-to-End Testing with Playwright

### What is Playwright?

Playwright is a tool that controls a real browser (Chrome, Firefox, Safari) and simulates a complete user session. It is used for **End-to-End (E2E) tests**.

Unlike unit and component tests that run in a virtual DOM, Playwright:

- Opens a real browser window
- Navigates to real URLs
- Clicks, types, and scrolls like a real user
- Can take screenshots and record videos

### Installation

```bash
npm init playwright@latest
```

This sets up a `playwright.config.js` file and a `tests/` folder.

### Basic Playwright Test

```js
// tests/login.spec.js
import { test, expect } from "@playwright/test";

test("user can log in successfully", async ({ page }) => {
  // Navigate to the login page
  await page.goto("http://localhost:5173/login");

  // Fill in the form
  await page.fill('input[name="email"]', "user@example.com");
  await page.fill('input[name="password"]', "password123");

  // Submit
  await page.click('button[type="submit"]');

  // Assert we're redirected to the dashboard
  await expect(page.locator("h1")).toHaveText("Welcome back!");
  await expect(page).toHaveURL("/dashboard");
});
```

### Running Playwright Tests

```bash
# Run all tests
npx playwright test

# Run with browser visible (headed mode)
npx playwright test --headed

# Show test report
npx playwright show-report
```

### Full E2E Flow Example - Shopping Cart

```js
test("user can add item to cart and checkout", async ({ page }) => {
  await page.goto("http://localhost:5173/shop");

  // Add item to cart
  await page.click('[data-testid="add-to-cart-1"]');

  // Go to cart
  await page.click('[aria-label="Cart"]');
  await expect(page.locator(".cart-item")).toHaveCount(1);

  // Proceed to checkout
  await page.click("text=Checkout");
  await expect(page).toHaveURL("/checkout");

  // Fill payment info
  await page.fill("#card-number", "4242424242424242");
  await page.click("text=Pay Now");

  // Confirm success
  await expect(page.locator("text=Order confirmed")).toBeVisible();
});
```

---

## 10. What to Test (and What Not to Test)

### Focus on User Behavior - Not Implementation

| Test This                         | Not This                                      |
| --------------------------------- | --------------------------------------------- |
| Button click updates UI           | Internal state variable value                 |
| Form shows error on invalid input | Exact CSS class names                         |
| API data appears after loading    | How `useEffect` is structured internally      |
| User can navigate to a page       | Whether a component calls a specific function |

### Critical Things to Always Test

- Components render without crashing
- Buttons and interactive elements work
- Forms validate and submit correctly
- API data loads and displays
- Loading and error states appear correctly
- Authentication flows (login, logout, protected routes)
- Navigation between pages
- Cart or data-management flows

---

## 11. Testing Strategy for Projects

Use this strategy to decide which tool to use for each test:

| Feature                          | Test Type      | Tool                  |
| -------------------------------- | -------------- | --------------------- |
| Date formatter, price calculator | Unit Test      | Vitest                |
| Form validation logic            | Unit Test      | Vitest                |
| Component renders correctly      | Component Test | React Testing Library |
| Button click changes UI          | Component Test | React Testing Library |
| Form submission with validation  | Component Test | React Testing Library |
| API data displays after loading  | Component Test | React Testing Library |
| Full login flow                  | E2E Test       | Playwright            |
| Full checkout flow               | E2E Test       | Playwright            |
| Page navigation                  | E2E Test       | Playwright            |

> **Start with component tests. Add unit tests for complex logic. Add E2E tests for the most critical user paths.**

---

## 12. Setting Up a Full Testing Stack (Vite + React)

### Full Installation

```bash
# Testing framework and component testing
npm install -D vitest @testing-library/react @testing-library/user-event @testing-library/jest-dom

# E2E testing
npm init playwright@latest
```

### `vite.config.js`

```js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: "jsdom",
    setupFiles: "./src/setupTests.js",
  },
});
```

### `src/setupTests.js`

```js
import "@testing-library/jest-dom";
```

### `package.json` Scripts

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:ui": "vitest --ui",
    "test:e2e": "playwright test"
  }
}
```

---

## 13. File Naming Conventions

| File Type      | Convention               | Example              |
| -------------- | ------------------------ | -------------------- |
| Unit test      | `filename.test.js`       | `cartUtils.test.js`  |
| Component test | `ComponentName.test.jsx` | `LoginForm.test.jsx` |
| E2E test       | `feature.spec.js`        | `login.spec.js`      |

> Tests live next to the files they test, or in a `__tests__/` folder.

---

## 14. Learning Outcomes

By the end of Testing Basics, you should be able to:

- [ ] Explain why testing is essential in professional development
- [ ] Write unit tests for utility functions using Vitest
- [ ] Set up React Testing Library with Vitest and jsdom
- [ ] Test React components render correctly
- [ ] Simulate user interactions (click, type, submit)
- [ ] Test form validation and submission
- [ ] Test async components that fetch API data
- [ ] Write end-to-end tests using Playwright
- [ ] Choose the right type of test for any given scenario
- [ ] Apply a practical testing strategy on real projects

---

## 15. Practice Projects

Apply your testing knowledge to these projects:

| Project           | Suggested Tests                                  |
| ----------------- | ------------------------------------------------ |
| Counter App       | Click increments/decrements, reset button works  |
| Login Form        | Validation errors, successful login, redirect    |
| Shopping Cart     | Add item, remove item, total calculation         |
| Blog App          | Posts load from API, post detail renders         |
| Job Board App     | Job list loads, filter works, apply form submits |
| Notes App         | Create note, edit note, delete note              |
| Dashboard UI      | Stats render, charts appear, data loads          |
| Landing Page Form | Newsletter form validation and submission        |

---

## Quick Reference

```
Unit Test (Vitest)
  → Test pure functions and logic in isolation
  → describe() / it() / expect()

Component Test (React Testing Library)
  → Test React components from the user's perspective
  → render() / screen.getByX() / userEvent.click()

E2E Test (Playwright)
  → Test full user flows in a real browser
  → page.goto() / page.fill() / page.click() / expect(page)
```
