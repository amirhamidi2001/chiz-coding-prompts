# Epic 17 — Admin Dashboard — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–16 are fully merged, including Epic 14's integer-Rial currency migration (Task 14.1.2.2) and Epic 9's coupon system.

**Important grounded discovery for this epic — read before starting Feature 17.1.1:** `backend/dashboard/services.py` **already contains a remarkably complete analytics layer** — `get_admin_overview()`, `get_order_status_distribution()`, `get_monthly_revenue()`, `get_top_products()`, `get_user_stats()`, `get_product_stats()`, `get_recent_orders()`, and `get_user_summary()` all already exist and are wired into dashboard endpoints. This epic's Feature 17.1.1 tasks are therefore **audit-and-extend** tasks in several cases, not greenfield builds — but there is a **real, currently-breaking bug** these functions will have picked up from Epic 14's currency migration that must be fixed **before** anything else in this Feature: several of these functions do money arithmetic assuming `Order.total`/`OrderItem.unit_price` are still `DecimalField`s (e.g. `get_top_products()` explicitly declares `output_field=DecimalField()` on a `Sum()` aggregate, and multiple functions default missing sums to `Decimal("0")` via `Coalesce`) — but per Epic 14 Task 14.1.2.2, these fields are now `PositiveBigIntegerField`/`BigIntegerField` (plain integers), **not** `Decimal`. Mixing an integer database column with a `Decimal("0")` Python default in a `Coalesce`/aggregate can raise a real database-level type error in Postgres, or silently produce `Decimal`-typed output masking what should now be plain integer Rial values — this is exactly the kind of migration-fallout bug Epic 14's own header warned would ripple outward, and Task 17.1.1.1 below is where it gets caught and fixed for this specific file.

