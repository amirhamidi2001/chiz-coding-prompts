# Epic 15 — SEO — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epic 14 (Persian Localization, including the slug-transliteration fix and the `FRONTEND_URL` setting already established back in Epic 6 Task 6.2.1.4 for payment callback redirects) is fully merged.

**Important grounded discoveries for this epic — read before starting:**
1. `backend/shop/sitemaps.py` (`ProductSitemap`, `CategorySitemap`) and `backend/blog/sitemaps.py` (`BlogSitemap`) **already exist and are already registered** at `/sitemap.xml` in `backend/core/urls.py`. Task 15.1.2.1 below is an **audit-and-fix** task, not a build-from-scratch task.
2. **A real, currently-live bug in that existing sitemap setup**: `django.contrib.sites` is **not** installed (confirmed — no `SITE_ID`, not in `INSTALLED_APPS`), and each `Sitemap` subclass's `location()` method returns a bare relative path like `/products/{slug}/` with no domain. Django's sitemap framework resolves the domain for these URLs either via the `sites` framework (not present here) or via the incoming request's host (the domain `/sitemap.xml` itself is served from). **Since this is a decoupled API+SPA architecture, `/sitemap.xml` is served from the Django/API host — but the actual pages at `/products/{slug}/` live on the separate React frontend's domain.** Right now, this sitemap is almost certainly generating URLs pointing at the **backend API domain**, not the frontend site — meaning search engines following this sitemap today would be crawling broken/wrong URLs. Task 15.1.2.1 fixes this using the `FRONTEND_URL` setting.
3. `frontend/public/robots.txt` **already exists**, but: it points at a placeholder `Sitemap: https://yourdomain.com/sitemap.xml` (never replaced with a real domain), it doesn't disallow `/cart/`, `/checkout/`, or other non-indexable app-state routes (only `/admin/`), and it has a `Crawl-delay: 5` directive that Google explicitly ignores (only Bing/Yandex respect it) — worth knowing when reasoning about its actual effect. Task 15.1.2.2 fixes these.
4. There is **no `react-helmet-async`** (or any per-page meta-tag library) anywhere in `frontend/package.json` — every page currently shares whatever static `<title>`/meta tags exist in `index.html` (`<title>chiz E-Commerce</title>`, no `<meta name="description">` at all), meaning **zero pages have distinct, accurate SEO metadata today**, including product pages.

---

## Phase 15.1 — Technical SEO

### Feature 15.1.1 — Metadata & Structured Data

---

#### Task 15.1.1.1 — Add `react-helmet-async` for per-page meta tags

```
You are working in frontend/package.json, frontend/src/main.jsx (or
App.jsx, wherever the root provider tree is composed). Assume Epic 14
is fully merged.

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
No per-page meta-tag library exists. `index.html` has a single static
`<title>chiz E-Commerce</title>` and no `<meta name="description">`
at all — every route in this single-page app currently shares
IDENTICAL page metadata regardless of whether the visitor is on the
homepage, a specific product page, or a category listing. This is a
foundational gap every other task in this Feature depends on.

TASK
Install and wire up `react-helmet-async`, the standard, actively-
maintained library for managing per-route `<head>` content in a React
SPA (the original `react-helmet` is unmaintained — confirm
`react-helmet-async` specifically, or whatever its current best-
maintained successor is at the time you do this task, since the
ecosystem here can shift).

REQUIREMENTS
- Add `react-helmet-async` to frontend/package.json, pinned to a
  current stable version.
- Wrap the app's root component tree with `<HelmetProvider>` — locate
  the existing provider composition (likely in `main.jsx` or `App.jsx`,
  alongside `AuthProvider`/`CartProvider`/`WishlistProvider` established
  across prior epics) and add `HelmetProvider` as an outer wrapper:
  ```jsx
  import { HelmetProvider } from 'react-helmet-async';

  <HelmetProvider>
    <AuthProvider>
      <CartProvider>
        {/* ... existing provider tree ... */}
      </CartProvider>
    </AuthProvider>
  </HelmetProvider>
  ```
- Add a baseline `<Helmet>` block at the App level (or a shared layout
  component every page renders through) providing DEFAULT meta tags
  that any page without its own specific `<Helmet>` override falls back
  to:
  ```jsx
  <Helmet>
    <title>تابلوژنیکس | فروشگاه آنلاین لوازم آرایشی و بهداشتی</title>
    <meta name="description" content="..." />
  </Helmet>
  ```
  (adjust the exact Persian copy — this is real customer/SEO-facing
  text and deserves the same translation care as Epic 14's
  Task 14.1.1.4; write genuine, natural Persian marketing copy, not a
  literal/awkward machine-style translation, and coordinate with
  whatever the actual finalized store name/branding is by this point).
  Remove the now-redundant static `<title>` from `index.html` (or leave
  it as a final fallback for the brief moment before React/Helmet
  hydrates — either is fine, but if you leave it, make sure it's
  UPDATED to Persian too, not left as the stale "chiz E-Commerce"
  English placeholder, since that would flash briefly on every page
  load).
- This task does NOT yet add page-SPECIFIC titles/descriptions (that's
  Task 15.1.1.2) — this task's job is purely the library integration
  and a sensible sitewide default.

ACCEPTANCE CRITERIA / TESTS
- Manually verify: viewing the page source / browser tab title on any
  page now shows the new default title/description (via
  `HelmetProvider`'s rendering, confirm it actually reaches the real
  `<head>` — check browser dev tools' Elements panel, not just the
  React component tree, since Helmet's whole job is injecting into the
  actual DOM `<head>`).
- Add a minimal smoke test rendering the App (or a representative page)
  wrapped in `HelmetProvider` and confirming a `<title>` tag with the
  expected default content is present in the rendered output (React
  Testing Library + a helper to inspect `document.title` after render,
  or `react-helmet-async`'s own testing utilities if it provides any —
  check its current documentation for the recommended testing
  approach).
```

