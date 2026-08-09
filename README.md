# chiz → Persian Cosmetics Platform — Master Implementation Backlog

**Structure:** Epic → Phase → Feature → Task
**Task ID scheme:** `E.P.F.T` (Epic.Phase.Feature.Task)
**Task size:** 30 min – 2 hrs, single coding session
**Priority scale:** P0 (blocker/launch-critical) · P1 (high) · P2 (medium) · P3 (nice-to-have)
**This document is sorted in overall execution order.** Epics near the top must land before epics near the bottom can safely be built on top of them.

---

## How to read this document
Each Feature has a task table with columns: **ID | Title | Priority | Difficulty | Est. Time | Dependencies | Description | Deliverable**. Dependencies reference other Task IDs. "None" means it can start immediately once its Phase is reached.

---

# EPIC 1 — Core Backend Stability
*Goal: fix the money-and-inventory-integrity bugs identified in the review before anything else is built on top of them.*

## Phase 1.1 — Transactional Order Safety

### Feature 1.1.1 — Atomic Order Creation

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 1.1.1.1 | Wrap order creation in `transaction.atomic()` | P0 | Easy | 30m | None | Wrap `OrderCreateSerializer.create()` body in an atomic block so a mid-request failure can't leave a half-created order. | Updated serializer, passing existing order tests |
| 1.1.1.2 | Add `select_for_update()` on product stock read | P0 | Medium | 1h | 1.1.1.1 | Lock product rows during checkout to prevent two concurrent orders from oversubscribing the same stock. | Updated queryset in order creation path |
| 1.1.1.3 | Decrement product stock on order creation | P0 | Medium | 1h | 1.1.1.2 | Subtract ordered quantity from `Product.stock` inside the atomic block; raise validation error if insufficient. | Stock decrement logic + unit test for oversell prevention |
| 1.1.1.4 | Restore stock on order cancellation | P0 | Easy | 45m | 1.1.1.3 | When `OrderDetailView.patch` cancels an order, return quantities to `Product.stock`. | Updated cancel endpoint + test |
| 1.1.1.5 | Add out-of-stock race condition test suite | P1 | Medium | 1.5h | 1.1.1.3 | Simulate concurrent order creation against limited stock to prove no oversell. | New pytest file `test_stock_concurrency.py` |

### Feature 1.1.2 — Extract Order Pricing Service

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 1.1.2.1 | Create `order/services/pricing.py` | P1 | Medium | 1h | None | Move tax/shipping/discount math out of the serializer into a standalone, unit-testable `PricingService`. | New service module |
| 1.1.2.2 | Refactor `OrderCreateSerializer` to call `PricingService` | P1 | Easy | 45m | 1.1.2.1, 1.1.1.1 | Replace inline calculations with service calls. | Refactored serializer |
| 1.1.2.3 | Unit tests for `PricingService` | P1 | Easy | 45m | 1.1.2.1 | Cover tax rate, shipping cost, discount edge cases (zero, negative rejection, over-total rejection). | Test file with ≥90% branch coverage of service |

## Phase 1.2 — Discount/Coupon Integrity
*(Coupon feature itself is built in Epic 9; this phase only closes the current security hole.)*

### Feature 1.2.1 — Remove Client-Controlled Discount

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 1.2.1.1 | Remove free-text `discount` field from checkout API | P0 | Easy | 30m | None | Delete the client-writable `discount` field from `OrderCreateSerializer`; discount becomes server-derived only (0 until Epic 9 ships). | Patched serializer, updated frontend checkout payload |
| 1.2.1.2 | Add regression test: forged discount is ignored | P0 | Easy | 30m | 1.2.1.1 | Post a checkout request with a manually injected discount and assert it has no effect on total. | New security test |

## Phase 1.3 — Review Authenticity

### Feature 1.3.1 — Authenticated, Purchase-Verified Reviews

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 1.3.1.1 | Add `user` FK to `Review` model | P0 | Easy | 30m | None | Replace free-text `name` with FK to `User`, keep `name` as denormalized display cache. | Migration + model change |
| 1.3.1.2 | Add `is_verified_purchase` boolean to `Review` | P1 | Easy | 20m | 1.3.1.1 | Flag set true if the reviewing user has a delivered `OrderItem` for this product. | Migration + field |
| 1.3.1.3 | Require authentication on `ProductReviewCreateView` | P0 | Easy | 20m | 1.3.1.1 | Change permission class from `AllowAny` to `IsAuthenticated`; auto-attach `request.user`. | Updated view |
| 1.3.1.4 | Enforce one review per user per product | P1 | Easy | 30m | 1.3.1.1 | Add `unique_together` constraint + serializer validation with friendly error. | Migration + validation + test |
| 1.3.1.5 | Compute `is_verified_purchase` at review creation | P1 | Medium | 45m | 1.3.1.2, 1.3.1.3 | Query user's delivered orders for the product at save time. | Updated `perform_create` logic |

---

# EPIC 2 — Authentication & Authorization (Iran-Market)
*Goal: add phone/OTP auth required for the Iranian market while keeping email as a secondary option.*

## Phase 2.1 — Phone Number Foundation

### Feature 2.1.1 — User Model Extension

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 2.1.1.1 | Add `phone_number` field to `User` model (not Profile) | P0 | Easy | 30m | None | Move phone to the auth model itself, normalized E.164/Iranian format, unique, nullable during migration. | Migration + model field |
| 2.1.1.2 | Add Iranian phone number validator | P0 | Easy | 30m | 2.1.1.1 | Regex validator for `09XXXXXXXXX` / `+989XXXXXXXXX` formats. | `validators.py` addition + tests |
| 2.1.1.3 | Data migration: backfill phone from `Profile.phone_number` | P1 | Easy | 30m | 2.1.1.1 | One-off migration copying existing profile phones to the new user field. | Migration file |

### Feature 2.1.2 — OTP Infrastructure

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 2.1.2.1 | Create `OTPCode` model | P0 | Easy | 45m | 2.1.1.1 | Fields: phone, code (hashed), purpose (login/register/reset), expires_at, attempts, is_used. | Model + migration |
| 2.1.2.2 | Add OTP generation service | P0 | Medium | 1h | 2.1.2.1 | 6-digit random code, hashed storage, configurable TTL (default 2 min), rate-limited generation per phone. | `accounts/services/otp.py` |
| 2.1.2.3 | Add OTP verification service | P0 | Medium | 1h | 2.1.2.2 | Validates code against hash, enforces max attempts (5), expiry, marks used. | Service + unit tests |
| 2.1.2.4 | Add per-phone OTP request throttle | P0 | Medium | 1h | 2.1.2.2 | Max 3 OTP requests per phone per 10 minutes to prevent SMS-bombing abuse. | DRF throttle class |

## Phase 2.2 — SMS Provider Integration
*(Full Kavenegar integration lives in Epic 16; this phase wires the auth flow to a swappable interface.)*

### Feature 2.2.1 — SMS Gateway Abstraction

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 2.2.1.1 | Define `SMSProvider` interface | P0 | Easy | 30m | None | Abstract base class with `send(phone, message)` so Kavenegar can be swapped later without touching auth code. | `notifications/sms/base.py` |
| 2.2.1.2 | Add console/dev SMS backend | P0 | Easy | 20m | 2.2.1.1 | Prints OTP to console/log in dev settings instead of sending real SMS. | Dev backend class |
| 2.2.1.3 | Wire OTP service to SMS provider interface | P0 | Easy | 30m | 2.2.1.1, 2.1.2.2 | OTP generation triggers `SMSProvider.send()`. | Updated OTP service |

## Phase 2.3 — OTP Auth Endpoints

### Feature 2.3.1 — Login/Register via OTP

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 2.3.1.1 | `POST /api/auth/otp/request/` endpoint | P0 | Medium | 1h | 2.1.2.2, 2.2.1.3 | Accepts phone, creates/finds user, sends OTP. | View + serializer + URL |
| 2.3.1.2 | `POST /api/auth/otp/verify/` endpoint | P0 | Medium | 1h | 2.1.2.3, 2.3.1.1 | Verifies OTP, creates user if new, issues JWT pair on success. | View + serializer + URL |
| 2.3.1.3 | Auto-create `Profile` on first OTP registration | P1 | Easy | 20m | 2.3.1.2 | Ensure the existing `post_save` signal handles OTP-created users correctly. | Verified/updated signal |
| 2.3.1.4 | Add OTP flow integration tests | P0 | Medium | 1h | 2.3.1.2 | Full request→verify→JWT issuance test, including expired/wrong-code/rate-limit cases. | Test suite |

