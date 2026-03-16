# Styling in React

## Learning Goals

By the end of this topic, learners should be able to:

- Style React components using multiple approaches and understand the **trade-offs** of each
- Build **responsive layouts** that work across mobile, tablet, and desktop
- Apply **Flexbox** and **CSS Grid** confidently for real, production-style layouts
- Avoid CSS conflicts as apps grow in size and team size
- Choose a styling approach **intentionally** depending on the project context
- Understand **why** each approach exists and **when** to reach for it

---

## 1) The React Truth About Styling

React does **not** come with a styling system built in. This is by design - React focuses on one thing: building UI with components. Styling is handled using normal web technologies (CSS) or libraries you add on top.

This is why the React ecosystem has multiple valid styling approaches. There is no single "right way." The best approach depends on your project size, team, and goals.

> **Why does this matter?**
> Because when you join a team or start a new project, you need to understand the trade-offs and make a deliberate decision - not just copy what you saw last time.

In most real-world React apps, styling comes from some combination of:

| Layer | Purpose |
|---|---|
| Global CSS | Baseline resets, typography, layout, CSS variables |
| Component-scoped CSS | Styles that apply only to one component |
| Utility classes (Tailwind) | Rapid layout and styling without writing CSS files |
| UI libraries / component kits | Pre-built, reusable, accessible patterns |

Understanding all four layers - and when to use each - is the goal of this topic.

---

# Part A - Plain CSS: The Foundation

## 2) Global CSS in a React App

In a Vite + React project, you typically have a global stylesheet at `src/index.css`. This file is imported by `src/main.jsx` (your entry point), which means its styles apply to your entire app.

```jsx
// src/main.jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App.jsx';
import './index.css'; // ← This is your global stylesheet

ReactDOM.createRoot(document.getElementById('root')).render(<App />);
```

Vite handles CSS imports natively - it bundles CSS for you in both development and production, with no extra configuration needed.

### What to Put in Your Global CSS

Your global stylesheet is a good place for things that affect the **whole app**:

```css
/* src/index.css */

/* 1. Box model reset - prevents width/padding calculation issues */
*,
*::before,
*::after {
  box-sizing: border-box;
}

/* 2. Body defaults */
body {
  margin: 0;
  font-family: 'Inter', sans-serif;
  font-size: 16px;
  line-height: 1.5;
  color: #1a1a1a;
  background-color: #ffffff;
}

/* 3. CSS Variables - your design tokens */
:root {
  --color-primary: #6366f1;
  --color-primary-dark: #4f46e5;
  --color-text: #1a1a1a;
  --color-muted: #6b7280;
  --color-surface: #f9fafb;
  --color-border: #e5e7eb;

  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 40px;

  --radius-sm: 6px;
  --radius-md: 12px;
  --radius-lg: 20px;

  --shadow-sm: 0 1px 3px rgba(0, 0, 0, 0.08);
  --shadow-md: 0 4px 12px rgba(0, 0, 0, 0.1);
}

/* 4. Links */
a {
  color: var(--color-primary);
  text-decoration: none;
}

a:hover {
  text-decoration: underline;
}

/* 5. Images */
img {
  max-width: 100%;
  display: block;
}
```

> **Why CSS variables?**
> CSS variables (also called custom properties) let you define your design values in one place and reuse them everywhere. When you decide to change your brand color, you update one line - not a hundred.

### The Principle: Separate Global from Local

| Global CSS | Component CSS |
|---|---|
| Resets and base styles | Component-specific layout |
| Typography defaults | Unique colors or borders |
| CSS variables | Hover states |
| Shared utilities | Responsive adjustments |

---

## 3) Responsive Design - Mobile-First CSS

### Why Mobile-First?

More than half of all web traffic is on mobile. Writing mobile-first CSS means:

1. Your **default styles** are for the smallest screens
2. You **add complexity** for larger screens with `min-width` media queries

This is the industry standard. Here's why it works better than the alternative:

```css
/* - Desktop-first (harder to maintain) */
.card {
  width: 360px; /* set for desktop... */
}

@media (max-width: 768px) {
  .card {
    width: 100%; /* ...then undo for mobile */
  }
}

/* - Mobile-first (cleaner, additive) */
.card {
  width: 100%; /* default: works on all small screens */
}

@media (min-width: 768px) {
  .card {
    width: 360px; /* add larger size for bigger screens */
  }
}
```

Mobile-first means you're **adding** features for larger screens rather than **removing** features for smaller ones. Additive is always simpler than subtractive.

### Standard Breakpoints

