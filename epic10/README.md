# Epic 10 — Reviews & Ratings — AI Coding Prompts

Repo: `tablogenix-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–9 are fully merged. Specifically relevant here: Epic 1 Feature 1.3.1 already closed the core review-authenticity gap — `Review` now has a `user` FK (`on_delete=SET_NULL`, nullable), an `is_verified_purchase` boolean computed at creation time from the user's delivered order history, `ProductReviewCreateView` now requires `IsAuthenticated` (not `AllowAny`), `ReviewCreateSerializer` no longer accepts a client-supplied `name` (it's populated server-side from the authenticated user), and a `unique_together = ("product", "user")` constraint prevents duplicate reviews. This epic builds moderation, reporting, images, and admin replies on top of that already-authenticated foundation — it does **not** need to re-solve authenticity.

**A real, load-bearing detail confirmed directly from the repo that Task 10.1.1.1 must handle correctly:** `ProductReviewCreateView._refresh_product_stats()` currently recomputes `Product.rating`/`reviews_count` via `product.reviews.aggregate(avg_rating=Avg("rating"), total=Count("id"))` — an aggregate over **every** review with no filtering at all. Once a moderation `is_approved` flag exists, this aggregate **must** be updated to only count approved reviews, or an unmoderated/rejected review would still silently pull the public-facing star rating around before a human ever looked at it.

---

## Phase 10.1 — Review Moderation & Richness

### Feature 10.1.1 — Moderation

---

#### Task 10.1.1.1 — Add `is_approved` flag with admin moderation queue

```
You are working in backend/shop/models.py, views.py, admin.py, and
backend/dashboard/ (serializers.py, views.py, urls.py). Assume Epic 1
Feature 1.3.1 is fully merged (Review has user/is_verified_purchase,
auth-required creation, one-per-user-per-product constraint).

CONTEXT
Every review currently goes live immediately and publicly the moment
it's created — there's no moderation step at all. Real storefronts
typically want at least a lightweight moderation gate, especially for
non-verified-purchase reviews, to catch spam/abuse/inappropriate
content before it's publicly visible, while ideally not adding friction
for the common, low-risk case of a genuine verified purchaser leaving a
normal review.

