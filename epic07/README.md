# Epic 7 — Shipping (Iranian Carriers) — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–6 are fully merged. Specifically relevant here, confirmed directly from the repo: `order/serializers.py` currently hardcodes `SHIPPING_COST = Decimal("9.99")` at module level, and `order/models.py`'s `Order.shipping_cost` field defaults to `9.99` — both are flat, fake constants with zero relationship to actual destination, weight, or carrier. Per Epic 1 Feature 1.1.2, this constant now lives inside `order/services/pricing.py`'s `calculate_order_totals(subtotal, discount)` function rather than inline in the serializer. Per Epic 5 Task 5.2.1.4, the `Address` model (`backend/dashboard/models.py`) already has Iran-specific `province` (choices) and `postal_code` (validated 10-digit) fields — this epic's rate lookups key off `province`/`city`, not the old generic `state`/`zip`. Per Epic 6, a `PaymentGateway` ABC + settings-driven factory pattern (`payments/gateways/base.py`, `get_payment_gateway()`) was established — this epic mirrors that exact same abstraction shape for carriers.

**A note on external carrier API specifics:** Iran Post, Tipax, SnapBox, and AloPeyk's exact endpoint URLs, authentication methods, and request/response shapes may not be something to assert confidently from training data, and may have changed. Every task involving a specific carrier explicitly instructs the implementing agent to verify current, authoritative documentation before hardcoding endpoint details — same standing caveat as Epic 6's payment gateway tasks.

---

## Phase 7.1 — Shipping Domain

### Feature 7.1.1 — Shipping Model & Rate Logic

---

#### Task 7.1.1.1 — Create `shipping` app

```
You are working in backend/. Assume Epics 1–6 are fully merged.

CONTEXT
There is no `shipping` app in this codebase. `INSTALLED_APPS` in
backend/core/settings/base.py and the URL includes in
backend/core/urls.py currently list `accounts`, `contact`, `shop`,
`cart`, `order`, `dashboard`, `chat`, `blog`, and (per Epic 6) `payments`
— shipping needs the same treatment.

TASK
Scaffold a new `shipping` Django app, registered and routed exactly
like `payments` was in Epic 6 Task 6.1.1.1, with no functional code
yet.

REQUIREMENTS
- Run `python manage.py startapp shipping` from within backend/ (or
  manually create the equivalent structure), matching the file layout
  of every other app in this project.
- Name the config class `ShippingConfig` in `shipping/apps.py`.
- Add `"shipping.apps.ShippingConfig"` to `INSTALLED_APPS` in
  backend/core/settings/base.py, placed after
  `"payments.apps.PaymentsConfig"` (shipping logically depends on
  orders/payments existing first, continuing this project's
  roughly-dependency-ordered `INSTALLED_APPS` convention).
- Create a minimal `shipping/urls.py`:
  ```python
  from django.urls import path

  app_name = "shipping"

  urlpatterns = []
  ```
- Register it in backend/core/urls.py:
  `path("api/shipping/", include("shipping.urls")),`
  positioned after the `payments.urls` include.
- Create `shipping/tests/__init__.py` as a package from the start
  (matching the convention established since Epic 1 Task 1.1.1.5 and
  continued through Epic 6's `payments/tests/`).

ACCEPTANCE CRITERIA / TESTS
- `python manage.py check` passes with no errors.
- A trivial smoke test confirming the app is registered, mirroring
  Epic 6 Task 6.1.1.1's equivalent test exactly.
```

---

#### Task 7.1.1.2 — `ShippingCarrier` model

```
You are working in backend/shipping/models.py. Assume Task 7.1.1.1 is
already merged.

CONTEXT
No representation of "a shipping carrier this platform can use" exists
anywhere. This model is the shipping-domain equivalent of Epic 6's
`PaymentTransaction.Gateway` choices, but needs to be a real DB model
(not just a `TextChoices` enum) because each carrier needs its own
admin-configurable API credentials/settings and an active/inactive
toggle independent of code deploys.

TASK
Create a `ShippingCarrier` model.

REQUIREMENTS
- Add:
  ```python
  class ShippingCarrier(models.Model):
      class Code(models.TextChoices):
          POST = "post", "Iran Post"
          TIPAX = "tipax", "Tipax"
          SNAPBOX = "snapbox", "SnapBox"
          ALOPEYK = "alopeyk", "AloPeyk"

      code = models.CharField(max_length=20, choices=Code.choices, unique=True)
      display_name = models.CharField(max_length=100)
      is_active = models.BooleanField(default=True)
      config = models.JSONField(
          default=dict, blank=True,
          help_text="Carrier-specific settings (API base URL overrides, account IDs, etc.) NOT including secret API keys — those belong in environment variables, never in this DB-stored JSON field.",
      )
      supports_same_day = models.BooleanField(
          default=False,
          help_text="Whether this carrier can offer same-day/local delivery (relevant for AloPeyk-style city-restricted couriers).",
      )
      created_at = models.DateTimeField(auto_now_add=True)
      updated_at = models.DateTimeField(auto_now=True)

      class Meta:
          ordering = ["display_name"]

      def __str__(self):
          return self.display_name
  ```
  Note the explicit doc-comment on `config` — this is a deliberate
  security boundary decision: admin-editable JSON config is fine for
  non-secret operational settings, but API keys/secrets must always
  come from environment variables (via `python-decouple`'s `config()`
  pattern, matching Epic 6's `ZARINPAL_MERCHANT_ID`-style settings),
  never stored in a DB row that's visible through the Django admin —
  make sure every subsequent task in this epic that touches carrier
  credentials respects this boundary.
- Generate the migration.
- Register in backend/shipping/admin.py as a normal, EDITABLE admin
  form (unlike the audit-log-style models in Epic 4/6 — this is
  legitimately operational configuration an admin should be able to
  toggle):
  ```python
  @admin.register(ShippingCarrier)
  class ShippingCarrierAdmin(admin.ModelAdmin):
      list_display = ("display_name", "code", "is_active", "supports_same_day")
      list_filter = ("is_active", "supports_same_day")
  ```
- Add a data migration (or a management command, if this project has
  an established seed-command convention — check
  backend/shop/management/commands/seed_shop.py from earlier epics'
  grounding for precedent) that creates the four carrier rows
  (`post`, `tipax`, `snapbox`, `alopeyk`) with `is_active=False` by
  default (a carrier shouldn't become live/selectable at checkout
  until its actual integration task in Phase 7.2 is done and an admin
  deliberately activates it — seeding them as inactive-by-default rows
  is safer than active-by-default rows with no working integration
  behind them yet).

ACCEPTANCE CRITERIA / TESTS
- Migration applies cleanly.
- Add a model test confirming a `ShippingCarrier` can be created,
  `str()` returns `display_name`, and the seed migration/command
  produces exactly the 4 expected carrier rows, all `is_active=False`.
```