```css
/* Mobile: default - no media query needed */

/* Tablet */
@media (min-width: 768px) { ... }

/* Desktop */
@media (min-width: 1024px) { ... }

/* Large Desktop */
@media (min-width: 1280px) { ... }
```

### Responsive Typography Example

```css
h1 {
  font-size: 1.75rem;   /* mobile */
}

@media (min-width: 768px) {
  h1 {
    font-size: 2.25rem; /* tablet */
  }
}

@media (min-width: 1024px) {
  h1 {
    font-size: 3rem;    /* desktop */
  }
}
```

### Units That Make Responsive Design Easier

| Unit | What it's relative to | Best for |
|---|---|---|
| `px` | Fixed pixels | Borders, shadows, fine details |
| `rem` | Root font size (usually 16px) | Typography, spacing |
| `%` | Parent element | Fluid widths |
| `vw` / `vh` | Viewport width / height | Full-screen sections |
| `clamp()` | Min, preferred, max | Fluid type that scales smoothly |

```css
/* clamp() - fluid font size between 1rem and 3rem */
h1 {
  font-size: clamp(1.5rem, 5vw, 3rem);
}
```

### Practical Must-Know Habits

- Use `max-width` on your content containers with `margin: 0 auto` to center them
- Never set fixed heights on sections (use `min-height` when needed, or let content drive height)
- Always test at all 3 breakpoints - don't just check on your laptop

```css
.container {
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 var(--spacing-md);
}
```

---

## 4) Flexbox for Layout

### When to Use Flexbox

Flexbox is a **one-dimensional** layout system - it works powerfully along a single axis (either row or column). Use it when you need to:

- Align items horizontally in a navbar
- Center something both horizontally and vertically
- Distribute space between buttons or cards in a row
- Stack items vertically inside a column

### The Core Flexbox Properties

```css
.container {
  display: flex;

  /* Direction */
  flex-direction: row;          /* row (default), row-reverse, column, column-reverse */

  /* Horizontal alignment (main axis) */
  justify-content: space-between; /* flex-start, center, flex-end, space-around, space-evenly */

  /* Vertical alignment (cross axis) */
  align-items: center;          /* flex-start, flex-end, stretch, baseline */

  /* Gap between items */
  gap: 16px;

  /* Allow items to wrap to the next line */
  flex-wrap: wrap;
}
```

### Real Example: Navbar

```css
/* navbar.css */
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 var(--spacing-lg);
  height: 64px;
  background: #ffffff;
  border-bottom: 1px solid var(--color-border);
}

.nav-links {
  display: flex;
  gap: var(--spacing-md);
  list-style: none;
  margin: 0;
  padding: 0;
}
```

```jsx
function Navbar() {
  return (
    <nav className="navbar">
      <span className="logo">MyApp</span>
      <ul className="nav-links">
        <li><a href="/">Home</a></li>
        <li><a href="/about">About</a></li>
        <li><a href="/pricing">Pricing</a></li>
      </ul>
    </nav>
  );
}
```

### Real Example: Centering Content

The most common use case - centering something inside a container:

```css
.hero {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  min-height: 80vh;
  text-align: center;
}
```

### flex: 1 - What It Does

```css
.sidebar { flex: 0 0 240px; }  /* fixed 240px width, won't grow or shrink */
.main-content { flex: 1; }      /* takes up ALL remaining space */
```

This pattern is extremely useful for sidebar + content layouts.

### Flexbox Responsive Card Row

```css
.card-row {
  display: flex;
  flex-wrap: wrap;
  gap: var(--spacing-md);
}

.card {
  flex: 1 1 280px; /* grow, shrink, base width of 280px */
  /* This means: cards fill available space, but never shrink below 280px */
  /* When they can't fit, they wrap to the next row */
}
```

---

## 5) CSS Grid for Layout

### When to Use Grid

Grid is a **two-dimensional** layout system - it controls rows AND columns simultaneously. Use it when you need to:

- Build dashboard layouts (sidebar + main + header)
- Create card grids (products, portfolio items)
- Design multi-column article layouts
- Build admin panels or complex page structures

The key mental model: **Flexbox = one direction. Grid = both directions at once.**

### The Core Grid Properties

```css
.grid-container {
  display: grid;

  /* Define columns */
  grid-template-columns: 200px 1fr;         /* fixed sidebar + flexible content */
  grid-template-columns: repeat(3, 1fr);    /* 3 equal columns */
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr)); /* responsive! */

  /* Define rows (usually let content drive row height) */
  grid-template-rows: auto;

  /* Gap between cells */
  gap: 24px;
  row-gap: 16px;
  column-gap: 24px;
}
```

### Real Example: Responsive Card Grid

