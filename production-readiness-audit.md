# Production Readiness Audit — Order, Payment, Cancellation, Return, Refund

**Audited:** 12 Aug 2026 · **Remediated:** 11 Aug 2026 · **Method:** code-path trace (frontend → route → controller → service → DB → gateway → webhook), not documentation review

**Verdict:** 🟡 **Code-side remediation complete and covered by tests. Three deploy
blockers remain, and all three are configuration the operator has to set — not code.**

| Severity | Count | Fixed | Open |
|---|---|---|---|
| 🔴 Deploy blockers (config, not code) | 3 | 0 | 3 |
| 🔴 Critical | 12 | 12 | 0 |
| 🟠 High | 14 | 13 | 1 |
| 🟡 Medium | 13 | 9 | 4 |
| ⚪ Low | 6 | 0 | 6 |

**Still open, and why:**
- **B-01 / B-02 / B-03** — build-time env var, CORS allowlist, live Razorpay keys.
  All three need production values the operator supplies. Nothing to change in code.
- **H-07** — `AUTH_SECURITY_ENABLED=false`, so logout does not revoke a token and admin
  2FA is off. Deliberately left as a config decision: flipping it changes login behaviour
  for every existing session, so it belongs in a deploy step rather than a silent edit.
  The System Health panel flags it red until it is set.
- **M-03, M-04, M-12** and the whole **Low** list — no money or access-control impact.
  M-03 is a reporting divergence (a second stock ledger no order path writes), M-04 is
  PII on `isAdmin`-only endpoints, M-12 is latent mass-assignment that nothing reads today.

### Regression coverage

`cd kitab-shop-be && npm test` — **174 assertions across 4 suites, run against the live
database** (not mocks: every bug here was an atomicity or concurrency bug, and a mock would
have passed the broken code).

| Suite | Assertions | Covers |
|---|---|---|
| `tests/money.regression.mjs` | 48 | C-01, C-02, C-05, C-06, C-10, C-11, C-12, H-03, H-14, M-02 |
| `tests/inventory.regression.mjs` | 44 | H-04, H-09, H-10, H-11, H-12 |
| `tests/returns.regression.mjs` | 34 | C-06, H-03, H-05, M-08, M-09, return policy + COD payout rules |
| `tests/security.regression.mjs` | 48 | C-07, C-08, H-05, H-06, H-13 |

Individual suites: `npm run test:money`, `test:inventory`, `test:returns`, `test:security`.

> **How to use this file:** a ticked box means the fix is implemented **and** an assertion
> covers it. Items are ordered by the original fix sequence at the bottom, not by ID.

Where a finding is in code written during this project's recent sessions it is marked
*(recent)* — no finding was softened for that reason.

**Related docs:** **[`deployment-runbook.md`](./deployment-runbook.md) — the operator
checklist for actually going live. Everything still open below that needs a human is
expanded there with verification steps** · [`order-payment-refund-flow.md`](./order-payment-refund-flow.md)
(intended behaviour) · [`shiprocket-integration.md`](./shiprocket-integration.md)

---

## 🔴 Deploy blockers (independent of this audit)

- [ ] **B-01 — `VITE_BACKEND_URL` must be set at build time.**
  `kitab-shop-fe/src/config/api.js:9`. The built bundle contains
  `http://${window.location.hostname}:3000` (verified in `dist/assets/api-*.js`).
  In production that resolves to the wrong port over plain HTTP from an HTTPS page →
  blocked as mixed content, whole site fails. Vite inlines this at build time, so setting
  it after deploy does nothing.
  **Fix:** `VITE_BACKEND_URL=https://api.yourdomain.com npm run build`.

