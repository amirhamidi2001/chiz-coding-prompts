# Epic 1 — Core Backend Stability — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** each task below is a complete, standalone prompt you can hand directly to a coding agent (Claude Code, Cursor, etc.) working inside the repo. Run them **in order** — later tasks assume earlier ones in the same Feature (and, where noted, earlier Features) are already merged. Each prompt is scoped to one sitting (30 min–2 hrs) and includes the current relevant code so the agent doesn't need to guess at context, exact file paths to touch, hard requirements, edge cases to handle, and the acceptance criteria/tests that define "done."

Do not paste multiple task prompts into the same agent session at once — feed them one at a time and let each be committed/reviewed before starting the next.

---

## Phase 1.1 — Transactional Order Safety

### Feature 1.1.1 — Atomic Order Creation

---

#### Task 1.1.1.1 — Wrap order creation in `transaction.atomic()`

```
You are working in a Django + DRF e-commerce backend at backend/order/.

CONTEXT
The order creation flow lives in backend/order/serializers.py, in
OrderCreateSerializer.create(). It currently does several unguarded
database writes in sequence: creates the Order row, loops over cart
items creating OrderItem rows, then deletes the cart's items. If any
step after the first raises an exception (e.g. a bad product image URL,
a DB constraint violation, a server restart), the Order is left
committed with missing or partial OrderItem rows and the cart is not
cleared — corrupting order history and confusing the customer.

TASK
Wrap the entire body of OrderCreateSerializer.create() (from Order.objects.create(...)
through cart.items.all().delete()) in a single django.db.transaction.atomic()
block, so the whole operation succeeds or fails as one unit.

REQUIREMENTS
- Import `from django.db import transaction` at the top of order/serializers.py.
- Use `with transaction.atomic():` wrapping the existing logic — do not change
  the order of operations or any field values, this task is purely about
  atomicity.
- Any exception raised inside the block (e.g. from Order.objects.create or
  OrderItem.objects.create) must roll back ALL writes made in this method,
  including the Order row itself.
- Do not touch OrderListCreateView.post() in order/views.py — the atomic
  block belongs inside the serializer's create() method, not the view.
- Do not change any public method signatures.

ACCEPTANCE CRITERIA / TESTS
Add a test to backend/order/tests.py (or convert it to a tests/ package
matching the pattern used in backend/shop/tests/ if that's cleaner — your
call, but stay consistent with the existing test runner config) that:
1. Mocks/monkeypatches OrderItem.objects.create to raise an exception on
   the second call (simulating a mid-loop failure with a cart containing
   2+ items).
2. Asserts that after the exception propagates, Order.objects.count() is
   unchanged from before the call (i.e. no orphaned Order row was
   committed).
3. Asserts the cart's items were NOT deleted (cart.items.count() unchanged).

Run the full existing order test suite afterward and confirm nothing
that previously passed now fails.
```

---

#### Task 1.1.1.2 — Add `select_for_update()` on product stock read

```
You are working in backend/order/serializers.py, inside
OrderCreateSerializer.create(), which you previously wrapped in
transaction.atomic() (Task 1.1.1.1 — assume that change is already merged).

CONTEXT
Right now the loop over cart items reads `cart_item.product` (already
loaded via the queryset `cart.items.select_related("product").prefetch_related("product__images")`)
without locking the underlying Product row. If two users check out
carts that reference the same low-stock product at the same moment,
both requests can read the same stock value before either writes,
allowing both orders to succeed even if there isn't enough stock for
both — an overselling race condition.

TASK
Introduce a row-level lock on each Product referenced by the cart at
the start of order creation, so concurrent checkouts against the same
product serialize instead of racing.

REQUIREMENTS
- Inside the existing `transaction.atomic()` block (a lock only holds
  within a transaction), before creating OrderItem rows, fetch the
  Product rows referenced by the cart using
  `Product.objects.select_for_update().filter(id__in=<product ids from cart>)`
  and build a dict keyed by product id for O(1) lookup in the loop that
  follows. Do not lock rows that aren't part of this checkout.
- Replace the `product = cart_item.product` reference in the existing
  loop with a lookup into this locked dict, so the object used to build
  the OrderItem (name, slug, price, image, stock) comes from the locked
  instance, not the earlier unlocked select_related instance.
- Preserve all existing OrderItem field population logic exactly
  (product_name, product_slug, product_image, unit_price, quantity) —
  only the source of the `product` object should change to the locked
  version.
- Note: `select_for_update()` requires a real transaction and is not
  supported on SQLite locking the same way as Postgres — check
  backend/core/settings/*.py for which DB engine is configured for
  tests, and if tests run against SQLite, make sure this doesn't break
  the test suite (e.g. guard with `connection.vendor != 'sqlite'` only
  if truly necessary, but prefer just running tests against Postgres
  via the existing docker-compose test service if one exists — check
  docker-compose.yml / CI config first before adding a workaround).

ACCEPTANCE CRITERIA / TESTS
- Add a test confirming `select_for_update` is actually used in the
  query (e.g. via `str(queryset.query)` containing "FOR UPDATE" when run
  against Postgres, or via mocking `select_for_update` and asserting
  it was called with the expected product ids).
- Re-run the full order test suite; all existing tests must still pass
  since output behavior (the created OrderItem fields) is unchanged —
  only locking semantics are added.
```

