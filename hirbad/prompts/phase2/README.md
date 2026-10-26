# معماری و پرامپت‌های اجرایی: فیلد «تلفن» و «سابقه تدریس»

پروژه: hirbad-platform · فاز ۱: بررسی پروژه، طراحی معماری، پرامپت برای ایجنت‌های هوش مصنوعی

## ۱. خلاصه تصمیم‌ها

- `phone_number` روی مدل `User` (نه روی دو پروفایل): یک ستون، یک اعتبارسنجی، و ادمین‌ها (`is_staff`) که پروفایل ندارند هم پوشش داده می‌شوند.
- `teaching_experience_years` روی `TeacherProfile`: عدد صحیح ۰ تا ۶۰ و nullable (null یعنی «ثبت نشده»، پس معلم‌های فعلی نیازی به backfill ندارند).
- تلفن فقط در سه خروجی ظاهر می‌شود: `/api/accounts/me/` (فقط خود کاربر)، `AdminUserSerializer` و `AdminTeacherVerificationSerializer` (به‌علاوه Django admin). این قانون با تست خودکار قفل می‌شود، نه با حافظه‌ی توسعه‌دهنده.
- سابقه تدریس عمومی است: صفحه‌ی عمومی معلم، کارت لیست معلم‌ها و پروفایل خود معلم.
- وابستگی جدید اضافه نمی‌شود؛ نرمال‌سازی شماره در `apps/common/phone.py` (و آینه‌ی آن در فرانت).

## ۲. یافته‌های بررسی پروژه

بک‌اند: Django 5.2 + DRF + Celery با لایه‌بندی views ← serializers (فقط اعتبارسنجی) ← services (نوشتن) / selectors (خواندن). فرانت: React 19 + Vite + TypeScript + react-hook-form + zod + i18next (فقط فارسی) + vitest.

| موضوع | محل | نکته‌ی مهم برای این کار |
| --- | --- | --- |
| مدل‌ها | `apps/users/models.py` | `User` + `StudentProfile` + `TeacherProfile`؛ پروفایل با signal ساخته می‌شود. آخرین migration: `0008` |
| نوشتن پروفایل | `users/services.py` | فیلدهای قابل‌ویرایش با whitelist (`_TEACHER_PROFILE_EDITABLE_FIELDS`) تعیین می‌شوند؛ فیلد جدیدی که اینجا اضافه نشود **بی‌صدا نادیده گرفته می‌شود** |
| دو مسیر نوشتن پروفایل معلم | `PATCH /api/accounts/me/` و `PATCH /api/teachers/me/` | هر کدام serializer ورودی جدا دارند (`TeacherProfileSerializer` و `TeacherProfileUpdateSerializer`)؛ هر دو باید هماهنگ شوند |
| خروجی عمومی معلم | `teachers/serializers.py` | `TeacherDetailSerializer` هم برای کاربر ناشناس (`/api/teachers/<id>/`) و هم برای `/teachers/me/` استفاده می‌شود؛ هر فیلدی که آنجا اضافه شود عمومی است. `TeacherSummarySerializer` لیست عمومی است |
| شرکت‌کننده‌ها | serializerهای `bookings` و `sessions` | فقط `id / full_name / avatar_url` برمی‌گردانند؛ امن‌اند و نباید دست بخورند. معلم لیست دانش‌آموزان جلسه را با همین‌ها می‌بیند |
| ادمین | `adminapi/serializers.py` | `AdminUserSerializer` و `AdminTeacherVerificationSerializer`؛ دسترسی با `IsAdminUser` (= `is_staff`) |
| لاگ | `apps/common/logging.py` | ماسک کردن بر اساس **نام پارامتر** است (`password/token/secret/email/authorization`). `phone` باید اضافه شود؛ `AdminAuditLog.details` هم مقادیر JSON را عیناً نگه می‌دارد |
| تست وابسته | `tests/adminapi/test_users.py` | مجموعه‌ی دقیق فیلدهای `AdminUserSerializer` را چک می‌کند (`_USER_FIELDS`)؛ باید به‌روز شود |
| فرانت | `features/profile`, `features/teachers`, `features/admin` | `ProfileForm.tsx`, `profile.types.ts`, `TeacherPublicProfilePage.tsx`, `TeacherCard.tsx`, `AdminUsersPage.tsx` و فایل‌های i18n در `shared/i18n/locales/fa` |

