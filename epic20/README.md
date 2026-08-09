# Epic 20 — Media Management — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–19 are fully merged.

**Confirmed directly from the repo — the complete inventory of `ImageField`s this epic must cover:**
1. `shop.Category.image`
2. `shop.Brand.logo`
3. `shop.Product.thumbnail`
4. `shop.ProductImage.image` (multiple images per product)
5. `accounts.Profile.image` (avatar)
6. `blog.Post.cover_image`
7. `shop.ReviewImage.image` (added in Epic 10 Task 10.1.1.3)

Also confirmed: `Pillow` is already installed (`pillow==12.1.1`), storage is **local filesystem only** (`MEDIA_ROOT`/`MEDIA_URL`, no S3/cloud storage backend configured anywhere), and there is **no existing thumbnail/resizing pipeline** of any kind.

**A real, currently-exploitable security gap confirmed directly in the repo — this is the central finding Task 20.1.1.5 exists to fix:** `backend/dashboard/serializers.py`'s `AvatarUploadSerializer.validate_image()` (the ONLY image-upload validation that currently exists anywhere in this codebase) validates file type via `value.content_type not in ["image/jpeg", "image/png", "image/webp"]` — **the client-supplied `Content-Type` header from the multipart upload request, not the actual file's real content.** This header is trivially spoofable — a malicious upload can set `Content-Type: image/jpeg` on a file that is not actually a JPEG at all (an HTML file, a script, anything) and this check does nothing to catch it. This is precisely the "don't trust file extensions/content-type headers" gap the project's original architecture review flagged, and it exists in code that already shipped. Every OTHER `ImageField` in the enumerated list above (6 of the 7) currently has **zero validation of any kind** — no size limit, no type check at all.

---

## Phase 20.1 — Image Pipeline

### Feature 20.1.1 — Optimization

---

#### Task 20.1.1.1 — Add `django-imagekit` (or equivalent) for thumbnail generation

