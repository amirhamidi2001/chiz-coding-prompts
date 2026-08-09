# Epic 5 — Cart & Checkout — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–4 are fully merged. Specifically relevant here: `CartItem.product` was changed to `CartItem.variant` (FK to `shop.ProductVariant`) in Epic 3 Task 3.1.1.3, meaning `cart/views.py`'s `CartView.post()` now accepts `variant_id` (not `product_id`) and `cart/serializers.py`'s `CartItemSerializer` validates against `ProductVariant` (not `Product`) — this document's prompts assume that migrated state throughout, even though a fresh read of the raw pre-Epic-3 repo still shows `product_id`/`Product` in these files; treat every reference below to "the cart's variant-based add/update flow" as the Epic-3-migrated version of that code. Also assumed: `order/serializers.py` `OrderCreateSerializer` no longer accepts a client-supplied `discount` field (Epic 1 Task 1.2.1.1), order creation is wrapped in `transaction.atomic()` with variant-level `select_for_update()` stock locking (Epic 1 + Epic 3 Task 3.1.1.5), pricing goes through `order/services/pricing.py`'s `calculate_order_totals()` (Epic 1 Feature 1.1.2), expired-variant checkout is already blocked (Epic 3 Task 3.3.1.2), and every stock change is logged via `StockMovement` (Epic 4 Feature 4.1.1).

**Important grounded discovery for this epic:** an `Address` model, `AddressSerializer`, and full `AddressViewSet` CRUD **already exist** in `backend/dashboard/models.py` / `serializers.py` / `views.py`, registered at `/dashboard/addresses/` — it just wasn't ever wired into the checkout flow, and its fields (`state`, `zip_code`, `country` defaulting to `"US"`) are generic/US-shaped, not Iran-specific. Task 5.2.1.3 below is scoped around ADAPTING this existing model into checkout, not building address CRUD from scratch — read that task carefully before assuming a bigger build is needed than actually is.

---

## Phase 5.1 — Guest Cart Support

### Feature 5.1.1 — Session-Based Cart

---

#### Task 5.1.1.1 — Make `Cart.user` nullable, add `session_key` field

```
You are working in backend/cart/models.py. Assume Epic 3's variant
migration (CartItem.variant) is already merged.

CONTEXT
The current Cart model is:

    class Cart(models.Model):
        user = models.OneToOneField(
            settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="cart",
        )
        created_at = models.DateTimeField(auto_now_add=True)
        updated_at = models.DateTimeField(auto_now=True)

Every cart REQUIRES an authenticated user (`OneToOneField`, non-nullable)
— per-app views (`CartView`, `CartItemView`, `CartClearView` in
cart/views.py) all have `permission_classes = [IsAuthenticated]`. This
means a visitor has to create an account before they can add a single
item to a cart, which is a well-known conversion-rate killer for
e-commerce (most shoppers browse and add-to-cart before ever deciding
to check out/register). This task lays the schema groundwork to
support anonymous, session-based carts.

TASK
Make `Cart.user` nullable and add a `session_key` field so a cart can
exist and be identified either by a logged-in user OR by an anonymous
session token — but NOT require both, and never allow a cart to have
neither.

REQUIREMENTS
- Change `user` from `OneToOneField(..., on_delete=models.CASCADE)` to
  `OneToOneField(..., on_delete=models.CASCADE, null=True, blank=True)`.
- Add:
  `session_key = models.CharField(max_length=64, null=True, blank=True, unique=True, db_index=True)`
  — this will store Django's session key (NOT a custom-invented token;
  reuse Django's built-in session framework's `request.session.session_key`,
  which already exists project-wide since `django.contrib.sessions` is
  a standard Django app — confirm `SessionMiddleware` is in
  `MIDDLEWARE` in backend/core/settings/base.py, it should already be
  there as part of Django's default project scaffold).
- Add a `Meta.constraints` check ensuring a Cart always has EXACTLY ONE
  of `user`/`session_key` set, never both and never neither:
  ```python
  class Meta:
      ordering = ["-updated_at"]
      constraints = [
          models.CheckConstraint(
              check=(
                  models.Q(user__isnull=False, session_key__isnull=True)
                  | models.Q(user__isnull=True, session_key__isnull=False)
              ),
              name="cart_has_exactly_one_owner",
          )
      ]
  ```
  (this is a real, DB-enforced invariant, not just an application-level
  convention — it should be genuinely impossible to create an orphaned
  cart with neither owner, or an ambiguous one with both).
- Generate the migration. Since existing Cart rows all currently have a
  non-null `user`, this migration is safe for existing data (every
  existing row already satisfies the new constraint's first branch)
  — no data migration is needed, just the schema change itself.
- Do NOT touch `cart/views.py`'s `get_or_create_cart()` helper in this
  task — that's Task 5.1.1.2's job specifically. This task is schema
  only.

ACCEPTANCE CRITERIA / TESTS
Add model tests to backend/cart/tests.py (or convert to a tests/
package matching the convention established in other apps across
prior epics, e.g. Epic 1 Task 1.1.1.5's order/tests/ restructuring —
check whether cart/tests.py is still a single file or already a
package by this point and follow whatever's already there):
1. A Cart with only `user` set (no `session_key`) saves successfully.
2. A Cart with only `session_key` set (no `user`) saves successfully.
3. A Cart with BOTH `user` and `session_key` set fails to save
   (`IntegrityError` from the CheckConstraint).
4. A Cart with NEITHER set fails to save.
5. Existing tests that create a Cart via `user=some_user` continue to
   pass unchanged (proving this is a backward-compatible additive
   change for the authenticated-cart case).
```

---

#### Task 5.1.1.2 — Cart middleware/helper to resolve cart by user-or-session