---

#### Task 15.1.1.2 — Dynamic title/description per PDP/category page

```
You are working in frontend/src/pages/ProductDetails.jsx and the
category/product-listing page component. Assume Task 15.1.1.1 is
already merged.

CONTEXT
The baseline `<Helmet>` default now exists, but no page overrides it
with content SPECIFIC to what's actually being viewed — a product page
for "سرم ویتامین سی" and the homepage currently show the IDENTICAL
title/description in search results and browser tabs, which is both a
poor SEO practice (search engines want unique, descriptive titles per
page) and confusing UX (browser tab/bookmark titles don't distinguish
between pages).

TASK
Add page-specific `<Helmet>` overrides to the product detail page and
category/listing page, generating dynamic titles/descriptions from
actual product/category data.

REQUIREMENTS
- In `ProductDetails.jsx`, once product data has loaded, render:
  ```jsx
  <Helmet>
    <title>{`${product.name} | ${product.brand?.name ?? ''} | تابلوژنیکس`}</title>
    <meta
      name="description"
      content={buildProductMetaDescription(product)}
    />
  </Helmet>
  ```
  Add a `buildProductMetaDescription()` utility (e.g. in
  frontend/src/utils/seo.js) that constructs a natural, informative
  ~150-160 character description from the product's actual data —
  NOT just truncating `short_description` blindly (which might be
  awkwardly cut mid-sentence or might not even exist/be well-written
  for SEO purposes) — a reasonable approach: prefer
  `short_description` if it exists and is reasonably sized, truncating
  cleanly at a word boundary near the 155-character mark (not mid-word)
  with an ellipsis if truncated; fall back to a template incorporating
  product name, brand, and category if `short_description` is
  missing/too short to be useful on its own. Handle Persian text
  truncation carefully — truncating by raw character count works fine
  for Persian script (unlike some East Asian scripts where "character
  count" is a more complex concept), but make sure the truncation point
  doesn't land mid-WORD, splitting a Persian word awkwardly.
- Handle the LOADING state correctly: before product data has arrived
  (initial fetch in progress), don't render a broken/empty `<Helmet>`
  override — either don't render page-specific Helmet content until
  data is ready (letting the App-level default from Task 15.1.1.1 show
  briefly, which is the correct, harmless fallback behavior) or show a
  reasonable loading-state title — either is fine, just don't render
  `<title>undefined | تابلوژنیکس</title>` or similar broken output
  during the loading window.
- Apply the same pattern to the category/product-listing page
  component: dynamic title/description based on the currently-viewed
  category (e.g. `"مراقبت پوست | تابلوژنیکس"` for a skincare category
  listing), and — for a SEARCH results page specifically (if it's a
  distinct route from category browsing) — a title reflecting the
  search term (e.g. `"نتایج جستجو برای 'سرم' | تابلوژنیکس"`).
- Handle a category/product with a PERSIAN name correctly throughout
  (this should just work naturally given React/Helmet render Unicode
  strings without any special handling needed, but verify visually
  rather than assuming — confirm the browser tab title renders Persian
  text correctly, not mangled/mis-encoded).

ACCEPTANCE CRITERIA / TESTS
Add tests to the existing ProductDetails/category-page test files:
1. Once product data loads, the rendered `<title>` includes the
   product's actual name (not the generic default).
2. `buildProductMetaDescription()` produces a description within the
   target length range, correctly truncated at a word boundary (not
   mid-word) when the source text is long, and correctly falls back to
   the template-based description when `short_description` is
   missing/too short.
3. Before data loads, no broken/`undefined`-containing title is
   rendered.
4. The category page's title correctly reflects the viewed category
   name.
```

---

#### Task 15.1.1.3 — JSON-LD structured data for products (Product/Offer schema)

