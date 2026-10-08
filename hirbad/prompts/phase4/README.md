# Phase 4 — فارسی‌سازی کانال ایمیل

## Implementation Prompts

این بخش (قواعد کلی + راهنمای متن) را **بالای هر Prompt** paste کنید.

```
GLOBAL RULES (apply to this task; backend only unless stated):
- Repository: hirbad-platform (Django backend in hirbad-backend/). Do not touch hirbad-frontend/.
- FIRST read all code relevant to this task and summarize what you found in 5-10 lines. Line numbers quoted come from the original codebase and may have shifted; locate things by name.
- Minimal changes. No rewrites outside the files named, no renaming, no reformatting, no new dependencies. Keep the existing architecture (service layer, thin Celery tasks, send_email_task(subject, template_base, context, recipient_id, event_type) signature unchanged).
- Celery task signatures: only ADD trailing parameters with defaults.
- Create or update tests for everything you change. Run the relevant suites. If something cannot be run in your environment, say so explicitly (NOT RUN + exact command) instead of claiming success.
- At the end print: files changed, tests run + results, assumptions, discrepancies versus this prompt. If the code contradicts this prompt, STOP and report.

PERSIAN COPY STYLE GUIDE (for every template/subject you write):
- Polite, warm-formal register; address the reader as «شما»; gender-neutral; no emoji; no exclamation-heavy tone.
- Correct Persian orthography: ZWNJ (U+200C) in «می‌شود، به‌زودی، جلسه‌ی، ایمیل‌ها، نمی‌توانید»; Persian punctuation «،» «؛» «؟»; Persian digits via filters, never typed Latin digits.
- Brand: use {{ brand_name }} (value «هیربَد»), never the English name in visible text.
- Never print a variable that can be None/empty without {% if %} and a grammatical fallback sentence. NEVER output the word "None".
- Dates ONLY via |jdatetime / |jdate (input is an ISO string); amounts ONLY via |money:currency (Rial -> Toman). Never print raw ISO strings or raw amounts.
- User-authored text (names, messages, teacher notes, admin reasons) is displayed as-is inside an element with dir="auto"; system text must never be English.
- URLs and link text go in their own paragraph with dir="ltr" style="text-align:left;direction:ltr;word-break:break-all". Button labels are Persian.
- Keep the template structure: {% extends "emails/base.html" %} with blocks title/preheader/heading/content, the {% include "users/emails/_cta_button.html" %} pattern, all href values and Outlook conditional comments unchanged. Put {% load fa_format %} right after {% extends %}.
- The .txt template mirrors the HTML content in plain Persian (no HTML), URLs on their own lines.
- Keep every context variable name currently used (add new ones only where this prompt says). Do not change recipients, event_type values or dispatch logic.
```

### Prompt 1 — هسته‌ی تقویم شمسی
```
TASK: Add a dependency-free Gregorian<->Jalali conversion core with a verified golden test. Pure addition.

1. Read apps/common/ conventions, requirements/*.txt (confirm tzdata is present; do NOT add jdatetime/persiantools), and how tests under tests/common/ are organized.
2. Create apps/common/jalali.py with:
   - gregorian_to_jalali(gy, gm, gd) -> (jy, jm, jd) and jalali_to_gregorian(jy, jm, jd) (arithmetic 33-year-cycle algorithm, same as jalaali-js; valid for 1178..1633 AP; raise ValueError outside).
   - JALALI_MONTHS = فروردین، اردیبهشت، خرداد، تیر، مرداد، شهریور، مهر، آبان، آذر، دی، بهمن، اسفند; JALALI_WEEKDAYS (Python weekday() order, Monday=0) = دوشنبه، سه‌شنبه، چهارشنبه، پنجشنبه، جمعه، شنبه، یکشنبه (use ZWNJ in سه‌شنبه).
   - to_persian_digits(text) translating 0-9 to ۰-۹.
   - format_jalali_date(dt_or_date, tz=None) -> "۱۵ مهر ۱۴۰۵" and format_jalali_datetime(dt, tz) -> "چهارشنبه ۱۵ مهر ۱۴۰۵، ساعت ۱۴:۳۰": aware datetimes are converted to tz (ZoneInfo) BEFORE taking the date/weekday; naive datetimes are treated as UTC; time uses 24h HH:MM with Persian digits.
3. Golden data: generate a JSON fixture tests/common/fixtures/jalali_golden.json with EVERY day from 2024-03-01 to 2027-12-31 using Node (node must be available; use Intl.DateTimeFormat("en-US-u-ca-persian", {timeZone:"UTC", year:"numeric", month:"numeric", day:"numeric"}) with numeric parts so month/day/year are Latin digits; commit the generator script under scripts/ as well). If Node is unavailable, say so and fall back to this hand-verified set only: 2024-03-20=1403/1/1, 2025-03-20=1403/12/30, 2025-03-21=1404/1/1, 2026-03-21=1405/1/1, 2026-10-07=1405/7/15; mark the full-range test as NOT RUN.
4. Tests (tests/common/test_jalali.py): full-range golden comparison; round trip jalali_to_gregorian(gregorian_to_jalali(x)) == x; leap handling (30 اسفند 1403 valid, ۱۴۰۴ has 29); timezone edge: 2026-03-20T20:45:00Z in Asia/Tehran is 2026-03-21 00:15 (date rolls over, weekday follows the Tehran date); naive input; out-of-range raises.

ACCEPTANCE: tests green (or NOT RUN with reason); no dependency added; no existing file modified except requirements untouched.
```

