## Phase 1 — Implementation Prompts (English)

### Prompt 1 — Build the `PaymentGatewayProvider` Interface + `InternalSimulatedGatewayProvider`

```
Goal: Build the abstract payment gateway layer for the Afra project,
without changing any other existing file and without wiring it into any
current code path. By the end of this prompt, the current logic of
capture_payment_for_booking must exist, one-to-one, as an implementation
of this interface — but nothing should call it yet.

Before starting, read these files in full:
1. apps/payments/services.py — the capture_payment_for_booking function
   (lines ~91-199), in full detail, including all logic that changes
   Payment.status and Booking.status and calls
   notify_new_booking_request
2. apps/payments/models.py — the Payment class (to understand the
   status/provider/gateway_charge_id/failure_reason fields)
3. apps/common/money.py (if Phase 0 has been implemented) to understand
   the Money value object

What to build:

1. Create the folder apps/payments/gateways/ with an empty __init__.py.

2. Create apps/payments/gateways/base.py containing:
   - A dataclass GatewayCreateResult with fields:
     redirect_url: str | None (None means no external redirect is
     needed — the internal simulated case)
     gateway_reference: str (the identifier used later for verification)
     requires_redirect: bool
   - A dataclass GatewayVerification with fields:
     success: bool
     gateway_transaction_id: str | None
     failure_reason: str
     raw_response: dict (for full logging)
   - A dataclass GatewayRefundResult with fields:
     success: bool
     gateway_refund_id: str | None
     failure_reason: str
   - An abstract class PaymentGatewayProvider(ABC) with three abstract
     methods:
     create_payment(self, *, payment: "Payment", callback_url: str) -> GatewayCreateResult
     verify_callback(self, *, request_data: dict) -> GatewayVerification
     refund(self, *, payment: "Payment", amount: "Money") -> GatewayRefundResult
   - Use TYPE_CHECKING to import Payment/Money so no circular import
     occurs.

3. Create apps/payments/gateways/internal.py containing:
   A class InternalSimulatedGatewayProvider(PaymentGatewayProvider) that:
   - create_payment: immediately (with no real network call), based on
     the current simulated success rate (_SUCCESS_RATE from
     services.py — move this constant from services.py into this file,
     don't copy it; if any other constant it depends on exists, move
     that too) generates a random gateway_reference (e.g.
     f"internal-{uuid4()}") and stores the outcome (success/failure)
     that was decided at this exact moment — either inside the
     gateway_reference itself or in a short-lived in-memory/cache
     mechanism — so verify_callback can later read back exactly that
     same outcome deterministically, without any external call.
     requires_redirect=False (since this is simulated, no real
     redirect is needed).
   - verify_callback: returns the predetermined outcome.
   - refund: always returns success=True (since there's no real
     gateway behind it, a simulated refund succeeds instantly).
   - This class must also be usable for deterministic testing: add an
     optional constructor parameter force_result: bool | None = None
     that, when set, returns that exact value instead of using
     random.random() — this is needed for writing deterministic tests
     in later prompts.

4. Create a new test file: tests/payments/test_gateways_internal.py
   that:
   - Verifies create_payment with force_result=True returns a
     successful GatewayCreateResult
   - Verifies verify_callback afterward returns success=True with the
     same gateway_reference
   - Repeats the same for force_result=False (failure)
   - Verifies refund with any valid input returns success=True

Files affected (only these):
- apps/payments/gateways/__init__.py (new)
- apps/payments/gateways/base.py (new)
- apps/payments/gateways/internal.py (new)
- tests/payments/test_gateways_internal.py (new)

Do not touch any other file (services.py, models.py, views.py, urls.py) —
wiring happens in later prompts.

Acceptance Criteria:
- pytest tests/payments/test_gateways_internal.py -v passes completely
- git status shows only the new files above

Verification Steps:
1. pytest tests/payments/test_gateways_internal.py -v
2. python -c "from apps.payments.gateways.base import PaymentGatewayProvider; print(PaymentGatewayProvider.__abstractmethods__)"
   and confirm the three abstract methods
3. Run git status and git diff
```

### Prompt 2 — Implement `ZarinpalGatewayProvider`

```
Goal: A real implementation of PaymentGatewayProvider for the Zarinpal
gateway, behind the same interface built in the previous prompt. Not yet
connected to any existing view/service.

Before starting:
1. Read apps/payments/gateways/base.py and
   apps/payments/gateways/internal.py to fully understand the
   interface contract.
2. Read apps/common/money.py — Zarinpal expects the amount in Rial (an
   integer); use Money.to_minor_units or equivalent to ensure the
   correct integer amount is sent (since, per the Phase 0 design,
   Payment.amount is already stored in Rial with no decimals, no
   further conversion should be needed — document this assumption
   with an explicit assertion in the code).
3. Consider the Zarinpal API documentation (or an equivalent chosen
   gateway) for three endpoints: PaymentRequest (create the
   transaction), PaymentVerification (verify), and Refund (if
   supported by the chosen gateway — if the chosen gateway has no
   refund API, document this explicitly in the code and implement the
   appropriate behavior — raise NotImplementedError with a clear
   message, not a silent failure).

What to build:

Create apps/payments/gateways/zarinpal.py containing:

Class ZarinpalGatewayProvider(PaymentGatewayProvider):
- Constructor: takes merchant_id: str from
  settings.ZARINPAL_MERCHANT_ID (as a constructor parameter, not
  hardcoded, so it's testable with a fake value)
- create_payment:
  - Sends an HTTP POST to Zarinpal's payment-request endpoint with
    merchant_id, amount (Rial), callback_url, description
  - Use requests (or httpx — whichever is already present in
    requirements, check first) with an explicit timeout (e.g. 10
    seconds) — never make a network call without an explicit timeout
  - On a successful response: return GatewayCreateResult with
    redirect_url (the Zarinpal gateway URL + Authority),
    gateway_reference=Authority, requires_redirect=True
  - On a network error/HTTP error/invalid gateway response: raise a
    dedicated exception (define a new GatewayError class in base.py)
    instead of letting a raw exception (requests.RequestException)
    leak to the caller — the caller must be able to handle this
    uniformly regardless of which provider is in use.
  - Log every call (success or failure) via logger.info/warning with
    the gateway_reference and HTTP status code — never log any
    sensitive data (e.g. the entire raw payload if it contains
    anything sensitive; for Zarinpal this is usually not an issue, but
    follow this principle rigorously).
- verify_callback:
  - request_data contains Authority and Status (the query parameters
    Zarinpal sends on the callback redirect)
  - If Status != "OK" (per Zarinpal's docs, meaning the user canceled
    the payment), return GatewayVerification(success=False) without
    any network call
  - Otherwise, always make a server-to-server PaymentVerification call
    with merchant_id/Authority/amount — never conclude success based
    solely on the callback's query parameters (which can be forged);
    document this explicitly in the docstring, since it is a serious
    security concern.
  - Map the verification result to GatewayVerification
    (gateway_transaction_id = Zarinpal's RefID on success)
- refund:
  - If the chosen gateway supports a refund API, implement it;
    otherwise raise NotImplementedError("Zarinpal does not support
    automated refunds; process manually via merchant panel.") — and
    document this limitation in the class docstring (this is an
    important point the payments team needs to be aware of).

Add a new exception GatewayError(Exception) in
apps/payments/gateways/base.py (if not already added in the previous
prompt) that both implementations (internal and zarinpal) use for
unexpected gateway-communication errors.

Add new settings in config/settings/base.py (only this prompt adds
them, nothing else uses them yet):
ZARINPAL_MERCHANT_ID: str = config("ZARINPAL_MERCHANT_ID", default="")
ZARINPAL_CALLBACK_BASE_URL: str = config("ZARINPAL_CALLBACK_BASE_URL", default="")
ZARINPAL_API_BASE_URL: str = config("ZARINPAL_API_BASE_URL", default="https://api.zarinpal.com/pg/v4/")
(use the sandbox URL as the default for the dev environment if
Zarinpal's docs have a separate sandbox URL; document in the settings
docstring which one is for production)

Then create a test file: tests/payments/test_gateways_zarinpal.py that
mocks every HTTP call using responses or unittest.mock (no real network
call is allowed in the test) and covers:
- create_payment on a successful gateway response returns the correct
  redirect_url and gateway_reference
- create_payment on an HTTP error response raises GatewayError
- verify_callback with Status != "OK" returns success=False without any
  network call (confirm this by asserting the mock was never called)
- verify_callback with Status == "OK" and a successful verification
  response from the gateway returns success=True and the correct
  gateway_transaction_id
- verify_callback with Status == "OK" but a failed verification
  response from the gateway returns success=False
- refund correctly demonstrates the NotImplementedError behavior (or
  the real implementation, whichever was chosen)

Files affected:
- apps/payments/gateways/zarinpal.py (new)
- apps/payments/gateways/base.py (adding GatewayError, if needed)
- config/settings/base.py (only adding the three new settings)
- tests/payments/test_gateways_zarinpal.py (new)
- requirements/base.txt (only if the needed HTTP library isn't already
  installed)

Acceptance Criteria:
- pytest tests/payments/test_gateways_zarinpal.py -v passes completely
  and makes no real network call
- verify_callback never returns success=True without a server-to-server
  call (this must be confirmed by an explicit test)

Verification Steps:
1. pytest tests/payments/test_gateways_zarinpal.py -v
2. grep -n "requests.get\|requests.post\|httpx" apps/payments/gateways/zarinpal.py
   and manually confirm every call has a timeout
3. pytest tests/payments/ -v (confirm the previous prompt is still
   passing)
4. git diff --stat
```

### Prompt 3 — Factory + Provider-Selection Settings (No Wiring to the Main Path)

