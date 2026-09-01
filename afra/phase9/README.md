# Phase 9 — Group Sessions

## Implementation Prompts

### Prompt 1 — فیلدهای جدید `Session` + سرویس `create_group_session`

```
Goal: Add min_students and price_per_seat fields to Session, and a new
create_group_session service function allowing a teacher to open an
independent group class (no BookingRequest involved).

Before starting, read these files completely:
1. apps/sessions/models.py — Session's full current definition
2. apps/sessions/services.py — create_session_from_booking (the closest
   existing precedent for creating a Session, though this one has no
   booking)
3. apps/common/money.py (Phase 0) — Money/quantize_amount, for
   price_per_seat's precision
4. apps/teachers/selectors.py (Phase 2) — the verification__status=
   "APPROVED" gating pattern, since only an approved teacher should be
   able to create a group session

What to build:

a) In apps/sessions/models.py, add to Session:
   min_students = models.PositiveSmallIntegerField(null=True, blank=True)
   price_per_seat = models.DecimalField(max_digits=10, decimal_places=0, null=True, blank=True)
   Document in comments: both are only meaningful when
   session_type="GROUP"; null/blank for ONE_ON_ONE sessions, which
   continue to use booking.locked_price instead.
   Run makemigrations, review/rename.

b) In apps/sessions/services.py, add:

   @log_service_call
   def create_group_session(
       *, teacher: User, skill: TeacherSkill, start_time: datetime,
       end_time: datetime, capacity: int, min_students: int,
       price_per_seat: Decimal,
   ) -> Session:
       """Create an independent group session a teacher opens for
       enrollment — no BookingRequest involved, unlike
       create_session_from_booking."""
       if not hasattr(teacher, "teacher_profile") or \
          teacher.teacher_profile.verification.status != "APPROVED":
           raise PermissionDeniedError("Only approved teachers can create group sessions.")
       if capacity < 2:
           raise ApplicationError("Group sessions require a capacity of at least 2.", code="invalid_capacity")
       if min_students < 1 or min_students > capacity:
           raise ApplicationError("min_students must be between 1 and capacity.", code="invalid_min_students")
       if price_per_seat <= 0:
           raise ApplicationError("price_per_seat must be positive.", code="invalid_price")
       if end_time <= start_time:
           raise ApplicationError("end_time must be after start_time.", code="invalid_time_range")

       session = Session.objects.create(
           teacher=teacher, skill=skill, booking=None,
           session_type="GROUP", start_time=start_time, end_time=end_time,
           max_students=capacity, min_students=min_students,
           price_per_seat=price_per_seat,
       )

       from apps.sessions.tasks import send_session_reminder
       reminder_eta = start_time - _REMINDER_LEAD_TIME
       transaction.on_commit(
           lambda: send_session_reminder.apply_async(args=[str(session.id)], eta=reminder_eta)
       )
       return session

   Note: no Meeting is created here yet — confirm whether
   create_meeting_for_session (Phase 7) should also be called here for
   consistency with the 1:1 flow (group sessions need a video room
   too); if so, add that call following Phase 7's exact non-fatal
   pattern (a provider failure must not block group session creation,
   same principle as it doesn't block 1:1 session creation).

Files affected:
- apps/sessions/models.py
- apps/sessions/migrations/00xx_*.py (new)
- apps/sessions/services.py

Then write tests: tests/sessions/test_group_creation.py covering happy
path, unapproved-teacher rejection, invalid capacity/min_students/price
validation, and confirming a Meeting is created (or the deliberate
decision not to, whichever was chosen and documented).

Acceptance Criteria:
- pytest tests/sessions/test_group_creation.py -v passes completely
- python manage.py makemigrations --check --dry-run is empty

Verification Steps:
1. pytest tests/sessions/test_group_creation.py -v
2. python manage.py makemigrations --check --dry-run
3. python manage.py migrate
4. git diff --stat
```

