# Phase 6 — سخت‌سازی (Hardening) و پاک‌سازی نهایی

## Implementation Prompts

این بخش را **بالای هر Prompt** paste کنید.

```
GLOBAL RULES (apply to this task; the prompt states whether it is backend-only, frontend-only or both):
- Repository: hirbad-platform (Django backend in hirbad-backend/, React frontend in hirbad-frontend/).
- FIRST read all code relevant to this task and summarize what you found in 5-10 lines. Line numbers quoted come from the original codebase and may have shifted after earlier phases; locate things by name.
- Minimal changes. No rewrites, no renaming, no reformatting of untouched code, no new dependencies. Preserve the architecture (service layer, thin Celery tasks with lazy imports, try/except-log-swallow convention).
- Celery task signatures: only ADD trailing parameters with defaults; arguments must be JSON-serializable.
- Persian user-facing copy rules from the email phase apply to any user-visible text you add (Persian digits, ZWNJ, no English prose).
- Create or update tests for everything you change. Run the relevant suites. If something cannot be run, say NOT RUN + exact command; never claim success.
- At the end print: files changed, tests run + results, assumptions, discrepancies versus this prompt. If the code contradicts this prompt, STOP and report.
```

### Prompt 1 — بررسی فقط‌خواندنی
```
TASK (READ-ONLY, change NOTHING): verify the findings this phase relies on. Report with file:function:line evidence and a verdict CONFIRMED / REFUTED / UNCLEAR for each.

Q1. emails/services.send_transactional_email: confirm that the final SENT bookkeeping (log.status="SENT" + save) runs OUTSIDE the try/except, so a DB error after message.send() propagates to send_email_task and triggers self.retry (duplicate email). Also confirm each retry creates a new EmailLog row.
Q2. List EVERY send_email_task.delay(...) call site in apps/ (file:function:event_type:the object id that identifies "this logical email") including apps/users/tasks.py. For EMAIL_VERIFICATION and PASSWORD_RESET: is there a per-request stable id (e.g. the token row id) available at the call site?
Q3. teachers/services.py: at each of the four dispatch points (submit, approve, reject, suspend), is the TeacherVerificationEvent row created BEFORE the transaction.on_commit dispatch, in the same transaction, and is the event object/id in scope?
Q4. Is SENTRY_DSN set in the deployment configs/docs (docker-compose.prod.yml, .env.example, docs)? Does the Sentry logging integration capture logger.error events (look at config/settings/production.py sentry_sdk.init)?
Q5. FINANCE OBSERVATIONS (do not fix): (a) do auto_refund_on_rejection and refund_held_payment_for_no_show call provider.refund(...) anywhere (directly or indirectly)? if not, say so plainly; (b) is a HELD_IN_ESCROW payment of a COMPLETED group-enrollment session ever released to the teacher (look at release_held_payments, release_payment_to_teacher and any group-specific path)? (c) what happens when a SUCCESS gateway callback arrives for a payment already deleted by the reaper?
Q6. Session model: which field tells when a session became CANCELLED (updated_at? SessionStatusLog.created_at?) and what are the exact status/session_type literals for a cancelled group session.
Q7. Existing size/volume hints for EmailLog (is there a retention/cleanup task?) to decide whether an index must be created concurrently.

Also state what you could NOT verify.
```

