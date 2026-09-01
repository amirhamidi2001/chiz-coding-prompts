# Phase 8 — Admin Enhancements & Audit Log

## Implementation Prompts

### Prompt 1 — مدل `AdminAuditLog` + Decorator `log_admin_action`

```
Goal: Build the AdminAuditLog model and a log_admin_action decorator
mirroring apps.common.logging.log_service_call's exact structure and
conventions, without wiring it into any existing service function yet.

Before starting, read these files completely:
1. apps/common/logging.py — the full file, especially log_service_call's
   implementation, _SENSITIVE_NAME_MARKERS, and the redaction logic —
   log_admin_action must reuse this exact redaction approach, not
   reinvent it
2. apps/payments/models.py — PaymentEvent, as the append-only-model
   style reference
3. apps/teachers/services.py (Phase 2) — approve_teacher's exact
   signature (specifically confirming it takes admin_user as a keyword
   argument, since the decorator needs to extract it by name)

What to build:

1. Create apps/common/models.py (if this project's "common" app
   doesn't already have a models.py with existing content — check
   first; if apps/common currently has no models.py, this is the
   first model in it, which is fine since it's a natural home for a
   cross-cutting concern like this) or, if apps.common isn't a full
   Django app with migrations capability, create a new small app
   apps/audit/ instead — check whether apps.common is already in
   INSTALLED_APPS with its own migrations directory first, and follow
   whichever path is structurally correct for this project.

   class AdminAuditLog(models.Model):
       """Append-only record of state-changing actions taken by staff
       users against other users' data (teacher verification
       decisions, dispute resolutions, review moderation, manual
       payout confirmations). Complements, but does not replace,
       apps.common.logging.log_service_call's file-based structured
       logging — that log covers every service call from every user;
       this model exists specifically so staff actions are queryable
       and reviewable through Django Admin."""

       id = UUIDField(primary_key=True, default=uuid4, editable=False)
       actor = ForeignKey("users.User", on_delete=SET_NULL, null=True, related_name="admin_audit_logs")
       action = CharField(max_length=100, db_index=True)
       target_type = CharField(max_length=100)  # e.g. "TeacherVerification"
       target_id = CharField(max_length=64)      # stringified id, not a real FK (target spans multiple models)
       details = JSONField(default=dict, blank=True)
       created_at = DateTimeField(auto_now_add=True, db_index=True)

       class Meta:
           ordering = ["-created_at"]
           indexes = [models.Index(fields=["action", "created_at"])]

       def __str__(self) -> str:
           return f"{self.action} on {self.target_type}({self.target_id}) by {self.actor}"

2. In apps/common/logging.py, add:

   def log_admin_action(action: str, *, target_param: str = "verification_id"):
       """Decorator for admin-triggered service functions. Wraps a
       function whose kwargs include an `admin_user` (the acting
       staff member) and a target-id kwarg (named by `target_param`,
       e.g. "verification_id", "review_id", "dispute_id",
       "entry_id" — since different admin-action functions across
       this project name their target-id kwarg differently). After
       the wrapped function returns successfully, records one
       AdminAuditLog row. Records nothing if the function raises —
       only successful state changes are audited, never failed
       attempts, since a failed attempt didn't actually change
       anything and logging it as if it did would be misleading to
       anyone reviewing this log later."""

       def decorator(func):
           @functools.wraps(func)
           def wrapper(*args, **kwargs):
               result = func(*args, **kwargs)  # let exceptions propagate un-caught, un-logged-here
               admin_user = kwargs.get("admin_user")
               target_id = kwargs.get(target_param)
               if admin_user is not None and target_id is not None:
                   from apps.common.models import AdminAuditLog  # (or apps.audit.models, per your app-location decision)
                   # Reuse the exact same sensitive-field redaction
                   # logic already used by log_service_call, applied to
                   # a shallow copy of kwargs minus admin_user/the
                   # target id itself, for the `details` JSON field.
                   safe_details = {
                       k: ("***" if _is_sensitive_name(k) else v)
                       for k, v in kwargs.items()
                       if k not in ("admin_user", target_param)
                   }
                   AdminAuditLog.objects.create(
                       actor=admin_user,
                       action=action,
                       target_type=type(result).__name__,
                       target_id=str(target_id),
                       details=safe_details,
                   )
               return result
           return wrapper
       return decorator

   Extract the sensitive-name-check logic from log_service_call into a
   small shared helper (_is_sensitive_name(key: str) -> bool) if it
   isn't already factored out that way, so both decorators use
   identical redaction rules rather than two copies that could drift.

Files affected:
- apps/common/models.py or apps/audit/models.py + migrations (new,
  per your app-location decision — report which you chose and why)
- apps/common/logging.py

Do not apply this decorator to any actual service function yet — that
wiring happens in the next prompt.

Then write tests: tests/common/test_log_admin_action.py (or
tests/audit/, matching your app location) covering:
- A dummy decorated function that succeeds creates exactly one
  AdminAuditLog row with the correct actor/action/target_id
- A dummy decorated function that raises creates zero AdminAuditLog
  rows
- A sensitive-named kwarg (e.g. one containing "token") in the details
  is redacted to "***"
- Calling the decorated function without an admin_user kwarg at all
  (e.g. if reused incorrectly on a non-admin function) doesn't crash —
  it simply skips logging, since there's nothing meaningful to attribute

Acceptance Criteria:
- pytest tests/common/test_log_admin_action.py (or equivalent path) -v
  passes completely
- No existing service function has been modified in this prompt

Verification Steps:
1. pytest tests/common/test_log_admin_action.py -v (adjust path per
   your app-location decision)
2. python manage.py makemigrations --check --dry-run
3. python manage.py migrate
4. git diff --stat
```

