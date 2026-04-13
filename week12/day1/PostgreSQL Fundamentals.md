# Week 12, Day 1: PostgreSQL Fundamentals

> **AI boundaries this week:** 45% manual / 55% AI. Habit: *Never trust AI with auth -- the second permanent hard rule, alongside Week 10's "never trust AI with money".* See [ai.md](../ai.md).

By the end of today, you will have PostgreSQL installed and running on your laptop, a database called `crm` with the three tables from last week's WhatsApp CRM recreated in proper relational form, and you will be comfortable enough in `psql` to insert a lead, attach a conversation to it, attach messages to the conversation, and pull the whole thing back with a single `JOIN` query. You will not touch Node today. Today is about the database itself -- next to SQL, everything else is plumbing.

This is the start of a reconciliation week. Weeks 7-11 were heavy on React and real integrations (M-Pesa, WhatsApp), but we skipped two pieces that the rest of the Marathon depends on: a real relational database, and real authentication. Week 12 backfills both. Friday you will hand in a Week 11 CRM that has been migrated to Postgres, has users who log in, and where one sales agent cannot see another agent's leads. Without the backfill the payments reconciliation work in Week 17 and the multi-tenant capstone in Weeks 27-30 would collapse, so treat this week as load-bearing -- not review.

**Prior-week concepts you will use today:**
- SQL basics: `CREATE TABLE`, `INSERT`, `SELECT`, `FOREIGN KEY` (Week 10, Day 1 and Week 11, Day 1)
- The three-table schema from the Week 11 CRM: `leads`, `conversations`, `messages` (Week 11, Day 1)
- Shell comfort: navigating directories, running commands, editing files (Weeks 1-2)
- Environment variables and `.env` discipline (Week 10, Day 1)

**Estimated time:** 3-4 hours

---

## The Week Ahead

| Day | What You Build |
|---|---|
| Day 1 (today) | Install Postgres, learn `psql`, rebuild the Week 11 CRM schema relationally, practise real SQL. |
| Day 2 | Connect Node to Postgres with the `pg` driver. Refactor the Week 11 server into controllers, services, and a db layer. Centralised error handling. |
| Day 3 | Users table, signup and login routes, bcrypt password hashing, JWT issuance, `requireAuth` middleware protecting the CRM API. |
| Day 4 | Multi-user CRM -- per-user lead ownership, admin vs agent roles, "assigned to" filters on the dashboard. Short Mongo/Mongoose aside. |
| Day 5 | Week recap, Phase 2 checklist, Friday peer coding session. |
| Weekend | **Lock It Down** -- migrate your Boda Dispatch CRM from Week 11 to Postgres + auth, with two user roles. |

Each day depends on the one before it. Do not touch Day 2 until today's checkpoint passes.

---

## Why Postgres, Why Now

You have been shipping real software for three weeks using SQLite. SQLite is wonderful -- it is one file on disk, has zero moving parts, and the queries you write against it are almost identical to the queries you will write against Postgres. So why change?

Three reasons, in order of importance:

**1. Concurrency.** SQLite allows exactly one writer at a time. For the Week 11 CRM, running on your laptop, with one WhatsApp bot and one dashboard user, that is fine. But the moment you have ten sales agents refreshing the dashboard while a hundred leads are coming in through WhatsApp in a Saturday morning rush, SQLite starts locking up. Postgres handles hundreds of concurrent writers without thinking about it. Every production system you build from Week 13 onwards assumes a real database.

**2. Types and constraints.** SQLite is dynamically typed: if you declare a column `INTEGER` and insert the string `"hello"`, SQLite stores `"hello"`. Postgres refuses the insert. This sounds annoying until you spend an evening debugging why your `amount` column has two numbers and a receipt URL in it. Real types catch bugs at write time, not at read time three weeks later.

**3. It is the industry standard for the stack we are building.** The Week 17 payments reconciliation module, the Week 23 BullMQ queue state, and the multi-tenant capstone in Weeks 27-30 are all designed around Postgres features (proper transactions, `JSONB`, row-level security, `UNIQUE` constraints on composite keys). If you learn Postgres now, everything that follows will click.

There is a fourth reason nobody writes down: hiring. Every Nairobi backend job listing says "Postgres". Nobody advertises SQLite skills.

### What about MongoDB?

