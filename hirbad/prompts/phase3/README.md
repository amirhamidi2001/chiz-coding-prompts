# Phase 3 — Payload ساختاریافته و رفع باگ `skill.name`

## Implementation Prompts

این بخش را **بالای هر Prompt** paste کنید.

```
GLOBAL RULES (apply to this task):
- Repository: hirbad-platform (Django backend in hirbad-backend/, React frontend in hirbad-frontend/). Each prompt states whether it is backend-only or frontend-only; do not touch the other side.
- FIRST read all code relevant to this task and summarize what you found in 5-10 lines before changing anything. Line numbers quoted come from the original codebase and may have shifted after earlier phases; locate things by name.
- Minimal changes only. No rewrites, no renaming, no reformatting of untouched code, no new dependencies.
- Preserve the architecture: service layer, Celery tasks with lazy imports, try/except-log-swallow convention.
- Celery task signatures: you may only ADD trailing parameters with defaults. Never remove, reorder or rename existing parameters (messages already in the queue use the old signature).
- Create or update tests for everything you change. Run the relevant suites and fix failures you caused. If something cannot be run in your environment, say so explicitly instead of claiming success.
- At the end print: files changed, tests run + results, assumptions, and any discrepancy versus this prompt. If the code contradicts this prompt, STOP and report.
```

### Prompt 1 — Hotfix: `session.skill` می‌تواند null باشد (backend، مستقل)
```
TASK (HOTFIX): Session.skill is a nullable FK (on_delete=SET_NULL). notify_group_enrollment_confirmed in apps/sessions/tasks.py reads session.skill.name twice (in the create_notification payload ~line 449 and in the send_email_task context ~line 472); with skill=None both raise AttributeError inside their try blocks, so the in-app notification AND the email are silently lost.

1. Read that task, the Session/SessionEnrollment models, tests/sessions/test_tasks.py and any existing group-enrollment tests (grep notify_group_enrollment_confirmed in tests/).
2. Before the two try blocks compute once: skill_name = session.skill.name if session.skill else None (same idiom as apps/bookings/tasks.py). Use skill_name in BOTH the payload and the email context. Change nothing else in the task.
3. grep apps/ (excluding tests/migrations/serializers/views/admin) for other `.skill.` attribute chains on a nullable FK inside tasks/services and list each as safe/unsafe in your report; fix only unsafe ones that are the same bug (a nullable skill FK dereferenced without a guard), report anything else without changing it.
4. Tests: (a) skill present -> payload and email context carry the skill name exactly as before; (b) skill=None -> exactly one GROUP_ENROLLMENT_CONFIRMED Notification is created (payload skill_name is None) and send_email_task.delay is called once with skill_name None; no exception raised.

ACCEPTANCE: new tests pass; existing sessions/email tests pass; diff touches only the task and tests.
```

### Prompt 2 — کدها و `Payment.failure_code` (backend)
```
TASK: Introduce machine-readable failure codes. Depends on nothing but is a prerequisite for later prompts.

1. Read apps/common/ (conventions), apps/payments/models.py (Payment.failure_reason), the gateway-failure handler in apps/payments/services.py (the line that sets payment.failure_reason = reason or "Card declined."), the reaper in apps/payments/tasks.py (reap_stale_pending_group_enrollment_payments, failure_reason "Payment not completed within..."), and grep apps/ for every place that sets Payment.status = "FAILED" or assigns failure_reason.
2. Create apps/common/codes.py:
   class PaymentFailureCode(models.TextChoices): GATEWAY_DECLINED, TIMEOUT, UNKNOWN (value == name).
   class SessionCancellationCode(models.TextChoices): TEACHER_CANCELLED, BOOKING_CANCELLED, MIN_ENROLLMENT_NOT_MET (value == name).
3. Add Payment.failure_code = models.CharField(max_length=32, choices=PaymentFailureCode.choices, blank=True, default=""). Generate ONE migration; check sqlmigrate shows a plain ADD COLUMN with a constant default (no table rewrite) and that it is reversible.
4. Set failure_code next to every place failure_reason is set: gateway handler -> GATEWAY_DECLINED; reaper -> TIMEOUT; any other FAILED transition you find -> UNKNOWN. Keep failure_reason text exactly as is (it is for admins/audit). Include failure_code in the same save(update_fields=[...]) calls.
5. Tests: each failure path stores the right code and still stores the old text.

ACCEPTANCE: makemigrations --check clean, reversible migration, payments tests green.
```

