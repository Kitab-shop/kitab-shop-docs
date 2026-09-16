# Deployment Runbook — AstroMart Go-Live

**What this is:** the operator-side checklist. Everything here is configuration,
infrastructure or a verification step — **no code changes are required to go live.**
The code-side findings are tracked separately in
[`production-readiness-audit.md`](./production-readiness-audit.md).

**How to use it:** work top to bottom. Phase 0 → 3 must all pass before you take real
money. Phase 4 is the first-hour watch. Anything marked ⛔ will take the site or the
money handling down if you skip it.

**Fastest way to see where you stand:** log into the admin panel →
**Operations → System Health**. It derives every check below from the running
configuration, so it will tell you what is still red without reading this file. Two
useful properties: it never prints a secret, and it weighs settings in
*combination* — e.g. Razorpay enabled with test keys while `NODE_ENV=production` is
reported as critical, not as two separate warnings.

---

## Phase 0 — Before you touch the server

- [ ] **Decide the two public hostnames** and write them down; six later steps need them.
  - Storefront, e.g. `https://astromart.example`
  - API, e.g. `https://api.astromart.example`
- [ ] **⛔ Take a database backup and prove you can restore it.**
  ```bash
  cd kitab-shop-be
  npm run db:backup     # scripts/backup-db.sh
  npm run db:restore    # scripts/restore-db.sh — test this into a SCRATCH database
  ```
  A backup you have never restored is not a backup. Do the restore into a throwaway
  database name, not over the real one.
- [ ] **Run the regression suite against a copy of production data**, not just dev:
  ```bash
  cd kitab-shop-be && npm test     # 174 assertions, 4 suites
  ```
  It creates and deletes its own namespaced fixtures, but point it at a *restored copy*
  rather than live production — it writes to the database it is given.

---

## Phase 1 — ⛔ The four things that break real money

These are the deploy blockers. Each one silently produces wrong behaviour rather than
an obvious error, which is why they are first.

### 1.1 ⛔ Frontend must be built with the API URL baked in

Vite **inlines** `VITE_BACKEND_URL` at build time. Setting it on the server after
deploying does nothing at all.

```bash
cd kitab-shop-fe
VITE_BACKEND_URL=https://api.astromart.example \
VITE_GOOGLE_CLIENT_ID=<your-google-client-id> \
npm run build
# then serve kitab-shop-fe/dist/
```

**Why it matters:** with it unset, the bundle falls back to
`http://<current-hostname>:3000`. From an HTTPS page that is a plain-HTTP request to a
port that isn't open — the browser blocks it as mixed content and *every* API call
fails. The site loads and then does nothing.

- [ ] Built with `VITE_BACKEND_URL` set
- [ ] **Verify:** `grep -o 'https://api\.[^"]*' kitab-shop-fe/dist/assets/index-*.js | head`
      shows your API host and **no** `:3000` fallback remains reachable in the bundle

### 1.2 ⛔ CORS allowlist

```bash
CORS_ALLOWED_ORIGINS=https://astromart.example,https://www.astromart.example
```

Comma-separated, exact origins including the scheme. **Leave it unset and the server
reflects whatever origin asks** — combined with `credentials: true`, that means any
website a logged-in customer visits can call this API as them.

Notes on the matching, so the values you enter are right:
- Scheme and host must match exactly. `http://` will **not** match an `https://` entry.
- `www.` is a different origin. List both if both resolve.
- Requests with **no** `Origin` header are always allowed — that is curl, health probes,
  and the Razorpay/Shiprocket webhooks. They are never browser cross-origin requests.

- [ ] Set to your real storefront origin(s)
- [ ] **Verify** a refusal and an allow:
  ```bash
  # allowed → echoes your origin
  curl -sI -H "Origin: https://astromart.example" https://api.astromart.example/api/v1/product | grep -i access-control-allow-origin
  # refused → NO access-control-allow-origin header at all
  curl -sI -H "Origin: https://evil.example"      https://api.astromart.example/api/v1/product | grep -i access-control-allow-origin
  ```
- [ ] System Health → Security → **CORS policy** is green

### 1.3 ⛔ Razorpay live keys and the webhook

