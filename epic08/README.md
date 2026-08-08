# Epic 8 — Order Management (Admin Side) — AI Coding Prompts

Repo: `tablogenix-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–7 are fully merged (payments and shipping are now real).

**Important grounded discovery for this epic — read before starting Task 8.1.1.3 in particular:** `backend/dashboard/views.py` already has a working `AdminOrderViewSet` registered at `/dashboard/admin/orders/` (list/retrieve/status-update, DELETE deliberately disabled), backed by `AdminOrderFilter` (`backend/dashboard/filters.py`) which **already supports** `status`, `payment_method`, `date_from`/`date_to`, and `min_total`/`max_total`, plus `search_fields = ["order_number", "email", "first_name", "last_name", "phone"]`. This means the backlog's Task 8.1.1.3 ("Admin order search/filter") is **largely already built** — that task below is scoped around confirming and filling small gaps, not building from scratch. Also confirmed: `AdminOrderStatusSerializer.validate_status()` currently only checks that a submitted status is *a valid choice at all* (`value in [c[0] for c in Order.Status.choices]`) — it does **not** enforce any actual state-machine ordering (e.g. nothing stops flipping a `DELIVERED` order back to `PENDING`, or moving `CANCELLED` → `PROCESSING`). That's the real, concrete gap Task 8.1.1.1 closes.

**Also note:** the customer-facing cancellation path (`OrderDetailView.patch()` in `backend/order/views.py`, built out across Epics 1/3/4) and this epic's admin status-update path (`AdminOrderViewSet.partial_update()`) are **two separate code paths** today. Task 8.1.1.1 addresses how they should relate to each other rather than leaving them to silently diverge.

---

## Phase 8.1 — Order Lifecycle

### Feature 8.1.1 — Admin Order Operations

---

#### Task 8.1.1.1 — Admin order status transition endpoint with validation

```
You are working in backend/dashboard/serializers.py
(AdminOrderStatusSerializer), backend/dashboard/views.py
(AdminOrderViewSet), and backend/order/. Assume Epics 1–7 are fully
merged.

CONTEXT — THE REAL GAP, CONFIRMED DIRECTLY FROM THE REPO
`AdminOrderStatusSerializer` currently is:

    class AdminOrderStatusSerializer(serializers.ModelSerializer):
        class Meta:
            model = Order
            fields = ["status"]

        def validate_status(self, value):
            valid = [choice[0] for choice in Order.Status.choices]
            if value not in valid:
                raise serializers.ValidationError(
                    f"'{value}' is not a valid status. Choose from: {valid}"
                )
            return value

This only checks the submitted value is SOME valid `Order.Status`
choice — an admin (or a scripted/buggy client hitting this API) can
currently move an order from `DELIVERED` back to `PENDING`, or from
`CANCELLED` directly to `SHIPPED`, or any other nonsensical transition,
with zero guardrails. There's also a SEPARATE existing code path —
`OrderDetailView.patch()` in `backend/order/views.py` (the
customer-facing cancellation endpoint, built across Epics 1/3/4 with
its own atomic stock-restoration logic) — that enforces its OWN narrow
rule (can only cancel if not yet shipped/delivered) completely
independently of this admin endpoint. These two order-mutating code
paths currently have no shared logic and could easily drift out of
sync.

TASK
Add a real state-machine validation to admin status transitions, and
extract the stock-restoration-on-cancellation logic (currently living
only inside `OrderDetailView.patch()`) into a shared service function
that BOTH the customer cancellation endpoint and this admin endpoint
call, so cancelling an order behaves identically and safely regardless
of which path triggers it.

