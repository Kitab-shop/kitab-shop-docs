# Order, Payment, Cancellation, Return & Refund Flow

How money and orders actually move through this system, and the rules enforced at each step.

**Core principle:** a status is never allowed to claim something that hasn't happened. An order says
`Refunded` only once money has demonstrably moved; a return says `refunded` only after the payout
succeeded. Where money is owed but has not moved, the state is **`Refund Pending`** — never
`Refunded`. Where the outcome is *unknown* (a gateway timeout), it is recorded as unknown and
reconciled against the gateway rather than guessed.

**Reading this doc:** the invariants below are enforced in code and covered by
`cd kitab-shop-be && npm test` (174 assertions, run against a real database — every bug these
guard against was a concurrency bug, and a mock would have passed the broken code). Related:
[`production-readiness-audit.md`](./production-readiness-audit.md) for what was wrong and why ·
[`deployment-runbook.md`](./deployment-runbook.md) for the config these paths depend on ·
[`audit-2-flow-conformance.md`](./audit-2-flow-conformance.md) for where this still diverges from the
intended design ·
[`shiprocket-integration.md`](./shiprocket-integration.md).

---

## 1. Two payment lifecycles

Razorpay and COD share one order/cancellation/return framework but invert *when* payment happens.

```
RAZORPAY:   PAYMENT → SHIPPING → DELIVERY
COD:        ORDER   → SHIPPING → DELIVERY → PAYMENT
```

| | Razorpay | COD |
|---|---|---|
| On order creation | `paymentStatus: Pending`, `orderStatus: Pending` | `paymentStatus: Pending`, `orderStatus: Confirmed` |
| On payment verified | **`Paid` / `Confirmed`** | n/a |
| On delivery | unchanged | **`paymentStatus: Paid`** (cash collected) |
| Refund route | back to the original payment method via API | manual payout (UPI / bank transfer) |
| Confirmed by | Razorpay's `refund.processed` webhook | a human recording a UTR |

**Order-first.** A Razorpay order row is created *before* the customer pays, so
there is an order number to show immediately and a durable record to reconcile a
captured-but-unmatched payment against. It carries no money and no commitments:
coupon redemption, the wallet debit and the stock decrement all still happen at
capture, inside the transaction — burning a coupon on a checkout abandoned thirty
seconds later would be wrong.

The promotion to `Paid`/`Confirmed` is a **compare-and-swap** on
`paymentStatus: "Pending"`, so a verify call and a `payment.captured` webhook
arriving together promote once and run the compensations once.

> **The rule this design lives or dies by:** an unpaid prepaid checkout is a row in
> `orders`, but it is **not an order**. Everything that produces revenue, counts,
> charts, exports or customer-facing lists must exclude it — see §14. A COD order at
> `Pending` is genuinely real (the customer placed it and owes cash on delivery), so
> the exclusion is deliberately narrow: prepaid **and** unpaid, nothing else.
>
> Abandoned checkouts are swept to `Cancelled`/`Failed` after 20 minutes rather than
> sitting `Pending` forever.

COD is **never** marked paid at creation. It becomes `Paid` the moment the order reaches
`Delivered`, in both paths that can set that status:

- `UpdateOrderStatus` (admin) — guarded on `paymentStatus === "Pending"`, so a later status edit
  cannot overwrite a refund that has since been issued.
- The Shiprocket webhook — same guard, expressed as a filtered update
  (`{paymentMethod: "COD", paymentStatus: "Pending"}`), and additionally gated on the transition
  having been *accepted*. A stale `Delivered` event for an order the customer already cancelled does
  not book phantom COD revenue.

---

## 2. Full flow

