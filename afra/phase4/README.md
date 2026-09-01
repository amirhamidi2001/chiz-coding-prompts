# Phase 4 — Transactional Email Matrix

## Implementation Prompts

### Prompt 1 — ساخت `apps/emails`: `EmailLog` + `send_transactional_email` مرکزی + انتقال Base Template

```
Goal: Create a new "emails" Django app that centralizes transactional
email sending and logging, by extracting and generalizing the existing
_send_transactional_email helper currently private to apps/users/tasks.py.
This is a refactor/extraction, not a rewrite — the existing sending
logic and base HTML template are proven and should be reused as-is,
only relocated and made shareable.

Before starting, read these files completely:
1. apps/users/tasks.py — the full file, with absolute focus on
   _send_transactional_email (its exact multipart-email construction
   logic) and _first_name, and the _RETRY_DELAY constant and its
   comment explaining eager-mode vs real-Celery timing
2. apps/users/templates/users/emails/base.html — the full file, to
   understand exactly what it expects from callers (current_year,
   recipient_email context vars; the preheader/heading/content blocks)
3. apps/users/templates/users/emails/verify_email.html and .txt — as
   the reference example of how a child template extends base.html
4. apps/payments/models.py — PaymentEvent, as this project's
   established append-only-log pattern (which EmailLog should mirror)
5. config/settings/base.py — INSTALLED_APPS, and check for
   DEFAULT_FROM_EMAIL / EMAIL_BACKEND settings already present

What to build:

1. Create the app structure:
   apps/emails/
     __init__.py
     apps.py (matching this project's existing AppConfig style)
     models.py
     services.py
     tasks.py
     admin.py
     migrations/__init__.py
     templates/emails/base.html

2. Move apps/users/templates/users/emails/base.html to
   apps/emails/templates/emails/base.html verbatim — do not alter its
   content in this prompt (visual/localization changes are explicitly
   out of scope; that's Phase 5). Delete the original file after
   moving (don't leave a duplicate).

3. In apps/emails/models.py:

   class EmailLog(models.Model):
       """Append-only record of every transactional email send
       attempt, successful or not — the project's single source of
       truth for 'was this email actually sent, and when.' Mirrors
       apps.payments.models.PaymentEvent's append-only pattern."""

       id = UUIDField(primary_key=True, default=uuid4, editable=False)
       recipient = ForeignKey("users.User", null=True, blank=True, on_delete=SET_NULL,
                               related_name="email_logs")
       event_type = CharField(max_length=50, db_index=True)
       to_email = EmailField()
       STATUS = [("QUEUED", "Queued"), ("SENT", "Sent"), ("FAILED", "Failed")]
       status = CharField(max_length=10, choices=STATUS, default="QUEUED", db_index=True)
       error = TextField(blank=True)
       created_at = DateTimeField(auto_now_add=True)
       sent_at = DateTimeField(null=True, blank=True)

       class Meta:
           verbose_name = "email log"
           verbose_name_plural = "email logs"
           ordering = ["-created_at"]
           indexes = [models.Index(fields=["event_type", "status"])]

       def __str__(self) -> str:
           return f"{self.event_type} to {self.to_email} ({self.status})"

4. In apps/emails/services.py:

   def send_transactional_email(
       *, subject: str, template_base: str, context: dict,
       recipient: "User", event_type: str,
   ) -> EmailLog:
       """Render {template_base}.html + .txt and send a multipart
       transactional email, logging the attempt and its outcome to
       EmailLog.

       This is the one place in the project that should ever construct
       and send a transactional email — every domain's Celery task
       calls this rather than touching django.core.mail directly.

       Args:
           subject: Email subject line.
           template_base: Template path prefix, e.g.
               "bookings/emails/booking_accepted" — both .html and .txt
               must exist there.
           context: Template context; current_year and recipient_email
               are added automatically (matching the existing base.html
               contract).
           recipient: The User to email (used for the EmailLog FK and
               to read .email).
           event_type: A short identifier for EmailLog/observability,
               e.g. "BOOKING_ACCEPTED" — should match the corresponding
               Notification `type` value used by the same event where
               one exists, for easy cross-referencing.

       Raises:
           Whatever the underlying EmailMultiAlternatives.send() raises
           on failure — the caller (the Celery task in
           apps.emails.tasks) is responsible for catching this and
           retrying; this function's job is only to send-and-log, not
           to implement retry policy itself.
       """
       Move the exact rendering/EmailMultiAlternatives logic from
       apps.users.tasks._send_transactional_email here verbatim (same
       multipart construction, same automatic current_year/
       recipient_email context injection). Before attempting to send,
       create an EmailLog row with status="QUEUED". On successful
       send, update it to status="SENT", sent_at=timezone.now(). On any
       exception during rendering or sending, update it to
       status="FAILED", error=str(exc), then re-raise the exception
       (so the calling Celery task's own retry logic, built in the
       next step, still fires) — do not swallow the exception here.

5. In apps/emails/tasks.py:

   @shared_task(bind=True, max_retries=3, default_retry_delay=_RETRY_DELAY)
   def send_email_task(self, *, subject, template_base, context,
                        recipient_id, event_type) -> None:
       """Celery wrapper around services.send_transactional_email —
       looks up the recipient by id (Celery tasks should never receive
       live model instances as arguments) and retries on failure,
       exactly matching apps.users.tasks.send_verification_email's
       existing retry pattern."""
       Fetch the User by recipient_id (log a warning and return if not
       found, matching this project's existing convention). Call
       services.send_transactional_email(..., recipient=user, ...)
       inside a try/except that calls self.retry(exc=exc) on failure,
       exactly like send_verification_email does. Reuse (move, don't
       duplicate) the _RETRY_DELAY constant and its exact comment from
       apps.users.tasks — it's a general Celery-eager-mode concern, not
       users-specific, so it belongs here now.

6. In apps/emails/admin.py:
   Register EmailLog as a read-only ModelAdmin (list_display =
   [event_type, to_email, status, created_at, sent_at], list_filter =
   [status, event_type], search_fields = [to_email], and
   readonly_fields = every field, with has_add_permission returning
   False — this is a log, not something staff should hand-edit).

7. Add "apps.emails" to INSTALLED_APPS in config/settings/base.py, in
   the same style/position as other domain apps.

8. Run: python manage.py makemigrations emails
   Review and confirm it's a clean initial migration.

Files affected:
- apps/emails/__init__.py, apps.py, models.py, services.py, tasks.py,
  admin.py, migrations/__init__.py, migrations/0001_initial.py,
  templates/emails/base.html (all new)
- apps/users/templates/users/emails/base.html (deleted, moved)
- config/settings/base.py (INSTALLED_APPS only)

Do not touch apps/users/tasks.py yet — that refactor (to actually use
this new service) is the next prompt.

Acceptance Criteria:
- python manage.py makemigrations --check --dry-run is empty
- python manage.py migrate runs cleanly
- python manage.py check reports no errors related to the moved
  template path

Then write tests: tests/emails/test_services.py covering:
- send_transactional_email with a valid recipient/template creates an
  EmailLog with status="SENT" after success (use Django's test email
  backend — django.core.mail.outbox — to confirm an email was actually
  queued, and check its subject/recipient list)
- A rendering/sending failure (mock EmailMultiAlternatives.send to
  raise) results in an EmailLog with status="FAILED" and a non-empty
  error field, and the exception still propagates to the caller
- send_email_task (the Celery wrapper) retries up to 3 times on
  failure (mock the underlying service to always raise, assert
  self.retry was invoked the expected number of times — check how this
  project's existing send_verification_email retry behavior is already
  tested, if at all, and follow that same testing approach)

Verification Steps:
1. python manage.py makemigrations --check --dry-run
2. python manage.py migrate
3. pytest tests/emails/ -v
4. git diff --stat
```