---

#### Task 1.1.1.3 — Decrement product stock on order creation

```
You are working in backend/order/serializers.py
(OrderCreateSerializer.create()) and backend/shop/models.py (Product model).
Assume Tasks 1.1.1.1 (atomic block) and 1.1.1.2 (select_for_update locking)
are already merged — you now have a dict of locked Product instances
available inside the transaction before the OrderItem creation loop.

CONTEXT
Product.stock (a PositiveIntegerField on backend/shop/models.py) is
currently only ever checked when items are added to the cart
(backend/cart app) — it is never decremented when an order is actually
placed. This means stock never goes down as orders come in, and the
"stock" number becomes meaningless / customers can order far more than
physically exists.

TASK
Inside the existing atomic order-creation block, for every cart item,
validate that the locked product has enough stock for the requested
quantity, and if so, decrement Product.stock by that quantity and save
it. If stock is insufficient for ANY item, the whole order must fail
and nothing should be committed (leverage the atomic block from Task
1.1.1.1 — raising an exception inside it rolls everything back).

REQUIREMENTS
- Before creating each OrderItem, check:
  `if locked_product.stock < cart_item.quantity: raise serializers.ValidationError(...)`
  with a clear, per-product message identifying which product doesn't
  have enough stock (include product name and the quantity available),
  e.g. `{"stock": f"Only {locked_product.stock} of '{locked_product.name}' left in stock."}`
- Do the stock check for ALL items in the cart BEFORE creating ANY
  OrderItem or decrementing ANY stock — i.e. two passes over the cart
  items (validate-all, then commit-all), OR validate-then-decrement per
  item is acceptable too SINCE we're inside transaction.atomic() and a
  raised exception will roll back partial decrements either way. Prefer
  whichever is simpler to read; a single loop that checks-then-decrements
  per item is fine given the atomic wrapper guarantees rollback.
- After the check passes, decrement: `locked_product.stock -= cart_item.quantity`
  and save with `locked_product.save(update_fields=["stock"])`.
- This validation error must surface as a 400 response with a clear
  field-level error, matching the existing DRF error-handling pattern
  used elsewhere in the serializer (look at how `validate()` raises
  serializers.ValidationError with a dict for the "cart" field as a
  reference pattern).
- Do NOT decrement stock in the `validate()` method — validation should
  remain side-effect free; the decrement belongs in `create()`.

ACCEPTANCE CRITERIA / TESTS
Add tests to the order test suite:
1. Ordering a quantity <= available stock succeeds and
   Product.objects.get(pk=product.pk).stock is reduced by exactly the
   ordered quantity.
2. Ordering a quantity > available stock raises a ValidationError / the
   API returns 400, AND no Order or OrderItem rows exist afterward, AND
   the product's stock is completely unchanged (proving the atomic
   rollback works end-to-end with this new logic).
3. A cart with two items, where the first has enough stock and the
   second doesn't, results in NEITHER item's stock being decremented
   (proving partial-success isn't possible).
```

---

#### Task 1.1.1.4 — Restore stock on order cancellation

```
You are working in backend/order/views.py (OrderDetailView.patch) and
backend/shop/models.py. Assume Task 1.1.1.3 (stock decrement on order
creation) is already merged.

CONTEXT
OrderDetailView.patch() lets a user cancel their own order (only if it
hasn't shipped/delivered yet), setting order.status to
Order.Status.CANCELLED. It does not currently return any reserved
stock to the product — meaning every cancelled order permanently
"loses" that inventory from the system even though nothing was ever
actually shipped.

TASK
When an order is successfully cancelled via this endpoint, restore the
quantity of every OrderItem back to its corresponding Product's stock.

REQUIREMENTS
- In OrderDetailView.patch(), after the existing checks pass (status is
  not shipped/delivered) and before/around setting
  `order.status = Order.Status.CANCELLED`, wrap the status change and
  stock restoration in `transaction.atomic()` (import from
  django.db.transaction, matching the pattern from Task 1.1.1.1).
- For each `order.items.all()` (note: OrderItem.product is
  `on_delete=models.SET_NULL`, so `product` can be None if the product
  was deleted after the order was placed — skip stock restoration for
  those items gracefully, don't raise an error).
- Use `select_for_update()` on the Product being restored, same locking
  pattern as Task 1.1.1.2, to avoid races with a concurrent purchase of
  the same product while this cancellation is being processed.
- Increment `product.stock` by `order_item.quantity` and save with
  `update_fields=["stock"]`.
- Keep the existing 400 response for attempting to cancel a
  shipped/delivered order unchanged.
- The response payload (OrderSerializer output) should remain
  unchanged in shape — this task only adds a side effect, not new
  response fields.

ACCEPTANCE CRITERIA / TESTS
Add tests to the order test suite:
1. Cancelling a pending order with items restores each item's quantity
   to the corresponding product's stock (assert exact before/after
   stock values).
2. Cancelling an order where one OrderItem's product was deleted
   (product=None) does not raise an exception and still restores stock
   for the remaining valid items.
3. Attempting to cancel a shipped order still returns 400 and does NOT
   modify any product's stock (regression check that the existing
   guard clause still runs before any stock logic).
```