REQUIREMENTS
- Define the valid state machine as a simple mapping, e.g. in a new
  `backend/order/services/state_machine.py`:
  ```python
  from order.models import Order

  VALID_TRANSITIONS = {
      Order.Status.PENDING: {Order.Status.PROCESSING, Order.Status.CANCELLED},
      Order.Status.PROCESSING: {Order.Status.SHIPPED, Order.Status.CANCELLED},
      Order.Status.SHIPPED: {Order.Status.DELIVERED},
      Order.Status.DELIVERED: set(),   # terminal
      Order.Status.CANCELLED: set(),   # terminal
  }

  def is_valid_transition(current: str, new: str) -> bool:
      if current == new:
          return True  # no-op re-submission of the same status is harmless
      return new in VALID_TRANSITIONS.get(current, set())
  ```
  — review this transition map against how this codebase's payment
  flow (Epic 6) actually sets order status: recall that
  `PaymentCallbackView`'s success path moves `PENDING → PROCESSING`
  directly (skipping nothing), and its failure path moves
  `PENDING → CANCELLED`. Also recall `SHIPPED`/`DELIVERED` are set by
  Epic 7's shipment-tracking polling task. Confirm the map above is
  consistent with every place in the codebase that currently sets
  `Order.status` anywhere (search `\.status = Order\.Status\.` and
  `status=Order.Status\.` across the whole backend to find every call
  site from every prior epic) — adjust the map if you find a
  legitimate transition this initial map doesn't account for, rather
  than leaving a real code path broken by an overly strict state
  machine.
- Update `AdminOrderStatusSerializer.validate_status()` to also check
  the transition is valid given the order's CURRENT status (accessible
  via `self.instance.status` inside a serializer's field-level
  validator, since `self.instance` is the existing `Order` being
  updated during a `partial_update`):
  ```python
  def validate_status(self, value):
      valid = [choice[0] for choice in Order.Status.choices]
      if value not in valid:
          raise serializers.ValidationError(
              f"'{value}' is not a valid status. Choose from: {valid}"
          )
      if self.instance and not is_valid_transition(self.instance.status, value):
          raise serializers.ValidationError(
              f"Cannot change status from '{self.instance.status}' to '{value}'."
          )
      return value
  ```
  Import `is_valid_transition` from `order.services.state_machine` at
  the top of dashboard/serializers.py.
- Extract the stock-restoration-on-cancellation logic currently inline
  in `OrderDetailView.patch()` into a shared function, e.g.
  `order/services/cancellation.py`:
  ```python
  def cancel_order(order, actor=None):
      """
      Cancel `order`: set status to CANCELLED and restore stock for
      every item, atomically. Raises ValueError if the order is in a
      terminal state that cannot be cancelled.
      """
      from django.db import transaction
      from shop.models import ProductVariant, StockMovement
      from order.models import Order

      if not is_valid_transition(order.status, Order.Status.CANCELLED):
          raise ValueError(f"Order in status '{order.status}' cannot be cancelled.")

      with transaction.atomic():
          order.status = Order.Status.CANCELLED
          order.save(update_fields=["status"])
          for item in order.items.select_related("variant").all():
              if item.variant is None:
                  continue
              locked_variant = ProductVariant.objects.select_for_update().get(pk=item.variant.pk)
              locked_variant.stock += item.quantity
              locked_variant.save(update_fields=["stock"])
              StockMovement.objects.create(
                  variant=locked_variant,
                  reason=StockMovement.Reason.CANCELLATION,
                  quantity_delta=item.quantity,
                  stock_after=locked_variant.stock,
                  actor=actor,
                  related_order=order,
                  note="Cancelled via admin" if actor else "Cancelled by customer",
              )
  ```
  (this is the exact same logic already built and tested across Epic 1
  Task 1.1.1.4, Epic 3 Task 3.1.1.5, and Epic 4 Task 4.1.1.3 — you are
  MOVING it out of `OrderDetailView.patch()`, not rewriting its
  behavior; the goal is one authoritative implementation, not a
  rewrite).
- Update `OrderDetailView.patch()` in order/views.py to call
  `cancel_order(order, actor=None)` instead of its inline logic,
  catching `ValueError` and translating it to the existing 400 response
  the endpoint already returns for a non-cancellable order.
