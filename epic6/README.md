# Epic 6 — Payments (Iranian Gateways) — AI Coding Prompts

Repo: `tablogenix-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next. This epic closes the single biggest "this isn't a real store yet" gap identified in the original architecture review: there is currently **zero real payment processing** anywhere in this codebase — checkout collects a fake card number, stores only its last 4 digits, and creates the order immediately with no actual money changing hands.

**Assumed prerequisites:** Epics 1–5 are fully merged (atomic, variant-locked, price/stock-revalidated checkout with guest cart and address-book support). **Important grounded fact confirmed directly from `backend/requirements.txt`:** the `requests` library is not installed anywhere in this project — Task 6.1.1.3 adds it, and it is a hard prerequisite for every gateway-calling task in this epic.

**A real, load-bearing bug this epic must fix, found in `order/serializers.py`:** `OrderCreateSerializer.create()` currently sets `status=Order.Status.PROCESSING` **unconditionally at order creation time**, before any payment has actually been collected. Once real payment gateways exist, an order must not be `PROCESSING` (implying it's being fulfilled) until payment is actually confirmed — Task 6.2.1.2 below fixes this by creating orders as `PENDING` and only advancing them to `PROCESSING` on confirmed payment.

**A note on external API specifics:** ZarinPal/Zibal/IDPay's exact endpoint URLs, request/response field names, and API versions can change over time and may have shifted since this document was written. Every task involving a specific gateway explicitly instructs the implementing agent to verify current, authoritative API documentation before hardcoding endpoint URLs or payload shapes — treat any endpoint URLs mentioned in these prompts as a reasonable starting point to verify, not gospel.

---

## Phase 6.1 — Payment Foundation

### Feature 6.1.1 — Payment Domain Model

---

#### Task 6.1.1.1 — Create `payments` Django app

```
You are working in backend/. Assume Epics 1–5 are fully merged.

CONTEXT
There is no `payments` app in this codebase at all — `INSTALLED_APPS`
in backend/core/settings/base.py currently lists:

    "accounts.apps.AccountsConfig",
    "contact.apps.ContactConfig",
    "shop.apps.ShopConfig",
    "cart.apps.CartConfig",
    "order.apps.OrderConfig",
    "dashboard.apps.DashboardConfig",
    "chat.apps.ChatConfig",
    "blog.apps.BlogConfig",

and backend/core/urls.py routes each app under its own `api/` prefix
(e.g. `path("api/orders/", include("order.urls"))`). Every payment task
in this epic needs a home.

TASK
Scaffold a new `payments` Django app, registered and routed exactly
like every other app in this project, with no functional code yet
(that starts in Task 6.1.1.2 onward).

REQUIREMENTS
- Run `python manage.py startapp payments` from within backend/ (or
  manually create the equivalent file structure — `__init__.py`,
  `apps.py`, `models.py`, `admin.py`, `views.py`, `urls.py`,
  `migrations/__init__.py` — matching the exact structure of the other
  existing apps in this project, e.g. `cart/`).
- In `payments/apps.py`, name the config class `PaymentsConfig`
  matching the naming convention of every other app
  (`CartConfig`, `OrderConfig`, etc.).
- Add `"payments.apps.PaymentsConfig"` to `INSTALLED_APPS` in
  backend/core/settings/base.py, placed after `"order.apps.OrderConfig"`
  in the list (payments logically depends on orders existing first,
  matching this project's existing ordering convention of listing apps
  roughly in dependency order).
- Create a minimal `payments/urls.py`:
  ```python
  from django.urls import path

  app_name = "payments"

  urlpatterns = []
  ```
  (empty for now — populated starting in Task 6.2.1.2).
- Register it in backend/core/urls.py:
  `path("api/payments/", include("payments.urls")),`
  — add this line in the same "# Apps" block as the other app includes,
  positioned after the `order.urls` include for the same
  dependency-ordering reason as above.
- Create an empty `payments/tests/__init__.py` package (rather than a
  single `tests.py` file) from the start, matching the package-based
  test-organization convention already established across prior epics'
  work in other apps (e.g. Epic 1 Task 1.1.1.5's restructuring of
  `order/tests.py` into `order/tests/`).

ACCEPTANCE CRITERIA / TESTS
- `python manage.py check` runs with no errors after the app is
  registered.
- `python manage.py showmigrations payments` shows the app is
  recognized (even with zero migrations yet, since there are no models
  defined in this task).
- A trivial smoke test in `payments/tests/test_app.py` confirming the
  app is installed: `assert "payments" in [a.split(".")[0] for a in django.apps.apps.app_configs]`
  or equivalent — this is a minimal, almost ceremonial test, but it
  locks in that the app-registration wiring itself doesn't silently
  break in a future refactor.
```

---

#### Task 6.1.1.2 — Create `PaymentTransaction` model

```
You are working in backend/payments/models.py. Assume Task 6.1.1.1 is
already merged.

CONTEXT
There is currently no record anywhere in the database of a payment
attempt, its gateway, its status, or its outcome — `Order.payment_method`
and `Order.card_last_four` are the ONLY payment-related fields that
exist today, and they describe intent/UI state, not an actual
transaction record. Every real gateway integration in this epic needs
somewhere to record "we asked gateway X to charge amount Y for order Z,
here's the reference code, here's whether it succeeded."

TASK
Create a `PaymentTransaction` model recording every payment attempt
against an order, across any gateway.

REQUIREMENTS
- Add to backend/payments/models.py:
  ```python
  from django.db import models


  class PaymentTransaction(models.Model):
      class Gateway(models.TextChoices):
          ZARINPAL = "zarinpal", "ZarinPal"
          ZIBAL = "zibal", "Zibal"
          IDPAY = "idpay", "IDPay"

      class Status(models.TextChoices):
          PENDING = "pending", "Pending"
          SUCCESS = "success", "Success"
          FAILED = "failed", "Failed"

      order = models.ForeignKey(
          "order.Order", on_delete=models.CASCADE, related_name="payment_transactions"
      )
      gateway = models.CharField(max_length=20, choices=Gateway.choices)
      status = models.CharField(
          max_length=20, choices=Status.choices, default=Status.PENDING, db_index=True
      )
      authority = models.CharField(
          max_length=100, blank=True, db_index=True,
          help_text="Gateway-issued reference code for this transaction (ZarinPal calls this 'Authority').",
      )
      ref_id = models.CharField(
          max_length=100, blank=True,
          help_text="Gateway-issued final transaction reference, populated on success.",
      )
      amount = models.DecimalField(max_digits=12, decimal_places=2)
      raw_callback_payload = models.JSONField(
          null=True, blank=True,
          help_text="Full raw payload received from the gateway's callback, stored for audit/debugging.",
      )
      created_at = models.DateTimeField(auto_now_add=True)
      updated_at = models.DateTimeField(auto_now=True)

      class Meta:
          ordering = ["-created_at"]
          indexes = [models.Index(fields=["gateway", "authority"])]

      def __str__(self):
          return f"{self.gateway} — {self.order.order_number} — {self.status}"
  ```
  Note `order` uses a STRING cross-app FK reference (`"order.Order"`),
  matching the exact pattern already established for cross-app FKs
  elsewhere in this codebase (`CartItem.product = models.ForeignKey("shop.Product", ...)`,
  `StockMovement.related_order = models.ForeignKey("order.Order", ...)` from Epic 4).
- Generate the migration.
- Register `PaymentTransaction` in backend/payments/admin.py as a
  MOSTLY read-only admin view (this is a financial audit record — an
  admin should be able to LOOK at transaction history and, per Task
  6.4.2.2 later in this epic, make a deliberate manual status override
  through a dedicated action, but should never be able to freely
  hand-edit arbitrary fields like `amount` or `ref_id` through a plain
  admin form, which would corrupt financial record-keeping):
  ```python
  from django.contrib import admin
  from .models import PaymentTransaction


  @admin.register(PaymentTransaction)
  class PaymentTransactionAdmin(admin.ModelAdmin):
      list_display = ("id", "order", "gateway", "status", "authority", "amount", "created_at")
      list_filter = ("gateway", "status")
      search_fields = ("order__order_number", "authority", "ref_id")
      readonly_fields = [f.name for f in PaymentTransaction._meta.fields]
      ordering = ("-created_at",)

      def has_add_permission(self, request):
          return False

      def has_delete_permission(self, request, obj=None):
          return False
  ```

