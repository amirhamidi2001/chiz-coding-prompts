# Epic 24 — Testing — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–23 are fully merged.

**Important grounded discovery for this epic — read before starting Tasks 24.1.1.1 through 24.1.1.3:** per the master backlog's own stated principle — critical-path tests should be written incrementally alongside each dependent epic as it ships, not batched at the end — **this document series has actually followed that principle throughout**, meaning substantial real coverage for three of this epic's five listed tasks **already exists**, built directly into earlier epics' acceptance criteria:

- **Task 24.1.1.1 ("end-to-end checkout test")**: Epic 6 Task 6.2.1.7 already built a comprehensive ZarinPal integration test suite covering the full checkout → initiate → callback → order-confirmed flow, including payment-failure stock release, customer-cancellation, double-initiate rejection, and gateway-network-failure scenarios.
- **Task 24.1.1.2 ("coupon abuse test suite")**: Epic 9 Tasks 9.1.1.4, 9.1.1.5, and 9.1.1.7 already built extensive adversarial coverage — expired/inactive/not-yet-valid coupons, `max_uses`/`uses_per_user` exhaustion (with explicit in-progress-vs-completed redemption distinction), unauthenticated-guest rejection, category/product-restriction bypass attempts, and cancellation-frees-the-coupon behavior.
- **Task 24.1.1.3 ("OTP flow abuse tests")**: Epic 2 Task 2.3.1.4 already built a dedicated integration suite covering wrong-code, expired-code, max-attempts-exceeded, and request-rate-limiting scenarios end-to-end.

**This means Tasks 24.1.1.1–24.1.1.3 below are audit-and-consolidate tasks, not greenfield builds** — their real job is verifying that existing coverage is genuinely complete and current (a lot of epics have touched the underlying code since each of those tests was written), closing any real gaps found, and making this coverage clearly discoverable as "the critical-path suite" rather than scattered findable-only-by-searching-old-epics'-work. **Tasks 24.1.1.4 and 24.1.1.5 are genuinely new** — confirmed via `frontend/package.json` and `backend/requirements.txt`, this project has **no E2E browser-testing tool (Playwright/Cypress) and no load-testing tool (k6/Locust) anywhere** — these two are real, from-scratch infrastructure additions.

---

## Phase 24.1 — Coverage Gaps

### Feature 24.1.1 — Critical-Path Test Coverage

---

#### Task 24.1.1.1 — End-to-end checkout test (cart → payment → order confirmed)