```
You are working in backend/cart/views.py. Assume Task 5.1.1.1 (nullable
Cart.user + session_key) is already merged.

CONTEXT
Every cart view currently does:

    def get_or_create_cart(user):
        cart, _ = Cart.objects.get_or_create(user=user)
        return cart

    class CartView(APIView):
        permission_classes = [IsAuthenticated]
        def get(self, request):
            cart = get_or_create_cart(request.user)
            ...

This needs to become the single, central place that decides "does this
request belong to a logged-in user's cart, or an anonymous session's
cart" — and every view (`CartView`, `CartItemView`, `CartClearView`)
needs to route through it instead of assuming `request.user` is always
valid AND instead of requiring `IsAuthenticated` at all.

TASK
Rewrite `get_or_create_cart()` to accept the full `request` object
(not just `user`) and resolve to the correct cart for either an
authenticated user or an anonymous session, and change all three cart
views' `permission_classes` to `AllowAny`.

REQUIREMENTS
- Replace the helper:
  ```python
  from rest_framework.permissions import AllowAny

  def get_or_create_cart(request):
      """
      Resolve the cart for this request: the authenticated user's cart
      if logged in, otherwise a session-based anonymous cart.
      """
      if request.user.is_authenticated:
          cart, _ = Cart.objects.get_or_create(user=request.user)
          return cart

      if not request.session.session_key:
          request.session.create()
      session_key = request.session.session_key
      cart, _ = Cart.objects.get_or_create(session_key=session_key)
      return cart
  ```
  — note `request.session.create()` is required BEFORE reading
  `request.session.session_key` for a brand-new anonymous visitor
  (Django doesn't populate `session_key` until the session has been
  explicitly saved/created at least once) — this is a common gotcha,
  make sure it's handled exactly as shown, not assumed to already
  exist.
- Update every call site in cart/views.py
  (`CartView.get/post`, `CartItemView._update/delete`'s underlying
  cart-ownership check, `CartClearView.delete`) to call
  `get_or_create_cart(request)` instead of `get_or_create_cart(request.user)`.
- Change `permission_classes = [IsAuthenticated]` to
  `permission_classes = [AllowAny]` on `CartView`, `CartItemView`, and
  `CartClearView` — anonymous users must now be able to use the cart
  fully.
- `CartItemView._get_item()` currently does:
  `CartItem.objects.select_related("cart__user").get(pk=item_id, cart__user=request.user)`
  — this ONLY works for authenticated users and will break entirely
  for anonymous carts (it filters by `cart__user=request.user`, which
  for an `AnonymousUser` would incorrectly match nothing, or worse,
  incorrectly match `cart__user=None` rows belonging to OTHER
  anonymous sessions if not handled carefully). Rewrite it to resolve
  the requesting cart first via `get_or_create_cart(request)`, then
  filter `CartItem.objects.get(pk=item_id, cart=resolved_cart)` — this
  is both correct for both auth states AND actually more efficient
  (one clear cart resolution step instead of an ambiguous cross-table
  filter).
- Django's `SessionMiddleware` must run before these views execute for
  `request.session` to exist at all — confirm it's present and ordered
  correctly in `MIDDLEWARE` in backend/core/settings/base.py (it should
  already be there by default; this task doesn't need to add it, just
  confirm and note if somehow missing).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/cart/tests/:
1. An anonymous (unauthenticated) client GETting `/api/cart/` for the
   first time gets 200 with an empty cart, and a session cookie is set
   in the response.
2. The SAME anonymous client (reusing its session cookie across
   requests — use DRF's `APIClient`, which persists cookies across
   calls within one client instance) adding an item and then GETting
   the cart again sees that item still present (proving session
   persistence works).
3. TWO DIFFERENT anonymous clients (two separate `APIClient` instances,
   simulating two different browsers/visitors with no shared session)
   each get their OWN independent empty cart, and adding an item to one
   does not affect the other.
4. An authenticated user's cart behavior is completely unchanged from
   before this task (re-run all pre-existing cart tests and confirm
   they still pass with zero modification needed to their assertions —
   only their permission setup, if any relied on the exact
   `IsAuthenticated` 401 behavior for logged-out requests, which no
   longer applies).
```

---

#### Task 5.1.1.3 — Merge guest cart into user cart on login

