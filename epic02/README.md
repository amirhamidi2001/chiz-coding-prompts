# Epic 2 — Authentication & Authorization (Iran-Market OTP) — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as Epic 1 — each task is a standalone prompt, feed them one at a time in order, let each be committed/reviewed before starting the next. Tasks within a Feature must run in the printed order; Features across Phases mostly run in order too (called out explicitly wherever a task can safely run in parallel with another Phase).

**Assumed prerequisite:** Epic 1 (Core Backend Stability) is fully merged — order creation is atomic, transaction-safe, and pricing goes through `order/services/pricing.py`. Epic 2 does not modify order logic, so this is a soft dependency (mainly relevant because later epics that depend on both will assume both are done), but build in the order given regardless.

---

## Phase 2.1 — Phone Number Foundation

### Feature 2.1.1 — User Model Extension

---

#### Task 2.1.1.1 — Add `phone_number` field to `User` model (not `Profile`)

```
You are working in backend/accounts/models.py.

CONTEXT
The current User model (AbstractBaseUser + PermissionsMixin) is:

    class User(AbstractBaseUser, PermissionsMixin):
        email = models.EmailField(_("email address"), unique=True)
        is_staff = models.BooleanField(default=False)
        is_active = models.BooleanField(default=True)
        is_verified = models.BooleanField(default=False)
        type = models.IntegerField(choices=UserType.choices, default=UserType.CUSTOMER)
        created_date = models.DateTimeField(auto_now_add=True)
        updated_date = models.DateTimeField(auto_now=True)
        USERNAME_FIELD = "email"
        REQUIRED_FIELDS = []
        objects = UserManager()

There is a `phone_number` field today, but it lives on the separate
`Profile` model (OneToOneField to User, auto-created via the
`create_profile` post_save signal), and it's just a free CharField with
no format validation. For OTP-based login/registration to work, phone
number has to be a first-class, unique, indexed identifier on the AUTH
model itself — you can't authenticate against a field on a related
model efficiently or safely, and two different users must never be able
to register with the same phone number.

TASK
Add a `phone_number` field directly to the `User` model (leave
`Profile.phone_number` in place for now — Task 2.1.1.3 handles
migrating/reconciling the two; don't delete anything yet in this task).

REQUIREMENTS
- Add to the `User` model:
  `phone_number = models.CharField(max_length=15, unique=True, null=True, blank=True, db_index=True)`
  — nullable because existing users registered via email/password have
  no phone number yet, and because a single OTP-first-touch flow may
  eventually let phone_number be set async of account creation. Unique
  constraint prevents two accounts sharing one number (Django's unique
  constraint on a nullable field correctly allows multiple NULLs while
  still enforcing uniqueness among non-null values on Postgres — confirm
  this is also true for whatever DB backend the test suite runs
  against).
- Do NOT change `USERNAME_FIELD` (stays `"email"`) — email remains the
  primary login identifier for the admin/existing email+password flow;
  phone is an additional, parallel authentication path, not a
  replacement. (This decision is deliberate: don't "fix" it by trying
  to make phone_number the USERNAME_FIELD — that would break every
  existing admin/superuser account created via `create_superuser` which
  uses email.)
- Generate the migration with `python manage.py makemigrations accounts`.
- Add `phone_number` as a read-only field on `CurrentUserSerializer`
  (backend/accounts/serializers.py) so the frontend can see whether the
  logged-in user has a verified phone attached.
- Do not add any validation regex yet (that's Task 2.1.1.2) — plain
  CharField constraints only in this task.

ACCEPTANCE CRITERIA / TESTS
- Migration applies cleanly on top of the existing accounts migration
  (backend/accounts/migrations/0001_initial.py).
- Add a model test in backend/accounts/tests/test_models.py:
  1. Two users can both be created with `phone_number=None`
     (proving nullable + unique doesn't conflict on NULLs).
  2. Creating a second user with the same non-null phone_number as an
     existing user raises an IntegrityError / fails validation.
- Run the full accounts test suite; nothing existing should break since
  this is a purely additive field.
```

---

#### Task 2.1.1.2 — Add Iranian phone number validator

```
You are working in backend/accounts/models.py (or a new
backend/accounts/validators.py if that's cleaner — check whether a
validators module already exists in this app before creating one).
Assume Task 2.1.1.1 (phone_number field on User) is already merged.

CONTEXT
The new `User.phone_number` field currently accepts any string up to
15 characters with no format checking — "abc", "123", or a
non-Iranian number would all be accepted today. Iranian mobile numbers
follow a specific, well-known format: `09XXXXXXXXX` (11 digits,
starting with 09) in local format, or `+989XXXXXXXXX` /
`00989XXXXXXXXX` in international format. The OTP system being built
in this Epic depends on having a single normalized, valid format to
send SMS to.

TASK
Add a RegexValidator for Iranian mobile numbers and apply it to
`User.phone_number`, plus a normalization helper that converts any
accepted input format into one canonical stored format.

REQUIREMENTS
- Create `backend/accounts/validators.py` (new file) with:
  1. `iranian_phone_regex = RegexValidator(regex=r'^(\+98|0098|0)?9\d{9}$', message="Enter a valid Iranian mobile number (e.g. 09123456789).")`
     — adjust the exact regex if you find a more precise/commonly-used
     pattern, but it must accept `09123456789`, `+989123456789`, and
     `00989123456789`, and reject anything not matching Iranian mobile
     prefixes (must start with 9 after any country/trunk prefix is
     stripped, followed by exactly 9 more digits).
  2. A `normalize_iranian_phone(value: str) -> str` function that takes
     any of the three accepted input formats and returns a single
     canonical stored format — use `09XXXXXXXXX` (local format, 11
     digits) as the canonical form stored in the DB, since that's what
     Iranian SMS gateways (Kavenegar, etc.) typically expect, but
     double check this assumption against Kavenegar's documented
     input format before finalizing — if their API actually expects
     `98XXXXXXXXXX` without a leading zero or plus, use THAT as the
     canonical form instead and note the reason in a code comment. Pick
     one and be consistent everywhere.
- Apply `validators=[iranian_phone_regex]` to `User.phone_number` in
  models.py.
- Override `User.save()` (or add a `clean()` method — whichever fits
  the existing model's pattern; there's no existing override on this
  model currently, so either is fine, but prefer `clean()` +ensure
  callers run full_clean, OR normalize directly in `save()` since DRF
  serializers typically call `.save()` not `.full_clean()` by default —
  pick `save()` normalization to guarantee it actually runs in the API
  path, and call `normalize_iranian_phone()` on `self.phone_number`
  before calling `super().save()`, only if `self.phone_number` is
  truthy).
- Add a migration if the validator addition requires one (validators
  don't always require a migration in Django, but generate one if
  `makemigrations` produces one — don't force it if not needed).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/accounts/tests/test_models.py:
1. `normalize_iranian_phone("+989123456789")`,
   `normalize_iranian_phone("00989123456789")`, and
   `normalize_iranian_phone("09123456789")` all return the same
   canonical value.
2. Saving a User with an invalid phone format (e.g. `"12345"`, a US
   number, or a landline-looking number) raises a ValidationError when
   `full_clean()` is called, OR is rejected at the serializer layer once
   Task 2.1.2/2.3.1 serializers exist (note: at the model level alone,
   Django doesn't enforce validators on plain `.save()" unless you
   explicitly call `full_clean()` — verify what your `save()` override
   from the previous bullet actually enforces, and write the test
   against the REAL enforcement point, not an assumed one).
3. Saving a User with a valid phone in any of the three formats results
   in the same normalized value being persisted to the DB regardless of
   which input format was used.
```

---

#### Task 2.1.1.3 — Data migration: backfill phone from `Profile.phone_number`

```
You are working in backend/accounts/migrations/. Assume Tasks 2.1.1.1
and 2.1.1.2 are already merged (User.phone_number field + normalizer
exist).

CONTEXT
Existing users who signed up via the current email/password flow may
already have a phone number stored on their `Profile.phone_number`
(a free-text CharField with no validation, set via ProfileView PATCH).
That data would otherwise be silently orphaned now that phone lives on
User — someone who already gave their number should not have to
re-enter it just because the schema moved.

TASK
Write a Django data migration that copies valid phone numbers from
Profile.phone_number to the new User.phone_number field, normalizing
them and skipping/logging anything that doesn't validate.

REQUIREMENTS
- Create a new migration file in backend/accounts/migrations/ using
  `python manage.py makemigrations accounts --empty --name backfill_user_phone_from_profile`
  and fill in the `RunPython` operation yourself.
