# Executable prompts for the AI coding agents (English)

All decisions D-1 to D-5 are now recorded as approved and baked into the prompts. Each prompt is self-contained. Paths are relative to `hirbad-backend/` unless stated otherwise. Nothing in the repository has been changed.

**Run order:** Agent 1, then Agent 2, then Agent 3 (it may be developed in parallel after Agent 1, but merge it after Agent 2 and rebase its migration dependency), then Agent 4. Deploy Agent 1 before Agent 3, otherwise the group-payment backlog would be released at a 0% commission and frozen that way.

---

## Agent 1: Default commission rate and rate history

```
ROLE
You are a senior Django engineer working on the Hirbad platform (repo root contains
hirbad-backend/ and hirbad-frontend/). Stack: Django 5.2, Celery 5.6, pytest with
filterwarnings=error (any warning fails a test), black/isort/flake8 at line length 88.
Migrations are excluded from lint.

GROUND RULES
- Change only what this task describes. Do not alter release, refund, dispute or no-show
  logic. Do not touch unrelated apps.
- Read the real code before acting. Anything you cannot verify in the code must be reported
  as an assumption, not silently assumed.
- Run the tests and lint and report the REAL output. If you cannot run something, say so
  explicitly instead of claiming success.
- Do not add services, models or dependencies beyond what is listed here.

GOAL
The platform default commission rate must be 10%. It must be seeded automatically in new
environments, the system must behave safely and predictably when no rate is configured
(it must NEVER silently fall back to 0%), and the history of rate changes must be preserved
(append-only).

CONFIRMED ROOT CAUSES (verify each in code first)
1. No data migration or seed creates a PlatformCommission row. get_commission_rate in
   apps/payments/services.py returns Decimal("0") when no row applies, so a fresh
   environment silently charges 0% commission.
2. PlatformCommission.rate (apps/payments/models.py) has no validators and no DB
   constraint. A rate above 1 could produce a negative teacher payout.
3. PlatformCommissionAdmin (apps/payments/admin.py) allows editing and deleting rows,
   contradicting the append-only intent documented in the model docstring. Rate changes
   are not written to AdminAuditLog.
4. Tie-breaking for equal effective_from values is undefined.

FILES TO REVIEW
- apps/payments/models.py (PlatformCommission)
- apps/payments/services.py (get_commission_rate, release_payment_to_teacher)
- apps/payments/admin.py (PlatformCommissionAdmin)
- apps/common/logging.py (log_admin_action; note target_from_result=True)
- apps/common/models.py (AdminAuditLog)
- config/settings/base.py and config/settings/test.py
- apps/payments/migrations/ (latest existing: 0014_payment_failure_code)
- tests/payments/test_services.py (about 42 commission-related mentions; a test around
  line 161 asserts the rate is 0 when none is configured)
- tests/conftest.py and tests/payments/ for fixtures
- hirbad-backend/docs/ and docs/DEPLOYMENT.md

REQUIREMENTS
1. Setting: add PLATFORM_COMMISSION_DEFAULT_RATE to config/settings/base.py, read from env
   with config(..., default="0.10", cast=Decimal). Validate 0 <= rate <= 1 at import time and
   raise ImproperlyConfigured otherwise.
2. get_commission_rate resolution order: teacher-specific override, then global row, then the
   settings fallback. Only rows with effective_from <= now apply. Break ties with
   order_by("-effective_from", "-created_at"). When the fallback is used, emit logger.warning
   with a clear message. Never return 0 because "nothing is configured". An intentional row
   with rate 0 created by an admin still wins over the fallback.
3. Model: add validators (0 to 1) on rate and a CheckConstraint enforcing 0 <= rate <= 1.
   Check the Django 5.2 API for the constraint keyword (check= versus condition=) before
   writing it.
4. Schema migration (next free number, expected 0015): AlterField + AddConstraint. Before
   adding the constraint, run a RunPython precheck that detects any existing row with rate
   outside [0, 1] and aborts with a clear message listing the offending row ids.
5. Data migration (next free number, expected 0016): if no global row exists
   (teacher__isnull=True), create one with rate=Decimal("0.1000") and
   effective_from=timezone.now() at migration time. It must be idempotent (never create a
   second default, never overwrite an admin-created global row). reverse_code is a noop.
6. New service set_commission_rate(*, admin_user, rate, teacher=None, effective_from=None)
   in apps/payments/services.py, decorated with @log_service_call and
   @log_admin_action("commission_rate_set", target_from_result=True). It always INSERTS a new
   row, validates 0 <= rate <= 1, defaults effective_from to now, and rejects an
   effective_from in the past with ApplicationError (a rate change must never rewrite
   history). Return the created row.
7. PlatformCommissionAdmin: existing rows must not be editable or deletable
   (has_change_permission and has_delete_permission return False for existing objects).
   Creating a row goes through save_model, which must call set_commission_rate with
   request.user. Show a helpful note that rates are append-only.
8. Existing tests must keep their meaning: add an autouse fixture in
   tests/payments/conftest.py that deletes seeded commission rows and overrides the settings
   fallback to Decimal("0"), unless a test opts out with a new marker (for example
   real_commission_defaults). Register the marker in pyproject.toml so
   filterwarnings=error does not fail on an unknown marker. All pre-existing tests must pass
   without changing their business logic.
9. Documentation: update the payments/commission section in hirbad-backend/docs and
   docs/DEPLOYMENT.md (new setting, seed behavior, append-only history, how to change the
   rate through the admin).

REQUIRED TESTS
- Fresh DB after migrate has exactly one global row with rate 0.1000.
- Seed migration is idempotent: with an existing admin-created global row it adds nothing.
- Fallback applies when no row exists: returns the settings value and logs a warning.
- An intentional rate-0 row overrides the fallback.
- Teacher override wins over the global rate; a future effective_from is not applied yet.
- Tie-break on equal effective_from uses created_at.
- Validators, CheckConstraint and the service all reject rate < 0 and rate > 1.
- Service rejects a past effective_from; it creates an AdminAuditLog row on success and none
  on failure.
- Admin: existing rows are not editable or deletable; adding a row creates it through the
  service and writes the audit log.
- Release at 10% produces the correct commission and teacher payout, and the frozen values
  on Payment do not change when the rate changes later.
- Invalid PLATFORM_COMMISSION_DEFAULT_RATE in settings raises ImproperlyConfigured.

ACCEPTANCE CRITERIA
- The full pytest suite, flake8, black --check and isort --check-only are green.
- migrate on an empty database yields one 10% global row.
- No code path can reach a 0% commission merely because nothing is configured.
- Existing rate rows cannot be edited or deleted; every change is audited.
- Final report lists: changed files, migration numbers, the exact commands run and their real
  output, and any unverified assumption.
```

