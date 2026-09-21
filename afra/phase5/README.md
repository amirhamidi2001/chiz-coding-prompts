# Phase 5 — Admin Dashboard سفارشی (React)

## Implementation Prompts

### Prompt 1 — افشای `is_staff` + `apps/adminapi` Skeleton + Dashboard Stats

```
Goal: Expose is_staff on the login/me user serializer (this unblocks
every frontend guard in this Phase), create the apps/adminapi app
skeleton, and build its first endpoint: a JSON dashboard-stats
endpoint reusing the exact counting logic from the previous roadmap's
Django Admin dashboard view.

Before starting, read these files completely:
1. apps/users/serializers.py — the exact serializer class used for
   login/`/me` responses (confirm its exact name and current `fields`
   list before editing)
2. Wherever the previous roadmap's AdminDashboardView (server-rendered
   Django Admin dashboard) lives — grep -rn "def admin_dashboard_view"
   apps/ to find it — read its exact four counting queries (pending
   teacher verifications, open disputes, pending payouts, review
   reports) to replicate identically, not re-derive from scratch
3. apps/common/permissions.py — confirm no custom IsStaff class
   already exists (it doesn't, per this project's current state) so
   DRF's built-in IsAdminUser is the correct, standard choice
4. Any existing app's apps.py (e.g. apps/reviews/apps.py) for this
   project's exact AppConfig pattern

What to build:

a) In apps/users/serializers.py, add "is_staff" to the fields list of
   whichever serializer produces the login/me response (confirmed in
   step 1 of your reading) — a single-line addition, nothing else in
   that serializer changes.

b) Create apps/adminapi/ app structure: __init__.py, apps.py (matching
   the exact AppConfig style found in step 4), urls.py, permissions.py,
   views/__init__.py.

c) In apps/adminapi/permissions.py:
   from rest_framework.permissions import IsAdminUser
   # Every view in apps.adminapi uses DRF's built-in IsAdminUser
   # (checks request.user.is_staff) directly. This module exists
   # purely so every admin view in this app imports permissions from
   # one place, making it trivial to audit that no admin endpoint
   # forgot to set permission_classes.

d) Create apps/adminapi/views/dashboard.py:
   class AdminDashboardStatsView(APIView):
       permission_classes = [IsAdminUser]
       def get(self, request):
           from apps.teachers.models import TeacherVerification
           from apps.payments.models import Dispute, PayoutLedgerEntry
           from apps.reviews.models import ReviewReport
           return Response({
               "pending_verifications": TeacherVerification.objects.filter(status="UNDER_REVIEW").count(),
               "open_disputes": Dispute.objects.filter(status="OPEN").count(),  # confirm exact status value from step 1's reading
               "pending_payouts": PayoutLedgerEntry.objects.filter(payout_status="PENDING_TRANSFER").count(),
               "flagged_reviews": ReviewReport.objects.count(),
           })
   (Use the EXACT same field/status values the existing Django Admin
   dashboard view uses — do not guess or slightly alter them; this is
   meant to be the identical logic in a new transport format, not a
   redesign.)

e) apps/adminapi/urls.py:
   urlpatterns = [
       path("dashboard/stats/", AdminDashboardStatsView.as_view()),
   ]
   Wire into config/urls.py: path("api/admin/", include("apps.adminapi.urls"))

f) Add "apps.adminapi" to INSTALLED_APPS in config/settings/base.py.

Files affected:
- apps/users/serializers.py (one line)
- apps/adminapi/__init__.py, apps.py, urls.py, permissions.py,
  views/__init__.py, views/dashboard.py (new)
- config/urls.py
- config/settings/base.py

Then write tests:
- tests/users/test_serializers.py (or wherever login/me is already
  tested): confirm is_staff now appears in the response for both a
  staff and non-staff user
- tests/adminapi/test_dashboard.py: a staff user gets 200 with correct
  counts (create known fixture data and assert exact numbers, matching
  the previous roadmap's dashboard test approach if one exists); a
  non-staff authenticated user gets 403; an unauthenticated request
  gets 401

Acceptance Criteria:
- pytest tests/adminapi/ tests/users/ -v passes completely
- python manage.py makemigrations --check --dry-run is empty (this app
  has no models, so this should trivially hold)

Verification Steps:
1. pytest tests/adminapi/ tests/users/ -v
2. python manage.py check
3. git diff --stat
```

### Prompt 2 — Endpointهای مدیریت کاربران (`Users`)

