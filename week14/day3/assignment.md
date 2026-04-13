# Week 14 - Day 3 Assignment

## Title
Dynamic Routes, Product Pages, And A Small Product Catalogue

## Overview
Today you build the first shop pages: a product list and product detail pages. You will create a `products` table, seed it with real data, and render it via a dynamic route. This is the skeleton your Week 16 e-commerce project will grow from.

## Learning Objectives Assessed
- Create and seed a `products` table in Postgres
- Use dynamic segments (`[slug]`) for detail pages
- Use `generateStaticParams` for pre-rendering
- Handle notFound and metadata per product

## Prerequisites
- Days 1-2 completed

## AI Usage Rules

**Ratio:** 40/60. **Habit:** Framework docs first. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Table rendering boilerplate.
- **NOT ALLOWED FOR:** Schema decisions or naming.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Products schema

**What to do:**
```sql
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL,
  description TEXT,
  price_cents INTEGER NOT NULL,
  image_url TEXT,
  in_stock INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Seed with at least 8 real products (pick a niche: books, coffee, fabric, anything).

**Expected output:**
8+ rows in `products`.

### Task 2: Products listing page

**What to do:**
Create `app/products/page.js`:

```jsx
import pool from "@/lib/db";
import Link from "next/link";

export default async function ProductsPage() {
  const { rows: products } = await pool.query(
    "SELECT id, slug, name, price_cents, image_url FROM products ORDER BY name"
  );

  return (
    <main>
      <h1>All Products</h1>
      <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
        {products.map((p) => (
          <Link key={p.id} href={`/products/${p.slug}`}>
            <div className="border rounded p-4">
              <img src={p.image_url} alt={p.name} className="w-full h-40 object-cover" />
              <h2 className="font-semibold">{p.name}</h2>
              <p>KES {(p.price_cents / 100).toLocaleString()}</p>
            </div>
          </Link>
        ))}
      </div>
    </main>
  );
}
```

**Expected output:**
Grid of products rendered at `/products`.

### Task 3: Dynamic product detail

**What to do:**
Create `app/products/[slug]/page.js`:

```jsx
import pool from "@/lib/db";
import { notFound } from "next/navigation";

export async function generateMetadata({ params }) {
  const { rows } = await pool.query("SELECT name FROM products WHERE slug = $1", [params.slug]);
  const product = rows[0];
  return { title: product ? product.name : "Not found" };
}

export default async function ProductPage({ params }) {
  const { rows } = await pool.query("SELECT * FROM products WHERE slug = $1", [params.slug]);
  const product = rows[0];
  if (!product) notFound();

  return (
    <main>
      <img src={product.image_url} alt={product.name} className="w-full max-w-md" />
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p>KES {(product.price_cents / 100).toLocaleString()}</p>
      <p>{product.in_stock > 0 ? `${product.in_stock} in stock` : "Out of stock"}</p>
    </main>
  );
}
```

Visit a few slugs. Verify metadata updates the browser tab.

**Expected output:**
Each product has its own page. Tab title changes per product.

### Task 4: Pre-render with generateStaticParams

**What to do:**
Add to the same file:

```jsx
export async function generateStaticParams() {
  const { rows } = await pool.query("SELECT slug FROM products");
  return rows.map((row) => ({ slug: row.slug }));
}
```

Build the project: `npm run build`. Watch the output -- it should pre-render one static page per product.

**Expected output:**
Build output shows static routes for each product.

### Task 5: notFound page

**What to do:**
Create `app/products/[slug]/not-found.js`:

```jsx
export default function NotFound() {
  return <main><h1>Product not found</h1></main>;
}
```

Visit `/products/non-existent-slug`. See the not-found page.

**Expected output:**
Friendly not-found page renders.

## Stretch Goals (Optional - Extra Credit)

- Add image optimisation with `next/image` instead of plain `<img>`.
- Add a related-products section below each product.
- Add OpenGraph metadata for social sharing.

## Submission Requirements

- **What to submit:** Repo with products pages, seed data, screenshots.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Products schema + seed | 15 | 8+ real products seeded. |
| Products listing page | 25 | Grid renders with images, names, prices. |
| Dynamic detail page | 25 | Slug routing works. notFound triggers. |
| generateStaticParams | 15 | Build output shows pre-rendered pages. |
| generateMetadata | 10 | Tab title updates per product. |
| Clean commits | 10 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Using `<img>` instead of `next/image` in production.** Fine for now, but the stretch goal teaches the difference.
- **Forgetting to re-query in generateMetadata.** It runs separately from the page function. Both queries are fine; they share the same cache.
- **Slugs with spaces or uppercase.** Use `slug` as URL-safe kebab-case.

## Resources

- Day 3 reading: [Dynamic Routes and Product Pages.md](./Dynamic%20Routes%20and%20Product%20Pages.md)
- Week 14 AI boundaries: [../ai.md](../ai.md)