### Prompt 2 — فیلترهای قالب و تنظیم منطقه‌ی زمانی
```
TASK: Expose Persian formatting to templates. Depends on Prompt 1.

1. Read config/settings/base.py (how settings are declared with config()), apps/emails/ layout, and confirm apps.emails is in INSTALLED_APPS.
2. Add to settings: EMAIL_DISPLAY_TIMEZONE: str = config("EMAIL_DISPLAY_TIMEZONE", default="Asia/Tehran").
3. Create apps/emails/templatetags/__init__.py and fa_format.py registering filters:
   - jdate, jdatetime: accept an ISO-8601 string (with or without offset, with trailing Z), a datetime or a date; use apps.common.jalali with ZoneInfo(settings.EMAIL_DISPLAY_TIMEZONE); on None/""/unparseable input return "" and never raise.
   - fa_digits: any value -> string with Persian digits.
   - toman: Rial amount (str/int/Decimal) -> "۱۵۰٬۰۰۰ تومان" (divide by 10, round half up to whole Toman, group thousands with the Persian thousands separator "٬"). Invalid -> "".
   - money(amount, currency): IRR -> toman; USD -> "۴۵٫۰۰ دلار" (2 decimals, Persian decimal separator "٫"); other non-empty currency -> Persian-digit number + space + currency code; invalid amount -> "".
   - rating5: numeric 1..5 -> "۴ از ۵"; otherwise "".
4. Tests (tests/emails/test_fa_format.py): each filter with valid, None, "", garbage, Z-suffixed and offset ISO strings; Tehran conversion (…T20:45:00Z -> next-day 00:15); toman rounding (e.g. 155 Rial -> ۱۶ تومان); render through a real template string with {% load fa_format %}.

ACCEPTANCE: tests green; no filter can raise on bad input.
```