```
                              CUSTOMER
                                 │
                            Place Order
                                 │
                  ┌──────────────┴──────────────┐
                  ▼                             ▼
            RAZORPAY                          COD
     paymentStatus = Pending          paymentStatus = Pending
     orderStatus   = Pending          orderStatus   = Confirmed
                  │                             │
          payment verified                      │
       (CAS: exactly one wins)                  │
                  │                             │
                  ▼                             │
       paymentStatus = Paid                     │
       orderStatus   = Confirmed                │
                  │                             │
        (unpaid after 20 min                    │
         → swept to Cancelled)                  │
                  └──────────────┬──────────────┘
                                 │
                     ┌───────────┴───────────┐
                     ▼                       │
              CANCEL (window)                ▼
              atomic claim —               Packed
              exactly one wins               │
                     │                       ▼
                     │                    Shipped
                     │                       ▼
                     │                Out For Delivery
                     │                  ┌────┴────┐
                     │                  ▼         ▼
                     │              DELIVERED    NDR ──┐
                     │           COD → Paid here   │   │ reattempt
                     │                  │          │◄──┘
                     │                  │          ▼
                     │                  │         RTO
                     │                  │    (initiated /
                     │                  │     in transit)
                     │                  │          │
                     │                  │          ▼
                     │                  │    RTO RECEIVED
                     │                  │   parcel back with us
                     │                  │          │
                     │                  │   ┌──────┴───────┐
                     │                  │   ▼              ▼
                     │                  │ MONEY          STOCK
                     │                  │   │              │
                     │                  │ ┌─┴──┐      inspection
                     │                  │ ▼    ▼        (human)
                     │                  │RZP  COD          │
                     │                  │owed  no       ┌───┴────┐
                     │                  │     refund    ▼        ▼
                     │                  │           RESELLABLE DAMAGED
                     │                  │            restock   write off
                     │                  ▼
                     │           RETURN REQUEST
                     │           per-product policy
                     │           window from deliveredAt
                     │                  ▼
                     │               APPROVED
                     │                  ▼
                     │          PICKUP SCHEDULED
                     │                  ▼
                     │               RECEIVED
                     │                  │
                     │               QC CHECK
                     │          ┌───────┴───────┐
                     │          ▼               ▼
                     │       PASSED           FAILED
                     │          │               │
                     │     ┌────┴────┐       REJECTED
                     │     ▼         ▼       admin note
                     │  REFUND   REPLACEMENT (goods go back
                     │     │         │        to customer)
                     │     │         ▼
                     │     │    Replacement
                     │     │    fulfilment
                     │     └────┬────┘
                     │          │
                     │   INVENTORY DISPOSITION
                     │   (independent of the
                     │    customer's resolution)
                     │          │
                     │    ┌─────┴─────┐
                     │    ▼           ▼
                     │ RESELLABLE  DAMAGED
                     │  restock    write off
                     │
                     ▼
              ┌─────────────────┐
              │ REFUND REQUIRED │
              └────────┬────────┘
                       │
        NOTE: a COD order cancelled BEFORE delivery
        owes nothing — no money was ever collected.
        Only a delivered-then-returned COD order does.
                       │
                ┌──────┴──────┐
                ▼             ▼
             RAZORPAY        COD
                │             │
          Refund Pending  Refund Pending
          (owed→created)  (owed, manual)
                │             │
          Razorpay Refund  Manual Payout
                │             │
          refund.processed  UPI / Bank UTR
          (gateway         (human confirms,
           confirms)        reference required)
                └──────┬──────┘
                       ▼
                   REFUNDED
                       │
              refund.failed reverses
              it back out of Refunded
```

**Two things this diagram is careful about**, because both were wrong in earlier
versions:

1. **`NDR` can be reattempted.** Couriers try two or three times before giving up,
   so `NDR → Out For Delivery` and `NDR → Delivered` are both legal. Forcing every
   failed first attempt straight to RTO would be wrong.
2. **The refund fork is not symmetrical.** A cancelled COD order owes nothing; a
   returned COD order owes cash. Collapsing them into one "refund required" box is
   the mistake that leads to refunding customers who never paid.

---

## 3. Cancellation

**Eligibility — both rules must pass:**
1. `orderStatus` is `Pending` or `Confirmed` (nothing shipped yet)
2. Within `cancellationWindowHours` of placement (Admin → Operations → Checkout; **0 = no limit**)

Past either rule the self-service button disappears and the customer is pointed at support.
Enforced server-side in `CancelOrder` (`CANCEL_WINDOW_CLOSED`), not just hidden in the UI.

**The transition is claimed atomically.** The first statement inside the transaction is a
`findOneAndUpdate` whose *filter* carries the precondition:

```js
{ _id: orderId, orderStatus: { $in: ["Pending", "Confirmed"] } }
```

Losing that claim returns **409**. This matters more than it looks: `session.withTransaction()`
retries its callback on a write conflict, so an eligibility check read *before* the transaction could
be passed twice — re-crediting the wallet, re-restocking, and pushing a second refund for one order.
Putting the precondition in the filter means exactly one execution can ever win.