---

#### Task 7.1.1.3 — `ShippingRate` model (by weight/city tiers)

```
You are working in backend/shipping/models.py. Assume Task 7.1.1.2 is
already merged.

CONTEXT
Real shipping costs vary by carrier, destination, and weight — a flat
`SHIPPING_COST = 9.99` constant (confirmed directly in
`order/serializers.py`) has no relationship to any of that. Before
wiring up live carrier APIs (Phase 7.2, which may or may not expose
real-time rate quoting depending on each carrier's actual API
capabilities), this platform needs its own rate table as a reliable
fallback/baseline — many Iranian courier services are actually priced
via published rate CARDS rather than a live quote API, so a local rate
table is likely necessary regardless of what Phase 7.2's APIs turn out
to support.

TASK
Create a `ShippingRate` model: a rate table keyed by carrier,
destination (province, with an optional more specific city override),
and a weight bracket.

REQUIREMENTS
- Add:
  ```python
  class ShippingRate(models.Model):
      carrier = models.ForeignKey(
          ShippingCarrier, on_delete=models.CASCADE, related_name="rates"
      )
      province = models.CharField(
          max_length=30, choices=IranProvince.choices,
          help_text="Destination province this rate applies to.",
      )
      city = models.CharField(
          max_length=100, blank=True,
          help_text="Optional: leave blank to apply to the whole province; set to override for a specific city (e.g. same-city rates for AloPeyk).",
      )
      min_weight_g = models.PositiveIntegerField(default=0)
      max_weight_g = models.PositiveIntegerField(
          help_text="Upper bound of this weight bracket, in grams."
      )
      price = models.DecimalField(max_digits=10, decimal_places=2)
      estimated_days_min = models.PositiveSmallIntegerField(default=1)
      estimated_days_max = models.PositiveSmallIntegerField(default=3)
      is_active = models.BooleanField(default=True)

      class Meta:
          ordering = ["carrier", "province", "min_weight_g"]
          indexes = [models.Index(fields=["carrier", "province", "city"])]

      def __str__(self):
          return f"{self.carrier.display_name} — {self.get_province_display()} ({self.min_weight_g}-{self.max_weight_g}g): {self.price}"
  ```
  Import `IranProvince` from `dashboard.models` (the enum established
  in Epic 5 Task 5.2.1.4) — check for circular-import risk between
  `shipping` and `dashboard` apps first; if `dashboard` doesn't import
  anything from `shipping` (it shouldn't at this point), a direct
  top-level import is safe. If a genuine circularity risk exists,
  consider whether `IranProvince` should actually live in a
  lower-level shared location (e.g. a small `core` app or a
  `common/choices.py` module) that both `dashboard` and `shipping` can
  import from without either depending on the other — this is worth
  raising as a design question if the circular import actually bites,
  rather than working around it with an awkward local import inside a
  function body, which is the less clean fallback option; use your
  judgment on which fix is proportionate here, given `IranProvince` is
  fundamentally a general-purpose "Iran's provinces" concept that
  arguably shouldn't be owned by the `dashboard` app specifically in
  the first place.
- Generate the migration.
- Add a `clean()` validation ensuring `max_weight_g > min_weight_g` and
  raising a clear error otherwise (mirroring the
  `expiration_date`/`manufacture_date` ordering-validation pattern
  established in Epic 3 Task 3.2.1.7).
- Register in shipping/admin.py:
  ```python
  @admin.register(ShippingRate)
  class ShippingRateAdmin(admin.ModelAdmin):
      list_display = ("carrier", "province", "city", "min_weight_g", "max_weight_g", "price", "is_active")
      list_filter = ("carrier", "province", "is_active")
      search_fields = ("city",)
  ```
- Add a `ShippingRate.objects` manager method or plain module-level
  function (your call) for looking up the applicable rate:
  ```python
  @classmethod
  def find_rate(cls, carrier, province, city, weight_g):
      """
      Find the most specific matching active rate: prefer an exact
      city match over a province-wide (blank city) rate.
      """
      qs = cls.objects.filter(
          carrier=carrier, province=province, is_active=True,
          min_weight_g__lte=weight_g, max_weight_g__gte=weight_g,
      )
      city_specific = qs.filter(city=city).first()
      if city_specific:
          return city_specific
      return qs.filter(city="").first()
  ```
  (as a `classmethod` on `ShippingRate` itself, or a standalone
  function in a new `shipping/services.py` — pick whichever fits your
  overall file organization better, but be consistent with how similar
  lookup helpers were organized in prior epics, e.g. Epic 6's
  `payments/gateways/__init__.py` factory function pattern).

ACCEPTANCE CRITERIA / TESTS
Add model tests to backend/shipping/tests/test_models.py:
1. `find_rate()` returns the correct rate for a weight that falls
   within a defined bracket.
2. `find_rate()` returns `None` when no matching rate exists (weight
   outside all defined brackets, or no rate configured for that
   province/carrier combination at all) — confirm the calling code
   (Task 7.1.1.4) handles a `None` result gracefully rather than
   crashing.
3. A city-specific rate is preferred over a province-wide rate when
   both exist and match.
4. An `is_active=False` rate is never returned by `find_rate()`.
5. `clean()` rejects a rate where `max_weight_g <= min_weight_g`.
```

---

#### Task 7.1.1.4 — Replace hardcoded `SHIPPING_COST = 9.99` with rate lookup