```
You are working in backend/accounts/views.py (LoginView, and Epic 2's
OTPVerifyView) and backend/cart/. Assume Task 5.1.1.2 is already
merged.

CONTEXT
A visitor can now build up an anonymous, session-based cart before
logging in (Task 5.1.1.2). Once they DO log in (via either the
existing email/password `LoginView` OR Epic 2's `OTPVerifyView`), their
items need to end up in their real, persistent user-owned cart —
otherwise the anonymous cart becomes orphaned and invisible the moment
they authenticate, silently losing everything they added, which is a
severe, trust-destroying bug for a real store (imagine adding 3 items,
logging in to check out, and the cart is suddenly empty).

TASK
Add cart-merge logic that runs on successful login (both auth paths),
combining any existing anonymous session cart's items into the user's
persistent cart.

REQUIREMENTS
- Create a shared merge function in backend/cart/services.py (new
  file):
  ```python
  from django.db import transaction
  from .models import Cart, CartItem

  def merge_session_cart_into_user_cart(request, user):
      """
      If the current session has an anonymous cart, merge its items
      into `user`'s persistent cart, then delete the anonymous cart.
      Existing quantities are summed when the same variant appears in
      both carts.
      """
      session_key = request.session.session_key
      if not session_key:
          return

      try:
          session_cart = Cart.objects.prefetch_related("items").get(session_key=session_key)
      except Cart.DoesNotExist:
          return

      if not session_cart.items.exists():
          session_cart.delete()
          return

      with transaction.atomic():
          user_cart, _ = Cart.objects.get_or_create(user=user)
          for item in session_cart.items.all():
              existing = user_cart.items.filter(variant=item.variant).first()
              if existing:
                  existing.quantity += item.quantity
                  existing.save(update_fields=["quantity"])
              else:
                  item.pk = None
                  item.cart = user_cart
                  item.save()
          session_cart.delete()
  ```
  — note the `item.pk = None; item.cart = user_cart; item.save()`
  pattern for moving a CartItem to a different cart by cloning it
  rather than trying to reassign the FK on the original row directly
  and re-saving (either approach technically works in Django, but
  cloning-via-pk-reset is the clearer, more explicit pattern here since
  `session_cart` is about to be deleted anyway and we don't want any
  ambiguity about which row survives).
- Call this from `LoginView` in backend/accounts/views.py: after
  successful authentication (wherever the existing view currently
  obtains the authenticated `user` object, right before or after
  issuing tokens — check the exact structure of `LoginView` since it
  extends `TokenObtainPairView`, meaning the "user" isn't trivially
  available in the same way a plain `APIView` would have it; you may
  need to override `post()` to call `super().post()` first to get the
  token response, THEN look up the user from the validated
  credentials/serializer, THEN call the merge function, THEN return the
  original response — inspect `TokenObtainPairView`'s implementation
  from `rest_framework_simplejwt` to find the cleanest override point,
  likely via overriding `post()` and accessing
  `serializer.user` after `serializer.is_valid()` inside your override,
  mirroring how SimpleJWT's own view accesses the authenticated user
  internally).
- Also call it from `OTPVerifyView.post()` (Epic 2 Task 2.3.1.2) at the
  point where the user is resolved (either the existing or
  newly-created user), before returning the token response — this is a
  more straightforward call site since `OTPVerifyView` is a plain
  `APIView` already directly constructing/fetching the `user` object.
- Also call it from `RegisterView` (the plain email/password
  registration flow) for the same reason — a visitor might build a
  cart anonymously, then register a brand-new account rather than
  logging into an existing one, and should get the same merge
  treatment.
- In all three call sites, pass `request` (for session access) and the
  resolved `user` object to `merge_session_cart_into_user_cart()`.

ACCEPTANCE CRITERIA / TESTS
Add tests covering all three login/registration paths:
1. An anonymous client adds 2 different variants to its session cart,
   then logs in via email/password (`LoginView`) as an EXISTING user
   who has an EMPTY persistent cart — after login, that user's cart
   contains both items with correct quantities, and the old session
   cart row no longer exists.
2. Same setup, but the existing user's persistent cart ALREADY contains
   one of the two variants (different quantity) — after merge, that
   variant's quantity is the SUM of both quantities, and the other
   variant is added as a new line.
3. Same merge behavior verified for the OTP login path
   (`OTPVerifyView`) and for fresh registration (`RegisterView`).
4. An anonymous client with NO session cart at all (never added
   anything) logging in doesn't error and doesn't create any spurious
   empty cart artifacts.
5. Logging in twice in a row (second login has no session cart left
   to merge, since it was deleted after the first merge) doesn't
   error on the second attempt.
```

---

#### Task 5.1.1.4 — Update frontend cart calls to work pre-login

```
You are working in frontend/src/context/CartContext.jsx and
frontend/src/services/api.js. Assume Task 5.1.1.2 (backend now supports
AllowAny anonymous carts via session) is already merged and deployed.

CONTEXT
`CartContext.jsx` currently gates almost everything behind
`isAuthenticated()`:

    const fetchCart = useCallback(async () => {
        if (!isAuthenticated()) {
            setCart(null);
            return;
        }
        ...
    }, []);

    const handleAddToCart = async (productId, quantity = 1) => {
        if (!isAuthenticated()) {
            window.location.href = '/login';
            return { success: false, message: 'Please log in to add items to your cart.' };
        }
        ...
    };

This actively PREVENTS anonymous users from using the now-functional
guest cart backend — the frontend is the last remaining blocker for
guest checkout to actually work end-to-end.

TASK
Remove the authentication gates from cart operations in the frontend,
letting anonymous visitors use the cart exactly like authenticated
users, relying on the browser's session cookie (already sent
automatically by axios if `withCredentials`/cookie config is already
set up — verify this in frontend/src/services/api.js's axios instance
configuration; Django's session cookie needs to actually be included
in cross-origin requests if the frontend and backend are on different
origins/ports in development, which likely requires
`axios.defaults.withCredentials = true` or equivalent per-request
config, plus corresponding `CORS_ALLOW_CREDENTIALS = True` and a
specific (not wildcard) `CORS_ALLOWED_ORIGINS` in Django settings —
check backend/core/settings for the existing CORS configuration and
confirm/fix this, since anonymous cart persistence via session cookie
will silently fail across origins without it).

REQUIREMENTS
- In `fetchCart()`: remove the `if (!isAuthenticated())` early return —
  call `getCart()` unconditionally on mount for every visitor,
  authenticated or not, since the backend now returns a valid
  (possibly empty) cart for anonymous sessions too.
- In `handleAddToCart()`: remove the
  `if (!isAuthenticated()) { window.location.href = '/login'; ... }`
  guard entirely — anonymous add-to-cart should just work now, hitting
  the same `addToCart()` API call as an authenticated user.
- Review `handleUpdateItem`/`handleRemoveItem`/`handleClearCart` for
  any similar implicit auth assumptions (they don't appear to have
  explicit auth checks currently, per the read code, but verify since
  they rely on cart-scoped endpoints that previously always assumed an
  authenticated `request.user` — confirm no other part of the frontend
  is relying on `cartCount`/`cart` being `null` as an implicit "is
  logged in" signal, since that assumption is no longer valid — search
  the codebase for `cart === null` or `!cart` used as an auth check
  anywhere outside CartContext itself and fix any such usage).
- Check the `useEffect` that listens for `'auth-change'` and
  `'storage'` events to re-fetch the cart on login/logout — this should
  now ALSO correctly refetch after the backend-side merge (Task
  5.1.1.3) happens during login, so the user sees their merged cart
  immediately post-login without a stale pre-merge cart lingering in
  frontend state. Verify the existing event-driven refetch already
  covers this (it likely does, since login already fires
  `'auth-change'` per the existing pattern — just confirm by reading
  wherever `'auth-change'` is dispatched, likely in AuthContext's
  `login()`/`loginWithOtp()` methods from Epic 2).
- Any UI copy that says something like "log in to add items to your
  cart" (search frontend/src for this or similar strings) should be
  removed/updated, since it's no longer true.

ACCEPTANCE CRITERIA
- Manually verify in the browser (or via a Playwright/Cypress-style
  E2E check if that tooling exists by this point in the project;
  otherwise manual verification is acceptable for this specific
  frontend task): opening the storefront in a fresh incognito/private
  window (no auth), adding items to the cart, refreshing the page, and
  confirming the cart persists (session cookie working); then logging
  in and confirming the same items appear in the now-authenticated
  cart view (merge working end-to-end through the full stack).
- Add/update Vitest component tests for `CartContext` confirming
  `addToCart()` no longer redirects to `/login` and instead calls the
  API directly regardless of auth state (mock `isAuthenticated()` to
  return `false` and assert the API call still happens, unlike the
  previous behavior).
```

