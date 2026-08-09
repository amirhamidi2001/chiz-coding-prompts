# Epic 13 — Recommendation Engine — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–12 are fully merged.

**Important grounded discovery for this epic — read before starting Task 13.1.1.3:** a `RelatedProductsView` (`backend/shop/views.py`) and a "Related Products" section on the product detail page (`frontend/src/pages/ProductDetails.jsx`) **already exist** — but it is a **category-similarity** recommender ("same category, sorted by rating, top 8"), not a **behavioral co-purchase** recommender. This epic's Task 13.1.1.1/13.1.1.3 build a genuinely different, complementary recommendation type based on actual `OrderItem` purchase co-occurrence data ("customers who bought X also bought Y"). **Do not confuse or merge these two features** — the existing `RelatedProductsView`/"Related Products" section stays exactly as-is; this epic adds a **second, additional** section to the PDP, not a replacement.

---

## Phase 13.1 — Baseline Recommendations

### Feature 13.1.1 — Rule-Based Recommendations

---

#### Task 13.1.1.1 — "Frequently bought together" (co-occurrence in `OrderItem`)

```
You are working in backend/shop/views.py, urls.py (or a new
backend/shop/recommendations.py service module). Assume Epics 1–12 are
fully merged, including Epic 3's variant-based `OrderItem.variant` FK
and Epic 8's confirmed `Order.Status` values.

CONTEXT
No behavioral recommendation logic exists anywhere in this codebase —
the only "related" concept is the existing category-based
`RelatedProductsView` (per this document's header context, NOT what
this task builds). `OrderItem` (backend/order/models.py) has both
`product` (FK to `shop.Product`, `SET_NULL`, nullable) and `variant`
(FK to `shop.ProductVariant`, `SET_NULL`, nullable, per Epic 3 Task
3.1.1.4) — for a co-occurrence recommendation, work at the PRODUCT
level (not variant level), since "customers who bought this SHADE also
bought this OTHER SHADE of the same product" is a much less useful
signal than product-level co-purchase patterns, and variant-level
co-occurrence would badly fragment an already-sparse signal across many
shade/size combinations.

TASK
Add a service function computing "frequently bought together" products
for a given product, based on real `OrderItem` co-occurrence across
completed orders, plus an API endpoint exposing it.

REQUIREMENTS
- Create backend/shop/recommendations.py:
  ```python
  from django.db.models import Count, Q
  from order.models import Order, OrderItem
  from .models import Product


  def get_frequently_bought_together(product_id: int, limit: int = 4) -> list[Product]:
      """
      Return up to `limit` products most frequently purchased in the
      SAME order as `product_id`, based on completed (non-cancelled)
      order history, ranked by co-occurrence frequency.
      """
      order_ids = OrderItem.objects.filter(
          product_id=product_id,
          order__status__in=[Order.Status.PROCESSING, Order.Status.SHIPPED, Order.Status.DELIVERED],
      ).values_list("order_id", flat=True).distinct()

      co_occurring_product_ids = (
          OrderItem.objects.filter(order_id__in=order_ids)
          .exclude(product_id=product_id)
          .exclude(product_id__isnull=True)
          .values("product_id")
          .annotate(co_occurrence_count=Count("order_id", distinct=True))
          .order_by("-co_occurrence_count")[:limit]
      )
      product_ids_ranked = [row["product_id"] for row in co_occurring_product_ids]
      if not product_ids_ranked:
          return []

      # preserve the co-occurrence ranking order (a plain .filter(id__in=...)
      # does NOT preserve list order — fetch then re-sort in Python to keep
      # the actual ranking rather than whatever arbitrary DB order comes back)
      products = Product.objects.filter(id__in=product_ids_ranked).select_related("category", "brand")
      products_by_id = {p.id: p for p in products}
      return [products_by_id[pid] for pid in product_ids_ranked if pid in products_by_id]
  ```
  Note the deliberate exclusion of `PENDING`/`CANCELLED` orders — an
  order that was never actually completed (abandoned payment, per Epic
  6, or fully cancelled) shouldn't count as a real "customers bought
  these together" signal; only orders that genuinely progressed past
  payment confirmation should count.
  Note `.exclude(product_id__isnull=True)` — per Epic 4/8's stock-
  restoration and general cross-app patterns, `OrderItem.product` can
  be `None` if the underlying product was later deleted; such rows
  can't meaningfully recommend "a deleted product," so exclude them.
  Note the explicit re-sort-in-Python step after the `filter(id__in=...)`
  fetch — this is a genuine, easy-to-miss correctness detail: Django's
  `filter(id__in=[...])` does NOT preserve the order of the ID list
  you passed in, so without this re-sort step, the carefully-computed
  co-occurrence RANKING would be silently discarded by the time results
  reach the API response.
- Add `FrequentlyBoughtTogetherView`:
  ```python
  class FrequentlyBoughtTogetherView(generics.ListAPIView):
      permission_classes = [AllowAny]
      serializer_class = ProductListSerializer
      pagination_class = None

      def get_queryset(self):
          slug = self.kwargs.get("slug")
          product = get_object_or_404(Product, slug=slug)
          return get_frequently_bought_together(product.id)
  ```
  Import `get_frequently_bought_together` from `.recommendations` at
  the top of shop/views.py.
  Register the URL: `path("products/<slug:slug>/frequently-bought-together/", views.FrequentlyBoughtTogetherView.as_view(), name="product-fbt"),`
  in shop/urls.py — placed alongside the existing
  `products/<slug:slug>/related/`-style URL pattern
  (`RelatedProductsView`'s existing registration) for consistency.
- Performance consideration: this query touches `OrderItem` across
  potentially many orders for a popular product — for now, a direct,
  un-cached query is acceptable given this is a read replica-friendly,
  cacheable-later concern (Epic 21's caching work, if landed by this
  point, is the natural place to add a short-TTL cache on this specific
  endpoint given co-occurrence patterns change slowly; if Epic 21
  hasn't landed yet, don't build caching infrastructure prematurely
  inside this task — note it as a natural future optimization instead).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_recommendations.py (new file):
1. Two products that appear together in 3 completed orders, and a
   third product that appears together in only 1 completed order with
   the target product, are ranked with the 3-co-occurrence product
   FIRST.
2. A product that ONLY co-occurs in a `CANCELLED` or `PENDING` order is
   NOT included in results (the completed-orders-only filter).
3. The target product itself never appears in its own results (the
   `.exclude(product_id=product_id)` guard).
4. A product with NO purchase history / no co-occurring products
   returns an empty list, not an error.
5. Results respect the `limit` parameter (default 4, confirm a
   different limit value works if you parameterize it via query param
   — the backlog doesn't explicitly require this, a fixed sensible
   default is fine, but if you DO expose it, test it).
6. The API endpoint correctly serializes results using
   `ProductListSerializer` (the same lightweight serializer used
   elsewhere for listing contexts, not the heavier detail serializer).
```

