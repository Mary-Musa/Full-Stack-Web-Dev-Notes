# Week 26 - Day 2 Assignment

## Title
Docker Publishing And Deploys

## Overview
Day 2 builds your Docker image in CI, pushes it to GitHub Container Registry, and deploys it to a target (Render, Fly, or a VPS via SSH). Every merge to main becomes a deploy.

## Learning Objectives Assessed
- Build and push Docker images from GitHub Actions
- Tag images with git SHA and `latest`
- Trigger a remote deploy from CI
- Roll back safely

## Prerequisites
- Day 1 completed
- Dockerfile from Week 25

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** Shipping week. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Workflow YAML.
- **NOT ALLOWED FOR:** Rollback plan (you decide).
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Publish workflow

**What to do:**
`.github/workflows/publish.yml`:

```yaml
name: Publish
on:
  push:
    branches: [main]
jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository_owner }}/mctaba-api:latest
            ghcr.io/${{ github.repository_owner }}/mctaba-api:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Expected output:**
Image appears under the repo's Packages tab.

### Task 2: Choose a deploy target

**What to do:**
Pick ONE:
- Render: add a Web Service pointing at the GHCR image
- Fly.io: `fly launch` then `fly deploy` from the workflow
- VPS: SSH into a Hetzner/DigitalOcean box and `docker compose pull && up -d`

Document the choice in `deploy-choice.md` with the reason.

**Expected output:**
Target set up. First deploy succeeds.

### Task 3: Deploy workflow

**What to do:**
Add a `deploy` job depending on `build-push`. For a VPS:

```yaml
deploy:
  needs: build-push
  runs-on: ubuntu-latest
  steps:
    - uses: appleboy/ssh-action@v1.0.3
      with:
        host: ${{ secrets.DEPLOY_HOST }}
        username: deploy
        key: ${{ secrets.DEPLOY_KEY }}
        script: |
          cd /srv/mctaba
          docker compose pull
          docker compose up -d
          docker image prune -f
```

For Render/Fly, use their actions.

**Expected output:**
Merge to main deploys automatically.

### Task 4: Rollback plan

**What to do:**
In `rollback.md`, write 5-7 sentences:
- How do you roll back if the new image breaks prod?
- Why is `:sha` pinning better than `:latest` for rollback?
- What is "blue/green" deployment and when is it worth it?
- What database migrations do to rollbacks (they make them hard).

Your own words.

**Expected output:**
`rollback.md` committed.

### Task 5: Smoke test

**What to do:**
Add a final step to the deploy job that curls `https://your-domain/health` and fails the workflow if it does not return 200 within 30 seconds.

**Expected output:**
Workflow fails red if the deploy is broken.

## Stretch Goals (Optional - Extra Credit)

- Automatic rollback on smoke-test failure.
- Deploy to staging on every PR, prod on merge.
- Use `docker buildx` for multi-arch images.

## Submission Requirements

- **What to submit:** Repo, workflows, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Publish workflow | 25 | Image in GHCR. |
| Deploy target chosen | 10 | Documented. |
| Deploy workflow | 25 | Merge -> deploy. |
| Rollback notes | 20 | Four questions answered. |
| Smoke test | 15 | Fails on bad deploy. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **`:latest` only.** Rollback becomes "build the old commit". Use SHA tags.
- **Deploy without a smoke test.** You only find out it is broken from users.
- **SSH key in the repo.** Use GitHub secrets.

## Resources

- Day 2 reading: [Docker Publishing and Deploys.md](./Docker%20Publishing%20and%20Deploys.md)
- Week 26 AI boundaries: [../ai.md](../ai.md)
