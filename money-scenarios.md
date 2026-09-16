# Payment, Cancellation, Return and Refund — Every Scenario

Traced from the current code. Each row names the trigger, who is allowed to do it,
and exactly what happens to **money**, **stock** and **status**.

Two rules hold everywhere and explain most of the design:

1. **The server decides every amount.** Nothing the client sends about price,
   discount, stock, refund amount or eligibility is trusted.
2. **Money never moves inside a database transaction.** Stock, wallet, coupon and the
   refund *record* commit together; the actual gateway call happens after the commit,
   so a courier or gateway failure can never roll back a placed order.

---

## Part 1 — PAYMENT

### 1.1 The two methods behave differently at almost every step

| | `RAZORPAY` | `COD` |
|---|---|---|
| Money at order time | captured **before** the order exists | none — collected at the door |
| Stock decremented | at payment-intent creation (reserved) | at order creation |
| Order status on create | `Confirmed` | `Confirmed` |
| Payment status on create | `Paid` | `Pending` |
| Becomes `Paid` | at capture | **at delivery** |
| Refund execution | automatic via gateway | **manual** — UPI or bank transfer, UTR recorded |
| Owed on RTO | yes | **nothing** — they never paid |
| Abandonment sweep | swept when unpaid | never swept — a `Pending` COD order is real |
| Pre-order gate | none | **verified OTP required** |

### 1.2 Payment scenarios

| # | Scenario | What happens |
|---|---|---|
| P1 | **COD, OTP verified** | Order created `Confirmed`/`Pending`. Stock decremented in-transaction. OTP records **deleted** in the same transaction — one OTP buys exactly one order. |
| P2 | **COD, no OTP** | `403 COD_OTP_REQUIRED`. Nothing created. |
| P3 | **COD disabled** | `403 COD_DISABLED`. This is a **database** setting (`codEnabled`), not an env flag; a fresh database defaults it off. |
| P4 | **COD below/above value limits** | `400 COD_BELOW_MIN` / `COD_ABOVE_MAX` from `codMinOrderAmount` / `codMaxOrderAmount`. |
| P5 | **COD delivered** | `paymentStatus` `Pending → Paid`. Delivery **is** the payment event. Guarded so it cannot overwrite a refund already issued, and gated on the status transition having actually applied — otherwise it would book cash for an order that stayed cancelled. |
| P6 | **Razorpay, paid in browser** | `PaymentIntent` → capture → browser verify → `completeCapturedIntent` → order `Confirmed`/`Paid`. |
| P7 | **Razorpay, customer closes the tab after paying** | The `payment.captured` webhook runs **the same function**. Order still created, cart still cleared. This single funnel is why the browser is not load-bearing. Requires `RAZORPAY_WEBHOOK_SECRET`. |
| P8 | **Both arrive at once** (verify + webhook) | One order. The intent claim admits one; cart cleanup is claimed on `order.cartClearedAt`, so quantities are subtracted once. |
| P9 | **Razorpay abandoned** (never paid) | Stock was already decremented at intent creation. The sweeper releases the reservation (`incrementStock`, variant-aware) every 5 min and cancels the order. Meanwhile `EXCLUDE_AWAITING_PAYMENT` hides it from every revenue query, count, chart, export and customer list. |
| P10 | **Reservation expired, then payment lands** | Confirmation re-checks inside the session: `if (enforced && !activeReservation)` it decrements **now**, or fails `409 "no longer has enough stock"` if someone else took it. **No double decrement** when the reservation is still alive. |
| P11 | **Payment captured but order creation fails** | An **orphaned capture** — money taken, no order. Detected and **auto-refunded** (policy decision), with `countUnresolvedOrphanedCaptures` surfaced in System Health. |
| P12 | **Wallet covers part of the order** | `walletDiscount` reduces `totalAmount`; the balance is re-checked **inside** the order transaction, so two concurrent checkouts cannot both spend the same credit. |
| P13 | **Coupon applied** | Redemption recorded in the same transaction (`usedBy.$.count`, `usage`). |
| P14 | **Payments disabled** (`PAYMENTS_ENABLED=false`) | Online checkout unavailable, and `canAutoRefund()` is false — so **every refund becomes manual**, including cancellations. |

---

## Part 2 — CANCELLATION

### 2.1 Who can cancel

| Path | Auth | Notes |
|---|---|---|
| `PATCH /order/:orderId/cancel` | **`TokenVerify` only** | The **customer** cancels their own order. No admin approval. |
| `PATCH /order/:orderId/partial-cancel` | `orders:manage` | Admin only. |
| Status dropdown → `Cancelled` | — | **Refused** (`USE_CANCEL_ENDPOINT`): cancelling must restore stock, return wallet credit, free the coupon and record the refund owed. A bare status edit does none of that. |