```css
.products-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: var(--spacing-lg);
}
```

This single line creates a grid that:
- Has as many columns as will fit
- Each column is at least 240px wide
- If space is available, columns expand to fill it
- On small screens, items naturally stack to one column

No media queries needed!

### Real Example: Dashboard Layout (App Shell)

```css
.app-shell {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main";
  grid-template-columns: 240px 1fr;
  grid-template-rows: 64px 1fr;
  min-height: 100vh;
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main    { grid-area: main; }
```

```jsx
function AppShell({ children }) {
  return (
    <div className="app-shell">
      <header className="header">...</header>
      <aside className="sidebar">...</aside>
      <main className="main">{children}</main>
    </div>
  );
}
```

`grid-area` with named areas makes complex layouts extremely readable.

### Making Grid Responsive

```css
.app-shell {
  display: grid;
  grid-template-areas:
    "header"
    "main";
  grid-template-columns: 1fr;
  grid-template-rows: 64px 1fr;
  min-height: 100vh;
}

/* Sidebar appears on tablet+ */
@media (min-width: 768px) {
  .app-shell {
    grid-template-areas:
      "header header"
      "sidebar main";
    grid-template-columns: 240px 1fr;
  }
}
```

### Flexbox vs Grid - A Decision Guide

| Situation | Use |
|---|---|
| Single row of items (navbar, button group) | Flexbox |
| Centering content | Flexbox |
| Items that wrap into rows based on available space | Flexbox |
| Dashboard layout (rows + columns defined) | Grid |
| Product or card grid | Grid |
| Full page layout structure | Grid |
| Content inside a card component | Flexbox |

> **The real answer:** They work together. Use Grid for the big page structure, Flexbox for the smaller UI pieces inside each section.

---

# Part B - CSS Modules: Scoped Styles

## 6) Why CSS Modules Exist

Plain global CSS works fine for small projects. But as your app grows, global class names start to collide:

```
// You have a .card class in Dashboard.css
// You have a .card class in Profile.css
// One of them accidentally overrides the other
// You spend 20 minutes figuring out why the border-radius looks wrong
```

This is the **global CSS collision problem**, and it gets worse as teams grow.

CSS Modules solve this by **scoping** class names locally to the file they're defined in. The CSS you write looks exactly the same - but at build time, class names get transformed into unique strings that can never collide.

```
/* What you write: */
.card { border-radius: 12px; }

/* What gets compiled into the HTML: */
.card_3xMk2 { border-radius: 12px; }
```

You never need to think about this transformation. You just write CSS, and the scoping is automatic.

## 7) How CSS Modules Work in Vite + React

With Vite, any CSS file ending in `.module.css` is treated as a CSS Module automatically - no configuration needed.

### Step 1: Create the Module File

```css
/* Button.module.css */

.button {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 10px 20px;
  border: none;
  border-radius: var(--radius-md);
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s ease, transform 0.1s ease;
}

.primary {
  background-color: var(--color-primary);
  color: white;
}

.primary:hover {
  background-color: var(--color-primary-dark);
  transform: translateY(-1px);
}

.secondary {
  background-color: transparent;
  color: var(--color-primary);
  border: 2px solid var(--color-primary);
}

.secondary:hover {
  background-color: var(--color-primary);
  color: white;
}
```

### Step 2: Import and Use in Your Component

```jsx
// Button.jsx
import styles from './Button.module.css';

export default function Button({ children, variant = 'primary', onClick }) {
  return (
    <button
      className={`${styles.button} ${styles[variant]}`}
      onClick={onClick}
    >
      {children}
    </button>
  );
}
```

```jsx
// Usage in another file
import Button from './components/Button';

function App() {
  return (
    <div>
      <Button variant="primary">Save Changes</Button>
      <Button variant="secondary">Cancel</Button>
    </div>
  );
}
```

### Composing Multiple Classes

When you need to combine classes conditionally, a common pattern is template literals:

```jsx
<button className={`${styles.button} ${isLoading ? styles.loading : ''}`}>
  Submit
</button>
```

Or use the `clsx` library for cleaner conditional classes:

```bash
npm install clsx
```

```jsx
import clsx from 'clsx';
import styles from './Button.module.css';

function Button({ children, variant, isLoading, disabled }) {
  return (
    <button
      className={clsx(
        styles.button,
        styles[variant],
        isLoading && styles.loading,
        disabled && styles.disabled
      )}
    >
      {children}
    </button>
  );
}
```

### What Stays Global vs What Goes in Modules?

