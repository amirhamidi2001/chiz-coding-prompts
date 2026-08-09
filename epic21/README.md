# Epic 21 — Caching (Redis) — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–20 are fully merged.

**Confirmed directly from the repo:** Redis is **already a running dependency** of this project (`redis==7.4.0`, `channels_redis==4.3.0` in `requirements.txt`) — but **only** for Django Channels' chat/notification layer (`CHANNEL_LAYERS` in `backend/core/settings/base.py`, connecting via `(REDIS_HOST, REDIS_PORT)` with **no explicit database index specified**, which means it implicitly connects to **Redis DB 0**, Redis's default). There is **no `CACHES` setting anywhere**, **no `django-redis` package installed**, and **`SESSION_ENGINE` is unset**, meaning Django is using its default database-backed session store. This epic is the first time Redis gets used for anything beyond the WebSocket layer.

**A real, easy-to-get-wrong detail this epic's first task must handle correctly:** since Channels already implicitly occupies Redis DB 0, and if Epic 22's Celery work has landed by this point it very likely ALSO defaults to Redis DB 0 as its broker (Celery's common default, unless explicitly configured otherwise) — **this epic's new cache connection must use a genuinely distinct database index**, or key collisions/unexpected cross-contamination between the WebSocket channel layer, the Celery broker's internal bookkeeping keys, and Django's cache could occur. Redis supports up to 16 logical databases (0–15) by default; this epic should claim its own explicitly, and the task below documents a full allocation scheme rather than picking one number in isolation.

---

## Phase 21.1 — Cache Layer

### Feature 21.1.1 — Django Cache Configuration

---

#### Task 21.1.1.1 — Configure `CACHES` with `django-redis` (separate DB index from Channels)

```
You are working in backend/requirements.txt, core/settings/base.py.
Assume Epics 1–20 are fully merged.

CONTEXT — READ THIS DOCUMENT'S HEADER CAREFULLY BEFORE STARTING
Redis is already running as a dependency for Channels (implicitly on
DB 0, since `CHANNEL_LAYERS`'s config specifies no explicit `db` index
at all). This task adds a SECOND, logically separate use of the same
Redis instance for Django's cache framework — using a DIFFERENT
database index is essential; sharing DB 0 with the channel layer's
internal group/connection bookkeeping keys risks confusing key
collisions or, at minimum, makes it impossible to reason clearly about
which subsystem owns which keys when debugging.

TASK
Add `django-redis` and configure `CACHES`, using an explicitly
different Redis DB index from the Channels layer, and document a full
Redis DB allocation scheme for this project going forward (not just
this one setting in isolation).

REQUIREMENTS
- Add `django-redis` to backend/requirements.txt, pinned to a current
  stable version compatible with the already-installed `redis==7.4.0`
  client library (verify actual compatibility between the two package
  versions before finalizing the pin — `django-redis` wraps the
  underlying `redis-py` client and version mismatches between the two
  can cause real, confusing runtime errors).
- Add explicit `REDIS_DB_CACHE`/`REDIS_DB_CHANNELS` settings (rather
  than hardcoding index numbers inline in multiple places) to
  backend/core/settings/base.py, and DOCUMENT the full allocation
  scheme in a comment block above them:
  ```python
  # ─── Redis DB Allocation ───────────────────────────────────────────
  # This project uses a SINGLE Redis instance for multiple purposes.
  # Each subsystem MUST use a distinct logical DB index to avoid key
  # collisions between unrelated systems:
  #   DB 0 — Django Channels (chat/notifications WebSocket layer)
  #   DB 1 — Celery broker/result backend (see Epic 22, if landed)
  #   DB 2 — Django cache framework (this task)
  #   DB 3 — Django sessions (Task 21.1.1.5, if using a SEPARATE cache
  #          alias from the general-purpose cache above — see that
  #          task for the actual decision on whether sessions share
  #          DB 2 or get their own DB 3)
  # If you are implementing this task BEFORE Epic 22's Celery work has
  # landed, confirm — once Epic 22 IS implemented — that its broker
  # configuration is updated to explicitly target DB 1 rather than
  # accepting whatever Celery's own default happens to be, so this
  # documented scheme stays accurate and collision-free going forward.
  REDIS_DB_CACHE = config("REDIS_DB_CACHE", default=2, cast=int)
  ```
  Update `CHANNEL_LAYERS`'s existing config to ALSO be explicit about
  its DB index (making the currently-implicit DB 0 usage explicit and
  self-documenting, rather than leaving it as an unstated default that
  someone could accidentally break later by adding a `db` param
  elsewhere without realizing Channels was silently relying on the
  implicit default):
  ```python
  CHANNEL_LAYERS = {
      "default": {
          "BACKEND": "channels_redis.core.RedisChannelLayer",
          "CONFIG": {
              "hosts": [
                  {
                      "address": (config("REDIS_HOST", default="127.0.0.1"), int(config("REDIS_PORT", default=6379))),
                      "db": 0,  # explicit, matching the documented allocation scheme above
                  }
              ],
              "capacity": 1500,
              "expiry": 10,
          },
      }
  }
  ```
  (verify `channels_redis`'s actual current supported config shape for
  specifying a DB index on a host entry — the dict-with-`db`-key form
  shown is illustrative; confirm against the installed `channels_redis`
  version's real documentation, since this library's exact
  host-config schema has evolved across versions).
- Add the `CACHES` setting:
  ```python
  CACHES = {
      "default": {
          "BACKEND": "django_redis.cache.RedisCache",
          "LOCATION": f"redis://{config('REDIS_HOST', default='127.0.0.1')}:{config('REDIS_PORT', default=6379)}/{REDIS_DB_CACHE}",
          "OPTIONS": {
              "CLIENT_CLASS": "django_redis.client.DefaultClient",
          },
          "KEY_PREFIX": "chiz",  # namespace cache keys, in case this Redis instance is ever shared with another project/environment
          "TIMEOUT": 300,  # sensible default TTL (5 min) for any cache.set() call that doesn't specify its own explicit timeout
      }
  }
  ```
- Confirm the `REDIS_HOST`/`REDIS_PORT` env vars are consistently used
  (they already are, per the existing `CHANNEL_LAYERS` config) rather
  than introducing a second, parallel set of Redis-connection env vars
  for the cache specifically — one Redis instance, one set of
  host/port env vars, multiple DB indices layered on top.

ACCEPTANCE CRITERIA / TESTS
- `python manage.py check` passes with the new cache backend
  configured.
- Add a test confirming `cache.set("test_key", "test_value")` followed
  by `cache.get("test_key")` round-trips correctly against a REAL
  Redis connection (not mocked — this is exactly the kind of
  infrastructure wiring that's worth verifying against the real
  dependency, not just asserting the settings dict looks correct).
- Add a test confirming the cache and Channels layer are GENUINELY
  using different DB indices — connect directly via `redis-py` to each
  configured DB index, set a distinct marker key in the CACHE'S db,
  and confirm it's NOT visible when querying the CHANNELS db directly
  (a real, concrete proof of isolation, not just a settings-file
  assertion).
```

---

#### Task 21.1.1.2 — Cache category tree endpoint

```
You are working in backend/shop/views.py. Assume Task 21.1.1.1 is
already merged.

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
`CategoryListView` already exists exactly as described by this task's
title — it returns top-level categories with prefetched nested
children (`Category.objects.filter(parent__isnull=True).prefetch_related("children")`).
Category data changes rarely (an admin adding/renaming/reorganizing
categories is an infrequent, deliberate action) but is requested on
EVERY storefront page load that renders navigation/filter UI —
exactly the profile of data that benefits most from caching: high
read frequency, low write frequency.

TASK
Cache `CategoryListView`'s response, with signal-based invalidation on
any `Category` change (not a blind TTL-only approach, which would
serve stale navigation data for however long the TTL runs after a
genuine admin change).

REQUIREMENTS
- Use Django's low-level cache API directly in the view (rather than
  a decorator-based approach like `@cache_page`, which caches the
  RAW HTTP response including headers and doesn't integrate as cleanly
  with the explicit signal-based invalidation this task also requires
  — a manual `cache.get()`/`cache.set()` pattern gives more precise
  control over exactly what's cached and how it's invalidated):
  ```python
  from django.core.cache import cache

  CATEGORY_TREE_CACHE_KEY = "category_tree_v1"
  CATEGORY_TREE_CACHE_TTL = 60 * 60  # 1 hour — long TTL is fine given signal-based invalidation handles real changes promptly


  class CategoryListView(generics.ListAPIView):
      permission_classes = [AllowAny]
      serializer_class = CategorySerializer

      def get_queryset(self):
          return Category.objects.filter(parent__isnull=True).prefetch_related("children")

      def list(self, request, *args, **kwargs):
          cached = cache.get(CATEGORY_TREE_CACHE_KEY)
          if cached is not None:
              return Response(cached)
          response = super().list(request, *args, **kwargs)
          cache.set(CATEGORY_TREE_CACHE_KEY, response.data, timeout=CATEGORY_TREE_CACHE_TTL)
          return response
  ```
  The `_v1` suffix on the cache key is a deliberate, cheap safety
  measure: if the `CategorySerializer`'s OUTPUT SHAPE ever changes in
  a future epic (new fields added/removed), bumping this suffix to
  `_v2` guarantees old, differently-shaped cached data is never
  accidentally served after a serializer change — a manual but simple
  and effective versioning discipline worth establishing as a pattern
  other cached-endpoint tasks in this Feature should also follow.
- Add signal-based invalidation in shop/signals.py (the module already
  established since Epic 4 Task 4.1.2.3, wired via `shop/apps.py`'s
  `ready()`):
  ```python
  from django.db.models.signals import post_save, post_delete
  from django.dispatch import receiver
  from django.core.cache import cache
  from .models import Category

  @receiver([post_save, post_delete], sender=Category)
  def invalidate_category_tree_cache(sender, **kwargs):
      cache.delete("category_tree_v1")
  ```
  Note this fires on BOTH save AND delete (a category being deleted
  also needs to invalidate the cached tree, not just edits) — and
  fires on EVERY save regardless of which specific field changed
  (unlike Epic 8 Task 8.1.1.2's more surgical "only fire when status
  actually changed" pattern) — this coarser approach is fine and
  appropriate here, since Category saves are infrequent enough that
  invalidating on every save (even ones that didn't strictly need to,
  e.g. touching an unrelated field) costs essentially nothing, and the
  simpler implementation is preferable to the added complexity of
  precise field-change detection for such a low-frequency-write model.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_views.py:
1. The FIRST request to `CategoryListView` populates the cache (assert
   `cache.get(CATEGORY_TREE_CACHE_KEY)` is populated after the request).
2. A SECOND request returns the SAME data without re-querying the
   database (verify via `assertNumQueries(0)` for the queryset-fetching
   portion specifically, or by mocking the queryset method and
   confirming it's NOT called on the second request).
3. Creating, updating, or deleting a `Category` correctly invalidates
   the cache (assert `cache.get(...)` returns `None` immediately after
   any of these three operations), and the NEXT request after
   invalidation correctly reflects the change (not stale cached data).
4. Two DIFFERENT test runs (or a test explicitly clearing cache between
   assertions) don't leak cached state between them — ensure test
   setup/teardown correctly clears the cache (check whether this
   project's test configuration already handles Redis cache clearing
   between tests, e.g. via a fixture/conftest.py addition, or add one
   if missing, since a real Redis-backed cache does NOT automatically
   reset between test runs the way Django's in-memory test database
   does).
```

---

#### Task 21.1.1.3 — Cache homepage/product-list responses (short TTL)

```
You are working in backend/shop/views.py. Assume Task 21.1.1.2 is
already merged (establishing the manual cache.get/set + versioned-key
pattern this task follows).

CONTEXT
`ProductListView` (extended across Epics 3/12 with cosmetics filters
and full-text search) is the single highest-traffic read endpoint in
this entire application — every storefront browse/search/filter
interaction hits it. Unlike the category tree (Task 21.1.1.2, low
write frequency), PRODUCT data changes far more often (stock levels
shift with every order per Epic 1/3/4, prices change via Epic 17's
bulk tools, flash sales activate/deactivate per Epic 9) — meaning this
endpoint needs a genuinely SHORT TTL and QUERY-PARAMETER-AWARE cache
keys, not the long-TTL, single-key approach used for categories.

TASK
Add short-TTL caching to `ProductListView`, with cache keys that
correctly vary by the full set of active query parameters (filters,
search term, sort, page).

REQUIREMENTS
- Build a cache key incorporating EVERY query parameter that affects
  the response — critically, this must be done correctly or two
  DIFFERENT filtered views could collide on the same cache key and
  serve each other's results, which would be a real, user-visible
  correctness bug, not just a performance nitpick:
  ```python
  import hashlib
  from django.core.cache import cache

  PRODUCT_LIST_CACHE_TTL = 60  # short — 60 seconds — given how frequently underlying stock/price/sale data changes


  class ProductListView(generics.ListAPIView):
      ...

      def list(self, request, *args, **kwargs):
          cache_key = self._build_cache_key(request)
          cached = cache.get(cache_key)
          if cached is not None:
              return Response(cached)
          response = super().list(request, *args, **kwargs)
          cache.set(cache_key, response.data, timeout=PRODUCT_LIST_CACHE_TTL)
          return response

      def _build_cache_key(self, request):
          # Sort query params for a stable, order-independent key —
          # ?category=1&brand=2 and ?brand=2&category=1 must produce
          # the SAME cache key, since they're semantically identical
          # requests.
          params = sorted(request.query_params.items())
          params_str = "&".join(f"{k}={v}" for k, v in params)
          params_hash = hashlib.md5(params_str.encode()).hexdigest()
          return f"product_list_v1:{params_hash}"
  ```
  The `sorted()` call on query params is a genuinely important
  correctness detail, not a stylistic nicety — without it, two
  requests carrying the exact same filters in a different PARAMETER
  ORDER (which browsers/frontends can easily produce depending on how
  a filter UI constructs its query string) would incorrectly be
  treated as different cache keys, defeating much of the cache's
  effectiveness (a real cache-hit-rate problem, not a correctness bug,
  but still worth getting right from the start).
  Using an `md5` hash of the sorted param string (rather than the raw
  string itself) keeps cache KEYS a bounded, predictable length
  regardless of how many filters are active — a request with a dozen
  active cosmetics filters (per Epic 12) could otherwise produce a very
  long raw cache key string.
- Do NOT cache requests that include a logged-in-user-specific
  component if any exist on this endpoint — check whether
  `ProductListView`'s response ever varies by AUTHENTICATED user
  identity (e.g. does it show personalized pricing, wishlist-state
  indicators, or anything user-specific inline in the list response,
  as opposed to being fetched separately by the frontend?) — if the
  response is genuinely identical for every anonymous/authenticated
  visitor given the SAME query parameters (the more likely case, given
  wishlist/cart state is typically fetched via SEPARATE endpoints per
  established patterns from prior epics), this caching is safe to
  apply universally; if you find any per-user variation in this
  specific response, EXCLUDE authenticated requests from caching
  entirely (or key by user ID too, which would badly fragment cache
  effectiveness) rather than risk serving one user's personalized data
  to another — verify this carefully before proceeding, since caching
  personalized data under a shared key would be a real, serious data-
  leakage bug, not just a stale-data inconvenience.
- Apply the same pattern to the HOMEPAGE endpoint if it's a distinct
  view from the general product list (check for a dedicated homepage-
  data endpoint — e.g. "featured products," "new arrivals," flash sale
  data per Epic 9 Task 9.2.1.3 — these are natural additional
  candidates for the same short-TTL caching treatment, since homepage
  traffic is typically even higher than any single filtered listing
  view).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_views.py:
1. Two requests with the SAME query parameters (including ones with
   parameters in a DIFFERENT ORDER) hit the SAME cache key and only
   query the database once.
2. Two requests with GENUINELY DIFFERENT query parameters (different
   filters/search terms) produce DIFFERENT cache entries and correctly
   return their own distinct, correct results — NOT each other's
   cached data (the critical correctness test for this whole task).
3. The cache TTL is genuinely short (mock/advance time past the TTL
   and confirm a subsequent request re-queries the database rather
   than serving stale data indefinitely).
4. If the endpoint has ANY authenticated-user-specific response
   variation, confirm caching is correctly scoped/excluded per
   whichever decision you made above — construct a concrete test
   proving no cross-user data leakage is possible.
```

---

#### Task 21.1.1.4 — Cache invalidation on product/variant save

```
You are working in backend/shop/signals.py. Assume Task 21.1.1.3 is
already merged.

CONTEXT
Task 21.1.1.3's product-list caching uses a short (60s) TTL as its
PRIMARY staleness bound — acceptable for most changes, but a genuinely
bad experience for time-sensitive ones: a customer who just bought the
LAST unit of a product (via Epic 1/3's stock-decrement logic) could
still see it listed as "in stock" on a cached listing page for up to
60 more seconds; an admin activating a flash sale (Epic 9) wants the
discounted price to appear on listings IMMEDIATELY, not up to a minute
later. This task adds signal-based invalidation for genuinely
significant changes, on top of (not replacing) the short TTL safety
net.

TASK
Add signal-based invalidation firing on meaningful `Product`/
`ProductVariant` changes, clearing the relevant cached product-list
entries.

REQUIREMENTS
- Unlike Task 21.1.1.2's category cache (a SINGLE well-known cache
  key, trivial to invalidate directly), Task 21.1.1.3's product-list
  cache uses MANY different keys (one per unique filter-parameter
  combination, per that task's hashed-key design) — there is no single
  key to delete, and Django's low-level cache API has no built-in
  "delete all keys matching a pattern" operation in a backend-agnostic
  way. Given this project now uses `django-redis` specifically (Task
  21.1.1.1), leverage its Redis-specific pattern-deletion capability
  rather than trying to track every individual cache key that might
  need invalidating:
  ```python
  from django_redis import get_redis_connection
  from django.db.models.signals import post_save, post_delete
  from django.dispatch import receiver
  from .models import Product, ProductVariant

  def invalidate_product_list_cache():
      """
      Clear ALL cached product-list responses (every filter-parameter
      combination) — a blunt but simple and correct approach, given
      there's no cheap way to know exactly which cached filter
      combinations a given product change might affect.
      """
      redis_conn = get_redis_connection("default")
      keys = redis_conn.keys("*product_list_v1:*")  # matches the KEY_PREFIX-prefixed pattern
      if keys:
          redis_conn.delete(*keys)

  @receiver(post_save, sender=Product)
  def invalidate_cache_on_product_save(sender, **kwargs):
      invalidate_product_list_cache()

  @receiver([post_save, post_delete], sender=ProductVariant)
  def invalidate_cache_on_variant_save(sender, **kwargs):
      invalidate_product_list_cache()
  ```
  Note the KEY_PREFIX (`"chiz"`, established in Task 21.1.1.1's
  `CACHES` config) is automatically prepended by `django-redis` to
  every cache key — confirm the actual, real prefixed key format by
  inspecting Redis directly (`redis-cli KEYS "*"` against the cache DB)
  rather than assuming the exact pattern string, since django-redis's
  precise prefixing format can vary slightly by version; get this
  pattern genuinely correct, since a wrong pattern would silently fail
  to invalidate anything, which is worse than no caching at all (stale
  data with no way to notice why it's stale).
  This "clear everything" approach is DELIBERATELY blunt rather than
  surgical (unlike, say, tracking which specific cache keys correspond
  to which category/filter combination and only clearing the relevant
  subset) — given product changes are frequent enough that fine-grained
  tracking would add real complexity for uncertain benefit (the cache
  will simply repopulate on the next request to each filter
  combination, and the 60-second TTL means uncached-but-frequently-hit
  combinations were never far from refreshing anyway) — document this
  as a deliberate simplicity-over-precision tradeoff, not an
  unconsidered shortcut.
- Be mindful of INVALIDATION FREQUENCY vs. CACHE EFFECTIVENESS: if
  `ProductVariant.stock` changes on EVERY single order (per Epic 1/3's
  decrement-on-purchase logic), and this signal fires a full cache-wipe
  on every such change, the product-list cache could end up being
  invalidated so frequently under real order volume that it rarely
  actually serves a cache HIT at all — defeating much of this
  Feature's purpose. Consider whether STOCK changes specifically
  warrant a full-cache invalidation, or whether the existing 60-second
  TTL is an ACCEPTABLE staleness window for stock-level display
  specifically (a customer seeing "in stock" for up to 60 seconds after
  the actual last unit sold is a minor, common e-commerce UX
  imperfection — most real platforms accept exactly this tradeoff
  rather than paying the cache-effectiveness cost of instant
  invalidation on every single stock decrement). RECOMMEND: invalidate
  on `Product` field changes (name, description, category, price-
  adjacent fields, active/inactive status) and `ProductVariant` PRICE
  changes specifically, but do NOT invalidate on pure STOCK-quantity
  changes alone — implement this distinction by checking WHICH fields
  actually changed (the same `pre_save`-capture-then-`post_save`-
  compare pattern already established in Epic 8 Task 8.1.1.2) rather
  than firing unconditionally on every save:
  ```python
  @receiver(pre_save, sender=ProductVariant)
  def _capture_previous_variant_state(sender, instance, **kwargs):
      if instance.pk:
          try:
              instance._previous = ProductVariant.objects.only("stock", "price").get(pk=instance.pk)
          except ProductVariant.DoesNotExist:
              instance._previous = None
      else:
          instance._previous = None

  @receiver(post_save, sender=ProductVariant)
  def invalidate_cache_on_variant_save(sender, instance, created, **kwargs):
      previous = getattr(instance, "_previous", None)
      price_changed = created or previous is None or previous.price != instance.price
      if price_changed:
          invalidate_product_list_cache()
      # deliberately NOT invalidating on stock-only changes — see task
      # rationale; the existing 60s TTL is the accepted staleness bound
      # for stock-level display specifically.
  ```

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_signals.py (or wherever this
project's signal tests live, per precedent from Epic 4/8/11's
equivalent signal-testing conventions):
1. Changing a `ProductVariant.price` correctly clears ALL cached
   product-list entries (populate several distinct cached filter
   combinations first, then trigger a price change, then confirm every
   one of them is gone).
2. Changing ONLY `ProductVariant.stock` (price unchanged) does NOT
   clear the cache — the deliberate, documented exception — confirm
   previously-cached entries remain present after a stock-only change.
3. Changing a `Product` field (e.g. `name`, `description`) correctly
   invalidates the cache.
4. Deleting a `ProductVariant` correctly invalidates the cache.
5. Creating a brand-new `ProductVariant` (which should newly appear in
   listings) correctly invalidates the cache (the `created` branch of
   the price-changed check above).
```

---

#### Task 21.1.1.5 — Session backend moved to Redis

```
You are working in backend/core/settings/base.py. Assume Task 21.1.1.1
is already merged.

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
`SESSION_ENGINE` is currently UNSET in every settings file, meaning
Django uses its DEFAULT database-backed session store
(`django.contrib.sessions.backends.db`) — every session read/write
(including, notably, Epic 5's guest cart resolution and Epic 2's
anonymous-visitor session handling, both of which lean on
`request.session` heavily) currently hits Postgres. Moving sessions to
Redis reduces database load for what is a very high-frequency,
low-durability-requirement kind of data (sessions are inherently
ephemeral and don't need the same durability guarantees as, say, order
records).

TASK
Move Django's session backend to Redis, using a genuinely separate
cache ALIAS (not necessarily a separate DB index, though that's also a
valid choice — see the requirement below for the actual decision) from
the general-purpose cache configured in Task 21.1.1.1.

REQUIREMENTS
- Decide: should sessions share the SAME Redis cache connection/DB
  index as the general-purpose cache (Task 21.1.1.1's `CACHES["default"]`,
  on DB 2), or get a fully separate DB index (DB 3, per the allocation
  scheme documented in Task 21.1.1.1's header comment)? Two real
  considerations: (a) sessions and general product/category caching
  have very DIFFERENT invalidation/eviction characteristics — a
  session needs to persist reliably for its configured lifetime (don't
  want a cache-memory-pressure eviction policy accidentally logging
  users out early), while general-purpose cache entries (Tasks 21.1.1.2/
  21.1.1.3) are explicitly designed to be disposable/regenerable at
  any time; mixing the two in one Redis DB with one eviction policy
  risks the general cache's disposable entries crowding out session
  data under memory pressure (or vice versa, less critically). (b) A
  separate DB index gives cleaner operational visibility (you can
  directly inspect `redis-cli -n 3 KEYS "*"` to see ONLY session data,
  useful for debugging/monitoring). RECOMMEND a separate DB index (DB
  3, per the documented allocation scheme) and a SEPARATE named cache
  alias specifically for this purpose:
  ```python
  REDIS_DB_SESSIONS = config("REDIS_DB_SESSIONS", default=3, cast=int)

  CACHES = {
      "default": {
          "BACKEND": "django_redis.cache.RedisCache",
          "LOCATION": f"redis://{config('REDIS_HOST', default='127.0.0.1')}:{config('REDIS_PORT', default=6379)}/{REDIS_DB_CACHE}",
          "OPTIONS": {"CLIENT_CLASS": "django_redis.client.DefaultClient"},
          "KEY_PREFIX": "chiz",
          "TIMEOUT": 300,
      },
      "sessions": {
          "BACKEND": "django_redis.cache.RedisCache",
          "LOCATION": f"redis://{config('REDIS_HOST', default='127.0.0.1')}:{config('REDIS_PORT', default=6379)}/{REDIS_DB_SESSIONS}",
          "OPTIONS": {"CLIENT_CLASS": "django_redis.client.DefaultClient"},
          "KEY_PREFIX": "chiz_session",
          # No blanket TIMEOUT override here — session expiry is
          # governed by SESSION_COOKIE_AGE below, not this cache
          # backend's generic default timeout.
      },
  }

  SESSION_ENGINE = "django.contrib.sessions.backends.cache"
  SESSION_CACHE_ALIAS = "sessions"
  SESSION_COOKIE_AGE = config("SESSION_COOKIE_AGE", default=1209600, cast=int)  # 2 weeks, Django's own default — confirm this project doesn't already override it elsewhere before adding a redundant/conflicting setting
  ```
  Check whether `SESSION_COOKIE_AGE` (or other session-related
  settings) is already configured anywhere in this project's settings
  files before adding it — avoid introducing a duplicate/conflicting
  override.
- Consider `django.contrib.sessions.backends.cache` (pure cache-backed,
  FASTEST, but sessions are LOST if Redis restarts/evicts them under
  memory pressure) vs.
  `django.contrib.sessions.backends.cached_db` (cache-first with a
  database fallback/backup — slightly slower but more durable, since a
  Redis restart wouldn't silently log out every active session) — this
  is a real reliability-vs-performance tradeoff worth a deliberate
  choice rather than defaulting to whichever is simpler to configure.
  RECOMMEND `cached_db` specifically for THIS project, given Epic 5's
  guest-cart feature makes session persistence genuinely
  business-important (a lost anonymous session means a lost guest
  cart, not just a mildly annoying re-login) — the added database
  fallback cost is a reasonable price for that durability, especially
  since sessions were ALREADY on the database exclusively before this
  task, so `cached_db` is a pure improvement (adds Redis as a fast
  read-through layer) rather than the pure-cache option's regression
  risk in cases of Redis instability. Adjust the `SESSION_ENGINE`
  setting above to `"django.contrib.sessions.backends.cached_db"`
  accordingly, and confirm this still requires the existing
  `django.contrib.sessions` app + database table (it does — `cached_db`
  uses BOTH the cache and the DB table, unlike the pure `cache` backend
  which uses neither the DB table nor needs migrations for sessions
  specifically) — no session-table migration changes are needed either
  way, since the DB-backed sessions app/table already exists and
  continues to be used as the fallback layer.

ACCEPTANCE CRITERIA / TESTS
- `python manage.py check` passes with the new session configuration.
- Add a test confirming a session created via a normal request round-
  trips correctly — set a value in `request.session`, save it, and
  confirm a SUBSEQUENT request with the same session cookie correctly
  retrieves it (this exercises the real session read/write path against
  the new backend, not just settings-file correctness).
- Add a test specifically confirming Epic 5's guest-cart flow (session-
  based anonymous cart persistence) STILL WORKS CORRECTLY after this
  session-backend change — re-run Epic 5 Task 5.1.1.2's existing guest-
  cart persistence tests and confirm they pass unmodified, since this
  is exactly the feature most directly dependent on session reliability
  and the one most worth explicitly re-verifying after a session-
  backend infrastructure change.
- If you implemented `cached_db`, add a test simulating a cache MISS
  (e.g. directly deleting the session's Redis key while leaving the
  database row intact) and confirm the session is still correctly
  retrieved via the database fallback — the actual proof the
  durability benefit this backend choice is meant to provide really
  works, not just that it's configured.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 21.1.1.1 | Configure CACHES with django-redis (separate DB index) | ☐ |
| 21.1.1.2 | Cache category tree endpoint | ☐ |
| 21.1.1.3 | Cache homepage/product-list responses (short TTL) | ☐ |
| 21.1.1.4 | Cache invalidation on product/variant save | ☐ |
| 21.1.1.5 | Session backend moved to Redis | ☐ |

Once Epic 21 is fully merged, the next epic to generate prompts for is
**Epic 22 — Celery & Async Tasks**, which — per this document's own
header note — several PRIOR epics' prompts have already been written
assuming exists (Epic 3's expiry sweep, Epic 4's stock/price alerts,
Epic 6's payment reconciliation, Epic 16's notification retries all
explicitly flagged Celery as a soft dependency). If Epic 22 hasn't
actually been built yet by the time you reach it, that epic's Task
22.1.1.2/22.1.1.3 (the `core/celery.py` bootstrap and
docker-compose worker/beat services) is where all of those deferred
assumptions finally get a real foundation — and per this Epic 21's
Task 21.1.1.1, its broker configuration should explicitly target Redis
DB 1, matching the allocation scheme documented there.