```
You are working in backend/payments/tests/ (Epic 6 Task 6.2.1.7's
existing integration suite) and, if warranted, a new consolidated
location. Assume Epics 1–23 are fully merged.

CONTEXT — READ THIS DOCUMENT'S HEADER FIRST
Epic 6 Task 6.2.1.7 built a full checkout→payment→confirmation
integration suite. Since that task shipped, SIGNIFICANT parts of the
checkout flow it exercises have changed: Epic 7 added mandatory
shipping-carrier/rate selection to `OrderCreateSerializer` (Task
7.1.1.4 — checkout now REQUIRES `shipping_carrier_id`/`shipping_rate_id`,
fields that didn't exist when Epic 6's test suite was written), Epic 9
added optional coupon application, Epic 14 migrated every monetary
field from `Decimal` to integer Rial, and Epic 5 added guest/session-
based checkout as an entirely parallel path alongside authenticated
checkout. There is a REAL risk this suite's test payloads are now
STALE — either failing outright (if the new required shipping fields
were never added to its test fixtures) or, worse, silently passing
while no longer exercising realistic checkout payloads.

TASK
Audit Epic 6 Task 6.2.1.7's existing end-to-end checkout suite against
the CURRENT, full checkout contract, fix any staleness found, and
extend it to cover the scenarios that have been added since (guest
checkout, coupon-applied checkout) which weren't in scope when that
suite was originally written.

REQUIREMENTS
- Run the EXISTING suite first, as-is, against the current codebase —
  do not assume it currently passes; this is a genuine verification
  step, not a formality. If it fails, diagnose whether the failure
  reflects a genuine regression (a real bug introduced by a later
  epic) or simply a stale test fixture missing a now-required field
  (e.g. missing `shipping_carrier_id`/`shipping_rate_id`, per Epic 7)
  — fix test fixtures for the latter case, and treat the former as a
  genuine bug requiring its own investigation and fix, not something
  to paper over by loosening the test.
- Update every test payload in the suite to include the FULL current
  checkout contract: address (with Epic 5's `address_id`-or-manual-
  fields pattern), shipping carrier/rate selection (Epic 7), optional
  coupon (Epic 9), and confirm all monetary assertions use plain
  integer Rial comparisons (Epic 14), not stale `Decimal("X.XX")`-
  shaped expected values.
- Extend the suite with scenarios NOT covered by the original Epic 6
  task (since those didn't exist as concepts yet when it was written):
  1. **Guest checkout, full flow**: an anonymous session (per Epic 5
     Feature 5.1.1) builds a cart, proceeds through checkout with
     manually-entered address fields (no saved `address_id`, since a
     guest has none), selects a shipping option, completes payment,
     and the resulting order correctly has `user=None` with the
     shipping-address-derived contact info used for the order-
     confirmation notification (per Epic 16 Task 16.2.1.2's guest-
     aware `notify()` call).
  2. **Checkout with a coupon applied**: cart has a valid coupon
     applied (Epic 9 Task 9.1.1.6), checkout completes, the resulting
     `Order.discount` and `CouponRedemption` are correct, and the
     amount actually sent to the payment gateway (mock and inspect the
     `request_payment()` call's `amount` argument) reflects the
     POST-DISCOUNT total, not the pre-discount subtotal — a real,
     financially-important assertion worth being explicit about.
  3. **Full flow with an authenticated user's SAVED address** (Epic 5
     Task 5.2.1.3's `address_id` path) rather than manually-entered
     fields, confirming the order's shipping snapshot correctly
     matches the saved address's data.
  4. **A combined worst-case**: guest checkout is NOT combinable with a
     saved address by definition, but confirm an AUTHENTICATED
     checkout combining a saved address AND an applied coupon AND a
     specific shipping carrier selection all work together correctly
     in one single request — the realistic "everything at once" case a
     real customer's checkout actually looks like, which no single
     existing test necessarily covers in combination even if each
     piece is separately tested elsewhere.
- Re-run the FULL suite (existing + new scenarios) together and confirm
  everything passes against the current, complete codebase.

ACCEPTANCE CRITERIA / TESTS
The updated, verified, extended test suite itself is this task's
deliverable. Produce a brief summary (in your task response) of: what
was found stale and fixed, whether any genuine regression was
discovered and how it was resolved, and confirmation the full combined
suite passes cleanly against the current codebase.
```

---

#### Task 24.1.1.2 — Coupon abuse test suite (reuse, expired, over-cap)