| In `index.css` (global) | In `.module.css` (scoped) |
|---|---|
| CSS variables (`:root`) | Component-specific styles |
| Body, typography resets | Component hover states |
| Shared utility classes | Layout inside a component |
| Keyframe animations | Unique visual treatments |

---

# Part C - Tailwind CSS: Utility-First Styling

## 8) Why Tailwind Exists

Traditional CSS development has a tension:

- You write CSS classes in one file
- You apply them in another file (JSX/HTML)
- When a component changes, you update both files
- Over time, you accumulate unused CSS that nobody is sure is safe to delete

Tailwind takes a different approach: **put the styling directly in the markup**, using small, single-purpose utility classes.

```jsx
/* Instead of this: */
// card.css
.card {
  padding: 24px;
  border-radius: 12px;
  background-color: white;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

// Card.jsx
<div className="card">...</div>

/* You write this: */
<div className="p-6 rounded-xl bg-white shadow-sm">...</div>
```

The styles live next to the markup. When you delete the component, the styles disappear with it. No dead CSS.

### Why Teams Love Tailwind

1. **Speed** - No switching between files. No naming classes.
2. **Consistency** - All spacing, colors, and border-radius come from a shared scale.
3. **Responsive built-in** - `md:flex lg:grid` - responsive modifiers are prefix-based.
4. **No specificity wars** - Every class does one thing. No overrides.

## 9) The Tailwind Mindset Shift

The key insight: **Tailwind's classes are your design system.**

Instead of inventing names like `.big-blue-button` or `.hero-section-container`, you compose from small building blocks.

### Spacing Scale

Tailwind uses a spacing scale based on multiples of 4px:

| Class | Value |
|---|---|
| `p-1` | 4px |
| `p-2` | 8px |
| `p-4` | 16px |
| `p-6` | 24px |
| `p-8` | 32px |
| `p-12` | 48px |

The same scale applies to margin (`m-`), width (`w-`), height (`h-`), and gap (`gap-`). This keeps your UI consistent because **all measurements come from the same grid**.

### Common Class Families

```
Layout:     flex, grid, block, hidden, relative, absolute
Spacing:    p-4, px-6, py-2, mx-auto, gap-4, mt-8
Sizing:     w-full, max-w-xl, h-screen, min-h-[200px]
Typography: text-sm, text-lg, font-bold, text-center, leading-relaxed
Colors:     bg-white, text-gray-700, border-gray-200
Border:     border, border-2, rounded-lg, rounded-full
Shadow:     shadow-sm, shadow-md, shadow-xl
States:     hover:bg-blue-600, focus:ring-2, active:scale-95
Responsive: sm:flex-col, md:grid-cols-2, lg:grid-cols-3
```

### Real Example: Card Component in Tailwind

```jsx
function ProductCard({ title, price, image, description }) {
  return (
    <div className="bg-white rounded-xl shadow-md overflow-hidden hover:shadow-lg transition-shadow">
      <img
        src={image}
        alt={title}
        className="w-full h-48 object-cover"
      />
      <div className="p-5">
        <h3 className="text-lg font-semibold text-gray-900 mb-1">{title}</h3>
        <p className="text-sm text-gray-500 mb-4">{description}</p>
        <div className="flex items-center justify-between">
          <span className="text-xl font-bold text-indigo-600">${price}</span>
          <button className="px-4 py-2 bg-indigo-600 text-white text-sm font-medium rounded-lg hover:bg-indigo-700 transition-colors">
            Add to Cart
          </button>
        </div>
      </div>
    </div>
  );
}
```

### Real Example: Responsive Grid with Tailwind

```jsx
function ProductGrid({ products }) {
  return (
    <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
      {products.map(product => (
        <ProductCard key={product.id} {...product} />
      ))}
    </div>
  );
}
```

- Mobile: 1 column
- Small screens (≥640px): 2 columns
- Large screens (≥1024px): 3 columns
- Extra large (≥1280px): 4 columns

All in one class string. No media query files.

## 10) Tailwind Best Practices in React

### Extract Repeated Patterns into Components

Tailwind's utility classes get repetitive. The React answer to this is: **extract into components**.

```jsx
// - Repeating the same classes everywhere
<button className="px-4 py-2 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700">Save</button>
<button className="px-4 py-2 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700">Submit</button>

// - Extract into a reusable component
function Button({ children, variant = 'primary', size = 'md', ...props }) {
  const base = 'inline-flex items-center font-medium rounded-lg transition-colors';

  const variants = {
    primary: 'bg-indigo-600 text-white hover:bg-indigo-700',
    secondary: 'border border-indigo-600 text-indigo-600 hover:bg-indigo-50',
    ghost: 'text-gray-600 hover:bg-gray-100',
    danger: 'bg-red-600 text-white hover:bg-red-700',
  };

  const sizes = {
    sm: 'px-3 py-1.5 text-sm',
    md: 'px-4 py-2 text-sm',
    lg: 'px-6 py-3 text-base',
  };

  return (
    <button
      className={`${base} ${variants[variant]} ${sizes[size]}`}
      {...props}
    >
      {children}
    </button>
  );
}
```