```
Goal: Build a single point for selecting the active PaymentGatewayProvider
based on environment settings, replacing the current
settings.PAYMENTS_USE_STRIPE pattern. Not yet connected to the existing
services.py — that's the next prompt.

Before starting:
1. Read apps/payments/gateways/base.py, internal.py, zarinpal.py
2. Read config/settings/base.py, especially the current STRIPE_* section
   and PAYMENTS_USE_STRIPE (lines ~341-365) to understand what's being
   replaced

What to build:

1. Create apps/payments/gateways/factory.py:
   A function get_gateway_provider() -> PaymentGatewayProvider that:
   - Reads settings.PAYMENT_GATEWAY (value "internal" or "zarinpal")
   - For "internal", returns InternalSimulatedGatewayProvider()
   - For "zarinpal", returns
     ZarinpalGatewayProvider(merchant_id=settings.ZARINPAL_MERCHANT_ID)
   - For any other value, raises ImproperlyConfigured (from
     django.core.exceptions) with a clear message listing the allowed
     values
   - Does not cache the result (returns a fresh instance on every
     call) unless the provider is stateless and caching is explicitly
     safe — for simplicity and predictability, do not cache for now.

2. In config/settings/base.py:
   - Add: PAYMENT_GATEWAY: str = config("PAYMENT_GATEWAY", default="internal")
   - Add ZARINPAL_MERCHANT_ID, ZARINPAL_CALLBACK_BASE_URL,
     ZARINPAL_API_BASE_URL (if not already added in the previous
     prompt)
   - Do NOT remove PAYMENTS_USE_STRIPE, STRIPE_SECRET_KEY,
     STRIPE_WEBHOOK_SECRET, STRIPE_CONNECT_REFRESH_URL,
     STRIPE_CONNECT_RETURN_URL yet — just add a comment above them:
     "# DEPRECATED: replaced by PAYMENT_GATEWAY / Zarinpal in Phase 1;
     kept for the legacy STRIPE payout code path until it's fully
     retired." (Their actual removal happens in a later prompt within
     this same Phase, once we're confident nothing still depends on
     them.)

3. Create a test file: tests/payments/test_gateway_factory.py that:
   - With override_settings(PAYMENT_GATEWAY="internal"),
     get_gateway_provider() returns an instance of
     InternalSimulatedGatewayProvider
   - With override_settings(PAYMENT_GATEWAY="zarinpal"), returns an
     instance of ZarinpalGatewayProvider
   - With override_settings(PAYMENT_GATEWAY="something_invalid"),
     raises ImproperlyConfigured

Files affected:
- apps/payments/gateways/factory.py (new)
- config/settings/base.py (only adding/commenting the settings above)
- tests/payments/test_gateway_factory.py (new)

Acceptance Criteria:
- pytest tests/payments/test_gateway_factory.py -v passes completely
- No services.py/views.py file has been changed

Verification Steps:
1. pytest tests/payments/test_gateway_factory.py -v
2. PAYMENT_GATEWAY=internal python manage.py check (no errors)
3. git diff --stat
```

### Prompt 4 — Wire Up the Real Checkout Flow (Redirect + Callback)

```
Goal: Replace the current "booking → fire-and-forget Celery task →
immediate result" flow with a real "booking → explicit checkout request →
redirect → callback → verify" flow. This is the most important and
highest-risk behavioral change in this Phase — proceed with great care,
and do not change any existing state-machine logic (only the entry
point).

Before starting, read these files completely and carefully:
1. apps/payments/services.py — the full file, with special focus on:
   capture_payment_for_booking (91-199), handle_payment_intent_succeeded
   (214-265), handle_payment_intent_failed (272-332),
   _record_payment_event
2. apps/payments/webhooks.py — the full file (to precisely understand
   the structure of StripeWebhookView, which needs to be replaced)
3. apps/payments/views.py — the full file
4. apps/payments/urls.py — the full file
5. apps/bookings/services.py — the create_booking function, especially
   line ~306, which calls capture_payment_for_booking.delay(...)
6. apps/bookings/tasks.py — the reap_stale_pending_booking_requests
   function (line ~154) to understand the current sweep logic for
   stuck bookings
7. apps/payments/gateways/base.py, factory.py (from previous prompts)

What to build:

a) In apps/payments/services.py:

1. Split the capture_payment_for_booking function into two functions:

   create_payment_for_booking(*, booking_id: str) -> Payment:
   - Same as the first part of the current function: check
     booking.status == "PENDING_PAYMENT", create the Payment with
     status="PENDING", record a PaymentEvent — without making an
     immediate success/failure decision (that part is removed from
     here)
   - This function no longer has the random-outcome logic
     (_SUCCESS_RATE/random.random) — that logic now lives in
     InternalSimulatedGatewayProvider (Prompt 1)
   - Stays idempotent: if a Payment already exists for this booking,
     return it (preserve current behavior)

   initiate_payment_checkout(*, payment_id: str) -> GatewayCreateResult:
   - Fetches the Payment with select_for_update, checks status == "PENDING"
   - Uses apps.payments.gateways.factory.get_gateway_provider()
   - Builds callback_url via a helper function (e.g.
     f"{settings.ZARINPAL_CALLBACK_BASE_URL}/api/payments/{payment.id}/callback/{payment.provider.lower()}/")
   - Calls provider.create_payment(payment=payment, callback_url=callback_url)
   - Takes the result: saves gateway_reference onto
     payment.gateway_charge_id (save with update_fields)
   - On a GatewayError, sets payment.status="FAILED" +
     failure_reason, records a PaymentEvent, and re-raises as an
     ApplicationError to the caller (the view) so the user sees an
     appropriate message
   - This function must be atomic (transaction.atomic), following the
     same pattern as similar functions in this file

2. Rename and re-shape handle_payment_intent_succeeded and
   handle_payment_intent_failed into
   handle_gateway_callback_success(*, gateway_reference: str,
   gateway_transaction_id: str | None) and
   handle_gateway_callback_failed(*, gateway_reference: str, reason: str)
   — preserve the internal logic exactly (idempotency, status change,
   _record_payment_event, calling
   notify_payment_received/notify_payment_failed), only:
   - Do the lookup on gateway_charge_id=gateway_reference (instead of
     payment_intent_id — same field, clearer parameter name)
   - Change the success target status from "COMPLETED" to
     "HELD_IN_ESCROW" (since these functions now sit in the main
     escrow path, not the legacy enrollment-based path) — and move the
     subsequent steps that were in the success branch of the current
     capture_payment_for_booking (booking.status =
     "AWAITING_TEACHER_REVIEW", notify_new_booking_request.delay) into
     this function
   - Keep the failure target status as "FAILED", and move
     booking.status = "CANCELLED" (from the failure branch of the
     current capture_payment_for_booking) into this function

3. Add a function verify_and_settle_payment_callback(*, provider_name:
   str, request_data: dict) -> Payment that:
   - Gets the correct provider from the factory (based on provider_name
     from the URL)
   - Calls provider.verify_callback(request_data=request_data)
   - Logs the result in a new PaymentGatewayLog (add this model in this
     same prompt — a simple table: nullable payment FK,
     gateway_reference, event_type=["CREATE","VERIFY","REFUND"], a
     raw_response JSONField, created_at)
   - Based on success, calls handle_gateway_callback_success or
     handle_gateway_callback_failed
   - Returns the final Payment

b) apps/payments/models.py:
   Add the new PaymentGatewayLog model (as described above) + a
   migration.

c) apps/payments/views.py:
   1. Add PaymentCheckoutView (APIView, POST):
      /api/payments/<payment_id>/checkout/ — IsAuthenticated +
      IsPaymentOwner (the existing permission, the same one
      PaymentDetailView uses)
      Calls services.initiate_payment_checkout, returns
      {"redirect_url": ..., "requires_redirect": ...}

   2. Add PaymentGatewayCallbackView (APIView, GET and POST, AllowAny):
      /api/payments/<payment_id>/callback/<provider_name>/
      Calls services.verify_and_settle_payment_callback and redirects
      (HttpResponseRedirect) to a frontend URL (e.g. built from
      settings or a fixed pattern
      f"{FRONTEND_BASE_URL}/payments/callback?status=...") so the user
      sees a UI, not raw JSON — make sure this has
      permission_classes = [AllowAny] and authentication_classes = []
      exactly like the old StripeWebhookView, since the gateway
      doesn't send any user JWT.

   3. Old StripeWebhookView: only remove the
      payment_intent.succeeded/failed handling from it (since it's
      been moved to the new flow); leave charge.dispute.created and
      account.updated untouched for now — these relate to Payout and
      will be addressed in later prompts within this same Phase.

d) apps/payments/urls.py:
   Add routes for PaymentCheckoutView and PaymentGatewayCallbackView.

e) apps/bookings/services.py:
   Change the line in create_booking that calls
   capture_payment_for_booking.delay(...) to
   create_payment_for_booking.delay(...) (the function name has
   changed, but it's still async via Celery — it just no longer decides
   success/failure, it merely prepares the Payment in a PENDING state
   so the user can immediately follow up with a checkout request).

f) apps/payments/tasks.py:
   Update any Celery task that directly called
   capture_payment_for_booking to create_payment_for_booking (rename
   the function reference, leave the task's internal logic untouched).

g) apps/bookings/tasks.py:
   In reap_stale_pending_payment_bookings, update the comments to
   reflect that this sweep now covers both "the Celery task was lost"
   and "the user never went to / returned from the gateway"; check the
   current timeout window and if it's too short (suited to an
   immediate Celery decision, not a real user waiting at the gateway),
   change it to a more reasonable value (e.g. 30 minutes) via a new
   setting settings.PAYMENT_CHECKOUT_TIMEOUT_MINUTES (add this new
   setting to config/settings/base.py too).

h) Before anything else, run a grep to find every reference:
   grep -rn "capture_payment_for_booking" apps/
   and align every match found with the new name.

Files affected:
- apps/payments/services.py
- apps/payments/models.py (+ new migration for PaymentGatewayLog)
- apps/payments/views.py
- apps/payments/urls.py
- apps/payments/webhooks.py
- apps/payments/tasks.py
- apps/bookings/services.py
- apps/bookings/tasks.py
- config/settings/base.py (adding PAYMENT_CHECKOUT_TIMEOUT_MINUTES)

Then run all related existing tests (pytest tests/payments/
tests/bookings/ -v) and update any test that depends on the old
function name or the old "immediate result" behavior to reflect the new
two-step (create, then checkout, then callback) behavior — be sure to
add three new tests:
1. Full happy-path test: create_payment_for_booking →
   initiate_payment_checkout (using InternalSimulatedGatewayProvider
   with force_result=True) → verify_and_settle_payment_callback →
   Payment.status == "HELD_IN_ESCROW" and Booking.status ==
   "AWAITING_TEACHER_REVIEW"
2. The same path with force_result=False → Payment.status == "FAILED"
   and Booking.status == "CANCELLED"
3. An idempotency test: two consecutive calls to
   verify_and_settle_payment_callback with the same gateway_reference —
   the second call must not produce any duplicate side effect (e.g.
   sending notify_payment_received again)

Acceptance Criteria:
- pytest tests/payments/ tests/bookings/ -v passes completely
- grep -rn "capture_payment_for_booking" apps/ returns no results (all
  references renamed)
- The three new tests listed above exist and pass
- python manage.py makemigrations --check --dry-run is empty

Verification Steps:
1. pytest tests/payments/ tests/bookings/ -v
2. grep -rn "capture_payment_for_booking" apps/
3. python manage.py makemigrations --check --dry-run
4. python manage.py migrate
5. git diff --stat (review the full list of changes)
```