The original Marathon roadmap listed MongoDB + Mongoose in Phase 2. We are deliberately not teaching it hands-on this week. MongoDB is a fine database for some problems, but for the problems this Marathon is building -- CRMs, payments, multi-tenant SaaS -- relational integrity matters more than schemaless flexibility, and every day of Mongo practice is a day not spent getting comfortable in SQL. On Day 4 you will spend forty-five minutes understanding what Mongo is, when you would pick it, and reading a Mongoose schema side by side with the Postgres one you built this week. That is enough. If a future project needs Mongo, you will pick it up in an afternoon.

---

## Installing PostgreSQL

Pick the path for your operating system. I will assume Ubuntu/Debian Linux (what most of the cohort is on). Mac and Windows notes are below.

### Ubuntu / Debian / WSL

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```

When it finishes, Postgres is running as a background service and has created a Linux user called `postgres`. That user owns a database superuser also called `postgres`. Verify it is running:

```bash
sudo systemctl status postgresql
```

You should see `active (exited)` or `active (running)`. If not:

```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql   # start on boot
```

### Mac (Homebrew)

```bash
brew install postgresql@16
brew services start postgresql@16
```

Homebrew creates a database superuser with your Mac username, not `postgres`. Keep that in mind when you see me type `sudo -u postgres psql` below -- on Mac you just type `psql`.

### Windows

Download the installer from https://www.postgresql.org/download/windows/ and run it. During setup it will ask for a password for the `postgres` superuser -- write it down. Accept the default port (5432). After install, open "SQL Shell (psql)" from the start menu and log in with the password you set.

### Verifying the install

Regardless of OS, run:

```bash
psql --version
```

You should see something like `psql (PSQL) 16.1`. Any version 14 or higher is fine for this course.

---

## Meet `psql`

`psql` is the command-line client for Postgres. It is to Postgres what `node` (the REPL) is to JavaScript -- a place to type a command and see a result. Every GUI database tool (DBeaver, Postico, TablePlus, pgAdmin) is a pretty wrapper around what `psql` does in plain text. Learn `psql` first, install a GUI later if you want one.

On Linux, log in as the `postgres` superuser:

```bash
sudo -u postgres psql
```

On Mac/Homebrew:

```bash
psql postgres
```

Your prompt changes to:

```
postgres=#
```

The `#` means "you are a superuser, you can do anything". Be careful in here. The first thing to notice is that `psql` is *both* a SQL prompt and a meta-command prompt. SQL commands end with a semicolon. Meta-commands start with a backslash and do not take a semicolon.

Try these meta-commands right now:

```
\l        -- list all databases
\du       -- list all users (Postgres calls them "roles")
\conninfo -- show how you are connected
\q        -- quit
```

Those four will get you out of 80 percent of situations. Write them somewhere you will remember.

---

## Users, Databases, and the Mental Model

Before you create anything, understand how Postgres organises the world. This trips up everyone coming from SQLite.

- A **cluster** is one running Postgres server. You installed one cluster.
- A cluster has many **databases**. One cluster typically runs one database per project.
- A cluster has many **roles** (users). Roles can own databases and have permissions.
- Inside a database are **schemas** (usually just one, called `public`), and inside schemas are **tables**.

In SQLite you pointed at a file and that was your database. In Postgres you point at a *host, port, user, and database name*. That is four pieces of connection information instead of one. This is the price of a real multi-user system.

### Creating a user and a database for the CRM

Staying inside `psql` as the superuser:

```sql
CREATE USER crm_user WITH PASSWORD 'crm_dev_password';
CREATE DATABASE crm OWNER crm_user;
GRANT ALL PRIVILEGES ON DATABASE crm TO crm_user;
```

Three statements. Read them out loud:

1. Make a new user called `crm_user` with this password.
2. Make a new database called `crm` owned by that user.
3. Give that user every right on that database.

**Never use `crm_dev_password` in production.** Today you are on your laptop and the database is not exposed to the internet. A weak password is fine. When you deploy in Week 26 we will change it.

Exit the superuser session and log in as the new user directly against the new database:

```bash
\q
psql -U crm_user -d crm -h localhost
```

If it asks for a password, type `crm_dev_password`. Your prompt changes to:

```
crm=>
```

The `=>` (instead of `=#`) tells you that you are now a regular user, not a superuser. You cannot accidentally drop the whole cluster from here. Good. This is the account your Node app will use tomorrow.

> **Trouble?** If the login is refused with `peer authentication failed`, edit `/etc/postgresql/16/main/pg_hba.conf` and change the line that starts with `local all all peer` to `local all all md5`, then `sudo systemctl restart postgresql`. On Mac/Homebrew you will not hit this.

---

## The Week 11 Schema, Rebuilt in Postgres

Last week your three tables looked like this in SQLite:

