# Phase 5 — اعلان‌های مفقود: ریفاند، شکست پرداخت گروهی، لغو رزرو

## Implementation Prompts

این بخش را **بالای هر Prompt** paste کنید.

```
GLOBAL RULES (apply to this task; the prompt states whether it is backend-only or frontend-only):
- Repository: hirbad-platform (Django backend in hirbad-backend/, React frontend in hirbad-frontend/).
- FIRST read all code relevant to this task and summarize what you found in 5-10 lines. Line numbers quoted come from the original codebase and may have shifted after earlier phases; locate things by name.
- Minimal changes. No rewrites, no renaming, no reformatting of untouched code, no new dependencies. Preserve the architecture (service layer, thin Celery tasks with lazy imports, try/except-log-swallow convention).
- Celery task signatures: only ADD trailing parameters with defaults. Task arguments must be JSON-serializable (str/int/bool/None/list/dict of those).
- Persian user-facing copy rules from the previous phase apply: Persian digits, ZWNJ, never print None, never English prose.
- Create or update tests for everything you change. Run the relevant suites. If something cannot be run, say NOT RUN + exact command; never claim success.
- At the end print: files changed, tests run + results, assumptions, discrepancies versus this prompt. If the code contradicts this prompt, STOP and report.
```

### Prompt 1 — بررسی فقط‌خواندنی (بدون هیچ تغییری)
```
TASK (READ-ONLY, change NOTHING): verify the findings this phase is built on and answer open questions. Output a report only.

Answer each with file:function:line evidence and a verdict CONFIRMED / REFUTED / UNCLEAR:
Q1. After payments.services.refund_payment succeeds (payment REFUNDED), does ANY code (service, task, signal, serializer, view, frontend caller) cancel the related BookingRequest or Session, or notify the teacher? grep REFUNDED across apps/, read RefundPaymentView and the frontend code that calls the refund endpoint.
Q2. What does sessions.services.leave_session do for an enrollment that already has a HELD_IN_ESCROW payment? Is there a refund or a notification on that path?
Q3. Exact reverse accessor names between Payment and BookingRequest/SessionEnrollment (Payment.booking / Payment.enrollment are OneToOne): how do I load "the payment of this booking" safely (RelatedObjectDoesNotExist)? 
Q4. Which booking statuses can each role cancel via bookings.services.cancel_booking, and can an AWAITING_TEACHER_REVIEW booking be cancelled by anyone through it?
Q5. What happens in handle_gateway_callback_success when a SUCCESS callback arrives for a group-enrollment payment the reaper already deleted (payment not found)? Is the student charged without an enrollment? Is anything logged/alerted?
Q6. Confirm these claims: (a) notify_payment_failed returns silently when the payment row is gone; (b) reap_stale_pending_group_enrollment_payments has no notification dispatch; (c) auto_refund_on_rejection, refund_group_session_cancellation and refund_held_payment_for_no_show dispatch no notification; (d) cancel_booking dispatches no notification for a PENDING booking cancelled by either role.
Q7. List every send_email_task / create_notification call that already involves the words refund, cancelled or failed so nothing is duplicated.

Also state what you could NOT verify.
```