### Prompt 5 — Replace Payout: From Stripe Connect to `TeacherPayoutAccount` + `PayoutProvider`

```
Goal: Remove the Stripe Connect dependency from the teacher-payout path
and replace it with an Iranian bank-account model (Shaba/IBAN number) +
an abstract PayoutProvider that, for now, handles payouts manually
(with admin confirmation).

Before starting, read these files completely:
1. apps/payments/services.py — the functions _validate_stripe_release
   (681-709), release_payment_to_teacher (710 through roughly
   870-916, with special focus on the stripe.Transfer.create block
   around lines ~870-914), and the similar section in resolve_dispute
   (around lines ~1297-1360)
2. apps/users/models.py — TeacherProfile, especially
   stripe_account_id, stripe_onboarding_status, and the is_payout_ready
   property
3. apps/teachers/services.py — start_stripe_onboarding (246-291),
   mark_stripe_onboarding_complete/incomplete (296-341)
4. apps/teachers/views.py — StripeOnboardingView,
   StripeOnboardingStatusView
5. apps/teachers/urls.py — the corresponding routes
6. apps/payments/webhooks.py — the _handle_account_updated section

What to build:

a) A new model in apps/users/models.py (or a new file
apps/payments/models.py if that's more logical — decide based on
whether this model belongs more to "teacher" or "payment"; since this
project already follows the pattern of putting the teacher's financial
data — hourly_rate, currency — on apps.users.TeacherProfile, add it
there for consistency):

class TeacherPayoutAccount(models.Model):
    """The teacher's bank account information for receiving payouts —
    replaces Stripe Connect onboarding. For now, transfers to this
    account are handled entirely manually (by an admin); this model
    just holds the information needed for that manual transfer, not an
    automated bank API connection."""

    id = UUIDField(primary_key=True, default=uuid4, editable=False)
    teacher_profile = OneToOneField(TeacherProfile, on_delete=CASCADE,
                                     related_name="payout_account")
    account_holder_name = CharField(max_length=255)
    bank_name = CharField(max_length=100)
    shaba_number = CharField(max_length=26, unique=True)  # IR + 24 digits
    is_verified = BooleanField(default=False)
    verified_by = ForeignKey(User, null=True, blank=True, on_delete=SET_NULL,
                              related_name="verified_payout_accounts")
    verified_at = DateTimeField(null=True, blank=True)
    created_at = DateTimeField(auto_now_add=True)
    updated_at = DateTimeField(auto_now=True)

Add a clean() method that validates shaba_number (must start with "IR"
and have exactly 24 digits after it — a simple regex is enough; full
Iranian IBAN checksum validation is out of scope for this Phase, just
validate the format).

b) Rewrite TeacherProfile.is_payout_ready:
   Change from checking stripe_account_id/stripe_onboarding_status to
   checking:
   hasattr(self, "payout_account") and self.payout_account.is_verified
   Fully update the docstring to explain this no longer relates to
   Stripe at all.

   Do not remove stripe_account_id and stripe_onboarding_status from
   the model (per this Phase's architectural decision — keeping them
   for safe rollback), just stop using them in is_payout_ready. Add a
   DEPRECATED comment above both fields.

c) apps/payments/payouts/ (new folder):
   base.py:
   - A dataclass PayoutInitiationResult(status: str, reference: str |
     None, requires_manual_action: bool)
   - An abstract class PayoutProvider(ABC) with the method
     initiate_payout(self, *, entry: "PayoutLedgerEntry") -> PayoutInitiationResult

   manual.py:
   - A class ManualBankTransferPayoutProvider(PayoutProvider):
     initiate_payout always returns
     PayoutInitiationResult(status="PENDING_TRANSFER", reference=None,
     requires_manual_action=True) — this implementation has no network
     call, it simply marks the entry for the admin's manual follow-up.

   factory.py:
   - A function get_payout_provider() -> PayoutProvider that, for now,
     always returns ManualBankTransferPayoutProvider() (designed for
     future extension if an automated payout provider becomes
     available — document this in the docstring).

d) apps/payments/models.py — the PayoutLedgerEntry class:
   - Add a new field:
     payout_status = CharField(max_length=20, choices=[
         ("PENDING_TRANSFER", "Pending manual transfer"),
         ("TRANSFERRED", "Transferred"),
         ("FAILED", "Failed"),
     ], default="PENDING_TRANSFER", db_index=True)
   - Rename the stripe_transfer_id field to transfer_reference
     (use a migration with RenameField, not AlterField+delete — to
     preserve data) and update its docstring to note it's now a
     generic reference (a manual bank-transfer tracking number, not
     necessarily Stripe).
   - Run makemigrations, review the migration and give it a readable
     name.

e) apps/payments/services.py:
   - The _validate_stripe_release function: generalize its name and
     logic — it should no longer just check provider=="STRIPE";
     instead of validating stripe_account_id/onboarding, it should
     validate TeacherProfile.is_payout_ready (which now works off
     TeacherPayoutAccount) for *any* payment that needs a real payout.
     If is_payout_ready is False, raise the same ApplicationError as
     before but change the code from missing_gateway_charge_id to
     something like "payout_account_not_ready" and update the message.
   - In release_payment_to_teacher: completely remove the
     stripe.api_key = .../stripe.Transfer.create(...) block (lines
     ~870-914) and replace it with:
     from apps.payments.payouts.factory import get_payout_provider
     entry = PayoutLedgerEntry.objects.create(..., payout_status="PENDING_TRANSFER")
     result = get_payout_provider().initiate_payout(entry=entry)
     entry.transfer_reference = result.reference
     entry.payout_status = result.status
     entry.save(update_fields=["transfer_reference", "payout_status"])
     (Do not touch the existing logic that calculates
     gross_amount/commission_amount/payout_amount — only the "how the
     money actually moves" part is replaced)
   - Make the same replacement in resolve_dispute (the similar
     section, around lines ~1297-1360).

f) apps/teachers/services.py:
   - Completely remove start_stripe_onboarding,
     mark_stripe_onboarding_complete, mark_stripe_onboarding_incomplete.
   - Add new functions:
     submit_payout_account(*, teacher_profile, account_holder_name,
     bank_name, shaba_number) -> TeacherPayoutAccount — creates or
     updates (update_or_create on teacher_profile), always resets
     is_verified to False (any change requires re-verification by an
     admin — explain this in the docstring)
     verify_payout_account(*, payout_account_id, admin_user) ->
     TeacherPayoutAccount — staff-only (perform this check in the
     view, not here, following the project's existing pattern of
     putting authorization in the view/permission layer), sets
     is_verified=True, verified_by, verified_at.

g) apps/teachers/views.py, apps/teachers/urls.py:
   - Remove StripeOnboardingView, StripeOnboardingStatusView
   - Add PayoutAccountView (an APIView; GET returns the current status,
     POST/PUT calls submit_payout_account) at
     /api/teachers/me/payout-account/
   - (Admin verification will get a proper endpoint with a full admin
     panel in Phase 8; for now it just needs to be doable via plain
     Django Admin — that's added to admin.py in the next prompt of
     this Phase)

h) apps/payments/webhooks.py:
   - Completely remove _handle_account_updated and its reference in
     post() (since Stripe Connect no longer exists). If removing this
     function makes the charge.dispute.created section meaningless
     too (since no new Payment will ever get provider="STRIPE" going
     forward), leave that section untouched but add a comment: "legacy:
     only relevant to historical provider=STRIPE payments, if any
     exist."

Files affected:
- apps/users/models.py (+ migration)
- apps/payments/payouts/base.py, manual.py, factory.py (new)
- apps/payments/models.py (+ migration)
- apps/payments/services.py
- apps/teachers/services.py
- apps/teachers/views.py
- apps/teachers/urls.py
- apps/payments/webhooks.py
- apps/payments/admin.py (if TeacherPayoutAccount needs initial
  registration in admin so it can at least be viewed/verified through
  the default Django Admin)

Then update/add all related tests, including:
- A test for release_payment_to_teacher with a verified
  TeacherPayoutAccount → a PayoutLedgerEntry with
  payout_status="PENDING_TRANSFER" is created
- A test for release_payment_to_teacher when TeacherPayoutAccount is
  not verified → ApplicationError with code payout_account_not_ready
- Tests for submit_payout_account and verify_payout_account
- Any existing test that assumed
  stripe_account_id/stripe_onboarding_status or
  start_stripe_onboarding — remove or update (run grep -rn "stripe"
  tests/ to find every occurrence)

Acceptance Criteria:
- pytest -x (full backend suite) passes
- grep -rn "stripe.Transfer\|stripe.Account\|stripe.AccountLink" apps/
  returns no results
- TeacherProfile.is_payout_ready no longer depends on any stripe field
- python manage.py makemigrations --check --dry-run is empty

Verification Steps:
1. pytest -x -v
2. grep -rn "stripe.Transfer\|stripe.Account\|stripe.AccountLink" apps/
3. grep -rln "stripe" apps/*/services.py apps/*/views.py (manually
   review whatever remains — it should only appear in the designated
   gateway/legacy files)
4. python manage.py makemigrations --check --dry-run
5. python manage.py migrate
6. git diff --stat
```

### Prompt 6 — Fix `refund_payment` to Call the Real Gateway + Remove Hardcoded Quantization

