# Epic 12 — Search & Filtering — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–11 are fully merged, including Epic 3 Task 3.2.1.15's `ProductFilter` extensions (skin_type, hair_type, min/max SPF, is_vegan/is_cruelty_free/is_organic) and Epic 14's localization work if it has landed by this point (this epic assumes Persian is at least partially in play, since Task 12.1.1.2 specifically deals with Persian text normalization — if Epic 14 hasn't landed yet in your actual build order, this epic's Persian-specific tasks are still buildable in isolation, they just won't have much real Persian content to test against yet; use representative Persian test strings regardless).

**Confirmed directly from the repo:** `backend/core/settings/base.py` configures `"ENGINE": "django.db.backends.postgresql"` unconditionally (no SQLite fallback anywhere in settings) — this epic's Postgres-specific full-text search work (Task 12.1.1.1) is safe to build without needing a database-engine compatibility shim; this project always runs on Postgres. Also confirmed: `ProductListView` (`backend/shop/views.py`) currently uses DRF's plain `filters.SearchFilter` with `search_fields = ["name", "short_description", "description", "brand__name", "category__name"]` — a basic `icontains`/`istartswith`-style `LIKE` query with no relevance ranking, no weighting between fields, and no language-aware stemming/normalization. This is exactly the gap the original architecture review flagged and this epic's Task 12.1.1.1 replaces.

---

## Phase 12.1 — Search Quality

### Feature 12.1.1 — Persian-Aware Search

---

#### Task 12.1.1.1 — Evaluate & integrate Postgres full-text search (`SearchVector`)