**یافته‌ی جانبی (خارج از این کار):** `TeacherDetailSerializer` فیلد `email` معلم را به کاربر ناشناس هم نشان می‌دهد. در `features/teachers` فرانت جایی از آن استفاده نشده (بقیه‌ی فرانت را برای این مورد نگشتم). اگر قصدتان پنهان کردن اطلاعات تماس معلم‌هاست، جداگانه تصمیم بگیرید. به ایجنت‌ها گفته‌ام آن را تغییر ندهند.

## ۳. معماری

### ۳.۱ مدل داده

- `User.phone_number = CharField(max_length=16, blank=True, default='')` بدون unique و بدون index. unique نیست چون ممکن است اعضای یک خانواده شماره‌ی مشترک بدهند، و چون unique بودن راهی برای لو رفتن وجود یک حساب (account enumeration) است.
- ذخیره‌ی نرمال‌شده: موبایل ایران به شکل `+989121234567` و شماره‌های بین‌المللی به شکل E.164. ورودی با ارقام فارسی/عربی و قالب‌هایی مثل `0912…`، `0098…`، فاصله و خط تیره پذیرفته می‌شود.
- `TeacherProfile.teaching_experience_years = PositiveSmallIntegerField(null=True, blank=True)` با سقف ۶۰.
- یک migration افزایشی و برگشت‌پذیر (`0009`).

### ۳.۲ ماتریس دسترسی

| داده | کاربر ناشناس | معلم یا دانش‌آموزِ دیگر | خود کاربر | ادمین |
| --- | --- | --- | --- | --- |
| تلفن | ✗ | ✗ | ✓ مشاهده و ویرایش | ✓ مشاهده |
| سابقه تدریس | ✓ | ✓ | ✓ مشاهده و ویرایش (معلم) | ✓ |

فرض من: خود کاربر تلفن خودش را می‌بیند و ویرایش می‌کند (بدون این، ثبت تلفن ممکن نیست). اگر مخالفید فقط سطح `UserMeSerializer` عوض می‌شود.

### ۳.۳ قرارداد API (منبع حقیقت برای بک‌اند و فرانت)

| Endpoint | تغییر |
| --- | --- |
| `GET/PATCH /api/accounts/me/` | فیلد سطح‌بالای `phone_number` (رشته؛ خالی یعنی ثبت نشده؛ خطای اعتبارسنجی با کد `invalid_phone_number`). برای معلم `profile.teaching_experience_years` (عدد یا null) |
| `PATCH /api/teachers/me/` | پذیرش `teaching_experience_years` |
| `GET /api/teachers/` و `/api/teachers/<id>/` | خروجی `teaching_experience_years` |
| `GET /api/admin/users/` و `/<id>/` | خروجی `phone_number` |
| `GET /api/admin/teachers/verifications/…` | خروجی `teacher_phone_number` |

### ۳.۴ دفاع چندلایه‌ی حریم خصوصی

1. **ساخت امن:** تلفن فقط در سه serializer مجاز تعریف می‌شود؛ هیچ `fields='__all__'` یا `exclude` روی `User` و پروفایل‌ها پذیرفته نیست.
2. **تست نگهبان (allowlist):** همه‌ی serializerها اسکن می‌شوند و اگر serializer دیگری `phone_number` داشته باشد تست می‌شکند.
3. **تست canary نشت:** به همه‌ی کاربران شماره‌ی متمایز داده می‌شود، همه‌ی endpointهای عمومی و بین‌کاربری (معلم‌ها، رزرو، جلسه‌ی گروهی، نظرات، پرداخت، اعلان‌ها) با نقش‌های مختلف صدا زده می‌شود و در بدنه‌ی پاسخ‌ها، ایمیل‌های خروجی، payload اعلان‌ها و لاگ‌ها دنبال شماره گشته می‌شود.
4. **لاگ:** افزودن `phone` به `_SENSITIVE_NAME_MARKERS`.
5. تلفن هرگز در فیلتر یا جستجوی عمومی نمی‌آید؛ جستجو با تلفن فقط برای ادمین (API ادمین و Django admin).