```
Goal: Build read-only user listing/detail and activate/deactivate
endpoints in apps.adminapi, adding the two missing tiny service
functions (activate/deactivate don't exist yet anywhere) following
this project's exact @log_admin_action convention.

Before starting, read:
1. apps/users/models.py — User's exact fields (is_active, role,
   date_joined or created_at, etc.)
2. apps/users/services.py — the file's existing structure, to add two
   new small functions in the same style
3. apps/common/logging.py — log_admin_action's exact signature
4. apps/teachers/views.py — VerificationStatusView or similar, as a
   precedent for pagination_class usage in this project

What to build:

a) In apps/users/services.py, add:
   @log_admin_action("user_deactivated", target_param="user_id")
   def deactivate_user(*, user_id: str, admin_user: "User") -> "User":
       """Deactivate a user account (blocks login; does not delete any
       data). Idempotent-safe: deactivating an already-inactive user
       is a harmless no-op, not an error — unlike the state-machine
       transitions elsewhere in this project (verification, disputes),
       this is a simple boolean toggle with no invalid-transition
       concept."""
       with transaction.atomic():
           user = User.objects.select_for_update().get(id=user_id)
           user.is_active = False
           user.save(update_fields=["is_active"])
       return user

   @log_admin_action("user_activated", target_param="user_id")
   def activate_user(*, user_id: str, admin_user: "User") -> "User":
       (Mirror the above, setting is_active = True.)

   Raise NotFoundError if user_id doesn't match any row, matching this
   project's established convention.

b) Create apps/adminapi/serializers.py (or views/users.py directly, if
   this project's other admin-facing serializers are typically inlined
   — check apps/adminapi/views/dashboard.py's precedent from Prompt 1,
   which had none; for a list/detail view a serializer is genuinely
   needed here, so create one):
   class AdminUserSerializer(serializers.ModelSerializer):
       class Meta:
           model = User
           fields = ["id", "email", "full_name", "role", "is_active", "is_staff", "is_email_verified", "date_joined"]
           read_only_fields = fields
   (Confirm the exact field name for account-creation timestamp on
   User — date_joined is Django's default AbstractUser field name;
   verify this project's User model actually uses that name rather
   than a custom created_at, before assuming.)

c) apps/adminapi/views/users.py:
   class AdminUserListView(generics.ListAPIView):
       permission_classes = [IsAdminUser]
       serializer_class = AdminUserSerializer
       pagination_class = StandardResultsPagination
       def get_queryset(self):
           qs = User.objects.all().order_by("-date_joined")
           role = self.request.query_params.get("role")
           if role:
               qs = qs.filter(role=role)
           search = self.request.query_params.get("search")
           if search:
               qs = qs.filter(Q(email__icontains=search) | Q(full_name__icontains=search))
           return qs

   class AdminUserDetailView(generics.RetrieveAPIView):
       permission_classes = [IsAdminUser]
       serializer_class = AdminUserSerializer
       queryset = User.objects.all()
       lookup_field = "id"

   class AdminUserActivateView(APIView):
       permission_classes = [IsAdminUser]
       def post(self, request, user_id):
           user = services.activate_user(user_id=user_id, admin_user=request.user)
           return Response(AdminUserSerializer(user).data)

   class AdminUserDeactivateView(APIView):
       (mirror, calling services.deactivate_user)

d) Wire routes in apps/adminapi/urls.py:
   users/, users/<uuid:user_id>/, users/<uuid:user_id>/activate/,
   users/<uuid:user_id>/deactivate/

Files affected:
- apps/users/services.py
- apps/adminapi/serializers.py (new)
- apps/adminapi/views/users.py (new)
- apps/adminapi/urls.py

Then write tests: tests/adminapi/test_users.py covering list (with
role/search filters), detail, activate, deactivate, permission
rejection for non-staff, and confirm each activate/deactivate call
creates exactly one AdminAuditLog row.

Acceptance Criteria:
- pytest tests/adminapi/ tests/users/ -v passes completely

Verification Steps:
1. pytest tests/adminapi/ -v
2. git diff --stat
```

### Prompt 3 — Endpointهای مدیریت تأیید مدرس (`Teachers`)

```
Goal: Build admin endpoints for the full teacher verification review
flow, calling the existing apps.teachers.services functions from the
previous roadmap's Phase 2 with zero new business logic.

Before starting, read:
1. apps/teachers/services.py — the exact current signatures of
   approve_teacher, reject_teacher, suspend_teacher, unsuspend_teacher
   (confirmed earlier: approve_teacher(*, verification_id, admin_user),
   reject_teacher and suspend_teacher additionally take a reason —
   confirm reject_teacher/suspend_teacher's exact parameter name for
   the reason argument before writing the view)
2. apps/teachers/models.py — TeacherVerification's fields, and
   TeacherDocument's fields (for the serializer)
3. apps/teachers/serializers.py — TeacherVerificationSerializer /
   TeacherDocumentSerializer, if these already exist from the previous
   roadmap's Phase 2 (they likely do, for the teacher's own
   self-service verification view) — REUSE them directly in
   apps.adminapi rather than building parallel serializers, if their
   field set is already suitable for admin viewing (it likely exposes
   everything needed); only build a new admin-specific serializer if
   the existing one is missing a field an admin genuinely needs (e.g.
   the teacher's name/email, which the self-service serializer
   wouldn't need since the teacher already knows who they are)

What to build:

a) If needed, extend or wrap the existing TeacherVerificationSerializer
   with an admin variant that adds teacher_name/teacher_email (via
   source="teacher_profile.user.full_name" etc.) — confirm first
   whether this is actually necessary before adding a new serializer
   class.

b) apps/adminapi/views/teachers.py:
   class AdminTeacherVerificationListView(generics.ListAPIView):
       permission_classes = [IsAdminUser]
       serializer_class = <the serializer from step a>
       pagination_class = StandardResultsPagination
       def get_queryset(self):
           qs = TeacherVerification.objects.select_related("teacher_profile__user").order_by("-created_at")
           status = self.request.query_params.get("status")
           if status:
               qs = qs.filter(status=status)
           return qs

   class AdminTeacherVerificationDetailView(generics.RetrieveAPIView):
       (list documents too — confirm whether the existing serializer
       already nests TeacherDocument, per the previous roadmap's design;
       if not, this view may need prefetch_related("documents") and a
       nested documents field)

   class AdminTeacherVerificationApproveView(APIView):
       permission_classes = [IsAdminUser]
       def post(self, request, verification_id):
           verification = teachers_services.approve_teacher(
               verification_id=verification_id, admin_user=request.user,
           )
           return Response(serializer_class(verification).data)

   class AdminTeacherVerificationRejectView(APIView):
       def post(self, request, verification_id):
           reason = request.data.get("reason", "")
           verification = teachers_services.reject_teacher(
               verification_id=verification_id, admin_user=request.user, reason=reason,
           )
           return Response(...)
       (Confirm reject_teacher actually requires a non-empty reason —
       if so, validate reason is present here and return 400 via a
       small input serializer rather than letting an empty string
       silently pass through to the service layer if that's not
       actually valid there either — check the service function's own
       validation first; if it already validates and raises
       ApplicationError for an empty reason, this view doesn't need to
       duplicate that check, just let the exception propagate through
       this project's standard exception-handling middleware.)

   class AdminTeacherVerificationSuspendView(APIView): (mirror reject,
       calling suspend_teacher)

   class AdminTeacherVerificationUnsuspendView(APIView): (mirror
       approve, calling unsuspend_teacher, no reason needed)

c) Wire routes in apps/adminapi/urls.py under
   teachers/verifications/... matching the pattern from Prompt 2.

CRITICAL: every one of these five views must contain ZERO business
logic beyond input extraction and calling the existing service
function — no status checks, no conditional branching on
verification.status, nothing beyond what's shown above. If you find
yourself writing an if statement checking business state in one of
these views, stop — that logic belongs in apps.teachers.services, not
here, and if it's missing there, that's a sign this view is wrong, not
that the check should be added here.

Files affected:
- apps/teachers/serializers.py (only if an admin-specific serializer
  addition was genuinely needed — report the decision)
- apps/adminapi/views/teachers.py (new)
- apps/adminapi/urls.py

Then write tests: tests/adminapi/test_teachers.py covering list
(filterable by status), detail (including documents), and all four
action endpoints (happy path + invalid-transition error propagation,
e.g. approving an already-APPROVED verification returns the same
ApplicationError-mapped response the previous roadmap's service-layer
test already covers — this view test should confirm the error reaches
the HTTP layer correctly, not re-test the state machine itself, which
is already covered by apps.teachers.services' own test suite).

Acceptance Criteria:
- pytest tests/adminapi/ -v passes completely
- Manual code review confirms zero business-logic branching in
  apps/adminapi/views/teachers.py

Verification Steps:
1. pytest tests/adminapi/ -v
2. git diff --stat
```

