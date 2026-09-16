# Shiprocket — End-to-End Test Plan

How to prove the whole shipping integration works, in the order that lets each failure
be diagnosed on its own.

Companion to [002_shiprocket-integration.md](002_shiprocket-integration.md), which
describes what the integration *does*. This describes how to find out whether it does.

**Read this first:** the phases are ordered so that nothing downstream can fail for an
upstream reason. Phase 3 will look broken if Phase 1 was skipped, and you will spend an
afternoon debugging a shipment when the real problem was a password. Do them in order.

---

## Current state (verified 2026-09-02)

| Layer | State |
|---|---|
| `SHIPROCKET_ENABLED` (`.env` and `.env.production`) | `false` |
| Shiprocket email / password (DB) | empty |
| Pickup postcode, webhook token (DB) | empty |
| `shipmentsEnabled` / `autoPushEnabled` / `deliveryWebhookEnabled` | all `false` |
| `reverseShipmentsEnabled` | `false` |
| Orders ever pushed to Shiprocket | 0 of 7 |
| Products with weight + all dimensions | 51 of 51 |

So this is a first-time bring-up, not a regression check on a live integration. Phase 0
is the only phase you can run today.

---

## Prerequisites

1. **A Shiprocket account** with at least one **pickup address registered** in
   Settings → Pickup Addresses. Note the nickname exactly — case, spaces and all.
2. **A Shiprocket API user.** Dashboard → Settings → API → Configure. This is a
   *separate* credential from your dashboard login. Using the dashboard password is the
   single most common bring-up failure and it surfaces as a generic auth error.
3. **A publicly reachable backend URL** (Railway). Needed from Phase 4 onward — Shiprocket
   must be able to POST to you. `localhost` cannot receive webhooks; use a tunnel
   (`ngrok http 3000`) if you are testing before deploy.
4. **An admin login** with the `orders:manage` permission (and `returns:manage` for
   Phase 8).
5. **Wallet balance** in Shiprocket. AWB assignment fails on an empty wallet, with an
   error that does not obviously say so.

---

## Phase 0 — Everything that needs no Shiprocket account

Runs today, against the current codebase. If any of this fails, stop; nothing later
will be meaningful.

### 0.1 Pure logic suites

```bash
cd kitab-shop-be
npm run test:shipment-status-mapping   # 38 assertions — courier status → order status
npm run test:order-package             # 28 assertions — parcel weight and dimensions
```

No database, no network. Both must be `0 failed`.

These two exist because of bugs found on 2026-09-02: a failed delivery (code 21,
`"UNDELIVERED"`) never reached the `NDR` status, and RTO arrival was keyed to code 43
(`SELF FULFILLED`) instead of 10 (`RTO DELIVERED`). If either suite ever fails, the
NDR buttons and the RTO refund liability are the things that break.

### 0.2 Full regression suite

```bash
npm test
```

⚠️ **This writes to the database in `.env` — currently the live Atlas cluster.** Fixtures
are namespaced per process and removed in a `finally` block, but run it against a copy
first if that cluster holds anything you care about.

### 0.3 Manual fulfilment still works with Shiprocket off

Prove the store ships without Shiprocket before adding Shiprocket. With all switches off:

- [ ] Place an order (COD and Razorpay).
- [ ] Admin → Order Detail: no Shiprocket buttons are shown, and the notice explains why.
- [ ] `PUT /api/v1/order/:orderId/shipment/manual` — record a carrier and tracking
      number by hand. Sending the same values twice is a no-op, not a second shipment.
- [ ] Walk the status by hand: Pending → Confirmed → Packed → Shipped → Out For Delivery
      → Delivered.
- [ ] `deliveredAt` is stamped, and the return window counts down from it.
- [ ] Cancel an order — it must succeed with no Shiprocket call attempted.

### 0.4 Package measurement in the product form

- [ ] Admin → Products → edit a book. The Shipping Package block shows **Volumetric
      weight** and **Billed at (per unit)**.
- [ ] Clear the weight field → it turns amber and warns that the store default is
      substituted *per unit*.
- [ ] Enter `30 × 25 × 15` cm with weight `0.5` → billed weight becomes **2.25 kg**,
      labelled *"charged by size, not weight"*. This is the volumetric rule that makes
      an oversized box expensive; if it doesn't appear, the form is not wired up.

---

## Phase 1 — Credentials and connection

Nothing here creates a shipment. This phase exists so a wrong password is discovered as
a wrong password.

1. Set `SHIPROCKET_ENABLED=true` in `.env` and **restart the server**. This is a deploy
   switch, not an admin toggle — the panel cannot change it.