### 2.2 The ordering, and why

```
╔═ withTransaction ═══════════════════════════════════════════════════════╗
║ ① CLAIM   {_id, orderStatus: {$in: ["Pending","Confirmed"]}} → Cancelled ║
║           shipment fields deliberately NOT touched — the courier has     ║
║           not been told yet, and a status must never claim otherwise     ║
║ ② incrementStock per line          (variant-aware)                      ║
║ ③ restoreWalletCredit              (capped by order.walletRefunded)     ║
║ ④ coupon release  $inc {"usedBy.$.count": -1, usage: -1}                ║
║ ⑤ recordRefundObligation                                                ║
╚═════════════════════════════════════════════════════════════════════════╝
                    │ commit
                    ▼  ⑥ settleGatewayRefund   ← money moves here
                    ▼  ⑦ cancel courier shipment ← LAST
```

Previously the courier was called *before* the local claim, so two concurrent
cancellations both killed the parcel while only one could win the claim — and if the
winner's transaction then failed, the order stayed active with a dead shipment.

### 2.3 Cancellation scenarios

| # | Scenario | Money | Stock | Result |
|---|---|---|---|---|
| C1 | **Customer cancels prepaid, before dispatch** | **Automatic Razorpay refund** of the outstanding balance | restored | `Cancelled`. No admin involvement — see §2.5 |
| C2 | **Customer cancels COD** | **Nothing** — no money was collected | restored | `Cancelled`, `paymentStatus` stays `Pending`, **zero refund rows** |
| C3 | **Cancel after `Shipped`** | — | — | `400` — the claim only accepts `Pending`/`Confirmed` |
| C4 | **Second cancellation** | — | **not restored again** | `400 "Order cannot be cancelled"` |
| C5 | **Two simultaneous cancellations** | one refund | restored **once** | one `200`, one `409`; a single `Cancelled` history entry |
| C6 | **Outside the cancellation window** | — | — | `400` naming the window. Enforced only when `cancellationWindowHours > 0`; **`0` means no limit** — currently `0` |
| C7 | **Admin partial cancel, some units** | line's share of `(total − shipping)`, valued at what was **paid** not list price | those units restored | order stays open |
| C8 | **Admin partial cancel, the last unit** | refund **topped up to the full remaining headroom**, including shipping — nothing ships, so no freight is incurred | restored | order becomes `Cancelled`; courier told |
| C9 | **Cancel with wallet + coupon used** | cash share refunded to the gateway; **wallet share returns to the wallet balance**, not the card; coupon redemption freed | restored | all four in one transaction |
| C10 | **Courier cancellation fails** | refund already done | already restored | **order stays cancelled.** Never throws. `lastError` recorded, pending state derivable, admin retry at `POST /:orderId/shipment/cancel` (refuses unless the order is locally cancelled) |
| C11 | **Shiprocket disabled** | as C1/C2 | restored | the shipment is deliberately left alone rather than marked cancelled — it really does still exist at the courier |

### 2.4 What "partial cancel" refunds

```
line share = (unitPrice × qty ÷ subtotal) × (totalAmount − shippingCharge)
           capped at remaining headroom
last unit  → refundAmount = headroom   (shipping comes back too)
```

### 2.5 The one path with no admin approval

**A customer cancelling their own unshipped prepaid order triggers a real Razorpay
refund automatically.** It is bounded — the claim only accepts `Pending`/`Confirmed`
so nothing has shipped, the amount is server-computed, the ceiling is enforced inside
the update filter, and `cancellationWindowHours` can restrict it further (currently
`0` = unrestricted).

If you want a human in the loop here, the smallest change is to record the refund as
`owed` instead of calling `settleGatewayRefund`. The order still cancels and restocks
immediately, and the money appears in **Admin → Outstanding Refunds** to be released
by hand. The ledger already supports this; no architecture change.

---

## Part 3 — RETURN

### 3.1 Lifecycle

```
pending → approved → pickup_scheduled → received ─┬→ refunded                    ●
                                                   ├→ replacement_dispatched ○
                                                   │        → replacement_delivered ●
                                                   └→ rejected                   ●

legacy: replaced ● terminal, retained for old records, unreachable now

○ open — occupies the order line     ● terminal — releases it
OPEN = {pending, approved, pickup_scheduled, received, replacement_dispatched}
```

Eligibility runs from **`deliveredAt`**, not the order date, against the product's own
`returnPolicy.windowDays`.

### 3.2 Return scenarios

