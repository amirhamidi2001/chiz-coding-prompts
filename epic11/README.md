# Epic 11 — Wishlist — AI Coding Prompts

Repo: `tablogenix-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–10 are fully merged.

**Important grounded discovery for this epic — read before starting:** a full `Wishlist` model, `WishlistViewSet` (list/create/delete, CRUD-complete), `WishlistSerializer`/`WishlistProductSerializer` (with nested price/discount/thumbnail data), and duplicate-prevention validation **already exist** in `backend/dashboard/models.py`/`views.py`/`serializers.py`, registered at `/dashboard/wishlist/`. A corresponding `WishlistContext.jsx` + `WishlistTab.jsx` already exist on the frontend too. This matches the master backlog's own scoping exactly — Epic 11's tasks were never "build a wishlist," only "Wishlist Extensions" on top of one that already works. Nothing in this epic rebuilds that foundation; every task below is additive.

**One deliberate, existing design choice worth keeping in mind throughout:** `Wishlist.product` is a FK to `shop.Product` directly, **not** to `ProductVariant` (unlike `Cart`/`Order`, which became variant-based in Epic 3). This is arguably correct as-is for a wishlist — a customer typically wishlists "this foundation," not "this foundation in shade 320 specifically" — but it means Task 11.1.1.3 ("move to cart") has to bridge a real gap: cart requires a specific variant, wishlist only knows the product. That task addresses this explicitly rather than glossing over it.

---

## Phase 11.1 — Wishlist Enhancements

### Feature 11.1.1 — Wishlist Extensions

---

#### Task 11.1.1.1 — Guest wishlist (session-based, mirrors cart pattern)

```
You are working in backend/dashboard/models.py, views.py, and
frontend/src/context/WishlistContext.jsx. Assume Epic 5 Feature 5.1.1
(guest, session-based Cart) is already merged — this task applies the
EXACT SAME pattern to Wishlist, one epic later.

CONTEXT — READ EPIC 5's CART IMPLEMENTATION BEFORE STARTING
`Wishlist.user` is currently a required, non-nullable
`ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)`, and
`WishlistViewSet` has `permission_classes = [IsAuthenticated]` with
every method (`get_queryset`, `perform_create`, `get_object`) filtering
directly on `request.user`. This means, exactly like the cart before
Epic 5 Feature 5.1.1, a visitor must create an account before saving
even a single item to their wishlist — the same conversion-friction
problem Epic 5 already solved for the cart, not yet solved here.

TASK
Apply the identical nullable-user-plus-session_key pattern established
in Epic 5 Tasks 5.1.1.1/5.1.1.2 to `Wishlist`, and merge a guest's
session wishlist into their account on login, mirroring Epic 5 Task
5.1.1.3.

REQUIREMENTS
- Change `Wishlist.user` to nullable:
  `user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, null=True, blank=True, related_name="wishlist_items")`
  Add `session_key = models.CharField(max_length=64, null=True, blank=True, db_index=True)`.
  UNLIKE `Cart` (a `OneToOneField` to user, one cart per user), `Wishlist`
  is already a one-row-per-ITEM model (`unique_together = ("user", "product")`,
  many rows per user) — so the CheckConstraint pattern from Epic 5 Task
  5.1.1.1 (enforcing exactly one owner field set) needs adjusting to fit
  this per-item shape:
  ```python
  class Meta:
      unique_together = ("user", "product")  # existing — still correct for authenticated items
      constraints = [
          models.CheckConstraint(
              check=(
                  models.Q(user__isnull=False, session_key__isnull=True)
                  | models.Q(user__isnull=True, session_key__isnull=False)
              ),
              name="wishlist_item_has_exactly_one_owner",
          ),
      ]
  ```
  Additionally add a SEPARATE uniqueness constraint for the session-based
  case, since `unique_together = ("user", "product")` doesn't prevent
  duplicate `(session_key, product)` pairs (both `user` values would be
  `NULL` for two different anonymous sessions' rows, and NULL != NULL in
  a unique constraint, so this needs its own explicit constraint, not a
  reuse of the existing one):
  ```python
  constraints = [
      ...,  # the CheckConstraint above
      models.UniqueConstraint(
          fields=["session_key", "product"],
          condition=models.Q(session_key__isnull=False),
          name="wishlist_item_unique_session_product",
      ),
  ]
  ```
  Generate the migration (existing rows all have `user` set already, so
  this is safe/additive for existing data, same as Epic 5 Task 5.1.1.1's
  equivalent Cart migration).