```sql
CREATE TABLE leads (
  id TEXT PRIMARY KEY,
  wa_phone TEXT NOT NULL UNIQUE,
  ...
  created_at TEXT DEFAULT (datetime('now'))
);
```

Postgres has real types. Let us use them.

Inside the `crm=>` prompt, create `leads`:

```sql
CREATE TABLE leads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  wa_phone TEXT NOT NULL UNIQUE,
  name TEXT,
  email TEXT,
  inquiry_type TEXT,
  status TEXT NOT NULL DEFAULT 'new'
    CHECK (status IN ('new', 'contacted', 'qualified', 'converted', 'lost')),
  notes TEXT,
  assigned_to TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Four things changed from the SQLite version. Each one is worth understanding.

**`UUID` with `gen_random_uuid()` instead of `TEXT` + `uuid.v4()` in Node.** Postgres can generate UUIDs itself. You no longer need to import the `uuid` package just to get a primary key. `gen_random_uuid()` is built into Postgres 13+ (no extension needed since v13) and runs at insert time on the server. One fewer thing for your Node code to worry about.

**`TIMESTAMPTZ` instead of `TEXT DEFAULT (datetime('now'))`.** `TIMESTAMPTZ` is "timestamp with time zone". It stores a real instant in time, not a string, and Postgres will hand it back to your Node code as a JavaScript `Date`. No more parsing `"2026-04-12 10:23:14"` strings. The `TZ` part matters because your Nairobi server, your UTC production database, and your customer's phone in Mombasa might all disagree about what "now" means, and `TIMESTAMPTZ` stores the right answer.

**`CHECK (status IN (...))` constraint.** In Week 11 we said "we enforce the allowed values in the Express route before writing". That works until someone writes a second route and forgets. A `CHECK` constraint enforces it in the database. Try inserting a lead with `status = 'pending'` later today -- Postgres will refuse, with a clear error message. The database becomes the last line of defence, not your JavaScript.

**`NOT NULL` on `status`, `created_at`, `updated_at`.** SQLite was permissive about NULLs in columns with defaults. Postgres lets you be strict. "A lead must have a status" is a rule your schema can now enforce.

Now `conversations`:

```sql
CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_id UUID NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
  state TEXT NOT NULL DEFAULT 'awaiting_name',
  last_message_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

The new thing here is `REFERENCES leads(id) ON DELETE CASCADE`. `REFERENCES` is the same foreign key you wrote last week. `ON DELETE CASCADE` is new: it says "if a lead row is deleted, delete its conversation row automatically". SQLite supports this too, but you have to enable foreign keys with a pragma. Postgres enforces them by default, always. One less footgun.

And `messages`:

```sql
CREATE TABLE messages (
  id BIGSERIAL PRIMARY KEY,
  lead_id UUID NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
  direction TEXT NOT NULL CHECK (direction IN ('in', 'out')),
  body TEXT,
  raw_payload JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Two Postgres-specific things here:

**`BIGSERIAL` is an auto-incrementing 64-bit integer.** For a `messages` table that might accumulate millions of rows over a year, a regular `SERIAL` (32-bit) could theoretically run out. `BIGSERIAL` is the "I will not think about this again" choice.

**`JSONB` for `raw_payload` instead of `TEXT`.** Last week you stringified the Meta webhook payload with `JSON.stringify()` and stored it as `TEXT`. That works, but reading it required `JSON.parse()` every time, and you could not query inside it. `JSONB` stores the JSON in a binary format that Postgres can index and query. Later in the Marathon you will write a query like "find all messages where the raw payload's `type` field is `'image'`" without ever parsing on the Node side. Today, just know the column is there and it is stronger than a text column.

### Indexes

One more thing while you are here -- indexes. The Week 11 dashboard searches leads by name and filters by status. Every search is a `SELECT ... WHERE name ILIKE '%something%'`. On a table with ten leads that runs in a millisecond. On a table with ten thousand leads it crawls, because Postgres has to read every row. An **index** is a secondary data structure the database maintains that makes specific lookups fast.

```sql
CREATE INDEX idx_leads_status ON leads(status);
CREATE INDEX idx_leads_created_at ON leads(created_at DESC);
CREATE INDEX idx_messages_lead_id ON messages(lead_id);
```

Three indexes. Each one answers a specific question the dashboard asks:

- "Show me all `new` leads" -- `idx_leads_status`.
- "Sort leads newest first" -- `idx_leads_created_at`.
- "Load all messages for this lead" -- `idx_messages_lead_id`.

Rule of thumb for this course: index foreign key columns, index columns you filter by, index columns you sort by. Do not index everything -- indexes cost disk space and slow down writes. Three targeted indexes on a CRM of this size is the right amount.

### Check your work

Still inside `psql`:

```
\dt             -- list tables
\d leads        -- describe the leads table
\d conversations
\d messages
```

You should see all three tables with the columns, types, and constraints you just created. `\d leads` will show you the indexes at the bottom.

---

## Practising Real SQL

Now for the part the textbooks skip. You know `INSERT` and `SELECT` syntax, but SQL is not about syntax -- it is about thinking in sets. Let us put some realistic data in the database and then query it the way the dashboard will tomorrow.

### Inserting

```sql
INSERT INTO leads (wa_phone, name, email, inquiry_type)
VALUES ('+254712000001', 'Wanjiru Kamau', 'wanjiru@example.com', '3-bedroom Kilimani');
```

Notice what you did *not* write: you did not write `id`, you did not write `status`, you did not write `created_at`. Postgres filled those in from your defaults. Run `SELECT * FROM leads;` and you will see a full row.

Add a few more so the queries below are meaningful:

```sql
INSERT INTO leads (wa_phone, name, email, inquiry_type, status) VALUES
  ('+254712000002', 'Brian Otieno', 'brian@example.com', 'Land in Kitengela', 'contacted'),
  ('+254712000003', 'Amina Said', NULL, 'Car loan', 'qualified'),
  ('+254712000004', 'Peter Mwangi', 'peter@example.com', 'Insurance quote', 'new'),
  ('+254712000005', 'Grace Achieng', 'grace@example.com', '2-bedroom Westlands', 'lost');