**A reason is mandatory** — a fixed list, kept in `CANCEL_REASONS` in `order.controller.js` and
mirrored in `orderDetail.helpers.js` on the frontend. Stored on `order.cancellation` and echoed into
the refund record. An unrecognised reason falls back to `"Other"` rather than failing the
cancellation; a bad string should never block someone's refund.

**What cancellation does:**

| Action | Detail |
|---|---|
| Restock | Excludes units already cancelled (`quantity − cancelledQuantity`), so a partial cancel followed by a full cancel cannot double-restock. Variant-aware. |
| Wallet | Returns the wallet portion, **capped by `order.walletRefunded`** — see §5. |
| Coupon | A single conditional update (`usedBy.$.count` `$gte: 1` in the filter) frees the redemption atomically. Mongoose validators do not run on update operators, so this no longer depends on a subdocument `min`. |
| Shiprocket | Cancels the shipment if one exists. |
| Refund | Recorded as **owed**, then executed after commit — see below. |

**Refund on cancellation:**

| Order | Behaviour |
|---|---|
| COD (`paymentStatus: Pending`) | **No refund** — no money was ever collected |
| Razorpay (`Paid` or `Partially Refunded`) | `Refund Pending` → gateway refund → `Refunded`, or stays `Refund Pending` with `failureReason` |

`Partially Refunded` counts as money-collected. Gating only on `Paid` meant that once a partial
cancellation had moved the status, a later full cancellation refunded **nothing**. The amount owed is
the **outstanding balance** (`totalAmount − sumRefunded`), not the whole order total.

The gateway call happens **after** the DB transaction commits, deliberately. Calling an external
payment API inside a transaction risks the money moving while the transaction rolls back, leaving a
refunded customer holding a live order.

**Cancelling is not a status change.** `UpdateOrderStatus` explicitly refuses `Cancelled`
(`USE_CANCEL_ENDPOINT`) and points the admin at the cancel endpoint. It performs none of the
compensation above, and it used to be the only path that could cancel a `Shipped` or `Delivered`
order.

---

## 4. Order status transitions

`orderStatus` is no longer free-form. `order-status.rules.js` holds the permitted moves, enforced in
both the admin endpoint and the courier webhook, and mirrored in the admin dropdown
(`orderStatus.rules.js` on the frontend, asserted to match).

| From | May move to |
|---|---|
| `Pending` | `Confirmed`, `Packed`, `Cancelled` |
| `Confirmed` | `Packed`, `Shipped`, `Cancelled` |
| `Packed` | `Shipped`, `Cancelled` |
| `Shipped` | `Out For Delivery`, `Delivered`, `NDR`, `RTO` |
| `Out For Delivery` | `Delivered`, `NDR`, `RTO` |
| `NDR` | `Out For Delivery`, `Delivered`, `RTO` |
| `RTO` | `RTO Received`, `Cancelled` |
| `RTO Received` | `Cancelled` (close out once any refund owed is settled) |
| `Delivered` | — **terminal** |
| `Cancelled` | — **terminal** |

Re-submitting the same status is a legal no-op rather than an error, so a double-click doesn't spam
`statusHistory`.

**Why a table and not just an enum check:** the enum was satisfied by any value, so a replayed or
out-of-order courier event could regress `Delivered → Shipped` (locking the customer out of returns
while their window kept expiring) or resurrect `Cancelled → Delivered` (overselling plus phantom COD
revenue on an order already restocked and refunded).

Every change appends to **`order.statusHistory`** (`from`, `to`, `changedBy`, `source`, `changedAt`).
`changedBy` is null for machine changes, so a courier webhook is distinguishable from a human.

---

## 5. Refund amounts: discount-aware, and split by how the money was taken

Item prices are stored **pre-discount**; coupon and wallet reductions apply once at the order level.
So `price × qty` is what an item *listed* for, not what the customer *paid*.

A returned line is therefore worth two separate amounts, because they are settled by two different
mechanisms:

```
cash share   = (unitPrice × qty) × min(1, orderTotal ÷ orderSubtotal)     ← proportionalRefundAmount
wallet share = (unitPrice × qty) × min(1, walletDiscount ÷ orderSubtotal) ← proportionalWalletRefund
```

Both use the same subtotal share, so together they always equal the full value of the returned goods.

**Example** — ₹1000 of goods, ₹200 paid from wallet credit, ₹800 charged to the card. A full return
owes **₹800 back to the card and ₹200 back to the wallet balance.** Returning only half owes ₹400 and
₹100.