### ۳.۵ تصمیم‌های باز (پیش‌فرض‌ها در پرامپت‌ها اعمال شده؛ نظرتان را بگویید)

- **ثبت‌نام:** تلفن به فرم ثبت‌نام اضافه نمی‌شود، فقط در پروفایل و اختیاری. اجباری کردنش بعداً ساده است.
- **تلفن ثابت:** فعلاً فقط موبایل ایران و شماره‌ی بین‌المللی.
- **ویرایش تلفن کاربران توسط ادمین:** فقط از Django admin.
- **فیلتر یا مرتب‌سازی لیست معلم‌ها بر اساس سابقه:** خارج از این فاز.
- **متن سیاست حریم خصوصی** (StaticPage در دیتابیس) را باید خودتان به‌روز کنید.

## ۴. نقشه‌ی اجرا

| ایجنت | کار | وابستگی |
| --- | --- | --- |
| A | بک‌اند: مدل، migration، اعتبارسنجی، سرویس، `/me`، لاگ | — |
| B | بک‌اند: خروجی عمومی و ادمین، seed، تست‌ها | بعد از A |
| C | تست‌های امنیتی (canary + allowlist) | بعد از B |
| D | فرانت‌اند | موازی با A تا C (قرارداد ثابت است) |
| E | بازبینی نهایی و گزارش | آخر از همه |

پیشنهاد: A→B→C پشت سر هم روی یک branch، D روی branch جدا، و E روی branch ادغام‌شده. هر پرامپت با «Common preamble» شروع می‌شود.

## ۵. پرامپت‌ها (انگلیسی)

### Common preamble (ابتدای هر پنج پرامپت paste شود)

```text
You are working in the monorepo 'hirbad-platform', a Persian-language tutoring marketplace.
- Backend: hirbad-backend/ (Django 5.2, DRF, Celery, pytest, factory_boy; black, isort, flake8).
- Frontend: hirbad-frontend/ (React 19, Vite, TypeScript, react-hook-form + zod, TanStack Query, i18next with Persian-only locales, Tailwind, vitest).

GOAL OF THE OVERALL FEATURE
1. phone_number on every User (teacher and student). PRIVATE: visible only to (a) the owner through GET/PATCH /api/accounts/me/ and (b) staff (is_staff). It must never appear in any response seen by another teacher, another student, or an anonymous visitor.
2. teaching_experience_years on TeacherProfile only. PUBLIC: visible to everybody. Integer 0-60, nullable (null = not provided).
You are one of several agents; do only the scope given in your task section.

API CONTRACT (source of truth)
- GET/PATCH /api/accounts/me/: top-level 'phone_number' (string; '' = unset; input accepts common Iranian/international formats and is stored canonical, e.g. '+989121234567'; '' clears it; invalid input -> HTTP 400 with field error code 'invalid_phone_number'). For teachers also 'profile.teaching_experience_years' (integer or null).
- PATCH /api/teachers/me/: accepts 'teaching_experience_years' (integer 0-60 or null).
- GET /api/teachers/ and GET /api/teachers/<id>/: include 'teaching_experience_years'.
- GET /api/admin/users/ and /api/admin/users/<id>/: include 'phone_number'. Admin teacher-verification endpoints include 'teacher_phone_number'.
- 'phone_number' appears in NO other response, email, notification payload or log line.

RULES
- Read the neighbouring code first and imitate its style, naming and docstring tone (comments explain why).
- Layering: views -> serializers (validate/shape only) -> services (all writes, decorated like their siblings) / selectors (reads). No ORM writes in views or serializers.
- Serializers list fields explicitly. Never use fields='__all__' or exclude.
- UI text lives only in the Persian i18n JSON files; no hardcoded user-facing strings.
- Stay in scope. If you notice an unrelated bug or smell (for example the public teacher endpoint exposing 'email'), do NOT change it; mention it in your final report.
- Do not add dependencies.
- Run the checks listed in your task. Never claim a check passed unless you ran it; if you cannot run something, say so.
- Final report: files changed, decisions and assumptions, open questions, commands run with results.
```