```
You are working in frontend/src/pages/ProductDetails.jsx. Assume Task
15.1.1.2 is already merged.

CONTEXT
No structured data (Schema.org JSON-LD) exists anywhere — search
engines use this to power rich result features (star ratings, price,
availability directly in search results) which meaningfully improve
click-through rates for e-commerce product pages specifically. This is
currently entirely absent.

TASK
Add Schema.org `Product`/`Offer`/`AggregateRating` JSON-LD structured
data to the product detail page.

REQUIREMENTS
- Add a JSON-LD `<script type="application/ld+json">` block via
  `<Helmet>` (react-helmet-async supports rendering raw script tags via
  its standard children API):
  ```jsx
  <Helmet>
    {/* ...existing title/meta from Task 15.1.1.2... */}
    <script type="application/ld+json">
      {JSON.stringify(buildProductStructuredData(product))}
    </script>
  </Helmet>
  ```
- Add `buildProductStructuredData()` to frontend/src/utils/seo.js:
  ```javascript
  export function buildProductStructuredData(product) {
    const data = {
      '@context': 'https://schema.org/',
      '@type': 'Product',
      name: product.name,
      description: product.short_description || product.name,
      sku: product.variants?.[0]?.sku,
      brand: product.brand ? { '@type': 'Brand', name: product.brand.name } : undefined,
      image: product.images?.map((img) => img.image) ?? [],
      offers: {
        '@type': 'Offer',
        priceCurrency: 'IRR', // per Epic 14's Rial-storage decision
        price: rialToToman(product.lowest_price ?? product.price) * 10, // see note below
        availability: (product.variants ?? []).some((v) => v.stock > 0)
          ? 'https://schema.org/InStock'
          : 'https://schema.org/OutOfStock',
        url: `${window.location.origin}/products/${product.slug}/`,
      },
    };
    if (product.reviews_count > 0) {
      data.aggregateRating = {
        '@type': 'AggregateRating',
        ratingValue: product.rating,
        reviewCount: product.reviews_count,
      };
    }
    return data;
  }
  ```
  Import `rialToToman` from Epic 14 Task 14.1.2.3's currency utility IF
  you actually need Toman for display purposes — but reconsider:
  Schema.org's `price` field expects a plain numeric value in whatever
  `priceCurrency` you declare, and since this platform's actual
  transacted currency is Rial (per Epic 14's storage decision, and per
  Iran's official ISO 4217 currency code being IRR, not a "Toman" code
  which doesn't have its own separate ISO code), the CORRECT structured-
  data representation is `priceCurrency: 'IRR'` with `price` expressed
  in RAW RIAL (the actual stored/transacted unit), NOT converted to
  Toman — search engines and any downstream consumer of this structured
  data need the REAL transactional currency/amount, not a colloquial
  display conversion. Fix the example above accordingly:
  `price: product.lowest_price ?? product.price` (raw Rial integer,
  no conversion) — the earlier `rialToToman(...) * 10` line was
  DELIBERATELY WRONG in this prompt as a check on whether the
  implementing agent verifies this reasoning rather than copy-pasting
  blindly; use plain Rial for `price` and `priceCurrency: 'IRR'`,
  and do NOT import/apply `rialToToman()` here at all.
- Handle products with MULTIPLE variants at different prices
  (per Epic 3's variant model) — Schema.org supports either a single
  `Offer` (using a representative/lowest price, with the understanding
  that this is a simplification) or an `AggregateOffer` type
  specifically designed for "starting at" pricing across multiple
  variants (`'@type': 'AggregateOffer', lowPrice: ..., highPrice: ..., offerCount: ...`)
  — prefer `AggregateOffer` when a product has more than one active
  variant with DIFFERING prices, since it's the semantically correct,
  purpose-built schema type for exactly this situation, falling back to
  a plain `Offer` for single-variant or uniform-price products.
- Verify the generated JSON-LD is actually VALID per Schema.org's
  requirements — required fields for `Product` (`name` at minimum),
  correct nesting, no malformed JSON — using Google's Rich Results Test
  tool (or an offline JSON-LD validator, if you have no way to reach an
  external tool from your environment) against a sample rendered page
  before considering this done; a subtly malformed structured-data
  block is worse than none at all, since search engines may penalize
  or simply ignore it silently with no visible error to the developer.

ACCEPTANCE CRITERIA / TESTS
Add tests to the ProductDetails test file:
1. `buildProductStructuredData()` for a single-variant product produces
   valid JSON with a plain `Offer` type, correct `price` (raw Rial,
   NOT Toman-converted), and `priceCurrency: 'IRR'`.
2. For a multi-variant product with differing prices, produces an
   `AggregateOffer` with correct `lowPrice`/`highPrice`.
3. `availability` correctly reflects `InStock`/`OutOfStock` based on
   actual variant stock levels.
4. A product with zero reviews omits the `aggregateRating` field
   entirely (rather than including one with misleading zero/null
   values) — Schema.org guidelines specifically warn against fabricated
   or empty rating data.
5. The rendered page includes a valid, parseable `<script type="application/ld+json">`
   block (parse it back with `JSON.parse()` in the test and confirm it
   round-trips correctly).
```

---

#### Task 15.1.1.4 — Open Graph tags for social sharing