---

#### Task 1.1.1.5 — Add out-of-stock race condition test suite

```
You are adding tests to the backend/order app. Assume Tasks 1.1.1.1
through 1.1.1.4 are already merged (atomic order creation, row locking,
stock decrement, stock restoration on cancel).

CONTEXT
Individual unit tests for each piece of stock logic already exist from
the previous four tasks, but there's no test that actually proves
concurrent requests against the same low-stock product can't oversell
in practice — the thing this entire feature exists to prevent.

TASK
Create a new test file backend/order/tests/test_stock_concurrency.py
(convert backend/order/tests.py into a backend/order/tests/ package with
an __init__.py if it isn't one already, preserving any existing tests
inside tests.py by moving them into tests/test_order.py or similar —
check the file first and migrate its contents rather than deleting it).

REQUIREMENTS
Write a concurrency test that:
1. Creates a Product with stock=1.
2. Creates two separate Users, each with a Cart containing a CartItem
   for that same product with quantity=1.
3. Uses Python's `threading` module (or `concurrent.futures.ThreadPoolExecutor`)
   to fire both users' POST /api/orders/ requests at approximately the
   same time (use Django's test Client or APIClient per thread — note
   each thread needs its own DB connection; wrap each thread's DB work
   appropriately, e.g. using `django.test.TransactionTestCase` instead
   of `TestCase`, since `TestCase` wraps each test in a transaction that
   doesn't work correctly across threads with `select_for_update`).
4. Asserts that exactly ONE of the two requests succeeds (201) and the
   other fails with a stock-related 400 error.
5. Asserts that after both requests complete, Product.objects.get(pk=...).stock == 0
   and there is exactly one Order/OrderItem for this product, not two.

Also add a second scenario test:
- Same setup but stock=5, and 3 concurrent users each ordering quantity=2
  (total demand of 6 against supply of 5) — assert final stock is never
  negative and at most 2 of the 3 orders succeed.

NOTES
- This test class should use `TransactionTestCase`, not `TestCase`,
  because `select_for_update` locking behavior needs real transaction
  boundaries across threads.
- If the project's test settings point at SQLite, you likely need to
  point this specific test module at the Postgres service already
  defined in docker-compose (check backend/core/settings/development.py
  and any pytest.ini/pytest-django config for DB routing) since SQLite
  doesn't support real row locking — document this clearly in a comment
  at the top of the file if a settings override is required to run it.

ACCEPTANCE CRITERIA
Both concurrency scenarios pass reliably when run repeatedly (run the
new test file 5 times in a loop locally to check for flakiness before
considering this done — a race-condition test that's itself racy is
worse than no test).
```

---

### Feature 1.1.2 — Extract Order Pricing Service

---

#### Task 1.1.2.1 — Create `order/services/pricing.py`

```
You are working in the backend/order Django app.

CONTEXT
backend/order/serializers.py currently has pricing logic inline inside
OrderCreateSerializer.create():

    TAX_RATE = Decimal("0.10")  # 10%
    SHIPPING_COST = Decimal("9.99")
    ...
    subtotal = Decimal(str(cart.subtotal))
    tax = (subtotal * TAX_RATE).quantize(Decimal("0.01"))
    total = (subtotal + SHIPPING_COST + tax - discount).quantize(Decimal("0.01"))

This math has no test coverage of its own (it's only exercised
indirectly through full order-creation tests) and will need to grow
significantly in later work (coupon-based discounts replacing the flat
discount field, real shipping-rate lookups replacing the flat $9.99,
Iranian tax rules). It needs to be extracted into its own testable unit
now, before more logic is piled onto it.

TASK
Create backend/order/services/__init__.py and
backend/order/services/pricing.py containing a `PricingService` class
(or a set of pure functions, if that reads more cleanly — your call, but
be consistent) that owns all order total calculation.

REQUIREMENTS
- Move `TAX_RATE` and `SHIPPING_COST` constants into pricing.py.
- Expose a function/method, e.g.:
  `calculate_order_totals(subtotal: Decimal, discount: Decimal = Decimal("0")) -> dict`
  returning `{"subtotal": ..., "shipping_cost": ..., "tax": ..., "discount": ..., "total": ...}`
  with the exact same math and rounding (`.quantize(Decimal("0.01"))`) as
  the current inline code, so behavior is byte-for-byte identical.
- Add input validation: raise `ValueError` (or a custom
  `PricingError` exception defined in the same module) if `subtotal` is
  negative, or if `discount` is negative, or if `discount > subtotal + shipping_cost + tax`
  (a discount should never be able to make the total negative) — this
  guards against a future misuse of the service, not just today's code
  path.
- Do NOT wire this into OrderCreateSerializer yet — that's Task 1.1.2.2.
  This task only creates and unit-tests the standalone service.
- Type-hint all public functions/methods.

ACCEPTANCE CRITERIA / TESTS
Task 1.1.2.3 covers full test creation, but before considering this
task done, write at least a smoke test confirming
`calculate_order_totals(Decimal("100.00"))` returns the same values the
current inline formula would produce for a $100 subtotal with the
default tax rate and shipping cost, so the extraction itself introduced
zero behavior change.
```