### Prompt 2 — رفکتور `apps/users/tasks.py` برای استفاده از سرویس مرکزی

```
Goal: Refactor apps/users/tasks.py's send_verification_email and
send_password_reset_email to use the new apps.emails.services /
apps.emails.tasks infrastructure from Prompt 1, removing the
now-duplicated _send_transactional_email and _RETRY_DELAY. No change
to either function's external behavior or business logic (token
lookup/creation) — this is purely wiring to the new shared
infrastructure.

Before starting, read these files completely:
1. apps/users/tasks.py — the full current file (both email functions
   in full, plus the helpers being removed)
2. apps/emails/services.py, apps/emails/tasks.py (from Prompt 1) — the
   exact function signatures you'll now call

What to build:

In apps/users/tasks.py:

1. Remove _send_transactional_email and _RETRY_DELAY entirely (both
   now live in apps.emails).

2. Rewrite send_verification_email and send_password_reset_email to,
   instead of calling the now-removed local _send_transactional_email
   directly and handling their own try/except self.retry, dispatch to
   apps.emails.tasks.send_email_task with the same subject/
   template_base/context they were already building — since
   send_email_task already has its own retry policy internally, these
   two functions in apps/users/tasks.py can now either:
   (a) become simple wrappers that just call
       send_email_task.delay(subject=..., template_base=...,
       context=..., recipient_id=str(user.id), event_type="EMAIL_VERIFICATION")
       directly (no longer needing bind=True/self.retry themselves,
       since the retry now lives centrally in send_email_task), or
   (b) stay as their own @shared_task (bind=True, ...) wrappers that
       call apps.emails.services.send_transactional_email synchronously
       and handle their own retry, bypassing send_email_task's queueing
       layer entirely.
   Prefer option (a) — it's simpler and centralizes retry policy in one
   place, matching this Phase's stated goal of a single point of
   control for transactional email. Keep the token-lookup-or-create
   logic (the part before the send call) completely unchanged in both
   functions — only the dispatch mechanism at the end changes.

   Note that with option (a), send_verification_email and
   send_password_reset_email are still triggered exactly as before
   from wherever they're currently called (e.g.
   apps.users.services.register_user) — you're not changing their
   external Celery task names or their call sites, just what happens
   inside their bodies.

Files affected:
- apps/users/tasks.py

Then run the existing tests for these two functions
(tests/users/test_tasks.py or wherever they live — find via grep -rln
"send_verification_email\|send_password_reset_email" tests/) and fix
any that directly asserted against the now-removed
_send_transactional_email internals (e.g. mocking
apps.users.tasks._send_transactional_email directly) — update those
mocks to target apps.emails.services.send_transactional_email or
apps.emails.tasks.send_email_task instead, whichever this refactor
actually calls. The externally-observable behavior (an email with the
right subject/template ends up in django.core.mail.outbox during
tests, EmailVerificationToken is created/reused correctly) must be
identical to before this prompt — only the internal call path changed.

Acceptance Criteria:
- pytest tests/users/ -v passes completely with zero behavioral
  regression
- grep -rn "_send_transactional_email\|_RETRY_DELAY" apps/users/
  returns no results (fully removed, not just unused)
- An EmailLog row is now created for every verification/password-reset
  email send (confirm with a quick manual/test check, since this is a
  new side effect of the refactor that didn't exist before)

Verification Steps:
1. pytest tests/users/ -v
2. grep -rn "_send_transactional_email\|_RETRY_DELAY" apps/users/
3. pytest tests/emails/ tests/users/ -v (both together, to confirm
   nothing between them regressed)
4. git diff --stat
```

