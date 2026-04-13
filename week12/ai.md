# Week 12 - AI Boundaries

**Ratio this week: 45% Manual / 55% AI**
**Habit introduced: "Never trust AI with auth."**
**Shift from last week: Same ratio, second hard security rule. Week 10 was "never trust AI with money". Week 12 is "never trust AI with auth". These two rules are permanent.**

This is a reconciliation week. You back-fill four things the programme did not teach hands-on earlier: PostgreSQL (replacing SQLite), clean server architecture (controllers/services/repositories), JWT authentication with password hashing, and multi-user ownership with roles. Every one of them was a topic where going too fast would hurt the Capstone later. This week catches up.

The new habit: auth code is manual-first, manual-audit, manual-review. AI can scaffold the signup form HTML. AI cannot write your password hashing, your JWT signing, your token validation, or your access-control middleware. Every one of those lines is yours.

---

## Why The Ratio Held But The Habit Hardened

45/55 held from Week 11 because most of this week's work is patterns you have seen before, applied in a new context. Postgres queries are similar in shape to the SQLite queries you wrote in Week 11. Layered architecture is a refactor of things you already built. AI can help with both.

But auth is where real users get hurt. A missing bcrypt. A weak JWT secret. A token that never expires. A middleware that forgets to check the token. Each one is a real incident. Each one is easy to accidentally ship when AI writes fast and you trust fast. So this week the second hard rule activates: **auth lives on the manual side, forever.**

You now have two permanent hard rules:

1. Week 10: **Never trust AI with money.**
2. Week 12: **Never trust AI with auth.**

Both stay active for the rest of the programme.

---

## What You Will Feel This Week

- Postgres will feel more serious than SQLite. It has a real server, real roles, real connection strings. That seriousness is the point.
- Writing `bcrypt.hash()` and `bcrypt.compare()` yourself the first time will feel disproportionately important. It is.
- The `requireAuth` middleware will feel like magic once it is working -- one line on a route makes it private.
- Refactoring Week 11's CRM into a clean layered architecture will be tedious and valuable. You will look at the "before" and laugh at how tangled it was.

---

## What You MUST Do Manually (auth side is 100% manual, data side is 40%)

### Day 1 -- PostgreSQL fundamentals
- Install Postgres yourself. Create a `crm` database. Create a user with limited privileges (not the superuser) and connect as that user.
- Write every migration by hand: `leads`, `conversations`, `messages`, matching the SQLite schema from Week 11 but using proper Postgres types (UUID, TIMESTAMPTZ, TEXT with CHECK constraints).
- Migrate the data from Week 11's SQLite into Postgres. Write a small script if you want, but run the migration yourself.
- Verify every table with `\d tablename` in `psql`. Read your own schema.

### Day 2 -- Node + Postgres with clean architecture
- Install `pg` manually. Read the `pg` README.
- Build a connection pool, export it, use it from every repository module.
- Refactor Week 11's server into three layers:
  - **Routes** (thin, no business logic)
  - **Services** (business rules, no SQL)
  - **Repositories** (SQL only, no business rules)
- No AI for the layer boundaries. The boundaries are the whole point. AI can help with the SQL inside a repo function after you have defined the function shape.

### Day 3 -- JWT auth with bcrypt (100% MANUAL ZONE)
- Install `bcrypt` and `jsonwebtoken` manually.
- Write the signup route by hand: validate input, hash the password, insert the user, return a token.
- Write the login route by hand: look up the user, compare the password, issue a token, return it.
- Write the `requireAuth` middleware by hand: read the Authorization header, verify the JWT, attach `req.user`.
- Write the `requireRole('admin')` middleware by hand: after `requireAuth`, check `req.user.role`.
- **Forbidden this week:** asking AI to "write me a JWT auth flow". You already know how to prompt AI -- do not use that power for auth.

### Day 4 -- Multi-user CRM
- Add a `users` table. Every lead now belongs to a user. Add `user_id` foreign keys where appropriate.
- Add row-scoped access: agents see only their leads; admins see all.
- Write integration tests for auth:
  - Signup with a duplicate email -> 409.
  - Login with wrong password -> 401.
  - Accessing a protected route without a token -> 401.
  - Accessing another user's lead -> 403.
- These tests are the contract. No auth PR merges without them.

### Day 5 -- Mongo aside + recap
- Read a short Mongo introduction (the docs, not AI). Understand how documents differ from tables.
- Write two or three example Mongo queries by hand in the shell. That is it. You do not need Mongo for the Capstone; you just need to recognise it if you meet it on the job.

---

## The Auth Rule In Detail

Every line that:

- hashes or verifies a password
- signs or verifies a JWT
- reads the Authorization header
- attaches a user to a request
- checks a role or permission
- enforces row-scoped access

must be written by you, reviewed by you line-by-line, and explained verbally to yourself before commit.

If AI wrote any of those lines, rewrite them by hand. Do not "edit AI output". Rewrite. The act of typing the characters trains your eye for what belongs there and what is a red flag.

This rule is permanent. It applies in Week 18 (webhook security + idempotency), Week 27 (capstone architecture), Week 28 (capstone core with RLS), and any future week that touches auth. No exceptions.

