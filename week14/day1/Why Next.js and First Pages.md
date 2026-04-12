# Week 14, Day 1: Why Next.js and First Pages

By the end of today, you will have a fresh Next.js 14 project running on your laptop, understand why we are switching from Vite/CRA to Next.js now, have your first pages with the new App Router, and know the difference between Server Components and Client Components in your bones -- not just on paper. You will not have a working e-commerce app yet. Today is the shift of mental model.

This is the start of Phase 3 of the Marathon: **commerce, payments, and automation**. The entire phase is built on Next.js. Three weeks from today you will have a working online shop that takes real M-Pesa payments and sends WhatsApp confirmations. That is a lot of ground to cover, and Next.js -- more than React alone -- is the tool that makes it cover-able in three weeks.

**Prior-week concepts you will use today:**
- React components, props, hooks, JSX (Weeks 7-9)
- The Week 11-13 CRM backend in Express (Weeks 11-13)
- File-based routing from React Router (Week 8, Day 3) -- Next.js takes this further
- Node, npm, environment variables (Week 10, Day 1)

**Estimated time:** 3-4 hours

---

## The Week Ahead

| Day | What You Build |
|---|---|
| Day 1 (today) | Fresh Next.js project, the App Router, first pages, Server vs Client Components. |
| Day 2 | Data fetching: Server Components, `fetch` caching, `async` components, loading states. |
| Day 3 | Dynamic routes, a product catalog page backed by a Postgres `products` table. |
| Day 4 | Tailwind refresh, layouts, metadata, SEO basics for e-commerce. |
| Day 5 | Week recap + peer session. |
| Weekend | Catalogue MVP -- a browsable shop with real products, no checkout yet. |

You are not going to ship a checkout this week. Week 15 handles cart and checkout state machines; Week 16 ships Project 3 (E-commerce Lite + WhatsApp confirmations). This week is foundation. Do not rush.

---

## Why Next.js, Why Now

You are perfectly capable of building e-commerce in plain React + Express. People do. You would:

- Set up React with Vite.
- Set up Express with a `/api/products` endpoint.
- Fetch on the client with `useEffect`, handle loading and error states by hand.
- Wire React Router for `/products`, `/products/:id`, `/cart`, `/checkout`.
- Deploy the frontend and backend separately, wire CORS, set up env vars on two hosts.

This works. It is also 30% boilerplate -- data fetching boilerplate, routing boilerplate, deployment boilerplate. Next.js collapses all three into "the framework handles it". Here is the short list of what you get for the switch:

1. **File-system routing** -- `app/products/page.tsx` is automatically the `/products` route. No router config.
2. **Server Components** -- a React component that runs on the server, talks to the database directly, and sends only HTML down to the client. No `useEffect`, no loading spinner, no API endpoint in between.
3. **Server Actions** -- a function marked `"use server"` that the client can call like a normal JS function but executes on the server. No `/api` route needed for mutations.
4. **Built-in image optimisation** (`<Image>`) which is huge for an e-commerce site with product photos.
5. **SEO basics** (`metadata` exports, server-rendered HTML) which matter because people find products on Google.
6. **One deploy** -- Vercel takes the repo and runs both the "frontend" and "backend" as one app. No CORS. No separate Express server for the shop.

If you are asking "do I still need the Express server from Weeks 11-13?" -- yes, for two reasons. First, the WhatsApp webhook, the USSD endpoint, and the M-Pesa callback still live there. Those are public third-party integrations and the Next.js app does not replace them. Second, the CRM dashboard from Week 12 stays on the Express + React setup. This week you are building a *second* frontend -- the customer-facing shop -- using Next.js. The two frontends share the same Postgres database. The Express server owns the webhooks and the CRM. The Next.js app owns the shop.

In Week 15 the two apps will start talking to each other directly -- the shop will insert orders into the same Postgres database, and the Express CRM will start seeing those orders appear. This is the architecture pattern called "one backend, many frontends", and it is the normal shape of real companies.

### Why not stick with Vite

Vite is a wonderful tool for pure client-side apps. The Week 11 CRM dashboard is a pure client-side app -- the user logs in, the browser fetches JSON, the browser renders it -- and Vite is the right tool for that. An e-commerce site is different:

