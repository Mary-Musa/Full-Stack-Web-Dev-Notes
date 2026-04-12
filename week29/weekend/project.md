# Week 29 Weekend: Bug Fixing and Stability

No new features. The Capstone is feature-complete. The weekend is a stability sprint.

**Estimated time:** 4-6 hours.

---

## Tasks

- [ ] Run the full demo end to end ten times. Fix every bug you find.
- [ ] Run the isolation tests. Fix anything that fails.
- [ ] Load test: 100 orders in 60 seconds across two tenants. Verify no data bleed.
- [ ] Chaos test: stop the notification-service mid-demo. Verify queue recovery.
- [ ] Backup test: dump Postgres, wipe it, restore, verify the demo still works.
- [ ] Review every admin page for empty states and errors.
- [ ] Run `npm test` in every service; fix any failures.
- [ ] Rewatch your Week 29 Day 5 demo recording and note things to improve.

---

## Deliverables

- Demo runs smoothly in <5 minutes with zero manual interventions.
- All tests pass.
- Load test results documented in `PERFORMANCE.md`.
- Backup/restore procedure documented in `RUNBOOK.md`.

---

## Hints

**Do not add features this weekend.** If you have the urge, write it down in `NEXT.md` as a post-capstone idea. Come back after Week 30.

**Record your demo on Saturday.** Watch it Sunday. You will notice things you did not catch live.

**Reset the database between rehearsal runs.** Demo data should be repeatable. A fresh reset + demo should always work identically.

Next week: launch.