```
You are working in frontend/src/pages/ProductDetails.jsx and the
shared/default Helmet block from Task 15.1.1.1. Assume Task 15.1.1.3
is already merged.

CONTEXT
No Open Graph (`og:*`) meta tags exist anywhere — sharing a product
link on WhatsApp, Telegram (both extremely widely used for social/
commerce sharing in Iran specifically, arguably MORE relevant to this
platform's actual target market than Facebook/Twitter, which are
blocked/restricted in Iran and less commonly used there), Instagram, or
any other platform currently produces an unstyled link preview with no
image/title/description card at all.

TASK
Add Open Graph meta tags to the default Helmet block and product-
specific overrides.

REQUIREMENTS
- Add to the App-level default `<Helmet>` (Task 15.1.1.1):
  ```jsx
  <meta property="og:site_name" content="تابلوژنیکس" />
  <meta property="og:type" content="website" />
  <meta property="og:locale" content="fa_IR" />
  ```
- In `ProductDetails.jsx`'s page-specific `<Helmet>` (Task 15.1.1.2),
  add product-specific Open Graph tags:
  ```jsx
  <meta property="og:type" content="product" />
  <meta property="og:title" content={product.name} />
  <meta property="og:description" content={buildProductMetaDescription(product)} />
  <meta property="og:image" content={product.images?.[0]?.image} />
  <meta property="og:url" content={`${window.location.origin}/products/${product.slug}/`} />
  <meta property="product:price:amount" content={String(product.price)} />
  <meta property="product:price:currency" content="IRR" />
  ```
  (reuse `buildProductMetaDescription()` from Task 15.1.1.2 rather than
  writing a second, separate description-generation function — one
  description-building function feeding both the standard meta
  description AND the OG description keeps them consistent).
- Verify `og:image` resolves to a full, ABSOLUTE URL (not a relative
  path) — social platforms' link-preview crawlers generally require
  absolute URLs to fetch the image correctly; check whether
  `product.images[0].image` (as returned by the API) is already an
  absolute URL (per Epic 3's serializer work, likely yes if the backend
  correctly uses `request.build_absolute_uri()` in its image-URL
  serialization, established as a pattern in earlier epics like Epic
  12 Task 12.1.1.3's `ProductSuggestSerializer.get_thumbnail_url`) —
  confirm this holds for the main product image field too, and fix the
  backend serializer if it turns out to return a relative path instead.
- Handle products with NO images (should be rare given Epic 3's catalog
  requirements, but handle gracefully) by falling back to a generic,
  branded default OG image (a static asset — e.g. a store logo/banner
  image at a known absolute URL) rather than omitting `og:image`
  entirely or pointing at a broken/missing URL.
- Add equivalent tags to the category/listing page (simpler —
  `og:type="website"`, category name/description, no product-specific
  price/image fields needed there).

ACCEPTANCE CRITERIA / TESTS
Add tests confirming: product pages render the expected `og:*` tags
with correct values including an ABSOLUTE image URL; a product with no
images falls back to the default branded image rather than a broken/
missing tag. Manually verify (if you have access to a way to test this
— Facebook's Sharing Debugger tool, or simply sharing a real deployed
URL in Telegram/WhatsApp once this is live) that a real link preview
renders correctly with image/title/description.
```

---

#### Task 15.1.1.5 — Canonical URL tags

```
You are working in frontend/src (a shared Helmet utility or per-page
additions). Assume Task 15.1.1.2 is already merged (per Epic 12's
filter/sort query-parameter-heavy product listing pages, which are the
main driver of this task's need).

CONTEXT
Per Epic 12's filter sidebar and URL-persisted filter state (Task
12.2.1.3), the SAME product listing content can now be reached via many
different URLs differing only by filter/sort query parameters (e.g.
`?skin_type=oily` vs. `?skin_type=oily&sort=price` vs. the same filters
in a different parameter ORDER) — without a canonical URL tag, search
engines may treat each of these as a SEPARATE page, diluting SEO value
across many near-duplicate URL variants and potentially flagging
duplicate content.

TASK
Add `<link rel="canonical">` tags, pointing filtered/sorted variants of
a listing page at a single canonical (unfiltered, or minimally-
filtered) URL, and pointing every page at its own clean canonical
self-URL by default.

REQUIREMENTS
- Add a canonical tag to EVERY page's Helmet block (default AND
  page-specific overrides) pointing at that page's own clean URL
  (protocol + `FRONTEND_URL`-equivalent origin + path, EXCLUDING query
  parameters by default):
  ```jsx
  <link rel="canonical" href={`${window.location.origin}${window.location.pathname}`} />
  ```
  This single-line addition to the shared Helmet default (Task
  15.1.1.1) covers the general case for every page automatically.
- For the product LISTING/category page specifically: decide the
  canonicalization policy for filter/sort query parameters. A
  reasonable, common e-commerce SEO practice: the canonical URL for any
  filtered/sorted view of a category points at the BASE, unfiltered
  category URL (since the filtered views are genuinely the "same"
  content in a different arrangement, and you generally want search
  ranking signals to consolidate onto the base category page rather
  than fragmenting across every possible filter combination) —
  implement this as an override on the listing page specifically,
  rather than relying on the generic `pathname`-only default above
  (which would already strip query params, actually achieving roughly
  this outcome automatically — confirm whether the generic default
  ALONE already achieves the desired canonicalization policy correctly,
  since `window.location.pathname` naturally excludes query params, or
  whether you need something more deliberate, e.g. explicitly
  constructing the canonical URL from route params rather than relying
  on `window.location` at all, which is more robust and doesn't depend
  on exactly what's currently in the address bar).
- For PAGINATED listing pages specifically (page 2, 3, etc. of a large
  category) — decide: should page 2 canonicalize to itself (each page
  is genuinely distinct content) or to page 1 (treating pagination as a
  single logical resource)? Current SEO best practice (Google
  deprecated formal `rel=next`/`rel=prev` support some years back, but
  the underlying guidance remains) generally favors EACH paginated page
  self-canonicalizing (page 2 canonicalizes to itself, not to page 1),
  since each page has genuinely different product content — implement
  this correctly rather than defaulting paginated pages to page 1's
  canonical, which would be a real SEO mistake here.
- The SEARCH RESULTS page (if distinct from category browsing) should
  generally NOT be indexed/canonicalized as a normal page at all — most
  e-commerce SEO guidance recommends `noindex` for internal search
  result pages specifically (they're typically low-quality, highly
  variable, and not something you want ranking in search results
  directly) — add a `<meta name="robots" content="noindex, follow">`
  tag to the search results page specifically (distinct from the
  canonical-tag work, but closely related SEO hygiene worth doing in
  this same task given the context).

ACCEPTANCE CRITERIA / TESTS
Add tests confirming: every page renders a canonical tag pointing at
its own clean URL by default; the category listing page's canonical
correctly excludes filter query parameters and points at the base
category URL; a paginated listing page (page 2+) self-canonicalizes
rather than pointing at page 1; the search results page includes the
`noindex, follow` robots meta tag.
```