```bash
PAYMENTS_ENABLED=true
PAYMENT_PROVIDER=razorpay
RAZORPAY_KEY_ID=rzp_live_xxxxxxxx
RAZORPAY_KEY_SECRET=<live secret>
RAZORPAY_WEBHOOK_ENABLED=true
RAZORPAY_WEBHOOK_SECRET=<webhook secret from the Razorpay dashboard>
```

> `RAZORPAY_WEBHOOK_SECRET` is **a different value** from `RAZORPAY_KEY_SECRET`. You set
> it yourself when creating the webhook in the Razorpay dashboard. If they are the same
> value, every webhook fails signature verification and is rejected with 401.

In the **Razorpay dashboard → Settings → Webhooks**, register:

| Setting | Value |
|---|---|
| URL | `https://api.astromart.example/api/v1/payment/razorpay/webhook` |
| Secret | the same value you put in `RAZORPAY_WEBHOOK_SECRET` |
| Events | `payment.captured`, `payment.failed`, `refund.processed`, `refund.failed` |

**Why all four:** `payment.captured` is what creates the order when the customer's
browser dies after paying — without it, order creation depends entirely on the browser
reaching `/razorpay/verify`. `refund.processed` / `refund.failed` are what keep the
refund ledger honest: refunds at `speed: normal` settle asynchronously and can fail
afterwards, and without these handlers a gateway-failed refund would sit in your
records as "Refunded" forever with the customer already notified.

- [ ] Live keys set
- [ ] Webhook registered with all four events
- [ ] `RAZORPAY_WEBHOOK_SECRET` set and **different** from the key secret
- [ ] **Verify** with the dashboard's "Send test webhook": expect **HTTP 200**. A **401**
      means the secret is wrong. A **500** means it was received and processing failed —
      check the server log; the event is *not* marked delivered, so Razorpay will retry.
- [ ] System Health → Payments shows **LIVE** keys and a green webhook

⚠️ **Once live keys are in, the admin refund button moves real money in one click, with
no confirmation prompt** (audit item M-13, still open). Tell whoever operates the admin
panel before you hand it over.

### 1.4 ⛔ MongoDB must be a replica set

```bash
mongo_url=mongodb://user:pass@host:27017/E-commerce?replicaSet=rs0&authSource=admin
```

Checkout, cancellation, partial cancellation and payment capture all run inside
`session.withTransaction()`. **MongoDB only supports transactions on a replica set or a
sharded cluster.** Point this at a standalone `mongod`, or omit `replicaSet`, and every
one of those paths throws immediately — checkout is 100% down, not degraded.

Dev is verified as `rs0`. Production is a different URI and has **not** been verified.

- [ ] Production URI includes `replicaSet=`
- [ ] **Verify against the real production URI before deploying:**
  ```bash
  cd kitab-shop-be
  node -e "
  const m=require('mongoose');
  (async()=>{
    await m.connect(process.env.mongo_url);
    const info = await m.connection.db.admin().command({hello:1});
    console.log('setName:', info.setName || 'NONE — TRANSACTIONS WILL FAIL');
    const s = await m.startSession();
    await s.withTransaction(async()=>{ await m.connection.db.collection('__probe').insertOne({t:1},{session:s}); throw new Error('rollback'); }).catch(e=>{
      console.log(e.message==='rollback' ? 'withTransaction: OK' : 'withTransaction FAILED: '+e.message);
    });
    await s.endSession(); await m.disconnect();
  })();"
  ```
  You want `setName: rs0` (or whatever yours is called) **and** `withTransaction: OK`.
- [ ] System Health → Database → **Transactions / replica set** is green

---

## Phase 2 — Security posture

### 2.1 `AUTH_SECURITY_ENABLED` — a real decision, not a default

```bash
AUTH_SECURITY_ENABLED=true
```

With it **off** (the current setting): **logging out does not invalidate the access
token.** It stays valid until it expires. Admin two-factor authentication is also
disabled.

This was deliberately left off rather than flipped silently, because turning it on
changes login behaviour for every existing session — that belongs in a deploy step you
choose, not a code edit. System Health reports it as **critical** while
`NODE_ENV=production`.