```
You are working in backend/order/services/pricing.py and
backend/order/serializers.py. Assume Task 7.1.1.3 is already merged.

CONTEXT — READ THE CURRENT PRICING SERVICE SIGNATURE CAREFULLY
Per Epic 1 Feature 1.1.2, `order/services/pricing.py` currently exposes
`calculate_order_totals(subtotal: Decimal, discount: Decimal = Decimal("0")) -> dict`,
which internally uses a hardcoded `SHIPPING_COST = Decimal("9.99")`
module-level constant to compute `shipping_cost` as part of its
returned totals dict. This task's job is to make shipping cost an
INPUT to this function (computed by the new rate-lookup system) rather
than an internal constant — this is a real signature change with
ripple effects into `OrderCreateSerializer`, so read through exactly
how that serializer currently calls this function before editing.

TASK
1. Change `calculate_order_totals()`'s signature to accept an explicit
   `shipping_cost: Decimal` parameter instead of computing it
   internally from a constant.
2. Add a shipping-rate-resolution step to the checkout flow: given the
   destination address and cart weight, look up the applicable
   `ShippingRate` and pass its `price` into `calculate_order_totals()`.

REQUIREMENTS
- In `order/services/pricing.py`:
  - Remove the module-level `SHIPPING_COST = Decimal("9.99")` constant
    entirely.
  - Change the function signature to:
    `calculate_order_totals(subtotal: Decimal, shipping_cost: Decimal, discount: Decimal = Decimal("0")) -> dict`
  - Update the internal math to use the passed-in `shipping_cost`
    parameter instead of the removed constant everywhere it was
    previously referenced.
  - Update this function's existing validation (from Epic 1 Task
    1.1.2.1 — negative-value checks, discount-exceeds-total checks) to
    also validate `shipping_cost >= 0`, raising the same
    `PricingError`/`ValueError` pattern already established for the
    other invalid-input cases.
- In `order/serializers.py` `OrderCreateSerializer`:
  - Add a required field: `shipping_carrier_id = serializers.IntegerField()`
    and `shipping_rate_id = serializers.IntegerField()` — the customer
    must have already selected a specific carrier+rate BEFORE
    submitting checkout (this selection happens via the quote endpoint
    built in Task 7.2.1.6's checkout UI, which calls a rate-lookup API
    endpoint you'll also need — see the note below — and the frontend
    passes the chosen `shipping_rate_id` back at submission time,
    mirroring exactly how Epic 5 Task 5.2.1.3's `address_id` pattern
    already works for saved addresses: reference an ID, resolve
    server-side, don't trust client-computed values).
  - In `validate()`, resolve the `ShippingRate` by
    `shipping_rate_id` (raise a `ValidationError` if it doesn't exist,
    is now `is_active=False`, or doesn't match the resolved shipping
    destination's province — RE-VALIDATE this server-side rather than
    trusting the client's earlier quote request matched what's being
    submitted now, following the exact same "never trust client-supplied
    pricing, always re-derive/re-validate server-side" principle
    already established for stock (Epic 5 Task 5.2.1.1) and price
    (Epic 5 Task 5.2.1.2)). Store the resolved `ShippingRate` instance
    in `attrs` for use in `create()`.
  - In `create()`, pass `shipping_cost=resolved_rate.price` into the
    `calculate_order_totals()` call (replacing whatever the call
    currently passes, per Epic 1's original two-argument
    `calculate_order_totals(subtotal=..., discount=...)` call site —
    add the new required argument).
  - Store the resolved carrier/rate on the `Order` itself: add
    `shipping_carrier = models.ForeignKey("shipping.ShippingCarrier", on_delete=models.SET_NULL, null=True, blank=True)`
    to `order/models.py`'s `Order` model (generate the migration), and
    set it in `create()`. The existing `Order.shipping_cost` field
    (already present, currently defaulting to the now-removed `9.99`
    constant) continues to store the actual resolved price as a frozen
    snapshot — change its Python-level `default=9.99` to no default at
    all (it's now always explicitly set from the resolved rate, so a
    hardcoded fallback default is no longer appropriate and could mask
    a bug if ever accidentally relied upon).
- You'll need SOME way to compute "cart weight" for the rate lookup —
  check whether `ProductVariant` has weight data (per Epic 3 Task
  3.2.1.4's `weight_g`/`volume_ml` fields — note `volume_ml` doesn't
  directly give you a shippable weight; for variants with only
  `volume_ml` set and no `weight_g`, you'll need a reasonable
  fallback, e.g. treating each `volume_ml` unit as roughly 1 gram for
  liquid products as a rough approximation, or defaulting to a
  conservative flat per-item weight assumption like 100g when NEITHER
  is set — document whichever heuristic you choose clearly in a code
  comment as an approximation, since exact packaging/product weight
  data is inherently imperfect at this stage of the project, and this
  is a reasonable, explicitly-flagged simplification rather than a
  silent inaccuracy). Compute total cart weight as
  `sum(item.variant.weight_g_or_estimate() * item.quantity for item in cart.items.all())`
  — implement `weight_g_or_estimate()` as a small helper method/property
  on `ProductVariant` encapsulating this fallback logic in one place.

ACCEPTANCE CRITERIA / TESTS
- Update EVERY existing test across Epics 1/3/5/6 that calls
  `calculate_order_totals()` directly (the pricing-service unit tests
  from Epic 1 Task 1.1.2.3 in particular) to pass an explicit
  `shipping_cost` argument instead of relying on the now-removed
  constant — this is a deliberate, expected breaking change to that
  function's test suite, not a regression to silently patch around.
- Add new tests:
  1. Checkout with a valid `shipping_rate_id` matching the destination
     province results in `Order.shipping_cost` exactly matching that
     rate's `price`, and `Order.shipping_carrier` correctly set.
  2. Checkout with a `shipping_rate_id` that exists but belongs to a
     DIFFERENT province than the submitted shipping address is
     rejected with a 400 (the re-validation check).
  3. Checkout with a `shipping_rate_id` referencing an `is_active=False`
     rate is rejected.
  4. Re-run the full order test suite (all prior epics' checkout tests)
     with the new required fields added to their test payloads, and
     confirm total calculations remain otherwise correct.
```

---

## Phase 7.2 — Carrier Integrations

### Feature 7.2.1 — Carrier APIs

---

#### Task 7.2.1.1 — Define `CarrierProvider` interface

