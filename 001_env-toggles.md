# Environment Toggles — What Each One Actually Switches

Traced from the code, not from `.env.example`. Every claim below names the file that
reads the flag and the behaviour that changes.

Read this first if a feature "doesn't work" — in this codebase the usual cause is a
flag that is off, a flag whose **parent** is off, or a setting that lives in the
**database** rather than in `.env` at all.

---

## 1. How a toggle is parsed

All boolean flags go through `readBool` in `src/config/features.config.js`:

```js
const TRUE_VALUES = new Set(["1", "true", "yes", "on", "enabled"]);
// empty / missing  → the FALLBACK, not false
// value            → lowercased + trimmed, then matched against TRUE_VALUES
```

Three consequences that cause real confusion:

| You write | Result |
|---|---|
| `FLAG=true` `FLAG=TRUE` `FLAG=1` `FLAG=yes` `FLAG=on` `FLAG=enabled` | **on** |
| `FLAG=y` `FLAG=Y` `FLAG=t` | **off** — not in the list |
| `FLAG=` (present but empty) | **the default**, which for some flags is `true` |
| `FLAG` absent | the default |

So commenting out a flag is *not* the same as setting it to `false` — for
`INVENTORY_ENFORCE_STOCK`, `SECURITY_HEADERS_ENABLED`, `COMPRESSION_ENABLED` and
`RATE_LIMIT_ENABLED`, absent means **on**.

---

## 2. Configuration lives in three places

This is the single most important thing to understand.

```
┌─ .env ONLY ──────────────────────────────────────────────────────┐
│  Behavioural switches: payments, shipping, stock enforcement,    │
│  notifications, HTTP hardening, rate limiting.                   │
│  Changing these requires a restart.                              │
└──────────────────────────────────────────────────────────────────┘

┌─ DATABASE, with .env as FALLBACK ────────────────────────────────┐
│  Shiprocket credentials and package defaults.                    │
│  src/modules/shipping/shiprocket.service.js:20                   │
│      resolved(dbValue, envName, fallback)                        │
│      → a non-empty DB value WINS over .env                       │
│  Edit under Admin → Operations → Shipping.                       │
└──────────────────────────────────────────────────────────────────┘

┌─ DATABASE ONLY — no .env equivalent exists ──────────────────────┐
│  codEnabled, codServiceabilityCheckEnabled, shippingRatesEnabled,│
│  codMinOrderAmount, codMaxOrderAmount, cancellationWindowHours   │
│      src/modules/orders/CheckoutSetting.model.js                 │
│  lowStockThreshold (LOW_STOCK_THRESHOLD only SEEDS it once)      │
│      src/modules/inventory/InventorySetting.model.js:29          │
│  Edit under Admin → Operations → Checkout / Inventory.           │
└──────────────────────────────────────────────────────────────────┘
```

**Cash on Delivery is not an env toggle.** Searching `.env` for a COD switch finds
nothing, because `codEnabled` is a database setting. A fresh database defaults it to
**off**, and the checkout returns `403 COD_DISABLED`.

---

## 3. Dependency chains — flags that silently do nothing alone

A child flag set to `true` under a `false` parent is not an error and produces no
warning. It simply has no effect.

```
PAYMENTS_ENABLED ─────┬─▶ isPaymentEnabled()
PAYMENT_PROVIDER=razorpay ┘   both required

SHIPROCKET_ENABLED ───┬─▶ isShippingEnabled()
SHIPPING_PROVIDER=shiprocket ┘   both required
                              │
                              └─▶ isShippingWebhookEnabled()
                                       ▲
                     SHIPROCKET_WEBHOOK_ENABLED

NOTIFICATIONS_ENABLED ─▶ NOTIFY_SMS_ENABLED
                       ─▶ NOTIFY_WHATSAPP_ENABLED
                       ─▶ NOTIFY_PHONE_ENABLED
```

> **The one that bites hardest.** `SHIPROCKET_WEBHOOK_ENABLED=true` with
> `SHIPROCKET_ENABLED=false` makes the courier webhook answer
> `200 {"message":"Shiprocket webhook disabled"}` for every event — including
> correctly-signed ones. It looks like the webhook is working, because it returns
> 200. `isShippingWebhookEnabled()` is `isShippingEnabled() && webhookEnabled`
> (`features.config.js:78`).

---

## 4. Quick reference — every toggle

Default column = what happens when the key is absent or empty.

### Payments