### Feature 2.3.2 — Frontend OTP Auth UI

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 2.3.2.1 | Build phone-entry screen | P0 | Medium | 1h | 2.3.1.1 | Replace/augment `Login.jsx` with phone-number entry step. | React component |
| 2.3.2.2 | Build OTP-code entry screen with resend timer | P0 | Medium | 1.5h | 2.3.1.2 | 6-digit input, countdown before "resend" is enabled, error states. | React component |
| 2.3.2.3 | Update `AuthContext` to support OTP flow | P0 | Medium | 1h | 2.3.2.2 | Add `requestOtp`/`verifyOtp` methods alongside existing email/password methods. | Updated context |
| 2.3.2.4 | Add OTP flow component tests | P1 | Medium | 1h | 2.3.2.3 | Vitest coverage of happy path + invalid-code path. | Test file |

## Phase 2.4 — Authorization Hardening

### Feature 2.4.1 — Role & Permission Cleanup

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 2.4.1.1 | Replace magic numbers `(2,3)` with `UserType` enum in permission checks | P2 | Easy | 20m | None | `dashboard/permissions.py` currently hardcodes integers; use the existing `UserType` choices. | Refactored permissions file |
| 2.4.1.2 | Add object-level permission tests for `IsOwnerOrAdmin` | P2 | Easy | 30m | 2.4.1.1 | Cover owner access, non-owner denial, admin override. | Test additions |
| 2.4.1.3 | Add DRF throttle classes globally (anon/user scopes) | P0 | Medium | 1h | None | Configure `DEFAULT_THROTTLE_CLASSES`/`RATES` in settings for anon and authenticated users. | Settings update |
| 2.4.1.4 | Add stricter throttle scope for auth endpoints | P0 | Easy | 30m | 2.4.1.3 | Login/OTP/password-reset get a tighter rate than general API. | Throttle scope config |

---

# EPIC 3 — Product Catalog & Cosmetics Data Model
*Goal: reshape the generic product model into a real cosmetics/skincare catalog.*

## Phase 3.1 — Product Variant System

### Feature 3.1.1 — Variant Model

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 3.1.1.1 | Design `ProductVariant` model | P0 | Medium | 1h | None | Fields: product FK, sku, barcode, price override, stock, volume_ml, shade/color FK, is_active. | Model design doc + migration |
| 3.1.1.2 | Migrate existing `Product.stock`/`price` to a default variant | P0 | Hard | 2h | 3.1.1.1 | Data migration creating one variant per existing product, preserving stock/price/color data from `ProductColor`. | Migration + verification script |
| 3.1.1.3 | Update `Cart`/`CartItem` to reference variant instead of product | P0 | Hard | 2h | 3.1.1.2 | FK change, uniqueness constraint on (cart, variant). | Migration + model updates |
| 3.1.1.4 | Update `OrderItem` to snapshot variant SKU | P0 | Medium | 1h | 3.1.1.3 | Add `variant_sku`, `variant_attributes_json` snapshot fields. | Migration + model update |
| 3.1.1.5 | Update stock-decrement logic for variants | P0 | Medium | 1h | 3.1.1.4, 1.1.1.3 | Point Epic 1's atomic stock logic at `ProductVariant.stock` instead of `Product.stock`. | Updated service |
| 3.1.1.6 | Admin UI for managing variants (Django admin) | P1 | Medium | 1.5h | 3.1.1.1 | Inline variant editor on Product admin page. | Django admin config |

### Feature 3.1.2 — SKU & Barcode Generation

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 3.1.2.1 | Auto-generate SKU on variant save | P1 | Easy | 45m | 3.1.1.1 | Deterministic SKU pattern (e.g. `CAT-BRAND-ID`). | `save()` override + test |
| 3.1.2.2 | Add barcode field with format validation (EAN-13) | P2 | Easy | 30m | 3.1.1.1 | Optional field, validated if provided. | Validator + migration |

## Phase 3.2 — Cosmetics-Specific Attributes

### Feature 3.2.1 — Product Attribute Fields

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 3.2.1.1 | Add `skin_type` choice field | P0 | Easy | 20m | None | oily/dry/combination/sensitive/normal/all. | Migration + field |
| 3.2.1.2 | Add `hair_type` choice field | P1 | Easy | 20m | None | straight/wavy/curly/coily/all. | Migration + field |
| 3.2.1.3 | Add `spf` integer field | P1 | Easy | 15m | None | Nullable, only relevant for sun-care products. | Migration + field |
| 3.2.1.4 | Add `volume_ml`/`weight_g` fields | P0 | Easy | 20m | None | On variant, not product (differs per SKU). | Migration on `ProductVariant` |
| 3.2.1.5 | Add `ingredients` long-text field (INCI list) | P0 | Easy | 20m | None | TextField, searchable later. | Migration + field |
| 3.2.1.6 | Add `country_of_origin` field | P1 | Easy | 15m | None | CharField with country choices. | Migration + field |
| 3.2.1.7 | Add `expiration_date`/`manufacture_date` fields | P0 | Easy | 20m | 3.1.1.1 | On `ProductVariant` (batch-specific), nullable. | Migration + field |
| 3.2.1.8 | Add `batch_number` field | P1 | Easy | 15m | 3.1.1.1 | On `ProductVariant`. | Migration + field |
| 3.2.1.9 | Add `usage_instructions` and `warnings` text fields | P1 | Easy | 20m | None | Two TextFields on Product. | Migration + fields |
| 3.2.1.10 | Add `gender` choice field | P2 | Easy | 15m | None | unisex/female/male. | Migration + field |
| 3.2.1.11 | Add `is_cruelty_free`/`is_vegan`/`is_organic` booleans | P2 | Easy | 20m | None | Three flags, filterable on storefront. | Migration + fields |
| 3.2.1.12 | Add `irc_regulatory_code` field | P0 | Easy | 20m | None | Iran FDA cosmetics registration code; required for legal compliance, shown in admin, optional public display. | Migration + field |
| 3.2.1.13 | Admin verification workflow for regulatory code | P1 | Medium | 1h | 3.2.1.12 | Admin flag `regulatory_verified`, filterable list, cannot publish product without it (configurable). | Admin action + validation |
| 3.2.1.14 | Serializer updates to expose new attributes | P0 | Easy | 45m | 3.2.1.1–3.2.1.11 | Add all new fields to `ProductDetailSerializer`/`ProductListSerializer` as appropriate. | Updated serializers |
| 3.2.1.15 | Update `ProductFilter` for new filterable attributes | P0 | Medium | 1h | 3.2.1.14 | Add filters for skin_type, hair_type, spf range, vegan/cruelty-free flags. | Updated `filters.py` |

## Phase 3.3 — Expiration & Batch Tracking

### Feature 3.3.1 — Expiry Management

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 3.3.1.1 | Add "near expiry" admin filter/report | P2 | Easy | 45m | 3.2.1.7 | List variants expiring within 90 days. | Django admin filter |
| 3.3.1.2 | Prevent checkout of expired variants | P0 | Medium | 1h | 3.2.1.7, 3.1.1.5 | Validation in cart/checkout blocking add-to-cart or order creation for expired stock. | Validation logic + test |
| 3.3.1.3 | Celery task: nightly expiry sweep | P2 | Medium | 1h | Epic 22 Phase 22.1 | Auto-deactivate variants past expiration. | Scheduled task |

---

# EPIC 4 — Inventory Management

## Phase 4.1 — Stock Operations