```
You are working in backend/shipping/providers/ (new package). Assume
Feature 7.1.1 is fully merged.

CONTEXT
Mirrors Epic 6 Task 6.1.1.4's `PaymentGateway` abstraction exactly, one
layer over in the shipping domain: four different carriers (Post,
Tipax, SnapBox, AloPeyk) each need their own concrete implementation,
while the rest of the codebase should work with "whichever carrier is
configured" generically.

TASK
Define an abstract `CarrierProvider` base class with `get_rate()`,
`create_shipment()`, and `track()` methods, plus a settings-driven
factory, following the exact same structural pattern as Epic 6's
`PaymentGateway`/`get_payment_gateway()`.

REQUIREMENTS
- Create `backend/shipping/providers/__init__.py` and
  `backend/shipping/providers/base.py`:
  ```python
  from abc import ABC, abstractmethod
  from dataclasses import dataclass
  from decimal import Decimal


  @dataclass
  class RateQuoteResult:
      success: bool
      price: Decimal = Decimal("0")
      estimated_days_min: int = 0
      estimated_days_max: int = 0
      error_message: str = ""


  @dataclass
  class ShipmentCreateResult:
      success: bool
      tracking_number: str = ""
      label_url: str = ""
      error_message: str = ""


  @dataclass
  class TrackingResult:
      success: bool
      status: str = ""
      status_description: str = ""
      error_message: str = ""


  class CarrierProvider(ABC):
      @abstractmethod
      def get_rate(self, origin: dict, destination: dict, weight_g: int) -> RateQuoteResult:
          """Get a live rate quote from the carrier's API, if supported."""
          raise NotImplementedError

      @abstractmethod
      def create_shipment(self, order, destination: dict) -> ShipmentCreateResult:
          """Book an actual shipment with the carrier and get a tracking number/label."""
          raise NotImplementedError

      @abstractmethod
      def track(self, tracking_number: str) -> TrackingResult:
          """Get current delivery status for a tracking number."""
          raise NotImplementedError
  ```
  — note `get_rate()` returning `success=False` is a VALID, expected
  outcome for carriers whose API doesn't support live quoting at all
  (many Iranian courier services are priced via the static rate-card
  table from Task 7.1.1.3, not a live API) — concrete implementations
  for such carriers should raise `NotImplementedError` OR return
  `RateQuoteResult(success=False, error_message="Live quoting not supported; use rate table.")`
  rather than being forced to fake a live quote; document this
  explicitly on the ABC method's docstring so implementers understand
  it's an acceptable, anticipated outcome, not a bug.
- Add the factory function in `providers/__init__.py`, structurally
  identical to Epic 6's `get_payment_gateway()`:
  ```python
  from django.core.exceptions import ImproperlyConfigured
  from django.utils.module_loading import import_string
  from .base import CarrierProvider


  def get_carrier_provider(code: str) -> CarrierProvider:
      from django.conf import settings
      provider_classes = settings.CARRIER_PROVIDER_CLASSES
      if code not in provider_classes:
          raise ImproperlyConfigured(f"Unknown carrier provider: {code}")
      provider_class = import_string(provider_classes[code])
      instance = provider_class()
      if not isinstance(instance, CarrierProvider):
          raise ImproperlyConfigured(f"{provider_classes[code]} does not implement CarrierProvider")
      return instance
  ```
- Add to backend/core/settings/base.py:
  ```python
  CARRIER_PROVIDER_CLASSES = {
      # populated incrementally as each carrier is implemented in
      # Tasks 7.2.1.2 through 7.2.1.5
  }
  ```

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shipping/tests/test_providers.py mirroring Epic 6
Task 6.1.1.4's equivalent tests exactly (unknown-provider error,
ABC-instantiation-fails, a minimal dummy test-only subclass proves the
contract works).
```

---

#### Task 7.2.1.2 — Implement Post (Iran Post) provider

```
You are working in backend/shipping/providers/post.py (new file).
Assume Task 7.2.1.1 is already merged. Same "verify current API docs
before hardcoding" caveat as every carrier/gateway task in this and the
prior epic applies here in full — Iran Post's actual API availability,
authentication, and endpoint shapes need to be confirmed against
current, authoritative documentation before implementation; do not
assume any specific endpoint URL or payload shape is accurate without
verifying it.

CONTEXT
Iran Post (Post-e Iran) is a national postal service; depending on what
API access is actually available to a business integrating with them
(this may require a business account/contract, not just a public API
key — verify what's actually accessible before assuming a fully
self-service API integration is even possible), rate quoting may only
be available via the static rate-card table (Task 7.1.1.3) rather than
a live API, while shipment creation/label generation and tracking may
have real API support.

TASK
Implement `PostCarrierProvider(CarrierProvider)`.

REQUIREMENTS
- Add settings:
  `IRAN_POST_API_KEY = config("IRAN_POST_API_KEY", default="")`
  (adjust the exact credential shape — API key, account number,
  username/password — to match whatever Iran Post's actual
  documented integration mechanism turns out to be).
- Implement `get_rate()`: if live quoting isn't actually available per
  your documentation research, implement it to explicitly return
  `RateQuoteResult(success=False, error_message="Iran Post rate quoting uses the static rate table.")`
  rather than attempting to fake a network call — don't build a
  nonfunctional stub that pretends to call an API that doesn't exist
  for this purpose; be honest in the implementation about what's
  actually supported.
- Implement `create_shipment()` and `track()` with the same
  error-handling/timeout discipline established in Epic 6's gateway
  implementations (`try/except (requests.RequestException, ValueError)`,
  `timeout=15`, never raising an unhandled exception, returning the
  appropriate dataclass result type with `success=False` and a
  populated `error_message` on any failure) — IF a real API exists for
  these operations; if genuinely no programmatic API exists for
  shipment creation/tracking either (fully manual/portal-based
  fulfillment is common for some carriers), implement these as
  explicit `NotImplementedError`-raising stubs with a clear docstring
  explaining that Iran Post fulfillment for this integration is manual
  (an admin manually enters a tracking number after booking through
  Iran Post's own portal/counter service) rather than API-driven — this
  is a legitimate, expected outcome for this specific carrier, not a
  failure to complete the task; document your findings clearly in your
  task summary regardless of which case applies.
- Add to `CARRIER_PROVIDER_CLASSES` in settings:
  `"post": "shipping.providers.post.PostCarrierProvider",`

ACCEPTANCE CRITERIA / TESTS
Add backend/shipping/tests/test_post.py covering whichever methods
actually have real API-calling logic (mocked, never real network
calls) following Epic 6's gateway test structure — and for any method
implemented as an explicit `NotImplementedError` stub instead, add a
test confirming it raises that clearly rather than silently doing
nothing or crashing unpredictably.
```

---

#### Task 7.2.1.3 — Implement Tipax provider