ACCEPTANCE CRITERIA / TESTS
Add a model test in backend/payments/tests/test_models.py confirming a
`PaymentTransaction` can be created linked to an `Order`, defaults to
`status=PENDING`, and `str()` produces a readable representation.
```

---

#### Task 6.1.1.3 — Add `requests` to `requirements.txt`

```
You are working in backend/requirements.txt.

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
The `requests` library is not present anywhere in `requirements.txt`
today (grep the file — it genuinely isn't there). Every gateway
integration in this epic (ZarinPal, and later Zibal/IDPay) needs to
make outbound HTTP calls to the gateway's API, and `requests` is the
standard, expected library for this in a Django codebase — without it,
none of Phase 6.2 or 6.3's tasks can function.

TASK
Add `requests` to backend/requirements.txt and install it in the
development environment.

REQUIREMENTS
- Add a `requests` line to backend/requirements.txt. Pin it to a
  specific, current stable version (check what the latest stable
  `requests` release is at the time you do this task — don't blindly
  copy an old pin from memory; the rest of this project's
  requirements.txt is fully pinned with exact versions, e.g.
  `Django==5.2.11`, so match that convention with an exact `==` pin,
  not a loose `>=`).
- Place it alphabetically in the file, matching the existing
  alphabetical ordering convention already visible in
  requirements.txt (e.g. it would land between `redis==...` and
  `rpds-py==...` or wherever it alphabetically falls — check the
  existing file's exact ordering and insert correctly, don't just
  append to the end).
- Run `pip install -r backend/requirements.txt` (or
  `pip install requests==<version> --break-system-packages` per this
  project's documented pip usage convention, if working directly in
  this sandboxed environment) to confirm the pin actually resolves and
  installs cleanly with no conflicts against the rest of the pinned
  dependency set.
- If this project has a Docker-based dev setup (check
  docker-compose.yml / Dockerfile for how the backend image installs
  dependencies), confirm whether the image needs to be rebuilt for the
  new dependency to be picked up, and note that in your task summary
  if relevant (don't necessarily rebuild the image as part of this
  task if that's a larger CI/deploy concern, but flag it).

ACCEPTANCE CRITERIA / TESTS
No code changes to test in this task — the acceptance criterion is
simply: `python -c "import requests; print(requests.__version__)"`
succeeds inside the project's backend environment, and
`pip check` (or equivalent dependency-conflict checker) reports no
conflicts introduced by the new pin.
```

---

#### Task 6.1.1.4 — Define `PaymentGateway` abstract interface

```
You are working in backend/payments/gateways/ (new package). Assume
Tasks 6.1.1.2 and 6.1.1.3 are already merged.

CONTEXT
Three different gateways (ZarinPal, Zibal, IDPay) will each need their
own concrete implementation, but the rest of the codebase (the
initiate/callback views built in Phase 6.2) should be able to work with
"whichever gateway is configured" without knowing gateway-specific
details — this mirrors the exact abstraction pattern already
established for SMS providers in Epic 2 Task 2.2.1.1
(`SMSProvider` ABC + settings-driven factory function) — follow that
SAME pattern here for consistency across the codebase.

TASK
Define an abstract `PaymentGateway` base class with `request_payment()`
and `verify_payment()` methods, plus a settings-driven factory function
mirroring `get_sms_provider()` from Epic 2.

REQUIREMENTS
- Create `backend/payments/gateways/__init__.py` and
  `backend/payments/gateways/base.py`:
  ```python
  from abc import ABC, abstractmethod
  from dataclasses import dataclass
  from decimal import Decimal


  @dataclass
  class PaymentRequestResult:
      success: bool
      authority: str = ""
      redirect_url: str = ""
      error_message: str = ""


  @dataclass
  class PaymentVerifyResult:
      success: bool
      ref_id: str = ""
      error_message: str = ""
      raw_response: dict | None = None


  class PaymentGateway(ABC):
      @abstractmethod
      def request_payment(
          self, amount: Decimal, callback_url: str, description: str = ""
      ) -> PaymentRequestResult:
          """Ask the gateway to initiate a payment. Returns an authority
          code and a URL to redirect the customer to."""
          raise NotImplementedError

      @abstractmethod
      def verify_payment(self, authority: str, amount: Decimal) -> PaymentVerifyResult:
          """Confirm a payment was actually completed successfully after
          the customer returns from the gateway."""
          raise NotImplementedError
  ```
  — using small `@dataclass` result types (rather than plain dicts or
  tuples) makes the interface's contract explicit and self-documenting
  for whoever implements a concrete gateway next; import `Decimal` from
  `decimal` at the top.
- Add a factory function in `payments/gateways/__init__.py`:
  ```python
  from django.conf import settings
  from django.utils.module_loading import import_string
  from django.core.exceptions import ImproperlyConfigured
  from .base import PaymentGateway


  def get_payment_gateway(name: str | None = None) -> PaymentGateway:
      """
      Return an instance of the configured payment gateway. If `name` is
      not given, uses settings.DEFAULT_PAYMENT_GATEWAY.
      """
      name = name or settings.DEFAULT_PAYMENT_GATEWAY
      gateway_classes = settings.PAYMENT_GATEWAY_CLASSES
      if name not in gateway_classes:
          raise ImproperlyConfigured(f"Unknown payment gateway: {name}")
      gateway_class = import_string(gateway_classes[name])
      instance = gateway_class()
      if not isinstance(instance, PaymentGateway):
          raise ImproperlyConfigured(
              f"{gateway_classes[name]} does not implement PaymentGateway"
          )
      return instance
  ```
  — this dotted-path-per-gateway-name dict approach (rather than a
  single `SMS_PROVIDER_CLASS` string like Epic 2's simpler single-
  provider case) is needed here specifically because Phase 6.3 adds
  MULTIPLE simultaneously-available gateways with admin-configurable
  selection/fallback (Task 6.3.1.3), unlike SMS which only ever needs
  one active provider at a time — don't over-simplify this to match
  Epic 2's exact shape, the requirements genuinely differ.
- Add to backend/core/settings/base.py:
  ```python
  DEFAULT_PAYMENT_GATEWAY = config("DEFAULT_PAYMENT_GATEWAY", default="zarinpal")
  PAYMENT_GATEWAY_CLASSES = {
      "zarinpal": "payments.gateways.zarinpal.ZarinPalGateway",
      # "zibal": "payments.gateways.zibal.ZibalGateway",       # added in Task 6.3.1.1
      # "idpay": "payments.gateways.idpay.IDPayGateway",       # added in Task 6.3.1.2
  }
  ```
  (leave the Zibal/IDPay entries commented out for now since those
  classes don't exist yet until Phase 6.3 — uncommenting them before
  the classes exist would break `python manage.py check` via the
  `import_string` factory failing at first actual use).
- Do NOT implement `ZarinPalGateway` in this task — that's Task 6.2.1.1.
  This task is the abstract interface and factory only.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/payments/tests/test_gateways.py:
1. `get_payment_gateway("nonexistent")` raises `ImproperlyConfigured`.
2. A minimal dummy test-only subclass of `PaymentGateway` (defined
   inline in the test file, implementing both abstract methods
   trivially) can be instantiated and passes the `isinstance` check —
   proving the ABC contract itself works correctly before any real
   gateway exists.
3. Confirm `PaymentGateway()` (attempting to instantiate the ABC
   directly, not a subclass) raises `TypeError` (standard Python ABC
   behavior when abstract methods aren't implemented — this is really
   testing that `@abstractmethod` is doing its job, not custom code,
   but worth locking in as documentation of intent).
```

---

## Phase 6.2 — ZarinPal Integration (primary gateway)

### Feature 6.2.1 — ZarinPal Flow

---

#### Task 6.2.1.1 — Implement `ZarinPalGateway.request_payment()`

```
You are working in backend/payments/gateways/zarinpal.py (new file).
Assume Task 6.1.1.4 (PaymentGateway interface) is already merged.

CONTEXT
No concrete gateway implementation exists yet — only the abstract
interface. ZarinPal is this platform's primary/default gateway per
`DEFAULT_PAYMENT_GATEWAY` from Task 6.1.1.4.

IMPORTANT — VERIFY CURRENT API DETAILS BEFORE HARDCODING
ZarinPal's API endpoint URLs and exact request/response payload shape
may have changed since this task was written. Before implementing,
verify the CURRENT, authoritative ZarinPal Payment Gateway REST API
documentation (their official merchant/developer docs) for: (a) the
exact request-payment endpoint URL for both sandbox and production,
(b) the exact required/optional request fields (merchant ID field
name, amount field name and whether amount is expected in Rial or
Toman — this matters a lot, getting this wrong either overcharges or
undercharges customers by a factor of 10), and (c) the exact response
shape (where the "Authority" code and the numeric status/error code
live in the response JSON). Do not proceed with hardcoded field names
from memory alone if you have any way to verify against current
documentation — a wrong field name here means every single payment on
this platform silently fails.

TASK
Implement `ZarinPalGateway.request_payment()`, calling ZarinPal's
payment-request API and returning a `PaymentRequestResult` per the
interface from Task 6.1.1.4.

REQUIREMENTS
- Add settings for ZarinPal credentials/mode to
  backend/core/settings/base.py:
  ```python
  ZARINPAL_MERCHANT_ID = config("ZARINPAL_MERCHANT_ID", default="")
  ZARINPAL_SANDBOX = config("ZARINPAL_SANDBOX", default=True, cast=bool)
  ```
  matching the existing `python-decouple` `config()` pattern used
  throughout this settings file.
- Implement:
  ```python
  import requests
  from decimal import Decimal
  from django.conf import settings
  from .base import PaymentGateway, PaymentRequestResult, PaymentVerifyResult


  class ZarinPalGateway(PaymentGateway):
      REQUEST_URL = (
          "https://sandbox.zarinpal.com/pg/v4/payment/request.json"
          if settings.ZARINPAL_SANDBOX
          else "https://api.zarinpal.com/pg/v4/payment/request.json"
      )
      # NOTE: verify these exact URLs/response shapes against current
      # ZarinPal documentation before relying on them — see task
      # context above.
      START_PAY_URL = (
          "https://sandbox.zarinpal.com/pg/StartPay/{authority}"
          if settings.ZARINPAL_SANDBOX
          else "https://www.zarinpal.com/pg/StartPay/{authority}"
      )

      def request_payment(self, amount, callback_url, description=""):
          # ZarinPal amounts are typically expected in Rial (IRR) as an
          # integer, not Toman and not a decimal — CONFIRM this against
          # current docs, and convert `amount` (this platform's internal
          # currency unit, decided in the project's localization work)
          # accordingly before sending.
          payload = {
              "merchant_id": settings.ZARINPAL_MERCHANT_ID,
              "amount": int(amount),  # adjust unit conversion per verified docs
              "callback_url": callback_url,
              "description": description or "Order payment",
          }
          try:
              response = requests.post(self.REQUEST_URL, json=payload, timeout=15)
              response.raise_for_status()
              data = response.json()
          except (requests.RequestException, ValueError) as exc:
              return PaymentRequestResult(success=False, error_message=str(exc))

          # Adjust the exact response-parsing keys below to match the
          # ACTUAL verified response shape from current ZarinPal docs —
          # this is illustrative, not guaranteed byte-accurate:
          result_data = data.get("data", {})
          errors = data.get("errors", {})
          if result_data and result_data.get("code") == 100:
              authority = result_data["authority"]
              return PaymentRequestResult(
                  success=True,
                  authority=authority,
                  redirect_url=self.START_PAY_URL.format(authority=authority),
              )
          return PaymentRequestResult(
              success=False,
              error_message=str(errors) or "ZarinPal payment request failed.",
          )

      def verify_payment(self, authority, amount):
          raise NotImplementedError  # implemented in Task 6.2.1.3
  ```
  Use a real `timeout` on every outbound HTTP call to the gateway (15
  seconds is a reasonable starting point, but this is a genuinely
  important detail — an outbound call with NO timeout can hang a
  request-handling worker indefinitely if the gateway is unresponsive,
  which is a real production incident risk, not a style nitpick).
- Catch `requests.RequestException` (covers connection errors,
  timeouts, and HTTP error statuses if you call `raise_for_status()`)
  AND `ValueError`/`json.JSONDecodeError` (a malformed/non-JSON
  response body) — both must degrade gracefully to a
  `PaymentRequestResult(success=False, ...)`, never let an unhandled
  exception propagate out of this method and become a raw 500 for the
  customer at checkout.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/payments/tests/test_zarinpal.py using
`responses` or `requests-mock` (check requirements.txt/dev
dependencies for whether either is already available; if neither is
installed, add one as a test-only dependency — `responses` is the more
commonly used, lightweight choice for mocking `requests` calls in
tests) — NEVER make real HTTP calls to ZarinPal's actual sandbox in
automated tests:
1. A successful mocked response (matching the code-100 success shape)
   returns a `PaymentRequestResult` with `success=True` and a correctly
   formatted `redirect_url` containing the returned authority.
2. A mocked error response returns `success=False` with a populated
   `error_message`, no exception raised.
3. A mocked network timeout/connection error is caught and returns
   `success=False` gracefully.
4. A mocked response with malformed (non-JSON) body is caught and
   returns `success=False` gracefully.
5. Confirm the `timeout` parameter is actually passed on the outbound
   `requests.post()` call (assert via the mock library's request
   history/call inspection, not just by reading the code).
```

---

#### Task 6.2.1.2 — `POST /api/payments/initiate/` endpoint

```
You are working in backend/payments/views.py, serializers.py, urls.py,
and backend/order/serializers.py (OrderCreateSerializer). Assume Task
6.2.1.1 is already merged.

CONTEXT — A REAL BUG THIS TASK MUST FIX
`OrderCreateSerializer.create()` currently sets
`status=Order.Status.PROCESSING` immediately at order creation, before
any payment has been collected — appropriate for the current
no-real-payment checkout flow, but wrong once real gateways exist: an
order shouldn't be "processing" (implying active fulfillment) until
payment is actually confirmed. This task changes checkout to create the
order in a `PENDING`-payment state, then immediately kicks off a
payment request against the configured gateway, returning a redirect
URL to the frontend instead of a "your order is placed" confirmation.

TASK
1. Fix `OrderCreateSerializer.create()` to create orders with
   `status=Order.Status.PENDING` instead of `PROCESSING`.
2. Add `POST /api/payments/initiate/` accepting an `order_id`, calling
   the configured gateway's `request_payment()`, creating a
   `PaymentTransaction` row, and returning the gateway's redirect URL.

REQUIREMENTS
- In order/serializers.py, change the single line
  `status=Order.Status.PROCESSING,` to
  `status=Order.Status.PENDING,` inside `OrderCreateSerializer.create()`'s
  `Order.objects.create(...)` call. This is intentionally the ONLY
  change to that method in this task — do not touch anything else
  about order creation (stock locking, pricing, item snapshotting all
  stay exactly as built across Epics 1/3/5).
- Create `PaymentInitiateSerializer` in backend/payments/serializers.py:
  ```python
  from rest_framework import serializers

  class PaymentInitiateSerializer(serializers.Serializer):
      order_id = serializers.IntegerField()
  ```
- Create `PaymentInitiateView` in backend/payments/views.py:
  ```python
  from django.conf import settings
  from django.urls import reverse
  from rest_framework.views import APIView
  from rest_framework.response import Response
  from rest_framework import status
  from rest_framework.permissions import IsAuthenticated
  from order.models import Order
  from .models import PaymentTransaction
  from .serializers import PaymentInitiateSerializer
  from .gateways import get_payment_gateway


  class PaymentInitiateView(APIView):
      permission_classes = [IsAuthenticated]

      def post(self, request):
          serializer = PaymentInitiateSerializer(data=request.data)
          serializer.is_valid(raise_exception=True)
          order_id = serializer.validated_data["order_id"]

          try:
              order = Order.objects.get(pk=order_id, user=request.user)
          except Order.DoesNotExist:
              return Response(
                  {"order_id": "Order not found."}, status=status.HTTP_404_NOT_FOUND
              )

          if order.status != Order.Status.PENDING:
              return Response(
                  {"detail": "This order has already been paid or is no longer payable."},
                  status=status.HTTP_400_BAD_REQUEST,
              )

          gateway_name = settings.DEFAULT_PAYMENT_GATEWAY
          gateway = get_payment_gateway(gateway_name)
          callback_url = request.build_absolute_uri(
              reverse("payments:callback", kwargs={"gateway": gateway_name})
          )
          result = gateway.request_payment(
              amount=order.total,
              callback_url=callback_url,
              description=f"Order {order.order_number}",
          )

          if not result.success:
              return Response(
                  {"detail": result.error_message or "Payment initiation failed."},
                  status=status.HTTP_502_BAD_GATEWAY,
              )

          PaymentTransaction.objects.create(
              order=order,
              gateway=gateway_name,
              authority=result.authority,
              amount=order.total,
              status=PaymentTransaction.Status.PENDING,
          )

          return Response({"redirect_url": result.redirect_url}, status=status.HTTP_200_OK)
  ```
  Note: `order.user` requires the order to actually belong to the
  authenticated requester — this filter (`user=request.user`) is a
  real, deliberate security check, not incidental: without it, any
  authenticated user could submit any other user's numeric order ID and
  initiate/hijack payment flow against someone else's order.
  The 502 status for a failed gateway request (rather than 400 or 500)
  deliberately signals "we tried, an upstream service failed" — this is
  the semantically correct HTTP status for a failed call to an external
  dependency, distinct from a client input error (400) or an internal
  bug (500).
- Register the URL in backend/payments/urls.py:
  `path("initiate/", views.PaymentInitiateView.as_view(), name="initiate"),`
  (the `"payments:callback"` URL name referenced via `reverse()` above
  is created in Task 6.2.1.4 — this task's code references it by name
  ahead of its existence, which is fine in Django as long as it exists
  by the time `reverse()` is actually CALLED at runtime, not at import
  time; just don't expect this view to fully work end-to-end until
  Task 6.2.1.4 lands too).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/payments/tests/test_views.py:
1. Initiating payment for a valid, PENDING order belonging to the
   requesting user succeeds (200), returns a `redirect_url`, and
   creates a `PaymentTransaction` row with the correct `order`,
   `gateway`, `amount`, and `authority`. Mock the gateway's
   `request_payment()` call (don't hit a real network endpoint in this
   test).
2. Initiating payment for an order belonging to a DIFFERENT user
   returns 404 (proving the ownership check).
3. Initiating payment for an order that's already `PROCESSING` (i.e.
   already paid) is rejected with 400.
4. A mocked gateway failure (`result.success=False`) returns 502 and
   creates NO `PaymentTransaction` row.
5. Re-run the order test suite and confirm every existing order-creation
   test that asserted `order.status == Order.Status.PROCESSING`
   immediately after checkout is updated to assert `PENDING` instead —
   this is an intentional, expected behavior change from this task, not
   a regression; find and fix every such assertion.
```

---

#### Task 6.2.1.3 — Implement `ZarinPalGateway.verify_payment()`

```
You are working in backend/payments/gateways/zarinpal.py. Assume Task
6.2.1.1 (request_payment) is already merged. Same "verify current API
docs before hardcoding" caveat from Task 6.2.1.1 applies here in full.

CONTEXT
`request_payment()` starts a payment and gets an authority code back;
after the customer completes (or abandons) payment on ZarinPal's
hosted page, ZarinPal redirects them back to this platform's
`callback_url` — at that point, the platform must call ZarinPal's
VERIFY API to confirm the payment actually succeeded server-to-server
(never trust the mere fact that the customer was redirected back with
a "success" query parameter as proof of payment — that redirect is
client-controlled and could be forged; the verify API call is the only
trustworthy confirmation).

TASK
Implement `ZarinPalGateway.verify_payment()`.

REQUIREMENTS
- Replace the `raise NotImplementedError` stub from Task 6.2.1.1:
  ```python
  VERIFY_URL = (
      "https://sandbox.zarinpal.com/pg/v4/payment/verify.json"
      if settings.ZARINPAL_SANDBOX
      else "https://api.zarinpal.com/pg/v4/payment/verify.json"
  )
  # NOTE: verify against current ZarinPal docs before relying on this.

  def verify_payment(self, authority, amount):
      payload = {
          "merchant_id": settings.ZARINPAL_MERCHANT_ID,
          "authority": authority,
          "amount": int(amount),  # same unit-conversion caveat as request_payment
      }
      try:
          response = requests.post(self.VERIFY_URL, json=payload, timeout=15)
          response.raise_for_status()
          data = response.json()
      except (requests.RequestException, ValueError) as exc:
          return PaymentVerifyResult(success=False, error_message=str(exc), raw_response=None)

      result_data = data.get("data", {})
      errors = data.get("errors", {})
      # Verify the exact success-code semantics against current docs —
      # ZarinPal typically distinguishes "100" (fresh success) from
      # "101" (already verified) as BOTH being valid non-error outcomes
      # for an idempotent verify call — confirm this and handle both as
      # success if that's accurate, since a customer double-hitting the
      # callback URL (e.g. via browser back-button/refresh) should not
      # be treated as a payment failure on the second call.
      if result_data and result_data.get("code") in (100, 101):
          return PaymentVerifyResult(
              success=True,
              ref_id=str(result_data.get("ref_id", "")),
              raw_response=data,
          )
      return PaymentVerifyResult(
          success=False,
          error_message=str(errors) or "ZarinPal payment verification failed.",
          raw_response=data,
      )
  ```
  Make sure the exact same `amount` value (matching whatever unit
  conversion was used in `request_payment()`) is sent to verify — a
  mismatched amount between request and verify is itself grounds for
  the gateway to reject the verification, per how these APIs typically
  work, so consistency here matters for correctness, not just style.

ACCEPTANCE CRITERIA / TESTS
Add tests to test_zarinpal.py (same mocking approach as Task 6.2.1.1):
1. A mocked success response (code 100) returns `success=True` with the
   correct `ref_id`.
2. A mocked "already verified" response (code 101, if that's confirmed
   accurate per current docs) ALSO returns `success=True` — proving
   idempotent verification is handled correctly.
3. A mocked error/failure response returns `success=False` with an
   error message.
4. Network failure and malformed-response cases degrade gracefully,
   matching the same pattern as `request_payment()`'s equivalent tests.
```

---

#### Task 6.2.1.4 — `GET /api/payments/callback/zarinpal/` endpoint

```
You are working in backend/payments/views.py, urls.py. Assume Task
6.2.1.3 (verify_payment) and Task 6.2.1.2 (PaymentInitiateView,
Order.status=PENDING at creation) are already merged. Also assume Epic
4's `StockMovement` model exists for logging, and Epic 1/3's atomic,
variant-locked stock logic in order creation is already in place (this
task does NOT touch stock decrement — that already happened at ORDER
creation time in Epic 1/3's flow, not at payment time; this task's job
is purely to confirm payment and transition order status accordingly —
see the important design note below).

CONTEXT — IMPORTANT DESIGN DECISION TO GET RIGHT
This codebase's existing order-creation flow (Epic 1 + Epic 3) already
decrements variant stock AT THE MOMENT THE ORDER IS CREATED (inside
`OrderCreateSerializer.create()`'s atomic block), which happens BEFORE
the customer ever reaches the payment gateway (per Task 6.2.1.2's new
flow: create order as PENDING → THEN initiate payment). This means
stock is already reserved the instant an order row exists, even before
payment succeeds — which is actually a reasonable design choice
(prevents someone else from buying the last unit while a customer is
mid-payment-flow on an external gateway page), but it has a real
consequence THIS task must handle correctly: if payment ultimately
FAILS or the customer abandons checkout, that reserved stock must be
RELEASED back (mirroring the exact stock-restoration logic already
built for order cancellation in Epic 1 Task 1.1.1.4 / Epic 3 Task
3.1.1.5 / Epic 4 Task 4.1.1.3) — otherwise stock silently and
permanently "leaks" every time a payment fails or a customer abandons
the gateway page, since it was already decremented and would never be
given back. This is a genuinely important correctness requirement of
this task, not an edge case to skip.

TASK
Add the ZarinPal callback endpoint: verify the payment, and either (a)
mark the order `PROCESSING` and the transaction `SUCCESS`, or (b) mark
the transaction `FAILED`, RELEASE the reserved stock back (reusing the
existing restoration logic), and leave/mark the order in a terminal
non-payable state.

REQUIREMENTS
- Add an `order.Status` choice if one doesn't already fit: check
  `Order.Status` (`PENDING, PROCESSING, SHIPPED, DELIVERED, CANCELLED`)
  — there's no explicit "payment failed" status distinct from
  `CANCELLED`; decide whether a failed/abandoned payment should
  transition the order to `CANCELLED` (reusing the existing status and
  its existing stock-restoration code path directly) or whether a new
  status is warranted. Reusing `CANCELLED` is the simpler, more
  consistent choice — it already has well-tested restoration semantics
  from prior epics, and semantically "payment never completed" is a
  reasonable specific case of "this order did not proceed" — do this
  unless you find a strong reason not to, and note the choice in a
  comment.
- Implement the view in backend/payments/views.py:
  ```python
  from django.db import transaction
  from django.shortcuts import redirect
  from rest_framework.views import APIView
  from rest_framework.permissions import AllowAny
  from shop.models import ProductVariant
  from order.models import Order
  from shop.models import StockMovement
  from .models import PaymentTransaction
  from .gateways import get_payment_gateway


  class PaymentCallbackView(APIView):
      """GET /api/payments/callback/<gateway>/"""
      permission_classes = [AllowAny]  # the gateway itself calls this, not an authenticated user

      def get(self, request, gateway):
          authority = request.GET.get("Authority") or request.GET.get("authority")
          gateway_status = request.GET.get("Status") or request.GET.get("status")

          try:
              txn = PaymentTransaction.objects.select_related("order").get(
                  gateway=gateway, authority=authority
              )
          except PaymentTransaction.DoesNotExist:
              return redirect(f"{settings.FRONTEND_URL}/checkout/failed?reason=unknown_transaction")

          if txn.status != PaymentTransaction.Status.PENDING:
              # Already processed (e.g. duplicate callback hit) — redirect
              # based on the ALREADY-recorded outcome, don't re-verify or
              # re-mutate anything (idempotency).
              if txn.status == PaymentTransaction.Status.SUCCESS:
                  return redirect(f"{settings.FRONTEND_URL}/order-confirmation/{txn.order.id}")
              return redirect(f"{settings.FRONTEND_URL}/checkout/failed?reason=already_failed")

          gateway_instance = get_payment_gateway(gateway)

          if gateway_status == "NOK":
              # Gateway itself reports the customer cancelled/failed
              # before even reaching verification — no need to call
              # verify_payment for a known-cancelled flow.
              self._mark_failed(txn)
              return redirect(f"{settings.FRONTEND_URL}/checkout/failed?reason=cancelled")

          result = gateway_instance.verify_payment(authority=authority, amount=txn.order.total)

          if result.success:
              with transaction.atomic():
                  txn.status = PaymentTransaction.Status.SUCCESS
                  txn.ref_id = result.ref_id
                  txn.raw_callback_payload = result.raw_response
                  txn.save()
                  txn.order.status = Order.Status.PROCESSING
                  txn.order.save(update_fields=["status"])
              return redirect(f"{settings.FRONTEND_URL}/order-confirmation/{txn.order.id}")

          self._mark_failed(txn)
          return redirect(f"{settings.FRONTEND_URL}/checkout/failed?reason=verification_failed")

      def _mark_failed(self, txn):
          """Mark the transaction failed, cancel the order, and release
          the stock that was reserved at order-creation time."""
          with transaction.atomic():
              txn.status = PaymentTransaction.Status.FAILED
              txn.save(update_fields=["status"])

              order = txn.order
              if order.status == Order.Status.PENDING:
                  order.status = Order.Status.CANCELLED
                  order.save(update_fields=["status"])
                  for item in order.items.select_related("variant").all():
                      if item.variant is None:
                          continue
                      locked_variant = ProductVariant.objects.select_for_update().get(
                          pk=item.variant.pk
                      )
                      locked_variant.stock += item.quantity
                      locked_variant.save(update_fields=["stock"])
                      StockMovement.objects.create(
                          variant=locked_variant,
                          reason=StockMovement.Reason.CANCELLATION,
                          quantity_delta=item.quantity,
                          stock_after=locked_variant.stock,
                          related_order=order,
                          note=f"Payment failed for order {order.order_number}",
                      )
  ```
  Import `settings` from `django.conf` at the top. Add
  `FRONTEND_URL = config("FRONTEND_URL", default="http://localhost:5173")`
  to backend/core/settings/base.py (this is a genuinely new, currently
  entirely absent setting in this codebase — check first to confirm it
  doesn't already exist under a different name like `SITE_URL`; if it
  does, reuse that existing setting instead of introducing a
  duplicate).
  Note the exact query parameter names ZarinPal uses on its callback
  redirect (`Authority`/`Status` above, with a `NOK` value for
  cancelled) are illustrative based on commonly-documented ZarinPal
  behavior — VERIFY these exact parameter names/casing against current
  ZarinPal documentation before relying on them, per this epic's
  standing caveat.
  This `_mark_failed` logic deliberately DUPLICATES the stock-
  restoration pattern from Epic 1/3/4 rather than calling a shared
  helper — consider whether extracting a shared
  `order.services.stock.release_reserved_stock(order)` function (used
  by BOTH `OrderDetailView.patch()`'s cancellation path AND this
  payment-failure path) would be cleaner; if the duplication is small
  enough, either approach is acceptable, but if you find yourself
  copying more than this handful of lines, extract the shared helper
  into `order/services/stock.py` and have both call sites use it —
  don't leave two independently-maintained copies of financially
  important logic if a shared function is a reasonably small lift.
- Register the URL in backend/payments/urls.py:
  `path("callback/<str:gateway>/", views.PaymentCallbackView.as_view(), name="callback"),`
  (matching the `reverse("payments:callback", kwargs={"gateway": ...})`
  call already written in Task 6.2.1.2's `PaymentInitiateView`).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/payments/tests/test_views.py:
1. A successful verified callback marks the `PaymentTransaction`
   `SUCCESS`, the `Order` `PROCESSING`, and redirects to the order
   confirmation URL — and does NOT touch stock (it was already
   correctly decremented at order-creation time, so no double-
   decrement should happen here).
2. A failed/rejected verification marks the transaction `FAILED`, the
   order `CANCELLED`, redirects to the failure URL, AND restores the
   previously-reserved stock for every item on that order (assert
   variant stock values before/after match expectations) with a
   corresponding `StockMovement` row created.
3. A callback for a `gateway_status=NOK` (customer-cancelled) case
   short-circuits to the failed path WITHOUT calling `verify_payment()`
   at all (mock it and assert it was never called).
4. A duplicate callback hit against an ALREADY-`SUCCESS` transaction
   redirects correctly without re-mutating anything or calling
   `verify_payment()` again (idempotency).
5. A callback with an authority that doesn't match any known
   `PaymentTransaction` redirects to a generic failure page without
   raising an unhandled exception.
```

---

#### Task 6.2.1.5 — Idempotency guard on callback endpoint

```
You are working in backend/payments/views.py (PaymentCallbackView).
Assume Task 6.2.1.4 is already merged (which already includes a basic
idempotency check via `if txn.status != PENDING: ...`).

CONTEXT
Task 6.2.1.4 already added a first layer of idempotency (checking
`txn.status` before processing), which correctly handles the common
case of a customer hitting back/refresh on the confirmation page. But
there's a subtler race: if the gateway calls back TWICE in very close
succession (or a customer double-clicks something that triggers two
near-simultaneous requests to the callback URL), BOTH requests could
read `txn.status == PENDING` before EITHER has finished writing its
update — a classic check-then-act race condition, the exact same class
of bug that Epic 1's `select_for_update()` work was built to prevent
elsewhere in this codebase, just not yet applied here.

TASK
Harden `PaymentCallbackView` against concurrent duplicate callback
processing using the same row-locking pattern established throughout
this codebase since Epic 1.

REQUIREMENTS
- Wrap the transaction-status-check-and-update logic in
  `PaymentCallbackView.get()` in a `transaction.atomic()` block with
  `select_for_update()` on the `PaymentTransaction` row, locking it
  BEFORE checking its current status — mirroring the exact
  lock-then-check-then-act pattern from Epic 1 Task 1.1.1.2/1.1.1.3:
  ```python
  with transaction.atomic():
      try:
          txn = PaymentTransaction.objects.select_for_update().select_related("order").get(
              gateway=gateway, authority=authority
          )
      except PaymentTransaction.DoesNotExist:
          return redirect(f"{settings.FRONTEND_URL}/checkout/failed?reason=unknown_transaction")

      if txn.status != PaymentTransaction.Status.PENDING:
          # already processed — same handling as Task 6.2.1.4
          ...
          return redirect(...)

      # only ONE concurrent request can reach this point per transaction
      # row at a time; the other will block here until the first
      # commits, then see status != PENDING and take the early-exit path.
      ...
  ```
  Restructure Task 6.2.1.4's existing view body to fit this locking
  shape — the KEY change is that the `select_for_update()`-locked fetch
  and the subsequent status check must happen INSIDE the SAME atomic
  block that also performs the eventual status-mutating writes,
  ensuring no other request can interleave between the check and the
  write for the SAME transaction row.
  Note: the OUTBOUND call to `gateway_instance.verify_payment()` (a
  network call to ZarinPal) happening WHILE HOLDING a row lock is worth
  a deliberate decision — holding a DB lock during a slow external
  network call ties up a DB connection/lock for however long that
  external call takes, which is generally something to avoid. Consider
  restructuring so the external verify call happens BEFORE acquiring
  the lock (verify first, then lock+check+write), accepting that in the
  rare double-callback race, verify might be called twice (which is
  fine — Task 6.2.1.3 already confirmed ZarinPal's verify endpoint is
  itself idempotent, code 101 for "already verified"), while the
  actual DATABASE state mutation (the part that must never happen
  twice) stays inside the tight lock-check-write block. This ordering
  (network call outside the lock, only the DB mutation inside it) is
  the better design — implement it this way rather than holding the
  lock across the network call.

ACCEPTANCE CRITERIA / TESTS
Add a concurrency test to backend/payments/tests/, mirroring the style
of Epic 1 Task 1.1.1.5's `test_stock_concurrency.py` (using
`TransactionTestCase` and `threading`/`ThreadPoolExecutor` to fire two
near-simultaneous requests at the SAME callback URL for the SAME
transaction):
1. Two concurrent callback requests for the same PENDING transaction,
   both with a mocked successful `verify_payment()` response, result in
   the order being marked `PROCESSING` and StockMovement/order-status
   changes happening EXACTLY ONCE, not twice (e.g. assert
   `Order.objects.get(pk=order.pk).status == PROCESSING` and that no
   duplicate side effects occurred — the exact assertion depends on
   what's easiest to observe as a "did this run twice" signal, such as
   checking `order.updated_at` was only touched once in a way you can
   verify, or more simply asserting the final DB state is correct and
   consistent rather than corrupted/double-applied).
```

---

#### Task 6.2.1.6 — Sandbox/test-mode config toggle

```
You are working in backend/core/settings/base.py,
backend/core/settings/development.py, backend/core/settings/production.py.
Assume Task 6.2.1.1 (which already introduced `ZARINPAL_SANDBOX`) is
merged.

CONTEXT
Task 6.2.1.1 already added `ZARINPAL_SANDBOX = config("ZARINPAL_SANDBOX", default=True, cast=bool)`
to base.py, which the `ZarinPalGateway` class already reads to pick
between sandbox/production URLs. This task's job is narrower than it
might sound: confirm the ENVIRONMENT-SPECIFIC defaults are actually
correct and safe (a production deployment accidentally defaulting to
sandbox mode would silently accept "successful" test payments without
collecting real money; a development environment accidentally
defaulting to production mode risks hitting real payment infrastructure
during local development/testing).

TASK
Ensure `ZARINPAL_SANDBOX` and `ZARINPAL_MERCHANT_ID` are correctly,
safely defaulted per environment, and document the required
environment variables.

REQUIREMENTS
- In backend/core/settings/base.py, the existing default of
  `default=True` for `ZARINPAL_SANDBOX` is a SAFE default (fails toward
  sandbox/test mode if the env var is somehow missing, rather than
  silently defaulting to production) — confirm this stays as-is, don't
  change the base default.
- In backend/core/settings/production.py, do NOT override
  `ZARINPAL_SANDBOX` with a hardcoded `False` — production mode should
  be an EXPLICIT, deliberate choice made via the actual deployment
  environment's `ZARINPAL_SANDBOX=False` env var, not silently forced
  by the settings file itself; a settings-file override would make it
  impossible to run a genuine sandbox-mode smoke test against a
  production-configured deployment (e.g. staging) if ever needed. Add
  a comment in production.py near wherever other payment-adjacent
  settings might live noting that `ZARINPAL_SANDBOX` and
  `ZARINPAL_MERCHANT_ID` MUST be explicitly set via environment
  variables for a real production deployment, and that the code will
  silently stay in harmless sandbox mode (collecting no real money) if
  they're forgotten — which is the safe failure direction, but should
  still be caught before going live.
- Add a startup-time sanity check: in `production.py`, add a check (if
  this project has any existing pattern for settings validation at
  startup — check for a `django.core.checks` custom check registration
  anywhere in the codebase first; if none exists, a simple assertion or
  a Django system check is a reasonable new addition) that raises a
  clear error if running with `DEBUG=False` (production) AND
  `ZARINPAL_SANDBOX=True` AND some indicator that this is genuinely a
  production deploy (this is inherently a bit fuzzy — a legitimate
  staging environment might deliberately run production-like settings
  with sandbox payments on purpose — so consider making this a WARNING
  logged at startup rather than a hard startup-blocking error, since a
  hard block could be actively wrong for a valid staging use case; use
  judgment here and document your choice).
- Confirm/create a `.env.example` file at backend/ (check if one
  already exists — grounded search earlier found none) documenting
  every payment-related environment variable now required:
  ```
  DEFAULT_PAYMENT_GATEWAY=zarinpal
  ZARINPAL_MERCHANT_ID=
  ZARINPAL_SANDBOX=True
  FRONTEND_URL=http://localhost:5173
  ```
  If a `.env.example` file doesn't exist AT ALL in this project yet
  (confirmed true from the earlier grounding search), consider whether
  creating one now — populated with EVERY env var this project actually
  reads via `config(...)` calls across all settings files, not just the
  payment ones added in this epic — is a reasonable, valuable addition;
  if that's a much bigger sweep than fits this task's scope, create it
  scoped to just the payment-related variables added across this epic
  so far, and leave a `# TODO: expand this file to cover all project
  env vars` note, rather than silently expanding this task's scope
  into a full settings audit.

ACCEPTANCE CRITERIA / TESTS
- Add a settings-level test (if this project has any precedent for
  testing settings values directly — a simple test asserting
  `settings.ZARINPAL_SANDBOX is True` under default/no-env-var
  conditions is reasonable and cheap to add regardless of precedent)
  confirming the safe default holds.
- Manually verify: running the Django dev server locally with no
  `ZARINPAL_MERCHANT_ID` set doesn't crash the app at import/startup
  time (it should only fail when an actual payment is attempted,
  surfacing a clear error via the existing `request_payment()`
  error-handling from Task 6.2.1.1, not an unhandled exception at
  server boot).
```

---

#### Task 6.2.1.7 — ZarinPal integration test suite (mocked HTTP)

```
You are adding tests to backend/payments/tests/. Assume Tasks 6.2.1.1
through 6.2.1.6 are all merged.

CONTEXT
Individual unit tests exist per-task (mocked `request_payment`/
`verify_payment` calls, view-level tests for initiate/callback,
concurrency test for idempotency) — but there's no single test proving
the FULL, real, end-to-end ZarinPal flow works correctly through the
actual HTTP API surface a browser/frontend would hit, the way Epic 2
Task 2.3.1.4 built one comprehensive OTP integration suite after its
individual endpoint tasks.

TASK
Write a comprehensive integration test suite covering the complete
checkout → initiate → (mocked gateway redirect) → callback → order-
confirmed flow, using `responses`/`requests-mock` for the OUTBOUND
gateway calls and DRF's `APIClient` for the INBOUND API calls a real
client would make.

REQUIREMENTS — scenarios to cover end-to-end
1. **Full happy path:** authenticated user has items in cart → POST
   `/api/orders/` creates a `PENDING` order (per Task 6.2.1.2's fix) →
   POST `/api/payments/initiate/` with that order's id (mocked
   ZarinPal request-payment success) returns a `redirect_url` and
   creates a `PENDING` `PaymentTransaction` → simulate the customer
   completing payment by GETting the callback URL with a valid
   authority (mocked ZarinPal verify success) → assert the order is now
   `PROCESSING`, the transaction is `SUCCESS`, and the response redirects
   to the order-confirmation URL.
2. **Payment failure releases stock:** same setup, but mock the
   ZarinPal verify call to fail → assert the order becomes `CANCELLED`,
   the transaction becomes `FAILED`, AND every variant's stock that was
   decremented at order-creation time is correctly restored to its
   pre-order value (this is the single most financially important
   assertion in this whole test suite — verify it precisely, checking
   exact before/after stock numbers, not just "some restoration
   happened").
3. **Customer cancels on gateway page:** callback hit with the
   cancelled-status query param (per Task 6.2.1.4's `NOK` handling) →
   same order-cancelled/stock-restored outcome as scenario 2, WITHOUT
   `verify_payment()` ever being called (assert the mock was never
   invoked).
4. **Attempting to initiate payment twice for the same order:** first
   initiate succeeds; a second `POST /api/payments/initiate/` for the
   SAME already-non-PENDING order (after scenario 1's flow completed
   it) is rejected with 400.
5. **Gateway network failure during initiate:** mock a connection
   error/timeout on the request-payment call → `POST /api/payments/initiate/`
   returns 502, and — importantly — confirm the ORDER remains in
   `PENDING` state with its stock STILL reserved (a failed INITIATE,
   before any transaction/authority even exists, is different from a
   failed VERIFY — there's no `PaymentTransaction` row to mark
   `FAILED` yet in this scenario, so nothing should be auto-cancelled;
   the customer should simply be able to retry initiating payment
   again for the same still-pending order). Confirm this distinction is
   handled correctly and doesn't accidentally trigger any stock
   restoration logic that shouldn't fire yet.

ACCEPTANCE CRITERIA
All 5 scenarios pass. Run the FULL payments + order test suites
together at the end (`pytest backend/payments/ backend/order/`) and
confirm zero regressions across everything built in this epic and all
prior epics' order/stock logic.
```

---

## Phase 6.3 — Secondary Gateways

### Feature 6.3.1 — Zibal & IDPay

---

#### Task 6.3.1.1 — Implement `ZibalGateway`

```
You are working in backend/payments/gateways/zibal.py (new file).
Assume Phase 6.2 is fully merged (ZarinPal fully working end-to-end,
establishing the concrete pattern to follow).

CONTEXT
`PAYMENT_GATEWAY_CLASSES` in backend/core/settings/base.py has a
commented-out entry for Zibal (from Task 6.1.1.4) — this task
implements it for real, giving the platform a second, independently
usable payment gateway option, following the exact same
`PaymentGateway` interface `ZarinPalGateway` already implements.

Same standing caveat as every gateway task in this epic: verify Zibal's
CURRENT, authoritative API documentation for exact endpoint URLs,
request/response field names, and amount-unit expectations before
hardcoding anything — do not assume Zibal's API shape mirrors
ZarinPal's exactly just because they're both Iranian payment gateways;
they are different companies with their own independently-designed
APIs.

TASK
Implement `ZibalGateway(PaymentGateway)` with `request_payment()` and
`verify_payment()`, following the exact same structural pattern as
`ZarinPalGateway` (Tasks 6.2.1.1/6.2.1.3) for consistency, error
handling, and timeout discipline.

REQUIREMENTS
- Add settings:
  ```python
  ZIBAL_MERCHANT_ID = config("ZIBAL_MERCHANT_ID", default="")
  ```
  (Zibal's sandbox convention may differ from ZarinPal's — e.g. some
  Iranian gateways use a special fixed "test" merchant identifier
  rather than a separate sandbox subdomain; verify Zibal's actual
  documented sandbox/test mechanism and implement whatever pattern
  ZarinPal — a full separate `ZIBAL_SANDBOX` boolean if Zibal uses
  separate endpoints, or a special test-mode merchant ID constant if
  that's how Zibal's sandbox actually works — don't assume it's
  identical to ZarinPal's approach without checking).
- Implement both methods with the SAME structural shape as
  `ZarinPalGateway`: a `try/except (requests.RequestException, ValueError)`
  wrapping every outbound call, a `timeout=15` on every request,
  returning `PaymentRequestResult`/`PaymentVerifyResult` dataclass
  instances (never raising an unhandled exception out of either
  method), and the same idempotent-verify handling if Zibal's API has
  an equivalent "already verified" success case (confirm from their
  docs).
- Uncomment the Zibal entry in `PAYMENT_GATEWAY_CLASSES` in
  backend/core/settings/base.py:
  `"zibal": "payments.gateways.zibal.ZibalGateway",`

ACCEPTANCE CRITERIA / TESTS
Add backend/payments/tests/test_zibal.py mirroring
test_zarinpal.py's exact test structure and scenario coverage
(successful request, error response, network failure, malformed
response, for both request_payment and verify_payment) — the goal is
symmetric, equally thorough coverage across both gateways, not a
lighter-weight test suite just because ZarinPal was implemented first.
```

---

#### Task 6.3.1.2 — Implement `IDPayGateway`

```
You are working in backend/payments/gateways/idpay.py (new file).
Assume Task 6.3.1.1 is already merged (for the established pattern).

CONTEXT
Same rationale and standing API-verification caveat as Task 6.3.1.1,
now for IDPay as the third gateway option.

TASK
Implement `IDPayGateway(PaymentGateway)` following the identical
structural pattern established by `ZarinPalGateway` and `ZibalGateway`.

REQUIREMENTS
- Add settings:
  `IDPAY_API_KEY = config("IDPAY_API_KEY", default="")`
  (verify IDPay's actual authentication mechanism — API key in a header
  vs. a merchant ID in the payload — from their current docs; don't
  assume it matches either prior gateway's exact auth pattern).
- Implement both interface methods with the same error-handling/
  timeout/dataclass-return discipline as the two prior gateway
  implementations.
- Uncomment the IDPay entry in `PAYMENT_GATEWAY_CLASSES`:
  `"idpay": "payments.gateways.idpay.IDPayGateway",`

ACCEPTANCE CRITERIA / TESTS
Add backend/payments/tests/test_idpay.py mirroring the same test
structure/coverage as the other two gateway test files.
```

---

#### Task 6.3.1.3 — Gateway selection logic (admin-configurable default + fallback)

```
You are working in backend/payments/ (new services module) and
backend/dashboard/. Assume Tasks 6.3.1.1 and 6.3.1.2 are already
merged (three working gateways now exist).

CONTEXT
`DEFAULT_PAYMENT_GATEWAY` (a settings-level, deployment-time constant)
currently picks ONE gateway for the entire platform with no runtime
flexibility and no automatic fallback if that gateway is having an
outage — a real operational gap: if ZarinPal has downtime, every
checkout on the platform fails until someone manually changes an
environment variable and redeploys.

TASK
Add an admin-configurable "active gateway" setting stored in the
database (not just an env var), plus a fallback order so a failed
initiate attempt against the primary gateway automatically tries the
next one in the configured order before giving up.

REQUIREMENTS
- Create a simple `PaymentGatewayConfig` model in
  backend/payments/models.py (a singleton-style config row, following
  whatever singleton-config pattern this project might already use
  elsewhere if one exists — check for precedent first; if none exists,
  a straightforward `PaymentGatewayConfig` model with
  `objects.first()`-based access and a `save()` override enforcing only
  one row can ever exist is a reasonable, simple approach):
  ```python
  class PaymentGatewayConfig(models.Model):
      active_gateway = models.CharField(
          max_length=20, choices=PaymentTransaction.Gateway.choices,
          default=PaymentTransaction.Gateway.ZARINPAL,
      )
      fallback_order = models.JSONField(
          default=list, blank=True,
          help_text="Ordered list of gateway names to try if the active gateway's request fails, e.g. ['zibal', 'idpay'].",
      )
      updated_at = models.DateTimeField(auto_now=True)

      def save(self, *args, **kwargs):
          self.pk = 1  # enforce singleton
          super().save(*args, **kwargs)

      @classmethod
      def get_solo(cls):
          obj, _ = cls.objects.get_or_create(pk=1)
          return obj
  ```
  Generate the migration.
- Register it in backend/payments/admin.py as a simple editable admin
  form (unlike `PaymentTransaction`, THIS model genuinely should be
  hand-editable by an admin — that's the entire point of this task):
  ```python
  @admin.register(PaymentGatewayConfig)
  class PaymentGatewayConfigAdmin(admin.ModelAdmin):
      list_display = ("active_gateway", "fallback_order", "updated_at")

      def has_add_permission(self, request):
          return not PaymentGatewayConfig.objects.exists()  # enforce singleton in the UI too

      def has_delete_permission(self, request, obj=None):
          return False
  ```
- Update `PaymentInitiateView` (from Task 6.2.1.2) to use
  `PaymentGatewayConfig.get_solo().active_gateway` instead of
  `settings.DEFAULT_PAYMENT_GATEWAY` as the PRIMARY choice (keep
  `settings.DEFAULT_PAYMENT_GATEWAY` as the fallback default used only
  when NO `PaymentGatewayConfig` row exists yet, e.g. immediately after
  a fresh deploy before an admin has configured anything — the
  `get_solo()` classmethod's `get_or_create` already handles creating a
  sensible default row using the model field's own `default=`, so this
  should naturally work without extra glue code, just confirm it does).
- Implement fallback: if `gateway.request_payment()` fails for the
  active gateway, iterate `PaymentGatewayConfig.get_solo().fallback_order`
  and retry with each listed gateway in turn until one succeeds or the
  list is exhausted; only return the 502 error response (from Task
  6.2.1.2) if EVERY gateway in the fallback chain also failed. Log
  (via Python's `logging` module) which gateway ultimately succeeded
  and, if any earlier ones failed first, that they failed — this
  operational visibility matters for noticing a gateway outage pattern
  before it becomes a bigger problem.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/payments/tests/test_views.py:
1. With `active_gateway=zarinpal` and no fallback configured, a mocked
   ZarinPal failure results in the existing 502 behavior (regression
   check against Task 6.2.1.2's original behavior).
2. With `active_gateway=zarinpal` and `fallback_order=["zibal"]`, a
   mocked ZarinPal failure followed by a mocked Zibal SUCCESS results
   in a 200 response with Zibal's redirect URL, and the created
   `PaymentTransaction.gateway` correctly records `"zibal"` (not
   `"zarinpal"`) as the gateway that actually got used.
3. With a full fallback chain where EVERY gateway fails, the final
   response is still 502, and no `PaymentTransaction` row is created
   for any of the failed attempts (only a successful `request_payment()`
   call should result in a persisted transaction row — confirm this
   matches the existing Task 6.2.1.2 behavior of only creating the row
   AFTER a successful gateway response).
4. Changing `PaymentGatewayConfig.active_gateway` via the admin/directly
   via the ORM changes which gateway is used on the NEXT initiate call,
   without requiring a server restart (proving this is genuinely
   runtime-configurable, unlike the old settings-only approach).
```

---

#### Task 6.3.1.4 — Callback routes for Zibal/IDPay

```
You are working in backend/payments/views.py, urls.py. Assume Task
6.3.1.3 is already merged.

CONTEXT
`PaymentCallbackView` (Task 6.2.1.4/6.2.1.5) already accepts a
`gateway` URL path parameter and dynamically resolves the correct
gateway instance via `get_payment_gateway(gateway)`, so it's ALREADY
gateway-agnostic in its core logic — this task should mostly be
CONFIRMING that genuinely holds true for Zibal/IDPay, not writing much
new code, plus handling any gateway-specific callback query-parameter
naming differences.

TASK
Confirm/adapt `PaymentCallbackView` to correctly handle Zibal's and
IDPay's callback query parameters (which likely use different
parameter names than ZarinPal's `Authority`/`Status`), and verify the
existing single URL pattern (`callback/<str:gateway>/`) correctly
routes all three gateways' callbacks to the same view.

REQUIREMENTS
- Check Zibal's and IDPay's actual documented callback query parameter
  names (per this epic's standing "verify current docs" caveat) — they
  will almost certainly NOT be called `Authority`/`Status` like
  ZarinPal; each gateway has its own convention (e.g. Zibal might use
  `trackId`, IDPay might use `id`/`status` with different value
  semantics for what counts as "success" vs "cancelled").
- Refactor `PaymentCallbackView.get()`'s currently ZarinPal-specific
  query-parameter extraction (`request.GET.get("Authority")`, the
  `NOK` status check) into a per-gateway-aware extraction step — the
  cleanest approach is to add a method to the `PaymentGateway` ABC
  itself (back in Task 6.1.1.4's base.py) such as
  `extract_callback_params(self, request) -> dict` returning a
  normalized `{"authority": ..., "is_customer_cancelled": bool}`
  shape regardless of which gateway's specific parameter names were
  used, and have each concrete gateway class (`ZarinPalGateway`,
  `ZibalGateway`, `IDPayGateway`) implement this extraction using its
  own actual parameter names — then `PaymentCallbackView` calls
  `gateway_instance.extract_callback_params(request)` instead of
  hardcoding ZarinPal's specific parameter names directly, making the
  view genuinely gateway-agnostic rather than accidentally
  ZarinPal-shaped with a gateway parameter bolted on. This requires
  going back and adding this new abstract method to `PaymentGateway`
  and implementing it retroactively on `ZarinPalGateway` too (moving
  the `Authority`/`Status`/`NOK` extraction logic that currently lives
  inline in the view into `ZarinPalGateway.extract_callback_params()`),
  not just on the two new gateways — do this refactor rather than
  special-casing Zibal/IDPay with if/elif branches inside the view
  itself, which would defeat the purpose of having an abstraction layer
  at all.

ACCEPTANCE CRITERIA / TESTS
Add tests confirming:
1. `ZarinPalGateway.extract_callback_params()` correctly parses a
   mocked ZarinPal-shaped callback request (regression test against
   the refactor — the view's overall behavior for ZarinPal callbacks
   must remain byte-identical to before this task).
2. `ZibalGateway.extract_callback_params()` and
   `IDPayGateway.extract_callback_params()` correctly parse their
   respective actual documented callback shapes.
3. `PaymentCallbackView` end-to-end tests (mirroring Task 6.2.1.7's
   integration suite structure) for BOTH Zibal and IDPay callback
   flows, covering success, failure, and customer-cancelled scenarios,
   with the same rigor as the existing ZarinPal integration tests —
   proving the whole callback pipeline genuinely works for all three
   gateways, not just the one it was originally built against.
```

---

## Phase 6.4 — Payment UX & Reliability

### Feature 6.4.1 — Frontend Payment Flow

---

#### Task 6.4.1.1 — Checkout "Pay Now" redirect flow

```
You are working in frontend/src/pages/Checkout.jsx and
frontend/src/services/api.js. Assume Phase 6.2 (ZarinPal, at minimum)
is fully merged on the backend.

CONTEXT — READ THE CURRENT CHECKOUT CODE CAREFULLY
`Checkout.jsx`'s `handleSubmit()` currently does this (confirmed
directly from the repo):

    const payload = {
      first_name: form.firstName, ..., 
      payment_method: paymentMethod,
      card_last_four: paymentMethod === 'credit_card' ? card.number.slice(-4) : '',
      discount: discount.toFixed(2),
      notes: form.notes,
    };
    const { data } = await createOrder(payload);
    navigate(`/order-confirmation/${data.id}`);

This collects a fake card number client-side, sends only its last 4
digits, and navigates straight to an order-confirmation page as if
payment already happened — because, currently, it did nothing at all
resembling real payment. This task replaces that flow with the real
one: create the order (now PENDING per Task 6.2.1.2), then redirect the
browser to the gateway.

Also note: `discount: discount.toFixed(2)` in the current payload sends
a client-computed discount value to the backend — per Epic 1 Task
1.2.1.1, `OrderCreateSerializer` no longer accepts or reads a
client-supplied `discount` field at all, so this line in the payload is
now silently ignored by the backend already; remove it from the
payload as dead code while you're in this file, since sending it is
misleading (implies it does something) even though Epic 1 already
closed the actual vulnerability server-side.

TASK
Replace the fake-payment checkout submission with: create order →
call `/api/payments/initiate/` → redirect the full browser window to
the returned gateway URL.

REQUIREMENTS
- Add to frontend/src/services/api.js (find the existing
  order-related API functions like `createOrder` and add alongside
  them, matching their exact style):
  ```javascript
  export const initiatePayment = (orderId) =>
    api.post('/payments/initiate/', { order_id: orderId });
  ```
- Update `Checkout.jsx`'s `handleSubmit()`:
  ```javascript
  const payload = {
    first_name: form.firstName,
    last_name: form.lastName,
    email: form.email,
    phone: form.phone,
    address: form.address,
    apartment: form.apartment,
    city: form.city,
    state: form.state,
    zip: form.zip,
    country: form.country,
    billing_same: form.billingSame,
    notes: form.notes,
    // address_id / save_address from Epic 5 Task 5.2.1.3/5.2.1.5's
    // saved-address picker, if that UI is already wired into this
    // form's state by this point — include those fields here too if so.
  };

  const { data: order } = await createOrder(payload);
  const { data: paymentInit } = await initiatePayment(order.id);
  window.location.href = paymentInit.redirect_url;
  ```
  — note `payment_method`/`card_last_four` are REMOVED from the
  payload entirely, since there's no card-entry step in the new flow
  at all (the gateway's own hosted page collects payment details, not
  this platform's checkout form) — this also means the existing
  card-number input fields in the checkout form itself are now
  obsolete; that removal is Task 6.4.1.3's job specifically, don't do
  it as a side effect of this task, keep this task scoped to the
  submission logic only (the old card fields can remain visually
  present-but-now-unused for this task if that keeps the diff focused,
  though flagging them as dead is fine).
  Using `window.location.href = ...` (a full page navigation) rather
  than React Router's `navigate()` is DELIBERATE and correct here — the
  gateway's URL is an entirely different origin/domain, and
  `useNavigate()` only works for in-app SPA routes; a true external
  redirect requires a real browser navigation.
- Error handling: if `createOrder` succeeds but `initiatePayment` fails
  (e.g. the mocked-502-worthy gateway-down scenario from Task 6.2.1.7),
  the order still EXISTS in `PENDING` state — show a clear error
  message ("Payment could not be started, please try again") with a
  retry option that re-calls `initiatePayment(order.id)` for the SAME
  already-created order, rather than creating a brand-new duplicate
  order — check whether the existing checkout page has any concept of
  "resume a pending order" already; if not, at minimum store the
  created `order.id` in component state after the first
  `createOrder()` success so a retry can skip straight to
  `initiatePayment()` without re-submitting the whole form/re-creating
  the order.

ACCEPTANCE CRITERIA
- Manually verify (against a backend running with
  `ZARINPAL_SANDBOX=True` and a valid sandbox merchant ID, if you have
  one available in the dev environment; otherwise mock the network
  response in dev tools to simulate the redirect): submitting the
  checkout form creates a `PENDING` order, then navigates the full
  browser to a ZarinPal (sandbox) URL.
- Update/add Vitest tests for `Checkout.jsx`'s submit handler (check
  frontend/src/__tests__/Checkout.test.jsx, which already exists per
  earlier grounding — update its existing assertions to match the new
  create-then-initiate flow instead of the old create-then-navigate-to-
  confirmation flow) confirming: successful submission calls
  `createOrder` then `initiatePayment` with the correct order id, and
  sets `window.location.href` to the mocked `redirect_url` (you'll need
  to mock/stub `window.location` appropriately in the test environment
  — check how, if at all, this project's existing tests handle
  navigation side effects, and follow precedent or use a standard
  Vitest/jsdom `window.location` mocking approach).
```

---

#### Task 6.4.1.2 — Payment result page (success/failure)

```
You are working in frontend/src (new page component(s) and routing).
Assume Task 6.4.1.1 and the backend's `PaymentCallbackView` (Phase 6.2)
redirect the browser to `${FRONTEND_URL}/order-confirmation/{id}` on
success and `${FRONTEND_URL}/checkout/failed?reason=...` on failure
(per Task 6.2.1.4's implementation).

CONTEXT
Check whether an `/order-confirmation/:id` route/page ALREADY exists
in this frontend (the old, pre-payment checkout flow already navigated
there directly per Task 6.4.1.1's "before" code, so it likely does —
confirm and reuse it rather than building a duplicate). There is
currently no `/checkout/failed` page at all, since payment failure
wasn't a reachable state before this epic.

TASK
Confirm/adapt the existing order-confirmation page to work correctly
when arrived at via a real gateway redirect (not just an in-app
`navigate()` call), and build a new checkout-failed page.

REQUIREMENTS
- Order confirmation page: since the browser arrives here via a REAL
  external redirect (not an in-SPA navigation), confirm the page
  correctly fetches the order's current data fresh from the API on
  mount (using the `:id` route param) rather than relying on any
  React Router `location.state` that might have been passed via the
  old `navigate()` call's second argument (a common pattern that
  silently breaks when arriving via an external redirect instead of
  in-app navigation, since `location.state` is only populated by
  React Router's own navigation, never by a plain browser URL
  redirect) — if the existing page relies on `location.state` for any
  of its displayed data, fix it to fetch via the order-detail API
  instead.
- Create a new `/checkout/failed` page/route, reading the `?reason=`
  query parameter and showing an appropriate, specific message per
  reason value:
  - `cancelled` → "You cancelled the payment. Your items are still in
    your cart." (note: per Task 6.2.1.4's backend logic, the ORDER
    itself is cancelled and stock released, but the CART's items were
    already cleared at order-creation time per the existing
    `cart.items.all().delete()` behavior from Epic 1 — so "items are
    still in your cart" would actually be FALSE with the current
    order-creation flow's timing; check this carefully: does this
    epic's PENDING-order-then-pay flow still clear the cart at ORDER
    CREATION time, before payment is even attempted? If so, that's a
    real UX problem worth flagging — a customer whose payment fails or
    who cancels has now lost their cart contents entirely with no easy
    way to reorder. Consider whether cart-clearing should be MOVED to
    happen only on CONFIRMED payment success (in
    `PaymentCallbackView`'s success path) rather than at order creation
    time — this is a meaningful design fix worth making as part of this
    task if it's not disproportionately large, since leaving it as-is
    creates a genuinely bad failed-payment experience; if you make this
    change, it belongs in the BACKEND (`OrderCreateSerializer.create()`
    should stop clearing the cart, and `PaymentCallbackView`'s success
    path should clear it instead), so flag clearly in your task summary
    that this task ended up touching backend code beyond its nominal
    "frontend result page" scope, and why).
  - `verification_failed` → "Your payment could not be confirmed.
    Please try again or contact support."
  - `unknown_transaction` / anything else → a generic
    "Something went wrong with your payment" fallback message.
  - In every failure case, offer a clear "Try again" action — ideally
    linking back to checkout with the cart intact (per the cart-timing
    fix above) rather than forcing the customer to rebuild their cart
    from scratch.

ACCEPTANCE CRITERIA
Manually verify all reason values render distinct, correct messaging,
and that the "try again" flow actually works end-to-end (cart still has
items, checkout can be resubmitted). Add basic component tests for the
new failed-checkout page covering each `reason` value's rendered
message.
```

---

#### Task 6.4.1.3 — Remove fake `card_last_four`/card-brand UI

```
You are working in frontend/src/pages/Checkout.jsx. Assume Tasks
6.4.1.1 and 6.4.1.2 are already merged (the real payment flow works
end-to-end; the card-entry form fields are now confirmed dead/unused).

CONTEXT
The checkout form currently includes card-number/expiry/CVV-style
input fields (referenced in the current code via `card.number`,
feeding the now-removed `card_last_four` payload field) — this was
always checkout-form theater (per the original architecture review's
finding: no real card processing ever happened, only the last 4 digits
were stored) and is now fully dead code following Task 6.4.1.1's
removal of `payment_method`/`card_last_four` from the actual submission
payload.

TASK
Remove the now-entirely-unused card-entry UI and any related client-
side state/validation from the checkout form.

REQUIREMENTS
- Remove the card-number/expiry/CVV input fields and their surrounding
  JSX from Checkout.jsx.
- Remove the `card` state object (`card.number`, etc.) and any
  `paymentMethod` state/selector UI that was previously used to choose
  between `credit_card`/`paypal`/`apple_pay` — since real payment
  method selection now happens on the GATEWAY's own hosted page, not
  this platform's checkout form, there's no reason for this platform to
  ask the customer to pick a payment method type at all anymore (unless
  Phase 6.3's multi-gateway support means the CUSTOMER should get to
  choose WHICH GATEWAY to pay through, as opposed to which card
  network — that's a different, legitimate future UX question, out of
  scope for this task, which is purely about removing the OLD fake
  card-entry theater, not adding new gateway-choice UI).
- Remove any client-side card-number validation logic (Luhn-check-style
  functions, expiry-date validation, etc., if present) that's now
  entirely unused.
- Remove the corresponding `card`/`paymentMethod` field entries from
  the `validate()` function used by `handleSubmit()` (per the earlier-
  read code, there's a `validate()` call before submission — check it
  for any card-related validation rules and remove them).
- Update/remove any related test assertions in
  frontend/src/__tests__/Checkout.test.jsx that reference the now-
  removed card fields.

ACCEPTANCE CRITERIA
- Manually verify the checkout form no longer shows any card-entry UI,
  and submitting it (per Task 6.4.1.1's new flow) still works correctly
  with only shipping/contact information collected on this platform.
- Run the full frontend test suite and confirm no lingering test
  references the removed fields/state, and no other component
  elsewhere in the app was relying on the removed `paymentMethod`
  state/props (search for any cross-component usage before deleting,
  e.g. an order-summary sidebar that might have displayed "Paying with:
  Credit Card" — if such displays exist, remove or adapt them too,
  since that information no longer exists in this platform's checkout
  state at all).
```

---

### Feature 6.4.2 — Payment Reliability

---

#### Task 6.4.2.1 — Celery task: reconcile stuck "pending" transactions

```
You are working in backend/payments/tasks.py (new file). Assume Celery
infrastructure exists (per this epic's Epic 22 dependency, same caveat
noted in Epic 3/4's Celery-dependent tasks — CONFIRM
backend/core/celery.py and django-celery-beat scheduling exist before
starting; if not, this task depends on Epic 22 Phase 22.1 landing
first). Assume Phase 6.2 is fully merged.

CONTEXT
A `PaymentTransaction` can get stuck in `PENDING` status indefinitely
if the customer never returns from the gateway's hosted page at all
(closes the tab, loses connectivity, etc.) — the callback endpoint
(Task 6.2.1.4/6.2.1.5) only ever runs if the gateway actually calls
back, which won't happen for a truly abandoned session. This leaves
orders sitting in `PENDING` status forever with stock permanently
reserved against them, unless something proactively reconciles these
stuck transactions.

TASK
Add a periodic Celery task that finds `PaymentTransaction` rows stuck
`PENDING` for longer than a configurable threshold, actively
re-verifies them against the gateway (in case the callback silently
failed to reach this platform even though payment actually succeeded —
a real, if rarer, failure mode worth checking for before giving up on
the transaction), and finalizes them one way or the other.

REQUIREMENTS
- Add a setting:
  `PAYMENT_RECONCILIATION_THRESHOLD_MINUTES = config("PAYMENT_RECONCILIATION_THRESHOLD_MINUTES", default=30, cast=int)`
  to backend/core/settings/base.py.
- Implement in backend/payments/tasks.py:
  ```python
  from celery import shared_task
  from django.conf import settings
  from django.utils import timezone
  from datetime import timedelta

  @shared_task
  def reconcile_stuck_payment_transactions():
      from .models import PaymentTransaction
      from .gateways import get_payment_gateway
      from .views import PaymentCallbackView  # reuse finalization logic — see note below

      cutoff = timezone.now() - timedelta(minutes=settings.PAYMENT_RECONCILIATION_THRESHOLD_MINUTES)
      stuck = PaymentTransaction.objects.filter(
          status=PaymentTransaction.Status.PENDING, created_at__lt=cutoff
      ).select_related("order")

      reconciled_success, reconciled_failed, still_pending = 0, 0, 0
      for txn in stuck:
          gateway_instance = get_payment_gateway(txn.gateway)
          result = gateway_instance.verify_payment(authority=txn.authority, amount=txn.order.total)
          if result.success:
              # finalize as success — actually paid, callback must have failed to arrive
              ...
              reconciled_success += 1
          else:
              # genuinely failed/abandoned — finalize as failed, release stock
              ...
              reconciled_failed += 1
      return f"Reconciled: {reconciled_success} success, {reconciled_failed} failed."
  ```
  Note the finalization logic for BOTH the success and failure branches
  here is IDENTICAL to what `PaymentCallbackView` already does in Task
  6.2.1.4/6.2.1.5 (mark transaction, mark order, release stock on
  failure) — this is a strong signal that logic should be extracted
  into a SHARED function (e.g.
  `payments/services.py`'s `finalize_transaction_success(txn, result)`
  and `finalize_transaction_failure(txn)`) used by BOTH
  `PaymentCallbackView` AND this Celery task, rather than duplicating
  financially-important logic in two places that could drift out of
  sync over time. Extract this shared logic now as part of this task —
  go back and refactor `PaymentCallbackView` to use the same extracted
  functions rather than leaving its inline version as a second,
  independent copy.
- Register this as a periodic task (e.g. running every 10-15 minutes)
  using whatever `django-celery-beat` scheduling mechanism Epic 22's
  Celery infrastructure established, matching the pattern already used
  for Epic 3 Task 3.3.1.3's `deactivate_expired_variants` and Epic 4
  Task 4.1.2.3's notification task.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/payments/tests/test_tasks.py:
1. A `PENDING` transaction older than the threshold, where a mocked
   `verify_payment()` call now returns success, is reconciled to
   `SUCCESS`/order `PROCESSING` — proving the "callback silently
   failed to arrive but payment actually succeeded" recovery case
   works.
2. A `PENDING` transaction older than the threshold where verify
   returns failure is reconciled to `FAILED`/order `CANCELLED` with
   stock restored.
3. A `PENDING` transaction NEWER than the threshold is left untouched
   (not yet eligible for reconciliation — still might get a real
   callback any moment).
4. An already-`SUCCESS`/`FAILED` transaction (regardless of age) is
   never touched by this task (the `.filter(status=PENDING)` clause
   already excludes it, but confirm with an explicit test).
5. Re-run `PaymentCallbackView`'s existing tests from Task 6.2.1.4/
   6.2.1.5/6.2.1.7 after the shared-function refactor and confirm they
   all still pass unchanged — proving the extraction was behavior-
   neutral for the existing callback path.
```

---

#### Task 6.4.2.2 — Admin manual payment status override (with audit trail)

```
You are working in backend/payments/admin.py, models.py. Assume Task
6.1.1.2 (PaymentTransaction, currently fully locked-down/read-only in
admin) is merged.

CONTEXT
`PaymentTransactionAdmin` (Task 6.1.1.2) currently blocks ALL editing
through the admin entirely (`has_change_permission` isn't even
overridden to allow it, and `readonly_fields` covers every field) —
appropriate as a DEFAULT posture for a financial audit record, but real
support scenarios genuinely need a deliberate, logged manual override
path: e.g. a customer support agent confirms via the ZarinPal merchant
dashboard directly that a payment actually succeeded despite this
platform's records showing it failed (a discrepancy that can
legitimately happen), and needs to manually correct the order/
transaction status — WITHOUT opening a full unrestricted edit form that
could let someone accidentally corrupt other fields.

TASK
Add a narrow, deliberate, audit-logged admin ACTION (not a general
edit form) for manually marking a stuck/incorrect `PaymentTransaction`
as succeeded or failed, reusing the same finalization logic extracted
in Task 6.4.2.1.

REQUIREMENTS
- Add an `AdminOverride` audit model (or extend `StockMovement`-style
  audit logging conventions from Epic 4 — but a distinct model is
  cleaner here since this isn't specifically a stock event):
  ```python
  class PaymentAdminOverride(models.Model):
      transaction = models.ForeignKey(
          PaymentTransaction, on_delete=models.CASCADE, related_name="admin_overrides"
      )
      actor = models.ForeignKey(
          settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True, related_name="+"
      )
      previous_status = models.CharField(max_length=20)
      new_status = models.CharField(max_length=20)
      reason = models.TextField()
      created_at = models.DateTimeField(auto_now_add=True)
  ```
  Import `settings` from `django.conf` at the top if not already
  imported. Generate the migration.
- Add a custom Django admin action (NOT a general-purpose edit form) on
  `PaymentTransactionAdmin`:
  ```python
  @admin.action(description="Manually mark selected transactions as SUCCESS (requires confirmation)")
  def force_mark_success(modeladmin, request, queryset):
      # Django admin actions can't easily prompt for a free-text reason
      # inline via the standard action dropdown — implement this as an
      # intermediate confirmation page (a custom admin view/template)
      # that requires the admin to type a reason before the override is
      # applied, rather than a one-click bulk action with no
      # justification captured. Research Django's documented pattern
      # for "admin actions with an intermediate confirmation page" (it
      # involves returning an HttpResponseRedirect to a custom view from
      # the action function when no confirmation has been given yet)
      # and implement it following that standard pattern.
      ...
  ```
  The actual status-changing logic inside this action must call the
  SAME shared `finalize_transaction_success()`/`finalize_transaction_failure()`
  functions extracted in Task 6.4.2.1 (not a third independent copy of
  the mark-success/mark-failed logic), passing through the acting
  admin user and reason so a `PaymentAdminOverride` row gets created
  alongside the transaction/order state change.
  Restrict this action to staff with a specific permission (e.g. reuse
  `IsAdminOrSuperuser`-equivalent checks, or Django's built-in
  permission system via `has_permission` checks on the action itself)
  — this is a financially sensitive override capability that shouldn't
  be available to every staff user with basic admin access.
- Register `PaymentAdminOverride` in admin.py as a fully read-only
  audit log (same pattern as `StockMovement`/`StockAlertSubscription`
  from Epic 4 — no add/change/delete permission).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/payments/tests/test_admin.py:
1. A staff user with appropriate permissions using the force-success
   action on a `PENDING` transaction (with a reason provided) results
   in the transaction becoming `SUCCESS`, the order becoming
   `PROCESSING`, and a `PaymentAdminOverride` row recording the actor,
   reason, and before/after status.
2. Attempting the action without providing a reason (if your
   confirmation-page implementation requires one) is rejected and no
   state changes occur.
3. A non-privileged staff user cannot access/execute this action at
   all.
```

---

#### Task 6.4.2.3 — Refund tracking model/flow (manual, non-automated)

```
You are working in backend/payments/models.py, admin.py,
backend/dashboard/. Assume Task 6.4.2.2 is already merged.

CONTEXT
There is no way to record that an order was refunded anywhere in this
codebase. Per the backlog's own scoping note, ACTUALLY processing a
refund through the gateway's API is out of scope here (that's real
money movement requiring careful, gateway-specific implementation and
is explicitly deferred) — this task only tracks refund REQUESTS and
their status, assuming the actual money movement happens manually
through the ZarinPal/Zibal/IDPay merchant dashboard outside this
platform for now, with staff recording the outcome here afterward.

TASK
Create a `RefundRequest` model and a minimal admin workflow for
tracking refund requests and their resolution status.

REQUIREMENTS
- Add:
  ```python
  class RefundRequest(models.Model):
      class Status(models.TextChoices):
          REQUESTED = "requested", "Requested"
          APPROVED = "approved", "Approved"
          PROCESSED = "processed", "Processed (refunded outside platform)"
          REJECTED = "rejected", "Rejected"

      order = models.ForeignKey(
          "order.Order", on_delete=models.CASCADE, related_name="refund_requests"
      )
      transaction = models.ForeignKey(
          PaymentTransaction, on_delete=models.SET_NULL, null=True, blank=True,
          related_name="refund_requests",
      )
      requested_by = models.ForeignKey(
          settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True,
          related_name="refund_requests",
      )
      amount = models.DecimalField(max_digits=12, decimal_places=2)
      reason = models.TextField()
      status = models.CharField(
          max_length=20, choices=Status.choices, default=Status.REQUESTED
      )
      admin_notes = models.TextField(blank=True)
      created_at = models.DateTimeField(auto_now_add=True)
      resolved_at = models.DateTimeField(null=True, blank=True)

      class Meta:
          ordering = ["-created_at"]

      def __str__(self):
          return f"Refund request for {self.order.order_number} — {self.status}"
  ```
  Generate the migration.
- Add a CUSTOMER-facing endpoint to REQUEST a refund (this is new
  customer-facing API surface, not just admin tooling): a simple
  `POST /api/orders/{id}/refund-request/` accepting a `reason` and
  optional `amount` (defaulting to the order's full `total` if not
  specified — a partial refund is a legitimate request a customer or
  admin might want to specify), creating a `RefundRequest` with
  `status=REQUESTED`, restricted to the order's owning user and only
  for orders in a sensible state (e.g. `PROCESSING`, `SHIPPED`, or
  `DELIVERED` — not `PENDING`, which hasn't even been paid yet, and
  not already-`CANCELLED`).
- Add admin actions on a `RefundRequestAdmin` to transition status
  (`REQUESTED → APPROVED → PROCESSED`, or `REQUESTED/APPROVED →
  REJECTED`), with `resolved_at` set automatically when reaching a
  terminal status (`PROCESSED`/`REJECTED`). This is deliberately a
  MANUAL, staff-driven workflow — no automatic gateway refund API call
  happens anywhere in this task, matching the explicit out-of-scope
  note above.
- Notify the customer of status changes (if Epic 16's notification
  system already exists by this point — if not, a `# TODO: Epic 16 —
  notify customer of refund status change` comment is sufficient, don't
  build ad-hoc email sending here if the proper notification
  infrastructure is coming separately).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/payments/tests/:
1. A customer can request a refund for their own `DELIVERED` order;
   the request is rejected for a `PENDING` order (never paid) and for
   an order belonging to a different user.
2. An admin can transition a `RefundRequest` through its valid status
   states; `resolved_at` is set correctly upon reaching `PROCESSED` or
   `REJECTED`.
3. The default `amount` when not specified matches the order's `total`.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 6.1.1.1 | Create payments Django app | ☐ |
| 6.1.1.2 | Create PaymentTransaction model | ☐ |
| 6.1.1.3 | Add requests to requirements.txt | ☐ |
| 6.1.1.4 | Define PaymentGateway abstract interface | ☐ |
| 6.2.1.1 | Implement ZarinPalGateway.request_payment() | ☐ |
| 6.2.1.2 | POST /api/payments/initiate/ endpoint | ☐ |
| 6.2.1.3 | Implement ZarinPalGateway.verify_payment() | ☐ |
| 6.2.1.4 | GET /api/payments/callback/zarinpal/ endpoint | ☐ |
| 6.2.1.5 | Idempotency guard on callback endpoint | ☐ |
| 6.2.1.6 | Sandbox/test-mode config toggle | ☐ |
| 6.2.1.7 | ZarinPal integration test suite | ☐ |
| 6.3.1.1 | Implement ZibalGateway | ☐ |
| 6.3.1.2 | Implement IDPayGateway | ☐ |
| 6.3.1.3 | Gateway selection logic + fallback | ☐ |
| 6.3.1.4 | Callback routes for Zibal/IDPay | ☐ |
| 6.4.1.1 | Checkout "Pay Now" redirect flow | ☐ |
| 6.4.1.2 | Payment result page (success/failure) | ☐ |
| 6.4.1.3 | Remove fake card-entry UI | ☐ |
| 6.4.2.1 | Celery reconciliation for stuck transactions | ☐ |
| 6.4.2.2 | Admin manual payment status override | ☐ |
| 6.4.2.3 | Refund tracking model/flow (manual) | ☐ |

Once Epic 6 is fully merged, the next epic to generate prompts for is
**Epic 7 — Shipping (Iranian Carriers)**, which builds on this epic's
now-real checkout/payment flow to replace the hardcoded
`SHIPPING_COST = 9.99` constant with real Iranian carrier rate lookups
and label generation.