```
You are working in backend/requirements.txt, core/settings/base.py,
and shop/models.py (plus the other 6 models with ImageFields, per this
document's header inventory). Assume Epics 1–19 are fully merged.

CONTEXT
No thumbnail generation exists anywhere — every image URL served to
the frontend today is the FULL, original-resolution uploaded file,
regardless of whether it's being displayed as a tiny product-card
thumbnail or a full-size PDP gallery image. This wastes bandwidth
significantly (a customer browsing a product GRID downloads full-
resolution images for dozens of products just to display them at
~200px) and is a real, measurable performance cost.

TASK
Add `django-imagekit` and configure automatic thumbnail generation for
every product-facing image field.

REQUIREMENTS
- Add `django-imagekit` to backend/requirements.txt, pinned to a
  current stable version matching this project's fully-pinned
  dependency convention. Add `"imagekit"` to `INSTALLED_APPS` in
  backend/core/settings/base.py.
- For EACH of the 7 confirmed `ImageField`s (per this document's
  header), add `ImageSpecField` thumbnail variants appropriate to how
  that image is actually USED across the frontend (check actual usage
  contexts before picking sizes — don't apply one blanket size to
  every field):
  - `shop.Product.thumbnail` — the PRIMARY driver of this task, given
    it's the single most-requested image across the site (every
    product card everywhere). Add:
    ```python
    from imagekit.models import ImageSpecField
    from imagekit.processors import ResizeToFill

    class Product(models.Model):
        thumbnail = models.ImageField(upload_to="products/thumbnails/")
        thumbnail_small = ImageSpecField(
            source="thumbnail", processors=[ResizeToFill(200, 200)],
            format="WEBP", options={"quality": 80},
        )
        thumbnail_medium = ImageSpecField(
            source="thumbnail", processors=[ResizeToFill(400, 400)],
            format="WEBP", options={"quality": 85},
        )
        thumbnail_large = ImageSpecField(
            source="thumbnail", processors=[ResizeToFill(800, 800)],
            format="WEBP", options={"quality": 90},
        )
    ```
    (three sizes — small for grid/list views, medium for standard
    cards, large for PDP hero — reasonable starting breakpoints;
    adjust exact pixel dimensions if you find the frontend's actual
    rendered sizes differ meaningfully from these guesses by checking
    real component CSS).
  - `shop.ProductImage.image` — same three-size treatment (PDP gallery
    images need the same responsive range as the primary thumbnail).
  - `shop.Category.image`, `shop.Brand.logo` — a single reasonable
    size each (these render at more consistent, predictable sizes
    across the site than product photos — e.g. category/brand cards) —
    add ONE `ImageSpecField` per model at an appropriate fixed size
    rather than three variants, since the multi-size responsive need
    is much weaker here.
  - `accounts.Profile.image` (avatar) — one small, fixed-size variant
    (e.g. 150×150) — avatars are consistently small everywhere they
    appear.
  - `blog.Post.cover_image` — two sizes (a list/card thumbnail and a
    larger hero size for the post detail page).
  - `shop.ReviewImage.image` — one moderate size (review photos are
    typically shown as a modest gallery, not full-bleed hero images).
  Generate migrations for every model touched.
- `ImageSpecField`s are LAZILY generated on first access by default
  (django-imagekit's standard behavior) — the first request for a
  given thumbnail variant triggers generation and caching to disk;
  subsequent requests serve the cached file. Confirm this default
  behavior is acceptable (a brief one-time generation delay on first
  access per image) or whether PRE-generation at upload time is
  preferred for this project's actual traffic patterns (avoiding any
  customer-facing delay, at the cost of slower admin upload response
  time while thumbnails generate synchronously) — RECOMMEND lazy
  generation as the simpler default for now, but flag that
  django-imagekit also supports `ImageSpecField(..., cachefile_strategy=...)`
  configurations for pre-generation if this becomes a real UX concern
  once there's actual production traffic to observe.
- Update every serializer currently exposing the ORIGINAL full-size
  image URL (across `shop`, `accounts`, `blog` serializers, built out
  over many prior epics) to ALSO expose the new thumbnail variant URLs
  where appropriate — e.g. `ProductListSerializer` (used for browse/
  grid contexts) should expose `thumbnail_medium`, not the full-size
  original; `ProductDetailSerializer` can expose `thumbnail_large` for
  the PDP hero plus the full original for a "view full size" zoom
  interaction if the frontend has one. Audit each serializer's actual
  USE CONTEXT rather than mechanically adding every variant to every
  serializer.

ACCEPTANCE CRITERIA / TESTS
- Add model tests confirming each `ImageSpecField` generates a
  correctly-sized, correctly-formatted (WEBP, per this task's spec)
  output when accessed, for at least the highest-traffic fields
  (`Product.thumbnail`, `ProductImage.image`).
- Add serializer tests confirming the correct thumbnail variant URL
  (not the original full-size URL) is returned in list/grid-context
  serializers.
- Manually verify: uploading a large (e.g. 4000×4000px) product image
  through the admin, then viewing the product listing page, shows a
  correctly-downscaled thumbnail — inspect the actual served image's
  dimensions/file size via browser dev tools to confirm it's genuinely
  the small variant, not the original being downscaled purely via CSS
  (which would defeat the entire bandwidth-saving purpose of this
  task).
```

---

#### Task 20.1.1.2 — Convert product images to WebP on upload