2. Admin → Operations → Shipping: enter the **API user** email and password, the pickup
   location nickname, and the pickup postcode.
3. Press **Test connection**.

| Result | Meaning |
|---|---|
| Success | The API user is valid. Continue. |
| Auth error | Wrong credential, or you used the dashboard login rather than the API user. |
| Unreachable | Network or base URL. Not a credentials problem. |

- [ ] **Test connection** succeeds.
- [ ] **Load from Shiprocket** on the pickup location returns your real locations, and
      the one you typed is among them.
- [ ] System Health does **not** report `critical` for the pickup location.
- [ ] Negative test: change one character of the password, save, press Test connection.
      It must fail *at the button*, not silently. Restore the password afterwards.

> The pickup location name must match Shiprocket's nickname **exactly**. A mismatch is
> rejected at order creation with a Shiprocket-side error this app cannot pre-validate.
> Use the dropdown rather than typing it.

---

## Phase 2 — Serviceability and rates (read-only)

Still no shipment. These are GET-shaped questions to Shiprocket.

```
GET /api/v1/order/shipping/serviceability?deliveryPostcode=<pin>
```

- [ ] A metro pincode returns couriers.
- [ ] A remote pincode (try a J&K or North-East PIN) returns a different, smaller list —
      this is Zone E, and it is where freight is highest.
- [ ] A nonsense pincode (`000000`) returns *not serviceable* rather than an error page.
- [ ] Admin → Order Detail → **Show couriers & rates** lists couriers cheapest-first with
      ETDs and COD support.

**Rates must match the parcel.** The courier list and the shipment now compute the
package the same way. Cross-check once: note the weight in the rate call, then compare it
with the weight on the created shipment in Phase 3. They must be identical.

**Do not switch COD coverage to "Follow Shiprocket serviceability" yet.** That mode
fails closed — if credentials or the API are unavailable it refuses *every* COD order.
It belongs after Phase 4.

---

## Phase 3 — First shipment, end to end

Use a real order to a real address you control. Book a pickup you can cancel, or accept
that a courier will arrive.

### 3.1 Create

Auto-push (`autoPushEnabled` + `SHIPROCKET_AUTO_CREATE_ORDER=true`) or manually:

```
POST /api/v1/order/:orderId/shipment
```

- [ ] The order appears in the Shiprocket dashboard.
- [ ] `shiprocket.orderId` and `shiprocket.shipmentId` are both saved locally. A shipment
      with only one of them cannot be cancelled, labelled or tracked.
- [ ] `syncStatus` is `created`, with **no AWB** — this is expected, not a failure.
- [ ] Declared weight and dimensions match what the products say. For a multi-book order,
      weights add, heights stack, and the widest footprint is kept.

### 3.2 The three manual steps

Each is a separate admin click. **None of them happens automatically**, even with
auto-push on.

```
POST /api/v1/order/:orderId/shipment/awb       { courierId }
POST /api/v1/order/:orderId/shipment/pickup
POST /api/v1/order/:orderId/shipment/label
POST /api/v1/order/:orderId/shipment/manifest
```

- [ ] AWB assigned; `awbCode` and `courierName` saved.
- [ ] Pickup scheduled; a date comes back.
- [ ] Label PDF opens and is printable.
- [ ] Manifest PDF generates.
- [ ] **Reassign Courier** issues a *new* AWB, and refuses rather than claiming success
      if Shiprocket returns no new AWB.

### 3.3 Invoices

- [ ] The **customer** invoice is generated client-side and works with Shiprocket off.
- [ ] The Shiprocket invoice (`POST .../shipment/invoice`) is a separate document. Don't
      confuse the two — one is your accounting record, the other is the courier's.

---

## Phase 4 — The webhook

This is what makes status flow back on its own. Everything after this phase depends on it.

### 4.1 Register it

1. Set a **Webhook token** in Operations → Shipping.
2. `SHIPROCKET_WEBHOOK_ENABLED=true` in `.env`, restart.
3. Register in Shiprocket → Settings → API → Webhooks:
   ```
   https://<your-backend>/api/v1/order/shipping/webhook
   ```
   with the same token. It is sent as the **`x-api-key`** header.

### 4.2 Simulate it before trusting it

You do not need a real parcel to test the mapping. Replace `<TOKEN>`, `<SR_ORDER_ID>`:

```bash
curl -X POST https://<your-backend>/api/v1/order/shipping/webhook \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: <TOKEN>' \
  -d '{"sr_order_id": <SR_ORDER_ID>, "awb": "<AWB>",
       "shipment_status_id": 6, "shipment_status": "SHIPPED",
       "courier_name": "Delhivery"}'
```