---

## Agent 2: Commission in `RESOLVED_SPLIT` disputes

```
ROLE
You are a senior Django engineer working on the Hirbad platform (hirbad-backend and
hirbad-frontend). Stack: Django 5.2, DRF, pytest with filterwarnings=error, black/isort/
flake8 at line length 88; frontend with React, zod, i18n and its own tests.
PREREQUISITE: Agent 1 is merged (get_commission_rate has the settings fallback, and the
tests/payments commission fixture exists).

GROUND RULES
- Change only what this task describes. No changes to release sweep, refund-on-rejection,
  no-show or cancellation logic.
- Read the real code before acting; report unverifiable points as assumptions.
- Run backend tests, lint, and frontend tests; report the REAL output. If something cannot
  run, say so.
- Do not change the notification payload contract (apps/notifications/payloads.py and types).

APPROVED POLICY (D-1, D-4, D-5) - implement exactly this
- Basis: share = round_half_up(amount * split_teacher_percentage / 100), using
  quantize_amount from apps/common/money.py (ROUND_HALF_UP, whole IRR).
- Rate: if the payment was already released, use commission_rate from the original PAYOUT
  ledger entry (entry_type="PAYOUT", dispute is null). Otherwise use
  get_commission_rate(teacher=payment.teacher) evaluated at resolution time.
- commission = round_half_up(share * rate)
  teacher_net = share - commission
  student_refund = amount - share
  Invariant: commission + teacher_net + student_refund == amount, exactly.
- The platform takes no commission on the refunded portion.
- split_teacher_percentage must be an integer from 1 to 99 inclusive. 0 and 100 are rejected
  with a message telling the admin to use RESOLVED_REFUND or RESOLVED_TEACHER.
- Dispute money accounting must end consistent with the final decision: net ledger movement
  for a dispute equals the target. See requirement 4 (delta logic) for chargeback disputes.
- Payment.commission_amount, teacher_payout_amount and refund_amount are updated to the final
  settled values when a dispute is settled (documented exception to the "frozen after
  release" rule), with a PaymentEvent recording it. The original PAYOUT ledger row is never
  mutated.

CONFIRMED ROOT CAUSES (verify each in code first)
In apps/payments/services.py, resolve_dispute:
1. Unreleased + RESOLVED_SPLIT creates the PAYOUT entry with commission_amount=0 and
   commission_rate=0 and sets Payment.commission_amount=0, so no commission is charged,
   unlike RESOLVED_TEACHER.
2. Released + RESOLVED_SPLIT computes the clawback as payout_net * (100-pct)/100 through
   _create_clawback_ledger_entry; this is implicit and not exactly derived from the target.
3. open_dispute_from_chargeback already records a full CLAWBACK for an already-released
   payment; RESOLVED_SPLIT then creates another one with no guard (double clawback).
   RESOLVED_REFUND has the guard; RESOLVED_SPLIT does not.
4. RESOLVED_TEACHER on a chargeback dispute that already has a full clawback returns no
   money to the teacher.
5. pct=0 creates a zero-amount PAYOUT entry and requires a verified payout account;
   pct=100 yields status REFUNDED with refund_amount 0.

FILES TO REVIEW
- apps/payments/services.py (resolve_dispute, _create_clawback_ledger_entry,
  open_dispute_from_chargeback, release_payment_to_teacher, _validate_payout_ready)
- apps/payments/models.py (Dispute, PayoutLedgerEntry, Payment)
- apps/payments/serializers.py (DisputeSerializer, ResolveDisputeSerializer),
  apps/payments/forms.py, apps/payments/admin.py (DisputeAdmin)
- apps/adminapi/serializers.py and apps/adminapi/views/payments.py
- apps/payments/tasks.py (_resolution_summary; review only)
- hirbad-frontend/src/features/admin/components/ResolveDisputeDialog.tsx, its tests, the
  i18n keys (e.g. disputes.resolveDialog.validation.splitInvalid) and admin types/fixtures
- tests/payments/test_disputes.py (split tests around lines 407-570, TestChargebackWebhook)
- apps/payments/migrations/ (use the next free number after Agent 1's, expected 0017)
- hirbad-backend/docs and docs/ADMIN_MANUAL_WALKTHROUGH.md

REQUIREMENTS
1. Add nullable fields to Dispute: settled_commission_amount, settled_teacher_amount,
   settled_student_refund_amount (DecimalField max_digits=10, decimal_places=0) and
   settled_commission_rate (DecimalField max_digits=5, decimal_places=4). Additive migration,
   no backfill: historical disputes keep nulls and must render without errors.
2. Unreleased RESOLVED_SPLIT: call _validate_payout_ready; create a PAYOUT entry with
   gross_amount=payment.amount, commission_amount=commission, payout_amount=teacher_net,
   commission_rate=rate, dispute=dispute, then initiate the payout through the payout provider
   as the current code does; set Payment commission/payout/refund to final values; status
   stays REFUNDED (do not add an enum value).
3. Released RESOLVED_SPLIT: target clawback K = original_payout_amount - teacher_net (an exact
   difference, not a fractional multiplication). Never mutate the original PAYOUT row.
   Update Payment to the final values and write a PaymentEvent.
4. Delta logic for disputes that already have ledger entries (chargeback), applied to all three
   resolutions: delta = K_target - sum(existing CLAWBACK entries for this dispute). delta > 0:
   create a CLAWBACK for delta. delta < 0: create a PAYOUT entry with dispute=dispute for
   -delta (return of an excess clawback, status PENDING_TRANSFER). delta == 0: create nothing.
   For RESOLVED_REFUND K_target is the full original net; for RESOLVED_TEACHER it is 0.
   Implement this once in a single private helper used by all branches and remove the
   existing duplicated guard logic. Double clawback must be impossible.
5. Fill all four settled_* fields on every resolution (REFUND: commission 0, teacher 0,
   refund = amount; TEACHER: the release values). Expose them read-only in Django Admin and in
   DisputeSerializer/admin API.
6. Enforce 1-99 in ResolveDisputeSerializer, apps/payments/forms.py (min_value/max_value) and
   service-level validation (raise ApplicationError with a stable code). Update the zod schema
   and the i18n messages in ResolveDisputeDialog, and its frontend tests.
7. Keep everything inside the existing transaction.atomic with select_for_update on Dispute
   and Payment. Compute inside the lock. A second resolve call still raises ConflictError.
   If a verified payout account is required for the payout part and is missing, the whole
   operation rolls back (current behavior).
8. Documentation: update the dispute/commission section in hirbad-backend/docs and
   docs/ADMIN_MANUAL_WALKTHROUGH.md with the policy and the numeric examples below.

WORKED EXAMPLES (must be encoded as tests)
- Amount 1,000,000 IRR, rate 10%, teacher share 60%, unreleased: share 600,000, commission
  60,000, teacher_net 540,000, refund 400,000.
- Same, but already released with original net 900,000 / commission 100,000: target net
  540,000, clawback 360,000, platform retains 60,000.
- Amount 100, rate 10%, share 25%: commission round_half_up(2.5)=3, teacher_net 22,
  refund 75; released case with original net 90: clawback 68.

REQUIRED TESTS
- All worked examples above, unreleased and released.
- Invariant commission + teacher_net + student_refund == amount across many amounts, rates and
  percentages (including rounding edges and amounts that do not divide evenly).
- Released payments use the frozen original rate even if the rate changed afterwards;
  unreleased payments use the rate at resolution time.
- pct 0, 100, negative and over 100 are rejected at API, form, service and UI levels with
  guidance to the other resolutions.
- Chargeback dispute + RESOLVED_SPLIT: no double clawback (net equals target); chargeback +
  RESOLVED_TEACHER returns the excess through a dispute-linked PAYOUT; chargeback +
  RESOLVED_REFUND creates no extra clawback; delta == 0 creates no entry.
- Resolving twice raises ConflictError; missing payout account rolls back everything (no
  ledger rows, dispute still open).
- settled_* fields set correctly for all three resolutions; historical disputes with nulls
  serialize and render fine.
- dispute_resolved notification payload contract unchanged.
- Existing split tests are changed only where the new policy deliberately changes the result,
  and every such change is justified in your report.

ACCEPTANCE CRITERIA
- Backend suite, lint, and frontend tests are green.
- No path can create a double clawback.
- For every split, the three amounts sum to the payment amount exactly.
- Final report lists changed files, the migration number, commands run with real output, every
  modified existing test with justification, and any assumption.
```

