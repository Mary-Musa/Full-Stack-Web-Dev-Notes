# Week 25, Day 1: Docker Basics

By the end of today, you can build a Docker image of your API service, run it locally, understand what a Dockerfile is, and know the difference between an image and a container.

**Prior-week concepts you will use today:**
- The two-service split from Week 24.
- Node, package.json, npm scripts.

**Estimated time:** 3 hours

---

## The Week Ahead

| Day | Focus |
|---|---|
| Day 1 (today) | Docker basics: images, containers, Dockerfile. |
| Day 2 | docker-compose: running multiple containers together. |
| Day 3 | Volumes, environment variables, networks. |
| Day 4 | Nginx reverse proxy, TLS, production-shaped deploy. |
| Day 5 | Recap. |
| Weekend | Dockerise the whole system end-to-end. |

---

## What Docker Is

A Docker **image** is a filesystem snapshot plus a command to run. Think of it as "a zip file of an app plus the instructions to start it". Images are immutable.

A Docker **container** is a running instance of an image. You can start, stop, restart containers. Multiple containers can run from the same image.

A Docker **Dockerfile** is the recipe to build an image. You write instructions; Docker builds the image.

Three concepts, no more:

```
Dockerfile -> (build) -> Image -> (run) -> Container
```

Docker solves three problems at once:

1. **"It works on my machine."** Dockerise the app and it works the same on every machine that has Docker.
2. **Dependency conflicts.** Two apps need different Node versions? Run them in different containers.
3. **Deploy portability.** The same image runs on your laptop, a dev server, and production. No differences.

---

## Install Docker

### Ubuntu / Debian
```bash
sudo apt install docker.io docker-compose
sudo usermod -aG docker $USER
# log out and back in for group change to take effect
```

### Mac
Install Docker Desktop from https://docker.com/products/docker-desktop. Works on both Intel and Apple Silicon.

### Windows
Docker Desktop with WSL 2.

### Verify
```bash
docker --version
docker ps    # shows running containers, empty list at first
```

---

## Your First Dockerfile

Create `api/Dockerfile`:

```dockerfile
# Base image: a pre-built Linux image with Node 20
FROM node:20-alpine

# Set the working directory inside the container
WORKDIR /app

# Copy package files first (for better caching)
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy the rest of the source
COPY . .

# The app listens on port 5000
EXPOSE 5000

# Start the app
CMD ["node", "index.js"]
```

Eight lines. Walk through each:

- **`FROM node:20-alpine`** -- base image. Alpine Linux is a small Linux distro (~5 MB) with Node pre-installed. The image is about 50 MB, vs 350 MB for `node:20` (Debian-based).
- **`WORKDIR /app`** -- the directory inside the container where everything else runs.
- **`COPY package*.json ./`** -- copy just the package files first. This is a cache optimisation: if only source changed but dependencies did not, the `npm ci` step can use Docker's cache.
- **`RUN npm ci --only=production`** -- install only production dependencies.
- **`COPY . .`** -- copy the rest of the source code.
- **`EXPOSE 5000`** -- documentation that the container listens on 5000. Does not actually open the port; just metadata.
- **`CMD ["node", "index.js"]`** -- the command to run when the container starts.

---

## Build The Image

```bash
cd api
docker build -t mctaba-api:dev .
```

The `-t mctaba-api:dev` tags the image with a name (`mctaba-api`) and a version (`dev`). The `.` is the build context -- Docker sends everything in this directory to the Docker daemon.

On the first run it downloads the base image (takes a minute), then runs each step. You see output like:

```
Step 1/7 : FROM node:20-alpine
Step 2/7 : WORKDIR /app
Step 3/7 : COPY package*.json ./
...
Successfully built 1a2b3c4d
Successfully tagged mctaba-api:dev
```

---

## Run The Container

```bash
docker run -p 5000:5000 --env-file .env mctaba-api:dev
```

- `-p 5000:5000` -- publish the container's port 5000 to the host's port 5000. Without this, the port is inaccessible from outside.
- `--env-file .env` -- pass environment variables from your `.env` file.
- `mctaba-api:dev` -- the image to run.

You should see your API start. Visit `http://localhost:5000/health` -- it should respond.

To stop the container, press Ctrl+C. The container is now "stopped" but still exists on your system. List all containers (running and stopped):

```bash
docker ps -a
```

Remove stopped containers:

```bash
docker container prune
```

---

## The `.dockerignore` File

When `COPY . .` runs, it copies *everything* in the directory, including `node_modules/`, `.env`, `.git/`. You do not want any of those in the image.

Create `api/.dockerignore`:

```
node_modules
npm-debug.log
.git
.env
.env.*
!.env.example
*.log
.DS_Store
coverage/
.vscode/
```

Standard. Most projects have this. Rebuild the image and it is 100 MB smaller.

---

## Multi-Stage Builds

A more efficient Dockerfile uses multiple stages. One stage installs dependencies (with dev deps for building); another stage is the final minimal runtime image:

```dockerfile
# Stage 1: builder
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build || echo "no build step"

# Stage 2: runner
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app .
EXPOSE 5000
CMD ["node", "index.js"]
```

Two `FROM` lines. The first stage builds everything; the second stage copies only what it needs into a fresh minimal image. Final size is the same but build caching is better.

For a plain Node API with no build step, a single-stage Dockerfile is fine. Multi-stage matters more for Next.js (which has a `next build` step that produces different artifacts).

---

## Dockerfile For Next.js

The Next.js shop has a build step:

```dockerfile
# shop/Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["npm", "start"]
```

Two stages: build with full deps, run with production deps + build output.

Newer Next.js versions support "standalone output" which produces an even smaller runtime. Add `output: 'standalone'` to `next.config.js`.

---

## Images vs Containers, Explained One More Time

- An image is a **thing on disk**. Static. Immutable. You build it once, run it many times.
- A container is a **running instance**. It has an id, a state (running/stopped/dead), logs, and a process.
- You can run 100 containers from the same image. They are all independent.
- Stopping a container does not delete it. Removing it does.
- Removing an image does not affect running containers of that image.

The confusion is worth resolving before Day 2. Most of this week's commands operate on one or the other, and mixing them up produces frustrating errors.

---

## Checkpoint

1. `docker --version` prints a version number.
2. `docker build -t mctaba-api:dev .` completes without errors.
3. `docker run -p 5000:5000 --env-file .env mctaba-api:dev` starts the API.
4. `curl http://localhost:5000/health` returns 200.
5. `docker ps` shows the running container.
6. `docker stop <id>` stops it.
7. The image size with `.dockerignore` is substantially smaller than without.
8. `docker build -t mctaba-shop:dev ./shop` builds the Next.js image.

Commit:

```bash
git add .
git commit -m "feat: dockerfile for api and shop"
```

---

## What You Learned

- Docker builds an image from a Dockerfile; containers run from images.
- Alpine Linux is smaller than Debian.
- `.dockerignore` keeps `node_modules` and secrets out.
- Multi-stage builds separate build-time and run-time dependencies.
- Next.js needs the build output copied into the runtime image.

Tomorrow: docker-compose runs all three containers (API, notification service, shop) plus Postgres and Redis with one command.