### Prompt 3 — زیرساخت: base، subjectها، سرویس
```
TASK: Persian infrastructure for emails. Depends on Prompts 1-2. Do not rewrite per-event templates yet.

1. Read apps/emails/services.py, tasks.py, templates/emails/base.html, apps/users/templates/users/emails/_cta_button.html, tests/emails/test_services.py. Read the frontend brand string in hirbad-frontend/src/shared/i18n/locales/fa/common.json (brand.name) and any tagline key there; use the SAME brand string (read-only look at the frontend).
2. Create apps/emails/constants.py: BRAND_NAME_FA (exactly the frontend brand string) and BRAND_TAGLINE_FA (use the frontend tagline if one exists, else «بهترین معلم خود را پیدا کنید»).
3. Create apps/emails/subjects.py: EMAIL_SUBJECTS_FA dict keyed by event_type with the Persian subjects below, and build_subject(event_type, **params) that formats with a safe-format helper (missing/empty params use the stated fallback; unknown event_type returns a generic Persian subject «پیامی از {BRAND_NAME_FA}» and logs a warning).
   EMAIL_VERIFICATION «تأیید ایمیل شما در هیربَد»; PASSWORD_RESET «بازیابی رمز عبور هیربَد»; NEW_BOOKING_REQUEST «درخواست رزرو جدید از {student_name}» (fallback «درخواست رزرو جدید»); BOOKING_ACCEPTED «درخواست رزرو شما پذیرفته شد»; BOOKING_REJECTED «به‌روزرسانی درخواست رزرو شما»; BOOKING_EXPIRED «درخواست رزرو شما منقضی شد»; SESSION_REMINDER «یادآوری جلسه‌ی پیش‌رو»; SESSION_CANCELLED «جلسه‌ی شما لغو شد»; SESSION_COMPLETED_REVIEW_PROMPT «جلسه‌ی شما چطور بود؟»; GROUP_ENROLLMENT_CONFIRMED «جایگاه شما در جلسه‌ی گروهی تأیید شد»; PAYMENT_RECEIVED «پرداخت شما دریافت شد»; PAYMENT_FAILED «پرداخت شما انجام نشد»; REFUND_PROCESSED «مبلغ پرداختی شما بازگردانده شد»; PAYOUT_RELEASED «تسویه‌ی درآمد شما آغاز شد»; DISPUTE_OPENED «برای پرداخت شما اختلاف ثبت شد»; DISPUTE_RESOLVED «اختلاف پرداخت شما بررسی شد»; TEACHER_VERIFICATION_SUBMITTED «مدارک شما دریافت شد»; TEACHER_VERIFICATION_APPROVED «حساب معلمی شما تأیید شد»; TEACHER_VERIFICATION_REJECTED «به‌روزرسانی بررسی مدارک شما»; TEACHER_VERIFICATION_SUSPENDED «حساب معلمی شما تعلیق شد»; NEW_REVIEW_RECEIVED «نظر جدیدی برای شما ثبت شد»; NEW_CONTACT_MESSAGE «پیام جدید از فرم تماس: {subject}» (fallback «پیام جدید از فرم تماس»).
   Verify every event_type string against the real send_email_task.delay(... event_type=...) calls (grep) and report any mismatch instead of guessing.
4. services.py send_transactional_email: add to full_context current_year_jalali (Jalali year of now in EMAIL_DISPLAY_TIMEZONE, int), brand_name, brand_tagline (do not remove current_year or recipient_email); add headers={"Content-Language": "fa"} to EmailMultiAlternatives. Do not change the function signature or logging.
5. base.html: <html lang="fa" dir="rtl">, body dir="rtl", content cells text-align:right; font stacks 'Vazirmatn', Tahoma, 'Segoe UI', Arial, sans-serif everywhere (keep the existing inline-style structure); header text {{ brand_name }}; title default {{ brand_name }}; footer: «این ایمیل به آدرس {{ recipient_email }} ارسال شده است، چون این آدرس به یک حساب {{ brand_name }} متصل است.» (recipient_email in a dir="ltr" span) and footer_extra default «این یک اعلان ضروری حساب کاربری است؛ اگر کار شما نبوده است، نیازی به انجام کاری نیست.»; copyright «© {{ current_year_jalali|fa_digits }} {{ brand_name }} — {{ brand_tagline }}». Keep the mso conditionals, preheader padding and media queries. Update the header comment of the file.
6. _cta_button.html: keep structure; make text centered/RTL-safe (dir="rtl" on the table) and update the usage comment to a Persian label example.
7. Tests (tests/emails/test_services.py + new tests/emails/test_subjects.py): build_subject for every event incl. missing params; no Latin letters in any subject; sent message has Content-Language fa; context contains the new keys; base.html renders lang="fa" dir="rtl" with a trivial child template; decoded Subject header equals the Persian subject (use email.header.decode_header).

ACCEPTANCE: existing email service tests still pass; new tests green.
```

### Prompt 4 — ایمیل‌های حساب کاربری
```
TASK: Persian verify_email and password_reset. Depends on Prompt 3.

1. Read apps/users/tasks.py (both dispatch sites incl. the context keys), apps/users/templates/users/emails/verify_email.{html,txt}, password_reset.{html,txt}, and their tests (grep tests/users for subject/template assertions).
2. Rewrite both template pairs in Persian following the style guide. Required content: verify_email — greeting, why they received it, button «تأیید آدرس ایمیل», link fallback paragraph, link expiry (use the existing expiry variable if one exists, formatted with |fa_digits; if the context has no expiry, do not invent one), «اگر شما ثبت‌نام نکرده‌اید، این ایمیل را نادیده بگیرید». password_reset — greeting, request explanation, button «تنظیم رمز عبور جدید», link fallback, expiry (same rule), «اگر این درخواست از طرف شما نبوده است، نیازی به انجام کاری نیست و رمز عبور شما تغییری نمی‌کند».
3. In users/tasks.py replace the two subject literals with build_subject("EMAIL_VERIFICATION") / build_subject("PASSWORD_RESET"). Change nothing else.
4. Update/extend tests: subjects are Persian; HTML contains the link exactly once as href and once as text; txt contains the link; no "None".

ACCEPTANCE: users + emails tests green.
```