---

#### Task 1.1.2.2 — Refactor `OrderCreateSerializer` to call `PricingService`

```
You are working in backend/order/serializers.py. Assume Task 1.1.2.1
(order/services/pricing.py with calculate_order_totals()) and Task
1.1.1.1 (transaction.atomic() wrapping) are already merged.

CONTEXT
OrderCreateSerializer.create() still computes subtotal/tax/total inline
even though the logic now also exists in order/services/pricing.py.
This is dead-code duplication risk — the two must be consolidated.

TASK
Replace the inline pricing math in OrderCreateSerializer.create() with a
call to the new pricing service.

REQUIREMENTS
- Import `calculate_order_totals` from `order.services.pricing` at the
  top of serializers.py.
- Delete the inline `TAX_RATE`/`SHIPPING_COST` module-level constants
  and the inline subtotal/tax/total calculation lines from create().
- Replace them with a single call:
  `totals = calculate_order_totals(subtotal=Decimal(str(cart.subtotal)), discount=discount)`
  and use `totals["subtotal"]`, `totals["shipping_cost"]`,
  `totals["tax"]`, `totals["discount"]`, `totals["total"]` when
  constructing the Order object.
- Catch the `PricingError`/`ValueError` raised by the service (per Task
  1.1.2.1's validation) and re-raise it as a
  `serializers.ValidationError` with an appropriate field key (e.g.
  `{"discount": str(e)}`) so it surfaces as a normal 400 API response
  rather than a 500.
- Do not change any other part of create() (customer info, shipping
  address, OrderItem creation, cart clearing).

ACCEPTANCE CRITERIA / TESTS
- Re-run the entire existing order test suite (including the new tests
  from Feature 1.1.1) — all must still pass with identical output
  values, proving the refactor is behavior-neutral.
- Add one new test that a malformed discount (e.g. one larger than the
  order total, if that's reachable at this layer — check whether Task
  1.2.1 has already removed the discount input field by the time you do
  this; if so this edge case may be untestable via the public API
  anymore and you should note that in a comment rather than force a
  test) is rejected with a 400 and a clear error message.
```

---

#### Task 1.1.2.3 — Unit tests for `PricingService`

```
You are adding tests for backend/order/services/pricing.py. Assume Task
1.1.2.1 exists.

CONTEXT
The pricing service is the single most financially sensitive piece of
code in the codebase — every dollar amount charged to every customer
flows through it. It currently only has a smoke test from Task 1.1.2.1.
It needs full, dedicated unit test coverage independent of the Order
model or any Django views, since it's meant to be a pure, fast-to-test
unit.

TASK
Create backend/order/tests/test_pricing_service.py (or add to the
tests/ package structure established in Task 1.1.1.5) with comprehensive
coverage of `calculate_order_totals`.

REQUIREMENTS — test cases to include
1. Standard case: known subtotal, zero discount → assert exact
   subtotal/tax/shipping/total values match hand-calculated expected
   Decimals.
2. Discount reduces total correctly, and total is still rounded to 2
   decimal places.
3. Zero subtotal (edge case — should this even be allowed? Decide and
   assert the actual chosen behavior — either it computes cleanly to
   zero-plus-shipping-plus-tax-on-zero, or it raises; pick whichever
   the service currently does and lock it in with a test).
4. Negative subtotal raises the expected exception.
5. Negative discount raises the expected exception.
6. Discount exactly equal to (subtotal + shipping + tax) results in
   total == 0, not negative — boundary condition.
7. Discount greater than (subtotal + shipping + tax) raises the
   expected exception rather than producing a negative total.
8. Verify rounding behavior explicitly with a subtotal that produces a
   repeating decimal in the tax calculation (e.g. subtotal that isn't a
   clean multiple of 10) to confirm `.quantize(Decimal("0.01"))` is
   applied correctly and consistently (banker's rounding vs. standard
   rounding — confirm which Python's Decimal.quantize default does and
   that it matches what the business expects; note this in a test
   comment even if you don't change behavior, so future readers know
   it was a conscious check).

Aim for these tests to run with zero database hits (no Django TestCase
needed if the service has no ORM dependency — use plain `unittest.TestCase`
or pytest functions if that matches the project's existing test
tooling; check pytest.ini/setup.cfg for the configured test runner
first).

ACCEPTANCE CRITERIA
All listed cases pass, and running `pytest backend/order/tests/test_pricing_service.py -v`
(or the Django-equivalent test command used elsewhere in this repo) shows
each case as an individually named, readable test — not one giant test
function.
```