Previously only the cash half was returned, so every return silently destroyed the customer's prepaid
credit. There is no card transaction to reverse wallet money against, which is why it goes back as
wallet credit rather than through the gateway.

Both amounts are recorded on the return at request time (`refundAmount` and `walletRefundAmount`), so
the customer is quoted the whole sum they are owed and the operator can see which half is automatic.

**The wallet half is capped by `order.walletRefunded`**, incremented by a conditional `$inc` whose
filter contains the ceiling. That makes it idempotent: a partial cancel that returned ₹120 leaves
only ₹80 available to a later full cancel, and five concurrent restores pay out ₹200 in total rather
than ₹1000. It is credited **before** the gateway call and on every path, including the
already-refunded early return — the wallet operation is idempotent while the gateway call is not, so
the prepaid portion reaches the customer even if the card refund then fails.

The cash ratio is capped at 1 so an order whose total exceeds its subtotal can never inflate a refund
above the item's own price. It falls back to the full line value when subtotal is 0.

> **On restoring an old database snapshot:** returns created before the discount-aware change carry
> a pre-discount `refundAmount`, which on a discounted order would trip the over-total guard. The
> live database was audited at the time — 6 returns, 1 open, and that one was already correct — so no
> migration was needed. Re-check open returns on coupon/wallet orders if you restore an older
> snapshot.

---

## 6. Returns & replacements

**Per-product policy** (`product.returnPolicy`, set in the admin product form):

| `kind` | Meaning |
|---|---|
| `return` | refundable within `windowDays` of delivery |
| `replacement` | replacement only — never a refund |
| `none` | not eligible at all |

The window counts from **`order.deliveredAt`**, stamped the *first* time the order reaches
`Delivered` and never re-stamped — re-confirming the status would otherwise silently re-open an
expired window. Orders delivered before that field existed skip the window check rather than being
blocked. The policy is shown on the product page *before* purchase and enforced in
`CreateReturnRequest`, so the return option is never offered for something the server would reject.

**Refused at creation, not after the goods have been collected:**

| Condition | Response |
|---|---|
| Order not `Delivered` | 400 |
| Product policy is `none` | 400 |
| Past the policy window | 400, naming the window and its length |
| Quantity exceeds `quantity − cancelledQuantity` | 400, quoting the **returnable** quantity |
| Every unit already cancelled | 400 `NOTHING_LEFT_TO_RETURN` |
| A refund on an order already fully `Refunded` | 400 `ORDER_ALREADY_REFUNDED` |
| A return already exists for that line | 409 |
| Not the customer's own order | 403 |

The last two of the amount checks matter because they used to land *later*: a unit already cancelled
and refunded was still fully returnable (paying the customer twice for it), and the
already-refunded block only fired when the admin tried to settle — by which point a courier had been
booked and the customer's goods collected for a return that could never be paid.

**Status machine:**

```
pending ──→ approved ──→ pickup_scheduled ──→ received ──→ refunded    (resolutionType: refund)
   │           │                                   │   └──→ replaced   (resolutionType: replacement)
   └──→ rejected ←───────────────────────────────  ┘   (QC failure — admin note REQUIRED)
```

- `resolutionType` is decided **at request time** from the product's policy — not a customer choice.
  A replacement return can never be marked `refunded`, and vice versa.
- **`received → rejected` is the QC gate.** Without it the admin was forced to refund or replace
  whatever arrived in the box, even if damaged, used, or the wrong item. Rejecting after receipt
  requires an admin note, because the customer is shown that reason.
- **The status change is a compare-and-swap** (`findOneAndUpdate` filtered on the previous status)
  taken *before* any money moves. Two admins clicking simultaneously: one wins, the other gets 409
  `RETURN_STATUS_CONFLICT`. If the refund then fails, the status is reverted to where it was — a
  return must never read as `refunded` with nothing settled.

**Resolution and inventory disposition are two separate decisions.** Whether the customer gets
their money and whether the goods can be sold again are independent, and conflating them forced a bad
choice: a genuinely faulty item meant either refunding the customer *and* putting a defective unit
back on sale, or rejecting a legitimate return to protect the shelf.

Resolving a return therefore requires an explicit `disposition`, with **no default** in either
direction — defaulting to `resellable` reintroduces the defective-stock bug, defaulting to `damaged`
quietly writes off good stock:

| `disposition` | Effect |
|---|---|
| `resellable` | units return to sellable stock |
| `damaged` | units are written off; **the customer is still refunded or replaced** |