### Prompt 4 — Endpointهای Disputes + Payouts

```
Goal: Build admin endpoints for dispute resolution and payout transfer
confirmation, calling the existing apps.payments.services functions
(resolve_dispute, mark_payout_as_transferred) from the previous
roadmap with zero new business logic.

Before starting, read:
1. apps/payments/services.py — resolve_dispute's and
   mark_payout_as_transferred's exact current signatures (confirmed
   earlier to exist; re-read their full parameter lists before writing
   the input serializers, since resolve_dispute likely needs a
   resolution note and/or split percentages per the previous roadmap's
   Phase 8 design)
2. apps/payments/models.py — Dispute and PayoutLedgerEntry's exact
   fields
3. apps/payments/forms.py — DisputeResolutionForm, if it exists from
   the previous roadmap's Phase 8 Django-Admin custom resolve page —
   its field list is the authoritative source for what
   resolve_dispute actually needs as input; build a matching DRF
   serializer with the same fields, don't guess

What to build:

a) apps/adminapi/serializers.py, add:
   class AdminDisputeSerializer(serializers.ModelSerializer):
       class Meta:
           model = Dispute
           fields = [...]  # match Dispute's real fields
   class ResolveDisputeInputSerializer(serializers.Serializer):
       (fields matching DisputeResolutionForm's real fields exactly —
       confirm before writing)
   class AdminPayoutSerializer(serializers.ModelSerializer):
       class Meta:
           model = PayoutLedgerEntry
           fields = [...]
   class MarkPayoutTransferredInputSerializer(serializers.Serializer):
       bank_reference_number = serializers.CharField()

b) apps/adminapi/views/payments.py:
   class AdminDisputeListView(generics.ListAPIView):
       permission_classes = [IsAdminUser]
       serializer_class = AdminDisputeSerializer
       pagination_class = StandardResultsPagination
       def get_queryset(self):
           qs = Dispute.objects.select_related("payment").order_by("-created_at")
           status = self.request.query_params.get("status")
           if status:
               qs = qs.filter(status=status)
           return qs

   class AdminDisputeResolveView(APIView):
       permission_classes = [IsAdminUser]
       def post(self, request, dispute_id):
           input_serializer = ResolveDisputeInputSerializer(data=request.data)
           input_serializer.is_valid(raise_exception=True)
           dispute = payments_services.resolve_dispute(
               dispute_id=dispute_id, admin_user=request.user,
               **input_serializer.validated_data,
           )
           return Response(AdminDisputeSerializer(dispute).data)

   class AdminPayoutListView(generics.ListAPIView):
       (mirror the dispute list, filterable by payout_status)

   class AdminPayoutMarkTransferredView(APIView):
       def post(self, request, entry_id):
           input_serializer = MarkPayoutTransferredInputSerializer(data=request.data)
           input_serializer.is_valid(raise_exception=True)
           entry = payments_services.mark_payout_as_transferred(
               entry_id=entry_id, admin_user=request.user,
               bank_reference_number=input_serializer.validated_data["bank_reference_number"],
           )
           return Response(AdminPayoutSerializer(entry).data)

c) Wire routes in apps/adminapi/urls.py under payments/disputes/... and
   payments/payouts/....

Same CRITICAL constraint as Prompt 3: zero business logic in these
views beyond input validation and a single service call.

Files affected:
- apps/adminapi/serializers.py
- apps/adminapi/views/payments.py (new)
- apps/adminapi/urls.py

Then write tests: tests/adminapi/test_payments.py covering both list
views (with status filters), resolve (happy path + invalid-status
propagation), and mark-transferred (happy path + invalid-status
propagation), plus permission checks.

Acceptance Criteria:
- pytest tests/adminapi/ -v passes completely
- Manual code review confirms zero business-logic branching in
  apps/adminapi/views/payments.py

Verification Steps:
1. pytest tests/adminapi/ -v
2. git diff --stat
```

### Prompt 5 — Endpoint نظارت بر نظرات (`Reviews`)

```
Goal: Build admin endpoints for review moderation, calling the
existing apps.reviews.services.moderate_review function with zero new
business logic.

Before starting, read:
1. apps/reviews/services.py — moderate_review's exact signature
   (confirmed: moderate_review(*, review_id, admin_user, new_status))
2. apps/reviews/models.py — Review and ReviewReport's exact fields

What to build:

a) apps/adminapi/serializers.py, add:
   class AdminReviewSerializer(serializers.ModelSerializer):
       student_name = serializers.CharField(source="student.full_name", read_only=True)
       teacher_name = serializers.CharField(source="session.teacher.full_name", read_only=True)
       report_count = serializers.IntegerField(source="reports.count", read_only=True)
       class Meta:
           model = Review
           fields = ["id", "session", "student_name", "teacher_name", "rating", "comment", "status", "report_count", "created_at"]
           read_only_fields = fields

b) apps/adminapi/views/reviews.py:
   class AdminReviewListView(generics.ListAPIView):
       permission_classes = [IsAdminUser]
       serializer_class = AdminReviewSerializer
       pagination_class = StandardResultsPagination
       def get_queryset(self):
           qs = Review.objects.select_related("student", "session__teacher").order_by("-created_at")
           status = self.request.query_params.get("status")
           if status:
               qs = qs.filter(status=status)
           flagged_only = self.request.query_params.get("flagged") == "true"
           if flagged_only:
               qs = qs.filter(reports__isnull=False).distinct()
           return qs

   class AdminReviewModerateView(APIView):
       permission_classes = [IsAdminUser]
       def post(self, request, review_id):
           new_status = request.data.get("new_status")
           review = reviews_services.moderate_review(
               review_id=review_id, admin_user=request.user, new_status=new_status,
           )
           return Response(AdminReviewSerializer(review).data)

c) Wire routes in apps/adminapi/urls.py under reviews/....

Same CRITICAL constraint as Prompts 3-4.

Files affected:
- apps/adminapi/serializers.py
- apps/adminapi/views/reviews.py (new)
- apps/adminapi/urls.py

Then write tests: tests/adminapi/test_reviews.py covering list (status
+ flagged filters, confirming the distinct() correctly avoids
duplicate rows for a review with multiple reports), moderate (happy
path + invalid new_status propagation).

Acceptance Criteria:
- pytest tests/adminapi/ -v passes completely
- The flagged-filter distinct() regression test explicitly passes

Verification Steps:
1. pytest tests/adminapi/ -v
2. git diff --stat
```

