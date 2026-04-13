# Week 14 - Day 4 Assignment

## Title
Layouts, Tailwind Styling, And SEO

## Overview
Day 4 is pre-weekend polish day. Today you apply Tailwind to your product catalogue, build a shared site layout, add metadata for SEO, and generate a sitemap. By end of day your catalogue looks like something you would share with a friend.

## Learning Objectives Assessed
- Use Tailwind classes effectively in Next.js
- Build nested layouts for shop pages
- Add page-level metadata for SEO
- Generate a sitemap for search engines

## Prerequisites
- Days 1-3 completed

## AI Usage Rules

**Ratio:** 40/60. **Habit:** Framework docs first. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Tailwind class suggestions, layout wrappers.
- **NOT ALLOWED FOR:** Blindly accepting classes you do not understand.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Shop layout wrapper

**What to do:**
Create `app/(shop)/layout.js` (route group -- the parentheses don't affect URL):

```jsx
import Link from "next/link";

export default function ShopLayout({ children }) {
  return (
    <div>
      <header className="bg-gray-900 text-white p-4">
        <div className="max-w-6xl mx-auto flex justify-between">
          <Link href="/" className="font-bold">Shop</Link>
          <nav className="space-x-4">
            <Link href="/products">All products</Link>
            <Link href="/about">About</Link>
          </nav>
        </div>
      </header>
      <div className="max-w-6xl mx-auto p-4">{children}</div>
      <footer className="bg-gray-100 p-4 text-center">Mctaba Shop</footer>
    </div>
  );
}
```

Move your `app/products/page.js` into the `(shop)` group. Same for about.

**Expected output:**
Consistent header, body, footer on every shop page.

### Task 2: Tailwind polish

**What to do:**
Apply Tailwind to the product grid and detail pages. Make them visually pleasant. At minimum:
- Responsive grid (1 col mobile, 2 tablet, 4 desktop)
- Hover effect on product cards
- Proper typography (font-semibold for titles, text-gray-600 for prices)
- Padding and max-width containers

Use `clamp()` or Tailwind's responsive prefixes (`md:`, `lg:`). AI-assisted is fine.

**Expected output:**
Products page looks professional. Screenshot `day4-tailwind.png`.

### Task 3: Per-page metadata

**What to do:**
Add `metadata` exports to:
- `app/(shop)/page.js` -- Home page metadata
- `app/(shop)/products/page.js` -- Products list metadata
- `app/(shop)/products/[slug]/page.js` -- Already has `generateMetadata` from Day 3; extend to include description and openGraph

Example:

```jsx
export const metadata = {
  title: "Mctaba Shop",
  description: "Your local online store",
  openGraph: {
    title: "Mctaba Shop",
    description: "Your local online store",
    type: "website",
  },
};
```

**Expected output:**
Tab titles and meta tags visible. View page source.

### Task 4: Sitemap

**What to do:**
Create `app/sitemap.js`:

```javascript
import pool from "@/lib/db";

export default async function sitemap() {
  const { rows: products } = await pool.query("SELECT slug, created_at FROM products");

  const base = "http://localhost:3000";
  const productUrls = products.map((p) => ({
    url: `${base}/products/${p.slug}`,
    lastModified: p.created_at,
  }));

  return [
    { url: `${base}/`, lastModified: new Date() },
    { url: `${base}/products`, lastModified: new Date() },
    ...productUrls,
  ];
}
```

Visit `/sitemap.xml`. It should return generated XML.

**Expected output:**
Sitemap renders.

### Task 5: Pre-weekend checklist

**What to do:**
Create `CHECKLIST.md`:

```markdown
## Week 14 Day 4 Pre-Weekend Checklist

- [ ] Next.js project running
- [ ] lib/db connects to Postgres cleanly
- [ ] Products listing page works
- [ ] Dynamic product detail pages work
- [ ] generateStaticParams pre-renders products
- [ ] Shared shop layout with header and footer
- [ ] Tailwind polish applied
- [ ] Metadata set per page
- [ ] Sitemap generates at /sitemap.xml
- [ ] AI_AUDIT.md with Server/Client classification current
- [ ] Repo pushed and clean
```

Tick honestly.

**Expected output:**
`CHECKLIST.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Add dark mode support with Tailwind's `dark:` prefix.
- Add `next/font` to self-host a Google font.
- Add `robots.js` alongside `sitemap.js`.

## Submission Requirements

- **What to submit:** Repo, screenshots, `CHECKLIST.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Shop layout wrapper | 20 | Route group. Header/footer on every shop page. |
| Tailwind polish | 25 | Responsive grid and hover effects. Screenshot confirms. |
| Per-page metadata | 20 | At least three pages with their own metadata. |
| Sitemap generation | 15 | /sitemap.xml returns real data. |
| Pre-weekend checklist | 10 | All boxes honest. |
| Clean commits | 10 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Mixing Tailwind with inline styles randomly.** Pick one approach per component.
- **Hardcoding the domain in the sitemap.** Use an env var so it works in dev and prod.
- **Adding metadata to a Client Component.** Metadata exports only work in Server Components.

## Resources

- Day 4 reading: [Layouts Tailwind and SEO.md](./Layouts%20Tailwind%20and%20SEO.md)
- Week 14 AI boundaries: [../ai.md](../ai.md)
- Next.js metadata: https://nextjs.org/docs/app/building-your-application/optimizing/metadata
