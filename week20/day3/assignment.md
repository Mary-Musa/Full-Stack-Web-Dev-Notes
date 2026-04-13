# Week 20 - Day 3 Assignment

## Title
Group Management -- Admin Commands, Mute, Kick

## Overview
Today you deploy the bot to a Telegram group (yours or a test one) and build admin-only commands: mute a user, kick a user, pin a message. The bot must be a group admin to do most of these.

## Learning Objectives Assessed
- Promote the bot to group admin
- Detect whether a command comes from a group or private chat
- Check if the sender is a group admin
- Call moderation APIs (kickChatMember, restrictChatMember)

## Prerequisites
- Days 1-2 completed

## AI Usage Rules

**Ratio:** 40/60. **Habit:** New channel. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Repeated routing logic.
- **NOT ALLOWED FOR:** Admin check logic -- auth-adjacent.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Promote the bot

**What to do:**
1. Create a Telegram group (or use an existing test group).
2. Add your bot to it as a member.
3. Promote the bot to admin (tap bot > Promote to admin). Grant: can delete messages, can restrict members.

**Expected output:**
Screenshot `day3-promoted.png`.

### Task 2: Detect group context

**What to do:**
In your webhook handler, check `message.chat.type`:

```javascript
const chat = message.chat;
if (chat.type === "group" || chat.type === "supergroup") {
  // Group-only behaviour
}
```

Only respond to `/admin` commands when in a group.

**Expected output:**
Private chat and group chat handled differently.

### Task 3: Check if sender is admin

**What to do:**
Before running moderation commands, verify the sender is a group admin:

```javascript
async function isAdmin(chatId, userId) {
  const res = await axios.get(`${TELEGRAM_API}/getChatMember`, {
    params: { chat_id: chatId, user_id: userId },
  });
  const status = res.data.result.status;
  return status === "creator" || status === "administrator";
}
```

**Expected output:**
Function works. Non-admins are refused.

### Task 4: Kick command

**What to do:**
Implement `/kick` (must be a reply to a message):

```javascript
if (text === "/kick" && message.reply_to_message) {
  const target = message.reply_to_message.from.id;
  if (!(await isAdmin(chat.id, message.from.id))) {
    await axios.post(`${TELEGRAM_API}/sendMessage`, {
      chat_id: chat.id,
      text: "Only admins can kick.",
    });
    return res.sendStatus(200);
  }
  await axios.post(`${TELEGRAM_API}/banChatMember`, {
    chat_id: chat.id,
    user_id: target,
  });
  await axios.post(`${TELEGRAM_API}/sendMessage`, {
    chat_id: chat.id,
    text: `User kicked.`,
  });
}
```

Hand-type it. Admin check is auth-adjacent.

**Expected output:**
Kick works when an admin runs it; refused otherwise.

### Task 5: Mute command

**What to do:**
Implement `/mute 10` (mute for 10 minutes). Use `restrictChatMember` with an `until_date` timestamp.

**Expected output:**
Mute works.

## Stretch Goals (Optional - Extra Credit)

- Add `/pin` to pin a replied message.
- Add `/warn` that stores warnings in your DB; auto-mute after 3 warnings.
- Log every mod action to an audit table.

## Submission Requirements

- **What to submit:** Repo, screenshots, `AI_AUDIT.md`.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Bot promoted to group admin | 10 | Screenshot. |
| Group vs private detection | 15 | Admin commands refuse in private chat. |
| Admin check function | 20 | Hand-typed. Works. |
| Kick command | 30 | Full flow including permission check. |
| Mute command | 15 | Temporary mute with until_date. |
| Clean commits | 10 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Banning instead of kicking.** Telegram's `banChatMember` is "kick + ban". To just kick, call ban then immediately unban.
- **Not handling the reply_to_message check.** Without it, `/kick` with no target crashes or kicks the sender.
- **Trusting the `from` field.** Always re-check with getChatMember for admin status.

## Resources

- Day 3 reading: [Group Management.md](./Group%20Management.md)
- Week 20 AI boundaries: [../ai.md](../ai.md)
