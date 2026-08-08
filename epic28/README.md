# Epic 28 — Marketing Tools — AI Coding Prompts

Repo: `tablogenix-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–27 are fully merged, including Epic 16's unified `notify()` system, Epic 9's `Coupon` model, and Epic 22's Celery infrastructure.

**Confirmed directly from the repo:** `accounts.Profile` already has `newsletter`/`promotions`/`order_updates` boolean flags (established since before this document series, extended by Epic 18's audit) — but these are attached to `Profile`, meaning they only exist for someone who **already has an account**. There is currently no capture mechanism at all for a visitor who wants to subscribe to marketing emails WITHOUT creating a full account — a common, lower-friction entry point real e-commerce sites offer (a footer "subscribe" box), which Task 28.1.1.1 below adds. `RegisterView` (`backend/accounts/views.py`) is a plain `APIView` accepting `{email, first_name, last_name, password}` with no referral/promo-code concept anywhere.

---

## Phase 28.1 — Growth Features

### Feature 28.1.1 — Campaign Tools

---

#### Task 28.1.1.1 — Newsletter signup capture (reuse `Profile.newsletter` flag)

```
You are working in backend/accounts/models.py, serializers.py,
views.py, urls.py, and frontend/src/components/Footer.jsx (or wherever
a newsletter signup box makes sense — check for an existing, empty
placeholder in the footer from earlier epics' grounding, since a
"subscribe" box is a common baseline footer element that may already
have SOME UI shell even without working backend logic). Assume Epics
1–27 are fully merged.

CONTEXT — READ THIS DOCUMENT'S HEADER FIRST
`Profile.newsletter` already exists and already correctly feeds Epic
16's `notify()` channel-selection logic for AUTHENTICATED users — this
task does NOT touch that existing mechanism at all. This task's real
job: capture newsletter interest from a visitor who has NO account at
all, and correctly RECONCILE that captured interest if/when they later
DO create an account (so a guest who subscribed doesn't end up as a
disconnected, orphaned record with no relationship to their eventual
real account).

TASK
Add a standalone `NewsletterSubscriber` model and capture endpoint for
non-account visitors, with reconciliation logic linking a captured
subscription to a real account if one is later created with the same
email.

REQUIREMENTS
- Add to backend/accounts/models.py:
  ```python
  class NewsletterSubscriber(models.Model):
      email = models.EmailField(unique=True)
      subscribed_at = models.DateTimeField(auto_now_add=True)
      unsubscribed_at = models.DateTimeField(null=True, blank=True)
      user = models.OneToOneField(
          settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True, blank=True,
          related_name="newsletter_subscription",
          help_text="Linked automatically if/when this email later registers a real account.",
      )

      class Meta:
          ordering = ["-subscribed_at"]

      @property
      def is_active(self):
          return self.unsubscribed_at is None

      def __str__(self):
          return f"{self.email} ({'active' if self.is_active else 'unsubscribed'})"
  ```
  Generate the migration.