### Prompt 2 — سیم‌کشی `log_admin_action` روی توابع ادمین‌محور موجود (Phase 2 و Phase 3)

```
Goal: Apply the log_admin_action decorator from Prompt 1 to the four
existing teacher-verification transition functions (Phase 2) and
moderate_review (Phase 3), so every existing admin action across the
project starts being recorded to AdminAuditLog.

Before starting, read these files completely:
1. apps/teachers/services.py — approve_teacher, reject_teacher,
   suspend_teacher, unsuspend_teacher (their exact current signatures,
   to confirm the target-id kwarg name is "verification_id" in every
   one, or note any inconsistency)
2. apps/reviews/services.py — moderate_review's exact signature
   (confirm its target-id kwarg is "review_id")
3. apps/common/logging.py (from Prompt 1) — log_admin_action's exact
   decorator signature

What to build:

In apps/teachers/services.py, add
@log_admin_action("teacher_verification_approved", target_param="verification_id")
above approve_teacher, and the equivalent for reject_teacher
("teacher_verification_rejected"), suspend_teacher
("teacher_verification_suspended"), unsuspend_teacher
("teacher_verification_unsuspended") — each with its own distinct
action string so they're individually filterable in Django Admin later,
even though they share the same decorator machinery.

In apps/reviews/services.py, add
@log_admin_action("review_moderated", target_param="review_id") above
moderate_review.

Import log_admin_action at the top of each file, following this
project's existing import-ordering convention.

Files affected:
- apps/teachers/services.py
- apps/reviews/services.py

Then run the existing test suites for both files and confirm nothing
broke (the decorator should be fully transparent to existing callers —
same return value, same exceptions on failure), then add new
assertions to the existing tests for each of these five functions:
after a successful call, exactly one AdminAuditLog row exists with the
correct action string and actor; after a failed call (e.g. invalid
status transition), zero new AdminAuditLog rows exist.

Acceptance Criteria:
- pytest tests/teachers/ tests/reviews/ -v passes completely, including
  the new AdminAuditLog assertions
- Every one of the five decorated functions is confirmed to log
  exactly on success, never on failure

Verification Steps:
1. pytest tests/teachers/ tests/reviews/ -v
2. git diff --stat
```

