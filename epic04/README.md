# Epic 4 — Inventory Management — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–3 are fully merged. Specifically: `ProductVariant` (backend/shop/models.py) is now the authoritative source of stock/price per shade/size, `order/serializers.py` `OrderCreateSerializer.create()` decrements `ProductVariant.stock` inside a `transaction.atomic()` + `select_for_update()` block (Epic 3 Task 3.1.1.5), and `OrderDetailView.patch()` restores `ProductVariant.stock` on cancellation.

**Known pre-existing gap this epic will keep bumping into:** `backend/dashboard/services.py` (`get_admin_overview`, `get_product_stats`) and `backend/dashboard/serializers.py` (`AdminProductSerializer`) still reference `Product.stock`/`Product.price` directly — these were NOT updated as part of Epic 3 (that epic only touched the `shop` app's own storefront-facing views/serializers/filters, not the `dashboard` admin app). Several tasks below explicitly call this out and fix the specific piece relevant to that task; a few remaining call sites may still need attention beyond this epic's scope — flag anything you find but don't fix in a `# TODO` comment rather than silently expanding scope.

---

## Phase 4.1 — Stock Operations

### Feature 4.1.1 — Inventory Adjustments

---

#### Task 4.1.1.1 — Create `StockMovement` audit log model

```
You are working in backend/shop/models.py. Assume Epic 3 (ProductVariant)
is fully merged.

CONTEXT
There is currently zero audit trail for inventory changes anywhere in
the codebase. `ProductVariant.stock` gets silently decremented on order
creation and incremented on cancellation (Epic 3 Task 3.1.1.5), but
nothing records WHY a given stock number is what it is at any point in
time — if a customer disputes an order, if numbers look wrong, or if
an admin manually corrects stock, there's no history to consult. This
is a real operational gap for any business running actual inventory.

TASK
Create a `StockMovement` model in backend/shop/models.py that records
every change to a variant's stock, with enough context to reconstruct
the full history of any variant's inventory over time.

REQUIREMENTS
- Add:
  ```python
  class StockMovement(models.Model):
      class Reason(models.TextChoices):
          SALE = "sale", "Sale"
          CANCELLATION = "cancellation", "Order Cancellation"
          MANUAL = "manual", "Manual Adjustment"
          RESTOCK = "restock", "Restock"
          EXPIRY_SWEEP = "expiry_sweep", "Expiry Deactivation"

      variant = models.ForeignKey(
          "shop.ProductVariant", on_delete=models.CASCADE, related_name="stock_movements"
      )
      reason = models.CharField(max_length=20, choices=Reason.choices)
      quantity_delta = models.IntegerField(
          help_text="Positive for increases, negative for decreases."
      )
      stock_after = models.PositiveIntegerField(
          help_text="Variant's stock value immediately after this movement."
      )
      actor = models.ForeignKey(
          settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True, blank=True,
          related_name="stock_movements",
          help_text="Admin user who triggered this (manual adjustments only); null for system-triggered movements.",
      )
      note = models.CharField(max_length=255, blank=True)
      related_order = models.ForeignKey(
          "order.Order", on_delete=models.SET_NULL, null=True, blank=True,
          related_name="stock_movements",
      )
      created_at = models.DateTimeField(auto_now_add=True)

      class Meta:
          ordering = ["-created_at"]
          indexes = [models.Index(fields=["variant", "-created_at"])]

      def __str__(self):
          sign = "+" if self.quantity_delta >= 0 else ""
          return f"{self.variant} {sign}{self.quantity_delta} ({self.reason})"
  ```
  Import `settings` from `django.conf` at the top of models.py if not
  already imported (check — Task from Epic 1/2 may have already added
  this import; don't duplicate it).
- `quantity_delta` is a signed `IntegerField` (not `PositiveIntegerField`)
  since decreases are negative and increases are positive — this is
  deliberately different from `PositiveIntegerField` used elsewhere in
  this codebase for plain counts.
- `related_order` uses a STRING reference `"order.Order"` (cross-app FK,
  matching the exact pattern already used by `CartItem.product =
  models.ForeignKey("shop.Product", ...)` and `OrderItem.product =
  models.ForeignKey("shop.Product", ...)` elsewhere in this codebase)
  to avoid a circular import between the `shop` and `order` apps.
- Generate the migration.
- Register `StockMovement` in backend/shop/admin.py as a READ-ONLY
  admin view (this is an audit log — nobody should be able to edit or
  delete history through the admin UI):
  ```python
  @admin.register(StockMovement)
  class StockMovementAdmin(admin.ModelAdmin):
      list_display = ("id", "variant", "reason", "quantity_delta", "stock_after", "actor", "created_at")
      list_filter = ("reason", "created_at")
      search_fields = ("variant__sku", "variant__product__name", "note")
      ordering = ("-created_at",)
      readonly_fields = [f.name for f in StockMovement._meta.fields]

      def has_add_permission(self, request):
          return False

      def has_change_permission(self, request, obj=None):
          return False

      def has_delete_permission(self, request, obj=None):
          return False
  ```
  (blocking add/change/delete entirely in the admin — the ONLY way
  rows should ever be created is through application code, per Tasks
  4.1.1.2/4.1.1.3/4.1.1.4, never hand-typed by an admin directly into
  this audit table).

ACCEPTANCE CRITERIA / TESTS
- Migration applies cleanly.
- Add a model test confirming a `StockMovement` can be created linked
  to a `ProductVariant`, with `str()` producing a readable
  representation, and that `actor`/`related_order`/`note` are all
  genuinely optional (a movement with none of them set still saves
  successfully — e.g. for the future automated expiry-sweep reason).
```

---

#### Task 4.1.1.2 — Log stock movement on order creation

```
You are working in backend/order/serializers.py
(OrderCreateSerializer.create()). Assume Task 4.1.1.1 (StockMovement
model) is already merged, and that Epic 3 Task 3.1.1.5's variant-based
stock decrement logic is already in place inside this same method.

CONTEXT
The order-creation flow already does, inside its `transaction.atomic()`
block: lock the variant via `select_for_update()`, check sufficient
stock, decrement `locked_variant.stock`, save. None of this is logged
anywhere per Task 4.1.1.1's new audit table.

TASK
Add a `StockMovement` row for every stock decrement that happens during
order creation, inside the same atomic block (so the audit row and the
stock change are guaranteed to commit or roll back together — an audit
log entry for a decrement that got rolled back would be a lie, so this
MUST happen inside the same transaction, not after it).

REQUIREMENTS
- Import `StockMovement` from `shop.models` at the top of
  order/serializers.py (alongside whatever `shop` model imports already
  exist there from prior epics' work).
- Immediately after decrementing and saving `locked_variant.stock` in
  the existing loop (per Epic 3 Task 3.1.1.5), add:
  ```python
  StockMovement.objects.create(
      variant=locked_variant,
      reason=StockMovement.Reason.SALE,
      quantity_delta=-cart_item.quantity,
      stock_after=locked_variant.stock,
      related_order=order,
      note=f"Order {order.order_number}",
  )
  ```
  — note `related_order=order` requires the `Order` instance to already
  exist at this point in the method; confirm the existing code creates
  the `Order` row BEFORE the OrderItem/stock-decrement loop runs (it
  should, per Epic 1's structure — `Order.objects.create(...)` comes
  first, then the loop over cart items) so `order` is available here;
  if for some reason it isn't yet available at this exact point,
  restructure minimally so the Order exists first, but don't otherwise
  change the method's logic/ordering beyond what's needed for this.
  `actor` is left `None` here — sales are system-triggered, not admin
  actions, so no actor is attributed.

ACCEPTANCE CRITERIA / TESTS
Add tests to the order test suite:
1. Placing a successful order creates exactly one `StockMovement` per
   distinct variant ordered, each with `reason="sale"`,
   `quantity_delta` equal to the negative of the ordered quantity,
   `stock_after` matching the variant's actual post-order stock value,
   and `related_order` pointing at the created order.
2. A failed order (e.g. insufficient stock, triggering the existing
   Epic 3 validation error) creates ZERO `StockMovement` rows — proving
   the atomic rollback correctly discards the audit entry along with
   everything else (this is the single most important test in this
   task: an audit log must never contain phantom entries for
   transactions that didn't actually happen).
```

---

#### Task 4.1.1.3 — Log stock movement on order cancellation

```
You are working in backend/order/views.py (OrderDetailView.patch()).
Assume Task 4.1.1.2 is already merged, and Epic 3 Task 3.1.1.5's
variant-based stock restoration logic is already in place in this
method.

CONTEXT
Mirrors Task 4.1.1.2 exactly, but for the cancellation/restoration side
of the stock lifecycle (from Epic 1 Task 1.1.1.4, repointed to
variants by Epic 3 Task 3.1.1.5) — order cancellation restores stock to
`ProductVariant.stock` but doesn't log it anywhere.

TASK
Add a `StockMovement` row for every stock restoration that happens
during order cancellation, inside the same atomic block used for the
restoration itself.

REQUIREMENTS
- Import `StockMovement` from `shop.models` at the top of
  order/views.py (if not already imported there from other work).
- Immediately after incrementing and saving each restored variant's
  stock in the existing cancellation loop, add:
  ```python
  StockMovement.objects.create(
      variant=order_item.variant,
      reason=StockMovement.Reason.CANCELLATION,
      quantity_delta=order_item.quantity,
      stock_after=order_item.variant.stock,
      related_order=order,
      note=f"Cancellation of order {order.order_number}",
  )
  ```
  — remember the existing logic (per Epic 3 Task 3.1.1.5) already
  skips restoration entirely when `order_item.variant is None`
  (deleted variant); this StockMovement creation must live INSIDE that
  same guard, not run unconditionally, since there's nothing to log a
  movement against if the variant no longer exists.
  `actor` should be set to `self.request.user` here (unlike the sale
  case in Task 4.1.1.2, a cancellation IS explicitly triggered by a
  specific user — either the customer cancelling their own order, or
  potentially an admin doing it on their behalf later; recording who
  triggered it is meaningful audit information here in a way it isn't
  for an automated sale).

ACCEPTANCE CRITERIA / TESTS
Add tests to the order test suite:
1. Cancelling a pending order with variant-backed items creates exactly
   one `StockMovement` per restored variant, `reason="cancellation"`,
   `quantity_delta` equal to the positive restored quantity,
   `stock_after` correct, `actor` matching the cancelling user,
   `related_order` pointing at the cancelled order.
2. Cancelling an order where one item's variant was deleted (`variant=None`
   scenario from Epic 3 Task 3.1.1.5) creates NO `StockMovement` for
   that specific item, but still creates movements for the other valid
   items — mirroring the equivalent partial-skip test from that
   earlier task.
3. Attempting to cancel an already-shipped order (which the existing
   guard clause rejects with 400) creates ZERO `StockMovement` rows.
```

---

#### Task 4.1.1.4 — Admin manual stock adjustment endpoint

```
You are working in backend/dashboard/ (views.py, serializers.py,
urls.py). Assume Task 4.1.1.1 (StockMovement model) is already merged.

CONTEXT
Store admins currently have no way to manually adjust a variant's stock
through the API/admin dashboard with any audit trail — the ONLY
existing way to change `ProductVariant.stock` is via the raw Django
admin change form (Task 3.1.1.6's `ProductVariantAdmin`), which bypasses
the new StockMovement audit log entirely (a raw admin form edit doesn't
trigger any custom logging). Real operational scenarios — receiving a
new shipment, correcting a miscount, writing off damaged stock — need a
proper API-level "adjust stock with a reason" action that's both
audit-logged AND accessible from whatever admin dashboard frontend
consumes the `dashboard` app's API (check
`backend/dashboard/urls.py`/`views.py` — this is clearly a
React-admin-consumed REST API, not just Django's built-in admin, based
on the extensive `AdminXxxViewSet` pattern already established there).

TASK
Add a dedicated `POST /dashboard/admin/variants/{id}/adjust-stock/`
endpoint that lets an admin set a stock delta with a required reason
and optional note, updates the variant's stock, and creates a
StockMovement row — all atomically.

REQUIREMENTS
- Create `AdjustStockSerializer` in backend/dashboard/serializers.py:
  ```python
  class AdjustStockSerializer(serializers.Serializer):
      quantity_delta = serializers.IntegerField()
      reason = serializers.ChoiceField(
          choices=[StockMovement.Reason.MANUAL, StockMovement.Reason.RESTOCK],
      )
      note = serializers.CharField(max_length=255, required=False, allow_blank=True)

      def validate_quantity_delta(self, value):
          if value == 0:
              raise serializers.ValidationError("Adjustment cannot be zero.")
          return value
  ```
  — deliberately restrict `reason` to only `MANUAL`/`RESTOCK` via the
  choices list (not the full `StockMovement.Reason` enum) since `SALE`/
  `CANCELLATION`/`EXPIRY_SWEEP` are exclusively system-triggered and
  must never be selectable through this admin-facing manual-adjustment
  endpoint — an admin manually creating a fake "sale" or "expiry"
  movement would corrupt the audit trail's meaning. Import
  `StockMovement` from `shop.models` at the top of
  dashboard/serializers.py.
- Add a view (either a new `APIView` in dashboard/views.py, or a custom
  `@action` on a new `AdminProductVariantViewSet` if you decide the
  project needs full variant CRUD in the admin dashboard beyond just
  this one action — check whether such a viewset already exists first;
  based on the grounded view of dashboard/views.py, it currently does
  NOT have any variant-specific admin viewset, only `AdminProductViewSet`
  operating on `Product` directly. Given that, the minimal-scope choice
  is a standalone `APIView`, not a full new ViewSet — building a full
  `AdminProductVariantViewSet` for CRUD is a larger, separate concern
  not asked for by this task; stick to just the adjustment action):
  ```python
  class AdminVariantAdjustStockView(APIView):
      """POST /dashboard/admin/variants/{id}/adjust-stock/"""
      permission_classes = [IsAdminOrSuperuser]

      def post(self, request, pk):
          variant = get_object_or_404(ProductVariant, pk=pk)
          serializer = AdjustStockSerializer(data=request.data)
          serializer.is_valid(raise_exception=True)
          delta = serializer.validated_data["quantity_delta"]
          reason = serializer.validated_data["reason"]
          note = serializer.validated_data.get("note", "")

          with transaction.atomic():
              locked_variant = ProductVariant.objects.select_for_update().get(pk=variant.pk)
              new_stock = locked_variant.stock + delta
              if new_stock < 0:
                  return Response(
                      {"quantity_delta": "This adjustment would result in negative stock."},
                      status=status.HTTP_400_BAD_REQUEST,
                  )
              locked_variant.stock = new_stock
              locked_variant.save(update_fields=["stock"])
              StockMovement.objects.create(
                  variant=locked_variant,
                  reason=reason,
                  quantity_delta=delta,
                  stock_after=locked_variant.stock,
                  actor=request.user,
                  note=note,
              )
          return Response(
              {"id": locked_variant.id, "stock": locked_variant.stock},
              status=status.HTTP_200_OK,
          )
  ```
  Import `transaction` from `django.db`, `ProductVariant` and
  `StockMovement` from `shop.models`, at the top of
  backend/dashboard/views.py alongside the existing `shop.models`
  imports there (`from shop.models import Brand, Category, Product,
  Review` already exists — extend it).
- Register the URL in backend/dashboard/urls.py:
  `path("admin/variants/<int:pk>/adjust-stock/", views.AdminVariantAdjustStockView.as_view(), name="admin-variant-adjust-stock")`
  (a plain `path()`, matching the style of the other non-router admin
  analytics endpoints already in that file, since this is a single
  custom action rather than full ViewSet CRUD).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/test_views.py (matching whatever
existing test/fixture conventions that file already uses for admin
endpoints):
1. An admin user posting a positive `quantity_delta` with
   `reason="restock"` increases the variant's stock by the correct
   amount and creates a matching `StockMovement` with `actor` set to
   the admin.
2. An admin user posting a negative `quantity_delta` (e.g. writing off
   damaged stock) with `reason="manual"` decreases stock correctly.
3. An adjustment that would take stock negative is rejected with 400
   and neither the variant's stock nor any StockMovement is changed/created.
4. A `quantity_delta` of 0 is rejected by serializer validation.
5. A `reason` value outside the allowed `MANUAL`/`RESTOCK` set (e.g.
   attempting to submit `"sale"`) is rejected by serializer validation
   — proving an admin can't forge a fake sale/cancellation movement
   through this endpoint.
6. A non-admin (regular customer) user gets 403.
```

---

#### Task 4.1.1.5 — Low-stock threshold field + admin alert list

```
You are working in backend/shop/models.py, backend/shop/admin.py, and
backend/dashboard/services.py. Assume Task 4.1.1.1 is merged.

CONTEXT — a known, pre-existing gap
`backend/dashboard/services.py`'s `get_admin_overview()` and
`get_product_stats()` functions currently compute low-stock/out-of-stock
counts directly off `Product.stock`:

    low_stock = Product.objects.filter(stock__gt=0, stock__lte=5).count()
    out_of_stock = Product.objects.filter(stock=0).count()

This is now WRONG (or at minimum meaningless) since Epic 3 moved
authoritative stock tracking to `ProductVariant.stock` — `Product.stock`
is a stale, unmaintained field left in place deliberately (per Epic 3
Task 3.1.1.5's explicit note that it was intentionally NOT removed yet).
This task both adds the new per-variant threshold field AND fixes these
two specific stale call sites, since they're directly the "admin
low-stock report" this task is building toward.

TASK
Add a configurable `low_stock_threshold` field to `ProductVariant`
(defaulting to a sane project-wide value but overridable per variant),
and fix the dashboard's low-stock/out-of-stock reporting to query
`ProductVariant` instead of the stale `Product.stock`.

REQUIREMENTS
- Add to `ProductVariant`:
  `low_stock_threshold = models.PositiveIntegerField(default=5)`
  — a per-variant override (some fast-moving products may want a higher
  threshold, some low-volume ones lower), defaulting to 5 to match the
  hardcoded `5` already used in the existing (soon-to-be-fixed)
  dashboard service code, so the migrated behavior is a drop-in
  equivalent by default rather than a silent behavior change.
- Generate the migration.
- Add a `is_low_stock` property to `ProductVariant`:
  ```python
  @property
  def is_low_stock(self) -> bool:
      return 0 < self.stock <= self.low_stock_threshold
  ```
- Add `low_stock_threshold` to `ProductVariantInline.fields` and
  `ProductVariantAdmin.list_display`/`list_filter` (from Epic 3 Task
  3.1.1.6's admin config — extend it, don't replace it).
- Add a custom `SimpleListFilter` to `ProductVariantAdmin`, mirroring
  the exact pattern established in Epic 3 Task 3.3.1.1's
  `NearExpiryFilter`, called `LowStockFilter`:
  ```python
  class LowStockFilter(admin.SimpleListFilter):
      title = "stock status"
      parameter_name = "stock_status"

      def lookups(self, request, model_admin):
          return (("low", "Low stock"), ("out", "Out of stock"))

      def queryset(self, request, queryset):
          if self.value() == "low":
              return queryset.filter(stock__gt=0, stock__lte=F("low_stock_threshold"))
          if self.value() == "out":
              return queryset.filter(stock=0)
          return queryset
  ```
  Import `F` from `django.db.models` at the top of admin.py if not
  already imported. Add `LowStockFilter` to
  `ProductVariantAdmin.list_filter`.
- FIX the stale call sites in backend/dashboard/services.py:
  Replace
  `low_stock = Product.objects.filter(stock__gt=0, stock__lte=5).count()`
  and
  `out_of_stock = Product.objects.filter(stock=0).count()`
  in `get_admin_overview()`, and the equivalent lines in
  `get_product_stats()`
  (`"out_of_stock": Product.objects.filter(stock=0).count()`,
  `"low_stock": Product.objects.filter(stock__gt=0, stock__lte=5).count()`),
  with variant-based equivalents:
  ```python
  from shop.models import ProductVariant  # add to existing imports

  low_stock = ProductVariant.objects.filter(
      stock__gt=0, stock__lte=F("low_stock_threshold"), is_active=True
  ).count()
  out_of_stock = ProductVariant.objects.filter(stock=0, is_active=True).count()
  ```
  (import `F` from `django.db.models` in services.py alongside the
  other already-imported `django.db.models` functions there). Note
  these now count VARIANTS, not products — a product with 3 variants
  where 1 is low-stock now contributes 1 to this count, not counted at
  the product level; this is a more accurate representation for a
  variant-based catalog and should be reflected if the frontend admin
  dashboard displays a label like "Low stock products" — flag this
  label-accuracy consideration in a comment, but don't attempt to fix
  frontend copy as part of this backend task; note it as a follow-up.

ACCEPTANCE CRITERIA / TESTS
- Add model tests confirming `is_low_stock` returns correct
  True/False/edge-case (stock exactly equal to threshold, stock=0)
  values.
- Add/update tests in backend/dashboard/tests/test_services.py
  confirming `get_admin_overview()`'s and `get_product_stats()`'s
  low_stock/out_of_stock counts now correctly reflect
  `ProductVariant` data (construct variants with known stock/threshold
  combinations and assert the exact expected counts), and that
  inactive variants (`is_active=False`) are excluded from both counts
  (a deactivated/expired variant per Epic 3 Task 3.3.1.3 shouldn't
  show up as an actionable "needs restocking" alert).
```

---

### Feature 4.1.2 — Back-in-Stock Notifications

---

#### Task 4.1.2.1 — `StockAlertSubscription` model

```
You are working in backend/shop/models.py. Assume Epic 3
(ProductVariant) is fully merged.

CONTEXT
There's currently no way for a customer to ask to be notified when an
out-of-stock variant becomes available again — a standard, expected
feature on real cosmetics storefronts (popular shades/products
frequently sell out).

TASK
Create a `StockAlertSubscription` model tracking which users want to be
notified when a specific out-of-stock variant is replenished.

REQUIREMENTS
- Add:
  ```python
  class StockAlertSubscription(models.Model):
      user = models.ForeignKey(
          settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
          related_name="stock_alert_subscriptions",
      )
      variant = models.ForeignKey(
          "shop.ProductVariant", on_delete=models.CASCADE,
          related_name="alert_subscriptions",
      )
      created_at = models.DateTimeField(auto_now_add=True)
      notified_at = models.DateTimeField(null=True, blank=True)

      class Meta:
          unique_together = ("user", "variant")
          ordering = ["-created_at"]

      def __str__(self):
          return f"{self.user.email} — alert for {self.variant}"
  ```
  — `unique_together` prevents a user subscribing to the same variant
  twice (the "subscribe" endpoint in Task 4.1.2.2 should treat a repeat
  subscribe attempt as a harmless no-op, not an error — handle that at
  the view/serializer layer, not by letting this constraint raise an
  ugly 500).
  `notified_at` is nullable and starts unset; Task 4.1.2.3 sets it once
  a notification has actually been sent, both to avoid double-notifying
  the same person for the same restock event and to give admins
  visibility into whether the notification pipeline is actually
  working.
- Generate the migration.
- Register in backend/shop/admin.py as a mostly-read-only admin view
  (staff should be able to see subscription volume for demand
  forecasting, but this table is populated by application logic, not
  meant for hand-editing):
  ```python
  @admin.register(StockAlertSubscription)
  class StockAlertSubscriptionAdmin(admin.ModelAdmin):
      list_display = ("id", "user", "variant", "created_at", "notified_at")
      list_filter = ("notified_at",)
      search_fields = ("user__email", "variant__sku", "variant__product__name")
      readonly_fields = ("user", "variant", "created_at", "notified_at")

      def has_add_permission(self, request):
          return False
  ```
  (allowing delete but not add/edit — an admin might reasonably want to
  clear out a stale subscription, but shouldn't be hand-creating or
  editing them).

ACCEPTANCE CRITERIA / TESTS
Add a model test confirming a subscription can be created, the
uniqueness constraint prevents duplicates at the DB level (assert
`IntegrityError` on a raw duplicate `.objects.create()` call, to prove
the constraint exists — the graceful no-op handling itself is tested at
the endpoint level in Task 4.1.2.2), and `notified_at` defaults to
`None`.
```

---

#### Task 4.1.2.2 — "Notify me" API endpoint

```
You are working in backend/shop/views.py, serializers.py, urls.py.
Assume Task 4.1.2.1 (StockAlertSubscription model) is already merged.

CONTEXT
The model exists but there's no API surface for the storefront frontend
to actually create/remove a subscription yet.

TASK
Add subscribe/unsubscribe endpoints for stock alerts.

REQUIREMENTS
- Create `StockAlertSubscriptionSerializer` in shop/serializers.py:
  ```python
  class StockAlertSubscriptionSerializer(serializers.ModelSerializer):
      class Meta:
          model = StockAlertSubscription
          fields = ("id", "variant", "created_at")
          read_only_fields = ("id", "created_at")
  ```
  Import `StockAlertSubscription` into the existing model-import line
  at the top of serializers.py.
- Create a view (following the existing `generics`-based style used
  throughout shop/views.py rather than introducing a new pattern):
  ```python
  class StockAlertSubscribeView(generics.CreateAPIView):
      """POST /api/products/variants/{variant_id}/notify-me/"""
      permission_classes = [IsAuthenticated]
      serializer_class = StockAlertSubscriptionSerializer

      def create(self, request, *args, **kwargs):
          variant = get_object_or_404(ProductVariant, pk=self.kwargs["variant_id"])
          if variant.stock > 0:
              return Response(
                  {"detail": "This item is currently in stock."},
                  status=status.HTTP_400_BAD_REQUEST,
              )
          subscription, created = StockAlertSubscription.objects.get_or_create(
              user=request.user, variant=variant
          )
          serializer = self.get_serializer(subscription)
          return Response(
              serializer.data,
              status=status.HTTP_201_CREATED if created else status.HTTP_200_OK,
          )


  class StockAlertUnsubscribeView(generics.DestroyAPIView):
      """DELETE /api/products/variants/{variant_id}/notify-me/"""
      permission_classes = [IsAuthenticated]

      def get_object(self):
          return get_object_or_404(
              StockAlertSubscription,
              user=self.request.user,
              variant_id=self.kwargs["variant_id"],
          )
  ```
  — using `get_or_create` (not plain `create`) in the subscribe view is
  exactly what handles the "repeat subscribe attempt" no-op case noted
  in Task 4.1.2.1, returning 200 instead of 201 for an already-existing
  subscription rather than erroring.
  Import `IsAuthenticated` from `rest_framework.permissions`,
  `ProductVariant` and `StockAlertSubscription` into the existing
  model-import line, at the top of shop/views.py.
- Register both URLs in backend/shop/urls.py:
  ```python
  path("products/variants/<int:variant_id>/notify-me/", views.StockAlertSubscribeView.as_view(), name="stock-alert-subscribe"),
  path("products/variants/<int:variant_id>/notify-me/", views.StockAlertUnsubscribeView.as_view(), name="stock-alert-unsubscribe"),
  ```
  Wait — check whether Django's URL routing allows two different views
  registered at the identical path differentiated only by HTTP method
  via two separate `path()` calls (it does NOT reliably work this way
  with plain `path()`, since each `path()` entry maps one URL pattern
  to one view, and Django tries them in order — the second one would
  never be reached for GET/POST but WOULD be reached correctly for the
  different HTTP methods IF you're using `View.as_view()`-based CBVs
  that already dispatch by method internally... but two SEPARATE view
  classes at the same path is genuinely ambiguous routing). Instead,
  combine both into ONE view class handling both `post`/`delete`, OR
  use a single `APIView` with both methods defined, rather than two
  separate `generics` views at the same URL — restructure using a
  single `APIView` subclass with both `post()` and `delete()` methods
  defined on it, registered at ONE url pattern, which is the correct
  way to handle this in DRF.

ACCEPTANCE CRITERIA / TESTS
Add tests to shop/tests/test_views.py:
1. Subscribing to an out-of-stock variant succeeds (201) and creates a
   subscription.
2. Subscribing to an IN-stock variant is rejected with 400 (matches the
   `if variant.stock > 0` guard).
3. Subscribing twice to the same out-of-stock variant returns 200 (not
   an error) on the second call, and only one subscription row exists.
4. Unsubscribing removes the subscription (204) and unsubscribing again
   when none exists returns 404.
5. An unauthenticated request to either endpoint returns 401.
```

---

#### Task 4.1.2.3 — Trigger notification when stock replenished

```
You are working in backend/shop/models.py or a new
backend/shop/signals.py, and backend/shop/tasks.py (created in Epic 3
Task 3.3.1.3, assuming Celery infrastructure exists per that task's
same caveat — CONFIRM Celery/django-celery-beat is actually set up
before starting this task; if not, this task depends on Epic 22 Phase
22.1 being done first, same as Epic 3 Task 3.3.1.3 noted). Assume Task
4.1.2.1 (subscription model) and Task 4.1.1.4 (admin stock adjustment,
which is one of the two real code paths that increases stock — the
other being order cancellation restoration) are already merged.

CONTEXT
Subscriptions can be created (Task 4.1.2.2) but nothing ever fires a
notification when the subscribed variant's stock actually goes from 0
to something positive — the two places stock can increase are: (a)
order cancellation restoration (Epic 3 Task 3.1.1.5 / Task 4.1.1.3),
and (b) the new admin manual stock adjustment endpoint (Task 4.1.1.4,
specifically the `RESTOCK` reason case).

TASK
Detect the 0→positive stock transition wherever it happens and queue a
Celery task to notify all pending subscribers for that variant, then
mark them as notified.

REQUIREMENTS
- The cleanest, least-duplicated-logic approach: rather than adding
  ad-hoc "check if this was a restock" logic separately inside BOTH the
  order-cancellation code path AND the admin-adjustment code path
  (duplicating the same detection logic in two places, prone to drift),
  hook into the `StockMovement` creation itself (from Task 4.1.1.1) via
  a Django `post_save` signal — EVERY stock increase already creates a
  `StockMovement` row with a positive `quantity_delta` (per Tasks
  4.1.1.3's cancellation-restoration and 4.1.1.4's manual-adjustment
  logging), so a single signal handler listening for
  `StockMovement` creation with `quantity_delta > 0` catches BOTH cases
  automatically without touching either of those two views again.
- Create backend/shop/signals.py:
  ```python
  from django.db.models.signals import post_save
  from django.dispatch import receiver
  from .models import StockMovement
  from .tasks import notify_stock_alert_subscribers

  @receiver(post_save, sender=StockMovement)
  def handle_stock_increase(sender, instance, created, **kwargs):
      if not created or instance.quantity_delta <= 0:
          return
      variant = instance.variant
      # Only notify if the variant actually WENT FROM zero — a restock
      # from e.g. 3 to 8 units was never "out of stock" from the
      # subscriber's perspective and shouldn't spam anyone.
      stock_before = instance.stock_after - instance.quantity_delta
      if stock_before <= 0 and instance.stock_after > 0:
          notify_stock_alert_subscribers.delay(variant.id)
  ```
  — the `stock_before <= 0 and stock_after > 0` check is the actual
  "replenished from empty" detection, computed from the StockMovement's
  own recorded fields rather than re-querying current variant state
  (which could have changed again by the time the signal handler runs
  if there's any async delay — using the movement's own snapshot values
  is more correct).
- Ensure `shop/signals.py` is actually connected — check whether
  backend/shop/apps.py already has a `ready()` method wiring up signals
  (search for existing signal-connection patterns elsewhere in the
  codebase, e.g. accounts/models.py's `create_profile` signal — how is
  THAT one connected? If it's a `@receiver` decorator directly in
  models.py with no explicit `ready()` import needed because Django
  auto-discovers signal handlers defined in `models.py`, note that
  `signals.py` is a DIFFERENT file and does need an explicit import in
  `apps.py`'s `ready()` method to be registered — add that if missing:
  ```python
  # shop/apps.py
  class ShopConfig(AppConfig):
      ...
      def ready(self):
          import shop.signals  # noqa
  ```
  ).
- Add `notify_stock_alert_subscribers` to backend/shop/tasks.py
  (alongside the `deactivate_expired_variants` task from Epic 3 Task
  3.3.1.3):
  ```python
  @shared_task
  def notify_stock_alert_subscribers(variant_id):
      from .models import ProductVariant, StockAlertSubscription
      try:
          variant = ProductVariant.objects.get(pk=variant_id)
      except ProductVariant.DoesNotExist:
          return "Variant no longer exists."
      subscriptions = StockAlertSubscription.objects.filter(
          variant=variant, notified_at__isnull=True
      ).select_related("user")
      count = 0
      for sub in subscriptions:
          # Actual email/SMS dispatch is Epic 16's notification system —
          # this task only marks subscribers as notified and is
          # deliberately structured so Epic 16 can plug in the real
          # send() call at the marked location below without touching
          # this detection/dedup logic.
          # TODO: Epic 16 — replace this comment with a real
          # notify(user, event="back_in_stock", context={"variant": variant})
          # call once the unified notification service exists.
          sub.notified_at = timezone.now()
          sub.save(update_fields=["notified_at"])
          count += 1
      return f"Notified {count} subscriber(s) for variant {variant_id}."
  ```
  Import `timezone` from `django.utils` at the top of tasks.py.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/ (test_signals.py or test_tasks.py,
match existing convention):
1. Creating a `StockMovement` that takes a variant's stock from 0 to a
   positive number triggers a call to
   `notify_stock_alert_subscribers.delay()` (mock/patch `.delay` and
   assert it was called with the correct variant id — don't require a
   real Celery worker/broker for this test).
2. Creating a `StockMovement` that increases stock from a NON-zero
   starting point (e.g. 3→8) does NOT trigger the notification task
   (proving the "was actually empty before" check works).
3. Creating a `StockMovement` with a NEGATIVE `quantity_delta` (a sale)
   never triggers the notification task.
4. Directly testing `notify_stock_alert_subscribers(variant_id)`: given
   2 unnotified subscriptions and 1 already-notified subscription for
   the same variant, running the task marks only the 2 unnotified ones
   with a `notified_at` timestamp, leaving the already-notified one's
   original timestamp untouched, and returns a count of 2.
```

---

#### Task 4.1.2.4 — Frontend "Notify Me" button on out-of-stock PDP

```
You are working in the frontend/src React app (the product detail page
component — locate it via `find frontend/src/pages -iname "*product*"`
and inspect its structure before editing). Assume Task 4.1.2.2 (the
subscribe/unsubscribe API endpoints) is already merged and reachable.

CONTEXT
The product detail page currently has no awareness of per-variant stock
at all (per-variant rendering itself may still be in progress depending
on how much of Epic 3's frontend migration has landed by this point —
check whether the PDP component currently renders variant selection,
e.g. a color/shade swatch picker, or still shows only the flat
`product.stock`/`product.price` fields from before Epic 3; if the
frontend hasn't yet been updated to be variant-aware at all, that's a
gap outside this specific task's scope — flag it clearly rather than
silently doing a much bigger frontend migration as a side effect of
what's meant to be a small "add a button" task. This task assumes SOME
variant selection UI already exists or is minimally stubbed; if it
doesn't, implement the smallest possible variant-aware stock check
needed to make this feature work, and note in your summary that full
variant-selector UI is tracked separately).

TASK
When the currently-selected variant is out of stock, replace the
"Add to Cart" button with a "Notify Me When Available" button (or show
both, with Add to Cart disabled — your call based on what reads better
in the existing UI, but the out-of-stock state must be visually
distinct, not just a disabled-looking button with no explanation).

REQUIREMENTS
- Add API methods to frontend/src/services/api.js (find the existing
  product/shop-related API object, likely `shopAPI` or similar, and
  follow its existing method/JSDoc style exactly):
  ```javascript
  /**
   * POST /products/variants/{variantId}/notify-me/
   */
  subscribeStockAlert: (variantId) =>
    api.post(`/products/variants/${variantId}/notify-me/`),

  /**
   * DELETE /products/variants/{variantId}/notify-me/
   */
  unsubscribeStockAlert: (variantId) =>
    api.delete(`/products/variants/${variantId}/notify-me/`),
  ```
- On the PDP, when the selected variant's `stock === 0`:
  - Render a "Notify Me When Available" button in place of (or
    alongside, disabled) "Add to Cart".
  - If the user is not authenticated (check via `useAuth()` /
    `AuthContext`, established in Epic 2), clicking it should redirect
    to login (or show a prompt) rather than silently failing with a
    401 from the API — this mirrors how any other auth-gated action
    elsewhere in the app should behave; check for an existing pattern
    (e.g. how "Add to Wishlist" or a similar auth-gated action already
    handles this, if one exists, and match it).
  - On successful subscribe, change the button to a confirmed/disabled
    state (e.g. "We'll notify you!" with a checkmark), and persist that
    UI state for the remainder of the page view at minimum (a full
    "already subscribed" check on page load, calling some kind of GET
    endpoint, is NOT specified by the backend tasks above — Task
    4.1.2.2 only built subscribe/unsubscribe, no "check subscription
    status" GET endpoint — so don't invent a backend capability that
    doesn't exist; if this matters, flag it as a small missing backend
    task rather than building a frontend workaround that guesses at
    state).
  - Handle the 400 case where the variant is actually back in stock by
    the time the request lands (a genuine possible race in a live
    site) — show a message prompting the user to just add it to cart
    instead, rather than a generic error.

ACCEPTANCE CRITERIA
Manually verify: viewing a product with an out-of-stock variant selected
shows the "Notify Me" button instead of "Add to Cart"; clicking it while
logged in successfully calls the subscribe endpoint and updates the
button state; clicking it while logged out redirects to/prompts login.
Automated component tests are optional for this task given the PDP's
variant-selection UI maturity is uncertain at this point in the
project (per the CONTEXT note above) — if you're able to test the
"Notify Me" button in isolation as its own small component, do so
following this project's existing Vitest conventions; if it's too
entangled with in-progress PDP variant UI to cleanly isolate, skip
automated tests for this specific task and note why.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 4.1.1.1 | Create StockMovement audit log model | ☐ |
| 4.1.1.2 | Log stock movement on order creation | ☐ |
| 4.1.1.3 | Log stock movement on order cancellation | ☐ |
| 4.1.1.4 | Admin manual stock adjustment endpoint | ☐ |
| 4.1.1.5 | Low-stock threshold field + admin alert list | ☐ |
| 4.1.2.1 | StockAlertSubscription model | ☐ |
| 4.1.2.2 | "Notify me" subscribe/unsubscribe API | ☐ |
| 4.1.2.3 | Trigger notification when stock replenished | ☐ |
| 4.1.2.4 | Frontend "Notify Me" button on out-of-stock PDP | ☐ |

Once Epic 4 is fully merged, the next epic to generate prompts for is
**Epic 5 — Cart & Checkout**, which builds guest cart support and
checkout hardening directly on top of the variant-based `CartItem`
model this epic's StockMovement/alert system now fully audits.