```
You are working in backend/shop/models.py, views.py, migrations.
Assume Epics 1–11 are fully merged and confirm the project's DATABASES
engine is genuinely `django.db.backends.postgresql` (per this
document's header context) before starting — this task's entire
approach depends on that being true.

CONTEXT
`ProductListView.search_fields = ["name", "short_description",
"description", "brand__name", "category__name"]` with DRF's plain
`filters.SearchFilter` translates to a series of `icontains` `LIKE`
queries ORed together — every matching field is weighted identically,
there's no relevance ranking (results come back in whatever order the
rest of the queryset dictates, typically `-created_at`, not "best
match first"), and performance degrades linearly with catalog size
since `LIKE '%term%'` can't use a standard B-tree index. Postgres has
built-in full-text search (`SearchVector`/`SearchQuery`/`SearchRank`)
that solves all three problems and is already available given this
project's confirmed Postgres-only database engine.

TASK
Replace `ProductListView`'s plain `SearchFilter` with Postgres
full-text search, weighted so `name` matches rank higher than
`description` matches, with a persisted, indexed search vector column
for performance at scale.

REQUIREMENTS
- Add a stored, indexed search vector column to `Product`:
  ```python
  from django.contrib.postgres.search import SearchVectorField
  from django.contrib.postgres.indexes import GinIndex

  class Product(models.Model):
      ...
      search_vector = SearchVectorField(null=True, blank=True, editable=False)

      class Meta:
          ...
          indexes = [
              ...,  # keep any existing indexes
              GinIndex(fields=["search_vector"], name="product_search_vector_idx"),
          ]
  ```
  Generate the migration.
- Populate `search_vector` with WEIGHTED field contributions —
  `name`/`brand__name` should rank higher than `short_description`/
  `description`, since a name match is a much stronger relevance signal
  than a passing mention deep in a product description. Use a
  `pre_save` signal or an overridden `Product.save()` to keep this
  column current on every save (a full-text search vector column must
  be actively maintained — it doesn't magically stay in sync with the
  underlying text fields on its own):
  ```python
  from django.contrib.postgres.search import SearchVector

  def save(self, *args, **kwargs):
      ...  # existing slug-generation logic stays exactly as-is
      super().save(*args, **kwargs)
      Product.objects.filter(pk=self.pk).update(
          search_vector=(
              SearchVector("name", weight="A", config="simple")
              + SearchVector("brand__name", weight="A", config="simple")
              + SearchVector("category__name", weight="B", config="simple")
              + SearchVector("short_description", weight="B", config="simple")
              + SearchVector("description", weight="C", config="simple")
          )
      )
  ```
  Note the explicit `config="simple"` on every `SearchVector` — this is
  deliberate: Postgres's LANGUAGE-AWARE search configs (like
  `"english"`) apply stemming/stopword-removal rules that are wrong for
  Persian content and would actively degrade search quality for
  non-English product names/descriptions (which this platform will have
  plenty of, given its target market). The `"simple"` config does
  tokenization without any language-specific stemming, which is the
  correct, safe default for a genuinely multi-lingual/Persian-primary
  catalog — a dedicated Persian search configuration is a possible
  FUTURE enhancement (Postgres doesn't ship a built-in Persian text
  search config the way it does for English/French/etc., so this would
  require either a custom Postgres text search configuration or an
  external search engine like Elasticsearch/OpenSearch with proper
  Persian analyzers — explicitly out of scope for this task, which
  sticks to `"simple"` as the correct, honest baseline rather than
  pretending `"english"` config is acceptable for this platform's real
  content).
  IMPORTANT: `SearchVector("brand__name", ...)` and
  `SearchVector("category__name", ...)` reference RELATED fields —
  confirm this works correctly with Django's `SearchVector` (it does
  support related-field lookups via the same `__` syntax as regular
  querysets, but double-check behavior when `brand`/`category` is
  `None` — a product with no brand assigned shouldn't crash the save,
  it should just contribute nothing from that field; verify this
  degrades gracefully with a test rather than assuming).
- Add a data migration to backfill `search_vector` for all EXISTING
  Product rows (the `save()` override above only maintains it going
  forward for future saves — pre-existing rows have `NULL` until
  something re-saves them):
  ```python
  # in the new migration's RunPython:
  def backfill_search_vectors(apps, schema_editor):
      from django.contrib.postgres.search import SearchVector
      Product = apps.get_model("shop", "Product")
      Product.objects.update(
          search_vector=(
              SearchVector("name", weight="A", config="simple")
              + SearchVector("brand__name", weight="A", config="simple")
              + SearchVector("category__name", weight="B", config="simple")
              + SearchVector("short_description", weight="B", config="simple")
              + SearchVector("description", weight="C", config="simple")
          )
      )
  ```
  (note: `apps.get_model()`-obtained historical models generally CAN'T
  follow relations the same way live models can in every Django
  version's migration framework for `SearchVector` specifically — test
  this migration actually runs correctly against real data before
  considering it done; if it turns out `SearchVector` on a historical
  model doesn't work reliably inside a data migration, fall back to a
  regular Django management command run once post-deploy instead of a
  migration-embedded `RunPython`, and document that operational step
  clearly in your task summary).
- Replace `ProductListView`'s search backend: remove `filters.SearchFilter`
  and `search_fields` from `filter_backends`/attributes, and implement
  custom search handling in `get_queryset()`:
  ```python
  from django.contrib.postgres.search import SearchQuery, SearchRank

  def get_queryset(self):
      queryset = (
          Product.objects.select_related("category", "brand")
          .prefetch_related("colors__color")
      )
      search_term = self.request.query_params.get("search")
      if search_term:
          query = SearchQuery(search_term, config="simple")
          queryset = queryset.filter(search_vector=query).annotate(
              rank=SearchRank("search_vector", query)
          ).order_by("-rank")
      return queryset
  ```
  Note ordering by `-rank` OVERRIDES the view's existing default
  `ordering = ["-created_at"]` when a search term is present — this is
  correct and intentional (relevance should win over recency when
  actively searching), but confirm the existing `OrderingFilter`
  backend (still present for non-search browsing/sorting) doesn't
  fight with this — if a client explicitly requests `?ordering=price`
  WHILE ALSO searching, which should win? A reasonable, defensible
  choice: an explicit `?ordering=` param should still override
  relevance ranking (the customer explicitly asked to sort by price,
  so honor that), meaning `-rank` should only be the DEFAULT ordering
  when searching, not an unconditional override — check DRF's
  `OrderingFilter` behavior here (it typically only applies ordering if
  `?ordering=` is actually present in the request) and confirm your
  implementation lets an explicit ordering param take precedence over
  the rank-based default, adjusting `get_queryset()`'s logic if needed
  so it doesn't unconditionally clobber that.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_views.py (these tests REQUIRE a
real Postgres connection, not SQLite — confirm the test settings/CI
already point at Postgres per this project's established convention
from Epic 1's grounding notes):
1. Searching for a term that appears in a product's `name` ranks that
   product ABOVE one where the term only appears in `description` (both
   products otherwise having the same `created_at`) — construct this
   scenario explicitly and assert the ordering.
2. A search term matching nothing returns an empty result set (not an
   error).
3. A search term with partial/prefix matching still finds relevant
   results (Postgres FTS does stemmed/lexeme matching, not naive
   substring matching — confirm behavior with a realistic query and
   don't assume it behaves identically to the old `icontains` approach
   for every case; document any meaningfully different behavior you
   discover).
4. A newly-created Product immediately has a correctly-populated
   `search_vector` (proving the `save()` override works, not just the
   backfill migration).
5. Combining `?search=foo&ordering=price` results in price-based
   ordering, not rank-based ordering (per the precedence decision
   above).
6. The backfill migration correctly populates `search_vector` for
   pre-existing rows when run against representative seed data.
```

---

#### Task 12.1.1.2 — Persian text normalization for search (ك/ک, ي/ی, zero-width chars)