- A logged-out visitor must see the shop. (Vite's single-page bundle can do this but hurts SEO.)
- The product page must be shareable on WhatsApp and Instagram with a nice preview image. (Requires server-rendered HTML with Open Graph tags.)
- The product listing page should be fast on 3G. (Server rendering + streaming wins.)
- Prices and stock change frequently. (Cache invalidation needs to be easy.)

Next.js handles each of these out of the box. In React+Vite you would hand-roll each.

### A word on the App Router vs the Pages Router

Next.js used to have a `pages/` folder (the "Pages Router"). In version 13 they shipped a new `app/` folder (the "App Router") which is now the default. Every tutorial on the web is split between the two. We use the App Router exclusively. If you see a tutorial importing from `next/router` or using `getServerSideProps`, that is the old Pages Router -- ignore it for this course.

---

## Creating The Project

Spin up a fresh Next.js 14 project. Do this *outside* the Week 12 CRM directory -- it is a separate app.

```bash
cd ~/Code
npx create-next-app@latest shop
```

The installer will ask you several questions. Here are the right answers for this course:

```
Would you like to use TypeScript?      No    (we stay on plain JS)
Would you like to use ESLint?          Yes
Would you like to use Tailwind CSS?    Yes
Would you like to use src/ directory?  No
Would you like to use App Router?      Yes   (the whole point)
Would you like to use Turbopack?       Yes   (faster dev server)
Would you like to customize default import alias? No
```

TypeScript is fine if you are comfortable with it, but the course examples and the peer reviews assume plain JS. Stick with what the cohort is using.

When it finishes:

```bash
cd shop
npm run dev
```

Visit http://localhost:3000. You should see the default Next.js landing page. Kill the server (Ctrl+C).

### Folder tour

Open the project in your editor. The structure you care about today:

```
shop/
  app/
    layout.js        # the root layout -- wraps every page
    page.js          # the home page, /
    globals.css      # tailwind imports
  public/            # static files (images, favicon)
  next.config.js
  tailwind.config.js
```

The `app/` folder is the heart of everything. The rule is: any file called `page.js` becomes a route, and the path of the route matches the folder path.

Let us prove it. Create `app/about/page.js`:

```jsx
// app/about/page.js
export default function AboutPage() {
  return (
    <main className="p-8">
      <h1 className="text-3xl font-bold">About Us</h1>
      <p className="mt-4">We sell good things at fair prices.</p>
    </main>
  );
}
```

Save. Visit http://localhost:3000/about. It is there. No config file, no router registration, no imports. The file *is* the route.

Create `app/contact/page.js`:

```jsx
// app/contact/page.js
export default function ContactPage() {
  return (
    <main className="p-8">
      <h1 className="text-3xl font-bold">Contact</h1>
      <p>Call +254712000000 or email hi@mctaba.co.ke</p>
    </main>
  );
}
```

Visit `/contact`. There it is. This is the single biggest thing that makes Next.js feel fast to work in: adding a page is always "make a new folder, put `page.js` in it".

### Nesting

Nested routes are nested folders. Create `app/products/phones/page.js`:

```jsx
export default function PhonesPage() {
  return <main className="p-8"><h1>Phones</h1></main>;
}
```

Visit `/products/phones`. Works. You can nest as deep as you want.

---

## The Layout

Open `app/layout.js`. It looks like:

```jsx
import "./globals.css";

export const metadata = {
  title: "Shop",
  description: "A fullstack marathon shop",
};

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

`RootLayout` wraps every page on the site. It is exactly what it sounds like -- the thing you would have put in `App.jsx` in React, minus the router. Add a header and a footer:

```jsx
// app/layout.js
import "./globals.css";
import Link from "next/link";

export const metadata = {
  title: "Mctaba Shop",
  description: "Good things at fair prices",
};

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body className="min-h-screen flex flex-col">
        <header className="border-b bg-white">
          <nav className="max-w-5xl mx-auto flex items-center justify-between p-4">
            <Link href="/" className="font-bold text-xl">Mctaba</Link>
            <div className="flex gap-6">
              <Link href="/products/phones">Phones</Link>
              <Link href="/about">About</Link>
              <Link href="/contact">Contact</Link>
            </div>
          </nav>
        </header>

        <main className="flex-1">{children}</main>

        <footer className="border-t p-4 text-center text-sm text-gray-500">
          &copy; {new Date().getFullYear()} Mctaba Shop
        </footer>
      </body>
    </html>
  );
}
```

Three things are new compared to React Router.

**`import Link from "next/link"`** -- Next.js has its own Link component. It does client-side navigation (no page reload) and also tells Next.js to prefetch the linked page in the background. Using a plain `<a>` would force a full reload.

**`metadata`** is an exported object -- this becomes `<title>` and `<meta name="description">` in the rendered HTML. Page-level metadata overrides the root layout's. In Week 14 Day 4 we will use dynamic metadata on product pages for WhatsApp/Instagram sharing previews.

**`new Date().getFullYear()`** runs on the *server*. This is a Server Component (the default in Next.js 14). The year is computed once on the server, sent as HTML, and never runs in the browser. If you viewed the page source, you would see the literal year in the HTML.

---

## Server Components vs Client Components

This is the concept that trips everyone up. Take your time.

In Next.js 14 with the App Router, every component is a **Server Component by default**. That means the component runs on the server -- on the Node.js process -- and produces HTML that is sent to the browser. The browser never sees the component's code.

Server Components can:
- Fetch data (directly from the database, no API route needed).
- Read files, env vars, secrets.
- Import and use Node libraries.
- Be `async` functions.

Server Components cannot:
- Use hooks like `useState` or `useEffect`.
- Handle click events.
- Access `window` or `document`.
- Re-render based on user input.

If you need any of those things, you declare a **Client Component** by putting `"use client"` at the top of the file:

```jsx
"use client";
import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

