# React Foundations

## Learning Objectives
By the end of this lesson, you will be able to:
- Explain what React is and why developers choose it over plain JavaScript.
- Describe the difference between a traditional website and a Single-Page Application (SPA).
- Understand and apply **component-based architecture**.
- Understand React's "Virtual DOM" concept at a practical, beginner level.
- Scaffold a new React project using **Vite**.
- Run and understand the local development server.
- Read and navigate the basic file structure of a React project.
- Write simple components using **JSX**.
- Render components inside other components (component composition).
- Use `export default` and named exports correctly in a React project.

---

## 1. What is React?

React is a **JavaScript library for building user interfaces**. It was created and is maintained by Meta (Facebook).

Instead of thinking of a web page as one big HTML file, React encourages you to break your UI into **small, self-contained, reusable pieces called components**. Think of it like building with LEGO bricks. Each brick (component) can be built once and used anywhere.

> **Simple analogy:** Imagine a card on an e-commerce site (image, price, title, "Add to Cart" button). Without React, you would copy-paste that HTML for every product. With React, you build it once as a `ProductCard` component and simply reuse it wherever you need it.

A React app is still just JavaScript running in the browser. React gives you a **structured way** to build UI, reuse it, and keep the screen in sync with data and user interactions.

---

## 2. Why Does React Exist?

Before React, developers often used **vanilla JavaScript** to update the DOM directly in response to user actions. This works on small sites but becomes difficult to maintain as applications grow. UI update code becomes scattered, hard to track, and easy to break.

React solved this through a few core ideas:

| Problem | React's Solution |
|---|---|
| Scattered UI update logic | Declarative components |
| Copy-pasted code | Reusable components |
| No clear UI structure | Tree-based component hierarchy |
| UI out of sync with data | Data-driven rendering |

---

## 3. Traditional Websites vs. Single-Page Applications (SPAs)

### Traditional Websites
When you click a link on a traditional website, your browser sends a request to the server and loads an **entirely new HTML page**. This causes a full page reload.

### Single-Page Applications (SPAs)
An SPA loads **one main HTML page** and uses JavaScript to update only the parts of the page the user interacts with - no full reloads required.

```
Traditional: click link → browser requests new page → full reload
SPA:         click link → JavaScript updates the current page → no reload
```

React is commonly used to build SPAs. It is especially powerful when your app has:
- Dynamic, interactive interfaces
- Dashboards and data tables
- Forms with live feedback
- Repeated UI sections
- Pages that consume data from APIs

---

## 4. Component-Based Architecture

The most important idea in React: **UI is built from components**.

A **component** is a JavaScript function that returns markup (JSX). It can be as small as a button or as large as an entire page section.

```jsx
function Welcome() {
  return <h1>Welcome to Mctaba Labs!</h1>;
}
```

### Why components matter
- **Reusability**: Write once, use anywhere.
- **Separation of concerns**: Each component handles one specific piece of UI.
- **Readability**: Smaller, focused files are easier to understand.
- **Maintainability**: You can fix a bug in one component without touching any other part of the app.

### Thinking in Components
When you look at a UI design, practice breaking it into discrete pieces:

```
App
├── Navbar
├── Hero
├── Features
│   ├── FeatureCard
│   ├── FeatureCard
│   └── FeatureCard
└── Footer
```

This hierarchical "break the UI into a component tree" approach is described in the official React **Thinking in React** guide, it's the mental model React developers use from day one.

---

## 5. The UI as a Tree

Components are nested inside one another, forming **parent-child relationships** - just like a family tree.

For example:
```
App
└── Layout
    ├── Header
    └── Main
        ├── Hero
        │   └── Button
        └── AboutSection
```

Understanding this tree mental model matters because:
- It makes it clear **which component contains which**.
- Later, it explains **how data flows** from parent to child (through props).
- It helps you design **where to put shared state**.