### Prompt 2 — هسته‌ی idempotency ایمیل (backend)
```
TASK (BACKEND): make transactional emails idempotent per logical email. Depends on Prompt 1's answers (stop and report if Q1 is REFUTED).

1. Read apps/emails/models.py, services.py, tasks.py, admin.py (if any), tests/emails/, apps/common/ for existing helper conventions.
2. EmailLog: add dedupe_key = CharField(max_length=160, null=True, blank=True, default=None), attempts = PositiveSmallIntegerField(default=0), last_attempt_at = DateTimeField(null=True, blank=True); Meta.constraints += UniqueConstraint(fields=["recipient","dedupe_key"], condition=Q(dedupe_key__isnull=False), name="emaillog_recipient_dedupe_uniq"). ONE migration; verify with sqlmigrate; reversible. If Prompt 1 Q7 shows the table can be very large, make the index creation concurrent (AddIndexConcurrently with atomic = False) and say so; otherwise a plain AddConstraint.
3. Create apps/emails/dedupe.py: build_email_dedupe_key(event_type: str, object_id, recipient_id) -> str = f"EMAIL:{event_type}:{object_id}:{recipient_id}".
4. services.send_transactional_email: add trailing keyword dedupe_key: str | None = None. Behavior:
   - dedupe_key None: today's behavior (new EmailLog row per call) EXCEPT the bookkeeping fix below.
   - dedupe_key set: claim via a helper _claim_email_log(recipient, event_type, dedupe_key): try create(status="QUEUED", attempts=1, last_attempt_at=now, dedupe_key=...) inside transaction.atomic(); on IntegrityError load the existing row: if status == "SENT" -> return (row, False) and log INFO "already sent"; if QUEUED and last_attempt_at newer than settings.EMAIL_INFLIGHT_STALE_SECONDS (new setting, default 900) -> return (row, False) and log INFO "in flight"; otherwise (FAILED, or stale QUEUED) -> single atomic UPDATE ... WHERE pk=? AND (status="FAILED" OR (status="QUEUED" AND last_attempt_at < stale_cutoff)) setting status="QUEUED", attempts=F("attempts")+1, last_attempt_at=now, error="" ; claimed = rows_updated == 1. Only a claimed row proceeds to send; an unclaimed call returns the existing row WITHOUT sending and WITHOUT raising.
   - Bookkeeping fix (both paths): after message.send() succeeds, the SENT save must be wrapped in its own try/except that logs logger.exception(...) and does NOT raise (the email is already delivered; a retry would duplicate it).
   - Failure path unchanged: mark FAILED with the error and re-raise so the task retries (with a key, the retry reuses the same row via the claim above).
5. send_email_task: add trailing parameter dedupe_key: str | None = None passed through. Update docstrings.
6. Tests (tests/emails/test_idempotency.py): (a) same key twice sequentially -> message.send called once, one row, second call returns the same row; (b) key + simulated failure then retry -> same row, attempts == 2, status SENT, send called twice in total; (c) post-send save error (patch EmailLog.save to raise only for the SENT update) does not raise and does not resend on a second invocation; (d) concurrent claim simulation: pre-create a QUEUED row with a fresh last_attempt_at -> call returns without sending; with a stale last_attempt_at -> sends and attempts increments; (e) different recipients with the same event key are independent; (f) no key -> legacy behavior (two calls, two sends, two rows); (g) old-style task call without dedupe_key works.

ACCEPTANCE: emails tests green; makemigrations --check clean; reversible migration.
```

### Prompt 3 — اعمال کلید ایمیل در همه‌ی محل‌ها (backend)
```
TASK (BACKEND): pass dedupe_key at every send_email_task.delay call. Depends on Prompt 2 and Prompt 1 Q2.

1. Use the call-site inventory from Prompt 1 Q2 (re-grep to be sure nothing was added since).
2. For each call add dedupe_key=build_email_dedupe_key(<event_type literal used in that call>, <object id>, <recipient id>) (lazy import in the task). Object ids mirror the in-app dedupe keys from the earlier phase: bookings -> booking.id (NEW_BOOKING_REQUEST, BOOKING_ACCEPTED, BOOKING_REJECTED, BOOKING_EXPIRED, BOOKING_CANCELLED); payments -> payment.id for PAYMENT_RECEIVED, PAYMENT_FAILED, REFUND_PROCESSED (for PAYMENT_FAILED built from a snapshot use the snapshot payment_id), entry.id for PAYOUT_RELEASED, dispute.id for DISPUTE_OPENED/RESOLVED; sessions -> session.id for SESSION_REMINDER (teacher and student are different recipient ids), SESSION_CANCELLED, SESSION_COMPLETED_REVIEW_PROMPT, enrollment.id for GROUP_ENROLLMENT_CONFIRMED; reviews -> review.id; contact -> message.id (per staff recipient).
3. DO NOT add keys to TEACHER_VERIFICATION_* (next prompt) nor to EMAIL_VERIFICATION / PASSWORD_RESET unless Prompt 1 Q2 proved a per-request stable id exists (then key on that id; otherwise leave them without a key and say so in the report).
4. Change nothing else (no payload/context/subject changes).
5. Tests: per domain at least one test that runs the notify task twice with the same id and asserts exactly one send per recipient (patch EmailMultiAlternatives.send or use the locmem outbox) and one EmailLog row per recipient.

ACCEPTANCE: all domain suites green; the diff only adds dedupe_key arguments/imports and tests.
```

