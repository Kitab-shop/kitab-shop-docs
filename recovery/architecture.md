# Architecture Notes

## Application Shape

- The repository is a two-app bookstore system: `kitab-shop-fe` is a React/Vite/Tailwind frontend and `kitab-shop-be` is an Express/MongoDB backend.
- The frontend uses React Router in `kitab-shop-fe/src/App.jsx`, Redux Toolkit slices in `kitab-shop-fe/src/store/`, and API helpers in `kitab-shop-fe/src/config/api.js`.
- The backend uses feature modules under `kitab-shop-be/src/modules/`, Mongoose models for persistence, and route registration from `kitab-shop-be/src/index.js`.

## Referral Components

- Customer route wiring is in `kitab-shop-fe/src/App.jsx`: `/refer-and-earn` and `/account/refer-and-earn` both render `kitab-shop-fe/src/pages/Referral.jsx`.
- The customer page reads stats from `kitab-shop-fe/src/store/referralSlice.js`, which calls `GET /api/v1/user/profile/referral-stats`.
- Account dashboard and checkout also read referral wallet state through `fetchReferralStats` from `kitab-shop-fe/src/store/referralSlice.js`.
- Admin referral UI is `kitab-shop-fe/src/pages/admin/AdminReferrals.jsx`, which calls `/api/v1/referral/admin/*` endpoints for settings, stats, details, and deletes.
- Backend customer stats are served by `GetReferralStats` in `kitab-shop-be/src/modules/profiles/profile.controller.js`, routed by `kitab-shop-be/src/modules/profiles/user-profile.routes.js`; this response also includes current referral reward settings for customer-facing copy.
- Backend admin referral endpoints are in `kitab-shop-be/src/modules/referral/referral.routes.js` and `kitab-shop-be/src/modules/referral/referral.controller.js`.
- Referral program settings are stored in `kitab-shop-be/src/modules/referral/ReferralSetting.model.js`.

## Referral Data Boundaries

- `kitab-shop-be/src/modules/profiles/UserProfile.model.js` stores `referralCode`, `referredBy`, `walletBalance`, `totalReferrals`, and `totalWalletCreditEarned`.
- Email/password signup in `kitab-shop-be/src/modules/auth/auth.controller.js` validates referral codes, creates the user profile, and creates a single-use assigned coupon for the new user.
- Google signup in `kitab-shop-be/src/modules/auth/auth.controller.js` records `referredBy` for new profiles and uses the same referred-user coupon helper as email/password signup.
- First-order reward credit is applied in the COD order path in `kitab-shop-be/src/modules/orders/order.controller.js` and in the Razorpay completion path in `kitab-shop-be/src/modules/payments/payment-order.service.js`.
- Checkout wallet spend is priced in `kitab-shop-be/src/modules/orders/order-pricing.service.js`, displayed in `kitab-shop-fe/src/pages/Checkout.jsx`, and deducted during order completion.