Even at the foundations stage, start actively drawing this tree whenever you look at a new design or app.

---

## 6. The Virtual DOM - Practical Explanation

You'll often hear the term **Virtual DOM**. It sounds complex, but the beginner idea is simple.

Normally, when data changes on a page, a developer must manually:
1. Find the right DOM element.
2. Update its text or class.
3. Insert or remove child elements.

React changes this. You simply **describe what the UI should look like** in your components. When data changes, React automatically figures out the minimum updates needed and applies them to the real DOM efficiently.

> **Analogy:** Think of a paper form vs a smart form. With a paper form (vanilla JS), if someone's address changes, you manually erase and rewrite every field that references it. With React (the smart form), you update the data once and the form updates itself everywhere automatically.

**The key takeaway:**  
*React lets you describe the UI declaratively, and React handles updating the browser view when things change.*

---

## 7. Setting Up a React Project with Vite

### Why Vite?
The official React documentation now recommends **Vite** as the build tool for custom React setups. **Create React App (CRA) has been officially sunset** and should not be used for new projects.

Vite gives you:
- A blazing-fast development server.
- Simple, clean project setup.
- Modern JavaScript defaults.

### Setup Steps

Run this command in your terminal to create a new project:

```bash
npm create vite@latest my-react-app
```

Then, when prompted, select **React** as the framework and **JavaScript** as the variant.

Next, install dependencies and start the dev server:

```bash
cd my-react-app
npm install
npm run dev
```

Open your browser and navigate to the URL shown (usually `http://localhost:5173`). Your React app is live!

---

## 8. The Development Server

When you run `npm run dev`, Vite starts a **local development server**. This is a tool that:
- Serves your React app in the browser at a local URL.
- **Auto-refreshes** the browser whenever you save a file (Hot Module Replacement).
- Shows errors directly in the browser if something breaks.

This tight feedback loop, edit code → save → see result, is what makes modern frontend development enjoyable. You never need to manually refresh the page.

---

## 9. React Project File Structure

When Vite scaffolds a React project, the key files are:

```text
my-react-app/
├── index.html          ← Browser entry point
├── package.json        ← Project config & dependencies
├── vite.config.js      ← Vite configuration
└── src/
    ├── main.jsx        ← App startup file
    ├── App.jsx         ← Root component (where you start)
    ├── App.css
    └── assets/
```

### What each file does

#### `index.html`
The single HTML file the browser loads. Contains a `<div id="root">` where React "mounts" the app. You rarely need to edit this.

#### `src/main.jsx`
This is the **app's entry point**. It connects your React component tree to the browser DOM using the `createRoot` API:

```jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.jsx';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

- `createRoot(...)` - tells React which DOM element to use as the root.
- `.render(<App />)` - tells React to display the `App` component inside it.
- `<StrictMode>` - a development wrapper that helps detect potential problems early. It has no effect in production.

#### `src/App.jsx`
The **root component** - where you will spend most of your time. You can start by clearing out the default content and building your own UI here:

```jsx
export default function App() {
  return <h1>Hello React!</h1>;
}
```

---

## 10. JSX - JavaScript + HTML Markup

**JSX** is a syntax extension for JavaScript that lets you write HTML-like markup directly inside a JavaScript file.

```jsx
function Greeting() {
  const name = 'Bonaventure';

  return (
    <div>
      <h1>Hello, {name}!</h1>
      <p>Welcome to React Foundations.</p>
    </div>
  );
}
```

### JSX is NOT exactly HTML
While JSX looks like HTML, it follows JavaScript rules. Key differences:

| HTML | JSX Equivalent |
|---|---|
| `class="..."` | `className="..."` |
| `<br>` | `<br />` (must be self-closing) |
| `onclick="..."` | `onClick={...}` (camelCase) |
| `for="..."` | `htmlFor="..."` |

### Embedding JavaScript in JSX
Use `{ }` curly braces to embed any JavaScript **expression** directly inside your markup:

```jsx
const score = 95;
return <p>Your score is: {score}</p>;         // Value
return <p>Double score: {score * 2}</p>;      // Expression
return <p>Result: {score > 50 ? 'Pass' : 'Fail'}</p>; // Ternary
```

### One Parent Element Rule
A component must return **one** root element. If you need to return sibling elements without adding an extra `<div>`, use a **Fragment** (`<>...</>`):

```jsx
// This will cause an error:
return (
  <h1>Hello</h1>
  <p>World</p>
);

