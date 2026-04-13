# Week 1 - AI Boundaries

**Ratio this week: 90% Manual / 10% AI**
**Habit introduced: "Ask AI to explain, not to do."**

This is the highest-manual week of the entire 30-week programme. If that feels strict, it is on purpose. Every week from here on loosens the rules -- but only because the foundation we build this week is strong enough to hold the loosening. Skip the discipline this week and the whole programme stops working.

---

## What These Numbers Mean

The two lines at the top of this file -- the **ratio** and the **habit** -- show up at the top of every weekly `ai.md` file from Week 1 to Week 30. Read them like the first line of a recipe. They are the rules of engagement for the week.

### The Ratio (90% Manual / 10% AI)

The ratio is a rough split of where your hands-on work should come from this week. Out of every ten lines of code, terminal commands, or written explanations you produce, nine should be typed by you with no AI involvement, and only one should come from an AI tool. The number is not a quota you measure to the decimal. It is a target that tells you where the centre of gravity should be.

This week the centre of gravity is firmly on you. By Week 12 it has shifted to the middle. By Week 30 it has tipped toward AI -- but only because by then you can read AI output critically, fix what it gets wrong, and use it the way a senior engineer uses any tool.

The ratio you see at the top of each week's `ai.md` is not arbitrary. It moves up when you encounter a new technology (because you need to learn it from scratch), and it moves down when you are composing things you already know (because AI is genuinely faster at boilerplate).

### The Habit ("Ask AI to explain, not to do")

The habit is one specific AI-usage discipline introduced this week that you carry forward for the rest of the programme. Each week introduces exactly one new habit. By Week 12 you will have twelve habits stacked on top of each other -- which is roughly the toolkit a working engineer uses to stay in control of their own thinking when AI is at their elbow.

This week's habit is the foundation: **ask AI to explain things, do not ask AI to do things for you.** That is it. Every prompt you write this week should end in a question mark, not a request. We will sharpen this habit in the rest of this document with examples of what counts and what does not.

### What You Are Actually Building This Week

While reading this AI guide, do not lose sight of what Week 1 is teaching: setting up your developer workshop, learning how the web works, and building a multi-page semantic HTML portfolio site by the weekend. Day-by-day the curriculum walks you through VS Code, Git, GitHub, the terminal, how the internet routes a request, the anatomy of an HTML document, common HTML tags, semantic page structure, HTML5 media (video, audio, iframe), forms, tables, and accessible markup. The weekend project ties all of that together into a real portfolio you can show people.

The ratio and the habit apply to every part of that journey. Keep them in mind whenever you are tempted to type a prompt instead of typing a tag.

---

## Why The Ratio Is This High

You will feel tempted. AI tools can set up a VS Code environment, explain what `git init` does, write your first HTML page, and style it in Tailwind -- in about ninety seconds. Doing the same thing manually will take you an afternoon.

Do the afternoon.

The reason is physical. Typing `git commit -m "first commit"` yourself and watching the output is a different neurological event from reading AI-generated instructions that say "now run this command". The first one creates a memory. The second one creates the feeling that you understand. Two different things. By Week 6 when you are debugging an Express server at midnight, only the first kind of memory helps you.

The other reason is that every tool you install this week is a tool you will be living inside for thirty weeks. VS Code, Git, the terminal -- they are your workshop. A woodworker learns their chisels by hand before they start carving. You are learning your chisels.

---

## What You Will Feel This Week

This week will feel different from what you expect.

- You will feel slow.
- You will feel like others are moving faster than you.
- You will be tempted to use AI "just this once."
- You may feel like you are not getting it.

All of this is normal.

Speed is not the goal this week. **Understanding is.**

If you feel slow, it usually means you are actually thinking. That is the point.

---

## What You MUST Do Manually (90%)

### Environment setup (Day 1)

- Type every installation command in the terminal yourself. Do not copy-paste from AI.
- Install VS Code and every extension one at a time. Read what each extension does from its own page, not from AI.
- Configure Git with your name and email manually: `git config --global user.name ...`
- Generate an SSH key for GitHub yourself, follow GitHub's own docs.