TASK
Add an `is_approved` field to `Review`, with configurable
auto-approval for verified purchases, update the public review list to
only show approved reviews, fix the rating-aggregate calculation to
only count approved reviews (per the CONTEXT note above — this is not
optional polish, it's a correctness requirement), and add an admin
moderation queue.

REQUIREMENTS
- Add to `Review` (backend/shop/models.py):
  `is_approved = models.BooleanField(default=False)`
- Add a settings flag controlling auto-approval behavior, matching the
  existing `python-decouple` `config()` pattern used throughout
  backend/core/settings/base.py:
  `REVIEW_AUTO_APPROVE_VERIFIED_PURCHASES = config("REVIEW_AUTO_APPROVE_VERIFIED_PURCHASES", default=True, cast=bool)`
- In `ProductReviewCreateView.perform_create()` (already setting
  `is_verified_purchase` per Epic 1 Task 1.3.1.5), also set
  `is_approved` at creation time based on the new setting:
  ```python
  def perform_create(self, serializer) -> None:
      product = self._get_product()
      is_verified = self._is_verified_purchase(self.request.user, product)
      auto_approve = is_verified and settings.REVIEW_AUTO_APPROVE_VERIFIED_PURCHASES
      serializer.save(
          product=product,
          user=self.request.user,
          is_verified_purchase=is_verified,
          is_approved=auto_approve,
      )
      self._refresh_product_stats(product)
  ```
  Import `settings` from `django.conf` at the top of shop/views.py if
  not already imported.
- **Fix `_refresh_product_stats()` to only aggregate approved reviews**
  (the correctness fix flagged in this document's header context):
  ```python
  def _refresh_product_stats(self, product: Product) -> None:
      stats = product.reviews.filter(is_approved=True).aggregate(
          avg_rating=Avg("rating"),
          total=Count("id"),
      )
      product.rating = round(stats["avg_rating"] or 0.0, 1)
      product.reviews_count = stats["total"] or 0
      product.save(update_fields=["rating", "reviews_count"])
  ```
- The PUBLIC review list (wherever `ReviewSerializer` is used to list a
  product's reviews for storefront display — locate this view/queryset
  in shop/views.py, likely part of the product detail endpoint or a
  dedicated review-list view) must filter to `is_approved=True` only —
  find that queryset and add `.filter(is_approved=True)` to it. A
  reviewer should still be able to see their OWN pending review on
  their own account/order history page if such a page exists (check
  whether one does; if not, this is out of scope for this task, don't
  build new customer-facing "my reviews" UI as a side effect).
- **Critical follow-through:** since moderation now means a review can
  become approved AFTER creation (an admin approving it later, per the
  moderation queue below), `_refresh_product_stats()` needs to be
  called AGAIN whenever a review's `is_approved` status changes via the
  admin moderation action, not just at creation time — otherwise a
  newly-approved review would never actually affect the public rating
  until some unrelated future review creation happened to trigger a
  recalculation. Wire this in as part of the moderation action below.
- Admin moderation queue: add an `AdminReviewViewSet` to
  backend/dashboard/, mirroring the `AdminCouponViewSet`/
  `AdminOrderViewSet` pattern already established in that app:
  ```python
  # dashboard/serializers.py
  from shop.models import Review

  class AdminReviewSerializer(serializers.ModelSerializer):
      product_name = serializers.CharField(source="product.name", read_only=True)
      user_email = serializers.CharField(source="user.email", read_only=True, default=None)

      class Meta:
          model = Review
          fields = [
              "id", "product", "product_name", "user", "user_email", "name",
              "rating", "headline", "comment", "is_verified_purchase",
              "is_approved", "created_at",
          ]
          read_only_fields = [
              "product", "user", "name", "rating", "headline", "comment",
              "is_verified_purchase", "created_at",
          ]  # admin can ONLY toggle is_approved through this endpoint, nothing else

  # dashboard/views.py
  class AdminReviewViewSet(viewsets.ModelViewSet):
      permission_classes = [IsAdminOrSuperuser]
      pagination_class = DashboardPagination
      serializer_class = AdminReviewSerializer
      filter_backends = [filters.SearchFilter, filters.OrderingFilter]
      search_fields = ["product__name", "user__email", "comment"]
      ordering_fields = ["created_at", "rating"]
      ordering = ["-created_at"]
      http_method_names = ["get", "patch", "head", "options"]  # no create/delete via admin API

      def get_queryset(self):
          qs = Review.objects.select_related("product", "user")
          status_param = self.request.query_params.get("status")
          if status_param == "pending":
              qs = qs.filter(is_approved=False)
          elif status_param == "approved":
              qs = qs.filter(is_approved=True)
          return qs

      def perform_update(self, serializer):
          instance = serializer.save()
          from shop.views import ProductReviewCreateView
          ProductReviewCreateView()._refresh_product_stats(instance.product)
  ```
  Note the `read_only_fields` deliberately locks down every field
  EXCEPT `is_approved` on this admin endpoint — an admin moderating
  reviews should only be able to approve/reject, never rewrite a
  customer's actual review text/rating (that would be a serious trust
  violation distinct from legitimate moderation). Instantiating
  `ProductReviewCreateView()` just to reuse its
  `_refresh_product_stats()` method is slightly awkward — consider
  extracting that method into a standalone function in
  `shop/services.py` (or similar) that BOTH `ProductReviewCreateView`
  and `AdminReviewViewSet` can call directly, which is the cleaner
  long-term fix; do this extraction now rather than leaving the
  awkward cross-view instantiation in place, since it's a small,
  contained refactor.
  Register the URL: `router.register(r"admin/reviews", views.AdminReviewViewSet, basename="admin-review")`
  in dashboard/urls.py.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/:
1. A review from a user WITH a verified purchase, with
   `REVIEW_AUTO_APPROVE_VERIFIED_PURCHASES=True` (default), is created
   with `is_approved=True` immediately.
2. A review from a user WITHOUT a verified purchase is created with
   `is_approved=False` regardless of the setting.
3. With `REVIEW_AUTO_APPROVE_VERIFIED_PURCHASES=False` (use
   `@override_settings`), even a verified-purchase review is created
   with `is_approved=False`.
4. `Product.rating`/`reviews_count` only reflect APPROVED reviews —
   create a mix of approved and pending reviews with different ratings
   and confirm the aggregate matches only the approved subset.
5. The public review list endpoint never includes an unapproved review.
Add tests to backend/dashboard/tests/test_views.py:
6. An admin can list reviews filtered by `?status=pending` and
   `?status=approved` correctly.
7. An admin PATCHing `is_approved=true` on a pending review updates it,
   AND correctly triggers a `Product.rating`/`reviews_count` recompute
   that now includes that review (this is the single most important
   test in this task — confirm the "approving later actually updates
   the public rating" follow-through works, not just that the flag
   itself flips).
8. An admin attempting to PATCH `comment` or `rating` through this
   endpoint has that change silently ignored (blocked by
   `read_only_fields`) — the review's actual content is unchanged.
9. A non-admin user gets 403 on all `AdminReviewViewSet` actions.
```

---

#### Task 10.1.1.2 — Review reporting ("flag as inappropriate")

```
You are working in backend/shop/models.py, views.py, serializers.py,
urls.py, and backend/dashboard/. Assume Task 10.1.1.1 is already
merged.

CONTEXT
There's no way for a customer browsing reviews to flag one as
inappropriate/spam/abusive — moderation (Task 10.1.1.1) only happens
proactively for NEW reviews via the auto-approve gate; nothing lets the
community surface a problem with an ALREADY-approved review after the
fact.

TASK
Add a `ReviewReport` model, a customer-facing "report this review"
endpoint, and an admin view of flagged reviews.

REQUIREMENTS
- Add to backend/shop/models.py:
  ```python
  class ReviewReport(models.Model):
      class Reason(models.TextChoices):
          SPAM = "spam", "Spam or fake"
          OFFENSIVE = "offensive", "Offensive content"
          IRRELEVANT = "irrelevant", "Not relevant to the product"
          OTHER = "other", "Other"

      review = models.ForeignKey(Review, on_delete=models.CASCADE, related_name="reports")
      reported_by = models.ForeignKey(
          settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True, related_name="review_reports"
      )
      reason = models.CharField(max_length=20, choices=Reason.choices)
      details = models.TextField(blank=True)
      created_at = models.DateTimeField(auto_now_add=True)
      resolved = models.BooleanField(default=False)

      class Meta:
          ordering = ["-created_at"]
          unique_together = ("review", "reported_by")

      def __str__(self):
          return f"Report on review #{self.review_id} — {self.reason}"
  ```
  Import `settings` from `django.conf` if not already imported.
  `unique_together = ("review", "reported_by")` prevents the same user
  spamming repeated reports on the same review (mirroring the
  duplicate-prevention pattern already established for
  `Review.unique_together = ("product", "user")` in Epic 1).
  Generate the migration.
- Add `ReviewReportSerializer`/`ReviewReportCreateView` in
  backend/shop/serializers.py / views.py:
  ```python
  class ReviewReportCreateSerializer(serializers.Serializer):
      reason = serializers.ChoiceField(choices=ReviewReport.Reason.choices)
      details = serializers.CharField(required=False, allow_blank=True, max_length=1000)


  class ReviewReportCreateView(APIView):
      """POST /api/reviews/{review_id}/report/"""
      permission_classes = [IsAuthenticated]

      def post(self, request, review_id):
          review = get_object_or_404(Review, pk=review_id)
          serializer = ReviewReportCreateSerializer(data=request.data)
          serializer.is_valid(raise_exception=True)
          report, created = ReviewReport.objects.get_or_create(
              review=review, reported_by=request.user,
              defaults=serializer.validated_data,
          )
          return Response(
              {"detail": "Report submitted." if created else "You have already reported this review."},
              status=status.HTTP_201_CREATED if created else status.HTTP_200_OK,
          )
  ```
  — `get_or_create` (rather than plain `create`) handles the "already
  reported" case gracefully as a no-op-with-200 rather than an ugly
  500 from the unique constraint, matching the exact pattern already
  established for `StockAlertSubscription` in Epic 4 Task 4.1.2.2.
  Register the URL: `path("reviews/<int:review_id>/report/", views.ReviewReportCreateView.as_view(), name="review-report"),`
  in backend/shop/urls.py.
- Add a threshold-based auto-hide safety net (optional but worth
  including): if a review accumulates 3+ UNRESOLVED reports, flip
  `is_approved=False` automatically (pulling it from public view pending
  admin review), rather than leaving a heavily-flagged review publicly
  visible indefinitely until an admin happens to check the queue:
  ```python
  # in ReviewReportCreateView.post(), after creating the report:
  if created:
      unresolved_count = review.reports.filter(resolved=False).count()
      if unresolved_count >= 3 and review.is_approved:
          review.is_approved = False
          review.save(update_fields=["is_approved"])
          from shop.services import refresh_product_stats  # per Task 10.1.1.1's extraction
          refresh_product_stats(review.product)
  ```
  (using the `refresh_product_stats` standalone function extracted in
  Task 10.1.1.1 — this is exactly the kind of second call site that
  motivated extracting it there instead of leaving it as a method only
  reachable via `ProductReviewCreateView`).
- Add flagged-reviews visibility to the admin moderation queue from
  Task 10.1.1.1: extend `AdminReviewSerializer` with a
  `report_count = serializers.SerializerMethodField()` (counting
  unresolved reports), and add a `?has_reports=true` filter option to
  `AdminReviewViewSet.get_queryset()` so admins can specifically triage
  reported content, not just newly-pending reviews.
- Add a simple "resolve report" admin action — a custom `@action` on
  `AdminReviewViewSet` or a dedicated `AdminReviewReportViewSet`, your
  call, but SOMETHING must let an admin mark reports as `resolved=True`
  after reviewing them (otherwise the `resolved=False` count used for
  auto-hiding above only ever grows and can never legitimately reset
  even after an admin has looked at a report and decided it's not
  actually a problem).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/:
1. An authenticated user can report a review; a duplicate report from
   the SAME user on the SAME review is a no-op (200, not a new row).
2. A review accumulating 3 unresolved reports is automatically
   `is_approved=False`'d, and the product's rating/count are correctly
   recomputed to exclude it.
3. An unauthenticated request to report a review is rejected (401).
Add tests to backend/dashboard/tests/:
4. Admin can filter the review queue by `?has_reports=true` and see the
   correct `report_count` per review.
5. Admin resolving reports correctly updates their `resolved` status,
   and does NOT automatically re-approve the review (re-approval, if
   warranted, should be an explicit separate admin action via the
   existing `is_approved` toggle from Task 10.1.1.1, not an automatic
   side effect of resolving reports — a review could be legitimately
   both reported AND deserving of staying hidden).
```

---

#### Task 10.1.1.3 — Review image upload support

```
You are working in backend/shop/models.py, serializers.py, views.py.
Assume Task 10.1.1.1 is already merged.

CONTEXT
Reviews are currently text-only (`headline`, `comment`) — customers
have no way to attach photos, a common and valuable feature for
cosmetics specifically (showing actual swatch/application results,
which matters far more for makeup/skincare purchase decisions than for
most other product categories). `ProductImage` (backend/shop/models.py)
already establishes the `ImageField(upload_to="products/...")` pattern
this task should follow for consistency.

TASK
Add multi-image upload support to reviews, restricted to verified
purchasers only (per the backlog's explicit scoping).

REQUIREMENTS
- Add a `ReviewImage` model:
  ```python
  class ReviewImage(models.Model):
      review = models.ForeignKey(Review, on_delete=models.CASCADE, related_name="images")
      image = models.ImageField(upload_to="reviews/images/")
      created_at = models.DateTimeField(auto_now_add=True)

      def __str__(self):
          return f"Image for review #{self.review_id}"
  ```
  Generate the migration.
- Enforce a maximum image count per review (e.g. 5) — add validation
  either at the serializer layer (Task below) or via a `clean()`-style
  check; a serializer-layer check is simpler and sufficient here.
- Enforce file-type/size validation matching whatever pattern was
  established for other image uploads in this project (check for an
  existing image-upload validator — if Epic 3's product image handling
  or any other prior epic added a reusable file-validation utility,
  reuse it; if none exists, add a straightforward validator checking
  content-type is an actual image format (not just a `.jpg` extension —
  validate actual file content/MIME, following the same
  "don't trust file extensions alone" principle noted as a security gap
  in the original project review) and a reasonable max file size (e.g.
  5MB per image)).
- Update `ReviewCreateSerializer` to accept multiple image files:
  ```python
  class ReviewCreateSerializer(serializers.ModelSerializer):
      images = serializers.ListField(
          child=serializers.ImageField(), required=False, max_length=5, write_only=True,
      )

      class Meta:
          model = Review
          fields = ["rating", "headline", "comment", "images"]

      def validate_images(self, value):
          for image in value:
              validate_review_image(image)  # the validator discussed above
          return value
  ```
  Update `ProductReviewCreateView.perform_create()` to reject image
  uploads from non-verified purchasers BEFORE saving anything:
  ```python
  def perform_create(self, serializer) -> None:
      product = self._get_product()
      is_verified = self._is_verified_purchase(self.request.user, product)
      images = serializer.validated_data.pop("images", [])
      if images and not is_verified:
          raise serializers.ValidationError(
              {"images": "Only verified purchasers can attach photos to a review."}
          )
      auto_approve = is_verified and settings.REVIEW_AUTO_APPROVE_VERIFIED_PURCHASES
      review = serializer.save(
          product=product, user=self.request.user,
          is_verified_purchase=is_verified, is_approved=auto_approve,
      )
      for image in images:
          ReviewImage.objects.create(review=review, image=image)
      self._refresh_product_stats(product)
  ```
  Note `images` is popped from `validated_data` BEFORE `serializer.save()`
  since `Review` itself has no `images` field to save directly (it's a
  reverse FK relation, populated via separate `ReviewImage.objects.create()`
  calls after the review exists) — `ModelSerializer.save()` would error
  attempting to pass an unrecognized `images` kwarg to `Review.objects.create()`
  if left in `validated_data`, so explicitly popping it is necessary,
  not optional cleanup.
- Update `ReviewSerializer` (the read/display serializer) to include
  nested image URLs:
  ```python
  class ReviewImageSerializer(serializers.ModelSerializer):
      class Meta:
          model = ReviewImage
          fields = ["id", "image"]

  # on ReviewSerializer:
  images = ReviewImageSerializer(many=True, read_only=True)
  ```

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/:
1. A verified purchaser can submit a review WITH 1-5 images; all
   `ReviewImage` rows are created and correctly linked.
2. A non-verified-purchase user attempting to submit images gets a 400
   with the specific "only verified purchasers" message, and NO review
   or images are created at all (the validation must happen before ANY
   database write, not partially succeed).
3. Submitting more than 5 images is rejected by the `max_length`
   constraint.
4. Submitting a non-image file (e.g. a `.txt` or `.pdf` disguised with
   an image extension, or genuinely wrong content-type) is rejected by
   the file-validation logic.
5. `ReviewSerializer` output correctly includes nested image URLs for a
   review that has them, and an empty list for one that doesn't.
```

---

#### Task 10.1.1.4 — Admin reply to reviews

```
You are working in backend/shop/models.py, serializers.py, and
backend/dashboard/. Assume Task 10.1.1.1 is already merged.

CONTEXT
There's no way for the store to publicly respond to a review (a common
pattern — e.g. addressing a complaint, thanking a customer, correcting
a factual error a reviewer made about the product) — this needs to be
a store-official response, visually and structurally distinct from
customer reviews, not just another `Review` row.

TASK
Add a `store_response` field to `Review`, an admin-only way to set it,
and display it on the public review list.

REQUIREMENTS
- Add to `Review`:
  ```python
  store_response = models.TextField(blank=True)
  store_response_at = models.DateTimeField(null=True, blank=True)
  ```
  Generate the migration. Note this is deliberately a SIMPLE single
  text field (one response per review), not a separate model — a
  threaded back-and-forth conversation on a review is explicitly out
  of scope per the backlog ("Admin reply to reviews" — a single
  store-response field, matching the simplicity of most real e-commerce
  platforms' review-reply features, not a full comment thread).
- Extend `AdminReviewSerializer` (from Task 10.1.1.1) to allow the
  admin to WRITE `store_response` (unlike every other review field,
  which is deliberately read-only on that serializer per Task
  10.1.1.1's trust boundary — `store_response` is the one field an
  admin legitimately writes there since it's THEIR content, not the
  customer's):
  ```python
  class AdminReviewSerializer(serializers.ModelSerializer):
      ...
      class Meta:
          model = Review
          fields = [
              ..., "store_response", "store_response_at",
          ]
          read_only_fields = [
              "product", "user", "name", "rating", "headline", "comment",
              "is_verified_purchase", "created_at", "store_response_at",
          ]  # store_response is NOT in this list — it's the one writable content field
  ```
  Set `store_response_at` automatically when `store_response` changes
  from empty to non-empty (or is updated at all) — override
  `perform_update()` on `AdminReviewViewSet` (already being touched by
  Task 10.1.1.1's `is_approved` follow-through logic — extend that same
  override):
  ```python
  def perform_update(self, serializer):
      if "store_response" in serializer.validated_data:
          serializer.validated_data["store_response_at"] = timezone.now()
      instance = serializer.save()
      refresh_product_stats(instance.product)  # only actually matters if is_approved also changed, but harmless either way
  ```
  Import `timezone` from `django.utils` at the top of dashboard/views.py
  if not already imported.
- Add `store_response`/`store_response_at` to the public-facing
  `ReviewSerializer` (read-only there, obviously) so the storefront can
  display it.

REQUIREMENTS — frontend
- Admin review-moderation UI (from Task 10.1.1.1's admin page, if
  already built by this point, or as a small addition to it): add a
  "Reply" text area under each review with a save action, calling the
  `AdminReviewViewSet` PATCH endpoint with `store_response`.
- Storefront review display: when `store_response` is present, render
  it visually distinct from the customer's review content (e.g. an
  indented, differently-styled box labeled "Response from [Store Name]"
  with the `store_response_at` timestamp).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/:
1. An admin can set `store_response` on a review via PATCH, and
   `store_response_at` is automatically set to the current time.
2. Updating `store_response` again later updates `store_response_at` to
   the NEW time (not left stale from the first response).
3. An admin attempting to change `comment`/`rating` in the SAME request
   as setting `store_response` has only `store_response` applied (the
   other fields remain locked via `read_only_fields`, consistent with
   Task 10.1.1.1's trust boundary).
Add a test to backend/shop/tests/ confirming the public
`ReviewSerializer` correctly includes `store_response`/
`store_response_at` when present, and blank/`null` when absent.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 10.1.1.1 | is_approved flag + admin moderation queue | ☐ |
| 10.1.1.2 | Review reporting ("flag as inappropriate") | ☐ |
| 10.1.1.3 | Review image upload support | ☐ |
| 10.1.1.4 | Admin reply to reviews | ☐ |

Once Epic 10 is fully merged, the next epic to generate prompts for is
**Epic 11 — Wishlist**, which the master backlog's execution-order
notes flag as able to run in parallel with this one — both are
lower-risk, additive features that only depend on Epic 1 (core
stability) and Epic 3 (product/variant model) already being in place.