```
You are working in backend/promotions/tests/ (Epic 9's existing test
suites across Tasks 9.1.1.4/9.1.1.5/9.1.1.7). Assume Epics 1–23 are
fully merged.

CONTEXT — READ THIS DOCUMENT'S HEADER FIRST
Epic 9's coupon validation service and its tests already cover the
core abuse scenarios explicitly. This task's real job: confirm that
coverage remains accurate given `validate_coupon()`'s signature
changed TWICE since it was first written (Epic 9 Task 9.1.1.4's
initial `(code, user, subtotal)` signature, then Task 9.1.1.7's
breaking change to `(code, user, cart)` to support category/product
restrictions) — and add abuse scenarios that specifically target the
INTERACTION between coupons and OTHER systems built in LATER epics,
which wouldn't have been in scope when Epic 9 itself was written.

TASK
Audit Epic 9's existing coupon abuse coverage for currency (post the
signature changes), and add cross-system abuse scenarios that only
became possible/relevant once later epics landed.

REQUIREMENTS
- Confirm every existing test in Epic 9's suites was correctly updated
  when `validate_coupon()`'s signature changed to accept `cart` instead
  of `subtotal` (per Task 9.1.1.7's own acceptance criteria, which
  explicitly called for this) — verify this actually happened rather
  than assuming; if any test was missed in that migration and is
  currently broken or silently testing against a shim/wrapper rather
  than the real current signature, fix it now.
- Add these ADDITIONAL abuse scenarios, each targeting a specific
  cross-epic interaction:
  1. **Coupon + guest checkout combination attempt**: per Epic 9's own
     design decision (documented in Task 9.1.1.3), coupon redemption
     is authenticated-only — confirm a GUEST (per Epic 5's session-
     based cart) attempting to apply ANY coupon code is correctly
     rejected with the "please log in" message at the CART-application
     endpoint (Task 9.1.1.6), not just at final checkout — verify this
     is actually enforced at BOTH points consistently, not just one.
  2. **Race condition on `max_uses`**: two DIFFERENT users
     simultaneously attempting to redeem the LAST remaining use of a
     coupon with `max_uses` nearly exhausted — construct a concurrency
     test (mirroring the exact `TransactionTestCase` + threading
     pattern established in Epic 1 Task 1.1.1.5's stock-concurrency
     test) proving the coupon's usage limit is genuinely enforced under
     concurrent load, not just in a single-threaded test — check
     whether `validate_coupon()`'s usage-count check
     (`CouponRedemption.objects.filter(...).count() >= coupon.max_uses`)
     has the SAME class of race-condition vulnerability Epic 1 Task
     1.1.1.2 originally fixed for stock (a check-then-act race with no
     row locking) — if this test reveals the coupon system has NEVER
     actually been protected against this specific race (a real,
     newly-discovered gap, not previously flagged anywhere in this
     document series), this is a genuine bug to fix as part of this
     task, adding `select_for_update()`-based locking to the coupon
     redemption-counting/creation path in
     `OrderCreateSerializer.create()`'s coupon-handling section (Epic 9
     Task 9.1.1.5), mirroring the exact locking pattern already
     established for stock.
  3. **Coupon amount discount does not interact incorrectly with the
     currency migration**: apply a percentage coupon and confirm the
     computed `discount_amount` (Epic 9 Task 9.1.1.4's calculation)
     produces a correctly-ROUNDED whole-Rial integer (per Epic 14's
     "round at every step" invariant established in that epic's
     currency ADR) — not a fractional value that would be silently
     truncated somewhere downstream, and not a value that drifts from
     what `order/services/pricing.py`'s `calculate_order_totals()`
     independently computes when handling the same discount.
  4. **Cancelling an order twice**: Epic 9 Task 9.1.1.5's "cancellation
     frees the coupon" behavior deletes the `CouponRedemption` on
     cancellation — confirm attempting to cancel an ALREADY-cancelled
     order (which Epic 8 Task 8.1.1.1's state-machine validation
     should already reject as an invalid transition) doesn't somehow
     attempt to delete a redemption a SECOND time or otherwise misbehave.

ACCEPTANCE CRITERIA / TESTS
All listed scenarios are implemented as real, passing tests. If the
concurrency test in scenario 2 reveals a genuine, previously-unfixed
race condition, the corresponding fix (with `select_for_update()`
locking added to the real coupon-redemption code path) must be
implemented and verified as part of completing this task — a
discovered vulnerability with no fix is not an acceptable outcome for
a task in the SECURITY-adjacent "abuse testing" category.
```

---

#### Task 24.1.1.3 — OTP flow abuse tests (rate limit, expired, brute force)