### Prompt 3 — ایمیل‌های دامنهٔ Bookings (۴ رویداد)

```
Goal: Add real transactional emails for the four booking-lifecycle
events: new booking request (to teacher), booking accepted (to
student), booking rejected (to student), and booking expired (to
student — a brand-new notification, since none currently exists for
this event).

Before starting, read these files completely:
1. apps/bookings/tasks.py — the full file (notify_new_booking_request,
   notify_booking_accepted, notify_booking_rejected) and its module
   docstring explaining the "thin task, lazy imports, no retry policy"
   convention
2. apps/bookings/services.py — reap_stale_pending_booking_requests (or
   wherever the booking-expiry sweep task lives — it might be in
   tasks.py; confirm) to see exactly what happens when a booking
   expires and confirm no notification currently fires there at all
3. apps/emails/services.py, apps/emails/tasks.py (from Prompts 1-2)
4. apps/users/templates/users/emails/verify_email.html and .txt (as
   the reference example of extending emails/base.html — note the
   import path changed from users/emails/base.html to emails/base.html
   after Prompt 1's move)
5. apps/bookings/models.py — BookingRequest, to know exactly which
   fields (student, teacher, skill, requested_start, requested_end,
   locked_price, locked_currency) are available for template context

What to build:

a) Create templates (English content, LTR — Persian localization is
   explicitly out of scope, that's Phase 5):

   apps/bookings/templates/bookings/emails/new_booking_request.html
   and .txt — to the teacher: "You have a new booking request from
   {{ student_name }} for {{ skill_name }}, requested for
   {{ requested_start }}." with a CTA link to
   {{ frontend_url }}/bookings/{{ booking_id }} (build this URL in the
   task using settings.FRONTEND_URL, matching the pattern already used
   in send_verification_email for verification_link)

   apps/bookings/templates/bookings/emails/booking_accepted.html and
   .txt — to the student: "{{ teacher_name }} accepted your booking
   request for {{ skill_name }}." + CTA link

   apps/bookings/templates/bookings/emails/booking_rejected.html and
   .txt — to the student: "{{ teacher_name }} was unable to accept
   your booking request for {{ skill_name }}." + a note that they were
   refunded automatically (this project's existing auto-refund-on-
   rejection behavior — reference it reassuringly, don't invent new
   claims about timing beyond what's actually true; check
   apps.payments.services for the actual refund behavior on rejection
   to phrase this accurately)

   apps/bookings/templates/bookings/emails/booking_expired.html and
   .txt — to the student only (not the teacher — the teacher never
   took any action, so notifying them has little value; document this
   choice with a one-line comment in the task): "Your booking request
   for {{ skill_name }} expired because the teacher didn't respond in
   time. You've been refunded automatically."

   Each .html extends "emails/base.html" following verify_email.html's
   exact block structure (preheader/heading/content); each .txt is a
   plain-text equivalent, matching verify_email.txt's style.

b) In apps/bookings/tasks.py, extend the three existing tasks
   (notify_new_booking_request, notify_booking_accepted,
   notify_booking_rejected): after the existing create_notification
   call (inside the same try/except, right after it), add a call to
   apps.emails.tasks.send_email_task.delay(
       subject=..., template_base="bookings/emails/...",
       context={...}, recipient_id=str(<recipient>.id),
       event_type="<matching the Notification type used just above>",
   )
   Build the context dict from the already-fetched booking object's
   fields (student.full_name, teacher.full_name, skill.name,
   requested_start.isoformat(), a booking_url built from
   settings.FRONTEND_URL). Since send_email_task already has its own
   internal retry, do not wrap this call in the same try/except that's
   already there for create_notification if that would cause a
   create_notification failure to also skip the email, or vice versa —
   think through this carefully: if create_notification fails, should
   the email still be attempted? Yes — they're independent concerns;
   structure the code so a failure in one doesn't prevent the other
   from being attempted (e.g. two separate try/except blocks, each
   logged and swallowed independently, rather than one that short-
   circuits the other).

c) Add a new task notify_booking_expired(booking_id: str) in
   apps/bookings/tasks.py, following the exact same structural pattern
   as the three existing tasks (fetch the booking, create a
   Notification of a new type "BOOKING_EXPIRED" for the student, then
   independently attempt the email send, each in its own try/except).

d) Find wherever the booking-expiry sweep currently is (likely
   apps.bookings.tasks.reap_stale_pending_booking_requests or similar
   — confirm the exact name via grep -rn "expire\|reap_stale" apps/bookings/)
   and add a call to notify_booking_expired.delay(str(booking.id)) for
   each booking it expires, following whatever dispatch pattern
   (immediate .delay() call vs transaction.on_commit) is already used
   at that call site for other side effects of this sweep, if any — if
   the sweep task itself already runs outside of a request-response
   cycle (it's a scheduled Celery task, not something wrapped in an
   HTTP-request transaction), a direct .delay() call is fine and
   transaction.on_commit isn't necessary there (that pattern exists
   specifically for HTTP-request-triggered service functions, not for
   a Celery beat task that's already running standalone) — confirm this
   reasoning against how the sweep task itself is structured (is it
   already inside its own transaction.atomic block? If so, still use
   on_commit for consistency; if not, a direct call is fine) and
   document your choice with a comment.

Files affected:
- apps/bookings/templates/bookings/emails/*.html, *.txt (8 new files)
- apps/bookings/tasks.py

Then write/update tests:
- tests/bookings/test_tasks.py: for each of the four notify_* tasks,
  confirm (a) the Notification row is created as before, and (b) an
  email now appears in django.core.mail.outbox (or, if using Celery
  eager mode with send_email_task, confirm an EmailLog with
  status="SENT" is created) with the correct recipient/subject
- A test confirming a create_notification failure (mocked to raise)
  does NOT prevent the email from being sent, and vice versa
  (independent failure isolation, per the design decision above)
- A test for notify_booking_expired specifically
- Update the sweep task's existing test(s) to confirm
  notify_booking_expired now gets dispatched for each expired booking

Acceptance Criteria:
- pytest tests/bookings/ -v passes completely
- All four booking events send a real templated email (visible in
  django.core.mail.outbox during tests), not just an in-app
  Notification
- The independent-failure-isolation test passes

Verification Steps:
1. pytest tests/bookings/ -v
2. pytest tests/emails/ -v (confirm Prompt 1/2's tests still pass)
3. git diff --stat
```

