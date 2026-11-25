# Payments — rules and edge cases

Read fully before writing any payment, checkout, subscription, refund, or credits code. Examples use Stripe + Postgres + TypeScript, but every rule applies to any gateway (PayPal, Paddle, Lemon Squeezy, Razorpay, SSLCommerz, bKash, etc.).

## Contents
1. The golden rules
2. Data model
3. The checkout flow (step by step)
4. Duplicate payment prevention
5. Concurrent users and race conditions
6. Connection loss and timeouts
7. Webhooks
8. Subscriptions
9. Refunds, disputes, chargebacks
10. Credits / wallets / balances
11. Pricing, currency, tax, coupons
12. Gateway-specific & regional quirks
13. Fraud and abuse
14. Reconciliation
15. Testing matrix
16. Edge case master list

---

## 1. The golden rules

1. **The webhook (or a server-side status query) is the only thing that marks something paid.** Never the client, never the redirect URL, never a success page load.
2. **Create the order in your DB before talking to the gateway.** You must have a record of the intent even if everything after it fails.
3. **Every call to the gateway that creates or moves money carries an idempotency key** derived from your order ID, not a random value generated per attempt.
4. **One order → at most one successful charge.** Enforced by a unique constraint, not by hope.
5. **Fulfillment is idempotent** and keyed on the order. Granting access twice must be a no-op.
6. **Amounts are computed on the server** from your price table. The client sends a product/plan ID and quantity, nothing else.
7. **Store the gateway's IDs** (payment intent, charge, customer, subscription, invoice, refund) on your rows. You will need them for support, refunds, and reconciliation.
8. **Reconcile daily.** Your DB and the gateway will drift. Find out before the customer does.
9. **Never store card numbers, CVV, or full PANs.** Use the gateway's hosted fields/checkout. You stay out of PCI scope.

---

## 2. Data model

```sql
create table orders (
  id              uuid primary key default gen_random_uuid(),
  user_id         uuid not null references users(id),
  product_id      text not null,
  amount          integer not null check (amount >= 0),   -- minor units
  currency        char(3) not null,
  status          text not null default 'pending'
                  check (status in ('pending','processing','paid','failed',
                                    'canceled','refunded','partially_refunded','disputed')),
  idempotency_key text not null unique,                    -- what we send to the gateway
  gateway         text not null,
  gateway_payment_id text unique,                          -- e.g. Stripe PaymentIntent id
  fulfilled_at    timestamptz,                             -- set once, idempotently
  expires_at      timestamptz,                             -- abandon pending orders
  created_at      timestamptz not null default now(),
  updated_at      timestamptz not null default now()
);

-- Only one open or paid order per user per one-time product
create unique index one_active_order_per_user_product
  on orders (user_id, product_id)
  where status in ('pending','processing','paid');

create table payment_events (            -- raw webhook log, also dedupe table
  gateway_event_id text primary key,
  gateway          text not null,
  type             text not null,
  payload          jsonb not null,
  received_at      timestamptz not null default now(),
  processed_at     timestamptz,
  error            text
);

create table order_events (              -- audit trail of every state change
  id bigserial primary key,
  order_id uuid not null references orders(id),
  from_status text, to_status text not null,
  source text not null,                  -- 'webhook','api','reconcile','admin'
  detail jsonb,
  created_at timestamptz not null default now()
);

create table entitlements (              -- what the user actually gets
  user_id uuid not null,
  feature text not null,
  source_order_id uuid references orders(id),
  valid_until timestamptz,
  primary key (user_id, feature)
);
```

Adjust the partial unique index to your business: for consumables (credits, tickets) where buying twice is legitimate, drop it and rely on idempotency keys instead.

---

## 3. The checkout flow

