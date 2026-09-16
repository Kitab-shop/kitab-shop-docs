# Shiprocket Integration

What's already built, what it needs to go live, and known gaps to watch for.

## How automated is it actually? (read this first)

A common wrong assumption: *"once credentials are in and Shiprocket is enabled, Shiprocket handles everything and we just pack the box."* Not quite — **the middle three steps are still manual clicks per order.**

Verified end-to-end flow with everything switched on:

```
Customer places order (COD or Razorpay — both paths auto-sync)
   ↓  [AUTOMATIC]  pushed into Shiprocket, appears in their dashboard
Admin: Assign AWB        ← manual click (picks courier, gets tracking number)
Admin: Schedule Pickup   ← manual click (books courier collection)
Admin: Generate Label    ← manual click (the sticker you print)
   ↓
You print the label and pack the box
   ↓
Courier collects
   ↓  [AUTOMATIC]  webhook pushes status back: Shipped → Out for Delivery → Delivered
                   (also NDR / RTO), updating the order with no admin action
```

Why: auto-create only calls Shiprocket's `/orders/create/adhoc`, which creates the *order* and leaves it at `syncStatus: "created"` with no AWB assigned. `assignAwb`, `schedulePickup`, and `generateLabel` are never invoked automatically anywhere in the codebase — only from the admin Order Detail buttons.

### Admin utilities

Three things that used to require a developer:

| In the panel | Replaces |
|---|---|
| **Test connection** (Operations → Shipping) | Discovering a wrong password later, as a failed shipment. Performs a real Shiprocket login and reports what Shiprocket said. A button, not a poll — each click is a genuine auth call. |
| **Load from Shiprocket** on the pickup location | Typing a name that must match their dashboard EXACTLY. Becomes a dropdown of the real locations; System Health reports `critical` and lists the real names if the configured one isn't registered. Cached 5 minutes, bypassed by the button, dropped when credentials change. |
| **Show couriers & rates** on an order | Guessing a `courierId`. Lists the couriers that will actually carry THIS parcel with real rates, ETDs and COD support, cheapest first, with the current one marked. |
| **Generate Manifest** on an order | Couriers refusing handover for a missing manifest. |
| **Re-attempt / Send it back** on an NDR order | Resolving failed deliveries in Shiprocket's dashboard. |
| **Reassign Courier** on an order | Nothing — this is new. Issues a new AWB with a different courier; refuses if Shiprocket returns no new AWB, so it never claims a reassign that didn't happen. |
| **Book Courier Pickup** on an approved return | Arranging collection outside the platform and typing the details in |
| **Book Replacement Parcel** on a received return | Retyping the courier and AWB at dispatch — Dispatch now pre-fills them from the booking |
| **COD Reconciliation** (Operations → Money) | Cross-checking COD cash order by order |

### How much of Shiprocket do you want to use?

Operations → Shipping asks this directly and offers three levels. Each is a preset over
three individual capability switches, which stay visible underneath — so a level never
hides what it changed, and an unusual combination is still expressible.

| Level | What happens |
|---|---|
| **Manual fulfilment** | Nothing is sent to Shiprocket. You record carrier + tracking by hand on each order and move it Shipped → Delivered yourself. |
| **Shiprocket basics** | You click Create Shipment on the orders you choose, then assign courier, book pickup, print the label. Status is still updated by you. |
| **Shiprocket end to end** | Orders auto-push on confirmation, and the webhook advances them to Shipped / Out for Delivery / Delivered / RTO on its own — which for COD is also what records the cash as collected. |