`rejected` never restocks either — those goods go back to the customer. The restock is claim-first
(`restockedAt` stamped under a conditional update), so a second status update or a double-clicked
button restocks nothing. A write-off is reported explicitly in the response rather than being
inferred from the absence of a restock message, because it is a real inventory loss.

---

## 7. Executing a refund

Routed by `canAutoRefund(order)` — `paymentMethod === "RAZORPAY"` **and** a `razorpayPaymentId`
exists **and** payments are enabled.

### Razorpay path — intent first, then the gateway

1. A refund record is persisted with `status: "created"` **before** `razorpay.payments.refund()` is
   called, tagged with `notes.returnNumber`.
2. The gateway is called.
3. The record is updated with whatever the gateway actually said — `processed`, or `failed` if
   Razorpay reported failure.

The ordering is the point. Previously the gateway was called first, so a **response timeout on a
refund Razorpay had already processed left no trace at all** — and the operator's retry paid the
customer a second time. With the intent on disk, a retry finds it and asks Razorpay
(`payments.fetchMultipleRefund()`, matched on `notes.returnNumber`) instead of blindly re-refunding.

On failure the record stays **`created`, not `failed`**, with `failureReason` set. That distinction is
load-bearing: `failed` records are excluded from the refund total and from the
one-refund-per-return lookup, so marking an unknown outcome as `failed` would make the next attempt
skip reconciliation and risk a double payment. `created` means *"outcome unknown — verify before
retrying"*.

If Razorpay is unreachable during reconciliation the operation **fails loudly**
(`REFUND_RECONCILE_UNAVAILABLE`) rather than assuming the earlier refund never happened. Unknown is
not the same as didn't-happen.

### Manual path — COD and anything else

There is no payment to reverse, so:

1. The **customer supplies a payout destination** when raising the return — UPI ID, or account name +
   number + IFSC, all validated server-side. Required for COD refunds; not asked for replacements or
   Razorpay orders.
2. The admin sees that destination in the refund dialog, pays the customer, then records **method +
   reference number** (UPI txn id, bank UTR).
3. Without a reference the refund is **refused** — otherwise we'd recreate the "says refunded but
   isn't" problem for COD.

Those destination details are sensitive, so reads are gated accordingly: a non-owner needs
`returns:manage` (not merely an admin-tier role — `themeEditor` could previously enumerate customer
bank accounts), and the owner sees their own account number **masked** to the last four digits. Only
an operator who actually has to push the money receives it in full.

### The four-state ledger

`owed` and `attempted` are different liabilities, and collapsing them hid money — a COD return that
passed QC had **no ledger row at all** until someone paid it, so "how much do we owe customers right
now?" was unanswerable from the data.

| Status | Meaning |
|---|---|
| `owed` | liability recorded, nothing attempted. A COD payout in the queue, or a prepaid RTO refund awaiting an operator. |
| `created` | the gateway was called and the outcome is **unknown** (timeout). |
| `processed` | money has demonstrably moved. |
| `failed` | the gateway confirmed it did not move. |

Each row also records `confirmationMethod`, so a payout waiting on a **human** is distinguishable
from a refund waiting on **Razorpay**. Both read as `Refund Pending` on the order, but they need
different queues and different chasing:

| | `gateway` | `manual` |
|---|---|---|
| Confirmed by | `refund.processed` webhook | an admin recording a UPI txn id / bank UTR |
| Can reach `processed` without evidence? | no — the webhook is the evidence | no — the reference is mandatory |

`sumRefunded` (the ceiling) counts `owed` and `created`; `sumSettledRefunds` (which decides
`paymentStatus`) counts only `processed`. `sumOwedRefunds` is the payout queue.

**Never owe money to someone who didn't pay.** `recordRefundObligation` refuses an order whose
`paymentStatus` shows nothing was collected. That check lives in the function, not only in its
callers, because an RTO is exactly the case where it is easy to get wrong — a parcel coming back
*looks* like a return.

### Guards

- The amount comes from the return's own `refundAmount` — no free-text field to mistype
- Cumulative refunds can never exceed `order.totalAmount`, and the ceiling counts in-flight
  (`created`) records so a second refund cannot slip past it
- One refund per return, linked via `refunds[].returnRequest`
- A repeat attempt is idempotent — it returns the existing record, or reconciles against the gateway
- A wallet-only refund (a line paid entirely from credit) completes without fabricating a
  zero-value gateway call

---

## 8. Payment status is derived, never asserted

