# Afra — Audit معماری و طراحی Target Architecture برای Online Private Tutor Marketplace

> این سند بر اساس بررسی واقعی کد Repository (`afra-platform-main.zip`) تهیه شده — نه فرض یا حدس.
> Backend: Django 5.2 + DRF + Postgres + Celery + SimpleJWT
> Frontend: React 19 + TypeScript + Vite + TanStack Query + Tailwind + shadcn/radix

---

## 0. جمع‌بندی یک‌خطی

کد فعلی یک **backend مهندسی‌شده و بالغ** برای بخش «رزرو ۱:۱ + Escrow Payment + Dispute + Payout Ledger» است — سطح کیفیت آن (constraint سطح دیتابیس برای overlap، Exclusion Constraint، append-only PaymentEvent/PayoutLedgerEntry، idempotency، reconciliation، dispute resolution) واقعاً production-grade است و **نباید دور ریخته شود**. اما محصول هنوز به مدل «Marketplace فارسی» نرسیده: **Stripe/USD hard-coded**، **بدون Teacher Verification**، **بدون Review/Rating**، **بدون RTL/Jalali/فارسی‌سازی**، **بدون Meeting Provider واقعی (Skyroom)**، و **Email Matrix بسیار ناقص** (فقط ۲ نوع ایمیل واقعی از حدود ۱۵ رویداد لازم).

استراتژی درست: **Refactor حول لایه‌های abstraction (Payment Gateway / Meeting Provider / Money)** + **Add دامنه‌های گم‌شده (Verification, Reviews, Emails)**، نه بازنویسی.

---

## 1. Current Architecture

### 1.1 ساختار واقعی

```text
afra-backend/
  apps/
    users/        → User, StudentProfile, TeacherProfile, tokens (verify/reset), JWT auth
    teachers/      → TeacherSkill, AvailabilitySlot (weekly recurring), Stripe Connect onboarding
    bookings/      → BookingRequest (state machine کامل + DB-level overlap constraint)
    sessions/      → Session, SessionEnrollment, SessionStatusLog
    payments/      → Payment, PaymentEvent, PlatformCommission, PayoutLedgerEntry,
                     CancellationPolicy, Dispute (+ Stripe webhook receiver)
    notifications/ → Notification (فقط in-app، نه ایمیل)
    common/        → permissions, pagination, exceptions, health checks, logging
```

هیچ app جداگانه‌ای برای: `reviews`, `verification`, `meetings`, `emails`, `moderation`, `admin (custom)`, `catalog` وجود ندارد.

### 1.2 نقاط قوت واقعی (باید حفظ شوند)

| حوزه | چرا قوی است |
|---|---|
| **Booking concurrency** | دو لایه محافظت از overlap: `select_for_update` در سرویس (Layer 1) + Postgres `ExclusionConstraint` با `btree_gist` روی `tstzrange(requested_start, requested_end)` (Layer 2). این دقیقاً همان چیزی است که اکثر مارکت‌پلیس‌ها اشتباه پیاده می‌کنند. |
| **Payment ledger** | `Payment` → `PaymentEvent` (append-only audit trail هر state transition) → `PayoutLedgerEntry` (append-only، با `CLAWBACK` برای dispute) — طراحی صحیح double-entry-like برای پول. |
| **Escrow flow** | `capture_payment_for_booking` → `HELD_IN_ESCROW` → `release_payment_to_teacher` بعد از `COMPLETED` + hold window → commission محاسبه و freeze می‌شود در لحظه release (نه در لحظه تسویه بعدی). |
| **Idempotency** | `idempotency_key` روی Payment، `gateway_charge_id` unique، webhook handler که duplicate delivery را با `stripe_dispute_id` unique تشخیص می‌دهد. |
| **State machine صریح** | `BookingRequest.STATUS`, `Payment.STATUS`, `Session.STATUS/OUTCOME` با کامنت‌های دقیق درباره چرایی هر انتقال — کد خودمستند است. |
| **Celery sweep jobs** | `expire_stale_booking_requests`, `reap_stale_pending_payment_bookings`, `release_held_payments`, `reconcile_stuck_payments`, `scan_and_update_session_statuses`, `auto_mark_ongoing/completed` — چرخه کامل lifecycle بدون دخالت دستی. |
| **DB indexing** | ایندکس‌های ترکیبی هدفمند (`GinIndex` روی trigram برای جستجوی نام، composite index برای query الگوهای واقعی) — نه ایندکس‌گذاری کورکورانه. |
| **CI/CD** | GitHub Actions جدا برای backend/frontend + workflow deploy، Docker چندمرحله‌ای برای dev/prod. |
| **تست** | ۴۱ فایل تست backend (service-layer محور)، تست‌های frontend با MSW برای mock API. |

### 1.3 نقاط ضعف / خلأهای اصلی (خلاصه — جزئیات در Gap Analysis)