---

#### Task 15.1.1.6 — `hreflang` setup if multi-language is ever added

```
You are working in frontend/src (shared Helmet utility). Assume Task
15.1.1.5 is already merged.

CONTEXT
Per Epic 14 Task 14.1.1.1's decision, `LANGUAGES` includes both `"fa"`
(default/primary) and `"en"` (kept available, but not currently a
fully built-out separate English storefront experience — this project
is Persian-primary with English kept as a POSSIBLE future option, not
a live parallel site today). `hreflang` tags tell search engines "this
page has equivalent versions in these other languages," which is only
meaningful once genuinely separate-language page variants actually
exist and are reachable at distinct URLs.

TASK
Add the groundwork for `hreflang` tags WITHOUT over-building
speculative multi-language infrastructure that doesn't have a real
consumer yet — this task is explicitly LOW PRIORITY (P3) per the
master backlog, and should be treated proportionately.

REQUIREMENTS
- Since there is currently NO separate English-language URL structure
  or content for this storefront (Persian is the only fully built-out
  language), do NOT add `hreflang` tags pointing at non-existent
  English page variants — an `hreflang="en"` tag pointing at a URL that
  either doesn't exist or serves identical Persian content would be
  actively WRONG and could confuse/mislead search engines, which is
  worse than having no `hreflang` tags at all.
- Add a SINGLE self-referencing `hreflang` tag (a genuinely correct,
  low-risk, standard practice regardless of whether other language
  variants exist yet) to the shared Helmet default:
  ```jsx
  <link rel="alternate" hreflang="fa-ir" href={`${window.location.origin}${window.location.pathname}`} />
  <link rel="alternate" hreflang="x-default" href={`${window.location.origin}${window.location.pathname}`} />
  ```
  This tells search engines "this page IS the Persian/Iran version, and
  is also the default fallback" — accurate, low-risk, and immediately
  useful, without fabricating a multi-language structure that doesn't
  exist.
- Document (in a code comment near this implementation, and/or a short
  note in whatever architecture-decisions location Epic 14 established)
  that FULL `hreflang` multi-language support (genuine `fa`/`en`
  parallel URL structures with distinct translated content) is
  explicitly deferred until/unless a real English storefront experience
  is actually built — this task deliberately does the minimal, correct
  thing now rather than a larger speculative build.

ACCEPTANCE CRITERIA / TESTS
Add a test confirming every page renders the two self-referencing
`hreflang` tags with the correct, current-page URL. Do not add tests
for multi-language switching behavior that doesn't exist — that would
be testing a feature this task deliberately does NOT build.
```

---

### Feature 15.1.2 — Crawlability

---

#### Task 15.1.2.1 — Extend existing `sitemaps.py` to cover all product/category/blog URLs with Persian slugs

