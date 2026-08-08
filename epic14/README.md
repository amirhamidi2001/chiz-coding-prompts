# Epic 14 — Persian Localization & i18n — AI Coding Prompts

Repo: `tablogenix-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next. This is one of the largest, most cross-cutting epics in the whole backlog — it touches settings, every monetary field in the system, every date, the entire frontend layout direction, and the URL/slug strategy. Budget accordingly; several tasks here are genuinely harder and higher-risk than a typical 30-min–2-hr task despite being sized that way in the original backlog, particularly Task 14.1.2.2's currency migration.

**Assumed prerequisites:** Epics 1–13 are fully merged — meaning by this point the codebase has substantial `Decimal`-based financial logic spread across `order`, `payments`, `shipping`, and `promotions` (built across Epics 1, 6, 7, and 9 respectively). This epic's currency-unit change (Task 14.1.2.2) is genuinely the highest-blast-radius task in this entire backlog so far — it touches more files than any single task in any prior epic. The master backlog's own execution-order notes explicitly warned this should ideally happen EARLY, before more monetary fields pile up — by Epic 14 that ship has sailed, so this epic pays that accumulated cost. Take Task 14.1.2.1/14.1.2.2 seriously and do not rush them.

**Two real, confirmed bugs this epic must fix, found directly in the repo:**
1. `backend/core/settings/base.py` currently has `LANGUAGE_CODE = "en-us"` and `TIME_ZONE = "UTC"` — a Persian-market platform running in English/UTC by default.
2. `Product.save()` (and `Category`/`Brand`'s equivalent `save()` methods) call Django's default `slugify(self.name)` **without `allow_unicode=True`**. Django's default `slugify()` strips all non-ASCII characters — meaning a Persian product name like `"سرم ویتامین C"` currently produces an effectively **empty or garbage slug** (whatever ASCII characters happen to survive, likely just `"c"`), not a meaningful URL segment. This is a live, currently-shipping bug for any Persian-named product already in the catalog by this point in the project, and Task 14.4.1.2 fixes it.

---

## Phase 14.1 — Backend i18n Foundation

### Feature 14.1.1 — Django i18n Setup

---

#### Task 14.1.1.1 — Set `LANGUAGE_CODE = "fa"`, configure `LANGUAGES`

```
You are working in backend/core/settings/base.py. Assume Epics 1–13
are fully merged.

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
`LANGUAGE_CODE = "en-us"` currently. This is the first, most basic
localization fix and intentionally the smallest task in this epic —
larger structural changes (RTL, currency, dates) are separate tasks.

TASK
Set the project's default language to Persian and declare the
supported languages list.

REQUIREMENTS
- Change:
  ```python
  LANGUAGE_CODE = "fa"
  LANGUAGES = [
      ("fa", "Persian"),
      ("en", "English"),  # kept available for admin/staff convenience and any future i18n expansion
  ]
  ```
  Keeping English in `LANGUAGES` (even though the site defaults to
  Persian) is deliberate — Django's admin interface itself has decent
  built-in translations for many languages including Persian, but
  keeping English available as an option costs nothing and is useful
  for any English-speaking developer/support staff who might prefer
  it, without requiring a whole separate task to add it later if
  needed.
- Confirm `USE_I18N = True` remains set (it already is, per the repo)
  — this flag being `True` is what makes `LANGUAGE_CODE`/`LANGUAGES`
  actually take effect at all; don't accidentally disable it.
- This task does NOT yet add any actual Persian translation strings
  (`.po`/`.mo` files) — that's Task 14.1.1.4. Setting `LANGUAGE_CODE`
  alone changes Django's default locale-aware behaviors (number/date
  formatting defaults, which admin language loads by default) but
  doesn't translate any of THIS project's own strings yet.

ACCEPTANCE CRITERIA / TESTS
- `python manage.py check` passes with no errors after the change.
- Add a trivial settings test confirming `settings.LANGUAGE_CODE == "fa"`
  and `"fa"` appears in `settings.LANGUAGES`.
- Manually verify: the Django admin login page, after this change,
  renders with at least Django's own built-in Persian translations for
  standard admin chrome (login button labels, etc.) — this is a quick,
  visible sanity check that the setting is actually taking effect.
```

---

#### Task 14.1.1.2 — Set `TIME_ZONE = "Asia/Tehran"`

```
You are working in backend/core/settings/base.py. Assume Task 14.1.1.1
is already merged.

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
`TIME_ZONE = "UTC"` currently, with `USE_TZ = True` (meaning Django
stores all datetimes internally as UTC-aware in the database regardless
of this setting — `TIME_ZONE` controls the DEFAULT DISPLAY/interpretation
timezone for the admin and any `timezone.localtime()` calls, not the
storage format). Every `created_at`/`updated_at`/order-timestamp/etc.
across the whole system, by this point spanning 13 epics of work, is
currently displayed to admin staff in UTC — confusing for
Iran-based staff who need to reason about "did this order come in this
morning" relative to their own local time.

TASK
Set the default display timezone to Iran Standard Time.

REQUIREMENTS
- Change: `TIME_ZONE = "Asia/Tehran"`
- This is a DISPLAY-layer change only — confirm `USE_TZ` stays `True`
  (storage remains UTC internally, which is correct and should NOT
  change; only the zone used when RENDERING/localizing datetimes for
  admin display, and the zone Django's `timezone.now()`-adjacent
  localtime helpers assume, changes).
- Audit anywhere in the codebase that might have been WRITTEN assuming
  UTC display specifically (rather than using Django's
  timezone-aware helpers correctly) — search for any raw
  `datetime.now()` (non-timezone-aware, a red flag on its own
  regardless of this task) or any hardcoded UTC-offset assumptions in
  business logic (e.g. Epic 3's near-expiry filters, Epic 4's
  stock-movement timestamps, Epic 6/7's Celery periodic task scheduling
  — none of these SHOULD have hardcoded UTC assumptions if they were
  built correctly using `django.utils.timezone` throughout per this
  project's established conventions, but this is a good moment to
  actually verify that holds, not just assume it). Flag anything found
  rather than silently fixing potentially-large unrelated code in this
  small task — if you find a genuine bug, note it clearly as a
  follow-up rather than expanding this task's scope unpredictably.
- Consider whether Iran's historical DST (daylight saving time)
  observance affects anything — Iran discontinued DST observance in
  2022 (verify this is still accurate at the time you do this task,
  since policy could theoretically change again) — Python's `zoneinfo`/
  `pytz` timezone database should handle whatever the CURRENT rule
  is correctly as long as the underlying tzdata package is reasonably
  up to date; this isn't something to hand-code, just be aware it's a
  live consideration if you see any date-arithmetic bugs later.

ACCEPTANCE CRITERIA / TESTS
- `python manage.py check` passes.
- Add a test confirming `settings.TIME_ZONE == "Asia/Tehran"`.
- Manually verify: a newly-created object's `created_at`, when viewed
  in the Django admin (which localizes display times to `TIME_ZONE`),
  shows a time consistent with Iran Standard Time, not UTC.