```
Goal: Add a real call to PaymentGatewayProvider.refund inside the
refund_payment function (which today only marks the refund internally,
with no gateway call), and fix the hardcoded
quantize(Decimal("0.01")) which is inconsistent with the Money
architecture (Phase 0).

Before starting:
1. apps/payments/services.py — read the refund_payment function in full
   (362-445), especially the line:
   refund_amount = (payment.amount * Decimal(refund_percentage) / Decimal(100)).quantize(Decimal("0.01"))
2. Read apps/common/money.py (quantize_amount)
3. Read apps/payments/gateways/base.py, factory.py

What to build:

In apps/payments/services.py, edit the refund_payment function:

1. Replace the quantize(Decimal("0.01")) line with
   apps.common.money.quantize_amount(raw_refund_amount, payment.currency)
   (add the import at the top of the file).

2. After computing refund_amount and before finally committing
   status="REFUNDED", add the gateway call:
   from apps.payments.gateways.factory import get_gateway_provider
   from apps.common.money import Money
   provider = get_gateway_provider()
   try:
       result = provider.refund(
           payment=payment,
           amount=Money(amount=refund_amount, currency=payment.currency),
       )
   except NotImplementedError:
       # The chosen gateway doesn't support automated refunds — treat
       # this as a "manual refund required" state, not a full
       # operation failure: payment.status = "REFUNDED" is still
       # recorded (since, from a business standpoint, the platform's
       # commitment to refund is final) but the reason in
       # PaymentEvent must explicitly say that the actual refund at
       # the gateway needs to be processed manually, and a
       # high-visibility warning log (logger.warning) must be recorded
       # for the finance/support team to see.
       result = None
   except GatewayError as exc:
       # If the chosen gateway genuinely does support this but this
       # particular call failed, this must stop the entire
       # refund_payment operation (raise ApplicationError) — because
       # in this case (unlike NotImplementedError), we expect the
       # automated refund to work, and its failure is a real error
       # that must not be silently swallowed.
       raise ApplicationError(
           "Refund could not be processed by the payment gateway. "
           "Please try again or contact support.",
           code="gateway_refund_failed",
       ) from exc

   If result is present (i.e. the gateway actually supported it and
   succeeded), adjust the reason text in _record_payment_event to also
   include gateway_refund_id (for traceability).

Do not make any other change to refund_payment's existing logic
(ownership check, allowed-status check, computing refund_percentage
from get_refund_percentage).

Files affected:
- apps/payments/services.py (only the refund_payment function)

Then run the existing refund_payment tests and add/update the
following:
- A test: with InternalSimulatedGatewayProvider (whose refund always
  succeeds), refund_payment behaves as before and status="REFUNDED"
- A test: mock provider.refund to raise GatewayError → expect
  ApplicationError with code gateway_refund_failed, and Payment.status
  must remain unchanged (not REFUNDED) — i.e. the transaction must be
  rolled back (check whether the gateway call happens inside the same
  existing transaction.atomic or must be outside it; make the correct
  decision and document it: since a network call should not hold a
  database lock (select_for_update) open for a long time, if the
  current refund_payment architecture uses select_for_update, take
  that into account and explain why calling the gateway before or
  after the commit is more appropriate.)
- A test: mock provider.refund to raise NotImplementedError → expect
  status="REFUNDED" (the platform's commitment is preserved) and a
  PaymentEvent whose reason mentions that a manual refund is required

Acceptance Criteria:
- pytest tests/payments/ -v passes completely
- No hardcoded Decimal("0.01") remains in refund_payment
- All three refund scenarios (automated success / gateway failure /
  gateway unsupported) each have a separate, passing test

Verification Steps:
1. pytest tests/payments/ -v -k refund
2. grep -n "0.01" apps/payments/services.py (should no longer appear in
   refund_payment)
3. git diff apps/payments/services.py
```

### Prompt 7 — Frontend: Checkout Redirect Flow

```
Goal: Build the real user-facing payment flow in the frontend: after a
booking is created, the user must explicitly click "Pay," be redirected
to the gateway, and then see the result after returning. This is an
entirely new capability (since no checkout UI has existed in the
frontend until now).

Before starting:
1. Fully review src/features/bookings/ (pages, components, api, hooks)
   to get a precise understanding of the current "create a booking"
   flow — especially where the user is currently sent after a booking
   is submitted
2. Read src/features/payments/api/payments.api.ts to understand the
   existing pattern of API calls (axios/fetch wrapper, error handling)
3. Read src/app/routes/router.tsx and src/app/routes/lazyPages.ts to
   understand the existing pattern for adding a new route
4. Backend note: POST /api/payments/<payment_id>/checkout/ returns
   {"redirect_url": string, "requires_redirect": boolean}, and after
   returning from the gateway, the backend itself redirects the user to
   a frontend URL (per the backend Phase 1 design, something like
   /payments/callback?status=success or ?status=failed).

What to build:

1. In src/features/payments/api/payments.api.ts:
   Add a function initiateCheckout(paymentId: string): Promise<{redirect_url:
   string | null, requires_redirect: boolean}> that calls
   POST /api/payments/{paymentId}/checkout/ (using the same axios
   instance/pattern as the other functions in this file).

2. Build a new component/page (choose the exact path based on the
   existing structure of src/features/bookings/pages/, e.g.
   BookingPaymentPage.tsx) that:
   - After the booking is created (and its associated Payment — check
     what the booking-creation API response returns about the payment;
     if payment_id isn't in the response, report this as a finding and
     use a separate call to fetch the payment associated with the
     booking instead)
   - Has a clear "Pay" button (this can stay in English for now since
     Persian localization is Phase 5 — just build the structure)
   - On click, calls initiateCheckout
   - If requires_redirect === true and redirect_url is present:
     window.location.href = redirect_url (a full browser redirect, not
     client-side routing — since the gateway is an external domain)
   - If requires_redirect === false (the internal simulated case, no
     real redirect needed): directly route the user to the result page
     (the same PaymentCallbackPage built in the next step) with
     appropriate state (since in the simulated case, there's no need
     to go to an external domain)
   - Implement loading and error states following the same pattern used
     by other components in this project (check whether react-query's
     useMutation is used — likely yes, given the project's stack)

3. Build a new page src/features/payments/pages/PaymentCallbackPage.tsx
   that handles the /payments/callback route:
   - Reads the status query param (from the URL the backend redirected
     the user to after verification)
   - If status === "success", show a success message + a "View My
     Booking" button (linking to the booking detail page, if its ID is
     also present in the query param)
   - If status === "failed", show a failure message + a "Try Again"
     button (returning the user to BookingPaymentPage for the same
     booking)
   - This page is simple state, no extra API call is strictly needed
     since the backend has already put the result in the URL (if the
     backend design isn't like this — i.e. if the backend only sends a
     generic status without enough detail — also add a GET call for
     the payment's current status so the page confirms the result from
     an authoritative source, not merely a query param that could be
     tampered with; prefer this approach since it's safer — a user must
     not be able to reach status=success by editing the URL without a
     payment that actually succeeded)

4. In src/app/routes/router.tsx and lazyPages.ts:
   Add routes for /bookings/:id/payment (or a path matching the
   existing structure) and /payments/callback, with lazy loading
   exactly matching the pattern used by the rest of this file's pages.

Then build new component tests (using the same testing pattern already
used in the project — check whether React Testing Library + MSW are
used):
- BookingPaymentPage: clicking the pay button → initiateCheckout is
  called → window.location.href is set (mock window.location)
- PaymentCallbackPage: with status=success in the URL, the success
  message is displayed; with status=failed, the failure message and
  retry button are displayed

Files affected:
- src/features/payments/api/payments.api.ts
- src/features/bookings/pages/BookingPaymentPage.tsx (new, or a name
  matching the project's existing structure)
- src/features/payments/pages/PaymentCallbackPage.tsx (new)
- src/app/routes/router.tsx
- src/app/routes/lazyPages.ts
- corresponding test files (new)
- src/test/mocks/handlers/*.ts (if MSW is used, add a new handler for
  POST /api/payments/:id/checkout/)

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build runs without type errors
- The full flow (manually, in a dev environment with
  PAYMENT_GATEWAY=internal) can be completed from booking creation to
  seeing the success message

Verification Steps:
1. npx vitest run
2. npm run build
3. Manual run: bring up the backend with PAYMENT_GATEWAY=internal,
   create a booking, walk through the payment flow, and confirm the
   Payment eventually becomes HELD_IN_ESCROW (via Django Admin or the
   API)
4. git diff --stat
```

### Prompt 8 — Frontend: Replace Stripe Connect Onboarding with a Bank Account Form

```
Goal: Fully remove the Stripe Connect onboarding UI for teachers and
replace it with a form for entering Shaba/bank account information,
matching the new backend API (/api/teachers/me/payout-account/ from
Prompt 5).

Before starting:
1. Read src/features/teachers/components/StripeOnboardingBanner.tsx in
   full
2. Read src/features/teachers/pages/TeacherStripeReturnPage.tsx and
   TeacherStripeRefreshPage.tsx
3. Read src/features/teachers/api/teachers.api.ts (the existing stripe
   functions)
4. Read src/features/teachers/types/teacher.types.ts (the stripe fields
   in the types)
5. Read src/features/teachers/pages/TeacherDashboardPage.tsx (where
   StripeOnboardingBanner is used)
6. Read src/features/teachers/hooks/useTeacherProfile.ts

What to build:

1. In src/features/teachers/api/teachers.api.ts:
   - Remove the stripe-related functions (starting onboarding, getting
     status)
   - Add: getPayoutAccount(): Promise<PayoutAccount | null>
     (GET /api/teachers/me/payout-account/)
     submitPayoutAccount(data: {account_holder_name, bank_name,
     shaba_number}): Promise<PayoutAccount>
     (POST or PUT to the same path — matching what was implemented on
     the backend in Prompt 5; if unsure, first check
     apps/teachers/urls.py and apps/teachers/views.py on the backend to
     confirm the exact method)

2. In src/features/teachers/types/teacher.types.ts:
   - Remove the stripe_account_id and stripe_onboarding_status fields
     from the TeacherProfile-related type
   - Add a new type:
     interface PayoutAccount {
       id: string;
       account_holder_name: string;
       bank_name: string;
       shaba_number: string;
       is_verified: boolean;
     }

3. Remove
   src/features/teachers/components/StripeOnboardingBanner.tsx;
   replace it with PayoutAccountBanner.tsx that:
   - Uses useQuery to fetch the current payout account status
   - If no account has been registered yet, or is_verified === false,
     shows a banner with an appropriate message ("Please provide/complete
     your bank account information to receive payouts" or "Your bank
     account is pending verification") + a link to the form
   - If is_verified === true, shows nothing (or a small verified badge,
     per the project's existing visual pattern — look at similar
     existing components in the project for the visual style)

4. Build a new page
   src/features/teachers/pages/TeacherPayoutAccountPage.tsx:
   - A simple form with three fields (account_holder_name, bank_name,
     shaba_number)
   - Basic client-side validation for shaba_number (must start with
     "IR" and have 24 digits — the same rule the backend also checks;
     use an existing validation library in the project if one exists
     (zod/yup or similar — check first), otherwise write a simple
     validation function)
   - Submit → submitPayoutAccount → show a success message ("Submitted,
     pending admin verification")

5. Completely remove TeacherStripeReturnPage.tsx and
   TeacherStripeRefreshPage.tsx; also remove every reference to them in
   router.tsx/lazyPages.ts.

6. In TeacherDashboardPage.tsx, replace the reference to
   StripeOnboardingBanner with PayoutAccountBanner.

7. router.tsx/lazyPages.ts: add a new route for
   TeacherPayoutAccountPage (e.g. /teacher/payout-account), remove the
   stripe-return/stripe-refresh routes.

8. src/test/mocks/handlers/teacher.handlers.ts and
   src/test/mocks/fixtures/teacher.fixtures.ts:
   - Remove the stripe handlers/fixtures
   - Add handlers/fixtures for GET and POST payout-account

Then remove/update the following test files:
- src/features/teachers/components/__tests__/StripeOnboardingBanner.test.tsx
  → remove, replace with PayoutAccountBanner.test.tsx (covering three
  states: no account / unverified account / verified account)
- Any other test (grep -rln "stripe\|Stripe" src/ to find them all)
  still referencing the old flow — fix it

Files affected: per the full list above

Acceptance Criteria:
- grep -rln "stripe\|Stripe" src/ (case-insensitive) returns no results
- npx vitest run passes the entire frontend test suite
- npm run build succeeds without errors

Verification Steps:
1. grep -rln "stripe\|Stripe" src/
2. npx vitest run
3. npm run build
4. git diff --stat
```