- In the forward function:
  1. Iterate all `Profile` rows (via the historical model API —
     `apps.get_model("accounts", "Profile")` and
     `apps.get_model("accounts", "User")` — do NOT import the real
     model classes directly in a data migration, that's a Django
     anti-pattern that breaks under future schema changes).
  2. For each Profile with a non-blank `phone_number`, attempt to
     normalize it using the same logic as
     `normalize_iranian_phone()` — since you can't easily import
     app code into a migration cleanly across Django versions, either
     (a) duplicate the minimal regex/normalization logic inline in the
     migration file with a comment explaining why it's duplicated, or
     (b) import the function directly if the project's Django version
     and migration conventions safely support it (check whether other
     migrations in this repo import from app modules — if none do,
     stick with (a) for safety).
  3. If normalization succeeds and the resulting number doesn't already
     exist on another User (respect the unique constraint — check via
     `User.objects.filter(phone_number=normalized).exclude(pk=profile.user_id).exists()`
     before setting it), set `user.phone_number = normalized` and save.
  4. If normalization fails (invalid format) or the number is already
     taken by a different user, skip that row silently (do not raise —
     a migration must not fail the deploy over dirty legacy data) but
     print a warning via `print()` or Django's migration-safe logging so
     it's visible in migration output for manual follow-up.
- Add a no-op (or best-effort reverse) `reverse_code` function — since
  this is a backfill from one already-existing field to another, the
  reverse can simply be a no-op (`RunPython.noop`) since undoing it
  isn't meaningful/necessary, but note this decision in a migration
  comment.