- Add a `get_or_create_wishlist_owner_filter(request)` helper in
  dashboard/views.py (or wherever fits best), mirroring Epic 5 Task
  5.1.1.2's `get_or_create_cart(request)` helper conceptually, but
  returning a QUERYSET FILTER rather than a single object (since
  wishlist is multi-row, there's no single "the wishlist" object to
  get-or-create the way there is for `Cart`):
  ```python
  def get_wishlist_owner_kwargs(request):
      """Return the correct filter/creation kwargs for the current
      request's wishlist owner: user or session."""
      if request.user.is_authenticated:
          return {"user": request.user}
      if not request.session.session_key:
          request.session.create()
      return {"session_key": request.session.session_key}
  ```
- Update `WishlistViewSet`:
  - `permission_classes = [AllowAny]`
  - `get_queryset()`: `return Wishlist.objects.filter(**get_wishlist_owner_kwargs(self.request)).select_related(...)`
  - `perform_create()`: `serializer.save(**get_wishlist_owner_kwargs(self.request))`
  - `get_object()`: `get_object_or_404(Wishlist, pk=self.kwargs["pk"], **get_wishlist_owner_kwargs(self.request))`
  - Update `WishlistSerializer.validate()` (the existing duplicate-check
    logic, currently `Wishlist.objects.filter(user=user, product=...)`)
    to use the same owner-kwargs helper instead of assuming
    `user=self.context["request"].user` is always valid.
- Add merge-on-login logic mirroring Epic 5 Task 5.1.1.3 exactly:
  ```python
  # dashboard/services.py (or wherever cart's merge_session_cart_into_user_cart lives, for consistency)
  def merge_session_wishlist_into_user_wishlist(request, user):
      session_key = request.session.session_key
      if not session_key:
          return
      session_items = Wishlist.objects.filter(session_key=session_key)
      for item in session_items:
          if not Wishlist.objects.filter(user=user, product=item.product).exists():
              item.pk = None
              item.user = user
              item.session_key = None
              item.save()
          else:
              pass  # already wishlisted under the real account — just drop the duplicate session row
      session_items.filter(session_key=session_key).delete()  # clean up whatever's left (dupes that were skipped)
  ```
  Call this from the SAME three login/registration call sites Epic 5
  Task 5.1.1.3 already modified (`LoginView`, `OTPVerifyView`,
  `RegisterView`) — add this call alongside the existing
  `merge_session_cart_into_user_cart()` call at each site, not as a
  separate new integration point.

REQUIREMENTS — frontend
- Update `WishlistContext.jsx`'s `fetchWishlist()`: remove the
  `if (!isAuthenticated()) { setWishlist([]); return; }` early return —
  call the API unconditionally for every visitor, mirroring exactly how
  Epic 5 Task 5.1.1.4 updated `CartContext.jsx`'s equivalent guard.
- Update `WishlistTab.jsx`/wherever "add to wishlist" is triggered
  (e.g. a heart icon on product cards) to remove any auth-gating that
  redirects to `/login` before adding — mirror Epic 5 Task 5.1.1.4's
  removal of the equivalent cart guard.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/, mirroring Epic 5 Task 5.1.1.2's
cart test structure exactly:
1. An anonymous client can add/list/remove wishlist items via session,
   with persistence across requests using the same session cookie.
2. Two different anonymous sessions have fully independent wishlists.
3. Logging in merges the session wishlist into the user's real
   wishlist, with duplicate products (already wishlisted on the real
   account) correctly NOT double-added.