- Update `AdminOrderViewSet` (dashboard/views.py): when an admin
  submits `status=cancelled` via `partial_update`, route through the
  SAME `cancel_order(order, actor=request.user)` function rather than
  letting the plain `ModelSerializer.save()` just set the field
  directly without restoring stock — this means overriding
  `AdminOrderViewSet.partial_update()` (or `perform_update()`) to
  special-case the cancellation transition:
  ```python
  def perform_update(self, serializer):
      new_status = serializer.validated_data.get("status")
      order = self.get_object()
      if new_status == Order.Status.CANCELLED and order.status != Order.Status.CANCELLED:
          cancel_order(order, actor=self.request.user)
      else:
          serializer.save()
  ```
  (adjust exact wiring as needed given DRF's `ModelViewSet` update flow
  — the key requirement is: an admin cancelling an order through THIS
  endpoint must trigger the identical atomic stock-restoration
  behavior as the customer-facing cancellation path, not a bare status
  flip with no stock consequence).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/test_views.py:
1. An admin attempting an invalid transition (e.g. `DELIVERED` →
   `PENDING`) via `PATCH /dashboard/admin/orders/{id}/` gets a 400 with
   a clear message; the order's status is unchanged.
2. An admin performing a VALID transition (e.g. `PROCESSING` →
   `SHIPPED`) succeeds.
3. An admin cancelling a `PENDING`/`PROCESSING` order via this endpoint
   correctly restores stock for every item (assert exact before/after
   variant stock values) and creates `StockMovement` rows with `actor`
   set to the admin user.
4. Re-run the FULL existing order test suite (the customer-facing
   cancellation tests from Epic 1/3/4) after the extraction refactor
   and confirm every existing assertion still passes unchanged —
   proving the extraction was behavior-neutral for the pre-existing
   customer path.
5. Submitting the SAME status the order is already in (a no-op
   resubmission) succeeds without error and without triggering
   cancellation logic even if that status happens to be `CANCELLED`
   (don't double-restore stock for an order that's already cancelled —
   confirm the `order.status != Order.Status.CANCELLED` guard in
   `perform_update` correctly prevents this).
```

---

#### Task 8.1.1.2 — Order status change triggers notification

```
You are working in backend/order/ (a new signal or explicit call site).
Assume Task 8.1.1.1 is already merged.

CONTEXT
There is no notification system in this codebase yet — Epic 16
("Notifications") in the project's master backlog is the epic
responsible for building the actual unified `notify(user, event,
context)` service, SMS/email dispatch, and templates. This task's job
is narrower: make sure order status changes are wired to CALL that
system once it exists, via a clean, single hook point — not to build
notification infrastructure prematurely or duplicate work Epic 16 will
do properly.

TASK
Add a single, well-placed hook point that fires whenever an order's
status changes, with a clear extension point for Epic 16 to plug real
notification dispatch into later, plus a placeholder/logging
implementation for now so the hook's behavior is at least verifiable
before Epic 16 lands.

REQUIREMENTS
- Add a Django signal-based hook, mirroring the exact pattern already
  established in Epic 4 Task 4.1.2.3 (the `StockMovement` post_save
  signal that later hooks into stock-alert notifications) — use
  `post_save` on `Order` itself, checking whether `status` actually
  changed (a plain `post_save` fires on EVERY save, including saves
  that don't touch `status` at all, so you need to detect an actual
  status CHANGE, not just any save):
  ```python
  # order/signals.py
  from django.db.models.signals import pre_save, post_save
  from django.dispatch import receiver
  from .models import Order

  @receiver(pre_save, sender=Order)
  def _capture_previous_status(sender, instance, **kwargs):
      if instance.pk:
          try:
              instance._previous_status = Order.objects.only("status").get(pk=instance.pk).status
          except Order.DoesNotExist:
              instance._previous_status = None
      else:
          instance._previous_status = None

  @receiver(post_save, sender=Order)
  def _handle_status_change(sender, instance, created, **kwargs):
      previous = getattr(instance, "_previous_status", None)
      if created or (previous is not None and previous != instance.status):
          order_status_changed.send(sender=Order, order=instance, previous_status=previous)
  ```
  — this `pre_save`-captures-then-`post_save`-compares pattern is the
  standard, reliable way to detect "did this specific field actually
  change" in Django (a plain `post_save` alone can't tell you the OLD
  value, since by the time it fires the new value is already saved).
  Define a custom Django signal
  `order_status_changed = django.dispatch.Signal()` (with
  `providing_args` no longer needed in modern Django — just document
  the expected kwargs in a comment) near the top of order/signals.py.