### Prompt 2 — Discovery: `list_group_sessions` + Endpointهای Create/List/Detail

```
Goal: Expose group session creation, marketplace discovery, and detail
viewing via the API.

Before starting, read apps/sessions/selectors.py, views.py, urls.py,
serializers.py in full, plus apps/teachers/selectors.py's
verification__status="APPROVED" filter pattern from Phase 2.

What to build:

a) apps/sessions/selectors.py, add:
   def list_group_sessions(*, skill_name=None, date_from=None) -> QuerySet[Session]:
       """Publicly discoverable group sessions: SCHEDULED, teacher
       APPROVED (Phase 2 gating, same principle as
       apps.teachers.selectors.list_teachers), not yet full."""
       qs = Session.objects.filter(
           session_type="GROUP", status="SCHEDULED",
           teacher__teacher_profile__verification__status="APPROVED",
           enrolled_count__lt=F("max_students"),
       ).select_related("teacher", "skill")
       if skill_name:
           qs = qs.filter(skill__name__icontains=skill_name)
       if date_from:
           qs = qs.filter(start_time__gte=date_from)
       return qs

b) apps/sessions/serializers.py: add GroupSessionCreateSerializer
   (teacher-facing input: skill_id, start_time, end_time, capacity,
   min_students, price_per_seat) and confirm SessionSerializer already
   exposes price_per_seat/min_students for GROUP sessions (add if
   missing).

c) apps/sessions/views.py, add:
   CreateGroupSessionView (POST, IsAuthenticated+IsTeacher, calls
   services.create_group_session)
   GroupSessionListView (GET, AllowAny, paginated, calls
   selectors.list_group_sessions, with skill/date_from query params
   via filterset or manual query param parsing — check whether
   SessionFilter, already used by StudentSessionListView, can be
   reused/extended for this)
   GroupSessionDetailView (GET, AllowAny — public discovery, matching
   apps.teachers's public profile pattern from earlier phases)

d) apps/sessions/urls.py: wire the three new routes.

Files affected:
- apps/sessions/selectors.py
- apps/sessions/serializers.py
- apps/sessions/views.py
- apps/sessions/urls.py

Then write tests: tests/sessions/test_group_discovery.py — a PENDING
teacher's group session doesn't appear; a full session doesn't appear;
a cancelled session doesn't appear; skill/date filters work correctly;
CreateGroupSessionView rejects a student caller (403).

Acceptance Criteria:
- pytest tests/sessions/test_group_discovery.py -v passes completely

Verification Steps:
1. pytest tests/sessions/test_group_discovery.py -v
2. git diff --stat
```

### Prompt 3 — تعمیم `refund_payment`/`release_payment_to_teacher` برای مسیر `enrollment`