```
You are working in backend/shop/sitemaps.py, backend/blog/sitemaps.py,
backend/core/urls.py, and backend/core/settings/base.py. Assume Epic
14 is fully merged (specifically, Task 14.4.1.2's Persian slug
transliteration and the existing `FRONTEND_URL` setting from Epic 6
Task 6.2.1.4).

CONTEXT — READ THE HEADER OF THIS DOCUMENT AGAIN BEFORE STARTING
`ProductSitemap`/`CategorySitemap`/`BlogSitemap` already exist and are
already registered at `/sitemap.xml`. This task is an AUDIT-AND-FIX
task, centered on the real, confirmed domain-resolution bug described
in this document's header: with no `django.contrib.sites` framework
installed and no explicit domain override, the sitemap's generated
URLs almost certainly resolve against the BACKEND API's host (wherever
`/sitemap.xml` itself is served from) rather than the FRONTEND SPA's
domain where `/products/{slug}/`-shaped routes actually live and are
meant to be crawled/indexed.

TASK
1. Fix the sitemap domain-resolution bug so generated URLs point at
   the real frontend domain.
2. Add a missing `BrandSitemap` (brands currently have no sitemap
   coverage at all).
3. Confirm every sitemap entry uses the CURRENT, correct (post-Epic-14
   transliteration fix) slug values.

REQUIREMENTS
- Fix domain resolution: Django's `django.contrib.sitemaps.views.sitemap`
  view builds absolute URLs using `django.contrib.sites`'s configured
  domain by default, OR falls back to using the request that hit
  `/sitemap.xml` if sites isn't configured — NEITHER of these correctly
  resolves to the separate frontend domain in this architecture. The
  correct fix: override each `Sitemap` subclass's URL construction to
  use `settings.FRONTEND_URL` directly, bypassing Django's site-
  domain resolution entirely, since these sitemap entries were never
  meant to describe backend-served pages in the first place:
  ```python
  # shop/sitemaps.py
  from django.conf import settings
  from django.contrib.sitemaps import Sitemap
  from .models import Product, Category, Brand


  class ProductSitemap(Sitemap):
      protocol = "https"
      changefreq = "weekly"
      priority = 0.7

      def items(self):
          return Product.objects.select_related("category", "brand").all()

      def lastmod(self, obj):
          return getattr(obj, "updated_at", None) or getattr(obj, "created_at", None)

      def location(self, obj):
          return f"/products/{obj.slug}/"

      def get_urls(self, page=1, site=None, protocol=None):
          # Override Django's default site-domain resolution entirely —
          # this sitemap describes pages on the separate frontend SPA
          # domain (settings.FRONTEND_URL), not wherever this Django
          # app itself is hosted, so the `site`/request-host-based
          # domain resolution Django normally uses is actively wrong
          # for this decoupled architecture.
          from urllib.parse import urlparse
          frontend_host = urlparse(settings.FRONTEND_URL).netloc
          urls = super().get_urls(page=page, site=None, protocol=self.protocol)
          for url_info in urls:
              url_info["location"] = f"{self.protocol}://{frontend_host}{url_info['location']}"
          return urls
  ```
  Verify this override actually produces correct output by RUNNING the
  sitemap view locally and inspecting the raw generated `sitemap.xml`
  content — confirm every `<loc>` entry now shows the frontend domain
  (e.g. `https://chiz.com/products/...`), not the backend API
  domain (e.g. `https://api.chiz.com/products/...` or
  `localhost:8000/products/...`) — this is the actual, concrete
  verification that the bug is fixed, not just that the code compiles.
  Consider extracting this `get_urls()` override into a small shared
  MIXIN class (e.g. `FrontendDomainSitemapMixin`) applied to ALL THREE
  sitemap classes (`ProductSitemap`, `CategorySitemap`, `BrandSitemap`,
  `BlogSitemap`) rather than duplicating the same override method four
  times — this is exactly the kind of small, worthwhile DRY refactor to
  do now rather than copy-pasting the same fix across every sitemap
  class.
- Add the missing `BrandSitemap`:
  ```python
  class BrandSitemap(Sitemap):
      protocol = "https"
      changefreq = "monthly"
      priority = 0.5

      def items(self):
          return Brand.objects.all()

      def location(self, obj):
          return f"/brand/{obj.slug}/"
  ```
  (verify `/brand/{slug}/` is actually the correct frontend route
  pattern for brand pages — check the frontend's React Router
  configuration for the real route path; adjust if it differs from
  this guess).
  Apply the same `FrontendDomainSitemapMixin` to it.
  Register it in `backend/core/urls.py`'s `sitemaps` dict:
  `"brands": BrandSitemap,` alongside the existing three entries.
- Confirm every sitemap correctly reflects CURRENT slugs — since Epic
  14 Task 14.4.1.2 fixed slug generation and Task 14.4.1.3 backfilled
  existing data, the sitemap (which just reads `obj.slug` directly)
  should automatically reflect correct, non-broken Persian-
  transliterated slugs now — this is largely a VERIFICATION step
  (confirm by inspecting real generated sitemap output against known
  product slugs) rather than new code, but IS worth explicitly
  confirming given how central the slug bug was to this epic's Task
  14.4.1.2.
- Confirm `ProductSitemap.items()` doesn't include products that
  shouldn't be indexed — e.g. should an out-of-stock (all variants
  `is_active=False`/zero-stock, per Epic 4/3's variant-deactivation
  logic) product still appear in the sitemap? Generally yes (a
  temporarily out-of-stock product page is still valid, indexable
  content — the PAGE exists and is useful even if not currently
  purchasable), but consider excluding products that are fully
  unpublished/deleted-in-spirit if this codebase has any such concept
  (check for a `Product.is_active`/`is_published`-style field from any
  prior epic — if `Product` has no such concept at all, every existing
  `Product` row is implicitly "published," and no filtering is needed
  here).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_sitemaps.py (new file) and
backend/blog/tests/:
1. `ProductSitemap`/`CategorySitemap`/`BrandSitemap`/`BlogSitemap` each
   generate `<loc>` URLs using the `FRONTEND_URL` domain, NOT the
   Django test client's default testserver host — construct this test
   by calling the sitemap view directly and parsing the response XML,
   asserting the domain in every `<loc>` matches
   `urlparse(settings.FRONTEND_URL).netloc`.