### Feature 4.1.1 — Inventory Adjustments

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 4.1.1.1 | Create `StockMovement` audit log model | P0 | Medium | 1h | Epic 3 Phase 3.1 | Track every stock change: reason (sale/return/manual/restock), quantity delta, actor, timestamp. | Model + migration |
| 4.1.1.2 | Log stock movement on order creation | P0 | Easy | 30m | 4.1.1.1, 1.1.1.3 | Every decrement writes a `StockMovement` row. | Updated order service |
| 4.1.1.3 | Log stock movement on order cancellation | P0 | Easy | 30m | 4.1.1.1, 1.1.1.4 | Every restock writes a `StockMovement` row. | Updated cancel logic |
| 4.1.1.4 | Admin manual stock adjustment endpoint | P1 | Medium | 1h | 4.1.1.1 | Admin can add/remove stock with a reason note. | View + serializer + permission check |
| 4.1.1.5 | Low-stock threshold field + admin alert list | P1 | Easy | 45m | Epic 3 Phase 3.1 | `low_stock_threshold` on variant, admin dashboard widget listing items below it. | Field + admin view |

### Feature 4.1.2 — Back-in-Stock Notifications

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 4.1.2.1 | `StockAlertSubscription` model | P2 | Easy | 30m | Epic 3 Phase 3.1 | User + variant, created when a customer requests notification on an out-of-stock item. | Model + migration |
| 4.1.2.2 | "Notify me" API endpoint | P2 | Easy | 30m | 4.1.2.1 | Subscribe/unsubscribe endpoint. | View + serializer |
| 4.1.2.3 | Trigger notification when stock replenished | P2 | Medium | 1h | 4.1.2.1, Epic 16 Phase 16.2 | On stock increase from zero, queue notification emails/SMS to subscribers, then clear subscriptions. | Signal + Celery task |
| 4.1.2.4 | Frontend "Notify Me" button on out-of-stock PDP | P2 | Easy | 45m | 4.1.2.2 | Button replacing "Add to Cart" when stock is 0. | React component update |

---

# EPIC 5 — Cart & Checkout

## Phase 5.1 — Guest Cart Support

### Feature 5.1.1 — Session-Based Cart

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 5.1.1.1 | Make `Cart.user` nullable, add `session_key` field | P0 | Medium | 1h | Epic 3 Phase 3.1 | Support anonymous carts keyed by session/device token. | Migration + model |
| 5.1.1.2 | Cart middleware/helper to resolve cart by user-or-session | P0 | Medium | 1h | 5.1.1.1 | Central `get_or_create_cart(request)` helper used by all cart views. | New utility module |
| 5.1.1.3 | Merge guest cart into user cart on login | P0 | Medium | 1.5h | 5.1.1.2, Epic 2 Phase 2.3 | On successful auth, combine session cart items into the user's persistent cart. | Merge logic + test |
| 5.1.1.4 | Update frontend cart calls to work pre-login | P0 | Medium | 1h | 5.1.1.2 | Remove auth-required assumption from `CartContext`. | Updated context |

## Phase 5.2 — Checkout Flow Hardening

### Feature 5.2.1 — Checkout Validation

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 5.2.1.1 | Re-validate stock at checkout submit (not just cart-add time) | P0 | Medium | 1h | 1.1.1.3 | Prevent checkout with stale cart referencing since-sold-out stock. | Validation logic + test |
| 5.2.1.2 | Re-validate price at checkout (server-authoritative) | P0 | Medium | 45m | 5.2.1.1 | Never trust any client-sent price; always price from DB at submit time. | Validation logic + test |
| 5.2.1.3 | Address book model (`ShippingAddress`) | P1 | Medium | 1.5h | None | Let users save multiple addresses instead of re-typing every checkout. | Model + migration + CRUD endpoints |
| 5.2.1.4 | Iranian address fields (province, city, postal code format) | P0 | Medium | 1h | 5.2.1.3 | Replace generic state/zip fields with Iran-specific province/city dropdowns + 10-digit postal code validation. | Field updates + validator |
| 5.2.1.5 | Frontend address-book UI | P1 | Medium | 1.5h | 5.2.1.3 | Saved-address picker in checkout. | React component |

---

# EPIC 6 — Payments (Iranian Gateways)

## Phase 6.1 — Payment Foundation

### Feature 6.1.1 — Payment Domain Model

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 6.1.1.1 | Create `payments` Django app | P0 | Easy | 20m | None | New app registered in `INSTALLED_APPS`. | App scaffold |
| 6.1.1.2 | Create `PaymentTransaction` model | P0 | Medium | 1h | 6.1.1.1 | Fields: order FK, gateway, authority/ref_id, amount, status (pending/success/failed), created/updated timestamps, raw callback payload (JSON). | Model + migration |
| 6.1.1.3 | Add `requests` to `requirements.txt` | P0 | Easy | 5m | None | Currently missing entirely; needed for any gateway HTTP calls. | Updated requirements file |
| 6.1.1.4 | Define `PaymentGateway` abstract interface | P0 | Medium | 1h | 6.1.1.2 | Methods: `request_payment(amount, callback_url)`, `verify_payment(authority)`. | `payments/gateways/base.py` |

## Phase 6.2 — ZarinPal Integration (primary gateway)

### Feature 6.2.1 — ZarinPal Flow

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 6.2.1.1 | Implement `ZarinPalGateway.request_payment()` | P0 | Medium | 1.5h | 6.1.1.4 | Calls ZarinPal's payment request API, stores authority code. | Gateway implementation |
| 6.2.1.2 | `POST /api/payments/initiate/` endpoint | P0 | Medium | 1h | 6.2.1.1 | Takes order id, creates `PaymentTransaction`, redirects to gateway. | View + serializer |
| 6.2.1.3 | Implement `ZarinPalGateway.verify_payment()` | P0 | Medium | 1.5h | 6.2.1.1 | Calls verification API after redirect back. | Gateway method |
| 6.2.1.4 | `GET /api/payments/callback/zarinpal/` endpoint | P0 | Hard | 2h | 6.2.1.3 | Handles gateway redirect, verifies, updates transaction + order status atomically. | View + tests |
| 6.2.1.5 | Idempotency guard on callback endpoint | P0 | Medium | 1h | 6.2.1.4 | Prevent double-processing if gateway calls back twice or user refreshes. | Idempotency key check + test |
| 6.2.1.6 | Sandbox/test-mode config toggle | P1 | Easy | 30m | 6.2.1.1 | Env-based switch between ZarinPal sandbox and production merchant ID. | Settings + `.env.example` update |
| 6.2.1.7 | ZarinPal integration test suite (mocked HTTP) | P0 | Medium | 1.5h | 6.2.1.4 | Use `responses`/`requests-mock` to simulate gateway success/failure/timeout. | Test file |

## Phase 6.3 — Secondary Gateways

### Feature 6.3.1 — Zibal & IDPay

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 6.3.1.1 | Implement `ZibalGateway` | P2 | Medium | 1.5h | 6.1.1.4 | Same interface as ZarinPal. | Gateway implementation |
| 6.3.1.2 | Implement `IDPayGateway` | P2 | Medium | 1.5h | 6.1.1.4 | Same interface as ZarinPal. | Gateway implementation |
| 6.3.1.3 | Gateway selection logic (admin-configurable default + fallback) | P2 | Medium | 1h | 6.3.1.1, 6.3.1.2 | Setting to choose active gateway; fallback order if primary is down. | Settings + selection service |
| 6.3.1.4 | Callback routes for Zibal/IDPay | P2 | Medium | 1.5h | 6.3.1.1, 6.3.1.2 | Mirror the ZarinPal callback pattern. | Views + tests |

## Phase 6.4 — Payment UX & Reliability

### Feature 6.4.1 — Frontend Payment Flow

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 6.4.1.1 | Checkout "Pay Now" redirect flow | P0 | Medium | 1h | 6.2.1.2 | Replace fake card-entry UI with redirect-to-gateway button. | Updated `Checkout.jsx` |
| 6.4.1.2 | Payment result page (success/failure) | P0 | Medium | 1h | 6.2.1.4 | Handles gateway return URL, shows order confirmation or retry option. | New page component |
| 6.4.1.3 | Remove fake `card_last_four`/card-brand UI | P0 | Easy | 20m | 6.4.1.1 | Delete checkout-form theater fields no longer needed. | Cleanup |

