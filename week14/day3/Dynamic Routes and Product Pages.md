# Week 14, Day 3: Dynamic Routes and Product Pages

By the end of today, every product in your database has its own detail page at `/products/[slug]` with real data, a proper page title, an Open Graph image for WhatsApp/Instagram sharing previews, a "Related products" section powered by a second query, and a `notFound()` fallback when someone hits a bad slug. You will also understand the `generateMetadata` and `generateStaticParams` hooks that are unique to the App Router.

**Prior-week concepts you will use today:**
- Server Components fetching directly from Postgres (Week 14, Day 2)
- `<Link>` and file-based routing (Week 14, Day 1)
- SQL with prepared statements (Week 12)
- The Week 14 Day 2 `products` table with the `slug` column

**Estimated time:** 3 hours

---

## Dynamic Segments

A folder name wrapped in square brackets is a dynamic segment. Create `app/products/[slug]/page.js`:

```jsx
// app/products/[slug]/page.js
export default function ProductPage({ params }) {
  return <div>Product: {params.slug}</div>;
}
```

Visit `/products/nokia-105` -- it says "Product: nokia-105". Visit `/products/tecno-spark-10` -- "Product: tecno-spark-10". The segment value comes in through `params`.

That is the whole mechanism. The file `[slug]/page.js` matches *any* URL segment at that position and gives you the value as `params.slug`. A route with two dynamic segments (e.g. `[category]/[slug]/page.js`) gives you both.

---

## Fetching One Product

Replace the placeholder with a real query:

```jsx
// app/products/[slug]/page.js
import { notFound } from "next/navigation";
import Image from "next/image";
import Link from "next/link";
import { query } from "@/lib/db";
import AddToCartButton from "@/app/components/AddToCartButton";

export default async function ProductPage({ params }) {
  const { rows } = await query(
    `SELECT id, slug, name, description, price_cents, image_url, stock, category
     FROM products WHERE slug = $1`,
    [params.slug]
  );

  const product = rows[0];
  if (!product) notFound();

  const { rows: related } = await query(
    `SELECT slug, name, price_cents FROM products
     WHERE category = $1 AND slug != $2
     ORDER BY random()
     LIMIT 3`,
    [product.category, product.slug]
  );

  return (
    <div className="max-w-5xl mx-auto p-8">
      <nav className="text-sm text-gray-500 mb-6">
        <Link href="/">Home</Link> / <Link href="/products">Products</Link> / {product.name}
      </nav>

      <div className="grid md:grid-cols-2 gap-8">
        <div className="aspect-square bg-gray-100">
          {product.image_url && (
            <Image
              src={product.image_url}
              alt={product.name}
              width={800}
              height={800}
              className="w-full h-full object-cover"
            />
          )}
        </div>

        <div>
          <h1 className="text-3xl font-bold">{product.name}</h1>
          <p className="text-2xl mt-2">KSh {(product.price_cents / 100).toLocaleString()}</p>

          {product.stock > 0 ? (
            <p className="text-green-700 text-sm mt-2">In stock ({product.stock} available)</p>
          ) : (
            <p className="text-red-600 text-sm mt-2">Out of stock</p>
          )}

          <p className="mt-6 text-gray-700">{product.description}</p>

          <div className="mt-6">
            <AddToCartButton product={product} />
          </div>
        </div>
      </div>

      {related.length > 0 && (
        <section className="mt-16">
          <h2 className="text-xl font-bold mb-4">Related products</h2>
          <div className="grid grid-cols-3 gap-4">
            {related.map((r) => (
              <Link key={r.slug} href={`/products/${r.slug}`} className="border p-4 rounded hover:shadow">
                <div className="font-medium">{r.name}</div>
                <div className="text-sm text-gray-600">
                  KSh {(r.price_cents / 100).toLocaleString()}
                </div>
              </Link>
            ))}
          </div>
        </section>
      )}
    </div>
  );
}
```

Four things to notice.

**`notFound()`** from `next/navigation` is a special function that throws a control-flow error Next.js catches and renders as a 404 page. The rest of the function never runs. If you want to customise the 404, create `app/products/[slug]/not-found.js` with a component.

**Two queries back-to-back.** The second query runs after the first because we need `product.category`. If they were independent, you could run them in parallel with `Promise.all`:

```javascript
const [productResult, featuredResult] = await Promise.all([
  query("SELECT * FROM products WHERE slug = $1", [params.slug]),
  query("SELECT * FROM products WHERE featured = true LIMIT 3"),
]);
```

Get the habit. Every sequential `await` on independent data is wasted latency.

