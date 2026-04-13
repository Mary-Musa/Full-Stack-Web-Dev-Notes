# Week 14 - Day 1 Assignment

## Title
Scaffold A Next.js 14 Project And Build Your First Server Component

## Overview
Week 14 switches from Vite+React to Next.js 14 with the App Router. Today you scaffold a fresh Next.js project, delete the starter, write your first `page.js` and `layout.js`, and understand the difference between Server and Client Components. React primitives are the same; the rendering model is new.

## Learning Objectives Assessed
- Create a Next.js 14 project with the App Router
- Understand what runs on the server vs the client
- Write a Server Component that fetches data directly
- Add a Client Component with `"use client"`
- Use file-based routing

## Prerequisites
- Weeks 7-9 (React)

## AI Usage Rules

**Ratio this week:** 40% manual / 60% AI
**Habit:** Framework docs before framework prompts -- nextjs.org/docs first. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Generating additional pages after you wrote the first one.
- **NOT ALLOWED FOR:** Writing your first `page.js` or `layout.js`. Deciding which components are Server vs Client.
- **AUDIT REQUIRED:** Yes. Include a "Server/Client classification log" for every component you create.

## Tasks

### Task 1: Scaffold and clean

**What to do:**
```bash
npx create-next-app@latest shop-next
# Answers:
# TypeScript? No (stick to JS)
# ESLint? Yes
# Tailwind CSS? Yes
# src/ directory? No
# App Router? Yes
# Customize default import alias? No
```

```bash
cd shop-next
npm run dev
```

Visit http://localhost:3000. Delete the starter content in `app/page.js` and replace with a simple `<h1>Shop</h1>`.

**Expected output:**
Shop home page shows "Shop".

### Task 2: Your first Server Component with data

**What to do:**
Replace `app/page.js` with a Server Component that fetches data:

```jsx
export default async function HomePage() {
  const res = await fetch("https://jsonplaceholder.typicode.com/posts?_limit=5");
  const posts = await res.json();

  return (
    <main>
      <h1>Shop</h1>
      <ul>
        {posts.map((p) => (
          <li key={p.id}>{p.title}</li>
        ))}
      </ul>
    </main>
  );
}
```

Notice: no `useEffect`, no `useState`, no loading state. The data is fetched on the server before the page renders.

**Expected output:**
Home page shows 5 posts from jsonplaceholder. View source on the page -- the posts are in the HTML.

### Task 3: Add a Client Component

**What to do:**
Create `app/components/Counter.jsx`:

```jsx
"use client";
import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

Import and use it in `app/page.js`. Notice that the Server Component (page) contains the Client Component (Counter), and the `"use client"` directive marks the boundary.

**Expected output:**
Home page shows the list AND a working counter. Inspect source -- the counter markup is in the HTML, but the interactivity is hydrated on the client.

### Task 4: Nested layout

**What to do:**
Modify `app/layout.js` to add a nav:

```jsx
export const metadata = { title: "Shop", description: "Mctaba Shop" };

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <nav style={{ padding: 16, borderBottom: "1px solid #ddd" }}>
          <a href="/">Home</a> | <a href="/about">About</a>
        </nav>
        {children}
      </body>
    </html>
  );
}
```

Create `app/about/page.js`:

```jsx
export default function AboutPage() {
  return <h1>About the Shop</h1>;
}
```

Visit /about. The same nav shows. This is layout nesting.

**Expected output:**
Both pages share the same nav. Routing works without a client-side router library.

### Task 5: Server/Client classification log

**What to do:**
In `AI_AUDIT.md`, add a "Server/Client classification log". For every component you created today, list whether it is Server or Client and why:

```markdown
### app/page.js
- Classification: Server Component
- Reason: Fetches data; no state or event handlers.

### app/components/Counter.jsx
- Classification: Client Component
- Reason: Uses useState and onClick.

### app/layout.js
- Classification: Server Component
- Reason: No interactivity.
```

**Expected output:**
Audit updated.

## Stretch Goals (Optional - Extra Credit)

- Add `loading.js` in a route folder to show a loading state.
- Use `Metadata` exports per page for dynamic titles.
- Add a `<Link>` from next/link instead of `<a>`.

## Submission Requirements

- **What to submit:** Repo with `shop-next/` project, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Next.js project running | 10 | Dev server works. |
| Home page Server Component with fetch | 25 | Data in HTML. No useEffect. |
| Client Component with "use client" | 20 | Counter works and is explicitly marked. |
| Nested layout with nav | 15 | About page renders with same nav. |
| Classification log in audit | 25 | Every component classified with reasoning. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Putting `"use client"` in the layout.** Layouts should be Server Components unless they truly need to be interactive. Most do not.
- **Trying to use React hooks inside a Server Component.** Hooks are only allowed in Client Components. If you need state, create a Client Component.
- **Using `document` or `window` in a Server Component.** These do not exist on the server.

## Resources

- Day 1 reading: [Why Next.js and First Pages.md](./Why%20Next.js%20and%20First%20Pages.md)
- Week 14 AI boundaries: [../ai.md](../ai.md)
- Next.js App Router docs: https://nextjs.org/docs/app