```
Client                   Server                         Gateway
  │ POST /checkout {product_id, idem_key}
  │──────────────────────▶│
  │                       │ 1. auth + validate
  │                       │ 2. look up price server-side
  │                       │ 3. INSERT order (pending) ON CONFLICT → return existing
  │                       │ 4. create PaymentIntent/Session
  │                       │    (idempotency_key = order.idempotency_key,
  │                       │     metadata.order_id = order.id)
  │                       │───────────────────────────────▶│
  │                       │◀───────────────────────────────│
  │                       │ 5. save gateway_payment_id
  │◀──────────────────────│ return client_secret / redirect URL
  │ 6. user pays on gateway UI ───────────────────────────▶│
  │                       │◀──────── 7. webhook payment_succeeded
  │                       │ 8. verify, dedupe, mark paid, fulfill (one txn)
  │ 9. success page polls GET /orders/:id until paid/failed
```

Key details:
- **Step 3** is the double-payment shield. If the user already has a pending order for this product, return that one (and its existing client secret/session) instead of creating a new one.
- **Step 4** uses the order's stored key, so if step 4 times out and we retry, the gateway returns the same intent instead of creating a second one.
- **Step 4 is outside any DB transaction.** Commit the order first, then call the gateway.
- **If step 5 fails** (crash after gateway success), the reconciliation job and the webhook (which carries `metadata.order_id`) recover it.
- **Step 9:** the success page polls (or uses SSE) for up to ~30–60s, then shows "Payment is processing, we'll email you." Never shows "success" without server confirmation.

Fulfillment (step 8), atomic and idempotent:

```ts
await db.tx(async (t) => {
  const order = await t.oneOrNone(
    `UPDATE orders SET status = 'paid', updated_at = now()
     WHERE id = $1 AND status IN ('pending','processing','failed')
     RETURNING *`, [orderId]);
  if (!order) return; // already paid / refunded / canceled → no-op

  await t.none(`INSERT INTO order_events(order_id, from_status, to_status, source)
                VALUES ($1, 'pending', 'paid', 'webhook')`, [orderId]);
  await t.none(`INSERT INTO entitlements(user_id, feature, source_order_id)
                VALUES ($1, $2, $3)
                ON CONFLICT (user_id, feature) DO NOTHING`,
               [order.user_id, order.product_id, order.id]);
  await t.none(`UPDATE orders SET fulfilled_at = now()
                WHERE id = $1 AND fulfilled_at IS NULL`, [orderId]);
  await t.none(`INSERT INTO outbox(type, payload) VALUES ('send_receipt', $1)`,
               [{ orderId }]);   // email sent by a worker, not inline
});
```

Note `failed` → `paid` is allowed: a payment can fail, the user retries on the same intent, and it succeeds.

---

## 4. Duplicate payment prevention

Layers, from client to DB — use **all** of them:

| Layer | Protects against |
|---|---|
| Disable button on click, show spinner | Accidental double-click |
| Client-generated idempotency key per checkout attempt, kept in sessionStorage | Double-tap, retry after network blip |
| Server dedupes on `(user_id, idempotency_key)` | Two identical requests in flight |
| Partial unique index on active orders per user+product | Two tabs, two devices, back button, refresh |
| Gateway idempotency key = order key | Our own server retrying the gateway call |
| Conditional `UPDATE ... WHERE status IN (...)` on fulfillment | Duplicate/replayed webhooks, webhook + poller both firing |
| Unique on `gateway_payment_id` | Two orders pointing to one payment |
| Daily reconciliation: >1 succeeded charge per order | Anything that slipped through → auto-refund the extra |

Scenarios to explicitly handle:
- **User double-clicks Pay** → same idem key → same order → same intent.
- **User opens checkout in two tabs and pays in both** → second tab gets the existing pending order; if the gateway somehow charges twice (two separate sessions created before the index existed, or a different gateway), reconciliation detects two succeeded charges for one order → refund the newer one automatically and notify.
- **User pays, sees nothing, refreshes and pays again** → checkout endpoint sees order is `paid` → returns "Already purchased" instead of a new checkout.
- **User pays on phone, then on laptop** → same partial index rule.
- **Already-subscribed user hits subscribe again** → check for an active subscription before creating a checkout session; redirect to billing portal instead.
- **User buys, refunds, buys again** → refunded orders are excluded from the partial index, so a new purchase is allowed. Decide if that's intended.

