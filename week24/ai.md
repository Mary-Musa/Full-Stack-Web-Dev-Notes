# Week 24 - AI Boundaries

**Ratio this week: 45% Manual / 55% AI** (ARCHITECTURE DIP)
**Habit introduced: "Draw the split on paper."**
**Shift from last week: Manual bumps back up. Microservices decomposition is pure architecture work, and architecture is a human responsibility.**

This week you split the monolith into multiple services. Which pieces come apart? Which stay together? What is the boundary? How do they communicate? These are the questions senior engineers agonise over and AI cannot answer. AI can implement a split once you have decided the split; it cannot decide where to split.

---

## Why The Ratio Jumped Back Up

Architecture work compresses poorly. A wrong split costs weeks later. A right split is invisible because everything just works. Only humans can taste the difference -- AI has no concept of "this will hurt in three months". So for architecture weeks (24, 27), the manual ratio bumps up.

---

## What You MUST Do Manually (45%)

### Day 1 -- Why microservices
- Read at least two opposing perspectives: "microservices at scale" and "don't split prematurely". AI summaries are banned this week.
- Decide: what is the one hard reason you are splitting *this* codebase? Write it down. If you cannot write it, you should not split.

### Day 2 -- Extracting the notification service
- On paper: list every function that currently lives in your monolith and classify it as "belongs to notifications" or "belongs elsewhere". This is the hard part.
- Draw the communication boundary: what does the rest of the app know about notifications? (Ideally: a single interface.)
- Move the code. Keep the interface stable. Test that the monolith still works with the extracted service.

### Day 3 -- Inter-service communication
- Decide: sync HTTP, async messages, or shared database? Each has trade-offs. Draw them on paper.
- Implement the chosen approach for one call path (e.g., "order paid" triggers "send WhatsApp").
- Measure the latency difference from the monolithic version.

### Day 4 -- Contracts, versioning, observability
- Define a contract between services (e.g., JSON schema, TypeScript interface, gRPC proto).
- Version it. Know how to deploy a new version without breaking the old consumer.
- Add observability: logs, traces, health checks.

---

## What You CAN Use AI For (55%)

- Implementing the split once you decided it.
- Adapter code to maintain backward compatibility.
- Health check endpoints.
- Dashboard showing service status.

Forbidden:
- Deciding the split boundary.
- Choosing the communication pattern (sync vs async).
- Defining the contract shape.

---

## Things AI Is Bad At This Week

- **Your actual coupling.** AI cannot see your runtime behaviour. It splits based on file structure, which is usually wrong.
- **Organisational fit.** AI does not know which team owns what. A split that makes technical sense but crosses team boundaries is painful.
- **Premature splitting.** AI will happily split everything into tiny services. 90% of the time that is wrong.

---

## Core Mental Models For This Week

- **A service boundary is a deployment boundary.** Two things are separate services if and only if you deploy them independently.
- **A contract is the shape of a conversation.** Change the shape, break the conversation.
- **Sync calls are simple and fragile.** Async messages are complex and robust. Pick based on the blast radius.

---

## This Week's AI Audit Focus

Include the paper diagram of your split. Annotate with "why this boundary" for each line you drew.

---

## Assessment

- Walk through your split. Defend every boundary. "Why is this here and not there?"
- Live task: revert one split decision. Explain what you would do differently.

---

## One Sentence To Remember

"AI cannot taste architecture. Architecture is a human sense you develop by making decisions and living with them."