### Prompt 2 — کدها، نوع `BOOKING_CANCELLED` و قرارداد (backend)
```
TASK (BACKEND): register the new notification type and the new payload fields. Depends on Phases 0, 1(no), 3 and Prompt 1's findings (stop and report if Prompt 1 contradicted the plan).

1. Read apps/common/codes.py, apps/notifications/types.py (NotificationType, NOTIFICATION_SPECS, NotificationSpec optional keys), apps/notifications/payloads.py (builders), docs/NOTIFICATIONS.md, apps/notifications/migrations/ (latest).
2. apps/common/codes.py: add RefundReasonCode (STUDENT_CANCELLATION, GROUP_SESSION_CANCELLED, TEACHER_NO_SHOW) and BookingCancelledBy (STUDENT, TEACHER), same TextChoices style (value == name).
3. types.py: add NotificationType.BOOKING_CANCELLED ("BOOKING_CANCELLED"); spec recipient_role "party", required keys booking_id, cancelled_by_role, student_name, teacher_name; optional skill_name, requested_start, schema_version. Extend OPTIONAL keys: BOOKING_REJECTED and BOOKING_EXPIRED -> refunded, refund_amount, currency; REFUND_PROCESSED -> refund_reason_code; PAYMENT_FAILED -> payment_deleted.
4. Generate ONE migration for the notifications app (choices change only; verify with sqlmigrate that it does not alter the column definition beyond choices metadata) and make sure it is reversible.
5. payloads.py: add build_booking_cancelled_payload(booking, cancelled_by_role) and extend: booking rejected/expired builders accept optional payment (a Payment or None) and emit refunded (True only if payment.status == "REFUNDED"), refund_amount (str(payment.refund_amount) or None), currency (payment.currency or None); the refund builder accepts refund_reason_code (default STUDENT_CANCELLATION, validated against RefundReasonCode.values, unknown -> log warning and keep value); the payment-failed builder accepts payment_deleted: bool = False. All new keys are additive; schema_version stays.
6. docs/NOTIFICATIONS.md: add the new type row and new optional keys.
7. Tests: spec/builder contract tests extended (every key emitted is required or optional); the union/count test now expects 21 types; builder unit tests for each new key incl. refunded False/True and unknown refund_reason_code.

ACCEPTANCE: makemigrations --check clean; notifications tests green. Frontend and emails are NOT touched in this prompt (the Phase 1 frontend contract test may fail until Prompt 6; report that, do not hide it).
```

### Prompt 3 — اعلان ریفاند در همه‌ی مسیرها (backend)
```
TASK (BACKEND): one notification per refund, across all four refund paths. Depends on Prompt 2.

1. Read payments/services.py (refund_payment, refund_group_session_cancellation, auto_refund_on_rejection, refund_held_payment_for_no_show, _record_payment_event), payments/tasks.py (notify_refund_processed and its dedupe_key), bookings/services.py (reject_booking and expire_stale_booking_requests: where auto_refund_on_rejection is called and where notify_booking_rejected/expired are dispatched on_commit), bookings/tasks.py (notify_booking_rejected, notify_booking_expired), sessions/services.py report_no_show, tests/payments, tests/bookings, tests/sessions.
2. notify_refund_processed: add TRAILING parameter refund_reason_code: str = "STUDENT_CANCELLATION"; include it in the payload through the Prompt 2 builder; leave the email context for Prompt 7. refund_payment keeps dispatching with the default (explicitly pass RefundReasonCode.STUDENT_CANCELLATION).
3. refund_group_session_cancellation: after each payment is successfully marked REFUNDED (inside its own per-payment transaction.atomic()), dispatch transaction.on_commit(lambda pid=str(payment.id): notify_refund_processed.delay(pid, "GROUP_SESSION_CANCELLED")). Use default-argument binding to avoid late-binding bugs. Payments skipped (not HELD_IN_ESCROW) or failed at the gateway get NO notification (keep the existing log lines).
4. refund_held_payment_for_no_show: dispatch notify_refund_processed.delay(str(payment.id), "TEACHER_NO_SHOW") on_commit ONLY in the branch where the payment was just moved to REFUNDED (not when it was already REFUNDED/skipped).
5. auto_refund_on_rejection stays notification-free (its refund is folded into the cause event): in notify_booking_rejected and notify_booking_expired load the booking's payment safely (use the accessor you verified in Prompt 1, handle "no payment" and "not refunded"), pass it to the Prompt 2 builders, and add the same facts to the email contexts under new keys refunded (bool) and refund_amount (str|None) (templates are updated in Prompt 7; adding unused context keys is harmless). Do not add any new notification for these two events.
6. Tests: (a) group cancellation with 3 enrolled students (2 refundable, 1 gateway failure) -> exactly 2 REFUND_PROCESSED with code GROUP_SESSION_CANCELLED, none for the failed one, running the refund function twice adds none; (b) no-show refund -> one REFUND_PROCESSED TEACHER_NO_SHOW, repeated report/refund call adds none; (c) reject_booking and expire with a held payment -> exactly one notification each (BOOKING_REJECTED/EXPIRED) with refunded True and amount, and NO REFUND_PROCESSED; without a payment -> refunded False; (d) old signature notify_refund_processed(payment_id) still works.

ACCEPTANCE: payments/bookings/sessions tests green; no refund path produces two notifications for one payment.
```