### Prompt 5 — ایمیل‌های رزرو و جلسه (۸ قالب)
```
TASK: Persian booking and session emails. Depends on Prompts 3-4 and Phase 3 (cancellation_code).

1. Read apps/bookings/tasks.py and apps/sessions/tasks.py (all send_email_task.delay calls and their contexts), the 8 template pairs in bookings/templates/bookings/emails/ (new_booking_request, booking_accepted, booking_rejected, booking_expired) and sessions/templates/sessions/emails/ (session_reminder, session_cancelled, session_completed, group_enrollment_confirmed), and sessions/tasks.py:~234 ("your student" fallback).
2. Rewrite the 8 template pairs. Required content per email:
   - new_booking_request (to teacher): student name, skill only if present, requested start (|jdatetime), CTA «مشاهده و پاسخ به درخواست».
   - booking_accepted: teacher, skill if present, start time, CTA «مشاهده رزرو».
   - booking_rejected: teacher, skill if present, the teacher's note (existing `reason` context, user text, shown only if non-empty, labeled «پیام معلم»), and a NEUTRAL refund sentence: «اگر برای این رزرو مبلغی پرداخت کرده‌اید، وضعیت بازپرداخت را در بخش پرداخت‌های خود ببینید.» (do NOT claim a refund happened), CTA.
   - booking_expired: teacher, skill if present, «این درخواست در مهلت مقرر پاسخی دریافت نکرد و منقضی شد»؛ the full refund statement is correct here (expiry only exists in the escrow flow); CTA.
   - session_reminder: used for BOTH student and teacher; read how the context differs by recipient and write role-aware text; start time (|jdatetime), other party name, skill if present, join/booking link if the context has one.
   - session_cancelled: teacher/other party, original start time if present, reason by cancellation_code: TEACHER_CANCELLED «جلسه توسط معلم لغو شد»; BOOKING_CANCELLED «رزرو مربوط به این جلسه لغو شد»; MIN_ENROLLMENT_NOT_MET «ظرفیت حداقلی جلسه‌ی گروهی تکمیل نشد»; missing/unknown code -> a generic sentence without a reason.
   - session_completed (review prompt): teacher name, CTA «ثبت نظر».
   - group_enrollment_confirmed: teacher, skill ONLY if present (Phase 3 allows None), start time, CTA.
3. In the tasks: replace every subject literal with build_subject(<event_type>, ...) (NEW_BOOKING_REQUEST passes student_name); replace the "your student" fallback with Persian (use «دانش‌آموز شما») in a way that keeps the template sentence grammatical; add cancellation_code to the session_cancelled email context (the task parameter already exists after Phase 3); keep all other context keys and values (ISO strings stay ISO — formatting happens in templates).
4. Tests: update tests/bookings, tests/sessions and tests/emails/test_e2e_matrix.py as needed; add per-template render tests with (a) full context, (b) skill_name=None, (c) empty names, (d) legacy context lacking the new keys; assert no "None", no "{{", no Latin letters in visible text (strip tags, URLs, emails).

ACCEPTANCE: bookings/sessions/emails tests green.
```

### Prompt 6 — ایمیل‌های پرداخت (۶ قالب)
```
TASK: Persian payment emails and removal of English prose from their contexts. Depends on Prompts 3-5 and Phase 3.

1. Read apps/payments/tasks.py (notify_payment_received, notify_payment_failed, notify_refund_processed, notify_payout_released, notify_dispute_opened, notify_dispute_resolved, _RESOLUTION_SUMMARIES/_resolution_summary) and the 6 template pairs in payments/templates/payments/emails/. Confirm Payment.failure_code and Dispute.split_teacher_percentage exist (Phase 3).
2. Rewrite the 6 template pairs. Required content:
   - payment_received (student): amount (|money:currency), teacher, session time if present, CTA to the payment.
   - payment_failed: amount, teacher, reason by failure_code: GATEWAY_DECLINED «بانک یا درگاه پرداخت تراکنش را تأیید نکرد»; TIMEOUT «مهلت پرداخت به پایان رسید»; missing/UNKNOWN -> no reason sentence; instruction to retry; CTA.
   - refund_processed: refund amount, teacher, note that the time to reach the bank account depends on the bank; CTA.
   - payout_released (teacher): payout amount, student name, CTA to teacher payments.
   - dispute_opened (both parties): explain a dispute was opened for a payment, that the team will review, CTA. Do NOT display the dispute reason text.
   - dispute_resolved: outcome by resolution code: RESOLVED_REFUND, RESOLVED_TEACHER, RESOLVED_SPLIT (with split_teacher_percentage: «X٪ مبلغ به معلم تعلق گرفت و مابقی بازگردانده شد» using |fa_digits), unknown -> «نتیجه‌ی بررسی نهایی شد»; CTA.
3. In the tasks: pass failure_code (payment.failure_code) instead of failure_reason; pass resolution (dispute.status) and split_teacher_percentage instead of resolution_summary; delete _RESOLUTION_SUMMARIES and _resolution_summary ONLY if nothing else uses them (grep first, report); replace subject literals with build_subject(...). Keep other keys.
4. Tests: per-template render tests (full, minimal, legacy context with only the OLD keys incl. failure_reason/resolution_summary -> must render without error, without None, without printing the old English text); money formatting assertions; update payments tests.

ACCEPTANCE: payments + emails tests green; grep shows no English prose left in email contexts of payments tasks.
```