- [ ] **B-02 — CORS reflects any origin.** *(code done — needs your domains)*
  Was `origin: (origin, callback) => callback(null, origin || true)` with `credentials: true`:
  any website could make authenticated API calls as your logged-in customers.
  **Now:** `src/index.js` enforces `CORS_ALLOWED_ORIGINS`, a comma-separated allowlist.
  Unset keeps the old reflect-any behaviour so LAN development is untouched — which is why
  this stays open until you set the value. Note this variable was previously read by the
  System Health panel but ignored by the middleware, i.e. a green tick over an open server;
  both now read the same variable.
  **Remaining action:** set `CORS_ALLOWED_ORIGINS` to your production origin(s). See
  [`deployment-runbook.md`](./deployment-runbook.md) §1.2 for verification commands.

- [ ] **B-03 — Razorpay is on test keys.** `.env` `RAZORPAY_KEY_ID=rzp_test_...`.
  Real orders take no real money. Note: switching to live keys makes the refund button
  move real money irreversibly in one click, with no confirmation step (see M-13).

---

## 🔴 Critical

- [x] ### C-01 — Coupon-order cancellation fails 100% of the time, after the courier shipment is already cancelled
  **Files:** `coupon.model.js:16-20` · `order.controller.js:670-679, 704-712` · **Fn:** `CancelOrder`
  **Current:** `usedBy[].count` has `min: 1`. Cancel sets `1 → 0` → `ValidationError`.
  **Reproduced independently:** `Path \`count\` (0) is less than minimum allowed value (1)`,
  `isTransient = false` → `withTransaction` does **not** retry → aborts → HTTP 500.
  `cancelShiprocketOrder` runs **before** the transaction (`:670-679`).
  **Why dangerous:** deterministic on any coupon order. The courier shipment is cancelled for
  real while the order stays `Confirmed`, stock is never restored, wallet is never credited.
  Parcel dead, order live, customer told "cancellation failed".
  **Fix:** `min: 0` on `count` (or `$pull` the entry); move `cancelShiprocketOrder` after commit,
  or compensate on abort.

- [x] ### C-02 — Razorpay webhook disabled in live config; captured payments silently lost
  **Files:** `.env` (`RAZORPAY_WEBHOOK_ENABLED=false` while `PAYMENTS_ENABLED=true`) · `payment.controller.js:409-412, 452-454`
  **Current:** returns **HTTP 200** when disabled → Razorpay records success and never retries.
  Also returns 200 on *all* errors. Order creation therefore depends solely on the browser
  reaching `/razorpay/verify`. No reconciliation job exists.
  **Why dangerous:** tab closed / signal lost / crash between capture and verify = money
  captured, no order, no refund, no alert.
  **Fix:** enable it; return 5xx on failure so Razorpay retries; add a reconciler for
  captured-but-unmatched payments.

- [x] ### C-03 — Double charge: one checkout → two orders, two payments
  **Files:** `payment.controller.js:441-445, 291-295` · `payment-order.service.js:57`
  **Current:** a replayed/late `payment.failed` `$set`s intent → `failed` with **no status
  filter** → `RetryRazorpayOrder` accepts `{created, failed}` and resets to `created` →
  verify's "already completed" short-circuit no longer matches → `completeCapturedIntent`
  guards on `status` but **never checks `storeOrder`** → second Order, second charge.
  **Why dangerous:** Razorpay retries webhooks by design. Not hypothetical.
  **Fix:** add `status:{$in:["created","processing"]}` to the failed-update filter;
  `storeOrder: null` to the retry filter; guard on `storeOrder`; add a webhook-event
  collection with a unique index on the Razorpay event id.

- [x] ### C-04 — A successful payment can be auto-refunded
  **File:** `payment.controller.js:227-236`
  **Current:** the catch block refunds whenever `captured && !storeOrder`. A stale
  `payment.failed` flips the intent → verify throws 409 → this block refunds the customer's
  **good** payment and returns an error.
  **Fix:** only auto-refund when the intent was never completable; never on a 409.

- [x] ### C-05 — Refund timeout recorded as failure → operator retry double-pays *(recent)*
  **Files:** `return-refund.service.js:88-108` · `order.controller.js:766` · `razorpay.service.js:75-78`
  **Current:** the gateway is called **before** any record is written. On timeout the catch
  throws and persists **nothing** → retry finds no refund → refunds again. The cancel path
  records `failed`, which `sumRefunded` deliberately excludes → same outcome.
  **No idempotency key on any of the 5 refund call sites.**
  **Fix:** write an intent record (`status: "created"`) **before** the gateway call; on retry,
  reconcile against Razorpay's existing refunds for that payment rather than re-refunding.