### Prompt 4 — ایمیل‌های دامنهٔ Payments (۶ رویداد، شامل ۴ تسک کاملاً جدید)

```
Goal: Add real transactional emails for all six payment-lifecycle
events: payment received (extend existing), payment failed (extend
existing), refund processed (new), payout released to teacher (new),
dispute opened (new), dispute resolved (new). The last four have no
notification of any kind today (not even in-app) — this prompt builds
both the Notification and the email for them from scratch.

Before starting, read these files completely:
1. apps/payments/tasks.py — the full file (notify_payment_received,
   notify_payment_failed)
2. apps/payments/services.py — the full file, with special focus on:
   refund_payment (where the new refund-processed notification hooks
   in), release_payment_to_teacher (where payout-released hooks in),
   resolve_dispute (where dispute-resolved hooks in), and find wherever
   a dispute is first opened/created (grep -rn "Dispute.objects.create"
   apps/payments/ — this is where dispute-opened hooks in; it might be
   in webhooks.py from Phase 1's legacy charge.dispute.created handler,
   or a dedicated service function — confirm exactly where before
   writing the hook)
3. apps/payments/models.py — Payment, PayoutLedgerEntry, Dispute, to
   know exactly which fields are available for email context (amount,
   currency, refund_amount, payout_amount, dispute reason/resolution)
4. apps/common/money.py — to correctly format amounts in email context
   (these are IRR integers per Phase 0 — do not assume 2 decimal
   places in the email template context)

What to build:

a) Create templates (English, LTR, matching the exact structural
   pattern from Prompt 3's booking templates):

   apps/payments/templates/payments/emails/payment_received.html/.txt
   — "We've received your payment of {{ amount }} for your session
   with {{ teacher_name }}." (this is likely already partially covered
   if notify_payment_received exists — check whether it already has
   email logic; if not per this Phase's audit, build it fresh)

   apps/payments/templates/payments/emails/payment_failed.html/.txt —
   "Your payment for the session with {{ teacher_name }} could not be
   processed. Please try again."

   apps/payments/templates/payments/emails/refund_processed.html/.txt
   — to the student: "You've been refunded {{ refund_amount }} for
   your booking with {{ teacher_name }}."

   apps/payments/templates/payments/emails/payout_released.html/.txt
   — to the teacher: "A payout of {{ payout_amount }} for your session
   with {{ student_name }} has been initiated. It will be transferred
   to your registered bank account." (phrase this accurately given
   Phase 1's manual-transfer payout design — do not claim instant
   automated transfer)

   apps/payments/templates/payments/emails/dispute_opened.html/.txt —
   to both parties: "A dispute has been opened regarding your session
   payment. Our team will review it and follow up." (generic enough to
   send to both student and teacher without needing to know which side
   opened it)

   apps/payments/templates/payments/emails/dispute_resolved.html/.txt
   — to both parties: "The dispute regarding your session payment has
   been resolved. {{ resolution_summary }}."

b) In apps/payments/tasks.py:

   Extend notify_payment_received and notify_payment_failed exactly
   like Prompt 3 extended the booking tasks (independent try/except
   for the email call after the existing create_notification call).

   Add four new tasks, each following the exact same structural
   pattern as the existing ones in this file (fetch object with
   select_related, create_notification with a new type value, then an
   independent email-send attempt, each wrapped in its own try/except
   + logger.exception + swallow, ending with a logger.info):

   notify_refund_processed(payment_id: str) — recipient: the student
     (payment.booking.student, or however the student is reached from
     Payment — check the model's actual relationships)
   notify_payout_released(payout_ledger_entry_id: str) — recipient:
     the teacher
   notify_dispute_opened(dispute_id: str) — dispatched TWICE (once per
     party) or written to loop over both the student and teacher and
     notify each — your choice, but document it; simplest is probably
     two separate .delay() calls from the call site (one per
     recipient), keeping the task itself single-recipient like every
     other task in this file, rather than making this one task handle
     multiple recipients internally (which would break the pattern
     every other task in this file follows)
   notify_dispute_resolved(dispute_id: str) — same two-recipient
     approach as above

c) In apps/payments/services.py, add the appropriate
   transaction.on_commit(...) dispatch calls:
   - In refund_payment: after the refund is finalized, dispatch
     notify_refund_processed.delay(str(payment.id))
   - In release_payment_to_teacher: after the PayoutLedgerEntry is
     created (from Phase 1's design), dispatch
     notify_payout_released.delay(str(entry.id))
   - Wherever a Dispute is first created: dispatch
     notify_dispute_opened.delay(str(dispute.id)) twice — once for
     student, once for teacher (or via a small helper that does the
     loop, your call, as long as the underlying task stays
     single-recipient per (b) above)
   - In resolve_dispute: after the dispute's resolution is finalized,
     dispatch notify_dispute_resolved.delay(str(dispute.id)) the same
     way

Files affected:
- apps/payments/templates/payments/emails/*.html, *.txt (12 new files)
- apps/payments/tasks.py
- apps/payments/services.py

Then write/update tests:
- tests/payments/test_tasks.py: one test per task (6 total, extending
  2 existing + 4 new) confirming both the Notification and the email
  are correctly created/sent
- tests/payments/test_services.py: for refund_payment,
  release_payment_to_teacher, and resolve_dispute (and wherever
  dispute-opening happens), confirm the corresponding notify task is
  dispatched via transaction.on_commit (check how this project's
  existing tests already verify on_commit dispatch for
  notify_payment_received, e.g. mocking .delay and asserting it was
  called after a transaction commits in a TestCase, and follow that
  exact pattern)
- A test confirming a mocked email-send failure during
  release_payment_to_teacher does NOT roll back or affect the
  PayoutLedgerEntry itself (payout notification failure must never
  undo a real financial state change — this is the same
  failure-isolation principle from Prompt 3, applied to the
  highest-stakes domain in the project)

Acceptance Criteria:
- pytest tests/payments/ -v passes completely
- All six payment events send both a Notification and a real
  templated email
- The failure-isolation test passes: an email failure never affects
  Payment/PayoutLedgerEntry/Dispute state

Verification Steps:
1. pytest tests/payments/ -v
2. pytest tests/emails/ tests/bookings/ -v (confirm no cross-domain
   regression from previous prompts)
3. git diff --stat
```