- [ ] Decided and set. If you turn it on, log out and back in on one admin account and
      one customer account before opening the doors.

### 2.2 Production environment flags

```bash
NODE_ENV=production          # error responses stop including internal detail
SECURITY_HEADERS_ENABLED=true
RATE_LIMIT_ENABLED=true
RATE_LIMIT_STORE=redis       # only if you run more than one app process — see below
REDIS_URL=redis://...
```

- [ ] `NODE_ENV=production`
- [ ] **If you run more than one Node process or container:** `RATE_LIMIT_STORE=redis`
      with `REDIS_URL`. The default `memory` store counts per-process, so N processes
      means every limit is effectively N times looser.

### 2.3 Rotate every secret that has been in development

```bash
acess_token=<new long random string>     # note the spelling — it is the JWT signing key
refresh_token=<new long random string>
SECRET_KEY=<new long random string>
```

Anything committed, pasted, shared or used on a dev machine should be treated as known.
Rotating `acess_token` invalidates all existing sessions — expected, and best done
before launch rather than after.

- [ ] JWT signing secrets rotated
- [ ] `EMAIL` / `EMAIL_PASSWORD` are a real transactional sender (an app password, not a
      personal mailbox password)
- [ ] `GOOGLE_CLIENT_ID` matches `VITE_GOOGLE_CLIENT_ID`, and your production storefront
      origin is registered in the Google Cloud console

### 2.4 Admin accounts

- [ ] Every admin uses a role that matches their actual job. `themeEditor` can no longer
      read orders or customer bank details — use it for content staff.
- [ ] No leftover dev/test admin accounts. Blocking an account now revokes admin access
      immediately, so blocking is a valid way to retire one.
- [ ] Rotate the seeded admin password (`npm run admin:upsert`).

> Known and deliberately not changed: a plain `admin` can grant `superAdmin` via
> `UpdateUserPermissions`. You said you would raise that fix yourself — it is untouched.

---

## Phase 3 — Fulfilment and email

### 3.1 Shiprocket

Three **independent** switches. All three default to off, and each does something
different:

| Variable | What turning it on actually does |
|---|---|
| `SHIPROCKET_ENABLED` | Unlocks the integration. Off = every shipment endpoint returns 503. |
| `SHIPROCKET_AUTO_CREATE_ORDER` | Permits pushing each new paid order to Shiprocket automatically. Off = you click **Create Shipment** per order. |
| `SHIPROCKET_WEBHOOK_ENABLED` | Permits courier status callbacks. **Off = order status never updates itself and RTO parcels are never restocked.** |

These three are **ceilings, not the final word.** Each has a matching switch in
Operations → Shipping, and what the server does is `env AND admin choice` — so an admin
can narrow the store to manual fulfilment without a deploy, but can never enable
something you have forbidden here. The admin-side fields default to on, so setting these
three is sufficient for a fresh deployment; you only need to look at the panel if
someone has deliberately narrowed it. A capability chosen in the panel but blocked here
is shown there as "on, but not active" with this variable named.

Credentials live in the **admin panel** (Operations → Shipping), not in `.env` — env vars
are only a fallback. That is why the integration can be "enabled" with no credentials
saved, which is exactly the state System Health caught in dev: enabled, and every
shipment action failing with 503.

- [ ] Credentials saved in the admin panel (email, password, pickup location, pickup postcode)
- [ ] `SHIPROCKET_ENABLED=true`
- [ ] Webhook registered in the Shiprocket dashboard:
      `https://api.astromart.example/api/v1/order/shipping/webhook`,
      with the `x-api-key` header set to the **webhook token you saved in the admin panel**
- [ ] `SHIPROCKET_WEBHOOK_ENABLED=true`
- [ ] Decide `SHIPROCKET_AUTO_CREATE_ORDER` — on for hands-off, off if you want to review
      each order before booking a courier
- [ ] System Health → Shipping is green on credentials, webhook **and** pickup location

**Two things Shiprocket does not do**, so nobody plans around them:
1. **It never refunds a customer.** Its remittance is one-directional (D+8, to you).
   Every customer refund goes through Razorpay or a manual payout you record.