1. **Stripe به‌صورت مستقیم و پراکنده در ۶ فایل payments** کوپل شده (نه پشت یک Gateway abstraction).
2. **ارز پیش‌فرض `USD`** در `TeacherProfile.currency`, `Payment.currency` و `Intl.NumberFormat("en-US", ...)` در فرانت.
3. **بدون هیچ Teacher Verification workflow** — `TeacherProfile` هیچ فیلد status ندارد؛ لیستینگ عمومی مدرس‌ها (`GET /api/teachers/`) فقط `role=TEACHER, is_active=True` را فیلتر می‌کند، یعنی **هر مدرس ثبت‌نامی بلافاصله در Marketplace دیده می‌شود**.
4. **بدون هیچ Review/Rating model** — نه در backend، نه در frontend. فیلد `TeacherProfile.average_rating` وجود دارد ولی هیچ کد فعلی آن را update نمی‌کند (dead field).
5. **بدون Meeting Provider واقعی** — `session.meet_link` یک placeholder ساختگی است (`f"https://meet.afra.example.com/{session.id}"`)، هیچ ارتباطی با Skyroom یا هر سرویس واقعی ندارد.
6. **Email Matrix بسیار ناقص** — فقط `verify_email` و `password_reset` قالب HTML واقعی دارند (`apps/users/templates/users/emails/`) و یک `send_mail` ساده در `sessions/tasks.py` برای یادآوری جلسه. بقیهٔ رویدادهای حیاتی (booking accepted/rejected، payment received/failed، refund، payout، dispute، teacher approval...) فقط `Notification` درون‌برنامه‌ای می‌سازند، **هیچ ایمیلی ارسال نمی‌شود**.
7. **بدون i18n/RTL/Jalali در frontend** — هیچ کتابخانه i18n، هیچ `dir="rtl"`، `index.html` روی `lang="en"`، هیچ کتابخانه تقویم جلالی (`moment-jalaali`, `dayjs-jalali`, ...) در `package.json` نیست. `date-fns` (میلادی) استفاده شده.
8. **بدون Group Session واقعی** — مدل `Session` فیلدهای `session_type=GROUP`, `max_students`, `enrolled_count` و مدل `SessionEnrollment` را دارد (پایه‌ی داده آماده است)، اما هیچ Booking flow یا API عمومی برای ساخت/کشف/join یک Group Session به‌صورت مستقل از یک Booking ۱:۱ وجود ندارد — فقط `join_session`/`leave_session` روی یک Session موجود کار می‌کند.
9. **Admin استاندارد Django** — فقط `ModelAdmin` پایه برای مدل‌های موجود؛ بدون Verification review UI، بدون Reports dashboard، بدون Audit Log جامع (فقط `SessionStatusLog` و `PaymentEvent` به‌عنوان audit trail دامنه‌محور وجود دارند، نه یک Audit Log سراسری برای اقدامات ادمین).

---

## 2. Gap Analysis