Walk an order through by changing `shipment_status_id`:

| Send | Expect |
|---|---|
| `6`, `18`, `42` | `Shipped` |
| `17` | `Out For Delivery` |
| `7` | `Delivered`, `deliveredAt` stamped, **COD flips to Paid** |
| `21` | `NDR` — the fix; this is what unlocks the NDR buttons |
| `9` | `RTO` |
| `10` | `RTO Received` + a refund obligation on prepaid orders |
| `8`, `12`, `45` | **no status change**, but a `shiprocket_shipment_exception` log line |

### 4.3 The guards

These matter more than the happy path. Each is a bug that was fixed and must stay fixed.

- [ ] **Wrong token** → `401`. An absent token → `401`.
- [ ] **Webhook disabled** (admin switch off) → `200` with "disabled", not an error.
      A `4xx` would make Shiprocket retry forever.
- [ ] **Replay `7` twice** → `deliveredAt` does not move. First Delivered wins; re-stamping
      re-opens an expired return window.
- [ ] **Out-of-order event**: send `7` (Delivered), then `6` (Shipped). The order stays
      `Delivered`. A backwards walk would lock the customer out of returns.
- [ ] **Cancelled order**: cancel one, then send `7`. It must not resurrect — that would
      book COD revenue for cash nobody collected, on already-restocked stock.
- [ ] **Unknown order**: send a `sr_order_id` that doesn't exist → `200`, ignored, logged.
- [ ] **Manual shipment**: record a manual shipment, then send an event that resolves to
      it. The carrier name must **not** be overwritten with a Shiprocket claim.

Only once all of 4.3 passes should you switch COD coverage to **"Follow Shiprocket
serviceability"** in Operations → Checkout.

---

## Phase 5 — NDR (failed delivery)

This path was **unreachable before 2026-09-02** and has never run against a live account.
Test it deliberately rather than waiting for a real failed delivery.

- [ ] Send `shipment_status_id: 21` → order reaches `NDR`, `shiprocket.ndrReason` recorded
      if the payload carried one.
- [ ] The **Re-attempt** and **Send it back** buttons appear on Order Detail.
- [ ] `POST /api/v1/order/:orderId/shipment/ndr { "action": "reattempt" }`
      → order moves to `Out For Delivery`.
- [ ] `{ "action": "return" }` → order moves to `RTO`.
- [ ] An order **not** at `NDR` refuses the action with a message naming its actual status.

> ⚠️ **The one thing to watch.** `actOnNdr()` posts to `/ndr/{awb}/action` with
> `{action, comments}`. That shape is unverified against a live account. If Shiprocket
> rejects it, the action is **refused** and the order stays at `NDR` — safe, but the
> feature won't work until the shape is corrected. It is one line in
> `shiprocket.service.js`. **The first real NDR is when you find out.**

---

## Phase 6 — RTO (parcel comes back)

- [ ] `9` → `RTO`. `46` → still `RTO`.
- [ ] `10` → `RTO Received`.
- [ ] **Prepaid** order: a refund obligation is recorded as `owed`, and **not** pushed to
      the gateway. An unattended courier feed must never issue an irreversible refund.
- [ ] **COD** order: **no** refund obligation — the customer never paid.
- [ ] Stock is **not** restocked automatically. A parcel arriving is not the same event as
      someone opening and inspecting it.
- [ ] `PATCH /api/v1/order/:orderId/rto-disposition` records condition; restock happens
      here, and the order closes as `Closed` (not `Cancelled`).
- [ ] Replay `10` → exactly **one** refund obligation, not two.
- [ ] Negative test: send `43` with **no** status text. The order must **not** become
      `RTO Received` — 43 is `SELF FULFILLED`, and treating it as an RTO would book a
      refund against a delivered, paid order.

---

## Phase 7 — Cancellation

| Stage | Expected |
|---|---|
| Before AWB | Cancels locally and at Shiprocket |
| After AWB, before pickup | Cancels; Shiprocket cancellation must succeed first |
| After pickup | Not a normal cancellation — this becomes an RTO |

- [ ] Cancel a shipped-to-Shiprocket order → the Shiprocket shipment is cancelled too.
- [ ] **If Shiprocket's cancellation fails, the local cancellation is aborted as well.**
      Verify this: you must never have a cancelled order with a live parcel moving toward
      a customer.
- [ ] `POST /api/v1/order/:orderId/shipment/cancel` retries a shipment left awaiting
      cancellation (Shiprocket timed out, or the process died after the local commit).
- [ ] A cancelled order is terminal — it cannot be walked back to Confirmed or Delivered.

