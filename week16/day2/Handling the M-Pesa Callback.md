# Week 16, Day 2: Handling the M-Pesa Callback

By the end of today, the M-Pesa callback webhook is wired. When a customer completes an STK prompt, Safaricom calls your Express server, you parse the callback, update the order to `paid`, and the customer's "Awaiting Payment" page flips to "Confirmed" automatically. You will also handle the common failure cases: user cancelled, user timed out, wrong PIN.

**Prior-week concepts you will use today:**
- Yesterday's STK Push flow (Week 16, Day 1)
- Webhook verification patterns (Week 11, Day 2 -- HMAC for WhatsApp)
- The Week 12 order status state machine (Week 15, Day 4)

**Estimated time:** 3 hours

---

## The Callback Shape

When Daraja calls your webhook, the request body looks (roughly) like this:

```json
{
  "Body": {
    "stkCallback": {
      "MerchantRequestID": "29115-34620561-1",
      "CheckoutRequestID": "ws_CO_191220231115234523716",
      "ResultCode": 0,
      "ResultDesc": "The service request is processed successfully.",
      "CallbackMetadata": {
        "Item": [
          { "Name": "Amount", "Value": 1 },
          { "Name": "MpesaReceiptNumber", "Value": "NLJ7RT61SV" },
          { "Name": "TransactionDate", "Value": 20231220111523 },
          { "Name": "PhoneNumber", "Value": 254708374149 }
        ]
      }
    }
  }
}
```

On failure, `ResultCode` is non-zero and `CallbackMetadata` is missing. Common failure codes:

- `1032` -- Request cancelled by user.
- `1037` -- Timeout waiting for the customer to enter PIN.
- `1025` -- Wrong PIN.
- `2001` -- Insufficient funds.

The shape is pure 90s SOAP-style enterprise JSON -- arrays of `{Name, Value}` pairs instead of a plain object. Daraja has never improved it. You deal with it by parsing into a friendlier shape once, at the boundary.

---

## Parsing the Callback

Replace the stub in `server/services/payments.service.js`:

```javascript
function parseCallback(body) {
  const cb = body?.Body?.stkCallback;
  if (!cb) return null;

  const metadata = {};
  if (cb.CallbackMetadata?.Item) {
    for (const item of cb.CallbackMetadata.Item) {
      metadata[item.Name] = item.Value;
    }
  }

  return {
    checkoutRequestId: cb.CheckoutRequestID,
    merchantRequestId: cb.MerchantRequestID,
    resultCode: cb.ResultCode,
    resultDesc: cb.ResultDesc,
    amount: metadata.Amount,
    mpesaReceiptNumber: metadata.MpesaReceiptNumber,
    phoneNumber: metadata.PhoneNumber,
    transactionDate: metadata.TransactionDate,
  };
}

async function handleCallback(body) {
  const parsed = parseCallback(body);
  if (!parsed) {
    console.error("Invalid M-Pesa callback shape", body);
    return;
  }

  // Look up the order using checkoutRequestId
  const { rows } = await query(
    "SELECT id, subtotal_cents, payment_status FROM orders WHERE mpesa_checkout_request_id = $1",
    [parsed.checkoutRequestId]
  );
  const order = rows[0];

  if (!order) {
    console.warn("Callback for unknown checkoutRequestId:", parsed.checkoutRequestId);
    return;
  }

  // Idempotency: if we already processed this, skip
  if (order.payment_status === "success") {
    console.log("Duplicate callback, already processed");
    return;
  }

  if (parsed.resultCode === 0) {
    // Success
    // Sanity check: amount matches
    const expectedKsh = Math.ceil(order.subtotal_cents / 100);
    if (parsed.amount && parsed.amount !== expectedKsh) {
      console.error(`Amount mismatch for ${order.id}: expected ${expectedKsh}, got ${parsed.amount}`);
      // Do NOT mark paid. Flag for manual review.
      await query(
        "UPDATE orders SET payment_status = 'failed', updated_at = NOW() WHERE id = $1",
        [order.id]
      );
      return;
    }

    await query(
      `UPDATE orders
       SET payment_status = 'success',
           status = 'paid',
           mpesa_receipt_number = $1,
           updated_at = NOW()
       WHERE id = $2`,
      [parsed.mpesaReceiptNumber, order.id]
    );

    // Fire and forget: send the WhatsApp confirmation
    sendOrderConfirmation(order.id).catch((err) =>
      console.error("WhatsApp send failed:", err)
    );
  } else {
    // Failure
    const mappedStatus = mapFailureCode(parsed.resultCode);
    await query(
      `UPDATE orders SET payment_status = $1, updated_at = NOW() WHERE id = $2`,
      [mappedStatus, order.id]
    );
  }
}

function mapFailureCode(code) {
  if (code === 1032) return "cancelled"; // user cancelled
  if (code === 1037) return "cancelled"; // timeout
  return "failed";
}

async function sendOrderConfirmation(orderId) {
  // Real implementation in Day 3. For now, stub.
  console.log(`Would send WhatsApp confirmation for order ${orderId}`);
}

module.exports = { initiateStkPush, handleCallback, getStatus };
```