Now you use `<Button variant="primary">`, `<Button variant="danger" size="lg">` - clean, readable, reusable.

### Keeping Long Class Strings Readable

When class lists get long, break them across lines with string concatenation or `clsx`:

```jsx
<div
  className={clsx(
    'relative flex flex-col',          // layout
    'p-6 rounded-xl',                  // spacing & shape
    'bg-white shadow-md',              // appearance
    'hover:shadow-lg transition-shadow', // interaction
    isHighlighted && 'ring-2 ring-indigo-500' // conditional
  )}
>
```

### Tailwind Does NOT Replace CSS Knowledge

This is critical to understand as a learner:

> Tailwind rewards CSS knowledge. If you don't understand Flexbox, you won't understand `flex`, `justify-center`, `items-center`. If you don't understand the box model, you won't know when to use padding vs margin.

Tailwind is a **layer on top** of CSS, not a replacement for it. Learn CSS first. Then Tailwind feels like a superpower.

---

# Part D - Component Libraries: Production Speed

## 11) Why Component Libraries Come Later

Component libraries (shadcn/ui, Chakra, MUI, etc.) are incredibly powerful. But there is a specific reason we teach them **after** styling fundamentals:

If you start with a component library before understanding CSS and layout:

- You copy/paste components without knowing what makes them work
- You can't customize them when the design needs something custom
- You get frustrated when the pre-built component doesn't quite fit your design
- You become **dependent** on the library instead of empowered by it

The right order: **CSS → Flexbox/Grid → CSS Modules → Tailwind → Component Libraries**

By the time you reach component libraries, you should feel like: *"I could build this myself, but the library saves me time."* That's the empowered position.

---

## 12) shadcn/ui

shadcn/ui is not a traditional component library. Instead of installing components from `npm`, you **copy them directly into your codebase**.

This means:
- You own the code - no black box
- You can modify any component freely
- You learn from looking at real component implementations
- Your bundle only includes what you actually use

### How It Works

```bash
# 1. Initialize shadcn/ui in a Vite + Tailwind project
npx shadcn@latest init

# 2. Add specific components as you need them
npx shadcn@latest add button
npx shadcn@latest add card
npx shadcn@latest add dialog
npx shadcn@latest add input
```

Each command adds a component file to your `src/components/ui/` folder. You can open, read, and modify that file.

### Using shadcn/ui Components

```jsx
import { Button } from '@/components/ui/button';
import { Card, CardHeader, CardContent, CardTitle } from '@/components/ui/card';
import { Input } from '@/components/ui/input';

function LoginForm() {
  return (
    <Card className="w-full max-w-md">
      <CardHeader>
        <CardTitle>Sign In</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <Input type="email" placeholder="Email address" />
        <Input type="password" placeholder="Password" />
        <Button className="w-full">Sign In</Button>
      </CardContent>
    </Card>
  );
}
```

### When to Reach for shadcn/ui

- **Dashboards** and admin panels - they have excellent table, dialog, and form components
- **SaaS products** - professional, clean aesthetic out of the box
- **Rapid prototyping** - when you want a polished UI quickly
- Any project already using Tailwind (shadcn requires it)

> **Real-world note:** shadcn/ui has become one of the most popular component systems in the React ecosystem (2024–2025). Being comfortable with it is a genuine job-market skill.

---

## 13) Chakra UI

Chakra UI is a more traditional component library - you install it from npm, and you don't own the component code.

Key features:
- Strong **accessibility** defaults (correct ARIA attributes, keyboard navigation)
- Built-in **theming** system
- Style props (pass CSS-like props directly to components)
- Large component library out of the box

```bash
npm install @chakra-ui/react @emotion/react @emotion/styled framer-motion
```

```jsx
import { ChakraProvider, Box, Button, Text, VStack } from '@chakra-ui/react';

function App() {
  return (
    <ChakraProvider>
      <VStack spacing={4} p={8}>
        <Text fontSize="xl" fontWeight="bold">Welcome</Text>
        <Button colorScheme="purple">Get Started</Button>
      </VStack>
    </ChakraProvider>
  );
}
```

### Chakra vs shadcn/ui