---

## You Must Break Things On Purpose

- Generate a JWT with an expired timestamp manually. POST it to a protected route. Observe your validator catches it.
- Hash a password with bcrypt round=4 and round=12. Time both. Feel why 12 is a good default for 2025.
- Remove the `requireAuth` middleware from one route temporarily. Curl it without a token. Restore it.
- Store a plaintext password in the DB (briefly). Open psql and see it sitting there. Delete the row. Never again.
- Write a `requireRole('admin')` middleware that forgets to check for `admin` and always allows. Run your tests and watch them fail.

Every one of these is a real incident that has happened at real companies. Practise them.

---

## What You CAN Use AI For (55%)

### Full AI permissions on data-layer work

- Postgres migration SQL (AI is great at SQL)
- Connection pool setup
- Route handlers (after you define the layer boundaries)
- Repository functions (once the shape is clear)
- Dashboard updates to use the new auth token

### Restricted AI on auth work

- Explaining bcrypt rounds after you wrote the hash call
- Explaining JWT libraries after you used one
- Reviewing (not writing) your auth middleware: "read this and tell me if you see any token leakage or bypass"
- **Never writing** the auth code itself

### Good vs bad prompts this week

**Bad:** "Write my JWT auth middleware."
**Good:** "Here is my JWT middleware [paste]. Read it as a security reviewer. What are three things an attacker would try against this code?"

**Bad:** "Build a layered architecture for my CRM."
**Good:** "I am refactoring my Week 11 CRM into routes/services/repos. Here is my current server.js [paste]. What are the first three files I should split out? I will do the splitting myself."

**Bad:** "Migrate my SQLite schema to Postgres."
**Good:** "Here is my SQLite schema [paste]. I am moving to Postgres. What Postgres-specific types should I use for each column (TEXT vs VARCHAR, INT vs BIGINT, TIMESTAMP vs TIMESTAMPTZ)?"

---

## The 25-Minute Rule (Still)

Same 25-minute rule as Week 8. It applies to everything except the auth module, where the rule is stricter: on auth code, **you do not ask AI for generation at all**. You can ask for review, or for explanation, but never for "write this for me".

---

## Things AI Is Bad At This Week

- **bcrypt rounds.** AI sometimes suggests 4 or 6 rounds because they are "fast in tests". Use 10-12 in production code. Test with 4 only in unit tests.
- **JWT secret management.** AI will happily hardcode secrets or use `"secret"` as the secret. Always from env. Always long. Always never in Git.
- **Token expiry.** AI's default JWT has either a 1-day expiry (too long for many apps) or no expiry at all. Pick your expiry based on your threat model; defaults are suspect.
- **Role checking.** AI sometimes writes `if (user.role === 'admin' || user.role === 'user')` which passes for every logged-in user. Read every condition twice.
- **SQL injection.** Even in 2025, AI occasionally concatenates strings into SQL. Use parameterised queries always; audit every query manually.

---

## Core Mental Models For This Week

- **A user is a row in the users table; authentication proves they are that row.** Everything else is variations.
- **A password hash is one-way.** You never store the password. You never recover it. You never email it. If a user forgets, you reset; you do not retrieve.
- **A JWT is a signed note that says "user X is authenticated until time Y".** The signature is proof. The server verifies it on every request.
- **Authorization (who can do what) is different from authentication (who they are).** Both are necessary. Neither is sufficient.
- **A layered architecture separates concerns.** Routes handle HTTP. Services handle business rules. Repositories handle the database. Mixing them makes bugs harder to find.

---

## This Week's AI Audit Focus

Add a required section: **Auth manual-only proof.** List every file that contains auth code and state who wrote every line:

```markdown
### File: middleware/requireAuth.js
- **Lines 1-30:** Me. I wrote the token extraction, JWT verify, and req.user attachment myself.
- **Lines 31-40:** Me. Error handling.
- **AI involvement:** None. I asked AI to review it afterwards; see review notes below.
- **Review notes from AI:** "Consider using jwt.verify with a try/catch instead of the async variant." I agreed -- rewrote it.
```

An audit where any auth file has "AI generated" in the author column is a fail for this week. Rewrite it by hand, even if it already works.

---

## Assessment

Week 12 assessment is a live security review:

- Walk through your auth flow end to end. Signup, login, protected route, role check.
- Facilitator asks: "show me where you hash the password. What rounds? Why?"
- Facilitator asks: "where is your JWT secret stored? What happens if I change it?"
- Live task: "add a new role called `moderator` that has the same access as `agent` plus the ability to edit other agents' leads". You implement it on screen.
- Facilitator gives you a broken auth middleware (token never checked for expiry). You find and fix the bug in 5 minutes. No AI.

### Live Rebuild Check

Facilitator deletes your `requireAuth` middleware. You rewrite it from memory. If you cannot -- which is fine once -- come back next week with proof you can.

---

## One Sentence To Remember

"Two things in this programme are forever manual: money and auth. Everything else is negotiable." If you remember nothing else from Week 12, remember that.