### Prompt 3 — واریز دستی: `mark_payout_as_transferred` + اکشن Admin

```
Goal: Complete Phase 1's manual-payout design by adding a service
function and Django Admin action for staff to confirm a
PayoutLedgerEntry has actually been transferred via bank, capturing a
reference number and auditing the action.

Before starting, read these files completely:
1. apps/payments/models.py — PayoutLedgerEntry's exact current fields
   (payout_status, transfer_reference, from Phase 1)
2. apps/payments/services.py — release_payment_to_teacher (Phase 1),
   to match this project's exact concurrency-safety pattern
   (select_for_update inside transaction.atomic) for the new function
3. apps/teachers/admin.py — TeacherVerificationAdmin's existing
   approve/reject admin-action pattern (Phase 2), as the closest
   precedent for "a bulk-safe admin action calling a service function
   with ApplicationError-to-message_user error handling," which this
   prompt's payout action should match structurally
4. apps/common/logging.py (Prompt 1) — log_admin_action

What to build:

a) In apps/payments/services.py, add:

   @log_admin_action("payout_marked_transferred", target_param="entry_id")
   def mark_payout_as_transferred(*, entry_id: str, admin_user: User, bank_reference_number: str) -> PayoutLedgerEntry:
       """Confirm a payout was actually transferred via bank, recording
       the reference number staff obtained from the bank transfer
       receipt. Only valid from PENDING_TRANSFER — this action cannot
       be reversed through this function (there's no
       'un-transfer'; a genuine transfer error requires a separate,
       explicit correction process outside this Phase's scope, not a
       silent status rollback here)."""
       with transaction.atomic():
           entry = PayoutLedgerEntry.objects.select_for_update().get(id=entry_id)
           if entry.payout_status != "PENDING_TRANSFER":
               raise ApplicationError(
                   "This payout is not pending transfer.",
                   code="invalid_payout_status",
               )
           entry.payout_status = "TRANSFERRED"
           entry.transfer_reference = bank_reference_number
           entry.save(update_fields=["payout_status", "transfer_reference"])
       return entry
   (NotFoundError if entry_id doesn't match any row, matching this
   project's established convention.)

b) In apps/payments/admin.py, extend PayoutLedgerEntryAdmin:
   Add a custom admin action mark_as_transferred that, matching
   TeacherVerificationAdmin's exact bulk-action-requiring-input
   pattern from Phase 2 (which already had to solve "a bulk action
   needs one piece of free-text input" — check exactly how that was
   resolved there and replicate the same approach here for
   consistency, whether that was a single-row-only constraint with a
   message_user prompt, or a custom intermediate form page), calls
   services.mark_payout_as_transferred for the selected entry/entries,
   catching ApplicationError via self.message_user(level=messages.ERROR).

Files affected:
- apps/payments/services.py
- apps/payments/admin.py

Then write tests:
- tests/payments/test_services.py: mark_payout_as_transferred happy
  path (status becomes TRANSFERRED, transfer_reference set, one
  AdminAuditLog row created); invalid-status rejection (e.g. already
  TRANSFERRED) raises ApplicationError and creates zero AdminAuditLog
  rows; a concurrency test (two simultaneous calls for the same entry,
  matching this project's established select_for_update test pattern
  from earlier phases) — exactly one succeeds

Acceptance Criteria:
- pytest tests/payments/ -v passes completely, including the new tests
- The concurrency test passes reliably

Verification Steps:
1. pytest tests/payments/ -v
2. python manage.py runserver, manually mark a PENDING_TRANSFER entry
   as transferred via /admin/, confirm status and reference update
3. git diff --stat
```