`recomputeRefundState(order)` is the single place `paymentStatus` is decided, from two different sums:

| Sum | Includes | Used for |
|---|---|---|
| `sumRefunded` | everything except `failed` — **including in-flight `created`** | the "never refund more than was paid" ceiling (conservative on purpose) |
| `sumSettledRefunds` | `processed` only | deciding `paymentStatus` |

```
settled ≥ totalAmount   → Refunded
settled > 0             → Partially Refunded
committed > 0           → Refund Pending      (money owed, nothing moved yet)
neither, and the order  → Paid                (every refund failed — the money never left us)
was in a refund state
```

That last rule is why a gateway-failed refund does not leave an order stranded. It only applies to
orders already in a refund state, so a COD order still awaiting collection keeps its `Pending`.

**An intent record can never move an order to `Refunded`.** That is the whole invariant of this
module: a status may not claim something that hasn't happened.

---

## 9. Webhooks are the source of truth for outcomes

Registered events: `payment.captured`, `payment.failed`, `refund.processed`, `refund.failed`
(see [`deployment-runbook.md`](./deployment-runbook.md) §1.3 to set this up).

| Event | Effect |
|---|---|
| `payment.captured` | Creates the order if the browser never reached `/razorpay/verify`. Verifies the paid amount matches the checkout amount. |
| `payment.failed` | Marks the attempt failed — **only if it is still open** (`status: {$in: ["created","processing"]}`). |
| `refund.processed` | Marks the refund settled → `paymentStatus` recomputed. |
| `refund.failed` | Marks it failed → the order drops **back out of** `Refunded`. |

Three properties worth knowing:

- **Replay protection is a unique index**, not an application check. Each delivery claims its
  `x-razorpay-event-id` in a `webhookevents` collection with a unique `{provider, eventId}` index;
  a duplicate gets E11000 and returns 200 without reprocessing. Two concurrent deliveries of the
  same event race and exactly one wins — something a read-then-check could not guarantee.
- **A processing failure answers 5xx, not 200,** and releases the event claim so Razorpay's retry
  can reprocess. The handler previously swallowed every error as a success, so Razorpay recorded
  delivery and never retried: a transient DB error silently lost a real payment event forever.
- **`refund.processed` / `refund.failed` exist because normal-speed refunds settle asynchronously**
  and can subsequently fail. Without them, a gateway-failed refund stayed `Refunded` in our ledger
  forever with the customer already notified.

**Retries do not orphan payments.** A retry issues a *new* Razorpay order and used to overwrite
`razorpayOrderId` in place, so a payment completed against the old one became unresolvable — no
order, no auto-refund, customer charged for nothing. Superseded ids are now kept in
`previousRazorpayOrderIds`, and both the webhook and the verify endpoint match either. Verification
signs against the id the payment was actually made on.

---

## 10. RTO — return to origin

If the courier can't deliver and the parcel comes back:

- **COD:** the customer never paid, so nothing is owed. You absorb the freight cost.
- **Razorpay:** the customer *did* pay, so a refund is owed — handle it as a cancellation.

**`RTO` and `RTO Received` are separate states.** Shiprocket reports "RTO Initiated" and "RTO In
Transit" long before the goods reach the warehouse. `RTO` covers those; `RTO Received` is the parcel
physically back with us, detected from Shiprocket status id 43 (or matching
`RTO DELIVERED` / `RTO RECEIVED` text). Both the money and the stock consequences hang off *arrival*,
not off the courier giving up.

**On arrival the two consequences are handled separately, and neither is automatic in the wrong way:**

| | What happens |
|---|---|
| **Money** | A refund obligation is recorded as `owed` — **not** pushed to the gateway. This runs unattended from a courier feed, and an automatic irreversible refund triggered by a courier event is not something to do without a human. COD records nothing: the customer never paid. Idempotent on a stable dedupe key, so retried events record one liability. |
| **Stock** | **Nothing is restocked yet.** The goods have not been inspected, and a courier can damage a box in either direction. An operator records the disposition (`PATCH /order/:orderId/rto-disposition`, admin-only, requires `RTO Received`) and the restock happens then — claim-first via `order.rtoRestockedAt`, excluding units already cancelled. |

This path previously restocked every unit blind the moment the webhook reported arrival.

An order at `RTO` also cannot be re-dispatched: `AssignAwb`, `SchedulePickup`, `GenerateLabel` and
`GenerateInvoice` all refuse `Cancelled` **and** `RTO` with 409. Previously only `CreateShipment`
checked, so a cancelled — quite possibly already refunded — order could still be given an AWB, have
a courier pickup booked, and get a label printed, and the parcel would physically leave with the
money already returned.