### Prompt A — Backend core (مدل، اعتبارسنجی، مسیر نوشتن)

```text
TASK A — Backend core: data model, validation, write path (scope: hirbad-backend/apps/users, apps/common, and tests for them)

Read first: apps/users/{models,services,serializers,admin,signals}.py, apps/common/logging.py, apps/users/migrations/0008_*.py, tests/users/*, tests/factories/user_factories.py.

1. Create apps/common/phone.py with a pure function normalize_phone_number(raw: str) -> str that raises ValueError on invalid input.
   - Strip whitespace, '-', '.', '(' and ')'. Convert Persian (U+06F0-06F9) and Arabic-Indic (U+0660-0669) digits to ASCII.
   - Iranian mobile numbers in any of these forms are canonicalised to '+98' + 10 digits starting with 9: '09123456789', '9123456789', '989123456789', '+989123456789', '00989123456789'.
   - Any other input must be '+' (or a leading '00' converted to '+') followed by 8-15 digits, first digit non-zero; it is kept as is.
   - Empty or whitespace-only input returns ''.
   - Everything else raises ValueError.
2. apps/users/models.py:
   - User.phone_number = CharField(max_length=16, blank=True, default=''), no unique, no index. Comment why it is not unique (shared family numbers; uniqueness would also be an account-enumeration oracle) and that it is PRIVATE (owner + staff only).
   - TeacherProfile.teaching_experience_years = PositiveSmallIntegerField(null=True, blank=True, validators=[MaxValueValidator(60)]) with help_text; null means 'not provided'.
3. Generate ONE additive, reversible migration (0009) with `python manage.py makemigrations users`; give it a descriptive name; do not edit existing migrations; confirm `python manage.py makemigrations --check --dry-run` is clean.
4. apps/users/services.py:
   - Add 'teaching_experience_years' to _TEACHER_PROFILE_EDITABLE_FIELDS (the service silently ignores non-whitelisted keys). In update_teacher_profile raise ApplicationError if the value is not None and outside 0-60.
   - In update_user_profile handle top-level 'phone_number' next to 'full_name': normalise via normalize_phone_number (convert ValueError to ApplicationError with code 'invalid_phone_number'; the service must not trust the serializer) and issue a single user.save(update_fields=[...]) containing only the fields that were provided.
5. apps/users/serializers.py:
   - UserMeSerializer: add 'phone_number' to fields as a writable, optional field (CharField, required=False, allow_blank=True, max_length=32) with validate_phone_number that calls normalize_phone_number and raises ValidationError(code='invalid_phone_number'). All existing read-only fields (is_staff, role, ...) stay read-only. UserMeSerializer is the only non-admin serializer allowed to expose phone_number.
   - TeacherProfileSerializer: add 'teaching_experience_years' (IntegerField, min_value=0, max_value=60, allow_null=True, required=False).
   - Do not touch UserPublicSerializer or StudentProfileSerializer.
   - Prove by test that a nested PATCH {'profile': {'teaching_experience_years': 10}} reaches the service through UserMeSerializer.to_internal_value, and that PATCH {'phone_number': '...'} works for both roles.
6. apps/users/admin.py (Django admin is staff-only): add phone_number to the 'Personal info' fieldset and to UserAdmin.search_fields; add teaching_experience_years to TeacherProfileAdmin.list_display.
7. apps/common/logging.py: add 'phone' to _SENSITIVE_NAME_MARKERS with a comment (masking is by parameter name, so a kwarg named phone_number must be masked in both log_service_call and AdminAuditLog details). Add a test for both paths next to the existing ones in tests/common/.
8. Tests (pytest, existing factories; extend factories only if needed): table-driven tests for normalize_phone_number (all accepted forms, Persian digits, invalid cases, empty); service tests; /api/accounts/me/ GET and PATCH round-trips for student and teacher (set, change, clear with '', invalid -> 400 with code 'invalid_phone_number'); experience set via nested profile, null clears, -1 and 61 rejected; a student cannot set teaching_experience_years.

Out of scope: teachers and adminapi serializers, seed data, frontend.
Checks: make lint; pytest --no-cov -p no:cacheprovider tests/users tests/common; then the full suite.
Done when: the migration is clean, all new tests pass, no existing test was weakened or deleted, and phone_number is defined in exactly one serializer outside adminapi (UserMeSerializer).
```

