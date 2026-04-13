# Week 30, Day 1: Testing and QA

> **AI boundaries this week:** 20% manual / 80% AI. Habit: *Engineer-led, AI-powered -- you drive, AI accelerates.* See [ai.md](../ai.md).

By the end of today, the Capstone is hardened: critical paths have automated tests, a manual QA checklist catches the things tests cannot, and a staging environment mirrors production.

**Estimated time:** 4 hours.

---

## The Critical Paths

Five paths must work perfectly. Everything else is nice to have.

1. **Signup** -- create a tenant and land on the dashboard.
2. **Order via USSD** -- dial, buy, pay, get confirmation.
3. **Order via WhatsApp** -- message, reply, buy, pay.
4. **Dashboard view** -- owner sees the order appear and can act on it.
5. **Multi-tenant isolation** -- no data bleed under any attack.

Every test, every rehearsal, every manual QA check should start and end with one of these five. Everything else is distraction.

---

## Automated Tests That Matter

From Week 28 you have unit tests and the isolation suite. Add the missing pieces:

### End-to-end signup test

```javascript
test("full signup flow creates all required rows", async () => {
  const res = await request(app)
    .post("/api/auth/signup")
    .send({ companyName: "E2E Test", email: "e2e@test.com", password: "password123" });
  assert.equal(res.status, 200);

  const { rows: tenant } = await pgClient.query("SELECT id FROM tenants WHERE slug LIKE 'e2e-test%'");
  assert.ok(tenant[0]);

  const { rows: wallet } = await pgClient.query("SELECT balance_cents FROM wallets WHERE tenant_id = $1", [tenant[0].id]);
  assert.equal(wallet[0].balance_cents, 0);

  const { rows: member } = await pgClient.query("SELECT role FROM tenant_members WHERE tenant_id = $1", [tenant[0].id]);
  assert.equal(member[0].role, "owner");
});
```

### Order flow test (without real payment)

```javascript
test("creating an order with stock decrements the product", async () => {
  const tenant = await createTestTenant();
  const token = loginAs(tenant.ownerUserId, tenant.id);

  const product = await request(app)
    .post("/api/products")
    .set("Authorization", `Bearer ${token}`)
    .send({ sku: "TEST", name: "Test", priceCents: 1000, stock: 5 });

  const order = await request(app)
    .post("/api/orders")
    .set("Authorization", `Bearer ${token}`)
    .send({
      customerPhone: "+254711111111",
      items: [{ productId: product.body.data.id, quantity: 2 }],
      channel: "web",
    });
  assert.equal(order.status, 201);

  const { rows } = await pgClient.query("SELECT stock FROM products WHERE id = $1", [product.body.data.id]);
  assert.equal(rows[0].stock, 3);
});
```

### Payment callback test (with fake Daraja payload)

```javascript
test("a valid mpesa callback credits the wallet", async () => {
  const { tenant, order } = await setupPendingOrder(1000);

  const callback = {
    Body: {
      stkCallback: {
        CheckoutRequestID: order.mpesa_checkout_request_id,
        ResultCode: 0,
        CallbackMetadata: { Item: [
          { Name: "Amount", Value: 10 },  // KSh 10 = 1000 cents
          { Name: "MpesaReceiptNumber", Value: "TEST123" },
        ]},
      },
    },
  };

  await request(app).post("/webhook/mpesa/callback").send(callback).expect(200);

  await wait(500); // outbox worker
  const { rows } = await pgClient.query("SELECT balance_cents FROM wallets WHERE tenant_id = $1", [tenant.id]);
  assert.equal(rows[0].balance_cents, 1000);
});
```

Five to ten tests covering the happy paths + isolation. Not exhaustive. Sufficient for confidence.

---

## Manual QA Checklist

Tests miss UX. Print this and walk through it in a browser:

- [ ] Signup renders. Can you sign up?
- [ ] Dashboard loads with no errors. No 500s, no blank pages.
- [ ] Every nav link works.
- [ ] Empty states look friendly (Orders with zero orders shows the right message).
- [ ] Add a product form validates (negative price, empty sku, duplicate sku).
- [ ] Creating an order from the API reflects in the dashboard within 2 seconds.
- [ ] USSD demo flow works.
- [ ] WhatsApp demo flow works.
- [ ] Payment flow works.
- [ ] Confirmation WhatsApp arrives.
- [ ] Sign out -> sign in works.
- [ ] Mobile view (resize browser to 400px wide) is usable.
- [ ] Forgot password? (Probably not implemented; note in limitations.)
- [ ] Delete a tenant? (Probably not implemented; note.)
- [ ] Admin super-view? (Out of scope.)

Any unchecked box is a bug or a known limitation. Document the limitations in README.

---

## Staging Environment

A staging server matches production as closely as possible:

- Same Docker images as production (pulled from ghcr.io).
- Separate Postgres, separate Redis.
- Separate domain (e.g., `staging.smeos.co.ke`).
- Smaller resources if cost is a concern.
- The same `docker-compose.yml` with different env vars.

Use staging to rehearse the launch. Deploy there first, poke around for an hour, fix anything broken, then deploy to production.

---

## Performance Sanity

Run one last load test:

- 100 concurrent signups.
- 500 orders across 10 tenants in 2 minutes.
- 50 simultaneous USSD sessions.
- 50 simultaneous WhatsApp messages.

Watch for errors, latency spikes, queue depth. Record the numbers in PERFORMANCE.md.

If you see anything over 2 seconds p99 latency on normal requests, investigate. 500ms is the target; 1s is acceptable; 2s+ is a bug.

---

## Checkpoint

1. All automated tests pass.
2. Manual QA checklist has no red X marks.
3. Staging is deployed and runs the demo cleanly.
4. Performance numbers documented.

Commit:

```bash
git add .
git commit -m "test: capstone qa automated and manual"
```

---

## What You Learned

- Focus tests on the five critical paths.
- Manual QA catches UX issues tests cannot.
- Staging exists to catch deploy issues without risking production.
- Performance measured before launch is the only performance you trust.

Tomorrow: documentation.
