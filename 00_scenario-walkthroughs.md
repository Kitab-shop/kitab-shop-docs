# How The System Behaves, Scenario By Scenario

A walkthrough of what the code actually does in each real-world combination, written
to be read start to finish rather than looked up. At the end there is a section on
**what you can change and what it costs**, so you can take options to a client.

---

## How to read this

Three switches decide which scenario you are in. Everything else follows from them.

**Payment method** — chosen by the customer at checkout. `RAZORPAY` means the money
is already in your account before the order exists. `COD` means no money has moved and
the order is a promise to pay on delivery. These are not two flavours of the same
flow; they diverge at almost every step.

**Shiprocket** — `SHIPROCKET_ENABLED` in `.env`. When on, a courier feed drives your
order statuses and, for COD, the courier collects the cash. When off, you ship by hand
and an admin drives every status change. It is currently **off**, so manual shipping
is the live behaviour.

**Who acts** — the customer can place an order and cancel their own order. Everything
else is an admin action.

One rule underpins all four scenarios and is worth stating before we start: **money
never moves inside a database transaction.** Stock, wallet credit, coupon release and
the refund *record* all commit together as one unit. The actual call to Razorpay
happens after that commit, and the call to the courier happens after that. So a
gateway outage or a courier outage can never undo a placed order — it can only leave a
recorded, retryable task behind.

---

## Scenario A — Razorpay, Shiprocket off

This is the smoothest path and the one with the fewest moving parts. Use it for a
client demo.

### Placing the order

The customer fills the cart. Every line is checked against the *variant's* own stock,
not the product total, so a sold-out size is refused even when the product looks in
stock. When they proceed, the server prices the order from the catalogue — it ignores
whatever the browser claims about price or discount — and creates a payment intent.

At that moment stock is **decremented and reserved**. This is worth understanding
because it surprises people: the reservation is not a separate counter. The units come
straight out of sellable stock, and a reservation record with an expiry is written so
they can be given back if payment never completes.

The customer pays. Razorpay captures the money and the browser tells your server. The
server then, in one transaction, redeems the coupon, debits the wallet, and creates the
order as `Confirmed` / `Paid`. Because the reservation is still alive, it does **not**
decrement again.

After that transaction commits, three things happen in order: the purchased items are
removed from the cart, Shiprocket sync is skipped (it is off), and notifications fire.

### If the customer closes the tab after paying

Nothing is lost. Razorpay sends a `payment.captured` webhook and it runs the *same
function* the browser would have. The order is created, the cart is cleared. This is
why the browser is not load-bearing — but it does require `RAZORPAY_WEBHOOK_SECRET` to
be set, which it currently is not.

If both the browser and the webhook arrive at once, you get one order, not two.

### If they abandon payment entirely

The stock is sitting in a reservation. A sweeper runs every five minutes, gives the
units back, and cancels the order. In the meantime that unpaid row is hidden from every
revenue figure, count, chart, export and customer-facing list — so it never inflates
your numbers.

### Fulfilment

You ship it yourself. There is a dedicated endpoint for recording the courier name and
tracking number, and the customer then sees both on their order. Recording a manual
shipment also advances the order to `Shipped` through the normal transition rules.

From there an admin moves the order along: `Shipped → Out For Delivery → Delivered`.
The first time it reaches `Delivered`, the delivery timestamp is stamped — and only the
first time, because re-stamping it would silently push the customer's return window
forward.

### Cancellation

Before dispatch, the customer can cancel their own order. The order is claimed
atomically, stock is put back, wallet credit is returned **to the wallet**, the coupon
redemption is freed, and the refund is recorded — all in one transaction. Only after
that commits is Razorpay actually told to refund.

Two things worth knowing. First, this needs **no admin approval** — see the tweak
section. Second, if the same customer double-clicks, only one cancellation wins; the
other gets a clean refusal and stock is not restored twice.

After dispatch, cancellation is refused outright. At that point the route is a return.

### Return

The clock starts at delivery, not at the order date, and runs for the window set on
the product itself. The customer asks; an admin approves, schedules a pickup, and
marks it received. At *received* the admin answers one question that the system
deliberately keeps separate from the refund: **can these goods be sold again?**

If resellable, the units go back into stock. If damaged, they are written off. **Either
way the customer is refunded.** Conflating those two decisions used to force a bad
choice — refund the customer and put a defective unit back on the shelf, or reject a
legitimate return to protect the shelf.