```
You are working in backend/shipping/providers/tipax.py (new file).
Assume Task 7.2.1.2 is merged (for the established pattern, including
the "some methods may legitimately be unsupported/manual" precedent it
set). Same standing API-verification caveat applies.

CONTEXT
Tipax is a widely-used Iranian courier/logistics company with (per
general knowledge, to be verified) a more modern, developer-facing API
than some alternatives — but confirm this rather than assuming.

TASK
Implement `TipaxCarrierProvider(CarrierProvider)` following the exact
same structural pattern as `PostCarrierProvider`.

REQUIREMENTS
- Add settings: `TIPAX_API_KEY = config("TIPAX_API_KEY", default="")`
  (adjust per verified actual auth mechanism).
- Implement all three interface methods with real API calls where
  Tipax's documented API actually supports them, or explicit
  `NotImplementedError` stubs with clear docstrings where it doesn't —
  same honesty principle as Task 7.2.1.2.
- Add to `CARRIER_PROVIDER_CLASSES`:
  `"tipax": "shipping.providers.tipax.TipaxCarrierProvider",`

ACCEPTANCE CRITERIA / TESTS
Add backend/shipping/tests/test_tipax.py mirroring test_post.py's
structure and rigor.
```

---

#### Task 7.2.1.4 — Implement SnapBox provider

```
You are working in backend/shipping/providers/snapbox.py (new file).
Assume Task 7.2.1.3 is merged. Same standing caveat applies.

CONTEXT
SnapBox (part of the Snapp ecosystem) is a locker/pickup-point-oriented
delivery service in Iran — its integration shape may differ
meaningfully from a traditional door-to-door courier (e.g. shipment
creation might need to specify a destination LOCKER/pickup point ID
rather than just a street address) — verify this against current
documentation rather than assuming it works identically to Post/Tipax.

TASK
Implement `SnapBoxCarrierProvider(CarrierProvider)`.

REQUIREMENTS
- Add settings: `SNAPBOX_API_KEY = config("SNAPBOX_API_KEY", default="")`.
- If SnapBox's actual model is locker/pickup-point-based rather than
  door-to-door, this is a genuine interface-fit question worth
  surfacing rather than silently forcing it into the existing
  `create_shipment(order, destination: dict)` shape: consider whether
  `destination` needs an optional `pickup_point_id` key that this
  provider specifically requires and others ignore, documented clearly
  as a SnapBox-specific extension to the generic interface — don't
  redesign the shared `CarrierProvider` ABC for one carrier's
  peculiarity if a small documented extension to the existing `dict`
  parameter shape is sufficient; only escalate to changing the shared
  interface itself if genuinely no reasonable accommodation fits within
  it, and flag that decision clearly if you make it.
- Implement all three methods with the same real-API-or-honest-stub
  discipline as prior carrier tasks.
- Add to `CARRIER_PROVIDER_CLASSES`:
  `"snapbox": "shipping.providers.snapbox.SnapBoxCarrierProvider",`

ACCEPTANCE CRITERIA / TESTS
Add backend/shipping/tests/test_snapbox.py mirroring the established
structure, including a test for whatever locker/pickup-point handling
you implemented if applicable.
```

---

#### Task 7.2.1.5 — Implement AloPeyk provider (same-day/local)

```
You are working in backend/shipping/providers/alopeyk.py (new file).
Assume Task 7.2.1.4 is merged. Same standing caveat applies.

CONTEXT
AloPeyk is a same-day/local courier service — critically, it is NOT
available nationwide the way Post/Tipax broadly are; it's restricted to
specific cities where the service operates. `ShippingCarrier.supports_same_day`
(Task 7.1.1.2) already exists as a flag for this, but this task needs
to add the actual CITY-RESTRICTION logic, not just a same-day flag —
selecting AloPeyk for a destination city it doesn't serve must be
prevented before checkout, not discovered as a failure only when
attempting to book the shipment.

TASK
Implement `AloPeykCarrierProvider(CarrierProvider)`, including explicit
service-area/city-availability checking.

REQUIREMENTS
- Add settings: `ALOPEYK_API_KEY = config("ALOPEYK_API_KEY", default="")`.
- Add a method beyond the base `CarrierProvider` interface (this is a
  genuine AloPeyk-specific capability, appropriately added as an
  extra method on the concrete class rather than forced into the
  shared ABC, since no other carrier in this epic needs it):
  `def is_available_for_city(self, city: str, province: str) -> bool`
  — verify AloPeyk's actual documented list of served cities (likely
  major cities like Tehran, Mashhad, Isfahan, etc. — confirm the real,
  current list rather than guessing) and implement this check either
  against a hardcoded list of served cities (simplest, and reasonable
  if AloPeyk's service area doesn't change often) or a real API call
  if their API exposes a serviceability-check endpoint (preferred if
  available, since it stays accurate without requiring code changes
  when their service area expands).
- In `get_rate()`, if `is_available_for_city()` returns `False` for the
  requested destination, return
  `RateQuoteResult(success=False, error_message="AloPeyk is not available in this city.")`
  rather than attempting a quote request that would just fail anyway.
- Implement `create_shipment()`/`track()` with the same real-API-or-
  honest-stub discipline as prior tasks.
- Add to `CARRIER_PROVIDER_CLASSES`:
  `"alopeyk": "shipping.providers.alopeyk.AloPeykCarrierProvider",`
- IMPORTANT follow-through: the rate-lookup flow built in Task 7.1.1.4
  (`ShippingRate.find_rate()`) and the checkout carrier-selection UI in
  Task 7.2.1.6 both need to respect this city restriction too — a
  customer in a city AloPeyk doesn't serve should never even SEE it as
  a selectable option. Since Task 7.1.1.4's `find_rate()` already only
  returns matches from the `ShippingRate` table (which an admin
  presumably only populates with AloPeyk rates for cities it actually
  serves), this may already be correctly handled by the DATA (no
  AloPeyk `ShippingRate` rows exist for unserved cities) rather than
  needing additional CODE-level filtering — confirm this is a
  sufficient safeguard given how the rate table is expected to be
  populated, or add an explicit code-level check in the quote endpoint
  (Task 7.2.1.6) if you judge the data-only safeguard too fragile
  (e.g. an admin could mistakenly add an AloPeyk rate for an unserved
  city) — your call, but make a deliberate decision and document it.

ACCEPTANCE CRITERIA / TESTS
Add backend/shipping/tests/test_alopeyk.py:
1. `is_available_for_city()` returns `True` for a known-served city and
   `False` for one not served.
2. `get_rate()` for an unserved city returns `success=False` with the
   appropriate message WITHOUT attempting an actual API call (mock and
   assert the mock was never invoked).
3. Standard mocked-success/failure/network-error coverage for whichever
   methods have real API implementations, matching prior carrier tasks'
   test rigor.
```