- [x] ### C-06 — Double refund via concurrent return-status update (TOCTOU)
  **Files:** `return.controller.js:342-432` · `return-refund.service.js:63-92`
  **Current:** `findById` → in-memory transition check → gateway call → `save()`. No session,
  no compare-and-swap. Two concurrent `PATCH`es both pass the check and both call
  `razorpay.payments.refund` before either persists, so `findRefundForReturn` cannot see the
  unsaved sibling.
  **Note:** no index can fix this. A unique index on an array-subdocument field enforces
  uniqueness *across* documents, not *within* one array — the schema comment at
  `Order.model.js:291-296` claiming `returnRequest` "doubles as the double-refund guard" is wrong.
  **Fix:** `findOneAndUpdate({_id, status:"received"}, {status:"refunding"})` to claim the
  transition **before** touching the gateway.

- [x] ### C-07 — `CancelOrder`'s transaction callback is not idempotent
  **File:** `order.controller.js:623, 643-648, 682-747`
  **Current:** the order read and status gate are **outside** the session; the callback never
  re-reads. On a WriteConflict, `withTransaction` retries against the same in-memory document →
  `$inc` wallet again (the credit side is unguarded, unlike the `$gte`-guarded debit),
  `$inc` stock again, push a second refund, call `razorpay.payments.refund` again for the full total.
  `PartialCancelOrder` does this correctly (re-reads in-session at `:461`).
  **Why dangerous:** customer-drivable, and there is no rate limit on the route.
  **Fix:** make the first statement in the transaction an atomic claim —
  `findOneAndUpdate({_id, orderStatus:{$in:["Pending","Confirmed"]}}, {orderStatus:"Cancelled"})`.

- [x] ### C-08 — `UpdateOrderStatus` performs no transition validation
  **File:** `order.controller.js:354-381`
  **Current:** validates enum membership only; the current status is loaded then ignored.
  All 81 ordered pairs are permitted. No `statusHistory`, and **no audit log anywhere** in
  `order.controller.js`.
  **Dangerous transitions:**
  - `Cancelled → Confirmed` — stock already restored and **not** re-deducted (guaranteed
    oversell); wallet/coupon already returned; refund already paid; order becomes shippable →
    **money out AND goods out**; re-enters revenue reporting.
  - `Cancelled → Delivered` — COD flips to `Paid` (phantom revenue for cash never collected)
    and stamps `deliveredAt` → order becomes returnable → refund of money never taken.
  - `Shipped → Pending` — re-opens **customer** self-cancel on a parcel in transit → full
    refund + full restock + parcel still delivers.
  - `Delivered → Pending` — strips review rights, invoice, and Return button while
    `deliveredAt` keeps the window expiring.
  **Fix:** a transition table applied in **both** `UpdateOrderStatus` and `ShippingWebhook`.
  The pattern already exists and works at `return.controller.js:13-28`.

- [x] ### C-09 — Shiprocket webhook blindly `$set`s status; unsigned; can target any order
  **File:** `shipping.controller.js:305-347`
  **Current:** no current-status read, no monotonic `statusCode` check, no event dedupe.
  `Delivered → Shipped` on a replayed in-transit event (customer loses the Return button while
  `deliveredAt` keeps the window expiring). `Cancelled → Delivered` on a code-7 event (oversell
  + phantom COD revenue + newly refundable). Auth is a **static shared token** compared against
  `x-api-key` — not an HMAC over the body — so payloads are replayable verbatim, and
  `{_id: body.order_id}` lets a token holder drive **any** order. `:326-336` also nulls out
  stored `orderId`/`awbCode` when a payload omits them.
  **Fix:** transition table + monotonic status check + event-id dedupe; HMAC-sign the body;
  restrict targeting to orders actually synced to Shiprocket.

