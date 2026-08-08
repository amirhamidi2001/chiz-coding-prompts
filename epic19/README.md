# Epic 19 — Blog & Content — AI Coding Prompts

Repo: `tablogenix-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–18 are fully merged, including Epic 14 Task 14.4.1.2's Persian-to-Latin `persian_slugify()` transliteration function.

**Important grounded discovery for this epic — read before starting:** the `blog` app is **already substantially built out** — `Post`, `Category`, and `Comment` models (the latter supporting threaded replies via a self-referencing `parent` FK), a working `PostListView` (search/filter/ordering, category filtering via `PostFilter`), `PostDetailView` (with atomic view-count incrementing), `RelatedPostsView` (**already** returns related posts by shared category, with a most-recent-posts fallback — this is a **post-to-post** recommender, distinct from what Task 19.1.1.3 below builds, which links posts to **store products**), and `CommentCreateView`. This epic's tasks are additive extensions on top of this real foundation, not a greenfield build.

**A real, confirmed bug this epic should fix while already in these files:** `blog/models.py`'s `Post.save()` and `Category.save()` both call Django's default `slugify()` **without `allow_unicode=True`** — the **exact same bug class** Epic 14 Task 14.4.1.2 fixed for the `shop` app's `Product`/`Category`/`Brand` — but Epic 14's fix was scoped specifically to the `shop` app and **never touched `blog`**. A Persian-titled blog post (which, given this platform's Persian-primary content strategy, is presumably the norm, not the exception) is right now generating an empty or garbage slug identically to the pre-Epic-14 shop bug. Task 19.1.1.2 below fixes this while already working in `blog/models.py`.

**A related, out-of-scope-but-worth-flagging gap, not one of this epic's listed tasks:** `CommentCreateView` has `permission_classes = [AllowAny]` with a free-text `name`/`email` field and zero authentication or spam protection — structurally the *exact* pre-Epic-1 vulnerability shop `Review` had before Task 1.3.1.3 required authentication. This isn't in this epic's task list (which is scoped to content features, not comment moderation/security), but it's worth noting for whoever picks up comment-related hardening later — flagged here rather than silently expanded into this epic's scope.

---

## Phase 19.1 — Blog Enhancements

### Feature 19.1.1 — Content Features

---

#### Task 19.1.1.1 — Product tagging within blog posts (link posts to products)

```
You are working in backend/blog/models.py, serializers.py, migrations.
Assume Epics 1–18 are fully merged, including Epic 3's `ProductVariant`
model.

CONTEXT
No connection between blog content and the product catalog exists at
all — a common, valuable content-commerce pattern ("shop this look":
a blog post about a skincare routine linking directly to the specific
products featured in it) is currently unbuildable.

TASK
Add a many-to-many relationship between `Post` and `Product`, with
enough structure to support "shop this look"-style curated product
lists within a post (not just an unordered bag of linked products —
ORDER and optional per-link annotation matter for this use case, e.g.
"Step 1: cleanser," "Step 2: serum").