```
You are working in backend/accounts/tests/ (Epic 2 Task 2.3.1.4's
existing integration suite) and backend/notifications/ (Epic 16's
relocation of the SMS provider module). Assume Epics 1–23 are fully
merged.

CONTEXT — READ THIS DOCUMENT'S HEADER FIRST
Epic 2 Task 2.3.1.4 already built comprehensive OTP abuse coverage.
This task's real job: confirm that coverage survived Epic 16 Task
16.1.1.1's RELOCATION of the entire `SMSProvider`/`ConsoleSMSProvider`
module from `accounts.sms` to `notifications.sms` (a pure move-and-
rename that task explicitly required re-running "the ENTIRE Epic 2 OTP
test suite... updated only for the new import paths" — verify this
actually happened correctly and nothing was silently left broken or
skipped), and add abuse scenarios specific to Epic 16's SMS-retry
mechanism (Task 16.2.1.4), which introduces new behavior an attacker
could potentially target that didn't exist when Epic 2's original
suite was written.

TASK
Audit Epic 2's OTP abuse suite for correctness after the Epic 16
relocation, and add abuse scenarios targeting the SMS-retry mechanism
specifically.

REQUIREMENTS
- Run the full existing OTP abuse suite and confirm every test passes
  using the CURRENT `notifications.sms` module paths (post-relocation)
  — if any test still references the old `accounts.sms` paths (a sign
  the relocation task's own migration wasn't fully completed), fix it
  now.
- Add abuse scenarios targeting Epic 16 Task 16.2.1.4's SMS retry
  mechanism:
  1. **Retry doesn't bypass the underlying OTP expiry/attempt limits**:
     confirm that Epic 16's `retry_sms_notification` Celery task
     (which retries FAILED SMS SENDS, e.g. a transient Kavenegar
     outage) operates entirely independently of, and never interferes
     with, `OTPCode`'s own expiry/attempt-count logic (Epic 2 Task
     2.1.2.3) — a retry mechanism for DELIVERY failures should have
     zero interaction with the VERIFICATION side of OTP; confirm this
     boundary is genuinely clean by constructing a scenario where an
     OTP send initially fails and gets queued for retry, then the OTP
     naturally expires (per its normal TTL) BEFORE the retry
     successfully delivers it — confirm the customer correctly can't
     use the (now-expired, even if eventually delivered) code, proving
     the retry mechanism doesn't accidentally extend a code's effective
     lifetime.
  2. **Retry queue can't be abused to generate unlimited SMS sends
     against a phone number**: confirm Epic 2 Task 2.1.2.4's
     `PhoneOTPRequestThrottle` (limiting OTP REQUEST frequency per
     phone number) is NOT bypassable by triggering the failure/retry
     path repeatedly — the retry mechanism operates on a SPECIFIC
     already-generated code's delivery, not on generating NEW codes,
     so this should already be structurally safe, but construct an
     explicit test confirming a scenario where SMS delivery keeps
     failing (forcing repeated retries) doesn't somehow let an attacker
     cause more actual SMS message sends than the request-rate-limit
     would normally allow for genuinely successful sends.
- Consider whether the OTP flow's interaction with Epic 23 Task
  23.1.1.2's `django-axes` lockout mechanism needs any test coverage
  here — per that task's OWN explicit scoping decision, axes was
  DELIBERATELY NOT applied to `OTPVerifyView` (relying instead on
  OTP's own existing attempt-limit mechanism) — add a regression test
  here confirming that decision holds correctly: OTP verification
  failures do NOT trigger axes' account-lockout mechanism (which would
  be redundant with, and potentially conflict with, the OTP-specific
  attempt-limit already in place) — this is exactly the kind of
  cross-epic boundary worth explicit, permanent test coverage rather
  than trusting a settings-file scoping decision to hold forever
  without verification.

ACCEPTANCE CRITERIA / TESTS
All listed scenarios are implemented as real, passing tests, using the
current, post-relocation module paths throughout.
```

---

#### Task 24.1.1.4 — Frontend E2E smoke test (Playwright/Cypress setup)