### Prompt 4 — UI عملیاتی Dispute Resolution در Admin (بدون دور زدن سرویس)

```
Goal: Build a custom Django Admin page for resolving a Dispute, calling
the existing apps.payments.services.resolve_dispute function (the same
one the authenticated API endpoint already uses), preserving this
project's explicit existing design principle that dispute resolution
must go through one single code path regardless of interface.

Before starting, read these files completely:
1. apps/payments/admin.py — DisputeAdmin's full current implementation
   and its docstring explaining why it's currently read-only
2. apps/payments/services.py — resolve_dispute's full signature and
   logic (what parameters it needs: split percentages, a resolution
   note, etc. — confirm exactly, don't guess)
3. apps/payments/views.py — the existing authenticated
   ResolveDisputeView (or equivalent) that already calls
   resolve_dispute, as the reference for exactly what input this
   function needs and how its response/errors are already handled
4. Any existing custom admin view in this project (search for
   `get_urls` overrides in any admin.py file: grep -rn "def get_urls"
   apps/*/admin.py) — if any precedent exists for a custom admin page
   in this codebase, follow its exact structural pattern (template
   location, admin_site.admin_view wrapping, permission checks); if
   none exists, this is the first one, and you should follow Django's
   own documented best-practice pattern for a custom ModelAdmin page
   (overriding get_urls, wrapping the view function with
   self.admin_site.admin_view(...) for authentication/permission
   enforcement, and rendering a minimal Django template extending
   admin/base_site.html)

What to build:

1. In apps/payments/admin.py, extend DisputeAdmin:

   def get_urls(self):
       urls = super().get_urls()
       custom_urls = [
           path(
               "<uuid:dispute_id>/resolve/",
               self.admin_site.admin_view(self.resolve_view),
               name="payments_dispute_resolve",
           ),
       ]
       return custom_urls + urls

   def resolve_view(self, request, dispute_id):
       dispute = get_object_or_404(Dispute, id=dispute_id)
       if dispute.status != "OPEN":  # or whatever the actual open-state
                                       # value is per this project's Dispute
                                       # model — confirm exactly
           messages.error(request, "This dispute is not open.")
           return redirect("..")  # back to the change list

       if request.method == "POST":
           form = DisputeResolutionForm(request.POST)
           if form.is_valid():
               try:
                   services.resolve_dispute(
                       dispute_id=str(dispute.id),
                       admin_user=request.user,
                       **form.cleaned_data,
                   )
                   messages.success(request, "Dispute resolved.")
                   return redirect("admin:payments_dispute_changelist")
               except ApplicationError as exc:
                   messages.error(request, str(exc))
       else:
           form = DisputeResolutionForm()

       return render(
           request,
           "admin/payments/dispute_resolve.html",
           {"form": form, "dispute": dispute, **self.admin_site.each_context(request)},
       )

   Add a "Resolve" link/button in the Dispute change form or change
   list (via list_display's a method returning a formatted link to
   this new URL, or via change_form_template overriding — pick
   whichever is simpler given this is the first custom admin page in
   the project) for OPEN disputes only.

2. Create apps/payments/forms.py (if it doesn't exist) with
   DisputeResolutionForm(forms.Form), whose fields exactly match
   resolve_dispute's actual parameters (confirmed from your reading of
   step 2 above — e.g. resolution_note, split_teacher_percentage, or
   whatever the real signature requires; do not invent fields that
   don't correspond to real parameters).

3. Create apps/payments/templates/admin/payments/dispute_resolve.html,
   extending admin/base_site.html, rendering the dispute's key details
   (payment amount, opened_by, reason) read-only and the form below it
   with a submit button — keep this genuinely minimal (a working,
   correctly-permissioned form page, not a polished custom UI; visual
   polish is explicitly out of scope for this Phase).

4. Ensure @log_admin_action is already applied to resolve_dispute in
   apps/payments/services.py (per this Phase's architecture doc,
   confirm this was done — if it wasn't already added in an earlier
   prompt of this Phase, add it now: @log_admin_action("dispute_resolved",
   target_param="dispute_id")).

Files affected:
- apps/payments/admin.py
- apps/payments/forms.py (new, or extended if it exists)
- apps/payments/templates/admin/payments/dispute_resolve.html (new)
- apps/payments/services.py (only if the log_admin_action decorator
  wasn't already applied to resolve_dispute)

Then write tests:
- tests/payments/test_admin.py (new, if this project has no existing
  admin-view test precedent — check first per the same caution noted
  in earlier Phases; if a precedent genuinely doesn't exist anywhere in
  this project, build the minimal Django test-client-based test needed
  here rather than skipping, since this is now a real operational admin
  page, unlike the earlier Phases' simple bulk actions):
  - A non-staff user gets redirected/denied access to the resolve URL
  - A staff user can GET the resolve form for an OPEN dispute
  - Submitting valid form data calls resolve_dispute and redirects with
    a success message; the underlying Dispute/Payment state changes
    exactly as calling resolve_dispute directly would
  - Submitting for an already-resolved dispute shows an error and does
    not call resolve_dispute again
  - After a successful resolution, exactly one new AdminAuditLog row
    exists

Acceptance Criteria:
- pytest tests/payments/ -v passes completely, including the new admin
  view tests
- The resolve page is confirmed to call the exact same
  apps.payments.services.resolve_dispute function the API endpoint
  uses (not a duplicated/parallel implementation)

Verification Steps:
1. pytest tests/payments/ -v
2. python manage.py runserver, manually resolve an OPEN dispute via
   /admin/, confirm the Dispute/Payment state updates correctly
3. git diff --stat
```