| Toggle | Default | Switches |
|---|---|---|
| `PAYMENTS_ENABLED` | `false` | Master switch for Razorpay. Off ⇒ `canAutoRefund()` is false, so **every refund becomes manual** (recorded with a UTR instead of sent to the gateway), and online checkout is unavailable. |
| `PAYMENT_PROVIDER` | `razorpay` | Must equal `razorpay` or `isPaymentEnabled()` is false regardless of the flag above. |
| `RAZORPAY_WEBHOOK_ENABLED` | `false` | Only read by System Health to warn you. The webhook route itself is always mounted; what actually gates it is whether `RAZORPAY_WEBHOOK_SECRET` is set. |

**Required credentials:** `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`,
`RAZORPAY_WEBHOOK_SECRET`. Key prefix tells you the mode: `rzp_test_` vs `rzp_live_`.
Without the webhook secret, signature verification cannot pass — so the
*"customer paid then closed the tab"* recovery path is dead even though checkout works.

### Shipping / Shiprocket

Three of these are **ceilings, not switches.** Each has a matching admin-panel toggle
under Operations → Shipping, and the effective behaviour is `env AND admin choice`:

```
env says "permitted here"   AND   admin says "and we use it"   =   it happens
```

AND, never OR — an admin can narrow what the store uses but can never switch on
something the deployment forbids, so these stay a working kill switch. The admin-side
fields default to *on*, so a deployment that never opens the panel behaves exactly as
it did before they existed. `getShippingCapabilities()`
(`shiprocket.service.js`) is the single place this is resolved.

| Toggle | Default | Switches | Admin toggle |
|---|---|---|---|
| `SHIPROCKET_ENABLED` | `false` | Master switch. Off ⇒ shipment endpoints answer `503`, and cancelling an order **deliberately leaves the courier shipment alone** rather than pretending it was cancelled (`order-shipping.service.js`). | "Use Shiprocket for shipments" — off ⇒ `409` with an admin-actionable message instead of `503` |
| `SHIPPING_PROVIDER` | `shiprocket` | Must equal `shiprocket`, else the master switch is inert. | — |
| `SHIPROCKET_WEBHOOK_ENABLED` | `false` | Courier status events. Requires the master switch — see §3. | "Accept Shiprocket status updates" |
| `SHIPROCKET_AUTO_CREATE_ORDER` | `false` | When on, a confirmed order is pushed to Shiprocket automatically post-commit. | "Auto-push new orders" |

A fourth admin capability — **"Book return pickups & replacement parcels"** — has no env
flag of its own and inherits the shipments ceiling. It is the only one that defaults
**off**: the other three describe behaviour that already existed, while this one books real
courier collections at customer addresses, so a settings document written before the field
existed must not be treated as having opted in.

Auto-push and the status webhook are **nested under** shipments, not independent:
switching off "use Shiprocket for shipments" switches both off too, because pushing or
receiving status for shipments the store doesn't create through Shiprocket is
meaningless.

**Automatic vs. per-order fulfilment — what to set**

| Goal | `SHIPROCKET_ENABLED` | `SHIPROCKET_AUTO_CREATE_ORDER` | Admin: use Shiprocket | Admin: auto-push |
|---|---|---|---|---|
| Fully manual — never touch Shiprocket | `false` | any | any | any |
| Admin clicks Create Shipment per order | `true` | `false` | on | any |
| Same, but switchable without a restart | `true` | `true` | on | **off** |
| Auto-push on order confirmation | `true` | `true` | on | on |

Prefer row 3 over row 2: env changes need a restart, the admin toggle doesn't, so the
day the courier misbehaves an admin can stop auto-push without a deploy. Auto-push also
requires credentials to resolve — `capabilities.configured`, checked by
`syncOrderToShiprocketIfEnabled()`.

Auto-push fires post-commit from **two** call sites, not from `CreateShipment`:
`payment-order.service.js` (payment verified) and `order.controller.js` (order
confirmed). Both go through `syncOrderToShiprocketIfEnabled()` →
`createShiprocketOrder()` directly, so grepping for the controller will not find them.

**Credentials (DB wins, env is fallback):** `SHIPROCKET_EMAIL`,
`SHIPROCKET_PASSWORD`, `SHIPROCKET_PICKUP_LOCATION` (default `"Primary"`),
`SHIPROCKET_PICKUP_POSTCODE`, `SHIPROCKET_WEBHOOK_TOKEN`, `SHIPROCKET_BASE_URL`.