---

## Agent 3: Unified release of private and group payments

```
ROLE
You are a senior Django engineer working on the Hirbad platform (hirbad-backend). Stack:
Django 5.2, Celery 5.6 (Beat), pytest with filterwarnings=error, black/isort/flake8 at line
length 88.
PREREQUISITE: Agent 1 is merged, so the 10% default commission is live before any group
backlog is released.

GROUND RULES
- Do NOT modify release_payment_to_teacher, refund logic, dispute logic or commission logic.
  This task changes only which payments are selected, plus one DB constraint and one
  read-only command.
- Read the real code before acting; report unverifiable points as assumptions.
- Run tests and lint; report REAL output. If something cannot run, say so.

PROBLEM
release_held_payments (apps/payments/tasks.py) selects payments with
Payment.objects.filter(status="HELD_IN_ESCROW", booking__session__end_time__lte=cutoff).
A group-enrollment payment has booking=NULL and is linked only through enrollment, so it is
never selected and its escrow is never released. The same expression is duplicated in
apps/payments/reconciliation.py (_stuck_escrow_payments), so the monitoring that should flag
stuck escrow is blind to group payments too. Existing sweep tests only create booking-origin
payments (_held_escrow_payment sets enrollment=None). release_payment_to_teacher itself
already handles both origins via _resolve_payment_session_and_start (verify, and note the
existing TestReleasePaymentToTeacherEnrollmentOrigin).

APPROVED POLICY (D-2)
- A payment whose session has status CANCELLED, or outcome NO_SHOW_TEACHER, must never be
  auto-released by the sweep. It must remain visible in the reconciliation "stuck escrow"
  report for manual handling. Do NOT add any automatic refund.
- These guards live ONLY in the selector, not in release_payment_to_teacher, so that the
  manual admin path (resolve_dispute RESOLVED_TEACHER) keeps working.

FILES TO REVIEW
- apps/payments/tasks.py (release_held_payments)
- apps/payments/reconciliation.py and its callers (management command reconcile_payments,
  run_scheduled_reconciliation task)
- apps/payments/selectors.py, apps/payments/models.py (Payment, PayoutLedgerEntry, Dispute)
- apps/payments/services.py (release_payment_to_teacher, _resolve_payment_session_and_start,
  PayoutBlockedError) - read only
- apps/sessions/models.py (Session, SessionEnrollment)
- tests/payments/test_tasks.py, tests/payments/test_reconciliation_task.py,
  tests/payments/test_enrollment_payment_origin.py
- tests/factories/payment_factories.py and session_factories.py (PaymentFactory builds an
  enrollment by default; pass enrollment=None for booking-origin; for group use
  SessionFactory(session_type="GROUP", price_per_seat=...))
- apps/payments/management/commands/reconcile_payments.py (command pattern to follow)
- hirbad-backend/docs/NOTIFICATIONS.md ("Finance observations")
- apps/payments/migrations/ (use the next free number after Agent 2's, expected 0018)

REQUIREMENTS
1. In apps/payments/selectors.py add:
   - held_payments_past_window(cutoff): base queryset of HELD_IN_ESCROW payments annotated with
     session_end, session_status and session_outcome using
     Coalesce(booking__session__X, enrollment__session__X) with an explicit output_field,
     filtered by session_end <= cutoff.
   - payments_due_for_release(now): the base queryset with cutoff = now minus
     settings.PAYMENTS_ESCROW_HOLD_WINDOW_HOURS, then excluding (a) payments with a Dispute
     in status OPEN or UNDER_REVIEW (use an Exists subquery, not a multi-valued exclude),
     (b) session_status == "CANCELLED", (c) session_outcome == "NO_SHOW_TEACHER". Order by
     session_end then id.
2. release_held_payments and _stuck_escrow_payments must both read from these selectors. The
   stuck report uses held_payments_past_window with the existing 3x window and NO exclusions,
   so cancelled and teacher-no-show payments appear there. No duplicate query logic may remain.
3. Keep the per-payment loop and error handling: PayoutBlockedError is counted and logged,
   one failing payment never stops the sweep. Log ApplicationError separately from
   unexpected exceptions. Keep the final summary log.
4. Add a read-only management command report_release_backlog printing: count and total amount
   of payments currently due for release split by origin (booking vs enrollment), plus the
   count blocked by D-2 (cancelled / teacher no-show). It must write nothing to the database.
5. Migration: add UniqueConstraint(fields=["payment"],
   condition=Q(entry_type="PAYOUT", dispute__isnull=True), name="payout_one_release_per_payment")
   on PayoutLedgerEntry. The condition on dispute__isnull=True is deliberate: dispute-linked
   PAYOUT entries (created by split resolutions or chargeback give-backs) must remain allowed.
   Before adding it, run a RunPython precheck that finds payments with more than one matching
   PAYOUT entry and aborts with a clear message listing the payment ids.
6. Documentation: in hirbad-backend/docs/NOTIFICATIONS.md mark "Group-enrollment escrow is never
   released" as resolved and keep the cancelled-session and no-show items as open follow-ups.
   Document the selector as the single source of truth, the report_release_backlog command, and
   the deployment note: deploy after the commission default (Agent 1), run
   report_release_backlog first, then let the sweep run.

REQUIRED TESTS
- Booking-origin and enrollment-origin (GROUP session) payments are both released after the
  hold window and neither before it; test the exact boundary.
- Running the sweep twice produces exactly one PAYOUT ledger entry per payment.
- Re-invoking release on an already RELEASED payment creates no second entry (simulating an
  overlapping sweep).
- The DB constraint rejects a second non-dispute PAYOUT for the same payment but accepts a
  dispute-linked PAYOUT.
- CANCELLED-session and NO_SHOW_TEACHER payments are never auto-released and appear in the
  reconciliation stuck report; the stuck report also lists group-origin payments past 3x window.
- Payments with an OPEN/UNDER_REVIEW dispute (and DISPUTED status) are not released.
- A teacher without a verified payout account is blocked while other payments in the same sweep
  are released.
- Group release computes commission per payment at the current rate (price_per_seat based),
  using the 10% default.
- report_release_backlog is read-only (assert zero writes) and its counts are correct.
- A payment with neither booking nor enrollment does not crash the selector or the sweep.
- Regression: every existing release and reconciliation test passes unchanged.

ACCEPTANCE CRITERIA
- Group payments are released after the hold window; private payments behave as before except
  the deliberate D-2 exclusions.
- No path can create a second release PAYOUT for one payment.
- Reconciliation reports stuck payments of all origins.
- Full pytest, flake8, black --check, isort --check-only are green.
- Final report lists changed files, the migration number, commands run with real output, and
  any assumption.
```