**Also confirmed:** `AdminProductSerializer`/`AdminProductViewSet` (`backend/dashboard/serializers.py`/`views.py`) **still expose/operate on `Product.price`/`Product.stock` directly** — a known, previously-flagged gap (noted back in Epic 4's grounding) that was never fully resolved: real pricing/stock now lives on `ProductVariant` (Epic 3), but this admin serializer never migrated to reflect that. Feature 17.2.1's bulk-operation tasks below **must** operate on the real, current `ProductVariant` data, not blindly build on top of this stale serializer.

---

## Phase 17.1 — Dashboard Data & Reporting

### Feature 17.1.1 — Core Metrics

---

#### Task 17.1.1.1 — Revenue/order-count summary widget (existing `dashboard` app extension)

```
You are working in backend/dashboard/services.py. Assume Epics 1–16
are fully merged, including Epic 14's integer-Rial currency migration.

CONTEXT — READ THIS DOCUMENT'S HEADER BEFORE STARTING
`get_admin_overview()` already computes current-vs-previous-period
revenue, order count, new-user count, and a daily revenue chart —
this is NOT a greenfield build. Your FIRST job in this task is fixing
the currency-type breakage this file inherited from Epic 14, BEFORE
extending anything.

TASK
1. Audit and fix every money-arithmetic call in `dashboard/services.py`
   for correctness against Epic 14's integer-Rial fields.
2. Extend `get_admin_overview()`'s date-range options and add a
   dedicated, focused revenue/order-count summary endpoint if the
   existing one doesn't already cleanly serve as one (it likely does —
   confirm before adding a redundant second endpoint).

REQUIREMENTS — Part 1: the currency-type fix
- Change every `Coalesce(Sum("total"), Decimal("0"))`-style call (and
  the equivalent in `get_monthly_revenue()`, `get_user_summary()`) to
  `Coalesce(Sum("total"), 0)` — a plain Python `int` default, matching
  the now-integer field type. Search this file exhaustively for every
  occurrence of `Decimal(` and evaluate each one: is it still
  legitimately needed (e.g. `_pct_change()`'s `float(current or 0)`
  casting is fine and doesn't need to change), or is it a leftover
  assumption from the pre-Epic-14 Decimal-money scheme that needs to
  become a plain `int`?
- Fix `get_top_products()`'s explicit
  `output_field=DecimalField()` on its revenue `Sum(F("unit_price") * F("quantity"), ...)`
  aggregate — change to `output_field=BigIntegerField()` (or omit
  `output_field` entirely if Django can correctly infer it from the
  now-integer `unit_price`/`quantity` fields being multiplied — test
  both and use whichever is more robust/correct). Import
  `BigIntegerField` from `django.db.models` if needed.
- Every place in this file currently doing `float(some_money_value)`
  before returning it in a response dict (e.g. `"current": float(cur_rev)`)
  — reconsider whether `float()` is still the right conversion. Per
  Epic 14's currency-storage decision, these values are now integer
  RIAL amounts; converting to Python `float` for JSON serialization is
  still reasonable (JSON has no native Decimal/BigInt-safe type
  distinct from float for values within JS's safe integer range,
  which Rial amounts should comfortably stay within for realistic
  order/revenue totals) — but explicitly confirm this holds and isn't
  silently introducing floating-point imprecision for values that
  should stay exact integers; if there's any risk of the frontend
  needing EXACT integer precision (e.g. for further arithmetic rather
  than pure display), consider returning `int()` instead of `float()`
  for these Rial-amount fields specifically, reserving `float()` for
  genuinely fractional values like percentage changes
  (`_pct_change()`'s return value, which correctly stays a float
  since it's a percentage, not a currency amount).
- Re-run this fix against a REAL Postgres database (not a mock) with
  representative order data to confirm the aggregate queries actually
  execute without raising a type-mismatch error — this is exactly the
  kind of bug that can pass a superficial code review but fail at
  actual query-execution time against Postgres's stricter type
  checking.

REQUIREMENTS — Part 2: extend for admin dashboard UX
- Confirm `get_admin_overview(period)`'s existing `"7d"/"30d"/"90d"/"1y"`
  period options are sufficient, or add a custom date-range option
  (explicit `start`/`end` dates rather than only named presets) if the
  admin frontend needs it — check whether the EXISTING admin dashboard
  frontend page (find it — likely already built out to SOME degree
  given how complete the backend service layer already is) already has
  a date-range picker UI expecting this capability; if so, wire it in
  server-side; if the frontend doesn't have this yet either, this is a
  reasonable, in-scope small addition given the task's explicit mention
  of "date-range revenue queries."
- Add a `custom` period option to `get_date_range()`:
  ```python
  def get_date_range(period: str, start_param=None, end_param=None):
      if period == "custom" and start_param and end_param:
          return start_param, end_param
      end = timezone.now()
      mapping = {"7d": 7, "30d": 30, "90d": 90, "1y": 365}
      days = mapping.get(period, 30)
      return end - timedelta(days=days), end
  ```
  Wire the corresponding admin overview view/endpoint to accept
  `?period=custom&start=...&end=...` query params, parsing them
  correctly (validate they're well-formed dates, `start < end`, and
  return a clear 400 error otherwise rather than crashing on
  malformed input).

ACCEPTANCE CRITERIA / TESTS
Add/update tests in backend/dashboard/tests/test_services.py:
1. `get_admin_overview()`, `get_top_products()`, `get_monthly_revenue()`,
   `get_user_summary()` all execute successfully against a REAL test
   Postgres database (not mocked) with realistic post-Epic-14 integer
   order/order-item data, producing correct, verifiable numeric
   results — this is the actual regression proof the type-mismatch bug
   is fixed, run against the real database engine, not just "the code
   doesn't raise a Python exception."
2. Revenue/total figures in every function's output are correct integer
   Rial amounts (spot-check exact expected values against constructed
   test orders), not silently coerced to unexpected types.
3. The new `custom` period option correctly returns data for an
   explicit start/end range; malformed/missing start/end params with
   `period=custom` return a clear validation error, not a crash.
```

---

#### Task 17.1.1.2 — Top-selling products report

```
You are working in backend/dashboard/services.py, views.py. Assume
Task 17.1.1.1 is already merged (the currency-type fixes, which
directly affect this exact function).

CONTEXT
`get_top_products()` already exists and, per Task 17.1.1.1, has had its
currency-arithmetic type bug fixed. This task extends it: currently it
groups by `product_id`/`product_name` (the frozen `OrderItem` snapshot
fields, still valid post-Epic-3) but has no VARIANT-level breakdown at
all — an admin looking at "top products" today can't tell WHICH shade/
size of a top-selling product is actually driving those sales, which
matters a lot for a cosmetics platform's actual restocking/merchandising
decisions.

TASK
Extend the top-products report with variant-level detail and a
date-range filter (currently `get_top_products()` has no date
filtering at all — it aggregates across ALL-TIME order history
unconditionally, which is a real, separate gap worth closing here too).

REQUIREMENTS
- Add date-range filtering to `get_top_products()`:
  ```python
  def get_top_products(limit: int = 10, start=None, end=None) -> list:
      qs = OrderItem.objects.all()
      if start and end:
          qs = qs.filter(order__created_at__range=[start, end])
      rows = (
          qs.values("product_id", "product_name")
          .annotate(
              total_sold=Sum("quantity"),
              revenue=Coalesce(Sum(F("unit_price") * F("quantity")), 0),
          )
          .order_by("-total_sold")[:limit]
      )
      ...
  ```
  Wire the admin view/endpoint calling this to accept the same
  `?period=`/custom-range query params established in Task 17.1.1.1,
  for consistency across the dashboard's reporting endpoints.
- Add a SEPARATE, variant-level breakdown function:
  ```python
  def get_top_variants(limit: int = 10, start=None, end=None) -> list:
      qs = OrderItem.objects.exclude(variant_sku="")
      if start and end:
          qs = qs.filter(order__created_at__range=[start, end])
      rows = (
          qs.values("variant_sku", "product_name", "variant_attributes_json")
          .annotate(
              total_sold=Sum("quantity"),
              revenue=Coalesce(Sum(F("unit_price") * F("quantity")), 0),
          )
          .order_by("-total_sold")[:limit]
      )
      return [
          {
              "variant_sku": r["variant_sku"],
              "product_name": r["product_name"],
              "variant_attributes": r["variant_attributes_json"],
              "total_sold": r["total_sold"],
              "revenue": r["revenue"],
          }
          for r in rows
      ]
  ```
  (using the frozen `OrderItem.variant_sku`/`variant_attributes_json`
  snapshot fields, per Epic 3 Task 3.1.1.4 — these correctly reflect
  what was ACTUALLY sold historically even if the variant itself was
  later renamed/deleted, exactly the same "trust the snapshot, not
  live data" principle already established for invoicing in Epic 8).
- Add both as separate admin-dashboard API endpoints (or query-param-
  toggle on one existing endpoint — e.g. `?breakdown=variant` vs. the
  default product-level — your call, but pick whichever fits the
  existing dashboard endpoint conventions more naturally).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/test_services.py:
1. `get_top_products()` with a date range correctly excludes orders
   outside that range (construct orders both inside and outside a test
   range and confirm only the in-range ones contribute to the
   aggregate).
2. `get_top_variants()` correctly aggregates by variant SKU, not just
   product, distinguishing sales of different shades/sizes of the same
   underlying product.
3. `get_top_variants()` excludes `OrderItem` rows with a blank
   `variant_sku` (pre-Epic-3 historical orders, if any exist, or any
   edge case where the snapshot wasn't populated).
4. Revenue figures in both functions are correct integer Rial amounts.
```

---

#### Task 17.1.1.3 — Low-stock report widget

```
You are working in backend/dashboard/services.py, views.py. Assume
Epic 4 Task 4.1.1.5 is already merged (which already fixed
`get_admin_overview()`'s/`get_product_stats()`'s `low_stock`/
`out_of_stock` COUNTS to correctly query `ProductVariant` instead of
the stale `Product.stock` field).

CONTEXT
Epic 4 already fixed the low-stock/out-of-stock SUMMARY COUNTS shown on
the dashboard overview — but a count alone ("12 items low on stock") is
of limited operational use without a way to see WHICH 12 items and act
on them. There's currently no LIST endpoint an admin can actually click
into from that summary number.

TASK
Add a dedicated, actionable low-stock LIST endpoint/report.

REQUIREMENTS
- Add `get_low_stock_variants(limit: int = 50) -> list` to
  `dashboard/services.py`:
  ```python
  def get_low_stock_variants(limit: int = 50) -> list:
      from shop.models import ProductVariant
      from django.db.models import F

      variants = (
          ProductVariant.objects.filter(
              is_active=True, stock__gt=0, stock__lte=F("low_stock_threshold")
          )
          .select_related("product", "color")
          .order_by("stock")[:limit]
      )
      return [
          {
              "variant_id": v.id,
              "sku": v.sku,
              "product_name": v.product.name,
              "color": v.color.name if v.color else None,
              "stock": v.stock,
              "low_stock_threshold": v.low_stock_threshold,
          }
          for v in variants
      ]


  def get_out_of_stock_variants(limit: int = 50) -> list:
      from shop.models import ProductVariant
      variants = (
          ProductVariant.objects.filter(is_active=True, stock=0)
          .select_related("product", "color")
          .order_by("-updated_at")[:limit]
      )
      return [
          {
              "variant_id": v.id, "sku": v.sku, "product_name": v.product.name,
              "color": v.color.name if v.color else None,
          }
          for v in variants
      ]
  ```
  (reusing exactly the `low_stock_threshold`/`is_low_stock` fields
  already established on `ProductVariant` in Epic 4 Task 4.1.1.5 — this
  task is purely about surfacing that existing data as an actionable
  list, not inventing new stock-tracking concepts).
- Add corresponding admin API endpoints
  (`GET /dashboard/admin/inventory/low-stock/`,
  `GET /dashboard/admin/inventory/out-of-stock/`), and link each
  returned `variant_id` such that the admin frontend can navigate
  directly to that variant's edit form (in the `ProductVariantInline`/
  `ProductVariantAdmin`, per Epic 3 Task 3.1.1.6, or the equivalent
  React admin product-edit page if one exists by this point) to restock
  it — a low-stock report that doesn't link directly to the fix action
  is meaningfully less useful than one that does.

ACCEPTANCE CRITERIA / TESTS
Add tests confirming:
1. `get_low_stock_variants()` returns only variants genuinely below
   their OWN configured `low_stock_threshold` (not a hardcoded global
   number), sorted by stock ascending (most urgent first).
2. `get_out_of_stock_variants()` returns only `stock=0`, `is_active=True`
   variants — a deactivated (per Epic 3's expiry-sweep or manual
   deactivation) variant with 0 stock is correctly EXCLUDED (it's not
   actionable/restockable in the same way an active out-of-stock item
   is).
3. Both endpoints are admin-only (403 for non-admin).
```

---

#### Task 17.1.1.4 — Near-expiry inventory report widget

```
You are working in backend/dashboard/services.py, views.py. Assume
Epic 3 Task 3.3.1.1's `NearExpiryFilter` (Django admin) already exists.

CONTEXT
Epic 3 Task 3.3.1.1 added a Django-ADMIN-only near-expiry filter (a
`SimpleListFilter` on `ProductVariantAdmin`) — useful for staff working
directly in Django's raw admin, but not exposed via the `dashboard`
app's REST API at all, meaning it's invisible to whatever React admin
dashboard frontend consumes that API (the same one Feature 17.1.1's
other widgets feed into).

TASK
Add a near-expiry inventory report to the `dashboard` app's REST API,
mirroring the SAME query logic as Epic 3's admin filter but exposed as
dashboard data.

REQUIREMENTS
- Add `get_near_expiry_variants(days: int = 90, limit: int = 50) -> list`
  to `dashboard/services.py`:
  ```python
  def get_near_expiry_variants(days: int = 90, limit: int = 50) -> list:
      from datetime import timedelta
      from shop.models import ProductVariant

      today = timezone.now().date()
      variants = (
          ProductVariant.objects.filter(
              is_active=True,
              expiration_date__isnull=False,
              expiration_date__gte=today,
              expiration_date__lte=today + timedelta(days=days),
          )
          .select_related("product", "color")
          .order_by("expiration_date")[:limit]
      )
      return [
          {
              "variant_id": v.id, "sku": v.sku, "product_name": v.product.name,
              "color": v.color.name if v.color else None,
              "expiration_date": v.expiration_date.isoformat(),
              "days_until_expiry": (v.expiration_date - today).days,
              "stock": v.stock,
          }
          for v in variants
      ]
  ```
  Make `days` a query-parameter-configurable threshold on the
  corresponding endpoint (`GET /dashboard/admin/inventory/near-expiry/?days=30`),
  not hardcoded, since different admin workflows may want different
  urgency windows (a 30-day list for "act now" vs. a 90-day list for
  general awareness).
- Also add an EXPIRED (already past `expiration_date`, still
  `is_active=True`) variant report — this represents genuinely urgent,
  actionable data distinct from "near expiry": stock that's ALREADY
  expired but hasn't yet been caught by Epic 3 Task 3.3.1.3's nightly
  Celery sweep (or was deliberately reactivated, or the sweep task
  failed to run for some reason) — an admin seeing this list is a
  useful safety net independent of whether the automated sweep is
  working correctly:
  ```python
  def get_expired_still_active_variants(limit: int = 50) -> list:
      today = timezone.now().date()
      variants = (
          ProductVariant.objects.filter(is_active=True, expiration_date__lt=today)
          .select_related("product", "color")
          .order_by("expiration_date")[:limit]
      )
      return [...]  # same shape as above
  ```
  This is a genuinely valuable operational cross-check — if this list
  is ever non-empty, it likely indicates the Celery expiry-sweep task
  (Epic 3 Task 3.3.1.3) isn't running correctly, which is itself
  worth surfacing prominently (consider a visually distinct "warning"
  treatment on the frontend for this specific report vs. the routine
  near-expiry one).

ACCEPTANCE CRITERIA / TESTS
Add tests confirming:
1. `get_near_expiry_variants()` correctly includes variants within the
   configured day window and excludes ones outside it (both too-soon-
   to-worry-about-yet and already-expired cases — already-expired ones
   belong in the SEPARATE `get_expired_still_active_variants()` report,
   not this one).
2. `get_expired_still_active_variants()` correctly finds variants past
   their expiration date that are STILL somehow `is_active=True`
   (simulate the Celery sweep NOT having run yet).
3. Both correctly exclude variants with `expiration_date=None` (no
   expiry data at all — not "near expiry," just unknown).
4. The `days` query parameter correctly changes the near-expiry
   window's results.
```

---

#### Task 17.1.1.5 — Coupon usage report

```
You are working in backend/dashboard/services.py, views.py. Assume
Epic 9's `Coupon`/`CouponRedemption` models are already merged.

CONTEXT
No admin-facing report exists showing coupon PERFORMANCE (redemption
counts, total discount given, revenue impact) — Epic 9 Task 9.1.1.8
built basic admin CRUD for coupons (create/edit/deactivate), but
nothing analytical showing HOW a coupon actually performed once live.

TASK
Add a coupon usage/performance report.

REQUIREMENTS
- Add `get_coupon_usage_report(limit: int = 50) -> list` to
  `dashboard/services.py`:
  ```python
  def get_coupon_usage_report(limit: int = 50) -> list:
      from promotions.models import Coupon, CouponRedemption
      from django.db.models import Count, Sum

      coupons = (
          Coupon.objects.annotate(
              redemption_count=Count("redemptions", filter=Q(redemptions__order__isnull=False)),
              total_discount_given=Coalesce(
                  Sum("redemptions__discount_amount", filter=Q(redemptions__order__isnull=False)), 0
              ),
          )
          .order_by("-redemption_count")[:limit]
      )
      return [
          {
              "coupon_id": c.id,
              "code": c.code,
              "discount_type": c.discount_type,
              "redemption_count": c.redemption_count,
              "max_uses": c.max_uses,
              "usage_rate_pct": (
                  round((c.redemption_count / c.max_uses) * 100, 1) if c.max_uses else None
              ),
              "total_discount_given": c.total_discount_given,
              "is_active": c.is_active,
              "valid_until": c.valid_until.isoformat(),
          }
          for c in coupons
      ]
  ```
  Import `Q` from `django.db.models` at the top of services.py if not
  already imported. Note the `filter=Q(redemptions__order__isnull=False)`
  on both annotations — mirroring the exact same "only count COMPLETED
  redemptions, not in-progress cart-level applications" principle
  already established in Epic 9 Task 9.1.1.4's `validate_coupon()`
  usage-limit checks; a coupon report double-counting abandoned-cart
  coupon applications as real "usage" would give admins a misleading
  picture of actual campaign performance.
  `usage_rate_pct` is `None` for coupons with no configured `max_uses`
  (unlimited coupons don't have a meaningful "rate" — dividing by
  `None`/zero would be nonsensical) — the frontend should render this
  as "Unlimited" or similar rather than a percentage in that case.
- Also add a revenue-impact figure: total subtotal of orders where a
  given coupon was applied, alongside the discount given, so an admin
  can see "this coupon drove X in gross order value at a cost of Y in
  discounts" — a genuinely useful business metric beyond raw redemption
  counts:
  ```python
  total_order_value=Coalesce(
      Sum("redemptions__order__subtotal", filter=Q(redemptions__order__isnull=False)), 0
  ),
  ```
  Add this to the same annotation/output dict above.
- Add a corresponding admin endpoint:
  `GET /dashboard/admin/promotions/coupon-usage/`.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/test_services.py:
1. A coupon with 3 completed redemptions shows the correct
   `redemption_count`, `total_discount_given` (sum of the actual
   `discount_amount` from each redemption), and `total_order_value`.
2. An in-progress redemption (`order=None`, per Epic 9's cart-applied-
   but-not-yet-checked-out state) does NOT count toward any of these
   figures.
3. A coupon with `max_uses=None` (unlimited) correctly returns
   `usage_rate_pct: None`, not a division error.
4. A coupon with `max_uses` set correctly computes the percentage.
5. Results are correctly ordered by redemption count descending (most-
   used coupons first).
```

---

## Phase 17.2 — Admin UX

### Feature 17.2.1 — Bulk Operations

---

#### Task 17.2.1.1 — Bulk product price update tool

```
You are working in backend/dashboard/views.py, serializers.py, urls.py.
Assume Epics 1–16 are fully merged. RE-READ this document's header
warning about `AdminProductSerializer`'s staleness before starting —
this task must NOT build on top of that stale Product-level price
assumption.

CONTEXT — A REAL, PREVIOUSLY-FLAGGED GAP THIS TASK MUST NOT REPEAT
`AdminProductSerializer` still exposes `price`/`stock` as if they lived
directly on `Product` — but since Epic 3, the REAL, authoritative price
data lives on `ProductVariant`. A bulk price-update tool built naively
against `Product.price` would be adjusting a field that's been
functionally dead/unused in the actual storefront/checkout flow since
Epic 3 — a serious, silent correctness bug if not caught here.

TASK
Add an admin tool letting staff adjust prices by a percentage (or fixed
amount) across a FILTERED SET of `ProductVariant`s, correctly operating
on the real, current pricing data.

REQUIREMENTS
- Create `BulkPriceUpdateSerializer` in backend/dashboard/serializers.py:
  ```python
  class BulkPriceUpdateSerializer(serializers.Serializer):
      class AdjustmentType(models.TextChoices):
          PERCENT_INCREASE = "percent_increase", "Percentage Increase"
          PERCENT_DECREASE = "percent_decrease", "Percentage Decrease"
          FIXED_INCREASE = "fixed_increase", "Fixed Rial Increase"
          FIXED_DECREASE = "fixed_decrease", "Fixed Rial Decrease"

      adjustment_type = serializers.ChoiceField(choices=AdjustmentType.choices)
      value = serializers.IntegerField(min_value=1)  # percent points (1-100) or Rial amount, depending on type
      category_id = serializers.IntegerField(required=False, allow_null=True)
      brand_id = serializers.IntegerField(required=False, allow_null=True)
      variant_ids = serializers.ListField(
          child=serializers.IntegerField(), required=False, allow_empty=True,
      )
      dry_run = serializers.BooleanField(default=True)

      def validate(self, attrs):
          if not (attrs.get("category_id") or attrs.get("brand_id") or attrs.get("variant_ids")):
              raise serializers.ValidationError(
                  "Specify at least one of category_id, brand_id, or variant_ids to scope the update."
              )
          if attrs["adjustment_type"] == "percent_increase" and attrs["value"] > 100:
              raise serializers.ValidationError({"value": "Percentage adjustment cannot exceed 100."})
          return attrs
  ```
  (importing `models` from `django.db` at the top if not already
  imported, for the nested `TextChoices` — or move `AdjustmentType` to
  module level in `dashboard/models.py` if this project's convention
  prefers not nesting `TextChoices` inside a plain `Serializer` class;
  check precedent from prior epics and match it).
  Note the DELIBERATE `dry_run=True` DEFAULT — a bulk price mutation
  across potentially hundreds of variants is exactly the kind of
  operation that should default to SAFE preview mode, requiring an
  explicit, deliberate `dry_run=false` to actually commit — this is a
  meaningful safety design choice, not incidental.
- Add `BulkPriceUpdateView`:
  ```python
  class BulkPriceUpdateView(APIView):
      permission_classes = [IsAdminOrSuperuser]

      def post(self, request):
          serializer = BulkPriceUpdateSerializer(data=request.data)
          serializer.is_valid(raise_exception=True)
          data = serializer.validated_data

          qs = ProductVariant.objects.filter(is_active=True)
          if data.get("category_id"):
              qs = qs.filter(product__category_id=data["category_id"])
          if data.get("brand_id"):
              qs = qs.filter(product__brand_id=data["brand_id"])
          if data.get("variant_ids"):
              qs = qs.filter(id__in=data["variant_ids"])

          preview = []
          for variant in qs.select_related("product"):
              new_price = self._apply_adjustment(variant.price, data["adjustment_type"], data["value"])
              preview.append({
                  "variant_id": variant.id, "sku": variant.sku, "product_name": variant.product.name,
                  "old_price": variant.price, "new_price": new_price,
              })

          if data["dry_run"]:
              return Response({"dry_run": True, "affected_count": len(preview), "preview": preview[:50]})

          with transaction.atomic():
              for item in preview:
                  ProductVariant.objects.filter(pk=item["variant_id"]).update(price=item["new_price"])
          return Response({"dry_run": False, "updated_count": len(preview)})

      def _apply_adjustment(self, price: int, adjustment_type: str, value: int) -> int:
          if adjustment_type == "percent_increase":
              return max(0, round(price * (100 + value) / 100))
          if adjustment_type == "percent_decrease":
              return max(0, round(price * (100 - value) / 100))
          if adjustment_type == "fixed_increase":
              return price + value
          if adjustment_type == "fixed_decrease":
              return max(0, price - value)
          return price
  ```
  Import `transaction` from `django.db`, `ProductVariant` from
  `shop.models`. Note `price` arithmetic here operates on PLAIN
  INTEGERS throughout (per Epic 14's currency migration — no `Decimal`
  anywhere in this new code), and every price-decreasing path is
  floored at `max(0, ...)` to prevent an adjustment from ever producing
  a negative price.
  The preview response is CAPPED at 50 items shown (`preview[:50]`)
  even though `affected_count` reports the TRUE total — a bulk
  operation affecting thousands of variants shouldn't return a
  thousands-of-rows JSON preview payload; the count alone tells the
  admin the scale, and a capped sample confirms the adjustment LOGIC
  looks correct without a bloated response.
  Register the URL:
  `path("admin/products/bulk-price-update/", views.BulkPriceUpdateView.as_view(), name="admin-bulk-price-update"),`
- Log this operation for audit purposes — this is a real bulk financial
  change and deserves a record. Reuse `StockMovement`? No —
  `StockMovement` is specifically for STOCK changes, not price changes;
  a bulk price update doesn't naturally fit that model. Add a minimal
  `PriceAdjustmentLog` model (or reuse a more general audit-log pattern
  if one exists elsewhere by this point) recording: actor, adjustment
  parameters, affected variant count, timestamp — enough for an admin
  to later answer "who changed prices, when, and by how much" without
  needing to reconstruct it from scratch.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/test_views.py:
1. A `dry_run=true` request correctly PREVIEWS the price changes
   without actually modifying any `ProductVariant.price` in the
   database.
2. A `dry_run=false` request correctly APPLIES the changes, and a
   subsequent fetch of the affected variants confirms the new prices.
3. Each adjustment type (`percent_increase`/`percent_decrease`/
   `fixed_increase`/`fixed_decrease`) computes the mathematically
   correct new price, correctly floored at 0 for any decrease that
   would otherwise go negative.
4. Filtering by `category_id` only affects variants of products in that
   category; filtering by `variant_ids` only affects exactly those
   variants; requests with NONE of `category_id`/`brand_id`/
   `variant_ids` are rejected by the serializer's `validate()`.
5. A `PriceAdjustmentLog` (or equivalent) row is created recording the
   operation, with the correct actor and affected count.
6. A non-admin user gets 403.
```

---

#### Task 17.2.1.2 — Bulk product import via CSV/Excel

```
You are working in backend/dashboard/views.py, serializers.py, urls.py,
requirements.txt. Assume Epics 1–16 are fully merged, including every
cosmetics-specific `Product`/`ProductVariant` field from Epic 3.

CONTEXT
No bulk import capability exists — every product/variant currently has
to be created one at a time through the admin UI/API, which doesn't
scale for a real catalog launch or ongoing large-batch additions
(e.g. onboarding a new brand's full product line at once).

TASK
Add a CSV/Excel bulk product+variant import tool, correctly handling
every cosmetics attribute field established across Epic 3, with clear
per-row validation feedback (not an all-or-nothing failure on one bad
row).

REQUIREMENTS
- Add spreadsheet-parsing dependencies to backend/requirements.txt:
  `openpyxl` (for `.xlsx`) — Python's built-in `csv` module handles
  `.csv` with no extra dependency needed. Pin `openpyxl` to a current
  stable version matching this project's fully-pinned convention.
- Design the expected import file SCHEMA (document this clearly — e.g.
  in a downloadable TEMPLATE file the admin UI offers, per the
  requirement below) covering, at minimum: `product_name`, `category`,
  `brand`, `sku` (blank for new variant auto-generation, per Epic 3
  Task 3.1.2.1), `color`, `price`, `stock`, `volume_ml`, `skin_type`,
  `hair_type`, `spf`, `gender`, `is_cruelty_free`, `is_vegan`,
  `is_organic`, `ingredients`, `description` — reference the FULL set
  of cosmetics attribute fields from Epic 3 Feature 3.2.1 and decide,
  deliberately, which are IMPORT-REQUIRED vs. OPTIONAL (e.g.
  `product_name`/`category`/`price` are reasonably required; `spf`/
  `ingredients` reasonably optional) — document this schema clearly in
  a code comment and/or a README, since it's the CONTRACT this whole
  feature depends on.
- Add `POST /dashboard/admin/products/bulk-import/` accepting a file
  upload:
  ```python
  class BulkProductImportView(APIView):
      permission_classes = [IsAdminOrSuperuser]
      parser_classes = [MultiPartParser]

      def post(self, request):
          file = request.FILES.get("file")
          if not file:
              return Response({"file": "No file provided."}, status=400)
          dry_run = request.data.get("dry_run", "true").lower() != "false"

          try:
              rows = self._parse_file(file)
          except ValueError as exc:
              return Response({"file": str(exc)}, status=400)

          results = {"created": [], "updated": [], "errors": []}
          with transaction.atomic():
              savepoint = transaction.savepoint()
              for i, row in enumerate(rows, start=2):  # row 1 is the header
                  try:
                      self._process_row(row, results)
                  except Exception as exc:  # noqa: BLE001 — collect per-row errors, don't abort the whole batch
                      results["errors"].append({"row": i, "error": str(exc)})
              if dry_run:
                  transaction.savepoint_rollback(savepoint)
              else:
                  transaction.savepoint_commit(savepoint)

          return Response({
              "dry_run": dry_run,
              "created_count": len(results["created"]),
              "updated_count": len(results["updated"]),
              "error_count": len(results["errors"]),
              "errors": results["errors"][:100],  # cap error list size in response
          })
  ```
  The `transaction.savepoint()`/`savepoint_rollback()`/
  `savepoint_commit()` pattern is DELIBERATE here: this lets you
  actually RUN every row's real creation logic (so validation errors
  surface exactly as they would in a real commit) while still being
  able to cleanly discard everything for a `dry_run` — a true "run it
  for real, then decide whether to keep it" preview, which is more
  reliable than trying to separately simulate what WOULD happen without
  actually running the real code path (two parallel code paths — one
  for "real," one for "simulated" — are a common source of dry-run/
  real-run behavior drifting apart over time; running the SAME code
  both ways via a savepoint avoids that risk entirely).
  Note the broad `except Exception` PER ROW is deliberate (matching the
  same reasoning already established in Epic 16 Task 16.1.1.4 for
  `notify()` — one bad row's unexpected error must not silently abort
  processing of every OTHER row in a large batch import), while the
  OUTER operation still correctly wraps everything for the dry-run/
  commit decision.
- Implement `_parse_file()` (CSV via `csv.DictReader`, Excel via
  `openpyxl.load_workbook`, detecting format by file extension/content-
  type) and `_process_row()` (look up/create `Category`/`Brand`/
  `Color` by name if they don't already exist — decide whether
  auto-creating a missing category/brand/color is desired behavior, or
  whether a row referencing an unknown one should be an ERROR instead —
  RECOMMEND treating an unrecognized category/brand as an ERROR rather
  than silently auto-creating one, since a typo'd category name
  auto-creating a garbage new category is a worse failure mode than a
  clear per-row error telling the admin to fix the typo or add the
  category first; auto-creating a Color, in contrast, is more
  defensible since colors are a much lower-stakes, more open-ended
  taxonomy — use judgment per field type rather than one blanket rule).
  Look up the target `Product` by `name`+`sku` combination to decide
  CREATE vs. UPDATE — importing a row matching an EXISTING product/
  variant should update it, not create a duplicate.
- Add a downloadable IMPORT TEMPLATE endpoint
  (`GET /dashboard/admin/products/import-template/`) generating a
  blank CSV/Excel file with the correct header row and one example
  data row, so admins have a reliable starting point rather than
  guessing the expected column names/order.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/test_views.py:
1. A well-formed CSV with valid rows for NEW products, in `dry_run=true`
   mode, correctly PREVIEWS the would-be-created rows without actually
   creating anything in the database.
2. The SAME file with `dry_run=false` correctly creates the real
   `Product`/`ProductVariant` rows with all cosmetics attribute fields
   correctly populated from the CSV columns.
3. A row referencing an EXISTING product+SKU combination correctly
   UPDATES that variant rather than creating a duplicate.
4. A row with an invalid/unrecognized category name produces a clear,
   ROW-SPECIFIC error in the `errors` list, WITHOUT aborting processing
   of the other valid rows in the same file (confirm valid rows still
   succeed even when other rows in the same batch fail).
5. An Excel (`.xlsx`) file with equivalent data produces identical
   results to the CSV test cases (proving both format parsers are
   correctly implemented and equivalent).
6. The import-template endpoint returns a file with the correct header
   columns matching what `_process_row()` actually expects (a genuine
   round-trip consistency check — the template and the parser must
   agree).
7. A non-admin user gets 403 on both endpoints.
```

---

#### Task 17.2.1.3 — Bulk product export

```
You are working in backend/dashboard/views.py, urls.py. Assume Task
17.2.1.2 is already merged (the schema this task's export must match).

CONTEXT
The reverse of Task 17.2.1.2 — no way to export the current catalog
(or a filtered subset of it) as a spreadsheet, useful for offline
bulk-editing (export, edit in Excel, re-import via Task 17.2.1.2) or
for external reporting/backup purposes.

TASK
Add a catalog export endpoint producing a file in the EXACT SAME
schema Task 17.2.1.2's import expects, so export→edit→re-import is a
genuinely round-trippable workflow.

REQUIREMENTS
- Add `GET /dashboard/admin/products/bulk-export/` accepting the SAME
  filter query parameters already supported by `AdminProductViewSet`
  (`AdminProductFilter`, category/brand/price-range/etc.) so an admin
  can export a FILTERED subset, not always the entire catalog:
  ```python
  class BulkProductExportView(APIView):
      permission_classes = [IsAdminOrSuperuser]

      def get(self, request):
          fmt = request.query_params.get("format", "csv")
          queryset = AdminProductFilter(
              request.query_params, queryset=Product.objects.select_related("category", "brand")
          ).qs
          variants = ProductVariant.objects.filter(product__in=queryset).select_related(
              "product", "product__category", "product__brand", "color"
          )

          if fmt == "xlsx":
              return self._export_xlsx(variants)
          return self._export_csv(variants)
  ```
  Import `AdminProductFilter` from `.filters` (already exists, per
  Epic 8's grounding of `AdminOrderFilter` — confirm `AdminProductFilter`
  is the correct existing name by checking `dashboard/filters.py`).
  Implement `_export_csv`/`_export_xlsx` producing rows matching
  EXACTLY the column schema documented/established in Task 17.2.1.2 —
  one row PER VARIANT (not per product, since variant-level data —
  SKU, color, price, stock — can't be flattened to one row per product
  without losing information for multi-variant products), with the
  parent product's shared attributes (name, category, brand, skin_type,
  ingredients, etc.) repeated on every variant row (a standard,
  expected spreadsheet-export pattern for parent/child data — the
  redundancy is normal and expected for this kind of flat-file export,
  not a bug).
  Use `StreamingHttpResponse` for the CSV path (mirroring the exact
  memory-efficiency reasoning already established in Epic 8 Task
  8.1.1.4's order-CSV-export task) for large catalog exports; a
  full-workbook `.xlsx` export inherently can't stream the same way
  (openpyxl builds the whole workbook in memory) — for large catalogs,
  note this as a real, known limitation in a code comment rather than
  pretending it's unbounded-safe, and consider capping XLSX export
  size (e.g. reject/warn above some row count, directing the admin to
  the CSV format instead for very large exports) if that's a genuine
  practical concern at this project's expected catalog scale.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/test_views.py:
1. Exporting with no filters returns ALL variants across the catalog,
   one row each, with every cosmetics attribute field correctly
   populated from the actual `Product`/`ProductVariant` data.
2. Exporting with a category filter applied returns only variants of
   products in that category.
3. **Round-trip test (the most important test in this task)**: export
   the current catalog, then feed the EXACT exported file back into
   Task 17.2.1.2's import endpoint (in `dry_run=true` mode) and confirm
   it parses correctly with ZERO errors and correctly identifies every
   row as an UPDATE (matching existing products/SKUs) rather than
   misinterpreting anything as a new row — this is the concrete,
   automated proof that the export and import schemas genuinely agree
   with each other, not just that each independently "looks reasonable."
4. Both CSV and XLSX format exports produce files that, when parsed
   back, contain equivalent data.
5. A non-admin user gets 403.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 17.1.1.1 | Revenue/order-count summary (fix currency-type bug + extend) | ☐ |
| 17.1.1.2 | Top-selling products report (+ variant breakdown) | ☐ |
| 17.1.1.3 | Low-stock report widget (actionable list) | ☐ |
| 17.1.1.4 | Near-expiry inventory report widget | ☐ |
| 17.1.1.5 | Coupon usage report | ☐ |
| 17.2.1.1 | Bulk product price update tool | ☐ |
| 17.2.1.2 | Bulk product import via CSV/Excel | ☐ |
| 17.2.1.3 | Bulk product export | ☐ |

Once Epic 17 is fully merged, the next epic to generate prompts for is
**Epic 18 — Customer Dashboard**, which is worth noting has its own
related, still-outstanding discovered issue from this epic's grounding:
`dashboard/services.py`'s `get_user_summary()` currently computes a
customer's `reviews_count` by matching `Review.objects.filter(name=full_name)`
— a fragile STRING-based name match left over from before Epic 1 Task
1.3.1.1 added a real `Review.user` foreign key — this should be fixed
to `Review.objects.filter(user=user).count()` as part of Epic 18's
customer-facing account work, since it's a real, currently-live
accuracy bug in a number displayed back to the customer about their own
account.