```
Goal: Generalize the two payment functions that currently hard-require
payment.booking, so they work correctly for enrollment-based Group
payments too, without changing any existing 1:1-path behavior.

Before starting, read apps/payments/services.py's full refund_payment
and release_payment_to_teacher functions, with special attention to
every line reading payment.booking, and apps/payments/models.py's
Payment.enrollment field and its docstring.

What to build:

1. Add a shared helper in apps/payments/services.py:
   def _resolve_payment_session_and_start(payment: Payment) -> tuple[Session | None, datetime]:
       """Return (session, scheduling_start_time) regardless of
       whether this payment originated from a 1:1 booking or a group
       session enrollment — the two payment origins this project
       currently supports."""
       if payment.booking is not None:
           session = getattr(payment.booking, "session", None)
           return session, payment.booking.requested_start
       if payment.enrollment is not None:
           session = payment.enrollment.session
           return session, session.start_time
       raise ApplicationError(
           "This payment has no associated booking or enrollment.",
           code="payment_missing_context",
       )

2. In refund_payment: replace
       booking = payment.booking
       if booking is None:
           raise ApplicationError("This payment has no associated booking.")
       lead_time_hours = (booking.requested_start - timezone.now()).total_seconds() / 3600
   with:
       _session, start_time = _resolve_payment_session_and_start(payment)
       lead_time_hours = (start_time - timezone.now()).total_seconds() / 3600
   (Keep every other line of this function — the refund-percentage
   calculation, the gateway refund call from Phase 1, everything else
   — completely unchanged.)

3. In release_payment_to_teacher: replace
       booking = payment.booking
       session = getattr(booking, "session", None) if booking is not None else None
   with:
       session, _start_time = _resolve_payment_session_and_start(payment)
   (Keep every subsequent line — the no-show check, the dispute check,
   the payout provider call from Phase 1 — completely unchanged.)

Files affected:
- apps/payments/services.py (only these two functions plus the new
  helper)

Then run the FULL existing payments test suite to confirm zero
regression on the 1:1 path (this is the most important verification in
this prompt), then add new tests:
- refund_payment on an enrollment-based HELD_IN_ESCROW Payment (no
  booking) correctly computes lead time from the session's start_time
  and processes a refund
- release_payment_to_teacher on an enrollment-based Payment correctly
  resolves the session for the no-show check
- A Payment with neither booking nor enrollment set raises
  ApplicationError code "payment_missing_context" (a defensive/
  should-never-happen case, but confirm it fails safely rather than
  crashing with an AttributeError)

Acceptance Criteria:
- pytest tests/payments/ -v passes completely, including full
  regression on every existing 1:1-path test
- The new enrollment-path tests pass

Verification Steps:
1. pytest tests/payments/ -v
2. git diff --stat
```

### Prompt 4 — `enroll_in_group_session` + شاخه‌بندی Gateway Callback

