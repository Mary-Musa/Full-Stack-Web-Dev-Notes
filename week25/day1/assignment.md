# Week 25 - Day 1 Assignment

## Title
Docker Basics -- Build A Real Image For Your API

## Overview
Week 25 is infrastructure. Today you write a production-style Dockerfile for your API, understand layers and caching, and end up with an image you can run anywhere.

## Learning Objectives Assessed
- Write a multi-stage Dockerfile
- Understand build layers and cache
- Use `.dockerignore`
- Run the image locally with environment variables

## Prerequisites
- Weeks 1-24 completed

## AI Usage Rules

**Ratio this week:** 30% manual / 70% AI
**Habit:** AI writes infra, you verify boundaries. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Dockerfile boilerplate.
- **NOT ALLOWED FOR:** Deciding what goes in the image (secrets never).
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Multi-stage Dockerfile

**What to do:**
```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY src ./src
COPY package.json ./
ENV NODE_ENV=production
USER node
EXPOSE 3000
CMD ["node", "src/index.js"]
```

Build: `docker build -t mctaba-api .`

**Expected output:**
Image builds successfully.

### Task 2: .dockerignore

**What to do:**
Create `.dockerignore`:

```
node_modules
.git
.env
.env.*
*.log
coverage
.vscode
```

Rebuild and confirm the build context is small.

**Expected output:**
Small context logged by docker build.

### Task 3: Run with env vars

**What to do:**
```bash
docker run --rm -p 3000:3000 \
  -e DATABASE_URL=postgres://... \
  -e NODE_ENV=production \
  mctaba-api
```

Hit the API from another terminal. Confirm it works.

**Expected output:**
API responds.

### Task 4: Image size audit

**What to do:**
Run `docker images mctaba-api` and record the size in `image-size.md`. Answer:
- Why `node:20-alpine` instead of `node:20`?
- Why two stages?
- What layer is likely the biggest? (node_modules.)
- How does layer caching speed up rebuilds when only source changes?

Your own words.

**Expected output:**
`image-size.md` committed.

### Task 5: Secrets lecture

**What to do:**
In `secrets-in-docker.md`, write 4-6 sentences:
- Why you should NEVER `COPY .env` into the image.
- Why `ENV SECRET=...` in a Dockerfile is also wrong.
- What is the right way? (Runtime env vars, secret managers, Docker secrets for swarm.)

**Expected output:**
`secrets-in-docker.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Use `dive` or `docker history` to inspect the layers.
- Add a HEALTHCHECK instruction.
- Switch to `node:20-slim` and compare image size.

## Submission Requirements

- **What to submit:** Repo with Dockerfile, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 1

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Multi-stage Dockerfile | 25 | Builds and runs. |
| .dockerignore | 10 | Context small. |
| Run with env vars | 20 | API works. |
| Image size notes | 25 | Four questions answered. |
| Secrets notes | 15 | Three questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Running as root.** Use `USER node`.
- **COPY . .** early, breaking cache on every file change.
- **Including dev dependencies in production image.** `--omit=dev`.

## Resources

- Day 1 reading: [Docker Basics.md](./Docker%20Basics.md)
- Week 25 AI boundaries: [../ai.md](../ai.md)
- Dockerfile reference: https://docs.docker.com/engine/reference/builder/