| # | Scenario | Result |
|---|---|---|
| R1 | **Within window, quantity available** | Created `pending`. A refund quote is computed and stored on the return. |
| R2 | **Outside the window** | Refused. |
| R3 | **Policy `none`** | Not returnable — refused. |
| R4 | **Policy `replacement`** | Refund refused; replacement allowed. |
| R5 | **Partial quantity** (1 of 3) | Allowed. `returnableQuantity` subtracts units consumed by returns in any status **except `rejected`**, so 2 remain returnable. |
| R6 | **All units already returned** | `400 NOTHING_LEFT_TO_RETURN`. |
| R7 | **Quantity exceeds what is left** | `400 RETURN_QUANTITY_UNAVAILABLE`. |
| R8 | **Second return while one is open** | `409 OPEN_RETURN_EXISTS`, enforced by a **partial unique index** on the open statuses — not just an application check. |
| R9 | **Second return after the first is terminal** | **Allowed** — a terminal status releases the line. |
| R10 | **After a wrong QC rejection** | `rejected` is terminal and frees the slot, so the customer can re-request. There is no admin "reopen" action. |
| R11 | **Order already fully `Refunded`** | Refund return → `400 ORDER_ALREADY_REFUNDED`. A **replacement is still allowed** — it needs no money. |
| R12 | **Units already cancelled** | Excluded from the returnable count; a fully cancelled line gives `NOTHING_LEFT_TO_RETURN`. |
| R13 | **COD order, refund requested** | **Must supply a refund destination** — `UPI` (validated `name@bank`) or `bank_transfer` (account name + 9–18 digit number + valid IFSC). Otherwise `400 REFUND_DESTINATION_REQUIRED`. Without it the admin would have nowhere to send the money. |
| R14 | **Refund or replace before QC** | Impossible — `received` is mandatory and the terminal states are reachable only from it. |
| R15 | **QC rejection with no reason** | `400 QC_REJECTION_REASON_REQUIRED`. |
| R16 | **Marking `received` with no disposition** | `400 DISPOSITION_REQUIRED`. |
| R17 | **Disposition `resellable`** | Units returned to stock, claim-first on `restockedAt` so a double click restocks once. |
| R18 | **Disposition `damaged`** | Written off — nothing restocked. **The customer is still refunded.** Stock and money are separate decisions. |
| R19 | **Replacement dispatch** | Requires courier **and** tracking number. Deducts the **outbound** unit (claim-first), reverts the status and returns `409 INSUFFICIENT_REPLACEMENT_STOCK` if stock is short. Outbound deducts *before* inbound restocks, so a return cannot fund its own replacement from stock that is not back yet. Net drift: **0** resellable, **−1** damaged. |
| R20 | **Replacement statuses via the status endpoint** | Refused `USE_REPLACEMENT_ENDPOINT` — they have dedicated endpoints. |
| R21 | **Status change after `refunded`** | `refunded` is terminal — refused. |

---

## Part 4 — REFUND

### 4.1 Refunds are a ledger, not a status

Every rupee has a row in `order.refunds[]`. `paymentStatus` is **derived** from those
rows, never set by hand.

| `refunds[].status` | Means | Counts to the ceiling | Counts to `paymentStatus` |
|---|---|---|---|
| `owed` | admitted, not yet paid | yes | no |
| `created` | gateway called, outcome **unknown** | yes | no |
| `processed` | money demonstrably moved | yes | **yes** |
| `failed` | gateway confirmed it did not move | no | no |

`created` is never reused for "failed". *"Not yet"* is not *"never"* — that
distinction is what stops a lost response being refunded twice.

```
sumRefunded       = owed + created + processed   ← the CEILING (includes in-flight)
sumSettledRefunds = processed only               ← what paymentStatus may claim

settled ≥ total  → "Refunded"
settled > 0      → "Partially Refunded"
committed > 0    → "Refund Pending"    money owed, none moved yet
a failed refund reverts it to "Paid"
```

### 4.2 Who can cause a refund

| Path | Auth | Money moves |
|---|---|---|
| Customer cancel | `TokenVerify` **only** | **automatically** |
| Admin refund `POST /payment/razorpay/refund/:orderId` | `orders:manage` + typed-amount confirmation | yes |
| Admin partial cancel | `orders:manage` | yes |
| Return → `refunded` | `returns:manage` | yes |
| Reconcile `POST /:orderId/refunds/:refundId/reconcile` | `orders:manage` | only after Razorpay confirms none exists |
| RTO arrival (webhook, unattended) | none | **no** — records `owed` only |

### 4.3 Refund scenarios