### Prompt 3 — عبور `cancellation_code` (backend)
```
TASK: Carry a cancellation code from the cancel flows to the session-cancelled notification. Depends on Prompt 2 (codes module).

1. Read apps/sessions/services.py: cancel_session (its transaction.on_commit dispatch of notify_session_cancelled), cancel_session_from_booking (calls cancel_session with reason "The related booking was cancelled."), and apps/sessions/tasks.py: cancel_undersubscribed_group_sessions (calls cancel_session with reason "Minimum enrollment was not reached.") and notify_session_cancelled (payload + email). Grep every caller of cancel_session and of notify_session_cancelled (including tests).
2. cancel_session gets a new keyword-only parameter cancellation_code: str = SessionCancellationCode.TEACHER_CANCELLED and passes it to notify_session_cancelled.delay(session_id, cancellation_code). notify_session_cancelled gets a new TRAILING parameter cancellation_code: str = "TEACHER_CANCELLED" (default keeps old queued messages valid).
3. cancel_session_from_booking passes BOOKING_CANCELLED; cancel_undersubscribed_group_sessions passes MIN_ENROLLMENT_NOT_MET. Keep the existing English `reason` strings (they feed the internal status log only).
4. In notify_session_cancelled add to the in-app payload: "cancellation_code": <value> and "start_time": session.start_time.isoformat(). Do not touch the email context.
5. Validate the code defensively: unknown values are replaced by TEACHER_CANCELLED? NO - keep the value as given but log a warning if it is not in SessionCancellationCode.values.
6. Tests: each of the three paths yields the right code in the notification payload; calling notify_session_cancelled(session_id) with ONE argument still works (old signature) and defaults to TEACHER_CANCELLED.

ACCEPTANCE: existing session tests green; new tests green; no caller broken.
```

### Prompt 4 — ماژول payload builder (backend)
```
TASK: Create typed payload builders. Do not change any task yet. Depends on Phase 0 (apps/notifications/types.py exists).

1. Read apps/notifications/types.py (specs), docs/NOTIFICATIONS.md, and EVERY create_notification call in apps/*/tasks.py to record the exact payload keys each type sends today.
2. Create apps/notifications/payloads.py with PAYLOAD_SCHEMA_VERSION = 2; helpers user_display_name(user) -> str | None (full_name stripped, None if blank), iso(dt) -> str | None, money_str(value) -> str; and ONE builder function per notification type (20), each taking already-loaded model instances (no extra queries; document which relations must be select_related) and returning a dict.
3. Each builder must emit a SUPERSET of the keys the corresponding task sends today, plus "schema_version": PAYLOAD_SCHEMA_VERSION, with these exact changes: PAYMENT_FAILED: remove failure_reason, add failure_code (payment.failure_code or "UNKNOWN"); DISPUTE_RESOLVED: remove resolution_summary, add split_teacher_percentage (int for RESOLVED_SPLIT else None); DISPUTE_OPENED: remove reason; SESSION_CANCELLED: add cancellation_code and start_time; GROUP_ENROLLMENT_CONFIRMED: skill_name is None when the skill is missing. Everything else unchanged (including user-authored reason/subject fields on BOOKING_REJECTED, TEACHER_VERIFICATION_REJECTED/SUSPENDED, NEW_CONTACT_MESSAGE).
4. Tests (tests/notifications/test_payloads.py): per type, build from factory objects and assert the exact key set and value types; a null-skill test; a split-percentage test; a test that no builder output contains the keys failure_reason, resolution_summary, summary, message.

ACCEPTANCE: pure additions; tests green; a table in your report mapping type -> keys.
```

### Prompt 5 — استفاده از builderها در تسک‌ها (backend)
```
TASK: Replace the hand-written payload dicts with the builders. Depends on Prompts 2-4.

1. Read each task below and the builder it must use.
2. In bookings/tasks.py, sessions/tasks.py, payments/tasks.py, reviews/tasks.py, contact/tasks.py, teachers/tasks.py replace ONLY the payload={...} dict of each create_notification call with the matching builder (lazy import inside the task, like the other imports). Do not change recipients, dedupe_key arguments, email contexts, subjects, try/except structure, logging or task signatures (except what Prompt 3 already did).
3. For PAYMENT_FAILED, DISPUTE_OPENED, DISPUTE_RESOLVED make sure the objects the builder needs are already loaded by the task (no new queries; if one is missing, add it to the existing select_related).
4. Email contexts stay exactly as they are (they still use English text; that is a later phase). In particular payments/tasks.py _resolution_summary stays for the email path.
5. Tests: update existing task tests whose payload assertions changed; add one test per changed type asserting the final Notification.payload equals the contract (key set), and that the email context of the three changed tasks is byte-for-byte unchanged.
6. grep backend serializers/views/selectors and frontend src for any reader of payload.failure_reason, payload.resolution_summary or DISPUTE_OPENED payload.reason; report what you find (expected: none).

ACCEPTANCE: all task tests green; no email context changed; grep report attached.
```