### Terminal basics (Days 1-2)

- `pwd`, `cd`, `ls`, `mkdir`, `touch`, `mv`, `cp`, `rm` -- every one, typed, tried, broken, fixed. No "what does this command do?" to AI before you have run it at least twice.
- Navigate your filesystem using only the terminal for at least one hour of practice. No Finder, no Explorer.

### First HTML page (Days 2-3)

- Every tag, every attribute, every character -- typed. No "generate a personal profile page for me". The page will be ugly. That is fine. Ugly-but-understood beats pretty-but-mystery every time this week.
- Minimum elements: `<html>`, `<head>`, `<title>`, `<body>`, `<h1>`, `<p>`, `<ul>`/`<li>`, `<a>`, `<img>`. You should be able to name every one out loud without looking.

### First CSS (Day 4)

- Write every selector yourself. Box model, margins, padding, colours -- by hand. If a background colour does not work, debug it by reading the code, not by asking AI "why is my background not showing".
- Build a responsive nav bar using flexbox manually. No Tailwind this week; only plain CSS. Tailwind comes in Week 5 after you understand what it is abstracting.

### First Git workflow (Day 5)

- `git init`, `git add`, `git commit`, `git remote add`, `git push` -- each one typed. Do `git status` between every pair of commands so you see the state change.
- Make at least 10 commits this week. Not one big "final" commit. Small commits with real messages like "add profile bio section" or "fix broken image path".
- Resolve at least one merge conflict manually, even if you have to create it artificially.

---

## You Must Break Things On Purpose

This week is not just about building. It is also about breaking and fixing.

You are required to intentionally break your work at least 3 times and fix it.

Examples:

- Delete a closing HTML tag and observe what changes
- Rename your CSS file and debug why styles stop working
- Change an image path so it breaks, then fix it
- Create a Git merge conflict and resolve it

If you are afraid to break things, you will be afraid to learn.

---

## What You CAN Use AI For (10%)

**The only permitted AI use this week is asking for explanations of things you already tried.**

Not: "Write me a terminal command to create a new folder and cd into it."
But: "I typed `mkdir my-project && cd my-project` and it worked. What does `&&` mean?"

Not: "Generate an HTML profile page."
But: "I wrote `<img src=profile.jpg>` and the image does not appear. What am I missing?"

Not: "Explain HTML to me."
But: "The MDN docs say HTML is `semantic`. What does that mean with an example?"

### Good vs bad prompts (memorise these)

**Bad:** "How do I create a GitHub repo?"
**Good:** "I ran `git push -u origin main` and got `error: src refspec main does not match any`. What did I get wrong?"

**Bad:** "Write me a personal profile page."
**Good:** "Here is my HTML [paste]. The list items are showing bullets but I want dashes. Is that a CSS problem or an HTML problem?"

**Bad:** "Make this CSS look better."
**Good:** "My nav bar items are stacking vertically on mobile. I used `display: flex`. What do I need to add so they stay horizontal?"

The pattern: you have tried it, you have a specific symptom, you name what you already know, you ask one focused question. This is the same shape every senior engineer uses with Stack Overflow, with documentation, and now with AI. Learn the shape this week.

---

## The Shape of a Good Question

Every good question follows this structure:

1. What I tried
2. What happened
3. What I expected
4. My guess (optional)

Example:

> "I used `display: flex` on my nav bar. The items are still stacking vertically. I expected them to be horizontal. Is there something I am missing?"

If your question does not include what you tried, you are asking too early.

---

## The 10-Minute Rule (This Week Only)

Before asking AI anything this week, you must have spent at least ten minutes trying on your own. Ten minutes of:

1. Re-reading the error message out loud.
2. Searching the exact error text in a regular search engine.
3. Reading the official docs for the tool you are using.
4. Trying one small change at a time.

After ten minutes of this, if you are still stuck, you have earned the right to ask AI -- and your question will be much better for having tried.

Next week this becomes the 15-minute rule. Week 3 becomes 30 minutes. The pattern is: try first, then ask.

---

## What Counts As "Trying"

Sitting and staring at the screen is not trying.

Trying means:

- Running the code again
- Changing one small thing and observing the result
- Reading the error message carefully
- Looking at the documentation
- Testing a different approach

If nothing has changed between attempts, you are not actually trying.

---

## Definition of Understanding

You understand something if you can:

- Explain it in your own words without reading
- Change it and predict what will happen
- Recreate it from scratch without copying

If you cannot do these three things, you are still learning it -- which is fine. Just be honest about it.

---

## The AI Audit File

Even this week, with only 10% AI usage, you submit an `AI_AUDIT.md` file with your Week 1 project. Yes, even if you only asked AI one question. The audit builds the habit.

Create `week1-project/AI_AUDIT.md`:

```markdown
# AI Audit -- Week 1 Project

## AI tools used

- Claude (or Cursor, or whatever you are using)

## Summary

- Total files in project: 3 (index.html, style.css, README.md)
- Files with AI-generated code: 0
- Estimated % of code AI-generated: 0%

## AI usage log

### Debug: image not showing

- **Prompt used:** "I wrote `<img src=profile.jpg>` in my HTML but the image does not appear in the browser. The file profile.jpg exists in the same folder. What am I missing?"
- **What AI generated:** Explained that `src` attribute needs quotes, and that paths are case-sensitive on some systems.
- **What I changed and why:** Added quotes: `<img src="profile.jpg">`. It worked.
- **What I learned:** HTML attributes need quotes. Also paths are case-sensitive. On my Mac it worked without quotes, but on my friend's Linux it did not.
- **Would I prompt differently next time?** No -- I tried it first and gave the exact symptom.
```

A good audit takes five minutes to write per AI interaction and will save you hours later. Facilitators read these closely.

---

## Things AI Is Bad At This Week

- **Your specific machine.** If Windows has a different PATH problem than Mac, AI will guess wrong half the time. Read your own error message carefully.
- **Your network.** "My `git push` is hanging" might be a firewall, a proxy, a typo, or GitHub being slow. AI cannot tell. You have to.
- **Your learning.** AI does not know which concepts have clicked for you and which have not. Only you do.

When AI is guessing, it sounds just as confident as when it is right. That is the thing you are training against this week. Judgement begins with noticing.

---

## Core Mental Models (Memorise These)

- **HTML = Structure (what exists)**
- **CSS = Appearance (how it looks)**
- **Git = History (what changed and when)**
- **Terminal = Control (how you talk to your computer)**

If you get lost, come back to these.

---

## Daily 5-Minute Reflection (Required)

At the end of each day, write down:

- What did I learn today?
- What confused me?
- What did I fix that was broken?
- What took longer than expected?

This is not for grading. It is for awareness.

---

## Assessment

The Week 1 assessment is a live walkthrough with a facilitator. You share your screen, open your project, and answer questions like:

- Show me where you committed the first version of your profile page. How many commits are between then and now? What does each commit do?
- What does `git add .` do that `git add index.html` does not?
- Why is your `<h1>` before your `<p>` in the HTML? What happens if you swap them?
- How does the browser know to use your `style.css` file?
- What does `position: relative` mean in your CSS? Point at where you used it.

**No AI during this assessment.** If you cannot answer a question, that is the data. Write down what you could not answer, come back next week with the answer, and move on.

The goal is not a perfect score. The goal is honest self-assessment: what did I actually learn this week, and what did I only feel like I learned?

### Live Rebuild Check

During assessment, you may be asked to:

- Delete part of your code and rewrite it
- Recreate a small section from memory
- Fix a simple bug live

If you cannot do this, it means you relied too much on copying instead of understanding.

---

## If You Skip This Week Properly

If you rush or rely on AI too early:

- Week 3: You will copy code without understanding it
- Week 6: You will struggle to debug errors
- Week 10: You will depend fully on AI
- Later: You will struggle in interviews and real projects

This week prevents all of that.

---

## One Sentence To Remember

"I am training my brain, not my keyboard." This week, every slow, manual action is the training. Next week you start letting AI do the keyboard work, because your brain is ready for it.

---

## One Rule Above All

If you wrote it, you should be able to explain it.

If you cannot explain it, you did not write it -- even if your keyboard typed it.
