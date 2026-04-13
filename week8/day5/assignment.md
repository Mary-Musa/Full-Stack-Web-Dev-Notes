# Week 8 - Day 5 Assignment

## Title
Code Review Etiquette and Responding To Feedback

## Overview
Week 8 Day 5 is the softest-skill Git day so far. Git and GitHub are just tools -- real collaboration is about how you give and receive code review feedback. You will review a cohort mate's PR, leave at least 5 constructive comments, and then respond to a review on one of your own PRs.

## Learning Objectives Assessed
- Read a diff line by line, not just the files changed count
- Leave comments that are specific, constructive, and kind
- Receive feedback without defensiveness
- Distinguish "must fix" from "suggestion" in your comments

## Prerequisites
- Weeks 1-7 Day 5 completed
- You and at least one cohort mate both have active PRs

## AI Usage Rules

**Ratio:** 55/45. **Habit:** AI as pair partner. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Explaining a React pattern you are reviewing.
- **NOT ALLOWED FOR:** Generating review comments for you to post.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Pair with a cohort mate

**What to do:**
Exchange PR links with one cohort mate. Both PRs should be from one of this week's assignments (SignupForm, Router, styled Task Manager).

**Expected output:**
You have a PR link to review and someone has your PR link.

### Task 2: Review five points on their PR

**What to do:**
Open the "Files changed" tab on their PR. Read the diff carefully. Leave at least five line-level comments:

1. One compliment about something well done (be specific)
2. One "must fix" if you see a real bug or missing case
3. Two "suggestions" for improvements that are not blockers
4. One question where you do not understand why they did something

Use this language pattern:
- "Must fix: ..." -- something that blocks merging
- "Suggestion: ..." -- non-blocking idea
- "Question: ..." -- asking for clarification
- "Nice: ..." -- compliment on well-done work

**Expected output:**
5 comments posted on their PR. Screenshot `day5-review-given.png`.

### Task 3: Receive feedback on your own PR

**What to do:**
Wait for your cohort mate (or facilitator) to leave comments on your PR. When they arrive:

1. Read each one carefully before responding.
2. For each comment:
   - If it is a "must fix" you agree with: make the change, commit, reply "Done in commit abcd123"
   - If you disagree: reply politely with your reasoning, not your feelings
   - If it is a "suggestion" you like: make the change
   - If it is a "suggestion" you reject: reply with why
   - If it is a "question": answer it clearly

No defensiveness. No "well actually" tone.

**Expected output:**
All comments addressed or resolved. Screenshot `day5-review-received.png`.

### Task 4: Merge after approval

**What to do:**
Once all comments are resolved and your reviewer approves, merge the PR. Delete the branch after merging.

**Expected output:**
Merged PR. No loose ends.

### Task 5: Short reflection

**What to do:**
In `day5-review-notes.md`, write 5-8 sentences:
- What was the hardest comment to receive and why?
- What was the hardest comment to give?
- One review comment you plan to do differently next time
- One thing you realised about your own code from reading someone else's

Your own words.

**Expected output:**
Notes committed.

## Stretch Goals (Optional - Extra Credit)

- Use GitHub's "Request changes" formal review status in addition to comments.
- Use GitHub's "Review with comments" button to post all your comments in one batch (better review etiquette than trickling in individually).
- Watch a real code review video (from any senior engineer) and add one technique you observed to your review toolkit.

## Submission Requirements

- **What to submit:** Repo link, PR links (yours and theirs), screenshots, `day5-review-notes.md`, `AI_AUDIT.md`.
- **Deadline:** End of Day 5

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Five review comments on peer PR | 25 | Each comment specific, labelled (Must fix / Suggestion / Question / Nice). |
| Responded to all feedback on own PR | 25 | Each comment addressed or replied to. No dangling threads. |
| Merge after approval | 15 | PR merged cleanly. Branch deleted. |
| Review reflection notes | 20 | Four questions answered honestly. |
| Professional tone throughout | 10 | No defensiveness, no passive aggression. |
| Clean commits on feedback | 5 | Commit messages show responses to feedback (e.g., "fix: address review comments"). |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **"Why did you do X??"** with the double question mark. Passive-aggressive review comments ruin teams. Ask the same thing as a real question: "What was your reasoning for X? I would have done Y because..."
- **Defending everything.** When a reviewer says "this would be cleaner as Y", the right response is usually "good idea, done" -- not a paragraph defending why Y is objectively worse. Pick your battles.
- **Approving without reading the diff.** Rubber-stamp approvals help nobody. If you cannot understand the change, ask.

## Resources

- Prior Day 5 assignments (Weeks 1-7)
- Week 8 AI boundaries: [../ai.md](../ai.md)
- Google's Code Review guidelines: https://google.github.io/eng-practices/review/
- "How to do a code review" (Google Engineering Practices): https://google.github.io/eng-practices/review/reviewer/