// Use a Fragment:
return (
  <>
    <h1>Hello</h1>
    <p>World</p>
  </>
);
```

---

## 11. Writing Components

A React component is a function that returns JSX. Three rules to always remember:

### Rule 1: Names start with a capital letter
React uses this to distinguish your custom components from built-in HTML tags like `<div>` or `<p>`.

```jsx
function ProfileCard() { ... }  // This is a React component
function profileCard() { ... }  // React treats this as an HTML tag
```

### Rule 2: Components return JSX (or null)
The return value of a component is what gets rendered on screen.

### Rule 3: Components can be nested
You can render any component inside any other component:

```jsx
function Header() {
  return <h1>Mctaba Labs</h1>;
}

function App() {
  return (
    <div>
      <Header />
      <p>Building excellent engineers.</p>
    </div>
  );
}
```

> **Always use function components.** React still supports class components but the official docs state that function components are the recommended style for all new React code. Start with functions and stay with functions.

---

## 12. Reusable UI Thinking

One of the most important habits to build early in React is asking: *"Can this be a component?"*

Instead of writing three different profile cards by hand:

```jsx
// Repetitive code:
<div><h2>Alice</h2><p>Designer</p></div>
<div><h2>Bob</h2><p>Developer</p></div>
<div><h2>Carol</h2><p>PM</p></div>
```

Build one component and reuse it:

```jsx
// Build once, use many times:
function ProfileCard() {
  return (
    <div>
      <h2>Student Name</h2>
      <p>Aspiring Full Stack Developer</p>
    </div>
  );
}

export default function App() {
  return (
    <div>
      <ProfileCard />
      <ProfileCard />
      <ProfileCard />
    </div>
  );
}
```

> **Upcoming:** In the next lesson, you'll learn about **props** - which allow you to pass *different data* into the same component, making the ProfileCard above show different names and roles each time.

---

## 13. Exports & Imports

React projects use JavaScript ES modules. Each component typically lives in its own file and must be **exported** before it can be **imported** elsewhere.

### Default Export
One per file. Used for the main component in a file.

```jsx
// ProfileCard.jsx
export default function ProfileCard() {
  return <div>Profile Card</div>;
}
```
```jsx
// App.jsx
import ProfileCard from './ProfileCard.jsx';
```

### Named Export
Multiple per file. Used when a file exports several things.

```jsx
// Layout.jsx
export function Header() {
  return <h1>Header</h1>;
}

export function Footer() {
  return <p>Footer</p>;
}
```
```jsx
// App.jsx
import { Header, Footer } from './Layout.jsx';
```

### Rule of Thumb
- **`export default`** → one main component per file.
- **Named exports** → utility components and helpers that belong together.

---

## 14. Composing a Page from Components

This is how all real React apps are built. You never put everything in one file.

```jsx
// Components live in their own files
function Navbar() {
  return <nav>Navigation</nav>;
}

function Hero() {
  return <section>Hero Section</section>;
}

function Footer() {
  return <footer>Footer</footer>;
}

// App.jsx composes them all together
export default function App() {
  return (
    <div>
      <Navbar />
      <Hero />
      <Footer />
    </div>
  );
}
```

This is called **component composition** and it is the fundamental pattern you will use for the rest of your React career.

---

## 15. Suggested Folder Structure

As your project grows, organize your components into a dedicated folder:

```text
my-react-app/
├── public/
└── src/
    ├── components/
    │   ├── Navbar.jsx
    │   ├── ProfileCard.jsx
    │   └── Footer.jsx
    ├── App.jsx
    ├── main.jsx
    └── index.css
