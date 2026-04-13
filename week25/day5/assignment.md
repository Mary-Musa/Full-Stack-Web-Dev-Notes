# Week 25 - Day 5 Assignment

## Title
One-Command Deploy And Git Tag Signing

## Overview
Day 5 ties Week 25 together: a single script spins up everything locally. You also learn signed git tags so releases are verifiably yours.

## Learning Objectives Assessed
- Orchestrate full stack with one command
- Sign a git tag with GPG or SSH
- Understand tag vs commit signing
- Document a local setup in README

## Prerequisites
- Days 1-4 completed

## AI Usage Rules

**Ratio this week:** 30% manual / 70% AI
**Habit:** AI writes infra, you verify boundaries. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Script boilerplate.
- **NOT ALLOWED FOR:** Key management decisions.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: One-command script

**What to do:**
Create `scripts/dev-up.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
cp -n .env.dev.example .env.dev || true
docker compose --env-file .env.dev up -d
docker compose exec -T api npm run migrate || true
echo "Up: https://localhost"
```

Make it executable. Document in README.

**Expected output:**
`./scripts/dev-up.sh` from clean gets you running.

### Task 2: Generate a signing key

**What to do:**
Either GPG or SSH signing. For SSH (simpler):

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global tag.gpgsign true
```

Add your SSH key as a signing key on GitHub (Settings > SSH and GPG keys > New SSH key > Signing type).

**Expected output:**
Config set.

### Task 3: Sign a tag

**What to do:**
```bash
git tag -s v0.1.0 -m "week 25 milestone"
git push --tags
```

GitHub shows a "Verified" badge on the tag.

**Expected output:**
`day5-signed-tag.png` showing verified on GitHub.

### Task 4: Notes

**What to do:**
In `signing-notes.md`, write 5-7 sentences:
- Why sign a tag?
- What is the difference between signing a commit and signing a tag?
- What does "Verified" mean on GitHub?
- What happens if you lose your signing key? (You cannot sign under that identity; GitHub will show Unverified for future tags.)

Your own words.

**Expected output:**
`signing-notes.md` committed.

### Task 5: Week 25 wrap

**What to do:**
Update `README.md` with a Week 25 section and a quickstart block:

```markdown
## Quickstart

    git clone ...
    cd class-notes
    ./scripts/dev-up.sh
    open https://localhost
```

**Expected output:**
Pushed.

## Stretch Goals (Optional - Extra Credit)

- Enable commit signing too.
- Add `cosign` to sign Docker images.
- Publish a signed Docker image tag.

## Submission Requirements

- **What to submit:** Repo, script, screenshot, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| dev-up script | 25 | Works from clean. |
| Signing key set up | 15 | Config visible. |
| Signed tag pushed | 25 | Verified on GitHub. |
| Signing notes | 20 | Four questions answered. |
| README quickstart | 10 | Clear. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Using GPG when SSH would do.** SSH signing is simpler on modern Git.
- **Forgetting to upload the signing key to GitHub.** GitHub shows Unverified.
- **Signing old tags.** You cannot retroactively sign.

## Resources

- Day 5 reading: [Week 25 Recap.md](./Week%2025%20Recap.md)
- Week 25 AI boundaries: [../ai.md](../ai.md)
- Git SSH signing: https://calebhearth.com/sign-git-with-ssh-keys