### Prompt 9 — Final Cleanup, End-to-End Integration Test, and Documentation

```
Goal: Do a final review of the entire Phase 1 to ensure no forgotten
Stripe references remain (except ones deliberately kept as legacy), add
a full end-to-end integration test, and update the documentation.

Before starting:
1. Review the full diff produced across Prompts 1-8 of this same Phase
   (via git log --oneline, or by reviewing the changed files) to get
   the complete picture of the changes.
2. Check apps/docs/ (if the project has a docs folder) for any document
   that discusses payments/Stripe.

What to build:

1. Comprehensive search:
   grep -rln "stripe\|Stripe\|STRIPE" afra-backend afra-frontend
   --include=*.py --include=*.ts --include=*.tsx --include=*.md
   | grep -v node_modules | grep -v migrations

   Manually review every result and sort it into one of two categories:
   - "Deliberately kept as legacy" (e.g. the DEPRECATED comments added
     in models.py in Prompts 3 and 5, or the
     stripe_account_id/stripe_onboarding_status fields themselves,
     which were intentionally not removed) — leave these alone.
   - "Forgotten" (active code, outdated documentation, or unused
     settings) — clean these up.

2. requirements/base.txt: check whether the stripe package is still
   imported anywhere (grep -rn "^import stripe\|^from stripe" apps/).
   If no active import remains (only existed in removed files), remove
   the stripe package from requirements. If at least one active import
   remains (e.g. the legacy charge.dispute.created webhook section
   deliberately kept in Prompt 4), keep the package and document the
   reason in a comment above the relevant line in requirements.

3. Build a new end-to-end integration test:
   tests/payments/test_e2e_iranian_gateway_flow.py that covers the
   entire path using InternalSimulatedGatewayProvider (with
   force_result=True), start to finish:
   create a student and a teacher (with a verified
   TeacherPayoutAccount) → create_booking →
   create_payment_for_booking → initiate_payment_checkout →
   verify_and_settle_payment_callback → verify Payment.status ==
   HELD_IN_ESCROW and Booking.status == AWAITING_TEACHER_REVIEW →
   accept_booking (if such a function exists in
   apps.bookings.services) → mark the session as COMPLETED (via the
   appropriate existing function/fixture) →
   release_payment_to_teacher → verify a PayoutLedgerEntry with
   payout_status == PENDING_TRANSFER was created and that
   commission_amount + payout_amount == amount

4. Documentation:
   If any file such as afra-backend/docs/ or a README describes
   payments/Stripe, update that section to reflect the new
   architecture (the abstract PaymentGatewayProvider, Zarinpal,
   ManualBankTransferPayoutProvider) — concisely and accurately, not a
   full rewrite of the document.

5. config/settings/base.py: now that all of Phase 1 is complete, check
   whether any active code still depends on STRIPE_SECRET_KEY,
   STRIPE_WEBHOOK_SECRET, PAYMENTS_USE_STRIPE,
   STRIPE_CONNECT_REFRESH_URL, STRIPE_CONNECT_RETURN_URL (grep for
   them). If truly no active reference remains, remove these settings
   entirely (not just a DEPRECATED comment) and remove their
   counterparts from .env.example (if it exists) and from GitHub
   Actions secrets/workflow files. If even one active reference
   remains, leave it alone and list that reference in your final
   report so the user can decide.

Files affected: varies based on the search results — record the exact
list before making changes and report it in the final response.

Acceptance Criteria:
- pytest -x (full backend) passes
- npx vitest run (full frontend) passes
- The new e2e test exists and passes
- The final grep for stripe/Stripe shows only entries that were
  deliberately kept as legacy (list these entries in the output)

Verification Steps:
1. pytest -x -v
2. npx vitest run
3. grep -rln "stripe\|Stripe\|STRIPE" afra-backend afra-frontend
   --include=*.py --include=*.ts --include=*.tsx | grep -v node_modules
   | grep -v migrations
   (paste the output of this command into the final report so it's
   clear exactly what was intentionally kept)
4. python manage.py makemigrations --check --dry-run
5. npm run build
6. git diff --stat (final summary of the entire Phase 1)
```

# Phase 2 — Teacher Verification & Marketplace Gating

## Implementation Prompts

### Prompt 1 — مدل‌ها + Migration (بدون منطق سرویس یا فیلترینگ)

```
Goal: Add the TeacherVerification, TeacherDocument, and
TeacherVerificationEvent models to the Afra project, plus their
migration. No service logic, view, or marketplace filtering is added in
this prompt.

Before starting, read these files in full:
1. apps/users/models.py — the TeacherProfile class (to understand how
   it relates to User, and how avatar/ImageField is currently used, so
   TeacherDocument's FileField follows the same pattern)
2. apps/payments/models.py — the Payment and PaymentEvent classes
   (specifically their append-only, from_status/to_status/actor/reason
   event-log pattern) — TeacherVerificationEvent must follow this exact
   same pattern, not invent a new one
3. apps/teachers/models.py — the existing TeacherSkill and
   AvailabilitySlot classes, to match the project's existing style
   (UUIDField primary keys, Meta ordering, __str__ methods, docstring
   conventions)

What to build:

In apps/teachers/models.py, add three new models:

1. TeacherVerification:
   id = UUIDField(primary_key=True, default=uuid4, editable=False)
   teacher_profile = OneToOneField("users.TeacherProfile", on_delete=CASCADE,
                                    related_name="verification")
   STATUS = [
       ("PENDING", "Pending"),
       ("UNDER_REVIEW", "Under review"),
       ("APPROVED", "Approved"),
       ("REJECTED", "Rejected"),
       ("SUSPENDED", "Suspended"),
   ]
   status = CharField(max_length=15, choices=STATUS, default="PENDING", db_index=True)
   submitted_at = DateTimeField(null=True, blank=True)
   reviewed_at = DateTimeField(null=True, blank=True)
   reviewed_by = ForeignKey(User, null=True, blank=True, on_delete=SET_NULL,
                             related_name="reviewed_teacher_verifications")
   rejection_reason = TextField(blank=True)
   suspension_reason = TextField(blank=True)
   created_at = DateTimeField(auto_now_add=True)
   updated_at = DateTimeField(auto_now=True)

   Write a thorough class docstring explaining the state machine
   (PENDING -> UNDER_REVIEW -> APPROVED/REJECTED, APPROVED <->
   SUSPENDED), and explicitly note: "status defaults to PENDING for
   every new row — a teacher is invisible in the marketplace until an
   admin explicitly approves them; see
   apps.teachers.selectors.list_teachers for where this is enforced."

2. TeacherDocument:
   id = UUIDField(primary_key=True, default=uuid4, editable=False)
   verification = ForeignKey(TeacherVerification, on_delete=CASCADE,
                              related_name="documents")
   DOCUMENT_TYPE = [
       ("NATIONAL_ID", "National ID"),
       ("DEGREE", "Academic degree"),
       ("CERTIFICATE", "Teaching certificate"),
       ("OTHER", "Other"),
   ]
   document_type = CharField(max_length=15, choices=DOCUMENT_TYPE)
   file = FileField(upload_to="teacher_verification_documents/")
   uploaded_at = DateTimeField(auto_now_add=True)

   Note in the docstring that this directory is separate from
   avatars/ specifically so that access restrictions can be applied to
   it independently in a later security-hardening phase (do not
   implement any access restriction here — just note it).

3. TeacherVerificationEvent:
   id = UUIDField(primary_key=True, default=uuid4, editable=False)
   verification = ForeignKey(TeacherVerification, on_delete=CASCADE,
                              related_name="events")
   from_status = CharField(max_length=15, choices=TeacherVerification.STATUS, blank=True)
   to_status = CharField(max_length=15, choices=TeacherVerification.STATUS)
   actor = ForeignKey(User, null=True, blank=True, on_delete=SET_NULL,
                       related_name="teacher_verification_events")
   reason = TextField(blank=True)
   created_at = DateTimeField(auto_now_add=True)

   Meta: ordering = ["created_at"], with an index on
   ["verification", "created_at"] — mirror PaymentEvent's Meta exactly.

Add sensible Meta (verbose_name, ordering, indexes on
["teacher_profile", "status"] for TeacherVerification's own lookup
pattern) and __str__ methods for all three, following this project's
existing style in apps/teachers/models.py.

Then run:
python manage.py makemigrations teachers

Review the migration and give it a readable name (e.g.
0004_teacher_verification.py). In the migration's docstring/comment,
note that a follow-up data migration (next prompt) is required to
backfill TeacherVerification rows for any TeacherProfile that predates
this migration.

Files affected:
- apps/teachers/models.py
- apps/teachers/migrations/00xx_teacher_verification.py (new)

Do not touch services.py, selectors.py, views.py, urls.py, or admin.py
in this prompt.

Acceptance Criteria:
- python manage.py makemigrations --check --dry-run is empty
- python manage.py migrate runs cleanly
- git status shows only apps/teachers/models.py and the new migration

Verification Steps:
1. python manage.py makemigrations --check --dry-run
2. python manage.py migrate
3. python manage.py shell -c "from apps.teachers.models import TeacherVerification; print(TeacherVerification.STATUS)"
4. git diff --stat
```

### Prompt 2 — Data Migration (backfill) برای مدرس‌های موجود