---

#### Task 13.1.1.2 — "Recently viewed" tracking

```
You are working in frontend/src (primarily) and, optionally,
backend/shop/ if you choose a backend-persisted approach. Assume Epics
1–12 are fully merged.

CONTEXT
Nothing tracks which products a customer has recently viewed — a
common, valuable feature (helps a customer find their way back to a
product they were considering) currently absent.

TASK
Track recently-viewed products and expose them for display (this
task's own acceptance criteria is the tracking + storage + retrieval
mechanism; Task 13.1.1.4's homepage section is one CONSUMER of this
data, and Task 13.1.1.3 doesn't consume it at all — keep this task
scoped to the tracking mechanism itself).

REQUIREMENTS — DECIDE THE STORAGE APPROACH DELIBERATELY
Two reasonable approaches exist; pick ONE and implement it fully rather
than a half-built hybrid:

**Option A — Frontend-only (localStorage), simpler, works for guests:**
- Maintain a capped list (e.g. last 20) of `{productId, slug, name,
  thumbnail, price, viewedAt}` in `localStorage` under a dedicated key
  (e.g. `chiz_recently_viewed`).
- On the product detail page mount, push the current product to the
  front of this list (de-duplicating if already present — move it to
  front rather than adding a second entry), trim to the cap, and
  persist back to `localStorage`.
- Build a small utility module (e.g.
  `frontend/src/utils/recentlyViewed.js`) exposing `addRecentlyViewed(product)`
  and `getRecentlyViewed()`, used by both `ProductDetails.jsx` (to
  record a view) and Task 13.1.1.4's homepage section (to display the
  list) — a single source of truth for this logic, not duplicated
  across components.
- This approach is simpler, has zero backend changes, and naturally
  works for guest/anonymous visitors (no auth required) — but it's
  DEVICE-LOCAL only (a customer's recently-viewed list on their phone
  won't appear on their laptop) and is lost if they clear browser data.

**Option B — Backend-persisted, cross-device, requires auth for real
persistence:**
- Add a `RecentlyViewedProduct` model
  (user FK + product FK + `viewed_at`, `unique_together` on
  user+product with `viewed_at` UPDATED on repeat views rather than
  creating duplicate rows), a `POST /api/products/{slug}/record-view/`
  endpoint called on PDP mount, and a `GET /api/products/recently-viewed/`
  endpoint for retrieval.
- This requires authentication to be meaningful (an anonymous visitor
  has no stable cross-session identity to persist against, mirroring
  the exact same limitation already discussed for coupons in Epic 9 and
  price-drop notifications in Epic 11) — for guest visitors, you'd
  still need a localStorage-based fallback anyway, meaning Option B
  doesn't actually ELIMINATE the need for Option A's mechanism, it adds
  a SECOND mechanism on top of it for authenticated users specifically.

RECOMMENDATION: implement Option A (localStorage-only) for this task.
It fully satisfies the backlog's stated requirement ("Recently viewed
tracking"), works correctly for every visitor including guests without
any backend changes, and avoids taking on the real complexity of
merging/reconciling two different storage mechanisms for a feature
whose value (jogging a customer's memory of what they were just
looking at) doesn't meaningfully depend on cross-device sync. Document
this decision clearly, and note that a future cross-device version
(Option B) remains a reasonable enhancement if real user research ever
indicates the device-local limitation is actually costing conversions
— don't build it preemptively without evidence it's needed.

REQUIREMENTS (implementing Option A)
- `frontend/src/utils/recentlyViewed.js`:
  ```javascript
  const STORAGE_KEY = 'chiz_recently_viewed';
  const MAX_ITEMS = 20;

  export function addRecentlyViewed(product) {
    const existing = getRecentlyViewed();
    const filtered = existing.filter((p) => p.id !== product.id);
    const updated = [
      { id: product.id, slug: product.slug, name: product.name, thumbnail: product.thumbnail, price: product.price, viewedAt: Date.now() },
      ...filtered,
    ].slice(0, MAX_ITEMS);
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(updated));
    } catch (e) {
      // localStorage can throw in private-browsing modes or when full —
      // fail silently, this is a non-critical enhancement feature
    }
  }

  export function getRecentlyViewed() {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      return raw ? JSON.parse(raw) : [];
    } catch (e) {
      return [];
    }
  }

  export function clearRecentlyViewed() {
    try {
      localStorage.removeItem(STORAGE_KEY);
    } catch (e) { /* ignore */ }
  }
  ```
  The `try/catch` wrapping is deliberate and necessary — `localStorage`
  access can throw in Safari private browsing mode and similar
  restricted contexts; a "recently viewed" feature silently failing is
  far preferable to it crashing the entire product page.
- Call `addRecentlyViewed(product)` in `ProductDetails.jsx`, in the
  `useEffect` that already fetches product details on mount (add the
  call once product data is successfully loaded — don't call it before
  you actually have the product's name/thumbnail/price to store).
- Exclude the CURRENTLY-being-viewed product from its own recently-
  viewed display if you build a "you might also want to revisit" widget
  ON the PDP itself (this task doesn't require building such a widget —
  that's for whoever consumes `getRecentlyViewed()`, e.g. Task 13.1.1.4
  — but if you do add any recently-viewed display elsewhere, remember
  this exclusion).

ACCEPTANCE CRITERIA / TESTS
Add tests to frontend/src/utils/__tests__/recentlyViewed.test.js:
1. `addRecentlyViewed()` adds a new product to the front of the list.
2. Adding an ALREADY-present product moves it to the front (updates
   `viewedAt`) rather than creating a duplicate entry.
3. The list is capped at `MAX_ITEMS` — adding a 21st distinct product
   drops the oldest one.
4. `getRecentlyViewed()` returns `[]` (not an error) when
   `localStorage` is empty or contains malformed JSON.
5. Mock `localStorage.setItem` to throw (simulating private-browsing
   mode) and confirm `addRecentlyViewed()` doesn't propagate the
   exception.
Manually verify: viewing several products in sequence, then checking
`localStorage` in browser dev tools, shows the expected capped,
most-recent-first list.
```