### Prompt 7 — معلم، نظر، تماس (۶ قالب)
```
TASK: Persian teacher-verification, review and contact emails. Depends on Prompts 3-6.

1. Read apps/teachers/tasks.py, apps/reviews/tasks.py, apps/contact/tasks.py (email dispatches and contexts) and the 6 template pairs: teachers/emails/verification_submitted|approved|rejected|suspended, reviews/emails/new_review_received, contact/emails/new_contact_message_admin.
2. Rewrite them. Required content: verification_submitted (to the teacher) acknowledges receipt and says review is in progress; approved — congratulation, what happens next, CTA; rejected — the admin's rejection_reason (user/admin text, dir="auto", only if non-empty, labeled «دلیل») and how to resubmit, CTA to /teacher/verification link already in the context; suspended — admin's suspension_reason block if non-empty and how to contact support if the context has a support address (do not invent one); new_review_received — student name, rating via |rating5, the comment (dir="auto", only if non-empty); new_contact_message_admin (to staff) — sender name, sender email (dir="ltr"), subject, message body (dir="auto"), inbox CTA (keep the existing forward-looking URL).
3. Replace subject literals with build_subject(...) (NEW_CONTACT_MESSAGE passes the user's subject; NEW_REVIEW_RECEIVED has no params). Update tests/reviews/test_tasks.py:~162 (it asserts the old English subject) and any teachers/contact tests.
4. Tests: render tests as in the previous prompts (full, empty optional fields, legacy context).

ACCEPTANCE: teachers/reviews/contact/emails tests green.
```

### Prompt 8 — گارد جامع، پیش‌نمایش، سند متن، Regression
```
TASK: Whole-channel guard tests, a preview command, a copy document, and the final verification. Depends on Prompts 1-7. No feature changes; fix only defects from this phase.

1. Guard test tests/emails/test_persian_emails_contract.py: discover every template pair under apps/**/templates/**/emails/ (excluding base and partials) and render each with (a) a realistic full context, (b) an empty context {}, (c) a context with every value None. Assert for html and txt: no "None", no "{{" / "}}", no ISO timestamp pattern (\d{4}-\d{2}-\d{2}T), visible text (strip tags, <style>, comments, URLs, emails, and elements marked dir="auto") contains no [A-Za-z]; html has lang="fa" and dir="rtl". Also assert that every event_type used in a send_email_task.delay call (AST scan) exists in EMAIL_SUBJECTS_FA.
2. End-to-end: extend tests/emails/test_e2e_matrix.py with a helper asserting for every outbox message: Persian subject without Latin letters, Content-Language fa, multipart with both parts.
3. Management command apps/emails/management/commands/render_email_previews.py: writes <out>/<event>.html and .txt for every email using sample contexts defined in a single module (apps/emails/preview_samples.py), with realistic Persian sample names and one variant with optional fields missing; no sending. Add a test that it runs.
4. Generate docs/EMAIL_COPY_FA.md from the real templates/subjects via a small script (scripts/dump_email_copy.py), listing per event: subject, preheader, heading, body text (txt version) — for human proof-reading.
5. Run: makemigrations --check (should report nothing), the full backend suite, the repo's lint targets. List anything NOT RUN with the exact command.
6. Report: files changed, test counts before/after, grep proofs (no `resolution_summary`/`failure_reason`/"your student" left in email contexts; no English subject literals left), and the list of events whose preview you could render.

ACCEPTANCE: all green; preview command works; copy document committed.
```