### Prompt 6 — Endpointهای مدیریت محتوا (`Content`)

```
Goal: Build full admin CRUD endpoints for StaticPage and Article,
calling the existing apps.content.services functions from Phase 0 of
this roadmap with zero new business logic.

Before starting, read:
1. apps/content/services.py, models.py, serializers.py (this roadmap's
   Phase 0) — every function's exact signature
   (update_static_page, create_article, update_article,
   publish_article, unpublish_article, archive_article)

What to build:

a) apps/adminapi/serializers.py, add:
   class AdminStaticPageSerializer(serializers.ModelSerializer):
       class Meta:
           model = StaticPage
           fields = ["slug", "title", "body", "updated_at", "updated_by"]
           read_only_fields = ["updated_at", "updated_by"]

   class AdminArticleSerializer(serializers.ModelSerializer):
       class Meta:
           model = Article
           fields = ["id", "slug", "title", "excerpt", "body", "cover_image", "category", "status", "published_at", "author", "created_at"]
           read_only_fields = ["id", "status", "published_at", "author", "created_at"]

b) apps/adminapi/views/content.py:
   class AdminStaticPageListView(generics.ListAPIView):
       permission_classes = [IsAdminUser]
       serializer_class = AdminStaticPageSerializer
       queryset = StaticPage.objects.all()

   class AdminStaticPageUpdateView(APIView):
       permission_classes = [IsAdminUser]
       def patch(self, request, slug):
           page = content_services.update_static_page(
               slug=slug, title=request.data.get("title"),
               body=request.data.get("body"), admin_user=request.user,
           )
           return Response(AdminStaticPageSerializer(page).data)

   class AdminArticleListView(generics.ListAPIView):
       permission_classes = [IsAdminUser]
       serializer_class = AdminArticleSerializer
       pagination_class = StandardResultsPagination
       def get_queryset(self):
           qs = Article.objects.select_related("author").order_by("-created_at")
           status = self.request.query_params.get("status")
           if status:
               qs = qs.filter(status=status)
           return qs
       (Deliberately NOT filtered to PUBLISHED-only here, unlike the
       public content.api.ts from Phase 2 — admins need to see and
       manage DRAFT/ARCHIVED articles too; confirm this view uses a
       fresh queryset, NOT apps.content.selectors.list_published_articles,
       which is intentionally public-scoped and wrong for this
       admin-facing use case.)

   class AdminArticleCreateView(APIView):
       permission_classes = [IsAdminUser]
       def post(self, request):
           article = content_services.create_article(
               author=request.user, **request.data,
           )
           return Response(AdminArticleSerializer(article).data, status=201)

   class AdminArticleUpdateView(APIView):
       def patch(self, request, article_id):
           article = content_services.update_article(
               article_id=article_id, admin_user=request.user, **request.data,
           )
           return Response(...)

   class AdminArticlePublishView(APIView):
       def post(self, request, article_id):
           article = content_services.publish_article(article_id=article_id, admin_user=request.user)
           return Response(...)

   class AdminArticleUnpublishView, AdminArticleArchiveView: (mirror,
       calling unpublish_article/archive_article)

c) Wire routes in apps/adminapi/urls.py under content/pages/... and
   content/articles/....

Same CRITICAL constraint as Prompts 3-5.

Files affected:
- apps/adminapi/serializers.py
- apps/adminapi/views/content.py (new)
- apps/adminapi/urls.py

Then write tests: tests/adminapi/test_content.py covering static page
update, article list (including DRAFT/ARCHIVED visibility, unlike the
public API), create, update, publish/unpublish/archive (happy paths +
invalid-transition propagation, e.g. the published_at-preservation
behavior from Phase 0 should still hold when triggered through this
new admin endpoint — add an explicit test confirming this, since it's
this Phase's most subtle inherited behavior).

Acceptance Criteria:
- pytest tests/adminapi/ -v passes completely
- The published_at-preservation regression test (via this new
  endpoint) explicitly passes

Verification Steps:
1. pytest tests/adminapi/ -v
2. git diff --stat
```

### Prompt 7 — Endpoint `AuditLog` + بررسی نهایی Backend

