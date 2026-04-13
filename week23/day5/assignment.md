# Week 23 - Day 5 Assignment

## Title
Queue Observability And Git Sparse Checkout

## Overview
Day 5 adds observability to your queues (Bull Board UI + basic metrics) and teaches `git sparse-checkout` for working with just part of a monorepo.

## Learning Objectives Assessed
- Wire Bull Board to view queue state
- Emit basic queue metrics
- Use `git sparse-checkout` to pull only one package

## Prerequisites
- Days 1-4 completed

## AI Usage Rules

**Ratio this week:** 35% manual / 65% AI
**Habit:** You design the work shape, AI fills in. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Bull Board wiring, metric helpers.
- **NOT ALLOWED FOR:** Deciding what to monitor.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Bull Board

**What to do:**
```bash
npm install @bull-board/express @bull-board/api
```

Mount it at `/admin/queues`:

```javascript
const { createBullBoard } = require("@bull-board/api");
const { BullMQAdapter } = require("@bull-board/api/bullMQAdapter");
const { ExpressAdapter } = require("@bull-board/express");

const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath("/admin/queues");
createBullBoard({ queues: [new BullMQAdapter(notificationQueue)], serverAdapter });
app.use("/admin/queues", ensureAdmin, serverAdapter.getRouter());
```

Protect behind admin auth.

**Expected output:**
Screenshot `day5-bullboard.png`.

### Task 2: Simple metrics endpoint

**What to do:**
```javascript
app.get("/admin/metrics", ensureAdmin, async (_req, res) => {
  const counts = await notificationQueue.getJobCounts("waiting", "active", "completed", "failed", "delayed");
  res.json(counts);
});
```

**Expected output:**
JSON with counts.

### Task 3: Sparse checkout

**What to do:**
In a scratch folder:

```bash
git clone --filter=blob:none --no-checkout <your-repo> sparse-test
cd sparse-test
git sparse-checkout init --cone
git sparse-checkout set week23
git checkout main
```

You should only see `week23/` in the working tree.

**Expected output:**
`ls` shows only week23.

### Task 4: Notes

**What to do:**
In `sparse-notes.md`, write 5-7 sentences:
- What does sparse-checkout solve?
- When is it worth it (very large monorepos)?
- What is `--cone` mode and why is it simpler?
- What does `--filter=blob:none` do? (Partial clone -- fetches blobs lazily.)

Your own words.

**Expected output:**
`sparse-notes.md` committed.

### Task 5: Week 23 wrap

**What to do:**
Update `README.md` with Week 23 section. Push everything.

**Expected output:**
Pushed.

## Stretch Goals (Optional - Extra Credit)

- Expose Prometheus metrics (`prom-client`) for queue depth.
- Set up an alert when queue depth > threshold.
- Explore `git clone --depth 1` and compare.

## Submission Requirements

- **What to submit:** Repo, screenshot, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Bull Board mounted | 25 | Screenshot, protected. |
| Metrics endpoint | 20 | Returns counts. |
| Sparse checkout done | 20 | Works, only week23 visible. |
| Sparse notes | 20 | Four questions answered. |
| README update | 10 | Week 23 section. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Bull Board not protected.** Queue contents can be sensitive.
- **Sparse checkout with the wrong path.** Double-check with `git sparse-checkout list`.
- **Confusing sparse checkout with shallow clone.** Different features.

## Resources

- Day 5 reading: [Week 23 Recap.md](./Week%2023%20Recap.md)
- Week 23 AI boundaries: [../ai.md](../ai.md)
- git sparse-checkout: https://git-scm.com/docs/git-sparse-checkout