```
You are working in frontend/ (new tooling installation and
configuration) and .github/workflows/ci.yml. Assume Epics 1–23 are
fully merged. CONFIRMED: no E2E browser-testing tool exists anywhere
in this project currently — this is genuinely new infrastructure, not
an audit task.

CONTEXT
Every existing frontend test in this project (established since the
earliest epics) is a Vitest COMPONENT test — rendering individual
React components in isolation with mocked API calls. None of them
exercise a REAL BROWSER running the REAL, FULLY-BUILT application
against a REAL backend, clicking through an actual multi-page user
flow the way an actual customer would. This is a categorically
different, valuable kind of test coverage this project has never had.

TASK
Install and configure Playwright (RECOMMENDED over Cypress for a new
setup at this point — verify current comparative maturity/maintenance
status before finalizing, but Playwright has generally offered
stronger multi-browser support and a more modern, actively-developed
API in recent years; either is a defensible choice, pick one and
implement it fully rather than half-configuring both), and write ONE
genuinely meaningful smoke test: a real user completing the core
"browse → add to cart → checkout" flow end-to-end against a real
running instance of the application.

REQUIREMENTS
- Add Playwright to frontend/ as a dev dependency
  (`npm install --save-dev @playwright/test`), and run its browser-
  install step (`npx playwright install --with-deps chromium` at
  minimum — full multi-browser coverage across Chromium/Firefox/WebKit
  is a reasonable future enhancement, but ONE browser is sufficient
  for an initial smoke-test setup; don't over-scope this task's first
  pass).
- Add `playwright.config.ts` configured to run against a LOCAL
  development instance of the full stack — this test needs BOTH the
  backend AND frontend actually running (unlike Vitest's mocked-API
  component tests) — configure Playwright's `webServer` option to
  start the frontend dev server automatically if not already running,
  and document clearly (in the config file's comments and/or a README
  section) that the BACKEND must be separately running (e.g. via
  `docker compose -f docker-compose.dev.yml up`) before this test suite
  can execute, since Playwright's `webServer` config typically only
  manages ONE process (the frontend), not this project's full
  multi-service Docker Compose stack.
- Write ONE meaningful smoke test in frontend/e2e/checkout-flow.spec.ts:
  ```typescript
  import { test, expect } from '@playwright/test';

  test('customer can browse, add to cart, and reach checkout', async ({ page }) => {
    await page.goto('/');
    await expect(page.getByRole('heading', { level: 1 })).toBeVisible();

    // Navigate to a product listing page
    await page.getByRole('link', { name: /shop|products|فروشگاه/i }).first().click();
    await expect(page).toHaveURL(/\/(shop|products)/);

    // Open the first product
    await page.locator('[data-testid="product-card"]').first().click();
    await expect(page.getByRole('button', { name: /add to cart|افزودن به سبد/i })).toBeVisible();

    // Add to cart
    await page.getByRole('button', { name: /add to cart|افزودن به سبد/i }).click();
    await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1');

    // Go to cart, then proceed to checkout
    await page.goto('/cart');
    await expect(page.locator('[data-testid="cart-item"]')).toHaveCount(1);
    await page.getByRole('link', { name: /checkout|تسویه‌حساب/i }).click();
    await expect(page).toHaveURL(/\/checkout/);

    // Confirm the checkout form actually rendered with real content —
    // this test deliberately STOPS here rather than completing a real
    // payment (which would need real/sandbox gateway credentials and
    // genuinely mutate backend order/payment state) — the goal is
    // proving the full browse→cart→checkout PATH works end-to-end in a
    // real browser, not exercising the payment gateway itself, which
    // Epic 6/24.1.1.1's backend integration tests already cover
    // thoroughly with mocked gateway responses.
    await expect(page.getByRole('heading', { name: /checkout|تسویه‌حساب/i })).toBeVisible();
  });
  ```
  Note the DELIBERATE stopping point BEFORE actually submitting
  payment — this is a considered scope boundary, not laziness: a true
  E2E test that submits a REAL payment would need real/sandbox gateway
  credentials wired into a genuinely running instance, would leave real
  database side effects (a real Order row) that need cleanup, and
  would largely DUPLICATE what Task 24.1.1.1's backend integration
  suite already covers thoroughly (with fully mocked, controllable
  gateway responses, which is a MORE reliable way to test payment
  logic than a live/sandbox gateway call in a browser test would be
  anyway). This browser-level test's actual, distinct VALUE is proving
  the FRONTEND PATH — real rendering, real routing, real client-side
  cart state — genuinely works when clicked through by something
  simulating a real user, which no Vitest component test or backend
  integration test can prove on its own.
  Note the `[data-testid="..."]` selectors used above — confirm
  whether this project's components ALREADY use `data-testid`
  attributes anywhere (check for precedent); if not, add them to the
  SPECIFIC few elements this test needs to target reliably (product
  card, cart count badge, cart item) rather than relying on fragile
  text-content or CSS-class-based selectors, which break easily on
  copy/styling changes unrelated to actual functionality — this is a
  small, worthwhile addition to the relevant components as part of
  this task, not scope creep.
- Add a `frontend:e2e` script to package.json
  (`"test:e2e": "playwright test"`), and add a NEW, SEPARATE job to
  `.github/workflows/ci.yml` (not folded into the existing `frontend`
  Vitest job, since this one needs a running backend too, a
  meaningfully different setup) — check whether standing up the FULL
  backend (Postgres + Redis + migrations + runserver) inside a GitHub
  Actions job, mirroring the existing `backend` job's service
  containers, is straightforward given this project's confirmed CI
  structure, and wire it up so the E2E job runs both backend and
  frontend dev servers before executing Playwright tests against them.
  Mark this new job as NON-BLOCKING initially
  (`continue-on-error: true`, matching the exact precedent already
  established in this same workflow file for the frontend coverage
  step and Epic 23 Task 23.1.1.4's vulnerability-scanning rollout) —
  a BRAND NEW E2E job standing up a full multi-service stack for the
  first time in CI has real potential for environment-specific flakes
  that shouldn't immediately start blocking every PR; document this as
  an explicit, intentional initial-rollout choice with a note to
  revisit once it's proven stable over some real usage period, matching
  the same two-phase rollout pattern already established in this
  document series.

ACCEPTANCE CRITERIA / TESTS
- The Playwright test passes locally against a real running instance
  of the full stack.
- The new CI job successfully stands up both backend and frontend and
  runs the test in the GitHub Actions environment (verify by actually
  triggering the workflow, not just reviewing the YAML for
  correctness).
- `data-testid` attributes were added to the specific components this
  test needed, without disrupting any existing component tests/
  functionality (re-run the full existing Vitest suite and confirm no
  regression from these small markup additions).
```