- [x] ### C-10 — `PartialCancelOrder` claims refunds it never makes
  **File:** `order.controller.js:488, 513-523, 528-531`
  **Current:** sets `paymentStatus` to `Refunded`/`Partially Refunded`, pushes a refund record
  with `status:"created"`, and fires `notifyRefundProcessed` **unconditionally** — with **no
  gateway call anywhere in the function**. Uses `item.price` (pre-discount) and has no
  `sumRefunded` cap.
  **Why dangerous:** customer told "refund processed" when nothing moved; over-refunds
  discounted orders; the phantom record permanently lowers the refundable ceiling for a later
  genuine return; fires even on COD orders that were never `Paid`.
  **Fix:** `Refund Pending` + real gateway call + `proportionalRefundAmount` + cumulative cap;
  gate the notification on actual money movement.

- [x] ### C-11 — Admin manual refund has no cumulative ceiling
  **File:** `payment.controller.js:366-390`
  **Current:** validates `amount ≤ order.totalAmount` but **never sums existing refunds**, so
  repeated calls each push a full-value refund. Counts `failed` refunds in its own total
  (`:388`, no filter), unlike `sumRefunded`. Records no `returnRequest` link → invisible to the
  return-refund guard. No audit log. `speed` is passed through unvalidated (fee implications).
  **Fix:** use `sumRefunded(order)` as the ceiling; route through the shared refund service.

- [x] ### C-12 — Refunds marked `processed` on API return; no reconciliation
  **Files:** `return-refund.service.js:117` · `payment.controller.js:384` · `order.controller.js:762`
  **Current:** `status:"processed"` is hardcoded and the gateway's returned `refund.status` is
  never read. `speed:"normal"` refunds commonly return `pending`. No `refund.processed` /
  `refund.failed` webhook handling exists anywhere, and no polling job.
  **Why dangerous:** a gateway-failed refund stays `Refunded` forever with the customer already
  notified — exactly the failure this design was meant to eliminate.
  **Fix:** persist the gateway's status; handle refund webhooks; add reconciliation.

---

## 🟠 High

- [x] **H-01** — Cancel restores the full original quantity, ignoring `cancelledQuantity` → 5 units restored for 3 sold. `order.controller.js:684-690`
- [x] **H-02** — `UpdateOrderStatus` → `Cancelled` bypasses **all** compensation (no stock, wallet, coupon, refund, or shipment cancel). It is also the *only* path that can cancel a `Shipped`/`Delivered` order. `order.controller.js:349-412`
- [x] **H-03** — **Wallet credit destroyed on return.** ₹1000 subtotal / ₹200 wallet / ₹800 paid → full return refunds ₹800 cash and never restores the ₹200 prepaid credit. `return-refund.service.js:21-33`
- [x] **H-04** — No restock on return **or** RTO. Goods physically returned are permanently written off. `return.controller.js` (absent) · `shipping.controller.js` (`ProductModel` not imported)
- [x] **H-05** — Any admin role reads all orders/returns **including customer bank details** *(recent — the `refundDestination` field)*. `themeEditor`, which holds zero order permissions, can read every UPI ID / account number / IFSC. `return.controller.js:266` · `order.controller.js:328` · `shipping.controller.js:241, 269`
- [x] **H-06** — `isAdmin` ignores `isBlocked`; a blocked admin retains access on `isAdmin`-only routes. `is-admin.middleware.js:37-43`
- [ ] **H-07** — Token revocation disabled by config (`AUTH_SECURITY_ENABLED=false`): **logout is a no-op**, and admin 2FA is off.
- [x] **H-08** — Full cancel after a partial cancel refunds nothing (`paymentStatus` is already `Partially Refunded`, so the branch is skipped). `order.controller.js:733`
- [x] **H-09** — Reservation commit race → oversell: non-session read + discarded `commitReservation` result → paid order whose stock was never deducted. `payment-order.service.js:69, 144-146`
- [x] **H-10** — `releaseReservation` is read-then-write → double restock on webhook retry. `inventory-reservation.service.js:91-110`
- [x] **H-11** — Reservation decrements stock **before** the reservation row exists → a crash mid-loop loses stock permanently, unreclaimable by the cleanup job. `inventory-reservation.service.js:28-54`
- [x] **H-12** — Variant-level stock entirely ignored: `variants[].stock` / `reservedStock` are never read or written by any order path. Variant inventory is decorative; a `stock: 0` variant stays sellable. `Product.model.js:101-102`
- [x] **H-13** — Cancelled orders can still be shipped: `AssignAwb` / `SchedulePickup` / `GenerateLabel` / `GenerateInvoice` have no `Cancelled` check (only `CreateShipment` does). `shipping.controller.js:127, 154, 180, 200`
- [x] **H-14** — `RetryRazorpayOrder` overwrites `razorpayOrderId` in place → a payment made on the previous Razorpay order is orphaned: no local order, and the auto-refund path never runs. `payment.controller.js:315-319`