| # | Scenario | Result |
|---|---|---|
| F1 | **Prepaid cancel** | Automatic gateway refund of `totalAmount − sumRefunded`. |
| F2 | **COD cancel** | **Nothing.** No money was collected. |
| F3 | **Prepaid return** | Line's share of `(total − shipping)`. Tax included, shipping **excluded**. |
| F4 | **COD return** | Recorded `owed` with `confirmationMethod: "manual"`, paid out-of-band to the customer's UPI/bank, UTR recorded against the row. **Never sent to Razorpay.** |
| F5 | **RTO, prepaid** | `owed` obligation for the outstanding balance. **Never auto-pushed** — an unattended courier feed must not issue irreversible refunds. Appears in Admin → Outstanding Refunds. |
| F6 | **RTO, COD** | Nothing owed. |
| F7 | **RTO event replayed** | One obligation. Dedupe key `RTO <orderId>` with a conditional `$push`. |
| F8 | **Admin manual refund** | Requires the operator to **type the amount** to match, and the dialog supplies an idempotency key so a double-click cannot produce two refunds. |
| F9 | **Refund exceeding the order value** | Refused. `COMMITTED_REFUND_SUM` (a `$reduce`) is evaluated **inside the update filter**, so the ceiling is enforced by the database, not by a read. |
| F10 | **Two concurrent identical refunds** | **One** gateway call. `claimRefundSlot` then `claimGatewayAttempt`; the loser gets `409`. |
| F11 | **Gateway times out** | Row stays `created` (outcome unknown), attempt claim **released** so a retry is not blocked. `paymentStatus` does not claim "Refunded". |
| F12 | **Gateway succeeded but our DB write failed** | `notes.refundKey` is sent to Razorpay; `findGatewayRefundByNote` asks the gateway before retrying, so the existing refund is **adopted** rather than duplicated. `ReconcileOrderRefund` is the operator path. |
| F13 | **Retry after a lost response** | Safe — same idempotency key, verify-before-retry. |
| F14 | **Wallet-funded portion** | Returns to the **wallet balance**, capped by `walletRefunded` with the cap in the filter. |
| F15 | **Refund while one is unsettled** (`Refund Pending`) | Allowed. `Refund Pending` still means money was collected, so a further obligation can be recorded against the remaining balance. |
| F16 | **Payments disabled** | `canAutoRefund()` false ⇒ every refund becomes manual. |
| F17 | **Same gateway refund id on two orders** | Impossible — `refunds.providerRefundId` is unique. |

### 4.4 Shipping and tax

| Event | Goods | Tax | Shipping |
|---|---|---|---|
| Return | refunded | refunded | **not** refunded |
| Pre-dispatch cancellation | refunded | refunded | **refunded** |
| Partial cancel, last unit | refunded | refunded | topped up |

Tax follows the goods — it was levied on their value, so returning them reverses the
sale it was charged on; keeping it both short-changes the customer and over-reports
GST. Shipping does not follow the goods: the parcel shipped and the courier was paid.
A cancellation is the opposite case — nothing shipped.

> **Currently inert.** `shippingCharge` and `tax` are hardcoded `0` in
> `order-pricing.service.js`, so the arithmetic above is correct but has never run
> against non-zero values in production.

---

## Part 5 — RTO (return to origin)

| # | Scenario | Money | Stock |
|---|---|---|---|
| T1 | Courier reports `RTO` | nothing | nothing |
| T2 | `RTO Received` (arrival, status 43) | prepaid → `owed`; COD → nothing | **nothing — deliberately** |
| T3 | Disposition `resellable` | unchanged | restocked, claim-first |
| T4 | Disposition `damaged` | unchanged | written off |
| T5 | `Closed` | unchanged | unchanged — closing never restocks |
| T6 | Close **without** a disposition | — | **`409 RTO_DISPOSITION_REQUIRED`** |

Arrival does not restock because the goods have not been inspected — a courier can
damage a box in either direction. And `Closed` requires a disposition because
`RecordRtoDisposition` accepts only `RTO Received` while `Closed` is terminal:
closing early would strand the units permanently, neither restocked nor written off,
with no path back.

---

## Part 6 — Quick answers

| Question | Answer |
|---|---|
| Does an admin approve every refund? | **No** — a customer cancelling their own unshipped prepaid order refunds automatically. Everything else needs an admin. |
| Can a COD order be refunded to a card? | No. COD refunds are manual to UPI or bank transfer, with the destination collected at return time and a UTR recorded. |
| Can a customer be refunded twice? | No. Four layers: idempotency key, ceiling in the update filter, gateway-attempt claim, verify-before-retry. |
| Can a refund exceed the order value? | No — the ceiling is enforced inside the update filter. |
| Does RTO refund automatically? | No. It records `owed`; a human releases it. |
| Does a damaged return still get refunded? | Yes. Disposition decides **stock**, not money. |
| Can a customer return twice on one line? | One **open** return at a time; more once terminal, up to the purchased quantity. |
| Is wallet credit refunded as cash? | No — back to the wallet balance. |
| What happens to a coupon on cancellation? | The redemption is freed, in the same transaction. |
| Is there a cancellation deadline? | Only if `cancellationWindowHours > 0`. It is currently `0` = no limit. |