---

## Phase 5.2 — Checkout Flow Hardening

### Feature 5.2.1 — Checkout Validation

---

#### Task 5.2.1.1 — Re-validate stock at checkout submit (not just cart-add time)

```
You are working in backend/order/serializers.py
(OrderCreateSerializer). Assume Epic 1 + Epic 3's atomic,
variant-locked stock-decrement logic in `create()` is already merged
(it already re-checks stock via `select_for_update()` at the moment of
order creation, per Epic 1 Task 1.1.1.3 / Epic 3 Task 3.1.1.5) — this
task is about strengthening the EARLIER `validate()` step, which
currently only checks `cart.items.exists()` and does NOT surface a
clear, checkout-specific stock error before the user even attempts to
submit payment details.

CONTEXT
The existing `validate()` method:

    def validate(self, attrs):
        user = self.context["request"].user
        try:
            cart = user.cart
        except Exception:
            raise serializers.ValidationError({"cart": "No cart found for this user."})
        if not cart.items.exists():
            raise serializers.ValidationError({"cart": "Your cart is empty."})
        attrs["cart"] = cart
        return attrs

only checks that a cart exists and isn't empty — it does NOT check
whether every item in it is still actually purchasable (sufficient
stock, active variant) BEFORE the serializer proceeds to `create()`.
Technically `create()`'s atomic block already catches insufficient
stock and rolls back correctly (per Epic 1/3), so this isn't a
correctness bug — but it means a customer who fills out their entire
shipping/payment form only finds out about a stock problem AFTER
submitting the whole checkout form, rather than getting a clear
pre-submission signal. This task adds an EARLY, non-locking sanity
check in `validate()` as a UX improvement layered on top of the
authoritative, transaction-safe check that already exists in
`create()` — both checks matter and serve different purposes (this one
is fast UX feedback; the one in `create()` is the actual race-safe
guarantee).

TASK
Add a stock/availability pre-check to `validate()` that surfaces a
clear, itemized error if any cart item is no longer purchasable,
without needing to lock any rows (this is a best-effort, point-in-time
check — the real guarantee still comes from `create()`'s atomic
locking, so don't try to make this one authoritative or add locking
here).

REQUIREMENTS
- In `validate()`, after confirming the cart isn't empty, iterate
  `cart.items.select_related("variant").all()` and, for each item,
  check: `item.variant is None` (shouldn't normally happen for an
  active cart item, but defensive), `not item.variant.is_active`, or
  `item.variant.stock < item.quantity`. Collect ALL problems (not just
  the first one found) into a list, so the customer sees every issue at
  once rather than fixing one and resubmitting to discover the next.
- If any problems are found, raise:
  ```python
  raise serializers.ValidationError({
      "cart": [
          f"{problem.variant.product.name} ({problem.variant.sku}) is no longer available in the requested quantity."
          for problem in problems
      ]
  })
  ```
  (adjust exact message wording as you see fit, but it must identify
  WHICH item(s) have a problem, not just say "something in your cart is
  unavailable").
- This check must NOT use `select_for_update()` — it's explicitly a
  non-locking, best-effort pre-check; only `create()`'s existing logic
  should ever take row locks, to avoid holding unnecessary locks during
  what could be a slow client-side form-fill between `validate()` and
  actual submission (note: DRF actually calls `validate()` as part of
  the SAME request/response cycle as `create()` when the serializer's
  `.is_valid()` then `.save()` are called back-to-back in the view —
  so in practice there's no real time gap between this check and
  `create()`'s locking check within a single request; the real value
  of this task is a cleaner, itemized error message rather than a
  meaningfully earlier point in time — make sure your task summary
  reflects this accurate understanding rather than overstating what
  this check accomplishes).

ACCEPTANCE CRITERIA / TESTS
Add tests to the order test suite:
1. A cart with one item whose variant has since gone out of stock
   fails checkout submission with a 400 identifying that specific item
   by name/SKU.
2. A cart with one item whose variant has been deactivated
   (`is_active=False`) since being added fails checkout with a clear
   message.
3. A cart with MULTIPLE problem items surfaces ALL of them in one error
   response, not just the first.
4. A cart with no problems passes `validate()` cleanly and proceeds to
   `create()` as before (regression check — don't break the happy
   path).
```

