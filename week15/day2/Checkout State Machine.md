# Week 15, Day 2: The Checkout State Machine

By the end of today, the checkout flow is a formal state machine with five states, explicit transitions, and a multi-step UI that feels linear to the user but is safe to navigate forward and backward. You still will not be taking real money -- that is Week 16. Today is about getting the shape right.

**Prior-week concepts you will use today:**
- The cart Zustand store (Week 15, Day 1)
- State machines for USSD (Week 13, Day 2) -- same pattern, different UI
- Client Components (Week 14, Day 1)

**Estimated time:** 3 hours

---

## The Five States

```
  BROWSING  ->  CART  ->  INFO  ->  PAYMENT  ->  CONFIRMED
                  ^         |         |              |
                  |         v         v              |
                  +----  (back)  ---  (back) --------+
```

- **BROWSING** -- the user is on a product page. Cart exists but they are still shopping.
- **CART** -- the `/cart` page. Items listed, quantities editable.
- **INFO** -- first checkout step: name, phone, delivery address.
- **PAYMENT** -- second checkout step: pick payment method, type a reference, click pay.
- **CONFIRMED** -- the "thank you, order 1234" page.

Transitions:
- `cart_open` (BROWSING -> CART)
- `proceed_info` (CART -> INFO)
- `proceed_payment` (INFO -> PAYMENT)
- `pay_submit` (PAYMENT -> CONFIRMED)
- `back` (any -> previous)
- `cancel` (any -> BROWSING, clears checkout context)

This is the cleanest version. In real apps you will add variants: INFO might split into SHIPPING and BILLING, PAYMENT might offer multiple providers with their own sub-states, CONFIRMED might branch into PENDING and FAILED for async payments. Today we build the backbone.

### Why a state machine and not just pages

Each checkout step is a full Next.js page (`/checkout/info`, `/checkout/payment`, etc). Why do we need "states" if pages are already states?

Three reasons:

1. **Invariants.** A user should not be able to jump straight to `/checkout/payment` without first filling in `/checkout/info`. The state machine enforces that by checking the current state before rendering the payment page.
2. **Context.** The info page collects name/phone/address. The payment page needs those values. Without a shared store, you would have to pass them through URLs or cookies.
3. **Back navigation.** Browser back buttons are confusing on multi-step forms. A state machine defines what "back" means at each step explicitly.

---

## The Checkout Store

Add a second Zustand store for checkout state. Create `app/lib/checkoutStore.js`:

```javascript
// app/lib/checkoutStore.js
"use client";

import { create } from "zustand";
import { persist } from "zustand/middleware";

const STATES = {
  BROWSING: "browsing",
  CART: "cart",
  INFO: "info",
  PAYMENT: "payment",
  CONFIRMED: "confirmed",
};

const initial = {
  state: STATES.BROWSING,
  info: { name: "", phone: "", address: "" },
  paymentMethod: null,
  orderId: null,
};

export const useCheckout = create(
  persist(
    (set, get) => ({
      ...initial,
      STATES,

      goTo(nextState) {
        // Only allowed transitions. Guard against jumps.
        const allowed = {
          [STATES.BROWSING]: [STATES.CART],
          [STATES.CART]: [STATES.INFO, STATES.BROWSING],
          [STATES.INFO]: [STATES.PAYMENT, STATES.CART],
          [STATES.PAYMENT]: [STATES.CONFIRMED, STATES.INFO],
          [STATES.CONFIRMED]: [STATES.BROWSING],
        };
        const current = get().state;
        if (!allowed[current]?.includes(nextState)) {
          console.warn(`invalid checkout transition: ${current} -> ${nextState}`);
          return false;
        }
        set({ state: nextState });
        return true;
      },

      setInfo(info) {
        set({ info: { ...get().info, ...info } });
      },

      setPaymentMethod(method) {
        set({ paymentMethod: method });
      },

      setOrderId(orderId) {
        set({ orderId });
      },

      reset() {
        set(initial);
      },
    }),
    {
      name: "mctaba-checkout",
      partialize: (s) => ({
        state: s.state,
        info: s.info,
        paymentMethod: s.paymentMethod,
        orderId: s.orderId,
      }),
    }
  )
);
```

