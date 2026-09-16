# Execution Flow

## Startup

1. Frontend starts through `kitab-shop-fe/src/main.jsx`, renders `App`, and lazy-loads pages defined in `kitab-shop-fe/src/App.jsx`.
2. Backend starts through `kitab-shop-be/src/index.js`, loads `.env`, connects MongoDB, initializes upload directories, starts cleanup work, and registers Express routes.

## Customer Referral Flow

1. A signed-in user opens `/refer-and-earn` or `/account/refer-and-earn`.
2. `kitab-shop-fe/src/pages/Referral.jsx` dispatches `fetchReferralStats`.
3. `kitab-shop-fe/src/store/referralSlice.js` calls `GET /api/v1/user/profile/referral-stats`.
4. `kitab-shop-be/src/modules/profiles/profile.controller.js` returns the user's referral code, referral count, lifetime earned credit, current wallet balance from `UserProfile`, and current reward settings from `ReferralSetting`.
5. The page lets the user copy their code or share a signup link containing `?ref=<code>`.

## New User Signup With Referral

1. `kitab-shop-fe/src/pages/Signup.jsx` reads `?ref=` into the referral code field.
2. `kitab-shop-fe/src/api/authApi.js` sends the referral code to the backend signup route through the auth slice.
3. Email/password signup in `kitab-shop-be/src/modules/auth/auth.controller.js` validates the code, stores `referredBy`, creates the new user's own referral code, and creates an assigned one-use coupon based on `ReferralSetting`.
4. Google signup receives the referral code through `kitab-shop-fe/src/pages/Login.jsx` and `kitab-shop-fe/src/api/authApi.js`; the backend stores `referredBy` for a new profile and creates the same assigned one-use coupon.

## Referrer Reward Flow

1. The referred user places their first real order.
2. COD order creation in `kitab-shop-be/src/modules/orders/order.controller.js` checks prior non-awaiting-payment orders and credits the referrer if this is the first.
3. Razorpay payment completion in `kitab-shop-be/src/modules/payments/payment-order.service.js` performs the same first-order reward check.
4. The referrer's `walletBalance`, `totalWalletCreditEarned`, and `totalReferrals` are incremented.
5. Checkout fetches referral stats and lets the referrer apply wallet balance to a later order.

## Admin Referral Flow

1. Admin opens `/admin/referrals`.
2. `kitab-shop-fe/src/pages/admin/AdminReferrals.jsx` fetches admin referral stats, settings, and detail lists from `/api/v1/referral/admin/*`.
3. Backend admin routes require token verification, admin role, and `referrals:manage` permission.
4. Admin can update referral settings and delete/refund accounting display records using backend referral endpoints.