```
Goal: Build a read-only AdminAuditLog listing endpoint, then do a
comprehensive final review of the entire apps.adminapi app to confirm
the zero-new-business-logic principle held throughout every prompt in
this Phase.

Before starting, read apps/common/models.py's AdminAuditLog fields in
full.

What to build:

a) apps/adminapi/serializers.py, add:
   class AdminAuditLogSerializer(serializers.ModelSerializer):
       actor_email = serializers.CharField(source="actor.email", read_only=True, default=None)
       class Meta:
           model = AdminAuditLog
           fields = ["id", "actor_email", "action", "target_type", "target_id", "details", "created_at"]
           read_only_fields = fields

b) apps/adminapi/views/audit_log.py:
   class AdminAuditLogListView(generics.ListAPIView):
       permission_classes = [IsAdminUser]
       serializer_class = AdminAuditLogSerializer
       pagination_class = StandardResultsPagination
       def get_queryset(self):
           qs = AdminAuditLog.objects.select_related("actor").order_by("-created_at")
           action = self.request.query_params.get("action")
           if action:
               qs = qs.filter(action=action)
           return qs

c) Wire the route: audit-log/

Files affected:
- apps/adminapi/serializers.py
- apps/adminapi/views/audit_log.py (new)
- apps/adminapi/urls.py

Then, as the final step of this prompt, do a comprehensive review:
1. Read every file under apps/adminapi/views/ created across Prompts
   1-7 in full, one more time.
2. Confirm, for each view that performs a write (approve, reject,
   suspend, unsuspend, resolve dispute, mark payout transferred,
   moderate review, update/create/publish/unpublish/archive
   content, activate/deactivate user): it calls exactly one existing
   service function from apps.teachers/apps.payments/apps.reviews/
   apps.content/apps.users, with no additional business-rule checks
   written directly in the view. Report this confirmation explicitly
   in your final summary — list every write endpoint and the exact
   service function it calls, as a definitive proof-of-reuse table.
3. Run the full backend test suite: pytest -x
4. Confirm python manage.py makemigrations --check --dry-run is empty
   (apps.adminapi has no models of its own, so this should hold
   trivially, but verify nothing was accidentally added).

Files affected (continued):
- none beyond what's listed above — this is a review-and-verify step

Acceptance Criteria:
- pytest tests/adminapi/ -v passes completely
- pytest -x (full backend suite) passes — zero regression across every
  domain this Phase touched
- The proof-of-reuse table in your final summary accounts for every
  write endpoint built in Prompts 2-7

Verification Steps:
1. pytest tests/adminapi/ -v
2. pytest -x -v (full backend suite)
3. python manage.py makemigrations --check --dry-run
4. git diff --stat (summary of the entire backend half of this Phase)
```

### Prompt 8 — فرانت‌اند: `RequireStaff` Guard + Shell و Layout پنل ادمین

```
Goal: Build the frontend foundation for the admin panel: expose
is_staff in the auth types, add a RequireStaff route guard mirroring
this project's existing RequireRole.tsx exactly, and build the
AdminLayout shell (sidebar navigation) with its route tree skeleton —
no actual data-driven pages yet, just the shell and empty placeholder
pages for each planned section.

Before starting, read these files completely:
1. src/features/auth/types/auth.types.ts — AuthUser's current shape
2. src/shared/components/guards/RequireRole.tsx — the exact structural
   pattern to replicate precisely for RequireStaff
3. src/app/routes/router.tsx — the full current route tree, to find
   the correct place to nest a new /admin/* subtree (likely a sibling
   of the existing AppLayout-wrapped routes, since AdminLayout is a
   distinct layout, not a variant of AppLayout)
4. src/shared/components/layout/AppLayout.tsx and Sidebar.tsx — the
   closest existing precedent for "a layout component with a sidebar
   nav," to match its structural/styling conventions for AdminLayout
   (reuse the same underlying UI primitives — Button, etc. — not a
   parallel design system)
5. src/app/routes/lazyPages.ts — the exact lazy-loading registration
   pattern

What to build:

1. In src/features/auth/types/auth.types.ts, add is_staff: boolean to
   AuthUser (matching Prompt 1's backend addition — confirm the exact
   field name matches).

2. Create src/shared/components/guards/RequireStaff.tsx:
   export default function RequireStaff() {
     const { user, isLoading } = useAuth();
     if (isLoading) return <PageLoader />;
     if (!user?.is_staff) return <Navigate to="/dashboard" replace />;
     return <Outlet />;
   }
   (Byte-for-byte structural mirror of RequireRole.tsx, just checking
   is_staff instead of role — do not add any extra logic here; this
   guard exists purely for UX (hiding the admin UI from non-staff
   users), never as the actual security boundary — every backend
   endpoint independently enforces IsAdminUser regardless of what this
   guard does, per this Phase's explicit architecture principle.)

3. Create src/features/admin/components/AdminLayout.tsx: a layout with
   a sidebar listing links to every planned admin section (Dashboard,
   Users, Teacher Verification, Disputes, Payouts, Reviews, Content
   (Pages/Articles), Contact, Audit Log), an <Outlet /> for the nested
   route content, and a "back to main site" link. Reuse this project's
   existing Sidebar/Button/layout primitives for visual consistency
   rather than building new ones.

4. Create thin placeholder page components for now (real
   implementation comes in Prompts 9-15): 
   src/features/admin/pages/AdminDashboardPage.tsx,
   AdminUsersPage.tsx, AdminTeacherVerificationPage.tsx,
   AdminDisputesPage.tsx, AdminPayoutsPage.tsx, AdminReviewsPage.tsx,
   AdminContentPagesPage.tsx, AdminArticlesPage.tsx,
   AdminContactPage.tsx, AdminAuditLogPage.tsx — each just rendering a
   heading with the section name for now, e.g.
   export default function AdminUsersPage() { return <h1>Users</h1>; }

5. In lazyPages.ts, register lazy imports for AdminLayout and all ten
   placeholder pages.

6. In router.tsx, add a new top-level route:
   {
     path: "/admin",
     element: <RequireStaff />,
     children: [
       {
         element: <Suspense fallback={<PageLoader />}><AdminLayout /></Suspense>,
         children: [
           { index: true, element: <AdminDashboardPage /> },
           { path: "users", element: <AdminUsersPage /> },
           { path: "teachers/verification", element: <AdminTeacherVerificationPage /> },
           { path: "payments/disputes", element: <AdminDisputesPage /> },
           { path: "payments/payouts", element: <AdminPayoutsPage /> },
           { path: "reviews", element: <AdminReviewsPage /> },
           { path: "content/pages", element: <AdminContentPagesPage /> },
           { path: "content/articles", element: <AdminArticlesPage /> },
           { path: "contact", element: <AdminContactPage /> },
           { path: "audit-log", element: <AdminAuditLogPage /> },
         ],
       },
     ],
   }
   (Adjust the exact Suspense/wrapping structure to match this
   project's established pattern for nested layouts, per your reading
   of router.tsx in step 3 — this is illustrative, not necessarily the
   literal final shape.)

7. Add a small, conditionally-rendered "Admin" link somewhere in the
   existing main Navbar (visible only when user?.is_staff is true),
   linking to /admin — check Navbar.tsx's existing conditional-link
   pattern (e.g. how a teacher-only or student-only link is already
   shown conditionally, if any exist) and match it.

Files affected:
- src/features/auth/types/auth.types.ts
- src/shared/components/guards/RequireStaff.tsx (new)
- src/features/admin/components/AdminLayout.tsx (new)
- src/features/admin/pages/*.tsx (10 new placeholder files)
- src/app/routes/lazyPages.ts
- src/app/routes/router.tsx
- src/shared/components/layout/Navbar.tsx

Then write tests:
- RequireStaff.test.tsx: redirects a non-staff user, renders children
  for a staff user, shows loading state while auth is loading
- AdminLayout.test.tsx: renders the sidebar with all ten section links

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds
- Manual check: logging in as a non-staff user and visiting /admin
  redirects to /dashboard; logging in as staff shows the admin shell
  with working navigation between all ten (currently placeholder)
  sections

Verification Steps:
1. npx vitest run
2. npm run build
3. Manual check as described above
4. git diff --stat
```