Four things you should internalise from this function.

**Idempotency.** If Daraja retries the callback (it sometimes does on timeouts), we detect that we already marked the order `success` and skip. Without this check, a retry could overwrite a later status change and confuse everyone. **Every webhook handler you ever write must be idempotent.** Memorise it.

**Amount verification.** We compare the amount Safaricom says was paid to the amount we asked for. If it does not match, we flag for review instead of marking paid. This catches one of the oldest M-Pesa scams: a customer pays KSh 1 for a KSh 10,000 item because they intercepted the STK push and edited it on a rooted phone. Daraja protects against this in the STK flow, but verifying is free and paranoid is the right posture for money.

**Failure mapping.** Different result codes mean different things to the user. "Cancelled" is recoverable (the user just changes their mind); "failed" is recoverable too but with an error message; some codes could mean "retry is fine" vs "do not retry". We only map the two most common failure codes today; more get added over time as you see them in real traffic.

**Fire-and-forget WhatsApp notification.** We do not `await` the WhatsApp send inside the callback handler. Daraja has a 10-second timeout and will retry if we are slow. We reply fast, then do the slow stuff in the background.

---

## Responding to Daraja

Daraja expects a specific response from your callback endpoint:

```json
{ "ResultCode": 0, "ResultDesc": "Accepted" }
```

Any other response (HTTP non-200, missing `ResultCode`, or `ResultCode != 0`) makes Daraja retry the callback up to three times. The controller already returns this format, but verify you are doing it correctly.

Also: **reply before doing any slow work.** The order of operations in the controller should be:

```javascript
async function handleCallback(req, res) {
  // 1. Reply immediately to Daraja
  res.json({ ResultCode: 0, ResultDesc: "Accepted" });

  // 2. Process the callback after replying
  try {
    await paymentsService.handleCallback(req.body);
  } catch (err) {
    console.error("callback processing failed:", err);
  }
}
```

Do not await the service before responding. If the service takes 3 seconds and Daraja retries at 2 seconds, you process the callback twice (idempotency saves you, but you do not want to rely on it in the hot path).

---

## The Awaiting-Payment Page Closes the Loop

With the callback wired, the flow completes itself:

1. Customer clicks Pay.
2. Server Action inserts order, calls Express STK endpoint, returns `checkoutRequestId`.
3. Client navigates to `/checkout/awaiting-payment`.
4. Client polls `/api/payments/status/:id` every 2s.
5. Customer enters PIN on their phone.
6. Daraja calls Express webhook with success.
7. Express updates `orders` row to `payment_status = 'success'`, `status = 'paid'`.
8. Next poll from the client sees `success`, navigates to `/checkout/confirmed`.
9. Confirmation page shows the order id.

Test it with the Daraja sandbox. The sandbox has a test phone number `254708374149` that auto-accepts the STK prompt with PIN `1234`. Walk through a full purchase; watch the status column in `psql` flip from `initiated` to `success`; watch the awaiting page navigate to confirmed. The first time this works is genuinely satisfying -- you built real money movement.

---

## Failure Scenarios To Test

Hit each of these before moving on.