```
You are working in backend/shop/views.py (or a new
backend/shop/text_utils.py). Assume Task 12.1.1.1 is already merged.

CONTEXT
Persian and Arabic script share visually near-identical characters that
are actually DIFFERENT Unicode code points — most commonly, Arabic ك
(U+0643) vs. Persian ک (U+06A9), and Arabic ي (U+064A) vs. Persian ی
(U+06CC). Because many keyboards, input methods, and copy-pasted
content mix these inconsistently, the SAME intended word can be typed
two different ways depending on which specific character variant the
user's input method happened to produce — a search for a product name
typed with the Arabic-variant ك would silently fail to match a product
name stored with the Persian-variant ک, even though a human reading
both would consider them identical. Persian text also commonly contains
zero-width non-joiner characters (ZWNJ, U+200C, used for correct
compound-word rendering) that can appear inconsistently between what's
stored and what's typed, causing similar silent match failures.

TASK
Add a normalization function applied consistently to BOTH stored
product text (at save time, feeding into `search_vector`) AND incoming
search queries (before constructing the `SearchQuery`), so character
variant/zero-width inconsistencies never cause a legitimate match to be
missed.

REQUIREMENTS
- Create `backend/shop/text_utils.py`:
  ```python
  import re

  # Arabic-script variant -> canonical Persian-script character
  _CHAR_NORMALIZATION_MAP = {
      "\u0643": "\u06A9",  # Arabic KAF -> Persian KEHEH (ك -> ک)
      "\u064A": "\u06CC",  # Arabic YEH -> Persian FARSI YEH (ي -> ی)
      "\u0649": "\u06CC",  # Arabic ALEF MAKSURA -> Persian FARSI YEH (ى -> ی)
      "\u06C0": "\u0647",  # HEH WITH YEH ABOVE -> plain HEH, if encountered
  }

  _ZERO_WIDTH_CHARS = re.compile("[\u200B\u200C\u200D\uFEFF]")  # ZWSP, ZWNJ, ZWJ, BOM


  def normalize_persian_text(text: str) -> str:
      """
      Normalize Persian/Arabic script variants and strip zero-width
      characters so equivalent text matches consistently regardless of
      input-method quirks. Safe to apply to non-Persian text too (it's
      a no-op for text containing none of the targeted characters).
      """
      if not text:
          return text
      for arabic_char, persian_char in _CHAR_NORMALIZATION_MAP.items():
          text = text.replace(arabic_char, persian_char)
      text = _ZERO_WIDTH_CHARS.sub("", text)
      return text
  ```
  VERIFY the exact Unicode code points in `_CHAR_NORMALIZATION_MAP`
  against an authoritative reference before relying on them — Persian/
  Arabic Unicode normalization is a well-documented but detail-sensitive
  area, and getting even one code point wrong silently defeats the
  entire purpose of this task; if you have any way to verify these
  specific mappings against current, authoritative Unicode
  documentation, do so rather than trusting this prompt's code points
  as certainly correct.
- Apply normalization on the WRITE side: in `Product.save()`'s
  `search_vector`-population logic (Task 12.1.1.1), normalize each
  source text field BEFORE it feeds into `SearchVector` — this requires
  restructuring slightly, since `SearchVector` operates on DATABASE
  FIELD NAMES (doing the text processing in SQL, not Python), not
  Python string values directly. Two viable approaches: (a) normalize
  and store the normalized text into DEDICATED shadow columns (e.g.
  `name_normalized`, `description_normalized`) that `SearchVector`
  reads from instead of the original display fields, keeping the
  original `name`/`description` untouched for actual display purposes,
  or (b) normalize `name`/`description` etc. THEMSELVES at save time
  (overwriting the stored value with its normalized form), accepting
  that the DISPLAYED text also becomes normalized (which is likely
  fine/desirable anyway — a customer doesn't care whether a product
  name displays with ARABIC or PERSIAN kaf, and normalizing display
  text too actually improves consistency across the whole catalog, not
  just search). Prefer approach (b) — normalize the actual `name`/
  `short_description`/`description` fields themselves at save time
  (before the existing slug-generation and search-vector logic runs) —
  it's simpler, avoids duplicate shadow columns, and there's no real
  downside to a cosmetics platform's product text being
  Persian-character-canonicalized consistently everywhere.
  Add this normalization step at the TOP of `Product.save()`, before
  slug generation (Epic 3-era logic) and before the search-vector
  update (Task 12.1.1.1):
  ```python
  def save(self, *args, **kwargs):
      self.name = normalize_persian_text(self.name)
      self.short_description = normalize_persian_text(self.short_description)
      self.description = normalize_persian_text(self.description)
      ...  # existing slug generation, then existing search_vector update
  ```
  Import `normalize_persian_text` from `.text_utils` at the top of
  models.py.
- Apply normalization on the READ side: in `ProductListView.get_queryset()`
  (Task 12.1.1.1's search handling), normalize the incoming
  `search_term` the SAME way before constructing `SearchQuery`:
  ```python
  from .text_utils import normalize_persian_text

  search_term = self.request.query_params.get("search")
  if search_term:
      search_term = normalize_persian_text(search_term)
      query = SearchQuery(search_term, config="simple")
      ...
  ```
- Add a data migration re-saving every existing `Product` (triggering
  the now-normalizing `save()`) so historical catalog data gets
  normalized too, not just future saves — reuse the same backfill
  approach/caveats discussed in Task 12.1.1.1's data migration (if a
  bulk `.update()`-based approach can't easily invoke the full
  `save()` override's normalization logic, iterate and call
  `instance.save()` per row instead, accepting the slower migration
  performance for what should be a one-time operation; note the
  performance tradeoff in a comment if the catalog is large enough for
  this to matter).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_text_utils.py:
1. `normalize_persian_text()` converts Arabic ك to Persian ک, Arabic ي
   to Persian ی, and strips ZWNJ/ZWSP/ZWJ/BOM characters — test each
   mapping individually with real Unicode characters (not just
   describing them in a comment — include the actual characters in
   your test file, verified correct).
2. `normalize_persian_text()` is a safe no-op on English/Latin-script
   text and on text containing none of the targeted characters.
3. `normalize_persian_text("")`/`normalize_persian_text(None)` don't
   raise.
Add integration tests to test_views.py:
4. A product saved with the Persian-canonical character in its name is
   findable via a search query typed with the Arabic-variant character
   (and vice versa) — construct this exact scenario with real
   differing-but-equivalent Unicode strings and confirm the search
   returns the match.
5. A product name containing a ZWNJ character is still findable by a
   search query that omits the ZWNJ (a very common real-world
   mismatch), and vice versa.
```