### Prompt B — Backend exposure layer (خروجی عمومی و ادمین)

```text
TASK B — Backend exposure layer (scope: apps/teachers, apps/adminapi, seed_data, tests). Precondition: Task A is merged on your branch.

Read first: apps/teachers/{serializers,views,selectors,filters}.py, apps/adminapi/serializers.py, apps/adminapi/views/{users,teachers}.py, apps/common/management/commands/seed_data.py, tests/adminapi/test_users.py, tests/teachers/test_views.py.

1. apps/teachers/serializers.py (PUBLIC surface):
   - TeacherDetailSerializer: add 'teaching_experience_years' to fields (read-only). This serializer feeds the anonymous endpoint, so NEVER add anything phone-related here.
   - TeacherSummarySerializer: add a read-only 'teaching_experience_years' (source='teacher_profile.teaching_experience_years', allow_null). It must not add queries (list_teachers already select_related the profile; keep existing query-count tests green).
   - TeacherProfileUpdateSerializer (input of PATCH /api/teachers/me/): add 'teaching_experience_years' (IntegerField 0-60, allow_null, required=False) and verify end to end that the value is persisted and returned in the response.
2. apps/adminapi/serializers.py (STAFF surface, permission IsAdminUser):
   - AdminUserSerializer: add 'phone_number' (read-only).
   - AdminTeacherVerificationSerializer: add 'teacher_phone_number' (source='teacher_profile.user.phone_number', read-only) next to teacher_email.
   - AdminUserListView search: also match phone numbers. Take the digits of the search term (convert Persian digits, drop non-digits, strip leading zeros) and, when at least 3 digits remain, OR in phone_number__contains=<digits>. Stored numbers are canonical ('+98912...'), so a search for '0912345' must find them.
3. Public search and filters must never touch phone: do not add it to TeacherFilter, selectors.list_teachers or any search_fields except the staff-only Django admin.
4. Do NOT modify serializers or templates of bookings, sessions, reviews, payments, notifications, meetings or emails. Prove it: run `grep -rniE 'phone' apps/ --include=*.py --include=*.html --include=*.txt` and paste the result in your report; every hit must be explainable by Task A or B.
5. seed_data.py (DEBUG only): give seeded users plausible Iranian mobile numbers (Faker) and seeded teachers a teaching_experience_years.
6. If the project has an OpenAPI schema check (drf-spectacular), run it and make sure it still generates.
7. Tests: update _USER_FIELDS in tests/adminapi/test_users.py; admin list/detail expose phone_number while non-staff and anonymous callers are rejected; phone search works with '0912...' and '+98912...'; verification admin responses expose teacher_phone_number; public GET /api/teachers/<id>/ and the list expose teaching_experience_years (null and 10); PATCH /api/teachers/me/ sets and clears it, rejects 61 and -1, and a student gets 403.

Out of scope: users app internals (Task A), privacy canary tests (Task C), frontend.
Checks: make lint; pytest --no-cov -p no:cacheprovider tests/teachers tests/adminapi; then the full suite.
```