| | shadcn/ui | Chakra UI |
|---|---|---|
| Style system | Tailwind | Style props / Emotion |
| Code ownership | You own the component code | Library code (npm) |
| Customization | Full - just edit the file | Theme-based |
| Best for | Modern SaaS / dashboards | Apps needing accessibility + theming |
| Learning curve | Low | Medium |

In the McTaba path, **shadcn/ui** is the primary choice because it pairs with Tailwind (which you already know) and keeps you in control of your code.


---

# Part E - Advanced Patterns & Production Techniques

## 14) Dynamic Classes: The `cn` Utility

In professional React apps (especially those using shadcn/ui), we avoid messy string concatenation by using a utility function named `cn`. This function combines two powerful tools:

1.  **`clsx`**: Handles conditional logic (classes that are only applied if a condition is true).
2.  **`tailwind-merge`**: Intelligently merges Tailwind classes so that the last class "wins" in a conflict (e.g., if you pass `p-4` and then `p-2`, it ensures `p-2` is the one that actually applies).

### Setup

```javascript
// src/lib/utils.js
import { clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs) {
  return twMerge(clsx(inputs));
}
```

### Why Use This?

Without `cn`, you might write:
`className={`btn ${isActive ? 'active' : ''} ${className}``}`

With `cn`, it becomes clean and conflict-free:

```jsx
import { cn } from "@/lib/utils";

function Button({ isActive, className, children }) {
  return (
    <button className={cn(
      "px-4 py-2 rounded-md font-medium transition-colors", // Base styles
      "bg-gray-100 text-gray-900 hover:bg-gray-200",         // Default variant
      isActive && "bg-brand-500 text-white",                // Conditional style
      className                                             // Parent overrides
    )}>
      {children}
    </button>
  );
}
```

## 15) Tailwind Configuration & Design Tokens

A professional project rarely uses only the default Tailwind colors. You should define your project's **Design Tokens** (colors, spacing, fonts) in `tailwind.config.js`.

### Why Customize?
- **Brand Consistency**: Ensure the "Blue" in your buttons is the exact hex code from the marketing team.
- **Maintainability**: If the brand color changes, you update it in one file, and every component updates automatically.
- **Readability**: Using `bg-brand-primary` is more meaningful than `bg-indigo-600`.

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
export default {
  content: ["./index.html", "./src/**/*.{js,ts,jsx,tsx}"],
  theme: {
    extend: {
      colors: {
        brand: {
          50: "#f5f7ff",
          100: "#ebf0fe",
          500: "#6366f1", // Primary brand color
          600: "#4f46e5",
          700: "#4338ca", // Darker for hover states
        },
      },
      borderRadius: {
        'xl': '1rem',
        '2xl': '1.5rem',
      },
    },
  },
  plugins: [],
}
```

## 16) Accessibility & Focus States

Styling isn't just about looks - it's about **usability**.

### Interactive States
Never remove the focus ring (`outline-none`) without providing an alternative. Users who navigate with keyboards depend on that ring to know where they are.

**The `focus-visible` pattern:**
This utility only shows the focus ring when it's actually helpful (i.e., when a user is using a keyboard, not a mouse).

```jsx
<button className="focus-visible:ring-2 focus-visible:ring-brand-500 focus-visible:ring-offset-2 outline-none">
  Accessible Button
</button>
```

### Screen Reader Only Content
Sometimes you need text for accessibility that shouldn't be visible to sighted users.

```jsx
<button>
  <TrashIcon />
  <span className="sr-only">Delete item</span>
</button>
```

## 17) Dark Mode Implementation

Modern apps are expected to have a dark mode. Tailwind makes this simple using the `dark:` prefix.

### Step 1: Configuration
In `tailwind.config.js`, set `darkMode: 'class'`. This means Tailwind will apply dark styles whenever a parent (like `<html>`) has the `dark` class.

### Step 2: Implementation
Use the `dark:` prefix for every property that needs to change.

```jsx
<div className="bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-100 transition-colors">
  <h1 className="text-2xl font-bold">Hello World</h1>
  <p className="text-gray-600 dark:text-gray-400">
    This content adapts to your theme preference.
  </p>
</div>
```

> **Pro Tip:** In production React apps, we usually use a library like `next-themes` (for Next.js) or a custom React Context to manage toggling the `.dark` class on the `<html>` element.

## 18) Micro-animations with Framer Motion

A "premium" feel often comes from subtle movement. While CSS transitions are great for simple hovers, **Framer Motion** is the industry standard for React animations.

```bash
npm install framer-motion
```

### Example: Smooth Button Interaction