2. **It does not pack anything.** Auto-create books the shipment; you still pack the box.

### 3.2 Email

Order confirmations, refund notifications, COD OTP and password reset all use it.

- [ ] `NOTIFICATIONS_ENABLED=true`
- [ ] **⛔ `FRONTEND_URL` must be your production storefront.** It builds the clickable
      link in *both* the password-reset and email-verification messages. A stale value
      emails your customers a link to your dev machine. A trailing slash is now stripped
      automatically, and an unset value falls back to `localhost` rather than the literal
      string `undefined` — but neither fallback is something you want in a real email.
- [ ] `RESET_PASSWORD_FRONTEND_URL` — **optional.** Leave it unset unless you have moved
      the reset page; the link is derived from `FRONTEND_URL` when it is empty.
- [ ] **Verify end to end:** place one real test order and confirm the email arrives.
      Check spam. A new sending domain often needs SPF/DKIM before anything lands.
- [ ] **⛔ Click a real password-reset link and confirm the page loads.** Until this
      release the email pointed at `/reset-password.html`, which does not exist in this
      project — every reset link 404'd. It now points at the actual SPA route
      `/reset-password`. Verify it on the deployed site, because it also depends on your
      web server serving `index.html` for unknown paths (see below).
- [ ] **⛔ Your web server must serve `index.html` for any unmatched path.** This is a
      single-page app: `/reset-password`, `/verify-email`, `/account/returns/<id>` and
      every product URL are client-side routes with no file behind them. Without an
      SPA fallback, a direct visit or an emailed link returns 404 from the web server
      before React ever loads.
      - nginx: `location / { try_files $uri $uri/ /index.html; }`
      - Apache: `FallbackResource /index.html`

### 3.3 Inventory

```bash
INVENTORY_ENFORCE_STOCK=true
INVENTORY_RESERVE_DURING_PAYMENT=true
LOW_STOCK_THRESHOLD=5
```

- [ ] `INVENTORY_ENFORCE_STOCK=true` — with it off, stock is never checked or decremented
      and **the return/RTO restock is also skipped**, because there is no ledger to
      restock into
- [ ] Stock counts in the catalogue match physical reality before launch. Variant stock is
      now enforced, so a variant sitting at 0 becomes unsellable the moment you create one.

### 3.4 Brief whoever operates the admin panel on two new steps

Both are new decisions the system now asks for, and neither has a default:

- [ ] **Resolving a return asks for the goods' condition.** `Resellable` returns them to
      stock; `Damaged` writes them off. **The customer is refunded or replaced either
      way** — this only decides the stock. Operators need to hear that explicitly, or
      they will avoid `Damaged` thinking it penalises the customer.
- [ ] **An RTO parcel must be inspected.** The courier feed moves the order to
      `RTO Received` and records any refund owed to a prepaid customer, but **nothing is
      restocked until someone records the condition** (Orders → the order →
      RTO disposition). Parcels will silently pile up unrestocked if nobody is told to
      do this.
- [ ] **A prepaid RTO leaves a refund owed.** It is recorded, not sent — no automatic
      irreversible refund is triggered by a courier event. Someone has to action it.
      Watch for orders at `Refund Pending` with nothing in flight.

---

## Phase 4 — First hour live

Do these in order on the real site, with real money, before you advertise it.

- [ ] **One Razorpay order, start to finish.** Pay, confirm the order appears, confirm the
      email arrives, confirm stock decremented.
- [ ] **⛔ One deliberately abandoned checkout.** Start a Razorpay checkout, close the
      payment window, do not pay. Then confirm all of the following:
      it does **not** appear in the customer's My Orders; the admin dashboard revenue and
      order count are **unchanged**; if you used a coupon, it is **still usable**; and
      after ~20 minutes it shows as `Cancelled`. This is the single most important test
      of the order-first design — if revenue moved, stop and check `CORS`-style
      filtering on whatever screen you were looking at.
- [ ] **One RTO, end to end** (or simulate the webhook): confirm the order reaches
      `RTO Received`, that a prepaid order shows a refund **owed**, that a COD order shows
      **none**, and that stock only returns after you record the disposition.