### Prompt 5 — بازبینی Rate Limiting برای Endpointهای حساس Phaseهای ۱، ۲، ۷

```
Goal: Apply this project's existing ScopedRateThrottle + "sensitive"
throttle_scope pattern (already used in apps/users/views.py) to the
three sensitive endpoints built in earlier Phases that don't yet have
it: PaymentCheckoutView (Phase 1), SubmitVerificationDocumentsView
(Phase 2), and SessionJoinUrlView (Phase 7).

Before starting, read these files completely:
1. apps/users/views.py — the exact existing pattern (throttle_classes
   = [ScopedRateThrottle], throttle_scope = "sensitive") on whichever
   three views there currently have it, to copy verbatim
2. config/settings/base.py — DEFAULT_THROTTLE_RATES' existing
   "sensitive": "5/min" value and its surrounding comment
3. apps/payments/views.py — PaymentCheckoutView's full current
   implementation
4. apps/teachers/views.py — SubmitVerificationDocumentsView's full
   current implementation
5. apps/meetings/views.py — SessionJoinUrlView's full current
   implementation

For each of these views, think through whether "5/min" (this project's
existing single sensitive rate) is actually appropriate, or whether a
distinct, dedicated throttle scope makes more sense:
- PaymentCheckoutView: a student might legitimately retry a checkout a
  few times (e.g. if the gateway redirect failed) — 5/min is probably
  fine, matching the existing sensitive scope.
- SubmitVerificationDocumentsView: a teacher submits this rarely (once,
  then only again after a rejection) — 5/min is generous enough,
  reuse the existing scope.
- SessionJoinUrlView: this one is different — a legitimate user might
  reasonably click "Join Session" more than 5 times in a minute if the
  first attempt failed to open correctly, retried, etc., especially
  right at a session's start when multiple students are all trying to
  join around the same moment. Consider whether this specific endpoint
  needs its own, more generous throttle scope (e.g. a new "meeting_join"
  scope at a higher rate like "20/min") rather than reusing "sensitive"
  as-is — make this judgment call explicitly and document your
  reasoning in a comment at the settings entry, rather than blindly
  applying the same "sensitive" scope to all three without
  consideration.

What to build:

1. In config/settings/base.py's DEFAULT_THROTTLE_RATES, add (if you
   decided SessionJoinUrlView needs its own scope per the reasoning
   above):
   "meeting_join": "20/min",
   with a comment explaining why this differs from "sensitive".

2. Add throttle_classes = [ScopedRateThrottle] and the appropriate
   throttle_scope to each of the three views, matching the exact
   import and attribute-setting pattern already used in
   apps/users/views.py.

Files affected:
- apps/payments/views.py
- apps/teachers/views.py
- apps/meetings/views.py
- config/settings/base.py (only if a new "meeting_join" scope was
  added)

Then write tests confirming each view is actually throttled: for each
of the three views, write a test that makes more requests than the
configured rate allows in rapid succession and confirms the
over-the-limit request returns HTTP 429 — matching however this
project's existing throttle tests for apps/users/views.py already
verify this (check for a precedent test first and follow its exact
approach, e.g. how it works around DRF's throttle cache/timing in
tests).

Acceptance Criteria:
- pytest tests/payments/ tests/teachers/ tests/meetings/ -v passes
  completely, including the new throttle tests
- All three views are confirmed to return 429 once their configured
  rate is exceeded

Verification Steps:
1. pytest tests/payments/ tests/teachers/ tests/meetings/ -v -k throttle
2. pytest tests/payments/ tests/teachers/ tests/meetings/ -v (full
   regression check for these three apps)
3. git diff --stat
```