---

## Phase 1.2 — Discount/Coupon Integrity

### Feature 1.2.1 — Remove Client-Controlled Discount

---

#### Task 1.2.1.1 — Remove client-controlled `discount` field from checkout API

```
You are working in backend/order/serializers.py and the frontend
checkout flow. Assume all of Feature 1.1.1 and 1.1.2 are already
merged (order creation is atomic, stock-safe, and pricing goes through
PricingService).

CONTEXT — SECURITY ISSUE
OrderCreateSerializer currently has:

    discount = serializers.DecimalField(
        max_digits=10, decimal_places=2, min_value=Decimal("0"), default=Decimal("0")
    )

This field is read directly from the client-submitted checkout payload
with no server-side validation of WHY a discount is being applied —
there is no Coupon model, no minimum-order check, nothing. Any client
can submit an arbitrary `discount` value in the POST body (e.g. via
browser dev tools or a raw curl request) and receive that discount at
checkout. This is a real, exploitable revenue-loss vulnerability, not a
style issue — treat it as a security fix, not a refactor.

TASK
Remove client control over the discount value entirely. Until a real
coupon system exists (tracked separately as Epic 9 in the project
backlog — do not build coupons here, that's out of scope for this
task), discount must always be 0 and must not be settable via the
public checkout API at all.

REQUIREMENTS — backend
- Delete the `discount` field from `OrderCreateSerializer` entirely (do
  not just make it read-only — remove it so the client can't send it
  and have it silently ignored either; DRF will otherwise still parse
  and ignore unknown fields, which is fine, but the field must no
  longer be part of the serializer's declared inputs or validated
  range).
- In `create()`, call
  `calculate_order_totals(subtotal=..., discount=Decimal("0"))`
  — hardcode zero discount at the call site for now (leave a `# TODO:
  Epic 9 — replace with server-validated coupon discount` comment so
  the next engineer knows this is intentionally temporary, not
  forgotten).
- Confirm the `Order.discount` model field itself is untouched (it
  should remain on the model — Epic 9 will populate it from a real
  coupon service later; this task only removes the current insecure
  client-input path).

REQUIREMENTS — frontend
- Search the frontend checkout code (frontend/src, likely a Checkout
  component and/or CartContext) for anywhere a `discount` value is
  read from user input or sent in the checkout POST payload, and remove
  it from the outgoing request body. If there's UI showing a coupon-code
  input box that maps to this field, either remove that UI element or
  clearly disable/hide it with a comment noting coupons aren't
  implemented yet (do not leave dead UI that implies a working feature).

ACCEPTANCE CRITERIA / TESTS
Covered in Task 1.2.1.2 below — implement this task's code changes
first, then write that test to lock in the fix.
```

---

#### Task 1.2.1.2 — Add regression test: forged discount is ignored

```
You are adding a security regression test to backend/order/tests/.
Assume Task 1.2.1.1 (discount field removed from OrderCreateSerializer)
is already merged.

CONTEXT
This test exists specifically to prevent the vulnerability fixed in
Task 1.2.1.1 from silently regressing in the future (e.g. if a future
engineer re-adds a `discount` field to the serializer without also
adding the validation Epic 9 will eventually require).

TASK
Write a test that attempts to exploit the old vulnerability directly
against the live API and asserts it no longer works.

REQUIREMENTS
Add to backend/order/tests/ (whichever file holds general order-creation
API tests):

1. Build a normal, valid checkout POST payload (matching every field
   OrderCreateSerializer currently expects) for a cart with a known
   subtotal.
2. Manually inject an extra `"discount": "9999.99"` key into that
   payload dict before sending the POST request (simulating a forged
   request from a client that knows the old field name, e.g. via
   browser dev tools).
3. Assert the request either succeeds with the discount completely
   ignored (order.total matches the undiscounted expected total exactly)
   OR fails with a 400 due to the unexpected field — either outcome is
   acceptable AS LONG AS the resulting behavior never reduces the order
   total based on the injected value. Explicitly assert
   `order.discount == Decimal("0.00")` and `order.total` equals the
   value `calculate_order_totals(subtotal, discount=Decimal("0"))["total"]`
   would produce for that cart.
4. Add a second test that omits any discount field at all from the
   payload and confirms this still works exactly as before (no
   regression to the "happy path" checkout of a normal, non-attacking
   customer).

ACCEPTANCE CRITERIA
Both tests pass. Additionally, briefly grep the codebase
(`grep -rn "discount" backend/order/serializers.py`) as a manual sanity
check and confirm no client-writable discount path remains anywhere in
the checkout serializer before marking this done.
```