| Feature | Current State | Required Change | Priority |
|---|---|---|---|
| Payment Gateway | Stripe مستقیم در `services.py`/`webhooks.py`/`tasks.py` (۳۵+ ارجاع)؛ `PAYMENTS_USE_STRIPE` flag برای mock/real | طراحی `PaymentGatewayProvider` interface (`create_intent`, `verify`, `refund`) و پیاده‌سازی Zarinpal/IDPay/... پشت آن؛ Stripe یا حذف یا به یک provider اختیاری تبدیل شود | **P0** |
| ارز | `Payment.currency`/`TeacherProfile.currency` = رشته آزاد ۳ کاراکتری، پیش‌فرض `USD`؛ فرانت با `Intl.NumberFormat("en-US")` | پیش‌فرض `IRT`/`IRR` مشخص، `Money` value object (Decimal + currency) در backend، فرمت تومان در frontend، رفع خطر گرد‌کردن (rial ندارد decimal، تومان معمولاً بدون اعشار) | **P0** |
| Teacher Verification | `Partially Implemented` → در واقع **Not Implemented**؛ فقط `is_active` بولین ساده | افزودن state machine `PENDING → UNDER_REVIEW → APPROVED / REJECTED / SUSPENDED`، مدل مدارک، Admin review UI، فیلتر marketplace فقط روی `APPROVED` | **P0** |
| Review / Rating | `Not Implemented` — نه model نه UI؛ `average_rating` فیلد مرده | مدل `Review` (یک‌به‌یک با `BookingRequest` تکمیل‌شده)، محاسبه `average_rating` با سیگنال/task، moderation flag | **P0** |
| Meeting Provider (Skyroom) | `Not Implemented` — `meet_link` رشته ساختگی، بدون هیچ API call | `MeetingProvider` abstraction (`create_room`, `get_join_url`, `revoke_access`)، پیاده‌سازی Skyroom، ذخیره room id/token، access control بر اساس نقش و زمان | **P0** |
| Email Matrix | فقط verify_email + password_reset (HTML واقعی) + یک ایمیل ساده یادآوری جلسه. سایر ۱۰+ رویداد فقط Notification درون‌برنامه‌ای | طراحی کامل Transactional Email Matrix (بخش ۳.۴)، افزودن یک `apps/emails` سرویس مرکزی با template فارسی + retry/failure handling | **P0** |
| فارسی‌سازی UI | تمام رشته‌ها hardcoded انگلیسی؛ بدون i18n lib | افزودن `react-i18next`/`next-intl`-style (یا حداقل i18n key-based dictionary فارسی)، فارسی‌سازی validation/error backend (DRF error messages هم انگلیسی پیش‌فرض هستند) | **P0** |
| RTL | `Not Implemented` — بدون `dir="rtl"`، بدون RTL-aware Tailwind/logical properties | افزودن `dir="rtl"` سراسری، بازبینی همه componentهای `shared/components` برای margin/padding جهت‌دار (`ms-`/`me-` به‌جای `ml-`/`mr-`) | **P0** |
| تقویم جلالی | `Not Implemented` — `date-fns` میلادی، بدون کتابخانه جلالی | افزودن کتابخانه جلالی (`jalaali-js`/`dayjs` + پلاگین) فقط در presentation layer؛ backend بدون تغییر (UTC/timezone-aware که همین الان هم درست است) | **P0** |
| Timezone ایران | Backend: `USE_TZ=True`, `TIME_ZONE=UTC` (درست). Frontend: بدون تبدیل صریح به `Asia/Tehran` | نمایش زمان‌ها با `Asia/Tehran` در UI (با `date-fns-tz` که همین الان نصب است)، بدون تغییر ذخیره‌سازی backend | **P1** |
| Group Session | `Partially Implemented` — schema آماده (`session_type`, `max_students`, `SessionEnrollment`) اما بدون flow کشف/رزرو مستقل از booking ۱:۱ | معماری کامل در بخش ۳.۵؛ اجرا بعد از پایدارشدن ۱:۱ (طبق خواسته کارفرما) | **P2** |
| Admin پنل | `ModelAdmin` استاندارد برای مدل‌های موجود؛ بدون verification review، بدون گزارش‌گیری، بدون audit سراسری | افزودن اکشن‌های سفارشی Admin برای Approve/Reject Teacher، Resolve Review flags، dashboard گزارش (می‌تواند با Django Admin سفارشی یا یک پنل جدا) | **P1** |
| Dispute UI (Admin) | مدل `Dispute` و `resolve_dispute` سرویس کامل هستند؛ فقط Admin پایه دارد | بهبود Admin UI برای resolve dispute با نمایش کامل context (نه فقط raw fields) | **P2** |
| Idempotency پرداخت | `idempotency_key` فیلد وجود دارد ولی طبق کامنت خود کد **«not yet read or written by any code path»** | واقعاً به کار گرفتن idempotency_key در endpoint create payment | **P0** |
| Rate limiting | `DEFAULT_THROTTLE_CLASSES`/`RATES` در DRF تنظیم شده (سطح پایه موجود) | بازبینی نرخ‌ها برای endpointهای حساس (auth, payment callback) | **P1** |
| Audit Log عمومی ادمین | فقط دامنه‌محور (`PaymentEvent`, `SessionStatusLog`) | افزودن `AdminAuditLog` سراسری برای اکشن‌های حساس ادمین (verification decision، dispute resolution، manual refund) | **P1** |
| CI/CD | Workflow کامل backend/frontend + deploy | افزودن i18n lint/RTL visual regression check (اختیاری)، افزودن secrets برای گیت‌وی ایرانی | **P2** |

---

## 3. Target Architecture

### 3.1 الگو: Modular Monolith (نه Microservices)

با توجه به مقیاس فعلی و تیم کوچک، ادامه دادن روی **Django Modular Monolith** درست‌ترین تصمیم است. کد فعلی همین الگو را به‌خوبی رعایت می‌کند (`models / selectors / services / serializers / views / tasks` در هر app). این ساختار باید حفظ و به دامنه‌های جدید تعمیم داده شود.

### 3.2 نقشهٔ دامنه‌ها (Domain Map)

