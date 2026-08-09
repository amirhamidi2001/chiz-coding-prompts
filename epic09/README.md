# Epic 9 — Coupons & Promotions — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–8 are fully merged. This epic is the direct payoff of a debt Epic 1 deliberately left behind: Task 1.2.1.1 stripped out the insecure, client-controlled `discount` field from checkout and hardcoded `discount=Decimal("0")` at the `OrderCreateSerializer.create()` call site, leaving the comment `# TODO: Epic 9 — replace with server-validated coupon discount` (this comment should still be present in the codebase at this point — find it). Also relevant: `order/services/pricing.py`'s `calculate_order_totals()` signature, per Epic 7 Task 7.1.1.4, is now `calculate_order_totals(subtotal, shipping_cost, discount=Decimal("0"))` — **discount is already a parameter of this function**; this epic does **not** need to change that function's signature again, only change **what value** gets passed as `discount` at the call site in `OrderCreateSerializer`. Keep this distinction clear throughout — several tasks below are scoped narrowly around that single insertion point rather than touching the pricing service itself.

---

## Phase 9.1 — Coupon Engine

### Feature 9.1.1 — Coupon Model & Validation

---

#### Task 9.1.1.1 — Create `promotions` app

```
You are working in backend/. Assume Epics 1–8 are fully merged.

CONTEXT
No `promotions` app exists. Following the exact same scaffolding
pattern established in Epic 6 Task 6.1.1.1 (`payments`) and Epic 7 Task
7.1.1.1 (`shipping`), this epic needs its own app.

TASK
Scaffold a new `promotions` Django app, registered and routed
identically to `payments`/`shipping`.

REQUIREMENTS
- Run `python manage.py startapp promotions` from within backend/ (or
  create the equivalent structure by hand), matching every other app's
  file layout.
- Name the config class `PromotionsConfig` in `promotions/apps.py`.
- Add `"promotions.apps.PromotionsConfig"` to `INSTALLED_APPS` in
  backend/core/settings/base.py, placed after
  `"shipping.apps.ShippingConfig"` (continuing this project's
  roughly-dependency-ordered app list).
- Create a minimal `promotions/urls.py`:
  ```python
  from django.urls import path

  app_name = "promotions"

  urlpatterns = []
  ```
- Register in backend/core/urls.py:
  `path("api/promotions/", include("promotions.urls")),`
  positioned after the `shipping.urls` include.
- Create `promotions/tests/__init__.py` as a package from the start,
  matching the convention established since Epic 1.

ACCEPTANCE CRITERIA / TESTS
- `python manage.py check` passes with no errors.
- A trivial smoke test confirming the app is registered, mirroring the
  equivalent tests from Epic 6/7's app-scaffolding tasks.
```

---

#### Task 9.1.1.2 — `Coupon` model

```
You are working in backend/promotions/models.py. Assume Task 9.1.1.1 is
already merged.

CONTEXT
There is no `Coupon` model anywhere in this codebase — this is the
central gap Epic 1 Task 1.2.1.1 identified and deliberately deferred to
this epic.

TASK
Create the `Coupon` model.

REQUIREMENTS
- Add:
  ```python
  from django.db import models
  from django.core.validators import MinValueValidator, MaxValueValidator
  from decimal import Decimal


  class Coupon(models.Model):
      class DiscountType(models.TextChoices):
          PERCENT = "percent", "Percentage"
          FIXED = "fixed", "Fixed Amount"

      code = models.CharField(max_length=32, unique=True, db_index=True)
      discount_type = models.CharField(max_length=10, choices=DiscountType.choices)
      value = models.DecimalField(
          max_digits=10, decimal_places=2,
          help_text="Percentage (0-100) if discount_type=percent, or a fixed currency amount if discount_type=fixed.",
      )
      min_order_amount = models.DecimalField(
          max_digits=10, decimal_places=2, default=Decimal("0"),
          help_text="Cart subtotal must be at least this amount for the coupon to apply.",
      )
      max_uses = models.PositiveIntegerField(
          null=True, blank=True,
          help_text="Total number of times this coupon can be redeemed across all users. Leave blank for unlimited.",
      )
      uses_per_user = models.PositiveIntegerField(
          default=1,
          help_text="Maximum number of times a single user can redeem this coupon.",
      )
      valid_from = models.DateTimeField()
      valid_until = models.DateTimeField()
      is_active = models.BooleanField(default=True)
      created_at = models.DateTimeField(auto_now_add=True)

      class Meta:
          ordering = ["-created_at"]

      def __str__(self):
          return self.code

      def clean(self):
          from django.core.exceptions import ValidationError
          if self.valid_until <= self.valid_from:
              raise ValidationError({"valid_until": "Must be after valid_from."})
          if self.discount_type == self.DiscountType.PERCENT and not (0 < self.value <= 100):
              raise ValidationError({"value": "Percentage discount must be between 0 and 100."})
          if self.discount_type == self.DiscountType.FIXED and self.value <= 0:
              raise ValidationError({"value": "Fixed discount amount must be greater than 0."})
  ```
  Normalize `code` to uppercase on save (a common, expected UX
  convenience for coupon codes — `SUMMER20` and `summer20` should be
  treated as the same code) by overriding `save()`:
  ```python
  def save(self, *args, **kwargs):
      self.code = self.code.upper().strip()
      super().save(*args, **kwargs)
  ```
- Generate the migration.
- Register in backend/promotions/admin.py as a normal editable admin
  form (this IS meant to be hand-managed by marketing/admin staff,
  unlike the audit-log-style models in prior epics):
  ```python
  @admin.register(Coupon)
  class CouponAdmin(admin.ModelAdmin):
      list_display = ("code", "discount_type", "value", "is_active", "valid_from", "valid_until", "max_uses")
      list_filter = ("discount_type", "is_active")
      search_fields = ("code",)
      ordering = ("-created_at",)
  ```

ACCEPTANCE CRITERIA / TESTS
Add model tests to backend/promotions/tests/test_models.py:
1. `code` is normalized to uppercase on save regardless of input case.
2. `clean()` rejects `valid_until <= valid_from`.
3. `clean()` rejects a `PERCENT` discount with `value` outside (0, 100].
4. `clean()` rejects a `FIXED` discount with `value <= 0`.
5. A valid coupon of each `discount_type` saves successfully.
```

