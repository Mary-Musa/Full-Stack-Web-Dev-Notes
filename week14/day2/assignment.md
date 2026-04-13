# Week 14 - Day 2 Assignment

## Title
Data Fetching In Server Components -- Real DB, No API Route

## Overview
Today you connect Next.js directly to your Postgres `leads` table and render leads on a Server Component page with no API route in between. This is the superpower of the App Router: when the page IS the server, you can query the database without building a middle-tier.

## Learning Objectives Assessed
- Connect pg directly from a Next.js Server Component
- Query the existing CRM database
- Use `cache: "no-store"` to opt out of caching when needed
- Decide when caching helps and when it hurts

## Prerequisites
- Day 1 completed
- Postgres `crm_dev` still running with data from Week 12

## AI Usage Rules

**Ratio:** 40/60. **Habit:** Framework docs first. See [../ai.md](../ai.md).

- **ALLOWED FOR:** The repetitive parts of the component after you established the query shape.
- **NOT ALLOWED FOR:** Deciding the cache strategy -- that is a design call.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Share the pg pool

**What to do:**
Install `pg` in your Next.js app:

```bash
cd shop-next
npm install pg
```

Create `lib/db.js`:

```javascript
import { Pool } from "pg";

const globalForPg = globalThis;
const pool = globalForPg.__pgPool ?? new Pool({
  host: process.env.PG_HOST,
  port: parseInt(process.env.PG_PORT, 10),
  user: process.env.PG_USER,
  password: process.env.PG_PASSWORD,
  database: process.env.PG_DATABASE,
});
if (process.env.NODE_ENV !== "production") globalForPg.__pgPool = pool;

export default pool;
```

The `globalThis` trick keeps the pool from being re-created on every hot reload in dev.

**Expected output:**
`lib/db.js` in place. Add pg env vars to `.env.local`.

### Task 2: Leads page from Postgres

**What to do:**
Create `app/leads/page.js`:

```jsx
import pool from "@/lib/db";

export default async function LeadsPage() {
  const { rows } = await pool.query(
    "SELECT id, name, wa_phone, status, source FROM leads ORDER BY created_at DESC LIMIT 50"
  );

  return (
    <main>
      <h1>Leads</h1>
      <table>
        <thead>
          <tr><th>Name</th><th>Phone</th><th>Status</th><th>Source</th></tr>
        </thead>
        <tbody>
          {rows.map((lead) => (
            <tr key={lead.id}>
              <td>{lead.name || "--"}</td>
              <td>{lead.wa_phone}</td>
              <td>{lead.status}</td>
              <td>{lead.source}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </main>
  );
}
```

Visit /leads. Real data renders.

**Expected output:**
Leads table renders with real data from Postgres. Screenshot `day2-leads.png`.

### Task 3: Cache behaviour

**What to do:**
By default Next.js 14 caches data fetches. For a CRM where leads update constantly you want fresh data. Add:

```jsx
export const revalidate = 0; // always fresh
// or
export const dynamic = "force-dynamic";
```

In `cache-notes.md`, write 4-6 sentences answering:
- What does `revalidate = 0` mean?
- When would you NOT want fresh data (e.g., a blog post)?
- What is ISR (Incremental Static Regeneration) and when is it the sweet spot?

**Expected output:**
`cache-notes.md` committed.

### Task 4: Dynamic segment for one lead

**What to do:**
Create `app/leads/[id]/page.js`:

```jsx
import pool from "@/lib/db";
import { notFound } from "next/navigation";

export default async function LeadDetailPage({ params }) {
  const { rows } = await pool.query("SELECT * FROM leads WHERE id = $1", [params.id]);
  const lead = rows[0];
  if (!lead) notFound();
  return (
    <main>
      <h1>{lead.name || lead.wa_phone}</h1>
      <p>Status: {lead.status}</p>
      <p>Source: {lead.source}</p>
    </main>
  );
}
```

Visit `/leads/SOME_UUID` for an existing lead.

**Expected output:**
Lead detail renders. Bad IDs trigger the not-found page.

### Task 5: `loading.js` skeleton

**What to do:**
Create `app/leads/loading.js`:

```jsx
export default function Loading() {
  return <p>Loading leads...</p>;
}
```

Delay the query manually with `await new Promise((r) => setTimeout(r, 1500))` to see the skeleton. Remove the delay after.

**Expected output:**
Skeleton visible during slow loads.

## Stretch Goals (Optional - Extra Credit)

- Add a `error.js` to catch rendering errors.
- Add URL search params for filter (`?status=new`) and read them in the page.
- Use `Suspense` to stream parts of the page independently.

## Submission Requirements

- **What to submit:** Repo, `cache-notes.md`, screenshots, `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| pg pool shared correctly | 15 | Hot reload does not crash. |
| Leads page from DB | 25 | Real data rendered. |
| Cache strategy notes | 15 | Three questions answered in student's own words. |
| Dynamic route + notFound | 20 | Lead detail works. Bad ID triggers notFound. |
| loading.js | 10 | Skeleton visible. |
| Clean commits | 10 | Conventional messages. |
| Audit with classification | 5 | Pages classified as Server. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Accidentally shipping a Client Component that queries the DB.** Only Server Components can use `pool.query`. A Client Component trying to import `lib/db` will fail to build.
- **Leaking credentials to the client.** Env variables in `.env.local` stay on the server unless you prefix with `NEXT_PUBLIC_`.
- **Forgetting revalidate on stale data.** The default is aggressive caching.

## Resources

- Day 2 reading: [Data Fetching in Server Components.md](./Data%20Fetching%20in%20Server%20Components.md)
- Week 14 AI boundaries: [../ai.md](../ai.md)
- Next.js data fetching: https://nextjs.org/docs/app/building-your-application/data-fetching