---

#### Task 12.1.1.3 — Search-as-you-type suggestion endpoint

```
You are working in backend/shop/views.py, urls.py and
frontend/src/services/api.js + search UI component. Assume Task
12.1.1.2 is already merged.

CONTEXT
There's no autocomplete/suggestion endpoint — the only way to search
today is a full form submission hitting `ProductListView`'s paginated
list endpoint, which is too heavy (full serializer payload, pagination
metadata) for a lightweight "show a few suggestions as the user types"
UX.

TASK
Add a lightweight, fast autocomplete endpoint returning a small number
of top matches (name + thumbnail + price only, not the full product
payload), and wire it to a search-box dropdown on the frontend.

REQUIREMENTS
- Add `ProductSuggestSerializer` in backend/shop/serializers.py — a
  DELIBERATELY minimal serializer distinct from
  `ProductListSerializer` (which, per Epic 3 Task 3.2.1.14, already
  carries 6+ cosmetics-attribute fields for badge rendering — far more
  than an autocomplete dropdown needs):
  ```python
  class ProductSuggestSerializer(serializers.ModelSerializer):
      thumbnail_url = serializers.SerializerMethodField()

      class Meta:
          model = Product
          fields = ["id", "name", "slug", "price", "thumbnail_url"]

      def get_thumbnail_url(self, obj):
          request = self.context.get("request")
          if obj.thumbnail and request:
              return request.build_absolute_uri(obj.thumbnail.url)
          return None
  ```
  (check whether an equivalent thumbnail-URL-resolution pattern already
  exists elsewhere in shop/serializers.py, e.g. on
  `ProductListSerializer` — if so, match its EXACT existing
  implementation rather than writing a subtly different one).
- Add `ProductSuggestView`:
  ```python
  class ProductSuggestView(generics.ListAPIView):
      permission_classes = [AllowAny]
      serializer_class = ProductSuggestSerializer
      pagination_class = None  # no pagination metadata needed for a small suggestion list

      def get_queryset(self):
          term = self.request.query_params.get("q", "").strip()
          if len(term) < 2:  # avoid expensive queries on a single keystroke
              return Product.objects.none()
          term = normalize_persian_text(term)
          query = SearchQuery(term, config="simple")
          return (
              Product.objects.filter(search_vector=query)
              .annotate(rank=SearchRank("search_vector", query))
              .order_by("-rank")[:8]  # small, fixed limit — this is a dropdown, not a results page
          )
  ```
  Reuse the EXACT same `SearchQuery`/`SearchRank`/`normalize_persian_text`
  machinery already built in Tasks 12.1.1.1/12.1.1.2 — do not build a
  second, separate search implementation for this endpoint.
  Register the URL: `path("products/suggest/", views.ProductSuggestView.as_view(), name="product-suggest"),`
  in shop/urls.py.
- Frontend: add `getProductSuggestions: (q) => api.get('/products/suggest/', { params: { q } })`
  to api.js, and wire it into the existing header/navbar search input
  (find it — likely in a `Header.jsx`/`Navbar.jsx` component): debounce
  keystrokes (e.g. 300ms, using a standard debounce utility or a
  `useEffect` + `setTimeout`/cleanup pattern) before firing the request,
  show a dropdown of up to 8 results (thumbnail, name, price) below the
  search box, each linking to that product's detail page, and a
  "See all results for '...'" link at the bottom of the dropdown
  navigating to the full `ProductListView`-backed search results page
  with the typed term.
  Handle the empty-results case (no suggestions found) with a simple
  "No results" state rather than an empty dropdown shell, and close the
  dropdown on click-outside/escape-key (standard dropdown UX — check
  whether a reusable dropdown/popover pattern already exists elsewhere
  in this codebase and reuse it rather than hand-rolling a new one).

ACCEPTANCE CRITERIA / TESTS
Add backend tests:
1. A query of 1 character returns an empty queryset without hitting
   the database with a real search (verify via `assertNumQueries` or
   similar, or just confirm empty results — the short-circuit is
   primarily a performance/UX guard, but confirm it actually short-
   circuits).
2. A query of 2+ characters matching products returns UP TO 8 results,
   ranked by relevance, with the minimal field set only (confirm
   heavier fields like `description`/`ingredients` are NOT present in
   the response).
3. Persian character-variant normalization (Task 12.1.1.2) works
   identically here as in the main search endpoint.
Add frontend component tests for the search dropdown: debounces input
before firing requests (use fake timers), renders suggestion results
correctly, renders the empty state, closes on click-outside.
```