```

---

#### Task 14.1.1.3 — Wrap all admin-facing/user-facing strings in `gettext_lazy`

```
You are working across the ENTIRE backend/ codebase. Assume Tasks
14.1.1.1/14.1.1.2 are merged. This task is explicitly flagged in the
master backlog as larger than a typical single-session task ("2h
recurring, track as multiple sessions") — treat it as an ongoing sweep,
not a one-shot completion, and don't feel obligated to catch literally
every string in the codebase in one sitting; prioritize correctly and
document what's left.

CONTEXT
Every user-facing/admin-facing string across 13 epics of accumulated
work — validation error messages, model `verbose_name`/`help_text`,
choice labels, admin `list_display`/action descriptions — is currently
a plain Python string with no translation marker at all. None of it can
be translated into Persian (or anything else) without first being
wrapped in Django's translation functions.

TASK
Sweep the codebase wrapping translatable strings in
`django.utils.translation.gettext_lazy` (for anything evaluated at
class/module definition time — model field `verbose_name`/`help_text`,
`choices` labels) or `gettext` (for anything evaluated per-request,
like a view's error message string built inside a function body).

REQUIREMENTS
- Prioritize in this order (most customer-visible first):
  1. **Serializer validation error messages** — every
     `serializers.ValidationError({"field": "some string"})` across
     `order`, `cart`, `shop`, `accounts`, `payments`, `shipping`,
     `promotions` — these are messages a REAL CUSTOMER sees directly
     during checkout/account actions, the highest-value strings to
     translate first.
  2. **Model `verbose_name`/`help_text`** on customer-facing fields
     (less critical than #1 since these mostly surface in Django
     admin/DRF browsable API, but still worth doing).
  3. **`TextChoices`/`IntegerChoices` human-readable labels** — e.g.
     `Order.Status.PENDING = "pending", "Pending"` — the SECOND value in
     each choice tuple should be wrapped:
     `PENDING = "pending", _("Pending")`.
  4. Admin `list_display`/`list_filter`/action descriptions (lowest
     priority — admin staff can be assumed to use English admin chrome
     even on a Persian storefront, per Task 14.1.1.1's rationale for
     keeping English in `LANGUAGES`; don't spend disproportionate
     effort here).
- Import convention: `from django.utils.translation import gettext_lazy as _`
  at the top of each file touched, matching Django's own long-standing
  idiomatic alias (`_`) for this — this IS the standard, expected
  pattern, not a stylistic choice up for debate.
- Be careful with F-STRINGS containing translated content — an f-string
  like `f"Only {stock} left in stock"` (a real pattern already present
  in this codebase, e.g. Epic 3 Task 3.1.1.3's stock-check error
  message) CANNOT simply be wrapped as
  `_(f"Only {stock} left in stock")` — `gettext`/`gettext_lazy` need a
  STATIC string with `%(name)s`-style placeholders for correct
  translation-file extraction (translators need to see the placeholder
  structure, not a pre-interpolated string that changes every time):
  ```python
  # WRONG — can't be extracted/translated correctly:
  _(f"Only {stock} left in stock")

  # RIGHT:
  _("Only %(stock)s left in stock.") % {"stock": stock}
  ```
  Sweep for and fix every f-string-based user-facing error message you
  touch during this pass, converting to the `%`-placeholder style
  above — this is a real, easy-to-get-wrong detail, not a nitpick.
- Do NOT wrap: internal log messages (Python `logging` calls — these
  are for developers/ops, not customers, and translating them adds
  friction to debugging without user-facing benefit), code comments
  obviously, or any string that's actually a fixed protocol/API value
  (e.g. `Order.Status` CHOICE KEYS like `"pending"` themselves — only
  their human-readable LABEL should be wrapped, never the stored
  database value, which must remain a stable, untranslated string for
  the system to keep working correctly).
- Track progress: since this is explicitly a multi-session task, keep
  a simple running list (e.g. a `TRANSLATION_SWEEP_PROGRESS.md`
  scratch file, or just clear commit messages per app covered) of which
  apps have been fully swept vs. still pending, so subsequent sessions
  don't have to re-discover where the previous session left off.

ACCEPTANCE CRITERIA / TESTS
- After each app's sweep, run `python manage.py makemessages -l fa`
  (from Task 14.1.1.4, which can run incrementally even before that
  task is "done") and confirm the `.po` file picks up the newly-wrapped
  strings — this is the actual functional verification that wrapping
  was done correctly (a string that `makemessages` doesn't find wasn't
  wrapped correctly, or was wrapped in a way Django's extraction
  tooling can't parse, e.g. the f-string mistake above).
- Add a regression test somewhere appropriate confirming at least one
  KEY customer-facing error message (e.g. the insufficient-stock
  checkout error from Epic 1/3) is genuinely translatable — construct
  a minimal test that activates a different language via
  `django.utils.translation.activate()` and confirms the message
  changes (even if the ONLY translation available at this point is a
  test-provided one, since Task 14.1.1.4's real `.po` translations may
  not exist yet when this task runs) — this proves the wrapping
  mechanism works end-to-end, not just that `_()` was typed somewhere.
```

---

#### Task 14.1.1.4 — Generate and populate `fa` `.po`/`.mo` translation files

```
You are working in backend/locale/. Assume Task 14.1.1.3 has made
meaningful (even if not 100% complete) progress wrapping strings.

CONTEXT
Django's translation workflow requires: (1) `makemessages` to extract
every wrapped string into a `.po` file, (2) a human/service to fill in
the actual Persian translations for each extracted string, (3)
`compilemessages` to compile the `.po` into a binary `.mo` file Django
actually loads at runtime. None of this exists yet.

TASK
Set up the translation file workflow and populate real Persian
translations for the strings wrapped so far.

REQUIREMENTS
- Ensure `gettext` (the underlying system tool Django's
  `makemessages`/`compilemessages` shell out to) is available in the
  development/CI/Docker environment — this is a SYSTEM dependency, not
  a pip package; check the project's Dockerfile and add
  `apt-get install gettext` (or the equivalent for whatever base image
  this project uses) if it's not already present, since
  `makemessages`/`compilemessages` will fail outright without it.
- Add `LOCALE_PATHS = [BASE_DIR / "locale"]` to
  backend/core/settings/base.py if not already configured (check
  first).
- Run `python manage.py makemessages -l fa` from within backend/,
  generating `backend/locale/fa/LC_MESSAGES/django.po`.
- Populate REAL Persian translations for every extracted `msgid` in the
  generated `.po` file — this requires actual Persian-language
  translation work, not placeholder/machine-translated text passed off
  as final; if you have translation capability available (a bilingual
  reviewer, a translation service, or your own genuine Persian
  language capability), use it for real, accurate, natural-sounding
  Persian phrasing appropriate for an e-commerce checkout/account
  context (not overly literal/stilted translation) — this is
  customer-facing text on a commercial platform and deserves the same
  care as the English original.
- Run `python manage.py compilemessages` to produce the `.mo` file.
- Add BOTH `.po` and `.mo` to version control (some teams `.gitignore`
  compiled `.mo` files and rebuild them at deploy time instead — check
  whether this project's Dockerfile/CI already has a
  `compilemessages` step as part of its build process; if so, `.mo`
  can reasonably be gitignored and rebuilt at deploy time instead of
  committed, which is arguably cleaner — decide based on what's already
  in place, and if you add a NEW `compilemessages` build step, make
  sure it actually runs before the app starts, not just exists
  unused).

ACCEPTANCE CRITERIA / TESTS
- Add a test that activates Persian (`translation.activate("fa")`) and
  confirms a KNOWN, translated string (e.g. the checkout stock-error
  message from Task 14.1.1.3) renders in Persian, not English.
- Add a test that activates English (`translation.activate("en")`) and
  confirms the SAME message renders back in English — proving the
  translation system correctly falls back/switches both directions, not
  just one-way.
- Manually verify: setting `Accept-Language: fa` on an API request (or
  however this project's locale-detection middleware is configured —
  confirm `LocaleMiddleware` is actually present in `MIDDLEWARE`, and
  add it if missing, since it's required for automatic per-request
  language switching based on the `Accept-Language` header) results in
  a real API error response containing Persian text.
```

---

### Feature 14.1.2 — Currency Handling

---

#### Task 14.1.2.1 — Decide internal currency unit (store Rial as integer, display Toman)

```
You are working on an architecture decision, documented in a new
backend/docs/adr/0001-currency-storage.md (or wherever this project's
existing ADR convention lives, if one was established — check for a
`docs/adr/` folder from any prior epic's documentation work; if none
exists yet, create one, since this decision genuinely warrants a
permanent, discoverable record given how many files Task 14.1.2.2 is
about to touch based on it).

CONTEXT
Every monetary field in this system — confirmed across the whole
codebase built through Epic 13 — is currently a `DecimalField(max_digits=10,
decimal_places=2)` representing an abstract "dollars-and-cents"-shaped
number with no real-world currency semantics at all (it was originally
a USD-shaped template). Iranian currency has two commonly-used units:
**Rial** (Iran's official currency, ISO 4217 code IRR, no fractional
subunit in practical modern use) and **Toman** (colloquially used in
everyday speech and increasingly in e-commerce pricing, equal to 10
Rial — so 10,000 Rial = 1,000 Toman). Nearly all consumer-facing
Iranian e-commerce sites DISPLAY prices in Toman (it's what customers
expect and mentally calculate in) but this doesn't mean the DATABASE
must store Toman — a storage-vs-display distinction is worth making
deliberately.

TASK
Formally decide and document: (1) the internal storage unit and data
type for every monetary field, and (2) the display conversion/formatting
rule, before Task 14.1.2.2 executes the actual migration based on this
decision.

REQUIREMENTS — the decision to document (and the recommended answer)
- **Storage unit: Rial, as an integer (no decimal places).** Rationale:
  Rial has no meaningful fractional/subunit in modern practical use
  (unlike USD cents), so storing it as `DecimalField(decimal_places=2)`
  was always semantically wrong for this currency — an integer field is
  both more correct AND avoids floating-point-adjacent decimal
  precision concerns entirely for arithmetic (tax calculations,
  discount percentages, etc.), which is a genuine, non-cosmetic
  improvement over the current `Decimal` scheme, not just a rename.
  Django's `PositiveBigIntegerField` (or plain `BigIntegerField` if a
  negative value is ever legitimately needed, e.g. a refund/adjustment
  amount — decide per-field) is the correct type, NOT
  `PositiveIntegerField`, since Rial amounts for real purchases can
  exceed `PositiveIntegerField`'s ~2.1 billion max value surprisingly
  easily at typical Iranian Rial price points (a single mid-range
  product could easily be several million Rial) — this sizing detail
  matters and is worth getting right in the decision doc so Task
  14.1.2.2 doesn't have to re-derive it.
- **Display unit: Toman, computed at the presentation layer only**
  (never stored) — every price displayed to a customer is
  `stored_rial_value // 10`, computed in a formatting utility (frontend
  JS utility function AND/OR a Python serializer method if the API
  itself should expose a pre-converted Toman value — decide whether the
  API returns raw Rial with the frontend converting, or the API
  converts server-side; RECOMMEND the API continuing to expose RIAL
  values consistently as the single source of truth, with ALL
  Toman-conversion/formatting happening client-side in one shared
  utility — this keeps the backend's stored/transmitted values
  unambiguous and currency-unit-explicit, and centralizes the
  potentially-adjustable "how do we want to round/format this for
  display" logic in exactly one frontend location rather than
  duplicating conversion logic in both Python and JS).
- Document the rounding rule for the Rial-to-Toman display division
  (Rial values not evenly divisible by 10 — which shouldn't normally
  occur if all storage/arithmetic correctly stays in whole-Rial
  integers throughout, but could arise from percentage-based discount/
  tax calculations that don't divide evenly — decide: round down,
  round to nearest, or treat a non-multiple-of-10 Rial value as a bug
  to be prevented upstream via careful rounding at EVERY calculation
  step, not papered over at display time). RECOMMEND: ensure every
  calculation (tax, discount, shipping) rounds to whole Rial at each
  step (never carrying fractional Rial through a chain of
  calculations), making Toman division always clean; document this as
  a hard invariant Task 14.1.2.2's pricing-service updates must
  preserve.

ACCEPTANCE CRITERIA
This task's deliverable is the ADR document itself — reviewed and
"accepted" (in whatever lightweight sense this project treats ADRs) —
which Task 14.1.2.2 then implements verbatim. Include in the document
an explicit, complete ENUMERATION of every model+field this decision
affects (compiled by grep-searching the whole backend for
`DecimalField` and manually confirming which are genuinely monetary
vs. incidentally decimal for another reason, e.g. `Coupon.value` for a
PERCENT-type coupon is a percentage, not currency, and should NOT be
converted — only FIXED-type discount amounts and genuine currency
fields should move to the new integer-Rial scheme) — this enumeration
becomes Task 14.1.2.2's actual checklist, so get it right and complete
here rather than discovering missed fields mid-migration.
```

---

#### Task 14.1.2.2 — Migrate monetary `DecimalField`s to integer Rial

```
You are working across MANY files in backend/. Assume Task 14.1.2.1's
ADR is finalized. THIS IS THE HIGHEST-BLAST-RADIUS TASK IN THIS ENTIRE
PROJECT BACKLOG SO FAR — budget real, careful time for it, work
methodically through the checklist below, and re-run the ENTIRE
project's test suite (every app, every epic) at the end, not just the
apps you directly touched.

CONTEXT
Per Task 14.1.2.1's decision: every genuinely-monetary `DecimalField`
becomes an integer Rial field. Compiled from the whole codebase's
accumulated state through Epic 13, the fields requiring migration are:

- `shop.ProductVariant.price`, `original_price`
- `order.Order.subtotal`, `shipping_cost`, `tax`, `discount`, `total`
- `order.OrderItem.unit_price`
- `payments.PaymentTransaction.amount`
- `payments.RefundRequest.amount`
- `shipping.ShippingRate.price`
- `promotions.Coupon.min_order_amount`, and `Coupon.value` **ONLY when
  `discount_type == FIXED`** (the PERCENT case is a 0-100 percentage,
  NOT currency, and must stay a decimal/percentage field — do not
  convert it; if `Coupon.value`'s dual meaning makes a single-field
  integer conversion awkward given it serves two different semantic
  purposes depending on `discount_type`, consider whether splitting it
  into two separate fields, `percent_value`/`fixed_value_rial`, is
  actually the cleaner fix here rather than forcing one field to be
  "sometimes an integer Rial amount, sometimes a decimal percentage" —
  this is a real design improvement worth making as part of this
  migration rather than preserving an awkward dual-purpose field)
- `promotions.CouponRedemption.discount_amount`

REVIEW THIS LIST AGAINST THE ACTUAL CURRENT CODEBASE STATE before
starting — this task's own list was compiled from this document's
grounding across Epics 1–13, but confirm via
`grep -rn "DecimalField" backend/ --include=*.py` that nothing was
missed and nothing on this list has since changed shape.

TASK
For EVERY field above: change the field type, write a data migration
converting existing Decimal-dollars-shaped values to integer Rial
(multiply by 100 to get "cents-equivalent," which — given this
project's data was always semantically dollar-shaped test/seed data,
not real production Rial amounts yet at this stage of most real
implementations — actually just needs a clear, documented, CONSISTENT
conversion factor; if this migration runs against genuinely live
production financial data rather than test/seed data, the conversion
factor and rounding must be verified with extreme care and likely
signed off by whoever owns the business's actual pricing data, since
getting this wrong on live financial records would be a serious
incident — flag this prominently and do not treat it as a routine
schema change if real customer financial history exists in the target
database at migration time), update every Python call site that reads/
writes these fields, and update the pricing service.

REQUIREMENTS
- For each field: change
  `models.DecimalField(max_digits=10, decimal_places=2)` to
  `models.PositiveBigIntegerField()` (or `BigIntegerField()` for any
  field that legitimately needs to represent a negative
  amount/adjustment — review each field individually rather than
  applying one blanket type to all of them).
- Write a DATA migration (separate from the schema-altering migration,
  same two-step pattern already established in Epic 5 Task 5.2.1.4 for
  the Address model rename) for each app, converting existing values.
  Since this project's data through Epic 13 is realistically seed/test
  data in a dollar-shaped decimal scheme, NOT real Iranian Rial pricing
  yet, the "conversion" is really a RE-BASELINING: existing test data
  doesn't need to represent real-world-accurate Rial amounts, it just
  needs the SCHEMA and ARITHMETIC to be correct going forward. A
  reasonable, simple, documented approach:
  `new_integer_value = round(old_decimal_value * 10000)` (treating the
  old "$X.XX" test values as if they'd always been intended as
  "X0,000 Rial" — an arbitrary but clearly-documented scaling factor
  that produces realistic-LOOKING Rial magnitudes for existing
  seed/test data, e.g. an old `$29.99` product becomes `299,900` Rial —
  roughly ~$7 at typical exchange rates, a plausible product price
  point). Document this conversion factor decision explicitly in the
  migration file's comments, and in your task summary, since it's an
  arbitrary but necessary choice for non-production data — if this
  project's actual target database has REAL financial records at
  migration time, this arbitrary scaling approach is NOT appropriate
  and a real currency-accurate conversion (or a decision to preserve
  historical orders' original values as a frozen legacy reference while
  only NEW data uses the new scheme) would be required instead — flag
  this distinction clearly.
- Update `order/services/pricing.py`'s `calculate_order_totals()`:
  change all `Decimal` type hints/arithmetic to plain Python `int`
  arithmetic, remove all `.quantize(Decimal("0.01"))` rounding calls
  (meaningless for integer values — replace with `round()` only where
  a genuinely fractional intermediate calculation, like a percentage-
  based discount/tax, needs rounding to the nearest whole Rial per Task
  14.1.2.1's "round at every step" invariant).
- Update EVERY call site across `order`, `cart`, `payments`, `shipping`,
  `promotions` that constructs a `Decimal(...)` for one of these fields
  — this includes (non-exhaustively, confirm via a full grep sweep for
  `Decimal(` across the whole backend) checkout serializers, coupon
  validation (`promotions/services.py`), payment gateway integrations
  (Epic 6's `request_payment`/`verify_payment` amount handling — note
  those gateways likely already expect Rial as an integer per this
  epic's own header note in Epic 6's grounding, so this change may
  actually SIMPLIFY those call sites by removing a now-unnecessary
  `int(amount)` cast that was working around the OLD Decimal-dollar
  representation), shipping rate lookups, invoice PDF generation (Epic
  8), CSV export (Epic 8).
- Update every SERIALIZER exposing these fields — DRF's
  `serializers.DecimalField` becomes `serializers.IntegerField()`
  for each converted field, across every serializer in every app that
  touches money.
- Update the FRONTEND: every place currently formatting a price as
  `$X.XX`-shaped (search frontend/src for `toFixed(2)`,
  `.toLocaleString()` calls on price-shaped values, any currency
  symbol like `$`) needs updating to work with a raw Rial integer and
  the NEW Toman-display formatting utility (Task 14.1.2.3, which this
  task should coordinate with — implement 14.1.2.3's utility as PART of
  this task if it's not already done, since this task's frontend
  updates need it immediately to display anything sensible; the
  ordering in this document lists them separately for backlog-tracking
  clarity, but in practice do them together).

ACCEPTANCE CRITERIA / TESTS
- Every migration applies cleanly against representative seed data.
- Re-run the ENTIRE project test suite — order, cart, payments,
  shipping, promotions, shop, dashboard — and update EVERY test
  assertion currently comparing against a `Decimal("X.XX")`-shaped
  value to the new integer-Rial equivalent. This will touch a large
  number of tests across every prior epic's work; this is expected,
  not a sign of a bad migration.
- Add new tests specifically confirming NO fractional/decimal precision
  bugs exist anywhere in the new integer arithmetic — e.g. a checkout
  with a percentage discount applied produces a correctly-ROUNDED whole
  Rial total, not a value requiring further rounding at display time.
- Manually verify, end to end: browse a product (price displays
  correctly in Toman), add to cart, apply a coupon, check out, and
  confirm the ENTIRE chain — cart subtotal, coupon discount, shipping
  cost, tax, order total, payment gateway amount sent, invoice PDF — all
  show CONSISTENT, correctly-converted values with no silent
  truncation or drift anywhere in the pipeline.
```

---

#### Task 14.1.2.3 — Toman display formatting utility (frontend)

```
You are working in frontend/src/utils/ (new file, or extending an
existing formatting-utilities file if one already exists — check
first). Per Task 14.1.2.2's note, this task's implementation is likely
needed CONCURRENTLY with that task's frontend updates — if it's already
been built as part of 14.1.2.2, this task is really about confirming/
polishing/testing it thoroughly rather than building it fresh.

CONTEXT
Per Task 14.1.2.1's decision, the API transmits raw Rial integers; all
Toman conversion/formatting happens in exactly ONE frontend utility,
used everywhere a price is displayed.

TASK
Build `formatToman()`, a single, well-tested, universally-used
currency-formatting utility.

REQUIREMENTS
- Create frontend/src/utils/currency.js:
  ```javascript
  /**
   * Convert a stored Rial integer amount to a formatted Toman display
   * string with thousand separators, e.g. 299900 (Rial) -> "29,990 تومان".
   */
  export function formatToman(rialAmount) {
    if (rialAmount == null || Number.isNaN(rialAmount)) return '';
    const toman = Math.floor(rialAmount / 10);
    const formatted = toman.toLocaleString('en-US'); // thousand separators; digit script handled separately, see note below
    return `${formatted} تومان`;
  }

  /**
   * Convert a stored Rial integer to a plain Toman number (no
   * formatting/currency label) — for calculations or contexts that
   * need the raw numeric value rather than a display string.
   */
  export function rialToToman(rialAmount) {
    return Math.floor((rialAmount ?? 0) / 10);
  }
  ```
  Note the deliberate use of `'en-US'` locale for `toLocaleString()`
  thousand-separator formatting rather than `'fa-IR'` — this is worth a
  conscious decision, not an oversight: `'fa-IR'` locale formatting in
  JS produces Persian/Eastern Arabic numeral digits (۰۱۲۳...) by
  default, which is DIFFERENT from Persian-digit RENDERING as a pure
  display/typography concern (covered separately in Epic 14's broader
  Persian-digit work, if a dedicated task for it exists elsewhere in
  this project's backlog — check for one; if the master backlog's
  Task 14.1.2.4 "Persian digit rendering toggle" hasn't been done yet
  at this point, this utility should stay in Western digits for now
  and Task 14.1.2.4 layers Persian-digit conversion on top as an
  explicit, separate, TOGGLEABLE concern, per that task's own
  description — don't conflate the two here).
- Ensure `formatToman()` handles `0`, `null`, `undefined`, and negative
  values (e.g. a refund/discount display) sensibly — a negative Rial
  amount (e.g. showing "-29,990 تومان" for a discount line) should
  format correctly with the negative sign preserved, not silently
  dropped or causing a rendering bug.
- Sweep the frontend (the same search Task 14.1.2.2 already started)
  for EVERY place a price is currently displayed and replace ad-hoc
  formatting with `formatToman()` — product cards, PDP, cart, checkout
  summary, order history, admin dashboard revenue figures, invoice
  display — this utility should become the SINGLE, universal price-
  display mechanism across the entire frontend, with zero remaining
  ad-hoc `$`/`.toFixed(2)`-style formatting left anywhere.

ACCEPTANCE CRITERIA / TESTS
Add tests to frontend/src/utils/__tests__/currency.test.js:
1. `formatToman(299900)` returns `"29,990 تومان"`.
2. `formatToman(0)` returns `"0 تومان"`.
3. `formatToman(null)`/`formatToman(undefined)` return `""` (or
   whichever sensible empty-state you implemented — don't let it throw
   or render `"NaN تومان"`).
4. `formatToman(-50000)` correctly formats a negative amount.
5. `rialToToman()` correctly floor-divides by 10 for both positive and
   representative edge-case (non-multiple-of-10, if that can ever
   legitimately occur) values.
Grep the final frontend codebase for any remaining `toFixed(2)` or `$`
literal near a price-shaped variable and confirm none remain outside
this utility itself — a genuinely complete sweep, not a partial one.
```

---

#### Task 14.1.2.4 — Persian digit rendering toggle

```
You are working in frontend/src/utils/ (extending currency.js from
Task 14.1.2.3, or a new numerals.js). Assume Task 14.1.2.3 is merged.

CONTEXT
Persian text conventionally renders numerals using Eastern Arabic-
Persian digit glyphs (۰۱۲۳۴۵۶۷۸۹) rather than Western Arabic numerals
(0123456789) — but NOT universally; many real Iranian websites/apps
mix both depending on context (prices are often shown in Western
digits even on otherwise fully-Persian interfaces, since Western digits
are frequently considered clearer for reading exact numeric amounts,
while dates/counts elsewhere might use Persian digits) — there's no
single universally "correct" answer, which is exactly why this is
framed as a TOGGLE rather than an unconditional global replacement.

TASK
Build a Persian-digit conversion utility with a toggle, rather than
forcing one convention everywhere.

REQUIREMENTS
- Add to frontend/src/utils/numerals.js:
  ```javascript
  const PERSIAN_DIGITS = ['۰', '۱', '۲', '۳', '۴', '۵', '۶', '۷', '۸', '۹'];

  /**
   * Convert Western Arabic digits (0-9) in a string to Persian digits.
   * Non-digit characters are left untouched.
   */
  export function toPersianDigits(input) {
    if (input == null) return '';
    return String(input).replace(/[0-9]/g, (digit) => PERSIAN_DIGITS[Number(digit)]);
  }
  ```
- Add a project-wide setting/context controlling the DEFAULT digit
  convention for the site as a whole (e.g. a simple constant exported
  from a config file, `DEFAULT_NUMERAL_STYLE = 'western'` — recommend
  defaulting to WESTERN digits for prices specifically per the context
  note above, since exact numeric clarity matters most for money, while
  potentially defaulting to PERSIAN digits elsewhere, e.g. star ratings,
  review counts, or date displays, where the stylistic Persian-native
  feel matters more than split-second numeric precision reading) —
  don't hardcode one universal choice for literally every number on the
  site; make it a per-context decision using the same underlying
  utility.
- Extend `formatToman()` (Task 14.1.2.3) with an optional parameter:
  `formatToman(rialAmount, { persianDigits: false })` (default `false`,
  per the recommendation above), applying `toPersianDigits()` to the
  final formatted string when `true`.
- Apply Persian-digit conversion to a FEW deliberately-chosen,
  non-monetary contexts as this task's actual scope (rather than a
  sweeping site-wide change, which risks being the wrong call in places
  it hasn't been thought through for) — e.g. product rating display
  ("۴.۵ از ۵" / "4.5 out of 5"), review counts, and Jalali date displays
  (Feature 14.2.1, if that epic's work has landed — coordinate the
  exact set of dates/counts this touches with whatever's actually
  implemented at this point rather than guessing).

ACCEPTANCE CRITERIA / TESTS
Add tests to frontend/src/utils/__tests__/numerals.test.js:
1. `toPersianDigits("12345")` returns `"۱۲۳۴۵"`.
2. `toPersianDigits("Order #12345")` correctly converts only the
   digits, leaving other characters untouched.
3. `toPersianDigits(null)` returns `""` without throwing.
4. `formatToman(299900, { persianDigits: true })` returns the
   Persian-digit version of the formatted Toman string; the default
   call (no options) remains Western-digit, confirming the toggle's
   default matches the documented recommendation.
```

---

## Phase 14.2 — Jalali Calendar

### Feature 14.2.1 — Date Handling

---

#### Task 14.2.1.1 — Add `django-jalali`/`jdatetime` to backend

```
You are working in backend/requirements.txt, core/settings/base.py.
Assume Phase 14.1 is fully merged.

CONTEXT
Every date/datetime across this system — order timestamps, coupon
validity windows, flash sale windows, shipment tracking, expiration
dates — is currently displayed using the standard Gregorian calendar
throughout, including in the Django admin, which Iranian admin staff
will find genuinely awkward to reason about day-to-day compared to the
Jalali (Persian/Solar Hijri) calendar they actually use in daily life.

TASK
Add Jalali calendar support to the backend, primarily for Django admin
display purposes.

REQUIREMENTS
- Add `django-jalali` (which wraps `jdatetime` and provides Jalali-aware
  admin widgets/model fields) to backend/requirements.txt, pinned to a
  specific current version matching this project's fully-pinned
  dependency convention.
- Add `"django_jalali"` to `INSTALLED_APPS` in
  backend/core/settings/base.py (check the package's actual current
  app-config name/required settings against its current documentation
  — package APIs and required setup steps can change between versions,
  verify against the currently-installed version's actual docs rather
  than assuming a specific historical setup snippet is still accurate).
- This task is INSTALLATION/CONFIGURATION only — it does NOT require
  converting every DateField/DateTimeField in the system to a Jalali
  field type (that would be a much larger, more invasive, and likely
  unnecessary change, since the underlying STORAGE should almost
  certainly remain standard Gregorian/UTC internally regardless of
  display calendar, exactly mirroring the storage-vs-display split
  already established for currency in Task 14.1.2.1 — Jalali is a
  DISPLAY/admin-UX concern, not a storage concern). Task 14.2.1.2
  handles the actual admin-display wiring for specific models.

ACCEPTANCE CRITERIA / TESTS
- `python manage.py check` passes with the new app installed.
- Add a minimal smoke test confirming `jdatetime` is importable and can
  correctly convert a known Gregorian date to its expected Jalali
  equivalent (e.g. a well-known reference date pair) — a basic sanity
  check that the library itself is correctly installed/functioning
  before building admin-display features on top of it in the next task.
```

---

#### Task 14.2.1.2 — Jalali date display in Django admin

```
You are working across various backend/*/admin.py files. Assume Task
14.2.1.1 is already merged.

CONTEXT
Admin list/detail views across every app (order, shop, payments,
shipping, promotions, dashboard) currently show Gregorian dates for
every timestamp field.

TASK
Add Jalali-formatted date DISPLAY to the key, most-frequently-referenced
admin timestamp fields, without changing underlying storage.

REQUIREMENTS
- Prioritize the highest-value admin screens rather than attempting
  every single date field across the whole system in one task (matching
  this epic's established "prioritize, don't boil the ocean" pattern
  from Task 14.1.1.3): `OrderAdmin`/`AdminOrderViewSet`-adjacent date
  displays (`created_at`, shipment `last_tracked_at`), `PaymentTransactionAdmin`
  (`created_at`), `CouponAdmin`/`FlashSaleAdmin` (`valid_from`/
  `valid_until`/`starts_at`/`ends_at` — these in particular benefit from
  Jalali display since admin staff are actively SCHEDULING campaigns
  around real-world Persian calendar dates/holidays, making this one of
  the highest-value spots for this feature).
- For each targeted admin, use `django-jalali`'s provided admin
  field/widget helpers (check the package's current documentation for
  the exact recommended integration pattern for `ModelAdmin` — this
  varies by package version, verify against what's actually installed
  per Task 14.2.1.1) to render a Jalali-formatted, read-only display
  column alongside (or replacing, your call based on what reads better)
  the existing Gregorian display, WITHOUT changing the underlying model
  field's actual type (which remains a standard Django
  `DateTimeField`/`DateField` storing Gregorian/UTC internally, per
  Task 14.2.1.1's explicit storage-vs-display decision).
- A common, reliable pattern that works regardless of the exact
  third-party package API is a simple `list_display` method:
  ```python
  import jdatetime

  class OrderAdmin(admin.ModelAdmin):
      ...
      list_display = [..., "created_at_jalali"]

      def created_at_jalali(self, obj):
          return jdatetime.datetime.fromgregorian(datetime=obj.created_at).strftime("%Y/%m/%d %H:%M")
      created_at_jalali.short_description = "تاریخ ایجاد (شمسی)"
  ```
  This DIY approach (rather than relying entirely on `django-jalali`'s
  own admin widget magic, which may or may not fit every admin's exact
  needs cleanly) is a safe, always-available fallback if the package's
  built-in integration doesn't fit a specific case well — use whichever
  approach is cleaner PER admin class, don't force one universal
  pattern if the package's native integration is genuinely better for
  some cases.

ACCEPTANCE CRITERIA / TESTS
Add tests confirming the Jalali-display methods on each targeted admin
class correctly convert a KNOWN Gregorian timestamp to its expected
Jalali string representation (use the same reference date pair
verification approach as Task 14.2.1.1's smoke test). Manually verify
in the actual Django admin UI that the targeted screens now show
sensible, correctly-formatted Jalali dates alongside/instead of
Gregorian ones.
```

---

#### Task 14.2.1.3 — Frontend Jalali date library integration

```
You are working in frontend/package.json and a new
frontend/src/utils/jalaliDate.js. Assume Phase 14.1 is merged (this
task doesn't strictly depend on the backend Jalali work from Task
14.2.1.1/14.2.1.2, since it's an independent frontend concern, but
logically continues this epic's Jalali theme).

CONTEXT
Customer-facing dates (order placed date, estimated delivery window,
flash sale countdown end date, coupon expiry) are currently rendered
using whatever the browser/JS default Gregorian formatting produces —
no Jalali conversion exists on the frontend at all.

TASK
Add a Jalali date-formatting utility for customer-facing date displays.

REQUIREMENTS
- Add a Jalali-calendar-aware date library to frontend/package.json —
  `dayjs` with its Jalali plugin (`jalaliday` or similar — check current
  npm ecosystem for the actively-maintained option, since package
  availability/maintenance status can shift over time; verify whatever
  you choose is currently maintained and has reasonable adoption before
  committing to it) is a lightweight, commonly-used choice given this
  project may already use `dayjs` elsewhere (check frontend/package.json
  for an existing `dayjs` dependency from any prior epic's date-handling
  work and prefer extending that rather than introducing a second,
  competing date library like `moment`/`date-fns` alongside it).
- Create frontend/src/utils/jalaliDate.js:
  ```javascript
  import dayjs from 'dayjs';
  import jalaliday from 'jalaliday'; // or whichever plugin you selected
  dayjs.extend(jalaliday);

  /**
   * Format an ISO date string (as returned by the API, always
   * Gregorian/UTC per this project's storage convention) as a Jalali
   * date string for customer-facing display.
   */
  export function formatJalaliDate(isoString, format = 'YYYY/MM/DD') {
    if (!isoString) return '';
    return dayjs(isoString).calendar('jalali').locale('fa').format(format);
  }
  ```
  (adjust the exact API to match whichever library/plugin you actually
  selected — verify its current, real API against its own documentation
  rather than assuming this exact syntax is accurate for the specific
  version you install).
- Apply this utility to customer-facing date displays: order history
  dates, order confirmation page, estimated delivery windows (Epic 7's
  shipment tracking), coupon/flash-sale expiry displays (Epic 9) — a
  similar prioritized, not-exhaustive-in-one-pass sweep as this epic's
  other tasks.

ACCEPTANCE CRITERIA / TESTS
Add tests to frontend/src/utils/__tests__/jalaliDate.test.js:
1. `formatJalaliDate()` for a known, verified Gregorian ISO date string
   produces the correct, verified Jalali equivalent (use the SAME
   reference date pair already verified in Task 14.2.1.1's backend
   smoke test, for cross-stack consistency confidence — if the backend
   and frontend Jalali conversions ever disagreed on the same reference
   date, that would indicate a real bug worth catching).
2. `formatJalaliDate(null)`/`formatJalaliDate(undefined)` return `""`
   without throwing.
3. A custom `format` parameter is respected correctly.
```

---

#### Task 14.2.1.4 — Jalali date picker component (checkout/admin forms)

```
You are working in frontend/src/components/ (new component). Assume
Task 14.2.1.3 is already merged.

CONTEXT
Any form input requiring date selection anywhere in this platform
(check for existing date-input needs — Epic 5's checkout doesn't
obviously need one, but Epic 9's admin coupon/flash-sale creation
forms from Task 9.1.1.8/`FlashSaleAdmin` DO need `valid_from`/
`valid_until`/`starts_at`/`ends_at` date pickers, and these are
currently, if built at all, using a plain HTML `<input type="date">`
or a generic Gregorian date-picker component) has no Jalali-aware date
picker option.

TASK
Build a reusable `JalaliDatePicker` component and use it in the admin
coupon/flash-sale creation forms (Epic 9's admin UI, the most concrete,
already-identified consumer of this component).

REQUIREMENTS
- Build `frontend/src/components/JalaliDatePicker.jsx`: a controlled
  component accepting a Gregorian ISO value (matching what the backend
  API actually expects/returns, per this epic's storage-vs-display
  principle — the component's EXTERNAL interface stays Gregorian
  ISO strings, exactly like a normal date input would be wired to form
  state, with Jalali conversion happening ENTIRELY internally for
  display/interaction purposes) and an `onChange` callback also
  receiving a Gregorian ISO string — the calling form code should never
  need to know or care that Jalali conversion is happening internally.
  Use whatever date-picker UI library this project already has
  available (check for an existing generic date-picker dependency
  first; if none exists, either build a simple custom Jalali
  calendar-grid picker, or check whether the Jalali plugin/library
  selected in Task 14.2.1.3 has an accompanying picker COMPONENT
  available, which would be the least-effort path if it exists and is
  reasonably polished).
- Replace the date inputs in the admin coupon (Epic 9 Task 9.1.1.8) and
  flash-sale creation/edit forms with this component.
- Ensure the component correctly handles the storage-vs-display
  boundary: user picks a date visually in Jalali, component converts to
  Gregorian ISO before calling `onChange`, and correctly converts BACK
  to Jalali display when initialized with an existing Gregorian ISO
  value (editing an existing coupon's dates must show the CORRECT
  Jalali date corresponding to its actual stored Gregorian value, not a
  blank/reset picker).

ACCEPTANCE CRITERIA / TESTS
Add component tests for `JalaliDatePicker`:
1. Initializing with a known Gregorian ISO value displays the correct
   corresponding Jalali date.
2. Selecting a date in the picker calls `onChange` with the correct
   Gregorian ISO equivalent of the visually-selected Jalali date.
3. Manually verify in the admin coupon/flash-sale forms: creating a
   coupon with a Jalali-picked `valid_until` date, then checking the
   raw API request payload (via browser dev tools) or the resulting
   database value, confirms the correct Gregorian date was actually
   submitted.
```

---

## Phase 14.3 — RTL & Typography

### Feature 14.3.1 — RTL Layout

---

#### Task 14.3.1.1 — Set `dir="rtl"` and `lang="fa"` on root HTML

```
You are working in frontend/index.html. Assume prior phases of this
epic are underway (this task is independent and can run any time).

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
`frontend/index.html` currently has `<html lang="en">` with no `dir`
attribute at all (defaulting to `ltr`).

TASK
Set the root HTML document's language and text direction to Persian/RTL.

REQUIREMENTS
- Change: `<html lang="fa" dir="rtl">`
- This is a small, mechanical change but has WIDE-REACHING visual
  consequences immediately upon merging — every subsequent task in this
  Feature (14.3.1.2 through 14.3.1.5) exists specifically to fix the
  layout breakage this single-line change will immediately expose. Do
  NOT expect the site to look correct after this task alone — expect
  it to look meaningfully BROKEN in places until the following tasks
  land; that's the correct, expected sequence, not a sign this task was
  done wrong.
- If the project ever needs to support toggling between LTR (English)
  and RTL (Persian) dynamically in the future (per Task 14.1.1.1's
  decision to keep English available in `LANGUAGES`), note that this
  static HTML-file approach won't support that — a future enhancement
  would need to set `dir`/`lang` dynamically via JavaScript based on
  the active language rather than hardcoding it in `index.html`. This
  is explicitly OUT OF SCOPE for this task (the backlog's stated
  target market is Iran/Persian specifically, not a toggleable
  bilingual site), but worth a code comment noting the limitation for
  future reference.

ACCEPTANCE CRITERIA / TESTS
Manually verify: opening the site in a browser now visually mirrors the
overall page layout (even if individual components look broken/
misaligned — that's expected and addressed by subsequent tasks); browser
dev tools confirm `document.documentElement.dir === "rtl"` and
`document.documentElement.lang === "fa"`.
```

---

#### Task 14.3.1.2 — Configure Tailwind RTL plugin / logical properties

```
You are working in frontend/tailwind.config.js and sweeping component
files across frontend/src. Assume Task 14.3.1.1 is already merged (and
the site is now visibly, expectedly broken in RTL-sensitive spots).

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
`tailwind.config.js` currently has no RTL-related configuration at all
— just a `content` glob and a `marquee` animation extension. Every
`ml-*`/`mr-*`/`pl-*`/`pr-*`/`left-*`/`right-*`/`text-left`/`text-right`-
style Tailwind utility class used anywhere across this codebase (likely
extensive, given 13+ epics of frontend work) is PHYSICAL-direction-based
(always "margin LEFT" regardless of text direction), not
LOGICAL-direction-based (which would correctly flip based on `dir`) —
meaning every one of these classes now renders BACKWARDS in the new RTL
layout from Task 14.3.1.1 (a left-margin that should visually appear on
the RTL "start" side instead sits on the visual "end" side).

TASK
Convert the project's spacing/positioning utility classes from
physical (`ml-`/`mr-`/`left-`/`right-`) to logical (`ms-`/`me-`/
`start-`/`end-`) Tailwind properties, which automatically flip correctly
based on the `dir` attribute.

REQUIREMENTS
- Tailwind CSS (v3.3+) has BUILT-IN support for logical properties
  (`ms-*`/`me-*` for margin-inline-start/end, `ps-*`/`pe-*` for
  padding, `start-*`/`end-*` for positioning) WITHOUT needing a
  separate plugin — verify the currently-installed Tailwind version
  (check frontend/package.json) supports this natively; if the project
  is on an OLDER Tailwind version that predates this support, either
  upgrade Tailwind (preferred, since native logical-property support is
  cleaner than a plugin-based polyfill) or add a dedicated RTL plugin
  (`tailwindcss-rtl` or similar) as the alternative — verify current
  ecosystem options before choosing, since plugin maintenance status
  can change.
- This is a LARGE, mechanical, project-wide sweep: search across every
  `.jsx` file in frontend/src for physical-direction Tailwind classes
  (`\bml-`, `\bmr-`, `\bpl-`, `\bpr-`, `\bleft-`, `\bright-`,
  `text-left`, `text-right` — construct proper regexes for this sweep)
  and replace with their logical equivalents (`ml-` → `ms-`, `mr-` →
  `me-`, `pl-` → `ps-`, `pr-` → `pe-`, `left-` → `start-`, `right-` →
  `end-`, `text-left` → `text-start`, `text-right` → `text-end`) —
  given the likely scale of this sweep across 13+ epics of accumulated
  components, treat this as a MULTI-SESSION task exactly like Task
  14.1.1.3's translation-string sweep, prioritizing the highest-traffic
  pages first (Home, product listing, PDP, cart, checkout) before
  working through admin/less-visible screens.
- BE CAREFUL: not every `left`/`right` usage is actually a
  DIRECTION-relative spacing/position concern that should flip — some
  are genuinely meant to stay FIXED regardless of text direction (e.g.
  a specific icon that's supposed to always point the same real-world
  direction regardless of RTL/LTR, which is rare but does happen —
  review each replacement rather than blind find-and-replace across the
  whole codebase without visual verification).
- Flag (but don't necessarily fix as part of THIS task, since it's
  explicitly Task 14.3.1.3's job) any icon/carousel/chevron direction
  issues you notice during this sweep — icons pointing a fixed visual
  direction (like a ">" arrow meaning "next") need explicit MIRRORING
  in RTL, which logical Tailwind classes alone don't handle (a logical
  margin flips automatically; an SVG chevron pointing right does NOT
  automatically become a chevron pointing left just because the layout
  direction changed) — note these locations for the next task rather
  than fixing them here, to keep this task's scope to spacing/
  positioning specifically.

ACCEPTANCE CRITERIA / TESTS
- No automated test can fully verify "the layout looks visually
  correct in RTL" — this task is verified primarily through MANUAL
  visual review of each swept page/component in the actual RTL-rendered
  browser (per Task 14.3.1.1's change), checking that spacing/alignment
  reads correctly right-to-left.
- Where feasible, add a lightweight automated check: a test asserting
  NO remaining physical-direction Tailwind classes exist in the swept
  files (a simple grep-based test/lint rule checking `git grep` results
  for the target patterns return zero matches in the files you've
  completed) — this at least catches regressions/incomplete sweeps
  going forward, even though it can't verify visual correctness.
```

---

#### Task 14.3.1.3 — Audit and fix icon/layout mirroring (arrows, carousels)

```
You are working across frontend/src component files with directional
icons/carousels. Assume Task 14.3.1.2 is already merged (and has
flagged specific locations needing mirroring attention).

CONTEXT
Per Task 14.3.1.2's note: logical Tailwind spacing classes correctly
flip automatically in RTL, but VISUAL ICON DIRECTION does not — a
chevron/arrow icon meaning "next"/"forward" that visually points right
in LTR needs to visually point LEFT in RTL to retain the same MEANING
("forward," in reading-direction terms), even though the underlying
SVG/icon glyph itself doesn't know or care about text direction on its
own.

TASK
Find and fix every directionally-meaningful icon (carousel next/prev
arrows, breadcrumb separators, "read more" chevrons, any icon implying
motion/sequence) so it correctly mirrors in RTL while preserving its
intended MEANING.

REQUIREMENTS
- Locate every such icon across the codebase — likely candidates,
  based on a typical e-commerce app's structure: product image
  carousels/sliders on the PDP, homepage banner carousels, breadcrumb
  navigation separators, pagination prev/next controls, any "view all →"
  style link chevrons, checkout step-progress indicators. Use Task
  14.3.1.2's flagged locations as a starting checklist, then do your
  own sweep for anything missed (search for `bi-chevron`,
  `bi-arrow`, or whatever icon library class-naming convention this
  project uses — confirmed earlier as Bootstrap Icons via the CDN link
  in `index.html` — for `-left`/`-right`-suffixed icon classes
  specifically, e.g. `bi-chevron-right`, `bi-arrow-left`, which are the
  icon-library's own physically-named variants and are exactly the ones
  needing conditional flipping).
- Fix approach: for each such icon, either (a) use CSS to conditionally
  mirror it based on the `dir` attribute — a simple, broadly-applicable
  Tailwind/CSS rule like `[dir="rtl"] .icon-directional { transform: scaleX(-1); }`
  applied via a shared utility class added to every directionally-
  meaningful icon, which is the LEAST invasive fix requiring minimal
  per-component changes, OR (b) conditionally render the OPPOSITE
  icon variant based on direction (e.g.
  `dir === 'rtl' ? 'bi-chevron-left' : 'bi-chevron-right'`) if a CSS
  mirror transform would look wrong for a specific icon (some icons —
  e.g ones containing readable numerals or asymmetric detail — don't
  mirror cleanly via a simple CSS flip and need the conditional-icon-
  swap approach instead; review each on a case-by-case basis rather
  than applying one blanket technique everywhere).
- Carousels/sliders specifically often need BOTH the icon direction
  AND the actual SLIDE-ADVANCE direction logic corrected — verify
  whichever carousel library/implementation this project uses (check
  for an existing carousel component/library dependency) correctly
  advances in the RTL-appropriate direction when "next" is clicked, not
  just that the ARROW ICON looks right while the carousel itself still
  advances the "wrong" (LTR-logic) way — this is a common, easy-to-miss
  RTL bug distinct from pure icon mirroring.

ACCEPTANCE CRITERIA
Primarily manual visual verification, given the nature of this task:
walk through every identified directional-icon location in the actual
RTL-rendered browser and confirm both the icon's visual direction AND
its underlying behavior (carousel advance direction, breadcrumb reading
order) correctly match RTL reading expectations. Where a component has
existing automated tests, add/update assertions confirming the correct
icon CLASS/variant is rendered when `dir="rtl"` is active (for the
conditional-icon-swap cases specifically — the CSS-mirror cases aren't
meaningfully unit-testable beyond confirming the mirror class is
applied).
```

---

#### Task 14.3.1.4 — Integrate Persian web font (Vazirmatn/Yekan)

```
You are working in frontend/ (font files, tailwind.config.js, a global
CSS file). Assume this task is independent of the rest of Phase 14.3
and can run any time.

CONTEXT
No Persian-optimized web font is loaded anywhere — the site currently
relies on whatever default system/browser font stack applies, which
typically renders Persian script poorly or inconsistently across
different operating systems/browsers compared to a font specifically
designed for Persian/Arabic script legibility.

TASK
Add a self-hosted Persian web font (Vazirmatn is a strong, actively-
maintained, open-source, widely-used choice for Persian UI text —
confirm it's still the best current option, or an equally strong
alternative like Yekan Bakh, before committing, since font ecosystem
recommendations can shift) and configure it as the project's default
typeface.

REQUIREMENTS
- Self-host the font files (rather than loading from a third-party CDN)
  — download the Vazirmatn variable font file(s) (WOFF2 format,
  covering the weight range this project's design actually uses — check
  existing usage across the codebase for which font-weight utility
  classes, e.g. `font-bold`/`font-semibold`, are actually used, and only
  include the weights genuinely needed rather than bloating the
  download with every possible weight) and place them in
  frontend/public/fonts/ (or wherever this project's existing static-
  asset convention places such files).
- Add a `@font-face` declaration (in a new or existing global CSS file,
  e.g. frontend/src/index.css) with `font-display: swap` (avoiding a
  flash-of-invisible-text while the font loads, standard web
  performance best practice).
- Configure Tailwind's `theme.fontFamily` in tailwind.config.js to make
  the new font the DEFAULT sans-serif stack, with a sensible fallback
  chain for any moment before the font loads or in case it fails to
  load: `fontFamily: { sans: ['Vazirmatn', 'Tahoma', 'system-ui', 'sans-serif'] }`.
- Verify the font actually renders correctly for BOTH Persian text AND
  any Latin-script text that appears alongside it (product SKUs, prices
  in Western digits per Task 14.1.2.3's default, English admin labels)
  — Vazirmatn (and similar well-designed Persian fonts) typically
  includes reasonable Latin glyph coverage too, but confirm visually
  rather than assuming.

ACCEPTANCE CRITERIA
Manually verify: Persian text across the site renders using the new
font (confirm via browser dev tools' computed-styles inspector showing
the correct `font-family`, not just eyeballing it), the font loads
correctly on a fresh page load with no flash-of-unstyled/invisible text
issues, and page-load performance (check Network tab for font file
size/load time) remains reasonable — a bloated multi-hundred-KB font
file covering every possible weight/style would be a real, avoidable
performance regression worth catching before considering this done.
```

---

#### Task 14.3.1.5 — RTL visual regression test pass (manual checklist)

```
You are working across the entire frontend/. Assume Tasks 14.3.1.1
through 14.3.1.4 are all merged.

CONTEXT
RTL layout correctness is fundamentally a VISUAL concern that automated
component tests can only partially verify (per the acceptance-criteria
notes in Tasks 14.3.1.2/14.3.1.3) — this task is a deliberate,
comprehensive manual QA pass across every major page, treating RTL
layout correctness as seriously as any functional bug, since a broken
RTL layout on launch would be immediately, visibly embarrassing on a
platform whose entire target market reads right-to-left.

TASK
Produce and execute a written QA checklist covering every major page/
flow in the application, verified in the actual RTL-rendered browser,
documenting any remaining issues found for follow-up.

REQUIREMENTS
- Create a QA checklist document (e.g.
  frontend/docs/rtl-qa-checklist.md) covering, at minimum:
  Home page (hero/banner, product grids, flash sale countdown from
  Epic 9, "Recommended For You" from Epic 13), product listing/filter
  sidebar (Epic 12's cosmetics facets), product detail page (image
  gallery/carousel, variant selector, reviews section including images
  from Epic 10, "Related Products"/"Customers Also Bought" from Epic
  13), cart page (including the coupon-apply UI from Epic 9), checkout
  flow (address form, saved-address picker from Epic 5, shipping
  carrier selection from Epic 7, payment redirect flow from Epic 6),
  order confirmation/history/tracking widget (Epic 7), account/profile
  pages, wishlist (Epic 11), admin dashboard's key screens (order
  management, coupon management) — this list should be adjusted/
  extended to reflect whatever's ACTUALLY been built by this point in
  your real implementation order, since not every epic listed here may
  have landed yet depending on how you're sequencing this backlog.
- For EACH page/flow, verify: text alignment reads naturally
  right-to-left, spacing/margins look intentional (not mirrored-wrong
  or oddly asymmetric), icons point the correct semantic direction (per
  Task 14.3.1.3), form inputs and their labels align correctly, modal/
  dropdown positioning doesn't render off-screen or overlapping
  incorrectly, and any horizontally-scrolling content (carousels,
  tables) scrolls in the RTL-correct direction.
- Document every issue found with enough detail (page, component,
  screenshot if your environment supports capturing one, expected vs.
  actual) for it to be fixed as a follow-up task WITHOUT needing to
  re-discover/re-diagnose it from scratch — this checklist run is
  explicitly expected to surface real, still-outstanding issues (Tasks
  14.3.1.1–14.3.1.4 are thorough but can't realistically catch
  everything given the scale of this project by this point), and the
  VALUE of this task is producing that documented, actionable list, not
  necessarily fixing every single item found within this same task's
  time budget.
- Fix whatever SMALL, clearly-scoped issues you find time to address
  within this task's session; explicitly flag anything larger as a
  separate follow-up task rather than letting this task's scope balloon
  trying to fix everything discovered.

ACCEPTANCE CRITERIA
The completed, executed checklist document itself is this task's
primary deliverable, along with fixes for whatever small issues were
addressed inline. A clean run with zero issues found would be a
genuinely surprising, excellent outcome given the scale of prior RTL
work — more realistically, expect and plan for a real, prioritized
follow-up list to come out of this task.
```

---

## Phase 14.4 — Slug & URL Strategy

### Feature 14.4.1 — Persian-Friendly URLs

---

#### Task 14.4.1.1 — Decide slug strategy: transliteration vs. Persian-script slugs

```
You are working on an architecture decision, documented alongside Task
14.1.2.1's ADR (backend/docs/adr/0002-slug-strategy.md).

CONTEXT — A REAL, CONFIRMED BUG THIS DECISION MUST ADDRESS
Per this document's header context: `Product.save()`/`Category.save()`/
`Brand.save()` currently call Django's `slugify(self.name)` WITHOUT
`allow_unicode=True`, meaning Persian product names produce
effectively empty/garbage slugs TODAY, right now, for any Persian-named
product already in the catalog. This decision task determines the FIX
direction; Task 14.4.1.2 implements it.

TASK
Decide: should product/category/brand slugs be (a) Latin-script
TRANSLITERATIONS of the Persian name (e.g. "سرم ویتامین سی" →
"serum-vitamin-c"), or (b) native Persian-script slugs (e.g.
"سرم-ویتامین-سی", using `allow_unicode=True` to permit the actual
Persian characters directly in the URL)?

REQUIREMENTS — the decision to document (and the recommended answer)
- **Recommend: Latin-script transliteration**, for these reasons,
  which should be weighed and documented explicitly:
  1. **URL-sharing compatibility**: Persian-script URLs, while
     technically valid (modern browsers/servers handle Unicode URLs via
     IDNA/percent-encoding transparently), often display as ugly
     percent-encoded gibberish (`%D8%B3%D8%B1%D9%85-...`) when
     copy-pasted into certain contexts (older chat apps, some social
     media platforms, some SMS gateways, plain-text email clients) —
     genuinely degrading the "clean, shareable URL" benefit that slugs
     exist to provide in the first place.
  2. **SEO tooling compatibility**: many SEO analysis tools, some
     analytics platforms, and some ad-platform URL-tracking parameter
     conventions have historically had rougher, less reliable support
     for non-ASCII URLs — a transliterated URL sidesteps this whole
     category of friction entirely, which matters directly for Epic
     15's upcoming SEO work.
  3. Persian-script URLs ARE still fully valid and increasingly
     well-supported — this is a genuine tradeoff, not a clear-cut
     right/wrong answer, and Persian-script URLs are a perfectly
     legitimate choice many real Persian-language sites make
     successfully; document that the alternative was considered
     seriously, not dismissed.
- Document the chosen TRANSLITERATION approach's scope: a
  transliteration library/mapping must handle the full Persian
  alphabet plus common product-naming conventions (numbers, Latin
  brand names already embedded in Persian product names — e.g. "سرم
  ویتامین C" mixing Persian and a literal Latin "C," which should stay
  as "C" in the transliterated slug, not be mangled) — flag this mixed-
  script handling requirement explicitly for Task 14.4.1.2's
  implementation to address, since naive transliteration libraries
  sometimes mishandle mixed-script input.

ACCEPTANCE CRITERIA
This task's deliverable is the ADR document, reviewed and accepted,
providing Task 14.4.1.2 with an unambiguous implementation direction.
```

---

#### Task 14.4.1.2 — Implement Persian-to-Latin transliteration slugify function

```
You are working in backend/shop/models.py (or a new
backend/shop/text_utils.py, if Task 12.1.1.2's Persian text-utilities
module already exists — extend that same module for consistency rather
than creating a second, separate text-utilities file). Assume Task
14.4.1.1's decision (transliteration) is finalized.

CONTEXT — THIS FIXES A LIVE, CONFIRMED BUG
Per this document's header: `Product.save()` currently does
`self.slug = slugify(self.name)` — Django's default `slugify()`,
called WITHOUT `allow_unicode=True`, strips every non-ASCII character.
For a Persian product name, this produces a near-empty or nonsensical
slug TODAY. This task replaces that call with a real Persian-to-Latin
transliteration function.

TASK
Write a `persian_slugify()` function transliterating Persian text into
a clean, readable, Latin-script slug, and use it everywhere the
codebase currently calls plain `slugify()` on user-facing Persian
content.

REQUIREMENTS
- Either (a) use an existing, well-maintained Python transliteration
  library with reasonable Persian support (search for one — options in
  this space include general-purpose transliteration libraries; verify
  CURRENT maintenance status and actual Persian-script accuracy before
  committing to one, since not every general "transliterate any
  script" library handles Persian's specific letterforms/diacritics
  well) or (b) implement a direct Persian-character-to-Latin mapping
  table if no sufficiently well-maintained/accurate library is found —
  a hand-built mapping table is more work but gives full control over
  output quality and avoids a dependency on a library that might handle
  Persian as an afterthought; make this call based on what you actually
  find, and document your choice and reasoning.
- Whichever approach, the function must:
  ```python
  def persian_slugify(text: str) -> str:
      """
      Transliterate Persian text into a clean, URL-safe Latin-script
      slug. Mixed Persian/Latin input (e.g. a Persian product name
      containing an embedded Latin brand name or letter grade like "C")
      is handled correctly: existing Latin/ASCII characters and digits
      pass through unchanged; Persian characters are transliterated;
      the two are correctly concatenated with appropriate hyphenation.
      """
  ```
  - Preserve any ALREADY-Latin/ASCII substrings within a mixed-script
    input UNCHANGED (per Task 14.4.1.1's explicit requirement) rather
    than running them through transliteration too (which could mangle
    them) — this likely requires processing the input token-by-token
    (splitting on whitespace/punctuation) rather than character-by-
    character, transliterating only the tokens that are actually
    Persian script and leaving Latin/numeric tokens as-is.
  - Handle Persian numerals (۰-۹) appearing in a product name by
    converting them to Western digits in the slug (a slug convention
    choice — Western digits are more universally URL-safe/readable
    than Persian numeral Unicode code points in a URL context,
    regardless of Task 14.1.2.4's separate, DISPLAY-only digit-style
    toggle, which doesn't apply to URLs).
  - Apply the SAME Persian character NORMALIZATION already built in
    Epic 12 Task 12.1.1.2 (Arabic ك/ي variants → Persian ک/ی, strip
    zero-width characters) BEFORE transliterating — reuse that exact
    `normalize_persian_text()` function rather than re-implementing
    normalization logic a second time; import it from wherever Epic 12
    placed it.
  - Lowercase the final result and use hyphens as the word separator,
    matching standard slug conventions (consistent with how Django's
    own `slugify()` behaves for the Latin-script portions).
- Replace EVERY current call to plain `slugify(self.name)` across
  `Product.save()`, `Category.save()`, `Brand.save()` (and any other
  model with a similar slug-generation pattern — grep for `slugify(`
  across the whole backend to find all call sites) with
  `persian_slugify(self.name)`, PRESERVING the existing collision-
  avoidance loop pattern exactly (the `while Product.objects.filter(slug=slug).exclude(pk=self.pk).exists(): ...`
  loop already confirmed present in `Product.save()` — only the
  base-slug GENERATION function changes, not the uniqueness-loop logic
  around it).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_text_utils.py (or wherever Epic
12's equivalent tests live, for consistency):
1. `persian_slugify("سرم ویتامین سی")` (a purely Persian product name)
   produces a non-empty, readable, hyphenated Latin-script slug (assert
   the EXACT expected output once you've determined what your chosen
   transliteration approach actually produces for this specific
   string — don't just assert non-emptiness, verify actual correctness
   against a real, considered expected value).
2. `persian_slugify("سرم ویتامین C")` (MIXED Persian + a literal Latin
   "C") preserves the "C" unchanged in the output, correctly
   transliterating only the Persian portion — this is the single most
   important test in this task, directly verifying the mixed-script
   requirement flagged in Task 14.4.1.1.
3. `persian_slugify()` applied to a product name containing Persian
   numerals correctly converts them to Western digits in the slug.
4. Re-generate slugs for any EXISTING Persian-named seed/test products
   (via a data migration re-saving them, mirroring the pattern already
   established for other data-migration tasks in this epic) and confirm
   they now produce meaningful, non-empty slugs — this is the actual
   FIX verification for the confirmed live bug this task addresses; a
   before/after comparison against the CURRENT broken output is the
   clearest way to prove this task actually fixed the real problem.
5. The existing collision-avoidance uniqueness loop still works
   correctly with the new slugify function — two products with
   identical (or transliterate-to-identical) Persian names still
   receive distinct, non-colliding slugs (`-1`, `-2`, etc., matching
   the existing pattern).
```

---

#### Task 14.4.1.3 — Backfill slugs for future Persian product names

```
You are working in backend/shop/management/commands/seed_shop.py and
any other management command/fixture-loading code that creates
Product/Category/Brand instances. Assume Task 14.4.1.2 is already
merged.

CONTEXT
`seed_shop.py` (this project's existing seed/demo-data management
command, confirmed present from earlier grounding, already includes at
least one Persian-relevant category — "Beauty & Personal Care" — though
likely with mostly English demo content given the project's original
English-language template origins) and any other data-loading code
paths need to be confirmed to correctly exercise the NEW
`persian_slugify()` function rather than any lingering reference to the
old plain `slugify()`.

TASK
Audit and update `seed_shop.py` (and any other seed/fixture/import code
path that creates slugged models) to use realistic Persian product/
category/brand names as demo data, confirming the new slugify function
produces correct results when exercised through the actual seeding
workflow, not just in isolated unit tests.

REQUIREMENTS
- Review `seed_shop.py`'s current demo data — since this task follows
  directly from fixing a real transliteration bug, this is a natural,
  low-risk moment to also improve the QUALITY of the project's demo/
  seed data by making it genuinely representative of the target market
  (real, realistic Persian cosmetics product/category/brand names)
  rather than leaving it as generic English placeholder content
  inherited from the project's original template — this isn't
  strictly required by the task's narrow technical scope, but is a
  valuable, low-cost improvement worth making while already touching
  this file; use judgment on how much demo-data rewriting is
  proportionate here versus staying narrowly focused on the slug-
  generation verification.
- Confirm the seed command, when run against a fresh database, produces
  correctly-transliterated slugs for every Persian-named seed
  product/category/brand (this is your primary END-TO-END verification
  that Task 14.4.1.2's function works correctly through the REAL
  application code path, not just in isolated unit tests that call
  `persian_slugify()` directly).
- Check for and update any OTHER command/import/fixture-loading code
  path across the backend that creates slugged model instances (search
  for any additional `manage.py` custom commands, Django fixture JSON
  files, or CSV-import tooling from Epic 17's future bulk-import work if
  it's landed by this point) to ensure none of them bypass the model's
  `save()` method in a way that would skip slug generation entirely
  (e.g. `bulk_create()` calls, which — as established in several prior
  epics' Celery/signal-related tasks — do NOT trigger a model's custom
  `save()` override, meaning any bulk-creation code path would need
  EXPLICIT separate slug-generation logic rather than relying on
  `save()` to handle it automatically; audit for this specific gap).

ACCEPTANCE CRITERIA / TESTS
- Running `python manage.py seed_shop` (or whatever the actual seed
  command's name/invocation is) against a fresh database produces a
  catalog where every product/category/brand has a correct, non-empty,
  meaningful, properly-transliterated slug — manually inspect a sample
  of the seeded data to confirm.
- Add a test running the ACTUAL seed command (or a representative
  subset of its logic) and asserting no resulting Product/Category/
  Brand has an empty or obviously-broken slug (e.g. assert
  `Product.objects.filter(slug="").exists() is False` and spot-check a
  few specific expected slug values for known seed product names).
- If any `bulk_create()`-based code path was found during the audit
  that bypasses slug generation, fix it (either by switching that path
  to iterate-and-`.save()` instead of `bulk_create()`, accepting the
  performance tradeoff for what should be an infrequent
  admin/import operation, or by explicitly pre-computing slugs before
  the bulk-create call) and add a regression test proving that specific
  path now generates correct slugs too.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 14.1.1.1 | Set LANGUAGE_CODE="fa", configure LANGUAGES | ☐ |
| 14.1.1.2 | Set TIME_ZONE="Asia/Tehran" | ☐ |
| 14.1.1.3 | Wrap translatable strings in gettext_lazy | ☐ |
| 14.1.1.4 | Generate/populate fa .po/.mo translation files | ☐ |
| 14.1.2.1 | Decide currency storage unit (ADR) | ☐ |
| 14.1.2.2 | Migrate monetary DecimalFields to integer Rial | ☐ |
| 14.1.2.3 | Toman display formatting utility | ☐ |
| 14.1.2.4 | Persian digit rendering toggle | ☐ |
| 14.2.1.1 | Add django-jalali/jdatetime to backend | ☐ |
| 14.2.1.2 | Jalali date display in Django admin | ☐ |
| 14.2.1.3 | Frontend Jalali date library integration | ☐ |
| 14.2.1.4 | Jalali date picker component | ☐ |
| 14.3.1.1 | Set dir="rtl" and lang="fa" on root HTML | ☐ |
| 14.3.1.2 | Configure Tailwind RTL / logical properties | ☐ |
| 14.3.1.3 | Audit and fix icon/layout mirroring | ☐ |
| 14.3.1.4 | Integrate Persian web font (Vazirmatn) | ☐ |
| 14.3.1.5 | RTL visual regression test pass | ☐ |
| 14.4.1.1 | Decide slug strategy (ADR) | ☐ |
| 14.4.1.2 | Implement Persian-to-Latin transliteration slugify | ☐ |
| 14.4.1.3 | Backfill slugs / audit bulk-create code paths | ☐ |

Once Epic 14 is fully merged, the next epic to generate prompts for is
**Epic 15 — SEO**, which directly depends on this epic's slug strategy
(Task 14.4.1.1/14.4.1.2) and Persian localization (Task 14.1.1.3/4)
being in place before meaningful Persian-language meta content/
structured data can be built.