---

## Phase 1.3 — Review Authenticity

### Feature 1.3.1 — Authenticated, Purchase-Verified Reviews

---

#### Task 1.3.1.1 — Add `user` FK to `Review` model

```
You are working in backend/shop/models.py (Review model),
backend/shop/serializers.py (ReviewSerializer, ReviewCreateSerializer),
and will need a new migration.

CONTEXT
The current Review model (backend/shop/models.py) is:

    class Review(models.Model):
        product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name="reviews")
        name = models.CharField(max_length=200)
        rating = models.PositiveSmallIntegerField(choices=[(i, i) for i in range(1, 6)])
        headline = models.CharField(max_length=300, blank=True)
        comment = models.TextField()
        created_at = models.DateTimeField(auto_now_add=True)

`name` is free text supplied by whoever submits the review — there is
no link to an actual authenticated User. This is what makes fake/spam
reviews trivially easy today (see Task 1.3.1.3 for the permission fix;
this task is just the schema change it depends on).

TASK
Add a nullable `user` ForeignKey to `settings.AUTH_USER_MODEL` on the
Review model, while KEEPING the existing `name` field as a denormalized
display cache (so historical reviews and the public-facing display
logic don't need to change shape).

REQUIREMENTS
- Add:
  `user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True, blank=True, related_name="reviews")`
  to the Review model. Import `from django.conf import settings` if not
  already imported in shop/models.py.
- Make it nullable (`null=True, blank=True`) since existing rows in the
  database have no associated user and this must be a safe, non-destructive
  migration — do NOT attempt to backfill user from the free-text `name`
  field (there's no reliable way to match a name string to a real user
  account; leave existing rows with `user=None`).
- Generate the migration
  (`python manage.py makemigrations shop`) and include it in your
  changes — do not hand-write the migration file.
- Update `ReviewSerializer` (read serializer) in shop/serializers.py to
  include a read-only `user_id` field (do NOT expose the full nested
  User object — just the id, to avoid leaking email/other user PII in
  the public review list endpoint). Keep the existing `name` field in
  the output unchanged for backward frontend compatibility.
- Do NOT change `ReviewCreateSerializer`'s fields yet, and do NOT change
  the view's permission_classes yet — those are Tasks 1.3.1.3 onward.
  This task is model + migration + read-serializer only.

ACCEPTANCE CRITERIA / TESTS
- Migration applies cleanly against the existing SQLite/Postgres test
  DB with `python manage.py migrate` with no errors, on top of existing
  seed/fixture data if any exists in backend/shop/tests/.
- Add a model test confirming a Review can still be created with
  `user=None` (backward compatibility with any code path that hasn't
  been updated yet) and separately with a real `user` instance attached.
- Run the full existing shop test suite (backend/shop/tests/) and
  confirm no existing test breaks from this additive schema change.
```

---

#### Task 1.3.1.2 — Add `is_verified_purchase` boolean to `Review`

```
You are working in backend/shop/models.py. Assume Task 1.3.1.1 (user FK
on Review) is already merged.

CONTEXT
Once reviews are tied to real user accounts, the platform can
distinguish reviews from customers who actually bought the product from
anyone else — this is standard, expected e-commerce UX ("Verified
Purchase" badges) and is a prerequisite for Task 1.3.1.5, which
computes this value.

TASK
Add an `is_verified_purchase` boolean field to the Review model.

REQUIREMENTS
- Add:
  `is_verified_purchase = models.BooleanField(default=False)`
  to the Review model, placed logically near the `user` field added in
  Task 1.3.1.1.
- Generate and include the migration (default=False is safe for all
  existing rows — none of the historical free-text reviews are
  verifiable purchases).
- Add this field as a read-only field in `ReviewSerializer`'s output so
  the frontend can render a "Verified Purchase" badge later (frontend
  UI work itself is out of scope for this backend task).
- Do not add any logic yet that actually computes/sets this value —
  that's Task 1.3.1.5. This task only adds the field and exposes it
  for reading.

ACCEPTANCE CRITERIA / TESTS
- Migration applies cleanly.
- A serializer test confirms `is_verified_purchase` appears in
  ReviewSerializer output and defaults to `false` for a freshly created
  Review with no explicit value set.
```