```
Goal: Add a data migration that ensures every existing TeacherProfile
row (if any exist in the database at the time this migration runs) gets
a corresponding TeacherVerification row, so no existing teacher profile
is left without one after this Phase ships.

Before starting:
1. Read apps/teachers/migrations/00xx_teacher_verification.py (from the
   previous prompt) to know the exact migration this depends on.
2. Read apps/users/models.py's TeacherProfile class again to confirm
   its exact app label/model name for the migration's dependency graph.

What to build:

Create a new migration file (via
python manage.py makemigrations teachers --empty --name backfill_teacher_verification)
and implement it as a data migration:

- A forward function backfill_teacher_verification(apps, schema_editor)
  that:
  - Gets the historical TeacherProfile and TeacherVerification models
    via apps.get_model(...) (never import apps.teachers.models or
    apps.users.models directly in a migration — always use the
    historical model to stay safe against future field changes)
  - For every TeacherProfile that has no related TeacherVerification
    (use TeacherProfile.objects.filter(verification__isnull=True)),
    create a TeacherVerification with status="PENDING" explicitly.
  - IMPORTANT: create it with status="PENDING", not "APPROVED" — this
    is a deliberate safety-first decision documented in this project's
    Phase 2 architecture: pre-existing teacher profiles must be
    reviewed like any other, not grandfathered in. Write this
    reasoning as a comment directly above the status="PENDING" line so
    it's not mistaken for an oversight.
- A reverse function that deletes only the TeacherVerification rows
  this migration itself created (safe partial reversibility — do not
  attempt to delete verification rows a human might have created or
  modified after this migration ran; if that distinction can't be made
  safely, make the reverse a no-op and document why with a comment).
- Set dependencies in the migration to depend on the
  teacher_verification migration from the previous prompt.

Files affected:
- apps/teachers/migrations/00xx_backfill_teacher_verification.py (new)

Do not touch any other file.

Acceptance Criteria:
- python manage.py migrate runs cleanly on both an empty database and
  one with pre-existing TeacherProfile rows (test both scenarios)
- Every TeacherProfile row has exactly one TeacherVerification row
  after migrating
- python manage.py makemigrations --check --dry-run is empty

Verification Steps:
1. python manage.py migrate
2. python manage.py shell -c "
   from apps.users.models import TeacherProfile
   from apps.teachers.models import TeacherVerification
   assert TeacherProfile.objects.filter(verification__isnull=True).count() == 0
   print('OK: every TeacherProfile has a verification row')
   "
3. (If test fixtures create a TeacherProfile in tests/factories/, run
   pytest tests/teachers/ -v to confirm nothing breaks — some existing
   tests may need the new TeacherVerificationFactory from Prompt 4;
   note any failures here without fixing them yet, they'll be
   addressed in that prompt)
4. git diff --stat
```

### Prompt 3 — سرویس‌های State Machine + Selector Gating

```
Goal: Implement the service-layer functions that drive the
TeacherVerification state machine, and enforce marketplace/booking
gating based on verification status. This is the core behavioral
change of this Phase — read carefully before editing.

Before starting, read these files completely:
1. apps/teachers/models.py (from Prompts 1-2) — the three new models
2. apps/payments/services.py — the _record_payment_event function
   (~line 52) and capture_payment_for_booking's use of
   select_for_update inside transaction.atomic — TeacherVerification's
   equivalent functions must follow this exact same concurrency-safety
   pattern
3. apps/teachers/selectors.py — the full file, especially list_teachers
   (the queryset and its docstring) and get_teacher_public_profile
4. apps/bookings/services.py — the create_booking function, specifically
   line ~99: User.objects.get(id=teacher_id, role="TEACHER", is_active=True)
   — read the full function around it to understand the surrounding
   error handling and what exception type is raised and why
5. apps/common/exceptions.py — to see the exact NotFoundError,
   ConflictError, ApplicationError classes available and their exact
   constructor signatures

What to build:

a) In apps/teachers/services.py, add:

1. def ensure_verification_exists(*, teacher_profile: TeacherProfile) -> TeacherVerification:
   get_or_create(teacher_profile=teacher_profile) — defensive helper for
   any TeacherProfile that might not have one yet (mirrors this
   project's existing lazy-create pattern seen in
   update_teacher_profile/upload_avatar for TeacherProfile itself).

2. def _record_verification_event(*, verification, from_status, to_status,
   actor=None, reason=""):
   Create one TeacherVerificationEvent row. Copy the exact structure of
   apps.payments.services._record_payment_event (same parameter shape,
   same simplicity — no need to reinvent this).

3. def submit_verification_documents(*, teacher_profile, documents: list[dict]) -> TeacherVerification:
   documents is a list of {"document_type": str, "file": UploadedFile}.
   - Use transaction.atomic() + select_for_update() on the
     TeacherVerification row (via ensure_verification_exists first,
     then re-fetch with select_for_update inside the same atomic
     block, matching the concurrency pattern of
     capture_payment_for_booking)
   - Only allowed from status PENDING or REJECTED — raise
     ApplicationError (code "invalid_verification_status") for any
     other current status (e.g. can't resubmit while already
     UNDER_REVIEW or APPROVED)
   - Create a TeacherDocument for each entry in documents
   - Set status="UNDER_REVIEW", submitted_at=timezone.now(),
     rejection_reason="" (clear any previous rejection reason on
     resubmission)
   - Call _record_verification_event(from_status=old, to_status="UNDER_REVIEW")
   - transaction.on_commit: dispatch a Celery task
     notify_verification_submitted.delay(...) (task itself built in
     Prompt 6 — for now, wrap this call in a try/except ImportError or
     simply reference it by string path via apps.teachers.tasks import
     inside the function, exactly like this project's existing
     lazy-import convention seen throughout services.py; if the task
     module doesn't exist yet, this call will fail at runtime, not at
     import time — that's acceptable since Prompt 6 adds it before this
     is ever exercised in production, but note this ordering
     explicitly in a comment)

4. def approve_teacher(*, verification_id: str, admin_user: User) -> TeacherVerification:
   - select_for_update inside transaction.atomic
   - Only allowed from UNDER_REVIEW (raise ApplicationError otherwise,
     code "invalid_verification_status")
   - status="APPROVED", reviewed_at=timezone.now(), reviewed_by=admin_user
   - _record_verification_event(..., actor=admin_user)
   - transaction.on_commit: notify_verification_approved.delay(...)

5. def reject_teacher(*, verification_id: str, admin_user: User, reason: str) -> TeacherVerification:
   - Same concurrency pattern
   - Only allowed from UNDER_REVIEW
   - status="REJECTED", rejection_reason=reason, reviewed_at, reviewed_by
   - _record_verification_event(..., reason=reason, actor=admin_user)
   - transaction.on_commit: notify_verification_rejected.delay(...)

6. def suspend_teacher(*, verification_id: str, admin_user: User, reason: str) -> TeacherVerification:
   - Only allowed from APPROVED
   - status="SUSPENDED", suspension_reason=reason
   - _record_verification_event(..., reason=reason, actor=admin_user)
   - transaction.on_commit: notify_verification_suspended.delay(...)
   - In the docstring, explicitly document the known limitation from
     this Phase's architecture: suspending a teacher does NOT cancel
     or otherwise affect their existing ACCEPTED bookings or in-progress
     sessions — it only removes them from future marketplace visibility
     and blocks new bookings. Say this is a deliberate scope boundary
     for this Phase, not an oversight.

7. def unsuspend_teacher(*, verification_id: str, admin_user: User) -> TeacherVerification:
   - Only allowed from SUSPENDED
   - status="APPROVED", suspension_reason=""
   - _record_verification_event(..., actor=admin_user)

For every one of the above transition functions, if verification_id
doesn't match any row, raise NotFoundError("Teacher verification not
found."), matching this project's existing convention.

b) In apps/teachers/selectors.py:

1. Modify list_teachers: add
   teacher_profile__verification__status="APPROVED" to the base
   queryset filter (alongside the existing role=TEACHER, is_active=True,
   teacher_profile__isnull=False). Update the function's docstring to
   mention this new condition and why (marketplace gating, Phase 2).

2. Modify get_teacher_public_profile: add the same
   verification__status="APPROVED" condition to its lookup. If the
   profile exists but isn't approved, this must raise the exact same
   NotFoundError("Teacher not found.") as when the profile genuinely
   doesn't exist — write this explicitly as a comment: "Deliberately
   identical to the not-found case, to avoid leaking whether a given
   teacher id exists but is merely unapproved (user enumeration
   prevention)."

c) In apps/bookings/services.py:

Modify the User.objects.get(id=teacher_id, role="TEACHER",
is_active=True) call (around line 99) to add
teacher_profile__verification__status="APPROVED" to the filter. Confirm
the exception this already raises on no-match is NotFoundError (not a
new, differently-worded error) — if it currently raises something else,
keep that same exception type/message so the behavior is
indistinguishable from "teacher doesn't exist" (same reasoning as
get_teacher_public_profile above).

Files affected:
- apps/teachers/services.py
- apps/teachers/selectors.py
- apps/bookings/services.py

Then write new tests:
- tests/teachers/test_verification_services.py covering every
  transition function above (happy path + invalid-status rejection for
  each), plus a concurrency test: two simultaneous calls to
  approve_teacher on the same verification (using threads or a
  transaction-isolation test helper already used elsewhere in this
  project's test suite — check tests/bookings/test_concurrency.py for
  the existing pattern and follow it) only one should succeed cleanly,
  the other should see the already-changed status and raise
  invalid_verification_status.
- Update tests/teachers/test_selectors.py (or wherever list_teachers is
  currently tested): add a test that a PENDING teacher does NOT appear
  in list_teachers results, and one that an APPROVED teacher does.
- Update tests/bookings/ tests for create_booking: add a test that
  booking a PENDING/UNDER_REVIEW/REJECTED/SUSPENDED teacher raises
  NotFoundError with the same message as booking a nonexistent teacher
  id.

Note: many existing tests across the project's suite implicitly assume
every teacher fixture is bookable/listable. Running the full suite
after this change will likely surface several failures — do NOT fix
them in this prompt (that's Prompt 4, which builds the
TeacherVerificationFactory); just run pytest tests/teachers/
tests/bookings/ -v -k verification (or similarly scoped) to confirm the
new tests pass, and separately run the full pytest -x to record (not
fix) what else now fails.

Acceptance Criteria:
- The new verification-specific tests all pass
- list_teachers and get_teacher_public_profile correctly exclude
  non-APPROVED teachers
- create_booking correctly rejects booking a non-APPROVED teacher with
  the same error as a nonexistent teacher

Verification Steps:
1. pytest tests/teachers/test_verification_services.py -v
2. pytest tests/teachers/ tests/bookings/ -v -k "verif or approved or pending"
3. pytest -x (run the full suite and record, in your final response,
   the list of now-failing tests — do not fix them here)
4. git diff --stat
```

### Prompt 4 — تست Factory برای Verification + اصلاح تست‌های شکسته‌شده