---

#### Task 7.2.1.6 — Carrier selection UI in checkout

```
You are working in backend/shipping/views.py, urls.py, and frontend/src
(Checkout page). Assume Feature 7.1.1 and Task 7.2.1.1 (interface) are
merged — individual carrier provider implementations (Tasks 7.2.1.2–
7.2.1.5) are NOT a hard dependency for this task, since this task's
primary rate SOURCE is the `ShippingRate` static table (Task 7.1.1.3),
not live carrier APIs; live quoting, if any carrier ends up supporting
it, is an enhancement layered on top, not a blocker for this task's
core functionality.

CONTEXT
Task 7.1.1.4 made `OrderCreateSerializer` REQUIRE a
`shipping_carrier_id`/`shipping_rate_id` at submission time, but
there's currently no endpoint for the frontend to DISCOVER which
carriers/rates are even available for a given destination before the
customer reaches that point — this task builds that discovery/quote
endpoint and the checkout UI that consumes it.

TASK
Add a `POST /api/shipping/quote/` endpoint returning available
carrier+rate options for a destination and cart weight, and a checkout
UI component letting the customer pick one.

REQUIREMENTS — backend
- Create `ShippingQuoteSerializer` in backend/shipping/serializers.py:
  ```python
  class ShippingQuoteSerializer(serializers.Serializer):
      province = serializers.ChoiceField(choices=IranProvince.choices)
      city = serializers.CharField(max_length=100)
  ```
  (import `IranProvince` per the same source/circularity consideration
  as Task 7.1.1.3).
- Create `ShippingQuoteView` in backend/shipping/views.py:
  ```python
  class ShippingQuoteView(APIView):
      permission_classes = [AllowAny]  # must work for guest checkout too, per Epic 5

      def post(self, request):
          serializer = ShippingQuoteSerializer(data=request.data)
          serializer.is_valid(raise_exception=True)
          province = serializer.validated_data["province"]
          city = serializer.validated_data["city"]

          cart = get_or_create_cart(request)  # reuse Epic 5's cart resolution helper
          weight_g = sum(
              item.variant.weight_g_or_estimate() * item.quantity
              for item in cart.items.select_related("variant").all()
          )

          options = []
          for carrier in ShippingCarrier.objects.filter(is_active=True):
              rate = ShippingRate.find_rate(carrier, province, city, weight_g)
              if rate:
                  options.append({
                      "carrier_id": carrier.id,
                      "carrier_name": carrier.display_name,
                      "rate_id": rate.id,
                      "price": str(rate.price),
                      "estimated_days_min": rate.estimated_days_min,
                      "estimated_days_max": rate.estimated_days_max,
                  })
          return Response({"options": options}, status=status.HTTP_200_OK)
  ```
  Import `get_or_create_cart` from `cart.views` (the Epic 5 Task
  5.1.1.2 helper) — check for circular import risk between `shipping`
  and `cart` apps; if present, resolve via the same lazy-import
  approach discussed in Task 7.1.1.3's `IranProvince` consideration.
- Register the URL: `path("quote/", views.ShippingQuoteView.as_view(), name="quote"),`
  in backend/shipping/urls.py.

REQUIREMENTS — frontend
- Add to frontend/src/services/api.js:
  ```javascript
  export const getShippingQuote = (province, city) =>
    api.post('/shipping/quote/', { province, city });
  ```
- In Checkout.jsx: once the customer has entered/selected a destination
  province+city (this likely needs to trigger AFTER those specific
  fields are filled in the address form, before final submission —
  wire it to fire on blur/change of the city field, or via an explicit
  "Get shipping options" step, whichever fits the existing form flow
  better), call `getShippingQuote()` and render the returned `options`
  as a radio-button list (carrier name, price formatted per whatever
  currency-formatting utility exists from the project's localization
  work, estimated delivery window). Store the selected option's
  `carrier_id`/`rate_id` in checkout form state, and include them as
  `shipping_carrier_id`/`shipping_rate_id` in the final `createOrder()`
  payload (per Task 7.1.1.4's backend contract).
- Handle the empty-options case (no rate configured for that
  destination/weight combination at all) with a clear message rather
  than a silently empty/broken picker — e.g. "Shipping is not currently
  available for this address; please contact support" with a way to
  go back and try a different address.
- Disable/block the final checkout submit button until a shipping
  option has been selected (mirroring how other required checkout
  fields are already validated before submission, per the existing
  `validate()` function pattern in Checkout.jsx).

ACCEPTANCE CRITERIA / TESTS
Add backend tests to backend/shipping/tests/test_views.py:
1. A quote request for a province/city with matching active rates
   returns all matching carrier options, correctly reflecting cart
   weight-based bracket selection.
2. A quote request for a destination with NO matching rates returns an
   empty `options` list (200, not an error — an empty result is a
   valid, expected outcome the frontend must handle gracefully).
3. An inactive `ShippingCarrier` never appears in results even if it
   has matching active `ShippingRate` rows (the carrier-level
   `is_active` filter takes precedence).
4. Works correctly for BOTH authenticated and guest/anonymous carts
   (per the `AllowAny` + `get_or_create_cart(request)` pattern,
   confirming Epic 5's guest-cart support is respected here too).
Add frontend component tests for the shipping-option picker covering:
renders options correctly, selecting one updates form state, empty-
options case shows the fallback message, and submit is blocked until a
selection is made.
```

---

### Feature 7.2.2 — Shipment Tracking

---

#### Task 7.2.2.1 — `Shipment` model linking order to carrier tracking number