```
You are working in backend/shop/models.py (and the other 6 image-
bearing models). Assume Task 20.1.1.1 is already merged.

CONTEXT
Task 20.1.1.1's `ImageSpecField` THUMBNAIL variants already specify
`format="WEBP"` — but the ORIGINAL uploaded image (whatever format the
admin/customer originally uploaded — JPEG, PNG, sometimes even legacy
formats) is still stored and served AS-IS in its original format
whenever a thumbnail variant isn't the one being requested (e.g. any
"view full size" / zoom interaction, or serializer contexts that still
expose the original field directly). WebP typically produces
significantly smaller file sizes than JPEG/PNG at equivalent visual
quality — converting the STORED ORIGINAL itself (not just the
generated thumbnails) closes this gap for full-size image requests too.

TASK
Convert uploaded images to WebP format at SAVE time (not just for
generated thumbnail variants), while preserving a reasonable quality
level, across the same 7 image fields.

REQUIREMENTS
- Add a reusable image-conversion utility, e.g.
  backend/shop/media_utils.py (or wherever this project's established
  convention for shared cross-model utilities lives — check for
  precedent):
  ```python
  from io import BytesIO
  from django.core.files.base import ContentFile
  from PIL import Image


  def convert_to_webp(image_field, quality: int = 85) -> ContentFile:
      """
      Convert an uploaded image file to WebP format, returning a new
      ContentFile ready to replace the original field value. Handles
      RGBA/transparency correctly (WebP supports alpha, unlike JPEG).
      """
      img = Image.open(image_field)
      if img.mode not in ("RGB", "RGBA"):
          img = img.convert("RGBA" if "transparency" in img.info else "RGB")
      buffer = BytesIO()
      img.save(buffer, format="WEBP", quality=quality)
      buffer.seek(0)
      original_name = image_field.name.rsplit(".", 1)[0]
      return ContentFile(buffer.read(), name=f"{original_name}.webp")
  ```
- Apply this conversion in each model's `save()` method, ONLY when the
  image field has actually CHANGED (converting on every save
  regardless of whether the image changed would be wasteful and, worse,
  progressively degrade quality on each re-save of an already-converted
  WebP file through repeated lossy re-encoding — a real, easy-to-miss
  correctness issue):
  ```python
  def save(self, *args, **kwargs):
      if self.thumbnail and not self.thumbnail.name.endswith(".webp"):
          self.thumbnail = convert_to_webp(self.thumbnail)
      ...  # existing save logic (slug generation, etc.) continues as before
      super().save(*args, **kwargs)
  ```
  The `.endswith(".webp")` check is the (simple, adequate) guard against
  re-converting an already-converted file — confirm this correctly
  handles the case of a genuinely NEW upload replacing an existing
  WebP-named file (Django's `FieldFile` change-detection via
  `field.name` on a freshly-assigned `UploadedFile` should correctly
  reflect the NEW file's original name/extension at the point this
  check runs, before conversion — verify this holds with a real test
  rather than assuming, since subtle Django file-field lifecycle
  timing issues are a common source of bugs in exactly this kind of
  save-time file transformation).
- Apply this to ALL 7 image fields' respective `save()` methods (this
  is genuinely repetitive across 5 different model files — consider
  whether extracting a small mixin class,
  `class WebPConversionMixin: def save(self, *args, field_names=(), **kwargs): ...`,
  is worth the abstraction given how many models need the identical
  pattern; use judgment on whether the mixin genuinely simplifies
  things or just adds indirection for what's fundamentally a simple,
  short per-field check — either a mixin or straightforward repetition
  is acceptable, but don't half-do it with inconsistent per-model
  implementations of the same logic).
- Confirm `Content-Type`/served MIME type is correctly `image/webp` for
  converted files (Django's `FileField`/static-file serving typically
  infers content-type from the file EXTENSION, which this conversion
  correctly sets via the `.webp` filename — verify this is actually
  true for however this project serves media files, whether via
  Django's dev-server static serving, Nginx, or whatever production
  media-serving setup exists by this point).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/ (and equivalent for the other touched
apps):
1. Uploading a JPEG/PNG image results in the stored file being
   genuinely WebP format (verify by opening the saved file with
   Pillow and checking `img.format == "WEBP"`, not just checking the
   filename extension, which could theoretically be wrong if the
   conversion logic has a bug).
2. Uploading a PNG with actual transparency correctly preserves alpha
   in the converted WebP (a real, non-trivial correctness check —
   confirm the converted image genuinely has an alpha channel when the
   source did).
3. RE-SAVING a model instance whose image was ALREADY converted (no
   new file uploaded, e.g. just changing an unrelated field and
   calling `.save()` again) does NOT re-run the conversion a second
   time (assert e.g. via mocking `convert_to_webp` and confirming it's
   NOT called on a plain re-save with an unchanged image) — this is
   the direct regression test for the quality-degradation-via-repeated-
   lossy-reencoding concern.
4. Uploading a NEW image to REPLACE an existing (already-WebP) one
   correctly converts the NEW upload (proving the change-detection
   guard correctly distinguishes "no change" from "genuinely new
   file").
```

---

#### Task 20.1.1.3 — Frontend responsive `<img srcset>` usage