### Feature 6.4.2 — Payment Reliability

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 6.4.2.1 | Celery task: reconcile stuck "pending" transactions | P1 | Medium | 1.5h | Epic 22 Phase 22.1, 6.2.1.3 | Periodic job re-verifies transactions stuck pending >30 min. | Scheduled task |
| 6.4.2.2 | Admin manual payment status override (with audit trail) | P1 | Medium | 1h | 6.1.1.2 | For support cases; logs who changed what. | Admin action |
| 6.4.2.3 | Refund tracking model/flow (manual, non-automated) | P2 | Medium | 1.5h | 6.1.1.2 | Track refund requests and status even if actual refund is processed manually via gateway panel initially. | Model + admin workflow |

---

# EPIC 7 — Shipping (Iranian Carriers)

## Phase 7.1 — Shipping Domain

### Feature 7.1.1 — Shipping Model & Rate Logic

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 7.1.1.1 | Create `shipping` app | P0 | Easy | 20m | None | New app scaffold. | App |
| 7.1.1.2 | `ShippingCarrier` model | P0 | Easy | 30m | 7.1.1.1 | Name, is_active, config JSON (API keys etc.). | Model + migration |
| 7.1.1.3 | `ShippingRate` model (by weight/city tiers) | P1 | Medium | 1h | 7.1.1.2 | Rate table keyed by carrier + destination city/province + weight bracket. | Model + migration |
| 7.1.1.4 | Replace hardcoded `SHIPPING_COST = 9.99` with rate lookup | P0 | Medium | 1h | 7.1.1.3, Epic 1 Phase 1.1 | `PricingService` calls shipping rate calculator instead of constant. | Updated service |

## Phase 7.2 — Carrier Integrations

### Feature 7.2.1 — Carrier APIs

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 7.2.1.1 | Define `CarrierProvider` interface | P0 | Medium | 45m | 7.1.1.2 | Methods: `get_rate()`, `create_shipment()`, `track()`. | Base class |
| 7.2.1.2 | Implement Post (Iran Post) provider | P1 | Medium | 1.5h | 7.2.1.1 | Rate + label creation via Post API. | Provider implementation |
| 7.2.1.3 | Implement Tipax provider | P2 | Medium | 1.5h | 7.2.1.1 | Same interface. | Provider implementation |
| 7.2.1.4 | Implement SnapBox provider | P2 | Medium | 1.5h | 7.2.1.1 | Same interface. | Provider implementation |
| 7.2.1.5 | Implement AloPeyk provider (same-day/local) | P2 | Medium | 1.5h | 7.2.1.1 | Same interface, city-restricted availability logic. | Provider implementation |
| 7.2.1.6 | Carrier selection UI in checkout | P1 | Medium | 1.5h | 7.2.1.2 | Show available carriers + rates + ETA at checkout based on destination. | React component |

### Feature 7.2.2 — Shipment Tracking

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 7.2.2.1 | `Shipment` model linking order to carrier tracking number | P1 | Easy | 30m | 7.2.1.2 | Model + migration. | Model |
| 7.2.2.2 | Celery task polling tracking status | P2 | Medium | 1.5h | 7.2.2.1, Epic 22 Phase 22.1 | Periodic status sync, updates order status on delivery. | Scheduled task |
| 7.2.2.3 | Customer-facing tracking widget on order detail page | P1 | Medium | 1h | 7.2.2.1 | Show shipment status/timeline. | React component |

---

# EPIC 8 — Order Management (Admin Side)

## Phase 8.1 — Order Lifecycle

### Feature 8.1.1 — Admin Order Operations

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 8.1.1.1 | Admin order status transition endpoint with validation | P0 | Medium | 1h | Epic 1 Phase 1.1 | Enforce valid state machine (pending→processing→shipped→delivered; cancelled only from pending/processing). | View + state machine util + test |
| 8.1.1.2 | Order status change triggers notification | P1 | Easy | 30m | 8.1.1.1, Epic 16 Phase 16.2 | Hook into notification system on each transition. | Signal/service call |
| 8.1.1.3 | Admin order search/filter (by number, customer, status, date range) | P1 | Medium | 1h | None | Extend admin order list endpoint with `django-filter`. | Updated view + filterset |
| 8.1.1.4 | Order export (CSV) for accounting | P2 | Medium | 1h | None | Admin action to export filtered orders. | Export endpoint |
| 8.1.1.5 | Invoice PDF generation per order | P1 | Hard | 2h | None | Generate a Persian-formatted invoice PDF (Toman amounts, Jalali date) on demand. | PDF generation service + endpoint |

---

# EPIC 9 — Coupons & Promotions

## Phase 9.1 — Coupon Engine

### Feature 9.1.1 — Coupon Model & Validation

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 9.1.1.1 | Create `promotions` app | P0 | Easy | 20m | None | App scaffold. | App |
| 9.1.1.2 | `Coupon` model | P0 | Medium | 1h | 9.1.1.1 | Code, type (percent/fixed), value, min_order_amount, max_uses, uses_per_user, valid_from/until, is_active. | Model + migration |
| 9.1.1.3 | `CouponRedemption` model (usage tracking) | P0 | Easy | 30m | 9.1.1.2 | Tracks which user used which coupon on which order, prevents reuse abuse. | Model + migration |
| 9.1.1.4 | Coupon validation service | P0 | Medium | 1.5h | 9.1.1.2, 9.1.1.3 | Checks expiry, usage caps, min order amount, per-user limits. | `promotions/services/validate.py` + tests |
| 9.1.1.5 | Wire coupon service into `PricingService` | P0 | Medium | 1h | 9.1.1.4, Epic 1 Feature 1.1.2 | Replace the removed client-discount field with server-validated coupon-derived discount. | Updated pricing service |
| 9.1.1.6 | `POST /api/cart/apply-coupon/` endpoint | P0 | Medium | 1h | 9.1.1.5 | Validates and attaches coupon to current cart/session. | View + serializer |
| 9.1.1.7 | Coupon-restricted-to-category/product support | P2 | Medium | 1h | 9.1.1.2 | M2M fields limiting applicability. | Model extension |
| 9.1.1.8 | Admin coupon CRUD UI | P1 | Medium | 1.5h | 9.1.1.2 | List/create/edit/deactivate coupons. | Admin React pages |
| 9.1.1.9 | Frontend "apply coupon" UI in cart/checkout | P0 | Medium | 1h | 9.1.1.6 | Input field with validation feedback. | React component |

## Phase 9.2 — Flash Sales & Campaigns

### Feature 9.2.1 — Time-Boxed Promotions

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 9.2.1.1 | `FlashSale` model (products + discount % + time window) | P2 | Medium | 1h | 9.1.1.1 | Model + migration. | Model |
| 9.2.1.2 | Flash sale price resolution in product serializer | P2 | Medium | 1h | 9.2.1.1 | Show active sale price when within window. | Serializer logic |
| 9.2.1.3 | Homepage flash-sale countdown banner | P2 | Medium | 1.5h | 9.2.1.1 | React component with live countdown. | Component |

---

# EPIC 10 — Reviews & Ratings
*(Core safety already covered in Epic 1 Phase 1.3 — this epic adds the remaining review features.)*

## Phase 10.1 — Review Moderation & Richness

### Feature 10.1.1 — Moderation

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 10.1.1.1 | Add `is_approved` flag with admin moderation queue | P1 | Medium | 1h | Epic 1 Feature 1.3.1 | Reviews default to pending until approved (or auto-approve verified purchases, configurable). | Field + admin view |
| 10.1.1.2 | Review reporting ("flag as inappropriate") | P2 | Medium | 1h | 10.1.1.1 | User-facing flag button, admin review of flagged content. | Model + endpoint + UI |
| 10.1.1.3 | Review image upload support | P2 | Medium | 1.5h | Epic 1 Feature 1.3.1 | Let verified purchasers attach photos to reviews. | Model field + upload handling |
| 10.1.1.4 | Admin reply to reviews | P2 | Easy | 45m | 10.1.1.1 | Store-response field visible publicly under the review. | Field + UI |

---

# EPIC 11 — Wishlist

## Phase 11.1 — Wishlist Enhancements