```
Goal: Build the group-enrollment payment flow — creating a Payment tied
to a SessionEnrollment (reusing join_session's existing seat-reservation
logic) — and extend Phase 1's gateway callback success/failure handlers
to branch correctly between the 1:1 and group paths.

Before starting, read these files completely:
1. apps/sessions/services.py — join_session's full current
   implementation (do not change it — this prompt calls it as-is)
2. apps/payments/services.py — create_payment_for_booking (Phase 1) as
   the structural reference for Payment creation, and
   handle_gateway_callback_success/handle_gateway_callback_failed's
   full current 1:1-specific logic (Phase 1)
3. apps/payments/models.py — Payment.enrollment field
4. config/settings/base.py — the existing
   PAYMENT_CHECKOUT_TIMEOUT_MINUTES setting (Phase 1) pattern, for this
   prompt's new GROUP_ENROLLMENT_PAYMENT_TIMEOUT_MINUTES setting

What to build:

a) In apps/sessions/services.py, add:
   @log_service_call
   def enroll_in_group_session(*, session_id: str, student: User) -> "Payment":
       """Reserve a seat (via the existing join_session) and create the
       PENDING Payment for it — the student then follows the exact
       same checkout flow (POST /api/payments/{id}/checkout/) as a 1:1
       booking, per Phase 1's payment architecture, unchanged."""
       enrollment = join_session(session_id=session_id, student=student)
       from apps.payments.services import create_payment_for_group_enrollment
       return create_payment_for_group_enrollment(enrollment=enrollment)
   (join_session already raises the correct
   NotFoundError/ApplicationError/ConflictError for a full/closed/
   already-enrolled session — do not duplicate those checks here.)

b) In apps/payments/services.py, add:
   def create_payment_for_group_enrollment(*, enrollment: "SessionEnrollment") -> Payment:
       session = enrollment.session
       payment = Payment.objects.create(
           student=enrollment.student, teacher=session.teacher,
           enrollment=enrollment, amount=session.price_per_seat,
           currency="IRR", status="PENDING", provider="INTERNAL",
       )
       _record_payment_event(payment=payment, from_status="", to_status="PENDING", reason="Group enrollment payment created")
       return payment
   (Mirror create_payment_for_booking's exact PaymentEvent-recording
   convention.)

c) Extend handle_gateway_callback_success (Phase 1): after locating the
   Payment by gateway_reference, branch:
   if payment.booking is not None:
       <existing 1:1 logic, completely unchanged>
   elif payment.enrollment is not None:
       payment.status = "HELD_IN_ESCROW"
       payment.save(update_fields=["status"])
       _record_payment_event(payment=payment, from_status="PENDING", to_status="HELD_IN_ESCROW", reason="Group enrollment payment succeeded")
       # No teacher-approval step for group enrollment — it's
       # self-serve, matching join_session's existing direct-enrollment
       # semantics. Just notify the student their seat is confirmed.
       from apps.sessions.tasks import notify_group_enrollment_confirmed
       transaction.on_commit(lambda: notify_group_enrollment_confirmed.delay(str(enrollment.id)))

d) Extend handle_gateway_callback_failed similarly: for the enrollment
   branch, mark payment FAILED (same as before) AND free the seat —
   call apps.sessions.services.leave_session(session_id=..., student=...)
   (reusing the existing decrement logic exactly, rather than
   duplicating the F("enrolled_count") - 1 update inline).

e) Add notify_group_enrollment_confirmed task in apps/sessions/tasks.py,
   following this project's exact established notify_* pattern
   (Notification + independent email attempt, matching Phase 4's
   convention — add a minimal template
   apps/sessions/templates/sessions/emails/group_enrollment_confirmed.html/.txt).

f) Add settings:
   GROUP_ENROLLMENT_PAYMENT_TIMEOUT_MINUTES: int = config(
       "GROUP_ENROLLMENT_PAYMENT_TIMEOUT_MINUTES", default=30, cast=int
   )

Files affected:
- apps/sessions/services.py
- apps/payments/services.py
- apps/sessions/tasks.py
- apps/sessions/templates/sessions/emails/group_enrollment_confirmed.html/.txt (new)
- config/settings/base.py

Then write tests: tests/sessions/test_group_enrollment.py and extend
tests/payments/test_services.py covering:
- enroll_in_group_session happy path creates a SessionEnrollment and a
  PENDING Payment
- A full session correctly raises ConflictError before any Payment is
  created (join_session's existing check firing first)
- The full checkout->callback happy path (using
  InternalSimulatedGatewayProvider, matching Phase 1's test pattern)
  results in Payment HELD_IN_ESCROW and a confirmation notification
- The failure path correctly frees the seat (enrolled_count decrements,
  SessionEnrollment is removed) and sends no confirmation
- Confirm the existing 1:1 booking payment flow's tests still pass
  unchanged (full regression on handle_gateway_callback_success/failed)

Acceptance Criteria:
- pytest tests/sessions/ tests/payments/ -v passes completely, zero
  regression on the 1:1 path

Verification Steps:
1. pytest tests/sessions/test_group_enrollment.py -v
2. pytest tests/payments/ -v (full regression check)
3. git diff --stat
```

### Prompt 5 — Sweep برای پرداخت‌های Group معلق + Endpoint Checkout عمومی