### Prompt 6 — `AdminDashboardView` (خلاصهٔ عملیاتی)

```
Goal: Build a lightweight custom Django Admin dashboard page showing
key operational counters: teachers pending verification review, open
disputes, payouts pending manual transfer, and flagged/reported
reviews — a single starting point for the support team, not a full BI
tool.

Before starting, read:
1. apps/teachers/models.py — TeacherVerification.STATUS, to query the
   correct "pending review" count (likely UNDER_REVIEW, and possibly
   also PENDING — decide and document which states genuinely need
   staff attention vs which are just "not yet submitted by the
   teacher, nothing for staff to do yet")
2. apps/payments/models.py — Dispute's open-status value,
   PayoutLedgerEntry's PENDING_TRANSFER status
3. apps/reviews/models.py — ReviewReport, to count flagged/reported
   reviews needing attention
4. Whatever admin app structure was decided in Prompt 1 (apps.common
   or apps.audit) for where this dashboard view's code should live —
   place it alongside AdminAuditLogAdmin for consistency, or in
   whichever app makes the most sense given the app-location decision
   already made

What to build:

1. Create a view function/class (e.g. admin_dashboard_view) that
   queries:
   - TeacherVerification.objects.filter(status="UNDER_REVIEW").count()
   - Dispute.objects.filter(status="OPEN").count() (confirm the exact
     open-status value from your reading above)
   - PayoutLedgerEntry.objects.filter(payout_status="PENDING_TRANSFER").count()
   - ReviewReport.objects.count() (or, if reports should only count
     ones not yet acted upon and there's no "resolved" concept on
     ReviewReport per Phase 3's design, just the total — note this
     limitation if ReviewReport has no resolved/dismissed state to
     filter by, since Phase 3 designed it as purely advisory logging;
     report this as a known limitation of this dashboard rather than
     inventing a resolved-state field that doesn't exist)

2. Register this view in the Django Admin's URL configuration
   (following Prompt 4's precedent for wrapping a custom view with
   self.admin_site.admin_view(...) for auth, since this is the second
   custom admin page in the project — either attach it to one of the
   existing ModelAdmins' get_urls() or, if this project's admin.py
   files don't have a natural single "home" for a cross-cutting
   dashboard, register it directly against the default AdminSite
   instance in a project-level admin.py customization — check whether
   config/ or a top-level location already customizes django.contrib.admin.site
   anywhere, and follow that convention if it exists).

3. Create a minimal template
   templates/admin/dashboard.html extending admin/base_site.html,
   rendering the four counters as simple cards/numbers with links to
   the corresponding filtered admin change-list page for each (e.g.
   the "open disputes" counter links to
   /admin/payments/dispute/?status__exact=OPEN).

Files affected:
- wherever the dashboard view code lives (per your app-location
  decision, new file)
- templates/admin/dashboard.html (new)
- Django admin URL registration (wherever that lives in this project)

Then write tests: a test confirming the dashboard view requires staff
authentication (non-staff gets denied), and a test confirming the
counters are numerically correct given a known set of fixture data
(create 2 UNDER_REVIEW verifications, 1 PENDING one, 3 OPEN disputes, 1
RESOLVED one, etc., and assert the rendered/returned counts match only
the states that should count).

Acceptance Criteria:
- pytest (relevant test file) -v passes completely
- The counters are confirmed numerically correct against known fixture
  data, not just "the page loads without erroring"

Verification Steps:
1. pytest tests/ -v -k dashboard (or wherever this test file landed)
2. python manage.py runserver, visit the dashboard page manually,
   confirm the counts and links work
3. git diff --stat
```

