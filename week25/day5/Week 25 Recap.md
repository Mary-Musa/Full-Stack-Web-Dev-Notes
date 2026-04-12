# Week 25, Day 5: Week Recap

You took a multi-service Node app and turned it into a deployable stack with one `docker-compose up`. This is the unglamorous week that makes Week 26 possible.

---

## What You Built

1. Dockerfiles for api, notification-service, shop.
2. `.dockerignore` for each.
3. Multi-stage builds for the Next.js shop.
4. `docker-compose.yml` running five containers with networking.
5. `docker-compose.override.yml` for dev-time hot reload.
6. Named volumes for Postgres and Redis.
7. Named networks (`backend`, `frontend`) for isolation.
8. Resource limits and log rotation.
9. Healthchecks on every service.
10. Nginx reverse proxy with path-based routing.
11. Let's Encrypt TLS via certbot.
12. Rate limiting and security headers in Nginx.

---

## Self-Review Questions

1. What is the difference between an image and a container?
2. Why use Alpine base images?
3. What does `.dockerignore` do and why is it essential?
4. What is a multi-stage build and when does it help?
5. How do compose services talk to each other by name?
6. What are named volumes vs bind mounts?
7. What does `depends_on: condition: service_healthy` do?
8. Why does the API need to be in both `frontend` and `backend` networks?
9. Why does Nginx need the `X-Forwarded-For` header?
10. How often do Let's Encrypt certs expire and how do you renew them?

Target: 8/10.

---

## Peer Coding Session

### Track A: Multi-environment compose
Add `docker-compose.staging.yml` that points at a staging database and uses real secrets from env. Show switching between dev / staging / prod.

### Track B: Database migrations
Run a `db-migrate` service in compose that applies SQL migrations before the API starts. Use `flyway` or a small custom script. API depends on migrations being successful.

### Track C: Log aggregation
Add a simple log aggregation service (`loki` or `vector`) and ship all container logs to it. Query logs with `logcli`. Prints of concept.

### Track D: One-command deploy
Write a `deploy.sh` script that SSHes into a server, pulls the latest images, and restarts compose. Keep it under 20 lines.

---

## Weekend Prep

The weekend deploys the full stack to a real server. Details in `week25/weekend/project.md`.

Next week is CI/CD with GitHub Actions, and then Projects 5 and 6 get built on this foundation.