```
You are working across frontend/src component files displaying
product/blog/category/brand images. Assume Task 20.1.1.1 is already
merged (multiple thumbnail size variants now available via the API).

CONTEXT
Even with Task 20.1.1.1's multiple server-side thumbnail sizes now
available, the frontend currently has no mechanism to request the
RIGHT size for the actual viewport/display context — a plain `<img
src="...">` always requests whatever single URL the serializer
happened to provide, regardless of whether it's being viewed on a
small mobile screen or a large desktop grid.

TASK
Add `srcset`/`sizes` attributes to image-rendering components,
leveraging Task 20.1.1.1's multiple size variants so the browser can
select the most appropriate one for the actual rendering context.

REQUIREMENTS
- Ensure the relevant API serializers (Task 20.1.1.1's follow-through)
  expose ALL size variants needed to build a `srcset`, not just one
  chosen size — e.g. `ProductListSerializer` should expose
  `thumbnail_small`/`thumbnail_medium`/`thumbnail_large` URLs together
  (not just the one the LIST context primarily uses), so the frontend
  component can construct a proper `srcset` string offering the
  browser real choices — revisit Task 20.1.1.1's "audit each
  serializer's actual use context" guidance: for `srcset` to work at
  all, the serializer needs to expose MULTIPLE sizes, not just the
  single "primary" one that task's initial pass may have settled on;
  reconcile this now.
- Build a reusable `ResponsiveImage` component (e.g.
  frontend/src/components/ResponsiveImage.jsx) wrapping the `srcset`/
  `sizes` pattern consistently:
  ```jsx
  export default function ResponsiveImage({ srcSmall, srcMedium, srcLarge, alt, className, sizes }) {
    const srcSet = [
      srcSmall && `${srcSmall} 200w`,
      srcMedium && `${srcMedium} 400w`,
      srcLarge && `${srcLarge} 800w`,
    ].filter(Boolean).join(', ');

    return (
      <img
        src={srcMedium || srcLarge || srcSmall}
        srcSet={srcSet}
        sizes={sizes || '(max-width: 640px) 200px, (max-width: 1024px) 400px, 800px'}
        alt={alt}
        className={className}
        loading="lazy"
      />
    );
  }
  ```
  (the `sizes` default is a reasonable starting breakpoint set —
  adjust to match this project's ACTUAL Tailwind breakpoints/grid
  column counts at different viewport widths, checking real component
  CSS rather than guessing; a mismatched `sizes` attribute doesn't
  break anything visually but does undermine the browser's ability to
  make genuinely optimal size choices, defeating part of this task's
  purpose).
  Note `loading="lazy"` is included here directly since it's cheap and
  correct to always include (though Task 20.1.1.4 covers this more
  deliberately/comprehensively including for contexts where this
  shared component isn't used — don't consider this line alone as
  satisfying that task).
- Replace plain `<img>` tags across the highest-traffic image-display
  locations with `<ResponsiveImage>`: product cards (list/grid views,
  wherever `ProductListSerializer` output is rendered), the PDP main
  image, category/brand card images, blog post cards/cover images —
  prioritize by actual traffic/frequency of appearance (a product card
  rendered dozens of times per page load across every listing page is
  far higher-value to fix than a one-off avatar image), and treat this
  as a multi-session sweep if the codebase's scale warrants it (matching
  the same "prioritize, document remaining work" pattern established
  for large sweeps in Epic 14).

ACCEPTANCE CRITERIA / TESTS
Add component tests for `ResponsiveImage`:
1. Renders correct `srcSet` string when all three size props are
   provided.
2. Gracefully omits a size from `srcSet` when that specific prop is
   `undefined`/`null` (e.g. a model with only ONE thumbnail variant,
   per Task 20.1.1.1's "not every field needs three sizes" guidance)
   rather than producing a malformed `srcset` string with an empty
   entry.
3. Falls back sensibly to SOME valid `src` even if only one size is
   available.
Manually verify via browser dev tools' Network tab: on a narrow
(mobile-width) viewport, the browser genuinely requests the SMALL
image variant, not the large one — the actual, real-world proof this
task's bandwidth-saving goal is achieved, not just that the HTML
attributes are technically present.
```

---

#### Task 20.1.1.4 — Lazy-load below-the-fold images

