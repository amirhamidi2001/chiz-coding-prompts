# Epic 29 — Frontend Architecture Refactor — AI Coding Prompts

Repo: `tablogenix-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next. This epic directly closes out the ORIGINAL architecture review's frontend findings from the very start of this whole project — "no code-splitting," "no data-fetching/caching library," "no form library" — none of which were addressed incidentally by any epic in between; they're still exactly as they were on day one.

**Assumed prerequisites:** Epics 1–28 are fully merged. **This is a large amount of accumulated frontend surface area to refactor by this point** — 28 epics' worth of pages, contexts, and forms — treat several tasks in this epic (especially 29.1.2.2, 29.1.2.3, and the broader sweep implied by 29.1.1.1) as legitimately multi-session efforts, matching the same honest scoping already applied to Epic 14's RTL sweep and translation-string wrapping.

**Confirmed directly from the repo — the exact scope of what's being fixed:** `frontend/src/App.jsx` imports **every single page** — 11 admin pages (`AdminDashboard`, `AdminAnalytics`, `AdminOrders`, `AdminProducts`, `AdminCategories`, `AdminBrands`, `AdminUsers`, `AdminReviews`, `AdminMessages`, `AdminChat`, `AdminBlog`) and dozens of storefront pages — **eagerly, at the top of the file, with zero `React.lazy()` calls anywhere**. `frontend/vite.config.js` is the **bare, unmodified default** (`{ plugins: [react()] }`, no `build.rollupOptions.output.manualChunks`, no chunking configuration of any kind). `frontend/package.json` has **no `@tanstack/react-query`, no `react-hook-form`, no `zod`, no `formik`** — every data fetch across every epic in this series has gone through plain `useEffect` + direct `axios` calls, and every form (Checkout, Login, OTP, admin forms) has used hand-rolled `validate()` functions and local `useState`. This means today, literally every visitor to the storefront — including one who never touches `/admin` at all — downloads the code for the **entire admin dashboard** on first page load.

---

## Phase 29.1 — Performance Refactor

### Feature 29.1.1 — Code Splitting

---

#### Task 29.1.1.1 — Convert all route imports in `App.jsx` to `React.lazy()`

```
You are working in frontend/src/App.jsx. Assume Epics 1–28 are fully
merged. RE-READ THIS DOCUMENT'S HEADER — the eager-import list
confirmed there is the actual, complete scope of this task.

CONTEXT
Every route component is statically imported, meaning Vite/Rollup
bundles ALL of them into the initial JavaScript payload every visitor
downloads on first load, regardless of which single page they actually
requested — a customer landing on the homepage currently downloads the
entire admin dashboard's code (11 admin pages' worth of components,
charts, forms) for no benefit whatsoever.

TASK
Convert every route-level page import to `React.lazy()`, wrapped in a
single top-level `<Suspense>` boundary.

REQUIREMENTS
- Replace every static page import:
  ```javascript
  // before
  import Home from './pages/Home';
  import AdminDashboard from './pages/admin/AdminDashboard';

  // after
  import { lazy } from 'react';
  const Home = lazy(() => import('./pages/Home'));
  const AdminDashboard = lazy(() => import('./pages/admin/AdminDashboard'));
  ```
  Apply this to EVERY page-level import confirmed in this document's
  header — every `Admin*` page and every storefront page — this is a
  mechanical but exhaustive sweep; do not leave a subset un-converted
  "for now," since a route accidentally left as a static import
  defeats the purpose for THAT specific page and silently pulls its
  code back into the main bundle.
- Do NOT lazy-load: context providers (`AuthProvider`/`CartProvider`/
  `WishlistProvider`), shared layout components used on EVERY page
  (`Header`, `Footer`, `ChatWidget`, `AdminLayout`) — these are needed
  immediately regardless of which route is active, so lazy-loading them
  would just add an unnecessary loading flicker with no bundle-size
  benefit (they're shared across every route anyway, so they'd end up
  in the initial bundle either way via the shared-chunk mechanism Task
  29.1.1.2 sets up next).
- Wrap the `<Routes>` tree in a SINGLE `<Suspense>` boundary at the
  point where route switching happens:
  ```jsx
  import { Suspense } from 'react';
  import PageLoadingFallback from './components/PageLoadingFallback'; // built in Task 29.1.1.3

  <Suspense fallback={<PageLoadingFallback />}>
    <Routes>
      {/* ...unchanged route definitions... */}
    </Routes>
  </Suspense>
  ```
  A single top-level Suspense boundary (rather than one per-route) is
  the correct, simpler choice here — it means EVERY route transition
  shows the SAME fallback while its chunk loads, which is consistent,
  predictable UX; per-route Suspense boundaries would let you show
  DIFFERENT fallbacks per page, which is unnecessary complexity for
  this project's actual need (Task 29.1.1.3 builds a reasonable, single
  generic fallback rather than page-specific skeletons — see that
  task for the more targeted skeleton work, which composes WITH this
  single top-level boundary via nested Suspense where it adds real
  value, not by replacing this top-level one).