---

#### Task 13.1.1.3 — "Customers also bought" widget on PDP

```
You are working in frontend/src/pages/ProductDetails.jsx. Assume Task
13.1.1.1 is already merged. Re-read this document's header context
before starting — this task ADDS a new section, it does not touch the
existing "Related Products" section at all.

CONTEXT
`ProductDetails.jsx` already renders a "Related Products" section
(category-based, via the existing `RelatedProductsView`/
`getRelatedProducts` — confirmed unchanged, do not modify it). This
task adds a SEPARATE, second section powered by Task 13.1.1.1's
behavioral co-occurrence endpoint, giving the page two genuinely
different recommendation types: "Related Products" (same category) and
"Customers Also Bought" (actual purchase co-occurrence) — both valuable,
neither redundant with the other.

TASK
Add a "Customers Also Bought" section to the product detail page,
fetching from Task 13.1.1.1's endpoint.

REQUIREMENTS
- Add `getFrequentlyBoughtTogether: (slug) => api.get(`/products/${slug}/frequently-bought-together/`)`
  to frontend/src/services/api.js, matching the EXACT existing style of
  the already-present `getRelatedProducts` method (check its exact
  signature/implementation and mirror it precisely for consistency).
- In `ProductDetails.jsx`, add a new state variable
  (`frequentlyBoughtTogether`, mirroring the existing `relatedProducts`
  state variable's naming/structure) and fetch it in the SAME
  `useEffect`/`Promise.all` block that already fetches related
  products (check the existing code around the `getRelatedProducts`
  call — likely a `Promise.all([...])` fetching several things
  together on mount — add this new fetch alongside it rather than
  introducing a separate effect, for consistency and to avoid an extra
  render/loading-state cycle).
  ```javascript
  getFrequentlyBoughtTogether(slug).catch(() => ({ data: [] })),
  ```
  (matching the existing `.catch(() => ({ data: [] }))` graceful-
  degradation pattern already used for `getRelatedProducts` — a failed
  recommendation fetch should never break the whole product page, it
  should just result in that section not rendering).
- Render a new "Customers Also Bought" section, positioned near (but
  visually distinct from) the existing "Related Products" section —
  reasonable placement is directly ABOVE or BELOW it, using the exact
  same card component (`RelatedCard`, per the existing code) for visual
  consistency, just with a different heading and a different data
  source:
  ```jsx
  {frequentlyBoughtTogether.length > 0 && (
    <div className="mt-16">
      <div className="flex items-center justify-between mb-6">
        <h2 className="text-2xl font-bold text-gray-800">Customers Also Bought</h2>
      </div>
      <div className="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4 gap-5">
        {frequentlyBoughtTogether.map((p) => (
          <RelatedCard key={p.id} product={p} />
        ))}
      </div>
    </div>
  )}
  ```
  (no "View All" link here, unlike the existing Related Products
  section — there's no dedicated "frequently bought together" browsing
  page/category this could link to, so omit that element rather than
  linking somewhere that doesn't make sense).
- Handle the empty-results case (a product with no purchase-history-
  based recommendations yet, e.g. a brand-new product with zero orders)
  by simply not rendering the section at all, matching the existing
  `relatedProducts.length > 0 &&` conditional-rendering pattern already
  used for the Related Products section.

ACCEPTANCE CRITERIA / TESTS
Update frontend/src/__tests__/ProductDetails.test.jsx (which already
exists per grounding, with established mocking patterns for
`getRelatedProducts`) to add equivalent coverage for
`getFrequentlyBoughtTogether`:
1. Renders the "Customers Also Bought" section when results are
   present.
2. Doesn't render the section when results are empty.
3. A failed fetch (mocked rejection) doesn't break the page — the
   section simply doesn't appear, and the REST of the page (including
   the unrelated "Related Products" section) still renders correctly.
4. Confirm BOTH sections ("Related Products" AND "Customers Also
   Bought") can render simultaneously without interfering with each
   other, using distinct mocked data for each to prove they're
   genuinely independent data sources feeding independent UI sections.
```