---

#### Task 5.2.1.2 — Re-validate price at checkout (server-authoritative)

```
You are working in backend/order/serializers.py (OrderCreateSerializer)
and backend/cart/serializers.py. Assume Task 5.2.1.1 is merged.

CONTEXT
Read through the current codebase carefully: `Cart.subtotal` is
computed as `sum(item.subtotal for item in self.items.all())`, and
`CartItem.subtotal` is `self.unit_price * self.quantity`, where
`unit_price` is a `@property` that reads `self.variant.price` LIVE at
access time (per Epic 3's variant migration) — meaning the price used
at checkout is ALREADY always freshly read from the database at the
moment of order creation, not cached/stale from whenever the item was
added to the cart. This is actually already correct/safe by
construction — there's no separate "price sent by the client" anywhere
in the current checkout flow; `OrderCreateSerializer` never accepts a
price field from the request body at all, it only accepts checkout
FORM data (name/address/payment method/etc.) and derives all monetary
values server-side from `cart.subtotal`.

TASK
Since this codebase is already structurally safe against client-supplied
prices (no such field exists to remove, unlike the discount
vulnerability fixed in Epic 1 Task 1.2.1.1), THIS TASK is about adding
an explicit, defense-in-depth REGRESSION TEST and a code-level assertion
confirming this property holds — plus fixing one real remaining gap:
`Cart.subtotal` and `CartItem.subtotal` are computed via Python
properties that iterate `self.items.all()` in memory, which means if
this is ever called without `select_related("variant")` already
applied, it triggers an N+1 query per item AND — more importantly for
correctness — there's no guarding against a variant's price changing
mid-request between when `cart.subtotal` is first read (in
`validate()`, if at all) and when it's read AGAIN in `create()` (each
property access re-queries `self.variant.price` freshly, so if
`create()` reads it after `validate()`, they could theoretically
compute different subtotals if an admin changes a price in that exact
window — an unlikely but real TOCTOU-style edge case worth closing
given how much financial-safety work has already gone into this order
flow in prior epics).

TASK (restated concretely)
1. Confirm and lock in with a test that `OrderCreateSerializer` has NO
   client-writable price/subtotal/unit_price field anywhere in its
   declared fields (a grep-and-assert check, similar in spirit to Epic
   1 Task 1.2.1.2's discount regression test).
2. Ensure `cart.subtotal` is computed EXACTLY ONCE per order-creation
   request and that same value is used consistently throughout
   `create()` (both for the `Order.subtotal`/`total` fields AND for
   each `OrderItem.unit_price` snapshot), rather than potentially
   re-reading variant prices multiple times across the method in a way
   that could theoretically drift within a single request.

REQUIREMENTS
- Add a test asserting `"price" not in OrderCreateSerializer().fields`,
  `"subtotal" not in OrderCreateSerializer().fields`, and
  `"unit_price" not in OrderCreateSerializer().fields` — a structural
  guard against ever accidentally reintroducing a client-controlled
  price field in the future (mirrors the spirit of Epic 1 Task 1.2.1.2's
  discount regression test, but at the schema level rather than via a
  live request).
- In `create()`, refactor slightly so that each `cart_item`'s
  `unit_price` is read ONCE into a local variable
  (`unit_price = cart_item.variant.price`) and that same captured value
  is used both for the `OrderItem.unit_price` snapshot AND is what
  contributes to the `subtotal` sum used for `Order.subtotal`/total —
  rather than potentially computing `cart.subtotal` (which re-iterates
  and re-reads every item's live price) SEPARATELY from the per-item
  loop that creates `OrderItem` rows (which also reads live price via
  `cart_item.unit_price`). Currently these two computations
  (`cart.subtotal` used for the Order total, and `cart_item.unit_price`
  used per OrderItem) are two SEPARATE property-access paths that each
  independently query `variant.price` — if you can restructure `create()`
  to compute per-item prices ONCE up front (e.g. build a list of
  `(cart_item, unit_price)` tuples first, before touching the DB
  further), then derive BOTH the subtotal AND the per-item OrderItem
  snapshots from that single captured list, you close the
  theoretical drift window entirely. This is a real, if narrow,
  correctness improvement — implement it if it doesn't require an
  excessive rewrite of the surrounding, already-tested atomic-block
  logic from prior epics; if it turns out to require substantial
  restructuring, do the minimal version (capture each item's price into
  a local variable within the SAME iteration that also creates the
  OrderItem, and sum those local variables into the subtotal used for
  the Order, rather than calling `cart.subtotal` as a separate,
  independent computation elsewhere in the method).

ACCEPTANCE CRITERIA / TESTS
Add tests:
1. The schema-level guard test described above (no price/subtotal/
   unit_price field accepted as client input).
2. A test that changes a variant's price BETWEEN cart-item-add and
   order-submission (update `variant.price` directly via the ORM after
   items are already in the cart) and confirms the resulting Order's
   `subtotal`/`total` AND every `OrderItem.unit_price` reflect the
   variant's price AT THE MOMENT OF ORDER CREATION (i.e. the new,
   updated price), not whatever it was when first added to the cart —
   proving there's no stale/cached pricing anywhere in the flow.
3. Re-run the full order test suite to confirm the refactor (if you did
   the fuller version) produces byte-identical Order/OrderItem values
   to before this task for all existing test scenarios.
```

---

#### Task 5.2.1.3 — Address book model (`ShippingAddress`)

