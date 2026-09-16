# Audit 2 — Conformance of the Implementation to the Intended Flow

**Date:** 11 Aug 2026 · **Method:** code-path trace (route → controller → service → DB → gateway →
webhook), plus live-database index and arithmetic probes. **Code modified during this audit: none.**

**Scope note:** this is a *second* audit, run after the remediation of
[`production-readiness-audit.md`](./production-readiness-audit.md). It does **not** re-report the 44
items already fixed and covered by `npm test` (254 assertions across 5 suites). It answers one question: **where does
the implementation still diverge from the intended flow you supplied?**

Everything below was verified by reading the code or probing the running database. Where I could not
verify something, it says so.

---

## Executive Summary

| | |
|---|---|
| **Overall status** | **PASS WITH RISKS** |
| **Financial safety** | **SAFE** — for every path exercised today |
| **Critical issues** | **0** |
| **High issues** | **2** |
| **Medium issues** | **7** |
| **Low issues** | **3** |

**The two High issues are both non-financial in their most likely form** but one has a money
consequence in an unlikely form. Neither is a regression: both are pre-existing designs that this
audit's spec probes and the first audit did not.

**What changed since audit 1:** every Critical and 13 of 14 High items are fixed and covered by
assertions. The refund path is intent-first with gateway reconciliation, order status is a validated
transition table, cancellation and return status changes are compare-and-swap claims, webhooks have a
unique-index replay guard and answer 5xx on failure, and the wallet/stock/coupon compensations are all
idempotent. That work is documented in audit 1 and in
[`order-payment-refund-flow.md`](./order-payment-refund-flow.md).

**The honest headline:** the *money* is now correct. What remains diverging from your intended flow is
mostly **operational modelling** — RTO sub-states, replacement fulfilment, and a shipment layer that
is coupled to Shiprocket rather than provider-agnostic.

---

## Update — 11 Aug 2026: five flow changes implemented

The project owner adopted the design changes below after this audit. Implemented and
covered by `tests/lifecycle.regression.mjs` (78 assertions); the suite total is now
**254 across 5 suites**.