---

#### Task 1.3.1.3 — Require authentication on `ProductReviewCreateView`

```
You are working in backend/shop/views.py (ProductReviewCreateView) and
backend/shop/serializers.py (ReviewCreateSerializer). Assume Tasks
1.3.1.1 and 1.3.1.2 are already merged.

CONTEXT — SECURITY ISSUE
ProductReviewCreateView currently has `permission_classes = [AllowAny]`,
with this comment directly in the code:

    """
    Permissions: AllowAny — the Review model uses a free-text `name` field
    rather than a FK to User, so no authentication is required.
    """

That rationale no longer holds now that Review has a real `user` FK
(Task 1.3.1.1). Anonymous, unauthenticated review submission is a spam
and defamation vector — anyone can currently post unlimited fake
reviews under any name for any product with zero accountability. Treat
this as a security fix.

TASK
Require authentication to submit a review, and attach the authenticated
user to the created Review automatically (rather than trusting a
client-submitted name).

REQUIREMENTS
- Change `permission_classes = [AllowAny]` to
  `permission_classes = [IsAuthenticated]` on `ProductReviewCreateView`
  (import `IsAuthenticated` from `rest_framework.permissions`).
- Update the class docstring to remove the now-incorrect rationale and
  replace it with an accurate description of the new behavior.
- In `perform_create()`, in addition to the existing
  `serializer.save(product=product)`, also pass
  `user=self.request.user` so every new review is tied to the real
  authenticated user: `serializer.save(product=product, user=self.request.user)`.
- Update `ReviewCreateSerializer` to auto-populate the `name` field from
  `self.request.user` (e.g. the user's first name, or full name — check
  what display-name field/property exists on the User/Profile model in
  backend/accounts/models.py and use that) rather than accepting `name`
  as free client input. Remove `name` from the client-writable
  `ReviewCreateSerializer.Meta.fields` tuple, and instead set it inside
  `perform_create()` or via a serializer `validate()`/`create()`
  override, whichever fits the existing serializer pattern more
  cleanly — your call, but the end result must be that the client
  cannot submit an arbitrary `name` valueanymore.
- Update `ReviewCreateSerializer.Meta.fields` to be `("rating", "headline", "comment")`
  only — `name` and `user` come from the server side now, not the
  client payload.
- Update backend/shop/urls.py / any API documentation strings
  (drf-spectacular schema descriptions, if present on this view) if
  they reference the old AllowAny/anonymous behavior.

ACCEPTANCE CRITERIA / TESTS
Update/add tests in backend/shop/tests/test_views.py:
1. An unauthenticated POST to `/api/products/<slug>/reviews/` now
   returns 401 (not 201).
2. An authenticated POST succeeds (201) and the created Review's `user`
   field matches the requesting user, and `name` matches that user's
   display name — even if the client tries to submit a different `name`
   value in the payload (prove the server ignores client-submitted name
   entirely; this is the same "don't trust client input" pattern as
   Task 1.2.1.2).
3. Re-run the full shop test suite and update any existing review tests
   that assumed anonymous access — they should now authenticate first.

Note: do NOT change the frontend in this task unless the existing
review-submission UI would break entirely without auth (in that case,
flag it clearly as a follow-up rather than silently expanding scope —
the frontend auth-gating UX (e.g. "log in to leave a review" prompt) is
better tracked as its own small task, but if leaving it unaddressed
would mean the storefront's review form silently 401s with no user
feedback, add the minimal error-handling needed so it fails gracefully,
not silently.
```

---

#### Task 1.3.1.4 — Enforce one review per user per product

```
You are working in backend/shop/models.py and
backend/shop/serializers.py. Assume Task 1.3.1.3 (authenticated
reviews, real user FK) is already merged.

CONTEXT
Now that reviews are tied to real authenticated users, nothing stops
the same logged-in user from submitting the same product's review
endpoint repeatedly, flooding a product with duplicate reviews from one
account. This should be a hard constraint, not just a UX nicety.

TASK
Enforce, at both the database and API level, that a given user can only
have one Review per Product.

REQUIREMENTS
- Add `unique_together = ("product", "user")` to Review's `Meta` class
  in backend/shop/models.py. Be careful: since `user` is nullable
  (Task 1.3.1.1), confirm Django's unique_together handling of NULL
  values (Postgres treats NULL as distinct from every other NULL for
  uniqueness purposes, so this constraint will correctly still allow
  multiple historical `user=None` reviews on the same product without
  conflict — verify this is actually true for whichever DB backend the
  project uses, and note it in a code comment if it's a meaningful
  caveat).
- Generate and include the migration.
- Add a `validate()` method to `ReviewCreateSerializer` (or override
  wherever's most appropriate) that checks
  `Review.objects.filter(product=product, user=request.user).exists()`
  BEFORE attempting to save, and raises a friendly
  `serializers.ValidationError({"detail": "You have already reviewed this product."})`
  — don't rely solely on the DB IntegrityError bubbling up as an ugly
  500; handle it at the application layer for a clean 400 response.
  (You'll need access to `product` and `request.user` inside this
  serializer method — check how the view currently passes `product`
  into the serializer/view flow and use the same pattern, e.g. via
  `self.context`.)

ACCEPTANCE CRITERIA / TESTS
1. A user submitting a first review for a product succeeds (201).
2. The same user submitting a second review for the SAME product fails
   with 400 and the friendly error message, and no second Review row is
   created.
3. The same user submitting a review for a DIFFERENT product still
   succeeds (proving the constraint is scoped correctly to
   product+user, not just user).
4. Two DIFFERENT users can each review the same product without
   conflict.
```