### Feature 11.1.1 — Wishlist Extensions

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 11.1.1.1 | Guest wishlist (session-based, mirrors cart pattern) | P2 | Medium | 1h | Epic 5 Feature 5.1.1 | Reuse session-cart pattern for wishlist. | Model + view updates |
| 11.1.1.2 | Price-drop notification on wishlisted items | P2 | Medium | 1h | Epic 16 Phase 16.2 | Notify user when a wishlisted product's price decreases. | Signal + notification task |
| 11.1.1.3 | "Move to cart" bulk action from wishlist | P2 | Easy | 30m | None | Frontend action, one or all items. | React component update |

---

# EPIC 12 — Search & Filtering

## Phase 12.1 — Search Quality

### Feature 12.1.1 — Persian-Aware Search

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 12.1.1.1 | Evaluate & integrate Postgres full-text search (`SearchVector`) | P1 | Medium | 1.5h | None | Replace basic `LIKE`-based `SearchFilter` with weighted FTS across name/description/brand. | Updated queryset + migration for search vector column |
| 12.1.1.2 | Persian text normalization for search (ك/ک, ي/ی, zero-width chars) | P0 | Medium | 1h | 12.1.1.1 | Normalize Arabic vs Persian character variants before indexing/querying. | Normalization utility + tests |
| 12.1.1.3 | Search-as-you-type suggestion endpoint | P2 | Medium | 1.5h | 12.1.1.1 | Lightweight autocomplete endpoint (top N matches). | View + frontend dropdown |
| 12.1.1.4 | Search analytics logging (query + result count) | P2 | Easy | 45m | None | Log searches for later merchandising insight. | Model + logging hook |

## Phase 12.2 — Filter UX

### Feature 12.2.1 — Cosmetics Filter Facets

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 12.2.1.1 | Frontend filter sidebar for skin/hair type, SPF, vegan/cruelty-free | P0 | Medium | 1.5h | Epic 3 Feature 3.2.1 | Multi-select filter UI wired to `ProductFilter` query params. | React component |
| 12.2.1.2 | Filter result counts (facet counts) | P2 | Medium | 1h | 12.2.1.1 | Show "(42)" next to each filter option. | Backend aggregation + frontend display |
| 12.2.1.3 | Persist filter state in URL query params | P1 | Easy | 45m | 12.2.1.1 | Shareable/bookmarkable filtered URLs. | Router integration |

---

# EPIC 13 — Recommendation Engine

## Phase 13.1 — Baseline Recommendations

### Feature 13.1.1 — Rule-Based Recommendations

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 13.1.1.1 | "Frequently bought together" (co-occurrence in `OrderItem`) | P2 | Medium | 1.5h | None | Simple SQL aggregation of co-purchased products. | Service + endpoint |
| 13.1.1.2 | "Recently viewed" tracking | P1 | Medium | 1h | None | Session/localStorage or backend-tracked view history. | Model or frontend storage + component |
| 13.1.1.3 | "Customers also bought" widget on PDP | P2 | Easy | 1h | 13.1.1.1 | Frontend component consuming the co-occurrence endpoint. | React component |
| 13.1.1.4 | Personalized homepage section (based on browsing/purchase history) | P3 | Hard | 2h | 13.1.1.2 | Basic collaborative signal, not ML — category affinity scoring. | Service + component |

---

# EPIC 14 — Persian Localization & i18n

## Phase 14.1 — Backend i18n Foundation

### Feature 14.1.1 — Django i18n Setup

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 14.1.1.1 | Set `LANGUAGE_CODE = "fa"`, configure `LANGUAGES` | P0 | Easy | 20m | None | Base settings update. | Settings change |
| 14.1.1.2 | Set `TIME_ZONE = "Asia/Tehran"` | P0 | Easy | 10m | None | Settings change. | Settings change |
| 14.1.1.3 | Wrap all admin-facing/user-facing strings in `gettext_lazy` | P1 | Hard | 2h (recurring, track as multiple sessions) | 14.1.1.1 | Sweep models/serializers/error messages for translatable strings. | Multiple commits, `.po` source strings ready |
| 14.1.1.4 | Generate and populate `fa` `.po`/`.mo` translation files | P1 | Medium | 1.5h | 14.1.1.3 | `makemessages`/`compilemessages` workflow. | Translation files |

### Feature 14.1.2 — Currency Handling

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 14.1.2.1 | Decide internal currency unit (store Rial as integer, display Toman) | P0 | Medium | 1h | None | Architecture decision + migration plan for all monetary fields. | Decision doc + migration |
| 14.1.2.2 | Migrate monetary `DecimalField`s to integer Rial | P0 | Hard | 2h | 14.1.2.1 | Touches Product, Order, OrderItem, Cart, Coupon amounts. | Migrations across apps |
| 14.1.2.3 | Toman display formatting utility (frontend) | P0 | Easy | 45m | 14.1.2.2 | `formatToman()` helper with thousand separators. | Utility function + tests |
| 14.1.2.4 | Persian digit rendering toggle | P1 | Easy | 45m | 14.1.2.3 | Convert 0-9 to ۰-۹ for display (with a toggle, since some contexts prefer Western digits). | Utility function |

## Phase 14.2 — Jalali Calendar

### Feature 14.2.1 — Date Handling

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 14.2.1.1 | Add `django-jalali`/`jdatetime` to backend | P1 | Easy | 30m | None | Dependency install + admin date widget config. | Requirements update |
| 14.2.1.2 | Jalali date display in Django admin | P1 | Medium | 1h | 14.2.1.1 | Order/product created dates shown in Jalali for admin staff. | Admin config |
| 14.2.1.3 | Frontend Jalali date library integration | P0 | Medium | 1h | None | Add `dayjs`-jalali plugin or equivalent. | Frontend dependency + utility |
| 14.2.1.4 | Jalali date picker component (checkout/admin forms) | P1 | Medium | 1.5h | 14.2.1.3 | Reusable date-picker for any date input in the app. | React component |

## Phase 14.3 — RTL & Typography

### Feature 14.3.1 — RTL Layout

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 14.3.1.1 | Set `dir="rtl"` and `lang="fa"` on root HTML | P0 | Easy | 15m | None | `index.html` update. | Config change |
| 14.3.1.2 | Configure Tailwind RTL plugin / logical properties | P0 | Medium | 1.5h | 14.3.1.1 | Replace left/right-specific utility classes with logical (start/end) equivalents project-wide. | Tailwind config + component sweep |
| 14.3.1.3 | Audit and fix icon/layout mirroring (arrows, carousels) | P1 | Medium | 1.5h | 14.3.1.2 | Chevrons, sliders, breadcrumbs need RTL-aware direction. | Component fixes |
| 14.3.1.4 | Integrate Persian web font (Vazirmatn/Yekan) | P0 | Easy | 30m | None | Self-hosted font, added to Tailwind config. | Font files + config |
| 14.3.1.5 | RTL visual regression test pass (manual checklist) | P1 | Medium | 1h | 14.3.1.3 | Document + verify every major page in RTL mode. | QA checklist doc |

## Phase 14.4 — Slug & URL Strategy

### Feature 14.4.1 — Persian-Friendly URLs

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 14.4.1.1 | Decide slug strategy: transliteration vs. Persian-script slugs | P1 | Easy | 30m | None | Architecture decision (transliteration recommended for URL safety/SEO tooling compatibility). | Decision doc |
| 14.4.1.2 | Implement Persian-to-Latin transliteration slugify function | P1 | Medium | 1h | 14.4.1.1 | Replace Django's default `slugify` for Persian product/category names. | Utility + tests |
| 14.4.1.3 | Backfill slugs for future Persian product names | P1 | Easy | 30m | 14.4.1.2 | Verify seed/management commands use the new slugify. | Updated seed commands |

---

# EPIC 15 — SEO

## Phase 15.1 — Technical SEO