- Add `POST /api/auth/newsletter/subscribe/` and
  `POST /api/auth/newsletter/unsubscribe/`:
  ```python
  class NewsletterSubscribeSerializer(serializers.Serializer):
      email = serializers.EmailField()


  class NewsletterSubscribeView(APIView):
      permission_classes = [AllowAny]

      def post(self, request):
          serializer = NewsletterSubscribeSerializer(data=request.data)
          serializer.is_valid(raise_exception=True)
          email = serializer.validated_data["email"].lower().strip()

          # If this email ALREADY belongs to a real account, update that
          # account's Profile.newsletter flag directly instead of
          # creating a separate, disconnected NewsletterSubscriber row
          # — a logged-out visitor entering the SAME email they already
          # have an account under should transparently just toggle
          # their existing account's preference, not create a second,
          # parallel tracking record.
          existing_user = User.objects.filter(email=email).first()
          if existing_user:
              existing_user.profile.newsletter = True
              existing_user.profile.save(update_fields=["newsletter"])
              return Response({"detail": "Subscribed."}, status=status.HTTP_200_OK)

          subscriber, created = NewsletterSubscriber.objects.get_or_create(
              email=email, defaults={"user": None}
          )
          if not created and subscriber.unsubscribed_at:
              subscriber.unsubscribed_at = None
              subscriber.save(update_fields=["unsubscribed_at"])
          return Response({"detail": "Subscribed."}, status=status.HTTP_200_OK)
  ```
  Import `User` from the same app. Note the response is DELIBERATELY
  the same generic "Subscribed." message regardless of whether this
  was a brand-new subscription, a re-subscription, or an existing-
  account preference toggle — mirroring the exact "don't reveal whether
  an email is already registered" user-enumeration-avoidance principle
  already established for password-reset (`accounts/views.py`'s
  existing `PasswordResetRequestView`) and Epic 2 Task 2.3.1.1's OTP
  request endpoint — a newsletter-subscribe endpoint that responds
  DIFFERENTLY depending on whether the email already has an account
  would leak that same account-existence information through a new,
  unprotected side channel, undermining protection already carefully
  built elsewhere.
  Add the equivalent `NewsletterUnsubscribeView` (setting
  `unsubscribed_at`, or `profile.newsletter = False` for an
  account-linked email).
  Register both URLs in backend/accounts/urls.py, alongside the
  existing auth-adjacent endpoints.
  Apply `AuthSensitiveRateThrottle` (Epic 2 Task 2.4.1.4, already
  applied to `RegisterView`/`PasswordResetRequestView`) to
  `NewsletterSubscribeView` too — an unauthenticated, email-accepting
  endpoint is exactly the kind of surface that needs the same rate-
  limiting discipline already applied consistently elsewhere in this
  file.
- Add RECONCILIATION: when a NEW account is registered (via
  `RegisterView`, per this document's header's confirmed structure),
  check whether a `NewsletterSubscriber` record already exists for
  that email and, if so, link it and correctly carry the subscription
  preference forward onto the new account's `Profile.newsletter`:
  ```python
  # accounts/views.py, RegisterView.post(), after user = serializer.save()
  existing_subscriber = NewsletterSubscriber.objects.filter(
      email=user.email, unsubscribed_at__isnull=True
  ).first()
  if existing_subscriber:
      existing_subscriber.user = user
      existing_subscriber.save(update_fields=["user"])
      user.profile.newsletter = True
      user.profile.save(update_fields=["newsletter"])
  ```