The `goTo` function is the heart of the machine. It checks a table of allowed transitions and refuses invalid ones. This is exactly the pattern from Week 13's USSD dispatcher, just in-memory instead of in Redis because checkout is a per-browser concern.

### A note on persisting checkout

Checkout state is persisted to `localStorage` (same as cart) so that a user who closes their laptop mid-checkout and comes back later is not forced to restart. The risk: stale state. If prices have changed, stock has sold out, or the user abandoned three days ago, resuming is wrong.

Two mitigations:

1. On the `CONFIRMED` step, always `reset()`. A finished checkout should leave no trace.
2. On re-hydration, if the checkout state is older than a day, reset it automatically. We will add that later -- today just be aware.

---

## The Info Step

Create `app/checkout/info/page.js`:

```jsx
// app/checkout/info/page.js
import InfoForm from "./InfoForm";

export const metadata = { title: "Checkout: Your Info" };

export default function CheckoutInfoPage() {
  return (
    <div className="max-w-md mx-auto p-8">
      <h1 className="text-2xl font-bold mb-6">Delivery Information</h1>
      <p className="text-sm text-gray-500 mb-4">Step 1 of 3</p>
      <InfoForm />
    </div>
  );
}
```

Create `app/checkout/info/InfoForm.js`:

```jsx
// app/checkout/info/InfoForm.js
"use client";
import { useRouter } from "next/navigation";
import { useEffect, useState } from "react";
import { useCheckout } from "@/app/lib/checkoutStore";
import { useCart } from "@/app/lib/cartStore";

export default function InfoForm() {
  const router = useRouter();
  const info = useCheckout((s) => s.info);
  const setInfo = useCheckout((s) => s.setInfo);
  const goTo = useCheckout((s) => s.goTo);
  const STATES = useCheckout((s) => s.STATES);
  const state = useCheckout((s) => s.state);
  const items = useCart((s) => s.items);

  const [local, setLocal] = useState(info);
  const [hydrated, setHydrated] = useState(false);

  useEffect(() => {
    setHydrated(true);
    // Pull latest from store once hydrated
    setLocal(info);

    // Guard: if cart is empty, redirect
    if (items.length === 0) {
      router.replace("/cart");
      return;
    }

    // Guard: only allow this page from CART or PAYMENT states
    if (state !== STATES.INFO && state !== STATES.CART) {
      goTo(STATES.INFO);
    }
  }, [hydrated]);

  function handleSubmit(e) {
    e.preventDefault();
    if (!local.name || !local.phone || !local.address) return;
    setInfo(local);
    if (goTo(STATES.PAYMENT)) {
      router.push("/checkout/payment");
    }
  }

  if (!hydrated) return <p>Loading...</p>;

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <label className="block">
        <span className="text-sm font-medium">Full name</span>
        <input
          type="text"
          required
          value={local.name}
          onChange={(e) => setLocal({ ...local, name: e.target.value })}
          className="w-full border p-2 rounded mt-1"
        />
      </label>
      <label className="block">
        <span className="text-sm font-medium">Phone number</span>
        <input
          type="tel"
          required
          pattern="\+?\d{9,14}"
          placeholder="+254712..."
          value={local.phone}
          onChange={(e) => setLocal({ ...local, phone: e.target.value })}
          className="w-full border p-2 rounded mt-1"
        />
      </label>
      <label className="block">
        <span className="text-sm font-medium">Delivery address</span>
        <textarea
          required
          rows={3}
          value={local.address}
          onChange={(e) => setLocal({ ...local, address: e.target.value })}
          className="w-full border p-2 rounded mt-1"
        />
      </label>
      <button type="submit" className="w-full bg-brand text-white py-3 rounded">
        Continue to payment
      </button>
    </form>
  );
}
```

A few things worth explaining.

**Local state `local` with a sync from store.** We do not write directly to the store on every keystroke because that would trigger store subscribers (like the header) to re-render on every letter. Instead we keep a local copy, write to the store on submit. Reduced churn, same result.

**Guard: redirect to `/cart` if empty.** If a user bookmarks `/checkout/info` and comes back with an empty cart, they should go to `/cart`, not see a form for an invisible order.