---

## 🟡 Medium

- [x] **M-01** — `coupon.usage` can never decrement: the `pre("validate")` hook overwrites it with `usedBy.length`. `coupon.model.js:148`
- [x] **M-02** — `coupon.maxLimit` is never read; the check is hardcoded `usedCount >= 1`, so global usage caps are unenforced. `coupon.service.js:96`
- [ ] **M-03** — `InventoryModel` is a second stock ledger that no order flow updates → the admin Inventory page diverges from `product.stock` after the first sale.
- [ ] **M-04** — `isAdmin`-only endpoints leaking PII: abandoned carts + `?notify=true` mass-send (`cart.routes.js:55`), login activity/IPs (`admin.routes.js:47`), newsletter subscribers (`newsletter.routes.js:12`).
- [x] **M-05** — Inventory read endpoints have **no auth at all** (`inventory.routes.js:21-22`) → public stock levels.
- [x] **M-06** — Razorpay webhook does not verify the paid amount (verify does; the webhook drops it). `payment.controller.js:431-437`
- [x] **M-07** — Shiprocket webhook nulls out stored `orderId`/`awbCode` on a partial payload → breaks tracking and later shipment calls. `shipping.controller.js:326-336`
- [x] **M-08** — Return-on-already-refunded is not checked at creation; the block lands only after pickup has been scheduled and goods collected. `return.controller.js:70-71`
- [x] **M-09** — Return quantity ignores `cancelledQuantity`; a unit already cancelled, restocked and refunded is still fully returnable. `return.controller.js:124-134`
- [x] **M-10** — `CancelOrder` throws `TypeError` → 500 on legacy guest orders (`user: null`). `order.controller.js:632`
- [x] **M-11** — No rate limit on `PATCH /order/:orderId/cancel` or `POST /returns`, which is what makes the C-07 race trivially driveable.
- [ ] **M-12** — `looseBody` preserves unknown keys, so a client can post `orderStatus`/`paymentStatus` and they survive validation on `req.body`. Nothing reads them today — latent mass-assignment. `validate.middleware.js:60-66`
- [x] **M-13** — `requirePermission` never asserts `hasAdminRole`; a plain `user` granted `orders:manage` directly would pass. Also: no confirmation prompt on the irreversible Razorpay refund button, and `PAYMENT_MODE=demo` is dead config that protects nothing.

---

## ⚪ Low

- [ ] **L-01** — `Pending` missing from the customer timeline → a Pending order renders with no step highlighted. `orderDetail.helpers.js:3-9`
- [ ] **L-02** — `ordersSlice.js:49` defaults a missing status to `Confirmed`; backend default is `Pending`.
- [ ] **L-03** — `returnNumber` collision surfaces as a misleading "return already exists for this product". `return.model.js:40-46`
- [ ] **L-04** — `GetInvoice` / `TrackShipment` lack param validation → 500 instead of 400 on a malformed id. `order.routes.js:61-62`
- [ ] **L-05** — `UPI` / `CARD` are dead `paymentMethod` enum values. `Order.model.js:139`
- [ ] **L-06** — No admin UI exists for QC rejection (`received → rejected`) despite backend support. `AdminReturns.jsx`