```

Five leads, five different situations. The `NULL` for Amina is intentional -- not every lead gives you their email, and your schema allows it.

Now attach a conversation to the first lead, and a message to it. Because we need Wanjiru's `id` and we did not write it down, use a subquery:

```sql
INSERT INTO conversations (lead_id, state)
SELECT id, 'awaiting_email' FROM leads WHERE wa_phone = '+254712000001';

INSERT INTO messages (lead_id, direction, body, raw_payload)
SELECT id, 'in', 'Hi, I am interested in the Kilimani apartment', '{"type":"text"}'
FROM leads WHERE wa_phone = '+254712000001';
```

This is a pattern you will use a lot: insert into table B a row whose foreign key comes from a `SELECT` against table A. It saves you from round-tripping through your application to fetch the id.

### Selecting

Run the queries the dashboard will need tomorrow:

**All leads, newest first:**

```sql
SELECT id, name, wa_phone, status, created_at
FROM leads
ORDER BY created_at DESC;
```

**Only leads with status `new`:**

```sql
SELECT * FROM leads WHERE status = 'new';
```

**Search by name, case-insensitive:**

```sql
SELECT * FROM leads WHERE name ILIKE '%wanjir%';
```

`ILIKE` is `LIKE` but case-insensitive. It is a Postgres extension -- most other databases force you to write `LOWER(name) LIKE LOWER('%...%')`. `%` is the wildcard. Note that a leading `%` means Postgres cannot use an index on `name` for this query -- for a small CRM this is fine; for a search engine you would reach for a full-text index, which is Week 19 territory.

**Count leads by status -- the dashboard stats card:**

```sql
SELECT status, COUNT(*) AS total
FROM leads
GROUP BY status
ORDER BY total DESC;
```

`GROUP BY` collapses rows that share a value, and aggregate functions like `COUNT`, `SUM`, `AVG` operate on each group. This one query replaces the five separate SELECTs you might otherwise write.

### Joining

The payoff query. Get every lead along with their most recent message body:

```sql
SELECT
  l.name,
  l.wa_phone,
  l.status,
  m.body AS last_message,
  m.created_at AS last_message_at
FROM leads l
LEFT JOIN LATERAL (
  SELECT body, created_at
  FROM messages
  WHERE lead_id = l.id
  ORDER BY created_at DESC
  LIMIT 1
) m ON true
ORDER BY l.created_at DESC;
```

There is a lot going on here -- do not panic, you will not need to write a query this clever until Week 19. Read it piece by piece.

- `FROM leads l` -- the `l` is a short alias so we can write `l.name` instead of `leads.name`.
- `LEFT JOIN` -- include every lead even if they have no messages (inner join would drop leads without messages).
- `LATERAL (...)` -- for each lead row, run the subquery with `l.id` in scope, get the latest message.
- `ORDER BY created_at DESC LIMIT 1` inside -- "the most recent message".

The simpler, cheaper join you will write 95 percent of the time is this one:

```sql
SELECT l.name, l.wa_phone, m.direction, m.body, m.created_at
FROM leads l
INNER JOIN messages m ON m.lead_id = l.id
WHERE l.wa_phone = '+254712000001'
ORDER BY m.created_at;
```

"Give me everything Wanjiru has ever said or been told, in order." That is the query behind the lead detail page on the dashboard, and it is the query you will port to Node on Day 2.

### Updating and deleting

```sql
UPDATE leads SET status = 'contacted', updated_at = NOW()
WHERE wa_phone = '+254712000001';