```
Goal: Add the Celery sweep that expires stale PENDING group-enrollment
payments (freeing the seat), mirroring Phase 1's
reap_stale_pending_booking_requests exactly, and confirm/wire the
EnrollInGroupSessionView API endpoint that triggers
enroll_in_group_session.

Before starting, read apps/bookings/tasks.py's
reap_stale_pending_booking_requests in full (the exact structural
pattern to mirror) and apps/sessions/services.py's
enroll_in_group_session (Prompt 4).

What to build:

a) apps/payments/tasks.py, add:
   @shared_task(name="payments.reap_stale_pending_group_enrollment_payments")
   def reap_stale_pending_group_enrollment_payments() -> None:
       """Mirror of apps.bookings.tasks.reap_stale_pending_booking_requests
       for the group-enrollment payment path: a PENDING payment tied to
       an enrollment, past GROUP_ENROLLMENT_PAYMENT_TIMEOUT_MINUTES, is
       marked FAILED and its seat is freed."""
       cutoff = timezone.now() - timedelta(minutes=settings.GROUP_ENROLLMENT_PAYMENT_TIMEOUT_MINUTES)
       stale = Payment.objects.filter(
           status="PENDING", enrollment__isnull=False, created_at__lt=cutoff,
       ).select_related("enrollment", "enrollment__session")
       for payment in stale:
           with transaction.atomic():
               payment.status = "FAILED"
               payment.failure_reason = "Payment not completed within the allowed time window."
               payment.save(update_fields=["status", "failure_reason"])
               from apps.sessions.services import leave_session
               leave_session(session_id=str(payment.enrollment.session_id), student=payment.student)

   Register this task in Celery beat's schedule (find where
   reap_stale_pending_booking_requests is scheduled, e.g.
   config/settings/base.py's CELERY_BEAT_SCHEDULE, and add this new
   task at the same interval).

b) In apps/sessions/views.py, add:
   EnrollInGroupSessionView (POST, IsAuthenticated+IsStudent, calls
   services.enroll_in_group_session, returns the created Payment's id
   so the frontend can immediately call the existing
   POST /api/payments/{id}/checkout/ endpoint from Phase 1).
   Wire the route in urls.py.

Files affected:
- apps/payments/tasks.py
- config/settings/base.py (CELERY_BEAT_SCHEDULE entry)
- apps/sessions/views.py, urls.py

Then write tests:
- tests/payments/test_tasks.py: reap_stale_pending_group_enrollment_payments
  correctly expires a stale PENDING enrollment payment and frees the
  seat (enrolled_count decrements); a payment younger than the timeout
  is untouched; a HELD_IN_ESCROW payment is untouched (only PENDING
  ones are swept)
- tests/sessions/test_views.py: EnrollInGroupSessionView happy path
  returns a payment id usable with the existing checkout endpoint

Acceptance Criteria:
- pytest tests/payments/ tests/sessions/ -v passes completely

Verification Steps:
1. pytest tests/payments/test_tasks.py tests/sessions/ -v
2. git diff --stat
```

### Prompt 6 — لغو Group Session با بازپرداخت کامل + Sweep حداقل ظرفیت