### Prompt 9 — فرانت‌اند: `Dashboard` و `Users`

```
Goal: Implement the real AdminDashboardPage (stats cards) and
AdminUsersPage (list with filters, activate/deactivate actions),
replacing their Prompt 8 placeholders.

Before starting, read:
1. apps/adminapi's dashboard and users endpoints (Prompts 1-2, backend)
   for exact response shapes
2. src/features/teachers/pages/BrowseTeachersPage.tsx — the closest
   existing precedent for "a filterable, paginated admin-style list
   page" to match structurally
3. src/shared/components/ui/ (Card, Badge, Table or equivalent list
   primitives, Button) — reuse these exactly, no new UI primitives

What to build:

1. Create src/features/admin/api/admin.api.ts (the shared API client
   for the whole admin feature, grown across this and later prompts):
   export const adminApi = {
     getDashboardStats(): Promise<AxiosResponse<DashboardStats>> { ... },
     listUsers(params): Promise<AxiosResponse<PaginatedResponse<AdminUser>>> { ... },
     getUser(id): Promise<AxiosResponse<AdminUser>> { ... },
     activateUser(id): Promise<AxiosResponse<AdminUser>> { ... },
     deactivateUser(id): Promise<AxiosResponse<AdminUser>> { ... },
   };
   (Match the exact URL paths from the backend Prompts 1-2.)

2. Create src/features/admin/types/admin.types.ts with
   DashboardStats, AdminUser interfaces matching the backend
   serializers exactly.

3. Implement AdminDashboardPage.tsx: four stat cards (pending
   verifications, open disputes, pending payouts, flagged reviews),
   each clickable, linking to the corresponding admin section
   (/admin/teachers/verification?status=UNDER_REVIEW, etc.) — reuse
   this project's existing Card component.

4. Implement AdminUsersPage.tsx: a paginated table/list (role filter,
   search box), each row showing email/full_name/role/is_active, with
   an Activate/Deactivate button per row (calling the respective
   mutation, with a confirmation dialog — reuse this project's
   existing Dialog/AlertDialog component for the confirmation, matching
   how a similarly consequential action elsewhere in this project, e.g.
   suspending a teacher, already confirms before acting).

5. Add the "admin" i18n namespace: src/shared/i18n/locales/fa/admin.json
   with keys for this prompt's two pages (register the namespace in
   src/shared/i18n/index.ts).

Files affected:
- src/features/admin/api/admin.api.ts (new)
- src/features/admin/types/admin.types.ts (new)
- src/features/admin/pages/AdminDashboardPage.tsx (replacing placeholder)
- src/features/admin/pages/AdminUsersPage.tsx (replacing placeholder)
- src/shared/i18n/locales/fa/admin.json (new)
- src/shared/i18n/index.ts
- src/test/mocks/handlers/admin.handlers.ts (new, growing across this
  and later prompts)

Then write tests for both pages (rendering, filtering, and the
activate/deactivate mutation flow with confirmation dialog).

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds

Verification Steps:
1. npx vitest run
2. npm run build
3. Manual check against a running backend
4. git diff --stat
```

### Prompt 10 — فرانت‌اند: `Teacher Verification` Review

```
Goal: Implement the real AdminTeacherVerificationPage: a list
(filterable by status) plus a detail/review view showing submitted
documents with approve/reject/suspend/unsuspend actions.

Before starting, read:
1. apps/adminapi's teachers endpoints (Prompt 3, backend) for exact
   response shapes, including nested documents
2. src/features/teachers/pages/TeacherVerificationPage.tsx (the
   teacher's own self-service submission page, from the previous
   roadmap's Phase 2) — for the existing status-label
   translation/badge pattern (statusLabels.ts helper) to reuse, not
   reinvent
3. src/features/teachers/lib/statusLabels.ts — reuse this exact helper
   for displaying verification status badges in the admin view too

What to build:

1. Extend admin.api.ts with listTeacherVerifications, getVerification,
   approveVerification, rejectVerification (with reason),
   suspendVerification (with reason), unsuspendVerification.

2. Implement AdminTeacherVerificationPage.tsx as a list view: status
   filter tabs/select (PENDING/UNDER_REVIEW/APPROVED/REJECTED/
   SUSPENDED), each row showing teacher name/email, status badge
   (reusing statusLabels.ts), submitted_at, linking to a detail view.

3. Build a detail sub-view (either a separate route
   /admin/teachers/verification/:id or an expandable row/modal — match
   whichever pattern this project's other "list + detail action" admin
   flows use, or default to a separate route for a cleaner URL/
   shareable-link UX) showing: teacher info, each submitted document
   (as a viewable/downloadable link, per TeacherDocument's file field),
   rejection_reason/suspension_reason if present, and action buttons:
   - UNDER_REVIEW: Approve, Reject (reject opens a small form requiring
     a reason, matching this project's established
     react-hook-form+Zod pattern for any form requiring free-text
     input, e.g. SubmitReviewForm)
   - APPROVED: Suspend (with reason)
   - SUSPENDED: Unsuspend
   - PENDING: no actions available (nothing submitted yet to review)

4. Add corresponding i18n keys to admin.json.

Files affected:
- src/features/admin/api/admin.api.ts
- src/features/admin/types/admin.types.ts
- src/features/admin/pages/AdminTeacherVerificationPage.tsx (replacing
  placeholder)
- src/features/admin/pages/AdminTeacherVerificationDetailPage.tsx (new,
  if a separate route was chosen)
- src/app/routes/lazyPages.ts, router.tsx (only if a new detail route
  was added)
- src/shared/i18n/locales/fa/admin.json
- src/test/mocks/handlers/admin.handlers.ts

Then write tests covering list filtering, detail rendering with
documents, and each of the four action flows (happy path + a mocked
backend error showing the correct message via getApiErrorMessage).

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds

Verification Steps:
1. npx vitest run
2. npm run build
3. Manual check against a running backend: approve/reject/suspend a
   real teacher verification through the new UI
4. git diff --stat
```