### Prompt 4 — dedupe رویدادمحور برای تأیید معلم (backend)
```
TASK (BACKEND): one notification and one email per verification EVENT. Depends on Prompts 2-3 and Prompt 1 Q3.

1. Read teachers/services.py around the four dispatch points (submit_verification/approve/reject/suspend or their actual names), teachers/tasks.py (the four notify_verification_* tasks incl. their in-app and email blocks and the staff fan-out in notify_verification_submitted), TeacherVerificationEvent usage, and tests/teachers/.
2. In each service, capture the TeacherVerificationEvent created for that transition (the code already creates one inside the transaction; keep a reference) and dispatch notify_verification_*.delay(str(verification.id), str(event.id)) on_commit. If Prompt 1 Q3 showed the event is not in scope at some dispatch point, restructure minimally to keep the reference; do not change transition logic.
3. Each of the four tasks gets a TRAILING parameter event_id: str | None = None. When event_id is given: in-app dedupe_key = build_dedupe_key("<TYPE>", event_id) and email dedupe_key = build_email_dedupe_key("<TYPE>", event_id, <recipient id>); for notify_verification_submitted the staff fan-out keys are per staff recipient (same event id, different recipient) and the teacher confirmation email likewise. When event_id is None (messages queued before deploy): no dedupe key, exactly today's behavior.
4. Remove the "dedupe deferred" comments left in teachers/tasks.py by the first phase.
5. Tests: submit -> reject -> resubmit -> approve -> suspend -> approve -> suspend: each transition yields exactly one notification and one email per recipient (suspend twice yields TWO suspensions, both delivered); replaying the same task with the same event_id adds nothing; a call with the old single-argument signature works and is not deduped; staff fan-out is one per staff user per event.

ACCEPTANCE: teachers/emails/notifications suites green.
```

### Prompt 5 — شمارنده‌ی اعلان و پاسخ mark-all-read (backend)
```
TASK (BACKEND ONLY): add a light unread-count endpoint and fix the English mark-all-read message.

1. Read apps/notifications/views.py, urls.py, selectors.py, services.py (mark_all_notifications_read), tests/notifications/test_views.py, apps/common/jalali.py (to_persian_digits, if present from the email phase; otherwise write a tiny local digit translator in notifications/services or views and say so).
2. Add UnreadCountView (GET, IsAuthenticated) at path "unread-count/" (name "unread-count"), placed BEFORE the "<uuid:notification_id>/" route: returns {"unread_count": selectors.get_unread_count(user=request.user)}. Keep the list endpoint and its unread_count field untouched (backward compatible).
3. MarkAllNotificationsReadView.post returns {"marked_count": count, "message": "<Persian>"} where the message is «{n} اعلان به‌عنوان خوانده‌شده علامت‌گذاری شد» with n in Persian digits (use ZWNJ in «به‌عنوان» and «خوانده‌شده»; for count 0 use «اعلان خوانده‌نشده‌ای وجود نداشت»).
4. Tests: unauthenticated -> 401/403 (same as the list view); returns only the caller's count; another user's notifications are not counted; the number of DB queries is at most one COUNT plus the auth query(ies) (use django_assert_max_num_queries with the minimum you measure and explain) and no serialization of notification rows happens; mark-all-read response shape and Persian message (no Latin letters), count 0 case; the old list endpoint still returns unread_count.

ACCEPTANCE: notifications tests green; no change to the list endpoint's contract.
```

### Prompt 6 — فرانتند: استفاده از endpoint شمارنده
```
TASK (FRONTEND ONLY): use the new endpoint. The backend (Prompt 5) must be deployed first.

1. Read src/features/notifications/api/notifications.api.ts, types/notification.types.ts, hooks/useNotifications.ts (useUnreadCount, useMarkAllRead), the MSW handlers/fixtures that mock /api/notifications/, and every test touching useUnreadCount/unreadCount/markAllRead.
2. unreadCount() calls GET /api/notifications/unread-count/ and is typed Promise<AxiosResponse<{ unread_count: number }>>; useUnreadCount reads data.unread_count (query key, refetchInterval 30s and invalidation behavior unchanged). markAllRead() is typed {marked_count: number; message: string}; the hook keeps showing its own Persian toast from i18n (do not display the server message).
3. Update mocks/handlers and tests; add a test that the badge shows the count from the new endpoint and that the list endpoint is NOT called by useUnreadCount.

ACCEPTANCE: frontend tests, typecheck, lint, build pass.
```