### Prompt 5 — ایمیل‌های دامنهٔ Sessions (ارتقای Reminder + Cancelled + Completed جدید)

```
Goal: Upgrade the existing session-reminder email from a plain,
unstyled send_mail call to the project's proper templated
multipart-email pattern, add email to the existing
notify_session_cancelled task, and add a brand-new
notify_session_completed task (requesting a review — this is the
natural bridge to Phase 3's review feature).

Before starting, read these files completely:
1. apps/sessions/tasks.py — the full file, with absolute focus on
   send_session_reminder's current plain send_mail call (which this
   prompt replaces) and notify_session_cancelled's existing structure
2. apps/sessions/services.py — mark_session_completed (the function
   this prompt hooks a new notification into) and cancel_session
3. apps/reviews/ (from Phase 3, if implemented) — to confirm the exact
   URL pattern for "submit a review for this session" so the
   completed-session email's CTA link points somewhere real (e.g.
   {{ frontend_url }}/reviews/submit/{{ session_id }}, matching
   whatever route Phase 3's Prompt 8 actually created — check
   src/app/routes/router.tsx if available, or fall back to a
   reasonably-named path and note the assumption if Phase 3 hasn't
   been implemented in this codebase yet)
4. apps/emails/services.py, apps/emails/tasks.py

What to build:

a) Create templates:

   apps/sessions/templates/sessions/emails/session_reminder.html/.txt
   — to student and teacher separately (two distinct templates, or one
   shared template with a role-aware context variable — your call, but
   keep it simple; a single template with a
   {% if is_teacher %}...{% else %}...{% endif %} block is probably
   cleanest given how similar the two messages are): "Your session with
   {{ other_party_name }} starts in about 30 minutes, at
   {{ start_time }}."

   apps/sessions/templates/sessions/emails/session_cancelled.html/.txt
   — to enrolled students: "Your session with {{ teacher_name }},
   scheduled for {{ start_time }}, has been cancelled."

   apps/sessions/templates/sessions/emails/session_completed.html/.txt
   — to the student: "Your session with {{ teacher_name }} is
   complete! We'd love to hear how it went." + CTA button linking to
   the review-submission URL identified above.

b) In apps/sessions/tasks.py:

   Rewrite send_session_reminder: remove the raw
   `from django.core.mail import send_mail` import and the plain
   send_mail(...) call entirely. Replace it with two independent calls
   to apps.emails.tasks.send_email_task.delay(...) — one per enrolled
   student (looping, matching the existing create_notification loop
   already in this function) using template_base=
   "sessions/emails/session_reminder" with an is_teacher=False context
   flag, and one for the teacher with is_teacher=True. Keep this
   task's existing bind=True/max_retries/self.retry structure exactly
   as-is around the whole body (this task already has retry logic,
   unlike the pure-notification tasks in other domains — preserve
   that, don't remove it just because send_email_task also has its own
   internal retry; the outer retry here is fine to keep for the
   create_notification portion, or you can simplify by removing the
   outer retry now that email sending has its own independent retry
   path via send_email_task — think through which is cleaner and
   document your choice: since send_email_task now handles email
   retries independently via .delay(), and create_notification
   failures were always logged-and-swallowed everywhere else in this
   project rather than retried, the cleaner design is likely to
   REMOVE this task's bind=True/self.retry wrapping entirely and match
   every other notify_* task's simpler "try/except, log, swallow"
   convention — do this if it doesn't lose any meaningful behavior,
   and document the reasoning in a comment.)

   Extend notify_session_cancelled: add an independent email-send
   attempt per enrolled student, following the exact same pattern as
   Prompt 3/4's extensions.

   Add a new task notify_session_completed(session_id: str): fetch the
   session (with its booking/student, since review eligibility per
   Phase 3 is 1:1-only for now — if session.booking is None or
   session.session_type != "ONE_ON_ONE", skip sending this email
   entirely and just log an info message, matching Phase 3's own
   documented scope boundary for group sessions), create a
   Notification (new type "SESSION_COMPLETED_REVIEW_PROMPT") for the
   student, then independently attempt the email send.

c) In apps/sessions/services.py, in mark_session_completed: after the
   existing TeacherProfile.total_sessions increment, add
   transaction.on_commit(lambda: notify_session_completed.delay(str(session.id))).
   Import notify_session_completed lazily inside the function, matching
   this project's existing lazy-import convention for cross-cutting
   task dispatch.

Files affected:
- apps/sessions/templates/sessions/emails/*.html, *.txt (6 new files)
- apps/sessions/tasks.py
- apps/sessions/services.py

Then write/update tests:
- tests/sessions/test_tasks.py: send_session_reminder now sends
  templated emails (check django.core.mail.outbox for two entries —
  student and teacher — with the right subjects/recipients), not the
  old plain send_mail; notify_session_cancelled now also sends email;
  a new test for notify_session_completed (happy path for a 1:1
  session, and a skip-with-log-message test for a group-type session
  with booking=None)
- tests/sessions/test_services.py: mark_session_completed dispatches
  notify_session_completed via on_commit (following this project's
  established on_commit-testing pattern from earlier prompts/phases)

Acceptance Criteria:
- pytest tests/sessions/ -v passes completely
- send_session_reminder no longer imports or calls
  django.core.mail.send_mail directly
- notify_session_completed correctly no-ops (without error) for a
  group-type session

Verification Steps:
1. pytest tests/sessions/ -v
2. grep -n "send_mail" apps/sessions/tasks.py (should return nothing —
   fully replaced by the shared service)
3. pytest tests/emails/ tests/bookings/ tests/payments/ -v (confirm no
   cross-domain regression)
4. git diff --stat
```

