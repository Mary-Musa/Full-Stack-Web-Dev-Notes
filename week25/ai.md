# Week 25 - AI Boundaries

**Ratio this week: 30% Manual / 70% AI**
**Habit introduced: "AI is great at YAML; you verify every port."**
**Shift from last week: Back to 30/70 because Docker and Nginx config is AI's strongest zone.**

Docker files, docker-compose, Nginx config -- all of it is structured text that AI has seen thousands of examples of. AI can generate working Dockerfiles, compose stacks, and reverse proxy configs on the first try. The risk is that working configs look right and still have subtle problems: wrong port mappings, missing healthchecks, credentials in the wrong file, networks that do not connect.

The habit: treat AI-generated config as a first draft. Verify every port, every volume, every environment variable, every network.

---

## What You MUST Do Manually (30%)

### Day 1 -- Docker basics
- Install Docker yourself. `docker run hello-world`.
- Write your first Dockerfile for the Express API by hand. Understand `FROM`, `COPY`, `RUN`, `CMD`, `EXPOSE`.
- Build and run the image. See the container start.

### Day 2 -- Docker Compose
- Compose file for the full stack: api, notification worker, Postgres, Redis. Each as a service.
- Run the stack. `docker compose up`. Test that everything connects.

### Day 3 -- Volumes, environments, networks
- Volumes for Postgres persistence. Test by stopping and restarting -- data survives.
- Environment variables for secrets (not in the compose file; in a `.env`).
- Networks for service isolation.

### Day 4 -- Nginx and TLS
- Nginx reverse proxy in front of your API.
- Real TLS cert (Let's Encrypt via certbot) in a non-production environment.
- Test: hit your service over HTTPS.

---

## What You CAN Use AI For (70%)

- Dockerfile drafts.
- docker-compose drafts.
- Nginx config drafts.
- Healthcheck helpers.

After you review every line and verify every port and volume.

Forbidden:
- Accepting config without line-by-line review.
- Storing secrets in committed files.

### Good vs bad prompts this week

**Bad:** "Write me a Dockerfile."
**Good:** "Here is my Node app structure [paste]. I want a multi-stage Dockerfile that builds TypeScript then runs the JS output. Include a non-root user for security."

---

## Things AI Is Bad At This Week

- **Non-root users.** AI defaults to running as root. Always add a `USER` directive.
- **Healthchecks.** AI often forgets them. Always add.
- **Port mismatches.** AI sometimes exposes 3000 but the app listens on 8080. Verify.
- **Secrets in compose.** AI will happily paste `DATABASE_PASSWORD=prod` into compose. Always use an env file or secret manager.

---

## Core Mental Models For This Week

- **A container is a process in its own filesystem namespace.** Not a VM.
- **A Dockerfile is a recipe that produces an image.** The image is a template; the container is an instance.
- **A compose file is a multi-container recipe.** Services, networks, volumes.

---

## This Week's AI Audit Focus

For every AI-generated config file, include a line-by-line review comment: "verified port", "verified volume path", etc. Missing reviews = failed audit.

---

## Assessment

- Facilitator starts your stack on a fresh machine from your compose file. Everything should just work.
- Live task: add a new service to your compose stack. Connect it to the existing network.

---

## One Sentence To Remember

"Config looks right more often than it is right. Verify every line."