**These narrow `.env`, they don't replace it.** Effective behaviour is `env AND admin
choice`, so the deployment keeps a working kill switch and the panel can never enable
something the environment forbids:

| Switch | Env ceiling | If it's off |
|---|---|---|
| Use Shiprocket for shipments | `SHIPROCKET_ENABLED` | Manual fulfilment. Shipment actions answer `409` with a message naming the setting to change (an env-level block answers `503` instead — different problem, different owner). |
| Auto-push new orders | `SHIPROCKET_AUTO_CREATE_ORDER` | Orders don't auto-push — you click "Create Shipment" per order |
| Accept Shiprocket status updates | `SHIPROCKET_WEBHOOK_ENABLED` | Status doesn't flow back — you mark Delivered by hand, and COD cash collection is never recorded automatically |

The last two are nested under the first: turning off shipments turns both off, because
pushing or receiving status for shipments you don't create through Shiprocket is
meaningless.

A switch you've chosen but whose env ceiling forbids it is shown as **"on, but not
active"** with the `.env` variable named, rather than being silently ineffective. The
**Integration status** panel below still shows the raw `.env` state.

COD delivery coverage is deliberately *not* here — it's a checkout setting, under
Operations → Checkout.

### Could the middle steps be automated too?

Yes — Shiprocket's API supports auto-assigning AWB with a default courier and auto-scheduling pickup, and the service functions (`assignAwb`, `schedulePickup`) already exist in `shiprocket.service.js`. It would need wiring into `order-shipping.service.js` behind an admin toggle. Not built today.

## Services this integration provides

### Storefront-facing
- **Courier serviceability check by pincode** — tells checkout whether any courier delivers to a pincode, and specifically whether COD is accepted there. This is what the *"Follow Shiprocket serviceability"* COD coverage mode calls (Operations → Checkout). Order creation refuses COD when that answer is negative *or* unobtainable, so this API becomes load-bearing for COD once the mode is selected.
- **Live shipment tracking** — customers can see real-time tracking status for their order once it ships.

### Automatic, once turned on
- **Auto-create order in Shiprocket on placement** — pushed the moment the order is placed. Works for **both** COD (`order.controller.js` → `PlaceOrder`) and Razorpay (`payment-order.service.js`) paths. Requires all three of: integration enabled, `SHIPROCKET_AUTO_CREATE_ORDER=true`, and credentials saved. Creates the order only — no AWB, pickup, or label (see the automation section above).
- **Auto-cancel sync** — cancelling an order on your side also cancels the matching Shiprocket shipment. If the Shiprocket cancellation fails, the local cancellation is aborted too rather than leaving a live shipment moving toward pickup for an order you no longer intend to fulfil.
- **Live status updates via webhook** — Shiprocket pushes delivery events back automatically, updating order status: Shipped → Out for Delivery → Delivered, or NDR (failed delivery attempt) / RTO (returned to sender). Controlled by `SHIPROCKET_WEBHOOK_ENABLED` (separate flag, default off). Reaching Delivered also stamps `deliveredAt`, which is what the per-product return/replacement windows count down from.

### Manual, from the admin Order Detail page
These buttons are hidden, with an explanatory notice, whenever the integration is off or credentials are missing — so they never appear as actions that can only fail.

- Create/push a shipment to Shiprocket for a specific order (only needed if auto-create is off).
- **Assign an AWB** (courier tracking number) and pick a courier — *required per order even with auto-create on.*
- **Schedule a pickup** from your warehouse — *required per order even with auto-create on.*
- **Generate the printable shipping label** — *required per order even with auto-create on.*
- Generate the Shiprocket invoice document. (Note: the customer-facing invoice is generated client-side and does **not** depend on Shiprocket.)

### Admin configuration (Operations → Shipping tab)
- Shiprocket account email/password.
- Pickup location name and pickup pincode.
- Default package weight/dimensions — fallback used when a specific product hasn't had its own weight/size entered (products support their own weight/length/breadth/height in the product form, which take priority).
- Webhook security token.

## Running without Shiprocket at all (manual fulfilment)

Fully supported — Shiprocket is optional, not a dependency. Verified: `UpdateOrderStatus` has zero Shiprocket involvement.

- Drive the order status by hand from the admin Orders page or Order Detail: Pending → Confirmed → Packed → Shipped → Out For Delivery → Delivered.
- Reaching **Delivered** still stamps `deliveredAt`, so return/replacement windows work normally.
- Cancellations work (Shiprocket is only contacted if a shipment actually exists for that order).
- The whole returns flow is manual anyway: approve → schedule pickup → mark received → refund or replacement.
- Customer invoices are generated client-side as PDFs, independent of Shiprocket.

## Go-live checklist

1. Create/use a Shiprocket account (email + password) and register at least one pickup address in the Shiprocket dashboard.
2. Enter that email/password, pickup location name, and pickup pincode in **Operations → Shipping**.
   - ⚠️ The pickup location **name must exactly match** the nickname you gave that address in the Shiprocket dashboard (Settings → Pickup Addresses). A mismatch fails order creation with a Shiprocket-side error, not something this app validates up front.
3. Set `SHIPROCKET_ENABLED=true` in `.env` — the master switch for everything above. *(Already set.)*
4. Decide + set `SHIPROCKET_AUTO_CREATE_ORDER` — `true` to auto-push every order, `false` to keep pushing manual via the admin Order Detail page. Either way you still assign AWB / schedule pickup / generate the label per order.
5. If you want automatic status updates (Shipped/Delivered/NDR/RTO), set `SHIPROCKET_WEBHOOK_ENABLED=true`, set a **Webhook token** in Operations → Shipping, and register the webhook URL + that token in the Shiprocket dashboard (Settings → API → Webhooks). The exact URL is shown with a copy button on the Operations → Shipping tab, and is always:
   ```
   <your-backend-url>/api/v1/order/shipping/webhook
   ```
6. Once confirmed working, switch COD coverage to **"Follow Shiprocket serviceability"** in Operations → Checkout. Do this *after* step 5, not before: the mode fails closed, so selecting it while credentials are missing refuses every COD order (the tab warns you and offers a one-click switch back).

## Compatibility check — code readiness

The integration code itself is solid: proper Shiprocket auth-token caching with automatic re-login on expiry/401, request timeouts, and handling of Shiprocket's quirk of returning HTTP 200 with an embedded error status in the JSON body. Nothing here needs to be fixed before going live. The gaps are all **configuration/operational**, not bugs:

- **NDR actions use Shiprocket's `POST /ndr/{awb}/action` shape** — verify this against your own account on the first real failed delivery. If the contract differs, the action is refused and the order stays at NDR rather than claiming a re-attempt that never happened, so the failure is safe but the feature won't work until the shape is corrected. One line, in `actOnNdr()`.
- **A capability needs both its env ceiling and its admin switch** — turning on `SHIPROCKET_ENABLED` alone unlocks the manual buttons, not auto-push or status updates. The panel labels a chosen-but-blocked switch "on, but not active" and names the `.env` variable, so this is visible rather than a guessing game — but it is still two places, not one.
- **Reverse logistics is opt-in and off by default** — "Book return pickups & replacement parcels" (Operations → Shipping) is the only capability that defaults OFF, because it books real courier collections at customer addresses. Until it is switched on, collection is arranged outside the platform and the courier/AWB are typed in by hand, exactly as before.
- **A courier event never advances a return's status** — the webhook resolves reverse AWBs to the right leg and records where each parcel is, but `received` stays an operator decision. A parcel reaching the warehouse is not the same event as someone having opened and inspected it, and `received` is what gates refunds and restocking.
- **Live shipping rates are opt-in and off by default** — shipping is free on every order until "Charge live courier rates" is switched on under Operations → Checkout. Unlike the COD pincode check it **fails soft**: if the rate cannot be fetched, that order ships free rather than being blocked, because a courier API blip must not cost a sale.
- **Serviceability is not cached** — a 60s memo was tried, to spare the second call a COD checkout makes once rates are on. It was removed: the COD pincode gate reads the same endpoint, and caching made a business restriction answer from a stale snapshot of courier availability.
- **Live courier rates are fetched and discarded** — `/courier/serviceability/` returns rates; `shippingCharge` is computed internally instead, so customers never see courier pricing.
- **The checkout-time COD serviceability check doesn't send real cart weight/dimensions** — it always checks against your configured default package size, not the actual cart contents. Fine for a yes/no "is this pincode serviceable" answer; not exact for edge-case heavy/oversized orders.