---

#### Task 1.3.1.5 — Compute `is_verified_purchase` at review creation

```
You are working in backend/shop/views.py (ProductReviewCreateView.perform_create)
and will need to query backend/order/models.py (Order, OrderItem).
Assume Tasks 1.3.1.1–1.3.1.4 are already merged.

CONTEXT
`Review.is_verified_purchase` (Task 1.3.1.2) exists as a field but is
never actually set — it always defaults to False. This task wires up
the real computation: a review should be marked verified if the
reviewing user has an OrderItem for this exact product on an order that
has actually been delivered (not just placed/pending — a "verified
purchase" should mean the customer genuinely received the product, not
merely added it to a cart and checked out).

TASK
In `ProductReviewCreateView.perform_create()`, before saving the
review, determine whether the requesting user has a qualifying
delivered order containing this product, and set
`is_verified_purchase` accordingly.

REQUIREMENTS
- Add a private helper method on the view, e.g.
  `_is_verified_purchase(self, user, product) -> bool`, that queries:
  `OrderItem.objects.filter(order__user=user, order__status=Order.Status.DELIVERED, product=product).exists()`
  — import Order/OrderItem from `order.models` at the top of
  shop/views.py (check for any risk of circular imports between the
  `shop` and `order` apps before doing this; if `order/models.py`
  already imports from `shop.models` — which it does, via
  `product = models.ForeignKey("shop.Product", ...)` using a lazy
  string reference — a direct Python-level import of `order.models`
  from `shop/views.py` should be safe since it's the reverse direction
  and Django lazy FK strings avoid circularity at the model-definition
  level, but confirm by running the server/tests after adding the
  import).
- Update `perform_create()`:
  `serializer.save(product=product, user=self.request.user, is_verified_purchase=self._is_verified_purchase(self.request.user, product))`
- Do not change anything about how `rating`/`reviews_count` aggregation
  works (`_refresh_product_stats`) — that logic is unaffected by this
  change and should remain exactly as-is.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_views.py:
1. A user with a DELIVERED order containing the product being reviewed
   gets `is_verified_purchase=True` on their created review.
2. A user with an order for the product that is still PENDING or
   PROCESSING (not yet delivered) gets `is_verified_purchase=False`.
3. A user with no order history for the product at all gets
   `is_verified_purchase=False`.
4. A user with a delivered order for a DIFFERENT product (not the one
   being reviewed) gets `is_verified_purchase=False` for this review
   (proving the check is product-specific, not just "has any delivered
   order ever").

Run the full shop + order test suites together at the end of this task
to confirm the cross-app query didn't break anything in either app.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 1.1.1.1 | Atomic order creation | ☐ |
| 1.1.1.2 | select_for_update on stock read | ☐ |
| 1.1.1.3 | Decrement stock on order creation | ☐ |
| 1.1.1.4 | Restore stock on cancellation | ☐ |
| 1.1.1.5 | Stock concurrency test suite | ☐ |
| 1.1.2.1 | Create PricingService | ☐ |
| 1.1.2.2 | Refactor serializer to use PricingService | ☐ |
| 1.1.2.3 | Unit tests for PricingService | ☐ |
| 1.2.1.1 | Remove client-controlled discount field | ☐ |
| 1.2.1.2 | Regression test: forged discount ignored | ☐ |
| 1.3.1.1 | Add user FK to Review | ☐ |
| 1.3.1.2 | Add is_verified_purchase field | ☐ |
| 1.3.1.3 | Require auth on review creation | ☐ |
| 1.3.1.4 | One review per user per product | ☐ |
| 1.3.1.5 | Compute is_verified_purchase | ☐ |

Once Epic 1 is fully merged, the next epic to generate prompts for is **Epic 2 — Authentication & Authorization (OTP)**, which depends on Epic 1 only loosely (it can technically run in parallel, but Task 2.3.1's guest-cart-merge work assumes Epic 5's cart changes, which in turn assume Epic 3's variant model — so sequence per the Execution Order Summary in the master backlog).
