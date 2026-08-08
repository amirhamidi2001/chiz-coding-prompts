# Epic 16 — Notifications (SMS / Email / Push) — AI Coding Prompts

Repo: `tablogenix-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–15 are fully merged. This is the epic that finally implements real dispatch behind every notification-shaped placeholder deliberately left across the backlog so far:
- Epic 2 Task 2.2.1.1's `SMSProvider` ABC + `ConsoleSMSProvider` dev backend (OTP delivery) — this epic adds the real `KavenegarSMSProvider`.
- Epic 4 Task 4.1.2.3's `notify_stock_alert_subscribers` Celery task, explicitly left with a `# TODO: Epic 16 —` comment instead of a real send.
- Epic 8 Task 8.1.1.2's `order_status_changed` signal, explicitly built with only a `logger.info(...)` placeholder receiver and a `# TODO: Epic 16 —` comment.
- Epic 11 Task 11.1.1.2's `notify_price_drop_subscribers` Celery task, same placeholder pattern as Epic 4's.

**Important grounded discoveries — read before starting:**
1. `backend/accounts/views.py` **already sends real transactional email** — `_send_welcome_email()` and `_send_password_reset_email()` both call Django's `send_mail()` **synchronously, inline in the request/response cycle**, with `fail_silently=True`, English-only subject lines, and HTML templates at `accounts/emails/welcome.html` / `accounts/emails/password_reset.html`. `_send_password_reset_email()` also resolves `FRONTEND_URL` via an unusual `getattr(__import__("django.conf", fromlist=["settings"]).settings, "FRONTEND_URL", "http://localhost:3000")` construct instead of a normal `from django.conf import settings` import — a real, existing code smell worth cleaning up as part of this epic's email work, not a hypothetical.
2. `backend/chat/` already has a **fully working, JWT-authenticated Django Channels setup**: `chat/consumers.py` (`AsyncWebsocketConsumer`, group-based rooms), `chat/middleware.py` (`JWTAuthMiddleware`, authenticating WebSocket connections via a `?token=` query param against SimpleJWT access tokens), `chat/routing.py`, and `backend/core/asgi.py` wiring it all together via `ProtocolTypeRouter`. Task 16.4.1.2 below **reuses this exact infrastructure** for in-app notifications rather than building a second, parallel WebSocket auth mechanism.
3. `settings.FRONTEND_URL` already exists (confirmed via the `_send_password_reset_email` code above), defaulting to `"http://localhost:3000"` — reuse this existing setting throughout this epic's email templates rather than assuming it needs to be added (it was referenced this way even before Epic 6's more formal `config("FRONTEND_URL", ...)` usage — reconcile these into one clean, consistently-imported setting as part of this epic's work).

---

## Phase 16.1 — Notification Infrastructure

### Feature 16.1.1 — Unified Notification System

---

#### Task 16.1.1.1 — Create `notifications` app

```
You are working in backend/. Assume Epics 1–15 are fully merged.

CONTEXT
No dedicated `notifications` app exists yet. Per Epic 2 Task 2.2.1.1's
own note at the time, the `SMSProvider` interface was deliberately
built inside `backend/accounts/sms/` with an explicit acknowledgment
that it might later be "relocated/absorbed" into a proper
`notifications` app once this epic arrived — that moment is now.
CONFIRM where Epic 2's actual implementation ended up (check
`backend/accounts/` for an `sms/` package, or check
`backend/notifications/` in case a prior task already created a
minimal scaffold) before deciding whether this task is a fresh `startapp`
or a re-home of existing code.

TASK
Scaffold (or formalize, if partially begun) the `notifications` app,
following this project's exact `payments`/`shipping`/`promotions`
app-scaffolding pattern (Epic 6/7/9 Task X.1.1.1 equivalents), AND
relocate Epic 2's SMS provider interface into it.

REQUIREMENTS
- If `backend/notifications/` doesn't exist yet: run
  `python manage.py startapp notifications`, name the config class
  `NotificationsConfig`, add `"notifications.apps.NotificationsConfig"`
  to `INSTALLED_APPS` (placed after `"promotions.apps.PromotionsConfig"`,
  continuing this project's roughly-dependency-ordered app list), add
  `path("api/notifications/", include("notifications.urls")),` to
  `backend/core/urls.py`, and create `notifications/tests/__init__.py`
  as a package — all identical in spirit to every prior epic's app-
  scaffolding task.
- RELOCATE Epic 2's SMS provider code: move
  `backend/accounts/sms/base.py` (the `SMSProvider` ABC),
  `backend/accounts/sms/console.py` (`ConsoleSMSProvider`), and the
  factory function into `backend/notifications/sms/` (same internal
  structure, new home). Update:
  - `backend/accounts/services/otp.py`'s import of `get_sms_provider`
    to point at the new location.
  - `SMS_PROVIDER_CLASS` default in `backend/core/settings/base.py`
    (Epic 2 Task 2.2.1.1) from
    `"accounts.sms.console.ConsoleSMSProvider"` to
    `"notifications.sms.console.ConsoleSMSProvider"`.
  - Every test referencing the old module path.
  This relocation is a pure move-and-rename with ZERO behavior change
  — the OTP flow must work identically after this task as before it;
  this is purely about giving the notification-domain code a proper,
  single home before this epic adds SMS templates, email templates, and
  in-app notifications alongside it.

ACCEPTANCE CRITERIA / TESTS
- Re-run the ENTIRE Epic 2 OTP test suite (request/verify flow,
  cooldown, throttling) after the relocation and confirm every test
  still passes, updated only for the new import paths, with zero
  behavior changes.
- `python manage.py check` passes.
- A trivial smoke test confirming the `notifications` app is
  registered, mirroring prior epics' equivalent tests.
```

---

#### Task 16.1.1.2 — `NotificationTemplate` model (channel + event + Persian template text)