### Prompt 4 — شکست پرداخت گروهی با snapshot (backend)
```
TASK (BACKEND): make group-enrollment payment failures notify the student even though the Payment row is deleted by CASCADE. Depends on Prompts 2-3.

1. Read payments/services.py handle_gateway_callback_failed (the enrollment branch with leave_session and the on_commit dispatch), payments/tasks.py notify_payment_failed (the whole task incl. the email block) and reap_stale_pending_group_enrollment_payments, sessions/services.py leave_session (what it deletes), and tests for these.
2. Add a small helper in payments/services.py (private): _build_failure_snapshot(payment) -> dict of JSON-safe values captured BEFORE leave_session runs: payment_id, student_id, amount (str), currency, teacher_id, teacher_name, session_id, session_start_time (ISO string), failure_code (payment.failure_code or "UNKNOWN"). Only primitives.
3. notify_payment_failed gets a new TRAILING parameter snapshot: dict | None = None. Behavior: payment found -> exactly as today; payment missing AND snapshot given -> recipient = User.objects.get(id=snapshot["student_id"]) (if the user is gone, log and return), build the notification through the Prompt 2 payload builder with payment_deleted=True (payload payment_id = snapshot payment_id, session_id included), and send the email with a context built from the snapshot (same keys as the normal context; Prompt 7 adapts the template CTA); payment missing and no snapshot -> the existing warning and return. Keep the dedupe_key PAYMENT_FAILED:{payment_id}.
4. handle_gateway_callback_failed: in the enrollment branch build the snapshot before leave_session and dispatch notify_payment_failed.delay(str(payment.id), snapshot) on_commit. For the booking branch keep today's call without a snapshot. Fix the misleading comment about "tolerates the payment being gone" so it describes the snapshot.
5. reap_stale_pending_group_enrollment_payments: build the snapshot before leave_session inside the same per-payment transaction, and after the atomic block succeeds dispatch the task with the snapshot (use on_commit inside the atomic block, with default-argument binding). The existing per-payment exception isolation must stay: a failing leave_session rolls back the status change and sends nothing. Make sure failure_code is TIMEOUT there (Phase 3).
6. Tests: gateway-declined group enrollment -> one PAYMENT_FAILED (payment_deleted True, failure_code GATEWAY_DECLINED, session_id present) and one email; reaper timeout -> one PAYMENT_FAILED (TIMEOUT) and one email; 1:1 booking failure unchanged (payment_deleted False); calling the callback handler twice sends once; task called with an old single-argument message still works; leave_session raising in the reaper -> no notification and the payment keeps PENDING.

ACCEPTANCE: payments/sessions tests green; no PAYMENT_FAILED is lost for group enrollments.
```

