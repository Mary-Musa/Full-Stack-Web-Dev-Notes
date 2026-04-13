# Week 29 - Day 2 Assignment

## Title
WhatsApp Per Tenant

## Overview
Day 2 brings WhatsApp into the capstone. One shared webhook, per-tenant WABA (WhatsApp Business Account) config, and a shared worker that sends messages using the right tenant's credentials.

## Learning Objectives Assessed
- Store per-tenant WhatsApp credentials securely
- Route incoming webhooks to the right tenant
- Send messages using the tenant-scoped client
- Keep tokens out of logs

## Prerequisites
- Day 1 completed
- Week 13 WhatsApp setup

## AI Usage Rules

**Ratio this week:** 25% manual / 75% AI
**Habit:** Integrations week. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Webhook boilerplate.
- **NOT ALLOWED FOR:** Token handling and tenant resolution.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: whatsapp_configs table

**What to do:**
```sql
CREATE TABLE whatsapp_configs (
  tenant_id UUID PRIMARY KEY REFERENCES tenants(id),
  phone_number_id TEXT NOT NULL,
  access_token_encrypted TEXT NOT NULL,
  webhook_verify_token TEXT NOT NULL
);
```

Tokens are encrypted at rest with a local KMS helper (just AES-GCM with a key from env).

**Expected output:**
Encrypted storage.

### Task 2: Shared webhook with tenant resolution

**What to do:**
`POST /webhooks/whatsapp` receives messages. Resolve the tenant by the Meta `phone_number_id`:

```javascript
const { rows } = await pool.query("SELECT tenant_id FROM whatsapp_configs WHERE phone_number_id = $1", [phoneNumberId]);
```

Set RLS tenant and process the message within it.

**Expected output:**
Routing works.

### Task 3: Shared sender

**What to do:**
`sendWhatsApp(db, to, body)` uses `req.db` to find the tenant's decrypted token, then calls Meta. Never logs the token.

**Expected output:**
Sends work from both tenants.

### Task 4: Dedup

**What to do:**
Meta retries webhooks. Dedupe on `message_id` per tenant:

```sql
CREATE TABLE whatsapp_dedupe (
  tenant_id UUID NOT NULL,
  message_id TEXT NOT NULL,
  PRIMARY KEY (tenant_id, message_id)
);
```

**Expected output:**
Duplicates dropped silently.

### Task 5: Security notes

**What to do:**
`whatsapp-security.md`, 5-7 sentences:
- Why encrypt tokens at rest?
- Why per-tenant tokens and not one shared app?
- What is the verify challenge and why does Meta send it?
- What happens if a tenant revokes access?

Your own words.

**Expected output:**
Committed.

## Stretch Goals (Optional - Extra Credit)

- Rotate encryption keys without re-encrypting all rows.
- Send template messages via config.
- Meta business verification walkthrough in notes.

## Submission Requirements

- **What to submit:** Repo, webhook, sender, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 2

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| whatsapp_configs + encryption | 25 | Encrypted at rest. |
| Tenant resolution in webhook | 25 | Routes correctly. |
| Shared sender | 20 | No token logging. |
| Dedupe | 15 | Duplicates dropped. |
| Security notes | 10 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Token in plaintext in logs.** Scrub.
- **Single shared token across tenants.** No audit trail.
- **Dedupe on message text instead of message_id.** Not unique.

## Resources

- Day 2 reading: [WhatsApp Per Tenant.md](./WhatsApp%20Per%20Tenant.md)
- Week 29 AI boundaries: [../ai.md](../ai.md)