```
Goal: Extend cancel_session so cancelling a group session refunds every
enrolled student in full, and add the minimum-capacity auto-cancellation
sweep.

Before starting, read apps/sessions/services.py's cancel_session in
full (Prompt 3's session-resolution helper and Phase 1's gateway refund
mechanics from apps/payments/services.py's refund_payment, as the
closest reference for "how a real gateway refund call is made" — though
this prompt needs a DIFFERENT refund percentage policy, not
refund_payment's lead-time-based one).

What to build:

a) In apps/payments/services.py, add:
   def refund_group_session_cancellation(*, session: "Session") -> None:
       """Full (100%) refund for every HELD_IN_ESCROW payment tied to
       this group session's enrollments — used when the *platform or
       teacher* cancels a group session (not when a student cancels
       their own seat, which has no equivalent self-service path in
       this Phase). Deliberately a separate function from
       refund_payment: that function's lead-time-percentage policy is
       specifically about a student's own late cancellation, which is
       not the situation here — this cancellation isn't the student's
       choice or fault, so the refund is always full regardless of
       timing. Idempotent: a payment already REFUNDED or not
       HELD_IN_ESCROW is silently skipped, so this is safe to call more
       than once for the same session (e.g. if a sweep somehow runs
       twice)."""
       payments = Payment.objects.filter(
           enrollment__session=session, status="HELD_IN_ESCROW",
       ).select_for_update()
       provider = get_gateway_provider()
       for payment in payments:
           with transaction.atomic():
               payment.refresh_from_db()
               if payment.status != "HELD_IN_ESCROW":
                   continue  # already handled by a concurrent/prior call
               try:
                   result = provider.refund(payment=payment, amount=Money(amount=payment.amount, currency=payment.currency))
               except NotImplementedError:
                   result = None  # matches refund_payment's Phase 1 precedent for gateways without automated refunds
               except GatewayError as exc:
                   logger.error("refund_group_session_cancellation: gateway refund failed for payment=%s: %s", payment.id, exc)
                   continue  # don't mark REFUNDED if the gateway call genuinely failed; leave for manual follow-up
               payment.status = "REFUNDED"
               payment.refund_amount = payment.amount
               payment.save(update_fields=["status", "refund_amount"])
               _record_payment_event(payment=payment, from_status="HELD_IN_ESCROW", to_status="REFUNDED", reason="Group session cancelled")

b) In apps/sessions/services.py's cancel_session: after the existing
   status/log/notify logic, add:
   if session.session_type == "GROUP":
       from apps.payments.services import refund_group_session_cancellation
       transaction.on_commit(lambda: refund_group_session_cancellation(session=session))
   (Confirm whether this should run inside on_commit as a direct call
   or as a Celery task — since it iterates multiple payments and makes
   real gateway calls, prefer dispatching it as a Celery task rather
   than a direct on_commit call blocking the request; if so, wrap it:
   create a thin Celery task in apps/payments/tasks.py that just calls
   this function, and dispatch that task via on_commit instead, for
   consistency with this project's established "slow/network-bound work
   goes through Celery" convention seen throughout every other Phase.)

c) In apps/sessions/tasks.py, add:
   @shared_task(name="sessions.cancel_undersubscribed_group_sessions")
   def cancel_undersubscribed_group_sessions() -> None:
       cutoff = timezone.now() + timedelta(hours=settings.GROUP_SESSION_MIN_CAPACITY_CHECK_HOURS_BEFORE)
       undersubscribed = Session.objects.filter(
           session_type="GROUP", status="SCHEDULED",
           start_time__lte=cutoff, start_time__gt=timezone.now(),
           min_students__isnull=False,
           enrolled_count__lt=F("min_students"),
       )
       for session in undersubscribed:
           from apps.sessions.services import cancel_session
           cancel_session(
               session_id=str(session.id), teacher=session.teacher,
               reason="Minimum enrollment was not reached.",
           )

   Add GROUP_SESSION_MIN_CAPACITY_CHECK_HOURS_BEFORE to settings
   (default e.g. 2, config-driven), and register this task on Celery
   beat's schedule.

Files affected:
- apps/payments/services.py, tasks.py (only the new thin dispatch task,
  if that design was chosen)
- apps/sessions/services.py
- apps/sessions/tasks.py
- config/settings/base.py

Then write tests:
- tests/payments/test_services.py: refund_group_session_cancellation
  refunds every HELD_IN_ESCROW enrollment payment fully; already-
  REFUNDED payments are skipped (idempotency); a gateway refund failure
  leaves that specific payment untouched (for manual follow-up) without
  blocking refunds to other students in the same session
- tests/sessions/test_services.py: cancel_session on a GROUP session
  dispatches the refund flow; on a ONE_ON_ONE session, behavior is
  completely unchanged from before this Phase (explicit regression
  check)
- tests/sessions/test_tasks.py: cancel_undersubscribed_group_sessions
  correctly cancels an undersubscribed session within the window and
  leaves an adequately-subscribed one alone

Acceptance Criteria:
- pytest tests/payments/ tests/sessions/ -v passes completely, zero
  regression on the 1:1 cancel_session path

Verification Steps:
1. pytest tests/payments/ tests/sessions/ -v
2. git diff --stat
```

### Prompt 7 — تعمیم `submit_review` برای Group Sessions