```
You are working in backend/notifications/models.py. Assume Task
16.1.1.1 is already merged.

CONTEXT
No template storage exists — every notification-shaped message built
so far in this codebase (OTP SMS text, welcome/password-reset email
subject+body) is HARDCODED directly in Python string literals inline
in view/service code. This doesn't scale across the many notification
EVENTS this epic needs to support (order confirmation, shipping update,
back-in-stock, price-drop, order status change) and doesn't give
non-developer staff any way to adjust notification copy without a code
deploy.

TASK
Create an admin-editable `NotificationTemplate` model, keyed by
(channel, event), holding the actual Persian message text with
placeholder support.

REQUIREMENTS
- Add:
  ```python
  class NotificationTemplate(models.Model):
      class Channel(models.TextChoices):
          SMS = "sms", "SMS"
          EMAIL = "email", "Email"
          IN_APP = "in_app", "In-App"

      class Event(models.TextChoices):
          OTP_CODE = "otp_code", "OTP Verification Code"
          ORDER_CONFIRMED = "order_confirmed", "Order Confirmed"
          ORDER_STATUS_CHANGED = "order_status_changed", "Order Status Changed"
          SHIPPING_UPDATE = "shipping_update", "Shipping Update"
          BACK_IN_STOCK = "back_in_stock", "Back In Stock"
          PRICE_DROP = "price_drop", "Price Drop"
          WELCOME = "welcome", "Welcome"
          PASSWORD_RESET = "password_reset", "Password Reset"

      channel = models.CharField(max_length=10, choices=Channel.choices)
      event = models.CharField(max_length=30, choices=Event.choices)
      subject = models.CharField(
          max_length=200, blank=True,
          help_text="Email subject line. Ignored for SMS/in-app channels.",
      )
      body = models.TextField(
          help_text="Message body. Use {placeholder} syntax for dynamic values (e.g. {order_number}, {customer_name}).",
      )
      is_active = models.BooleanField(default=True)
      updated_at = models.DateTimeField(auto_now=True)

      class Meta:
          unique_together = ("channel", "event")
          ordering = ["event", "channel"]

      def __str__(self):
          return f"{self.get_event_display()} — {self.get_channel_display()}"

      def render(self, context: dict) -> str:
          """Render this template's body with the given context dict,
          leaving any unmatched {placeholder} untouched rather than
          raising, so a template referencing a not-yet-supported
          placeholder doesn't crash the whole notification send."""
          try:
              return self.body.format(**context)
          except (KeyError, IndexError):
              import logging
              logging.getLogger("notifications").warning(
                  "Template render for %s/%s missing context keys; sending partially unrendered.",
                  self.channel, self.event,
              )
              # Fall back to a safe partial-substitution rather than crashing —
              # use a defaultdict-like approach so missing keys become empty
              # rather than raising.
              class _SafeDict(dict):
                  def __missing__(self, key):
                      return f"{{{key}}}"
              return self.body.format_map(_SafeDict(context))
  ```
  The `_SafeDict` fallback in `render()` is a deliberate defensive
  choice: a missing template placeholder should degrade to visibly
  showing the unrendered `{placeholder}` token in logs/output (which an
  admin/developer will notice and fix) rather than silently crashing
  the ENTIRE notification pipeline for one malformed template — a
  broken order-confirmation SMS template should never prevent an order
  confirmation from being attempted at all.
- Generate the migration.
- Register in backend/notifications/admin.py as a normal, editable
  admin form (templates ARE meant to be hand-edited by admin/marketing
  staff, unlike the audit-log-style models elsewhere in this project):
  ```python
  @admin.register(NotificationTemplate)
  class NotificationTemplateAdmin(admin.ModelAdmin):
      list_display = ("event", "channel", "is_active", "updated_at")
      list_filter = ("channel", "event", "is_active")
  ```
- Add a data migration seeding a starting Persian template for EVERY
  (channel, event) combination that's actually going to be used by this
  epic's later tasks (OTP_CODE/sms already has a de-facto template — the
  hardcoded string from Epic 2 Task 2.2.1.3, `f"Your verification code is: {code}"`,
  which had an explicit `# TODO: Epic 14 — localize this message to
  Persian` comment attached — THIS is the natural place to finally
  close that TODO too, seeding a real Persian OTP template here). Write
  genuine, natural Persian copy for each seeded template (not machine-
  translated placeholder text), matching the same translation-quality
  bar established in Epic 14 Task 14.1.1.4/15.1.1.1.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/notifications/tests/test_models.py:
1. `render()` correctly substitutes all placeholders when the context
   dict has every needed key.
2. `render()` with a MISSING context key doesn't raise, logs a warning,
   and returns a string with the unmatched placeholder still visibly
   present (not silently blanked).
3. The `unique_together` constraint prevents two templates for the same
   (channel, event) pair.
4. The seed migration produces a real, non-empty, Persian-language
   template for OTP_CODE/sms at minimum (the highest-priority seeded
   template, given it closes a specific pre-existing TODO).
```

---

#### Task 16.1.1.3 — `NotificationLog` model (delivery tracking)

```
You are working in backend/notifications/models.py. Assume Task
16.1.1.2 is already merged.

CONTEXT
Nothing records whether a notification was actually sent, to whom, via
which channel, or whether it succeeded — mirroring the exact audit-log
gap already solved for stock movements (Epic 4), payment transactions
(Epic 6), and search queries (Epic 12) via dedicated log models; this
task applies the same established pattern to notifications.

TASK
Create a `NotificationLog` model.

REQUIREMENTS
- Add:
  ```python
  class NotificationLog(models.Model):
      class Status(models.TextChoices):
          SENT = "sent", "Sent"
          FAILED = "failed", "Failed"
          SKIPPED = "skipped", "Skipped"  # e.g. user opted out, or channel unavailable

      user = models.ForeignKey(
          settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True, blank=True,
          related_name="notification_logs",
      )
      channel = models.CharField(max_length=10, choices=NotificationTemplate.Channel.choices)
      event = models.CharField(max_length=30, choices=NotificationTemplate.Event.choices)
      recipient = models.CharField(
          max_length=255,
          help_text="Phone number, email address, or user ID depending on channel.",
      )
      status = models.CharField(max_length=10, choices=Status.choices)
      error_message = models.TextField(blank=True)
      sent_at = models.DateTimeField(auto_now_add=True)

      class Meta:
          ordering = ["-sent_at"]
          indexes = [models.Index(fields=["user", "event", "-sent_at"])]

      def __str__(self):
          return f"{self.event} to {self.recipient} via {self.channel} — {self.status}"
  ```
  Import `settings` from `django.conf` at the top of models.py.
  Generate the migration.
- Register as a read-only admin view (audit log, same pattern as
  `StockMovement`/`PaymentTransaction`/`SearchQueryLog`):
  ```python
  @admin.register(NotificationLog)
  class NotificationLogAdmin(admin.ModelAdmin):
      list_display = ("event", "channel", "recipient", "status", "sent_at")
      list_filter = ("channel", "event", "status")
      search_fields = ("recipient",)
      readonly_fields = [f.name for f in NotificationLog._meta.fields]

      def has_add_permission(self, request):
          return False

      def has_change_permission(self, request, obj=None):
          return False
  ```
  Add a useful admin filter for "recent failures" specifically
  (`status="failed"` combined with a recency filter) — mirroring the
  `NearExpiryFilter`/`LowStockFilter` pattern from Epics 3/4 — since a
  spike in notification failures (e.g. Kavenegar credentials expiring,
  SMTP misconfiguration) is exactly the kind of operationally-important
  signal this log exists to surface.

ACCEPTANCE CRITERIA / TESTS
Add a model test confirming a `NotificationLog` row can be created for
each status value, `user=None` is valid (e.g. a notification attempt
where the user account was since deleted), and `str()` produces a
readable representation.
```

---

#### Task 16.1.1.4 — Unified `notify(user, event, context)` service