**Guard: reset to `INFO` state on arrival.** If the store says we are in `CONFIRMED` but the user navigated to `/checkout/info`, the transition table rejects jumping backwards -- `CONFIRMED -> INFO` is not allowed. In that case the user is probably refreshing an old confirmed order; redirect them home. You can add that check if you want.

**`pattern="\+?\d{9,14}"`** is HTML5 form validation for phone numbers. Not bulletproof but catches obvious mistakes. The server will re-validate in Day 3 when we submit.

---

## The Payment Step

Create `app/checkout/payment/page.js` and `app/checkout/payment/PaymentForm.js`:

```jsx
// app/checkout/payment/page.js
import PaymentForm from "./PaymentForm";

export const metadata = { title: "Checkout: Payment" };

export default function PaymentPage() {
  return (
    <div className="max-w-md mx-auto p-8">
      <h1 className="text-2xl font-bold mb-6">Payment</h1>
      <p className="text-sm text-gray-500 mb-4">Step 2 of 3</p>
      <PaymentForm />
    </div>
  );
}
```

```jsx
// app/checkout/payment/PaymentForm.js
"use client";
import { useRouter } from "next/navigation";
import { useEffect, useState } from "react";
import { useCheckout } from "@/app/lib/checkoutStore";
import { useCart } from "@/app/lib/cartStore";

export default function PaymentForm() {
  const router = useRouter();
  const info = useCheckout((s) => s.info);
  const STATES = useCheckout((s) => s.STATES);
  const state = useCheckout((s) => s.state);
  const goTo = useCheckout((s) => s.goTo);
  const setPaymentMethod = useCheckout((s) => s.setPaymentMethod);
  const setOrderId = useCheckout((s) => s.setOrderId);
  const items = useCart((s) => s.items);
  const totalCents = useCart((s) => s.totalCents());
  const clearCart = useCart((s) => s.clear);
  const [method, setMethod] = useState("mpesa");
  const [submitting, setSubmitting] = useState(false);
  const [hydrated, setHydrated] = useState(false);

  useEffect(() => {
    setHydrated(true);
    if (items.length === 0) router.replace("/cart");
    if (!info.name || !info.phone) router.replace("/checkout/info");
  }, [hydrated]);

  async function handlePay() {
    setSubmitting(true);
    setPaymentMethod(method);
    // Day 3 we call a Server Action here. For today, fake it.
    await new Promise((r) => setTimeout(r, 1200));
    const fakeOrderId = `ORD-${Date.now().toString().slice(-6)}`;
    setOrderId(fakeOrderId);
    clearCart();
    goTo(STATES.CONFIRMED);
    router.push("/checkout/confirmed");
  }

  if (!hydrated) return <p>Loading...</p>;

  return (
    <div className="space-y-6">
      <div className="border rounded p-4">
        <h2 className="font-medium mb-2">Order summary</h2>
        <p className="text-sm text-gray-600">{items.length} item(s)</p>
        <p className="text-xl font-bold mt-1">
          KSh {(totalCents / 100).toLocaleString()}
        </p>
      </div>

      <div className="border rounded p-4">
        <h2 className="font-medium mb-2">Delivering to</h2>
        <p className="text-sm">{info.name}</p>
        <p className="text-sm">{info.phone}</p>
        <p className="text-sm text-gray-600">{info.address}</p>
      </div>

      <fieldset className="space-y-2">
        <legend className="font-medium">Payment method</legend>
        <label className="flex items-center gap-2">
          <input type="radio" name="method" value="mpesa" checked={method === "mpesa"} onChange={() => setMethod("mpesa")} />
          M-Pesa
        </label>
        <label className="flex items-center gap-2">
          <input type="radio" name="method" value="airtel" checked={method === "airtel"} onChange={() => setMethod("airtel")} />
          Airtel Money
        </label>
        <label className="flex items-center gap-2">
          <input type="radio" name="method" value="cod" checked={method === "cod"} onChange={() => setMethod("cod")} />
          Cash on delivery
        </label>
      </fieldset>

      <button
        onClick={handlePay}
        disabled={submitting}
        className="w-full bg-brand text-white py-3 rounded disabled:opacity-50"
      >
        {submitting ? "Processing..." : `Pay KSh ${(totalCents / 100).toLocaleString()}`}
      </button>
    </div>
  );
}
```