DELETE FROM leads WHERE wa_phone = '+254712000005';
```

Two warnings:

1. **Always write the `WHERE` clause first.** If you type `DELETE FROM leads` and forget the `WHERE`, you delete every lead. I have done it. You will do it. Get in the habit of typing the `WHERE` before you type the `UPDATE` or `DELETE` keyword.
2. **`updated_at = NOW()`** is manual for now. Day 2 we will add a trigger that does it automatically. For today, remember it.

Notice what happened to Grace's conversation and messages when you deleted her? Nothing, because we did not give her any. But if we had, `ON DELETE CASCADE` on the foreign key would have removed them too. Try it: add a conversation to Peter, delete Peter, check `conversations` -- the conversation is gone.

---

## Transactions, Briefly

One last concept today. A transaction is a group of statements that either all succeed or all fail. You wrap them in `BEGIN` and `COMMIT`:

```sql
BEGIN;
INSERT INTO leads (wa_phone, name) VALUES ('+254712999999', 'Test User');
INSERT INTO conversations (lead_id, state)
  SELECT id, 'awaiting_name' FROM leads WHERE wa_phone = '+254712999999';
COMMIT;
```

If the second `INSERT` blows up (say, `lead_id` was null), you can type `ROLLBACK;` instead of `COMMIT;` and the first `INSERT` is undone -- as if neither had happened. This is the property SQLite gives you inside a single `db.exec()` call, but Postgres lets you span multiple statements over multiple round trips. In Week 17 when you reconcile payments you will lean on this hard -- "debit the sender and credit the receiver" must either both happen or neither.

Transactions do not matter for today's checkpoint; just know the words `BEGIN`, `COMMIT`, `ROLLBACK` exist and that they group statements.

---

## Checkpoint

Before you log off, prove each of these works from a fresh `psql -U crm_user -d crm -h localhost` session. If any one fails, fix it now -- tomorrow's Node code assumes all of them pass.

1. `\dt` lists exactly three tables: `leads`, `conversations`, `messages`.
2. `\d leads` shows the `CHECK` constraint on `status`.
3. This insert **succeeds:**
   ```sql
   INSERT INTO leads (wa_phone, name, status) VALUES ('+254700111222', 'Check User', 'new');
   ```
4. This insert **fails** with a check-constraint error:
   ```sql
   INSERT INTO leads (wa_phone, name, status) VALUES ('+254700111223', 'Bad User', 'pending');
   ```
5. This insert **fails** with a unique-violation error (you already used that phone above):
   ```sql
   INSERT INTO leads (wa_phone, name) VALUES ('+254700111222', 'Duplicate');
   ```
6. `SELECT status, COUNT(*) FROM leads GROUP BY status;` returns at least two rows.
7. Your lead-plus-messages join query for `+254712000001` returns the message row you inserted earlier.
8. `SELECT gen_random_uuid();` returns a UUID (proves the function is available without installing the old `uuid-ossp` extension).

Once every bullet passes, commit a short note to your project repo:

```bash
mkdir -p whatsapp-crm/db
# inside that folder, save the SQL you wrote today as schema.sql
git add db/schema.sql
git commit -m "docs: week 12 day 1 postgres schema"
```

Saving the schema as a file (not just typing it into `psql`) matters -- tomorrow's Node app will load it, and by Week 13 you will have a proper migration workflow. Start the habit now.

---

## What To Read Before Tomorrow

- The `pg` package on npm: skim the README at https://node-postgres.com. Focus on the "Pools" section. That is what you will use tomorrow.
- Ten minutes on SQL joins visualised -- any of the "SQL JOIN Venn diagram" articles that come up first on Google. Inner vs left vs right vs full.
- Optional: `EXPLAIN` -- run `EXPLAIN SELECT * FROM leads WHERE status = 'new';` in `psql`. It prints the query plan. You do not need to understand it yet; it is here so you know the feature exists when you need it.

Tomorrow we build the Express + pg layer and refactor the Week 11 server into controllers, services, and a db folder. Bring your running database -- everything we do depends on today's schema being in place.