### Prompt 6 — Spec و گارد کلیدهای ممنوعه (backend)
```
TASK: Record optional keys in the specs and reject English-prose keys at creation time. Depends on Prompts 4-5.

1. Read apps/notifications/types.py (NotificationSpec, NOTIFICATION_SPECS), services.py create_notification (the strict/non-strict validation from Phase 0).
2. Add optional_payload_keys: tuple[str, ...] = () to NotificationSpec and fill it per type from the contract: schema_version for all; skill_name (booking types, GROUP_ENROLLMENT_CONFIRMED), reason (BOOKING_REJECTED), failure_code + session_id + session_start_time (PAYMENT_FAILED), split_teacher_percentage (DISPUTE_RESOLVED), cancellation_code + start_time (SESSION_CANCELLED), rejection_reason (VERIFICATION_REJECTED), suspension_reason (VERIFICATION_SUSPENDED), subject (already required for NEW_CONTACT_MESSAGE), plus whatever optional keys the current builders emit.
3. Add FORBIDDEN_PAYLOAD_KEYS = frozenset({"failure_reason", "resolution_summary", "summary", "message"}) and in create_notification: if any is present -> strict mode raises InvalidNotificationPayload, otherwise logger.error (same policy as incomplete payloads). Do not change the strict flag defaults.
4. Update docs/NOTIFICATIONS.md: payload contract table (required, optional, codes), rule "payload carries data and codes, never system-authored prose", schema_version meaning, and the list of user-authored text keys.
5. Tests: forbidden key raises in strict and logs in non-strict; optional keys never trigger "incomplete"; a contract test that every key emitted by every builder is in required or optional of its spec (so spec and builders cannot drift).

ACCEPTANCE: full backend suite green.
```

### Prompt 7 — نگاشت کدها به فارسی (frontend، وابسته به Phase 1)
```
TASK (FRONTEND ONLY): Render the new payload codes in Persian inside the Phase 1 registry. Backend must already be deployed or merged.

1. Read src/features/notifications/lib/notificationRegistry.ts, lib/payload.ts, fa/notifications.json, the registry tests, and the fixtures (mockAllNotificationTypeFixtures).
2. Add Persian copy under "summary" in fa/notifications.json and use it from the registry:
   - PAYMENT_FAILED by payload.failure_code (combine with the amount variant as today): GATEWAY_DECLINED -> "پرداخت شما به مبلغ {{amount}} ناموفق بود؛ بانک یا درگاه پرداخت تراکنش را تأیید نکرد" (no-amount variant: "پرداخت شما ناموفق بود؛ بانک یا درگاه پرداخت تراکنش را تأیید نکرد"); TIMEOUT -> "پرداخت شما به‌دلیل پایان مهلت پرداخت انجام نشد"; UNKNOWN/missing/unrecognized -> the existing Phase 1 texts.
   - SESSION_CANCELLED by payload.cancellation_code: TEACHER_CANCELLED -> "جلسه شما با {{teacherName}} توسط معلم لغو شد"; BOOKING_CANCELLED -> "رزرو مربوط به جلسه شما با {{teacherName}} لغو شد"; MIN_ENROLLMENT_NOT_MET -> "جلسه گروهی {{teacherName}} به‌دلیل کامل‌نشدن حداقل ظرفیت لغو شد"; missing/unrecognized -> the existing Phase 1 text.
   - DISPUTE_RESOLVED with resolution RESOLVED_SPLIT and an integer split_teacher_percentage between 1 and 99 -> "اختلاف پرداخت بررسی شد؛ {{percent}}٪ مبلغ به معلم تعلق گرفت و مابقی بازگردانده شد" (percent formatted with Intl.NumberFormat("fa-IR")); otherwise the Phase 1 split text.
3. Never render failure_reason, resolution_summary or reason. Unknown codes must fall back, never throw, never show English or the raw code.
4. Tests: every code, an unknown code, a missing code, and an old-style payload (without codes) for the three types; extend the Persian-only check (no [A-Za-z]) to these variants. Add the new backend-shaped payloads to mockAllNotificationTypeFixtures WITHOUT altering the legacy fixtures/lists.

ACCEPTANCE: frontend tests, typecheck, lint and build pass.
```

### Prompt 8 — Regression و گزارش نهایی
```
TASK: Verify the phase end to end; fix only defects introduced by this phase.

1. Backend, if Django/DB are available: makemigrations --check --dry-run; migrate on a clean DB; migrate payments back one step and forward; full pytest; the repo's lint targets. Frontend: tests, typecheck, lint, build. Anything you cannot run must be listed as NOT RUN with the exact command.
2. Static proofs: no remaining `.skill.name` without a guard in apps/**/tasks.py; no create_notification call builds its payload inline (all use apps/notifications/payloads builders); no builder emits failure_reason/resolution_summary/summary/message; notify_session_cancelled and every other changed task only gained trailing default parameters; email contexts of changed tasks are unchanged.
3. Report: migrations (with sqlmigrate output if available), files changed, test counts before/after, the type -> keys contract table as implemented, skipped items with reasons.
```