---

## 5. Concurrent users and race conditions

Many different users paying at the same time is **not** a problem by itself if each order is its own row — Postgres handles that. The real races are:

- **Shared limited resource** (last seat, last item, limited coupon uses, inventory):
  - Reserve before payment: `UPDATE products SET stock = stock - 1 WHERE id = $1 AND stock > 0 RETURNING stock`. Zero rows → sold out, don't create a checkout.
  - Store the reservation with `expires_at` (e.g. 15 min). A job releases expired holds and cancels the pending order (and the gateway session, so they can't pay late).
  - If payment succeeds after the hold expired and the item is gone → auto-refund with a clear email. This **will** happen; handle it.
- **Coupon with max N redemptions:** atomic increment with `WHERE redemptions < max_redemptions`. Count only on payment success, or reserve like stock.
- **Webhook and success-page poller both try to fulfill:** the conditional UPDATE guarantees only one wins.
- **Two webhooks for the same order at once** (e.g. `payment_intent.succeeded` and `checkout.session.completed`): pick **one** event type as the fulfillment trigger; treat others as informational. Still guarded by the conditional UPDATE.
- **Subscription upgrade while renewal invoice is processing:** serialize per customer with `pg_advisory_xact_lock(hashtext(customer_id))` in your handlers.
- **Wallet / credit spend from two requests at once:** atomic `UPDATE ... WHERE balance >= cost`. See §10.
- **Load spikes (launch day):** the gateway has rate limits — handle `429` with backoff; keep the checkout endpoint lightweight; webhooks are queued, not processed inline.

Test: fire 50 parallel `POST /checkout` for the same user+product and assert exactly one order and one intent exist. Fire 50 parallel duplicate webhooks and assert one fulfillment.

---

## 6. Connection loss and timeouts

Map every point where the connection can die and what state you're left in:

| Where it dies | State | Recovery |
|---|---|---|
| Client → server, before order created | Nothing | User retries; same idem key |
| After order created, before gateway call | Order `pending`, no intent | Retry reuses order; expiry job cancels if abandoned |
| Server → gateway, request sent, no response (timeout) | **Unknown** — intent may exist | Retry with same idempotency key → gateway returns the same object. Never create with a new key. |
| After gateway success, before we saved `gateway_payment_id` | Gateway has intent, we don't know its ID | Webhook carries `metadata.order_id` → link it. Reconciliation searches by metadata. |
| User's internet drops while on gateway's payment page | Payment may or may not complete | Webhook decides. When user returns, status page shows truth. |
| User closes tab after paying, before redirect | Paid, user didn't see success | Webhook fulfills; send receipt email; show "purchased" on next visit |
| 3DS / bank OTP abandoned | Intent `requires_action` | Stays unpaid; expiry job cancels; user can resume from order page |
| Server crashes mid-fulfillment | Transaction rolled back | Webhook retried by gateway (it gets non-2xx or times out) → reprocessed safely |
| Our webhook endpoint is down for hours | Gateway retries (Stripe: up to ~3 days) | Handlers idempotent; reconciliation as backstop |
| Email/receipt send fails | Order paid, no email | Outbox worker retries; email failure never rolls back payment |
| Mobile app backgrounded during payment | Unknown | On resume, fetch order status from server |

Rules:
- Set explicit gateway SDK timeouts and retries (Stripe Node: `timeout`, `maxNetworkRetries` — it reuses idempotency keys automatically on its own retries).
- On timeout, the API returns `202 { status: "processing", order_id }`, not an error. The client polls.
- Never tell the user "payment failed" unless the gateway explicitly said failed/declined. Unknown ≠ failed. Telling them "failed" when it succeeded is how double payments happen.

---

## 7. Webhooks

Handler skeleton:

```ts
app.post('/webhooks/stripe', express.raw({ type: 'application/json' }), async (req, res) => {
  let event;
  try {
    event = stripe.webhooks.constructEvent(req.body, req.headers['stripe-signature'], WEBHOOK_SECRET);
  } catch { return res.sendStatus(400); }            // bad signature → reject

  const inserted = await db.result(
    `INSERT INTO payment_events(gateway_event_id, gateway, type, payload)
     VALUES ($1,'stripe',$2,$3) ON CONFLICT DO NOTHING`,
    [event.id, event.type, event]);
  if (inserted.rowCount === 0) return res.sendStatus(200); // duplicate → ack

  await queue.enqueue('process_payment_event', { eventId: event.id });
  res.sendStatus(200);                               // ack fast (<2–3s)
});
```

Rules:
- **Verify signatures using the raw body.** Body parsers that re-serialize JSON break verification — a classic bug.
- **Dedupe by event ID** before processing.
- **Ack fast, process async.** Slow handlers cause timeouts → gateway retries → more load.
- **Out of order:** `payment_failed` can arrive after `payment_succeeded` (from an earlier attempt). Never downgrade a `paid` order based on an older event. Guard with state-machine UPDATEs, or **re-fetch the object from the gateway API** and act on its current state (most robust).
- **Late:** events can arrive minutes/hours later. Don't assume ordering relative to the user's redirect.
- **Missing:** webhooks occasionally never arrive. Reconciliation covers this.
- **Unknown event types:** log and ack 200. Don't 500 on events you don't handle.
- **Test vs live:** separate endpoints and secrets per environment. Check `event.livemode` matches your environment.
- **Secret rotation:** support two secrets during rotation.
- Return non-2xx only for transient failures you want retried; return 200 for permanent "can't process" and alert yourself.
- Match the event to the order via `metadata.order_id`, falling back to `gateway_payment_id`. If neither matches, store it as orphaned and alert.

---

## 8. Subscriptions

- Let the gateway own billing logic (proration, retries, dunning). Your DB mirrors **subscription status and current period end**; access is granted from that.
- **Grant access based on `status` in (`active`, `trialing`) and `current_period_end > now()`**, plus a small grace period.
- Handle every status: `incomplete`, `incomplete_expired`, `trialing`, `active`, `past_due`, `unpaid`, `canceled`, `paused`.
- **Renewal payment fails** → `past_due`. Keep access during the gateway's retry window (or a grace period you choose), email the user, then revoke on `canceled`/`unpaid`.
- **Cancel at period end** vs **cancel immediately** — be explicit; show the end date in UI.
- **Upgrade/downgrade:** use the gateway's proration; update entitlements on the `customer.subscription.updated` webhook, not on the button click.
- **Trial abuse:** one trial per user/payment fingerprint/email domain if it matters.
- **Card expires** → enable the gateway's card updater and pre-dunning emails.
- **Price changes:** existing subscribers stay on their price unless you migrate them deliberately. Never edit a price in place — create a new one.
- **One active subscription per user:** check before creating a checkout; enforce with a partial unique index on `(user_id) WHERE status IN ('active','trialing','past_due')`.
- **Billing portal** for card updates, invoices, cancellations — don't build these yourself.
- **Time zones & period boundaries:** access checks use UTC timestamps from the gateway, not "end of month" logic you compute.
- **Lifetime deals** next to subscriptions: model as an entitlement with `valid_until = null`, separate from the subscription row.

---

## 9. Refunds, disputes, chargebacks

- Refunds go **through your server** (admin action) → gateway API with idempotency key `refund:{order_id}:{n}` → mark `refund_pending` → webhook confirms → revoke entitlements.
- Partial refunds: track `amount_refunded`; status `partially_refunded`; decide whether access stays.
- Refunds can **fail** (closed card, insufficient balance in your account). Handle `refund.failed`.
- Never refund more than captured — gateway enforces, but check first for a clean error.
- **Disputes / chargebacks:** on `charge.dispute.created` → flag the order, optionally suspend access, alert yourself, gather evidence (logs, IP, usage, ToS acceptance). You usually have ~7–21 days to respond.
- Disputes cost a fee even if you win. Clear statement descriptors (`ACME INC`) reduce "I don't recognize this charge" disputes.
- Refunding a payment that has an open dispute → don't; respond to the dispute instead (or you can lose money twice).

---

## 10. Credits / wallets / balances

- **Ledger, not a balance column alone.** Every change is an immutable row: `(id, user_id, amount, reason, ref_id, created_at)`; the balance is the sum (cache it in a column updated in the same transaction).
- Unique `(reason, ref_id)` on the ledger so a purchase or refund can only credit once.
- Spending is atomic:
  ```sql
  UPDATE wallets SET balance = balance - $2
  WHERE user_id = $1 AND balance >= $2
  RETURNING balance;
  -- 0 rows → insufficient; then insert the ledger row in the same txn
  ```
- Never let a balance go negative unless that's an explicit product decision (CHECK constraint `balance >= 0`).
- Usage-based billing (e.g. AI tokens): **reserve** estimated cost before the call, **settle** actual cost after, **release** on failure. Otherwise concurrent requests overspend.
- Credits expire? Store `expires_at` per grant and spend oldest-first (FIFO).
- Refund of a credit purchase after credits were spent → decide policy (refund remainder only).

---

## 11. Pricing, currency, tax, coupons

- Prices live in a server-side table or the gateway's Price objects. Client sends `price_id`, server validates it's active and purchasable by this user.
- Zero-decimal currencies (JPY, KRW) and three-decimal (KWD, BHD) — use the gateway's minor-unit rules; don't hardcode `* 100`.
- Rounding: compute in integers; apply discounts and tax in a defined order; round once at the end.
- Minimum charge amounts exist (Stripe: ~$0.50 USD equivalent). 100%-off coupon → skip the gateway and mark paid via a free-order path (still idempotent, still logged).
- Coupons: validate server-side for expiry, max uses, per-user limit, eligible products, minimum amount, stacking rules. Never trust a discount amount from the client.
- Tax (HST in Ontario, VAT in EU, etc.): use the gateway's tax engine (Stripe Tax) rather than hand-rolled rates. Collect address/postal code as required.
- Price shown = price charged. Lock the price on the order row at creation time so a price change mid-checkout doesn't change what the user agreed to.
- Show currency explicitly (`CA$` vs `US$`).

---

## 12. Gateway-specific & regional quirks

- **3D Secure / SCA:** payments may require extra authentication; status goes to `requires_action`. Use the gateway's client SDK to handle it; never assume one-step success.
- **Async payment methods** (bank debits, ACH, SEPA, some wallets, Interac): `processing` can last days, then succeed **or fail**. Don't fulfill on `processing` unless you accept the risk; handle late failure by revoking.
- **Redirect-based gateways** (common in South Asia — SSLCommerz, bKash, Nagad, aamarPay): the redirect/callback URL can be spoofed or hit twice. **Always call the gateway's validation/query API server-side** with the transaction ID and check amount, currency, and status before marking paid. Treat IPN like a webhook (verify, dedupe). Store `tran_id` as unique.
- **Amount tampering:** always compare the gateway-reported amount and currency to the order's stored amount. Mismatch → don't fulfill, alert.
- **Auth vs capture:** if you authorize first and capture later, auths expire (~7 days on cards). Capture or release explicitly.
- **Currency conversion** by the card issuer can make the user see a different amount — mention it in UI for international customers.
- **Sandbox ≠ production:** test cards don't reproduce every real decline. Do a real small-value live transaction (and refund it) before launch.

---

## 13. Fraud and abuse

- Use the gateway's fraud tools (Stripe Radar). Block or review high-risk.
- **Card testing attacks:** bots hit your checkout with many cards at small amounts. Defenses: require login before checkout, rate-limit per IP/user/card fingerprint, CAPTCHA on repeated failures, minimum amount.
- Limit failed payment attempts per user per hour.
- Don't reveal precise decline reasons that help attackers (`incorrect_cvc` vs generic "card declined" — show generic to user, log specific).
- Watch for many accounts sharing one card fingerprint (trial abuse).
- Promo code brute-forcing → rate limit validation endpoint.

---

## 14. Reconciliation

A scheduled job (hourly for recent, daily for full) that compares your DB to the gateway:

- Orders `pending`/`processing` older than N minutes → query gateway → update to true state (this fixes missed webhooks).
- Succeeded charges at gateway with no `paid` order → link or create; fulfill; alert.
- `paid` orders with no succeeded charge at gateway → alert (possible bug or fraud).
- More than one succeeded charge per order → auto-refund extras, alert.
- Amount/currency mismatches → alert.
- Subscription status mismatch → sync from gateway (gateway is truth for billing state).
- Expired pending orders → cancel locally and cancel/expire the gateway session.
- Daily totals: sum of your paid orders vs gateway payout/balance report.

Every fix the job makes writes an `order_events` row with `source = 'reconcile'`.

---

## 15. Testing matrix

Automate these (gateway test mode + test clocks for subscriptions):

- [ ] Happy path one-time payment
- [ ] Card declined / insufficient funds / expired card / incorrect CVC
- [ ] 3DS required → completed; 3DS required → abandoned
- [ ] Double-click Pay → one charge
- [ ] 50 concurrent checkout requests same user/product → one order
- [ ] Two tabs paying simultaneously → one charge or auto-refund of the second
- [ ] Gateway call times out after success (simulate) → retry returns same intent
- [ ] Webhook delivered twice → one fulfillment
- [ ] Webhooks out of order (failed after succeeded) → stays paid
- [ ] Webhook never arrives → reconciliation fixes it
- [ ] Webhook with bad signature → 400
- [ ] Server crash between gateway success and DB write → recovered
- [ ] User closes tab right after paying → fulfilled, emailed
- [ ] Price tampered in client request → ignored
- [ ] Coupon at max redemptions under concurrency → never exceeds
- [ ] Last item stock with 10 concurrent buyers → one succeeds, others sold out or refunded
- [ ] Subscription: trial → active, renewal fail → past_due → recovered, → canceled
- [ ] Upgrade/downgrade proration
- [ ] Full refund, partial refund, refund failure
- [ ] Dispute created → flagged
- [ ] 100% off coupon → free order path
- [ ] Async payment method succeeds later / fails later
- [ ] Test-mode event sent to live endpoint → rejected

Use the Stripe CLI (`stripe listen`, `stripe trigger`) to replay events locally.

---

## 16. Edge case master list

Quick scan before shipping:

**User behavior:** double-click · refresh during payment · back button after paying · two tabs · two devices · closes tab mid-3DS · pays then immediately requests refund · pays with a card in another currency · changes email mid-flow · deletes account with active subscription · logged out mid-checkout · session expires on gateway page.

**Network:** client offline before submit · drop after submit · server→gateway timeout · gateway 5xx · gateway 429 · DNS failure · slow 3DS bank page · webhook endpoint down · webhook delayed hours.

**Concurrency:** many users at once (fine) · same user parallel requests · last-item race · coupon limit race · webhook + poller race · duplicate webhooks · simultaneous upgrade + renewal · wallet double spend.

**Money:** float rounding · zero-decimal currencies · minimum charge · 100% discount · negative totals from stacked coupons · tax changes · price changed mid-checkout · amount mismatch from gateway · partial capture · auth expiry · FX differences.

**Lifecycle:** payment succeeds after order expired · payment succeeds after item sold out · refund after access used · dispute after refund · subscription renews after user deleted account · card expires · trial ends with no card · plan discontinued with active subscribers.

**Ops:** test/live key mix-up · webhook secret rotated · gateway API version upgrade changes payload · migrations during live traffic · time zone/period-end bugs · logs leaking PII · no alert when payments silently stop (alert on "zero payments in last N hours" during normal hours).