### Prompt 5 — اعلان لغو رزرو (backend)
```
TASK (BACKEND): notify the other party when a PENDING booking is cancelled. Depends on Prompt 2 (and Phase 4 for emails).

1. Read bookings/services.py cancel_booking (and cancel_session_from_booking in sessions/services.py to confirm the ACCEPTED-by-teacher path already notifies via notify_session_cancelled), bookings/tasks.py (notify_booking_accepted as the style/email reference), apps/emails/subjects.py, the booking email templates folder, tests/bookings/.
2. Add task bookings.notify_booking_cancelled(booking_id: str, cancelled_by_role: str) in bookings/tasks.py following the exact structure of the sibling tasks (lazy imports, select_related student/teacher/skill, try/except per channel): recipient = the OTHER party (STUDENT cancelled -> notify booking.teacher; TEACHER cancelled -> notify booking.student); in-app via create_notification type BOOKING_CANCELLED with payload from build_booking_cancelled_payload and dedupe_key=build_dedupe_key("BOOKING_CANCELLED", booking.id); email via send_email_task with template_base "bookings/emails/booking_cancelled", subject=build_subject("BOOKING_CANCELLED", ...), event_type "BOOKING_CANCELLED", context student_name, teacher_name, skill_name (None allowed), requested_start (ISO), cancelled_by_role, booking_url.
3. cancel_booking: dispatch transaction.on_commit(lambda: notify_booking_cancelled.delay(str(booking.id), role)) ONLY when no session is cancelled (i.e. NOT when is_teacher and was_accepted - that path already sends SESSION_CANCELLED). role is "STUDENT" or "TEACHER" per the cancelling user. Do not change any permission/status logic.
4. Add to apps/emails/subjects.py: BOOKING_CANCELLED -> «درخواست رزرو لغو شد».
5. Create the template pair bookings/templates/bookings/emails/booking_cancelled.{html,txt} following the Persian style rules of the email phase ({% extends %}, {% load fa_format %}, blocks title/preheader/heading/content, CTA «مشاهده رزرو», guard skill_name with {% if %}, dates only via |jdatetime). Two branches by cancelled_by_role: STUDENT (recipient is the teacher): «{{ student_name }} درخواست رزرو خود را لغو کرد»; TEACHER (recipient is the student): «{{ teacher_name }} درخواست رزرو شما را لغو کرد». Unknown role -> a neutral sentence.
6. Tests: student cancels PENDING -> exactly one BOOKING_CANCELLED to the teacher (+1 email); teacher cancels PENDING -> one to the student; teacher cancels ACCEPTED -> exactly one SESSION_CANCELLED and NO BOOKING_CANCELLED; cancelling twice (second attempt rejected by status) adds nothing; rendering test of both template branches (no None, no Latin letters in visible text, no ISO).

ACCEPTANCE: bookings/emails tests green; the Phase 4 email contract tests pass with the new event.
```

### Prompt 6 — فرانتند (frontend-only)
```
TASK (FRONTEND ONLY): render the new type and payload facts. Backend contract: Prompt 2.

1. Read src/shared/types/common.types.ts, features/notifications/lib/notificationRegistry.ts, lib/payload.ts, fa/notifications.json, registry tests, fixtures (mockAllNotificationTypeFixtures), and the route table (router.tsx) for /bookings/:id and /group-sessions/:id.
2. Add BOOKING_CANCELLED to the NotificationType union (now 21) and a registry entry: icon CalendarX (or nearest existing lucide icon), target /bookings/:booking_id (fallback /bookings). Summary: cancelled_by_role STUDENT -> «{{studentName}} درخواست رزرو خود را لغو کرد»; TEACHER -> «{{teacherName}} درخواست رزرو شما را لغو کرد»; anything else -> «یک درخواست رزرو لغو شد». Names fall back to the existing fallback keys.
3. REFUND_PROCESSED by payload.refund_reason_code (with and without amount, as in Phase 1): STUDENT_CANCELLATION or missing/unknown -> existing texts; GROUP_SESSION_CANCELLED -> «به‌دلیل لغو جلسه‌ی گروهی، مبلغ {{amount}} به شما بازگردانده شد» / «به‌دلیل لغو جلسه‌ی گروهی، وجه شما بازگردانده شد»; TEACHER_NO_SHOW -> «به‌دلیل عدم حضور معلم، مبلغ {{amount}} به شما بازگردانده شد» / «به‌دلیل عدم حضور معلم، وجه شما بازگردانده شد».
4. BOOKING_REJECTED and BOOKING_EXPIRED when payload.refunded === true and a valid refund_amount: «رزرو شما با {{teacherName}} رد شد و مبلغ {{amount}} به شما بازگردانده شد» and «درخواست رزرو شما با {{teacherName}} بدون پاسخ ماند، منقضی شد و مبلغ {{amount}} بازگردانده شد» (use formatNotificationAmount with payload.currency); otherwise the existing texts. refunded false/missing must not mention a refund.
5. PAYMENT_FAILED target: when payload.payment_deleted === true -> /group-sessions/:session_id if session_id is a non-empty string, else /payments; otherwise /payments/:payment_id as before.
6. Tests: every new branch, unknown/missing codes, payloads from before this phase (no new keys), the Persian-only (no [A-Za-z]) check for each variant, target resolution for payment_deleted, the registry-vs-union exhaustiveness and the backend-contract test (it must now see 21 types if the backend file is available). Add backend-shaped fixtures WITHOUT altering legacy fixtures.

ACCEPTANCE: frontend tests, typecheck, lint, build pass.
```