```
You are working in backend/shipping/models.py. Assume Feature 7.2.1 is
merged.

CONTEXT
Once an order is paid (Epic 6) and ready to ship, there's currently no
record connecting that order to an actual carrier tracking number —
`Order.shipping_carrier` (Task 7.1.1.4) records WHICH carrier was
selected/quoted at checkout time, but not the actual shipment/tracking
number once it's physically booked and dispatched (which typically
happens later, when a warehouse/admin actually packs and ships the
order, not at the moment of payment).

TASK
Create a `Shipment` model.

REQUIREMENTS
- Add:
  ```python
  class Shipment(models.Model):
      class Status(models.TextChoices):
          PENDING = "pending", "Pending Pickup"
          IN_TRANSIT = "in_transit", "In Transit"
          OUT_FOR_DELIVERY = "out_for_delivery", "Out for Delivery"
          DELIVERED = "delivered", "Delivered"
          FAILED = "failed", "Failed"

      order = models.OneToOneField(
          "order.Order", on_delete=models.CASCADE, related_name="shipment"
      )
      carrier = models.ForeignKey(ShippingCarrier, on_delete=models.PROTECT)
      tracking_number = models.CharField(max_length=100, blank=True, db_index=True)
      label_url = models.URLField(blank=True)
      status = models.CharField(max_length=20, choices=Status.choices, default=Status.PENDING)
      last_tracked_at = models.DateTimeField(null=True, blank=True)
      created_at = models.DateTimeField(auto_now_add=True)
      updated_at = models.DateTimeField(auto_now=True)

      class Meta:
          ordering = ["-created_at"]

      def __str__(self):
          return f"Shipment for {self.order.order_number} — {self.status}"
  ```
  Note `carrier` uses `on_delete=models.PROTECT` (not `CASCADE` or
  `SET_NULL`) — deliberately different from most other FKs in this
  codebase: a `ShippingCarrier` should never be DELETABLE while any
  `Shipment` still references it (deleting the carrier would corrupt
  historical shipment records in a way that's worse than just
  preventing the deletion outright) — this is a genuine, deliberate
  choice worth calling out, not an oversight; if an admin genuinely
  needs to retire a carrier, `ShippingCarrier.is_active=False` is the
  correct mechanism, not deletion.
  `order` is a `OneToOneField` (not `ForeignKey`) since exactly one
  shipment record exists per order in this initial implementation (no
  support for split/multi-package shipments per order yet — note this
  as a known simplification in a comment, since a future "split
  shipment" feature would need to change this to a `ForeignKey`).
- Generate the migration.
- Register in shipping/admin.py:
  ```python
  @admin.register(Shipment)
  class ShipmentAdmin(admin.ModelAdmin):
      list_display = ("order", "carrier", "tracking_number", "status", "last_tracked_at")
      list_filter = ("carrier", "status")
      search_fields = ("order__order_number", "tracking_number")
  ```
- Add a way for an admin to BOOK the shipment (create the `Shipment`
  row and call the carrier's `create_shipment()`), likely as a custom
  admin action on `OrderAdmin` (check `backend/order/admin.py` for the
  existing `OrderAdmin` registration, or `dashboard/views.py`'s admin
  order-management endpoints from Epic 8's future work if that's
  already landed by this point — otherwise add a straightforward admin
  action here in `shipping/admin.py` or a new `ShipmentAdmin` action
  for now, and note that a fuller "book shipment" button integrated
  into the admin ORDER management UI is Epic 8's territory once that
  epic's admin order-operations tasks exist):
  ```python
  @admin.action(description="Book shipment with carrier")
  def book_shipment(modeladmin, request, queryset):
      for shipment in queryset.filter(tracking_number=""):
          provider = get_carrier_provider(shipment.carrier.code)
          result = provider.create_shipment(
              order=shipment.order,
              destination={
                  "address": shipment.order.shipping_address,
                  "city": shipment.order.shipping_city,
                  "province": shipment.order.shipping_state,
                  "postal_code": shipment.order.shipping_zip,
              },
          )
          if result.success:
              shipment.tracking_number = result.tracking_number
              shipment.label_url = result.label_url
              shipment.status = Shipment.Status.PENDING
              shipment.save()
          else:
              modeladmin.message_user(
                  request, f"Failed to book shipment for {shipment.order.order_number}: {result.error_message}",
                  level="ERROR",
              )
  ```

ACCEPTANCE CRITERIA / TESTS
Add model tests confirming a `Shipment` can be created linked to an
`Order`, `str()` works, and the `OneToOneField` constraint prevents a
second `Shipment` from being created for the same order (assert
`IntegrityError` on a duplicate attempt). Add an admin-action test
(mocking `get_carrier_provider`/`create_shipment`) confirming a
successful booking populates `tracking_number`/`label_url` and a
failed one leaves them blank with an error message shown.
```

---

#### Task 7.2.2.2 — Celery task polling tracking status