### Prompt 11 — فرانت‌اند: `Disputes` و `Payouts`

```
Goal: Implement the real AdminDisputesPage and AdminPayoutsPage.

Before starting, read:
1. apps/adminapi's payments endpoints (Prompt 4, backend) for exact
   response/input shapes (especially ResolveDisputeInputSerializer's
   real fields, confirmed against DisputeResolutionForm)
2. src/features/payments/lib/statusLabels.ts (from the previous
   roadmap's Phase 5) — reuse for payment/dispute status badges

What to build:

1. Extend admin.api.ts with listDisputes, resolveDispute, listPayouts,
   markPayoutTransferred.

2. AdminDisputesPage.tsx: list (status filter), each row showing
   payment amount (via formatToman from this project's established
   Phase 0 money-formatting utility — never a raw number), opened_by,
   created_at; a "Resolve" action opening a form matching
   ResolveDisputeInputSerializer's exact real fields (built with
   react-hook-form + Zod, consistent with every other form in this
   project).

3. AdminPayoutsPage.tsx: list (payout_status filter), each row showing
   teacher, payout_amount (formatToman), status; a "Mark as
   Transferred" action opening a small form for
   bank_reference_number, with a confirmation step (this moves real
   money bookkeeping state — treat it with the same care as the
   teacher-verification actions' confirmation dialogs).

4. Add i18n keys.

Files affected:
- src/features/admin/api/admin.api.ts
- src/features/admin/types/admin.types.ts
- src/features/admin/pages/AdminDisputesPage.tsx,
  AdminPayoutsPage.tsx (replacing placeholders)
- src/shared/i18n/locales/fa/admin.json
- src/test/mocks/handlers/admin.handlers.ts

Then write tests for both pages (list, filter, and the two action
flows).

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds
- Every monetary value on both pages uses formatToman, not a raw number
  (confirm with a quick grep of the new files for a bare {amount}
  interpolation)

Verification Steps:
1. npx vitest run
2. npm run build
3. grep -n "{.*amount.*}" src/features/admin/pages/AdminDisputesPage.tsx src/features/admin/pages/AdminPayoutsPage.tsx
   (manually confirm every match goes through formatToman)
4. git diff --stat
```

### Prompt 12 — فرانت‌اند: نظارت بر `Reviews`

```
Goal: Implement the real AdminReviewsPage.

Before starting, read apps.adminapi's reviews endpoint (Prompt 5,
backend), and src/features/reviews/components/StarRatingDisplay.tsx
(from the previous roadmap's Phase 3) — reuse this exact component for
rating display, don't rebuild it.

What to build:

1. Extend admin.api.ts with listReviews, moderateReview.

2. AdminReviewsPage.tsx: list (status filter + a "flagged only"
   toggle), each row showing student_name, teacher_name (reused
   StarRatingDisplay for rating), comment, report_count (highlighted
   if > 0), status; a "Publish"/"Hide" toggle action per row (calling
   moderateReview with the appropriate new_status).

3. Add i18n keys.

Files affected:
- src/features/admin/api/admin.api.ts
- src/features/admin/types/admin.types.ts
- src/features/admin/pages/AdminReviewsPage.tsx (replacing placeholder)
- src/shared/i18n/locales/fa/admin.json
- src/test/mocks/handlers/admin.handlers.ts

Then write tests for list rendering, the flagged filter, and the
moderate action.

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds

Verification Steps:
1. npx vitest run
2. npm run build
3. git diff --stat
```

### Prompt 13 — فرانت‌اند: مدیریت محتوا (`StaticPages` + `Articles`)

```
Goal: Implement the real AdminContentPagesPage (edit the five static
pages) and AdminArticlesPage (full article CRUD with publish workflow).

Before starting, read apps.adminapi's content endpoints (Prompt 6,
backend), and src/features/content/pages/StaticPageView.tsx /
ArticleDetailPage.tsx (Phase 2 of this roadmap) for the public-facing
equivalents' data shapes.

What to build:

1. Extend admin.api.ts with listStaticPages, updateStaticPage,
   listArticles (admin variant, unfiltered by status), createArticle,
   updateArticle, publishArticle, unpublishArticle, archiveArticle.

2. AdminContentPagesPage.tsx: a list of the five fixed static pages,
   each opening an edit form (title + body textarea) using
   react-hook-form, saving via updateStaticPage.

3. AdminArticlesPage.tsx: a paginated list (status + category filters,
   showing DRAFT/PUBLISHED/ARCHIVED all together, unlike the public
   list), a "New Article" button opening a create form (title, slug —
   auto-suggested from title but editable, excerpt, body, category,
   cover_image upload), and per-row actions (Edit, Publish/Unpublish,
   Archive) matching the state-machine transitions from Phase 0.
   Reflect the status with a colored badge.

4. Add i18n keys.

Files affected:
- src/features/admin/api/admin.api.ts
- src/features/admin/types/admin.types.ts
- src/features/admin/pages/AdminContentPagesPage.tsx,
  AdminArticlesPage.tsx (replacing placeholders)
- src/features/admin/components/ArticleEditorForm.tsx (new, shared
  between create and edit flows)
- src/shared/i18n/locales/fa/admin.json
- src/test/mocks/handlers/admin.handlers.ts

Then write tests: static page edit save flow; article list with
filters; create flow (including client-side slug validation matching
the backend's SlugField format rules from Phase 0); publish/unpublish/
archive actions, including a test confirming the UI correctly reflects
that publishing an already-PUBLISHED article is rejected (surfacing the
backend's invalid_article_status error via getApiErrorMessage).

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds

Verification Steps:
1. npx vitest run
2. npm run build
3. Manual check: create a DRAFT article, publish it, confirm it now
   appears on the public /articles page (Phase 2) and the homepage's
   LatestArticlesSection (Phase 4)
4. git diff --stat
```