ACCEPTANCE CRITERIA / TESTS
Write a migration test (check whether this project already has any
pattern for testing data migrations — search backend/accounts/tests/
and backend/*/migrations/ for precedent; if none exists, use
`django.test.TestCase` with `pytest-django`'s migration testing
utilities, or write a lightweight test that manually invokes the
migration's forward function against test data):
1. A Profile with a valid, unique phone number results in that number
   appearing on the corresponding User after the migration runs.
2. A Profile with an invalid phone number format is skipped without
   raising, and the User's phone_number remains None.
3. Two Profiles that happen to have the same phone number string (data
   quality issue in existing data) result in only ONE of the two Users
   getting that phone number set (the other stays None) — no
   IntegrityError crashes the migration.
```

---

### Feature 2.1.2 — OTP Infrastructure

---

#### Task 2.1.2.1 — Create `OTPCode` model

```
You are working in the backend/accounts Django app. Assume Feature
2.1.1 (User.phone_number) is already merged.

CONTEXT
There is currently no OTP/one-time-code infrastructure anywhere in the
codebase — authentication is entirely email+password via SimpleJWT
(see backend/accounts/views.py LoginView(TokenObtainPairView) and
RegisterView). This task lays the foundation model for phone-based OTP
login/registration.

TASK
Create an `OTPCode` model in backend/accounts/models.py (in the same
file as User/Profile, or a new backend/accounts/models_otp.py imported
into models.py if you prefer to keep files smaller — check the existing
file's length and decide; if you split it, make sure Django still
discovers it via an import in models.py or an appropriately configured
app config).

REQUIREMENTS
- Fields:
  - `phone_number = models.CharField(max_length=15, db_index=True)` —
    NOT a FK to User, because OTP requests can happen before a User
    exists yet (first-time registration via phone) — this must work
    standalone.
  - `purpose = models.CharField(max_length=20, choices=[("login", "Login"), ("register", "Register"), ("reset", "Password Reset")])`
  - `code_hash = models.CharField(max_length=128)` — NEVER store the
    raw 6-digit code in the database in plaintext; store a hash (Task
    2.1.2.2 handles hashing logic, this task just needs the field to
    exist).
  - `expires_at = models.DateTimeField()`
  - `attempts = models.PositiveSmallIntegerField(default=0)` — tracks
    failed verification attempts against this specific code.
  - `is_used = models.BooleanField(default=False)`
  - `created_at = models.DateTimeField(auto_now_add=True)`
- Add `class Meta: indexes = [models.Index(fields=["phone_number", "purpose", "is_used"])]`
  since the verification lookup will filter on exactly this
  combination.
- Add a `__str__` returning something like
  `f"OTP for {self.phone_number} ({self.purpose}) - {'used' if self.is_used else 'active'}"`.
- Do NOT add any methods for generating/hashing/verifying codes in this
  task — that's Tasks 2.1.2.2 and 2.1.2.3. This task is schema only.
- Generate the migration.
- Register `OTPCode` in backend/accounts/admin.py (read-only in admin —
  staff should be able to see OTP request volume/history for support
  and abuse investigation, but should NEVER be able to view/edit
  `code_hash`; exclude it from the admin list/detail view, or make the
  whole model list_display-only with no edit form, staff's call, but
  do not surface the hash even to superusers in the admin UI).

ACCEPTANCE CRITERIA / TESTS
- Migration applies cleanly.
- Add a basic model test confirming an OTPCode instance can be created
  with all required fields and `str()` produces a readable
  representation.
```

---

#### Task 2.1.2.2 — Add OTP generation service

```
You are working in backend/accounts/services/otp.py (new file/package).
Assume Task 2.1.2.1 (OTPCode model) is already merged.

CONTEXT
The OTPCode model exists but nothing populates it yet. This task builds
the service that generates a new 6-digit code, hashes it for storage,
and enforces a sane request rate before creating a new row (a full
per-phone throttle at the DRF-view level comes in Task 2.1.2.4 — this
task's rate check is a simpler in-service guard against generating
back-to-back codes for the same phone within the same TTL window,
independent of whatever throttle class ends up wrapping the eventual
view).

TASK
Create `backend/accounts/services/__init__.py` and
`backend/accounts/services/otp.py` with a service function/class that:
1. Generates a random 6-digit numeric code.
2. Hashes it before storage (never store the raw code).
3. Creates an OTPCode row with a configurable TTL.
4. Returns the RAW (unhashed) code so the caller can send it via SMS —
   this is the only place the raw code should ever exist outside of the
   SMS payload itself.

REQUIREMENTS
- Use Python's `secrets` module (NOT `random`) for generating the code
  — `secrets.randbelow(1_000_000)` zero-padded to 6 digits, or
  `"".join(secrets.choice("0123456789") for _ in range(6))` — either is
  fine, but it must be cryptographically secure, not `random.randint`.
- Hash the code using Django's existing password hashing infrastructure
  for consistency with the rest of the codebase — use
  `django.contrib.auth.hashers.make_password(code)` to hash and
  `check_password(code, hash)` to verify later (Task 2.1.2.3), rather
  than rolling custom hashing. This reuses Django's already-audited,
  already-configured hasher stack.
- Define a settings-driven TTL: add
  `OTP_CODE_TTL_SECONDS = config("OTP_CODE_TTL_SECONDS", default=120, cast=int)`
  to backend/core/settings/base.py (matching the existing
  `python-decouple` `config()` pattern already used throughout that
  file), defaulting to 2 minutes as specified in the backlog.
- The generation function signature should look roughly like:
  `def generate_otp(phone_number: str, purpose: str) -> str:` — creates
  the OTPCode row (with `expires_at = timezone.now() + timedelta(seconds=settings.OTP_CODE_TTL_SECONDS)`)
  and returns the raw code string. Import `timezone` from
  `django.utils`.
- Before creating a new OTPCode, check for an existing, still-valid,
  unused OTPCode for the same `(phone_number, purpose)` created within
  the last N seconds (use a short cooldown, e.g. 60 seconds — configurable
  via another settings constant `OTP_RESEND_COOLDOWN_SECONDS`) and, if
  one exists, raise a custom exception (e.g. `OTPCooldownError`) rather
  than silently issuing a second SMS — this prevents accidental
  double-submission from also being a full abuse-throttle bypass at the
  service layer (Task 2.1.2.4 adds the harder per-view rate limit; this
  is a lighter, always-on guard inside the service itself regardless of
  which view calls it).
- Define `OTPCooldownError` and any other custom exceptions in the same
  `otp.py` module.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/accounts/tests/ (new test_otp_service.py or
similar, matching the existing tests/ package structure):
1. `generate_otp()` returns a 6-digit numeric string.
2. Calling it creates exactly one OTPCode row with `is_used=False`,
   a non-empty `code_hash` that does NOT equal the raw returned code
   (proving it's actually hashed, not stored in plaintext).
3. `expires_at` is set correctly based on `OTP_CODE_TTL_SECONDS`.
4. Calling `generate_otp()` twice in immediate succession for the same
   phone+purpose raises `OTPCooldownError` on the second call.
5. Calling it for the same phone but a DIFFERENT purpose (e.g. "login"
   then "register") does NOT trigger the cooldown — cooldown is scoped
   per (phone, purpose).
```

---

#### Task 2.1.2.3 — Add OTP verification service

```
You are working in backend/accounts/services/otp.py. Assume Task
2.1.2.2 (generate_otp) is already merged.

CONTEXT
Codes can now be generated and hashed into OTPCode rows, but nothing
verifies a user-submitted code against them yet.

TASK
Add a `verify_otp(phone_number: str, purpose: str, submitted_code: str) -> bool`
function to the same otp.py service module that validates a submitted
code against the most recent matching OTPCode row, enforcing expiry and
a maximum attempt count.

REQUIREMENTS
- Look up the most recent, unused OTPCode for
  `(phone_number=phone_number, purpose=purpose, is_used=False)`,
  ordered by `-created_at`. If none exists, return `False` (or raise a
  distinct `OTPNotFoundError` — pick whichever makes the calling view's
  error handling cleaner and be consistent; raising distinct, specific
  exceptions for "not found" vs "expired" vs "max attempts" vs "wrong
  code" is preferable so the calling API view (Task 2.3.1.2) can return
  precise, user-friendly error messages rather than one generic
  "invalid code" for every failure mode).
- If `expires_at < timezone.now()`, mark it unusable somehow (either
  leave `is_used=False` but the expiry check itself blocks it every
  time, or explicitly do nothing to the row — don't mark expired codes
  as "used", since "used" should specifically mean "successfully
  verified", not "expired/abandoned" — keep these concepts distinct)
  and raise/return the expired-specific result.
- If `attempts >= 5` (define as a settings constant
  `OTP_MAX_VERIFICATION_ATTEMPTS = config(..., default=5, cast=int)` in
  base.py, matching the pattern from Task 2.1.2.2), raise/return a
  max-attempts-exceeded result WITHOUT even checking the code — a
  maxed-out code must be permanently unusable regardless of whether the
  final attempt happens to be correct (this closes a subtle bypass:
  otherwise an attacker gets unlimited guesses as long as they never
  actually enter the eventual real code).
- Otherwise, check `check_password(submitted_code, otp.code_hash)`. If
  correct: set `is_used=True`, save, and return `True` (success). If
  incorrect: increment `attempts`, save, and return `False` (or raise
  the wrong-code-specific exception), WITHOUT revealing in the response
  how many attempts remain in a way that could aid brute-forcing (a
  generic "incorrect code" message is fine; don't return "3 attempts
  remaining" type detail that helps an attacker calibrate).

ACCEPTANCE CRITERIA / TESTS
Add tests to test_otp_service.py:
1. Verifying the correct, unexpired code succeeds and marks the row
   `is_used=True`.
2. Verifying an incorrect code fails, increments `attempts` by 1, and
   the row remains usable for further attempts (until the cap).
3. Verifying after `expires_at` has passed fails even with the correct
   code (use `freezegun` or manual timestamp manipulation — check
   requirements.txt/test deps for whether a time-freezing library is
   already available; if not, directly construct an OTPCode with a
   past `expires_at` rather than needing to freeze real time).
4. Verifying after `attempts` has already reached the configured max
   fails immediately even with the correct code, and does not further
   increment attempts past the cap.
5. Verifying an already-`is_used=True` code fails (can't reuse a
   successfully-verified code twice).
6. Verifying against a phone number with no OTPCode at all fails
   cleanly without raising an unhandled exception.
```

---

#### Task 2.1.2.4 — Add per-phone OTP request throttle

```
You are working in backend/accounts/ (likely a new throttles.py, DRF's
throttle mechanism) and backend/core/settings/base.py. Assume Tasks
2.1.2.2 and 2.1.2.3 are already merged.

CONTEXT
Task 2.1.2.2 added a lightweight in-service cooldown (one code per
phone+purpose per ~60 seconds) to stop accidental double-taps, but that
alone doesn't stop a scripted attacker from requesting OTP codes for
many different purposes, or simply waiting out each cooldown
repeatedly, to spam a phone number with SMS messages (an
"SMS-bombing" cost/abuse attack against whoever's Kavenegar account
ends up paying per message once Epic 16 wires up real SMS sending).
This needs a harder, DRF-level rate limit independent of the service's
own cooldown logic.

TASK
Create a custom DRF throttle class scoped specifically to OTP requests,
keyed by phone number (not by IP/user, since the attacker's phone
number is the resource being protected, and IP-based throttling alone
is easy to route around with proxies while still hitting the same
victim phone number).

REQUIREMENTS
- Create backend/accounts/throttles.py with a class
  `PhoneOTPRequestThrottle(SimpleRateThrottle)` (import from
  `rest_framework.throttling`).
- Override `get_cache_key()` to build the cache key from the phone
  number in the request body (`request.data.get("phone_number")`), not
  from `request.user` or IP — DRF's default `SimpleRateThrottle.get_cache_key`
  assumes either user or IP; override it to fall back to a
  "no phone provided" bucket (e.g. IP-based) if the field is missing,
  so malformed requests still get some throttling rather than bypassing
  it entirely.
- Set `scope = "otp_request"` and add to
  `REST_FRAMEWORK["DEFAULT_THROTTLE_RATES"]` in base.py (this dict
  doesn't exist yet — you'll be creating
  `DEFAULT_THROTTLE_CLASSES`/`DEFAULT_THROTTLE_RATES` for the first
  time in this codebase; check whether Epic 1 or a prior task already
  added global throttle classes before assuming a clean slate — if
  `DEFAULT_THROTTLE_CLASSES` already exists in REST_FRAMEWORK, add to
  it rather than overwriting it) something like
  `"otp_request": "3/10min"` — 3 requests per phone number per 10
  minutes, matching the backlog's stated requirement.
- This throttle class will be applied to the actual OTP-request view in
  Task 2.3.1.1 (`throttle_classes = [PhoneOTPRequestThrottle]` on that
  view) — this task only builds and unit-tests the throttle class
  itself in isolation, since the view doesn't exist yet.

ACCEPTANCE CRITERIA / TESTS
Add tests to a new backend/accounts/tests/test_throttles.py:
1. Directly instantiate `PhoneOTPRequestThrottle` and call
   `.allow_request()` against a mock/fake DRF request object with a
   given phone_number in `request.data` 3 times in a row — first 3
   calls return True, 4th call within the window returns False.
2. Two DIFFERENT phone numbers each get their own independent 3-request
   allowance (proving the cache key is correctly scoped per phone, not
   globally).
3. A request with no phone_number in the body doesn't crash the
   throttle (falls back gracefully per the requirement above).

Note: DRF throttle tests typically need Django's cache backend
configured and cleared between tests (`cache.clear()` in setUp/teardown)
— check backend/core/settings/development.py for the configured
`CACHES` backend (if none exists yet, the default LocMemCache is fine
for this test) and use it correctly.
```

---

## Phase 2.2 — SMS Provider Integration

*(Full Kavenegar production integration is out of scope here — tracked separately as Epic 16. This phase only builds a swappable interface so Epic 2's auth flow doesn't hard-depend on a real SMS vendor yet.)*

### Feature 2.2.1 — SMS Gateway Abstraction

---

#### Task 2.2.1.1 — Define `SMSProvider` interface

```
You are creating a new backend/notifications app (or, if you prefer not
to create a whole new Django app just for this interface yet, a plain
Python package backend/accounts/sms/ — decide based on whether this
interface is likely to be reused outside of accounts soon; given Epic
16 of the project backlog explicitly plans a full `notifications` app
later, it's reasonable to create the lightweight interface now inside
backend/accounts/sms/ and let Epic 16 relocate/absorb it later, OR to
create the `notifications` app scaffold now and put it there directly
so Epic 16 doesn't have to move code — your call, but document the
decision in a short comment at the top of the module).

CONTEXT
The OTP system (Feature 2.1.2) generates codes but has no way to
actually deliver them to a phone yet. Rather than hard-coding a call to
a specific SMS vendor's SDK directly inside the OTP service (which
would make the auth flow untestable without real API keys and would
create tight coupling to one vendor), this task defines a small,
swappable interface that any SMS backend can implement.

TASK
Define an abstract `SMSProvider` base class with a single `send()`
method, plus a Django-settings-driven factory function that returns the
currently configured provider instance.

REQUIREMENTS
- Create the interface:
  ```python
  from abc import ABC, abstractmethod

  class SMSProvider(ABC):
      @abstractmethod
      def send(self, phone_number: str, message: str) -> bool:
          """Send `message` to `phone_number`. Returns True on success."""
          raise NotImplementedError
  ```
- Add a factory function, e.g. `get_sms_provider() -> SMSProvider`,
  that reads a new settings value `SMS_PROVIDER_CLASS` (a dotted import
  path string, e.g. `"accounts.sms.console.ConsoleSMSProvider"`) and
  dynamically imports/instantiates it, using Django's
  `django.utils.module_loading.import_string` helper. This mirrors how
  Django itself resolves things like `AUTH_USER_MODEL` and keeps the
  provider fully swappable via settings/env with zero code changes.
- Add `SMS_PROVIDER_CLASS = config("SMS_PROVIDER_CLASS", default="accounts.sms.console.ConsoleSMSProvider")`
  to backend/core/settings/base.py, following the existing
  `python-decouple` config() pattern.
- Do not implement any concrete provider in this task (that's Task
  2.2.1.2 for the console/dev backend).

ACCEPTANCE CRITERIA / TESTS
Add a test confirming `get_sms_provider()` raises a clear, actionable
error (not a cryptic ImportError) if `SMS_PROVIDER_CLASS` points at a
class that doesn't exist or doesn't implement the `SMSProvider`
interface (e.g. check `isinstance(instance, SMSProvider)` after
instantiation and raise `ImproperlyConfigured` with a helpful message
if it doesn't match).
```

---

#### Task 2.2.1.2 — Add console/dev SMS backend

```
You are working in the same location as Task 2.2.1.1's SMSProvider
interface. Assume that task is already merged.

CONTEXT
There's no way to test or locally develop the OTP flow yet without a
concrete SMSProvider implementation. Real vendor integration
(Kavenegar) is explicitly out of scope for this epic (tracked in Epic
16) — this task just needs something that works for local development
and CI without any external API calls or credentials.

TASK
Implement a `ConsoleSMSProvider` that "sends" SMS by printing/logging
the message instead of making any real network call, so the OTP flow
can be developed, demoed, and tested end-to-end today.

REQUIREMENTS
- Implement:
  ```python
  class ConsoleSMSProvider(SMSProvider):
      def send(self, phone_number: str, message: str) -> bool:
          logger.info("SMS to %s: %s", phone_number, message)
          print(f"[DEV SMS] To: {phone_number} | Message: {message}")
          return True
  ```
  using Python's `logging` module (get a logger named appropriately,
  e.g. `logging.getLogger("accounts.sms")`) IN ADDITION TO the print
  statement — the print is for immediate local-dev visibility in the
  terminal, the logger is for anything that captures Django logs (CI
  output, docker logs, etc.).
- Set `SMS_PROVIDER_CLASS` to point at this class by default in
  backend/core/settings/development.py specifically (confirm this
  settings file exists and extends base.py's default appropriately —
  do NOT set the console backend as the production.py default; leave
  production.py's value unset/inherited-from-base for now, since
  Task 2.2.1.3 and Epic 16 will need to make a deliberate decision
  about what production actually points at, and defaulting production
  to a no-op console backend that silently "succeeds" without sending
  real SMS would be a dangerous silent failure mode if anyone forgets
  to override it before a real launch).
- Add a code comment on `ConsoleSMSProvider` clearly stating it must
  NEVER be used in production, since it doesn't send anything.

ACCEPTANCE CRITERIA / TESTS
Add a test confirming `ConsoleSMSProvider().send(phone, message)`
returns `True` and doesn't raise, and (if feasible) captures the log
output via Python's `caplog`/`assertLogs` to confirm the message was
actually logged with the expected phone number and message content.
```

---

#### Task 2.2.1.3 — Wire OTP service to SMS provider interface

```
You are working in backend/accounts/services/otp.py. Assume Tasks
2.1.2.2 (generate_otp), 2.2.1.1 (SMSProvider interface + factory), and
2.2.1.2 (ConsoleSMSProvider) are already merged.

CONTEXT
`generate_otp()` currently creates an OTPCode row and returns the raw
code, but nothing actually sends it anywhere — that's the caller's
responsibility today, which is fragile (easy to forget, easy to
duplicate inconsistently across multiple call sites as more OTP-using
features get added later, e.g. password reset via phone).

TASK
Make `generate_otp()` (or a thin wrapper around it) responsible for
actually dispatching the SMS via the configured `SMSProvider`, so
callers get one single function that both creates and sends the code.

REQUIREMENTS
- Import `get_sms_provider` from the module created in Task 2.2.1.1.
- After successfully creating the OTPCode row and obtaining the raw
  code, build a Persian-appropriate message string, e.g.:
  `f"Your verification code is: {code}"` — NOTE: full Persian
  localization of this string is tracked separately under Epic 14
  (Persian Localization) in the project backlog; for THIS task, a
  plain English placeholder string is acceptable, but leave a
  `# TODO: Epic 14 — localize this message to Persian` comment so it
  isn't forgotten.
- Call `get_sms_provider().send(phone_number, message)`.
- If `send()` returns `False` (or raises an exception from a future
  real provider), the OTPCode row should still exist (don't roll it
  back — the code is still valid and the user could theoretically be
  informed via another channel or retry), but `generate_otp()` should
  propagate the failure to its caller in some way — either return a
  tuple `(code, sent_successfully)` instead of just `code`, or raise a
  dedicated `SMSDeliveryError` — pick one, but make sure whichever view
  ends up calling this (Task 2.3.1.1) can distinguish "the OTP was
  created but SMS delivery failed" from "everything worked," since
  those need different user-facing responses (the former should
  probably still return a generic success message to avoid confirming
  to an attacker whether a phone number is real/reachable, but should
  be logged/alertable for ops visibility).
- Update the function's docstring and any existing tests from Task
  2.1.2.2 that called `generate_otp()` directly — they'll now also
  trigger a "send" through the ConsoleSMSProvider (harmless in tests,
  just printed/logged), so no test behavior should break, but review
  them to confirm.

ACCEPTANCE CRITERIA / TESTS
Add/update tests in test_otp_service.py:
1. Calling `generate_otp()` results in exactly one call to the
   configured SMSProvider's `send()` method (use `unittest.mock.patch`
   to mock `get_sms_provider` and assert `send` was called once with
   the correct phone number and a message containing the raw code).
2. If the mocked `send()` returns False, `generate_otp()` still creates
   the OTPCode row (verify via `OTPCode.objects.count()`) but surfaces
   the failure per whichever mechanism you chose above.
```

---

## Phase 2.3 — OTP Auth Endpoints

### Feature 2.3.1 — Login/Register via OTP

---

#### Task 2.3.1.1 — `POST /api/auth/otp/request/` endpoint

```
You are working in backend/accounts/views.py, serializers.py, and
urls.py. Assume all of Feature 2.1.2 (OTP model/generate/verify) and
2.2.1 (SMS provider wiring) and Task 2.1.2.4 (throttle class) are
already merged.

CONTEXT
Existing auth endpoints (backend/accounts/urls.py) are:
  register/, login/, token/refresh/, user/, profile/, change-password/,
  password-reset/, password-reset/confirm/
None of them support phone/OTP. This task adds the first half of the
OTP flow: requesting a code be sent.

TASK
Add a new `OTPRequestView` (APIView) at `POST /api/auth/otp/request/`
that accepts a phone number, validates its format, and triggers
`generate_otp()` for the "login" purpose (this single endpoint is used
for BOTH login and registration — see Task 2.3.1.2 for why the split
happens at verify-time, not request-time: at request-time we don't yet
know if this is a new or returning user, and revealing that via a
different response would itself be a user-enumeration risk).

REQUIREMENTS
- Create `OTPRequestSerializer(serializers.Serializer)` in
  serializers.py with a single field:
  `phone_number = serializers.CharField()`, validated using the
  `iranian_phone_regex` validator from Task 2.1.1.2 (import it) inside
  a `validate_phone_number()` method, and normalize it via
  `normalize_iranian_phone()` before returning.
- Create `OTPRequestView(APIView)`:
  - `permission_classes = [AllowAny]` (phone/OTP requests happen before
    the user is authenticated by definition).
  - `throttle_classes = [PhoneOTPRequestThrottle]` (from Task 2.1.2.4).
  - `post()`: validate the serializer, then call
    `generate_otp(phone_number, purpose="login")` inside a try/except
    catching `OTPCooldownError` (from Task 2.1.2.2) — if raised, return
    a 429 (Too Many Requests) with a friendly "please wait before
    requesting another code" message; otherwise return 200 with a
    generic response like `{"detail": "Verification code sent."}` —
    deliberately vague, do NOT include the actual code, phone number
    confirmation details, or whether the phone is already registered
    (mirrors the existing PasswordResetRequestView pattern in this same
    file, which already deliberately always returns 200 "to prevent
    user enumeration" — follow that exact same principle here).
- Register the URL in backend/accounts/urls.py:
  `path("otp/request/", OTPRequestView.as_view(), name="otp-request")`.
- Add an `AllowAny` OpenAPI/drf-spectacular annotation if other views
  in this file use `@extend_schema` decorators — check the file for
  precedent and match the existing documentation style.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/accounts/tests/test_views.py:
1. Valid phone number → 200, and exactly one OTPCode row created with
   `purpose="login"`.
2. Invalid phone number format → 400 with a field-level error on
   `phone_number`.
3. Requesting twice in immediate succession for the same phone → second
   request returns 429 (the cooldown from Task 2.1.2.2 surfacing
   correctly through this view).
4. 4th request within the throttle window (per Task 2.1.2.4's 3/10min
   rate) returns 429 from the DRF throttle layer specifically (you may
   need to bypass/mock the service-level cooldown from bullet 3 to
   isolate testing the throttle layer, e.g. by using different phone
   numbers for the cooldown test vs. the throttle test, or by
   controlling time — pick whichever keeps the tests independent and
   readable).
```

---

#### Task 2.3.1.2 — `POST /api/auth/otp/verify/` endpoint

```
You are working in backend/accounts/views.py, serializers.py, and
urls.py. Assume Task 2.3.1.1 (OTP request endpoint) is already merged.

CONTEXT
This is the second half of the OTP flow: the user submits the code they
received via SMS, and the system either logs them into an existing
account or creates a brand-new one on the spot (Iranian consumer apps
overwhelmingly favor this "OTP creates the account automatically on
first success" pattern over a separate explicit registration step, and
it matches how RegisterView already issues JWTs immediately on success,
so this task follows the same shape).

TASK
Add `OTPVerifyView` at `POST /api/auth/otp/verify/` that verifies the
submitted code, finds-or-creates the User by phone_number, and returns
a JWT access/refresh pair exactly like LoginView does today.

REQUIREMENTS
- Create `OTPVerifySerializer(serializers.Serializer)` with:
  `phone_number = serializers.CharField()` (same validator/normalizer
  as Task 2.3.1.1) and `code = serializers.CharField(max_length=6, min_length=6)`.
- Create `OTPVerifyView(APIView)`:
  - `permission_classes = [AllowAny]`.
  - `post()`:
    1. Validate the serializer.
    2. Call `verify_otp(phone_number, purpose="login", submitted_code=code)`
       from Task 2.1.2.3. Handle each of its distinct failure modes
       (not found / expired / max attempts / wrong code — whichever
       exception scheme you chose in that task) with appropriate,
       specific 400 responses (e.g. `{"code": "This code has expired. Please request a new one."}`
       vs `{"code": "Incorrect code."}` vs
       `{"code": "Too many incorrect attempts. Please request a new code."}`)
       — unlike the request endpoint, THIS endpoint's error messages
       can be more specific since the user has already proven control
       of the phone number by receiving the code in the first place;
       user-enumeration concerns from Task 2.3.1.1 don't apply the same
       way here.
    3. On successful verification: look up
       `User.objects.filter(phone_number=phone_number).first()`. If
       found, use that existing user. If not found, create a new one:
       `User.objects.create_user(email=None, phone_number=phone_number, is_verified=True)`
       — WAIT: check `UserManager.create_user()` in models.py; it
       currently REQUIRES `email` and calls `self.normalize_email(email).lower()`
       unconditionally, which will crash on `email=None`. You need to
       either (a) update `UserManager.create_user()` to make `email`
       optional (only normalize/validate it if provided, and generate
       or leave blank/null otherwise — check whether `User.email` is
       currently required/unique at the DB level and whether making it
       nullable for phone-only accounts is safe given `USERNAME_FIELD = "email"`
       still requires SOME value for Django admin login purposes for
       staff — phone-only customer accounts likely don't need
       `is_staff` access anyway, so a null/blank email should be fine
       for pure customers, but flag this clearly since it's a
       non-trivial model-manager change with ripple effects on
       `USERNAME_FIELD` semantics — email must remain unique but can be
       null, matching the same nullable+unique pattern already used for
       phone_number in Task 2.1.1.1), or (b) generate a placeholder
       unique non-colliding value — option (a), making email properly
       optional, is the correct long-term fix; do that, not a
       placeholder-email hack.
    4. Set `user.is_verified = True` if not already (phone verification
       via OTP is itself a form of identity verification).
    5. Issue JWT tokens exactly like RegisterView does:
       `from rest_framework_simplejwt.tokens import RefreshToken; refresh = RefreshToken.for_user(user)`.
    6. Return `{"access": ..., "refresh": ..., "is_new_user": <bool>}`
       — include `is_new_user` so the frontend (Task 2.3.2) can decide
       whether to show a "complete your profile" prompt (first/last
       name are currently required-non-blank on Profile per
       `first_name = models.CharField(max_length=255, blank=False)` —
       a phone-only signup will have an empty Profile.first_name/last_name,
       which violates that `blank=False` constraint at the form/admin
       validation level, though NOT at the raw `.save()` level since
       Django only enforces `blank=False` via `full_clean()`/ModelForms,
       not via bare `.save()` — confirm this doesn't crash the
       auto-created Profile from the `create_profile` post_save signal,
       and flag in a comment that a "complete your profile" frontend
       step is needed to fill these in later, consistent with the
       `is_new_user` flag this endpoint returns).
- Register the URL: `path("otp/verify/", OTPVerifyView.as_view(), name="otp-verify")`.

ACCEPTANCE CRITERIA / TESTS
Covered together with Task 2.3.1.4's integration test suite — but
before moving on, at minimum confirm manually (or via a quick smoke
test) that:
1. Verifying a valid code for a brand-new phone number creates exactly
   one new User with that phone_number, is_verified=True, and returns
   201-or-200 (pick and document the status code) with access/refresh
   tokens and `is_new_user: true`.
2. Verifying a valid code for a phone number that already has a User
   logs into that SAME existing user (no duplicate created) and returns
   `is_new_user: false`.
```

---

#### Task 2.3.1.3 — Auto-create `Profile` on first OTP registration

```
You are working in backend/accounts/models.py (the existing
`create_profile` post_save signal) and backend/accounts/views.py
(OTPVerifyView from Task 2.3.1.2). Assume that task is already merged.

CONTEXT
The existing signal:

    @receiver(post_save, sender=User)
    def create_profile(sender, instance, created, **kwargs):
        if created:
            Profile.objects.create(user=instance)

fires for ANY new User, regardless of how they were created — this
already covers OTP-created users automatically, since Task 2.3.1.2's
`User.objects.create_user(...)` call triggers `post_save` with
`created=True` just like the existing email/password RegisterView does.
This task is primarily a VERIFICATION task, not a new-code task — but
there's one real gap to close: `Profile.first_name`/`last_name` are
`blank=False`, and an OTP-created user has no name data at all yet, so
the auto-created Profile will have empty strings in those fields, which
is a real (if soft) constraint violation waiting to cause confusing
`full_clean()` failures later (e.g. if the admin panel or any future
form calls `profile.full_clean()`).

TASK
Verify the existing signal correctly handles OTP-created users
end-to-end, and decide/implement how to handle the blank
first_name/last_name gap.

REQUIREMENTS
- Trace through the actual code path: does
  `User.objects.create_user(email=None, phone_number=..., is_verified=True)`
  (post Task 2.3.1.2's manager fix) correctly trigger `post_save` with
  `created=True`? (It should, since `create_user` ultimately calls
  `user.save(using=self._db)` — confirm no code path bypasses `.save()`
  entirely, e.g. via `bulk_create`, which would skip signals.)
- Decide how to handle blank name fields on OTP-created profiles: the
  cleanest option is to relax `Profile.first_name`/`last_name` to
  `blank=True` (they already have no `default`, so this just permits
  empty strings at the form-validation layer without changing DB
  schema — no destructive migration needed for existing data since
  blank strings were already technically insertable at the raw DB
  level, `blank=False` is a Django-level validation-only constraint,
  not a DB NOT-NULL-with-content constraint). Generate the trivial
  migration this produces.
- Update `Profile.get_fullname()` — it currently already handles empty
  names gracefully (`return _("new user")` fallback), so no change
  needed there; just confirm this via a quick read, don't modify it
  unless you find it's actually broken.
- In `OTPVerifyView`'s response (Task 2.3.1.2), when `is_new_user=True`,
  the frontend needs to know a name is missing so it can prompt for it
  — this is frontend scope (see Task 2.3.2), but on the backend, make
  sure `CurrentUserSerializer` (used by `GET /api/auth/user/`, which
  the frontend calls right after storing tokens) correctly reflects
  empty first_name/last_name as empty strings (not error out) so the
  frontend can detect "profile incomplete" by checking if those fields
  are falsy.

ACCEPTANCE CRITERIA / TESTS
1. Add a test confirming an OTP-created User (via the service/manager
   directly, not necessarily the full view) has an associated Profile
   automatically created via the signal, with `first_name=""` and
   `last_name=""`, and that `profile.full_clean()` no longer raises
   after the `blank=True` migration.
2. Add a test confirming `GET /api/auth/user/` for a freshly
   OTP-registered user returns 200 with empty-string name fields rather
   than erroring.
```

---

#### Task 2.3.1.4 — Add OTP flow integration tests

```
You are adding tests to backend/accounts/tests/. Assume Tasks 2.3.1.1
through 2.3.1.3 are all merged (full OTP request→verify→JWT flow
exists end-to-end).

CONTEXT
Individual unit tests exist for the OTP service (Feature 2.1.2) and
light coverage exists from the endpoint tasks themselves, but there's
no single test proving the FULL user-facing flow works correctly
end-to-end through the real HTTP API, the way a real client (the
frontend from Task 2.3.2, or a mobile app) would actually use it.

TASK
Write a comprehensive integration test suite covering the complete
request → verify → authenticated flow, plus the key abuse/edge cases,
using DRF's `APIClient` (matching whatever test client pattern the
existing backend/accounts/tests/test_views.py already uses — check it
first and stay consistent).

REQUIREMENTS — test scenarios to cover
1. **Happy path, new user:** POST to otp/request/ with a fresh phone
   number → 200. Retrieve the raw code (since ConsoleSMSProvider only
   logs/prints it, you'll need to either capture it via
   `unittest.mock.patch` on `generate_otp` to intercept the return
   value, or query `OTPCode.objects.latest("created_at")` and use
   `verify_otp`'s hash-check indirectly isn't possible from a black-box
   test since the raw code isn't retrievable from the hash — the
   cleanest approach is to mock/patch the OTP generation to use a KNOWN
   fixed code for this test, e.g. patch `secrets.choice`/whatever
   randomness source `generate_otp` uses, OR add a test-only helper
   that captures the code via mocking `SMSProvider.send` and parsing it
   out of the message string it was called with — pick whichever is
   less brittle). POST to otp/verify/ with that code → 200, returns
   access/refresh, `is_new_user: true`. Confirm exactly one new User
   exists with the submitted phone number.
2. **Happy path, returning user:** repeat the full flow for a phone
   number that already has a User → `is_new_user: false`, same User's
   `pk` is the one associated with the issued tokens (decode the JWT or
   call `GET /auth/user/` with the returned access token to confirm
   identity).
3. **Wrong code:** request a code, then verify with an intentionally
   wrong 6-digit string → 400, no tokens issued, `OTPCode.attempts`
   incremented.
4. **Expired code:** request a code, manually fast-forward
   `expires_at` into the past directly on the OTPCode row, then verify
   with the correct code → 400 (expired-specific message), no tokens.
5. **Max attempts exceeded:** submit the wrong code
   `OTP_MAX_VERIFICATION_ATTEMPTS` times, then submit the CORRECT code
   on the next attempt → still fails (proving the code is permanently
   dead after too many wrong guesses, not just rate-limited).
6. **Rate limiting on request:** hammer otp/request/ for the same phone
   number beyond the configured throttle rate → 429 responses once the
   limit is hit (this may overlap with Task 2.3.1.1's own tests — don't
   duplicate identical assertions, but DO include at least one
   end-to-end version here alongside the full flow for completeness).
7. **Cross-purpose isolation:** confirm a code generated for "login"
   purpose cannot be used to satisfy a verify call for a different
   purpose value if your verify endpoint ever exposes purpose as a
   parameter (if OTPVerifyView hardcodes `purpose="login"` always per
   Task 2.3.1.2's design, this test may be moot at the view level —
   in that case, skip it here and note that purpose-isolation is
   already covered at the service-layer tests from Task 2.1.2.3).

ACCEPTANCE CRITERIA
All 6–7 scenarios pass reliably. Run the full accounts test suite
(`pytest backend/accounts/`) at the end and confirm total pass count,
with zero regressions in the existing email/password auth tests.
```

---

### Feature 2.3.2 — Frontend OTP Auth UI

---

#### Task 2.3.2.1 — Build phone-entry screen

```
You are working in the frontend/src React app. Assume the backend OTP
endpoints from Feature 2.3.1 are already merged and reachable at
POST /api/auth/otp/request/ and POST /api/auth/otp/verify/.

CONTEXT
frontend/src/pages/Login.jsx currently handles only email/password
login, wired through frontend/src/context/AuthContext.jsx's `login()`
method, which calls `authAPI.login(credentials)` →
`api.post('/auth/login/', credentials)` (see frontend/src/services/api.js).
There is no phone-based entry point in the UI at all yet.

TASK
Add a phone-number-entry screen/step that kicks off the OTP flow,
either as a new tab/toggle on the existing Login.jsx page ("Continue
with Phone" vs "Continue with Email") or as a separate route — check
how Login.jsx is currently structured and pick whichever fits its
existing layout/component conventions with the least disruption; if
Login.jsx is small and simple, a toggle within the same page is
probably cleaner than a whole new route.

REQUIREMENTS
- Add a phone number input field with basic client-side format
  validation matching the backend's Iranian format expectations
  (09XXXXXXXXX pattern) — this is a UX nicety to catch obvious typos
  before hitting the API, NOT a security boundary (the backend remains
  the source of truth for validation).
- On submit, call a new `authAPI.requestOtp(phoneNumber)` method — this
  requires adding that method to frontend/src/services/api.js's
  `authAPI` object, matching the existing JSDoc-comment style used for
  every other method there:
  ```javascript
  /**
   * POST /auth/otp/request/
   * Body: { phone_number }
   */
  requestOtp: (phone_number) =>
    api.post('/auth/otp/request/', { phone_number }),
  ```
- On a successful (200) response, transition the UI to the code-entry
  step (built in Task 2.3.2.2) — for THIS task, a simple local
  component state toggle (`const [step, setStep] = useState('phone')`)
  is sufficient; don't over-engineer routing for a 2-step flow.
- Handle error responses gracefully: invalid phone format (400) shows
  an inline field error; 429 (throttled/cooldown) shows a
  "please wait before trying again" message — check the actual error
  response shape the backend returns (per Task 2.3.1.1's serializer
  errors) and match your error-display logic to it precisely, don't
  assume a shape.
- Match the existing visual style/component library used elsewhere in
  Login.jsx (form inputs, buttons, spacing) — don't introduce a new
  design pattern inconsistent with the rest of the page.

ACCEPTANCE CRITERIA
- Manually verify: entering a valid-format phone number and submitting
  triggers the network call and (assuming the dev backend's
  ConsoleSMSProvider logs the code to the Django server console) the
  UI transitions to a code-entry state.
- Entering an invalid phone number shows a client-side validation error
  without making a network call.
- This task does not need automated tests yet — Task 2.3.2.4 covers
  test coverage for the full OTP UI flow once both steps exist.
```

---

#### Task 2.3.2.2 — Build OTP-code entry screen with resend timer

```
You are working in the same area of frontend/src as Task 2.3.2.1.
Assume that task (phone-entry screen, transitions to a "code" step
locally) is already merged.

CONTEXT
After Task 2.3.2.1, the UI can request a code but has no way to submit
it. This task builds the second step: a 6-digit code input with a
resend cooldown that matches the backend's actual cooldown window
(Task 2.1.2.2's `OTP_RESEND_COOLDOWN_SECONDS`, default 60s — confirm
the exact value configured in backend/core/settings/base.py and use
the SAME number here so the frontend timer doesn't mislead the user
into thinking they can resend before the backend will actually accept
it).

TASK
Build the code-entry step: a 6-digit input, a countdown timer before
"Resend code" becomes clickable, and error states for wrong/expired
codes.

REQUIREMENTS
- 6 individual digit inputs (common OTP UX pattern) OR a single text
  input with `maxLength={6}` and `inputMode="numeric"` — pick whichever
  is simpler to implement well; a single input is significantly less
  code and equally usable, prefer it unless the existing design system
  already has a multi-box OTP component to reuse.
- On submit, call a new `authAPI.verifyOtp(phoneNumber, code)` method,
  added to api.js's `authAPI` object following the same pattern as
  Task 2.3.2.1:
  ```javascript
  /**
   * POST /auth/otp/verify/
   * Body: { phone_number, code }
   * Returns: { access, refresh, is_new_user }
   */
  verifyOtp: (phone_number, code) =>
    api.post('/auth/otp/verify/', { phone_number, code }),
  ```
- On success: this task builds the UI/network call only — actually
  storing tokens and updating auth state is Task 2.3.2.3's
  responsibility (AuthContext changes); for this task, just call the
  method and pass the response up via a prop/callback to whatever
  parent component will handle it next, rather than reaching into
  localStorage or AuthContext directly from this component (keep this
  component focused purely on the code-entry UX, not auth-state
  plumbing).
- Countdown timer: on mount (i.e. as soon as this step is shown), start
  a visible countdown (e.g. "Resend code in 0:58") counting down from
  the configured cooldown value; disable the "Resend" button/link until
  it reaches 0, then enable it and let the user trigger
  `authAPI.requestOtp()` again (reusing the function from Task
  2.3.2.1 — lift it to a shared location if it's currently scoped
  inside the phone-entry component only).
- Error handling: distinguish and display the backend's specific error
  messages for wrong code, expired code, and max-attempts-exceeded
  (per Task 2.3.1.2's distinct error responses) rather than one generic
  "something went wrong" message — the user should understand WHY it
  failed and what to do next (e.g. expired → prompt to resend; max
  attempts → prompt to resend a fresh code, since the old one is now
  permanently dead per the backend logic).

ACCEPTANCE CRITERIA
- Manually verify: submitting the correct code (read from the Django
  console log in dev) triggers a successful callback; submitting a
  wrong code shows an inline error and does not proceed; the resend
  button is disabled for the correct duration and becomes clickable
  after it elapses.
```

---

#### Task 2.3.2.3 — Update `AuthContext` to support OTP flow

```
You are working in frontend/src/context/AuthContext.jsx. Assume Tasks
2.3.2.1 and 2.3.2.2 are already merged (phone-entry and code-entry UI
components exist and can call the new authAPI methods, but don't yet
persist tokens or update global auth state on success).

CONTEXT
AuthContext currently exposes `login()`, `logout()`, `hydrateUser()`,
and `updateUser()`, all built around the email/password flow — e.g.
`login()` calls `authAPI.login(credentials)`, stores tokens via
`setTokens()`, then calls `authAPI.getUser()` to populate `user` state.
There is no equivalent method for completing the OTP flow yet, so
Task 2.3.2.2's code-entry component currently has nowhere authoritative
to hand off a successful verification response.

TASK
Add a `loginWithOtp` (or similarly named) method to AuthContext that
mirrors the existing `login()` method's shape and behavior, but is
driven by the already-completed `authAPI.verifyOtp()` call rather than
`authAPI.login()`.

REQUIREMENTS
- Add:
  ```javascript
  const loginWithOtp = useCallback(async (phoneNumber, code) => {
    const { data } = await authAPI.verifyOtp(phoneNumber, code);
    setTokens({ access: data.access, refresh: data.refresh });
    const { data: profile } = await authAPI.getUser();
    setUser(profile);
    return { profile, isNewUser: data.is_new_user };
  }, []);
  ```
  — deliberately mirroring `login()`'s exact structure (fetch tokens →
  store → hydrate user → return profile) for consistency, but also
  surfacing `is_new_user` from the verify response so the calling
  component (wherever Task 2.3.2.2's code-entry step is composed into
  the full page) can redirect a brand-new user to a
  "complete your profile" step if desired (per the gap noted in Task
  2.3.1.3 — empty first_name/last_name on OTP-created accounts).
- Add `loginWithOtp` to the context's exposed `value` object (alongside
  the existing `login`, `logout`, etc., in the `useMemo` block), and
  add it to the `useMemo` dependency array.
- Do NOT duplicate the `requestOtp`/`verifyOtp` API calls themselves
  inside AuthContext beyond what's needed here — the phone-entry
  component (Task 2.3.2.1) still calls `authAPI.requestOtp()` directly
  itself (requesting a code doesn't need to touch global auth state,
  only the final verify-success step does).
- Confirm `isAuthenticated`/`isAdmin` derived values in the existing
  `useMemo` continue to work correctly once `user` is populated via
  this new path — they should, since they're derived purely from
  `user` state regardless of which method populated it, but verify by
  reading through the logic once rather than assuming.

ACCEPTANCE CRITERIA / TESTS
Covered by Task 2.3.2.4's component tests — but manually verify first:
completing the phone-entry → code-entry flow in the browser results in
`useAuth().isAuthenticated` becoming `true` and the user being
redirected/logged-in exactly as the email/password flow already does.
```

---

#### Task 2.3.2.4 — Add OTP flow component tests

```
You are adding Vitest tests to the frontend. Assume Tasks 2.3.2.1
through 2.3.2.3 are all merged (full phone-entry → code-entry →
AuthContext flow works manually in the browser).

CONTEXT
backend/shop/tests/ and other backend apps have solid test coverage,
and the frontend already has Vitest configured with existing component
tests (check frontend/src for existing `*.test.jsx` files to find the
established testing patterns — mocking approach for API calls,
provider-wrapping conventions for components that consume
AuthContext, etc. — and match them exactly rather than introducing a
new testing style).

TASK
Write component/integration tests for the new OTP UI flow.

REQUIREMENTS — test cases to cover
1. Phone-entry component: entering a valid phone and submitting calls
   `authAPI.requestOtp` with the correctly normalized phone number
   (mock the api.js module — check how existing tests mock
   `frontend/src/services/api.js`, likely via `vi.mock`, and follow
   that exact pattern) and transitions to the code-entry step on a
   mocked 200 response.
2. Phone-entry component: entering an invalid phone shows a validation
   error and does NOT call the API.
3. Phone-entry component: a mocked 429 response from `requestOtp`
   displays the cooldown/throttle error message.
4. Code-entry component: submitting a correct code (mocked
   `verifyOtp` success response) triggers the expected success
   callback/navigation with the right data shape (`is_new_user`
   included).
5. Code-entry component: submitting a wrong code (mocked 400 response
   with the backend's specific error shape from Task 2.3.1.2) displays
   the correct inline error message text.
6. Code-entry component: the resend button is disabled immediately
   after the code-entry step mounts and becomes enabled after the
   countdown (use Vitest's fake timers — `vi.useFakeTimers()` +
   `vi.advanceTimersByTime()` — to avoid a real-time-based flaky test).
7. AuthContext: a test wrapping a consumer component in `<AuthProvider>`,
   calling `loginWithOtp()` with mocked `authAPI.verifyOtp` and
   `authAPI.getUser` responses, and asserting `isAuthenticated` becomes
   `true` and `user` matches the mocked profile data afterward (mirror
   however the existing `login()` method is already tested, if it is —
   check for an existing AuthContext test file first).

ACCEPTANCE CRITERIA
All new tests pass via `npm run test` (or whatever the project's
configured Vitest script is — check frontend/package.json), and running
the full existing frontend test suite alongside them shows zero
regressions.
```

---

## Phase 2.4 — Authorization Hardening

### Feature 2.4.1 — Role & Permission Cleanup

---

#### Task 2.4.1.1 — Replace magic numbers `(2,3)` with `UserType` enum in permission checks

```
You are working in backend/dashboard/permissions.py.

CONTEXT
The current file is:

    from rest_framework.permissions import BasePermission

    class IsAdminOrSuperuser(BasePermission):
        """
        Grants access only to users whose `type` is ADMIN (2) or SUPERUSER (3).
        Works with the custom UserType defined in accounts.models.
        """
        message = "Admin privileges are required to access this resource."

        def has_permission(self, request, view):
            return bool(
                request.user
                and request.user.is_authenticated
                and request.user.type in (2, 3)  # UserType.ADMIN, UserType.SUPERUSER
            )

    class IsOwnerOrAdmin(BasePermission):
        def has_object_permission(self, request, view, obj):
            if not request.user or not request.user.is_authenticated:
                return False
            if request.user.type in (2, 3):
                return True
            return getattr(obj, "user", None) == request.user

Both classes hardcode the literal integers `2` and `3` instead of using
the actual `UserType` enum already defined in backend/accounts/models.py
(`UserType.ADMIN = 2`, `UserType.SUPERUSER = 3`). This is fragile —
if the enum's values ever change, these checks silently break instead
of failing loudly, and it's harder to read/audit at a glance.

TASK
Replace the hardcoded tuples with references to the actual `UserType`
enum.

REQUIREMENTS
- Import `from accounts.models import UserType` at the top of
  permissions.py.
- Replace `request.user.type in (2, 3)` in BOTH classes with
  `request.user.type in (UserType.ADMIN, UserType.SUPERUSER)`.
- Update/remove the now-redundant inline comments
  (`# UserType.ADMIN, UserType.SUPERUSER`) since the code is now
  self-documenting.
- Double-check there's no circular import risk between `dashboard` and
  `accounts` apps (there shouldn't be — `accounts` doesn't import
  anything from `dashboard`), but run the server/tests after the change
  to confirm.
- Search the rest of the codebase for any OTHER place that might
  hardcode `type in (2, 3)` or similar magic-number role checks (e.g.
  `grep -rn "type ==" backend/ ; grep -rn "type in (" backend/`) and
  apply the same fix anywhere found, even outside dashboard/permissions.py
  — the goal is zero remaining magic-number role checks in the codebase,
  not just this one file.

ACCEPTANCE CRITERIA / TESTS
- Existing permission tests (if any — check for a
  backend/dashboard/tests/ directory) must continue to pass unchanged,
  since this is a pure refactor with zero behavior change.
- If no existing tests cover these permission classes, add minimal ones
  now: a CUSTOMER-type user is denied by `IsAdminOrSuperuser`, an
  ADMIN-type user is granted, a SUPERUSER-type user is granted.
```

---

#### Task 2.4.1.2 — Add object-level permission tests for `IsOwnerOrAdmin`

```
You are adding tests, likely to a new backend/dashboard/tests/
directory if one doesn't already exist (check first). Assume Task
2.4.1.1 is already merged.

CONTEXT
`IsOwnerOrAdmin.has_object_permission()` is used somewhere in the
dashboard app's views to gate access to individual objects (search
backend/dashboard/views.py for `permission_classes` referencing
`IsOwnerOrAdmin` to find real usage examples and confirm what kind of
objects it's actually applied to — this affects what "a plausible
object" looks like in your test setup). It currently has no dedicated
test coverage of its three distinct branches: unauthenticated denial,
admin-type override, and owner-match.

TASK
Write focused unit tests for `IsOwnerOrAdmin` covering all three
branches directly (instantiate the permission class and call
`has_object_permission()` directly with constructed request/view/obj
arguments — this is faster and more precise than testing it indirectly
through a full view, though you may ALSO want one integration-level
test through a real view if that's more representative of how it's
actually exercised in production; check existing test conventions in
this repo for which style is preferred, e.g. by looking at how
`shop/tests/test_views.py` tests permission-gated endpoints).

REQUIREMENTS — test cases
1. An unauthenticated request (`request.user` is `AnonymousUser` or
   `None` — check which DRF actually gives you in a test request
   factory) is denied regardless of the object.
2. An authenticated CUSTOMER-type user is denied access to an object
   whose `.user` attribute does NOT match `request.user`.
3. An authenticated CUSTOMER-type user IS granted access to an object
   whose `.user` attribute DOES match `request.user`.
4. An authenticated ADMIN-type user is granted access to an object
   regardless of ownership (the override branch).
5. An authenticated SUPERUSER-type user is granted access regardless of
   ownership.
6. An object with NO `.user` attribute at all doesn't crash the
   permission check (confirm the `getattr(obj, "user", None)` fallback
   in the existing code handles this gracefully and returns `False` for
   a non-admin, non-owner-determinable object, since `getattr(...) == request.user`
   would compare `None == request.user`, which is `False` unless
   `request.user` is somehow also `None` — but that's already excluded
   by the earlier unauthenticated check, so this should be safe;
   confirm with an actual test rather than just reading the code).

ACCEPTANCE CRITERIA
All 6 cases pass. Use Django REST Framework's
`rest_framework.test.APIRequestFactory` to construct lightweight fake
requests rather than spinning up a full view/URL round-trip, since this
is meant to be a fast, isolated unit test of the permission logic
itself.
```

---

#### Task 2.4.1.3 — Add DRF throttle classes globally (anon/user scopes)

```
You are working in backend/core/settings/base.py. Assume Task 2.1.2.4
(the OTP-specific `PhoneOTPRequestThrottle` and its scope/rate entry)
is already merged — this task adds the GENERAL, project-wide throttle
configuration that was missing entirely before that task added just
the one OTP-specific scope.

CONTEXT
Checked directly: `REST_FRAMEWORK` in base.py currently has no
`DEFAULT_THROTTLE_CLASSES` key at all (Task 2.1.2.4 may have added one
containing just the OTP scope's rate, but not general anon/user
throttling) — meaning every other endpoint in the entire API (product
listing, cart, checkout, login, everything) has ZERO rate limiting.
This is a real gap: someone can script unlimited requests against
login, search, or any read endpoint with no backend protection at all.

TASK
Add DRF's built-in `AnonRateThrottle` and `UserRateThrottle` as global
defaults, with sane starting rates, without breaking the OTP-specific
throttle scope from Task 2.1.2.4.

REQUIREMENTS
- In `REST_FRAMEWORK` in base.py, add (or extend, if
  `DEFAULT_THROTTLE_CLASSES` already exists from Task 2.1.2.4):
  ```python
  "DEFAULT_THROTTLE_CLASSES": [
      "rest_framework.throttling.AnonRateThrottle",
      "rest_framework.throttling.UserRateThrottle",
  ],
  "DEFAULT_THROTTLE_RATES": {
      "anon": "100/min",
      "user": "300/min",
      "otp_request": "3/10min",  # preserve this if already added by Task 2.1.2.4
  },
  ```
  — pick specific rate numbers thoughtfully: 100/min anonymous and
  300/min authenticated are reasonable generic starting points for an
  e-commerce storefront (generous enough not to break normal browsing/
  pagination/search-as-you-type usage, tight enough to blunt basic
  scripted abuse), but note in a settings comment that these are
  initial values that should be tuned based on real traffic patterns
  once the site has actual usage data — don't treat them as
  permanently correct.
- Confirm this doesn't conflict with or override the custom
  `PhoneOTPRequestThrottle` class from Task 2.1.2.4, which is applied
  per-view via `throttle_classes = [PhoneOTPRequestThrottle]` on
  `OTPRequestView` specifically — DRF applies BOTH the view-specific
  throttle_classes (if set) AND does NOT also apply the global default
  throttle classes on top when a view explicitly sets its own
  `throttle_classes` (check DRF's actual behavior here — by default,
  setting `throttle_classes` on a view OVERRIDES the global
  `DEFAULT_THROTTLE_CLASSES` for that view entirely, it doesn't add to
  them) — decide whether `OTPRequestView` should ALSO get general
  anon-rate protection in addition to its specific phone-based
  cooldown, and if so, explicitly list both throttle classes on that
  view: `throttle_classes = [PhoneOTPRequestThrottle, AnonRateThrottle]`.
  Make this decision deliberately and document it with a comment on the
  view.
- Verify `production.py`/`development.py` don't already define
  conflicting `REST_FRAMEWORK` overrides that would shadow this base.py
  change (check both files for a `REST_FRAMEWORK` key before assuming
  base.py's value is what's actually active).

ACCEPTANCE CRITERIA / TESTS
Add a test hitting some ordinary, previously-unthrottled endpoint (e.g.
the product list endpoint) more than 100 times in a tight loop as an
anonymous client within a test, and confirm a 429 response eventually
appears (this test may be slow/awkward at 100 real requests — consider
temporarily overriding `DEFAULT_THROTTLE_RATES` to a much lower test
value like `"3/min"` via Django's `@override_settings` decorator
specifically for this test, so you can prove the mechanism works
without actually making 100+ requests in a unit test).
```

---

#### Task 2.4.1.4 — Add stricter throttle scope for auth endpoints

```
You are working in backend/accounts/views.py and
backend/core/settings/base.py. Assume Task 2.4.1.3 (global anon/user
throttle defaults) is already merged.

CONTEXT
The new global defaults (100/min anon, 300/min authenticated) are
reasonable for general browsing, but login, register, and
password-reset-request are meaningfully higher-value targets for
scripted abuse (credential stuffing against LoginView, mass fake
account creation against RegisterView, email-enumeration/spam against
PasswordResetRequestView) and deserve a materially tighter rate than
general API browsing gets.

TASK
Add a dedicated, stricter throttle scope for these sensitive auth
endpoints, distinct from both the general anon/user rates and the
already-separate OTP-specific scope from Task 2.1.2.4.

REQUIREMENTS
- Add a new scope to `DEFAULT_THROTTLE_RATES` in base.py, e.g.
  `"auth_sensitive": "10/min"` (10 requests per minute per anonymous
  client/IP — tune as you see fit, but it should be noticeably tighter
  than the 100/min general anon rate).
- Create a small custom throttle class in
  backend/accounts/throttles.py (the file created in Task 2.1.2.4 for
  `PhoneOTPRequestThrottle` — add to it rather than creating a new
  file):
  ```python
  class AuthSensitiveRateThrottle(AnonRateThrottle):
      scope = "auth_sensitive"
  ```
- Apply `throttle_classes = [AuthSensitiveRateThrottle]` to:
  `LoginView`, `RegisterView`, `PasswordResetRequestView`, and
  `PasswordResetConfirmView` in backend/accounts/views.py (do NOT apply
  it to `CurrentUserView`, `ProfileView`, or `ChangePasswordView` —
  those require authentication already and are lower-risk / already
  covered by the general `user` rate scope from Task 2.4.1.3, plus
  applying this scope which extends `AnonRateThrottle` wouldn't even
  throttle authenticated users correctly).
- Double check: does `LoginView(TokenObtainPairView)` inherit cleanly
  in a way that lets you just set `throttle_classes` on it directly, or
  does `TokenObtainPairView` from `rest_framework_simplejwt` do
  anything unusual with throttling that needs extra care? (It shouldn't
  — DRF's `APIView.throttle_classes` mechanism is inherited normally —
  but verify by testing rather than assuming, since SimpleJWT views
  sometimes override more of the base APIView machinery than expected.)

ACCEPTANCE CRITERIA / TESTS
Add a test hitting `POST /api/auth/login/` (with intentionally wrong
credentials, so it doesn't matter that it always fails) more times than
the `auth_sensitive` rate allows within the test, and confirm a 429
appears once the limit is hit (same `@override_settings` pattern as
Task 2.4.1.3's test if the real 10/min rate is inconvenient to fully
exhaust in a fast unit test). Repeat for `POST /api/auth/register/`
with a fresh dummy payload each time (to avoid hitting the "email
already exists" validation error instead of actually reaching the
throttle check — vary the email per request). Confirm `CurrentUserView`
and `ProfileView` are NOT affected by this new stricter scope (a normal
authenticated user making a reasonable number of profile-related
requests should not suddenly start getting 429s from a scope that was
never meant to apply to them).
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 2.1.1.1 | Add phone_number to User model | ☐ |
| 2.1.1.2 | Iranian phone validator + normalizer | ☐ |
| 2.1.1.3 | Backfill phone from Profile | ☐ |
| 2.1.2.1 | Create OTPCode model | ☐ |
| 2.1.2.2 | OTP generation service | ☐ |
| 2.1.2.3 | OTP verification service | ☐ |
| 2.1.2.4 | Per-phone OTP request throttle | ☐ |
| 2.2.1.1 | Define SMSProvider interface | ☐ |
| 2.2.1.2 | Console/dev SMS backend | ☐ |
| 2.2.1.3 | Wire OTP service to SMS provider | ☐ |
| 2.3.1.1 | POST /auth/otp/request/ endpoint | ☐ |
| 2.3.1.2 | POST /auth/otp/verify/ endpoint | ☐ |
| 2.3.1.3 | Auto-create Profile on OTP registration | ☐ |
| 2.3.1.4 | OTP flow integration tests | ☐ |
| 2.3.2.1 | Phone-entry screen (frontend) | ☐ |
| 2.3.2.2 | OTP code-entry screen + resend timer | ☐ |
| 2.3.2.3 | AuthContext OTP support | ☐ |
| 2.3.2.4 | OTP flow component tests (frontend) | ☐ |
| 2.4.1.1 | Replace magic numbers with UserType enum | ☐ |
| 2.4.1.2 | IsOwnerOrAdmin permission tests | ☐ |
| 2.4.1.3 | Global anon/user throttle classes | ☐ |
| 2.4.1.4 | Stricter throttle scope for auth endpoints | ☐ |

Once Epic 2 is fully merged, the next epic to generate prompts for is
**Epic 3 — Product Catalog & Cosmetics Data Model**, which is the
biggest schema-change epic in the backlog (product variants, SKUs, and
all cosmetics-specific attribute fields) and several later epics
(Cart/Checkout, Inventory, Search) depend directly on it.