| # | Change | Audit item it closes |
|---|---|---|
| 1 | **Razorpay order-first** — created at `Pending`/`Pending`, promoted to `Paid`/`Confirmed` by a compare-and-swap on verification | *(new design; not an audit finding — I argued against it and was overruled, which was the owner's call to make)* |
| 2 | **`RTO Received`** as a real state, with an explicit *Razorpay-refund-owed vs COD-no-refund* branch | **H2-05** (partly — no `Closed` terminal) |
| 3 | **Inventory disposition** (`resellable` / `damaged`) separated from the customer's resolution, on returns **and** RTO | the defect this audit documented as correct behaviour — see below |
| 4 | **`owed` refund state + `confirmationMethod`** separating a COD payout awaiting a human from a gateway refund awaiting Razorpay | *(new; makes the COD liability visible for the first time)* |
| 5 | Atomic claims / idempotency preserved on every new path | verified by assertion |

### A correction to this audit

Under *Verified Correct* and in the flow doc I recorded "QC pass → restock, QC fail →
no restock" as correct. **It was not.** `restockReturnedItems` fired on `refunded` or
`replaced` with no condition check and there was no disposition concept anywhere, so a
customer returning a genuinely faulty item was refunded **and the defective unit went
back on sale**. The admin's only way to avoid that was to reject the return and deny a
legitimate refund. Change #3 fixes it; resolving a return now *requires* an explicit
disposition, with no default in either direction.

### Two bugs the new tests caught before deploy

- **`sparse` does not skip empty strings.** `razorpayOrderId` / `razorpayPaymentId` are
  `UNIQUE + sparse`; sparse skips *absent* fields, so a stored `""` is indexed like any
  value. Order-first creates many orders with no payment id — one `""` write would have
  made the **second** unpaid checkout fail with E11000 and taken checkout down
  entirely. Both fields now normalise `""`/`null` to `undefined` in a schema setter.
- **Two more unguarded aggregates.** The test that walks `src/` for order aggregates
  found `admin-response.service.js` and `admin.controller.js` after a manual sweep had
  missed them — a customer's order count and per-customer spend would both have
  included abandoned checkouts.

Also fixed while implementing: `recordRefundObligation` now refuses an order that
collected no money, so "never owe money to someone who didn't pay" is enforced at the
invariant rather than only in its callers.

### Still open from this audit

**H2-01** (non-unique `shiprocket.awbCode`), **H2-02** (one return per line forever),
**H2-03** (shipping/tax latent under-refund), **H2-04** (`refunds.providerRefundId`
unindexed), **H2-06** (replacement fulfilment), **H2-07** (provider-coupled shipment
layer), **H2-08** (no refund confirmation prompt), **H2-09** (`InventoryModel`), and the
Low list. The ranking below is unchanged — H2-01 and H2-03 are still where I would go
next.

---

## Critical Issues

**None found.**

I want to be explicit rather than reassuring about that. The specific scenario your spec calls out —

```
Refund API called → Razorpay processes it → network timeout → backend thinks it failed → retries
```

— was a real Critical in audit 1 (C-05) and is now handled: the refund record is persisted with
`status: "created"` **before** the gateway call, tagged `notes.returnNumber`, and a retry queries
`razorpay.payments.fetchMultipleRefund()` matched on that tag instead of re-refunding. If Razorpay is
unreachable during that reconciliation the operation **fails loudly** (`REFUND_RECONCILE_UNAVAILABLE`)
rather than assuming the refund never happened.

Your spec asks whether a **database uniqueness constraint** prevents duplicate refunds, and says to
flag `CRITICAL — FINANCIAL RISK` if not. My answer is a qualified no, and I am deliberately **not**
flagging it Critical:

- **Verified against the live database: there is no index of any kind on `refunds.*`.**
- **No index can express this constraint.** A unique index on a field inside an array subdocument
  enforces uniqueness *across documents*, not *within* one document's array. So "one refund per return
  per order" is not expressible as an index on `orders.refunds[].returnRequest`, whatever value it is
  given.
- The guard is therefore a **compare-and-swap on the return's status**, taken before any money moves:
  `findOneAndUpdate({_id, status: previousStatus}, {$set: {status}})`. Two concurrent admins: one wins,
  the other receives 409 `RETURN_STATUS_CONFLICT` and never reaches the refund call. This is a real
  atomic primitive, not an `if (!existingRefund)` check — the precondition is in the query filter, and
  MongoDB guarantees single-document atomicity for it.
- Covered by assertions in `tests/money.regression.mjs` and `tests/returns.regression.mjs`.

**If you want the DB-level constraint anyway**, the only shape that works is extracting refunds into
their own collection with a unique index on `{order, returnRequest}`. That is a schema migration, not
an index addition. I have not done it because the CAS already closes the race and the migration
carries its own risk. Your call.

---

## High-Priority Issues

### H2-01 — Shiprocket webhook can apply a status to the WRONG order

```
Issue:       Webhook order lookup falls back to a non-unique field
Severity:    HIGH
File:        src/modules/shipping/shipping.controller.js
Function:    ShippingWebhook
Line:        ~395-410 (the `query` ternary)
```

**Current behaviour.** The handler resolves the order by the first available of:

```js
sourceOrderId ? { _id: sourceOrderId }
: shiprocketOrderId ? { "shiprocket.orderId": shiprocketOrderId }
: awbCode ? { "shiprocket.awbCode": awbCode } : null
```

**Verified against the live database:** `shiprocket.orderId`, `shiprocket.shipmentId` and
`shiprocket.awbCode` all carry **plain, non-unique indexes**. Only `razorpayOrderId` and
`razorpayPaymentId` are unique.

`findOneAndUpdate` on a non-unique field silently picks **one arbitrary match**.

**Expected behaviour.** A courier event must resolve to exactly one order, or be rejected.

**Why this is dangerous.** If two orders ever share an AWB — courier-side reuse, a duplicated
Shiprocket record, or a manual re-entry — a `Delivered` event applies to the wrong order. Because
delivery is the COD payment event, that order is also marked **`Paid`**, booking revenue never
collected; and an `RTO DELIVERED` event would restock the wrong order's items. The transition table
limits the damage (it cannot resurrect a `Cancelled` order) but cannot detect wrong-order targeting.

**Probability caveat — NOT VERIFIED:** whether Shiprocket ever reuses an AWB across orders is not
determinable from this repository. I could not find documentation either way. Treat the likelihood as
unknown rather than low.

**Recommended fix.** Make `shiprocket.orderId` and `shiprocket.awbCode` unique+sparse; on webhook
receipt, `countDocuments` the query and refuse (log + 200, so Shiprocket stops retrying an ambiguity
that will not resolve itself) when it is not exactly 1.

---

### H2-02 — One return per line, forever — including after a wrong rejection

```
Issue:       Repeat returns are structurally impossible, with no operator override
Severity:    HIGH (customer-impacting dead end; not financial)
File:        src/modules/returns/return.model.js:176 · src/modules/returns/return.controller.js:194-204
Function:    CreateReturnRequest
```

**Current behaviour.** `returnSchema.index({order: 1, product: 1, user: 1}, {unique: true})`, plus an
application check that queries `{order, product, user}` with **no status filter** and returns 409.

So for a given line, exactly one return request can ever exist.

**Expected behaviour.** Your spec §7 asks explicitly about *"Multiple returns against the same item"*
and *"Return after another return"*. Both are currently impossible.

**Why this is dangerous.** Two concrete dead ends:

1. **Partial return exhausts the line.** A customer buys 3 units, returns 1 (approved, refunded).
   They can never return the other 2, even inside the policy window. The refund arithmetic supports
   this fine — the cumulative ceiling and `walletRefunded` both handle multiple returns — so the
   blocker is purely this uniqueness.
2. **A wrong QC rejection is terminal.** If the QC step rejects in error, the `rejected` record still
   occupies the unique slot. The customer cannot re-request and **no admin endpoint exists to reopen
   or delete a return**, so the only remedy is direct database surgery.

**Recommended fix.** Two options, and they are genuinely different products:
- **Minimal:** keep one *open* return per line — replace the unique index with a partial unique index
  on non-terminal statuses, and add `status: {$nin: ["rejected"]}` to the app check, so a rejection
  frees the slot. Add a `returnedQuantity` accumulator on the order line and cap
  `returnableQuantity` against it (the pattern already exists for `cancelledQuantity`).
- **Fuller:** an admin "reopen return" action with an audit entry.

I have not implemented either. The first is small but changes customer-visible policy — how many
returns you permit per line is your decision, not mine.

---

## Medium Issues

### H2-03 — Shipping and tax silently drop out of return refunds *(latent)*

```
File:      src/modules/orders/order-pricing.service.js:154-161 · return-refund.service.js (proportionalRefundAmount)
Severity:  MEDIUM (latent — inert today, wrong the moment shipping or tax is non-zero)
```

**Verified:** `shippingCharge = 0` and `tax = 0` are **hardcoded** in `prepareOrderData`, and
`totalAmount = subtotal + shippingCharge + tax − discount`. So today `totalAmount ≤ subtotal`, the
ratio is ≤ 1, and the arithmetic is correct — I probed it numerically.

The fields are nonetheless plumbed end-to-end (`Order.model.js`, `PaymentIntent.model.js`, and
`shiprocket.service.js` reads `order.shippingCharge`), so someone will eventually set them.

**What happens the moment they do** — probed, subtotal ₹1000 + shipping ₹200:

| | |
|---|---|
| Full **return** refunds | **₹1000** |
| Customer actually paid | ₹1200 |
| Shortfall (shipping dropped) | **₹200** |
| Full **cancel** refunds (`totalAmount − sumRefunded`) | **₹1200** |
| **Cancel and return disagree by** | **₹200** |

The ratio `min(1, orderTotal ÷ orderSubtotal)` caps at 1, so shipping and tax can never be
apportioned into a line refund — while the cancellation path refunds the true outstanding balance and
therefore *does* return them.

**Why it matters.** Not refunding shipping on a partial return is a defensible policy. Refunding it on
a cancellation but not on a full return, with no code expressing that decision, is not — it is an
accident that will present as a customer complaint.

**Recommended fix.** Decide the policy explicitly, then encode it: either apportion shipping/tax
across lines, or exclude them from the cancellation refund too, or add an explicit "refund shipping on
full return" branch. Whichever you pick, add the assertion.

### H2-04 — `refunds.providerRefundId` is the refund-webhook lookup key and is unindexed

```
File:      src/modules/payments/payment.controller.js (RazorpayWebhook, refund branch)
Severity:  MEDIUM
```

`refund.processed` / `refund.failed` resolve the order with
`OrderModel.findOne({"refunds.providerRefundId": refundId})`. **Verified: no index on `refunds.*`.**
Every refund webhook is therefore a full collection scan of `orders`, and there is no uniqueness, so a
gateway refund id appearing on two orders would resolve arbitrarily.

Correctness is fine at 55 orders; this degrades with volume and is the kind of thing that fails at the
worst moment. A non-unique index on `refunds.providerRefundId` is a one-line, zero-risk addition.

### H2-05 — `RTO_IN_TRANSIT` and `RTO_RECEIVED` do not exist as order statuses

```
File:      src/modules/orders/order-status.rules.js · Order.model.js (orderStatus enum)
Severity:  MEDIUM
```

Your intended flow specifies `RTO → RTO_IN_TRANSIT → RTO_RECEIVED → CLOSED`. The implementation has a
single `RTO`, and `RTO` may only move to `Cancelled`.

Partially mitigated: physical arrival **is** detected — `isRtoReceived()` matches Shiprocket status id
43 (or `RTO DELIVERED`/`RTO RECEIVED` text) and stamps `order.rtoRestockedAt` before restocking. So the
*inventory* consequence of arrival is handled correctly. What is missing is the **state**: the order
sits at `RTO` indefinitely, so there is no way to report "RTO parcels awaiting receipt" versus
"received and closed", and no `CLOSED` terminal.

Deliberate collapse, incidentally: mapping all three RTO strings to one status is right for the
customer-facing timeline. The gap is that operations needs the finer grain and has nowhere to read it.

### H2-06 — Replacement has no fulfilment entity

```
File:      src/modules/returns/return.model.js
Severity:  MEDIUM
```

**Verified:** the entire representation of a replacement is `resolutionType: "replacement"`,
`status: "replaced"`, and `replacedAt`. There is no outbound shipment, no AWB, no tracking, no
`REPLACEMENT_SHIPPED` / `REPLACEMENT_DELIVERED`.

Against your intended flow (`REPLACEMENT CREATED → PROCESSING → SHIPPED → DELIVERED`), everything after
"created" is missing. Operationally: once an admin clicks *replaced*, the system believes the case is
closed while the replacement parcel has not yet been packed — and the customer has nothing to track.

### H2-07 — The shipment layer is coupled to Shiprocket, so `provider = MANUAL` has nowhere to live

```
File:      src/modules/orders/Order.model.js (the `shiprocket` sub-document) · src/modules/shipping/*
Severity:  MEDIUM (architectural — see Shiprocket Readiness below)
```

**Verified:** there is no `Shipment` model and no `provider` field anywhere. All shipment state lives
in `order.shiprocket.{orderId, shipmentId, awbCode, courierId, courierName, status, statusCode,
syncStatus, ndrReason, rtoReason, package, labelUrl, invoiceUrl, lastError, lastSyncedAt}`. The
`shipping/` module contains only Shiprocket files.

Consequence: a seller shipping **manually** (which is the current mode —
`SHIPROCKET_AUTO_CREATE_ORDER` defaults false) has nowhere to record a courier name or tracking number
except by writing into a field named `shiprocket`.

### H2-08 — No confirmation prompt on the admin Razorpay refund *(carried from audit 1, M-13)*

One click, real money, irreversible, no second step. Harmless on test keys. **Verified still open.**

### H2-09 — `InventoryModel` is a second stock ledger no order path writes *(carried, M-03)*

**Verified:** no order, cancel, return or reservation path touches `InventoryModel`. The admin
Inventory page will diverge from `product.stock` after the first sale. `product.stock` is the real
number; this is a reporting defect, not a financial one.

---

## Low Issues

### H2-10 — Status names diverge from the intended flow *(cosmetic, but check your integrations)*

| Intended | Actual |
|---|---|
| `PENDING_PAYMENT` | `Pending` |
| `CONFIRMED` | `Confirmed` |
| `PROCESSING` / `PACKED` | `Packed` (no separate `Processing`) |
| `OUT_FOR_DELIVERY` | `Out For Delivery` (spaces, title case) |

Not a defect — the semantics match. Flagging it because the values are **stored strings with spaces**,
which will bite any future integration, export or filter that assumes an enum-like token.

### H2-11 — `PAYMENT_MODE=demo` is dead config *(carried)*

**Verified:** read into `getFeatures().payments.mode` and consumed by **nothing** (`grep` for
`payments.mode` returns no consumer). It protects nothing, and its name implies it does.

### H2-12 — Manual fulfilment records no shipment data *(carried)*

Follows from H2-07. No courier, no tracking number for non-Shiprocket orders.

---

## Verified Correct

These I traced and found genuinely right — not assumed from names.

**Payment / Razorpay**
- Signature verification on both verify (`isValidSignature`) and webhook (`isValidWebhookSignature`,
  HMAC-SHA256 over the raw body via `req.rawBody`, `timingSafeEqual`). The webhook secret is a
  separate variable from the key secret.
- `razorpayOrderId` and `razorpayPaymentId` both **UNIQUE + sparse** on `orders` and on
  `paymentintents` — verified in the live database. The same payment cannot back two orders.
- The paid amount **and** currency are checked against the intent on verify, and the amount is checked
  again in the webhook (audit 1's M-06).
- A payment cannot be attached to the wrong order: `capturedPayment.order_id` must be in
  `[intent.razorpayOrderId, ...intent.previousRazorpayOrderIds]`.
- An order cannot become `Confirmed` without verified payment — `completeCapturedIntent` requires
  `capturedPayment.status === "captured"` and creates the order inside the same transaction.
- Frontend success is **not** trusted: the browser's callback triggers server-side verification, and
  `payment.captured` creates the order independently if the browser never returns.
- Retries no longer orphan payments (`previousRazorpayOrderIds`, indexed).

**Webhooks**
- Replay protection is a **unique index** (`webhookevents` `{provider, eventId}` — verified UNIQUE in
  the live DB), not an application check. Concurrent duplicates race; exactly one wins.
- Failures answer **5xx and release the claim**, so Razorpay retries. Duplicates answer 200.
- Courier events are validated against the transition table before being applied.
- Shiprocket webhook authenticates via `timingSafeEqual` on `x-api-key`.
- Partial payloads no longer null out stored `orderId`/`awbCode` (audit 1's M-07).

**COD**
- **Not** marked paid at creation — `paymentStatus: "Pending"` is hardcoded in `PlaceOrder`.
- `Delivered → Paid` in both paths is guarded on `paymentStatus === "Pending"`, so it cannot overwrite
  a refund already issued; the webhook path is additionally gated on the transition being *accepted*,
  so a stale `Delivered` for a cancelled order books nothing.
- Cancel before delivery creates **no** refund (`moneyCollected` is false).
- RTO on COD creates **no** customer refund.
- A COD return after delivery **does** create a refund obligation, and cannot be closed without a
  method + reference.

**On your §3 question — does the code couple `delivery = payment collected`?** Yes, and I consider it
correct here: for COD, delivery *is* the payment event, and there is no other signal available. It is
safe because it is idempotent (the `Pending` guard) and because a rejected transition does not trigger
it. The residual risk is not the coupling but H2-01 — the courier event resolving to the wrong order.

**Refunds**
- Intent-first ordering, gateway reconciliation on retry, `created` vs `failed` distinction.
- `failed` refunds are excluded from the settled sum; **in-flight `created` records are included in
  the ceiling** (conservative) but excluded from `paymentStatus` (honest). Two separate sums,
  deliberately.
- Cumulative refunds cannot exceed `order.totalAmount` on any of the four paths (cancel, partial
  cancel, return, manual admin). The manual admin endpoint takes an amount from the request body but
  ceilings it at `totalAmount − sumRefunded` server-side.
- `Refunded` requires **settled** money. A gateway-failed refund drops the order back to `Paid`.
- Wallet and cash halves are settled separately and sum to the customer's real outlay — probed
  numerically: subtotal ₹1000, coupon ₹100, wallet ₹200 → cash ₹700 + wallet ₹200 = ₹900 = outlay.

**Refund arithmetic** (probed, not read)
- Rounding is clean: 3 × ₹33.33 of a ₹99.99 subtotal at ₹66.66 total → three line refunds summing to
  exactly ₹66.66, no cumulative overshoot.
- Sub-rupee: ₹0.01 line of a ₹0.03 subtotal at ₹0.02 total → ₹0.01.
- Free items (price 0) → 0. Negative price → 0. `NaN` quantity → 0.
- Zero subtotal falls back to full line value — **guarded by the order-total ceiling, not by the
  formula.** On a ₹0 order that ceiling refuses it. Worth knowing the formula alone is not the guard.

**Security / IDOR**
- All financial values are server-computed. The client supplies only `orderId`, `productId`,
  `quantity`, `reason`, `details`, `proofImages`, `refundDestination`. It cannot set `refundAmount`,
  `paymentStatus`, `orderStatus` or `returnStatus`.
- `GetSingleOrder` — confirmed to be the function actually wired to `GET /:orderId` — requires owner
  **or** `orders:manage`. `GetReturnById` requires owner **or** `returns:manage`. `GetMyOrders` and
  `GetMyReturns` scope to `req.user.id`.
- Every admin mutation on orders is behind `requirePermission(ORDERS_MANAGE)`, which now also asserts
  `hasAdminRole` — so a plain user granted an individual permission cannot pass.
- Customer bank details: non-owner reads require `returns:manage`; the owner sees the account number
  masked to the last four digits.
- `CancelOrder` checks ownership and tolerates legacy `user: null` guest orders with a clean 403
  rather than a 500.

**State machine**
- Transitions are validated in both the admin endpoint and the courier webhook, from one shared table,
  mirrored on the frontend and asserted to match.
- `Delivered` and `Cancelled` are terminal. Same-status is a legal no-op.
- `Cancelled` is refused by `UpdateOrderStatus` entirely (`USE_CANCEL_ENDPOINT`).

**Inventory idempotency** (the four duplicate scenarios in your §12)
- Duplicate cancellation → atomic claim, one winner.
- Duplicate webhook → unique-index event claim + `rtoRestockedAt` claim.
- Duplicate return processing → return-status CAS + `restockedAt` claim.
- Duplicate RTO events → `rtoRestockedAt` claim. Probed: 4 concurrent attempts restock once.
- Reservation release is a CAS; 4 concurrent releases restock once.
- Restock excludes units already cancelled.

**Coupon / wallet**
- Coupon restore is one conditional update with `$gte: 1` in the filter; cannot double-restore, cannot
  go negative.
- Wallet restore is capped by `order.walletRefunded` with the ceiling in the filter; 5 concurrent
  restores pay out once.

---

## State Machine Problems

**No invalid transitions are currently reachable.** The table is enforced on every write path. For the
record, these were reachable before remediation and are now refused:

```
Delivered  → Shipped     REFUSED   (was: locked customers out of returns while the window expired)
Cancelled  → Delivered   REFUSED   (was: oversell + phantom COD revenue on a refunded order)
Delivered  → Cancelled   REFUSED   (use a return)
Shipped    → Cancelled   REFUSED   (already with the courier)
any        → Cancelled   REFUSED via UpdateOrderStatus (no compensation) → USE_CANCEL_ENDPOINT
```

**Missing states, not invalid ones:**

```
RTO → RTO_IN_TRANSIT → RTO_RECEIVED → CLOSED      only `RTO → Cancelled` exists   (H2-05)
replaced → REPLACEMENT_SHIPPED → DELIVERED        does not exist                  (H2-06)
```

---

## Financial / Razorpay Problems

| Scenario | Status |
|---|---|
| Customer under-refunded | **Latent only** — H2-03, if shipping/tax ever become non-zero. Zero today. |
| Customer over-refunded | Not reachable. Ceiling is `totalAmount − sumRefunded`, counts in-flight refunds, enforced on all four refund paths. |
| Merchant refunds twice | Not reachable. Return-status CAS + intent-first + gateway reconciliation. **But there is no DB-level uniqueness** — see the Critical section for why an index cannot express it. |
| Refund marked successful without money moving | Not reachable. `Refunded` derives from `processed` records only; `refund.failed` reverses it. |
| Payment marked paid incorrectly | **One residual path:** H2-01 — a courier event resolving to the wrong order marks that order `Delivered` and, for COD, `Paid`. Requires duplicate AWBs. |

---

## COD Problems

1. **H2-01** — a mis-resolved courier event marks the wrong COD order `Paid`. The only remaining way
   COD can be marked paid incorrectly.
2. **H2-03** — if shipping becomes non-zero, a COD return under-refunds by the shipping amount, and
   the customer is owed cash the system will not compute.
3. Payout destination is collected at request time and **cannot be edited afterwards** — there is no
   endpoint to update `refundDestination`. If the customer mistypes their UPI ID, the only route is a
   new return request, which **H2-02 blocks**. These two interact into a dead end.
4. **NOT VERIFIED — insufficient evidence:** whether payout destinations are encrypted at rest. They
   are stored as plain schema strings; I found no field-level encryption, and MongoDB-level encryption
   is a deployment property I cannot determine from this repository. Access control is enforced
   (`returns:manage` + masking); at-rest protection is unverified.

---

## Return Problems

1. **H2-02** — one return per line forever; a rejection is terminal; no reopen endpoint.
2. **H2-06** — replacements have no fulfilment tracking.
3. Return **after cancellation** is handled: units already cancelled are excluded, and a fully
   cancelled line returns `NOTHING_LEFT_TO_RETURN`.
4. Return **after refund** is handled: a refund return on a fully `Refunded` order is refused at
   creation (`ORDER_ALREADY_REFUNDED`), while a **replacement** is still allowed — correct, since a
   replacement needs no money.
5. Return **after another return** — blocked by H2-02.
6. Refund/replace **before QC** is impossible: `pickup_scheduled → received` is mandatory, and
   `refunded`/`replaced` are reachable only from `received`.
7. A **rejected** return cannot be refunded — `rejected` is terminal, and rejection from `received`
   requires an admin note.
8. QC status **cannot** be changed after refund — `refunded` is terminal.
9. Evidence images are supported, capped at 5.

---

## RTO Problems

1. **H2-05** — no `RTO_IN_TRANSIT`, no `RTO_RECEIVED`, no `CLOSED`. Arrival is detected and drives the
   restock, but is not a state.
2. The implementation does **not** treat RTO as a customer return: no refund is issued automatically
   on RTO for either payment method. For COD that is correct and final. For Razorpay a refund **is**
   owed and must be raised manually as a cancellation — correct behaviour, but **nothing prompts the
   operator**, so a prepaid RTO can sit un-refunded indefinitely. Worth a dashboard surface.
3. Restock correctly waits for `RTO DELIVERED` rather than firing on `RTO`, and is idempotent.
4. Re-dispatch of an `RTO` order is refused on all five fulfilment endpoints.

---

## Shiprocket Readiness

**Ready:**
- Order, Payment, Return and Refund are cleanly separated modules with their own models and services.
  None of them import Shiprocket code.
- Shiprocket calls are isolated behind `shiprocket.service.js` and gated by three independent feature
  flags, so the integration can be switched off entirely without touching order or refund logic.
- Credentials live in the database (admin panel), not in code.
- The courier webhook already validates transitions through the shared table, so a second provider
  reusing that entry point inherits the safety.
- Refunds are **not** coupled to shipping — `canAutoRefund` keys on the payment method only. Nothing
  about adding a provider touches the money paths.

**Needs architectural change (H2-07):**
- There is **no shipment entity and no `provider` field.** Shipment state is a `shiprocket`-named
  sub-document on the order. Adding a second provider means either a parallel sub-document per
  provider, or a migration to a provider-agnostic `shipment { provider, externalIds, awb, courier,
  status, labelUrl, ... }` with `provider: "MANUAL" | "SHIPROCKET"`.
- ~~`order.shipments[]` exists for split shipments and could host that shape, but does not today.~~
  **Removed 16 Sep 2026.** Nothing ever read the array, so it was deleted rather than kept as a
  landing site — a provider-agnostic shipment entity is the right host for that shape, not a
  second array on the order.

**Would adding Shiprocket later require rewriting order/payment/return?** **No.** The rewrite is
confined to the shipment representation and the webhook's field mapping. That is the good news, and it
is why I am rating H2-07 Medium rather than High — it is a contained refactor, not a redesign.

---

## Recommended Fix Order

Nothing below is implemented. In dependency and value order:

```
1.  HIGH    H2-01  Unique shiprocket.orderId + awbCode; refuse ambiguous webhook matches
                   (smallest change with a real money consequence)
2.  HIGH    H2-02  Decide the repeat-return policy, then implement it
                   (partial unique index + returnedQuantity accumulator, or an admin reopen)
3.  MEDIUM  H2-03  Decide the shipping/tax refund policy and encode it in BOTH the cancel and
                   return paths, with an assertion — do this BEFORE anyone sets a shipping charge
4.  MEDIUM  H2-04  Non-unique index on refunds.providerRefundId (one line, zero risk)
5.  MEDIUM  H2-08  Confirmation step on the admin refund button — do this before live keys
6.  MEDIUM  H2-05  Add RTO_IN_TRANSIT / RTO_RECEIVED / CLOSED to the enum and the table;
                   map Shiprocket's RTO sub-statuses onto them
7.  MEDIUM  H2-07  Provider-agnostic shipment shape with provider: MANUAL | SHIPROCKET
                   (do this before H2-06, which builds on it)
8.  MEDIUM  H2-06  Replacement fulfilment tracking, on top of item 7
9.  MEDIUM  H2-09  Retire InventoryModel or make the order paths write it
10. LOW     H2-11  Delete PAYMENT_MODE
11. LOW     H2-10  Consider tokenising status values if any export/integration depends on them
12. LOW     H2-12  Follows from item 7
```

Items 1, 3, 4 and 5 are small and I would do them together. Items 6–8 are one coherent piece of work
on the fulfilment model and should be planned as such rather than piecemeal.

---

## What I could not verify

- **Whether Shiprocket reuses AWB codes across orders.** Determines the real likelihood of H2-01. Not
  answerable from this repository.
- **Whether payout destinations are encrypted at rest.** No field-level encryption in the schema;
  storage-level encryption is a deployment property.
- **Whether Razorpay deduplicates concurrent identical refund calls.** No idempotency key is sent.
  No longer load-bearing — the intent-first + reconciliation design prevents duplicates on our side
  regardless — but still unverified as a gateway behaviour.
- **Production MongoDB topology.** Dev is a `rs0` replica set with `withTransaction` probed working.
  Production is a different URI and unverified; if it lacks `replicaSet`, every transactional path
  fails and checkout is fully down. See [`deployment-runbook.md`](./deployment-runbook.md) §1.4.