```
You are working across frontend/src component files. Assume Task
20.1.1.3 is already merged (which already added `loading="lazy"` to
the shared `ResponsiveImage` component specifically).

CONTEXT
Task 20.1.1.3's `ResponsiveImage` component already sets
`loading="lazy"` for wherever IT is used — but (a) that sweep was
explicitly scoped as "prioritize, don't necessarily finish every
location in one pass," so plain `<img>` tags likely still remain in
lower-priority spots, and (b) native browser lazy-loading via the
`loading="lazy"` attribute, while broadly supported in modern browsers,
has known limitations worth being deliberate about (e.g. it doesn't
lazy-load images that are ALREADY within the initial viewport at page
load — which is actually the CORRECT, desired behavior, since
above-the-fold images should load immediately, not lazily).

TASK
Complete the lazy-loading sweep across any remaining below-the-fold
image locations Task 20.1.1.3 didn't already cover, and verify the
native `loading="lazy"` approach is actually sufficient for this
project's needs (vs. needing a more sophisticated Intersection-
Observer-based library for finer control).

REQUIREMENTS
- Sweep the codebase for remaining plain `<img>` tags NOT already
  using `ResponsiveImage` (search for `<img ` across frontend/src and
  cross-reference against what Task 20.1.1.3 already covered) —
  prioritize genuinely below-the-fold contexts: product review images
  (Epic 10), wishlist item thumbnails (Epic 11), "recently viewed"/
  "recommended" widget images (Epic 13), blog post listing thumbnails,
  admin dashboard images — add `loading="lazy"` directly to any of
  these NOT already routed through `ResponsiveImage`, or migrate them
  to use `ResponsiveImage` if they'd also benefit from that
  component's `srcset` handling (most product-adjacent images would;
  purely decorative/small icon-like images likely don't need the full
  responsive-image treatment, just the lazy-loading attribute).
- EXPLICITLY do NOT apply `loading="lazy"` to genuinely above-the-fold
  images — the homepage hero banner, the FIRST product image on a PDP
  (the one visible without scrolling), and similar immediately-visible
  content should load EAGERLY (`loading="eager"` or simply omitting
  the `loading` attribute, which defaults to eager) — lazy-loading an
  above-the-fold image actually HURTS perceived performance (a
  deliberate delay before the browser even starts fetching an image
  the user sees immediately) rather than helping it; audit
  `ResponsiveImage`'s usage sites and any newly-swept locations to
  confirm above-the-fold instances correctly OVERRIDE the component's
  lazy default (add a `loading` prop to `ResponsiveImage` allowing
  callers to override the default, defaulting to `"lazy"` but
  explicitly settable to `"eager"` for hero/above-the-fold call sites).
- Evaluate (briefly — this doesn't need to be a full separate spike
  task) whether native `loading="lazy"` is sufficient given this
  project's actual browser-support requirements, or whether a more
  robust Intersection-Observer-based approach (e.g. via a small custom
  hook, or a library if one's already a dependency) is warranted for
  finer control (e.g. lazy-loading with a larger pre-fetch margin
  before the image actually enters the viewport, for a smoother
  scrolling experience with less visible "pop-in") — native
  `loading="lazy"` has been broadly supported in modern browsers for
  long enough that it's almost certainly sufficient for this project's
  needs without added complexity; RECOMMEND sticking with the native
  attribute rather than adding a library/custom-hook dependency for
  marginal additional control, but document this as a considered
  decision rather than an unexamined default.

ACCEPTANCE CRITERIA / TESTS
- Manually verify via browser dev tools' Network tab: scrolling down a
  long product-listing or blog-listing page shows images loading
  progressively as they approach the viewport (confirming lazy-loading
  is genuinely active), while the FIRST, above-the-fold images on page
  load are already loaded/loading immediately without waiting for
  scroll.
- Add a component test confirming `ResponsiveImage`'s `loading` prop
  correctly overrides the default (`loading="eager"` passed explicitly
  results in that attribute on the rendered `<img>`, not the default
  `"lazy"`).
```

---

#### Task 20.1.1.5 — Enforce max upload size + file type validation on all image fields