---

## 11. Shiprocket does not refund customers

Worth stating explicitly, because it's a natural assumption and it's wrong.

Shiprocket moves COD money in one direction only: customer → courier → Shiprocket → **you**
(standard remittance **D+8**; Early COD plans D+2/D+3/D+4 for a fee). Their support docs are explicit
that returns and refunds are the seller's responsibility.

**Cash-flow consequence:** with D+8 remittance a customer can return an item *before* Shiprocket has
paid you, so you refund out of pocket and reconcile later.

---

## 12. Payment status reference

| Status | Meaning |
|---|---|
| `Pending` | Razorpay awaiting payment, **or** COD not yet delivered |
| `Paid` | Razorpay verified, **or** COD cash collected on delivery, **or** every refund attempt failed so the money never left |
| `Failed` | payment attempt failed |
| `Refund Pending` | **money is owed but has not moved** — recorded as `owed`, in flight as `created`, or a COD payout not yet made. `sumOwedRefunds` separates the queue from the in-flight. |
| `Partially Refunded` | some but not all of the order total has settled |
| `Refunded` | fully refunded, confirmed settled at the gateway |

---

## 13. Stock

Every product-stock write goes through one variant-aware helper (`variant.service.js`); no order path
touches `product.stock` directly, which is asserted by a test so a future change can't quietly
regress it.

- **Sale:** a single conditional update constrains the product total **and** the chosen variant's
  stock in the *filter*, so it applies to both atomically or to neither. `product.stock` is the sum
  across variants, so a product with 10 units split 10 Red / 0 Blue would otherwise happily sell a
  Blue. An inactive variant is unsellable at any stock level.
- **Reservation:** the reservation row is created **before** any stock is taken and each line is
  appended as its decrement succeeds, so the row always describes what was actually taken. A crash
  mid-loop is then recoverable by the expiry job — previously stock was deducted first and the row
  written afterwards, so a crash in between leaked units permanently with no record anywhere.
- **Release:** the status flip is a compare-and-swap taken before any stock moves, so a webhook retry
  and the expiry job cannot both restock the same reservation.
- **Commit:** the reservation is read *inside* the transaction session and the commit result is
  checked. A null means the reservation expired between the check and here, and the order is aborted
  rather than confirming a paid order with no inventory behind it.
- **Return / RTO:** restocked as described in §6 and §10, variant-aware. Units from a variant deleted
  since the sale fall back to the product-level pool rather than being written off.

With `INVENTORY_ENFORCE_STOCK=false` none of this runs — including the restock, because there is no
ledger to restock into.

---

## 14. Order visibility — the rule order-first depends on

Because a Razorpay order exists before payment, the `orders` collection contains
checkouts that were started and abandoned. **Those are not orders.** No money, no
committed stock, and the customer does not believe they bought anything. Counting one
as revenue is the single most likely way this design goes wrong.

`order-visibility.js` is the one definition:

```js
AWAITING_PAYMENT_MATCH   = { paymentMethod: "RAZORPAY", paymentStatus: "Pending" }
EXCLUDE_AWAITING_PAYMENT = { $nor: [AWAITING_PAYMENT_MATCH] }
isAwaitingPayment(order)
```

Deliberately narrow: **prepaid and unpaid**. A COD order at `Pending` is real — the
customer placed it and owes cash on delivery — so it is never excluded.

Applied at every site where including one would be wrong:

| Site | Why it matters |
|---|---|
| `totalRevenue` aggregate | an abandoned checkout would be booked as revenue |
| total order count | inflated order volume |
| 7-day orders/revenue chart | inflated daily figures |
| recent-orders list | shows things nobody bought |
| sales report (`$match`) | wrong totals by date, product, category, customer |
| per-customer order aggregate | wrong count and spend on a customer's profile |
| customer order count | same, on the admin user list |
| **coupon allowance check** | starting a checkout with a coupon and abandoning it would **burn the customer's use** |
| **referral first-order check** | one abandoned attempt would make a genuine first order look like a second and **deny the referrer their reward** |
| `GetMyOrders` | the customer would see a checkout they abandoned |
| `GetAllOrders` | hidden by default; `?includeAwaitingPayment=true` opts in for diagnosing a stuck checkout |

