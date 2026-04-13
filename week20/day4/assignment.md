# Week 20 - Day 4 Assignment

## Title
Broadcasts, Scheduled Sends, And The Week 20 Ship Checklist

## Overview
Day 4 is pre-weekend polish. Today you add a broadcast command that sends a message to all subscribers, use node-cron to schedule a daily message, and finalise the Week 20 ship checklist.

## Learning Objectives Assessed
- Persist subscribers in a database
- Send bulk messages while respecting rate limits
- Schedule a job with node-cron
- Handle failures gracefully (blocked users, etc.)

## Prerequisites
- Days 1-3 completed

## AI Usage Rules

**Ratio:** 40/60. **Habit:** New channel. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Cron scheduling, rate limit helpers.
- **NOT ALLOWED FOR:** Deciding the rate limit numbers.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Subscribers table

**What to do:**
```sql
CREATE TABLE telegram_subscribers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  chat_id BIGINT NOT NULL UNIQUE,
  joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  blocked BOOLEAN NOT NULL DEFAULT FALSE
);
```

When a user sends `/start`, insert them (upsert). When they send `/stop`, mark them blocked.

**Expected output:**
Subscribers persisted.

### Task 2: Broadcast command

**What to do:**
Implement `/broadcast <message>` (admin-only). It reads all non-blocked subscribers and sends the message. Rate-limited to 30 messages/second (Telegram's rough limit).

```javascript
async function broadcast(text) {
  const { rows } = await pool.query(
    "SELECT chat_id FROM telegram_subscribers WHERE blocked = false"
  );

  for (const row of rows) {
    try {
      await axios.post(`${TELEGRAM_API}/sendMessage`, {
        chat_id: row.chat_id,
        text,
      });
      await new Promise((r) => setTimeout(r, 35)); // ~30/sec
    } catch (err) {
      if (err.response?.data?.error_code === 403) {
        // User blocked the bot
        await pool.query(
          "UPDATE telegram_subscribers SET blocked = true WHERE chat_id = $1",
          [row.chat_id]
        );
      } else {
        console.error("Broadcast error:", err.message);
      }
    }
  }
}
```

**Expected output:**
Broadcast reaches multiple subscribers. Blocked users are detected.

### Task 3: Scheduled daily message

**What to do:**
Install node-cron:

```bash
npm install node-cron
```

Add a scheduler to your Express startup:

```javascript
const cron = require("node-cron");

cron.schedule("0 8 * * *", async () => {
  await broadcast("Good morning! Today's special: ...");
});
```

For testing, temporarily change to `"*/1 * * * *"` (every minute) to see it fire.

**Expected output:**
Scheduled job fires. Screenshot `day4-cron-fired.png`.

### Task 4: Pre-weekend checklist

**What to do:**
Create `CHECKLIST.md`:

```markdown
## Week 20 Day 4 Pre-Weekend Checklist

- [ ] Bot handshake complete
- [ ] Inline keyboards working
- [ ] Callback queries answered
- [ ] Group detection working
- [ ] Admin check function
- [ ] Kick and mute commands
- [ ] Subscribers table with upsert
- [ ] Broadcast command rate-limited
- [ ] Blocked subscribers detected on 403
- [ ] Daily cron job scheduled
- [ ] AI_AUDIT.md current
```

Tick honestly.

**Expected output:**
`CHECKLIST.md` committed.

### Task 5: Rate limit notes

**What to do:**
In `rate-limit-notes.md`, write 4-6 sentences:
- What is Telegram's official rate limit?
- What happens if you exceed it? (Hint: 429 too many requests.)
- How is this different from WhatsApp's rate limit?
- How would you scale broadcasts past the per-second limit? (Hint: batching, queues, multiple bots.)

**Expected output:**
`rate-limit-notes.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Use a queue (BullMQ preview) to decouple broadcast from the request.
- Add a "confirm before broadcast" step so a typo does not reach everyone.
- Track delivery status in a `broadcast_logs` table.

## Submission Requirements

- **What to submit:** Repo, CHECKLIST, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Subscribers table with upsert | 15 | /start and /stop work. |
| Broadcast with rate limit | 25 | Respects 30/sec cap. |
| 403 blocked detection | 15 | Blocked users flagged in DB. |
| Scheduled daily cron | 20 | Fires on schedule. |
| CHECKLIST complete | 10 | Honest. |
| Rate limit notes | 10 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Broadcasting without rate limiting.** You will get 429 and be throttled.
- **Storing the cron schedule in code.** Make it configurable per environment.
- **Not marking blocked users.** You will keep hitting the API and getting 403 forever.

## Resources

- Day 4 reading: [Broadcasts and Scheduled Sends.md](./Broadcasts%20and%20Scheduled%20Sends.md)
- Week 20 AI boundaries: [../ai.md](../ai.md)
- node-cron: https://www.npmjs.com/package/node-cron