- [ ] **One damaged return.** Refund the customer with disposition `Damaged` and confirm
      the units did **not** return to stock.
- [ ] **One deliberate browser-kill.** Pay, then close the tab before the success page
      loads. The order must still appear within a minute or two — that is
      `payment.captured` doing its job. **If it does not, stop and fix the webhook.**
      This is the single most valuable test here.
- [ ] **One COD order** through the OTP flow.
- [ ] **One cancellation** on a paid order. Check: status `Cancelled`, stock restored,
      refund recorded, and — if a coupon was used — the coupon usable again.
- [ ] **One refund** from the admin panel. Confirm it appears in the Razorpay dashboard
      and that `refund.processed` moves the order to `Refunded` on its own.
- [ ] **One return** end to end: request → approve → pickup → received → refund. Confirm
      the units come back into stock at the QC-pass step, not before.
- [ ] **System Health has no red rows.**
- [ ] Watch the server log for `Razorpay webhook processing failed` — the handler now
      answers 500 on failure so Razorpay retries, which means a burst of these is a real
      problem surfacing rather than being swallowed.

---

## Quick reference — full backend `.env`

Values are examples. `⛔` = the site or the money handling breaks without it.

```bash
# ── Core ─────────────────────────────────────────────────────────────────────
NODE_ENV=production                                    # ⛔
port=3000
mongo_url=mongodb://user:pass@host:27017/E-commerce?replicaSet=rs0   # ⛔ replicaSet
acess_token=<rotate>                                   # ⛔ JWT signing key (sic)
refresh_token=<rotate>                                 # ⛔
SECRET_KEY=<rotate>                                    # ⛔

# ── Public URLs (go into customer emails) ────────────────────────────────────
FRONTEND_URL=https://astromart.example                 # ⛔ builds reset + verify email links
SITE_URL=https://astromart.example                     # sitemap only
# RESET_PASSWORD_FRONTEND_URL=                         # optional override; derived from FRONTEND_URL

# ── Security ─────────────────────────────────────────────────────────────────
CORS_ALLOWED_ORIGINS=https://astromart.example,https://www.astromart.example   # ⛔
AUTH_SECURITY_ENABLED=true                             # off = logout does nothing
SECURITY_HEADERS_ENABLED=true
RATE_LIMIT_ENABLED=true
RATE_LIMIT_STORE=memory                                # redis if >1 process
# REDIS_URL=redis://...

# ── Payments ─────────────────────────────────────────────────────────────────
PAYMENTS_ENABLED=true                                  # ⛔
PAYMENT_PROVIDER=razorpay
RAZORPAY_KEY_ID=rzp_live_xxxx                          # ⛔ live, not test
RAZORPAY_KEY_SECRET=<live secret>                      # ⛔
RAZORPAY_WEBHOOK_ENABLED=true                          # ⛔
RAZORPAY_WEBHOOK_SECRET=<dashboard webhook secret>     # ⛔ NOT the key secret

# ── Shipping ─────────────────────────────────────────────────────────────────
SHIPROCKET_ENABLED=true
SHIPPING_PROVIDER=shiprocket
SHIPROCKET_AUTO_CREATE_ORDER=false                     # your call
SHIPROCKET_WEBHOOK_ENABLED=true                        # off = no status updates, no RTO restock
# credentials + webhook token: admin panel → Operations → Shipping

# ── Inventory ────────────────────────────────────────────────────────────────
INVENTORY_ENFORCE_STOCK=true                           # off also disables restock
INVENTORY_RESERVE_DURING_PAYMENT=true
LOW_STOCK_THRESHOLD=5
INVENTORY_RESERVATION_CLEANUP_ENABLED=true

# ── Email / notifications ────────────────────────────────────────────────────
NOTIFICATIONS_ENABLED=true
EMAIL=<transactional sender>
EMAIL_PASSWORD=<app password>

# ── Google sign-in ───────────────────────────────────────────────────────────
GOOGLE_CLIENT_ID=<must match VITE_GOOGLE_CLIENT_ID>
```

Frontend, at **build** time only:

```bash
VITE_BACKEND_URL=https://api.astromart.example         # ⛔ inlined at build
VITE_GOOGLE_CLIENT_ID=<same as backend GOOGLE_CLIENT_ID>
```