---

#### Task 24.1.1.5 — Load test script for product listing/checkout endpoints (k6/Locust)

```
You are working in a new backend/loadtests/ directory (or
frontend-adjacent, your call on placement — this tooling tests the
BACKEND API specifically, so co-locating it under backend/ is
reasonable). Assume Epics 1–23 are fully merged, including Epic 21's
Redis caching layer (Task 21.1.1.3's short-TTL product-list caching)
and Epic 22's Celery infrastructure. CONFIRMED: no load-testing tool
exists anywhere in this project — genuinely new infrastructure.

CONTEXT
This project's original architecture review explicitly raised
production-readiness-at-scale as an open, UNVERIFIED question ("Can
this project handle 10/100/1,000/10,000/100,000 users?") — every
answer given throughout that review, and every scaling-related
decision made across this entire 23-epic backlog (caching, Celery,
database indexing, connection pooling considerations), has been
REASONED ABOUT but never actually MEASURED against real load. This
task is the first point in the entire project where an actual,
repeatable performance BASELINE gets established.

TASK
Set up k6 (RECOMMENDED — a modern, scriptable-in-JavaScript load-
testing tool with good CI integration and a lower resource footprint
than some alternatives; Locust, Python-based, is an equally valid
choice if this project's team has stronger Python-scripting
familiarity than JS — pick one, implement it fully) and write load
test scripts covering the two most critical, highest-traffic endpoint
categories: product listing/search (Epic 12) and checkout (Epics 1/5/6/7/9
combined).

REQUIREMENTS
- Install k6 (a standalone binary, not an npm/pip package — document
  the installation step clearly, e.g. via the project's Dockerfile if
  running load tests in a containerized way, or as a documented local
  prerequisite).
- Write `backend/loadtests/product-listing.js`:
  ```javascript
  import http from 'k6/http';
  import { check, sleep } from 'k6';

  export const options = {
    stages: [
      { duration: '30s', target: 20 },   // ramp up to 20 virtual users
      { duration: '1m', target: 20 },    // hold steady
      { duration: '30s', target: 100 },  // spike to 100
      { duration: '1m', target: 100 },   // hold at spike
      { duration: '30s', target: 0 },    // ramp down
    ],
    thresholds: {
      http_req_duration: ['p(95)<500'],  // 95% of requests under 500ms — adjust based on real, agreed performance targets, not an arbitrary guess
      http_req_failed: ['rate<0.01'],    // less than 1% error rate
    },
  };

  const BASE_URL = __ENV.BASE_URL || 'http://localhost:8000/api';

  export default function () {
    // Mix of realistic browse patterns: plain listing, filtered, searched
    const scenarios = [
      `${BASE_URL}/products/`,
      `${BASE_URL}/products/?category=1`,
      `${BASE_URL}/products/?search=سرم`,
      `${BASE_URL}/products/?skin_type=oily&is_vegan=true`,
    ];
    const url = scenarios[Math.floor(Math.random() * scenarios.length)];
    const res = http.get(url);
    check(res, {
      'status is 200': (r) => r.status === 200,
      'response has results': (r) => JSON.parse(r.body).results !== undefined,
    });
    sleep(1);
  }
  ```
  Note the deliberately MIXED scenario set (plain/filtered/searched
  requests, randomly selected per virtual user iteration) — a load
  test hitting ONLY the plain unfiltered listing endpoint would
  systematically UNDER-test Epic 12's full-text search and Epic 3's
  filter-query performance, and would also produce an unrealistically
  HIGH cache-hit rate against Epic 21's product-list caching (since a
  real traffic pattern has far more query-parameter diversity than
  hammering one identical URL) — this mix is a meaningfully more
  realistic simulation, worth the small added complexity.
- Write `backend/loadtests/checkout-flow.js` — a MULTI-STEP scenario
  script (not a single endpoint hit), since checkout is inherently a
  sequence: create a test user session, add items to cart, initiate
  checkout. Given a REAL payment gateway call shouldn't be part of a
  load test (per the same reasoning as Task 24.1.1.4's E2E test
  stopping before actual payment submission — hitting a real/sandbox
  gateway at load-test VOLUME would be both slow and potentially
  costly/rate-limited by the gateway itself), this script should
  exercise UP TO the payment-initiation POST
  (`/api/payments/initiate/`) but MOCK or otherwise avoid the actual
  outbound gateway call — check whether this project's backend has any
  existing TEST-MODE gateway configuration (Epic 6 Task 6.2.1.6's
  `ZARINPAL_SANDBOX` setting) that could reasonably be left active
  during a load test run against a DEDICATED load-testing environment
  (never against real production), which would let the full flow
  including a REAL sandbox gateway round-trip be exercised — this is
  a legitimate, more thorough option if a safe sandbox environment is
  genuinely available and the sandbox gateway itself won't rate-limit
  or reject the load test's volume; otherwise, stop the script before
  the actual gateway call, same as the E2E test's scope boundary.
  ```javascript
  import http from 'k6/http';
  import { check, sleep } from 'k6';

  export const options = {
    stages: [
      { duration: '30s', target: 10 },
      { duration: '2m', target: 10 },
      { duration: '30s', target: 0 },
    ],
    thresholds: {
      http_req_duration: ['p(95)<1000'],  // checkout involves more work (stock locking, pricing) than a plain read, so a looser threshold than product listing is appropriate
      http_req_failed: ['rate<0.01'],
    },
  };

  const BASE_URL = __ENV.BASE_URL || 'http://localhost:8000/api';

  export default function () {
    // Register/login a fresh test user per iteration to realistically
    // simulate independent concurrent shoppers rather than all virtual
    // users sharing one account/cart, which would produce an
    // unrealistic contention pattern unrelated to real traffic.
    const email = `loadtest_${__VU}_${__ITER}@example.com`;
    const registerRes = http.post(`${BASE_URL}/auth/register/`, JSON.stringify({
      email, password: "LoadTest123!", first_name: "Load", last_name: "Test",
    }), { headers: { 'Content-Type': 'application/json' } });
    check(registerRes, { 'registered': (r) => r.status === 201 });

    const token = JSON.parse(registerRes.body).access;
    const authHeaders = { headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${token}` } };

    // Add a known, seeded test product variant to cart — VARIANT_ID
    // must reference real seed data in whatever environment this
    // script targets; parameterize via an env var, don't hardcode an
    // ID that only happens to exist in one specific database snapshot.
    const variantId = __ENV.TEST_VARIANT_ID || 1;
    const cartRes = http.post(`${BASE_URL}/cart/`, JSON.stringify({ variant_id: variantId, quantity: 1 }), authHeaders);
    check(cartRes, { 'added to cart': (r) => r.status === 201 || r.status === 200 });

    sleep(1);
  }
  ```
  Note the CRITICAL detail of registering a genuinely FRESH user per
  virtual-user-iteration (`__VU`/`__ITER`, k6's built-in unique
  identifiers) — this is essential for the STOCK-LOCKING behavior
  (Epic 1/3's `select_for_update()`-based atomic checkout) to be
  meaningfully exercised under REALISTIC concurrent-but-independent
  load, rather than every simulated user contending over ONE shared
  cart/account, which would produce misleading results dominated by
  artificial single-account contention rather than genuine
  many-independent-shoppers load.
- Add a README (backend/loadtests/README.md) documenting: how to run
  each script (`k6 run product-listing.js -e BASE_URL=...`), what
  environment they're safe to run against (NEVER production — a
  dedicated staging/load-test environment with representative seed
  data, clearly stated), how to interpret results, and — importantly —
  a note that these scripts establish a BASELINE, not a pass/fail gate
  in CI (running a real load test as part of every CI run would be
  slow, resource-intensive, and not particularly meaningful against
  CI's typically constrained/shared runner resources) — this tooling
  is for DELIBERATE, periodic, manually-triggered performance
  verification (e.g. before a major traffic event, or after a
  significant architecture change), not continuous automated gating.

ACCEPTANCE CRITERIA / TESTS
- Run both scripts against a real local/staging instance of the
  application and produce an ACTUAL results summary (requests/sec,
  p95/p99 latency, error rate at each traffic stage) — this is the
  actual, concrete deliverable this task exists to produce: a REAL
  measured baseline, not just working tooling that's never actually
  been run.
- Document the results and any findings (e.g. "checkout p95 latency
  degrades noticeably above N concurrent users, likely due to X" —
  if the load test surfaces a genuine performance problem, note it
  clearly as a finding for follow-up investigation, even though fixing
  it is out of this task's own scope, which is establishing the
  measurement capability and baseline, not necessarily every
  performance fix that baseline might reveal is needed).
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 24.1.1.1 | End-to-end checkout test (audit + extend existing Epic 6 suite) | ☐ |
| 24.1.1.2 | Coupon abuse test suite (audit + cross-epic scenarios) | ☐ |
| 24.1.1.3 | OTP flow abuse tests (audit post-relocation + retry-mechanism scenarios) | ☐ |
| 24.1.1.4 | Frontend E2E smoke test (Playwright setup) | ☐ |
| 24.1.1.5 | Load test script (k6, product listing + checkout) | ☐ |

**A note consistent with this document series' recurring theme:** as with Epic 20's file-validation work closing out most of Epic 23's Task 23.1.1.3, and Epic 22's own closing note about infrastructure epics ideally being pulled forward — this epic is the clearest illustration yet of the master backlog's own stated principle actually working as intended: because critical-path tests were written INCREMENTALLY throughout this series rather than deferred, three of this epic's five tasks turned out to be verification-and-extension work rather than greenfield builds, and the one genuinely novel discovery this epic surfaced (Task 24.1.1.2's coupon `max_uses` race-condition check) is exactly the kind of gap that a dedicated, deliberate "now go looking for abuse scenarios specifically" pass is well-suited to catch, even in code that already had substantial test coverage.

Once Epic 24 is fully merged, the next epics to generate prompts for
are **Epic 25 — DevOps/Docker/CI-CD** and **Epic 26 — Monitoring &
Logging**, both of which build directly on this epic's new CI jobs
(Task 24.1.1.4's E2E job, and Epic 23 Task 23.1.1.4's vulnerability
scanning) and should ideally finish before any real production launch,
per the master backlog's own execution-order guidance.