- Connect `order/signals.py` in `order/apps.py`'s `ready()` method,
  mirroring the exact wiring pattern established in Epic 4 Task
  4.1.2.3 for `shop/signals.py`.
- Add a receiver for `order_status_changed` that, FOR NOW (pending Epic
  16), just logs the event clearly via Python's `logging` module:
  ```python
  import logging
  logger = logging.getLogger("order.notifications")

  @receiver(order_status_changed)
  def _log_status_change_for_now(sender, order, previous_status, **kwargs):
      logger.info(
          "Order %s status changed: %s -> %s (TODO: Epic 16 — dispatch real customer notification here)",
          order.order_number, previous_status, order.status,
      )
  ```
  This is DELIBERATELY a placeholder, not a real notification — the
  actual customer-facing SMS/email dispatch is explicitly Epic 16's
  job; this task's contribution is ensuring the HOOK exists, fires
  correctly and exactly once per real status change, and is trivially
  easy for Epic 16 to attach real dispatch logic to later (by adding
  another receiver to the same `order_status_changed` signal, no
  changes needed to the order-mutating code paths themselves).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/order/tests/:
1. Creating a new Order fires `order_status_changed` exactly once (the
   `created` branch).
2. Changing an existing order's status (via either the admin path from
   Task 8.1.1.1 or the customer cancellation path) fires the signal
   exactly once with the correct `previous_status`/new `order.status`
   values.
3. Saving an order WITHOUT changing its status (e.g. updating `notes`
   only) does NOT fire the signal — this is the most important test in
   this task, proving the change-detection logic actually works and
   isn't just firing on every save.
4. Two back-to-back status changes in the same test each fire the
   signal exactly once with the correct respective previous/new values
   (not stale/cached from the first change).
```

---

#### Task 8.1.1.3 — Admin order search/filter (by number, customer, status, date range)

```
You are working in backend/dashboard/filters.py, views.py. Assume
Task 8.1.1.1 is already merged.

CONTEXT — READ THIS BEFORE WRITING ANY CODE, MOST OF THIS ALREADY EXISTS
Confirmed directly from the repo: `AdminOrderFilter` already provides
`status`, `payment_method`, `date_from`/`date_to` (both correctly using
`__date__gte`/`__date__lte` lookups against `created_at`), and
`min_total`/`max_total`. `AdminOrderViewSet` already wires this filter
in via `DjangoFilterBackend`, AND separately has
`search_fields = ["order_number", "email", "first_name", "last_name", "phone"]`
via `filters.SearchFilter`, which already covers free-text search by
order number, customer email, first/last name, and phone. This task is
NOT "build order search from scratch" — it's confirming this existing
implementation is genuinely complete against the backlog's stated
requirement ("by number, customer, status, date range" — all four are
already covered) and filling in any small, genuinely missing pieces.

TASK
Audit the existing `AdminOrderFilter`/`AdminOrderViewSet` search
capability against real admin usage needs, and add the specific small
gaps found (rather than re-implementing what already works).

REQUIREMENTS
- Confirm via manual testing (`GET /dashboard/admin/orders/?search=...`,
  `?status=...`, `?date_from=...&date_to=...`, `?min_total=...`) that
  every existing filter/search parameter genuinely works as documented
  — this is a real verification step, not a rubber stamp; actually run
  these queries against seeded test data and confirm correct results
  before assuming everything already works perfectly.
- Identify and add genuinely missing capability. Two concrete gaps
  worth checking and likely adding:
  1. Filtering by a SPECIFIC customer, not just free-text search — an
     admin viewing a customer's account might want a direct
     "all orders for user ID X" filter distinct from typing their name
     into search. Add:
     `user_id = django_filters.NumberFilter(field_name="user_id")`
     to `AdminOrderFilter`.
  2. Filtering by shipping destination — now that Epic 5/7 added real
     Iranian province data to checkout, an admin might reasonably want
     to filter orders by destination province (e.g. for regional
     fulfillment planning or carrier-specific reporting). Add:
     `shipping_province = django_filters.CharFilter(field_name="shipping_state")`
     (note: `Order.shipping_state` is the actual DB field name per the
     original model — it was NOT renamed as part of Epic 5 Task
     5.2.1.4, which deliberately left `Order`'s own snapshot fields as
     plain unconstrained CharFields; confirm this is still accurate by
     checking the current `Order` model before writing this filter).
  Do NOT add filters/search capability the backlog didn't ask for and
  that isn't a clear, small, obviously-needed gap — resist the urge to
  build a large new admin search feature set here; this task is
  explicitly scoped to confirming what exists and closing small,
  concrete gaps only.