```
You are working in backend/dashboard/models.py,
backend/order/serializers.py, and backend/order/views.py.

CONTEXT — READ THIS CAREFULLY BEFORE STARTING, THIS TASK IS SMALLER
THAN IT LOOKS
An `Address` model, `AddressSerializer`, and full `AddressViewSet` CRUD
(list/create/update/delete) ALREADY EXIST in this codebase today, in
backend/dashboard/models.py and views.py, registered at
`/dashboard/addresses/` via the router in backend/dashboard/urls.py:

    class Address(models.Model):
        class Label(models.TextChoices):
            HOME = "home", "Home"
            OFFICE = "office", "Office"
            OTHER = "other", "Other"
        user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="addresses")
        label = models.CharField(max_length=20, choices=Label.choices, default=Label.HOME)
        first_name = models.CharField(max_length=100)
        last_name = models.CharField(max_length=100)
        phone = models.CharField(max_length=30)
        address_line = models.CharField(max_length=255)
        apartment = models.CharField(max_length=100, blank=True)
        city = models.CharField(max_length=100)
        state = models.CharField(max_length=100)
        zip_code = models.CharField(max_length=20)
        country = models.CharField(max_length=10, default="US")
        is_default = models.BooleanField(default=False)
        ...
        def save(self, *args, **kwargs):
            # Ensure only one default address per user
            if self.is_default:
                Address.objects.filter(user=self.user, is_default=True).exclude(pk=self.pk).update(is_default=False)
            super().save(*args, **kwargs)

`AddressViewSet` at `/dashboard/addresses/` already supports
GET/POST/PATCH/DELETE scoped to the requesting user, with
`get_queryset()` correctly filtering `Address.objects.filter(user=self.request.user)`.

So: this task is NOT "build an address book from scratch" — that
already exists and works. This task is specifically: (1) confirm the
existing model is fit for purpose as a checkout-time address SOURCE
(it currently isn't wired into checkout at all — `OrderCreateSerializer`
still requires the customer to type their full address into the
checkout form every single time, with zero connection to their saved
`Address` book), and (2) add the ability to check out USING a saved
address by reference, rather than only via manually re-typed fields.
Iran-specific field changes to this model are handled separately in
Task 5.2.1.4, which runs AFTER this task — don't do that work here.

TASK
Wire the checkout flow to optionally accept a saved `address_id`
instead of requiring the full set of manually-typed address fields
every time.

REQUIREMENTS
- In `OrderCreateSerializer`, add an OPTIONAL field:
  `address_id = serializers.IntegerField(required=False, allow_null=True)`
- In `validate()`, if `address_id` is provided: look up
  `Address.objects.get(pk=address_id, user=request.user)` (raise a
  `serializers.ValidationError({"address_id": "Address not found."})`
  if it doesn't exist OR doesn't belong to the requesting user — this
  is a real security check, not just a UX nicety: without the
  `user=request.user` filter, any authenticated user could submit
  ANY OTHER user's numeric address ID and have their order shipped
  using someone else's saved address, leaking that address's contents
  into their own order and potentially misdirecting a shipment).
  Import `Address` from `dashboard.models` at the top of
  order/serializers.py (check for any circular-import risk between
  `order` and `dashboard` apps first — `dashboard/views.py` already
  imports `from order.models import Order`, so importing FROM
  `dashboard` INTO `order` would be the reverse direction and could be
  circular if not handled carefully; if a circular import risk is
  confirmed, resolve it by importing `Address` lazily inside the
  `validate()` method body itself rather than at module level, which
  sidesteps the circularity since the import only executes at call
  time, not at module-load time).
  If a valid address is found, populate the checkout's address-related
  fields (`first_name`, `last_name`, `phone`, `address`, `apartment`,
  `city`, `state`, `zip`, `country`) FROM the resolved `Address` object
  into `attrs`, OVERRIDING any manually-typed values for those same
  fields that may have also been submitted (an `address_id` takes
  precedence — if a client somehow sends both, the saved address wins,
  since that's the more trustworthy source and prevents a confusing
  "which one applies" ambiguity).
  If `address_id` is NOT provided, fall back to requiring the existing
  manually-typed fields exactly as before (don't make ALL the
  individual address fields required=False at the serializer level in
  a way that could let a checkout submit with NEITHER a saved address
  NOR manual fields — enforce in `validate()` that at least one path
  fully supplies a valid address, raising a clear error if neither
  does).
- Add an OPTIONAL `save_address = serializers.BooleanField(default=False)`
  field: if the customer checked out with manually-typed address fields
  (not an existing `address_id`) and set this flag, create a NEW
  `Address` row for them from the submitted checkout fields as part of
  `create()`, so future checkouts can reuse it — a natural, expected
  "save this address for next time" checkbox.

ACCEPTANCE CRITERIA / TESTS
Add tests to the order test suite:
1. Checkout with a valid `address_id` belonging to the requesting user
   succeeds and the resulting Order's shipping fields match the saved
   Address's fields exactly.
2. Checkout with an `address_id` belonging to a DIFFERENT user is
   rejected with a 400/404-style validation error (the ownership check
   from the security note above) — this is the most important test in
   this task.
3. Checkout with NO `address_id` and full manually-typed fields
   continues to work exactly as before (regression check).
4. Checkout with neither an `address_id` NOR complete manual fields
   fails with a clear validation error.
5. Checkout with manual fields AND `save_address=true` results in a
   new `Address` row being created for the user after the order
   completes, with fields matching what was submitted.
```

---

#### Task 5.2.1.4 — Iranian address fields (province, city, postal code format)