### Prompt 7 — بررسی نهایی: پوشش کامل Audit Log + تست یکپارچه

```
Goal: Final review confirming every admin-triggered state change across
the entire project (not just the ones this Phase explicitly built) is
covered by AdminAuditLog, and one end-to-end integration test tying the
whole Phase together.

Before starting, review the diffs from Prompts 1-6.

What to build:

1. Comprehensive search: grep -rn "admin_user" apps/*/services.py to
   find EVERY function across the entire project (not just the ones
   named in this Phase's architecture doc) that takes an admin_user
   parameter — this is the authoritative list of "things a staff
   member can do that change someone else's data." Cross-reference
   this list against which functions now have @log_admin_action
   applied (grep -rn "@log_admin_action" apps/). Report any gap found
   — a function that takes admin_user but has no audit logging is a
   real, previously-invisible coverage hole this Phase was specifically
   meant to close; fix any gap found by adding the decorator (following
   Prompts 2-4's exact pattern) rather than just reporting it if it's a
   straightforward, low-risk addition; only report-without-fixing if
   fixing it would require design decisions beyond this Phase's
   established patterns.

2. Add tests/common/test_e2e_admin_audit.py (or matching your
   app-location decision) covering, end to end: a staff user
   approves a teacher verification, resolves a dispute, moderates a
   review, and marks a payout as transferred — confirm all four
   actions appear in AdminAuditLog with correct actor/action/timestamps,
   ordered correctly by created_at, and confirm a non-staff user
   attempting any of these (via the appropriate permission-denied path
   for each) produces zero AdminAuditLog rows.

3. Confirm README/documentation (if this project has an admin-facing
   ops doc anywhere) mentions the new dashboard URL and the
   AdminAuditLog's existence for future staff onboarding — add a short
   note if such a doc exists, skip if it doesn't (report either way).

Files affected:
- any service function found to need @log_admin_action per step 1
  (report the exact list, even if empty)
- tests/common/test_e2e_admin_audit.py (new)
- documentation (only if applicable — report either way)

Acceptance Criteria:
- pytest -x (full backend suite) passes, including the new e2e test
- The admin_user-taking-functions vs @log_admin_action-decorated-
  functions cross-reference from step 1 is reported explicitly in your
  final summary, along with what (if anything) was fixed

Verification Steps:
1. pytest tests/common/test_e2e_admin_audit.py -v (or matching path)
2. pytest -x -v (full backend suite)
3. grep -rn "admin_user" apps/*/services.py
4. grep -rn "@log_admin_action" apps/
   (paste both outputs in your final summary for cross-reference)
5. git diff --stat (final summary of this entire Phase)
```