`$nor` rather than a negated `$and` so it composes with a caller's own
`paymentStatus`/`paymentMethod` conditions without colliding.

**This is enforced by a test, not by discipline.** `test/regression/lifecycle.regression.mjs`
walks every file under `src/`, finds anything that aggregates or counts orders, and
fails unless it either applies the filter or appears in an explicit exempt list with
a stated reason. It caught two files during implementation that a manual sweep had
missed.

**Abandoned checkouts are closed out, not left Pending.** After
`AWAITING_PAYMENT_TTL_MS` (20 minutes, matching the payment-intent TTL) the sweeper
moves them to `Cancelled` / `Failed` with a reason and a history entry. Cancelled
rather than deleted — an attempted checkout is a fact worth keeping. No stock or
money is touched, because an unpaid order held neither; reserved stock is released by
the reservation sweeper on its own expiry, so the two never both act on the same
units. The status filter makes it a claim, so a payment completing at that exact
moment wins and its order is never cancelled out from under it.

> **A trap worth knowing about**, because it would have taken checkout down entirely:
> `razorpayOrderId` and `razorpayPaymentId` are `UNIQUE + sparse`, and **`sparse`
> skips only ABSENT fields — a stored `""` is indexed like any other value.** Unpaid
> orders legitimately have no payment id, so if anything ever wrote `""` instead of
> leaving it unset, the *second* unpaid checkout would fail with E11000. Both fields
> now normalise `""` and `null` to `undefined` in a schema setter, making it
> structurally impossible rather than a convention every caller must remember. Caught
> by the test suite, not in production.

---

## 15. Coupons

`maxLimit` is the **per-user** allowance, enforced as `max(usedBy count, orders placed with it) ≥
maxLimit`. It is cross-checked against real order history as well as the counter, because the counter
is decremented on cancellation while the order history is the durable fact.

It was previously stored, editable in the admin UI, shown on the coupons report — and read by
nothing: the check was hardcoded to 1, so setting a coupon to "usable twice" had no effect.

Read as per-user rather than as a global cap deliberately. The default is 1 and the hardcoded check
it replaced was per-user, so every existing coupon behaves exactly as before while the ones
configured for 2 uses start working; a global cap would instead have retroactively killed every
coupon already used once. **If you meant it as a global cap, this is the line to change.**

`usage` is the total-redemptions counter for reporting. Cancellation decrements both atomically and
cannot drive either below zero.

---

## 16. Known gaps

Still true, and deliberately not addressed:

- **No confirmation prompt on the Razorpay refund button.** Harmless on test keys; on live keys it
  moves real money irreversibly in one click. *(audit M-13)*
- **Replacement dispatch isn't separately tracked** beyond the `replaced` status and `replacedAt` —
  there's no shipment record for the outbound replacement parcel. *(audit-2 H2-06)*
- **No `Closed` terminal state.** `RTO Received → Cancelled` closes an RTO out, which is
  serviceable but overloads `Cancelled` with two different meanings.
- **One return per line, forever.** `{order, product, user}` is uniquely indexed with no status
  filter, so a partial return exhausts the line and a wrong QC rejection is terminal with no admin
  reopen. *(audit-2 H2-02 — a policy decision, not yet made)*
- **Shipping and tax are hardcoded to 0.** The arithmetic is correct today, but the moment either
  goes non-zero a full *return* under-refunds by that amount while a full *cancel* refunds it —
  decide the policy before setting a shipping charge. *(audit-2 H2-03)*
- **`shiprocket.awbCode` is not unique**, and the courier webhook falls back to looking an order up
  by it. *(audit-2 H2-01)*
- **Manual fulfilment records no shipment data.** The courier and AWB fields live under
  `order.shiprocket`, so a seller shipping outside Shiprocket has nowhere to put a tracking number
  and the customer has nothing to track.
- **`InventoryModel` is a second stock ledger no order path writes.** The admin Inventory page will
  drift from real stock after the first sale. `product.stock` is the real number. *(audit M-03)*
- **`PAYMENT_MODE=demo` is dead config** — read into `getFeatures().payments.mode` and consumed by
  nothing. It protects nothing.
- **No DB-level unique index on `refunds[].returnRequest`,** confirmed absent in the live database.
  This is not fixable with an index: a unique index on an array-subdocument field enforces
  uniqueness *across* documents, not within one array, so no index can express "one refund per
  return per order". The guard is the return-status compare-and-swap in §6, which prevents two
  executions reaching the refund at all.