```text
┌─────────────────────────────────────────────────────────────────┐
│                         afra-backend                             │
│                                                                    │
│  Users ──────┬── Students                                        │
│              └── Teachers ── Verification (NEW)                  │
│                                                                    │
│  Catalog (skills, subjects — از teachers جدا می‌شود در آینده)     │
│  Availability (از teachers جدا می‌شود در فاز بعد)                │
│                                                                    │
│  Bookings ──depends-on──► Users, Catalog, Availability            │
│      │                                                            │
│      ▼ (accept)                                                   │
│  Sessions ──depends-on──► Bookings, Meetings (NEW)                │
│      │                                                            │
│      ▼ (complete)                                                  │
│  Reviews (NEW) ──depends-on──► Sessions, Bookings                 │
│                                                                    │
│  Payments ──depends-on──► Bookings, Users                         │
│      │  uses PaymentGatewayProvider (abstraction, NEW)            │
│      ▼                                                            │
│  Payouts (از payments جدا می‌شود در فاز بعد — PayoutLedgerEntry)  │
│                                                                    │
│  Notifications (in-app) ──┐                                       │
│  Emails (NEW, transactional) ──┴── هر دامنه رویداد می‌فرستد به این دو │
│                                                                    │
│  Moderation (NEW) ── Review flag، Teacher document review          │
│  Admin (custom actions روی همهٔ دامنه‌ها)                          │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 جدول دامنه‌ها با جزئیات

#### Users (KEEP, حداقل تغییر)
- **Responsibility:** احراز هویت، نقش (Student/Teacher)، پروفایل پایه.
- **Models:** `User`, `StudentProfile`, `TeacherProfile`, `EmailVerificationToken`, `PasswordResetToken` — **KEEP** همه.
- **تغییر لازم:** `TeacherProfile.currency` پیش‌فرض `IRT`؛ حذف تدریجی وابستگی `stripe_account_id`/`stripe_onboarding_status` از این مدل به یک مدل `PayoutAccount` جنریک (Gateway-agnostic) — یا حداقل تبدیل به فیلدهای generic.
- **Services:** `register_user`, `verify_email`, `reset_password`, ... (KEEP).
- **Events خروجی:** `user.registered`, `email.verified` → مصرف‌کننده: Emails domain.

#### Teachers / Verification (REFACTOR + ADD)
- **Responsibility:** مهارت، در دسترس‌بودن، **و اکنون: وضعیت تأیید هویت/صلاحیت مدرس**.
- **Models موجود:** `TeacherSkill`, `AvailabilitySlot` (KEEP).
- **مدل جدید:** 
  ```python
  class TeacherVerification(models.Model):
      teacher = OneToOneField(TeacherProfile)
      status = CharField(choices=[PENDING, UNDER_REVIEW, APPROVED, REJECTED, SUSPENDED])
      submitted_at, reviewed_at, reviewed_by (FK User/staff)
      rejection_reason = TextField(blank=True)

  class TeacherDocument(models.Model):
      verification = FK(TeacherVerification, related_name="documents")
      document_type = CharField(choices=[ID_CARD, DEGREE, CERTIFICATE, ...])
      file = FileField(...)
      uploaded_at
  ```
- **Services جدید:** `submit_for_review`, `approve_teacher`, `reject_teacher`, `suspend_teacher` — همه با audit trail (شبیه الگوی `PaymentEvent` موجود، یعنی یک `TeacherVerificationEvent` append-only).
- **API:** `POST /api/teachers/verification/documents/`, `POST /api/admin/teachers/{id}/approve/`, `.../reject/`.
- **قانون Marketplace:** `TeacherFilter`/selector عمومی باید `verification__status="APPROVED"` را اضافه کند (در حال حاضر این فیلتر اصلاً وجود ندارد).
- **Dependency:** Users, Admin/Moderation.

#### Bookings (KEEP کامل — این بهترین بخش کد است)
- بدون تغییر ساختاری. فقط `locked_currency` پیش‌فرض به `IRT` تغییر کند.

#### Sessions (REFACTOR بخش meeting)
- **KEEP:** state machine، `SessionEnrollment`، `SessionStatusLog`، celery sweeps.
- **REFACTOR:** حذف `meet_link` ساختگی به نفع فیلد `meeting_room_id` + رابطه به مدل `Meeting` (دامنه جدید).
- **ADD (برای Group):** یک سرویس `create_group_session(teacher, skill, capacity, price, start, end)` که مستقل از `BookingRequest` یک `Session(session_type=GROUP)` می‌سازد و از طریق Catalog/Marketplace قابل کشف باشد.

#### Meetings (ADD — دامنه کاملاً جدید)
- **Responsibility:** انتزاع ارائه‌دهندهٔ جلسهٔ آنلاین.
- **Models:**
  ```python
  class Meeting(models.Model):
      session = OneToOneField(Session)
      provider = CharField(choices=[SKYROOM, ...])
      external_room_id = CharField
      teacher_join_url = URLField
      student_join_url = URLField
      status = CharField(choices=[CREATED, ACTIVE, ENDED, FAILED])
      created_at, ended_at
  ```
- **Services:** `MeetingProvider` interface به‌صورت انتزاعی:
  ```python
  class MeetingProvider(ABC):
      def create_room(self, session) -> RoomHandle: ...
      def get_join_url(self, room_handle, user) -> str: ...
      def revoke_access(self, room_handle) -> None: ...
  ```
  پیاده‌سازی `SkyroomMeetingProvider(MeetingProvider)`.
- **Access Control:** join URL فقط برای دانش‌آموز/مدرس همان Session و فقط در بازهٔ زمانی مجاز (مثلاً از ۱۰ دقیقه قبل تا پایان) صادر شود — از طریق یک endpoint backend که URL موقت تولید می‌کند، نه ذخیره‌ی URL دائمی در دیتابیس در معرض دید عموم.
- **Error Handling:** اگر Skyroom fail شود، Session در وضعیت `SCHEDULED` بماند و retry (Celery) اجرا شود؛ اگر بعد از N تلاش fail بماند، به تیم پشتیبانی/ادمین alert برود.

#### Payments (REFACTOR شدید — حفظ منطق، جایگزینی گیت‌وی)
- **KEEP کامل:** `Payment`, `PaymentEvent`, `PlatformCommission`, `PayoutLedgerEntry`, `CancellationPolicy`, `Dispute` — این‌ها gateway-agnostic هستند و تغییر معناداری لازم ندارند.
- **REPLACE:** هر ارجاع مستقیم به Stripe SDK در `services.py`/`webhooks.py` باید پشت این abstraction برود:
  ```python
  class PaymentGatewayProvider(ABC):
      def create_payment(self, amount, currency, metadata) -> GatewayIntent: ...
      def verify_callback(self, request_data) -> GatewayVerification: ...
      def refund(self, gateway_ref, amount) -> GatewayRefundResult: ...

  class ZarinpalProvider(PaymentGatewayProvider): ...
  class StripeProvider(PaymentGatewayProvider): ...  # اختیاری، برای کاربران بین‌المللی احتمالی آینده
  ```
- **`Payment.provider`** از `[INTERNAL, STRIPE]` به `[INTERNAL, ZARINPAL, IDPAY, STRIPE, ...]` گسترش یابد (migration ساده).
- **Callback flow ایرانی:** `create → redirect به درگاه → callback (GET با Authority/status) → verify (سرور به سرور) → succeeded/failed` — این با معماری فعلی (`capture_payment_for_booking` → webhook handler → `handle_payment_intent_succeeded/failed`) کاملاً سازگار است، فقط نام‌گذاری و شکل payload عوض می‌شود.
- **Idempotency:** واقعاً به‌کارگیری `idempotency_key` که الان تعریف شده ولی استفاده نمی‌شود.

#### Payouts (اختیاری: جدا کردن از Payments در فاز بعد)
- در حال حاضر `PayoutLedgerEntry` داخل app پرداخت‌ها است — قابل قبول برای الان (**KEEP as-is**)، جدا کردن به app مستقل فقط وقتی ارزش دارد که منطق payout (مثلاً payout دسته‌ای بانکی ایرانی، شماره شبا/کارت) پیچیده‌تر شود.

#### Reviews (ADD — دامنه کاملاً جدید)
- **Models:**
  ```python
  class Review(models.Model):
      booking = OneToOneField(BookingRequest, related_name="review")
      student = FK(User)
      teacher = FK(User)
      rating = PositiveSmallIntegerField(1..5)
      comment = TextField(blank=True)
      status = CharField(choices=[PUBLISHED, HIDDEN, FLAGGED])
      created_at
  ```
- **قانون کسب‌وکار:** فقط زمانی قابل ثبت که `Session.status == COMPLETED` و `Session.outcome == COMPLETED` (نه no-show) و هنوز `Review` برای آن `booking` ثبت نشده (constraint `OneToOne`).
- **Services:** `submit_review(booking, student, rating, comment)` → پس از ذخیره، `recalculate_average_rating(teacher)` (Celery task، نه synchronous، تا race condition نداشته باشیم).
- **Moderation:** فلگ `FLAGGED` برای گزارش نامناسب، بررسی در Admin.
- **API عمومی:** `GET /api/teachers/{id}/reviews/` برای صفحه پروفایل.

#### Notifications (KEEP) + Emails (ADD)
- `Notification` (in-app) دست‌نخورده بماند.
- **دامنه جدید `apps/emails`:**
  ```python
  class EmailLog(models.Model):
      recipient = FK(User)
      event_type = CharField()  # همان کلیدهای ماتریس زیر
      to_email = EmailField()
      status = CharField(choices=[QUEUED, SENT, FAILED, BOUNCED])
      provider_message_id = CharField(null=True)
      error = TextField(blank=True)
      created_at, sent_at

  def send_transactional_email(*, event_type, recipient, context: dict) -> None:
      """نقطهٔ ورود واحد — دقیقاً مثل الگوی create_notification موجود."""
  ```
- هر دامنه (bookings/payments/sessions/teachers) به‌جای فقط `create_notification`، هم `create_notification` و هم `send_transactional_email` را در همان Celery task صدا بزند (دقیقاً همان جایی که الان `notify_*` تعریف شده — فقط یک خط اضافه).
- **Retry/Failure:** با `@shared_task(bind=True, max_retries=3, ...)` مشابه الگوی موجود در `apps/sessions/tasks.py` (`auto_mark_ongoing` از `bind=True` استفاده می‌کند) — همین الگو برای ایمیل هم تکرار شود؛ شکست نهایی → `EmailLog.status=FAILED` + لاگ برای بررسی، **نه throw کردن exception به کاربر**.

### 3.4 Transactional Email Matrix (طراحی)

| Event | Recipient | فعلاً ایمیل دارد؟ | Priority |
|---|---|---|---|
| ثبت‌نام / تأیید ایمیل | کاربر جدید | ✅ (`verify_email`) | — |
| فراموشی رمز عبور | کاربر | ✅ (`password_reset`) | — |
| رزرو جدید ثبت شد | مدرس | ❌ فقط in-app | P0 |
| مدرس رزرو را تأیید کرد | دانش‌آموز | ❌ فقط in-app | P0 |
| مدرس رزرو را رد کرد | دانش‌آموز | ❌ فقط in-app | P0 |
| رزرو به دلیل عدم پاسخ منقضی شد | دانش‌آموز + مدرس | ❌ | P1 |
| پرداخت با موفقیت دریافت شد (رسید) | دانش‌آموز | ❌ فقط in-app | P0 |
| پرداخت ناموفق بود | دانش‌آموز | ❌ فقط in-app | P0 |
| بازپرداخت انجام شد | دانش‌آموز | ❌ | P0 |
| یادآوری جلسه (پیش از شروع) | دانش‌آموز + مدرس | ⚠️ نیمه (send_mail ساده، بدون قالب HTML/فارسی) | P0 |
| جلسه لغو شد | طرفین | ❌ فقط in-app | P0 |
| جلسه تکمیل شد (درخواست ثبت نظر) | دانش‌آموز | ❌ (رویداد اصلاً وجود ندارد) | P0 |
| نظر جدید ثبت شد | مدرس | ❌ (دامنه وجود ندارد) | P1 |
| مدارک مدرس برای بررسی ثبت شد | مدرس + ادمین | ❌ (دامنه وجود ندارد) | P0 |
| مدرس تأیید شد | مدرس | ❌ | P0 |
| مدرس رد شد | مدرس | ❌ | P0 |
| مدرس معلق شد | مدرس | ❌ | P1 |
| واریزی به مدرس انجام شد (Payout) | مدرس | ❌ فقط in-app | P0 |
| اختلاف (Dispute) باز شد | طرفین | ❌ | P1 |
| اختلاف حل شد | طرفین | ❌ | P1 |
| عدم حضور گزارش شد (No-show) | طرف مقابل | ❌ | P1 |

**نکته کلیدی:** زیرساخت Celery/task-per-event از قبل وجود دارد؛ کار اصلی، افزودن قالب HTML فارسی + فراخوانی `send_transactional_email` در تسک‌های موجود است — این کار پرریسک یا معماری‌سنگین نیست، فقط حجیم است.

### 3.5 Group Session — وضعیت فعلی و معماری تکمیل

**وضعیت فعلی (دقیق):**
- `Session.session_type = GROUP` گزینه موجود است.
- `Session.max_students`, `enrolled_count` فیلد موجود است.
- `SessionEnrollment` مدل کامل با `unique_together(session, student)` موجود است.
- `join_session`/`leave_session` سرویس کاملاً کاربردی برای عضویت در یک Session موجود.
- **آنچه وجود ندارد:** هیچ راهی برای دانش‌آموز که یک Group Session را در Marketplace **کشف** کند (بدون این‌که از قبل ID آن را بداند)، هیچ Booking/Payment flow مخصوص Group (پرداخت سرانه به‌جای پرداخت کامل یک بازه اختصاصی)، و هیچ سرویس برای این‌که مدرس یک Group Session **بسازد** به‌طور مستقل از یک `BookingRequest`.

**پیشنهاد معماری (اجرا بعد از پایدارشدن ۱:۱):**
1. سرویس جدید `create_group_session(teacher, skill, start, end, capacity, price_per_seat)` — بدون نیاز به `BookingRequest`.
2. Endpoint کشف: `GET /api/sessions/group/?skill=...&date=...` (شبیه `TeacherFilter` موجود).
3. Enrollment = «رزرو + پرداخت سرانه» — می‌تواند از همان `Payment` model استفاده کند اما با `amount = price_per_seat` و بدون قفل تقویم انحصاری (چون ظرفیت چندنفره است، نه انحصار زمانی مثل ۱:۱).
4. Cancellation policy جدا: اگر تعداد ثبت‌نام‌کننده به حداقل نرسد، لغو خودکار + بازپرداخت کامل (سرویس مشابه `auto_refund_on_rejection` موجود، فقط trigger متفاوت: یک Celery beat که near-start capacity را چک می‌کند).
5. Review: هر دانش‌آموز enrolled می‌تواند جدا نظر بدهد (کلید یکتایی `Review` باید از `booking` به `(session, student)` تعمیم یابد وقتی Group اضافه شد — این را از همین حالا در طراحی مدل `Review` لحاظ کنید تا در فاز Group نیازی به migration شکستن نباشد؛ پیشنهاد: از ابتدا `Review.session` + `Review.student` باشد، نه `Review.booking`، و `booking` صرفاً nullable FK اضافه برای ۱:۱ باشد).

### 3.6 State Machineها

**Booking (موجود، تأیید می‌شود، بدون تغییر):**
```
DRAFT → PENDING_PAYMENT → AWAITING_TEACHER_REVIEW ─accept→ ACCEPTED ─(session created)
                                                   └─reject→ REJECTED (auto-refund)
