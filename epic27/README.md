# Epic 27 — Analytics — AI Coding Prompts

Repo: `tablogenix-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–26 are fully merged.

**A naming collision worth clearing up before starting — read this first:** grepping this codebase for "analytics" turns up an EXISTING `AdminAnalytics.jsx` page (`frontend/src/pages/admin/`), an "Analytics" nav item in `AdminLayout.jsx`, and analytics-labeled docstrings on `dashboard/views.py`'s `GET /dashboard/admin/overview/`, `/user-stats/`, `/product-stats/` endpoints. **This is Epic 17's business-metrics dashboard** (revenue, top products, low-stock, coupon usage — all internal, admin-facing KPI reporting built directly from this platform's own database) — it has **nothing to do** with what THIS epic builds. Confirmed via grep: there is **no `gtag`, `dataLayer`, or any actual customer-behavior tracking library anywhere in this codebase** — Epic 27's job is adding real, customer-facing WEB analytics (page views, e-commerce events, conversion funnels tracked by an external analytics platform), which is a genuinely different, currently entirely-absent capability. Don't confuse the two, and don't let "Analytics" already existing as a nav label make you think this epic's work is already done.

**A consideration specific to this platform's actual target market, worth weighing seriously in Task 27.1.1.1:** this platform's entire audience is Iran-based (per the project's stated goal throughout every prior epic). Google Analytics (GA4) is a Google-operated service, and Google services have a documented history of intermittent restrictions, blocking, or unreliable connectivity for users inside Iran due to US sanctions and Google's own service-availability decisions in that market — a real, concrete risk that GA4 could systematically UNDER-COUNT or entirely MISS a meaningful share of this platform's actual real-world traffic, undermining the entire point of collecting analytics in the first place. This is a genuine, non-generic reason to weigh self-hosted alternatives seriously here, not just a generic privacy-preference talking point.

---

## Phase 27.1 — Product & Business Analytics

### Feature 27.1.1 — Tracking Setup

---

#### Task 27.1.1.1 — Integrate a privacy-compliant analytics tool (self-hosted Plausible/Matomo, or GA4)

```
You are working in frontend/index.html (or main.jsx), a new
frontend/src/utils/analytics.js, and potentially docker-compose.prod.yml/
staging.yml if a self-hosted option is chosen. Assume Epics 1–26 are
fully merged. RE-READ THIS DOCUMENT'S HEADER before starting — the
GA4-reliability concern for this platform's actual Iran-based audience
is a real, weighty factor in this decision, not boilerplate.

CONTEXT
No analytics/tracking tool of any kind exists in this codebase. Three
realistic options exist, each with real tradeoffs specific to THIS
project:
- **GA4 (Google Analytics 4)**: free, extremely widely documented,
  richest ecosystem of integrations/reporting tools — but per this
  document's header, carries a genuine risk of significantly
  undercounting this platform's actual Iran-based traffic due to
  Google service reliability/restrictions in that market, which is a
  serious, not merely theoretical, concern for a platform whose ENTIRE
  audience is exactly the population most likely to be affected.