### Prompt C — Privacy verification (تست‌های امنیتی)

```text
TASK C — Independent privacy verification (scope: tests only; fix a leak only at its source and with the smallest change). Precondition: A and B are merged on your branch. Act as a red-team reviewer who assumes the implementers made mistakes.

1. tests/users/test_phone_privacy.py — canary leak test.
   - Build a realistic world with the existing factories: an APPROVED teacher T1 and a second teacher T2, students S1 and S2, and a staff user. Give EVERY user a different, distinctive phone number. For each number build needles: canonical ('+989121234567'), local ('09121234567') and bare ('9121234567').
   - Create data that links people: a booking S1<->T1 taken through accept, the resulting session, a group session with S1 and S2 enrolled, a published review, a payment, notifications and sent emails (Celery is eager in the test settings).
   - For each actor (anonymous, S1, S2, T1, T2) call every list/detail GET endpoint reachable for that actor (discover them from the URLconfs of teachers, bookings, sessions, group sessions, reviews, payments, notifications, contact, content) and assert that no needle belonging to ANOTHER user appears anywhere in response.content.decode(). Parametrize so a failure names the actor and the URL.
   - Assert an actor sees their OWN phone only at /api/accounts/me/, and that staff sees phones only on /api/admin/ endpoints.
   - Assert no needle appears in mail.outbox subjects/bodies/HTML alternatives, in Notification payloads, or in captured log records (caplog) produced during these flows and during PATCH /api/accounts/me/.
   - Assert GET /api/teachers/?search=<phone digits> returns no teacher (phone is not publicly searchable).
2. tests/common/test_phone_serializer_allowlist.py — architectural guard.
   - Import every apps.*.serializers module, walk all subclasses of rest_framework.serializers.BaseSerializer recursively, and read the declared and model-derived fields (instantiate defensively; skip classes that need required context, but report how many were skipped).
   - Fail if any serializer outside ALLOWLIST = {UserMeSerializer, AdminUserSerializer, AdminTeacherVerificationSerializer} has a field named phone_number or teacher_phone_number, or a field whose source contains 'phone_number'. The failure message must explain how to extend the allowlist deliberately.
   - Fail if any ModelSerializer for User, StudentProfile or TeacherProfile uses Meta.fields == '__all__' or Meta.exclude.
3. Report a short matrix (actor x endpoint group -> leak found yes/no), the number of endpoints actually exercised, and any endpoint you could not exercise and why.

Checks: pytest --no-cov -p no:cacheprovider tests/users tests/common; the full suite; make lint.
```

### Prompt D — Frontend