PENDING_PAYMENT ─(payment fail/timeout)→ CANCELLED
AWAITING_TEACHER_REVIEW ─(SLA expiry)→ CANCELLED (auto-refund)
ACCEPTED/PENDING/etc ─(student cancels)→ CANCELLED (refund per CancellationPolicy)
```

**Payment (موجود، تأیید می‌شود):**
```
PENDING → HELD_IN_ESCROW ─(session completed + hold window)→ RELEASED
HELD_IN_ESCROW ─(refund/cancel)→ REFUNDED
HELD_IN_ESCROW/RELEASED ─(dispute)→ DISPUTED ─resolve→ RELEASED / REFUNDED / (split: هر دو)
PENDING ─(gateway fail)→ FAILED
```

**Session (موجود، تأیید می‌شود):**
```
SCHEDULED → ONGOING → COMPLETED (outcome: COMPLETED / NO_SHOW_TEACHER / NO_SHOW_STUDENT)
SCHEDULED → CANCELLED
```

**Teacher Verification (جدید، پیشنهادی):**
```
PENDING ─(submit docs)→ UNDER_REVIEW ─approve→ APPROVED ─(violation)→ SUSPENDED
                                     └─reject→ REJECTED ─(resubmit)→ PENDING
```

---

## 4. Data Model Changes

| Model | وضعیت | توضیح |
|---|---|---|
| `User`, `StudentProfile` | **KEEP** | بدون تغییر |
| `TeacherProfile` | **MODIFY** | `currency` پیش‌فرض `IRT`؛ فیلدهای Stripe به فیلدهای generic (یا نگه‌داشتن به‌عنوان یکی از چند provider) |
| `EmailVerificationToken`, `PasswordResetToken` | **KEEP** | بدون تغییر |
| `TeacherSkill`, `AvailabilitySlot` | **KEEP** | بدون تغییر |
| `TeacherVerification` (جدید) | **ADD** | state machine تأیید مدرس |
| `TeacherDocument` (جدید) | **ADD** | مدارک آپلودی |
| `TeacherVerificationEvent` (جدید) | **ADD** | audit trail، مشابه الگوی `PaymentEvent` |
| `BookingRequest` | **MODIFY** | فقط پیش‌فرض `locked_currency` |
| `Session`, `SessionEnrollment`, `SessionStatusLog` | **KEEP** | بدون تغییر ساختاری |
| `Session.meet_link` | **REMOVE** (جایگزین با رابطه به `Meeting`) | ساختگی و ناامن (join url دائمی) |
| `Meeting` (جدید) | **ADD** | انتزاع Skyroom |
| `Payment`, `PaymentEvent` | **KEEP** (فقط گسترش `PROVIDER` choices) | `MODIFY` جزئی |
| `PlatformCommission`, `PayoutLedgerEntry`, `CancellationPolicy` | **KEEP** | بدون تغییر |
| `Dispute` | **KEEP** | بدون تغییر |
| `Review` (جدید) | **ADD** | طبق بخش ۳.۵ با فیلد `session` (نه فقط `booking`) برای سازگاری با Group آینده |
| `Notification` | **KEEP** | بدون تغییر |
| `EmailLog` (جدید) | **ADD** | ماتریس ایمیل |
| `AdminAuditLog` (جدید، پیشنهادی) | **ADD** | اکشن‌های حساس ادمین (تأیید/رد مدرس، حل اختلاف، بازپرداخت دستی) |

---

## 5. Implementation Roadmap

### Phase 0 — Foundations (بدون این‌ها ادامه دادن پرریسک است)
- Money/Currency value object + پیش‌فرض تومان (وابستگی: هیچ)
- `PaymentGatewayProvider` abstraction + پیاده‌سازی یک گیت‌وی ایرانی (وابستگی: هیچ، ولی روی کل Payments اثر می‌گذارد)
- `TeacherVerification` + فیلتر marketplace (وابستگی: هیچ)

### Phase 1 — Core Marketplace Trust
- `Review` domain کامل (وابستگی: نیازی به Phase 0 ندارد، موازی قابل انجام)
- `Emails` domain + پرکردن ماتریس P0 (وابستگی: هیچ، ولی هرچه دیرتر شود بدهی فنی بیشتر می‌شود)
- فارسی‌سازی + RTL frontend (وابستگی: هیچ، کار مستقل frontend)

### Phase 2 — Real Sessions
- `MeetingProvider` + Skyroom integration (وابستگی: Sessions موجود، بدون وابستگی به Phase 0/1)
- تقویم جلالی در UI (وابستگی: هیچ)

### Phase 3 — Hardening & Admin
- Admin verification/dispute UI بهبودیافته
- `AdminAuditLog`
- استفادهٔ واقعی از `idempotency_key`
- بازبینی rate limiting برای callback گیت‌وی

### Phase 4 — Group Sessions (طبق خواستهٔ صریح: فقط پس از پایداری ۱:۱)
- `create_group_session` + کشف + پرداخت سرانه + سیاست حداقل ظرفیت

---

## 6. First 10 Tasks (اگر توسعه از فردا شروع شود)

| # | Task | چرا اول این |
|---|---|---|
| 1 | تعریف `PaymentGatewayProvider` interface + `refactor` کردن `capture_payment_for_booking`/`webhooks.py` پشت آن، بدون تغییر منطق escrow | پرریسک‌ترین و پرکاربردترین وابستگی خارجی است؛ هرچه دیرتر انجام شود، کد بیشتری به Stripe کوپل می‌شود. منطق ledger/escrow فعلی حفظ می‌شود، فقط لایهٔ ارتباط با گیت‌وی عوض می‌شود |
| 2 | پیاده‌سازی provider واقعی گیت‌وی ایرانی (مثلاً Zarinpal) پشت interface بالا + تست sandbox | بدون این، هیچ تراکنش واقعی ایرانی ممکن نیست — مسدودکنندهٔ MVP |
| 3 | تغییر پیش‌فرض ارز به `IRT`/تومان در `TeacherProfile`, `Payment`, و `formatCurrency` فرانت (`Intl.NumberFormat("fa-IR", {currency: "IRR"/...})`) | خطای مالی نمایشی (نمایش $ برای تومان) ریسک اعتماد کاربر است؛ تغییر کم‌ریسک و مستقل |
| 4 | مدل `TeacherVerification` + `TeacherDocument` + سرویس `approve/reject` + فیلتر marketplace روی `APPROVED` | در حال حاضر **هر مدرس ثبت‌نامی بلافاصله عمومی است** — این یک نقص اعتماد/امنیتی محصول است، نه صرفاً feature ناقص |
| 5 | مدل `Review` + سرویس `submit_review` (فقط پس از `Session.status=COMPLETED`) + بازمحاسبهٔ `average_rating` | `average_rating` هم‌اکنون یک فیلد مرده روی schema است؛ بدون Review، کل مدل اعتماد Marketplace (که در بریف محصول تصریح شده) وجود ندارد |
| 6 | دامنهٔ `Emails` + اتصال به رویدادهای موجود (`notify_booking_accepted/rejected`, `notify_payment_received/failed`, `notify_session_cancelled`) — قالب پایه فارسی مشابه `base.html` موجود | زیرساخت Celery task-per-event از قبل آماده است؛ این‌ها فقط یک خط `send_transactional_email(...)` به هر تسک اضافه می‌کند — نسبت اثر به ریسک بسیار بالاست |
| 7 | افزودن i18n (dictionary فارسی) + `dir="rtl"` سراسری + بازبینی کامپوننت‌های اصلی (`Button`, `Card`, `Form`) در `shared/components` | تجربهٔ کاربری کل محصول را عوض می‌کند؛ هرچه بعدتر انجام شود، کامپوننت‌های بیشتری باید دوباره بررسی شوند |
| 8 | `MeetingProvider` abstraction + جایگزینی `meet_link` ساختگی با `Meeting` model + Skyroom integration اولیه | بدون این، حتی وقتی رزرو/پرداخت کار کند، جلسه واقعاً برگزار نمی‌شود — این جزو مسیر اصلی محصول (Flow اصلی بریف) است |
| 9 | افزودن تقویم جلالی در presentation layer (booking calendar، availability picker، session list) با `date-fns-tz` موجود برای `Asia/Tehran` | وابسته به هیچ‌کدام از موارد بالا نیست، اما برای کاربر فارسی‌زبان محسوس‌ترین نقص UX است |
| 10 | استفادهٔ واقعی از `idempotency_key` در endpoint ایجاد پرداخت + تست تکرار درخواست تحت race | فیلد از قبل در schema هست ولی به گفتهٔ کامنت خود کد استفاده نمی‌شود؛ در فضای گیت‌وی ایرانی (که callback/retry رفتار متفاوتی از Stripe دارد) این ریسک double-charge را می‌بندد قبل از رفتن به production |

---

## جمع‌بندی نهایی

Backend فعلی Afra را باید به‌عنوان یک **دارایی مهندسی معتبر** نگاه کرد، نه بدهی فنی — منطق booking/escrow/payout آن از خیلی از مارکت‌پلیس‌های واقعی دقیق‌تر است. کاری که باقی مانده عمدتاً **افزودن دامنه‌های گم‌شده (Verification, Reviews, Emails, Meetings)** و **جایگزینی یک وابستگی خارجی (Stripe→ایرانی) پشت یک abstraction** است — نه بازطراحی هستهٔ سیستم. فرانت‌اند نیازمند کار بیشتری برای فارسی‌سازی/RTL/Jalali است چون در حال حاضر عملاً صفر است در این حوزه‌ها.