- **Plausible** (self-hosted or cloud): lightweight, genuinely
  privacy-respecting by design (no cookies, no cross-site tracking, GDPR-
  compliant without a cookie-consent-banner requirement), simple event
  API — self-hosting it keeps traffic entirely on this platform's own
  infrastructure, sidestepping the GA4 reliability concern entirely
  (it's this project's own domain serving its own analytics endpoint,
  not a third-party Google domain a user's network/ISP might restrict).
- **Matomo** (self-hosted or cloud): more feature-rich than Plausible
  (closer to GA4's depth of reporting, including built-in funnel/goal
  tracking), but meaningfully heavier infrastructure (its own database,
  more operational overhead) and a steeper setup/maintenance cost.

TASK
Make a deliberate, documented choice and implement it. RECOMMEND
**self-hosted Plausible** for this project specifically: it directly
sidesteps the GA4 reliability risk (traffic to `analytics.tablogenix.com`
or a subpath of the platform's own domain is far less likely to be
selectively restricted than traffic to `google-analytics.com`), its
privacy-by-design approach means no cookie-consent-banner requirement
(genuinely simpler for both users and compliance than GA4's tracking-
cookie model), and its lighter operational footprint fits this
project's demonstrated Docker Compose-based, moderate-scale
infrastructure (per Epic 25's grounding) better than Matomo's heavier
footprint — but if you have a strong, specific reason to choose
differently (e.g. this project's actual team already has real
operational experience with Matomo/GA4 specifically, or genuinely needs
Matomo's built-in funnel tooling badly enough to justify its extra
weight), document that reasoning explicitly and proceed with your
chosen alternative instead; the important thing is a DELIBERATE,
DOCUMENTED choice, not blind adherence to this recommendation.

REQUIREMENTS (assuming self-hosted Plausible, adjust proportionally
if a different tool was chosen)
- Add a `plausible` service to `docker-compose.prod.yml` and
  `docker-compose.staging.yml` (per Epic 25's established compose-file
  conventions), following Plausible's own current, documented
  self-hosting Docker setup (verify against Plausible's current
  official self-hosting guide, since exact required services/env vars
  can evolve between versions — Plausible typically requires its own
  Postgres-adjacent + Clickhouse database services, distinct from this
  project's existing application Postgres instance; keep them
  genuinely separate, don't try to share this project's own `db`
  service with Plausible's internal storage needs).
  Expose it at a dedicated subdomain/path (e.g.
  `analytics.tablogenix.com` or `tablogenix.com/analytics-proxy/` —
  a PATH-based approach on the SAME domain as the main site, proxied
  through the existing nginx service, has a genuine additional benefit
  worth considering: some ad/tracker-blocking browser extensions
  specifically block known ANALYTICS SUBDOMAINS by pattern-matching
  domain names, while a same-origin PATH is less commonly targeted by
  such blocklists — this is a legitimate, additional reliability
  consideration for THIS platform's specific goal of getting genuinely
  representative traffic data, not just a stylistic choice; weigh it
  when deciding the exact hosting path).
- Add the Plausible tracking script to `frontend/index.html` (or via
  `react-helmet-async`'s shared default Helmet block per Epic 15 Task
  15.1.1.1, if per-page conditional loading is preferred — a simple
  static `<script>` tag in `index.html` is likely sufficient and
  simpler for a single-page-app-wide tracking need like this):
  ```html
  <script defer data-domain="tablogenix.com" src="https://analytics.tablogenix.com/js/script.js"></script>
  ```
  (adjust the exact domain/script path to match whatever hosting
  decision was made above).
- Create frontend/src/utils/analytics.js as the SINGLE, central place
  this project sends tracking events through (used by Task 27.1.1.2's
  e-commerce event work) — don't scatter direct calls to the tracking
  library throughout components:
  ```javascript
  export function trackPageview() {
    if (window.plausible) window.plausible('pageview');
  }

  export function trackEvent(eventName, props = {}) {
    if (window.plausible) window.plausible(eventName, { props });
  }
  ```
  Guarding every call with `if (window.plausible)` matters — the
  tracking script can fail to load (network issue, ad-blocker,
  genuinely restricted connectivity per this document's header
  concern) and this project's actual functionality must NEVER depend
  on or break because of that; analytics is observability, never a
  hard dependency of the checkout flow or any other real feature.
- Wire page-VIEW tracking into React Router navigation (SPAs don't
  produce natural full-page-loads the tracking script would detect
  automatically the way a traditional multi-page site would — a
  React-Router-aware pageview hook is needed):
  ```javascript
  // a small hook, used once near the app's router root
  import { useLocation } from 'react-router-dom';
  import { useEffect } from 'react';
  import { trackPageview } from '../utils/analytics';

  export function useAnalyticsPageviews() {
    const location = useLocation();
    useEffect(() => {
      trackPageview();
    }, [location.pathname]);
  }
  ```

ACCEPTANCE CRITERIA / TESTS
- Manually verify against a real staging deployment: the Plausible
  dashboard (or chosen tool's equivalent) shows real pageview data
  after browsing the staging site; SPA route changes correctly
  register as distinct pageviews, not just the initial page load.
- Add a frontend test confirming `trackEvent()`/`trackPageview()`
  degrade gracefully (don't throw) when `window.plausible` is
  undefined (simulating a blocked/failed-to-load tracking script) —
  the concrete regression test for the "analytics must never break
  real functionality" principle.
- Document (in a brief README section or Epic 25-established runbook
  location) the reasoning for the chosen tool, specifically referencing
  the GA4-reliability-in-Iran consideration, so this decision's context
  is preserved for anyone revisiting it later.
```

---

#### Task 27.1.1.2 — E-commerce event tracking (`view_item`, `add_to_cart`, `purchase`)

```
You are working across frontend/src/pages/ProductDetails.jsx,
context/CartContext.jsx, and the order-confirmation page. Assume Task
27.1.1.1 is already merged.

CONTEXT
Task 27.1.1.1 established the tracking mechanism and basic pageview
tracking — nothing yet tracks the SPECIFIC e-commerce funnel events
that actually matter for understanding conversion (a pageview alone
doesn't tell you whether a product view led to a cart add, or whether
a cart add led to a completed purchase).

TASK
Add the standard e-commerce event set — `view_item`, `add_to_cart`,
`begin_checkout`, `purchase` — at the correct, precise moments in the
user journey, using this project's ACTUAL, established data (currency,
product structure) at each point.

REQUIREMENTS
- `view_item`: fire in `ProductDetails.jsx` once product data has
  successfully loaded (the SAME point Epic 13 Task 13.1.1.2's
  `addRecentlyViewed()` call already fires, per that task's
  established pattern — add this tracking call ALONGSIDE that
  existing one, in the same `useEffect`):
  ```javascript
  trackEvent('view_item', {
    product_id: product.id,
    product_name: product.name,
    category: product.category?.name,
    price: product.price,  // raw Rial integer, per Epic 14's storage decision — see currency note below
  });
  ```
- `add_to_cart`: fire in `CartContext.jsx`'s `handleAddToCart()`
  (Epic 5 Task 5.1.1.4's now-guest-accessible add-to-cart handler),
  ONLY on a genuinely successful add (after the API call succeeds, not
  optimistically before confirming success):
  ```javascript
  trackEvent('add_to_cart', {
    variant_id: variantId,
    quantity: quantity,
    price: variantPrice,
  });
  ```
- `begin_checkout`: fire when the customer actually REACHES the
  checkout page (in the Checkout page component's mount effect), NOT
  merely clicking a "checkout" link/button elsewhere (the page
  actually loading is the more reliable, meaningful signal of genuine
  checkout intent).
- `purchase`: this is the event requiring the MOST care about WHEN it
  fires, given Epic 6's real payment flow — fire it ONLY on the
  order-CONFIRMATION page, reached ONLY after
  `PaymentCallbackView`'s SUCCESS path redirects the browser there
  (per Epic 6 Task 6.2.1.4) — NEVER at checkout FORM SUBMISSION time
  (which per Epic 6's actual flow only creates a `PENDING` order and
  initiates a gateway redirect; payment could still fail or be
  abandoned after that point, per Epic 6's stock-release-on-failure
  logic) — firing `purchase` too early would count failed/abandoned
  payments as completed sales, corrupting the most financially
  important metric this tracking setup produces:
  ```javascript
  // order-confirmation page, on mount, guard against double-firing on
  // page refresh (see de-duplication note below)
  useEffect(() => {
    if (order && order.status !== 'pending' && !hasTrackedPurchase.current) {
      trackEvent('purchase', {
        order_number: order.order_number,
        total: order.total,  // raw Rial integer
        items: order.items.length,
      });
      hasTrackedPurchase.current = true;
    }
  }, [order]);
  ```
  Add DE-DUPLICATION protection — a customer refreshing the order-
  confirmation page (a normal, expected action) must NOT re-fire the
  `purchase` event a second time for the SAME order, which would
  inflate revenue-tracking numbers with phantom duplicate purchases; a
  `useRef`-based in-component guard (shown above) handles the
  same-SESSION refresh case; consider whether a more robust guard is
  warranted (e.g. checking `sessionStorage` for
  `"purchase_tracked_{order_number}"` before firing, persisting across
  a full page reload where the `useRef` would reset) — RECOMMEND the
  `sessionStorage` approach specifically, since a `useRef` alone does
  NOT survive an actual browser refresh (only re-renders within the
  same mounted component instance), and a genuine page reload of the
  order-confirmation URL (a very plausible real action) is exactly the
  scenario needing protection.
- CURRENCY NOTE, applying to every event above: per Epic 14's storage
  decision, every price value in this project's data is a raw Rial
  INTEGER. Confirm whether the chosen analytics tool (Plausible, per
  Task 27.1.1.1's recommendation) has any built-in currency-
  formatting/revenue-reporting expectations about the SHAPE of numeric
  event properties it receives — pass raw Rial integers consistently
  (not Toman-converted, not formatted strings) so any revenue
  aggregation the tool performs operates on one single, correct,
  consistent unit throughout — document this explicitly in a code
  comment near `analytics.js`, given how easy a unit mismatch would be
  to introduce accidentally later without this being written down
  somewhere.

ACCEPTANCE CRITERIA / TESTS
Add frontend tests confirming each event fires at the correct moment
with correct data:
1. `view_item` fires exactly once when product data loads, with
   correct product details.
2. `add_to_cart` fires ONLY after a successful API response (mock a
   FAILED add-to-cart call and confirm the event does NOT fire).
3. `begin_checkout` fires on the checkout page mounting.
4. `purchase` fires on the order-confirmation page for a genuinely
   completed order, and does NOT fire for a `pending`/failed order
   context.
5. **The de-duplication test is the most important one in this task**:
   simulate the order-confirmation page mounting TWICE for the SAME
   order (representing a refresh) and confirm `purchase` fires only
   ONCE total across both mounts.
```

---

#### Task 27.1.1.3 — Admin conversion-funnel dashboard widget

```
You are working in backend/dashboard/services.py, views.py, and
frontend/src/pages/admin/AdminAnalytics.jsx (Epic 17's EXISTING
business-metrics page, confirmed present — this task ADDS a new
widget/section to it, per this document's header clarification that
this IS the correct, existing home for admin-facing metrics work,
unlike the confusion this document's header specifically existed to
prevent). Assume Task 27.1.1.2 is already merged.

CONTEXT
Task 27.1.1.2's events now flow into the EXTERNAL analytics tool
(Plausible/whichever was chosen), which has its OWN dashboard/UI for
viewing that data — but this project's admin staff already have an
established habit of using the EXISTING, IN-PLATFORM `AdminAnalytics`
page (Epic 17) for business metrics, and a conversion funnel
specifically (view→cart→checkout→purchase, with drop-off rates at each
step) is valuable enough to surface DIRECTLY in that existing admin
dashboard rather than requiring staff to separately log into a
different external tool's UI just to see this one specific view.

TASK
Add a conversion-funnel widget to the existing `AdminAnalytics` admin
page, computed from THIS PLATFORM'S OWN database records (not by
querying the external analytics tool's API) — since every step of this
specific funnel (a product view, a cart addition, a checkout page
visit, a completed purchase) EXCEPT the first has a real, authoritative
record in this project's own database already (Cart/CartItem records
per Epic 5, completed Orders per Epic 1/6) — using the platform's own
data is both simpler (no need to integrate with an external tool's
reporting API) and more reliable (the platform's own database is the
authoritative source of truth for cart/order data specifically, vs.
depending on client-side event tracking that can be lost to ad-
blockers/network issues for the SAME events).

REQUIREMENTS
- Add `get_conversion_funnel(start, end) -> dict` to
  `dashboard/services.py` (alongside Epic 17's existing analytics
  functions, matching that file's established style/currency-handling
  conventions post-Epic-14's fix):
  ```python
  def get_conversion_funnel(start, end) -> dict:
      from cart.models import Cart, CartItem
      from order.models import Order

      # "Carts with items" as a proxy for real, meaningful cart-add
      # activity in the window (a cart record itself doesn't carry a
      # timestamp for WHEN items were added distinctly from when the
      # cart row itself was created — use CartItem.added_at, which
      # Epic 5's model does track, as the actual signal).
      carts_with_items = (
          Cart.objects.filter(items__added_at__range=[start, end]).distinct().count()
      )
      orders_created = Order.objects.filter(created_at__range=[start, end]).count()
      orders_completed = Order.objects.filter(
          created_at__range=[start, end],
          status__in=[Order.Status.PROCESSING, Order.Status.SHIPPED, Order.Status.DELIVERED],
      ).count()

      def rate(numerator, denominator):
          return round((numerator / denominator) * 100, 1) if denominator else None

      return {
          "carts_with_items": carts_with_items,
          "orders_created": orders_created,       # includes abandoned/failed-payment orders (PENDING that never completed)
          "orders_completed": orders_completed,    # genuinely paid/confirmed orders
          "cart_to_checkout_rate": rate(orders_created, carts_with_items),
          "checkout_to_purchase_rate": rate(orders_completed, orders_created),
          "overall_cart_to_purchase_rate": rate(orders_completed, carts_with_items),
      }
  ```
  Note this funnel deliberately starts at "cart with items," NOT at
  raw product VIEWS — product-view data lives ONLY in the external
  analytics tool (Task 27.1.1.2's `view_item` events), not in this
  platform's own database at all, so a TRUE full funnel starting from
  views would require querying the external tool's reporting API,
  which is real additional integration work beyond what THIS platform-
  data-only approach needs. Decide explicitly: is a
  view→cart→checkout→purchase funnel (requiring the external tool's
  API) valuable enough to build now, or is the SIMPLER
  cart→checkout→purchase funnel (built entirely from this platform's
  own already-available data, per the implementation above) a
  reasonable, lower-effort starting point that still captures the most
  actionable drop-off signal (the checkout-to-purchase step, which is
  the step most directly indicating a payment-flow problem worth
  investigating, per Epic 6's own gateway-reliability concerns)?
  RECOMMEND: ship the SIMPLER cart-based funnel now (per the
  implementation above), and note the fuller view-inclusive funnel as
  a reasonable future enhancement requiring the chosen analytics
  tool's reporting API specifically — don't take on that extra
  integration complexity within this task unless it's clearly
  warranted.
- Add the endpoint: `GET /dashboard/admin/conversion-funnel/`,
  accepting the SAME `?period=`/custom-range query parameters
  established in Epic 17 Task 17.1.1.1, for consistency with this
  project's other analytics endpoints.
- Add the widget to `AdminAnalytics.jsx`: a simple funnel visualization
  (a series of horizontally-shrinking bars, or a simple numbered list
  with percentages — check whether this project has any existing chart/
  visualization library already in use elsewhere in the admin
  dashboard from prior epics' work and reuse it for visual consistency
  rather than introducing a new charting dependency for just this one
  widget) showing: carts with items → orders created → orders completed,
  with the drop-off rate labeled at each step.
- Add a clear visual/textual distinction between `orders_created` (ALL
  orders, including abandoned-payment ones) and `orders_completed`
  (genuinely paid) — the GAP between these two numbers is itself a
  directly actionable signal (a large gap suggests a payment-flow
  problem worth investigating, tying directly back to Epic 26 Task
  26.1.1.4's payment-failure alerting — this admin widget and that
  alerting mechanism are complementary views of the SAME underlying
  concern, one for real-time incident response, one for ongoing trend
  monitoring).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/test_services.py:
1. `get_conversion_funnel()` correctly counts carts-with-items,
   orders-created, and orders-completed within the specified date
   range, excluding activity outside it.
2. `orders_completed` correctly EXCLUDES `PENDING` and `CANCELLED`
   orders (only counting genuinely paid ones) while `orders_created`
   correctly INCLUDES them — the key distinction this widget's whole
   value depends on getting right.
3. Rate calculations correctly return `None` (not a division error)
   when the denominator is zero (e.g. a date range with no cart
   activity at all).
4. The endpoint respects admin-only permissions and the same period/
   custom-range query parameters as other Epic 17 analytics endpoints.
Add a frontend test confirming the funnel widget renders correctly
with mocked data, including the visual distinction between created and
completed order counts.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 27.1.1.1 | Integrate privacy-compliant analytics tool (self-hosted Plausible recommended) | ☐ |
| 27.1.1.2 | E-commerce event tracking (view_item/add_to_cart/purchase) | ☐ |
| 27.1.1.3 | Admin conversion-funnel dashboard widget | ☐ |

Once Epic 27 is fully merged, the next epic to generate prompts for is
**Epic 28 — Marketing Tools**, which the master backlog groups
alongside this one as post-launch growth work, and which directly
depends on this epic's tracking infrastructure (Task 27.1.1.2's
`begin_checkout`/`purchase` events in particular) for measuring
whether Epic 28's own features — like abandoned-cart recovery — are
actually working.