---

#### Task 9.1.1.3 — `CouponRedemption` model (usage tracking)

```
You are working in backend/promotions/models.py. Assume Task 9.1.1.2 is
already merged.

CONTEXT
`Coupon.max_uses`/`uses_per_user` describe LIMITS, but nothing tracks
actual usage yet — without a redemption record, there's no way to
enforce either limit or to know which order a coupon was actually
applied to (needed for reporting and for correctly reversing a
redemption if the associated order is later cancelled — see the note
in Task 9.1.1.5 about this).

TASK
Create a `CouponRedemption` model.

REQUIREMENTS
- Add:
  ```python
  class CouponRedemption(models.Model):
      coupon = models.ForeignKey(Coupon, on_delete=models.CASCADE, related_name="redemptions")
      user = models.ForeignKey(
          settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True, blank=True,
          related_name="coupon_redemptions",
      )
      order = models.OneToOneField(
          "order.Order", on_delete=models.CASCADE, related_name="coupon_redemption",
          null=True, blank=True,
      )
      discount_amount = models.DecimalField(
          max_digits=10, decimal_places=2,
          help_text="Actual currency amount discounted by this redemption (frozen at redemption time).",
      )
      redeemed_at = models.DateTimeField(auto_now_add=True)

      class Meta:
          ordering = ["-redeemed_at"]

      def __str__(self):
          return f"{self.coupon.code} redeemed by {self.user or 'guest'}"
  ```
  Import `settings` from `django.conf` at the top. Note `user` is
  nullable/`SET_NULL` (a guest checkout, per Epic 5's guest cart
  support, may not have an authenticated user at redemption time —
  whether guest checkout should even be ALLOWED to use coupons is a
  real design question worth flagging: `uses_per_user` enforcement is
  meaningless for an anonymous user with no stable identity across
  sessions, since a guest could just use a different browser/session
  to bypass a per-user limit. Decide and document: either (a) restrict
  coupon usage to authenticated checkouts only, enforced in the
  coupon-application endpoint (Task 9.1.1.6), which is the simpler,
  more defensible choice given the per-user-limit enforcement problem,
  or (b) allow guest coupon use but accept that `uses_per_user` can't
  be reliably enforced for guests and only `max_uses` (the global cap)
  applies to them. Choose option (a) — authenticated-only coupon
  redemption — unless you have a strong reason not to, since it avoids
  building a limit that's trivially bypassable, and note this decision
  clearly since it affects Task 9.1.1.6's endpoint design.
  `order` is nullable because a redemption is initially created when a
  coupon is APPLIED to a cart (before checkout completes), not only
  once an order exists — Task 9.1.1.6 discusses this lifecycle in
  detail.
- Generate the migration.
- Register in promotions/admin.py as a read-only audit view (similar
  to `StockMovement`/`PaymentTransaction` from prior epics — usage
  history shouldn't be hand-edited):
  ```python
  @admin.register(CouponRedemption)
  class CouponRedemptionAdmin(admin.ModelAdmin):
      list_display = ("coupon", "user", "order", "discount_amount", "redeemed_at")
      list_filter = ("coupon",)
      search_fields = ("coupon__code", "user__email", "order__order_number")
      readonly_fields = [f.name for f in CouponRedemption._meta.fields]

      def has_add_permission(self, request):
          return False

      def has_change_permission(self, request, obj=None):
          return False
  ```

ACCEPTANCE CRITERIA / TESTS
Add a model test confirming a `CouponRedemption` can be created with
`order=None` (representing an in-progress, not-yet-completed checkout)
and later have `order` set once checkout completes, and that `str()`
works correctly for both an authenticated user and `user=None`.
```

---

#### Task 9.1.1.4 — Coupon validation service

```
You are working in backend/promotions/services.py (new file). Assume
Task 9.1.1.3 is already merged.

CONTEXT
No validation logic exists yet — `Coupon.max_uses`/`uses_per_user`/
`valid_from`/`valid_until`/`min_order_amount`/`is_active` are all just
inert fields right now.

TASK
Build a `validate_coupon()` function that checks a coupon against every
one of its constraints for a given user and cart subtotal, returning a
clear result (valid + discount amount, or a specific reason for
rejection).

REQUIREMENTS
- Implement:
  ```python
  from dataclasses import dataclass
  from decimal import Decimal
  from django.utils import timezone
  from .models import Coupon, CouponRedemption


  @dataclass
  class CouponValidationResult:
      valid: bool
      coupon: "Coupon | None" = None
      discount_amount: Decimal = Decimal("0")
      error: str = ""


  def validate_coupon(code: str, user, subtotal: Decimal) -> CouponValidationResult:
      try:
          coupon = Coupon.objects.get(code=code.upper().strip())
      except Coupon.DoesNotExist:
          return CouponValidationResult(valid=False, error="Invalid coupon code.")

      if not coupon.is_active:
          return CouponValidationResult(valid=False, error="This coupon is no longer active.")

      now = timezone.now()
      if now < coupon.valid_from:
          return CouponValidationResult(valid=False, error="This coupon is not yet valid.")
      if now > coupon.valid_until:
          return CouponValidationResult(valid=False, error="This coupon has expired.")

      if subtotal < coupon.min_order_amount:
          return CouponValidationResult(
              valid=False,
              error=f"This coupon requires a minimum order of {coupon.min_order_amount}.",
          )

      if coupon.max_uses is not None:
          total_redemptions = CouponRedemption.objects.filter(coupon=coupon, order__isnull=False).count()
          if total_redemptions >= coupon.max_uses:
              return CouponValidationResult(valid=False, error="This coupon has reached its usage limit.")

      if user is None or not user.is_authenticated:
          return CouponValidationResult(valid=False, error="Please log in to use a coupon code.")

      user_redemptions = CouponRedemption.objects.filter(
          coupon=coupon, user=user, order__isnull=False
      ).count()
      if user_redemptions >= coupon.uses_per_user:
          return CouponValidationResult(valid=False, error="You have already used this coupon.")

      if coupon.discount_type == Coupon.DiscountType.PERCENT:
          discount_amount = (subtotal * coupon.value / Decimal("100")).quantize(Decimal("0.01"))
      else:
          discount_amount = min(coupon.value, subtotal)  # never discount more than the subtotal itself

      return CouponValidationResult(valid=True, coupon=coupon, discount_amount=discount_amount)
  ```
  Note the `order__isnull=False` filter on redemption-counting queries
  — this deliberately only counts COMPLETED redemptions (ones actually
  tied to a finished order) toward the usage limits, not
  in-progress/abandoned "applied to cart but never checked out"
  redemption rows that Task 9.1.1.6 will create — an abandoned cart
  with a coupon applied should not permanently consume that user's
  single allowed use.
  Note the authenticated-only requirement here directly implements the
  decision made in Task 9.1.1.3's context about guest coupon usage —
  keep this consistent with whatever you actually decided there.
  Note `min(coupon.value, subtotal)` for fixed-amount discounts
  prevents a $50-off coupon from producing a negative total on a $20
  order — capping the discount at the subtotal itself.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/promotions/tests/test_services.py covering EVERY
rejection branch individually plus the success path:
1. Nonexistent code → invalid, correct error.
2. Inactive coupon → invalid.
3. Not-yet-valid (before `valid_from`) → invalid.
4. Expired (after `valid_until`) → invalid.
5. Subtotal below `min_order_amount` → invalid.
6. `max_uses` reached (create the right number of completed
   `CouponRedemption` rows first) → invalid; confirm an in-progress
   redemption with `order=None` does NOT count toward this limit.
7. Unauthenticated user (`user=None` or `AnonymousUser`) → invalid with
   the "please log in" message.
8. `uses_per_user` already reached for this specific user → invalid;
   confirm a DIFFERENT user can still successfully use the same coupon.
9. A valid percentage coupon computes the correct `discount_amount`,
   correctly rounded to 2 decimal places.
10. A valid fixed-amount coupon computes the correct `discount_amount`,
    and is correctly CAPPED at the subtotal when the fixed value
    exceeds it (e.g. a $50 coupon against a $20 subtotal produces
    `discount_amount == Decimal("20.00")`, not `50.00`).
```