```
You are working in backend/dashboard/models.py (Address model) and
backend/dashboard/serializers.py (AddressSerializer). Assume Task
5.2.1.3 is already merged.

CONTEXT
The existing `Address` model uses generic US-shaped fields:
`state` (free CharField), `zip_code` (free CharField, no format
validation), `country` (CharField defaulting to `"US"`). This platform
targets the Iranian market exclusively (per the project's stated goal)
— these fields need to become Iran-appropriate: `state` should really
represent an Iranian PROVINCE (Iran has 31 official provinces, a fixed,
well-known list well-suited to a choices field rather than free text),
and `zip_code` should validate against Iran's actual postal code format
(a 10-digit numeric code, not the 5-digit US ZIP format the current
field's `max_length=20` vaguely accommodates but doesn't specifically
validate).

TASK
Replace the generic `state` field with an Iran-specific `province`
choices field, add real format validation to the postal code field,
and default `country` sensibly for this platform's single-market focus
— while keeping the underlying schema flexible enough not to hard-break
if the business ever expands beyond Iran (don't delete `country`
entirely, just change its default and add it to `list_filter` if
useful).

REQUIREMENTS
- Add an `IranProvince` TextChoices enum to backend/dashboard/models.py
  (or a new backend/dashboard/choices.py if you prefer to keep
  models.py from growing too large — check the file's current length
  first and decide) listing Iran's 31 provinces:
  ```python
  class IranProvince(models.TextChoices):
      TEHRAN = "tehran", "Tehran"
      ISFAHAN = "isfahan", "Isfahan"
      FARS = "fars", "Fars"
      KHORASAN_RAZAVI = "khorasan_razavi", "Khorasan Razavi"
      EAST_AZERBAIJAN = "east_azerbaijan", "East Azerbaijan"
      WEST_AZERBAIJAN = "west_azerbaijan", "West Azerbaijan"
      # ... continue with the complete, accurate list of all 31
      # Iranian provinces. Verify the full official list (Ostan) via a
      # reliable, authoritative source before finalizing — do not
      # guess at names/count from general knowledge alone, since an
      # incomplete or misspelled province list directly affects real
      # customers' ability to enter a valid shipping address; if you
      # have web-search or reference-lookup capability available in
      # your environment, use it to verify the complete, correctly
      # spelled/transliterated list of all 31 provinces before writing
      # this enum.
  ```
- Rename `Address.state` to `Address.province` with
  `choices=IranProvince.choices` (this is a field RENAME plus a type
  change from free-text to choices-constrained — Django's
  `makemigrations` will likely detect this as "delete state, add
  province" rather than a true rename unless you explicitly confirm
  the rename prompt during interactive `makemigrations`, or hand-edit
  the migration to use `migrations.RenameField` followed by an
  `AlterField` to add the choices constraint, which correctly PRESERVES
  existing data instead of dropping the column and losing every
  existing address's state value — do this the RenameField way, not
  the drop-and-recreate way, since this is existing user data).
  Since existing `Address.state` rows contain arbitrary free text (not
  necessarily matching any of the new `IranProvince` choice values —
  e.g. could contain `"CA"`, `"NY"`, garbage test data, etc.), a plain
  rename will leave those existing values technically present in the
  renamed column but NOT valid against the new choices constraint —
  decide and implement a sensible data-migration step: for any existing
  value that doesn't match a valid `IranProvince` choice, either clear
  it to blank (safest, avoids showing a garbage value in a dropdown
  that only offers valid options) or leave it as free text or, if this
  is genuinely pre-launch/test data with no real customers yet, treat
  data preservation as less critical and it's acceptable to note in a
  migration comment that this assumes no real production address data
  exists yet — make a deliberate, documented choice rather than an
  accidental one.
- Add postal code validation: add a validator to `zip_code` (rename to
  `postal_code` for clarity, applying the same RenameField-then-AlterField
  migration approach as above) using a RegexValidator for Iran's format:
  `RegexValidator(regex=r'^\d{10}$', message="Enter a valid 10-digit Iranian postal code.")`.
  Import `RegexValidator` from `django.core.validators`.
- Change `country`'s default from `"US"` to `"IR"` (this platform's
  single target market), but leave the field itself as a plain
  CharField rather than hard-restricting it to only `"IR"` via choices
  — a hard-coded single-value choices field would be needlessly
  restrictive if the business ever adds international shipping later,
  and the default alone is sufficient to make Iran the practical
  default without foreclosing future flexibility.
- Update `AddressSerializer` in backend/dashboard/serializers.py to
  reflect the renamed `province`/`postal_code` fields (check its
  current `Meta.fields` list and update field names accordingly).
- Update `OrderCreateSerializer`'s corresponding `state`/`zip` input
  fields (from Task 5.2.1.3's checkout wiring) to match — actually,
  reconsider: `Order.shipping_state`/`Order.shipping_zip` on the
  `Order` model itself are plain, unconstrained `CharField`s (no
  choices, no format validator) used purely as a frozen SNAPSHOT at
  order time, so they don't strictly need the same rename/constraint
  treatment as the live, editable `Address` model — but for
  consistency and to catch bad data at the CHECKOUT FORM input layer
  (not just the saved-address-book layer), update
  `OrderCreateSerializer`'s `state`/`zip` fields to also validate
  against `IranProvince` choices and the postal code regex,
  respectively, so a customer typing a manual (non-saved) address at
  checkout gets the same validation rigor as one using a saved address
  from the now-constrained `Address` model. Leave the underlying
  `Order.shipping_state`/`shipping_zip` CharFields on the Order model
  itself unchanged (they remain generic storage for the validated,
  already-correct snapshot value — no reason to add DB-level choice
  constraints to a historical snapshot field).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/:
1. Creating an `Address` with a valid `IranProvince` choice and a valid
   10-digit `postal_code` succeeds.
2. Creating an `Address` with an invalid/non-Iran province value fails
   `full_clean()`.
3. Creating an `Address` with a postal code that isn't exactly 10
   digits (too short, too long, contains letters) fails `full_clean()`.
4. The migration correctly preserves existing `Address` rows' OTHER
   fields (first_name, last_name, address_line, city, etc.) untouched
   — only `state`→`province` and `zip_code`→`postal_code` are affected.
Add/update tests to the order test suite confirming
`OrderCreateSerializer`'s manual (non-saved-address) checkout path now
rejects an invalid province/postal code the same way the Address model
does.
```