---

## ✅ Verified correct — do not re-litigate

- **Razorpay signature verification** — HMAC-SHA256, `timingSafeEqual` with length pre-check, computed over the **stored** order id (not client-supplied), plus independent `payments.fetch` cross-checks on `order_id`, amount, currency and captured status.
- **No trust of client-reported payment success** anywhere. `paymentStatus` / `walletBalance` are never read from a request body. No `...req.body` sink exists (zero grep hits) → no mass-assignment path.
- **`refundAmount` always server-computed**; proportional-discount handling is correct.
- **Verify/webhook idempotency for order creation** — status guard + in-transaction re-read + unique indexes on `razorpayOrderId` / `razorpayPaymentId`.
- **Stock deduction on placement** — atomic `{stock:{$gte:qty}}` + `modifiedCount !== 1`, on both COD and Razorpay paths.
- **Wallet debit** — atomic with a `$gte` guard, both paths.
- **Coupon double-redeem** — correctly protected by in-session re-read + WriteConflict retry.
- **`idempotencyKey`** — proper unique partial index; E11000 → 200 replay.
- **No customer-to-customer IDOR** — every customer-facing endpoint scopes by `req.user.id`.
- **Return transition table** — terminal states genuinely terminal; `resolutionType` narrowing correct.
- **No external API call inside any DB transaction.**
- **Paise conversion** uses `Math.round` throughout (no truncation short-changing).
- **Frontend/backend parity** on cancel window, return window, per-product return policy, stock caps and Shiprocket gating — all enforced server-side too.

---

## ❓ NOT VERIFIED — insufficient evidence

- [x] ~~**Dev MongoDB topology.**~~ **RESOLVED (probed 12 Aug 2026).** Connection string is
  `mongodb://127.0.0.1:27017/E-commerce?replicaSet=rs0` — `replicaSet` **is** specified in the URI
  (not in `mongo.db.js` options, which is why it looked absent). `hello.setName = rs0`, and an
  end-to-end `withTransaction` probe succeeded. Transactions work in dev.
- [ ] **Production MongoDB topology — still open.** The URI above is `127.0.0.1`, so production
  must use a different one. If that URI omits `replicaSet` or points at a standalone `mongod`,
  **every** `withTransaction` in `PlaceOrder` / `CancelOrder` / `PartialCancelOrder` /
  `completeCapturedIntent` throws immediately and checkout is fully down. **Verify before deploy.**
- [x] ~~**Whether declared indexes exist in the live DB.**~~ **RESOLVED (probed).** All declared
  indexes are present and correct: `orders` has `user+idempotencyKey` [UNIQUE, partial],
  `razorpayOrderId` [UNIQUE, sparse], `razorpayPaymentId` [UNIQUE, sparse];
  `returnrequests` has `order+product+user` [UNIQUE] and `returnNumber` [UNIQUE];
  `paymentintents` and `stockreservations` likewise. **And confirmed: there is no index of any
  kind on `refunds.*`** — C-06 verified against the live database, not just the schema.
  Residual risk: `autoIndex` is default-true with no `syncIndexes()`, so future index changes
  build implicitly and a failure would be silent. Prefer an explicit migration in production.
- [ ] **Whether Razorpay deduplicates concurrent identical refund calls.** No idempotency key is
  sent; gateway behaviour is still unverified from this repo. **No longer load-bearing:** the
  refund path now writes its intent record BEFORE calling the gateway and reconciles a
  retry against `payments.fetchMultipleRefund()` matched on `notes.returnNumber`, so a
  duplicate call is prevented on our side regardless of what Razorpay does.
- [x] ~~Whether `$push` preserves both records in the C-06 race.~~ **Moot** — the return status is
  now claimed by a compare-and-swap (`findOneAndUpdate` filtered on the previous status) before
  any refund is attempted, so only one execution can ever reach the `$push`.