---

#### Task 9.1.1.5 — Wire coupon service into `PricingService`

```
You are working in backend/order/serializers.py. Assume Task 9.1.1.4 is
already merged. Read the CONTEXT at the top of this document again
before starting — this task is narrower than it might sound.

CONTEXT — WHAT ALREADY EXISTS, DON'T REBUILD IT
`order/services/pricing.py`'s `calculate_order_totals(subtotal,
shipping_cost, discount=Decimal("0"))` ALREADY accepts a `discount`
parameter (added in Epic 7 Task 7.1.1.4) — nothing about that
function's signature or internal math needs to change in this task.
The ONLY thing that needs to change is: `OrderCreateSerializer.create()`
currently calls it with a HARDCODED `discount=Decimal("0")` (per Epic 1
Task 1.2.1.1's deliberate temporary fix, marked with a
`# TODO: Epic 9 — replace with server-validated coupon discount`
comment) — find that exact call site and that exact comment in the
current codebase before making any changes.

TASK
Replace the hardcoded `discount=Decimal("0")` with a real,
server-validated discount derived from a coupon the customer applied to
their cart (per Task 9.1.1.6's cart-level coupon application, which
this task assumes exists — build Task 9.1.1.6 first if you're doing
these out of order, though the documented order in this file already
has 9.1.1.5 before 9.1.1.6; if you'd rather sequence them the other
way in practice, that's a reasonable call, just be aware of the
dependency either way).