The fake `handlePay` creates an order id, clears the cart, moves to CONFIRMED, and navigates to the confirmation page. Day 3 replaces this with a Server Action that creates a real order row and sends a real M-Pesa STK push.

---

## The Confirmed Step

Create `app/checkout/confirmed/page.js`:

```jsx
// app/checkout/confirmed/page.js
import Confirmation from "./Confirmation";

export const metadata = { title: "Order Confirmed" };

export default function ConfirmedPage() {
  return (
    <div className="max-w-md mx-auto p-16 text-center">
      <Confirmation />
    </div>
  );
}
```

```jsx
// app/checkout/confirmed/Confirmation.js
"use client";
import Link from "next/link";
import { useEffect, useState } from "react";
import { useCheckout } from "@/app/lib/checkoutStore";

export default function Confirmation() {
  const orderId = useCheckout((s) => s.orderId);
  const reset = useCheckout((s) => s.reset);
  const [hydrated, setHydrated] = useState(false);

  useEffect(() => setHydrated(true), []);

  if (!hydrated) return null;

  return (
    <>
      <h1 className="text-3xl font-bold">Order confirmed</h1>
      <p className="mt-4 text-gray-700">
        Your order <strong>{orderId}</strong> has been received. You will get a WhatsApp message shortly.
      </p>
      <div className="mt-8 space-y-3">
        <Link href="/products" className="block bg-brand text-white py-3 rounded">
          Keep shopping
        </Link>
        <button
          onClick={reset}
          className="block w-full text-sm text-gray-500"
        >
          Start a new order
        </button>
      </div>
    </>
  );
}
```

The `reset` button on the confirmation page clears the checkout store -- otherwise refreshing `/checkout/confirmed` would keep showing the same order id forever. You can also auto-reset after 5 seconds with a `setTimeout` in a `useEffect`.

---

## Progress Indicator

A small UX touch: show the user where they are in the three steps. Create `app/checkout/ProgressBar.js`:

```jsx
"use client";
import { useCheckout } from "@/app/lib/checkoutStore";

export default function ProgressBar() {
  const state = useCheckout((s) => s.state);
  const STATES = useCheckout((s) => s.STATES);
  const steps = [
    { key: STATES.CART, label: "Cart" },
    { key: STATES.INFO, label: "Info" },
    { key: STATES.PAYMENT, label: "Pay" },
  ];
  const currentIndex = steps.findIndex((s) => s.key === state);

  return (
    <div className="flex items-center gap-2 mb-6">
      {steps.map((step, i) => (
        <div key={step.key} className="flex items-center gap-2">
          <div
            className={`w-8 h-8 rounded-full flex items-center justify-center text-sm ${
              i <= currentIndex ? "bg-brand text-white" : "bg-gray-200 text-gray-500"
            }`}
          >
            {i + 1}
          </div>
          <span className="text-sm">{step.label}</span>
          {i < steps.length - 1 && <div className="w-8 h-0.5 bg-gray-200" />}
        </div>
      ))}
    </div>
  );
}
```

Drop it at the top of each checkout page. Small, visual, and tells the user "you are two steps away from done". Feature-phone users cannot have this, but desktop and smartphone shoppers expect it.

---

## Checkpoint

1. From a product page, click Add to cart -> header shows 1 -> navigate to /cart -> click Proceed to checkout -> lands on /checkout/info.
2. Fill in the form, click Continue -> lands on /checkout/payment.
3. Click Pay -> short delay -> lands on /checkout/confirmed with a fake order id.
4. Cart counter is now 0; the cart is cleared.
5. Navigating backward through the browser to /checkout/info shows the form already filled in from the store.
6. Navigating directly to /checkout/payment when the cart is empty redirects to /cart.
7. Navigating directly to /checkout/payment without filling /checkout/info redirects to /checkout/info.
8. The ProgressBar lights up steps as you advance.

Commit:

```bash
git add .
git commit -m "feat: checkout state machine with info payment confirmed steps"
```

---

## What You Learned

- A state machine enforces invariants the URL alone cannot.
- Guard effects redirect users who skip steps.
- Local form state reduces store-subscriber churn.
- The confirmation page must reset state on navigation away.

Tomorrow we replace the fake `setTimeout` with a real Server Action that inserts an order row, deducts stock in a transaction, and opens the door to real payments on Week 16.