---

#### Task 12.1.1.4 — Search analytics logging (query + result count)

```
You are working in backend/shop/models.py, views.py. Assume Task
12.1.1.1 is already merged.

CONTEXT
Nothing records what customers actually search for — this is valuable
merchandising data (what are people looking for that the catalog
doesn't have? which searches return zero results and represent a
missed sale or a catalog gap? what are the most popular search terms,
useful for homepage/category curation decisions later) and currently
completely absent.

TASK
Log every search query (term + result count + timestamp + optional
user) to a new model, without adding meaningful latency to the actual
search request.

REQUIREMENTS
- Add a `SearchQueryLog` model:
  ```python
  class SearchQueryLog(models.Model):
      query = models.CharField(max_length=255)
      result_count = models.PositiveIntegerField()
      user = models.ForeignKey(
          settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True, blank=True,
          related_name="search_logs",
      )
      created_at = models.DateTimeField(auto_now_add=True, db_index=True)

      class Meta:
          ordering = ["-created_at"]

      def __str__(self):
          return f"'{self.query}' ({self.result_count} results)"
  ```
  Import `settings` from `django.conf` if not already imported.
  Generate the migration.
- Log a search in `ProductListView.get_queryset()` (Task 12.1.1.1),
  AFTER the query executes (so `result_count` reflects the actual
  match count) — but do this WITHOUT blocking the response on the log
  write, since search is a hot, latency-sensitive path and a slow
  logging write shouldn't degrade the customer-facing search
  experience. If Celery infrastructure exists by this point (per this
  project's established Epic-22 dependency pattern from prior epics),
  dispatch the log write as an async task:
  ```python
  # shop/tasks.py
  @shared_task
  def log_search_query(query, result_count, user_id):
      from .models import SearchQueryLog
      SearchQueryLog.objects.create(query=query, result_count=result_count, user_id=user_id)
  ```
  and call `log_search_query.delay(search_term, queryset.count(), request.user.id if request.user.is_authenticated else None)`
  from the view — BUT note this requires calling `.count()` on the
  queryset separately from whatever pagination/serialization later
  iterates it, which is an EXTRA database round-trip on every search
  request purely for logging purposes; DRF's pagination already
  computes a count internally for the response's pagination metadata
  (`count` field) — check whether you can hook into that ALREADY-
  computed count (e.g. via overriding `list()` and reading
  `self.paginator.page.paginator.count` after pagination has run,
  rather than calling `.count()` a second, redundant time) to avoid
  doubling the query cost of every single search request just for
  analytics. This performance detail matters — implement whichever
  approach avoids the redundant count query if reasonably achievable,
  and only fall back to a plain second `.count()` call if hooking into
  the existing paginated count turns out to be more complex than it's
  worth for this task's scope.
  If Celery isn't confirmed available in this build at this point, a
  direct synchronous `SearchQueryLog.objects.create(...)` call is an
  acceptable fallback (creating one small row is fast; it's not in the
  same league of concern as a network call), but prefer the async path
  if it's genuinely available — don't block this task on Celery being
  present if it isn't yet.
- Only log NON-EMPTY search terms (skip logging when
  `?search=`/`?q=` is absent or blank — that's just normal browsing,
  not a search).
- Add the model to backend/shop/admin.py as a read-only view (this is
  analytics data, not something to hand-edit):
  ```python
  @admin.register(SearchQueryLog)
  class SearchQueryLogAdmin(admin.ModelAdmin):
      list_display = ("query", "result_count", "user", "created_at")
      list_filter = ("created_at",)
      search_fields = ("query",)
      readonly_fields = [f.name for f in SearchQueryLog._meta.fields]

      def has_add_permission(self, request):
          return False
  ```
- Add a simple admin-facing "top zero-result searches" report — either
  a custom admin view/action, or (simpler) just confirm
  `SearchQueryLogAdmin` supports filtering by `result_count=0` via the
  standard admin list filters, which requires adding
  `list_filter = ("created_at", "result_count")` or a custom
  `SimpleListFilter` for the specific "zero results" case (mirroring
  the `NearExpiryFilter`/`LowStockFilter` pattern from Epics 3/4) —
  this is genuinely useful, actionable data (a recurring zero-result
  search term is a strong signal of either a catalog gap or a search-
  quality bug) and worth making easy for an admin to find.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/:
1. A search request with a non-empty term creates exactly one
   `SearchQueryLog` row with the correct `query`/`result_count`
   (mock/await the Celery task synchronously in the test if using the
   async path, matching whatever pattern prior epics' Celery task
   tests already use).
2. A request with no search term, or a blank one, creates NO log row.
3. `result_count` accurately reflects the actual number of matching
   products, not an unrelated/incorrect count.
4. An authenticated user's search log correctly records their `user`;
   an anonymous search logs `user=None`.
5. If you implemented the count-reuse optimization, add a test/assertion
   (via `assertNumQueries` or a query-counting context manager)
   confirming a search request does NOT execute the product-matching
   query twice — this is worth explicitly locking in given how easy it
   would be for a future refactor to silently reintroduce the redundant
   count call.
```