---

## Agent 4: Integration, regression and documentation review

```
ROLE
You are a reviewer/integrator for the Hirbad backend. Run ONLY after Agents 1, 2 and 3 are
merged.

GROUND RULES
- Do not add features. Create new files only to fix a discrepancy you report.
- Run everything and report the REAL output; state clearly anything you could not run.

TASKS
1. Migrations: confirm apps/payments has a single migration head with no conflicts. Run migrate
   on an empty database and on a database populated before these changes (use a fixture or
   seed data). Confirm the commission seed, constraints and settled_* fields appear as
   expected, and that the prechecks abort with clear messages when given deliberately bad data.
2. Run the full pytest suite, flake8, black --check, isort --check-only, and the frontend test
   suite. Report failures with their cause.
3. Write end-to-end tests (or confirm that equivalent ones exist):
   a. Group enrollment payment -> HELD_IN_ESCROW -> release sweep at 10% -> PENDING_TRANSFER
      entry -> mark_payout_as_transferred.
   b. 1:1 payment -> dispute -> RESOLVED_SPLIT unreleased and released; verify
      commission + teacher_net + student_refund == amount and ledger totals.
   c. Cancelled 1:1 session with a HELD payment: not released, shown in the stuck report.
4. Grep the code base: no remaining direct use of booking__session__end_time for release
   selection; no remaining commission_amount=Decimal("0") hard-coded in resolve_dispute.
5. Verify that no unapproved financial behavior changed: refund-on-rejection, no-show refunds,
   cancellation refunds, release_payment_to_teacher internals, and the notification payload
   contracts must be identical to before.
6. Reconcile documentation (hirbad-backend/docs/ARCHITECTURE.md, NOTIFICATIONS.md,
   docs/ADMIN_MANUAL_WALKTHROUGH.md, docs/DEPLOYMENT.md) with the final behavior. Keep these
   as explicitly open, out-of-scope items: refunds that never reach the payment gateway,
   leave_session cascading deletion of a HELD group payment, teacher cancellation of an
   accepted booking without refund, teacher no-show refunds for group sessions, the teacher
   dashboard showing gross earnings and total_refunded using full amounts, and the late gateway
   callback after the enrollment reaper.
7. Produce a report: discrepancies found, residual risks, the exact commands to run
   report_release_backlog safely in production, and the deployment order (commission default
   first, then the release unification).

ACCEPTANCE CRITERIA
- All suites green; documentation matches behavior; the open-items list is up to date; the
  report states clearly what was and was not executed.
```

---

## Integration notes

| Item | Detail |
|---|---|
| Migration chain | Agent 1: `0015` (constraints), `0016` (seed); Agent 2: `0017`; Agent 3: `0018`. Each agent must use the next free number if earlier ones differ. |
| Merge order | 1 → 2 → 3 → 4. If Agent 3 runs in parallel, rebase its migration dependency after Agent 2 merges. |
| Shared file | `apps/payments/services.py`: Agent 1 edits `get_commission_rate` and adds `set_commission_rate`; Agent 2 edits only `resolve_dispute` and its helpers; Agent 3 does not touch it. |
| Deployment | Deploy Agent 1 before Agent 3, then run `report_release_backlog`, then let the sweep release the backlog. Actual bank transfers remain manual. |
| Pre-deploy data checks | Run the rate-range and duplicate-PAYOUT prechecks on a production backup first. |

I can also package these four prompts as a single markdown file for you to download, if that is useful.