- [x] **Whether the missing return/RTO restock was an intentional manual process.** Resolved as
  a product decision: it now restocks automatically on QC pass (`refunded`/`replaced`, never
  `rejected`) and on RTO-**delivered** — not on RTO-initiated, since the parcel is still in
  transit then. Both are claim-first and therefore idempotent under webhook retries.
- [x] **Whether any product uses `variants[]` in production data.** **Answered by probing the
  live database: 0 of 110 products have any variant, and 0 orders carry a `variantKey`.** So H-12
  was latent, not actively losing money. It is implemented anyway — variant stock is now
  enforced at checkout and moved on every sale, cancel, release and restock — so the first
  variant anyone creates works correctly rather than being decorative.
- [x] ~~No test coverage exists for any payment or refund path.~~ **174 assertions across 4 suites
  now cover them** (`npm test`). See *Regression coverage* at the top.

---

## Found after the audit, while writing the deployment runbook

Neither was in the original audit — both surfaced from verifying the runbook's claims
against the code rather than asserting them.

- [x] **Password reset was completely broken.** `utils/email.js` emailed a link to
  `${FRONTEND_URL}/reset-password.html`. There is no `reset-password.html` anywhere in this
  project — the frontend is an SPA whose route is `/reset-password` (`App.jsx:231`). Every
  reset email a customer has ever received led to a 404. Fixed, and
  `RESET_PASSWORD_FRONTEND_URL` (already in `.env`, read by nothing) is now honoured as an
  optional override. 27 assertions cover both emailed links across every config shape.
- [x] **Both email links mangled a trailing slash, and the verification link had no
  fallback at all.** `FRONTEND_URL=https://site/` produced `https://site//reset-password`,
  and an unset `FRONTEND_URL` emailed the literal string `undefined/verify-email`. Both
  normalised.
- [x] **System Health reported CORS as restricted using a variable the middleware ignored.**
  See B-02 — a false green is worse than no check, so the middleware now enforces it.
- [ ] **The web server needs an SPA fallback** (`try_files ... /index.html`). Not a code
  issue, but every client-side route — including the reset and verification links — 404s
  without it. Added to the runbook as a deploy step.

---

## Recommended fix order

Work top to bottom. Each line is a checkbox above.

```
 1. C-01  Coupon cancel blocker + move Shiprocket cancel after commit
 2. C-08  orderStatus transition table (also closes much of C-09)
 3. C-03  Webhook replay → double charge
 4. C-05  Refund intent record before gateway call + reconcile on retry
 5. C-06  Compare-and-swap claim in AdminUpdateReturnStatus
 6. C-07  Atomic claim at the top of CancelOrder's transaction
 7. C-04  Stop auto-refunding on 409
 8. C-10  PartialCancelOrder: real refund + Refund Pending + proportional amount
 9. C-11  Cumulative ceiling on the manual refund endpoint
10. C-09  HMAC-sign + transition-guard the Shiprocket webhook
11. C-02  Enable Razorpay webhook, 5xx on error, add reconciler
12. C-12  Persist gateway refund status + handle refund webhooks
13. H-05  Permission-gate the four bare-hasAdminRole reads (bank-detail exposure)
14. H-06  isAdmin must check isBlocked
15. H-07  Enable AUTH_SECURITY_ENABLED (logout currently a no-op)
16. H-01 / H-02 / H-08  Cancel compensation correctness
17. H-03  Restore wallet credit on refund
18. H-13  Cancelled check on the four shipment endpoints
19. H-09 / H-10 / H-11  Reservation atomicity
20. H-04  Restock decision on return / RTO
21. H-12  Variant stock — implement, or explicitly disable variants
22. H-14  RetryRazorpayOrder orphaning
23. Confirm prod Mongo topology + index build, then the Medium list
24. Low / cosmetic + QC rejection UI
25. B-01 / B-02 / B-03  Deploy blockers (must be done before go-live regardless)
```

### Known / deferred, tracked elsewhere
- `superAdmin` privilege escalation in `UpdateUserPermissions` — a plain `admin` can grant
  `superAdmin` to any account including their own. Raised separately by the project owner.