```
Goal: Add a TeacherVerificationFactory (defaulting to APPROVED, so the
rest of the project's existing test suite — which implicitly assumes
every teacher fixture is bookable/listable — keeps working without
individually rewriting dozens of unrelated tests), wire it into the
existing TeacherProfileFactory, and fix every test that Prompt 3's
gating broke.

Before starting:
1. Read tests/factories/user_factories.py in full — specifically
   TeacherProfileFactory (or wherever TeacherProfile is factory-built)
2. Run pytest -x from the previous prompt's verification step and get
   the exact list of currently-failing tests (if you don't have this
   from context, run it now)
3. Read apps/teachers/models.py to confirm the exact field names on
   TeacherVerification

What to build:

1. In tests/factories/ (create tests/factories/teacher_factories.py if
   teacher-domain factories don't already have their own file, or add
   to the existing appropriate file — check the project's existing
   factory file organization first):

   class TeacherVerificationFactory(factory.django.DjangoModelFactory):
       class Meta:
           model = TeacherVerification
       teacher_profile = factory.SubFactory(TeacherProfileFactory)
       status = "APPROVED"
       reviewed_at = factory.LazyFunction(timezone.now)

   Document clearly in a docstring/comment: "Defaults to APPROVED
   deliberately — most of this project's existing tests assume a
   bookable, marketplace-visible teacher; tests that specifically need
   to exercise the verification state machine should override status
   explicitly (e.g. TeacherVerificationFactory(status='PENDING'))."

2. Modify TeacherProfileFactory: add a post_generation hook (using
   factory_boy's @factory.post_generation) that automatically creates a
   TeacherVerificationFactory(teacher_profile=self, status="APPROVED")
   for every TeacherProfile the factory builds, UNLESS the caller
   explicitly passes verification=None or a specific verification
   status override — check factory_boy's documented pattern for
   conditional post-generation and follow it precisely so existing
   factory call sites across the test suite (which don't know about
   verification at all) keep working exactly as before without any
   changes to their call sites.

3. Now run the full backend test suite:
   pytest -x
   Go through every remaining failure caused by Prompt 3's gating
   change (NOT unrelated pre-existing failures — if you find an
   unrelated failure, do not touch it, report it separately) and fix
   each one. The overwhelming majority should now be auto-fixed by the
   factory default above; any remaining ones are likely tests that
   explicitly constructed a User/TeacherProfile without going through
   the factory (e.g. raw User.objects.create_user + manual
   TeacherProfile.objects.create) — for those, add an explicit
   TeacherVerificationFactory(teacher_profile=profile, status="APPROVED")
   line right after the manual TeacherProfile creation.

Files affected:
- tests/factories/teacher_factories.py (new, or wherever appropriate)
- tests/factories/user_factories.py (modify TeacherProfileFactory)
- Any individual test file still failing after the factory fix (exact
  list depends on step 3's findings — report it in your final summary)

Acceptance Criteria:
- pytest -x (full backend suite) passes with zero failures
- git diff shows the factory changes plus only the minimal, targeted
  fixes needed in individual test files (not a wholesale rewrite of
  unrelated test logic)

Verification Steps:
1. pytest -x -v
2. git diff --stat (review that changes are proportional — mostly the
   two factory files, plus a small number of targeted test fixes)
```

### Prompt 5 — API Views (Submit Documents + Status) و Django Admin

```
Goal: Expose the verification submission/status endpoints for teachers,
and build the Django Admin interface for staff to review and act on
pending verifications.

Before starting, read these files:
1. apps/teachers/services.py (from Prompt 3) — the exact signatures of
   submit_verification_documents, approve_teacher, reject_teacher,
   suspend_teacher, unsuspend_teacher
2. apps/teachers/views.py — the full file, to match this project's
   existing view style (how IsTeacher is used, how APIView vs generic
   views are chosen, how serializers.py is organized for this app)
3. apps/teachers/urls.py — the full file
4. apps/payments/views.py — RefundPaymentView and ResolveDisputeView,
   as reference examples for "a POST action view that calls a single
   service function and returns a serialized result"
5. apps/users/admin.py and apps/teachers/admin.py — the full files, to
   match this project's existing admin style and the
   TeacherSkillInline/AvailabilitySlotInline pattern already used on
   TeacherProfileAdmin

What to build:

a) apps/teachers/serializers.py — add:
   - TeacherDocumentSerializer (read-only: id, document_type, file,
     uploaded_at)
   - TeacherVerificationSerializer (read-only: id, status,
     submitted_at, reviewed_at, rejection_reason, suspension_reason,
     documents = TeacherDocumentSerializer(many=True))
   - SubmitVerificationDocumentsSerializer (write-only input: a list of
     {document_type, file} — use a nested serializer or
     ListField(child=...) matching how this project already handles
     multi-file uploads elsewhere, if it does; if there's no existing
     precedent, implement the simplest DRF-idiomatic multipart list
     upload)

b) apps/teachers/views.py — add:
   - VerificationStatusView (RetrieveAPIView or a plain APIView,
     GET /api/teachers/me/verification/, permission_classes=[IsAuthenticated, IsTeacher]):
     calls ensure_verification_exists(teacher_profile=request.user.teacher_profile)
     (create the TeacherProfile too if it doesn't exist yet, following
     the same lazy-create pattern used elsewhere in this project for a
     teacher's first profile-related action) and returns
     TeacherVerificationSerializer(verification).data
   - SubmitVerificationDocumentsView (APIView, POST,
     permission_classes=[IsAuthenticated, IsTeacher]):
     validates input with SubmitVerificationDocumentsSerializer, calls
     services.submit_verification_documents, returns
     TeacherVerificationSerializer(result).data

c) apps/teachers/urls.py — add routes:
   me/verification/ -> VerificationStatusView
   me/verification/documents/ -> SubmitVerificationDocumentsView

d) apps/teachers/admin.py — add:
   - TeacherDocumentInline (TabularInline, readonly_fields on file/
     document_type/uploaded_at since documents shouldn't be edited from
     admin, only viewed)
   - TeacherVerificationAdmin (ModelAdmin):
     list_display = [teacher_profile, status, submitted_at, reviewed_at, reviewed_by]
     list_filter = [status]
     search_fields = [teacher_profile__user__email, teacher_profile__user__full_name]
     inlines = [TeacherDocumentInline]
     readonly_fields = [submitted_at, reviewed_at, reviewed_by] (these
     should only ever be set by the service functions, not hand-edited
     in admin)
     Add custom admin actions:
     @admin.action(description="Approve selected verifications")
     def approve_selected(self, request, queryset):
         for verification in queryset:
             try:
                 services.approve_teacher(verification_id=str(verification.id), admin_user=request.user)
             except ApplicationError as exc:
                 self.message_user(request, f"{verification}: {exc}", level=messages.ERROR)
     (same pattern for a reject_selected action — but since reject
     requires a reason, and Django admin bulk actions don't have a
     built-in way to collect free-text input per bulk action easily,
     implement reject as a bulk action that only works cleanly for a
     single selected row at a time: if queryset.count() != 1, show an
     error message via self.message_user asking the admin to select
     exactly one row and reject_selected redirects to an intermediate
     admin change-form-like page requesting the reason — OR, simpler
     and acceptable for this Phase's MVP: implement reject only as a
     custom admin view reached via a "Reject" button added to
     TeacherVerificationAdmin's change_form (via get_urls()/a custom
     admin template), taking a reason in a simple form. Choose
     whichever approach requires less new Django-admin-internals code
     and clearly document the choice and any UX limitation in a
     docstring/comment — a full custom review dashboard UI is out of
     scope for this Phase and is explicitly planned for Phase 8.)
     Register both approve_selected (bulk-safe) and a documented,
     even if minimal, single-verification reject/suspend/unsuspend
     path.

Files affected:
- apps/teachers/serializers.py
- apps/teachers/views.py
- apps/teachers/urls.py
- apps/teachers/admin.py

Then write tests:
- tests/teachers/test_views_verification.py: GET verification status
  (creates lazily if missing), POST submit documents (happy path +
  wrong status rejection), permission checks (a student calling these
  endpoints gets 403)
- A basic admin smoke test if this project has any existing admin
  tests to follow the pattern of (check for tests/*/test_admin.py
  files first); if no such pattern exists in this project at all, skip
  writing an admin test and note this in your final summary rather
  than inventing a new testing convention for admin views alone.

Acceptance Criteria:
- pytest tests/teachers/ -v passes completely
- A teacher can submit documents and see their status via the API
- Staff can approve a verification via Django Admin's
  approve_selected action and see it reflected via
  GET /api/teachers/me/verification/

Verification Steps:
1. pytest tests/teachers/ -v
2. python manage.py runserver, then manually: log in to /admin/, view
   TeacherVerification list, run the approve action on a PENDING row,
   confirm status changes
3. git diff --stat
```

### Prompt 6 — Celery Notification Tasks (In-App Only، بدون ایمیل)