### Prompt 7 — آمادگی strict mode: ابزار حسابرسی و runbook (backend)
```
TASK (BACKEND): tooling and documentation to decide on NOTIFICATIONS_STRICT_PAYLOAD in production. Do NOT change its default and do NOT enable it anywhere except test settings.

1. Read notifications/services.py (validation + logging added in the earlier phases), types.py (NOTIFICATION_SPECS, optional keys, FORBIDDEN_PAYLOAD_KEYS), docs/NOTIFICATIONS.md.
2. Make the two log lines stable and greppable: logger.error("notification_payload_incomplete type=%s missing=%s recipient_id=%s", ...) and logger.error("notification_payload_forbidden type=%s keys=%s recipient_id=%s", ...). Do not include payload values.
3. Create the management command audit_notification_payloads: options --since-days N (default 14), --type T (repeatable), --fail-on-violations. It iterates Notification rows with .iterator(chunk_size=2000) (read-only, no writes) and prints per type: total, missing-required-key counts by key, forbidden-key counts, and a schema_version distribution; plus the count of rows with an unknown type. Exit code 1 only when --fail-on-violations is set and violations > 0. It must not load rows older than --since-days.
4. docs/NOTIFICATIONS.md: add "Enabling strict mode" with the criteria (zero violations from audit_notification_payloads --since-days <days since the payload-contract release>; zero notification_payload_incomplete/forbidden events in logs/Sentry for 7 days; 3 days on staging with the flag on), the rollout (staging, then production via env var), the rollback (unset the env var), and the explicit consequence: with strict on, an invalid payload means NO notification is created and the exception is only visible in logs/Sentry because the tasks swallow it.
5. Tests: the command on a DB with valid rows, rows missing a required key, rows with a forbidden key, an unknown type; --fail-on-violations exit code; --since-days filtering; the log lines are emitted in non-strict mode (caplog).

ACCEPTANCE: tests green; setting defaults unchanged (assert it in a test).
```

### Prompt 8 — ریفاند گروهی خودترمیم (backend)
```
TASK (BACKEND): automatically retry stuck group-session refunds and alert once per run. Depends on Phase 5 and Prompt 1 Q6.

1. Read payments/services.refund_group_session_cancellation (idempotent, per-payment lock, GatewayError -> log + continue), payments/tasks.py (existing sweeps' style, e.g. reap_stale_pending_group_enrollment_payments and release_held_payments), config/celery.py beat_schedule, the setup_celery_beat command, sessions models (cancel timestamp field per Prompt 1 Q6), tests/payments.
2. Add @shared_task(name="payments.retry_stuck_group_refunds") in payments/tasks.py: select distinct sessions with session_type GROUP and status CANCELLED having at least one payment HELD_IN_ESCROW through enrollment (use the exact literals verified in Prompt 1); for each, call refund_group_session_cancellation(session=session) inside try/except (one session's failure must not stop the sweep). After all attempts recompute what is still HELD_IN_ESCROW; if any payment is still stuck AND its session was cancelled more than 2 hours ago (new setting GROUP_REFUND_STUCK_ALERT_AFTER_MINUTES default 120), emit ONE aggregated logger.error("stuck_group_refunds sessions=%d payments=%d oldest_session_id=%s", ...) per run (this reaches Sentry). Otherwise log INFO with counts. Do not change refund_group_session_cancellation itself and do not touch payment statuses in the sweep.
3. Register it in config/celery.py beat_schedule (crontab(minute="*/30")) AND in the setup_celery_beat command exactly like its siblings; add the route to CELERY_TASK_ROUTES if the siblings have one.
4. Tests: a cancelled group session with a held payment and a gateway that now succeeds -> the sweep refunds it and exactly one REFUND_PROCESSED (code GROUP_SESSION_CANCELLED) is created; running the sweep again adds nothing; a payment whose gateway still fails -> stays HELD_IN_ESCROW, no notification, and (caplog) one aggregated error only when older than the threshold (none when younger); one session raising does not stop another from being refunded; non-group and non-cancelled sessions are ignored; both beat registries contain the entry.

ACCEPTANCE: payments tests green; no duplicate notification on repeated sweeps.
```

### Prompt 9 — Regression و گزارش نهایی
```
TASK: verify the whole phase; make no feature changes; fix only defects introduced by this phase.

1. Backend, if Django/DB are available: makemigrations --check --dry-run; migrate a clean DB; migrate emails back one step and forward; full pytest; lint targets. Frontend: tests, typecheck, lint, build. List anything NOT RUN with the exact command.
2. Static proofs: (a) every send_email_task.delay call either passes dedupe_key or is listed as intentionally keyless with the reason (auth emails if no stable id); (b) the SENT bookkeeping is no longer able to raise out of send_transactional_email; (c) the four verification tasks only gained trailing default params; (d) the strict flag default is unchanged and not set in production settings; (e) the new beat entry exists in both registries; (f) the list endpoint contract is unchanged.
3. docs/NOTIFICATIONS.md: update "Known gaps": close idempotency of email, verification dedupe, unread-count, silent refund failures; keep open: refund_payment not cancelling booking/session (D1), NO_SHOW_REPORTED (D2), finance observations from Prompt 1 Q5 (state the verdicts), optional realtime/push.
4. Report: migrations (with sqlmigrate output if available), files changed, test counts before/after, per-event table "notification + email dedupe key as implemented", skipped items with reasons.
```
