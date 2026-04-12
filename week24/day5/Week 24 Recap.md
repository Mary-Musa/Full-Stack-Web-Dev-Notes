# Week 24, Day 5: Week Recap

The split is done and it was cheaper than the marketing about microservices suggests. Your next natural question -- "should I split more?" -- is answered "only when the pain of not splitting exceeds the cost of splitting". For the Marathon, two services is enough.

---

## What You Built

1. `ARCHITECTURE.md` with the split plan.
2. A new `notification-service/` folder with its own `package.json`.
3. Moved the BullMQ workers and senders out of the API.
4. A root-level `concurrently` script to run both services.
5. A small HTTP server on the notification service for synchronous queries.
6. Internal service auth via shared token.
7. Timeouts and retries on cross-service HTTP calls.
8. `@mctaba/contracts` package with job shape validation.
9. Request id propagation through logs and job payloads.
10. Health endpoints on both services.

---

## Self-Review Questions

1. What are the five good reasons to split a monolith?
2. What are the hidden costs of microservices?
3. Why start the split with notifications specifically?
4. What are the four shapes of inter-service communication?
5. When would you pick queues over HTTP?
6. Why should internal services not be on the public internet?
7. What does the contracts package prevent?
8. What does a request id let you do?
9. Why should every service have a `/health` endpoint?
10. If the notification service is down, what happens to the API?

Target: 8/10.

---

## Peer Coding Session

### Track A: Extract payments as a service
The Week 19 payments package is already halfway there. Run it as a separate process with its own HTTP endpoints for "initiate payment".

### Track B: Service discovery
Instead of hard-coded URLs (`http://localhost:5010`), use a small service-registry pattern: a `services.json` file or a Redis key that maps service names to URLs. Both services read it at startup.

### Track C: Write a distributed tracing demo
Make one test order go through the shop, API, notification service, and capture the full request flow in a single file with timestamps. Screenshot the result.

### Track D: Contracts for the HTTP side
Extend `@mctaba/contracts` to define the shape of every cross-service HTTP endpoint. Both services validate requests and responses.

---

## Weekend Prep

The weekend puts both services in Docker. No new features -- just containerising what you already have. Brief in `week24/weekend/project.md`.

Next week is Docker + docker-compose + Nginx. The split this week makes it easy: you already have two processes to containerise.