```
Goal: Remove Phase 3's group_review_not_supported restriction now that
group sessions are real, generalizing the eligibility check for
enrolled students rather than a single booking's student.

Before starting, read apps/reviews/services.py's submit_review in full
(Phase 3), specifically the current check that raises
ApplicationError(code="group_review_not_supported") for
session_type != "ONE_ON_ONE".

What to build:

In apps/reviews/services.py, modify submit_review's eligibility checks:

Replace:
   if session.session_type != "ONE_ON_ONE" or session.booking is None:
       raise ApplicationError("Reviews for group sessions are not yet supported.", code="group_review_not_supported")
   if student != session.booking.student:
       raise PermissionDeniedError(...)

with:
   if session.session_type == "ONE_ON_ONE":
       if session.booking is None or student != session.booking.student:
           raise PermissionDeniedError("You were not the student in this session.")
   elif session.session_type == "GROUP":
       if not session.enrollments.filter(student=student).exists():
           raise PermissionDeniedError("You were not enrolled in this session.")
   else:
       raise ApplicationError(f"Unknown session_type: {session.session_type}", code="unknown_session_type")

Keep every other eligibility check (status/outcome COMPLETED, unique
constraint, submission window) completely unchanged — this prompt only
touches the participant-identity check.

Files affected:
- apps/reviews/services.py

Then update tests/reviews/test_services.py: remove or repurpose the
existing group_review_not_supported test (since that behavior no
longer exists — repurpose it into a positive test: a group session
enrolled student CAN now submit a review), and add:
- An enrolled student can review a COMPLETED group session
- A non-enrolled user (authenticated, but never enrolled) cannot review
  a group session (PermissionDeniedError)
- The existing 1:1 eligibility tests all still pass unchanged
  (regression check)

Acceptance Criteria:
- pytest tests/reviews/ -v passes completely, zero regression on the
  1:1 review path

Verification Steps:
1. pytest tests/reviews/ -v
2. git diff --stat
```

### Prompt 8 — فرانت‌اند: Marketplace کشف Group Session + ثبت‌نام

```
Goal: Build the student-facing group session discovery/enrollment UI
and the teacher-facing group session creation form, reusing Phase 1's
existing checkout flow entirely.

Before starting, read src/features/sessions/api/sessions.api.ts,
src/features/bookings/pages/BookingPaymentPage.tsx (Phase 1, the
checkout-initiation pattern to reuse), and
src/features/teachers/pages/TeacherPublicProfilePage.tsx (for this
project's established public-listing page structure to mirror for the
new marketplace page).

What to build:

1. In sessions.api.ts, add: listGroupSessions(params),
   createGroupSession(data), enrollInGroupSession(sessionId) (returns
   { payment_id: string }).

2. src/features/sessions/pages/GroupSessionMarketplacePage.tsx (new):
   a searchable/filterable list of group sessions (skill, date),
   following this project's established list-page pattern (pagination,
   loading/empty states).

3. src/features/sessions/pages/GroupSessionDetailPage.tsx (new): shows
   session details, price_per_seat (via formatToman from Phase 0),
   remaining capacity, and an "Enroll" button that calls
   enrollInGroupSession, then reuses Phase 1's exact checkout-redirect
   pattern (navigate to the payment checkout flow with the returned
   payment_id — reuse BookingPaymentPage's redirect logic, either by
   extracting a shared hook/component from it or by routing to it with
   a payment_id param, whichever fits this project's existing routing
   conventions with the least duplication).

4. src/features/teachers/pages/CreateGroupSessionPage.tsx (new): a form
   for skill selection, start/end time (using JalaliCalendar from Phase
   6), capacity, min_students, price_per_seat (in Toman, converted to
   Rial via Phase 0's rialToToman inverse before sending to the API).

5. Wire all three routes in router.tsx/lazyPages.ts, and add a
   navigation link to the marketplace page from wherever this
   project's main navigation currently lists marketplace/teacher
   discovery (Navbar or a landing page CTA).

Files affected:
- src/features/sessions/api/sessions.api.ts
- src/features/sessions/pages/GroupSessionMarketplacePage.tsx (new)
- src/features/sessions/pages/GroupSessionDetailPage.tsx (new)
- src/features/teachers/pages/CreateGroupSessionPage.tsx (new)
- src/app/routes/router.tsx, lazyPages.ts
- src/shared/components/layout/Navbar.tsx (only the new nav link)

Then write component tests for each new page (list rendering, empty
state, enroll button triggering the checkout flow, create-form
validation and submission) following this project's established
testing conventions from every prior Phase.

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds
- A full manual walkthrough (dev environment, MEETING/PAYMENT_GATEWAY=
  internal) from browsing the marketplace to completing enrollment
  payment works end to end

Verification Steps:
1. npx vitest run
2. npm run build
3. Manual walkthrough as described above
4. git diff --stat
```