```
You are working in backend/shipping/tasks.py (new file). Assume Task
7.2.2.1 is merged and Celery infrastructure exists (same Epic 22
dependency caveat as every Celery task in prior epics — confirm before
starting).

CONTEXT
Once a `Shipment` has a `tracking_number`, nothing currently keeps its
`status` up to date — a customer-facing tracking widget (Task 7.2.2.3)
needs reasonably fresh status data, and the order's own fulfillment
status (per Epic 8's future order-lifecycle work) may need to advance
automatically when a shipment is actually delivered.

TASK
Add a periodic Celery task polling each carrier's `track()` method for
every non-terminal `Shipment` and updating its status.

REQUIREMENTS
- Implement in backend/shipping/tasks.py:
  ```python
  from celery import shared_task

  @shared_task
  def poll_shipment_tracking():
      from django.utils import timezone
      from .models import Shipment
      from .providers import get_carrier_provider

      active_shipments = Shipment.objects.exclude(
          status__in=[Shipment.Status.DELIVERED, Shipment.Status.FAILED]
      ).exclude(tracking_number="").select_related("carrier", "order")

      updated_count = 0
      for shipment in active_shipments:
          provider = get_carrier_provider(shipment.carrier.code)
          result = provider.track(shipment.tracking_number)
          if not result.success:
              continue  # transient failure, try again next run — don't mark FAILED just because one poll failed
          if result.status and result.status != shipment.status:
              shipment.status = result.status
              shipment.last_tracked_at = timezone.now()
              shipment.save(update_fields=["status", "last_tracked_at"])
              updated_count += 1
              if shipment.status == Shipment.Status.DELIVERED:
                  # Advance the order's own status too, once Epic 8's
                  # order-lifecycle state machine exists — for now, if
                  # Order.Status already has a DELIVERED value (it does,
                  # per the original model), set it directly here.
                  order = shipment.order
                  order.status = order.Status.DELIVERED
                  order.save(update_fields=["status"])
      return f"Updated {updated_count} shipment(s)."
  ```
  Note: `result.status` from a carrier's `track()` implementation
  (Task 7.2.1.2–7.2.1.5) needs to be normalized into one of THIS
  platform's `Shipment.Status` choice values, not just passed through
  raw carrier-specific status strings — each concrete
  `CarrierProvider.track()` implementation should be responsible for
  mapping the carrier's own status vocabulary to this platform's
  `Shipment.Status` values before returning `TrackingResult`, so this
  polling task can stay carrier-agnostic. If this mapping wasn't
  already built into each provider's `track()` implementation back in
  Phase 7.2.1's tasks, that's a gap worth going back and fixing in
  those provider files as part of this task, rather than trying to
  normalize carrier-specific statuses here in the generic polling
  task, which would defeat the purpose of the abstraction.
- Register as a periodic task (every 30-60 minutes is a reasonable
  polling interval for shipment tracking — not real-time-critical the
  way payment reconciliation was in Epic 6 Task 6.4.2.1), using the
  same `django-celery-beat` mechanism established in prior epics.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shipping/tests/test_tasks.py:
1. A shipment whose mocked `track()` call returns a NEW status updates
   the `Shipment` row and sets `last_tracked_at`.
2. A shipment whose mocked `track()` call returns the SAME status as
   currently recorded does not spuriously re-save (or does, but is
   idempotent either way — decide and test whichever behavior you
   implemented explicitly).
3. A shipment transitioning to `DELIVERED` also updates the linked
   `Order.status` to `DELIVERED`.
4. A `Shipment` already in a terminal status (`DELIVERED`/`FAILED`) is
   excluded from polling entirely (mock `get_carrier_provider`/`track`
   and assert it's never called for such shipments).
5. A mocked `track()` failure (`success=False`) leaves the shipment
   untouched and doesn't raise, allowing the next scheduled run to
   retry.
```

---

#### Task 7.2.2.3 — Customer-facing tracking widget on order detail page

```
You are working in frontend/src (order detail/account page) and
backend/order/serializers.py. Assume Task 7.2.2.1/7.2.2.2 are merged.

CONTEXT
Customers currently have no visibility into shipment status on their
order detail page — `OrderSerializer` (check its current `Meta.fields`
in order/serializers.py) doesn't include any shipment-related data at
all, since the `Shipment` model didn't exist before this epic.

TASK
Expose shipment status via the existing order API, and build a
tracking-status widget on the frontend order detail page.

REQUIREMENTS — backend
- Add a nested `ShipmentSerializer` (in backend/shipping/serializers.py)
  exposing `carrier` (display name), `tracking_number`, `status`
  (display value, not raw code), `last_tracked_at`:
  ```python
  class ShipmentSerializer(serializers.ModelSerializer):
      carrier_name = serializers.CharField(source="carrier.display_name", read_only=True)
      status_display = serializers.CharField(source="get_status_display", read_only=True)

      class Meta:
          model = Shipment
          fields = ("carrier_name", "tracking_number", "status", "status_display", "last_tracked_at")
  ```
- In `order/serializers.py`'s `OrderSerializer`, add:
  `shipment = serializers.SerializerMethodField()`
  ```python
  def get_shipment(self, obj):
      shipment = getattr(obj, "shipment", None)
      if shipment is None:
          return None
      from shipping.serializers import ShipmentSerializer
      return ShipmentSerializer(shipment).data
  ```
  (using a local import inside the method, rather than a top-level
  import of `shipping.serializers` into `order/serializers.py`, to
  avoid introducing a hard module-level circular-import risk between
  the two apps — this is a common, acceptable Django pattern for
  cross-app serializer composition where a top-level import would be
  circular; confirm whether it's actually necessary here or whether a
  top-level import works fine, and only fall back to the local-import
  pattern if genuinely needed).
  Add `"shipment"` to `OrderSerializer.Meta.fields`.

REQUIREMENTS — frontend
- On the order detail page (find the existing component — likely part
  of `Account.jsx` or a dedicated order-detail page/route), render a
  shipment-status section when `order.shipment` is present: carrier
  name, current status (as a friendly label, using `status_display`,
  possibly with a simple visual step-indicator: Pending → In Transit →
  Out for Delivery → Delivered), tracking number, and — if a real
  carrier tracking URL pattern exists (many carriers have a public
  "track your package" webpage that accepts a tracking number as a URL
  parameter) — a link to track directly on the carrier's own site
  (research whether each of the four carriers has such a public
  tracking page URL pattern; if so, construct it; if not, just display
  the raw tracking number as plain text).
  When `order.shipment` is `null` (order hasn't been booked/shipped
  yet), show an appropriate "Not yet shipped" state rather than an
  empty/broken section.

ACCEPTANCE CRITERIA / TESTS
Add backend serializer tests confirming `OrderSerializer` correctly
includes shipment data when present and `null` when absent. Add
frontend component tests for the tracking widget covering: renders
correctly with a shipment present at each status value, renders the
"not yet shipped" state correctly when `shipment` is null.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 7.1.1.1 | Create shipping Django app | ☐ |
| 7.1.1.2 | ShippingCarrier model | ☐ |
| 7.1.1.3 | ShippingRate model (weight/city tiers) | ☐ |
| 7.1.1.4 | Replace hardcoded SHIPPING_COST with rate lookup | ☐ |
| 7.2.1.1 | Define CarrierProvider abstract interface | ☐ |
| 7.2.1.2 | Implement Post (Iran Post) provider | ☐ |
| 7.2.1.3 | Implement Tipax provider | ☐ |
| 7.2.1.4 | Implement SnapBox provider | ☐ |
| 7.2.1.5 | Implement AloPeyk provider (same-day/local) | ☐ |
| 7.2.1.6 | Carrier selection UI in checkout | ☐ |
| 7.2.2.1 | Shipment model linking order to tracking number | ☐ |
| 7.2.2.2 | Celery task polling tracking status | ☐ |
| 7.2.2.3 | Customer-facing tracking widget | ☐ |

Once Epic 7 is fully merged, the next epic to generate prompts for is
**Epic 8 — Order Management (Admin Side)**, which builds admin order-
lifecycle operations (status transitions, search/filter, export,
invoicing) directly on top of the now-real payment and shipping data
this epic and Epic 6 established.