---

## Phase 12.2 — Filter UX

### Feature 12.2.1 — Cosmetics Filter Facets

---

#### Task 12.2.1.1 — Frontend filter sidebar for skin/hair type, SPF, vegan/cruelty-free

```
You are working in frontend/src (product listing page and a new
FilterSidebar component). Assume Epic 3 Task 3.2.1.15 is already merged
on the backend (ProductFilter already supports `skin_type`, `hair_type`,
`gender`, `min_spf`/`max_spf`, `is_cruelty_free`, `is_vegan`,
`is_organic` as query parameters, with `skin_type`/`hair_type` using
the comma-separated multi-value `__in` pattern already established for
`brand`/`color`).

CONTEXT
The backend filter capability for cosmetics-specific facets has existed
since Epic 3 — nothing on the frontend surfaces it yet. The product
listing page currently likely only exposes category/brand/color/price
filtering (whatever existed before Epic 3's cosmetics work) with no UI
for the newer attributes at all.

TASK
Build a filter sidebar exposing skin type, hair type, SPF range,
gender, and the three boolean certification flags (cruelty-free, vegan,
organic), wired to the existing backend query parameters.

REQUIREMENTS
- Check the existing product-listing page/component structure first
  (find it — likely `ProductList.jsx`/`Shop.jsx`/`Category.jsx` or
  similar) and identify whatever EXISTING filter UI already exists
  (category/brand/color/price, per the pre-Epic-3 baseline) — this
  task ADDS to that existing sidebar, it doesn't replace it; match its
  existing visual style/interaction pattern (checkboxes vs. pills,
  collapsible sections, etc.) exactly rather than introducing an
  inconsistent new filter UI pattern alongside the old one.
- Add filter sections for:
  - **Skin Type**: multi-select checkboxes (oily/dry/combination/
    sensitive/normal/all), mapped to `?skin_type=oily,dry` per the
    existing comma-separated backend convention.
  - **Hair Type**: same multi-select pattern, `?hair_type=...`.
  - **SPF**: a range control (min/max — could be two number inputs, or
    a dual-handle slider if this project already has a slider component
    available; a simple min/max input pair is perfectly acceptable if
    not) mapped to `?min_spf=`/`?max_spf=`.
  - **Gender**: single-select (unisex/female/male), mapped to `?gender=`.
  - **Certifications**: three independent checkboxes (Cruelty-Free,
    Vegan, Organic), each mapped to `?is_cruelty_free=true`/
    `?is_vegan=true`/`?is_organic=true` respectively — note these are
    independent booleans, not mutually exclusive, and each should only
    be INCLUDED in the query string when checked (an unchecked box
    should NOT send `?is_vegan=false` — omit the parameter entirely,
    since the backend filter's absence means "no restriction," which is
    the correct default state, not an explicit false-value filter).
- Each filter section should be collapsible/expandable if the existing
  sidebar pattern already supports that (match precedent); if the
  existing sidebar is always-fully-expanded, keep this consistent
  rather than introducing new collapse behavior just for these new
  sections.
- Show the active filter count/values somewhere visible (a "Filters
  (3)" indicator, or active filter chips with individual remove
  buttons) if the existing sidebar already has this pattern for
  category/brand/color filters — extend it to include the new facets
  rather than building a separate, parallel active-filters display.
- Add a "Clear all filters" action that resets every filter (existing
  AND new) back to unfiltered state.

ACCEPTANCE CRITERIA
Manually verify: selecting multiple skin types correctly narrows
results to products matching ANY of them; combining skin type + vegan +
min SPF correctly narrows to products matching ALL conditions
simultaneously (AND across facets, OR within a multi-select facet,
matching the backend's actual filter semantics from Epic 3 Task
3.2.1.15); clearing an individual filter and "clear all" both work
correctly. Add component tests for the new filter sections covering:
correct query-param construction for each filter type, multi-select
behavior for skin/hair type, and that unchecked booleans are omitted
from the query rather than sent as `false`.
```