```
You are working in backend/notifications/services.py (new file).
Assume Tasks 16.1.1.2/16.1.1.3 are already merged. This is the central,
load-bearing piece of this entire epic — every other Feature (SMS,
Email, In-App) plugs into this single entry point.

CONTEXT
Every notification-triggering call site across this codebase built so
far (Epic 2's OTP SMS, `accounts/views.py`'s welcome/password-reset
email, Epic 4's stock-alert TODO, Epic 8's order-status-change TODO,
Epic 11's price-drop TODO) currently either has its own ad-hoc, direct
send logic OR an explicit placeholder waiting for this task specifically.
None of them go through a single, consistent dispatch point that
handles: looking up the right template, determining which channel(s)
a given user/event should actually use, respecting notification
preferences, logging the outcome, and gracefully handling per-channel
failures without one channel's failure blocking another.

TASK
Build `notify(user, event, context)`, the single unified entry point
every notification-triggering call site in this codebase should route
through going forward.

REQUIREMENTS
- Implement:
  ```python
  from django.conf import settings
  from .models import NotificationTemplate, NotificationLog


  def notify(user, event: str, context: dict, channels: list[str] | None = None):
      """
      Send a notification to `user` for `event`, rendering the
      appropriate template(s) with `context`, across all applicable
      channels (or a specific subset via `channels`), logging each
      attempt.

      `user` may be None for notifications with no associated account
      (e.g. an OTP send during first-touch registration, before a User
      row necessarily exists yet, or a guest-checkout order confirmation)
      — in that case, `context` must supply an explicit recipient (e.g.
      `context["phone_number"]` / `context["email"]`) since there's no
      user object to derive one from.
      """
      target_channels = channels or _default_channels_for(user, event)
      for channel in target_channels:
          _send_single_channel(user, event, channel, context)


  def _default_channels_for(user, event: str) -> list[str]:
      """
      Determine which channels a notification should go out on by
      default, respecting user notification preferences (Profile's
      order_updates/promotions/newsletter flags, established in the
      original codebase) where applicable, and sensible per-event
      defaults otherwise.
      """
      # OTP is always SMS-only, non-negotiable — it's not a preference,
      # it's how the user proves phone ownership.
      if event == NotificationTemplate.Event.OTP_CODE:
          return [NotificationTemplate.Channel.SMS]

      channels = []
      if user is not None and hasattr(user, "profile"):
          if event in (
              NotificationTemplate.Event.ORDER_CONFIRMED,
              NotificationTemplate.Event.ORDER_STATUS_CHANGED,
              NotificationTemplate.Event.SHIPPING_UPDATE,
          ) and user.profile.order_updates:
              channels.extend([NotificationTemplate.Channel.SMS, NotificationTemplate.Channel.EMAIL])
          elif event in (
              NotificationTemplate.Event.BACK_IN_STOCK,
              NotificationTemplate.Event.PRICE_DROP,
          ) and user.profile.promotions:
              channels.append(NotificationTemplate.Channel.EMAIL)
      else:
          # No user / no profile (e.g. guest checkout) — fall back to
          # whatever explicit recipient info is present in context.
          if context.get("email"):
              channels.append(NotificationTemplate.Channel.EMAIL)
          if context.get("phone_number"):
              channels.append(NotificationTemplate.Channel.SMS)
      return channels


  def _send_single_channel(user, event, channel, context):
      try:
          template = NotificationTemplate.objects.get(channel=channel, event=event, is_active=True)
      except NotificationTemplate.DoesNotExist:
          NotificationLog.objects.create(
              user=user, channel=channel, event=event,
              recipient=_resolve_recipient(user, channel, context),
              status=NotificationLog.Status.SKIPPED,
              error_message="No active template configured for this channel/event.",
          )
          return

      recipient = _resolve_recipient(user, channel, context)
      if not recipient:
          NotificationLog.objects.create(
              user=user, channel=channel, event=event, recipient="",
              status=NotificationLog.Status.SKIPPED,
              error_message="No recipient address available for this channel.",
          )
          return

      body = template.render(context)
      try:
          if channel == NotificationTemplate.Channel.SMS:
              from .sms import get_sms_provider
              success = get_sms_provider().send(recipient, body)
          elif channel == NotificationTemplate.Channel.EMAIL:
              from .email import send_notification_email  # Feature 16.3.1
              subject = template.subject.format_map({**context, "__missing__": ""}) if template.subject else event
              success = send_notification_email(recipient, subject, body, context)
          elif channel == NotificationTemplate.Channel.IN_APP:
              from .services_inapp import create_in_app_notification  # Feature 16.4.1
              success = create_in_app_notification(user, event, body, context)
          else:
              success = False

          NotificationLog.objects.create(
              user=user, channel=channel, event=event, recipient=recipient,
              status=NotificationLog.Status.SENT if success else NotificationLog.Status.FAILED,
              error_message="" if success else "Provider reported failure.",
          )
      except Exception as exc:  # noqa: BLE001 — deliberately broad, see note below
          import logging
          logging.getLogger("notifications").exception(
              "Notification send failed: event=%s channel=%s", event, channel
          )
          NotificationLog.objects.create(
              user=user, channel=channel, event=event, recipient=recipient,
              status=NotificationLog.Status.FAILED, error_message=str(exc),
          )


  def _resolve_recipient(user, channel, context) -> str:
      if channel == NotificationTemplate.Channel.SMS:
          return context.get("phone_number") or (getattr(user, "phone_number", "") if user else "")
      if channel == NotificationTemplate.Channel.EMAIL:
          return context.get("email") or (getattr(user, "email", "") if user else "")
      if channel == NotificationTemplate.Channel.IN_APP:
          return str(user.id) if user else ""
      return ""
  ```
  The deliberately broad `except Exception` in `_send_single_channel`
  is a genuine, considered choice worth calling out explicitly rather
  than a lazy catch-all: `notify()` is called from many DIFFERENT
  business-critical code paths across this codebase (order creation,
  payment callbacks, stock movements) — a bug or transient failure in
  the notification subsystem (a malformed template, an SMS provider
  outage, an SMTP timeout) must NEVER be allowed to propagate up and
  break the actual business operation that triggered it (an order
  shouldn't fail to be CREATED just because the confirmation SMS
  failed to SEND) — this is the correct, deliberate boundary for a
  broad exception catch, unlike most other places in this codebase
  where narrow, specific exception handling is the right call.
  Modules referenced (`notifications/sms/`, `notifications/email.py`,
  `notifications/services_inapp.py`) are built in this epic's later
  Features — this task's job is the ORCHESTRATION layer; the imports
  inside `_send_single_channel` are LOCAL (function-scoped) imports
  deliberately, so this module doesn't hard-fail at import time before
  those other modules exist yet, and to avoid any circular-import risk
  given `email.py`/`services_inapp.py` will each need to import things
  FROM `notifications/models.py` themselves.
- Wire `notify()` into the THREE existing `# TODO: Epic 16` placeholder
  call sites right now, closing them out as part of this task (not
  deferred to later tasks — the placeholders were left EXACTLY for this
  moment):
  1. `shop/tasks.py`'s `notify_stock_alert_subscribers` (Epic 4 Task
     4.1.2.3): replace the TODO comment loop body with
     `notify(sub.user, NotificationTemplate.Event.BACK_IN_STOCK, {"product_name": variant.product.name, "variant_sku": variant.sku})`.
  2. `order/signals.py`'s `_log_status_change_for_now` receiver (Epic 8
     Task 8.1.1.2): replace/supplement the `logger.info(...)` placeholder
     with `notify(order.user, NotificationTemplate.Event.ORDER_STATUS_CHANGED, {"order_number": order.order_number, "old_status": previous_status, "new_status": order.status})`
     (keep the logging too, it's still useful operationally — just ADD
     the real notify call alongside it, don't necessarily remove the log
     line).
  3. `shop/tasks.py`'s `notify_price_drop_subscribers` (Epic 11 Task
     11.1.1.2): replace the TODO comment loop body with
     `notify(item.user, NotificationTemplate.Event.PRICE_DROP, {"product_name": product.name, "old_price": old_price, "new_price": new_price})`.
  Import `notify` and `NotificationTemplate` from `notifications.services`/
  `notifications.models` at each of these three call sites (check for
  circular-import risk given `notifications` now potentially needs to be
  imported from `shop` and `order` — this should be safe given
  `notifications` doesn't import FROM `shop`/`order` at module level,
  only inside function bodies per the local-import pattern already
  established above for its OWN internal cross-module calls).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/notifications/tests/test_services.py:
1. `notify()` with an event that has an active template for the
   determined channel(s) successfully calls the correct channel-specific
   send function (mock `get_sms_provider`/`send_notification_email`/
   `create_in_app_notification` and assert correct calls) and logs
   `SENT`.
2. `notify()` for an event/channel with NO active template logs
   `SKIPPED` with the correct reason, and does NOT attempt to call any
   send function.
3. `notify()` where the send function itself raises an exception is
   caught, logs `FAILED` with the exception message, and — critically —
   does NOT propagate the exception to the caller (construct a test
   confirming code calling `notify()` continues executing normally even
   when the underlying send fails catastrophically).
