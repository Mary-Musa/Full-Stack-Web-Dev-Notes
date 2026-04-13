# Week 3 - AI Boundaries

**Ratio this week: 80% Manual / 20% AI**
**Habit introduced: "The 15-minute rule."**
**Shift from last week: You get 5% more AI room, and the wait-before-asking window grows from 10 minutes to 15.**

This is the first week of real programming. Up to now you have been describing things (HTML) and decorating them (CSS). This week you tell the computer to do things -- make decisions, repeat itself, remember values. That is what a variable is. That is what a loop is. That is what a condition is. None of those three ideas are hard in isolation; what is hard is making them cooperate.

This week's habit is longer patience before you ask AI. Not harder questions. Just more trying.

---

## Why The Ratio Moved

You are about to write code that behaves. Not code that looks like something, but code that *does* something. The feedback loop is different: an HTML mistake shows you the wrong shape on screen; a JavaScript mistake shows you nothing, or an error nobody bothered to explain, or -- worst -- wrong output that looks plausible.

Debugging is the new skill this week. And debugging is a skill AI cannot hand to you. AI can suggest a fix, but only a human who has stared at a broken loop for ten minutes develops the instinct to spot the off-by-one before the debugger does. You are growing that instinct by suffering through it.

A small AI bump (10 -> 15 -> 20%) is allowed because by Day 4 you will be tired, and "I tried for fifteen minutes, here is what I tried, what am I missing?" is a legitimate question.

---

## What You Will Feel This Week

- You will write an `if` statement that looks right and does not run.
- You will write a loop that never ends and have to `Ctrl+C` your terminal.
- You will confuse `=`, `==`, and `===` and not realise for twenty minutes.
- You will be sure your code is correct and the computer is wrong. The computer is not wrong.
- The number guessing game at the end of the week will click for you all at once, in the middle of debugging something unrelated. That is how learning works.

---

## What You MUST Do Manually (80%)

### Day 1 -- Dev environment + Node setup
- Install Node.js yourself (already done in Week 1 -- verify with `node -v`).
- Create a `week3/` folder. Every exercise in its own `.js` file. Run each one with `node filename.js`.
- No REPL tricks, no "just paste it in an online sandbox". Files on disk, runs from the terminal.

### Day 2 -- Variables and data types
- `let`, `const`, `var` -- know all three and know which to use. Write at least 15 tiny exercises using each.
- Types: `string`, `number`, `boolean`, `null`, `undefined`, `object`, `array`. Write code that produces each type and logs its `typeof`.
- Template literals: type at least 10 examples with backticks and `${}` substitution.

### Day 3 -- Operators and expressions
- Arithmetic: `+`, `-`, `*`, `/`, `%`. Write a small calculator that chains them.
- Comparison: `===`, `!==`, `<`, `>`, `<=`, `>=`. Write five comparisons that return unexpected results, then explain why.
- Logical: `&&`, `||`, `!`. Write short-circuit examples and trace the evaluation order by hand on paper.
- String concatenation vs template literals -- compare both styles on the same output.

### Day 4 -- Control flow
- `if`/`else if`/`else` -- five exercises, each with at least three branches.
- `switch` -- write one, then convert it to a chain of `if` statements. Which is clearer?
- `for`, `while`, `do...while` -- write each one from scratch. Write a `for` loop that counts down. Write a `while` loop that reads input. Write a `do...while` that runs at least once even if the condition is false.
- Nested loops: print a multiplication table, a triangle of stars, and FizzBuzz. All three, all manual.

### Day 5 and weekend -- Number guessing game
- Build the game from the curriculum brief entirely by hand. No AI for the first version.
- Must handle: user input, random number generation, comparison, retry loop, win/lose messages, score tracking across rounds.
- Commit after each working feature. At least 6 commits by Monday.

---

## You Must Break Things On Purpose

Three new breaks this week:

- Change `===` to `=` in an `if` and see what JavaScript does. (It assigns. Your condition is now always truthy.)
- Remove the increment from a `for` loop. Let it hang, then kill it with Ctrl+C. You will never forget.
- Use a variable before declaring it with `const`. Read the error message.

Feel how different each kind of bug is. You will recognise the shapes later.

---

## What You CAN Use AI For (20%)

Three permitted uses this week:

1. **Explaining error messages you have already tried to understand.**
2. **Generating one alternative implementation of something that already works** (comparison pattern from Week 2).
3. **Extending the number guessing game with one stretch feature** after the base game is complete.

### Good vs bad prompts this week

**Bad:** "Write me a number guessing game in JavaScript."
**Good:** "My number guessing game works but when the user guesses right, the game does not exit. Here is my code [paste]. I think my `while` loop condition is wrong. Is it?"

**Bad:** "How do loops work in JavaScript?"
**Good:** "I wrote this `for` loop to count down from 10 to 1 [paste] and it only prints once. What is my mistake?"

**Bad:** "Debug this for me."
**Good:** "I expected `let x = 5; x += 2; console.log(x)` to print 7, and it does. But when I change `+=` to `==`, it prints `true`. Why does `==` not throw an error?"

The key phrase you must include in every prompt this week: **"I expected X, I got Y."** This forces you to articulate what you think is true, which is where learning happens.

---

## The 15-Minute Rule

Before asking AI anything this week, spend at least **fifteen minutes** of real effort. Not ten. Fifteen.

Fifteen minutes of:

1. Re-reading your code out loud, one line at a time.
2. Adding `console.log` to print the value of every variable inside the suspicious area.
3. Running the code with a smaller input (for example, a loop from 1 to 3 instead of 1 to 100).
4. Searching the error message in a search engine, not in AI.
5. Reading the MDN page for the function or operator you are using.

After fifteen minutes of this, you have earned the right to ask AI. The question will be better for having tried. Next week this becomes twenty.

---

## Things AI Is Bad At This Week

- **Your specific off-by-one.** AI will suggest generic fixes to loops. Only you know whether your loop should go to `array.length` or `array.length - 1`. Count on paper.
- **Silent type coercion.** `"5" == 5` is `true` in JavaScript. AI will often not catch a bug caused by this unless you tell it you suspect type coercion. Use `===` always.
- **Your mental model.** If you think a variable is a number but it is actually a string, AI cannot know. It will trust your framing. Log the type before you trust it.
- **The vibe of the bug.** Some bugs are "logic wrong". Some bugs are "I misread a character". Some bugs are "the file is not even running". AI will confidently try to fix the first kind when it is actually the third. Check your terminal output before you trust any fix.

---

## Core Mental Models For This Week

- **A variable is a labelled box.** `let name = "Wanjiru"` puts the string `"Wanjiru"` in a box labelled `name`. `name = "Brian"` replaces what is in the box.
- **A condition is a fork in the road.** `if (x > 10) { ... } else { ... }` asks one question, then the program takes exactly one of the two roads.
- **A loop is a repetition with an exit.** Every loop needs a way to stop. If you cannot point at the exit condition, the loop will run forever.
- **A function is a named recipe.** Not this week's main focus, but you will start writing `function doThing()` helpers in the guessing game. Treat each one as a tiny named recipe with ingredients and a result.

When something feels wrong, come back to these four. 90% of this week's bugs are "I confused two of these models".

---

## This Week's AI Audit Focus

For every AI interaction this week, answer one extra question in your `AI_AUDIT.md`:

> **What did I try in the 15 minutes before asking?**

Facilitators read this closely. "I tried for 15 minutes" without a specific list of attempts is a red flag. Real answers look like: "I added console.log on lines 4 and 8, I changed the loop condition from `i < 10` to `i <= 10`, I searched 'javascript for loop counting wrong' on Google, I re-read the operators page on MDN."

---

## Assessment

The Week 3 assessment is a whiteboard + code walkthrough:

- Draw a flowchart of your number guessing game. Arrows, decisions, loop-backs. Paper, not code.
- Walk through one iteration of the game on the whiteboard: what variables exist, what values they hold, what line runs next.
- Live debug: the facilitator introduces one bug into your game (removes a `break`, flips a comparison, renames a variable). You have five minutes to find and fix it.
- Explain the difference between `==` and `===` in thirty seconds.
- Explain why your `while` loop does not run forever.

**No AI during the assessment.** If you cannot trace through your own game by hand, you built it with too much help.

### Live Rebuild Check

You may be asked to delete one function from your game and rewrite it from memory. If you cannot, take note and come back next week with the rewrite working.

---

## One Sentence To Remember

"The computer is never wrong. Your mental model is." Every bug this week is a mismatch between what you thought would happen and what actually happened. Train the model, not the keyboard.