**`AddToCartButton` is a Client Component.** We import and use it, but it lives in `app/components/AddToCartButton.js` with `"use client"` at the top. Next.js passes the `product` object as a prop from server to client automatically -- you do not have to serialise anything.

Create that button now:

```jsx
// app/components/AddToCartButton.js
"use client";
import { useState } from "react";

export default function AddToCartButton({ product }) {
  const [added, setAdded] = useState(false);

  async function handleClick() {
    // Real cart logic is Week 15. For today, a placeholder.
    setAdded(true);
    setTimeout(() => setAdded(false), 2000);
  }

  return (
    <button
      onClick={handleClick}
      disabled={product.stock === 0}
      className="bg-black text-white px-6 py-3 rounded disabled:opacity-50"
    >
      {product.stock === 0 ? "Out of stock" : added ? "Added!" : "Add to cart"}
    </button>
  );
}
```

Week 15 we replace the placeholder with a real cart state machine.

---

## Dynamic Metadata

Sharing a product on WhatsApp sends a preview card: an image, a title, a short description. Those come from Open Graph meta tags in the HTML. In Next.js App Router, you export a `generateMetadata` function:

```jsx
// app/products/[slug]/page.js (add near the top, after imports)

export async function generateMetadata({ params }) {
  const { rows } = await query(
    "SELECT name, description, image_url, price_cents FROM products WHERE slug = $1",
    [params.slug]
  );
  const product = rows[0];
  if (!product) return { title: "Not found" };

  return {
    title: `${product.name} | Mctaba Shop`,
    description: product.description?.slice(0, 160) || "Available at Mctaba.",
    openGraph: {
      title: product.name,
      description: `KSh ${(product.price_cents / 100).toLocaleString()} - ${product.description}`,
      images: product.image_url ? [product.image_url] : [],
    },
  };
}
```

`generateMetadata` runs on the server before the page renders. Its return value becomes the `<head>` of the page. Paste a product URL into WhatsApp and you will see the title, description, and image as a preview card.

### The two queries concern

Notice `generateMetadata` and `ProductPage` both query for the same product. Is that wasteful? No -- Next.js deduplicates identical fetches within a single request. If you call `query` with the exact same SQL and params twice, only one round-trip happens. This is why the pattern "fetch in metadata, fetch in page" is not a bug.

(For `pg` direct queries rather than `fetch`, the dedup does not happen automatically -- you would have to wire `React.cache()` manually, which is a Day 4 topic. For now, accept the two queries; they are small and indexed.)

---

## `generateStaticParams` For SEO

If you know the slugs ahead of time, you can tell Next.js to **pre-render** the pages at build time instead of on every request. Add this function to the same file:

```jsx
export async function generateStaticParams() {
  const { rows } = await query("SELECT slug FROM products");
  return rows.map((p) => ({ slug: p.slug }));
}
```

Now when you run `npm run build`, Next.js loops over every slug, runs the page with those params, and emits static HTML files. Visits to `/products/nokia-105` are served from the CDN edge with zero database queries in production.

This is a huge win for catalogue pages that do not change on every request. The trade-off is that the pre-rendered pages are stale until the next build. For stock counts and prices that change often, mix it with `export const revalidate = 60` to regenerate every 60 seconds on demand:

```javascript
export const revalidate = 60;
```

The combination -- pre-render at build, revalidate every 60s on demand -- is called **Incremental Static Regeneration (ISR)**. It is Next.js's signature feature and it covers 90% of e-commerce catalogue needs.

### When not to use `generateStaticParams`

- When slugs are unknown at build time (e.g. user-generated content in very large quantities).
- When the content is personalised per visitor (e.g. recommended products based on browsing history).

For a shop with dozens or hundreds of products, use it. For a shop with millions, let Next.js generate on-demand with `dynamic = 'force-dynamic'`.

---

## The 404 Page

Create `app/products/[slug]/not-found.js`:

```jsx
// app/products/[slug]/not-found.js
import Link from "next/link";

export default function NotFound() {
  return (
    <div className="max-w-xl mx-auto p-16 text-center">
      <h1 className="text-3xl font-bold">Product not found</h1>
      <p className="mt-4 text-gray-600">
        We could not find the product you were looking for. It may have been sold or removed.
      </p>
      <Link href="/products" className="mt-6 inline-block bg-black text-white px-6 py-3 rounded">
        Browse all products
      </Link>
    </div>
  );
}
```

Now `/products/invalid-slug` shows this page instead of a generic 404. The error boundary is automatic -- `notFound()` throws, Next.js catches it, renders `not-found.js`.

---

## A Catalogue Navigation Page

Most e-commerce sites let users browse by category. Create `app/products/category/[cat]/page.js`:

```jsx
import Link from "next/link";
import { notFound } from "next/navigation";
import { query } from "@/lib/db";

export default async function CategoryPage({ params }) {
  const { rows: products } = await query(
    "SELECT slug, name, price_cents FROM products WHERE category = $1 ORDER BY name",
    [params.cat]
  );

  if (products.length === 0) notFound();

  return (
    <div className="max-w-5xl mx-auto p-8">
      <h1 className="text-3xl font-bold capitalize mb-6">{params.cat}</h1>
      <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
        {products.map((p) => (
          <Link key={p.slug} href={`/products/${p.slug}`} className="border p-4 rounded hover:shadow">
            <div className="font-medium">{p.name}</div>
            <div className="text-sm text-gray-600">
              KSh {(p.price_cents / 100).toLocaleString()}
            </div>
          </Link>
        ))}
      </div>
    </div>
  );
}

export async function generateStaticParams() {
  const { rows } = await query("SELECT DISTINCT category FROM products WHERE category IS NOT NULL");
  return rows.map((r) => ({ cat: r.category }));
}
```

Visit `/products/category/phones`. All your phones. Visit `/products/category/accessories`. Your power bank. Visit `/products/category/tvs` -- the 404 page from above.

### A search page

While we are on the topic, add a search page. Create `app/search/page.js`:

```jsx
import Link from "next/link";
import { query } from "@/lib/db";

export default async function SearchPage({ searchParams }) {
  const q = searchParams.q || "";

  let results = [];
  if (q.length >= 2) {
    const { rows } = await query(
      `SELECT slug, name, price_cents FROM products
       WHERE name ILIKE $1 OR description ILIKE $1
       ORDER BY name LIMIT 20`,
      [`%${q}%`]
    );
    results = rows;
  }

  return (
    <div className="max-w-3xl mx-auto p-8">
      <h1 className="text-3xl font-bold mb-6">Search</h1>
      <form action="/search" method="GET">
        <input
          type="search"
          name="q"
          defaultValue={q}
          placeholder="Search products..."
          className="w-full border p-3 rounded"
        />
      </form>

      {q && (
        <div className="mt-6">
          <p className="text-gray-500 mb-4">{results.length} result(s) for "{q}"</p>
          <ul className="space-y-2">
            {results.map((p) => (
              <li key={p.slug}>
                <Link href={`/products/${p.slug}`} className="hover:underline">
                  {p.name} - KSh {(p.price_cents / 100).toLocaleString()}
                </Link>
              </li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
}
```

Two things to notice.

**`searchParams`** is a prop Next.js passes to page components. It is an object with the parsed URL query string. `?q=phone` becomes `searchParams.q === 'phone'`. No client-side JavaScript needed for basic search -- the form submits, the page re-renders with new params, the server queries the database.

**The form uses `method="GET"`** and no JavaScript handler. It is the browser's built-in submit. This is a Server Component pattern -- lean on the browser's native form submission whenever you can, and your page will work without JavaScript at all. That is called "progressive enhancement" and it is a good habit for commerce sites (a customer on a flaky connection still gets search).

---

## Checkpoint

1. `/products/nokia-105` shows the Nokia 105 detail page with image, price, description, and an "Add to cart" button.
2. `/products/does-not-exist` shows the custom 404 page.
3. Pasting the product URL into WhatsApp (or using an Open Graph tester like https://www.opengraph.xyz/) shows a preview card with name, price, and image.
4. `/products/category/phones` lists only the phones.
5. `/search?q=nokia` returns the Nokia 105.
6. Running `npm run build` pre-renders pages for every product without errors. Check the output -- you should see `/products/nokia-105`, `/products/tecno-spark-10`, etc.
7. Changing a product in `psql` and running `npm run build` again shows the new data. Without `revalidate` set, live changes will not reflect until rebuild.
8. `Add to cart` button is disabled on out-of-stock products.

Commit:

```bash
git add .
git commit -m "feat: dynamic product pages with metadata and static params"
```

---

## What You Learned

- `[slug]` folders become dynamic routes; the value arrives as `params.slug`.
- `generateMetadata` runs on the server to set `<head>` for each page.
- `generateStaticParams` pre-renders known routes at build time.
- `notFound()` is a clean way to serve 404s from inside a data fetch.
- `searchParams` gives you query string parsing for free.
- Native HTML forms with `method="GET"` are the simplest, most reliable search UI.

Tomorrow we style with Tailwind, add layouts, and polish for real SEO -- Open Graph images generated at runtime, a sitemap, robots.txt, and a branded header. Friday is the recap and the weekend is a catalogue MVP.