### Feature 15.1.1 — Metadata & Structured Data

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 15.1.1.1 | Add `react-helmet-async` for per-page meta tags | P0 | Easy | 30m | None | Dependency + provider setup. | Package + provider |
| 15.1.1.2 | Dynamic title/description per PDP/category page | P0 | Medium | 1h | 15.1.1.1 | Persian meta content generated from product/category data. | Component updates |
| 15.1.1.3 | JSON-LD structured data for products (Product/Offer schema) | P1 | Medium | 1h | 15.1.1.2 | Rich snippets for price/availability/rating. | Component addition |
| 15.1.1.4 | Open Graph tags for social sharing | P2 | Easy | 30m | 15.1.1.2 | og:title/description/image per page. | Meta tag additions |
| 15.1.1.5 | Canonical URL tags | P1 | Easy | 20m | 15.1.1.2 | Prevent duplicate-content issues from filter query params. | Meta tag logic |
| 15.1.1.6 | `hreflang` setup if multi-language is ever added | P3 | Easy | 20m | 15.1.1.2 | Placeholder for future EN version. | Meta tag |

### Feature 15.1.2 — Crawlability

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 15.1.2.1 | Extend existing `sitemaps.py` to cover all product/category/blog URLs with Persian slugs | P1 | Easy | 45m | 14.4.1.2 | Verify shop/blog sitemaps already found in repo are complete and correctly registered. | Updated sitemap classes |
| 15.1.2.2 | `robots.txt` review and environment-specific config | P1 | Easy | 20m | None | Block staging, allow prod, disallow admin/cart/checkout paths. | robots.txt |
| 15.1.2.3 | Server-side rendering or prerendering evaluation for SPA SEO | P1 | Hard | 2h (spike) | None | Since this is a Vite SPA, Google can crawl JS but investigate prerendering (e.g. via a pre-render service) for reliability; produce a recommendation. | Spike report/decision doc |

---

# EPIC 16 — Notifications (SMS / Email / Push)

## Phase 16.1 — Notification Infrastructure

### Feature 16.1.1 — Unified Notification System

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 16.1.1.1 | Create `notifications` app | P0 | Easy | 20m | None | App scaffold. | App |
| 16.1.1.2 | `NotificationTemplate` model (channel + event + Persian template text) | P1 | Medium | 1h | 16.1.1.1 | Admin-editable templates for order confirmation, OTP, shipping update, etc. | Model + migration |
| 16.1.1.3 | `NotificationLog` model (delivery tracking) | P1 | Easy | 30m | 16.1.1.1 | Records what was sent, to whom, channel, success/failure. | Model + migration |
| 16.1.1.4 | Unified `notify(user, event, context)` service | P0 | Medium | 1.5h | 16.1.1.2, 16.1.1.3 | Single entry point dispatching to correct channel(s) based on user preference + event type. | Service module |

## Phase 16.2 — SMS (Kavenegar)