- Add both new filter fields to `AdminOrderFilter.Meta.fields`.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/test_filters.py (create this file
if it doesn't exist, checking for precedent first) confirming:
1. Each EXISTING filter (`status`, `payment_method`, `date_from`,
   `date_to`, `min_total`, `max_total`) and the existing free-text
   `search` parameter genuinely work correctly against seeded test
   orders — i.e. write the verification tests that arguably should
   have existed already but may not have, closing that test-coverage
   gap as part of this task even though the underlying feature code
   itself isn't new.
2. The two NEW filters (`user_id`, `shipping_province`) work correctly.
3. Combining multiple filters simultaneously (e.g.
   `?status=processing&date_from=2026-01-01&user_id=5`) correctly ANDs
   them together.
```

---

#### Task 8.1.1.4 — Order export (CSV) for accounting

```
You are working in backend/dashboard/views.py, urls.py. Assume Task
8.1.1.3 is already merged (the filtering this export should respect).

CONTEXT
There's no way for an admin to get order data OUT of this platform for
accounting/bookkeeping purposes (reconciling against ZarinPal/Zibal/
IDPay settlement reports, tax filing, etc.) other than manually reading
the admin UI — a genuine, common operational need for any real business
running this platform.

TASK
Add a CSV export endpoint that respects the SAME filters as the
existing `AdminOrderViewSet` list endpoint (Task 8.1.1.3), so an admin
can filter down to exactly the orders they want (e.g. a specific date
range) and export precisely that filtered set, not always the entire
order history.

REQUIREMENTS
- Add a custom `@action` on `AdminOrderViewSet` (DRF's standard
  mechanism for adding a non-CRUD endpoint to a ViewSet, reusing all
  its existing filter/permission wiring automatically):
  ```python
  from django.http import StreamingHttpResponse
  import csv
  from io import StringIO

  class Echo:
      """Write-target that just returns what it's given, for streaming CSV."""
      def write(self, value):
          return value


  class AdminOrderViewSet(viewsets.ModelViewSet):
      ...

      @action(detail=False, methods=["get"])
      def export_csv(self, request):
          queryset = self.filter_queryset(self.get_queryset())  # respects all existing filters/search
          fields = [
              "order_number", "created_at", "status", "first_name", "last_name",
              "email", "phone", "shipping_city", "shipping_state", "shipping_country",
              "payment_method", "subtotal", "shipping_cost", "tax", "discount", "total",
          ]

          def row_generator():
              writer = csv.writer(Echo())
              yield writer.writerow(fields)
              for order in queryset.iterator(chunk_size=500):
                  yield writer.writerow([getattr(order, f) for f in fields])

          response = StreamingHttpResponse(row_generator(), content_type="text/csv")
          response["Content-Disposition"] = 'attachment; filename="orders_export.csv"'
          return response
  ```
  Using `StreamingHttpResponse` + `queryset.iterator()` (rather than
  loading the entire filtered order set into memory as a Python list
  before generating the CSV) matters here specifically because an
  unbounded date-range export on a platform with real order volume
  could be tens of thousands of rows — building the whole response in
  memory first is a real scalability risk worth avoiding from the
  start, not a premature optimization.
  Import `action` from `rest_framework.decorators` at the top of
  dashboard/views.py if not already imported.
- The endpoint is reachable at
  `GET /dashboard/admin/orders/export_csv/` (DRF's router
  automatically derives this URL from the `@action` method name) and
  automatically respects any `?status=`, `?date_from=`, etc. query
  params exactly like the list endpoint, since it reuses
  `self.filter_queryset(self.get_queryset())`.
- Confirm `permission_classes = [IsAdminOrSuperuser]` (already set at
  the ViewSet level) correctly applies to this action too (DRF applies
  ViewSet-level `permission_classes` to all actions, including custom
  `@action`s, by default — no extra permission wiring should be needed,
  but verify this with a test rather than assuming).
- Add filename date-stamping for usability: change the static
  `orders_export.csv` filename to include the current date, e.g.
  `f'attachment; filename="orders_export_{timezone.now().strftime("%Y-%m-%d")}.csv"'`
  (import `timezone` from `django.utils`).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/test_views.py:
1. `GET /dashboard/admin/orders/export_csv/` returns a
   `text/csv` response with the correct header row and one row per
   existing order, with correct field values.
2. Adding filter query params to the export request (e.g.
   `?status=delivered`) results in ONLY matching orders appearing in
   the CSV output — proving filter reuse works correctly.
3. A non-admin user gets 403/401 attempting to access this endpoint.
4. The response has the expected `Content-Disposition` header
   triggering a browser download rather than inline display.
```

---

#### Task 8.1.1.5 — Invoice PDF generation per order

```
You are working in backend/order/ or backend/dashboard/ (new PDF
generation service), backend/requirements.txt. Assume Task 8.1.1.1 is
merged.

CONTEXT
No PDF generation library exists anywhere in this project's
`requirements.txt` (confirmed directly — grepped for `reportlab`,
`weasyprint`, `xhtml2pdf`, `fpdf`, `pdfkit`, none present) and there is
no invoice-generation capability at all. Customers/admins currently
have no way to get a formal, printable/downloadable invoice for an
order.

TASK
Add PDF invoice generation for a given order, downloadable by both the
order's owning customer and admin staff.

REQUIREMENTS
- Add a PDF generation library to backend/requirements.txt. Choose
  between `weasyprint` (renders HTML/CSS to PDF — generally the most
  pleasant to work with for a nicely-formatted document since you write
  a normal HTML template rather than imperative drawing calls, but has
  real system-level dependencies like Pango/Cairo that need to be
  present in the Docker image) and `reportlab` (pure-Python, no system
  dependencies, but a more verbose, imperative drawing API for layout).
  Given this project already has a Docker-based deployment (confirmed
  from prior epics' grounding), `weasyprint`'s system-dependency
  requirement is manageable via the Dockerfile — prefer `weasyprint`
  for the significantly nicer authoring experience (an HTML/CSS
  template is far easier to get looking professional and to later
  adapt for Persian/RTL formatting under Epic 14, versus hand-coding
  PDF layout coordinates in reportlab) UNLESS you find a specific
  reason this project's deployment can't accommodate the system
  dependencies, in which case fall back to `reportlab` and note why.
  Pin an exact version matching this project's existing
  fully-pinned-dependency convention.
  If choosing `weasyprint`, check/update the backend Dockerfile to
  install the required system packages (Pango, cairo, gdk-pixbuf —
  check WeasyPrint's current documented system dependency list for the
  exact package names on whatever base OS image this project's
  Dockerfile uses) — a Python-level pip install alone is NOT sufficient
  for WeasyPrint to actually function.
- Create an HTML invoice template (e.g.
  backend/templates/order/invoice.html, using Django's template
  engine, which — confirm — should already be configured since this is
  a Django project with an admin UI relying on Django's template
  system) containing: company/store name and details (placeholder for
  now — a real business would want this configurable, but hardcoding a
  clearly-labeled placeholder is fine for this task, with a
  `# TODO: make store details configurable via settings/admin` note),
  order number, order date, customer name/shipping address, an itemized
  table (product name, variant details if applicable, quantity, unit
  price, line subtotal), and the financial summary (subtotal, shipping
  cost, tax, discount, total) — pulling all of this from the `Order`
  and its related `OrderItem`s, using the SAME frozen-snapshot fields
  already established across prior epics (never re-derive current
  product/variant data — an invoice must reflect exactly what was
  actually ordered/charged at the time, which is precisely what the
  snapshot fields exist for).
- Add a service function, e.g. backend/order/services/invoice.py:
  ```python
  from django.template.loader import render_to_string
  import weasyprint

  def generate_invoice_pdf(order) -> bytes:
      html_string = render_to_string("order/invoice.html", {"order": order})
      return weasyprint.HTML(string=html_string).write_pdf()
  ```
- Add an endpoint: `GET /api/orders/{id}/invoice/` in
  backend/order/views.py (a new view, or a custom `@action` if using
  a ViewSet — check whether `OrderListCreateView`/`OrderDetailView`
  are plain `generics` views, per earlier grounding, not a ViewSet, so
  a separate dedicated `APIView` is more consistent with the existing
  style than trying to bolt a DRF-viewset-style `@action` onto a
  `generics`-based view):
  ```python
  class OrderInvoiceView(APIView):
      permission_classes = [IsAuthenticated]

      def get(self, request, pk):
          order = get_object_or_404(Order, pk=pk)
          if order.user != request.user and not (
              request.user.is_staff or request.user.type in (UserType.ADMIN, UserType.SUPERUSER)
          ):
              return Response(status=status.HTTP_403_FORBIDDEN)
          pdf_bytes = generate_invoice_pdf(order)
          response = HttpResponse(pdf_bytes, content_type="application/pdf")
          response["Content-Disposition"] = f'attachment; filename="invoice_{order.order_number}.pdf"'
          return response
  ```
  Note the permission check explicitly allows EITHER the order's owner
  OR staff/admin — an invoice is customer-facing (they need their own
  receipt) but admins also legitimately need access to any customer's
  invoice for support purposes; import `UserType` from
  `accounts.models` at the top of order/views.py.
- Register the URL in backend/order/urls.py:
  `path("<int:pk>/invoice/", OrderInvoiceView.as_view(), name="order-invoice"),`
- Add a "Download Invoice" link/button on the frontend order-detail
  page (Account.jsx or wherever order details are already rendered),
  pointing at this endpoint — since this returns a raw PDF binary
  response rather than JSON, this should be a plain `<a href="...">`
  link (with the auth token handled correctly — check how this
  project's axios instance attaches the Authorization header, and
  confirm whether a plain anchor-tag download would even carry that
  header at all; if not, since this is an authenticated endpoint, you
  likely need to fetch the PDF via axios with `responseType: 'blob'`
  and then trigger a client-side download via a generated object URL,
  rather than a naive `<a href="/api/orders/1/invoice/">`, which
  wouldn't include the auth header and would 401) rather than a JSON
  API call.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/order/tests/:
1. The order's owning user can successfully download a PDF
   (`content_type == "application/pdf"`, non-empty byte content) for
   their own order.
2. A staff/admin user can download ANY customer's invoice.
3. A different, non-staff customer attempting to access someone else's
   order invoice gets 403.
4. `generate_invoice_pdf()` correctly reflects the order's FROZEN
   snapshot data (product name, price at time of order) even if the
   underlying `Product`/`ProductVariant` has since changed name/price —
   construct a test where you deliberately change the product's current
   price AFTER the order was placed, generate the invoice, and confirm
   (by parsing the PDF's extractable text, e.g. via `pdfplumber` or
   similar test-only PDF-text-extraction tool — add such a library as a
   TEST-only dependency if none exists yet) that the invoice shows the
   ORIGINAL order-time price, not the current one.
Manually verify the generated PDF actually renders and looks reasonable
(open one in a real PDF viewer) — an automated text-extraction test
proves the DATA is correct but doesn't catch a badly broken visual
layout, so a manual visual check is a genuinely necessary complement
here, not redundant with the automated test.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 8.1.1.1 | Admin order status transition with state-machine validation | ☐ |
| 8.1.1.2 | Order status change notification hook | ☐ |
| 8.1.1.3 | Admin order search/filter (audit + close gaps) | ☐ |
| 8.1.1.4 | Order export (CSV) for accounting | ☐ |
| 8.1.1.5 | Invoice PDF generation per order | ☐ |

Once Epic 8 is fully merged, the next epic to generate prompts for is
**Epic 9 — Coupons & Promotions**, which finally builds the real
`Coupon` model this codebase has been missing ever since Epic 1 Task
1.2.1.1 had to strip out the insecure client-controlled discount field
— giving `order/services/pricing.py`'s `calculate_order_totals()` a
real, server-validated discount source instead of the hardcoded `0` it
has used since that fix.