This file, and anything it imports, is shipped as JavaScript to the browser and hydrated like a normal React component.

### The mental model that works

Think of it this way: the page starts as HTML generated by Server Components. Anywhere in that HTML that needs to be interactive, a Client Component is "sprinkled in" -- hydrated in place to add event handlers and state.

A typical e-commerce product page:

```
ProductPage                  [Server Component]
  ProductDetails             [Server Component, fetches from DB]
  RelatedProducts            [Server Component, fetches from DB]
  AddToCartButton            [Client Component, needs onClick + cart state]
  ReviewList                 [Server Component]
    ReviewForm               [Client Component]
```

The whole page is server-rendered except for the two interactive pieces. The browser downloads HTML for everything, plus a small JavaScript bundle for the two client islands. Result: fast first paint, low bundle size, real interactivity where it matters.

Contrast with plain React + Vite where *everything* is a Client Component and the browser must download and run every line of your app before showing anything. On 3G, that is seconds of blank screen. On Next.js, it is milliseconds.

### Try it yourself

Add a small interactive component. Create `app/components/Counter.js`:

```jsx
"use client";
import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button
      onClick={() => setCount(count + 1)}
      className="bg-black text-white px-4 py-2 rounded"
    >
      Clicked {count} times
    </button>
  );
}
```

Now edit `app/page.js` (the home page):

```jsx
// app/page.js
import Counter from "./components/Counter";

export default function HomePage() {
  return (
    <div className="max-w-5xl mx-auto p-8">
      <h1 className="text-4xl font-bold">Welcome to Mctaba</h1>
      <p className="mt-4 text-gray-600">A shop with good things at fair prices.</p>
      <div className="mt-6">
        <Counter />
      </div>
      <p className="mt-6 text-sm text-gray-500">
        Rendered at {new Date().toLocaleTimeString()}
      </p>
    </div>
  );
}
```

Run `npm run dev`, visit `/`. Click the counter -- it works. Refresh the page. Notice the timestamp changes. That timestamp is computed on the *server* every time you refresh; the counter state lives on the *client* and resets when you refresh.

Two components, two different execution contexts, in the same JSX. That is the App Router.

### The one rule that bites people

**A Server Component can import a Client Component. A Client Component cannot import a Server Component.**

Think about why: once you cross into client-land, everything in the import tree gets shipped to the browser. You cannot have a server-only database query inside code that must run in a browser -- it does not have a database connection. Next.js will throw a clear error if you try.

The fix is almost always to pass the server-computed thing as a *prop* from a Server Component to the Client Component:

```jsx
// Server Component
export default async function ProductPage() {
  const product = await db.query("SELECT * FROM products WHERE id = $1", [id]);
  return <AddToCartButton product={product} />;  // pass as prop
}

// AddToCartButton (Client Component)
"use client";
export default function AddToCartButton({ product }) {
  // uses product.id, product.price etc -- but didn't fetch them
}
```

Remember the pattern: **fetch in Server Components, use in Client Components via props**.

---

## Checkpoint

1. `npm run dev` starts the shop on port 3000 with no errors.
2. You have four routes: `/`, `/about`, `/contact`, `/products/phones`. Visit each.
3. The header and footer from the root layout appear on every page without being imported in each page.
4. The home page shows a Counter that increments on click.
5. You can articulate the difference between a Server Component and a Client Component in one sentence without looking.
6. You know what `"use client"` means and where it goes in a file.
7. Opening the browser DevTools -> Network tab, clicking between pages, you see that `next/link` navigates without a full page reload (look for missing "document" requests).
8. `view-source:http://localhost:3000/` in the browser shows the page content as HTML, including the server-rendered timestamp.

Commit:

```bash
git init
git add .
git commit -m "chore: scaffold next.js shop with basic routes and layout"
```

---

## What To Read Before Tomorrow

- The "Routing Fundamentals" page of the Next.js docs: https://nextjs.org/docs/app/building-your-application/routing
- Dan Abramov's "Server Components" post -- do not read it cover to cover, but skim the first third so the mental model lands: https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md
- Optional: the "Parallel Routes" page of the Next.js docs. Very powerful, not on the critical path for this week but worth being aware of.

Tomorrow we connect the shop to the Week 12 Postgres database and fetch real products inside Server Components. No REST API in between. You will be surprised how few lines it takes.