---

#### Task 5.2.1.5 — Frontend address-book UI

```
You are working in frontend/src (Checkout page and any existing
Account/Profile address-management page). Assume Tasks 5.2.1.3 and
5.2.1.4 are already merged and the `/dashboard/addresses/` API now
returns/accepts `province`/`postal_code` fields.

CONTEXT
Check first whether ANY frontend UI already exists for managing
addresses via the pre-existing `AddressViewSet` API (search
frontend/src/pages/Account.jsx and related components for any
"address" references) — since the backend CRUD has existed all along
(per Task 5.2.1.3's discovery), it's plausible SOME frontend UI already
half-exists for it even if checkout was never wired to use it; don't
assume a completely blank slate without checking. Whatever you find,
this task's job is: (1) ensure a working address-management UI exists
somewhere in the account area (build one if genuinely missing, using
the `AddressViewSet` API `list/create/update/delete` at
`/dashboard/addresses/`), and (2) add a SAVED-ADDRESS PICKER to the
checkout flow itself, which is the part that's definitely new
regardless of what account-page UI already existed.

TASK
Build/complete an address book management UI in the account area, and
add a saved-address selector to checkout that lets a logged-in user
either pick a saved address or enter a new one (with an option to save
it), per the backend capability from Task 5.2.1.3.

REQUIREMENTS
- Add API methods to frontend/src/services/api.js if not already
  present (check for an existing `dashboardAPI`/`addressAPI` object
  first):
  ```javascript
  listAddresses: () => api.get('/dashboard/addresses/'),
  createAddress: (data) => api.post('/dashboard/addresses/', data),
  updateAddress: (id, data) => api.patch(`/dashboard/addresses/${id}/`, data),
  deleteAddress: (id) => api.delete(`/dashboard/addresses/${id}/`),
  ```
  ensuring field names sent (`province`, `postal_code`) match the
  backend's post-Task-5.2.1.4 field names, not the old `state`/`zip_code`
  names.
- Account-area address management (build if missing): a list of the
  user's saved addresses (label, name, formatted address), each with
  edit/delete actions, and an "Add new address" form. The province
  field should render as a `<select>` populated from Iran's 31
  provinces (either hardcode the list in the frontend matching the
  backend's `IranProvince` enum exactly — keep these two lists in sync
  manually and note this in a comment, since there's no automatic
  single-source-of-truth sharing between a Django choices enum and a
  React component unless you add an API endpoint exposing valid
  choices dynamically, which is out of scope for this task — or fetch
  the choices from the DRF `OPTIONS` request on the addresses endpoint,
  which DRF automatically supports and would keep the two in sync
  automatically; prefer the OPTIONS-based approach if it's not
  significantly more complex to implement, since manual list
  duplication is a known source of drift bugs).
- Checkout page: when the user is authenticated (has saved addresses
  possible) AND has at least one saved address, show a picker (radio
  buttons or a dropdown of "Home — 123 Example St...", etc., plus an
  explicit "Enter a new address" option) ABOVE the manual address
  entry form. Selecting a saved address should populate the checkout
  submission with that `address_id` (per the backend's Task 5.2.1.3
  contract) rather than sending the individual address fields; choosing
  "enter a new address" reveals the full manual form (with province
  dropdown/postal code validation matching the account-page form) and
  should include a "Save this address for next time" checkbox wired to
  the `save_address` boolean from Task 5.2.1.3.
  For guest/anonymous checkout (per Feature 5.1.1's guest cart work),
  there's no saved-address concept at all — always show the manual
  entry form with no picker and no "save for next time" option (an
  anonymous user has nowhere to save it TO, since `Address.user` is
  required — don't attempt to build guest-address-saving, that's out of
  scope).

ACCEPTANCE CRITERIA
- Manually verify: a logged-in user with at least one saved address
  sees the picker at checkout, selecting one correctly completes
  checkout using that address; a logged-in user with NO saved addresses
  sees only the manual form (no empty/broken picker UI); entering a new
  address with "save for next time" checked results in it appearing in
  their account address book afterward; a guest/anonymous user sees
  only the manual form with no picker or save option.
- Add component tests (Vitest, matching existing conventions) for the
  address-picker component specifically: renders correctly with 0
  addresses, renders correctly with N addresses, selecting an address
  updates the checkout form state to reference that `address_id`, and
  choosing "enter a new address" reveals the manual form fields.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 5.1.1.1 | Make Cart.user nullable, add session_key | ☐ |
| 5.1.1.2 | Cart helper resolves by user-or-session | ☐ |
| 5.1.1.3 | Merge guest cart into user cart on login | ☐ |
| 5.1.1.4 | Frontend cart calls work pre-login | ☐ |
| 5.2.1.1 | Re-validate stock at checkout submit | ☐ |
| 5.2.1.2 | Re-validate price at checkout (server-authoritative) | ☐ |
| 5.2.1.3 | Wire checkout to existing Address model | ☐ |
| 5.2.1.4 | Iranian address fields (province/postal code) | ☐ |
| 5.2.1.5 | Frontend address-book UI | ☐ |

Once Epic 5 is fully merged, the next epic to generate prompts for is
**Epic 6 — Payments (Iranian Gateways)**, which is the single biggest
"this isn't a real store yet" gap identified in the original review —
it builds on this epic's now-hardened, guest-capable, address-aware
checkout flow to actually take real money via ZarinPal (and, later,
Zibal/IDPay).