```
You are working across backend/shop/serializers.py, accounts/
serializers.py, blog/serializers.py, dashboard/serializers.py (fixing
the existing `AvatarUploadSerializer`), and any other serializer
accepting one of the 7 confirmed `ImageField`s. Assume Epics 1–19 are
fully merged. RE-READ THIS DOCUMENT'S HEADER BEFORE STARTING — this
task fixes a real, currently-exploitable vulnerability, not a
hypothetical one.

CONTEXT — THE CONFIRMED VULNERABILITY THIS TASK FIXES
`dashboard/serializers.py`'s `AvatarUploadSerializer.validate_image()`
is the ONLY image validation anywhere in this codebase, and it checks
`value.content_type` — a value the CLIENT supplies in the multipart
upload request itself and can set to anything regardless of the
file's actual content. This means today, right now, an attacker can
upload a non-image file to the avatar-upload endpoint by simply
setting the `Content-Type` header to `image/jpeg` on the request,
completely bypassing this check. The other 6 `ImageField`s across this
codebase have ZERO validation of any kind — no size limit, no type
check whatsoever.

TASK
Build ONE shared, reusable, CONTENT-based (not header-based) image
validator, and apply it consistently across every image-accepting
serializer in the codebase, replacing the vulnerable
`content_type`-based check.

REQUIREMENTS
- Create a shared validator — given this needs to be used across
  MULTIPLE apps (`shop`, `accounts`, `blog`, `dashboard`), place it
  somewhere genuinely shared rather than duplicated per-app; check
  whether this project has any existing shared/`common` app or utility
  module convention from prior epics (if `IranProvince` ended up
  relocated out of `dashboard` per Epic 7 Task 7.1.1.3's discussion,
  that's a natural existing precedent/location to extend; otherwise,
  create a minimal new shared location, e.g. backend/core/validators.py):
  ```python
  from PIL import Image, UnidentifiedImageError
  from rest_framework import serializers

  ALLOWED_IMAGE_FORMATS = {"JPEG", "PNG", "WEBP"}
  MAX_IMAGE_SIZE_MB = 5


  def validate_uploaded_image(value, max_mb: int = MAX_IMAGE_SIZE_MB):
      """
      Validate an uploaded image by ACTUAL FILE CONTENT, not the
      client-supplied Content-Type header (which is trivially
      spoofable and must never be trusted for security purposes).
      """
      if value.size > max_mb * 1024 * 1024:
          raise serializers.ValidationError(f"Image must be under {max_mb} MB.")

      try:
          value.seek(0)
          img = Image.open(value)
          img.verify()  # raises if the file is corrupt or not a genuine image
          value.seek(0)
          # img.verify() invalidates the Image object for further use —
          # re-open to actually check the format after verification.
          img = Image.open(value)
          actual_format = img.format
      except (UnidentifiedImageError, OSError, ValueError):
          raise serializers.ValidationError("The uploaded file is not a valid image.")
      finally:
          value.seek(0)  # rewind so Django can still read the file for actual saving afterward

      if actual_format not in ALLOWED_IMAGE_FORMATS:
          raise serializers.ValidationError(
              f"Unsupported image format: {actual_format}. Allowed: JPEG, PNG, WEBP."
          )
      return value
  ```
  This is the CORE fix: `Image.open(value).format` (Pillow reading the
  file's actual internal structure/magic bytes to determine its real
  format) is what a security-meaningful check looks like — completely
  independent of whatever `Content-Type` header the client claims,
  which this validator doesn't even look at. `img.verify()` additionally
  catches truncated/corrupted files that might technically have valid
  magic bytes but aren't fully valid images.
  The `value.seek(0)` calls at multiple points are important and
  easy to get wrong — Pillow's `Image.open()` reads from the file's
  CURRENT position, and Django's subsequent save logic needs the file
  pointer back at the start afterward; get this sequencing right and
  test it explicitly, since a missed `seek(0)` would either make the
  validation read garbage or corrupt the actually-saved file.
- Replace `AvatarUploadSerializer.validate_image()`'s vulnerable
  `content_type`-based check with a call to this shared validator:
  ```python
  def validate_image(self, value):
      return validate_uploaded_image(value)
  ```
- Apply the SAME shared validator to every OTHER image-accepting
  serializer across the codebase, covering all 7 confirmed
  `ImageField`s from this document's header — `ProductImageSerializer`/
  admin product-image upload paths (`shop`), `CategorySerializer`/
  `BrandSerializer` admin upload paths, `ReviewCreateSerializer`'s
  image list (Epic 10 Task 10.1.1.3 — note this one validates a LIST
  of images, so apply `validate_uploaded_image()` to each item in the
  list, per that task's existing `validate_images()` method structure),
  `blog` admin post cover-image upload, and any bulk-import path from
  Epic 17 Task 17.2.1.2 that might accept image references (check
  whether the bulk importer handles image FILES directly or just
  references existing ones by URL/path — if the former, it needs this
  validation too; if the latter, it's out of scope for this specific
  file-upload validator).
- Audit each application point for a CONSISTENT max-size limit —
  `AvatarUploadSerializer`'s existing 5MB limit is a reasonable
  default, but consider whether every field genuinely needs the SAME
  limit (e.g. a product gallery image might reasonably need a slightly
  higher limit than a small avatar, or vice versa — use judgment, but
  pick DELIBERATE, documented limits per context rather than either
  blindly copying 5MB everywhere without consideration or leaving some
  fields with no limit at all).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/core/tests/test_validators.py (or wherever the
shared validator lives) — this is the most security-critical test
suite in this entire epic, treat it accordingly:
1. A genuine, valid JPEG/PNG/WEBP file passes validation.
2. A file EXCEEDING the size limit is rejected with a clear message.
3. **A file with a SPOOFED `Content-Type` header claiming to be an
   image, but whose ACTUAL CONTENT is not a valid image (e.g. a plain
   text file, an HTML file, or a script) is REJECTED** — this is the
   direct, concrete regression test proving the vulnerability described
   in this task's context is actually fixed; construct this test by
   creating an `InMemoryUploadedFile`/`SimpleUploadedFile` with
   genuinely non-image byte content but an image-claiming
   `content_type` parameter, and confirm `validate_uploaded_image()`
   rejects it based on actual content inspection.
4. A corrupted/truncated image file (valid magic bytes for a real
   format, but incomplete/damaged data) is rejected via `img.verify()`
   catching it.
5. A genuinely valid image file in an UNSUPPORTED format (e.g. a valid
   GIF or BMP, if those aren't in `ALLOWED_IMAGE_FORMATS`) is rejected
   with the correct format-specific error message.
6. Confirm the file pointer `seek(0)` sequencing is correct: after
   `validate_uploaded_image()` returns successfully, the SAME file
   object can still be correctly read/saved by Django's normal model-
   save machinery afterward (construct an end-to-end test: submit a
   valid image through a real serializer using this validator, and
   confirm the resulting saved file on disk is genuinely intact and
   correctly readable, not corrupted/truncated by the validation
   process having consumed/mishandled the stream).
Add integration tests confirming EACH of the 6 previously-unvalidated
serializers now correctly rejects an oversized/invalid-content upload
— don't just test the shared validator function in isolation; prove
it's actually WIRED UP correctly at every one of the 7 real API
endpoints, since a validator that exists but isn't actually called from
somewhere provides zero real protection.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 20.1.1.1 | Add django-imagekit for thumbnail generation | ☐ |
| 20.1.1.2 | Convert images to WebP on upload | ☐ |
| 20.1.1.3 | Frontend responsive `<img srcset>` usage | ☐ |
| 20.1.1.4 | Lazy-load below-the-fold images | ☐ |
| 20.1.1.5 | Max upload size + real content-based file validation | ☐ |

Once Epic 20 is fully merged, the next epics to generate prompts for
are **Epic 21 — Caching (Redis)** and **Epic 22 — Celery**, both
infrastructure-focused epics — worth noting several PRIOR epics'
prompts in this series (Epic 3's expiry sweep, Epic 4's stock/price
alerts, Epic 6's payment reconciliation, Epic 16's notification
retries) already explicitly assumed Celery infrastructure exists and
flagged it as a soft dependency — Epic 22 is where that assumption
finally gets a real foundation built under it, if it hasn't been
pulled forward already per the master backlog's own note that Celery
should realistically be stood up early rather than strictly saved for
its numbered position.