```text
TASK D — Frontend (scope: hirbad-frontend only). The API contract above is fixed; you may work in parallel with the backend by relying on the MSW mocks and fixtures under src/test/mocks.

Read first: src/features/profile/** (types, api, hooks, ProfileForm.tsx, ProfilePage.tsx), src/features/teachers/{types,components/TeacherCard.tsx,pages/TeacherPublicProfilePage.tsx}, src/features/admin/{types/admin.types.ts,pages/AdminUsersPage.tsx,pages/AdminTeacherVerificationDetailPage.tsx}, src/shared/lib/{validators,money,rating,form-errors}.ts, src/shared/i18n/locales/fa/{profile,teachers,admin}.json, src/test/mocks/**, scripts/check-rtl-classes.mjs and scripts/check-no-hardcoded-colors.mjs.

1. src/shared/lib/phone.ts: a client-side mirror of the backend rules (accept Persian/Arabic digits and spaces/dashes; Iranian mobile in 09…, 9…, 98…, +98… and 0098… forms; otherwise '+' followed by 8-15 digits), exporting normalizePhone, isValidPhone and formatPhoneForDisplay ('+989121234567' -> '0912 123 4567'; other numbers unchanged). Unit-test it table-driven. The server stays the authority; the client only gives early feedback. Do not hardcode English messages (validators.ts does, but new code must use i18n through t(), like ProfileForm does).
2. Types: UserProfile gets 'phone_number: string'; UserProfileDetails and UpdateProfileFields get 'teaching_experience_years?: number | null'; UpdateProfilePayload gets 'phone_number?: string'; TeacherProfile and TeacherSummary get 'teaching_experience_years: number | null'; AdminUser gets 'phone_number: string'; the admin verification type gets 'teacher_phone_number: string'. Do NOT add phone to AuthUser, the login/JWT state, localStorage, or any non-admin type.
3. ProfileForm.tsx: for ALL roles add a phone field (Input with dir='ltr', inputMode='tel', autoComplete='tel', placeholder '09xxxxxxxxx', optional) with a Persian helper text saying it is visible only to site administrators. Teachers additionally get a 'teaching experience (years)' number input (0-60, integer, empty -> null). Send phone_number at the top level of the PATCH payload and teaching_experience_years inside 'profile'. Show the stored phone through formatPhoneForDisplay when the form resets. Make sure server errors with code 'invalid_phone_number' land on the right field through mapApiErrorsToForm. Keep all existing behaviour (full_name, bio, learning_goals, hourly_rate, currency) intact.
4. Public display of experience: TeacherPublicProfilePage shows it next to the rating/session stats when not null (e.g. '10 سال سابقه تدریس'; reuse the digit-formatting helper the page already uses; 0 -> 'کمتر از یک سال') and hides it when null. TeacherCard shows a compact line only when not null, without layout shift in the grid.
5. Admin: AdminUsersPage gets a phone column (dir='ltr', '—' when empty); the verification detail page shows the teacher's phone under the email. The phone must not be rendered in any other component.
6. i18n: add Persian keys only in fa/profile.json, fa/teachers.json and fa/admin.json, following the existing key style. RTL: logical Tailwind classes only. Never pass form values (the phone) to shared/lib/analytics.ts or logger calls.
7. Tests (vitest + Testing Library): phone.ts; ProfileForm (phone shown for both roles, experience only for teachers, invalid phone blocks submit, payload shape, server-error mapping); TeacherPublicProfilePage shows/hides experience; AdminUsersPage shows the phone. Update fixtures and MSW handlers (teacher.fixtures.ts, admin fixtures, auth fixtures if the /me shape is mocked) so existing tests keep passing.

Checks: npm run typecheck && npm run lint && npm run check:rtl && npm run lint:colors && npm test.
```

### Prompt E — Final audit

```text
TASK E — Final audit (read-mostly). Precondition: A-D are merged into one integration branch. Review the complete diff (git diff main...HEAD) with fresh eyes and assume something was missed.

1. Privacy: run grep -rniE 'phone' across hirbad-backend/apps and hirbad-frontend/src (excluding tests, i18n and node_modules) and justify every hit against the allowed surfaces: UserMeSerializer, AdminUserSerializer, AdminTeacherVerificationSerializer, Django admin, ProfileForm, and the two admin pages. Anything else is a defect.
2. Run the canary and allowlist tests, the full backend suite, make lint, python manage.py makemigrations --check --dry-run, the OpenAPI generation if present, and the full frontend check set.
3. Migration safety: confirm 0009 is additive, reversible (apply then unapply on a scratch Postgres if available; otherwise state that it was not tested) and touches no existing data.
4. Contract: compare real responses of the endpoints in the contract with the TypeScript types, field by field.
5. Edge cases to probe (manually or by test): PATCH phone to '' and then to a Persian-digit number; two users with the same number; a teacher with null experience in list and detail; a student sending 'profile.teaching_experience_years'; a staff user without a profile calling /api/accounts/me/.
6. Docs: add the new fields wherever API fields are documented (backend README/docs) and add 3-4 manual checks to the existing smoke-test checklists (backend docs and frontend docs).
7. Report: PASS/FAIL per item, defects found (fix only trivial ones such as lint), residual risks, and the decisions that still belong to the owner: privacy-policy text, whether to hide the teacher 'email' on the public endpoint, landline support, phone at registration.
```