---

## Known open items you are launching with

Not blockers, but decide consciously rather than discovering them later. Full detail in
the [audit](./production-readiness-audit.md).

| Item | What it means in practice |
|---|---|
| **M-13** | The admin Razorpay refund button has no confirmation prompt. One click, real money, irreversible. |
| **M-03** | `InventoryModel` is a second stock ledger no order path writes. The admin Inventory page will drift from real stock after the first sale. Reporting only — `product.stock` is the real number. |
| **M-04** | Some `isAdmin`-only endpoints expose PII: abandoned carts (with a mass-send), login activity and IPs, newsletter subscribers. Not permission-gated by role. |
| **M-12** | `looseBody` keeps unknown keys, so a client can post `orderStatus`/`paymentStatus` in a body. Nothing reads them today — latent, not exploitable now. |
| **L-01…L-06** | Cosmetic: `Pending` missing from the customer timeline, a frontend status default mismatch, dead `UPI`/`CARD` enum values, no admin UI for QC rejection (the backend supports it — you would use the API or add the button). |
| **superAdmin escalation** | A plain `admin` can grant `superAdmin` via `UpdateUserPermissions`. Left untouched at your request. |
| **`autoIndex` is on** | Index changes build implicitly at boot and a failure is silent. Prefer an explicit migration in production. |

---

## If something goes wrong

| Symptom | Most likely cause | First thing to check |
|---|---|---|
| Site loads, nothing works, console shows blocked requests | `VITE_BACKEND_URL` was not set at **build** time | `grep -o 'http[s]*://[^"]*' dist/assets/index-*.js \| sort -u` |
| Every API call fails from the browser but curl works | Origin not in `CORS_ALLOWED_ORIGINS` (check `www.` and the scheme) | the two curl commands in §1.2 |
| Checkout 500s immediately, every time | Mongo is not a replica set | the probe in §1.4 |
| Customer paid, no order exists | `payment.captured` webhook not firing | Razorpay dashboard → Webhooks → delivery log |
| Webhooks return 401 | `RAZORPAY_WEBHOOK_SECRET` wrong, or set to the key secret by mistake | that they are two different values |
| Webhooks return 500 | Processing failed — the event is *not* marked delivered and will be retried | the server log for `Razorpay webhook processing failed` |
| Every shipment rejected, "pickup location not found" | The pickup name doesn't match one registered in Shiprocket | Operations → Shipping → **Load from Shiprocket**; System Health lists the real names |
| Credentials look saved but nothing works | Wrong password — "saved" only means a value exists | Operations → Shipping → **Test connection** |
| COD revenue looks too low | Delivered COD orders never flipped to Paid | Operations → **COD Reconciliation** → "Cash collected but not recorded" |
| Order status never updates after dispatch | `SHIPROCKET_WEBHOOK_ENABLED=false`, or the `x-api-key` token does not match | System Health → Shipping |
| Every shipment action returns 503 | Shiprocket enabled with no credentials saved | admin panel → Operations → Shipping |
| Order says "Refunded" but the customer has no money | Should no longer be possible — status is derived from settled gateway refunds only | the order's `refunds[]`; a `created` record means in flight, `failed` means it never moved |
| Password-reset link 404s | No SPA fallback on the web server, or a stale `FRONTEND_URL` | `try_files ... /index.html`; then the emailed URL's host and path |
| Emailed links point at localhost or your dev IP | `FRONTEND_URL` not set to production | `FRONTEND_URL` in the backend env |
| Refreshing any deep link 404s (e.g. an order page) | Same missing SPA fallback | as above |

**Rollback:** the frontend is a static `dist/` — keep the previous build and swap it
back. The backend is stateless; redeploy the previous commit. **Database changes are not
automatically reversible** — this release adds fields (`walletRefunded`,
`rtoRestockedAt`, `walletRefundAmount`, `restockedAt`, `previousRazorpayOrderIds`) and a
`webhookevents` collection. All are additive with safe defaults, so an older build
ignores them rather than breaking. That is why Phase 0 asks for a restore-tested backup.