---

#### Task 12.2.1.2 — Filter result counts (facet counts)

```
You are working in backend/shop/views.py, serializers.py (or a new
dedicated facet-count endpoint) and the frontend filter sidebar from
Task 12.2.1.1. Assume that task is already merged.

CONTEXT
The filter sidebar currently shows filter OPTIONS but no indication of
how many products match each option (e.g. "Oily (12)" telling the
customer 12 products match if they select that skin type) — a
well-known, valuable e-commerce filter UX pattern (facet counts) that
helps customers avoid selecting a combination that yields zero results,
and is currently entirely absent.

TASK
Add a facet-count endpoint returning, for the CURRENT filter context
(i.e. respecting whatever OTHER filters are already applied), how many
products match each possible value of each filterable facet.

REQUIREMENTS
- This is a genuinely non-trivial query problem: computing "if I ALSO
  selected skin_type=dry, how many results would there be" for EVERY
  possible facet value, for EVERY facet, all reflecting whatever
  filters are ALREADY applied (except the facet being counted itself —
  standard faceted-search behavior counts each facet's options against
  all OTHER active filters, not including its own current selection,
  so the customer can see what selecting a DIFFERENT value in the same
  facet would yield). Implement this as a dedicated endpoint rather
  than trying to bolt it onto the existing product-list response,
  since it requires running several additional aggregate queries:
  ```python
  # shop/views.py
  class ProductFacetCountsView(APIView):
      permission_classes = [AllowAny]

      def get(self, request):
          base_qs = Product.objects.select_related("category", "brand")
          # Apply all CURRENTLY active filters EXCEPT the facet being counted,
          # for each facet — reuse ProductFilter for consistency rather than
          # reimplementing filter logic here.
          counts = {}
          for facet_field in ["skin_type", "hair_type", "gender"]:
              qs = self._apply_filters_except(base_qs, request.query_params, exclude=facet_field)
              counts[facet_field] = list(
                  qs.exclude(**{f"{facet_field}": ""})
                  .values(facet_field)
                  .annotate(count=Count("id"))
                  .order_by(facet_field)
              )
          for bool_field in ["is_cruelty_free", "is_vegan", "is_organic"]:
              qs = self._apply_filters_except(base_qs, request.query_params, exclude=bool_field)
              counts[bool_field] = qs.filter(**{bool_field: True}).count()
          return Response(counts)

      def _apply_filters_except(self, queryset, query_params, exclude):
          params = query_params.copy()
          params.pop(exclude, None)
          return ProductFilter(params, queryset=queryset).qs
  ```
  Adjust the exact shape/field list to match whatever facets Task
  12.2.1.1 actually built in the sidebar (skin_type, hair_type, gender,
  the three boolean flags — SPF as a continuous range doesn't have
  discrete "counts per value" in the same way and can reasonably be
  excluded from this endpoint, since a min/max range slider doesn't
  need per-value counts the way a checkbox list does).
  Register the URL: `path("products/facet-counts/", views.ProductFacetCountsView.as_view(), name="product-facet-counts"),`
  in shop/urls.py.
- Be mindful of query cost: this endpoint runs SEVERAL aggregate
  queries per request (one per facet). For a request-per-page-load
  pattern (fetched once when the listing page loads and whenever
  filters change), this is acceptable, but confirm it isn't being
  called on every single keystroke or excessively — the frontend should
  call this endpoint alongside the main product-list fetch (same
  triggers: initial load, any filter change), not more frequently.
  Consider whether caching this response briefly (Task from Epic 21's
  caching work, if already landed — a short TTL cache keyed by the
  current filter query-string combination) is warranted; if Epic 21
  hasn't landed yet at this point in your build order, skip caching for
  now and note it as a natural future optimization rather than building
  cache infrastructure prematurely inside this task.

REQUIREMENTS — frontend
- Call this endpoint alongside the main product list fetch (same
  triggers: page load, filter change) and display the count next to
  each filter option label (e.g. "Oily (12)"), updating whenever
  filters change.
- Handle a facet value with a count of 0 sensibly — either grey it out/
  disable it (common pattern: still show the option but make clear
  selecting it yields nothing) or hide it entirely; either is
  defensible UX, pick one and be consistent.

ACCEPTANCE CRITERIA / TESTS
Add backend tests:
1. With no filters active, facet counts reflect the FULL unfiltered
   catalog's distribution across each facet value.
2. With ONE filter active (e.g. `?gender=female`), the facet counts for
   OTHER facets (skin_type, etc.) correctly reflect only products
   matching that active filter — but the facet counts FOR gender itself
   still show all gender options' counts as if gender weren't filtered
   (the "exclude self" behavior described above) — construct this
   scenario explicitly and verify both halves of the behavior.
3. Boolean facet counts (`is_vegan` etc.) return a correct single count
   of products matching `True`, respecting other active filters.
Add a frontend test confirming facet counts render next to filter
options and update after a filter change (mock the API response).
```

