# Week 24 Weekend: Two Services Running Cleanly

Run both services together for an extended session. No new features. Just verify the split is solid under real conditions. This is a setup weekend that pays off when Week 25 adds Docker.

**Estimated time:** 3-4 hours.

**Deadline:** Monday morning.

---

## Tasks

- [ ] Both services run via `npm run dev` at the repo root.
- [ ] Both services have `.env.example` files checked in.
- [ ] Both services have health endpoints.
- [ ] A "kill the notification service" test: verify the API keeps working; jobs queue up; messages send when the service returns.
- [ ] A "kill the API" test: verify the notification service keeps draining its queue.
- [ ] Run a load test: send 1000 orders through the shop in five minutes; verify both services survive, no jobs lost.
- [ ] Request id tracing works end-to-end.
- [ ] Contracts package validates every job.

---

## Load Test

A quick load test using `autocannon` or a simple script:

```javascript
// scripts/load-test.js
const fetch = require("node-fetch");

async function main() {
  const promises = [];
  for (let i = 0; i < 1000; i++) {
    promises.push(
      fetch("http://localhost:3000/api/orders", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ items: [/* ... */], customer: { /* ... */ } }),
      })
    );
    if (i % 50 === 0) await new Promise((r) => setTimeout(r, 100));
  }
  const results = await Promise.allSettled(promises);
  const ok = results.filter((r) => r.status === "fulfilled" && r.value.ok).length;
  console.log(`${ok} / 1000 orders accepted`);
}

main();
```

Run it. Watch Bull Board. Watch CPU and memory on both processes. Expected:

- API stays responsive.
- Notification queue fills up.
- Notification service drains it steadily at the rate-limited pace.
- Every job eventually completes.

---

## Deliverables

- A `PERFORMANCE.md` file with notes from the load test:
  - Orders per second the API accepted.
  - Peak queue depth.
  - Time to fully drain the queue after the test.
  - Any errors observed.

- Screenshots of Bull Board during and after the test.

---

## Grading Rubric (50 pts)

| Area | Points |
|---|---|
| Clean two-service run | 10 |
| Kill tests pass | 15 |
| Load test with numbers | 15 |
| PERFORMANCE.md quality | 5 |
| No regressions in existing features | 5 |

---

## Hints

**Start small.** 100 orders first, 500 next, then 1000. You will find bottlenecks in the API before the queue.

**Redis is the first bottleneck.** If BullMQ gets slow, check Redis CPU and memory.

**Postgres connection pool is the second.** Your notification service worker might exhaust its pool if concurrency is too high. Tune `max: 10` on the pool.

**Log to files, not stdout.** `npm run dev | tee logs/dev.log` captures for later analysis.

Next week is Docker.