- Add a footer signup component: an email input + submit button,
  calling `NewsletterSubscribeView`, with clear success/error feedback
  (Persian, per Epic 14's conventions) — check for any existing
  placeholder footer UI first and complete/wire it rather than
  necessarily building from a blank slate.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/accounts/tests/:
1. A brand-new email (no existing account) creates a
   `NewsletterSubscriber` row.
2. An email belonging to an EXISTING account instead updates that
   account's `Profile.newsletter=True` directly, WITHOUT creating a
   separate `NewsletterSubscriber` row.
3. Both cases return the SAME generic success response (the user-
   enumeration-avoidance regression test).
4. Unsubscribing correctly sets `unsubscribed_at` (standalone) or
   `Profile.newsletter=False` (account-linked) as appropriate.
5. Registering a NEW account whose email matches an existing, active
   `NewsletterSubscriber` record correctly links it and sets
   `Profile.newsletter=True` on the new account.
6. Re-subscribing a previously-unsubscribed standalone email correctly
   clears `unsubscribed_at` rather than creating a duplicate row
   (the `unique=True` constraint on `email` would prevent a duplicate
   anyway, but confirm the RE-subscribe path is handled gracefully,
   not as an unhandled `IntegrityError`).
```

---

#### Task 28.1.1.2 — Abandoned-cart recovery email/SMS (Celery scheduled)

```
You are working in backend/cart/tasks.py (new file),
backend/notifications/models.py (extending
`NotificationTemplate.Event`), and core/celery.py's seeded periodic-
task migration (Epic 22 Task 22.1.1.2's pattern). Assume Epics 1–27
are fully merged, including Epic 5's guest+authenticated cart model
and Epic 16's `notify()` system.

CONTEXT
Nothing currently detects or acts on an abandoned cart — a cart with
real items sitting idle for hours/days with no checkout completion is
a well-established, high-value recovery opportunity for any real
e-commerce business, and this platform has zero mechanism for it.

TASK
Add a Celery task detecting carts idle beyond a threshold with items
still present, sending a recovery notification via Epic 16's `notify()`
system, with careful de-duplication so the SAME cart doesn't get
repeatedly re-notified.

REQUIREMENTS
- Add `ABANDONED_CART = "abandoned_cart", "Abandoned Cart Recovery"`
  to `NotificationTemplate.Event` choices (Epic 16 Task 16.1.1.2),
  generate the migration, and seed a real Persian template for it
  (matching that task's established seeding-migration pattern and
  translation-quality bar).
- Add tracking fields to `Cart` (backend/cart/models.py) for de-
  duplication:
  ```python
  abandonment_notified_at = models.DateTimeField(null=True, blank=True)
  ```
  Generate the migration.
- Implement in backend/cart/tasks.py:
  ```python
  from celery import shared_task
  from django.conf import settings
  from django.utils import timezone
  from datetime import timedelta


  @shared_task
  def send_abandoned_cart_recovery_notifications():
      from notifications.services import notify
      from notifications.models import NotificationTemplate
      from .models import Cart

      threshold = timezone.now() - timedelta(hours=settings.ABANDONED_CART_THRESHOLD_HOURS)

      candidates = (
          Cart.objects.filter(
              items__isnull=False,  # has at least one item
              updated_at__lt=threshold,
              abandonment_notified_at__isnull=True,  # never notified before — see re-notification note below
          )
          .distinct()
          .select_related("user", "user__profile")
      )

      sent_count = 0
      for cart in candidates:
          if cart.user is None:
              continue  # per Epic 16's established pattern, only authenticated users have a reachable email/phone on file for this kind of proactive marketing notification — a guest session has no durable contact info to recover them through
          if not getattr(cart.user.profile, "promotions", False):
              continue  # respect the SAME marketing-preference flag Epic 16's notify() already checks for BACK_IN_STOCK/PRICE_DROP — abandoned-cart recovery is squarely a marketing/promotional notification, not a transactional order-update one, and must be gated the same way
          notify(
              cart.user, NotificationTemplate.Event.ABANDONED_CART,
              {
                  "first_name": cart.user.profile.first_name,
                  "item_count": cart.items.count(),
                  "cart_url": f"{settings.FRONTEND_URL}/cart/",
              },
          )
          cart.abandonment_notified_at = timezone.now()
          cart.save(update_fields=["abandonment_notified_at"])
          sent_count += 1

      return f"Sent {sent_count} abandoned-cart recovery notification(s)."
  ```
  Add `ABANDONED_CART_THRESHOLD_HOURS = config("ABANDONED_CART_THRESHOLD_HOURS", default=24, cast=int)`
  to settings.
  Note the explicit `if cart.user is None: continue` — this is a
  DELIBERATE, documented scope decision, not an oversight: per Epic
  5's guest-cart design and Epic 9's earlier-established "guest coupon
  redemption isn't reliably trackable" reasoning, a session-only cart
  has no durable, reachable contact info once that browser
  session/cookie eventually expires, making proactive recovery
  outreach to a guest fundamentally different (and less reliable) than
  to an authenticated user with a real email/phone on file — scope
  this task to authenticated-user carts only, matching the same
  authenticated-only pattern already established for coupons (Epic 9)
  and price-drop alerts (Epic 11) for the identical underlying reason.
  Note the `Profile.promotions` gate reuses Epic 16's EXISTING
  marketing-preference flag rather than inventing a new, separate
  "abandoned cart" opt-in — this is the correct categorization (this
  IS a promotional/marketing message, not a transactional order
  update) and keeps this project's notification-preference model
  simple and consistent rather than fragmenting into ever-more-granular
  per-feature opt-in flags without strong evidence that granularity is
  actually needed.
- RE-NOTIFICATION policy: the query above only notifies a cart ONCE
  EVER (`abandonment_notified_at__isnull=True`), which means a cart
  that gets a NEW item added after already being notified once won't
  be re-notified even if it becomes newly "abandoned" again later —
  decide whether this is the right policy, or whether
  `abandonment_notified_at` should be CLEARED whenever a NEW item is
  added to an already-notified cart (so a genuinely NEW abandonment
  event, distinct from the earlier one, can trigger a fresh
  notification) — RECOMMEND clearing it on new item-add specifically:
  wire this into `cart/views.py`'s add-to-cart handler (the same
  Epic 5-established `CartView.post()`), resetting
  `cart.abandonment_notified_at = None` whenever a NEW item is added
  (not merely an existing item's quantity updated) — since a
  meaningfully different cart (new product interest) genuinely
  warrants a fresh recovery opportunity, distinct from re-nagging about
  the exact same abandonment that was already messaged about once.
- Register as a periodic task (once daily is reasonable — an
  abandoned-cart check doesn't need high frequency, and running it more
  than once a day risks the notification feeling naggy/frequent to
  recipients) via the SAME `django_celery_beat` seeding-migration
  pattern established in Epic 22 Task 22.1.1.2.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/cart/tests/:
1. A cart idle past the threshold, belonging to an authenticated user
   with `promotions=True`, WITH items, and never previously notified,
   correctly triggers `notify()` and sets `abandonment_notified_at`.
2. A cart idle past the threshold but belonging to a GUEST session
   (`user=None`) is correctly SKIPPED.
3. A cart idle past the threshold belonging to a user with
   `promotions=False` is correctly SKIPPED.
4. A cart NOT yet past the threshold is correctly excluded.
5. A cart ALREADY notified once (`abandonment_notified_at` set) is
   correctly excluded from a SECOND run of the task, UNLESS a new item
   was added since (per the re-notification policy — construct this
   exact scenario: notify once, add a new item, confirm
   `abandonment_notified_at` was cleared by the add-to-cart handler,
   confirm the NEXT run of the task correctly re-notifies).
6. An EMPTY cart (no items) is never notified regardless of age.
```

---

#### Task 28.1.1.3 — Referral/promo-code-on-signup flow

```
You are working in backend/accounts/models.py, serializers.py,
views.py, and backend/promotions/models.py. Assume Epics 1–27 are
fully merged, including Epic 9's `Coupon`/`CouponRedemption` models.

CONTEXT
No referral or automatic new-user-coupon mechanism exists — `RegisterView`
(per this document's header) accepts a plain
`{email, first_name, last_name, password}` payload with no code/
referral concept whatsoever.

TASK
Add two related but distinct capabilities: (1) automatic issuance of a
welcome coupon to every new signup, and (2) an OPTIONAL referral-code
field at signup crediting BOTH the new user and whoever referred them.

REQUIREMENTS — Part 1: automatic welcome coupon
- Add a settings-driven welcome-coupon TEMPLATE (an admin-configured
  `Coupon` marked as the designated "welcome" coupon, rather than
  hardcoding discount parameters directly in Python — consistent with
  this project's established pattern of admin-configurable operational
  parameters, per Epic 6/7's gateway/carrier config):
  ```python
  # promotions/models.py — extend Coupon
  is_welcome_coupon = models.BooleanField(
      default=False,
      help_text="If true, this coupon is automatically issued to every new user at signup. Only one coupon should have this flag set at a time.",
  )
  ```
  Add a `clean()` validation ensuring at most ONE `Coupon` has
  `is_welcome_coupon=True` at any time (an admin accidentally flagging
  two different coupons as "the" welcome coupon would create ambiguous
  behavior):
  ```python
  def clean(self):
      ...  # existing clean() logic
      if self.is_welcome_coupon:
          conflicting = Coupon.objects.filter(is_welcome_coupon=True).exclude(pk=self.pk)
          if conflicting.exists():
              raise ValidationError({"is_welcome_coupon": "Another coupon is already set as the welcome coupon."})
  ```
  In `RegisterView.post()` (accounts/views.py), after successful user
  creation:
  ```python
  from promotions.models import Coupon, CouponRedemption

  welcome_coupon = Coupon.objects.filter(is_welcome_coupon=True, is_active=True).first()
  if welcome_coupon:
      # Note: this does NOT create a CouponRedemption yet — per Epic 9's
      # established model, a redemption represents actual USE of a
      # coupon (validated + applied to a real order), not mere
      # eligibility to use one. Issuing a welcome coupon at signup just
      # means the code becomes VALID FOR this specific user to redeem
      # later at checkout, going through the exact same
      # validate_coupon()/apply-to-cart/checkout flow as any other
      # coupon — no special-cased bypass of that existing, well-tested
      # validation pipeline.
      pass  # the coupon is usable by anyone with the code by default UNLESS restricted — see the per-user restriction note below
  ```
  Reconsider: a single, shared "welcome" coupon CODE, usable by
  literally anyone who discovers it (not just genuinely new users), is
  a real abuse vector — an EXISTING customer could just apply the
  "welcome" code too, or the code could leak/spread beyond genuinely
  new signups entirely. The cleaner mechanism: generate a UNIQUE,
  PER-USER coupon at signup time (cloning the welcome template's
  discount parameters but with a unique, user-specific code and
  `uses_per_user=1`/tightly scoped `max_uses=1`), not simply granting
  access to one shared code:
  ```python
  import secrets

  def issue_welcome_coupon(user):
      template = Coupon.objects.filter(is_welcome_coupon=True, is_active=True).first()
      if not template:
          return None
      unique_code = f"WELCOME-{secrets.token_hex(4).upper()}"
      return Coupon.objects.create(
          code=unique_code,
          discount_type=template.discount_type,
          value=template.value,
          min_order_amount=template.min_order_amount,
          max_uses=1,
          uses_per_user=1,
          valid_from=timezone.now(),
          valid_until=timezone.now() + timedelta(days=30),
          is_active=True,
      )
  ```
  Call `issue_welcome_coupon(user)` from `RegisterView.post()`, and
  include the generated code in the registration RESPONSE payload
  (`{"welcome_coupon_code": coupon.code if coupon else None, ...}`) so
  the frontend can immediately show it to the new user, and ALSO send
  it via the welcome email (Epic 16 Task 16.3.1.1's template,
  extending it with a coupon-code placeholder).

REQUIREMENTS — Part 2: referral code
- Add to `User` (or a lighter-weight dedicated `Referral` model —
  RECOMMEND a dedicated model, since a referral relationship has its
  own lifecycle/metadata distinct from just a flat field on `User`):
  ```python
  # accounts/models.py
  class Referral(models.Model):
      referrer = models.ForeignKey(
          settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="referrals_made"
      )
      referred_user = models.OneToOneField(
          settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="referred_by"
      )
      created_at = models.DateTimeField(auto_now_add=True)
      referrer_reward_issued = models.BooleanField(default=False)

      def __str__(self):
          return f"{self.referrer.email} referred {self.referred_user.email}"
  ```
  Add a `referral_code` property/field on `Profile` — the SIMPLEST,
  most robust referral-code SCHEME is each user's OWN unique
  identifier being their referral code directly (avoiding a separate
  code-generation/uniqueness-management system entirely):
  ```python
  # Profile
  @property
  def referral_code(self):
      return f"REF{self.user.id:06d}"  # deterministic, always available, no extra storage/generation needed
  ```
- Update `RegisterSerializer`/`RegisterView` to accept an OPTIONAL
  `referral_code` field:
  ```python
  referral_code = serializers.CharField(required=False, allow_blank=True)

  def validate_referral_code(self, value):
      if not value:
          return value
      match = re.match(r"^REF(\d{6})$", value)
      if not match:
          raise serializers.ValidationError("Invalid referral code format.")
      referrer_id = int(match.group(1))
      if not User.objects.filter(id=referrer_id).exists():
          raise serializers.ValidationError("Referral code not found.")
      return value
  ```
  In `RegisterView.post()`, after user creation, if a valid
  `referral_code` was provided: create the `Referral` record, and
  issue a SEPARATE, per-referrer-and-referee reward coupon to BOTH
  parties (reusing the SAME `issue_welcome_coupon()`-style per-user
  unique-coupon-generation approach from Part 1, but driven by a
  DIFFERENT admin-configured "referral reward" coupon template rather
  than conflating it with the plain welcome coupon — add a second
  `Coupon.is_referral_reward_coupon` boolean, mirroring
  `is_welcome_coupon`'s exact pattern, since referral rewards and
  generic welcome coupons are legitimately different campaigns an
  admin may want to configure with different values):
  ```python
  referral = Referral.objects.filter(referred_user=user).first()
  if referral and not referral.referrer_reward_issued:
      referrer_coupon = issue_referral_reward_coupon(referral.referrer)
      if referrer_coupon:
          from notifications.services import notify
          from notifications.models import NotificationTemplate
          notify(referral.referrer, NotificationTemplate.Event.REFERRAL_REWARD, {
              "coupon_code": referrer_coupon.code,
              "referred_user_name": user.profile.first_name,
          })
          referral.referrer_reward_issued = True
          referral.save(update_fields=["referrer_reward_issued"])
  ```
  (add the `REFERRAL_REWARD` event to `NotificationTemplate.Event`,
  seeded with a real Persian template, matching Task 28.1.1.2's exact
  established pattern for adding new notification events).
  A new user's OWN referral reward (as the person being referred, not
  the referrer) can simply be their normal welcome coupon from Part 1
  — don't build a THIRD, separate coupon-issuance path for "reward for
  being referred" unless there's a genuine reason the referred-user's
  reward should differ from the standard welcome coupon; keep this
  simple unless a real requirement demands otherwise.
- Add a "Referral" section to the customer account area
  (`frontend/src/components/`, alongside Epic 18's other account tabs)
  showing the user's own `referral_code`/shareable referral link
  (`{FRONTEND_URL}/register?ref={referral_code}`), and how many
  successful referrals they've made (`user.referrals_made.count()`).
  Update the registration PAGE to pre-fill the `referral_code` field
  from a `?ref=` URL query parameter if present, so a shared referral
  link works seamlessly without the new user needing to manually type
  a code.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/accounts/tests/:
1. Registering a new user, with an active `is_welcome_coupon=True`
   template configured, issues a unique, single-use coupon correctly
   cloning the template's discount parameters, and the code is
   returned in the registration response.
2. Registering with NO welcome-coupon template configured
   (`is_welcome_coupon` unset on any coupon) succeeds normally with
   `welcome_coupon_code: null`, not an error.
3. `Coupon.clean()` rejects a second coupon being flagged
   `is_welcome_coupon=True` while one already exists with that flag.
4. Registering with a VALID `referral_code` correctly creates a
   `Referral` record, issues a reward coupon to the REFERRER (not the
   new user), and triggers the `REFERRAL_REWARD` notification to the
   referrer.
5. Registering with an INVALID/malformed `referral_code` is rejected
   with a clear validation error, and does NOT create a partial/
   corrupted `Referral` record.
6. Registering with a referral code referencing a NON-EXISTENT user ID
   is rejected.
7. A referrer only receives ONE reward per referred user, even if
   somehow re-processed (the `referrer_reward_issued` guard).
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 28.1.1.1 | Newsletter signup capture (guest + account reconciliation) | ☐ |
| 28.1.1.2 | Abandoned-cart recovery email/SMS | ☐ |
| 28.1.1.3 | Referral/promo-code-on-signup flow | ☐ |

Once Epic 28 is fully merged, the next epic to generate prompts for is
**Epic 29 — Frontend Architecture Refactor**, which the master
backlog's own execution-order notes flag as work that should ideally
have been done INCREMENTALLY throughout the project (code-splitting,
data-fetching layer) rather than saved for the end — worth reading
that epic's own header framing carefully before assuming it's a single
clean greenfield epic, given how many of THIS document series' "already
built" discoveries have followed exactly that same pattern.