### Prompt 14 — فرانت‌اند: `Contact Inbox` (بازاستفادهٔ کامل Phase 1) + `Audit Log`

```
Goal: Implement AdminContactPage, consuming the contact-admin endpoints
that already fully exist from Phase 1 of this roadmap (no new backend
work needed here), and AdminAuditLogPage.

Before starting, read:
1. apps.contact.views (Phase 1 of this roadmap) — confirm
   ContactMessageListView, ContactMessageDetailView,
   RespondToContactMessageView, ArchiveContactMessageView's exact
   existing URL paths and response shapes — this prompt calls these
   directly, NOT via apps.adminapi (since they already exist under
   /api/contact/messages/... and don't need to be duplicated into the
   adminapi namespace)
2. apps.adminapi's audit-log endpoint (Prompt 7, backend)

What to build:

1. Extend src/features/contact/api/contact.api.ts (from Phase 2 of
   this roadmap) with the admin-facing calls: listMessages (status
   filter), getMessage (triggers the backend's auto NEW->READ
   transition, per Phase 1's design — the frontend doesn't need to
   know this happens, it just calls GET), respondToMessage,
   archiveMessage. (These belong in the contact feature's own API
   file, not admin.api.ts, since they're genuinely part of the contact
   domain — the admin panel is just another consumer of that feature's
   API layer, matching this project's established feature-boundary
   conventions.)

2. AdminContactPage.tsx (in src/features/admin/pages/, even though it
   calls contact.api.ts): a list (status filter tabs: New/Read/
   Responded/Archived), each row showing name/email/subject/
   created_at; clicking a row opens the full message (this GET call is
   what triggers the NEW->READ transition server-side — the UI should
   simply reflect the returned status afterward, no special
   client-side handling needed); Respond and Archive buttons per the
   status-machine rules from Phase 1 (Archive only enabled for
   READ/RESPONDED, matching the backend's own rejection of archiving a
   NEW message).

3. Extend admin.api.ts with listAuditLog (action filter).

4. AdminAuditLogPage.tsx: a paginated, read-only list showing
   actor_email, action, target_type/target_id, created_at, with an
   expandable/tooltip view of the details JSON for each row, and an
   action-type filter dropdown.

5. Add i18n keys for both pages.

Files affected:
- src/features/contact/api/contact.api.ts
- src/features/admin/api/admin.api.ts
- src/features/admin/pages/AdminContactPage.tsx,
  AdminAuditLogPage.tsx (replacing placeholders)
- src/shared/i18n/locales/fa/admin.json
- src/test/mocks/handlers/contact.handlers.ts, admin.handlers.ts

Then write tests for both pages (list, filter, the contact
respond/archive flow, and audit log rendering).

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds

Verification Steps:
1. npx vitest run
2. npm run build
3. Manual check: submit a real contact message via the public /contact
   form (Phase 2), confirm it appears in AdminContactPage as NEW,
   opening it transitions it to READ
4. git diff --stat
```

### Prompt 15 — بررسی نهایی جامع End-to-End (کل Phase)

```
Goal: Final comprehensive review of this entire Phase — full regression
across backend and frontend, a complete manual walkthrough checklist,
and confirmation that every admin action correctly produces an
AdminAuditLog entry visible in the new Audit Log page itself (closing
the full loop).

Before starting, review the diffs from Prompts 1-14.

What to build:

1. Run the full backend test suite: pytest -x — confirm zero
   regressions across every domain this Phase touched (users, teachers,
   payments, reviews, content, contact, adminapi).

2. Run the full frontend test suite: npx vitest run, and npm run build.

3. Add one frontend integration test
   (src/features/admin/__tests__/adminE2E.test.tsx or matching this
   project's existing integration-test location convention) that:
   - Renders the app as a logged-in staff user
   - Navigates to /admin, confirms the dashboard stats render
   - Navigates to /admin/teachers/verification, approves a mocked
     UNDER_REVIEW verification
   - Navigates to /admin/audit-log, confirms a new entry for
     "teacher_verification_approved" appears (using MSW to return an
     updated audit log list after the mutation)

4. Final grep sweeps across all of src/features/admin/:
   grep -rln "ml-[0-9]\|mr-[0-9]\|text-left\|text-right" src/features/admin/
   grep -rlnP '#[0-9a-fA-F]{3,8}\b|rgb\(' src/features/admin/
   grep -rn "formatDate\b\|formatDateTime\b" src/features/admin/
   (all three should return nothing, per this project's established
   RTL/color-token/Jalali-date conventions from earlier roadmap phases)

5. Manual walkthrough checklist (perform directly): as a staff user,
   exercise every one of the ten admin sections at least once against
   a running backend seeded with realistic data — dashboard counts
   update correctly after actions, teacher approve/reject/suspend,
   dispute resolve, payout mark-transferred, review moderate, static
   page edit, article full lifecycle, contact respond/archive, and
   confirm the audit log shows every one of these actions.

Files affected:
- One new integration test file
- Any regression fix found in steps 1-2 (report even if none needed)

Acceptance Criteria:
- pytest -x (full backend suite) passes with zero failures
- npx vitest run (full frontend suite) passes with zero failures
- npm run build succeeds
- All three grep sweeps in step 4 return no results
- The new integration test passes
- The manual walkthrough in step 5 completes without any broken action

Verification Steps:
1. pytest -x -v
2. npx vitest run
3. npm run build
4. grep -rln "ml-[0-9]\|mr-[0-9]\|text-left\|text-right" src/features/admin/
5. grep -rlnP '#[0-9a-fA-F]{3,8}\b|rgb\(' src/features/admin/
6. grep -rn "formatDate\b\|formatDateTime\b" src/features/admin/
7. Manual full walkthrough as described in step 5 above
8. git diff --stat (final summary of this entire Phase — the largest
   in this roadmap)
```