**1. Customer cancels.** On the sandbox simulator, click "Cancel" instead of accepting. The callback comes with `ResultCode: 1032`. Your code maps to `cancelled`. The page shows "Payment failed or cancelled" and offers a retry button.

**2. Customer types wrong PIN.** Sandbox does not easily simulate this but you can craft a fake callback and POST it to your endpoint with curl to test. Your code should mark the order `failed` and the page should say so.

**3. Daraja retries.** POST the same callback twice. The second time should log "Duplicate callback, already processed" and not change anything.

**4. Amount mismatch.** Craft a callback with a wrong amount and POST it. The order should go to `failed` with a warning in logs.

**5. Timeout.** Start a checkout and do not respond to the STK prompt. After about 60 seconds, Daraja times out. You should see `ResultCode: 1037`. Your code maps it to `cancelled`.

**6. Callback for unknown checkout id.** POST a callback with a random `CheckoutRequestID`. Your code should log a warning and not crash.

Write a small test file `server/test/callback-test.js` that POSTs fake callbacks to your local endpoint and asserts the resulting row status. Check it in.

---

## Admin Retry and Failed-Payment Recovery

Occasionally a payment succeeds on Safaricom's side but your callback never arrives (networks are weird, ngrok tunnels die). The order sits in `payment_status = 'initiated'` forever. Admins need a way to reconcile.

Add a "Retry payment check" button on the admin order detail page. The underlying Server Action calls a new Express endpoint:

```javascript
// server/controllers/payments.controller.js
async function queryMpesaStatus(req, res) {
  const { orderId } = req.body;
  const result = await paymentsService.queryDarajaStatus(orderId);
  res.json(result);
}
```

And in the service:

```javascript
async function queryDarajaStatus(orderId) {
  const { rows } = await query(
    "SELECT mpesa_checkout_request_id FROM orders WHERE id = $1",
    [orderId]
  );
  const checkoutId = rows[0]?.mpesa_checkout_request_id;
  if (!checkoutId) return { error: "No STK push recorded" };

  const token = await getDarajaToken();
  const timestamp = new Date().toISOString().replace(/[-T:.Z]/g, "").slice(0, 14);
  const password = Buffer.from(env.MPESA_SHORTCODE + env.MPESA_PASSKEY + timestamp).toString("base64");

  const res = await axios.post(
    "https://sandbox.safaricom.co.ke/mpesa/stkpushquery/v1/query",
    {
      BusinessShortCode: env.MPESA_SHORTCODE,
      Password: password,
      Timestamp: timestamp,
      CheckoutRequestID: checkoutId,
    },
    { headers: { Authorization: `Bearer ${token}` } }
  );

  // Translate Daraja response to our status
  const status = res.data.ResultCode === "0" ? "success" : "failed";
  await query(
    "UPDATE orders SET payment_status = $1, updated_at = NOW() WHERE id = $2",
    [status, orderId]
  );
  return { paymentStatus: status };
}
```

This is Daraja's "STK Push Query" API -- you give it a checkout id and it tells you the authoritative state. Use it as a last resort when a callback goes missing. This is your reconciliation tool.

---

## Checkpoint

1. Sandbox happy path: buy a product, complete STK, order flips to `paid`, customer sees confirmation.
2. Cancelled STK leaves the order in `payment_status = 'cancelled'`, customer sees retry button.
3. Fake duplicate callback (POST the same body twice) does not crash and does not change the row after the first success.
4. Fake wrong-amount callback marks the order `failed` with a warning log.
5. Order detail admin page has a "Check with M-Pesa" button that calls the status query API and updates the row.
6. Daraja retries when you purposely return HTTP 500 from the callback (verify in Express logs).

Commit:

```bash
git add .
git commit -m "feat: mpesa callback with idempotency amount check and admin reconciliation"
```

---

## What You Learned

- Webhook handlers must be idempotent. Check "already processed" first.
- Verify the amount on every payment callback.
- Reply to the payment provider *before* doing slow work.
- Map failure codes to user-meaningful statuses.
- Expose a reconciliation endpoint (STK Push Query) for admin recovery.

Tomorrow we wire the WhatsApp order confirmation. The customer will get a formatted message on their phone the moment their payment is confirmed.