4. Authenticated-user wishlist behavior is completely unchanged from
   before this task (re-run all pre-existing wishlist tests unmodified
   except for permission-related assumptions, exactly per Epic 5 Task
   5.1.1.2's equivalent regression note).
5. The new `UniqueConstraint`/`CheckConstraint` correctly prevent: two
   session rows for the same session+product, and a row with both
   `user` and `session_key` set (or neither).
```

---

#### Task 11.1.1.2 — Price-drop notification on wishlisted items

```
You are working in backend/shop/ (a signal hook) and
backend/dashboard/models.py. Assume Task 11.1.1.1 is already merged,
and Epic 4 Task 4.1.2.3's stock-alert notification pattern already
exists as a precedent to follow closely (same signal-based structure,
same "TODO: Epic 16" placeholder approach for actual dispatch).

CONTEXT
Nothing currently notifies a customer when a product they've
wishlisted drops in price — a valuable, expected feature (mirrors the
"back in stock" notification already built in Epic 4 Feature 4.1.2, but
triggered by a PRICE decrease rather than a STOCK increase). Note:
`Wishlist` is product-level (per this document's header context), but
price/stock actually live on `ProductVariant` (per Epic 3) — this task
needs to decide what "the wishlisted product's price" even means when a
product has multiple variants at different prices, since there's no
single "the price" to compare against.

TASK
Detect a meaningful price decrease on a wishlisted product and notify
subscribers, using the same signal + Celery-task + "Epic 16 TODO
placeholder" pattern already established in Epic 4 Task 4.1.2.3.

REQUIREMENTS
- Decide the price-comparison basis: since a product can have multiple
  variants, use the product's LOWEST active variant price as "the"
  comparison price (a reasonable, defensible convention — "starting at
  $X" is exactly how multi-variant products are typically displayed
  anyway, per how `ProductListSerializer` likely already surfaces a
  representative price; check whether it already computes something
  like this and reuse that logic/property rather than inventing a
  second, possibly-inconsistent way to compute "the" price for the same
  product elsewhere in the codebase). Add a helper if one doesn't
  already exist:
  ```python
  # on Product, in backend/shop/models.py
  @property
  def lowest_active_price(self):
      variant = self.variants.filter(is_active=True, stock__gt=0).order_by("price").first()
      return variant.price if variant else None
  ```
- Add price-history tracking to detect an actual DECREASE (you can't
  know a price "dropped" without knowing what it was before): add a
  lightweight `ProductPriceSnapshot` model, or reuse the existing
  `StockMovement`-adjacent audit pattern if there's already something
  suitable — check whether any price-history tracking already exists
  from prior epics (there isn't, per the models reviewed so far) — a
  minimal new model is warranted:
  ```python
  class ProductPriceSnapshot(models.Model):
      product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name="price_snapshots")
      price = models.DecimalField(max_digits=10, decimal_places=2)
      recorded_at = models.DateTimeField(auto_now_add=True)

      class Meta:
          ordering = ["-recorded_at"]
  ```
  Generate the migration.
- Add a `post_save` signal on `ProductVariant` (mirroring Epic 4 Task
  4.1.2.3's exact signal-detection pattern, adapted for price instead
  of stock): when a variant's `price` changes AND that change results
  in the PRODUCT's `lowest_active_price` decreasing compared to its most
  recent `ProductPriceSnapshot`, record a new snapshot and fire a
  notification task.
  ```python
  # shop/signals.py — add alongside the existing stock-alert signal from Epic 4
  @receiver(post_save, sender=ProductVariant)
  def handle_variant_price_change(sender, instance, created, **kwargs):
      if created:
          return  # only care about actual CHANGES, not new variants
      product = instance.product
      current_lowest = product.lowest_active_price
      if current_lowest is None:
          return
      last_snapshot = product.price_snapshots.first()
      if last_snapshot is None:
          ProductPriceSnapshot.objects.create(product=product, price=current_lowest)
          return
      if current_lowest < last_snapshot.price:
          ProductPriceSnapshot.objects.create(product=product, price=current_lowest)
          notify_price_drop_subscribers.delay(product.id, last_snapshot.price, current_lowest)
      elif current_lowest != last_snapshot.price:
          # price changed but went UP or is otherwise different — still
          # record a new snapshot as the new baseline, just don't notify
          ProductPriceSnapshot.objects.create(product=product, price=current_lowest)
  ```
  Note the `if created: return` guard mirrors the same "only react to
  actual changes, not the initial creation" principle already
  established in prior epics' signal-based tasks — a brand-new variant
  isn't a "price drop," it's just the product having a price for the
  first time.
- Add `notify_price_drop_subscribers` to backend/shop/tasks.py
  (alongside `deactivate_expired_variants` from Epic 3 and
  `notify_stock_alert_subscribers` from Epic 4), following the EXACT
  same "mark subscribers, leave a TODO for Epic 16's real dispatch"
  pattern as Epic 4 Task 4.1.2.3:
  ```python
  @shared_task
  def notify_price_drop_subscribers(product_id, old_price, new_price):
      from .models import Product
      from dashboard.models import Wishlist
      try:
          product = Product.objects.get(pk=product_id)
      except Product.DoesNotExist:
          return "Product no longer exists."
      subscribers = Wishlist.objects.filter(product=product, user__isnull=False).select_related("user")
      count = 0
      for item in subscribers:
          # TODO: Epic 16 — replace this comment with a real
          # notify(item.user, event="price_drop", context={"product": product, "old_price": old_price, "new_price": new_price})
          # call once the unified notification service exists.
          count += 1
      return f"Would notify {count} wishlist subscriber(s) for product {product_id} price drop {old_price} -> {new_price}."
  ```
  Note `user__isnull=False` — per Task 11.1.1.1's guest-wishlist
  support, a session-only (anonymous) wishlist item has no stable
  identity to notify (no email/phone to reach them at), so only
  AUTHENTICATED wishlist items are notification-eligible; this mirrors
  the same authenticated-only constraint already applied to coupon
  redemption in Epic 9 for the same underlying reason.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/:
1. A variant price decrease that lowers the product's overall
   `lowest_active_price` below its last recorded snapshot creates a new
   `ProductPriceSnapshot` AND triggers `notify_price_drop_subscribers.delay()`
   (mock `.delay` and assert correct args).
2. A variant price INCREASE updates the snapshot baseline but does NOT
   trigger the notification task.
3. A variant price change that doesn't affect the product's overall
   lowest price (e.g. a non-lowest-priced variant's price changes,
   while a different variant remains the cheapest and unchanged) does
   NOT trigger a false "price drop" — confirm this correctly, since
   it's the trickiest edge case in this task's logic.
4. `notify_price_drop_subscribers()` only counts/processes AUTHENTICATED
   (non-session-only) wishlist subscribers for the affected product.
5. A newly-CREATED variant (not a price change on an existing one)
   never triggers the notification path.
```

---

#### Task 11.1.1.3 — "Move to cart" bulk action from wishlist

```
You are working in backend/dashboard/views.py, serializers.py and
frontend/src/components/WishlistTab.jsx. Assume Task 11.1.1.1 is
already merged.

CONTEXT — THE REAL GAP TO SOLVE
Per this document's header context: `Wishlist.product` references a
`Product`, but `Cart`/`CartItem` require a specific `ProductVariant`
(per Epic 3's migration) — moving a wishlisted item "to cart" is not a
simple 1:1 data copy the way it might be for a single-variant product
catalog. For a product with only ONE active variant, the choice is
unambiguous and this can be fully automatic. For a product with
MULTIPLE active variants (e.g. several shades), the system genuinely
cannot know which one the customer wants without asking — this task
must handle both cases correctly, not just build the easy
single-variant path and silently break/guess wrong for multi-variant
products.

TASK
Add a "move to cart" action (single item and bulk/"move all") that
automatically resolves the variant when unambiguous, and returns a
clear, actionable response when the customer needs to choose a variant
themselves.

REQUIREMENTS — backend
- Add a custom `@action` on `WishlistViewSet`:
  ```python
  from rest_framework.decorators import action
  from cart.views import get_or_create_cart
  from cart.models import CartItem

  class WishlistViewSet(viewsets.ModelViewSet):
      ...

      @action(detail=True, methods=["post"], url_path="move-to-cart")
      def move_to_cart(self, request, pk=None):
          wishlist_item = self.get_object()
          product = wishlist_item.product
          active_variants = list(product.variants.filter(is_active=True, stock__gt=0))

          if len(active_variants) == 0:
              return Response(
                  {"detail": "This product is currently unavailable."},
                  status=status.HTTP_400_BAD_REQUEST,
              )
          if len(active_variants) > 1:
              variant_id = request.data.get("variant_id")
              if not variant_id:
                  return Response(
                      {
                          "detail": "This product has multiple options — please choose one.",
                          "variants": ProductVariantSerializer(active_variants, many=True).data,
                      },
                      status=status.HTTP_300_MULTIPLE_CHOICES,
                  )
              variant = next((v for v in active_variants if v.id == int(variant_id)), None)
              if variant is None:
                  return Response({"variant_id": "Invalid variant for this product."}, status=status.HTTP_400_BAD_REQUEST)
          else:
              variant = active_variants[0]

          cart = get_or_create_cart(request)
          cart_item, created = CartItem.objects.get_or_create(
              cart=cart, variant=variant, defaults={"quantity": 1}
          )
          if not created:
              cart_item.quantity += 1
              cart_item.save(update_fields=["quantity"])
          wishlist_item.delete()
          return Response({"detail": "Moved to cart."}, status=status.HTTP_200_OK)
  ```
  Import `ProductVariantSerializer` from `shop.serializers` (Epic 3
  Task 3.2.1.14) and `get_or_create_cart` from `cart.views` (Epic 5
  Task 5.1.1.2) at the top of dashboard/views.py — check for circular
  import risk between `dashboard` and `cart` apps first (should be
  safe; `cart` doesn't currently import from `dashboard`).
  Using HTTP 300 (Multiple Choices) for the "needs variant selection"
  case is a deliberate, semantically meaningful choice — it's not an
  error exactly, but it IS telling the client "I can't complete this
  without more input," which fits 300's actual meaning better than
  overloading 400 for what's really a legitimate, expected
  intermediate state, not a client mistake. If your team/project
  convention prefers avoiding less-common status codes for simplicity,
  200 with an `{"needs_selection": true, "variants": [...]}" body is
  an equally valid alternative — pick one and be consistent, but
  document the choice.
- Add a BULK "move all to cart" action:
  ```python
  @action(detail=False, methods=["post"], url_path="move-all-to-cart")
  def move_all_to_cart(self, request):
      items = self.get_queryset()
      moved, needs_selection, unavailable = [], [], []
      for item in items:
          active_variants = list(item.product.variants.filter(is_active=True, stock__gt=0))
          if not active_variants:
              unavailable.append({"product": item.product.name})
          elif len(active_variants) > 1:
              needs_selection.append({
                  "wishlist_item_id": item.id,
                  "product": item.product.name,
                  "variants": ProductVariantSerializer(active_variants, many=True).data,
              })
          else:
              cart = get_or_create_cart(request)
              cart_item, created = CartItem.objects.get_or_create(
                  cart=cart, variant=active_variants[0], defaults={"quantity": 1}
              )
              if not created:
                  cart_item.quantity += 1
                  cart_item.save(update_fields=["quantity"])
              item.delete()
              moved.append(item.product.name)
      return Response(
          {"moved": moved, "needs_selection": needs_selection, "unavailable": unavailable},
          status=status.HTTP_200_OK,
      )
  ```
  This deliberately moves everything UNAMBIGUOUS automatically in one
  pass, and reports back exactly which items still need the customer to
  pick a variant (rather than either silently skipping them with no
  explanation, or blocking the entire bulk operation until every single
  item has a manual choice made) — a reasonable, customer-friendly
  middle ground for a bulk action.

REQUIREMENTS — frontend
- Add API methods to frontend/src/services/api.js:
  ```javascript
  moveWishlistItemToCart: (id, variantId) =>
    dashboardAPI.post(`/dashboard/wishlist/${id}/move-to-cart/`, variantId ? { variant_id: variantId } : {}),
  moveAllWishlistToCart: () =>
    dashboardAPI.post('/dashboard/wishlist/move-all-to-cart/'),
  ```
  (adjust to match however this project's `dashboardAPI` object is
  actually structured — check its existing method conventions first).
- In `WishlistTab.jsx`: add a "Move to Cart" button per item. On a 300
  (or your chosen alternative) response indicating multiple variants,
  show a small inline variant picker (reusing whatever
  shade/size-selector component already exists on the product detail
  page, if one does, rather than building a second, inconsistent
  picker UI) and resubmit with the chosen `variant_id`.
- Add a "Move All to Cart" bulk button. On response, show a summary:
  which items moved successfully, which need the customer to pick a
  variant (with inline pickers for each), and which are now
  unavailable — don't just silently drop the ambiguous/unavailable ones
  from view without explanation.
- After any successful move, trigger both `WishlistContext` and
  `CartContext` to refetch (the item left one list and entered the
  other — both need to reflect the change immediately, mirroring how
  Epic 5/9's cross-context refresh patterns already work elsewhere in
  this app, e.g. cart refetching after a coupon is applied).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/:
1. Moving a wishlist item for a SINGLE-variant product succeeds
   immediately, removes the wishlist item, and creates/increments the
   correct cart item.
2. Moving a wishlist item for a MULTI-variant product WITHOUT a
   `variant_id` returns the "needs selection" response with the correct
   list of active variants, and does NOT modify the cart or delete the
   wishlist item.
3. Submitting the SAME request WITH a valid `variant_id` completes the
   move correctly.
4. Submitting an invalid/mismatched `variant_id` (e.g. one belonging to
   a different product) is rejected with 400.
5. A product with ZERO active/in-stock variants returns the
   "unavailable" response and does not move anything.
6. Bulk "move all" correctly partitions a mixed wishlist (some
   single-variant, some multi-variant, some unavailable) into the three
   correct response buckets, and only the genuinely-moved items are
   actually removed from the wishlist / added to the cart.
7. Moving an item already in the cart (same variant) increments the
   existing `CartItem.quantity` rather than creating a duplicate cart
   line.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 11.1.1.1 | Guest wishlist (session-based) | ☐ |
| 11.1.1.2 | Price-drop notification on wishlisted items | ☐ |
| 11.1.1.3 | "Move to cart" bulk action from wishlist | ☐ |

Once Epic 11 is fully merged, the next epics to generate prompts for
are **Epic 12 — Search & Filtering** and **Epic 13 — Recommendation
Engine**, both of which depend on Epic 3's catalog/variant model
(already merged) rather than on Epic 10 or 11 specifically, and can be
sequenced independently of them per the master backlog's execution
order notes.