```jsx
import { motion } from "framer-motion";

function PremiumButton({ children }) {
  return (
    <motion.button
      whileHover={{ scale: 1.02 }}
      whileTap={{ scale: 0.98 }}
      initial={{ opacity: 0, y: 10 }}
      animate={{ opacity: 1, y: 0 }}
      className="px-6 py-3 bg-brand-600 text-white rounded-xl shadow-lg"
    >
      {children}
    </motion.button>
  );
}
```

**Why Framer Motion?**
- **Declarative**: You describe the states (`initial`, `animate`, `exit`), and the library handles the math.
- **Physics-based**: Animations feel "natural" rather than robotic.
- **Layout Animations**: It can automatically animate elements moving into new positions in a grid or list.

## 19) Folder Structure for Styles

As your project grows, don't just dump everything in `index.css`. Follow a structured approach:

- `src/styles/`
  - `globals.css` (Resets, fonts, tailwind base)
  - `tokens.css` (CSS variables if not using Tailwind config)
- `src/lib/utils.js` (The `cn` helper)
- `src/components/ui/` (Atomic, reusable components like `Button.jsx`, `Input.jsx`)
- `src/features/[feature-name]/` (Keep feature-specific styles or modules close to the components)

---

# Choosing the Right Styling Approach

This is a decision guide. Use it to think like an engineer, not just a student.

## When to Use Plain CSS

- - You are learning CSS fundamentals
- - Small personal projects or simple landing pages
- - You want zero tooling complexity
- - Rapid prototyping without setup overhead

## When to Use CSS Modules

- - You want to write real CSS but avoid global collisions
- - Your app is medium-sized with many components
- - Your team is comfortable with CSS but not Tailwind
- - You want predictable, scoped styles with no runtime overhead

## When to Use Tailwind

- - You want to move fast without writing CSS files
- - You're building lots of UI screens quickly
- - Your app needs a consistent spacing and color system
- - The team knows Tailwind (consistent class usage across components)
- - Building dashboards, admin panels, SaaS-style UIs

## When to Use Component Libraries

- - Speed matters and you want pre-built accessible components
- - Building a professional app UI that needs forms, dialogs, tables
- - The team is senior enough to customize library components
- - You're already using Tailwind (for shadcn/ui)

## Real-World Stacks

| Project Type | Typical Stack |
|---|---|
| Portfolio / landing page | Plain CSS or Tailwind |
| Startup MVP | Tailwind + shadcn/ui |
| SaaS dashboard | Tailwind + shadcn/ui |
| Enterprise app | Chakra UI or MUI + custom theme |
| Design system team | CSS Modules + Storybook |

---

# Inline Styles and the `style` Prop

React also supports inline styles via the `style` prop. These are JavaScript objects, not CSS strings.

```jsx
// Syntax: style prop takes a JS object
<div style={{ backgroundColor: '#f9fafb', padding: '16px', borderRadius: '12px' }}>
  Hello
</div>

// Or define the object separately
const cardStyle = {
  backgroundColor: '#ffffff',
  boxShadow: '0 1px 3px rgba(0,0,0,0.1)',
  borderRadius: '12px',
  padding: '24px',
};

<div style={cardStyle}>...</div>
```

### When Inline Styles Make Sense

- **Dynamic values** that come from props, state, or user interaction
- Values that can't easily be expressed as a fixed class name

```jsx
// Good use case: progress bar width driven by state
function ProgressBar({ percent }) {
  return (
    <div className="progress-track">
      <div
        className="progress-fill"
        style={{ width: `${percent}%` }}  // ← dynamic value
      />
    </div>
  );
}
```

### When to Avoid Inline Styles