The refund itself covers the goods and the tax, but **not the outbound shipping** — the
parcel shipped and the courier was paid. That is a policy decision, and it is the
opposite of a cancellation, where nothing shipped so the shipping comes back too.

Only one return can be open per order line at a time, enforced by the database rather
than by application code. Once a return reaches a final state, the line is free again,
so a customer who bought three units and returned one can return the other two.

### RTO

There is no courier feed, so nothing arrives automatically. An admin can still walk the
order through `Shipped → RTO → RTO Received` by hand if a parcel comes back, and from
there the disposition and closure work exactly as in Scenario B.

---

## Scenario B — Razorpay, Shiprocket on

Everything about money is identical to Scenario A. What changes is that a courier now
drives your order statuses instead of an admin.

### Fulfilment

Either the order is pushed to Shiprocket automatically on confirmation
(`SHIPROCKET_AUTO_CREATE_ORDER`), or an admin clicks to create the shipment. Shiprocket
returns its own order id and shipment id, which are stored. An admin then assigns an
AWB, and the courier name and tracking number are captured.

One structural fact matters here: the request sent to Shiprocket uses **your order id
as its reference and includes the whole order's items**. The integration is therefore
one shipment per order by construction. There is no way today to split one order across
two parcels.

### The courier feed

From then on, status changes arrive by webhook. Each event is authenticated by a shared
token, and then — this is the important part — the system insists on resolving the event
to **exactly one** of your orders. It checks every identifier the event carries. If two
identifiers point at different orders, or one identifier matches two orders, the event
is **refused and nothing is touched**. It answers success to the courier anyway, because
retrying will not resolve an ambiguity and you do not want Shiprocket redelivering
forever.

A mapped status is only applied if it is a legal move from where the order actually is.
That single rule is what makes stale events harmless: a `Delivered` event arriving for
an order the customer already cancelled cannot resurrect it.

### Cancellation

Same as Scenario A, with one step added at the very end: after the order is cancelled
locally and the refund has been issued, the courier is told to cancel the shipment.

The ordering is deliberate and was fixed after causing a real problem. Previously the
courier was called *before* the local claim, so two concurrent cancellations both killed
the parcel while only one could win — and if the winner then failed, you had an active
order with a dead shipment.

If the courier call fails, **the order stays cancelled**. It never throws. The reason is
recorded, the shipment shows as awaiting cancellation, and there is an admin retry
button. Retrying is safe: if Shiprocket says the shipment was already cancelled, that
is treated as success.

### RTO — the path that only exists here

A failed delivery becomes `NDR`, and a parcel heading back becomes `RTO`. When it
physically arrives, the courier reports it and the order moves to `RTO Received`.

Two consequences fire, and they are deliberately not the same thing.

**Money.** A prepaid customer paid for goods they never received, so a refund is owed.
The system records that as an obligation — it does **not** push it to Razorpay. An
unattended courier feed should not be able to issue irreversible refunds. The liability
shows up in **Admin → Outstanding Refunds**, oldest first, for a human to release.

**Stock.** Nothing is restocked at arrival. The goods have not been inspected, and a
courier can damage a box in either direction. An admin records the disposition, and the
restock or write-off happens then.

Only once the disposition is recorded can the order be closed. Closing early is refused,
because the disposition endpoint accepts only `RTO Received` while `Closed` is final —
so an early close would strand the units forever, neither restocked nor written off.

---

## Scenario C — COD, Shiprocket off

This is where the behaviour differs most, and where the operational burden sits.

### Before an order can even be placed

Three gates, in this order. First, COD must be switched on — and this lives in the
**database**, under Admin → Operations → Checkout, not in `.env`. A fresh install has it
off. Second, the customer must have verified a one-time code sent to their email. Third,
the order total must sit inside any minimum and maximum you have configured.

The OTP is worth understanding properly. A six-digit code is emailed, valid for ten
minutes, with a 45-second cooldown before another can be requested. When the order is
placed, **the verification is consumed** — deleted in the same transaction as the order.
One code buys exactly one order; a second order needs a fresh code.

This also means: **if outbound email is not working, nobody can place a COD order.** The
mail credentials exist but the notifications switch is currently off, so today COD is
effectively unusable until that is turned on.

### Placing the order

No money moves. Stock is decremented inside the order transaction — there is no
reservation step, because there is no payment window to protect. The order is created
`Confirmed` / `Pending`, and `Pending` here is honest: the customer genuinely owes cash.