4. `OTP_CODE` events ALWAYS route to SMS only, regardless of any user
   profile preference (the non-negotiable channel override).
5. `_default_channels_for()` correctly respects `Profile.order_updates`/
   `promotions` flags — a user with `promotions=False` does not receive
   a `PRICE_DROP`/`BACK_IN_STOCK` notification even if otherwise
   eligible.
6. A `user=None` call (guest scenario) with `context={"phone_number": "09123456789"}`
   correctly resolves SMS as the channel and the phone number as
   recipient, with no crash from the missing user object.
7. Re-run the Epic 4/8/11 tests that were previously asserting against
   the placeholder TODO behavior (mocked `.delay` calls, log-only
   assertions) — update them to assert the REAL `notify()` call now
   happens with correct arguments at each of the three closed-out call
   sites.
```

---

## Phase 16.2 — SMS (Kavenegar)

### Feature 16.2.1 — Kavenegar Integration

---

#### Task 16.2.1.1 — Implement `KavenegarSMSProvider` (real implementation of Epic 2's interface)

```
You are working in backend/notifications/sms/kavenegar.py (new file,
in the relocated `sms/` package from Task 16.1.1.1). Assume that task
is already merged.

CONTEXT
`ConsoleSMSProvider` (Epic 2 Task 2.2.1.2, relocated in Task 16.1.1.1)
is currently the ONLY concrete `SMSProvider` implementation — it just
logs/prints, never sends a real SMS. Kavenegar is this platform's
chosen real Iranian SMS gateway (per the master backlog's explicit
naming). Same standing "verify current API docs before hardcoding"
caveat established throughout Epics 6/7 for external Iranian service
integrations applies here — Kavenegar's exact endpoint URLs/auth/
response shape should be verified against their current, authoritative
API documentation before implementation, not assumed from training
data alone.

TASK
Implement `KavenegarSMSProvider(SMSProvider)`, following the exact
same structural discipline established for every external-API
integration across Epics 6/7 (timeout on every request, broad-but-
logged exception handling degrading to a clean failure return rather
than propagating, never raising unhandled from the interface method).

REQUIREMENTS
- Add settings:
  `KAVENEGAR_API_KEY = config("KAVENEGAR_API_KEY", default="")`
  `KAVENEGAR_SENDER_LINE = config("KAVENEGAR_SENDER_LINE", default="")`
  (Kavenegar requires a registered sender line/number for outbound
  SMS — verify the exact current requirement/field name against their
  docs).
- Implement:
  ```python
  import requests
  from django.conf import settings
  from .base import SMSProvider


  class KavenegarSMSProvider(SMSProvider):
      BASE_URL = "https://api.kavenegar.com/v1"  # VERIFY against current docs

      def send(self, phone_number: str, message: str) -> bool:
          url = f"{self.BASE_URL}/{settings.KAVENEGAR_API_KEY}/sms/send.json"
          payload = {
              "receptor": phone_number,
              "message": message,
              "sender": settings.KAVENEGAR_SENDER_LINE,
          }
          try:
              response = requests.post(url, data=payload, timeout=15)
              response.raise_for_status()
              data = response.json()
          except (requests.RequestException, ValueError) as exc:
              import logging
              logging.getLogger("notifications.sms").error(
                  "Kavenegar send failed for %s: %s", phone_number, exc
              )
              return False

          # Verify the exact success-indication shape against current
          # Kavenegar docs — illustrative, not guaranteed accurate:
          return_data = data.get("return", {})
          status_code = return_data.get("status")
          if status_code == 200:
              return True
          import logging
          logging.getLogger("notifications.sms").error(
              "Kavenegar send rejected for %s: %s", phone_number, data
          )
          return False
  ```
  Note this returns a plain `bool` (matching the existing `SMSProvider.send()`
  interface contract established in Epic 2 Task 2.2.1.1 — `-> bool`),
  NOT a richer result dataclass like Epic 6's payment gateways use —
  don't change the interface signature this epic doesn't need to touch;
  Epic 2's simpler bool-return contract is sufficient for this
  provider's actual needs (unlike payments, there's no "verify" step
  or multi-stage flow for SMS sending that would justify a richer
  result type).
- Set `SMS_PROVIDER_CLASS` to point at this class in
  backend/core/settings/production.py specifically:
  `SMS_PROVIDER_CLASS = config("SMS_PROVIDER_CLASS", default="notifications.sms.kavenegar.KavenegarSMSProvider")`
  — leave `development.py` still defaulting to `ConsoleSMSProvider`
  (per Epic 2 Task 2.2.1.2's explicit warning that the console backend
  must never be used in production) — this is the actual, concrete
  moment that warning's counterpart (a real production default) gets
  implemented.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/notifications/tests/test_kavenegar.py, mocking
`requests.post` (never a real network call):
1. A mocked successful Kavenegar response returns `True`.
2. A mocked error/rejected response returns `False`, with the failure
   logged (assert via `caplog`/`assertLogs`).
3. A mocked network timeout/connection error returns `False` gracefully,
   never raising.