---

#### Task 12.2.1.3 — Persist filter state in URL query params

```
You are working in frontend/src (product listing page routing). Assume
Task 12.2.1.1 is already merged.

CONTEXT
Check whether filter state is ALREADY reflected in the URL for the
pre-existing category/brand/color/price filters (a common baseline
pattern that may already exist before this epic's cosmetics-facet
additions) — if it already exists for those, this task is narrower:
extend the SAME existing URL-sync mechanism to cover the new facets
from Task 12.2.1.1 rather than building URL-state-sync from scratch. If
NO filter state is currently reflected in the URL at all (filters live
only in component state, lost on refresh/back-button/share), this task
is the fuller build described below.

TASK
Ensure every active filter (existing and newly-added from Task
12.2.1.1) is reflected in the URL's query string, is restored correctly
on page load/refresh from that URL, and produces genuinely shareable/
bookmarkable filtered URLs.

REQUIREMENTS
- Use React Router's `useSearchParams()` hook (standard, idiomatic way
  to sync component filter state with the URL query string in a
  React-Router-based app — confirm this project uses React Router,
  which prior epics' grounding already established it does) as the
  SOURCE OF TRUTH for filter state, rather than maintaining filter
  state in separate `useState` that's manually kept in sync with the
  URL as a side effect — deriving state FROM the URL (reading
  `searchParams` directly, calling `setSearchParams()` to update)
  avoids an entire class of state-synchronization bugs compared to
  keeping two separate copies of the same state.
- On initial page load, read ALL filter values (skin_type, hair_type,
  min/max SPF, gender, the three boolean flags, plus whatever
  pre-existing filters already work this way — category, brand, color,
  price, search term, sort order) from `searchParams` and initialize the
  filter sidebar's displayed selections accordingly — a customer
  arriving via a shared/bookmarked filtered URL should see the SAME
  filter selections reflected in the sidebar UI, not just have the
  PRODUCT LIST silently filtered with no visible indication of why.
- On any filter change, update the URL via `setSearchParams()`
  (React Router will handle re-rendering/re-fetching appropriately) —
  use `{ replace: true }` for the navigation to avoid polluting browser
  history with every single checkbox toggle (a user hitting "back"
  after adjusting 5 filters shouldn't have to hit back 5 times to
  return to the unfiltered listing — replacing history entries for
  filter changes, while still pushing a real history entry for
  PAGE navigation, e.g. clicking into a product then back, is the
  correct, expected browser-history UX here).
- Confirm the browser back/forward buttons correctly restore prior
  filter states (a natural consequence of using `searchParams` as the
  source of truth correctly, but verify it explicitly rather than
  assuming).
- Multi-value filters (skin_type, hair_type — comma-separated per the
  backend convention) should encode/decode correctly to/from a single
  query param value (e.g. `?skin_type=oily,dry`), not multiple repeated
  params (`?skin_type=oily&skin_type=dry`) unless you confirm the
  backend's `ProductFilter` actually expects the repeated-param style
  instead — check Epic 3 Task 3.2.1.15's actual implementation
  (comma-split-then-`__in` pattern) and match its exact expected format
  precisely, since a mismatch here would silently break multi-select
  filtering via URL even though it works fine via direct UI
  interaction.

ACCEPTANCE CRITERIA
Manually verify: applying several filters (a mix of pre-existing and
new cosmetics facets) updates the URL correctly; copying that URL into
a new browser tab restores the exact same filter selections and
results; the browser back button correctly steps back through filter
changes without excessive history pollution; refreshing the page
preserves filter state (since it's read from the URL, not lost
component state). Add tests confirming: filters read from an initial
URL correctly populate sidebar state on mount, filter changes correctly
update the URL query string in the expected format (including correct
comma-separated encoding for multi-select facets), and
`{ replace: true }` navigation is used for filter changes specifically
(not full push-based navigation).
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 12.1.1.1 | Postgres full-text search (weighted SearchVector) | ☐ |
| 12.1.1.2 | Persian text normalization for search | ☐ |
| 12.1.1.3 | Search-as-you-type suggestion endpoint | ☐ |
| 12.1.1.4 | Search analytics logging | ☐ |
| 12.2.1.1 | Frontend filter sidebar for cosmetics facets | ☐ |
| 12.2.1.2 | Filter result counts (facet counts) | ☐ |
| 12.2.1.3 | Persist filter state in URL query params | ☐ |

Once Epic 12 is fully merged, the next epic to generate prompts for is
**Epic 13 — Recommendation Engine**, which the master backlog notes
depends on Epic 3's catalog model (already merged) rather than on this
epic specifically, and can be built independently of Epic 12's search
work.
