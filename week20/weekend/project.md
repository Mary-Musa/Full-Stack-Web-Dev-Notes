# Week 20 Weekend: A Real Community Bot

Pick a real group you are part of. Build a Telegram bot that solves one specific problem that group has. By Monday, a member of that group has used it for something real.

**Estimated time:** 6-8 hours.

**Deadline:** Monday morning.

---

## Pick A Community And A Problem

Like Week 13's USSD project, this is not hypothetical. Options:

- **A chama / sacco** -- log contributions, show balances.
- **A cohort WhatsApp group** -- schedule study sessions, share resources.
- **A church youth fellowship** -- event RSVPs, rota management.
- **A startup team** -- standup summaries, support ticket triage.
- **A fitness group** -- workout logging, streak tracking.

Ask the admin of the group what one thing eats the most of their time. That is your feature.

---

## Requirements

### Functional
- The bot runs in a real Telegram group (not just a private chat).
- At least 3 commands that do something useful.
- Inline keyboards for at least one flow.
- Data persisted to Postgres.
- At least one cron-scheduled reminder or summary.
- Admin-only commands exist and are enforced.
- Welcome message for new members.

### Technical
- Code lives in the same `server/` project as everything else, under `services/telegram/`.
- Per-group settings in a table; bot serves multiple groups from one instance.
- Rate-limited broadcasts.
- Idempotent on all mutations.
- `.env.example` lists required env vars.
- `README.md` describes the community, the problem, and the solution.

### Shipped
- The bot is deployed to a server or tunnelled via ngrok persistently through Monday.
- At least one real user from the target community has used it.

---

## Grading Rubric (100 pts)

| Area | Points |
|---|---|
| Real community fit | 15 |
| 3+ useful commands | 20 |
| Inline keyboard flow | 10 |
| Persistent data | 10 |
| Scheduled reminder | 10 |
| Admin commands + enforcement | 10 |
| Rate-limited broadcasts | 5 |
| Multi-group support | 5 |
| README quality | 10 |
| Proof of real use | 5 |

---

## Hints

**Do not build generic features.** Everything should answer a question only that community asked. A chama bot's `/contribute` makes sense; a chama bot's `/weather` does not.

**Keep the command list short.** 3 good commands beat 10 mediocre ones. If you can explain all features in a 30-second pitch, the scope is right.

**Test with real people.** Invite one member of the target community to the bot. Watch them try it. Note every confusion. Fix them.

**Screenshots, not words.** The README should have a screenshot of every command in use, in a real group.

Next week is cron in depth, and the week after is Project 4.