4. A mocked malformed (non-JSON) response returns `False` gracefully.
5. The outbound request includes a `timeout` parameter (assert via the
   mock's call inspection).
```

---

#### Task 16.2.1.2 — Order-confirmation SMS

```
You are working in backend/order/serializers.py. Assume Task 16.1.1.4
is already merged.

CONTEXT
Nothing currently sends an SMS confirming a placed order — this is one
of the most basic, expected transactional SMS events for any Iranian
e-commerce platform (Iranian consumers heavily expect SMS confirmation
for purchases, arguably even more so than email, given SMS/phone-
centric digital habits in this market).

TASK
Trigger `notify(..., event=ORDER_CONFIRMED, ...)` at order creation.

REQUIREMENTS
- In `OrderCreateSerializer.create()` (inside the existing atomic
  block, AFTER the order and all its items are successfully created —
  note this fires on ORDER creation per this codebase's existing flow,
  which per Epic 6 means the order is created in `PENDING` status
  BEFORE payment — decide whether "order confirmed" should fire at
  PENDING-creation time or wait until PAYMENT SUCCESS (Epic 6's
  `PaymentCallbackView` transitioning the order to `PROCESSING`).
  RECOMMEND: fire at PAYMENT SUCCESS, not at PENDING creation — a
  "your order is confirmed!" SMS sent before payment has even been
  attempted would be actively misleading if the payment subsequently
  fails/is abandoned (per Epic 6 Task 6.2.1.4's stock-release-on-
  failure logic — an order that never actually completes shouldn't
  have told the customer it was "confirmed"). This means the correct
  call site is actually `payments/views.py`'s `PaymentCallbackView`
  success path (Epic 6 Task 6.2.1.4), NOT `OrderCreateSerializer.create()`
  as this task's title might suggest at first glance — implement it
  there instead, and note this deliberate placement decision clearly in
  your task summary):
  ```python
  # in PaymentCallbackView's success branch, after order.status = PROCESSING
  from notifications.services import notify
  from notifications.models import NotificationTemplate

  notify(
      order.user, NotificationTemplate.Event.ORDER_CONFIRMED,
      {
          "order_number": order.order_number,
          "total": str(order.total),
          "phone_number": order.shipping_phone if not order.user else None,  # guest checkout support
      },
  )
  ```
  Handle GUEST checkout correctly (per Epic 5's guest cart/checkout
  support) — `order.user` may be `None` for a guest order; per Task
  16.1.1.4's `notify()` design, passing `context["phone_number"]`
  explicitly handles this case correctly even with `user=None`, since
  `_resolve_recipient()` falls back to the context-supplied value.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/payments/tests/:
1. A successful payment callback (per Epic 6's existing test structure)
   now ALSO triggers `notify()` with `ORDER_CONFIRMED` and the correct
   order number/total (mock `notify` and assert the call).
2. A GUEST order's confirmation notification correctly resolves the
   phone number from the order's shipping info rather than a `None`
   user object.
3. A FAILED payment does NOT trigger an order-confirmed notification
   (confirming the deliberate placement decision above holds correctly
   — no confirmation for an order that never actually completed).
```

---

#### Task 16.2.1.3 — Shipping-update SMS

```
You are working in backend/shipping/tasks.py
(poll_shipment_tracking, Epic 7 Task 7.2.2.2). Assume Task 16.1.1.4 is
merged.

CONTEXT
Epic 7's shipment-tracking polling task updates `Shipment.status` and,
when it transitions to `DELIVERED`, updates `Order.status` too — but
never notifies the customer of ANY shipping status change along the
way (dispatched, out for delivery, delivered).

TASK
Trigger `notify(..., event=SHIPPING_UPDATE, ...)` whenever
`poll_shipment_tracking` detects a real status change.

REQUIREMENTS
- In `poll_shipment_tracking()` (Epic 7 Task 7.2.2.2), inside the
  existing `if result.status and result.status != shipment.status:`
  branch (which already detects an actual change before updating), add:
  ```python
  from notifications.services import notify
  from notifications.models import NotificationTemplate

  notify(
      shipment.order.user, NotificationTemplate.Event.SHIPPING_UPDATE,
      {
          "order_number": shipment.order.order_number,
          "status": shipment.get_status_display(),
          "tracking_number": shipment.tracking_number,
          "phone_number": shipment.order.shipping_phone if not shipment.order.user else None,
      },
  )
  ```
  Placed alongside (not replacing) the existing `Order.status = DELIVERED`
  transition logic already in that branch for the delivered case — this
  notification should fire for EVERY detected status transition (per
  the task title "shipping update," not just the final delivered one),
  not just the delivered-specific one.
- Consider whether every single intermediate status change genuinely
  warrants a full SMS (which has a real per-message cost via Kavenegar)
  — a reasonable, defensible policy: notify on `OUT_FOR_DELIVERY` and
  `DELIVERED` specifically (the two statuses a customer most actively
  cares about being alerted to in real time), but not necessarily on
  `IN_TRANSIT` (a less actionable, lower-value update) — implement this
  as an explicit filter if you agree with this reasoning, or notify on
  every change if you judge otherwise; document whichever choice you
  make and why, since this is a genuine cost/value tradeoff worth a
  deliberate decision rather than an unexamined default.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shipping/tests/test_tasks.py:
1. A detected shipment status change (of whichever status values you
   decided warrant notification) triggers `notify()` with the correct
   order/status/tracking details.
2. A status "change" to the SAME value (already covered by the existing
   `!= shipment.status` guard from Epic 7) does not trigger a duplicate
   notification.
3. If you implemented the notify-only-on-specific-statuses filter,
   confirm a transition to an EXCLUDED status (e.g. `IN_TRANSIT`, if you
   chose to exclude it) updates the `Shipment` row correctly but does
   NOT call `notify()`.
```

---

#### Task 16.2.1.4 — SMS delivery failure handling/retry

```
You are working in backend/notifications/services.py, tasks.py (new
file). Assume Tasks 16.1.1.4 and 16.2.1.1 are already merged.

CONTEXT
`notify()`'s current failure handling (Task 16.1.1.4) logs a `FAILED`
`NotificationLog` row on any send failure but never RETRIES — a
transient Kavenegar API blip (network hiccup, brief rate-limit) would
currently just permanently fail that one notification with no recovery
mechanism, unlike Epic 6's payment reconciliation task, which DOES
retry stuck/failed states.

TASK
Add a bounded retry mechanism specifically for SMS channel failures
(the highest-value channel to retry, given its role in time-sensitive
order/shipping updates), via a Celery task rather than inline
synchronous retry (which would block the original request/task that
triggered the notification).

REQUIREMENTS
- Modify `_send_single_channel()` (Task 16.1.1.4) for the SMS channel
  specifically: on a `False` return from `get_sms_provider().send()`
  (a clean, non-exception failure), instead of immediately logging
  `FAILED` and stopping, queue a retry:
  ```python
  # inside the SMS branch of _send_single_channel, on failure:
  if not success:
      from .tasks import retry_sms_notification
      retry_sms_notification.apply_async(
          kwargs={"recipient": recipient, "body": body, "event": event, "user_id": user.id if user else None},
          countdown=60,  # wait 60s before first retry
      )
  ```
  Add `retry_sms_notification` to backend/notifications/tasks.py:
  ```python
  from celery import shared_task

  @shared_task(bind=True, max_retries=3, default_retry_delay=120)
  def retry_sms_notification(self, recipient, body, event, user_id):
      from .sms import get_sms_provider
      from .models import NotificationLog
      success = get_sms_provider().send(recipient, body)
      if success:
          NotificationLog.objects.create(
              user_id=user_id, channel="sms", event=event, recipient=recipient,
              status=NotificationLog.Status.SENT, error_message="(sent on retry)",
          )
          return
      if self.request.retries >= self.max_retries:
          NotificationLog.objects.create(
              user_id=user_id, channel="sms", event=event, recipient=recipient,
              status=NotificationLog.Status.FAILED,
              error_message=f"Failed after {self.max_retries} retries.",
          )
          return
      raise self.retry()
  ```
  Note the ORIGINAL `_send_single_channel()` call site should STILL log
  its own `FAILED` (or a new distinct status, e.g. consider whether
  `NotificationLog.Status` needs a `RETRYING` value to distinguish
  "permanently failed" from "failed once, retry queued" in the log —
  add this if it meaningfully improves admin visibility into what's
  actually happening, since an admin looking at the notification log
  should be able to tell the difference between "this never went out at
  all" and "this is currently being retried") — decide and implement
  clearly, don't leave the log ambiguous between these two genuinely
  different states.
- Do NOT apply this retry mechanism to EMAIL or IN_APP channels in this
  task — SMS specifically is the highest-value, most time-sensitive
  channel given Kavenegar's real-world transient-failure patterns and
  the OTP/order-confirmation use cases riding on it; email failures are
  lower-stakes (an email retry mechanism, if warranted, is a reasonable
  separate future task, not something to bundle into this one).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/notifications/tests/test_tasks.py:
1. `retry_sms_notification` succeeding on a retry attempt logs `SENT`
   with the "(sent on retry)" note.
2. `retry_sms_notification` failing on EVERY retry attempt up to
   `max_retries` eventually logs a final `FAILED` with the correct
   retry-count message, and does NOT retry indefinitely.
3. A failed initial SMS send (via `notify()`) correctly queues
   `retry_sms_notification` with the correct arguments (mock `.apply_async`
   and assert the call).
4. Confirm EMAIL/IN_APP channel failures do NOT trigger any retry
   queuing (only SMS does).
```

---

## Phase 16.3 — Email

### Feature 16.3.1 — Transactional Email

---

#### Task 16.3.1.1 — Persian-RTL HTML email templates (order confirmation, shipping, password reset)

```
You are working in backend/templates/ (email template directory) and
backend/notifications/email.py (new file). Assume Task 16.1.1.4 is
already merged, and Epic 14's RTL/Persian-font work is merged
(informing this task's email template styling).

CONTEXT
`accounts/emails/welcome.html` and `accounts/emails/password_reset.html`
(confirmed to exist per this document's header) are the ONLY existing
email templates — plain, English-language, presumably LTR-styled HTML.
This task builds out proper Persian-RTL email templates for this
epic's new transactional events, AND should audit/fix the two existing
templates to match (they predate this epic's localization work and are
very likely still English/LTR).

TASK
Build RTL-safe, table-based (for email client compatibility) HTML
email templates for order confirmation, shipping updates, and audit/
fix the existing welcome/password-reset templates for Persian/RTL
correctness.

REQUIREMENTS
- Email HTML has notoriously poor/inconsistent CSS support across
  clients (especially older Outlook versions, still relevant for some
  business users) — use TABLE-BASED layout (not flexbox/grid, which
  many email clients don't support), inline styles (not a `<style>`
  block, since many clients strip `<head>` content), and explicit
  `dir="rtl"` on the outermost table/body element PLUS `text-align: right`
  set explicitly via inline style on text-containing cells (don't rely
  on `dir="rtl"` alone to correctly cascade text-alignment across every
  email client, since email CSS support is inconsistent — be
  explicit/redundant here rather than assuming standard web-CSS
  inheritance behavior holds).
- Create backend/templates/notifications/emails/order_confirmed.html
  and shipping_update.html following this table-based, explicit-RTL
  pattern, using Django template variables matching the `context` dict
  keys `notify()` passes (order_number, total, tracking_number, etc.
  per Tasks 16.2.1.2/16.2.1.3's context payloads).
- Audit `accounts/emails/welcome.html` and `password_reset.html`:
  convert to the same table-based RTL pattern, translate their content
  to real Persian (matching the translation-quality bar established
  throughout Epic 14/15), and fix the `_send_welcome_email()`/
  `_send_password_reset_email()` functions' hardcoded English
  `subject`/plain-text `message` fallback strings (the `message=f"Hi..."`
  arguments passed to `send_mail()`) to Persian equivalents too — an
  HTML email's PLAIN-TEXT fallback (shown to email clients/screen
  readers that don't render HTML) currently being English while the
  HTML version becomes Persian would be an inconsistent, confusing
  regression to leave unfixed while you're already touching these
  functions.
- Fix the code-smell `FRONTEND_URL` resolution in
  `_send_password_reset_email()` (per this document's header note) —
  replace the awkward `getattr(__import__("django.conf", fromlist=["settings"]).settings, "FRONTEND_URL", "http://localhost:3000")`
  construct with a normal `from django.conf import settings` import at
  the top of the file and a plain `settings.FRONTEND_URL` reference —
  this is a small, safe cleanup with zero behavior change, appropriate
  to fix while already working in this exact function for this task's
  broader email-template work.

ACCEPTANCE CRITERIA / TESTS
- Manually verify (send a test email via the console/locmem backend in
  dev and inspect the rendered HTML, or use an email-preview tool if
  available in your environment) that each template renders correctly
  RTL-aligned with readable Persian text.
- Add/update tests confirming `_send_welcome_email()`/
  `_send_password_reset_email()` still successfully send (via the
  existing `locmem` test email backend, per
  `accounts/tests/conftest.py`'s established test configuration) with
  the new Persian content, and that the plain-text fallback `message`
  argument is now Persian, not the old English string.
- Add a test confirming the `FRONTEND_URL` cleanup didn't change
  behavior — the reset URL construction still correctly uses the
  configured `FRONTEND_URL` value.
```

---

#### Task 16.3.1.2 — Wire email sending through Celery (async, not blocking request)

```
You are working in backend/notifications/email.py,
backend/accounts/views.py. Assume Task 16.3.1.1 is already merged and
Celery infrastructure exists (per this project's established Epic-22
dependency pattern).

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
`_send_welcome_email()` and `_send_password_reset_email()` currently
call Django's `send_mail()` SYNCHRONOUSLY, INLINE in the request/
response cycle — meaning every registration or password-reset-request
API call currently BLOCKS on the full SMTP round-trip before returning
a response to the client. For a `production.py` SMTP backend (as
opposed to the `console`/`locmem` dev backends), this is a real,
measurable latency cost added directly to a customer-facing request,
and a genuine failure-mode risk (a slow/unresponsive SMTP server would
directly slow down or fail user registration).

TASK
Move email sending (both the two existing calls AND this epic's new
`notify()`-driven email dispatch) onto Celery, off the synchronous
request path entirely.

REQUIREMENTS
- Implement `send_notification_email()` (referenced by Task 16.1.1.4's
  `_send_single_channel()`) in backend/notifications/email.py as a
  Celery task, not a synchronous function:
  ```python
  from celery import shared_task
  from django.core.mail import send_mail
  from django.template.loader import render_to_string


  @shared_task
  def send_notification_email_task(recipient, subject, body_text, template_name, context):
      html_message = render_to_string(template_name, context) if template_name else None
      send_mail(
          subject=subject,
          message=body_text,
          from_email=None,
          recipient_list=[recipient],
          html_message=html_message,
          fail_silently=False,  # let Celery's own retry/failure tracking see real exceptions, unlike the old inline fail_silently=True
      )


  def send_notification_email(recipient, subject, body_text, context, template_name=None) -> bool:
      """Synchronous-looking wrapper called by notify()'s dispatch logic
      — actually just QUEUES the real send as a Celery task and returns
      True immediately (queuing succeeded), since the actual send
      success/failure is tracked separately via the Celery task's own
      execution, not synchronously observable here."""
      send_notification_email_task.delay(recipient, subject, body_text, template_name, context)
      return True
  ```
  Note the RETURN VALUE here is `True` (meaning "successfully QUEUED,"
  not "successfully DELIVERED") — this is a real, worth-documenting
  semantic difference from how the SMS channel's `send()` return value
  works (SMS providers in this codebase return `True` only on confirmed
  API-level acceptance) — email's async nature means `notify()`'s
  `NotificationLog` entry for the EMAIL channel will show `SENT` at the
  moment of successful QUEUING, not confirmed delivery; if more precise
  delivery tracking for email specifically is ever needed later, that
  would require the Celery task itself to update the
  `NotificationLog` row after actually attempting the send (a
  reasonable future enhancement, flag it as such rather than building
  it now unless it's cheap to add — consider adding it if
  straightforward, since having the log honestly reflect "queued" vs.
  "actually delivered" is a real, valuable distinction; use your
  judgment on whether to build the fuller version now or note it as a
  follow-up).
- Update `_send_welcome_email()`/`_send_password_reset_email()` in
  `accounts/views.py` to queue via Celery instead of calling
  `send_mail()` directly and synchronously:
  ```python
  from notifications.email import send_notification_email_task

  def _send_welcome_email(user):
      html_context = {"first_name": user.profile.first_name, "email": user.email}
      send_notification_email_task.delay(
          recipient=user.email,
          subject="به تابلوژنیکس خوش آمدید",  # per Task 16.3.1.1's Persian translation
          body_text=f"سلام {user.profile.first_name}، به تابلوژنیکس خوش آمدید!",
          template_name="accounts/emails/welcome.html",
          context=html_context,
      )
  ```
  (adjust `send_notification_email_task`'s signature slightly if
  needed to cleanly support this direct-template-name call pattern
  alongside `notify()`'s own dispatch path — both should route through
  the SAME underlying Celery task rather than maintaining two separate
  async-email-sending mechanisms).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/notifications/tests/test_email.py and update
backend/accounts/tests/:
1. Calling `_send_welcome_email()`/`_send_password_reset_email()` now
   queues a Celery task (`.delay()` called, mocked/asserted) rather
   than synchronously calling `send_mail()` directly — this is the core
   regression-proof for this task's actual goal (confirm the
   REQUEST-BLOCKING behavior is genuinely gone, not just that email
   still eventually gets sent somehow).
2. `send_notification_email_task` (run synchronously in the test,
   Celery's standard test-execution pattern) correctly sends via
   Django's configured email backend (using the `locmem` test backend,
   per existing test conventions) with the correct rendered HTML and
   plain-text content.
3. Re-run the FULL existing accounts test suite (registration,
   password reset) and confirm they still pass — these tests likely
   need updating from asserting against `django.core.mail.outbox`
   IMMEDIATELY after the API call (which won't work anymore now that
   sending is async) to either mocking the Celery task call directly,
   or executing the queued task synchronously within the test (check
   how this project's OTHER existing Celery-task tests, e.g. from Epic
   3/4/7's async work, handle this exact "assert on eventual side
   effect of an async task" pattern, and follow the same established
   convention for consistency).
```

---

## Phase 16.4 — In-App / Push (optional tier)

### Feature 16.4.1 — In-App Notifications

---

#### Task 16.4.1.1 — `UserNotification` model for in-app bell/inbox

```
You are working in backend/notifications/models.py. Assume Phase 16.1
is fully merged.

CONTEXT
No persisted, customer-visible-in-account in-app notification concept
exists — `NotificationLog` (Task 16.1.1.3) is an ADMIN-facing audit
log, not something a customer should see directly (it includes
provider error details, internal event-key names, etc., inappropriate
for direct customer display).

TASK
Create a `UserNotification` model — the actual customer-facing,
in-account notification-inbox record, distinct from the admin audit
log.

REQUIREMENTS
- Add:
  ```python
  class UserNotification(models.Model):
      user = models.ForeignKey(
          settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="in_app_notifications"
      )
      event = models.CharField(max_length=30, choices=NotificationTemplate.Event.choices)
      message = models.TextField()
      link_url = models.CharField(
          max_length=255, blank=True,
          help_text="Optional in-app relative URL to navigate to when this notification is clicked (e.g. an order detail page).",
      )
      is_read = models.BooleanField(default=False)
      created_at = models.DateTimeField(auto_now_add=True)

      class Meta:
          ordering = ["-created_at"]
          indexes = [models.Index(fields=["user", "is_read", "-created_at"])]

      def __str__(self):
          return f"{self.user.email} — {self.event} ({'read' if self.is_read else 'unread'})"
  ```
  Generate the migration.
- Add `create_in_app_notification()` in backend/notifications/services_inapp.py
  (referenced by Task 16.1.1.4's `_send_single_channel()` local import):
  ```python
  def create_in_app_notification(user, event, message, context) -> bool:
      if user is None:
          return False  # in-app notifications inherently require a real account
      link_url = _build_link_url(event, context)
      UserNotification.objects.create(user=user, event=event, message=message, link_url=link_url)
      return True


  def _build_link_url(event, context) -> str:
      # Map specific events to a sensible in-app destination — e.g. an
      # order-related event links to that order's detail page.
      if "order_number" in context:
          return f"/account/orders/{context.get('order_id', '')}/"
      if "product_name" in context and "product_slug" in context:
          return f"/products/{context['product_slug']}/"
      return ""
  ```
  Note `create_in_app_notification` correctly returns `False` (not an
  exception) for a `None` user — an in-app notification fundamentally
  requires a real account to attach to, unlike SMS/email which can work
  for guest checkouts via context-supplied contact info; this is a
  genuine, expected `SKIPPED`-worthy case in `notify()`'s logging, not
  a bug.
- Add a `GET /api/notifications/` endpoint (paginated, authenticated,
  scoped to `request.user`) and a `PATCH /api/notifications/{id}/`
  endpoint (mark as read) and a `POST /api/notifications/mark-all-read/`
  bulk action, in backend/notifications/views.py/urls.py, following
  this project's established `generics`/`APIView` conventions.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/notifications/tests/:
1. `create_in_app_notification()` for a real user creates a
   `UserNotification` row with a correctly-derived `link_url`.
2. `create_in_app_notification()` for `user=None` returns `False`
   without creating anything.
3. `GET /api/notifications/` returns only the requesting user's
   notifications, correctly paginated, most-recent-first.
4. `PATCH /api/notifications/{id}/` (mark read) only works for the
   OWNING user's own notifications (a different user's notification ID
   returns 404, not leaking existence).
5. `POST /api/notifications/mark-all-read/` correctly marks every
   unread notification for the requesting user, and doesn't touch other
   users' notifications.
```

---

#### Task 16.4.1.2 — Real-time delivery via existing Django Channels setup

```
You are working in backend/notifications/consumers.py, routing.py,
backend/core/asgi.py, and frontend/src. Assume Task 16.4.1.1 is
already merged. RE-READ this document's header context on the existing
`chat` app's Channels setup before starting — this task reuses that
infrastructure directly rather than building a parallel one.

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
`chat/consumers.py` (`AsyncWebsocketConsumer`, group-based rooms
keyed `f"chat_{room_id}"`), `chat/middleware.py`
(`JWTAuthMiddleware`, authenticating via a `?token=` query param
against SimpleJWT access tokens), and `core/asgi.py`
(`ProtocolTypeRouter` combining the HTTP app and
`JWTAuthMiddleware(URLRouter(websocket_urlpatterns))`) already provide
a complete, working, tested pattern for authenticated WebSocket
connections in this exact project. This task creates a SECOND consumer
using the SAME `JWTAuthMiddleware` and the SAME `ProtocolTypeRouter`,
not a parallel authentication mechanism.

TASK
Add a `NotificationConsumer` delivering real-time in-app notifications
to a connected, authenticated user, and wire `create_in_app_notification()`
(Task 16.4.1.1) to push over this channel when the user is currently
connected.

REQUIREMENTS
- Create backend/notifications/consumers.py, closely mirroring
  `chat/consumers.py`'s structure but simplified for this narrower
  purpose (no rooms/multi-party concept — each user has exactly ONE
  personal notification channel):
  ```python
  import json
  import logging
  from channels.generic.websocket import AsyncWebsocketConsumer
  from django.contrib.auth.models import AnonymousUser

  logger = logging.getLogger(__name__)


  class NotificationConsumer(AsyncWebsocketConsumer):
      async def connect(self):
          self.user = self.scope.get("user")
          if not self.user or isinstance(self.user, AnonymousUser):
              logger.warning("Unauthenticated notification WS rejected")
              await self.close(code=4001)
              return
          self.group_name = f"notifications_{self.user.id}"
          await self.channel_layer.group_add(self.group_name, self.channel_name)
          await self.accept()

      async def disconnect(self, close_code):
          if hasattr(self, "group_name"):
              await self.channel_layer.group_discard(self.group_name, self.channel_name)

      async def notification_message(self, event):
          """Handler for messages sent to this user's group — the method
          name MUST match the 'type' key used when sending to the group
          (Channels converts 'notification.message' <-> this method
          name automatically)."""
          await self.send(text_data=json.dumps(event["payload"]))
  ```
- Create backend/notifications/routing.py:
  ```python
  from django.urls import re_path
  from . import consumers

  websocket_urlpatterns = [
      re_path(r"^ws/notifications/$", consumers.NotificationConsumer.as_asgi()),
  ]
  ```
- Update backend/core/asgi.py to COMBINE both apps' `websocket_urlpatterns`
  under the SAME `JWTAuthMiddleware` wrapper (reusing it, not
  duplicating it):
  ```python
  from chat.routing import websocket_urlpatterns as chat_ws_patterns
  from notifications.routing import websocket_urlpatterns as notifications_ws_patterns

  application = ProtocolTypeRouter({
      "http": django_asgi_app,
      "websocket": JWTAuthMiddleware(URLRouter(chat_ws_patterns + notifications_ws_patterns)),
  })
  ```
- Update `create_in_app_notification()` (Task 16.4.1.1) to ALSO push
  over the channel layer if the user happens to be currently connected
  (a no-op, harmless send if they're not — Channels' group_send to an
  empty/nonexistent group is safe and simply delivers to nobody):
  ```python
  from channels.layers import get_channel_layer
  from asgiref.sync import async_to_sync

  def create_in_app_notification(user, event, message, context) -> bool:
      if user is None:
          return False
      link_url = _build_link_url(event, context)
      notification = UserNotification.objects.create(user=user, event=event, message=message, link_url=link_url)
      channel_layer = get_channel_layer()
      async_to_sync(channel_layer.group_send)(
          f"notifications_{user.id}",
          {
              "type": "notification.message",
              "payload": {
                  "id": notification.id, "event": event, "message": message,
                  "link_url": link_url, "created_at": notification.created_at.isoformat(),
              },
          },
      )
      return True
  ```
  Note `create_in_app_notification()` is called from `notify()`'s
  synchronous dispatch path (Task 16.1.1.4) — `async_to_sync()` is the
  correct bridge here since Channels' `channel_layer.group_send` is an
  async API but this whole call chain runs in a synchronous Django
  request/Celery-task context; confirm this bridging pattern doesn't
  introduce any issues given this project's existing async/sync
  boundaries elsewhere (Channels + `asgiref.sync` is the standard,
  well-supported way to call into a channel layer from synchronous
  code, so this should be safe, but verify functionally with a real
  test rather than assuming).

REQUIREMENTS — frontend
- Add a WebSocket connection (in a new `NotificationContext.jsx`,
  mirroring however `chat`'s existing frontend WebSocket connection
  code — if the chat feature has a frontend consumer already, find and
  mirror its exact connection/reconnection/auth-token-passing pattern
  closely, since it already correctly handles the `?token=` JWT query
  param convention this project's `JWTAuthMiddleware` expects) to
  `ws://.../ws/notifications/?token=<access_token>`, listening for
  incoming messages and updating a notification-bell unread count/list
  in real time (built out visually in Task 16.4.1.3).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/notifications/tests/test_consumers.py, mirroring
`chat/tests/test_consumers.py`'s existing test structure/conventions
closely:
1. An authenticated WebSocket connection to `/ws/notifications/`
   succeeds and joins the correct per-user group.
2. An UNauthenticated connection attempt is rejected (matching the
   `chat` consumer's exact rejection behavior/close code convention).
3. Calling `create_in_app_notification()` for a user with an ACTIVE
   WebSocket connection results in that connection receiving the
   correct real-time message payload (use Channels' testing utilities —
   `channels.testing.WebsocketCommunicator` — matching however
   `chat/tests/test_consumers.py` already tests the equivalent
   real-time-delivery behavior for chat messages).
4. Calling `create_in_app_notification()` for a user with NO active
   connection doesn't raise or hang — the group_send simply delivers to
   nobody, and the function still correctly creates/returns the
   persisted `UserNotification` row regardless of live-connection
   status.
```

---

#### Task 16.4.1.3 — Notification bell UI component

```
You are working in frontend/src/components/ (new component) and
wherever the site's main header/navbar lives. Assume Task 16.4.1.2 is
already merged.

CONTEXT
No notification UI exists at all — `NotificationContext.jsx` (Task
16.4.1.2) now maintains real-time notification state, but nothing
displays it to the customer.

TASK
Build a notification bell icon (in the header/navbar) with an unread-
count badge and a dropdown list of recent notifications.

REQUIREMENTS
- Add API methods to frontend/src/services/api.js:
  ```javascript
  listNotifications: () => api.get('/notifications/'),
  markNotificationRead: (id) => api.patch(`/notifications/${id}/`, { is_read: true }),
  markAllNotificationsRead: () => api.post('/notifications/mark-all-read/'),
  ```
- Build `NotificationBell.jsx`: a bell icon (using this project's
  established icon library, per Epic 14 Task 14.3.1.3's confirmed
  Bootstrap Icons usage) with a small unread-count badge (only shown
  when count > 0, capped display at e.g. "9+" for large counts rather
  than an unbounded number that could overflow the badge visually),
  placed in the header/navbar alongside existing icons (cart, wishlist,
  account — match their exact existing visual style/spacing).
- Clicking the bell opens a dropdown showing recent notifications
  (message text, relative timestamp — e.g. "2 hours ago," using
  whatever relative-time formatting utility this project already has,
  or Epic 14's Jalali date utility if a relative-time helper doesn't
  already exist), each clickable to navigate to its `link_url` (per
  Task 16.4.1.1's derived link) and mark itself read on click.
- Add a "mark all as read" action within the dropdown.
- On mount, fetch the initial notification list via
  `listNotifications()` (the REST endpoint, for the notifications that
  existed BEFORE this session's WebSocket connection was established),
  then MERGE in real-time updates received via `NotificationContext`'s
  WebSocket connection (Task 16.4.1.2) as they arrive — new
  notifications should appear at the top of the dropdown list and
  increment the unread badge WITHOUT requiring a page refresh or
  re-fetch, which is the whole point of the real-time channel existing.
- Handle the WebSocket DISCONNECTED state gracefully (e.g. the
  connection drops due to a network blip) — the bell/dropdown should
  still work correctly using the REST-fetched data as a fallback, and
  ideally reconnect automatically (check whatever reconnection pattern
  the existing `chat` frontend feature already uses, per Task 16.4.1.2's
  instruction to mirror it, and apply the same reconnection logic here
  for consistency).

ACCEPTANCE CRITERIA
Manually verify: the bell shows the correct unread count on page load;
triggering a real notification-worthy event elsewhere in the app (e.g.
an admin changing an order's status, per Task 16.1.1.4's wired-up
`ORDER_STATUS_CHANGED` event) while the customer has the page open
results in the bell's badge count incrementing in real time without a
page refresh; clicking a notification navigates correctly and marks it
read; "mark all as read" clears the badge. Add component tests for
`NotificationBell` covering: renders correct unread count, renders the
dropdown list, "mark all read" calls the correct API and clears the
badge, and a mocked incoming WebSocket message correctly updates
displayed state.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 16.1.1.1 | Create notifications app + relocate Epic 2's SMS module | ☐ |
| 16.1.1.2 | NotificationTemplate model | ☐ |
| 16.1.1.3 | NotificationLog model | ☐ |
| 16.1.1.4 | Unified notify(user, event, context) service | ☐ |
| 16.2.1.1 | Implement KavenegarSMSProvider | ☐ |
| 16.2.1.2 | Order-confirmation SMS | ☐ |
| 16.2.1.3 | Shipping-update SMS | ☐ |
| 16.2.1.4 | SMS delivery failure handling/retry | ☐ |
| 16.3.1.1 | Persian-RTL HTML email templates | ☐ |
| 16.3.1.2 | Wire email sending through Celery | ☐ |
| 16.4.1.1 | UserNotification model (in-app inbox) | ☐ |
| 16.4.1.2 | Real-time delivery via existing Channels setup | ☐ |
| 16.4.1.3 | Notification bell UI component | ☐ |

Once Epic 16 is fully merged, the next epics to generate prompts for
are **Epic 17 — Admin Dashboard** and **Epic 18 — Customer Dashboard**,
both of which consume data/features from most epics built so far
(orders, coupons, stock, notifications) rather than introducing major
new domains of their own.