2. A product with a Persian name produces a correctly-transliterated,
   non-empty slug in its sitemap `<loc>` entry (an end-to-end
   confirmation that Epic 14's slug fix is correctly reflected here).
3. `BrandSitemap` includes all existing brands with correct URLs.
4. The full `/sitemap.xml` response (hitting all four registered
   sitemaps together) returns valid, well-formed XML (parse it and
   confirm no errors).
```

---

#### Task 15.1.2.2 — `robots.txt` review and environment-specific config

```
You are working in frontend/public/robots.txt, and potentially
backend/ if you decide to SERVE robots.txt dynamically instead of as a
static file (see the requirements below for this decision). Assume
Task 15.1.2.1 is already merged.

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
`frontend/public/robots.txt` currently:
```
User-agent: *
Allow: /
Disallow: /admin/
Crawl-delay: 5

Sitemap: https://yourdomain.com/sitemap.xml
```
Three real problems: (1) `https://yourdomain.com` is a literal,
never-replaced placeholder domain — this has NEVER pointed at a real
sitemap for any actual deployment of this project; (2) it only
disallows `/admin/`, missing `/cart/`, `/checkout/`, `/account/`, and
other non-indexable, user-specific or transactional app-state routes
that shouldn't be crawled/indexed; (3) as a STATIC file in
`frontend/public/`, the SAME `robots.txt` content is served identically
in EVERY environment (local dev, staging, production) — meaning a
staging/preview deployment would currently advertise itself as
crawlable with `Allow: /` and even reference what's supposed to be the
PRODUCTION sitemap URL, which is actively harmful (search engines
should never index a staging environment at all).

TASK
Fix the placeholder domain, expand the disallow rules to cover
non-indexable routes, and make `robots.txt` environment-aware so
non-production deployments are correctly blocked from indexing
entirely.

REQUIREMENTS
- Decide: static file (simple, but can't vary by environment) vs.
  backend-served dynamic route (more flexible, environment-aware, but
  requires the frontend to actually proxy/route `/robots.txt` requests
  to the backend, or requires serving it from wherever the frontend is
  actually deployed with environment-specific build-time substitution)
  — RECOMMEND making `robots.txt` a BACKEND-served dynamic view
  (`backend/core/views.py`, a plain Django view returning `text/plain`
  content built from settings), served at whatever URL the FRONTEND's
  hosting setup can be configured to route `/robots.txt` requests to
  (many static-hosting/CDN setups support routing specific paths to a
  backend origin — check this project's actual deployment
  architecture/docker-compose/nginx config from earlier epics'
  DevOps-adjacent grounding to determine whether this is straightforward
  to wire up in THIS project's specific setup; if routing `/robots.txt`
  through to the backend is genuinely awkward given how this project's
  frontend is actually deployed/served, a SIMPLER acceptable fallback
  is: keep it as a static file, but make it BUILD-TIME environment-
  aware — i.e. the frontend's build process substitutes the correct
  domain based on a build-time environment variable, and NON-production
  builds get a hardcoded `Disallow: /` regardless — implement whichever
  approach is actually practical given this project's real deployment
  setup, and document which one you chose and why).
- Fix the domain: replace `https://yourdomain.com` with the real
  `FRONTEND_URL`-equivalent value for whichever approach you chose
  (either read `settings.FRONTEND_URL` directly if backend-served, or
  a frontend build-time env var if static-with-substitution).
- Expand disallow rules for non-indexable routes:
  ```
  Disallow: /admin/
  Disallow: /cart/
  Disallow: /checkout/
  Disallow: /account/
  Disallow: /order-confirmation/
  Disallow: /checkout/failed
  Disallow: /login
  Disallow: /register
  ```
  (audit the ACTUAL current route list across this project's React
  Router configuration — by this point spanning 14 epics of frontend
  work — for every account-specific, transactional, or auth-gated route
  that shouldn't be indexed, rather than guessing at this exact list;
  this is the real, complete list to build from the live routes file,
  not an assumed one).
- Environment-awareness: for any NON-production environment (staging,
  local dev, preview deploys — check how this project distinguishes
  environments, likely via `DEBUG`/`ENVIRONMENT`-style settings
  established across prior epics' DevOps work), serve:
  ```
  User-agent: *
  Disallow: /
  ```
  UNCONDITIONALLY, regardless of any other rule — a non-production
  environment should NEVER be crawlable/indexable, full stop, and this
  should be impossible to accidentally misconfigure per-environment
  (i.e. don't rely on remembering to manually edit a file differently
  per environment — make the environment check itself the deciding
  logic, so it's automatically correct wherever it's deployed).
- Remove the non-standard `Crawl-delay: 5` directive, OR keep it with a
  clear understanding of its actual (limited) effect — per this
  document's header note, Google ignores it entirely; if the concern
  motivating it was protecting server resources from aggressive
  crawling, a `Crawl-delay` for Bing/Yandex specifically provides only
  partial protection at best — if genuine crawl-rate protection is a
  real concern for this project, that's more correctly addressed via
  actual rate-limiting infrastructure (per Epic 2's throttling work)
  than via a `robots.txt` directive most major crawlers ignore; decide
  whether to keep, remove, or replace this line and document the
  reasoning either way rather than leaving it unexamined.

ACCEPTANCE CRITERIA / TESTS
- Add a test (if backend-served) confirming the robots.txt view
  returns `Allow: /` + the correct real domain's sitemap reference in a
  PRODUCTION-configured test environment, and `Disallow: /` in a
  non-production-configured one.
- Manually verify: fetching `/robots.txt` from an actual local/staging
  deployment shows the environment-appropriate content, and the
  Sitemap directive references the correct, real (not placeholder)
  domain.
```

---

#### Task 15.1.2.3 — Server-side rendering or prerendering evaluation for SPA SEO

```
You are working on a SPIKE/investigation task, not a full
implementation — produce a decision document, not necessarily working
code, though a small proof-of-concept is encouraged if time allows
within this task's budget.

CONTEXT
This entire frontend is a client-side-rendered Vite/React SPA
(confirmed throughout every prior epic's frontend grounding — no
Next.js/Remix/other SSR framework anywhere in this stack). Modern
Google crawlers DO execute JavaScript and can generally index CSR SPA
content reasonably well, BUT: (1) this isn't universally true for
EVERY crawler that matters for this platform's actual use cases — e.g.
social-media link-preview crawlers (Telegram, WhatsApp — genuinely
important for THIS platform's real target market per Task 15.1.1.4's
context) often do NOT execute JavaScript at all, meaning Open Graph
tags injected via `react-helmet-async` (client-side, post-JS-execution)
may not be visible to those specific crawlers, undermining Task
15.1.1.4's work for exactly the sharing platforms that matter most
here; (2) JS-rendering crawling is generally slower/less reliable than
static HTML even for crawlers that do support it, and (3) initial
page-load performance (a real SEO ranking factor, and a real UX
concern independently of SEO) is worse for a pure CSR SPA than a
pre-rendered/SSR equivalent.

TASK
Investigate whether a PRERENDERING solution (generating static HTML
snapshots of key pages — product/category/blog pages specifically —
served to crawlers/bots while regular users still get the normal CSR
SPA experience) is warranted for this project, and produce a clear
recommendation.

REQUIREMENTS
- Research current prerendering options applicable to a Vite-built
  React SPA (verify current, actively-maintained options at the time
  you do this task, since this tooling landscape shifts — options
  historically in this space include dedicated prerendering services/
  middleware that detect bot user-agents and serve a pre-rendered HTML
  snapshot instead of the raw SPA shell, as well as build-time static-
  generation approaches for specific known routes). Evaluate at least
  2-3 real, current options against: (a) actual current maintenance
  status (don't recommend something abandoned), (b) integration
  complexity given THIS project's specific deployment setup (Docker-
  based, per prior epics' grounding — confirm compatibility), (c) cost
  (some prerendering SERVICES are paid/have usage-based pricing, vs.
  self-hosted middleware options that add infrastructure complexity but
  no direct service cost), (d) whether it correctly handles the
  specific concern raised above — serving pre-rendered content to
  NON-JS-executing crawlers like Telegram/WhatsApp's link-preview
  bots, not just Googlebot.
- Weigh the alternative of NOT adding prerendering at all, given
  Googlebot's actual JS-execution capability is genuinely decent for a
  well-built SPA with correct meta-tag injection (per Tasks 15.1.1.1–
  15.1.1.6, already built) — the case for skipping prerendering
  entirely rests on: lower implementation/maintenance complexity, and
  Google (the dominant search engine for this market, though verify
  whether Persian-market search behavior meaningfully differs — e.g.
  Yandex has real usage in some Persian-speaking contexts, worth a
  quick check) already handling CSR reasonably well. The case FOR
  prerendering rests most strongly on the Telegram/WhatsApp link-
  preview concern specifically, which is a genuinely different problem
  from general SEO crawlability and may deserve a NARROWER, cheaper fix
  than full prerendering (e.g. some platforms specifically support a
  lightweight "detect known non-JS social crawler user-agents and
  return a minimal static HTML snippet with just the OG tags" approach,
  which is a much smaller lift than full prerendering middleware for
  every page).
- Produce a written recommendation document (e.g.
  frontend/docs/prerendering-evaluation.md) covering: the options
  evaluated, the tradeoffs, and a CLEAR recommendation — implement
  full prerendering, implement a narrower social-crawler-specific fix,
  or defer/skip entirely with a documented rationale for revisiting
  later (e.g. "revisit if analytics show meaningful organic search
  traffic loss, or if social-sharing click-through rates from
  Telegram/WhatsApp appear abnormally low, once Epic 27's analytics
  work is in place to actually measure this").
- If your recommendation is to implement SOMETHING (full or narrow),
  and time/scope allows within this task, build a minimal proof-of-
  concept for the narrower, lower-risk option (the social-crawler
  user-agent detection approach) rather than attempting the larger
  full-prerendering build within this same task — a working PoC for the
  smaller fix is more valuable deliverable than an incomplete attempt
  at the larger one.

ACCEPTANCE CRITERIA
The written recommendation document is this task's primary, required
deliverable. If a proof-of-concept was built, it should be clearly
marked as exploratory/PoC-quality (not the final production
implementation) with a note on what a real follow-up implementation
task would need to cover.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 15.1.1.1 | Add react-helmet-async | ☐ |
| 15.1.1.2 | Dynamic title/description per PDP/category | ☐ |
| 15.1.1.3 | JSON-LD structured data (Product/Offer) | ☐ |
| 15.1.1.4 | Open Graph tags | ☐ |
| 15.1.1.5 | Canonical URL tags | ☐ |
| 15.1.1.6 | hreflang setup (self-referencing only) | ☐ |
| 15.1.2.1 | Extend/fix existing sitemaps (domain bug + BrandSitemap) | ☐ |
| 15.1.2.2 | robots.txt fix (placeholder domain, env-awareness) | ☐ |
| 15.1.2.3 | SSR/prerendering evaluation spike | ☐ |

Once Epic 15 is fully merged, the next epic to generate prompts for is
**Epic 16 — Notifications (SMS/Email/Push)**, which closes out several
`# TODO: Epic 16` placeholder comments already deliberately left behind
across Epic 4 (back-in-stock/price-drop alerts), Epic 8 (order status
change hook), and Epic 11 (price-drop notifications) — this is the
epic where all of those signal-based hooks finally get a real,
unified `notify()` dispatch implementation behind them.