That is why the unpaid-checkout filter is careful to exclude only *prepaid* unpaid
orders. A pending COD order is real and must appear in your lists and reports; an
abandoned Razorpay checkout is not an order at all.

### Fulfilment, and the moment cash is booked

You ship it and record the courier and tracking number by hand. Then an admin moves the
order to `Delivered`.

**That click is when the money is booked.** Delivery *is* the payment event for COD:
`Pending` becomes `Paid` at that moment, and not before. The consequence is
operational rather than technical — if your team ships things but never marks them
delivered, the cash exists in your hand but is invisible in your reports, and any COD
return will incorrectly believe the customer never paid.

You collect the cash yourself. There is no third party in the loop.

### Cancellation

Before dispatch, the customer can cancel. Stock goes back, wallet credit and coupons are
restored — and **no refund is created at all**, because no money was ever collected. The
order sits `Cancelled` with payment still `Pending` and a completely empty refund
ledger. This is correct, and it is a useful thing to demonstrate to a client: the system
does not manufacture a refund obligation out of nothing.

### Return, and the part clients always ask about

Eligibility works exactly as in Scenario A. The difference is the money.

Because there is no card to reverse, a COD refund **cannot be automated**. The system
requires the customer to say where to send it at the moment they raise the return —
either a UPI ID or full bank details, both validated. Without that the request is
refused, because otherwise you would be left with an obligation and nowhere to send it.

The refund is then recorded as owed, marked as needing manual settlement, and appears in
Outstanding Refunds. Someone transfers the money and records the reference against the
ledger row. It is **never** sent to Razorpay.

### RTO

Manual only, as in Scenario A. Worth noting: an RTO on a COD order means **nothing is
owed to the customer** — they never paid. Your loss is the freight, not the goods.

---

## Scenario D — COD, Shiprocket on

The money rules are identical to Scenario C. Two things change, and both matter
commercially.

### The courier collects your cash

The request sent to Shiprocket marks the shipment as COD, so the courier collects the
money at the door and remits it to you on their own cycle. Your reconciliation is now
against Shiprocket's remittance statements, not against your own cash box. That is a
real accounting change, not just a technical one, and it is worth raising with a client
explicitly.

### Delivery is booked automatically

You no longer depend on an admin clicking `Delivered`. The courier reports delivery, and
the order flips from `Pending` to `Paid` on the strength of that event.

This is guarded in two ways that are easy to miss and important to state. The flip only
happens if the status change was actually **applied** — so a stale `Delivered` event for
an order that was already cancelled cannot book cash for money nobody collected. And the
flip cannot overwrite a refund that has since been issued.

### Cancellation and return

Cancellation behaves as in Scenario C, plus the courier cancellation step from
Scenario B. Return behaves exactly as in Scenario C — the refund is still manual and
still needs a destination, because the customer paid the courier in cash.

### RTO

Now automatic. The courier reports the parcel coming back and arriving. For a COD order,
**no refund obligation is created**, because nothing was collected. The stock still waits
for a disposition, and the order still cannot be closed until that is recorded.

---

## What changes between the four, in one paragraph each

**Money timing.** Razorpay: captured before the order exists. COD: never, until
delivery. This is why a Razorpay cancellation refunds and a COD cancellation does not.

**Who moves the order along.** Shiprocket off: an admin, click by click. Shiprocket on:
the courier feed, automatically. With COD this decides whether cash is booked by a human
remembering to click, or by a webhook.

**Who holds the cash.** Razorpay: you do, immediately. COD without Shiprocket: you do,
after collecting it yourself. COD with Shiprocket: the courier does, until they remit.

**What a return costs you.** Razorpay: an automatic gateway refund. COD: a manual bank
transfer to a destination the customer supplied, tracked by hand.

---

## Where you can tweak, and what it costs

These are the genuine decision points. Each says what happens now, where the control
lives, and what changes if you move it.

### 1. Should a customer's own cancellation refund automatically?

**Now:** yes. A customer cancelling their own unshipped prepaid order triggers a real
Razorpay refund with no admin involvement. It is bounded — nothing has shipped, the
amount is computed server-side, and the refund ceiling is enforced by the database — but
there is no human in the loop.

**To change:** record the refund as *owed* instead of settling it immediately. The order
still cancels and restocks instantly, so the customer experience is unchanged; the money
then waits in Outstanding Refunds for someone to release. Small change, no architectural
impact, and the ledger already supports it.

**Trade-off:** slower refunds, more support contacts, but every rupee leaving is a
deliberate act. Most clients with a fraud concern want this; most clients optimising for
customer experience do not.