`isShiprocketConfigured()` is true only when **both** email and password resolve, so
`SHIPROCKET_ENABLED=true` with no credentials still refuses every call.

**Package defaults** used when a product has no dimensions of its own:
`SHIPROCKET_DEFAULT_WEIGHT_KG` (0.5), `_LENGTH_CM` / `_BREADTH_CM` / `_HEIGHT_CM`
(10 each).

### Inventory

| Toggle | Default | Switches |
|---|---|---|
| `INVENTORY_ENFORCE_STOCK` | **`true`** | The big one. On ⇒ add-to-cart, quantity increase, pricing and both order paths refuse when stock is short. **Off ⇒ overselling is permitted** — the atomic `$inc` still runs, but nothing rejects an insufficient line. Read in 6 files including `cart.controller.js` and `order-pricing.service.js`. |
| `INVENTORY_RESERVE_DURING_PAYMENT` | **`true`** | On ⇒ creating a Razorpay payment intent **decrements stock immediately** and writes a `StockReservation` with an expiry. Off ⇒ stock is only decremented at confirmation, so two shoppers can both reach checkout for the last unit. (`payment.controller.js:173`) |
| `INVENTORY_RESERVATION_CLEANUP_ENABLED` | on | The sweeper that returns stock from abandoned checkouts. Off ⇒ **abandoned prepaid carts hold stock forever.** |
| `INVENTORY_RESERVATION_CLEANUP_INTERVAL_MS` | `300000` (5 min) | Sweep frequency. |
| `LOW_STOCK_THRESHOLD` | `5` | **Seeds the database setting once.** After that the DB value is authoritative and this is ignored — change it under Admin → Inventory. |

### Notifications

| Toggle | Default | Switches |
|---|---|---|
| `NOTIFICATIONS_ENABLED` | **`false`** | Master switch for all outbound messaging. **Off ⇒ the COD OTP is generated but never delivered, so no customer can complete a COD order.** Also silences order-placed, payment-success and shipment-update messages. |
| `NOTIFY_SMS_ENABLED` | `false` | Adds the SMS channel. Requires the master switch. |
| `NOTIFY_WHATSAPP_ENABLED` | `false` | Adds WhatsApp. Requires the master switch. |
| `NOTIFY_PHONE_ENABLED` | `false` | Adds the phone channel. Requires the master switch. |

**Credentials:** `EMAIL`, `EMAIL_PASSWORD`.

> Having `EMAIL` set is **not** enough. The credentials and the flag are independent:
> mail can be fully configured while `NOTIFICATIONS_ENABLED=false` silently drops
> every message, including the OTP that COD depends on.

### Auth & security

| Toggle | Default | Switches |
|---|---|---|
| `AUTH_SECURITY_ENABLED` | `false` | Token revocation checks in `auth.middleware.js` and `is-admin.middleware.js`. Off ⇒ a revoked token stays valid until it expires — blocking a compromised admin does not take effect immediately. |
| `RATE_LIMIT_ENABLED` | **`true`** | All rate limiters, including the refund limiter. |
| `RATE_LIMIT_STORE` | `memory` | `memory` or `redis` (needs `REDIS_URL`). Memory does not share counts across instances, so limits are per-process. |
| `REFRESH_TOKEN_COOKIE_ENABLED` | `true` | **Dead config** — read and never consumed. |
| `SECURITY_HEADERS_ENABLED` | **`true`** | Helmet-style response headers in `index.js`. |
| `COMPRESSION_ENABLED` | **`true`** | gzip responses. |
| `JSON_BODY_LIMIT` | `1mb` | Max JSON body size. |
| `CORS_ALLOWED_ORIGINS` | *(empty)* | Comma-separated allow-list. **Empty reflects ANY origin back with credentials** — any website can call the API as a logged-in customer. System Health rates this CRITICAL in production. |

**Secrets:** `acess_token` (access-token signing key — note the spelling in code),
`refresh_token`, `SECRET_KEY`, `GOOGLE_CLIENT_ID`,
`PASSWORD_RESET_TOKEN_TTL_MINUTES`, `REFRESH_TOKEN_COOKIE_NAME`.

### Database, URLs, storage