REQUIREMENTS
- Rather than a plain `ManyToManyField(Product)` (which loses ordering
  and can't carry per-link metadata), use an explicit THROUGH model:
  ```python
  class PostProductTag(models.Model):
      post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="product_tags")
      product = models.ForeignKey(
          "shop.Product", on_delete=models.CASCADE, related_name="blog_mentions"
      )
      order = models.PositiveSmallIntegerField(default=0)
      note = models.CharField(
          max_length=200, blank=True,
          help_text="Optional short annotation, e.g. 'Step 1: Cleanser' or 'Featured shade: 320'.",
      )

      class Meta:
          ordering = ["order"]
          unique_together = ("post", "product")

      def __str__(self):
          return f"{self.post.title} → {self.product.name}"
  ```
  Using a string reference `"shop.Product"` for the cross-app FK,
  matching the exact established pattern for every cross-app FK in
  this codebase since `CartItem.product`/`StockMovement.related_order`
  etc.
  Add to `Post`:
  `products = models.ManyToManyField("shop.Product", through=PostProductTag, related_name="tagged_in_posts", blank=True)`
- Generate the migration.
- Register `PostProductTagInline` on `PostAdmin` (check
  `blog/admin.py` for the existing `PostAdmin` registration and its
  current inline/field configuration before adding to it):
  ```python
  class PostProductTagInline(admin.TabularInline):
      model = PostProductTag
      extra = 1
      autocomplete_fields = ["product"]
      fields = ["product", "order", "note"]
  ```
  Add `PostProductTagInline` to `PostAdmin.inlines` (confirm
  `ProductAdmin`/whatever admin the `product` autocomplete needs
  already has `search_fields` configured, per Django's autocomplete
  requirement — it does, per Epic 3 Task 3.1.1.6's grounding).
- Update `PostDetailSerializer` (backend/blog/serializers.py) to
  include tagged products:
  ```python
  class PostProductTagSerializer(serializers.ModelSerializer):
      product_id = serializers.IntegerField(source="product.id", read_only=True)
      product_name = serializers.CharField(source="product.name", read_only=True)
      product_slug = serializers.CharField(source="product.slug", read_only=True)
      product_thumbnail = serializers.SerializerMethodField()
      product_price = serializers.SerializerMethodField()

      class Meta:
          model = PostProductTag
          fields = ["product_id", "product_name", "product_slug", "product_thumbnail", "product_price", "note", "order"]

      def get_product_thumbnail(self, obj):
          request = self.context.get("request")
          if obj.product.thumbnail and request:
              return request.build_absolute_uri(obj.product.thumbnail.url)
          return None

      def get_product_price(self, obj):
          variant = obj.product.variants.filter(is_active=True).order_by("price").first()
          return variant.price if variant else None
  ```
  Add `product_tags = PostProductTagSerializer(many=True, read_only=True)`
  to `PostDetailSerializer` (querying via the `related_name="product_tags"`
  already established on the through model above — note the ordering
  is already correctly applied via `PostProductTag.Meta.ordering`, so
  no extra `.order_by()` is needed in the serializer field itself).
- Update `PostDetailView.get_queryset()` to `.prefetch_related("product_tags__product__variants")`
  (avoiding N+1 queries when the serializer resolves each tagged
  product's lowest active variant price).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/blog/tests/test_models.py and test_serializers.py:
1. A `Post` can have multiple tagged products with a defined order and
   optional notes; `post.product_tags.all()` returns them correctly
   ordered.
2. `unique_together` prevents tagging the SAME product twice on the
   same post.
3. `PostDetailSerializer` output includes the correctly-ordered,
   correctly-serialized `product_tags` list with resolved thumbnail
   URL and current lowest-variant price.
4. A post with NO tagged products returns an empty `product_tags` list
   (not null/missing key), and doesn't cause any N+1 query regression
   (spot-check with `assertNumQueries` given the prefetch requirement
   above).
```

---

#### Task 19.1.1.2 — Blog category/tag taxonomy

```
You are working in backend/blog/models.py, serializers.py, filters.py,
admin.py. Assume Epics 1–18 are fully merged.

CONTEXT — READ THIS DOCUMENT'S HEADER BEFORE STARTING
`blog.Category` (single category per post, via `Post.category` FK) and
category-based filtering (`PostFilter`) ALREADY EXIST and work. This
task adds a SEPARATE, complementary concept: multi-valued TAGS (a post
can have many tags, unlike its single category) — a standard content-
taxonomy pattern distinct from category (which is typically a coarser,
single-choice classification; tags are typically finer-grained,
multi-valued, and more free-form). This task also fixes the confirmed
Persian-slug bug in this same file while already working in it.

TASK
1. Fix the confirmed `slugify()` Persian-transliteration bug in
   `Post.save()`/`Category.save()`.
2. Add a `Tag` model and multi-valued Post↔Tag relationship, with
   corresponding filtering.

REQUIREMENTS — Part 1: the slug bug fix
- Import `persian_slugify` from wherever Epic 14 Task 14.4.1.2 placed
  it (`shop/text_utils.py` or equivalent — check the actual location
  established in that task) into `blog/models.py`.
- Replace `self.slug = slugify(self.title)` in `Post.save()` and
  `self.slug = slugify(self.name)` in `Category.save()` with
  `persian_slugify(...)` calls, matching the exact collision-avoidance-
  loop-preservation approach already established when Epic 14 Task
  14.4.1.2 fixed the equivalent `shop` app bug (`Post.slug` is
  `unique=True` but currently has NO collision-avoidance loop at all,
  unlike `shop.Product.save()`'s loop — check this carefully: if
  `Post.save()` currently just does a bare `slugify()` assignment with
  no uniqueness handling whatsoever, two posts with the same/similar
  title would currently either collide with a raw `IntegrityError` at
  save time, or (if titles happen to transliterate to genuinely
  different slugs) simply not collide by luck — ADD a proper
  collision-avoidance loop here too, matching the established pattern
  from `shop.Product.save()`, since this task is directly fixing
  slug-generation correctness and a missing collision guard is
  squarely in scope, not a separate concern):
  ```python
  def save(self, *args, **kwargs):
      if not self.slug:
          base_slug = persian_slugify(self.title)
          slug = base_slug
          counter = 1
          while Post.objects.filter(slug=slug).exclude(pk=self.pk).exists():
              slug = f"{base_slug}-{counter}"
              counter += 1
          self.slug = slug
      word_count = len(self.content.split())
      self.read_time = max(1, round(word_count / 200))
      super().save(*args, **kwargs)
  ```
  Apply the equivalent fix to `Category.save()`.
- Add a data migration re-saving every existing `Post`/`Category` row
  with a Persian title/name to correct any currently-broken slugs
  (mirroring the exact backfill approach from Epic 14 Task 14.4.1.2/
  14.4.1.3).

REQUIREMENTS — Part 2: Tag model
- Add:
  ```python
  class Tag(models.Model):
      name = models.CharField(max_length=50, unique=True)
      slug = models.SlugField(max_length=60, unique=True, blank=True)

      class Meta:
          ordering = ["name"]

      def save(self, *args, **kwargs):
          if not self.slug:
              self.slug = persian_slugify(self.name)
          super().save(*args, **kwargs)

      def __str__(self):
          return self.name
  ```
  Add to `Post`: `tags = models.ManyToManyField(Tag, related_name="posts", blank=True)`
  (a plain M2M, not a through model — unlike `PostProductTag` from Task
  19.1.1.1, there's no per-link metadata/ordering need for tags, so a
  plain M2M is the correct, simpler choice here).
  Generate the migration.
- Register `Tag` in `blog/admin.py`, and add a `filter_horizontal = ("tags",)`
  to `PostAdmin` for a usable multi-select widget.
- Extend `PostFilter` (backend/blog/filters.py) with tag filtering,
  mirroring the exact comma-separated multi-value pattern already
  established for `shop.ProductFilter`'s `skin_type`/`hair_type` fields
  (Epic 3 Task 3.2.1.15):
  ```python
  tags = django_filters.CharFilter(method="filter_tags")

  def filter_tags(self, queryset, name, value):
      tag_slugs = [t.strip() for t in value.split(",") if t.strip()]
      return queryset.filter(tags__slug__in=tag_slugs).distinct()
  ```
  (the `.distinct()` is necessary here since filtering across a
  many-to-many relationship can produce duplicate rows when a post
  matches multiple requested tags — a detail worth getting right, not
  optional).
  Add `"tags"` to `PostFilter.Meta.fields`.
- Add a `GET /api/blog/tags/` endpoint (mirroring the existing
  `CategoryListView`'s exact pattern — published-post-count annotation,
  `AllowAny`, no pagination) so the frontend can build a tag-browsing
  UI.
- Add `tags` to `PostListSerializer`/`PostDetailSerializer` output (a
  simple nested list of `{id, name, slug}`).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/blog/tests/:
1. A Persian-titled Post/Category now produces a correct, non-empty,
   properly-transliterated slug (the direct fix verification, mirroring
   Epic 14 Task 14.4.1.2's equivalent test structure).
2. Two posts with identical/near-identical Persian titles receive
   distinct slugs via the newly-added collision-avoidance loop.
3. `?tags=skincare,makeup` correctly returns posts matching EITHER tag,
   with no duplicate rows in the result.
4. `GET /api/blog/tags/` returns only tags with at least one published
   post, correctly annotated with post count, matching
   `CategoryListView`'s exact established behavior.
5. `PostDetailSerializer`/`PostListSerializer` output correctly
   includes the post's tags.
```

---

#### Task 19.1.1.3 — Related-products widget on blog post page

```
You are working in frontend/src (blog post detail page component).
Assume Task 19.1.1.1 is already merged. RE-READ this document's header
— this task is DISTINCT from the already-existing `RelatedPostsView`
(post-to-post recommendations), which stays completely untouched.

CONTEXT
Task 19.1.1.1 built the backend data model and serializer for tagged
products on a post (`PostDetailSerializer`'s `product_tags` field) —
nothing on the frontend displays them yet.

TASK
Add a "Shop This Look" (or similarly-titled) widget to the blog post
detail page, rendering the post's tagged products.

REQUIREMENTS
- Locate the blog post detail page component (find it in frontend/src/
  pages — check for an existing `BlogPost.jsx`/`BlogDetail.jsx` or
  similar, which should already exist given the backend `PostDetailView`
  is confirmed live and presumably already consumed by SOME frontend
  page).
- Once post data loads (which now includes `product_tags`, per Task
  19.1.1.1), render a widget when `product_tags.length > 0`:
  ```jsx
  {post.product_tags?.length > 0 && (
    <div className="my-10 p-6 bg-gray-50 rounded-2xl">
      <h3 className="text-lg font-bold text-gray-800 mb-4">در این پست ببینید</h3>
      <div className="grid grid-cols-2 sm:grid-cols-3 gap-4">
        {post.product_tags.map((tag) => (
          <a key={tag.product_id} href={`/products/${tag.product_slug}/`} className="block">
            <img src={tag.product_thumbnail} alt={tag.product_name} className="rounded-lg w-full aspect-square object-cover" />
            {tag.note && <p className="text-xs text-teal-600 mt-1">{tag.note}</p>}
            <p className="text-sm font-medium text-gray-800 mt-1">{tag.product_name}</p>
            <p className="text-sm text-gray-500">{formatToman(tag.product_price)}</p>
          </a>
        ))}
      </div>
    </div>
  )}
  ```
  (adjust the Persian heading text/exact styling to match this
  project's established design system and translation conventions;
  import `formatToman` from Epic 14 Task 14.1.2.3's currency utility —
  confirm this blog page hasn't ALSO fallen into the same stale-USD-
  formatting trap discovered repeatedly across the account area in
  Epic 18, since blog is a separate page that may not have been swept
  during that epic's currency-formatting fixes; check for and fix any
  similar issue here while in this file).
  Position this widget sensibly within the post layout — commonly
  either near the top (immediately visible) or interspersed within/
  after the main content — check whichever placement this project's
  existing design conventions favor for supplementary content blocks,
  or use reasonable judgment if there's no clear precedent.
- Handle a product tag whose underlying `product_price` is `None`
  (per Task 19.1.1.1's serializer, this happens if the tagged product
  currently has NO active variants — e.g. fully discontinued/
  deactivated) — show the product card without a price rather than a
  broken/blank price display, or consider omitting fully-unavailable
  products from the widget display entirely (your call — either is
  defensible; a discontinued product still mentioned in old blog
  content arguably has some informational value even if unpurchasable,
  but a broken-looking price display is worth avoiding regardless of
  which you choose).

ACCEPTANCE CRITERIA
Manually verify: a blog post with tagged products shows the widget with
correct images/names/prices/notes in the correct order; a post with no
tagged products shows nothing (no empty/broken widget shell). Add
component tests confirming: renders correctly with tagged products
present, doesn't render when absent, correctly formats prices via
`formatToman()`, and handles a `null` product_price gracefully per
whichever handling approach you chose.
```

---

#### Task 19.1.1.4 — Blog RSS feed

```
You are working in backend/blog/feeds.py (new file), urls.py. Assume
Epics 1–18 are fully merged.

CONTEXT
No RSS/Atom feed exists for the blog — a standard, low-effort content-
syndication feature (lets readers subscribe via feed readers, and is
also a minor but real SEO/discoverability signal) currently absent.

TASK
Add an RSS feed for published blog posts using Django's built-in
syndication framework.

REQUIREMENTS
- Create backend/blog/feeds.py:
  ```python
  from django.conf import settings
  from django.contrib.syndication.views import Feed
  from django.utils.feedgenerator import Rss201rev2Feed
  from .models import Post


  class PersianRssFeedGenerator(Rss201rev2Feed):
      def rss_attributes(self):
          attrs = super().rss_attributes()
          attrs["xml:lang"] = "fa"
          return attrs


  class LatestPostsFeed(Feed):
      feed_type = PersianRssFeedGenerator
      title = "تابلوژنیکس — مجله زیبایی"  # match this project's real established blog/brand name
      link = "/blog/"
      description = "آخرین مقالات و راهنماهای زیبایی و مراقبت از پوست از تابلوژنیکس."

      def __call__(self, request, *args, **kwargs):
          self.request = request
          return super().__call__(request, *args, **kwargs)

      def items(self):
          return Post.objects.filter(status=Post.Status.PUBLISHED).order_by("-published_at")[:20]

      def item_title(self, item):
          return item.title

      def item_description(self, item):
          return item.excerpt or item.content[:300]

      def item_link(self, item):
          return f"{settings.FRONTEND_URL}/blog/{item.slug}/"

      def item_pubdate(self, item):
          return item.published_at

      def item_author_name(self, item):
          return item.author.profile.get_fullname() if item.author and hasattr(item.author, "profile") else "تابلوژنیکس"
  ```
  Note `item_link()` correctly points at the FRONTEND domain (via
  `settings.FRONTEND_URL`), NOT the backend API domain — this is
  EXACTLY the same domain-resolution principle already established
  (and fixed as a real, confirmed bug) in Epic 15 Task 15.1.2.1's
  sitemap work — a blog RSS feed pointing readers at backend API URLs
  instead of the actual readable frontend pages would be the identical
  class of bug; get this right from the start here rather than
  repeating that mistake in a new context.
  The `xml:lang="fa"` override via a custom `Rss201rev2Feed` subclass
  is a nice-to-have correctness detail (declaring the feed's language
  correctly for feed readers/aggregators) — worth including given how
  much of this epic's broader work has been about Persian-language
  correctness, but not critical if it proves fiddly to get exactly
  right; a working feed without this specific attribute is still a
  reasonable fallback if you hit real friction implementing it.
- Register the URL in backend/blog/urls.py:
  `path("feed/", LatestPostsFeed(), name="blog-rss-feed"),`
  (mounted under whatever prefix `blog/urls.py` is already included
  at in `core/urls.py` — confirm the final public URL, e.g.
  `/api/blog/feed/`, and consider whether an RSS feed URL under an
  `/api/` prefix reads oddly to feed-reader software/users expecting a
  more conventional `/blog/feed/` or `/feed/` path — if this project's
  URL structure makes a cleaner top-level path easy to add, consider
  also registering it directly in `core/urls.py` at a more
  conventional location; if that's meaningfully more complex given how
  this project's URL routing is structured, the `/api/blog/feed/` path
  is a perfectly functional fallback, just less conventional).
- Add a `<link rel="alternate" type="application/rss+xml">` tag to the
  frontend's blog LISTING page (via `react-helmet-async`, per Epic 15
  Task 15.1.1.1's established pattern) so browsers/feed-reader
  extensions can auto-discover the feed:
  ```jsx
  <link rel="alternate" type="application/rss+xml" title="تابلوژنیکس — مجله زیبایی" href={`${API_BASE_URL}/blog/feed/`} />
  ```

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/blog/tests/test_feeds.py (new file):
1. `GET /api/blog/feed/` returns a `200` with `content-type` indicating
   RSS/XML, and the response body is valid, parseable XML (parse it
   with Python's `xml.etree.ElementTree` or `feedparser` if available,
   and confirm no parse errors).
2. The feed includes only PUBLISHED posts, correctly excluding drafts
   (construct both and confirm only the published one appears).
3. Each feed item's `<link>` uses the FRONTEND domain
   (`settings.FRONTEND_URL`), not the backend/API domain — the direct
   regression test for the domain-resolution principle emphasized
   above.
4. Feed items are ordered most-recent-published-first, and capped at
   20 items even if more published posts exist.
5. Manually validate the feed against a real RSS validator (e.g. the
   W3C Feed Validation Service, if reachable from your environment) to
   confirm it's genuinely spec-compliant, not just superficially
   well-formed XML.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 19.1.1.1 | Product tagging within blog posts ("shop this look") | ☐ |
| 19.1.1.2 | Blog category/tag taxonomy (+ Persian slug bug fix) | ☐ |
| 19.1.1.3 | Related-products widget on blog post page | ☐ |
| 19.1.1.4 | Blog RSS feed | ☐ |

Once Epic 19 is fully merged, the next epic to generate prompts for is
**Epic 20 — Media Management**, which can run independently of this
epic and most others at this point in the backlog, focused on image
optimization/thumbnailing across every `ImageField` in the project
(product images, blog cover images, review images, profile pictures)
rather than any single app's domain logic.