```
Goal: Add in-app notification tasks for the verification lifecycle
events, following this project's exact existing Celery task pattern.
Email notifications are explicitly out of scope here — they belong to
Phase 4 (Transactional Email Matrix); this prompt only wires up
apps.notifications.services.create_notification, the same as every
other domain in this project currently does.

Before starting, read these files in full:
1. apps/bookings/tasks.py — the complete file, especially
   notify_new_booking_request and notify_booking_accepted, to copy the
   exact structure (thin task, lazy imports, try/except with
   logger.exception and a bare return, no retry policy, a final
   logger.info on success)
2. apps/notifications/services.py — create_notification's exact
   signature (recipient, type, payload)
3. apps/notifications/models.py — the Notification model's `type`
   choices field, to see whether new type values need to be added
   there or if it's a free-form CharField

What to build:

Create apps/teachers/tasks.py with four Celery tasks, each following
the exact structure of apps.bookings.tasks.notify_new_booking_request:

1. @shared_task(name="teachers.notify_verification_submitted")
   def notify_verification_submitted(verification_id: str) -> None:
   Fetches the TeacherVerification (select_related teacher_profile__user),
   then notifies... note that this needs a recipient who is an admin/
   staff member, not a specific single user like the other notification
   tasks in this project (which notify a specific booking's
   student/teacher). Check apps.users.models.User for how staff/admin
   users are identified (is_staff, is_superuser, or a specific role) and
   send this notification to every currently active staff user (loop
   over User.objects.filter(is_staff=True, is_active=True) and call
   create_notification for each — if this project has no existing
   precedent for "notify all staff," this is the first one; keep it
   simple, don't over-engineer a broadcast mechanism beyond a plain
   loop).

2. @shared_task(name="teachers.notify_verification_approved")
   def notify_verification_approved(verification_id: str) -> None:
   Notifies the teacher (verification.teacher_profile.user) with type
   "TEACHER_VERIFICATION_APPROVED".

3. @shared_task(name="teachers.notify_verification_rejected")
   def notify_verification_rejected(verification_id: str) -> None:
   Notifies the teacher with type "TEACHER_VERIFICATION_REJECTED",
   payload including rejection_reason.

4. @shared_task(name="teachers.notify_verification_suspended")
   def notify_verification_suspended(verification_id: str) -> None:
   Notifies the teacher with type "TEACHER_VERIFICATION_SUSPENDED",
   payload including suspension_reason.

If apps.notifications.models.Notification.type is a fixed-choice field
(not free-form CharField without choices), add the four new type
values above to that choices list — check first, and if it's already
free-form (no choices constraint), skip this and just use the string
values directly, matching how other domains in this project already do
it.

Now go back to apps/teachers/services.py (from Prompt 3) and confirm
each transaction.on_commit(...) call added there references these
exact task names correctly (submit_verification_documents ->
notify_verification_submitted.delay(str(verification.id)),
approve_teacher -> notify_verification_approved.delay(...), etc.) — fix
any mismatch found.

Files affected:
- apps/teachers/tasks.py (new)
- apps/notifications/models.py (only if a choices list needs
  extending — check first)
- apps/teachers/services.py (only to fix task-name references if
  Prompt 3 used placeholder names)

Then write tests:
- tests/teachers/test_tasks.py: each of the four tasks, called
  directly (not via .delay, call the function synchronously in tests as
  Celery tasks are normally tested in this project — check
  tests/bookings/test_tasks.py or similar existing test for the
  project's convention on testing Celery tasks) creates the expected
  Notification row(s) with the right recipient(s)/type/payload.
- Update the existing service-layer tests from Prompt 3 that assert
  transaction.on_commit behavior (if they didn't fully exercise it
  before because the task didn't exist yet) to now confirm the
  corresponding notification task actually gets dispatched (using
  Celery's eager-mode test settings or CELERY_TASK_ALWAYS_EAGER if this
  project's test settings already configure that — check
  config/settings/test.py or equivalent first).

Acceptance Criteria:
- pytest tests/teachers/ -v passes completely including the new task
  tests
- Every transition function in apps/teachers/services.py correctly
  dispatches its corresponding notification task

Verification Steps:
1. pytest tests/teachers/ -v
2. grep -n "notify_verification" apps/teachers/services.py apps/teachers/tasks.py
   (manually confirm every dispatch call matches an actual task name)
3. git diff --stat
```

### Prompt 7 — فرانت‌اند: فرم ارسال مدارک + بنر وضعیت

```
Goal: Build the teacher-facing UI for submitting verification documents
and seeing their current verification status, following this project's
existing patterns (specifically mirroring the PayoutAccountBanner /
TeacherPayoutAccountPage structure built in Phase 1, since this is a
near-identical shape of feature: a status banner + a submission form).

Before starting, read these files in full:
1. src/features/teachers/components/PayoutAccountBanner.tsx (from
   Phase 1) — use this as the direct structural template for
   VerificationStatusBanner
2. src/features/teachers/pages/TeacherPayoutAccountPage.tsx (from
   Phase 1) — use this as the direct structural template for
   TeacherVerificationPage
3. src/features/teachers/api/teachers.api.ts — the existing
   getPayoutAccount/submitPayoutAccount functions, to match their exact
   calling convention (axios instance, error handling, response typing)
4. src/features/teachers/types/teacher.types.ts — the existing
   PayoutAccount type, to match its structure
5. src/features/teachers/pages/TeacherDashboardPage.tsx — to see
   exactly how PayoutAccountBanner is currently rendered there, so
   VerificationStatusBanner can be added the same way

What to build:

1. In src/features/teachers/types/teacher.types.ts, add:
   type VerificationStatus = "PENDING" | "UNDER_REVIEW" | "APPROVED" | "REJECTED" | "SUSPENDED";
   interface TeacherDocument {
     id: string;
     document_type: "NATIONAL_ID" | "DEGREE" | "CERTIFICATE" | "OTHER";
     file: string;
     uploaded_at: string;
   }
   interface TeacherVerification {
     id: string;
     status: VerificationStatus;
     submitted_at: string | null;
     reviewed_at: string | null;
     rejection_reason: string;
     suspension_reason: string;
     documents: TeacherDocument[];
   }

2. In src/features/teachers/api/teachers.api.ts, add:
   getVerificationStatus(): Promise<TeacherVerification>
   (GET /api/teachers/me/verification/)
   submitVerificationDocuments(documents: {document_type: string, file: File}[]): Promise<TeacherVerification>
   (POST /api/teachers/me/verification/documents/, as multipart
   form-data — check how file uploads are already handled elsewhere in
   this project, e.g. the avatar upload call, and match that exact
   pattern for constructing FormData)

3. Build src/features/teachers/components/VerificationStatusBanner.tsx,
   structured exactly like PayoutAccountBanner.tsx:
   - useQuery to fetch verification status
   - PENDING: banner "Please submit your verification documents to be
     listed in the marketplace" + link to the submission page
   - UNDER_REVIEW: banner "Your documents are under review"
   - REJECTED: banner showing rejection_reason + a "Resubmit
     documents" link to the same submission page
   - SUSPENDED: banner showing suspension_reason (no action button —
     this requires contacting support, not a self-service resubmission,
     since suspension is a different state than rejection)
   - APPROVED: renders nothing

4. Build src/features/teachers/pages/TeacherVerificationPage.tsx,
   structured like TeacherPayoutAccountPage.tsx:
   - A form allowing the teacher to add one or more document entries,
     each with a document_type select and a file input
   - On submit, calls submitVerificationDocuments and shows a success
     message ("Submitted, pending review") or the API's validation
     error (e.g. if called while already UNDER_REVIEW)
   - If current status is REJECTED, prefill/show the previous
     rejection_reason above the form so the teacher knows what to fix

5. In TeacherDashboardPage.tsx, add <VerificationStatusBanner /> next
   to the existing <PayoutAccountBanner /> render.

6. In src/app/routes/router.tsx and lazyPages.ts, add a route for
   TeacherVerificationPage (e.g. /teacher/verification), following the
   exact pattern used for the payout-account route from Phase 1.

7. In src/test/mocks/handlers/teacher.handlers.ts and
   fixtures/teacher.fixtures.ts, add MSW handlers/fixtures for GET
   verification status and POST submit documents, covering at least
   the PENDING and APPROVED response shapes.

Files affected:
- src/features/teachers/types/teacher.types.ts
- src/features/teachers/api/teachers.api.ts
- src/features/teachers/components/VerificationStatusBanner.tsx (new)
- src/features/teachers/pages/TeacherVerificationPage.tsx (new)
- src/features/teachers/pages/TeacherDashboardPage.tsx
- src/app/routes/router.tsx
- src/app/routes/lazyPages.ts
- src/test/mocks/handlers/teacher.handlers.ts
- src/test/mocks/fixtures/teacher.fixtures.ts

Then write component tests mirroring the existing PayoutAccountBanner
tests structurally:
- VerificationStatusBanner.test.tsx: one test per status value
  (PENDING/UNDER_REVIEW/REJECTED/SUSPENDED render the expected message;
  APPROVED renders nothing)
- TeacherVerificationPage.test.tsx: submitting the form calls
  submitVerificationDocuments with the right payload; a success message
  appears after a successful submission

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds without type errors

Verification Steps:
1. npx vitest run
2. npm run build
3. git diff --stat
```

### Prompt 8 — بررسی نهایی: بدون فرض «همهٔ مدرس‌ها نمایش داده می‌شوند» در فرانت + تست یکپارچهٔ End-to-End

```
Goal: Final review to confirm no frontend code silently assumes every
registered teacher is marketplace-visible (since that assumption is no
longer true after this Phase), and add one full backend end-to-end
integration test covering the entire verification lifecycle combined
with booking gating.

Before starting:
1. Review the diffs from Prompts 1-7 of this Phase to have the full
   picture.
2. Read src/features/teachers/pages/TeacherListPage.tsx (or wherever
   the public marketplace listing lives) and
   TeacherPublicProfilePage.tsx to check for any hardcoded assumption
   (e.g. "if the API returns this teacher, always show a Book button"
   — that's fine, since the backend already filters; the concern is
   only for hardcoded mock data assuming an unfiltered set, e.g. inside
   Storybook fixtures or MSW handlers used in existing tests for these
   pages that pre-date this Phase and might include PENDING teachers in
   their mocked "list" responses without realizing the real API would
   never return them).

What to build:

1. grep -rln "PENDING\|UNDER_REVIEW\|REJECTED\|SUSPENDED" src/test/mocks/
   and review each mocked fixture: if any existing MSW fixture for the
   teacher-list endpoint includes teachers in a non-APPROVED state
   (which would now be unrealistic, since the real backend never
   returns them there), either remove those entries from the list
   fixture or move them to a verification-specific fixture set used
   only by the new VerificationStatusBanner/TeacherVerificationPage
   tests from Prompt 7.

2. Add one new backend integration test:
   tests/teachers/test_e2e_verification_and_booking.py covering:
   - Register a new teacher (via the actual registration service
     function, not a factory shortcut, to exercise the real lazy-create
     path) → confirm their verification status is PENDING and confirm
     they do NOT appear in list_teachers results
   - Attempt to create_booking against this teacher → confirm
     NotFoundError is raised
   - submit_verification_documents → confirm status becomes
     UNDER_REVIEW
   - approve_teacher (as an admin user) → confirm status becomes
     APPROVED
   - Confirm the teacher NOW appears in list_teachers results
   - create_booking against this teacher → confirm it now succeeds
   - suspend_teacher → confirm the teacher disappears from
     list_teachers again, and a NEW create_booking attempt now fails
     with NotFoundError, while asserting explicitly that the earlier
     successful booking object itself is untouched (still exists with
     its original status) — to confirm this Phase's documented scope
     boundary (suspension doesn't retroactively affect existing
     bookings) is actually true in code, not just in a docstring claim.

Files affected:
- src/test/mocks/handlers/teacher.handlers.ts and/or
  fixtures/teacher.fixtures.ts (only if step 1 finds an issue — report
  findings even if no changes were needed)
- tests/teachers/test_e2e_verification_and_booking.py (new)

Acceptance Criteria:
- pytest -x (full backend suite) passes, including the new e2e test
- npx vitest run (full frontend suite) passes
- The e2e test explicitly proves that suspending a teacher does not
  alter their pre-existing bookings

Verification Steps:
1. pytest tests/teachers/test_e2e_verification_and_booking.py -v
2. pytest -x -v (full backend suite)
3. npx vitest run (full frontend suite)
4. grep -rln "PENDING\|UNDER_REVIEW\|REJECTED\|SUSPENDED" src/test/mocks/
   (paste the output in your final summary along with what, if
   anything, was changed)
5. git diff --stat (final summary of this entire Phase)
```