### Prompt 6 — ایمیل‌های Teacher Verification (Phase 2) + Review جدید (Phase 3)

```
Goal: Add real transactional emails to the four teacher-verification
notification tasks built in Phase 2 (apps/teachers/tasks.py), and add
one brand-new task notifying a teacher when they receive a new review
(from Phase 3's submit_review flow).

Before starting, read these files completely:
1. apps/teachers/tasks.py (from Phase 2) — the full file:
   notify_verification_submitted, notify_verification_approved,
   notify_verification_rejected, notify_verification_suspended
2. apps/teachers/models.py — TeacherVerification, for the exact fields
   (rejection_reason, suspension_reason) needed in email context
3. apps/reviews/services.py, apps/reviews/tasks.py (from Phase 3) — the
   full submit_review function and recalculate_teacher_rating task, to
   know exactly where to add a new on_commit dispatch and how a new
   task in this file should be structured to match
   recalculate_teacher_rating's style
4. apps/emails/services.py, apps/emails/tasks.py

If this project's actual codebase does not yet have Phase 2 and/or
Phase 3 implemented (i.e. apps/teachers/tasks.py or apps/reviews/
doesn't exist yet), STOP and report this clearly rather than
fabricating files for phases that were supposed to already be
complete — this prompt assumes Phases 2 and 3 have already been
implemented and merged, per this Phase's stated dependency.

What to build:

a) Create templates:

   apps/teachers/templates/teachers/emails/verification_submitted.html/.txt
   — to the teacher (confirmation their documents were received): "We've
   received your verification documents and will review them shortly."
   (Note: per Phase 2's design, notify_verification_submitted currently
   notifies admin staff, not the teacher — decide here whether to also
   send this confirmation email to the teacher themselves, which is
   good UX practice even though the in-app Notification only went to
   admins; if you add a teacher-facing email here, this is a small,
   deliberate scope addition beyond Phase 2's original design — note
   this explicitly in your final summary as a considered addition, not
   an oversight. Do NOT add a new email to admin staff for this event
   in this prompt — email-to-staff-broadcast is a different, lower-
   priority concern and not part of this Phase's matrix; admins already
   get the in-app Notification from Phase 2.)

   apps/teachers/templates/teachers/emails/verification_approved.html/.txt
   — "Congratulations! Your teacher profile has been approved and is
   now visible in the marketplace."

   apps/teachers/templates/teachers/emails/verification_rejected.html/.txt
   — "Your verification documents were not approved. Reason:
   {{ rejection_reason }}. You may resubmit your documents at any time."

   apps/teachers/templates/teachers/emails/verification_suspended.html/.txt
   — "Your teacher profile has been suspended. Reason:
   {{ suspension_reason }}. Please contact support for more information."

   apps/reviews/templates/reviews/emails/new_review_received.html/.txt
   — to the teacher: "{{ student_name }} left you a {{ rating }}-star
   review: \"{{ comment }}\"" (if comment is blank, omit that sentence
   — handle this conditionally in the template with
   {% if comment %}...{% endif %})

b) In apps/teachers/tasks.py, extend all four existing tasks with an
   independent email-send attempt, following the exact same
   independent-try/except pattern established in Prompts 3-5. For
   notify_verification_submitted specifically, since it currently
   loops over all staff users for the in-app Notification, decide
   whether to also loop and email every staff member, or only send the
   teacher-facing confirmation email discussed above (recommended: only
   the teacher-facing confirmation email in this prompt, to avoid an
   email storm to every admin on every submission — document this
   choice with a comment).

c) In apps/reviews/tasks.py, add a new task:

   @shared_task(name="reviews.notify_teacher_of_new_review")
   def notify_teacher_of_new_review(review_id: str) -> None:
       """Notify a teacher that they've received a new published
       review."""
   Following the exact structural pattern of
   recalculate_teacher_rating (fetch the Review with select_related on
   session__teacher and student, handle DoesNotExist with a logged
   warning + return), create a Notification for the teacher (new type
   "NEW_REVIEW_RECEIVED"), then independently attempt the email send.

   Note: since a review can be created and then later HIDDEN via
   moderation, this notification should only fire on the INITIAL
   submission, not on every moderation status change (unlike
   recalculate_teacher_rating, which correctly re-runs on every
   moderation action) — make sure this task is only dispatched from
   submit_review, never from moderate_review, and add a one-line
   comment in moderate_review explicitly noting that recalculate_teacher_rating
   is re-dispatched there but notify_teacher_of_new_review deliberately
   is not (to prevent a confusing "you got a new review" email firing
   when an admin merely re-publishes a previously-hidden one).

d) In apps/reviews/services.py's submit_review function, add
   transaction.on_commit(lambda: notify_teacher_of_new_review.delay(str(review.id)))
   alongside the existing recalculate_teacher_rating dispatch.

Files affected:
- apps/teachers/templates/teachers/emails/*.html, *.txt (8 new files)
- apps/reviews/templates/reviews/emails/*.html, *.txt (2 new files)
- apps/teachers/tasks.py
- apps/reviews/tasks.py
- apps/reviews/services.py

Then write/update tests:
- tests/teachers/test_tasks.py: extend each of the four existing task
  tests to also confirm an email is sent
- tests/reviews/test_tasks.py: a new test for
  notify_teacher_of_new_review (Notification + email created correctly)
- tests/reviews/test_services.py: submit_review dispatches
  notify_teacher_of_new_review via on_commit; moderate_review does NOT
  dispatch it (only recalculate_teacher_rating) — write this as an
  explicit negative-assertion test (mock notify_teacher_of_new_review.delay,
  call moderate_review, assert it was NOT called)

Acceptance Criteria:
- pytest tests/teachers/ tests/reviews/ -v passes completely
- moderate_review never triggers a "new review" email, only
  submit_review does (explicit negative test passes)

Verification Steps:
1. pytest tests/teachers/ tests/reviews/ -v
2. pytest tests/emails/ tests/bookings/ tests/payments/ tests/sessions/ -v
   (full cross-domain regression check across every prompt in this
   Phase so far)
3. git diff --stat
```