---

#### Task 13.1.1.4 — Personalized homepage section (based on browsing/purchase history)

```
You are working in frontend/src (Home page) and backend/shop/
recommendations.py. Assume Tasks 13.1.1.1 and 13.1.1.2 are already
merged.

CONTEXT
The homepage currently has no personalization at all — every visitor
sees the identical page regardless of browsing or purchase history.
This task adds a LIGHTWEIGHT, HEURISTIC personalization layer — the
backlog explicitly frames this as "basic category affinity scoring,"
NOT machine learning — building a simple, explainable signal from
available data rather than a complex recommendation model.

TASK
Add a "Recommended For You" (or "Because You Viewed..." for a specific
recently-viewed item) homepage section, using: (a) Task 13.1.1.2's
recently-viewed data to seed a lightweight category-affinity query for
guests/all visitors, and (b) for AUTHENTICATED users specifically,
optionally blend in their actual purchase-category history too, since
that's a stronger signal than browsing alone when available.

REQUIREMENTS — backend
- Add `get_category_affinity_recommendations()` to
  backend/shop/recommendations.py:
  ```python
  def get_category_affinity_recommendations(
      viewed_product_ids: list[int], purchased_category_ids: list[int] | None = None, limit: int = 8
  ) -> list[Product]:
      """
      Simple heuristic: recommend products from categories the visitor
      has recently viewed or purchased from, excluding products they've
      already viewed, favoring higher-rated products within those
      categories. This is NOT a machine-learning recommender — it is a
      deliberately simple, explainable category-affinity heuristic.
      """
      category_ids = set(purchased_category_ids or [])
      if viewed_product_ids:
          viewed_categories = Product.objects.filter(
              id__in=viewed_product_ids
          ).values_list("category_id", flat=True)
          category_ids.update(viewed_categories)

      if not category_ids:
          return []

      return list(
          Product.objects.filter(category_id__in=category_ids)
          .exclude(id__in=viewed_product_ids)
          .select_related("category", "brand")
          .order_by("-rating", "-reviews_count")[:limit]
      )
  ```
- Add `HomeRecommendationsView`:
  ```python
  class HomeRecommendationsView(APIView):
      permission_classes = [AllowAny]

      def post(self, request):
          # viewed_product_ids comes from the CLIENT (localStorage, per
          # Task 13.1.1.2's frontend-only storage decision) — the
          # backend has no server-side record of an anonymous visitor's
          # browsing history, so the client must supply it. This is a
          # deliberate consequence of Task 13.1.1.2's Option A choice.
          viewed_product_ids = request.data.get("viewed_product_ids", [])[:20]
          purchased_category_ids = None
          if request.user.is_authenticated:
              from order.models import Order, OrderItem
              purchased_category_ids = list(
                  OrderItem.objects.filter(
                      order__user=request.user,
                      order__status__in=[Order.Status.PROCESSING, Order.Status.SHIPPED, Order.Status.DELIVERED],
                  ).values_list("product__category_id", flat=True).distinct()
              )
          products = get_category_affinity_recommendations(viewed_product_ids, purchased_category_ids)
          return Response(ProductListSerializer(products, many=True, context={"request": request}).data)
  ```
  This is deliberately a `POST` (not `GET`) endpoint despite not really
  "creating" anything, because the client needs to send a
  potentially-nontrivial `viewed_product_ids` array in the request BODY
  — a `GET` request with a large array crammed into query parameters is
  both awkward and has practical URL-length limits; a `POST` with a
  JSON body is the more appropriate tool here even though it's
  read-only in effect. Note this deliberately breaks REST purism for a
  practical reason, and that's a defensible, common real-world tradeoff
  — document it in a docstring/comment so it doesn't look like an
  oversight to a future reader.
  Register the URL: `path("products/home-recommendations/", views.HomeRecommendationsView.as_view(), name="home-recommendations"),`
  in shop/urls.py.

REQUIREMENTS — frontend
- Add `getHomeRecommendations: (viewedProductIds) => api.post('/products/home-recommendations/', { viewed_product_ids: viewedProductIds })`
  to api.js.
- On the Home page, after mount, call `getRecentlyViewed()` (Task
  13.1.1.2's utility) to get the local viewing history, extract just
  the product IDs, and call `getHomeRecommendations()` with them.
- Render a "Recommended For You" section with the results, using the
  same product-card component already used elsewhere on the homepage
  for visual consistency (check the Home page's existing product
  sections — e.g. "New Arrivals"/"Best Sellers" if such sections
  already exist — and match their exact card component/layout).
- Handle the NO-HISTORY case (a brand-new visitor with an empty
  recently-viewed list AND no purchase history) — the backend correctly
  returns an empty list in this case (per
  `get_category_affinity_recommendations`'s `if not category_ids: return []`
  guard); the frontend should simply not render the section at all
  rather than showing an empty/broken widget, consistent with every
  other optional-recommendation section built in this epic.
- Consider the section's HEADING: "Recommended For You" reads oddly for
  a first-time anonymous visitor with zero signal (though this case
  naturally never renders anything per the guard above, so it's mostly
  moot) — for a returning visitor with SOME recently-viewed history but
  no purchase history, "Because You Viewed..." framing (referencing
  their most recent view) may read more naturally than a generic
  "Recommended For You" — this is a UX judgment call, not a hard
  requirement; pick whichever framing you think reads better and is
  simplest to implement correctly, and don't over-engineer dynamic
  copy generation if a single consistent heading is simpler and still
  reasonable.

ACCEPTANCE CRITERIA / TESTS
Add backend tests to backend/shop/tests/test_recommendations.py:
1. `get_category_affinity_recommendations()` with viewed products from
   category A returns other, higher-rated products from category A,
   excluding the already-viewed products themselves.
2. With BOTH viewed-category and purchased-category signals present,
   results draw from the UNION of both categories (not just one or the
   other).
3. With no viewed products and no purchase history, returns an empty
   list.
4. `HomeRecommendationsView` for an AUTHENTICATED user with real
   purchase history correctly incorporates their purchased-category
   affinity even if `viewed_product_ids` sent from the client is empty.
5. `HomeRecommendationsView` for an ANONYMOUS user relies entirely on
   the client-supplied `viewed_product_ids` (since there's no
   server-side history to fall back on for a guest) — confirm an
   anonymous request with a populated `viewed_product_ids` array still
   produces sensible results.
Add a frontend test confirming the homepage section renders correctly
with mocked recommendations, doesn't render when results are empty,
and correctly reads from `getRecentlyViewed()` to build its request.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 13.1.1.1 | "Frequently bought together" (OrderItem co-occurrence) | ☐ |
| 13.1.1.2 | "Recently viewed" tracking | ☐ |
| 13.1.1.3 | "Customers also bought" widget on PDP | ☐ |
| 13.1.1.4 | Personalized homepage section | ☐ |

Once Epic 13 is fully merged, the next epics to generate prompts for
are **Epic 14 — Persian Localization & i18n** and **Epic 15 — SEO**,
which the master backlog flags as large, mostly-independent epics that
should ideally run in parallel with the feature epics rather than being
saved entirely for last — Epic 14 in particular touches currency
handling, which several already-merged epics' financial code paths
depend on getting right.