- Confirm `React.lazy()` works correctly with this project's existing
  route-guard components (any `<ProtectedRoute>`/`<AdminRoute>`-style
  wrapper components established across prior epics for auth-gating —
  check `App.jsx`'s route definitions for how admin/authenticated
  routes are currently protected, and confirm the lazy-loaded component
  still correctly receives whatever props/context that guard mechanism
  relies on).

ACCEPTANCE CRITERIA / TESTS
- Manually verify via browser dev tools' Network tab: loading the
  HOMEPAGE downloads a JS bundle that does NOT include any
  `Admin*`-named chunk — confirm by checking the Network tab's loaded
  JS files after a fresh homepage load, none should be admin-page-
  named chunks (this is the direct, concrete proof the eager-loading
  problem described in this task's header is actually fixed).
- Navigating TO an admin page (as an authenticated admin) correctly
  triggers loading of that page's chunk on demand, with the fallback
  UI showing briefly during the load.
- Re-run the full existing frontend test suite (Vitest component
  tests) — `React.lazy()` components need to be handled correctly in
  tests that render them (React Testing Library requires wrapping
  assertions in `waitFor`/using `Suspense` correctly in test renders
  for lazy components) — fix any test that breaks due to this change
  rather than reverting the lazy-loading to make tests pass; the tests
  should adapt to the (correct, intentional) new loading behavior, not
  the other way around.
```

---

#### Task 29.1.1.2 — Split admin bundle from storefront bundle

```
You are working in frontend/vite.config.js. Assume Task 29.1.1.1 is
already merged.

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
`vite.config.js` is the bare Vite scaffold default — `{ plugins: [react()] }`
— with zero build/chunking configuration. Task 29.1.1.1's per-page
`React.lazy()` conversion already means each individual admin page
loads its OWN chunk on demand — but without explicit chunk-grouping
configuration, Vite/Rollup's DEFAULT chunking heuristics may still
produce a suboptimal split: shared admin-only dependencies (e.g. a
charting library used only by `AdminAnalytics`, if such a dependency
exists) could end up duplicated across multiple admin-page chunks
rather than correctly de-duplicated into one shared "admin vendor"
chunk, and there's no EXPLICIT guarantee admin-specific code never
leaks into the shared/storefront chunk under Rollup's default
heuristics.

TASK
Add explicit Rollup chunking configuration ensuring a clean, verified
separation between storefront and admin code, with shared vendor
dependencies correctly de-duplicated.

REQUIREMENTS
- Configure `build.rollupOptions.output.manualChunks` in
  `vite.config.js`:
  ```javascript
  import { defineConfig } from 'vite';
  import react from '@vitejs/plugin-react';
  import path from 'path';

  export default defineConfig({
    plugins: [react()],
    build: {
      rollupOptions: {
        output: {
          manualChunks(id) {
            if (id.includes('node_modules')) {
              // Group all third-party vendor code into one shared chunk —
              // rarely changes between deploys, so it benefits most from
              // long-term browser caching across releases.
              return 'vendor';
            }
            if (id.includes(path.sep + 'pages' + path.sep + 'admin' + path.sep) ||
                id.includes(path.sep + 'components' + path.sep + 'admin' + path.sep)) {
              // Explicit, verified grouping — everything under
              // pages/admin/ or components/admin/ goes into a distinct
              // 'admin' chunk, regardless of Rollup's default heuristics.
              return 'admin';
            }
          },
        },
      },
    },
  });
  ```
  Adjust the exact path-matching logic to correctly reflect this
  project's REAL directory structure for admin-specific components
  (confirmed `pages/admin/` exists; verify whether admin-specific
  SHARED components — e.g. `AdminLayout`, any admin-only form/table
  components — live under a `components/admin/` directory too, and
  include that path in the same grouping logic, so admin-specific
  shared UI components ALSO land in the admin chunk, not the general
  shared chunk that ships to every storefront visitor).
- Verify the split ACTUALLY worked by inspecting the real build
  output — run `npm run build` and inspect `dist/assets/` for the
  generated chunk files; confirm an `admin-[hash].js` chunk exists,
  separate from the main/vendor chunks, and — critically — verify NO
  admin-page-specific code appears in the MAIN/index chunk (the one
  loaded on every page) by checking its content/size doesn't include
  anything admin-specific — a chunking config that LOOKS correct in
  the vite.config.js file but doesn't actually achieve real separation
  in the build output would be a silent failure of this task's whole
  purpose.
- Add a bundle-size regression check to CI (extending Epic 25's
  established CI-hardening pattern): a step in the `frontend` job
  (`.github/workflows/ci.yml`) running `npm run build` and asserting
  the MAIN/initial chunk's size stays under some reasonable threshold
  (e.g. using a tool like `bundlesize`/`size-limit`, or a simple shell
  check against the built file sizes) — RECOMMEND adding this as
  `continue-on-error: true` initially (matching this project's now-
  well-established two-phase-rollout pattern for new CI checks, per
  Epic 23/24's precedent) until a real, considered size budget is
  established from actual measured output, rather than guessing at a
  threshold number blind.

ACCEPTANCE CRITERIA / TESTS
- `npm run build` produces a distinct `admin-*.js` chunk, verified by
  inspecting real build output (not just config correctness).
- Add a simple build-output test/script (can be a small Node script run
  in CI, not necessarily a Vitest test) asserting the main/initial JS
  bundle size is meaningfully SMALLER after this task than it was
  before (compare against a recorded baseline from before Task 29.1.1.1/
  29.1.1.2, if you have access to measure that, or simply document the
  before/after sizes in your task summary as the concrete evidence this
  task achieved its goal).
- Confirm the storefront still functions completely correctly after
  the build (a chunking misconfiguration can sometimes produce a
  build that WORKS in dev mode but breaks in the actual production
  build due to subtle module-loading-order issues — test against the
  REAL production build output, not just `npm run dev`).
```

---

#### Task 29.1.1.3 — Add route-level loading skeletons

```
You are working in frontend/src/components/PageLoadingFallback.jsx
(new component) and, optionally, page-specific skeleton components.
Assume Task 29.1.1.1 is already merged (which already wired a
`PageLoadingFallback` placeholder into the top-level Suspense boundary
— this task builds the actual component that was referenced there).

CONTEXT
Task 29.1.1.1's `<Suspense fallback={<PageLoadingFallback />}>` needs
an ACTUAL implementation — without one, every route transition would
show nothing/a blank screen during the (now-real, chunk-loading) delay
introduced by code-splitting, which is a worse experience than before
code-splitting existed at all if handled carelessly.

TASK
Build a generic, reasonable top-level loading fallback, plus (for the
highest-traffic pages specifically) more TARGETED skeleton screens that
approximate the actual page layout, reducing perceived loading time
and layout shift.

REQUIREMENTS
- Build a simple, generic `PageLoadingFallback.jsx` for the top-level
  Suspense boundary (Task 29.1.1.1) — a centered spinner or a subtle
  top-of-page progress bar (check whether this project has any
  existing loading-indicator pattern from prior epics' work, e.g. how
  individual data-fetching loading states are already shown elsewhere,
  and match that established visual language rather than introducing a
  third, inconsistent loading-UI style):
  ```jsx
  export default function PageLoadingFallback() {
    return (
      <div className="flex items-center justify-center min-h-[60vh]">
        <div className="animate-spin rounded-full h-10 w-10 border-b-2 border-teal-600" />
      </div>
    );
  }
  ```
- For the HIGHEST-traffic pages specifically (product listing/category
  page, product detail page — the pages a customer is MOST likely to
  navigate to and therefore most likely to actually experience a
  chunk-loading delay on), build more TARGETED skeleton screens that
  roughly match the real page's eventual layout (a grid of gray
  placeholder boxes matching where product cards will appear, a
  placeholder image + text-line shapes matching the PDP's real layout)
  — this reduces perceived load time and, more importantly, minimizes
  LAYOUT SHIFT when the real content finally replaces the skeleton
  (a skeleton that's shaped roughly like the real content prevents the
  page from visibly "jumping" once data arrives, a genuine, measurable
  UX quality signal, not just cosmetic polish).
  Wire these TARGETED skeletons via NESTED `<Suspense>` boundaries
  specifically around those high-traffic routes (composing with, not
  replacing, the top-level generic fallback from Task 29.1.1.1 — the
  top-level one remains the fallback for every OTHER route that
  doesn't have its own dedicated skeleton):
  ```jsx
  <Route
    path="/products"
    element={
      <Suspense fallback={<ProductListSkeleton />}>
        <ProductList />
      </Suspense>
    }
  />
  ```
- Don't over-invest in this task — a handful of targeted skeletons for
  the 2-3 highest-traffic pages, plus the generic fallback for
  everything else, is a reasonable, proportionate scope; building a
  perfectly custom skeleton for every single one of the 30+ routes in
  this project would be disproportionate effort for the actual
  marginal UX benefit beyond that point.

ACCEPTANCE CRITERIA / TESTS
- Add component tests confirming `PageLoadingFallback` and each
  targeted skeleton component render without error.
- Manually verify (throttle network speed in browser dev tools to make
  the loading window visibly observable): navigating to the product
  listing/PDP pages shows the TARGETED skeleton, not the generic
  fallback; navigating to any other, non-targeted route shows the
  generic fallback correctly.
```

---

### Feature 29.1.2 — Data Fetching Layer

---

#### Task 29.1.2.1 — Introduce React Query (TanStack Query)

```
You are working in frontend/package.json, main.jsx. Assume Epics 1–28
are fully merged, and Feature 29.1.1 is complete.

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
No `@tanstack/react-query` (or any data-fetching/caching library)
exists anywhere in this project — every single data fetch across every
epic in this entire series has been hand-rolled: a `useEffect` calling
a plain `axios`-based API function, manually managing loading/error
state via local `useState`, with NO request deduplication (two
components independently fetching the SAME data on the same page make
two separate, redundant network requests), NO automatic
background-refetch/staleness handling, and inconsistent loading/error-
state UX patterns duplicated slightly differently in every single
component that fetches data.

TASK
Install and configure TanStack Query as this project's data-fetching
layer, establishing shared conventions (query key structure, default
options) that Tasks 29.1.2.2/29.1.2.3 then use to migrate real fetch
sites onto.

REQUIREMENTS
- Add `@tanstack/react-query` to frontend/package.json, pinned to a
  current stable version. Consider also adding
  `@tanstack/react-query-devtools` as a DEV dependency (invaluable for
  debugging cache behavior during this migration specifically).
- Set up the `QueryClientProvider` in main.jsx/App.jsx, OUTSIDE (wrapping)
  the existing provider tree established across prior epics
  (`HelmetProvider` from Epic 15, `AuthProvider`/`CartProvider`/
  `WishlistProvider`):
  ```jsx
  import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,        // data considered fresh for 1 minute before a background refetch is triggered — a reasonable default for most of this project's data, tune per-query where a specific endpoint needs something different
        retry: 1,                    // one automatic retry on failure — matches a reasonable balance between resilience and not silently retrying a genuinely broken request indefinitely
        refetchOnWindowFocus: false, // avoid a surprising refetch storm every time a user tabs back to the site — a deliberate choice worth documenting, since React Query's OWN default is `true`; explicitly overriding it here is a considered decision for this specific project's UX, not an oversight
      },
    },
  });

  <QueryClientProvider client={queryClient}>
    <HelmetProvider>
      {/* ...existing provider tree... */}
    </HelmetProvider>
  </QueryClientProvider>
  ```
- Establish a QUERY KEY CONVENTION document (e.g. a comment block in a
  new frontend/src/queries/queryKeys.js, or inline documentation) that
  Tasks 29.1.2.2/29.1.2.3 will follow consistently — a centralized,
  factory-function-based key structure avoids the common React Query
  pitfall of scattered, inconsistently-shaped string keys across many
  files that are hard to correctly invalidate together later:
  ```javascript
  // frontend/src/queries/queryKeys.js
  export const queryKeys = {
    products: {
      all: ['products'],
      list: (filters) => ['products', 'list', filters],
      detail: (slug) => ['products', 'detail', slug],
    },
    cart: {
      current: ['cart'],
    },
    orders: {
      all: ['orders'],
      list: (params) => ['orders', 'list', params],
      detail: (id) => ['orders', 'detail', id],
    },
    // extended incrementally as Tasks 29.1.2.2/29.1.2.3 migrate real fetch sites
  };
  ```
  This factory-function pattern (rather than raw inline array literals
  scattered per-component) is what makes CACHE INVALIDATION correct
  and maintainable later — e.g. `queryClient.invalidateQueries({ queryKey: queryKeys.products.all })`
  correctly invalidates every products-related query regardless of
  its specific filter/detail variant, which would be error-prone to
  get right with ad-hoc inline key arrays duplicated slightly
  differently across many components.
- Do NOT migrate any actual fetch sites in THIS task — this task is
  purely the library installation, provider setup, and key-convention
  establishment; Tasks 29.1.2.2/29.1.2.3 do the actual migration work.

ACCEPTANCE CRITERIA / TESTS
- Add a minimal smoke test confirming a component using `useQuery()`
  inside a test wrapped in a `QueryClientProvider` (a fresh
  `QueryClient` per test, to avoid cache state leaking between tests —
  a common React Query testing pitfall worth being explicit about
  avoiding) correctly fetches and renders data from a mocked API call.
- Manually verify: React Query DevTools (if installed) correctly show
  up in a local dev build, confirming the provider is correctly wired.
```

---

#### Task 29.1.2.2 — Migrate product listing/PDP fetches to React Query

```
You are working in frontend/src/pages/ProductDetails.jsx and the
product listing/category page component. Assume Task 29.1.2.1 is
already merged. This is the FIRST real migration — treat it carefully
as the pattern-setting example the remaining migration work (Task
29.1.2.3, and any future fetch sites) will follow.

CONTEXT
Per this document's header, `ProductDetails.jsx` currently fetches
product data (plus, per Epic 13's work, related products AND
frequently-bought-together data, via a `Promise.all` block) using
manual `useEffect` + `useState` + direct `axios` calls, with hand-
rolled loading/error state. This is the highest-value page to migrate
first — it's the highest-traffic page in the entire application, and
already has the MOST complex multi-source fetching logic (per Epic
13's additions) that would benefit most from React Query's built-in
parallel-query handling.

TASK
Migrate `ProductDetails.jsx` and the product listing page to React
Query, replacing manual `useEffect`/`useState` fetch logic entirely.

REQUIREMENTS
- Replace the PDP's current fetch logic:
  ```javascript
  // before (illustrative of the current established pattern)
  const [product, setProduct] = useState(null);
  const [loading, setLoading] = useState(true);
  useEffect(() => {
    setLoading(true);
    getProduct(slug).then(({ data }) => setProduct(data)).finally(() => setLoading(false));
  }, [slug]);

  // after
  import { useQuery } from '@tanstack/react-query';
  import { queryKeys } from '../queries/queryKeys';

  const { data: product, isLoading, isError, error } = useQuery({
    queryKey: queryKeys.products.detail(slug),
    queryFn: () => getProduct(slug).then((res) => res.data),
  });
  ```
  Apply the SAME pattern to the related-products/frequently-bought-
  together fetches (Epic 13), each as its OWN `useQuery` call with its
  own query key (`queryKeys.products.related(slug)`,
  `queryKeys.products.frequentlyBoughtTogether(slug)` — extend
  `queryKeys.js` accordingly) — React Query automatically parallelizes
  independent `useQuery` calls within the same component, which
  replaces the current manual `Promise.all` orchestration with
  something both simpler to read AND individually cacheable/
  independently-error-handled (a failure in the FBT fetch, per Epic
  13's own established "gracefully degrade, don't break the page"
  principle, should still be handled the SAME way under React Query —
  confirm each query's OWN `isError` state is checked independently
  before conditionally rendering that section, exactly matching Epic
  13's original resilience requirement, just implemented through React
  Query's per-query error state instead of manual `.catch()` handling).
- Migrate the product LISTING/category page similarly, with its query
  key correctly incorporating the active filter/search/sort state (per
  Epic 12's filter sidebar work):
  ```javascript
  const { data, isLoading } = useQuery({
    queryKey: queryKeys.products.list(currentFilters),
    queryFn: () => getProducts(currentFilters).then((res) => res.data),
  });
  ```
  This is a MEANINGFUL improvement over the current manual approach
  specifically for this page: React Query automatically handles
  request DEDUPLICATION and CACHING per unique filter combination — a
  customer toggling a filter on, then back off, then back on again
  within a short window will correctly serve the SECOND "on" state from
  cache rather than re-fetching, which the current hand-rolled
  `useEffect` approach does not do at all (every filter change
  currently triggers a fresh network request unconditionally,
  regardless of whether that exact combination was already fetched
  moments ago).
- Update loading/error UI to use React Query's `isLoading`/`isError`/
  `error` states consistently, matching this project's existing visual
  loading/error patterns (don't introduce a new visual style for
  loading/error states as part of this migration — the UNDERLYING
  fetch mechanism changes, the visual presentation should stay
  consistent with what's already established).

ACCEPTANCE CRITERIA / TESTS
Update the existing test files for these two pages (which currently
mock `axios`/the API service functions directly, per established
convention) to work correctly with React Query — this typically means
wrapping test renders in a `QueryClientProvider` with a fresh
`QueryClient` per test (matching Task 29.1.2.1's own testing guidance),
and confirming:
1. Product data renders correctly once the (mocked) query resolves.
2. Loading state renders correctly before resolution.
3. Error state renders correctly on a mocked query failure.
4. The related-products/FBT sections independently handle their OWN
   failure without breaking the rest of the page (re-confirming Epic
   13's original resilience requirement holds under the new fetching
   mechanism).
5. Navigating between two DIFFERENT products, then BACK to the first
   one within the `staleTime` window, correctly serves the first
   product's data from cache WITHOUT a new network request (mock the
   API call and assert it's only called ONCE for the first product
   despite being "visited" twice) — the concrete, direct proof this
   migration actually delivers React Query's caching benefit, not just
   that the code compiles with a different library.
```

---

#### Task 29.1.2.3 — Migrate cart/order fetches to React Query

```
You are working in frontend/src/context/CartContext.jsx and the order-
history/detail components (Epic 18's `OrdersTab.jsx`). Assume Task
29.1.2.2 is already merged (establishing the pattern this task
follows).

CONTEXT
`CartContext.jsx` and `OrdersTab.jsx` both currently manage their own
fetch/loading/error state manually — and CART data specifically has a
genuine, valuable use case for React Query's MUTATION handling
(`useMutation`), not just query caching, since cart operations
(add/update/remove item, apply coupon) are WRITES that need to
correctly invalidate and refresh the cached cart state afterward —
exactly the kind of read/write consistency React Query is designed to
handle cleanly.

TASK
Migrate cart state management and order history/detail fetching to
React Query, including mutation handling for cart write operations.

REQUIREMENTS
- Migrate `CartContext`'s cart-fetching to `useQuery`:
  ```javascript
  const { data: cart, isLoading } = useQuery({
    queryKey: queryKeys.cart.current,
    queryFn: () => getCart().then((res) => res.data),
  });
  ```
- Migrate cart WRITE operations (add/update/remove item, apply/remove
  coupon per Epic 9) to `useMutation`, with correct cache invalidation
  on success:
  ```javascript
  const addToCartMutation = useMutation({
    mutationFn: ({ variantId, quantity }) => addToCart(variantId, quantity),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.cart.current });
    },
  });
  ```
  This REPLACES the manual "refetch cart after every mutating action"
  pattern already established (and explicitly called out) across
  MULTIPLE prior epics — Epic 9 Task 9.1.1.9's coupon-apply UI, Epic
  11 Task 11.1.1.3's "move to cart" action, Epic 5's cross-context
  refresh pattern — every one of those call sites currently manually
  triggers a cart refetch after its own specific action; under React
  Query's `invalidateQueries` mechanism, this becomes automatic and
  centralized (any mutation anywhere in the app that invalidates the
  `queryKeys.cart.current` key correctly triggers React Query to
  refetch it, without each individual call site needing its own
  manual "now go refresh the cart" logic) — find and update EVERY one
  of those previously-manual refresh call sites to use this
  centralized invalidation mechanism instead, removing the now-
  redundant manual refetch calls.
- Migrate `OrdersTab.jsx`'s order-list and order-detail fetches to
  `useQuery` following the SAME pattern established in Task 29.1.2.2,
  including the invoice-download action (Epic 18 Task 18.1.1.2) and
  reorder action (Epic 18 Task 18.1.1.5) as `useMutation`s where they
  represent WRITES (reorder genuinely mutates the cart, so it should
  ALSO invalidate `queryKeys.cart.current` on success, correctly
  reflecting the newly-added items without a manual refresh call —
  another concrete case of the SAME centralization benefit described
  above).
- Handle `CartContext`'s AUTH-STATE-DEPENDENT behavior correctly under
  React Query — per Epic 5's guest-cart work, the cart query needs to
  re-fetch when auth state CHANGES (login/logout, particularly the
  post-login MERGE behavior from Epic 5 Task 5.1.1.3) — use React
  Query's `enabled` option and/or explicit `invalidateQueries` calls
  triggered from `AuthContext`'s login/logout methods (rather than
  relying on the EXISTING event-based `'auth-change'` listener pattern
  alone, per Epic 5 Task 5.1.1.4's grounding — confirm whether that
  event-based mechanism can cleanly coexist with/be replaced by
  React Query's own invalidation triggered directly from
  `AuthContext`'s login/logout methods, which is likely the cleaner,
  more idiomatic-under-React-Query approach going forward).

ACCEPTANCE CRITERIA / TESTS
Update `CartContext`'s and `OrdersTab.jsx`'s existing tests for the
new React Query-based implementation:
1. Cart data fetches and renders correctly via the migrated query.
2. Adding/removing/updating a cart item correctly invalidates and
   triggers a refetch of the cart query (assert the mocked API is
   called again after a mutation succeeds).
3. Applying a coupon (Epic 9) correctly refreshes the DISPLAYED cart
   total via the SAME invalidation mechanism, without that
   component needing its own manual refresh call anymore (confirm the
   old manual refresh call was actually REMOVED, not just left in
   place redundantly alongside the new automatic mechanism).
4. Logging in correctly triggers a cart refetch reflecting the post-
   login merge (Epic 5 Task 5.1.1.3), verified under the new mechanism.
5. Reordering (Epic 18 Task 18.1.1.5) correctly invalidates BOTH the
   order-related query AND the cart query (since it mutates both).
Re-run the FULL existing frontend test suite after this migration and
confirm no regressions across any component that consumes
`CartContext` (which, by this point in the project, is a great many
components across many epics — Header's cart badge, the checkout page,
Epic 11's wishlist "move to cart" action, etc.).
```

---

### Feature 29.1.3 — Forms

---

#### Task 29.1.3.1 — Introduce `react-hook-form` + `zod` schema validation

```
You are working in frontend/package.json and a new
frontend/src/schemas/ directory. Assume Epics 1–28 are fully merged.

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
No form library exists anywhere. Every form across this entire project
— Checkout (Epic 5/6), Login/OTP (Epic 2), admin coupon/product forms
(Epic 9/17) — uses hand-rolled local `useState` per field plus a
custom `validate()` function returning an errors object, each
independently reimplementing roughly the same
validate-on-submit/show-errors/clear-on-fix logic slightly differently
per form.

TASK
Install `react-hook-form` and `zod`, and establish shared validation-
schema conventions that Tasks 29.1.3.2/29.1.3.3 then apply to the two
highest-value real forms in the project.

REQUIREMENTS
- Add `react-hook-form` and `zod` (plus `@hookform/resolvers`, the
  official bridge package connecting Zod schemas to React Hook Form's
  validation) to frontend/package.json, pinned to current stable
  versions.
- Establish a shared schemas directory
  (frontend/src/schemas/) with REUSABLE validation primitives matching
  this project's REAL, established backend validation rules — don't
  invent NEW validation rules that might drift from what the backend
  actually enforces; mirror the backend's real constraints:
  ```javascript
  // schemas/common.js
  import { z } from 'zod';

  export const iranianPhoneSchema = z
    .string()
    .regex(/^(\+98|0098|0)?9\d{9}$/, 'شماره موبایل معتبر نیست'); // matches Epic 2 Task 2.1.1.2's backend regex exactly

  export const postalCodeSchema = z
    .string()
    .regex(/^\d{10}$/, 'کد پستی باید ۱۰ رقم باشد'); // matches Epic 5 Task 5.2.1.4's backend validator exactly

  export const emailSchema = z.string().email('ایمیل معتبر نیست');
  ```
  Cross-reference EVERY shared schema against its REAL backend
  counterpart (the Iranian phone regex from Epic 2 Task 2.1.1.2, the
  postal code regex from Epic 5 Task 5.2.1.4) to confirm they
  genuinely match — a frontend validation rule that's SLIGHTLY looser
  or stricter than the backend's real rule produces a confusing UX
  (either rejecting genuinely valid input the backend would accept, or
  accepting input the backend will then reject anyway, wasting a round
  trip) — this cross-referencing is a real, necessary part of this
  task, not a formality.
- Do NOT migrate any actual form component in THIS task — establish
  the library/schema conventions only; Tasks 29.1.3.2/29.1.3.3 do the
  real migration work.

ACCEPTANCE CRITERIA / TESTS
- Add unit tests for each shared schema in
  frontend/src/schemas/__tests__/common.test.js confirming each
  correctly accepts valid input and rejects invalid input matching
  the SAME test cases already established in the corresponding
  BACKEND validator's own tests (e.g. Epic 2 Task 2.1.1.2's phone-
  format tests, Epic 5 Task 5.2.1.4's postal-code tests) — mirroring
  those exact test cases on the frontend is the concrete proof the two
  validation layers genuinely agree.
```

---

#### Task 29.1.3.2 — Migrate checkout form

```
You are working in frontend/src/pages/Checkout.jsx. Assume Task
29.1.3.1 is already merged, and Task 29.1.2.3 (React Query cart
migration) has ideally already landed too, since this form reads
cart/coupon state.

CONTEXT
Per this document's header and Epic 6 Task 6.4.1.1/6.4.1.3's grounding
across this series, `Checkout.jsx` has accumulated substantial hand-
rolled form logic across many epics — shipping address fields (Epic
5), saved-address picker (Epic 5), shipping carrier/rate selection
(Epic 7), coupon display (Epic 9) — all currently validated via a
single, growing, hand-maintained `validate()` function. This is the
single highest-value form in the entire project to migrate, given both
its complexity and its direct connection to real revenue (a confusing
or buggy checkout form directly costs completed sales).

TASK
Migrate the full checkout form to `react-hook-form` + `zod`.

REQUIREMENTS
- Build a comprehensive Zod schema for the checkout form, covering
  EVERY field the form currently validates (cross-reference the
  CURRENT `validate()` function's real logic before writing the
  schema, to ensure no validation rule is accidentally dropped in the
  migration):
  ```javascript
  // schemas/checkout.js
  import { z } from 'zod';
  import { iranianPhoneSchema, postalCodeSchema, emailSchema } from './common';

  export const checkoutSchema = z.object({
    firstName: z.string().min(1, 'نام الزامی است'),
    lastName: z.string().min(1, 'نام خانوادگی الزامی است'),
    email: emailSchema,
    phone: iranianPhoneSchema,
    addressId: z.number().nullable(),
    // manual address fields — required ONLY when no addressId is selected,
    // per Epic 5 Task 5.2.1.3's "saved address OR full manual fields" rule
    address: z.string().optional(),
    city: z.string().optional(),
    province: z.string().optional(),
    postalCode: postalCodeSchema.optional(),
    shippingRateId: z.number({ required_error: 'لطفاً یک روش ارسال انتخاب کنید' }),
    notes: z.string().max(500).optional(),
  }).refine(
    (data) => data.addressId || (data.address && data.city && data.province && data.postalCode),
    { message: 'لطفاً یک آدرس ذخیره‌شده انتخاب کنید یا اطلاعات آدرس را کامل وارد کنید', path: ['address'] }
  );
  ```
  The `.refine()` conditional-requirement rule is the important,
  non-trivial piece here — it correctly encodes Epic 5 Task 5.2.1.3's
  actual backend rule (saved address OR complete manual fields, not
  a blanket "every field always required") using Zod's cross-field
  validation capability, rather than a simpler schema that would
  incorrectly reject a valid saved-address checkout for "missing"
  manual address fields that were never supposed to be required in
  that case.
- Wire the form via `react-hook-form`'s `useForm` +
  `zodResolver(checkoutSchema)`, replacing the existing local
  `useState`-per-field + hand-rolled `validate()` entirely:
  ```javascript
  import { useForm } from 'react-hook-form';
  import { zodResolver } from '@hookform/resolvers/zod';
  import { checkoutSchema } from '../schemas/checkout';

  const { register, handleSubmit, watch, setValue, formState: { errors, isSubmitting } } = useForm({
    resolver: zodResolver(checkoutSchema),
  });
  ```
  Preserve ALL existing checkout behavior established across every
  prior epic that touches this form: the saved-address picker (Epic
  5), the shipping-quote fetch triggered on province/city entry (Epic
  7 Task 7.2.1.6), the payment-initiation flow on submit (Epic 6 Task
  6.4.1.1) — migrating the FORM STATE/VALIDATION mechanism must not
  change any of this surrounding business logic; this is purely a
  state-management/validation-library swap, not a checkout-flow
  redesign.
- Ensure error messages render in the SAME visual locations/style
  already established (inline, per-field error text) — the visual
  PRESENTATION of form errors shouldn't change, only the underlying
  mechanism producing them.

ACCEPTANCE CRITERIA / TESTS
Update `frontend/src/__tests__/Checkout.test.jsx` (confirmed to exist,
per Epic 6 Task 6.4.1.1's grounding) for the new implementation:
1. Every validation rule the OLD `validate()` function enforced is
   still correctly enforced under the new schema (a systematic,
   field-by-field comparison against the old logic — the real
   regression-prevention work of this task).
2. The conditional saved-address-vs-manual-fields rule works correctly
   in BOTH directions (selecting a saved address correctly bypasses
   manual-field requirements; NOT selecting one correctly requires
   them).
3. Submitting a valid, complete form correctly proceeds through the
   SAME create-order-then-initiate-payment flow established in Epic 6
   Task 6.4.1.1, unchanged.
4. Iranian phone/postal-code validation produces the SAME accept/
   reject behavior as before the migration (cross-referencing Task
   29.1.3.1's schema tests).
```

---

#### Task 29.1.3.3 — Migrate auth forms (OTP, login)

```
You are working in frontend/src/pages/Login.jsx and the OTP phone-
entry/code-entry components (Epic 2 Task 2.3.2.1/2.3.2.2). Assume Task
29.1.3.2 is already merged.

CONTEXT
Login and OTP entry are the FIRST forms most visitors ever interact
with on this platform — consistent, reliable validation UX here
matters disproportionately for first impressions, and per Epic 2's
grounding, these forms currently use the same hand-rolled local-state
validation pattern as everything else in this project.

TASK
Migrate the email/password login form and the OTP phone-entry/code-
entry forms to `react-hook-form` + `zod`.

REQUIREMENTS
- Build schemas:
  ```javascript
  // schemas/auth.js
  import { z } from 'zod';
  import { iranianPhoneSchema, emailSchema } from './common';

  export const loginSchema = z.object({
    email: emailSchema,
    password: z.string().min(1, 'رمز عبور الزامی است'),
  });

  export const otpPhoneSchema = z.object({
    phoneNumber: iranianPhoneSchema,
  });

  export const otpCodeSchema = z.object({
    code: z.string().length(6, 'کد باید ۶ رقم باشد').regex(/^\d+$/, 'کد باید فقط شامل عدد باشد'),
  });
  ```
- Migrate `Login.jsx`'s email/password form to `useForm` +
  `zodResolver(loginSchema)`, preserving the existing submit logic
  (calling `AuthContext`'s `login()` method, per Epic 2's original
  design) unchanged.
- Migrate the OTP phone-entry component (Epic 2 Task 2.3.2.1) to
  `useForm` + `zodResolver(otpPhoneSchema)`, preserving the existing
  `requestOtp()` call and step-transition logic (to the code-entry
  step) unchanged.
- Migrate the OTP code-entry component (Epic 2 Task 2.3.2.2) to
  `useForm` + `zodResolver(otpCodeSchema)` — note this component has
  its OWN established resend-cooldown-timer logic (per that task's
  `useFakeTimers`-tested countdown) which is UNRELATED to form
  validation and must be preserved exactly as-is, not touched as part
  of this migration; only the CODE-INPUT validation itself moves to
  the new schema-based mechanism, the surrounding timer/resend logic
  stays untouched.
- Confirm this migration correctly preserves Epic 2 Task 2.3.1.2's
  distinct backend ERROR MESSAGES (wrong code / expired code / max-
  attempts-exceeded) — these come back as SERVER-side errors from the
  verify API call, not client-side Zod validation errors (Zod's schema
  only validates the CODE'S FORMAT — 6 digits — before submission; it
  cannot know whether a well-formatted code is actually CORRECT,
  EXPIRED, or has hit its attempt limit, which are all server-side
  facts) — confirm the migrated component correctly displays these
  server-side error responses (via `setError()` from `react-hook-form`,
  applying a server-returned error message to the relevant field)
  alongside/distinctly from client-side Zod format-validation errors,
  preserving the SAME specific, distinct error messaging established
  back in Epic 2 Task 2.3.2.2, not collapsing everything into one
  generic error state.

ACCEPTANCE CRITERIA / TESTS
Update the existing test files for `Login.jsx` and the OTP components
(Epic 2 Task 2.3.2.4's established test suite):
1. Email/password login form validation (format checks) works
   correctly under the new schema.
2. OTP phone-entry format validation works correctly under the new
   schema.
3. OTP code-entry: a well-formatted-but-wrong code correctly shows the
   SERVER'S "incorrect code" message (not a generic client-side error);
   a malformed code (not 6 digits) correctly shows the CLIENT-side
   Zod format error, confirming both error sources still work
   correctly and distinctly after the migration.
4. The resend-cooldown timer (Epic 2 Task 2.3.2.2's `useFakeTimers`-
   tested logic) is UNCHANGED and still passes its existing tests
   exactly, confirming this migration didn't inadvertently touch
   unrelated logic it wasn't supposed to.
Re-run the FULL Epic 2 OTP integration test suite (Task 2.3.1.4/
2.3.2.4) end-to-end after this migration and confirm zero regressions
in the actual login/OTP flow's real functional behavior.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 29.1.1.1 | Convert all route imports to React.lazy() | ☐ |
| 29.1.1.2 | Split admin bundle from storefront bundle | ☐ |
| 29.1.1.3 | Route-level loading skeletons | ☐ |
| 29.1.2.1 | Introduce React Query (TanStack Query) | ☐ |
| 29.1.2.2 | Migrate product listing/PDP fetches | ☐ |
| 29.1.2.3 | Migrate cart/order fetches | ☐ |
| 29.1.3.1 | Introduce react-hook-form + zod | ☐ |
| 29.1.3.2 | Migrate checkout form | ☐ |
| 29.1.3.3 | Migrate auth forms (OTP, login) | ☐ |

Once Epic 29 is fully merged, the next epic to generate prompts for is
**Epic 30 — Documentation**, the final epic in the master backlog,
which the execution-order notes describe as a continuous, ongoing
concern formalized at the end of each major epic — worth reviewing
against everything this 29-epic document series has actually produced
(ADRs from Epic 14, runbooks from Epic 25/26, the master backlog
document itself) before treating it as a from-scratch effort.
