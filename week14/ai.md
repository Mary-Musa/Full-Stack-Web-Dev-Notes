# Week 14 - AI Boundaries

**Ratio this week: 40% Manual / 60% AI**
**Habit introduced: "Framework docs before framework prompts."**
**Shift from last week: A 5% loosening. Next.js is a new framework, but the primitives (React components, hooks) are not new. You get AI help faster this week than Week 13.**

This week you jump from Vite + React + Express to Next.js 14 with the App Router. It is the same React mental model wearing a more powerful jacket: file-based routing, Server Components, Server Actions, built-in SSR, production deployment built in. You will feel the first day's setup as confusion and the fifth day's polished catalogue page as a visible step up.

The habit is specifically about Next.js's own docs. react.dev is excellent. nextjs.org/docs is also excellent but larger and more opinionated. Read it directly. Do not let AI paraphrase it for you this week.

---

## Why The Ratio Moved

You already know React. The only genuinely new parts this week are:
- The App Router folder convention (`app/page.js`, `layout.js`, nested routes)
- Server Components vs Client Components (a real mental shift)
- Data fetching in Server Components (no `useEffect`, no client fetch)
- Server Actions (mutations without REST routes)
- Metadata and SEO helpers

Five concepts. Three days of manual work, two days of AI-assisted building. 40/60 is the honest ratio.

---

## What You Will Feel This Week

- You will forget `"use client"` at the top of a file and see a cryptic error. Once you add it, everything clicks.
- Server Components will feel wrong for a day ("where is my useEffect?") and then feel amazing.
- You will try to import a Node module into a Client Component and the bundler will yell. Good.
- The first build-time generated static page will take 20 seconds and then be instant on every visit. You will feel the performance physically.

---

## What You MUST Do Manually (40%)

### Day 1 -- Why Next.js and first pages
- Read the Next.js "Getting Started" docs end to end. Not an AI summary.
- Create a fresh Next.js 14 project with the App Router manually (`npx create-next-app@latest`). Accept all the defaults first time; understand what each default means.
- Write your first page at `app/page.js`. Run it. Open the browser.
- Create a nested route at `app/about/page.js`. Understand how the filesystem is the routing.

### Day 2 -- Data fetching in Server Components
- Read the docs on Server Components. Note the word "default" -- every component is a Server Component unless you say otherwise.
- Fetch data in a Server Component using plain `fetch`. No `useEffect`. No `useState`. See how the component is async.
- Use the built-in caching. Make the same request twice; observe it only hits the network once.
- Write a Client Component with `"use client"` that uses `useState`. Mix it with a Server Component parent. Pass data down as props.

### Day 3 -- Dynamic routes and product pages
- Create a `[slug]` dynamic route at `app/products/[slug]/page.js`. Read `params.slug` inside the component.
- Fetch one product from a mock database (a JSON file or a local Postgres) by slug.
- Handle `notFound()` when the slug does not exist.

### Day 4 -- Layouts, Tailwind, and SEO
- Create a root `app/layout.js` with your nav and footer. Every child page inherits it.
- Install and configure Tailwind yourself. Do not let AI run the install.
- Add `generateMetadata` for per-page titles and descriptions.
- Write one `robots.ts` file and one `sitemap.ts` file. Understand what the search engines do with them.

### Day 5 and weekend -- Catalogue MVP
- Build a small catalogue page that lists products and links to a detail page per product.
- Fully SSR, no client-side fetch. Measure the initial page load.

---

## What You CAN Use AI For (60%)

- **All prior permissions.**
- **Scaffolding route files** after you have built 3-5 pages by hand.
- **Tailwind class suggestions** (AI is strong here).
- **Migration from Week 13's Vite frontend** to Next.js -- but only the component bodies, not the Server vs Client classification.

AI is forbidden for:
- Deciding which components should be Server vs Client (this is a judgement call you need to own).
- Writing your `layout.js` for you (understand the layout tree yourself).
- Generating SEO metadata for real content you have not written.

### Good vs bad prompts this week

**Bad:** "Build me a Next.js catalogue."
**Good:** "Here is my Server Component that fetches a product by slug [paste]. I want to also show a list of related products from the same category. Should the related query happen in the same Server Component, or in a separate sub-component? What is the cheaper option?"

**Bad:** "Make this page SSR."
**Good:** "My product page uses `useEffect` to fetch. I know I should move this to a Server Component. Here is my code [paste]. What are the exact three lines that need to change?"

---

## The 25-Minute Rule

Still 25. New framework = still worth the patience.

---

## Things AI Is Bad At This Week

- **Server vs Client classification.** AI defaults to sprinkling `"use client"` everywhere. Prefer Server unless you need state/effects/event handlers.
- **Cache semantics.** Next.js has several caches (`fetch`, route segment, full route, router). AI often mixes them up. Read the docs.
- **Async Server Components.** AI sometimes writes `useState` inside an async component. That does not work. Catch it.

---

## Core Mental Models For This Week

- **File is route.** `app/path/page.js` = `/path` URL.
- **Server Component by default.** Only opt into client with `"use client"`.
- **Layouts wrap pages and persist across navigation.** Nav and footer go here.
- **Server Actions are RPC calls to the server** triggered by forms or event handlers in Client Components.

---

## This Week's AI Audit Focus

Add a section: **Server/Client classification log.** For every component you created, record whether it is Server or Client and why:

```markdown
### ProductList.js
- Classification: Server Component
- Reason: Only renders data, no interactivity.

### AddToCartButton.js
- Classification: Client Component
- Reason: Needs onClick and local state.
```

---

## Assessment

- Walk through your catalogue: explain which components are Server, which are Client, and why.
- Live task: convert one Client Component to a Server Component (or vice versa) and explain the trade-off.
- Facilitator asks: "what happens on a hard refresh of `/products/nokia-105`?" You describe the server-side render path.

### Live Rebuild Check

Facilitator deletes your dynamic route and asks you to rewrite it from memory.

---

## One Sentence To Remember

"Next.js is React with opinions. Learn the opinions from the docs, not from prompts."