- - For static design values (use CSS classes instead)
- - For hover states (you can't express `:hover` in inline styles)
- - For responsive styles (you can't express media queries inline)
- - For large numbers of properties (becomes unreadable quickly)

---

# CSS-in-JS (Brief Overview)

CSS-in-JS libraries like **Styled Components** and **Emotion** let you write CSS directly inside JavaScript files:

```jsx
// Styled Components example
import styled from 'styled-components';

const Button = styled.button`
  padding: 10px 20px;
  background: ${props => props.primary ? '#6366f1' : 'white'};
  color: ${props => props.primary ? 'white' : '#6366f1'};
  border: 2px solid #6366f1;
  border-radius: 8px;
  cursor: pointer;

  &:hover {
    opacity: 0.9;
  }
`;

// Usage
<Button primary>Save</Button>
<Button>Cancel</Button>
```

### Why We Don't Start Here

CSS-in-JS is powerful but adds complexity:
- Requires learning a library-specific API
- Can have runtime performance implications (styles are generated in JS)
- Can be overkill for most projects

> It was very popular circa 2018–2021. The ecosystem has largely moved toward Tailwind + CSS Modules or Tailwind + shadcn/ui for new projects. You'll encounter it in older codebases, but don't start new projects with it unless you have a specific reason.

---

# Practice Projects

## Project 1: Responsive Dashboard UI

**Goal:** Build a dashboard layout from scratch - no component library.

**Requirements:**
- Sidebar (fixed, 240px on desktop, hidden on mobile)
- Top navbar with logo and avatar
- Main content area with a stats grid (4 cards)
- Responsive: sidebar collapses on mobile (hamburger menu shows/hides it)

**What you'll practice:**
- `display: grid` with named areas
- Responsive CSS with media queries
- Flexbox inside the navbar and stat cards
- CSS variables for consistent spacing
- Toggling a class with React state for the mobile menu

---

## Project 2: Pricing Section

**Goal:** Build a pricing section showing 3 tiers.

**Requirements:**
- 3 pricing cards: Starter, Pro (highlighted as "Most Popular"), Enterprise
- Cards stack on mobile, show in a row on desktop
- "Pro" card has a different background, a badge, and a subtle drop shadow
- Each card has a feature list with check icons

**What you'll practice:**
- Grid layout
- Highlighting one item differently within a group
- Responsive stacking
- Reusing a PricingCard component with props

---

## Project 3: Landing Page

**Goal:** Build a complete marketing page from scratch.

**Requirements:**
- Hero section: headline, subtext, two CTA buttons
- Features grid: 6 features with icon + title + description
- Testimonials: 3 quote cards
- Pricing section (reuse from Project 2)
- Footer with links

**What you'll practice:**
- Composing multiple sections into a page
- Full responsive layout system
- Consistent spacing using CSS variables or Tailwind scale
- Visual hierarchy (font sizes, colors, spacing)

---

## Project 4: Admin Panel with shadcn/ui

**Goal:** Build a professional admin interface using Tailwind + shadcn/ui.

**Requirements:**
- Sidebar with nested navigation (icon + label)
- Header with user avatar dropdown
- Data table with sortable columns and row actions
- Dialogs for create/edit/delete actions
- Responsive structure (mobile layout)

**What you'll practice:**
- shadcn/ui component usage (Table, Dialog, DropdownMenu)
- Component composition
- Complex layout structure
- Managing UI state (open/close dialogs, selected rows)

---

# Common Mistakes to Avoid

## 1. Using `px` for Everything

```css
/* - Every value in px - breaks when user changes font size */
.card { padding: 16px; margin-top: 32px; }
h1 { font-size: 48px; }

/* - Use rem for typography and spacing */
.card { padding: 1rem; margin-top: 2rem; }
h1 { font-size: 3rem; }
```

## 2. Setting Fixed Heights on Flexible Containers

```css
/* - Breaks when content is longer than expected */
.card { height: 250px; }

/* - Let content drive height, set minimum if needed */
.card { min-height: 200px; }
```

## 3. Writing Desktop-First CSS

Always start mobile, then add layout complexity for larger screens.

## 4. Forgetting `box-sizing: border-box`

Without this, adding padding to an element increases its total size unexpectedly. Always include it in your global reset.

## 5. Overusing Inline Styles

Inline styles can't handle hover states, media queries, or pseudo-elements. Use classes for anything beyond truly dynamic values.

## 6. Using Component Libraries Before Understanding Layouts

If you can't build a 3-column grid yourself, you'll struggle to understand why the component library's grid isn't doing what you expect.

---

# Recommended Reading

| Resource | What It Covers |
|---|---|
| [Vite CSS Guide](https://vite.dev/guide/) | How Vite handles CSS imports, modules, and preprocessors |
| [CSS Modules Docs](https://github.com/css-modules/css-modules) | How CSS Modules compile and why they prevent collisions |
| [Tailwind Docs](https://tailwindcss.com/docs) | Full utility class reference + responsive, dark mode, theming |
| [shadcn/ui Docs](https://ui.shadcn.com/docs/installation/vite) | Official Vite installation + component list |
| [MDN Flexbox Guide](https://developer.mozilla.org/en-US/docs/Learn/CSS/CSS_layout/Flexbox) | Deep Flexbox reference |
| [MDN Grid Guide](https://developer.mozilla.org/en-US/docs/Learn/CSS/CSS_layout/Grids) | Deep CSS Grid reference |
| [CSS Tricks - A Complete Guide to Grid](https://css-tricks.com/snippets/css/complete-guide-grid/) | Visual reference for every Grid property |

---

*McTaba Labs - Week 8, Day 4 - Styling in React*