### 2. How long can a customer self-cancel?

**Now:** no limit before dispatch. The `cancellationWindowHours` setting exists and is
enforced, but it is set to **0**, which means unrestricted.

**To change:** set it under Admin → Operations → Checkout. Worth confirming that 0 is a
decision rather than an oversight.

### 3. Is shipping refunded on a return?

**Now:** no on a return, yes on a pre-dispatch cancellation. The reasoning is that once
the parcel has shipped the courier has been paid, whereas a cancellation ships nothing.
Tax follows the goods in both cases.

**Note:** shipping and tax are both hardcoded to zero today, so this policy is written
and tested but has never actually run against real values. If the client wants to start
charging shipping, this is the first thing to re-verify.

### 4. Manual shipping or Shiprocket?

**Now:** manual. Fully built — record carrier and tracking, and the customer sees both.

**To change:** configure Shiprocket credentials and enable it. You gain automatic status
updates, automatic COD collection, and RTO detection. You take on courier remittance
reconciliation, and you inherit a constraint: one shipment per order, no splitting.

### 5. Does COD need the OTP?

**Now:** yes, unconditionally, and the code is single-use.

**To change:** it is a hard gate in the order path, so removing it is a code change
rather than a setting. Before considering it, note that the OTP is currently the only
thing verifying that a COD customer's contact details are real.

### 6. COD order-value limits

**Now:** both minimum and maximum are configurable and enforced, and both are currently
0 (meaning no limit).

**To change:** Admin → Operations → Checkout. This is the usual lever for COD fraud and
return-abuse risk on high-value items.

### 7. Where COD can be delivered

Admin → Operations → Checkout asks *"Where can customers pay by Cash on Delivery?"* and
offers two modes. The choice is enforced server-side at order creation, so it holds even
against a tampered client.

**Deliver COD everywhere** (the default) — any valid Indian PIN code can pay by COD.
Shiprocket is never contacted at checkout, so COD keeps working whether or not Shiprocket
is set up. This is the right mode for most stores.

**Follow Shiprocket serviceability** — before the order is created, Shiprocket is asked
whether any courier will carry COD to that PIN code. If no courier will, the customer is
told COD isn't available for the address and is pointed at online payment.

The part to be deliberate about: this mode **fails closed**. If Shiprocket can't answer —
credentials missing, API down, unexpected response — the COD order is *refused* rather
than let through. That is intentional; a restriction that quietly stops applying during an
outage isn't a restriction. But it means this mode makes Shiprocket credentials
load-bearing for COD. Pick it only once Shiprocket is confirmed working.

If credentials are missing while this mode is on, the Checkout tab shows a red banner
saying COD orders are being refused, with a one-click switch back to COD-everywhere, and
System Health reports it as critical. Nothing about it is silent.

Both modes sit *underneath* the COD amount limits — a pincode being serviceable doesn't
exempt an order from the min/max, and the OTP step applies either way.

---

## Things to be straight with a client about

**Two integrations have never run for real.** Razorpay's checkout has valid test-mode
keys and the server-side path is complete, but completing a payment needs a browser, so
it has not been exercised end to end. Shiprocket has no credentials at all. Everything
internal — variant stock, cart, COD ordering, cancellation, the courier webhook's
authentication and identity handling, and both concurrency races — has been proven
against a live server and database.

**Email delivery has never been switched on.** The OTP is generated correctly and is
deliberately not exposed in the API response. Nothing has been sent, because the
notifications flag is off.

**Shipping and tax are zero everywhere.** All the apportioning logic is in place and
tested with synthetic values; it has never seen a real non-zero charge.

**A cosmetic imperfection, so it does not surprise you later.** If several identical
delivery events from the courier land at exactly the same moment, the order history can
show the same status change more than once. The money is unaffected — the delivery
timestamp is written once and the COD payment flips once, each independently guarded.
It is noise in an audit trail, not a financial fault, and it is a small fix if you want
it gone.

**One shipment per order.** Splitting an order across two parcels is not supported, and
a client expecting it should be told before they ask. The half-built groundwork — a
`CreateSplitShipment` endpoint writing an `order.shipments[]` array that nothing ever
read — was removed on 16 Sep 2026 rather than left to look like a working feature.
One parcel per order is a property of the Shiprocket integration too: it posts
`order_id: String(order._id)` with the whole order's items, and `shiprocket.orderId` /
`shiprocket.awbCode` are unique scalars. Real split shipments need a shipment entity
with its own identifiers.