### Prompt 7 — به‌روزرسانی قالب‌های ایمیل موجود (backend)
```
TASK (BACKEND, templates + small context changes): make the existing emails reflect the new facts. Depends on Prompts 3-5 and the email-phase infrastructure (fa_format filters, build_subject).

1. Read the templates: payments/emails/refund_processed|payment_failed, bookings/emails/booking_rejected|booking_expired, and notify_refund_processed / notify_payment_failed in payments/tasks.py.
2. refund_processed: pass refund_reason_code in the email context (task already has it) and add a sentence by code: STUDENT_CANCELLATION «مبلغ بر اساس قوانین لغو بازگردانده شد»؛ GROUP_SESSION_CANCELLED «جلسه‌ی گروهی لغو شد و مبلغ پرداختی شما به‌طور کامل بازگردانده می‌شود»؛ TEACHER_NO_SHOW «به‌دلیل عدم حضور معلم، مبلغ پرداختی شما به‌طور کامل بازگردانده می‌شود»؛ missing/unknown -> no reason sentence. Keep the note about the bank settlement time.
3. booking_rejected: replace the NEUTRAL refund sentence by: if refunded is true -> «مبلغ {{ refund_amount|toman }} به شما بازگردانده شد.» (currency-aware: use |money with the context currency if available); otherwise no refund statement at all (never claim a refund that did not happen). booking_expired: same logic, replacing the unconditional full-refund claim.
4. payment_failed: when the notification came from a snapshot (context key payment_deleted true, set by Prompt 4) the CTA must point to the group session page (use the frontend URL pattern already used by other emails, FRONTEND_URL + /group-sessions/<session_id>) with the label «تلاش دوباره برای ثبت‌نام»; otherwise unchanged.
5. Extend the preview samples (apps/emails/preview_samples.py) with: refund by each code, rejected with/without refund, expired with/without refund, failed (group, snapshot) and booking_cancelled (both roles).
6. Tests: render tests for each variant (no None, no Latin letters in visible text, no ISO); a test that booking_rejected/expired without a refund contains no refund sentence; the contract test from the email phase still passes.

ACCEPTANCE: emails/payments/bookings tests green; docs/EMAIL_COPY_FA.md regenerated if the script exists.
```

### Prompt 8 — Regression و گزارش
```
TASK: verify the whole phase; make no feature changes; fix only defects introduced by this phase.

1. Backend, if Django/DB are available: makemigrations --check --dry-run; migrate clean DB; migrate notifications back one step and forward; full pytest; lint targets. Frontend: tests, typecheck, lint, build. List anything NOT RUN with the exact command.
2. Static proofs: (a) every refund path dispatches at most one notification per payment (list the four paths and what each sends); (b) no code path sends BOOKING_CANCELLED for a booking whose session was cancelled; (c) notify_payment_failed and notify_refund_processed only gained trailing default params; (d) no create_notification payload is built inline (all via payloads.py); (e) NotificationType has 21 values and the frontend registry covers all.
3. Update docs/NOTIFICATIONS.md "Known gaps": remove the closed gaps (group failure, refund, student cancel) and list remaining ones: silent gateway-refund failures (ops alert), refund_payment not cancelling booking/session (decision D1), NO_SHOW_REPORTED (decision D2), late success callback after reaper (if Prompt 1 confirmed it).
4. Report: migrations, files changed, test counts before/after, the table notification-per-event as implemented, skipped items with reasons.
```