### Prompt 7 — بررسی نهایی Matrix، پاکسازی، و تست یکپارچهٔ End-to-End

```
Goal: Final review confirming every event in this Phase's Transactional
Email Matrix now actually sends an email, plus one comprehensive
integration test and a cross-project cleanup pass.

Before starting:
1. Review the diffs from Prompts 1-6 for the full picture.
2. Build (in your own working notes, not a file) the complete list of
   events this Phase was supposed to cover, matching the original
   Phase 4 architecture: booking (new request, accepted, rejected,
   expired), payment (received, failed, refund, payout released,
   dispute opened, dispute resolved), session (reminder, cancelled,
   completed), teacher verification (submitted, approved, rejected,
   suspended), review (new review received) — 16 events total.

What to build:

1. grep -rln "send_email_task\|send_transactional_email" apps/ | grep -v migrations
   and cross-reference every call site found against the 16-event list
   above. Report explicitly, in your final summary, which events (if
   any) are still missing a call site — do not silently skip reporting
   a gap; if Prompts 1-6 were followed exactly, this should be a
   complete match, but confirm rather than assume.

2. Add a comprehensive integration test:
   tests/emails/test_e2e_matrix.py that, for each of the 16 events,
   triggers the real underlying service function or task (not a
   shortcut) and asserts an email lands in django.core.mail.outbox with
   the expected recipient and a non-empty subject. This test can be
   long/repetitive — that's fine and appropriate for a matrix-coverage
   test; structure it as one test function per event group (booking
   events, payment events, session events, verification events, review
   event) rather than one giant test, for readable failure output.

3. Run the ENTIRE backend test suite:
   pytest -x
   and confirm zero regressions across every domain touched by this
   Phase.

4. Check EMAIL_BACKEND in config/settings/test.py (or wherever this
   project's test settings live) to confirm it's set to
   django.core.mail.backends.locmem.EmailBackend (Django's standard
   in-memory test backend, which is what makes django.core.mail.outbox
   work in tests) and NOT any real SMTP backend — if it's missing or
   set to something else, fix it, since without this, every test in
   this entire Phase would either fail or silently attempt real network
   sends.

5. Check EMAIL_BACKEND in the CI workflow files
   (.github/workflows/backend-ci.yml) for the same reason — confirm no
   real SMTP credentials or network access is required to run the test
   suite in CI.

Files affected:
- tests/emails/test_e2e_matrix.py (new)
- config/settings/test.py (only if EMAIL_BACKEND was missing/wrong)
- .github/workflows/backend-ci.yml (only if a fix was needed — report
  either way)

Acceptance Criteria:
- pytest -x (full backend suite) passes with zero failures
- The 16-event matrix test passes completely
- EMAIL_BACKEND is confirmed to be the in-memory test backend in both
  local test settings and CI

Verification Steps:
1. pytest tests/emails/test_e2e_matrix.py -v
2. pytest -x -v (full suite)
3. grep -n "EMAIL_BACKEND" config/settings/*.py .github/workflows/*.yml
4. grep -rln "send_email_task\|send_transactional_email" apps/ | grep -v migrations
   (paste this output in your final summary alongside the 16-event
   cross-reference from step 1)
5. git diff --stat (final summary of this entire Phase)
```