---

## Phase 8 — Returns, reverse pickup, replacement

**`reverseShipmentsEnabled` defaults to OFF** and is the only capability that does. It
books real courier collections at customer addresses. Turn it on deliberately.

- [ ] Customer requests a return on a Delivered order (`POST /api/v1/returns`) inside the
      product's return window.
- [ ] Admin approves (`PATCH /api/v1/admin/returns/:id/status`).
- [ ] `POST /api/v1/admin/returns/:id/pickup/book` — a reverse AWB is created, with the
      **customer's address as pickup** and the warehouse as destination.
- [ ] A webhook carrying the reverse AWB is attributed to the **return**, not the order,
      and the response says `scope: "return"` with `statusUnchanged: true`.
- [ ] **A courier event never advances the return's status.** `received` stays an operator
      decision, because it is what gates refunds and restocking.
- [ ] Mark received → QC → refund **or** replacement.
- [ ] `POST /api/v1/admin/returns/:id/replacement/book` then `/dispatch` — dispatch
      pre-fills courier and AWB from the booking.
- [ ] With `reverseShipmentsEnabled` off, booking is refused with a message naming the
      setting, and collection is arranged outside the platform as before.

---

## Phase 9 — Money

- [ ] **COD delivered** → `paymentStatus` becomes `Paid` only when the order actually
      transitioned to `Delivered`. A refused transition must not book cash.
- [ ] Operations → Money → **COD Reconciliation** matches delivered COD orders against
      what Shiprocket says it collected.
- [ ] Shipping charge: with "Charge live courier rates" **off**, shipping is free. With it
      on, a rate is charged — and if the rate call fails, the order **ships free** rather
      than being blocked. Verify the fail-soft: a courier blip must not cost a sale.
- [ ] Compare a real Shiprocket passbook entry against the declared weight. A mismatch
      means the product measurements are wrong, and it will recur on every order.

---

## Phase 10 — Weight accuracy (ongoing, not one-off)

The expensive failure mode is quiet. Check it monthly.

- [ ] No product has a blank weight or dimension. The form warns, but does not block.
- [ ] Pick three recent shipments. Compare declared weight against the passbook's applied
      weight. Any excess-weight charge means a measurement is wrong.
- [ ] Photograph packed parcels on a scale at pack time. Weight disputes have a short
      filing window and are unwinnable without evidence.
- [ ] Books are dense — a missing measurement is a large relative error, not a rounding
      one. A 3-copy order of an unmeasured book declares 3 × the store default.

---

## Kill switches

If anything goes wrong in production, in order of bluntness:

| Action | Effect |
|---|---|
| Operations → Shipping → **Manual fulfilment** | Stops all Shiprocket use; keeps orders working |
| Turn off **Accept status updates** | Status stops flowing; you mark Delivered by hand |
| Operations → Checkout → COD coverage off "Follow serviceability" | COD stops depending on the Shiprocket API |
| `SHIPROCKET_ENABLED=false` + restart | Environment-level block. Shipment actions answer `503` |

An admin switch answers `409` naming the setting to change; an env block answers `503`.
Different problem, different owner — the status code tells you which.

---

## Known gaps — do not raise these as bugs

- **AWB, pickup and label are manual per order.** The service functions exist; they are
  not wired to auto-run. This is a decision, not a defect.
- **`actOnNdr` payload shape unverified** (Phase 5).
- **Live courier rates are fetched and discarded** unless rate charging is on —
  `shippingCharge` is computed internally.
- **Serviceability is not cached**, deliberately. A COD restriction must not answer from
  a stale snapshot of courier availability.
- **The checkout COD serviceability check sends the default package size**, not real cart
  contents. Fine for a yes/no answer; not exact for heavy or oversized orders.
- **Status codes `8`, `12`, `23`, `24`, `25`, `39`, `45`, `76` change no status.** They are
  logged instead. `Cancelled` carries restock and refund compensation that an unattended
  courier feed must not perform.

---

## Sign-off

| Phase | Owner | Date | Result |
|---|---|---|---|
| 0 — Offline suites and manual fulfilment | | | |
| 1 — Credentials | | | |
| 2 — Serviceability | | | |
| 3 — First shipment | | | |
| 4 — Webhook + guards | | | |
| 5 — NDR | | | |
| 6 — RTO | | | |
| 7 — Cancellation | | | |
| 8 — Returns | | | |
| 9 — Money | | | |
| 10 — Weight accuracy | | | |

Phases 4.3, 5, 6 and 7 are the ones that cost money when they are wrong. If time is
short, those are the ones to do properly.