REQUIREMENTS
- Assume `Cart` (per Task 9.1.1.6, built either just before or just
  after this task) has a `coupon` field:
  `coupon = models.ForeignKey("promotions.Coupon", on_delete=models.SET_NULL, null=True, blank=True)`
  (added as part of Task 9.1.1.6 — if you're implementing this task
  first, add this field now as a prerequisite and note it in your
  summary so Task 9.1.1.6 doesn't try to re-add it).
- In `OrderCreateSerializer.validate()`, after resolving `cart` (the
  existing logic already does this), if `cart.coupon` is set, RE-VALIDATE
  it at this point (never trust that a coupon validated correctly when
  it was first applied to the cart is still valid now — time may have
  passed, the coupon's usage limit could have been hit by another
  concurrent checkout, etc., following the exact same "always
  re-validate at final checkout, don't just trust an earlier check"
  principle already established for stock in Epic 5 Task 5.2.1.1 and
  price in Epic 5 Task 5.2.1.2):
  ```python
  from promotions.services import validate_coupon

  # inside validate(), after cart is resolved:
  discount = Decimal("0")
  coupon_result = None
  if cart.coupon:
      coupon_result = validate_coupon(cart.coupon.code, user, cart.subtotal)
      if not coupon_result.valid:
          raise serializers.ValidationError({"coupon": coupon_result.error})
      discount = coupon_result.discount_amount
  attrs["discount"] = discount
  attrs["coupon_result"] = coupon_result
  ```
  Import `validate_coupon` from `promotions.services` at the top of
  order/serializers.py (check for circular-import risk between `order`
  and `promotions` apps — `promotions/models.py`'s `CouponRedemption`
  already has a string-reference FK to `"order.Order"`, which is the
  safe direction; importing `promotions.services` INTO `order` is the
  reverse direction and should be fine as long as `promotions/services.py`
  itself doesn't import anything from `order` at module level — confirm
  this holds).
- In `create()`, replace the hardcoded
  `calculate_order_totals(subtotal=..., shipping_cost=..., discount=Decimal("0"))`
  call with
  `calculate_order_totals(subtotal=..., shipping_cost=..., discount=attrs["discount"])`
  (or however the exact existing call site reads — just replace the
  hardcoded zero with the validated value from `validate()`).
- After the `Order` is successfully created (inside the same
  `transaction.atomic()` block already wrapping order creation), if a
  coupon was applied, create the `CouponRedemption` row and link it to
  the now-existing order:
  ```python
  if attrs.get("coupon_result") and attrs["coupon_result"].valid:
      CouponRedemption.objects.create(
          coupon=attrs["coupon_result"].coupon,
          user=order.user,
          order=order,
          discount_amount=attrs["coupon_result"].discount_amount,
      )
  ```
  Import `CouponRedemption` from `promotions.models`.
- Clear `cart.coupon` when the cart is cleared post-checkout (the
  existing `cart.items.all().delete()` call — or wherever cart clearing
  actually happens post-Epic-6's cart-clearing-timing discussion from
  Epic 6 Task 6.4.1.2, which may have MOVED cart-clearing to only
  happen on confirmed payment success rather than at order-creation
  time — check which is actually true in the current codebase and
  clear `cart.coupon = None; cart.save()` at whichever point cart items
  actually get cleared, so a stale coupon reference doesn't linger on
  an empty cart).
- Remove the now-outdated `# TODO: Epic 9 — replace with...` comment,
  since this task is exactly that replacement.
- Handle order cancellation: if an order with an associated
  `CouponRedemption` is later cancelled (via Task 8.1.1.1's
  `cancel_order()` service from Epic 8), should the coupon be
  "returned" to the user (i.e. should that redemption no longer count
  toward their `uses_per_user` limit)? This is a genuine business
  decision, not a technical detail — cancelling an order the coupon was
  used on arguably should free up that usage, especially since Epic 8's
  `cancel_order()` already fully reverses the order's other effects
  (stock restoration). Implement this: in `order/services/cancellation.py`'s
  `cancel_order()` function (from Epic 8 Task 8.1.1.1), after the
  existing cancellation logic, check for and DELETE (not just flag) any
  associated `CouponRedemption` for the cancelled order — deleting the
  redemption record entirely (rather than keeping a "cancelled"
  redemption row) is the simplest way to make it correctly NOT count
  toward `validate_coupon()`'s `order__isnull=False` usage-counting
  queries going forward, since a deleted row obviously can't be
  counted. Import `CouponRedemption` from `promotions.models` into
  `order/services/cancellation.py` (check for circular import risk;
  same direction/reasoning as above, should be safe).

ACCEPTANCE CRITERIA / TESTS
Add tests to the order test suite:
1. Checkout with a valid coupon applied to the cart results in
   `Order.discount` matching the coupon's computed discount amount, and
   a `CouponRedemption` row created linking the coupon, user, and
   order.
2. Checkout where the coupon has become invalid BETWEEN being applied
   to the cart and final checkout submission (e.g. simulate by
   exhausting `max_uses` via other redemptions in between) is rejected
   with a 400 identifying the coupon problem, and the order is NOT
   created.
3. Checkout with NO coupon applied continues to work exactly as before
   (`discount=Decimal("0")`, regression check).
4. Cancelling an order that had a coupon applied removes the
   `CouponRedemption` row, and the SAME user can subsequently use the
   SAME coupon again (up to their `uses_per_user` limit) — proving the
   "cancellation frees up the coupon" behavior works end-to-end.
5. Re-run the FULL order test suite from every prior epic and confirm
   no regression — this call site has now been touched by Epics 1, 7,
   and 9, so this is a good moment to confirm the whole checkout flow
   still works coherently end-to-end.
```

---

#### Task 9.1.1.6 — `POST /api/cart/apply-coupon/` endpoint

```
You are working in backend/cart/models.py, views.py, serializers.py,
urls.py. Assume Task 9.1.1.4 (coupon validation service) is already
merged, and Epic 5's guest-cart support (`get_or_create_cart(request)`)
is in place.

CONTEXT
Nothing lets a customer attach a coupon to their cart before checkout
yet — Task 9.1.1.5 assumes `cart.coupon` exists and gets validated at
final checkout, but the actual "type a code into a box and see the
discount applied" step happens here, at the cart level, before the
customer ever reaches the checkout form.

TASK
Add `Cart.coupon`, and `POST`/`DELETE /api/cart/apply-coupon/`
endpoints to apply/remove a coupon from the current cart.

REQUIREMENTS
- Add to `Cart` (backend/cart/models.py):
  `coupon = models.ForeignKey("promotions.Coupon", on_delete=models.SET_NULL, null=True, blank=True)`
  (per the note in Task 9.1.1.5 — if that task already added this
  field, skip re-adding it here and just confirm it exists; whichever
  task runs first in your actual implementation order should add it
  once).
  Generate the migration if not already done.
- Add a computed property to `Cart` exposing the discount preview
  (useful for the frontend to show "coupon applied: -$5.00" without a
  separate round-trip):
  ```python
  @property
  def coupon_discount_preview(self):
      if not self.coupon:
          return Decimal("0")
      from promotions.services import validate_coupon
      # NOTE: this is a best-effort PREVIEW using whatever user context
      # is available; it intentionally does NOT re-run the full
      # authentication/usage-limit checks tied to a specific request —
      # those are re-validated authoritatively at actual checkout time
      # (Task 9.1.1.5). This property is for cart-page DISPLAY only.
      ...
  ```
  Actually, on reflection: a model `@property` doesn't have access to
  `request.user` at all, and `validate_coupon()` genuinely needs a
  specific user to check `uses_per_user` correctly — this preview
  can't live cleanly as a bare model property. Instead, compute the
  discount preview in the VIEW/serializer layer (where `request.user`
  is available), not as a `Cart` model property — don't force this
  into the model; expose it via the cart-retrieval serializer instead
  (see below).
- Create `CartApplyCouponSerializer` in cart/serializers.py:
  ```python
  class CartApplyCouponSerializer(serializers.Serializer):
      code = serializers.CharField(max_length=32)
  ```
- Add a view in cart/views.py:
  ```python
  class CartApplyCouponView(APIView):
      permission_classes = [AllowAny]

      def post(self, request):
          serializer = CartApplyCouponSerializer(data=request.data)
          serializer.is_valid(raise_exception=True)
          cart = get_or_create_cart(request)
          result = validate_coupon(
              serializer.validated_data["code"], request.user, cart.subtotal
          )
          if not result.valid:
              return Response({"coupon": result.error}, status=status.HTTP_400_BAD_REQUEST)
          cart.coupon = result.coupon
          cart.save(update_fields=["coupon"])
          return Response(
              {"code": result.coupon.code, "discount_amount": str(result.discount_amount)},
              status=status.HTTP_200_OK,
          )

      def delete(self, request):
          cart = get_or_create_cart(request)
          cart.coupon = None
          cart.save(update_fields=["coupon"])
          return Response(status=status.HTTP_204_NO_CONTENT)
  ```
  Import `validate_coupon` from `promotions.services`,
  `get_or_create_cart` from within the same `cart/views.py` module
  (already defined there per Epic 5).
  Register the URL: `path("apply-coupon/", views.CartApplyCouponView.as_view(), name="apply-coupon"),`
  in cart/urls.py.
- Update whatever serializer already returns full cart contents (the
  existing `CartSerializer`, used by `GET /api/cart/`) to include the
  applied coupon's code and a live-recomputed discount preview (calling
  `validate_coupon()` again at serialization time, using
  `self.context["request"].user`, so the cart display always reflects
  CURRENT validity — e.g. if a coupon expired since being applied, the
  cart view should reflect that rather than showing a stale discount):
  ```python
  # in CartSerializer
  coupon_code = serializers.SerializerMethodField()
  coupon_discount = serializers.SerializerMethodField()
  coupon_error = serializers.SerializerMethodField()

  def _coupon_check(self, obj):
      if not obj.coupon:
          return None
      request = self.context.get("request")
      user = request.user if request else None
      return validate_coupon(obj.coupon.code, user, obj.subtotal)

  def get_coupon_code(self, obj):
      return obj.coupon.code if obj.coupon else None

  def get_coupon_discount(self, obj):
      result = self._coupon_check(obj)
      return str(result.discount_amount) if result and result.valid else "0"

  def get_coupon_error(self, obj):
      result = self._coupon_check(obj)
      return result.error if result and not result.valid else None
  ```
  (this calls `_coupon_check` up to 3 times per serialization if all
  three fields are accessed — acceptable given `validate_coupon()` is a
  handful of cheap DB queries, not a network call, but if you want to
  optimize, cache the result on `self` within one serialization pass;
  use your judgment on whether that optimization is worth the added
  complexity for this task).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/cart/tests/:
1. Applying a valid coupon code to the cart succeeds and sets
   `cart.coupon`.
2. Applying an invalid/expired/unauthenticated-required coupon code is
   rejected with 400 and `cart.coupon` remains unchanged.
3. Removing a coupon via DELETE clears `cart.coupon`.
4. `GET /api/cart/` after applying a coupon includes the correct
   `coupon_code`/`coupon_discount` in its response.
5. `GET /api/cart/` for a cart whose applied coupon has SINCE become
   invalid (e.g. expired) shows `coupon_error` populated and
   `coupon_discount` as `"0"`, without erroring the whole cart-fetch
   request.
6. Works correctly for both authenticated and guest carts (though per
   Task 9.1.1.3/9.1.1.4's authenticated-only coupon decision, a GUEST
   attempting to apply any coupon should get the "please log in" 400
   from `validate_coupon` — confirm this specific behavior is what
   actually happens end-to-end through this endpoint).
```

---

#### Task 9.1.1.7 — Coupon-restricted-to-category/product support

```
You are working in backend/promotions/models.py, services.py. Assume
Task 9.1.1.4 is already merged.

CONTEXT
Real-world coupon campaigns are frequently scoped to specific
categories or products (e.g. "20% off all skincare" or "$5 off this
specific serum") rather than being universally applicable — the current
`Coupon` model has no such restriction mechanism.

TASK
Add optional category/product restriction to `Coupon`, and update
`validate_coupon()` to compute the discount only against matching cart
items when restrictions are present.

REQUIREMENTS
- Add to `Coupon`:
  ```python
  categories = models.ManyToManyField(
      "shop.Category", blank=True,
      help_text="If set, this coupon only applies to items in these categories. Leave empty to apply to all categories.",
  )
  products = models.ManyToManyField(
      "shop.Product", blank=True,
      help_text="If set, this coupon only applies to these specific products. Leave empty (with no category restriction either) to apply cart-wide.",
  )
  ```
  Generate the migration.
- This is a REAL, non-trivial change to how discount is computed:
  `validate_coupon()` currently takes a flat `subtotal: Decimal` and
  discounts against the WHOLE cart. With restrictions, the discount
  must be computed against only the ELIGIBLE portion of the cart. This
  requires changing `validate_coupon()`'s signature to accept the
  actual cart (or its items) rather than just a subtotal number:
  ```python
  def validate_coupon(code: str, user, cart) -> CouponValidationResult:
      ...
      # after all the existing non-amount-related checks (active, dates,
      # authentication, usage limits) pass, compute the ELIGIBLE subtotal:
      eligible_items = cart.items.select_related("variant__product__category")
      if coupon.categories.exists() or coupon.products.exists():
          eligible_items = eligible_items.filter(
              models.Q(variant__product__category__in=coupon.categories.all())
              | models.Q(variant__product__in=coupon.products.all())
          )
      eligible_subtotal = sum(item.subtotal for item in eligible_items) or Decimal("0")

      if eligible_subtotal == 0:
          return CouponValidationResult(valid=False, error="This coupon does not apply to any items in your cart.")

      if eligible_subtotal < coupon.min_order_amount:
          return CouponValidationResult(
              valid=False,
              error=f"This coupon requires a minimum eligible order of {coupon.min_order_amount}.",
          )

      # discount calculation now uses eligible_subtotal instead of the
      # full cart subtotal, everywhere it previously used `subtotal`
      ...
  ```
  This is a genuine breaking signature change from `validate_coupon(code, user, subtotal)`
  to `validate_coupon(code, user, cart)` — go back and update EVERY
  existing call site from Tasks 9.1.1.5 and 9.1.1.6 to pass the actual
  `cart` object instead of `cart.subtotal`, and update every existing
  test from Task 9.1.1.4/9.1.1.5/9.1.1.6 accordingly. This ripple
  effect is expected and correct, not a sign something went wrong —
  restricted coupons fundamentally cannot be validated from a bare
  subtotal number alone, so the signature HAD to change once
  restrictions were introduced; there was no way to add this
  capability without touching every call site.
- Register `categories`/`products` on `CouponAdmin` (Task 9.1.1.2) using
  `filter_horizontal` for a usable multi-select widget:
  `filter_horizontal = ("categories", "products")`

ACCEPTANCE CRITERIA / TESTS
Add tests:
1. A coupon with no category/product restriction still discounts the
   FULL cart subtotal (regression check against the pre-restriction
   behavior).
2. A coupon restricted to a specific category only discounts items in
   that category — verify with a cart containing BOTH matching and
   non-matching items, confirming the discount amount reflects only the
   matching items' subtotal.
3. A coupon restricted to a category the cart has NO items in returns
   `valid=False` with the "does not apply to any items" message.
4. A coupon restricted to specific PRODUCTS (not categories) works
   correctly and independently of the category restriction path.
5. A coupon restricted to BOTH a category AND specific products
   correctly includes items matching EITHER condition (the `Q(...) | Q(...)`
   OR logic).
6. Re-run every test from Tasks 9.1.1.4, 9.1.1.5, and 9.1.1.6 after
   updating their call sites to the new signature, confirming zero
   regressions.
```

---

#### Task 9.1.1.8 — Admin coupon CRUD UI

```
You are working in backend/dashboard/ (serializers.py, views.py,
urls.py) and frontend/src (admin pages). Assume Task 9.1.1.7 is
already merged.

CONTEXT
`Coupon` currently only has the plain Django admin (`CouponAdmin`, Task
9.1.1.2) — per this project's established pattern (e.g.
`AdminOrderViewSet`, `AdminProductViewSet` from prior epics' grounding),
the REAL admin experience for this platform is a custom React admin
dashboard consuming the `dashboard` app's REST API, not Django's raw
built-in admin — marketing/admin staff managing coupon campaigns should
use that same consistent React admin UI, not a separate Django admin
screen.

TASK
Add `AdminCouponViewSet` to the `dashboard` app's API, and build a
corresponding React admin page for listing/creating/editing/deactivating
coupons.

REQUIREMENTS — backend
- Create `AdminCouponSerializer` in backend/dashboard/serializers.py:
  ```python
  from promotions.models import Coupon

  class AdminCouponSerializer(serializers.ModelSerializer):
      class Meta:
          model = Coupon
          fields = [
              "id", "code", "discount_type", "value", "min_order_amount",
              "max_uses", "uses_per_user", "valid_from", "valid_until",
              "is_active", "categories", "products", "created_at",
          ]
  ```
- Add `AdminCouponViewSet` to backend/dashboard/views.py, mirroring the
  exact structure of `AdminOrderViewSet`/other existing
  `Admin*ViewSet`s in this file (permission class, pagination class,
  filter backends):
  ```python
  class AdminCouponViewSet(viewsets.ModelViewSet):
      permission_classes = [IsAdminOrSuperuser]
      pagination_class = DashboardPagination
      serializer_class = AdminCouponSerializer
      filter_backends = [filters.SearchFilter, filters.OrderingFilter]
      search_fields = ["code"]
      ordering_fields = ["created_at", "valid_from", "valid_until"]
      ordering = ["-created_at"]

      def get_queryset(self):
          return Coupon.objects.prefetch_related("categories", "products")
  ```
  Import `Coupon` from `promotions.models` at the top.
- Register in backend/dashboard/urls.py:
  `router.register(r"admin/coupons", views.AdminCouponViewSet, basename="admin-coupon")`

REQUIREMENTS — frontend
- Add API methods to frontend/src/services/api.js (matching the
  existing admin-API method conventions in that file):
  ```javascript
  listAdminCoupons: (params) => api.get('/dashboard/admin/coupons/', { params }),
  createAdminCoupon: (data) => api.post('/dashboard/admin/coupons/', data),
  updateAdminCoupon: (id, data) => api.patch(`/dashboard/admin/coupons/${id}/`, data),
  deleteAdminCoupon: (id) => api.delete(`/dashboard/admin/coupons/${id}/`),
  ```
- Build an admin coupon-management page (check where other admin pages
  live in frontend/src — likely a dedicated admin section/route
  structure established by prior admin-dashboard work) with: a list
  view showing code/type/value/active status/valid date range with
  search; a create/edit form with fields for all `Coupon` attributes
  including a category/product multi-select (fetch available
  categories/products via the existing storefront-facing or admin
  product/category list endpoints); a quick "deactivate" toggle action
  on the list (PATCH `is_active=false`) for fast campaign shutoff
  without needing to open the full edit form.
- Add client-side validation mirroring the backend's `clean()` rules
  from Task 9.1.1.2 (percent value 0-100, valid_until after valid_from)
  for immediate feedback before submission, while still relying on the
  backend as the authoritative source of truth (the backend will reject
  invalid data regardless — this is a UX nicety, not a security
  boundary).

ACCEPTANCE CRITERIA / TESTS
Add backend tests to backend/dashboard/tests/test_views.py: full CRUD
cycle through `AdminCouponViewSet` (create, list, retrieve, update,
deactivate) as an admin user; a non-admin user gets 403 on all of
these. Add frontend component tests for the coupon list/form covering:
renders existing coupons, form validation catches obviously invalid
input before submission, deactivate action calls the correct PATCH
endpoint.
```

---

#### Task 9.1.1.9 — Frontend "apply coupon" UI in cart/checkout

```
You are working in frontend/src (CartContext, Cart page, Checkout
page). Assume Task 9.1.1.6 is already merged.

CONTEXT
The backend can now accept/validate/remove a coupon on the cart, and
`GET /api/cart/` returns `coupon_code`/`coupon_discount`/`coupon_error`
(per Task 9.1.1.6's serializer additions) — nothing on the frontend
surfaces any of this yet.

TASK
Add a coupon-code input to the cart page (and/or checkout page — your
call on which is the better UX location, though the cart page is
arguably the more natural place since it's where the customer already
sees subtotal/total before proceeding to checkout) with apply/remove
actions and live discount display.

REQUIREMENTS
- Add API methods to frontend/src/services/api.js:
  ```javascript
  applyCoupon: (code) => api.post('/cart/apply-coupon/', { code }),
  removeCoupon: () => api.delete('/cart/apply-coupon/'),
  ```
- Add a coupon input component to the cart page: a text input + "Apply"
  button; on success, show the applied code with a discount amount and
  a "Remove" action; on failure, show the specific error message
  returned by the backend (e.g. "This coupon has expired." — the
  backend already provides specific, user-meaningful messages per Task
  9.1.1.4, so surface them directly rather than a generic "invalid
  coupon" fallback).
- Update `CartContext` (or wherever cart state/totals are managed) to
  refetch/update cart data after a successful apply/remove, so the
  displayed subtotal/discount/total reflect the change immediately
  without requiring a full page reload.
- On the checkout page, display the applied coupon's code and discount
  amount as a read-only line item in the order summary (consistent with
  how shipping cost, tax, etc. are presumably already displayed there)
  — checkout itself doesn't need its OWN apply-coupon UI if the cart
  page already handles application, but the checkout summary should
  clearly show the discount is being applied to the order about to be
  placed.
- Handle the case where `coupon_error` is present in the cart response
  (per Task 9.1.1.6 — a previously-applied coupon that's since become
  invalid): show this clearly rather than silently ignoring it, with an
  obvious way to remove the now-invalid coupon.

ACCEPTANCE CRITERIA
Manually verify: applying a valid coupon on the cart page updates the
displayed total immediately; an invalid code shows a clear error and
doesn't apply; removing an applied coupon restores the original total;
the checkout page correctly reflects the applied discount in its
summary and in the final submitted order. Add component tests for the
coupon-input component covering apply success, apply failure (specific
error message rendering), and remove actions.
```

---

## Phase 9.2 — Flash Sales & Campaigns

### Feature 9.2.1 — Time-Boxed Promotions

---

#### Task 9.2.1.1 — `FlashSale` model (products + discount % + time window)

```
You are working in backend/promotions/models.py. Assume Feature 9.1.1
is fully merged.

CONTEXT
Coupons (Feature 9.1.1) require the customer to know/enter a code.
Flash sales are a DIFFERENT, simpler promotional mechanism: a
time-boxed, automatically-applied discount on specific products,
requiring no code entry — the discounted price should just show up on
the product listing/detail page while the sale is active, matching how
Sephora/Ulta/YesStyle-style "flash sale" sections typically work.

TASK
Create a `FlashSale` model.

REQUIREMENTS
- Add:
  ```python
  class FlashSale(models.Model):
      name = models.CharField(max_length=200)
      products = models.ManyToManyField("shop.Product", related_name="flash_sales")
      discount_percent = models.DecimalField(
          max_digits=5, decimal_places=2,
          validators=[MinValueValidator(Decimal("0.01")), MaxValueValidator(Decimal("100"))],
      )
      starts_at = models.DateTimeField()
      ends_at = models.DateTimeField()
      is_active = models.BooleanField(default=True)
      created_at = models.DateTimeField(auto_now_add=True)

      class Meta:
          ordering = ["-starts_at"]

      def __str__(self):
          return self.name

      def clean(self):
          from django.core.exceptions import ValidationError
          if self.ends_at <= self.starts_at:
              raise ValidationError({"ends_at": "Must be after starts_at."})

      @property
      def is_currently_active(self):
          from django.utils import timezone
          now = timezone.now()
          return self.is_active and self.starts_at <= now <= self.ends_at
  ```
  Import `MinValueValidator`, `MaxValueValidator` from
  `django.core.validators`, `Decimal` from `decimal`, at the top of
  models.py alongside the other imports already added by Feature 9.1.1.
- Generate the migration.
- Register in promotions/admin.py:
  ```python
  @admin.register(FlashSale)
  class FlashSaleAdmin(admin.ModelAdmin):
      list_display = ("name", "discount_percent", "starts_at", "ends_at", "is_active")
      list_filter = ("is_active",)
      filter_horizontal = ("products",)
  ```

ACCEPTANCE CRITERIA / TESTS
Add model tests: `is_currently_active` correctly returns `True` only
when `is_active=True` AND the current time falls within
`[starts_at, ends_at]`, `False` for a not-yet-started, already-ended, or
manually-deactivated sale; `clean()` rejects `ends_at <= starts_at`.
```

---

#### Task 9.2.1.2 — Flash sale price resolution in product serializer

```
You are working in backend/shop/serializers.py. Assume Task 9.2.1.1 is
already merged.

CONTEXT
`ProductListSerializer`/`ProductDetailSerializer` (extended across
Epic 3's cosmetics-attribute work) have no awareness of flash sales at
all — a product's displayed price is always its plain
`ProductVariant.price`/`original_price`, regardless of any active
`FlashSale`.

TASK
Add flash-sale-aware price fields to the product serializers.

REQUIREMENTS
- Add a helper, e.g. in backend/shop/models.py on `Product` itself (or
  `ProductVariant`, since price actually lives there per Epic 3 — the
  discount conceptually applies at the PRODUCT level per `FlashSale.products`,
  but must resolve down to a VARIANT-level discounted price since
  that's where price actually lives):
  ```python
  # on ProductVariant:
  @property
  def active_flash_sale(self):
      from django.utils import timezone
      now = timezone.now()
      return self.product.flash_sales.filter(
          is_active=True, starts_at__lte=now, ends_at__gte=now
      ).first()

  @property
  def effective_price(self):
      sale = self.active_flash_sale
      if sale:
          discount_multiplier = (Decimal("100") - sale.discount_percent) / Decimal("100")
          return (self.price * discount_multiplier).quantize(Decimal("0.01"))
      return self.price
  ```
  Import `Decimal` at the top of shop/models.py if not already present.
  Note: if MULTIPLE active flash sales somehow apply to the same
  product simultaneously (an admin configuration edge case — two
  overlapping sales both including the same product), `.first()`
  picks an arbitrary one based on default ordering (`-starts_at`, so
  the most recently STARTED sale wins) — this is a reasonable,
  simple tie-breaking rule; document it in a comment rather than
  building more complex "highest discount wins" logic unless there's a
  clear reason to prefer that instead.
- Update `ProductVariantSerializer` (from Epic 3 Task 3.2.1.14) to
  expose the sale-aware price:
  ```python
  effective_price = serializers.DecimalField(source="effective_price", max_digits=10, decimal_places=2, read_only=True)
  is_on_flash_sale = serializers.SerializerMethodField()
  flash_sale_ends_at = serializers.SerializerMethodField()

  def get_is_on_flash_sale(self, obj):
      return obj.active_flash_sale is not None

  def get_flash_sale_ends_at(self, obj):
      sale = obj.active_flash_sale
      return sale.ends_at if sale else None
  ```
  (add `flash_sale_ends_at` specifically so the frontend can render a
  countdown timer per Task 9.2.1.3, without needing a separate API call
  just to find out when the current sale ends).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_serializers.py and/or
test_models.py:
1. A variant with an active flash sale returns `effective_price`
   correctly discounted, `is_on_flash_sale=True`, and the correct
   `flash_sale_ends_at`.
2. A variant with NO active flash sale (none exists, or one exists but
   is outside its time window, or `is_active=False`) returns
   `effective_price == price` (unchanged) and `is_on_flash_sale=False`.
3. A variant belonging to a product with an UPCOMING (not yet started)
   flash sale is correctly treated as NOT currently on sale.
4. The tie-breaking behavior for overlapping sales (if you test this
   edge case) matches the documented `.first()`/`-starts_at` ordering
   rule.
```

---

#### Task 9.2.1.3 — Homepage flash-sale countdown banner

```
You are working in frontend/src (Home page component). Assume Task
9.2.1.2 is already merged.

CONTEXT
Nothing on the frontend surfaces flash sales at all — no banner, no
countdown, no dedicated sale-browsing view.

TASK
Add a homepage flash-sale banner with a live countdown timer, linking
to the products currently on sale.

REQUIREMENTS
- Add an API method to fetch active flash sales — check whether a
  general-purpose `GET /api/products/?is_on_flash_sale=true`-style
  filter is feasible given Task 9.2.1.2's serializer additions (if
  `ProductFilter`, from Epic 3 Task 3.2.1.15, can be extended with a
  flash-sale filter using a `method=` filter checking
  `active_flash_sale is not None` across the queryset — this requires
  a queryset-level annotation/filter, not just a serializer-level
  property, since `effective_price`/`is_on_flash_sale` are currently
  Python `@property`/`SerializerMethodField` values computed
  per-instance, NOT filterable at the database query level as written;
  either add a proper queryset-level filter (e.g. filtering
  `Product.objects.filter(flash_sales__is_active=True, flash_sales__starts_at__lte=now, flash_sales__ends_at__gte=now)`
  directly in a custom filter method) or add a simpler dedicated
  endpoint like `GET /api/promotions/active-flash-sales/` returning the
  currently-active `FlashSale` objects with their nested products —
  the dedicated-endpoint approach is likely simpler and more direct for
  a homepage banner's specific needs (which wants the SALE's metadata —
  name, end time — not just a filtered product list) — prefer building
  `GET /api/promotions/active-flash-sales/` in
  backend/promotions/views.py as a straightforward `ListAPIView` over
  `FlashSale.objects.filter(is_active=True, starts_at__lte=now, ends_at__gte=now)`,
  with a serializer including nested product summaries, rather than
  retrofitting `ProductFilter` for this specific homepage use case).
- Add the corresponding API method to frontend/src/services/api.js:
  `getActiveFlashSales: () => api.get('/promotions/active-flash-sales/'),`
- Build a homepage banner component: sale name, a live countdown to
  `ends_at` (update every second via `setInterval`/`useEffect`,
  clearing the interval on unmount to avoid memory leaks — a common
  React countdown-timer gotcha worth being deliberate about), and a
  link/button to view the sale's products (e.g. navigating to a
  filtered product list view, or a dedicated flash-sale landing
  section — whichever fits the existing site's navigation patterns
  better).
  Handle the no-active-sale case by simply not rendering the banner at
  all (rather than showing an empty/broken banner shell).
  Handle a sale ending WHILE the customer is viewing the countdown
  (i.e. the countdown reaches zero) by either hiding the banner or
  showing a "Sale has ended" state and re-fetching to check for any
  newly-active sale, rather than leaving a countdown frozen at
  `00:00:00` indefinitely.
- On product cards/listing pages (wherever `ProductListSerializer`
  output is already rendered elsewhere in the app), show a
  "flash sale" badge and the discounted `effective_price` (struck-through
  original price alongside it) for any product currently on sale, using
  the `is_on_flash_sale`/`effective_price` fields from Task 9.2.1.2 —
  check whether `ProductListSerializer` was updated in Task 9.2.1.2 as
  well (the task above only explicitly mentioned
  `ProductVariantSerializer` — extend `ProductListSerializer`
  similarly if it doesn't already surface variant-level sale data in a
  way the product-card component can use, since a product card
  typically shows PRODUCT-level info, and price/sale-status is
  genuinely VARIANT-level per this codebase's Epic 3 data model —
  decide how to surface "this product has at least one variant on
  sale, starting at $X" at the product-card level sensibly, since a
  product card can't reasonably show per-variant sale badges for every
  possible shade).

ACCEPTANCE CRITERIA
Manually verify: with an active `FlashSale` configured in the admin,
the homepage banner appears with a correctly counting-down timer, and
affected products show the discounted price/badge on listing pages.
With no active sale, the banner doesn't render. Add component tests
for the countdown banner covering: renders with sale data, doesn't
render with no active sale, countdown decrements correctly (use
Vitest fake timers, matching the pattern already established in Epic 2
Task 2.3.2.4's resend-timer tests), and interval cleanup on unmount
(assert no lingering timer/no state updates after unmount — a common
source of React console warnings/memory leaks worth explicitly
testing for).
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 9.1.1.1 | Create promotions Django app | ☐ |
| 9.1.1.2 | Coupon model | ☐ |
| 9.1.1.3 | CouponRedemption model | ☐ |
| 9.1.1.4 | Coupon validation service | ☐ |
| 9.1.1.5 | Wire coupon service into PricingService | ☐ |
| 9.1.1.6 | POST /api/cart/apply-coupon/ endpoint | ☐ |
| 9.1.1.7 | Coupon-restricted-to-category/product support | ☐ |
| 9.1.1.8 | Admin coupon CRUD UI | ☐ |
| 9.1.1.9 | Frontend apply-coupon UI | ☐ |
| 9.2.1.1 | FlashSale model | ☐ |
| 9.2.1.2 | Flash sale price resolution in serializer | ☐ |
| 9.2.1.3 | Homepage flash-sale countdown banner | ☐ |

Once Epic 9 is fully merged, the next epics to generate prompts for are
**Epic 10 — Reviews & Ratings** and **Epic 11 — Wishlist**, both
lower-risk epics that can run in parallel with each other once Epics 1
and 3 (already merged) are in place, per the master backlog's execution
order notes.