### Feature 16.2.1 — Kavenegar Integration

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 16.2.1.1 | Implement `KavenegarSMSProvider` (real implementation of Epic 2's interface) | P0 | Medium | 1h | Epic 2 Feature 2.2.1 | Real API calls replacing the console dev backend in production settings. | Provider implementation |
| 16.2.1.2 | Order-confirmation SMS | P0 | Easy | 30m | 16.2.1.1, 16.1.1.4 | Triggered on order creation. | Template + trigger |
| 16.2.1.3 | Shipping-update SMS | P1 | Easy | 30m | 16.2.1.1, Epic 7 Feature 7.2.2 | Triggered on shipment status change. | Template + trigger |
| 16.2.1.4 | SMS delivery failure handling/retry | P1 | Medium | 1h | 16.2.1.1 | Retry once on transient failure, log permanent failures. | Retry logic |

## Phase 16.3 — Email

### Feature 16.3.1 — Transactional Email

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 16.3.1.1 | Persian-RTL HTML email templates (order confirmation, shipping, password reset) | P1 | Medium | 1.5h | 16.1.1.2 | RTL-safe email HTML (tables-based for client compatibility). | Template files |
| 16.3.1.2 | Wire email sending through Celery (async, not blocking request) | P0 | Medium | 1h | Epic 22 Phase 22.1 | Move existing synchronous email sends to background tasks. | Updated views/tasks |

## Phase 16.4 — In-App / Push (optional tier)

### Feature 16.4.1 — In-App Notifications

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 16.4.1.1 | `UserNotification` model for in-app bell/inbox | P2 | Medium | 1h | 16.1.1.1 | Persisted notifications viewable in customer dashboard. | Model + endpoint |
| 16.4.1.2 | Real-time delivery via existing Django Channels setup | P2 | Medium | 1.5h | 16.4.1.1 | Reuse the `chat` app's Channels infrastructure for a notifications consumer. | Consumer + frontend listener |
| 16.4.1.3 | Notification bell UI component | P2 | Medium | 1h | 16.4.1.2 | Unread badge, dropdown list. | React component |

---

# EPIC 17 — Admin Dashboard

## Phase 17.1 — Dashboard Data & Reporting

### Feature 17.1.1 — Core Metrics

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 17.1.1.1 | Revenue/order-count summary widget (existing `dashboard` app extension) | P1 | Medium | 1h | None | Extend existing `dashboard/services.py` with date-range revenue queries. | Service + endpoint |
| 17.1.1.2 | Top-selling products report | P1 | Medium | 1h | None | Aggregation query + endpoint. | Service + endpoint |
| 17.1.1.3 | Low-stock report widget | P1 | Easy | 30m | Epic 4 Feature 4.1.1 | Surfaces `StockMovement`/threshold data on dashboard home. | Widget component |
| 17.1.1.4 | Near-expiry inventory report widget | P2 | Easy | 30m | Epic 3 Feature 3.3.1 | Surfaces products nearing expiration. | Widget component |
| 17.1.1.5 | Coupon usage report | P2 | Easy | 45m | Epic 9 Feature 9.1.1 | Redemption counts/revenue impact per coupon. | Report view |

## Phase 17.2 — Admin UX

### Feature 17.2.1 — Bulk Operations

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 17.2.1.1 | Bulk product price update tool | P2 | Medium | 1.5h | None | Admin can adjust price by % across a filtered product set. | Endpoint + UI |
| 17.2.1.2 | Bulk product import via CSV/Excel | P1 | Hard | 2h | Epic 3 Feature 3.2.1 | Upload spreadsheet to create/update products with cosmetics attributes. | Import service + validation + UI |
| 17.2.1.3 | Bulk product export | P2 | Medium | 1h | 17.2.1.2 | Reverse of import for catalog backups/edits. | Export endpoint |

---

# EPIC 18 — Customer Dashboard

## Phase 18.1 — Self-Service Account Features

### Feature 18.1.1 — Account Pages

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 18.1.1.1 | Order history with status timeline | P0 | Medium | 1h | Epic 7 Feature 7.2.2 | Enhance existing `Account.jsx` order list with tracking timeline. | Component update |
| 18.1.1.2 | Downloadable invoice from order detail | P1 | Easy | 30m | Epic 8 Feature 8.1.1 | Link to the PDF invoice generation endpoint. | UI addition |
| 18.1.1.3 | Address book management UI | P1 | Medium | 1h | Epic 5 Feature 5.2.1 | CRUD for saved addresses. | React page |
| 18.1.1.4 | Notification preferences UI (SMS/email toggles) | P2 | Easy | 45m | Epic 16 Phase 16.1 | Extend existing `Profile` boolean fields (`order_updates`, `promotions`, `newsletter`) with a settings page. | React page |
| 18.1.1.5 | Reorder ("buy again") button on past orders | P2 | Easy | 45m | None | Re-adds all items from a past order to the cart. | Button + logic |

---

# EPIC 19 — Blog & Content

## Phase 19.1 — Blog Enhancements

### Feature 19.1.1 — Content Features

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 19.1.1.1 | Product tagging within blog posts (link posts to products) | P2 | Medium | 1h | None | M2M field, "shop this look" style linking. | Model + serializer update |
| 19.1.1.2 | Blog category/tag taxonomy | P2 | Easy | 45m | None | Verify/extend existing blog models for filterable categories. | Model additions |
| 19.1.1.3 | Related-products widget on blog post page | P2 | Easy | 45m | 19.1.1.1 | Frontend component. | Component |
| 19.1.1.4 | Blog RSS feed | P3 | Easy | 45m | None | Django syndication framework feed. | Feed class |

---

# EPIC 20 — Media Management

## Phase 20.1 — Image Pipeline

### Feature 20.1.1 — Optimization

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 20.1.1.1 | Add `django-imagekit` (or equivalent) for thumbnail generation | P0 | Medium | 1h | None | Auto-generate multiple sizes on upload (thumbnail/medium/full). | Package + config |
| 20.1.1.2 | Convert product images to WebP on upload | P1 | Medium | 1h | 20.1.1.1 | Reduce payload size for storefront images. | Processor config |
| 20.1.1.3 | Frontend responsive `<img srcset>` usage | P1 | Medium | 1.5h | 20.1.1.1 | Serve appropriately-sized images per viewport. | Component updates |
| 20.1.1.4 | Lazy-load below-the-fold images | P1 | Easy | 45m | None | `loading="lazy"` + intersection observer where needed. | Component updates |
| 20.1.1.5 | Enforce max upload size + file type validation on all image fields | P0 | Easy | 45m | None | Currently no explicit validation found — close this gap. | Validators |

---

# EPIC 21 — Caching (Redis)

## Phase 21.1 — Cache Layer

### Feature 21.1.1 — Django Cache Configuration

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 21.1.1.1 | Configure `CACHES` with `django-redis` (separate DB index from Channels) | P0 | Easy | 30m | None | Redis is currently only used for Channels — add a proper cache backend. | Settings update |
| 21.1.1.2 | Cache category tree endpoint | P1 | Easy | 30m | 21.1.1.1 | Rarely-changing data, cache with signal-based invalidation on Category save. | View decorator + signal |
| 21.1.1.3 | Cache homepage/product-list responses (short TTL) | P1 | Medium | 1h | 21.1.1.1 | Cache with query-param-aware keys, short TTL (60–300s). | Caching middleware/decorator |
| 21.1.1.4 | Cache invalidation on product/variant save | P0 | Medium | 1h | 21.1.1.2, 21.1.1.3 | Signal handlers clearing relevant cache keys on writes. | Signal handlers |
| 21.1.1.5 | Session backend moved to Redis | P2 | Easy | 20m | 21.1.1.1 | `SESSION_ENGINE` update for performance at scale. | Settings update |

---

# EPIC 22 — Celery & Async Tasks

## Phase 22.1 — Celery Setup

### Feature 22.1.1 — Task Queue Infrastructure

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 22.1.1.1 | Add Celery + `django-celery-beat` to requirements | P0 | Easy | 20m | None | Dependency install. | Requirements update |
| 22.1.1.2 | Create `core/celery.py` app config | P0 | Easy | 30m | 22.1.1.1 | Standard Django-Celery bootstrap using existing Redis broker. | Config file |
| 22.1.1.3 | Add Celery worker + beat services to `docker-compose.dev.yml`/`prod.yml` | P0 | Medium | 1h | 22.1.1.2 | New containers, sharing the existing Redis service. | Compose file updates |
| 22.1.1.4 | Health check / smoke-test task | P1 | Easy | 20m | 22.1.1.3 | Trivial task to confirm the pipeline works end-to-end in CI. | Task + CI step |

---

# EPIC 23 — Security Hardening

## Phase 23.1 — Application Security

### Feature 23.1.1 — Header & Policy Hardening

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 23.1.1.1 | Add Content-Security-Policy header (`django-csp`) | P1 | Medium | 1h | None | Restrict script/style/img sources. | Package + config |
| 23.1.1.2 | Add `django-axes` for login lockout after repeated failures | P0 | Medium | 1h | None | Prevent brute-force on password/OTP login. | Package + config |
| 23.1.1.3 | File upload validation (MIME sniffing, not just extension) | P0 | Medium | 1h | None | Verify actual file content for all `ImageField` uploads (products, categories, brands, profile, reviews). | Validator utility applied across models |
| 23.1.1.4 | Dependency vulnerability scanning in CI (`pip-audit`, `npm audit`) | P1 | Easy | 45m | None | New CI step failing build on high-severity CVEs. | CI workflow update |
| 23.1.1.5 | Secrets audit — confirm no credentials in repo/history | P0 | Medium | 1h | None | Scan git history for leaked keys; rotate anything found. | Audit report |
| 23.1.1.6 | Admin panel IP allowlist or extra-auth layer | P2 | Medium | 1h | None | Restrict `/admin/` and internal dashboard API to known ranges or add 2FA. | Middleware/config |

---

# EPIC 24 — Testing

## Phase 24.1 — Coverage Gaps

### Feature 24.1.1 — Critical-Path Test Coverage

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 24.1.1.1 | End-to-end checkout test (cart → payment → order confirmed) | P0 | Hard | 2h | Epic 6 Phase 6.2 | Full integration test mocking the payment gateway. | Test suite |
| 24.1.1.2 | Coupon abuse test suite (reuse, expired, over-cap) | P0 | Medium | 1h | Epic 9 Feature 9.1.1 | Adversarial test cases. | Test suite |
| 24.1.1.3 | OTP flow abuse tests (rate limit, expired, brute force) | P0 | Medium | 1h | Epic 2 Phase 2.3 | Adversarial test cases. | Test suite |
| 24.1.1.4 | Frontend E2E smoke test (Playwright/Cypress setup) | P1 | Hard | 2h | None | New tooling — currently only unit/component tests exist. | Test framework config + one smoke test |
| 24.1.1.5 | Load test script for product listing/checkout endpoints (k6/Locust) | P2 | Medium | 1.5h | None | Baseline performance benchmark before scale claims are trusted. | Load test script + report |

---

# EPIC 25 — DevOps / Docker / CI-CD

## Phase 25.1 — Environment Maturity

### Feature 25.1.1 — Staging & Deployment Safety

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 25.1.1.1 | Add `docker-compose.staging.yml` | P1 | Medium | 1h | None | Mirrors prod but points at staging DB/gateway sandboxes. | Compose file |
| 25.1.1.2 | CI: run migrations check (`makemigrations --check`) | P0 | Easy | 20m | None | Fail CI if model changes lack a migration. | CI step |
| 25.1.1.3 | CI: deploy-to-staging job on `develop` branch | P1 | Medium | 1.5h | 25.1.1.1 | Auto-deploy to staging on merge. | CI workflow update |
| 25.1.1.4 | Blue-green or rolling restart strategy for backend container | P2 | Hard | 2h | None | Avoid downtime on deploy (current setup restarts a single container). | Deploy script/compose update |
| 25.1.1.5 | Automated DB backup job (scheduled `pg_dump` to storage) | P0 | Medium | 1.5h | None | Currently no backup automation exists. | Cron/Celery task + storage config |
| 25.1.1.6 | Backup restore drill documentation + test | P1 | Medium | 1h | 25.1.1.5 | Prove backups are actually restorable. | Runbook doc |

---

# EPIC 26 — Monitoring & Logging

## Phase 26.1 — Observability

### Feature 26.1.1 — Error Tracking & Logs

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 26.1.1.1 | Integrate Sentry (backend + frontend) | P0 | Medium | 1h | None | Error tracking currently entirely absent. | SDK integration both sides |
| 26.1.1.2 | Structured JSON logging config | P1 | Medium | 1h | None | Replace default Django logging with structured, environment-aware config. | `LOGGING` settings update |
| 26.1.1.3 | Uptime monitoring for API + frontend (external ping service) | P1 | Easy | 30m | None | Third-party uptime check configuration. | Monitoring config |
| 26.1.1.4 | Payment-gateway callback failure alerting | P0 | Medium | 1h | Epic 6 Phase 6.2, 26.1.1.1 | Alert on repeated verification failures — money-critical path. | Alert rule |

---

# EPIC 27 — Analytics

## Phase 27.1 — Product & Business Analytics

### Feature 27.1.1 — Tracking Setup

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 27.1.1.1 | Integrate a privacy-compliant analytics tool (e.g. self-hosted Plausible/Matomo, or GA4) | P1 | Medium | 1h | None | Decision + integration. | Tracking snippet + event config |
| 27.1.1.2 | E-commerce event tracking (view_item, add_to_cart, purchase) | P1 | Medium | 1.5h | 27.1.1.1 | Standard e-commerce funnel events. | Event instrumentation |
| 27.1.1.3 | Admin conversion-funnel dashboard widget | P2 | Medium | 1h | 27.1.1.2 | Cart→checkout→purchase drop-off visualization. | Dashboard widget |

---

# EPIC 28 — Marketing Tools

## Phase 28.1 — Growth Features

### Feature 28.1.1 — Campaign Tools

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 28.1.1.1 | Newsletter signup capture (reuse `Profile.newsletter` flag) | P2 | Easy | 45m | None | Standalone signup form + endpoint for non-account visitors. | Model + endpoint + form |
| 28.1.1.2 | Abandoned-cart recovery email/SMS (Celery scheduled) | P1 | Hard | 2h | Epic 22 Phase 22.1, Epic 16 Phase 16.2 | Detect carts idle >N hours with items, send reminder. | Scheduled task + template |
| 28.1.1.3 | Referral/promo-code-on-signup flow | P3 | Medium | 1.5h | Epic 9 Feature 9.1.1 | New-user coupon issuance. | Logic + endpoint |

---

# EPIC 29 — Frontend Architecture Refactor

## Phase 29.1 — Performance Refactor

### Feature 29.1.1 — Code Splitting

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 29.1.1.1 | Convert all route imports in `App.jsx` to `React.lazy()` | P0 | Medium | 1.5h | None | Currently every page (storefront + admin) is eagerly imported. | Refactored `App.jsx` + `Suspense` boundaries |
| 29.1.1.2 | Split admin bundle from storefront bundle | P0 | Medium | 1h | 29.1.1.1 | Ensure admin code never ships to storefront visitors. | Vite chunking config |
| 29.1.1.3 | Add route-level loading skeletons | P2 | Easy | 1h | 29.1.1.1 | Replace blank-screen flash during lazy load. | Skeleton components |

### Feature 29.1.2 — Data Fetching Layer

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 29.1.2.1 | Introduce React Query (TanStack Query) | P1 | Medium | 1.5h | None | Replace manual `useEffect` + `axios` fetching with caching/retry-aware hooks. | Package + provider setup |
| 29.1.2.2 | Migrate product listing/PDP fetches to React Query | P1 | Medium | 1.5h | 29.1.2.1 | Incremental migration, highest-traffic pages first. | Updated components |
| 29.1.2.3 | Migrate cart/order fetches to React Query | P1 | Medium | 1.5h | 29.1.2.1 | Same for account-area pages. | Updated components |

### Feature 29.1.3 — Forms

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 29.1.3.1 | Introduce `react-hook-form` + `zod` schema validation | P1 | Medium | 1.5h | None | Standardize form handling/validation across checkout, auth, admin forms. | Package + shared schema utilities |
| 29.1.3.2 | Migrate checkout form | P0 | Medium | 1.5h | 29.1.3.1, Epic 5 Feature 5.2.1 | Highest-value form to get right. | Updated component |
| 29.1.3.3 | Migrate auth forms (OTP, login) | P0 | Medium | 1h | 29.1.3.1, Epic 2 Feature 2.3.2 | Consistent validation UX. | Updated components |

---

# EPIC 30 — Documentation

## Phase 30.1 — Team-Readiness Docs

### Feature 30.1.1 — Engineering Documentation

| ID | Title | Priority | Difficulty | Time | Dependencies | Description | Deliverable |
|---|---|---|---|---|---|---|---|
| 30.1.1.1 | Architecture Decision Records (ADR) folder + backfill key decisions | P1 | Medium | 1.5h | None | Document currency-unit choice, slug strategy, gateway choice, etc. from this backlog. | `/docs/adr/*.md` |
| 30.1.1.2 | API documentation review (drf-spectacular schema completeness) | P1 | Medium | 1h | None | Repo already has `drf-spectacular`/`drf-yasg` — audit that all new endpoints are documented. | Schema annotations |
| 30.1.1.3 | Local dev setup guide update (OTP dev mode, Celery, new env vars) | P0 | Easy | 45m | All Epics | Keep `README.md` current as new services (Celery, payments sandbox, SMS dev backend) are added. | Updated README |
| 30.1.1.4 | Runbook: payment gateway outage procedure | P1 | Easy | 45m | Epic 6 | What to do if ZarinPal is down — fallback gateway, customer comms. | Runbook doc |
| 30.1.1.5 | Runbook: incident response (Sentry alert → triage → resolve) | P2 | Easy | 45m | Epic 26 | Standard on-call procedure doc. | Runbook doc |

---

## Execution Order Summary

The Epics are numbered in the order they should be tackled. Rationale for sequencing:

1. **Epic 1 (Core Stability)** — must land first; every other epic builds on order/inventory/pricing that is currently unsafe.
2. **Epic 2 (Auth/OTP)** — the Iranian market requires phone auth before real customers can use the site; also unblocks guest→user cart merge logic used later.
3. **Epic 3 (Product/Variants/Cosmetics fields)** — the data model change is foundational; Cart, Order, Search, and Admin all depend on variants existing.
4. **Epic 4 (Inventory)** — depends on variants; needed before real stock-sensitive traffic.
5. **Epic 5 (Cart/Checkout)** — depends on variants + auth; guest cart is a conversion-critical gap.
6. **Epic 6 (Payments)** — the single biggest "not a real store yet" gap; depends on Epic 1's atomic order logic and Epic 5's checkout hardening.
7. **Epic 7 (Shipping)** — depends on checkout being real; needed before launch but can be built in parallel with late Payments work.
8. **Epic 8 (Order Management/Admin)** — depends on 1, 6, 7 being functional.
9. **Epic 9 (Coupons)** — depends on Epic 1's pricing-service extraction and closes the discount vulnerability with a real feature.
10. **Epic 10 (Reviews)**, **Epic 11 (Wishlist)** — lower-risk, can run in parallel with 6–9 once Epic 1/3 land.
11. **Epic 12 (Search)**, **Epic 13 (Recommendations)** — depend on catalog (Epic 3) being finalized.
12. **Epic 14 (Localization)** — large and mostly independent technically, but should be scheduled *before* public launch and ideally in parallel with Epics 3–9 since it touches almost every layer (currency migration in particular should happen early, before more monetary fields are added elsewhere).
13. **Epic 15 (SEO)** — depends on localization (Persian slugs/content) being in place.
14. **Epic 16 (Notifications)** — depends on Epic 2 (OTP SMS reuse) and feeds into Epics 8, 9, 4.
15. **Epic 17/18 (Admin & Customer Dashboards)** — consume data from most epics above; scheduled after core domains stabilize.
16. **Epic 19 (Blog)**, **Epic 20 (Media)** — can run anytime after Epic 3, lower urgency.
17. **Epic 21 (Caching)**, **Epic 22 (Celery)** — introduce once there's enough real traffic/async work (payment reconciliation, notifications, expiry sweeps) to justify them — but Celery should be stood up early (Phase 22.1) since Epics 6, 16, and 28 all depend on it.
18. **Epic 23 (Security Hardening)** — ongoing, but the P0 items (throttling, file validation, axes) should be pulled forward alongside Epic 1–2, not left until the end.
19. **Epic 24 (Testing)** — continuous, but the critical-path tests should be written as each dependent epic (payments, coupons, OTP) ships, not batched at the end.
20. **Epic 25 (DevOps)**, **Epic 26 (Monitoring)** — needed before any real launch; schedule to finish before go-live, in parallel with the later feature epics.
21. **Epic 27 (Analytics)**, **Epic 28 (Marketing)** — post-launch growth work.
22. **Epic 29 (Frontend Refactor)** — the code-splitting/data-fetching work (29.1.1) should actually be pulled forward and done early/incrementally as new pages are added, rather than as a big-bang refactor at the end; listed here as the epic where it's *completed*, not where it *starts*.
23. **Epic 30 (Documentation)** — continuous throughout, formalized at the end of each major epic.

**Practical note:** several "late" epics (Celery infra, throttling, Sentry, code-splitting) have components that are cheap to do early and expensive to retrofit. Treat the epic numbering as topic grouping, not a strict "don't start until previous epic is 100% done" gate — a small cross-functional team should pull P0 items from Epics 21–26 forward whenever they unblock a P0 item in an earlier epic.

---

*Total backlog: 30 Epics, 60+ Phases, 100+ Features, 260+ individually estimated tasks. This document is the single source of truth for sprint planning; each task is intentionally scoped to convert directly into one ticket (and, in the next step, one AI coding prompt) without further breakdown.*