| Key | Notes |
|---|---|
| `mango_url` | **Resolved first** — `src/database/mongo.db.js:12`. |
| `mongo_url`, `MONGO_URI`, `MONGODB_URI`, `MONGO_URL` | Fallbacks, in that order. |
| `port` | Lowercase. `process.env.port \|\| 3000` — `PORT` is **not** read. |
| `FRONTEND_URL`, `SITE_URL`, `RESET_PASSWORD_FRONTEND_URL` | Links in emails and redirects. |
| `UPLOADS_DIR` | Where product images are written. |
| `NODE_ENV` | `production` escalates several System Health checks from warning to critical. |
| `PAYMENT_INTENT_RETENTION_MS` | How long abandoned payment intents are kept. |

> **Footgun.** The primary variable is the misspelled `mango_url`, and it beats
> `mongo_url`. Setting only `mongo_url` while `mango_url` is present in `.env` points
> the app at whatever `mango_url` says. Verify with the connection log line or
> Admin → System Health, which prints the connected database name.

### Frontend (`kitab-shop-fe`)

| Key | Notes |
|---|---|
| `VITE_BACKEND_URL` | **Inlined at build time**, so it must be set for `npm run build`, not just at runtime. It also decides where every product image loads from — a wrong value breaks the catalogue visually, not only the API. Absent in dev ⇒ falls back to `http://<current-hostname>:3000`. |

---

## 5. Recipes — "I want X, what do I set?"

**Accept online payments**
```
PAYMENTS_ENABLED=true
PAYMENT_PROVIDER=razorpay
RAZORPAY_KEY_ID=…   RAZORPAY_KEY_SECRET=…   RAZORPAY_WEBHOOK_SECRET=…
```

**Accept Cash on Delivery** — two places, both required
```
NOTIFICATIONS_ENABLED=true      # else the OTP never arrives
EMAIL=…   EMAIL_PASSWORD=…
```
plus **Admin → Operations → Checkout → COD enabled** (a database setting; optional
min/max order value and a cancellation window live there too).

**Ship with Shiprocket**
```
SHIPROCKET_ENABLED=true
SHIPPING_PROVIDER=shiprocket
SHIPROCKET_WEBHOOK_ENABLED=true
SHIPROCKET_AUTO_CREATE_ORDER=true          # or leave off and create shipments by hand
```
plus credentials in `.env` **or** Admin → Operations → Shipping (DB wins), and
`SHIPROCKET_WEBHOOK_TOKEN` matching what you register in the Shiprocket dashboard.

**Ship manually instead** — no configuration at all. Leave `SHIPROCKET_ENABLED=false`
and use `PUT /api/v1/order/:orderId/shipment/manual` to record carrier and tracking
number. The customer sees both.

**Harden for production**
```
CORS_ALLOWED_ORIGINS=https://yourdomain.com,https://www.yourdomain.com
AUTH_SECURITY_ENABLED=true
SECURITY_HEADERS_ENABLED=true
RATE_LIMIT_ENABLED=true
RATE_LIMIT_STORE=redis      REDIS_URL=…     # if running more than one instance
NODE_ENV=production
```

**Load-test or seed data without stock refusals**
```
INVENTORY_ENFORCE_STOCK=false    # permits overselling — never in production
```

---

## 6. Dead config

Read into `getFeatures()` and never consumed. Setting them has no effect; they are
listed so nobody assumes otherwise.

- `REFRESH_TOKEN_COOKIE_ENABLED`

`PAYMENT_MODE` was here too. It defaulted to `demo`, was read into
`getFeatures().payments.mode`, and was consumed by nothing — a variable whose name
promised a safety net it did not provide. Removed from the code on 16 Sep 2026;
delete it from any `.env` you still have it in.

Never read at all — not even into `getFeatures()`:

- `SHIPROCKET_ADMIN_ONLY` — the restriction it describes is real but hard-coded:
  every shipment route carries `requirePermission(ADMIN_PERMISSIONS.ORDERS_MANAGE)`.
  Setting it to `false` grants nobody anything.

---

## 7. Diagnosing a flag problem

1. **Admin → System Health.** It reports what is configured, what is off, and what is
   critical — database connection and name, transactions, duplicate-protection
   indexes, Razorpay keys and webhook, unmatched captured payments, COD, cancellation
   window, CORS. Check here before reading `.env`.
2. **Is a parent flag off?** See §3.
3. **Is the setting actually in the database?** See §2 — COD and low-stock threshold
   are not in `.env` at all, and Shiprocket credentials in the DB override `.env`.
4. **Did you restart?** Every `.env` value is read through `process.env` at call time,
   but `dotenv` only loads the file at boot. It also **does not override** variables
   already present in the environment.
5. **Is the value in `TRUE_VALUES`?** `y`, `t` and `Y` are all off.