```

This keeps your root files lean and your reusable components easy to find.

---

## 16. Beginner Workflow: Building a UI

Follow this process every time you build a new React UI:

1. **Look at the design.** What visible sections does this page have?
2. **Break it into components.** Draw the component tree on paper if it helps.
3. **Build a static version first.** No state, no data fetching yet - just structure and layout.
4. **Compose in `App.jsx`.** Render all your small components inside the root.
5. **Refactor repeated markup.** If you wrote similar JSX more than once, extract it into its own component.

---

## 17. Common Beginner Mistakes

| Mistake | Fix |
|---|---|
| Lowercase component name (`profileCard`) | Always capitalize: `ProfileCard` |
| Returning multiple sibling elements without a wrapper | Wrap in `<div>` or `<>...</>`  |
| Forgetting to export a component | Add `export default` or `export` in front of the function |
| Using `class` instead of `className` | Use `className` in JSX |
| Putting everything in `App.jsx` | Extract sections into their own component files |

---

## 18. Practice Projects

These projects fit perfectly at the React Foundations stage - all structure and composition, no state or data fetching yet.

### A. Profile Card App
**Goal:** Build a page displaying one or more profile cards.  
**Structure:** `App` → `ProfileCard` → `Avatar` + `Bio`  
**Concepts:** JSX, components, nesting, imports/exports.

### B. Simple Landing Page
**Goal:** Build a full landing page in React components.  
**Sections:** Navbar, Hero, Features, Footer.  
**Concepts:** Component architecture, UI as a tree, file organization.

### C. Quote Card  
**Goal:** Build a presentational quote card and display layout.  
**Concepts:** Reusable components, JSX expressions, modular file structure.

---

## 19. Checkpoint Questions

Use these to test your understanding before moving on:

1. What problem does React solve better than writing one huge page with plain DOM manipulation?
2. What is a component, in your own words?
3. Why must React component names start with a capital letter?
4. What is JSX and how is it different from HTML? Give two specific examples.
5. What does `createRoot` do in `main.jsx`?
6. What is the difference between `export default` and a named export? When would you use each?
7. Why is it useful to think of the UI as a tree?
8. If a page has a Navbar, Hero, and Footer, how would you break it into components?
9. Why is Vite the recommended setup tool instead of Create React App?
10. What is wrong with this JSX, and how would you fix it?

```jsx
function App() {
  return (
    <h1>Hello</h1>
    <p>World</p>
  );
}
```

---

## 20. Assignment: Profile Card App

Build a **Profile Card App** with the following requirements:

- Use **Vite** to scaffold the project.
- Create **at least 3 components** (e.g., `App`, `ProfileCard`, `Avatar`).
- Render components inside `App.jsx`.
- Use **JSX correctly** (proper casing, curly braces for expressions, single root element).
- Organize components into **separate files** inside a `components/` folder.
- Use **one default export** and **at least one named export**.
- Style the app so the distinct sections are visually clear.

> A strong submission demonstrates clear component thinking and structure. Prioritize structure over styling.

---

## Key References

- [React Quick Start](https://react.dev/learn)
- [Your First Component](https://react.dev/learn/your-first-component)
- [Writing Markup with JSX](https://react.dev/learn/writing-markup-with-jsx)
- [JavaScript in JSX with Curly Braces](https://react.dev/learn/javascript-in-jsx-with-curly-braces)
- [Understanding Your UI as a Tree](https://react.dev/learn/understanding-your-ui-as-a-tree)
- [Thinking in React](https://react.dev/learn/thinking-in-react)
- [Build a React App from Scratch (Vite setup)](https://react.dev/learn/build-a-react-app-from-scratch)