### Prompt 9 — بررسی نهایی End-to-End کامل و Regression جامع

```
Goal: Final comprehensive review of this entire Phase — one full
backend end-to-end test covering the complete group session lifecycle,
and a full regression pass confirming the 1:1 flow across every earlier
Phase is completely unaffected.

Before starting, review the diffs from Prompts 1-8.

What to build:

1. Add tests/sessions/test_e2e_group_lifecycle.py covering, using real
   service functions throughout:
   - An approved teacher creates a group session (capacity 3, min 2,
     price_per_seat set)
   - Two students enroll and complete payment (via
     InternalSimulatedGatewayProvider, following Phase 1/4's exact
     test pattern) → both payments HELD_IN_ESCROW, enrolled_count=2
   - A third student's payment fails at checkout → seat freed,
     enrolled_count stays 2
   - The session completes (mark_session_ongoing then
     mark_session_completed) → release_payment_to_teacher succeeds for
     both enrollment-based payments (proving Prompt 3's generalization
     actually works end to end, not just in isolated unit tests)
   - Both enrolled students can submit a review; a third,
     never-enrolled user cannot (proving Prompt 7's generalization)
   - Separately: create a second group session, enroll only 1 student
     (below min_students=2), advance time into the
     GROUP_SESSION_MIN_CAPACITY_CHECK_HOURS_BEFORE window, run
     cancel_undersubscribed_group_sessions → confirm the session is
     CANCELLED and the one enrolled student's payment is REFUNDED

2. Run the FULL backend test suite (pytest -x) and confirm zero
   regressions anywhere — this Phase touched shared payment functions
   (refund_payment, release_payment_to_teacher,
   handle_gateway_callback_success/failed) used by every prior Phase's
   1:1 flow, so this is the single most important verification step in
   the entire roadmap's final Phase.

3. Run the full frontend test suite (npx vitest run) and npm run build,
   confirming zero regressions there too.

4. Final grep sanity check:
   grep -rn "group_review_not_supported" apps/ (should return nothing
   in active code — only possibly in a repurposed/renamed test,
   confirm)
   grep -rn "payment.booking" apps/payments/services.py (confirm every
   remaining direct access is either inside
   _resolve_payment_session_and_start itself, or in a function
   genuinely specific to the 1:1 booking flow that has no group
   equivalent and legitimately doesn't need generalizing — report the
   list)

Files affected:
- tests/sessions/test_e2e_group_lifecycle.py (new)
- any regression fix found in step 2/3 (report even if none needed)

Acceptance Criteria:
- pytest -x (full backend suite) passes with zero failures
- npx vitest run (full frontend suite) passes with zero failures
- The new e2e group lifecycle test passes completely, proving payment,
  refund, review, and cancellation all correctly generalize to group
  sessions without breaking the 1:1 path

Verification Steps:
1. pytest tests/sessions/test_e2e_group_lifecycle.py -v
2. pytest -x -v (full backend suite — the definitive regression check
   for this entire roadmap)
3. npx vitest run (full frontend suite)
4. npm run build
5. grep -rn "group_review_not_supported" apps/
6. grep -rn "payment.booking" apps/payments/services.py
   (paste both outputs in your final summary)
7. git diff --stat (final summary of this entire Phase, and — since
   this is the roadmap's final Phase — a brief closing summary of the
   full 9-phase transformation is appropriate here)
```
