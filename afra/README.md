# Afra Platform — Payment System Remediation Roadmap

**Role:** Senior Software Architect / Technical Lead execution plan
**Input:** `afra-payment-flow-audit.md` (prior audit)
**Goal:** Move overall status from *"Contains critical issues — not production-ready"* → *"Production-ready secure payment architecture"*
**Current migration heads (for planning migration numbers below):** `bookings` → `0005`, `payments` → `0010`, `users` → `0004`, `teachers` → `0001`

This document is planning only. No code is written here — every phase ends in an AI coding prompt and an AI verification prompt intended to be handed to a coding assistant (or a developer) as the actual implementation instruction.

---

# PART 1 — Remediation Roadmap

Six phases, strictly ordered by **blast radius and dependency**, not by audit-finding severity alone — a few Medium/Low items get pulled earlier than their severity would suggest because later phases depend on them being fixed first (e.g., provider-aware release logic (Phase 1) must exist before we can safely touch the admin panel's ability to hand-set `status` (Phase 5), or a manually-forced status would still hit the same broken release path).

## Phase 0 — Observability & Safety Net (prerequisite, not in the original 10-item list)

**Objective:** Make every subsequent phase verifiable and reversible before touching payment-moving code.

**Problems being solved:** None directly from the audit — this phase exists because Phases 1–2 touch fund-movement logic, and the audit found **zero reconciliation invariant anywhere** (`sum(captured) == sum(released) + sum(refunded) + sum(held)` is never checked). Without this, a regression in Phase 1 would be silent.

**Files/modules affected:**
- New: `apps/payments/management/commands/reconcile_payments.py`
- New: `apps/payments/checks.py` (Django system check or a scheduled Celery task)

**Backend changes required:** A read-only reconciliation report: per-`Payment`, verify `provider`/`gateway_charge_id` consistency (a `STRIPE` payment must have a `gateway_charge_id`; an `INTERNAL` payment must not), and a global sum invariant across `Payment.status` buckets. Emit structured log/metric, not just print.

**Frontend changes required:** None.

**Database changes required:** None (read-only).

**Migration requirements:** None.

**Testing requirements:** Unit test the reconciliation command against fixtures with a deliberately-broken row (e.g., `INTERNAL` payment with a `gateway_charge_id` set) to confirm it's flagged.

**Risk level:** 🟢 Low (read-only, no behavior change).

**Dependencies:** None — this is the foundation everything else is checked against.

---

## Phase 1 — Payment Capture/Release Consistency & Provider Confusion *(Audit items 1, 2, 3)*

**Objective:** Eliminate the Critical finding: release logic must never call a real Stripe API for money that was never actually collected via Stripe, and vice versa.

**Problems being solved:**
- Critical #1 (partial): `release_payment_to_teacher` and `resolve_dispute`'s Stripe branches call `stripe.Transfer.create` unconditionally, with no `PAYMENTS_USE_STRIPE` gate and no check on `Payment.provider`.
- Audit item 3 (Stripe vs INTERNAL confusion) — this is the direct root cause.

**Files/modules affected:**
- `apps/payments/services.py` — `release_payment_to_teacher` (~L855-1013), `resolve_dispute` (~L1363-1411), `capture_payment_for_booking`
- `apps/payments/models.py` — `Payment.provider`, `Payment.gateway_charge_id`
- `apps/payments/tasks.py` — `release_held_payments`

**Backend changes required:**
1. Introduce an explicit **release strategy dispatch**: `release_payment_to_teacher` branches on `payment.provider` at the top. `STRIPE` → existing Stripe `Transfer.create` path (still gated by `settings.PAYMENTS_USE_STRIPE` as a hard safety belt). `INTERNAL` → a new `_release_payment_internal(payment, teacher_profile)` that only mutates DB state (status, `PayoutLedgerEntry`, commission split) — no external call.
2. Same dispatch applied inside `resolve_dispute`'s `RESOLVED_TEACHER`/`RESOLVED_SPLIT` branches.
3. Add a hard `ApplicationError` guard: if `payment.provider == "STRIPE"` and `payment.gateway_charge_id` is falsy (or vice versa), refuse to release and raise loudly — this makes Phase 0's invariant enforced at write-time, not just observed at audit-time.

**Frontend changes required:** None — this is entirely a backend fund-movement correctness fix; the payment status values returned to the client are unchanged.

**Database changes required:** None — `provider` and `gateway_charge_id` already exist (added in `payments/0005_payment_gateway_charge_id.py`).

**Migration requirements:** None.

**Testing requirements:**
- Unit test: `release_payment_to_teacher` on an `INTERNAL` payment never calls `stripe.Transfer.create` (assert `mock_stripe.Transfer.create.assert_not_called()`).
- Unit test: `release_payment_to_teacher` on a `STRIPE` payment with a valid `gateway_charge_id` does call it (existing behavior preserved).
- Unit test: `release_payment_to_teacher` on a `STRIPE` payment **without** `gateway_charge_id` raises `ApplicationError`, not a Stripe call.
- Regression: full existing `tests/payments/test_services.py` suite must still pass with the new dispatch (existing tests currently patch `stripe.Transfer.create` unconditionally — they'll need the `provider="STRIPE"` fixture explicitly set, since that mock will no longer fire for `INTERNAL` fixtures).

**Risk level:** 🔴 High (touches the exact code path that moves money) — mitigate by shipping behind a feature flag if the org has a flagging system, and by running Phase 0's reconciliation command before/after deploy on a copy of production data if possible.

**Dependencies:** Phase 0 (needs the reconciliation check to confirm correctness pre/post deploy).

---

## Phase 2 — Teacher Payout Flow: Onboarding Wiring & Escrow Release Unblocking *(Audit items 2, 4)*

**Objective:** Make `is_payout_ready` reachable for real, so escrow release can actually complete for `STRIPE`-provider payments, and so `INTERNAL`-provider payments (Phase 1's new path) don't depend on Stripe onboarding status at all.

**Problems being solved:**
- Critical #1 (remainder): `mark_stripe_onboarding_complete` is dead code; no `account.updated` webhook handler exists; `is_payout_ready` can never become `True` through any reachable path.
- Directly unblocks "escrow release logic" (audit item 2) for the Stripe path.

**Files/modules affected:**
- `apps/payments/webhooks.py` — add `account.updated` handling
- `apps/teachers/services.py` — `mark_stripe_onboarding_complete` (already exists, just needs a caller)
- `apps/payments/services.py` — `release_payment_to_teacher`'s payout-readiness check should now only apply to the `STRIPE` branch introduced in Phase 1 (an `INTERNAL` payment's "readiness" is trivial/always-true, or gated by a much simpler "has a payout destination configured" check appropriate for test/dev mode)
- `config/settings/base.py` — confirm `STRIPE_WEBHOOK_SECRET` covers the Connect webhook endpoint (Stripe recommends a **separate** webhook secret/endpoint for Connect `account.*` events vs. platform `payment_intent.*`/`charge.*` events — flag this as a design decision to confirm with whoever owns the Stripe account, not assumed).

**Backend changes required:**
1. New webhook handler `handle_account_updated(event)`: look up `TeacherProfile` by `stripe_account_id`, and if Stripe reports `charges_enabled=True` and `payouts_enabled=True`, call `mark_stripe_onboarding_complete`. If it reports the account has been *disabled* (e.g. `requirements.disabled_reason` set), the handler should also be able to move status back off `"COMPLETE"` — the audit didn't check for this but it's the natural symmetric case and worth doing in the same phase since the webhook plumbing is already being built.
2. Route the new event type through the existing signature-verification path in `StripeWebhookView` (already correctly implemented — just add the `event["type"] == "account.updated"` branch alongside the existing three).
3. Decouple `INTERNAL`-provider release from `is_payout_ready` entirely (this is really finishing Phase 1's dispatch — called out here because it's the piece that actually **unblocks** escrow release for the common/default-config case).

**Frontend changes required:** None strictly required for correctness, but recommended: `TeacherPaymentSummary.tsx` / onboarding status view should reflect the new "disabled" state if the symmetric handler is added (currently `apps/teachers/views.py`'s `me/stripe/status/` endpoint only ever reports what's in the DB, so once the DB is correctly updated by the webhook, no frontend code change is strictly needed — verify this assumption during implementation).

**Database changes required:** None — `stripe_onboarding_status` field already exists (`users/0003_teacherprofile_stripe_account_id_and_more.py`).

**Migration requirements:** None.

**Testing requirements:**
- Webhook test: valid `account.updated` event with `charges_enabled=True, payouts_enabled=True` → `TeacherProfile.stripe_onboarding_status == "COMPLETE"`.
- Webhook test: same event type but account not fully enabled → status unchanged (not prematurely marked complete).
- Webhook test: unknown `stripe_account_id` in payload → handled gracefully (log + 200, not a 500 — mirror existing pattern for unknown payment IDs in `handle_payment_intent_succeeded`).
- Integration test: full flow — `STRIPE`-provider payment `HELD_IN_ESCROW`, teacher `is_payout_ready=True` via the new webhook path, `release_held_payments` sweep succeeds end-to-end.
- Integration test: `INTERNAL`-provider payment releases successfully with **no** Stripe onboarding at all (proves the decoupling from Phase 1/2 works together).

**Risk level:** 🟠 Medium — new webhook surface, but additive (doesn't change existing webhook behavior) and the release-unblocking half is a direct continuation of Phase 1's already-reviewed change.

**Dependencies:** Phase 1 (the provider dispatch must exist before "unblock INTERNAL release from Stripe readiness" is a real change rather than a patch on the old unified path).

---

## Phase 3 — Double-Booking Race Condition *(Audit item 5)*

**Objective:** Guarantee two students can never both hold escrow on the same teacher/time-slot, at the database level, independent of transaction timing.

**Problems being solved:** Critical #2 — `select_for_update()` on a query matching zero rows locks nothing, so the very first booking into an open slot has no concurrency protection.

**Files/modules affected:**
- `apps/bookings/services.py` — `_check_for_conflicting_booking`, `create_booking`
- `apps/bookings/models.py` — `BookingRequest`
- New migration in `apps/bookings/migrations/`

**Backend changes required:**
Two independent layers, both recommended (defense in depth — audit explicitly noted the *code's own comment* incorrectly asserted the existing check was race-safe, so a single layer relying only on application code is exactly the failure mode to avoid repeating):
1. **Row-lock layer (fast fix, ships first):** Before the conflict check, take `select_for_update()` on the `TeacherProfile` row itself (guaranteed to exist), serializing *all* booking-creation attempts for that teacher. This alone closes the race without a schema change and can ship independently/faster than item 2.
2. **DB-constraint layer (durable fix):** Add a Postgres exclusion constraint using `btree_gist`: `EXCLUDE USING gist (teacher_id WITH =, tsrange(requested_start, requested_end) WITH &&) WHERE (status IN ('PENDING_PAYMENT','AWAITING_TEACHER_REVIEW','ACCEPTED'))`. This makes double-booking impossible even if a future code path forgets the row-lock (e.g., a bulk-import script, an admin action, a future async flow) — the audit's own root-cause language ("insufficient for first-row conflict prevention") points directly at needing a DB-enforced guarantee, not just better application code.

**Frontend changes required:** Handle the new failure mode gracefully — if the DB constraint rejects a booking that the application-level check missed (should be rare after item 1, but the constraint is the backstop), the API should translate the resulting `IntegrityError` into the same `ConflictError`/409 the existing conflict check already returns, so `BookingRequestForm.tsx`'s existing conflict-handling UI doesn't need new code — verify this during implementation rather than assuming.

**Database changes required:** Enable `btree_gist` extension; add the exclusion constraint on `BookingRequest`.

**Migration requirements:**
- `bookings/0006_enable_btree_gist.py` — `CREATE EXTENSION IF NOT EXISTS btree_gist` (via `django.contrib.postgres.operations.BtreeGistExtension`).
- `bookings/0007_bookingrequest_slot_exclusion_constraint.py` — add the exclusion constraint via `django.contrib.postgres.constraints.ExclusionConstraint`.
- **Pre-migration data check required:** before adding the constraint, run a one-off query for existing overlapping active bookings in production data (if any exist from the pre-fix race already having fired), since the migration will fail to apply if current data already violates it. This must be planned as a manual step, not assumed clean.

**Testing requirements:**
- New **threaded/concurrent** test (the audit explicitly noted the existing test is sequential and can't catch this class of bug): spawn two threads/processes calling `create_booking` for the same teacher/overlapping window simultaneously; assert exactly one succeeds and the other raises `ConflictError`.
- Constraint-level test: attempt a raw overlapping insert bypassing the service layer entirely (simulating an admin action or bug elsewhere) — assert the DB itself rejects it.
- Regression: existing sequential conflict test (`tests/bookings/test_services.py:57`) must still pass unchanged.
- Regression: back-to-back non-overlapping bookings for the same teacher must still succeed (constraint must not be over-broad).

**Risk level:** 🟠 Medium — the row-lock half is low-risk and additive; the DB constraint half carries **migration risk** (see Part 6) if production data already contains violations from the pre-fix race window.

**Dependencies:** None on Phases 1-2 — can technically run in parallel, but sequenced here after payment-fund-safety fixes on the principle of fixing "money can be misdirected" before "money can be double-committed," since Phase 1/2 changes are more foundational to every other payment operation.

---

## Phase 4 — Teacher Visibility of Unpaid Bookings *(Audit item 6)*

**Objective:** Enforce, at the data-access layer, the system's own stated design: a teacher cannot see a booking until payment succeeds.

**Problems being solved:** High finding — `list_teacher_bookings`, `get_booking_detail`, and `BookingFilter` all currently allow a teacher to retrieve `PENDING_PAYMENT` (and by extension `DRAFT`, if that status is reachable) booking data.

**Files/modules affected:**
- `apps/bookings/selectors.py` — `list_teacher_bookings`, `get_booking_detail`
- `apps/bookings/filters.py` — `BookingFilter`
- `apps/bookings/views.py` — `TeacherBookingListView`, `BookingDetailView`
- `afra-frontend/src/features/bookings/pages/BookingsPage.tsx` — teacher "All" tab (defensive; backend fix is authoritative)

**Backend changes required:**
1. `list_teacher_bookings`: unconditionally exclude `PENDING_PAYMENT`/`DRAFT` from the base queryset before any caller-supplied filter is applied (i.e., the exclusion must not be bypassable by a status filter parameter).
2. `get_booking_detail`: when the requesting party is the teacher (not the student), raise `NotFoundError` (not `PermissionDenied` — avoid confirming to a teacher that a booking exists in a state they shouldn't know about) for `PENDING_PAYMENT`/`DRAFT` bookings. The student side of this same function must be **unaffected** — a student must still see their own `PENDING_PAYMENT` booking (this is how the frontend's polling UI works today).
3. Consider whether `BookingFilter`'s `status` choices should be narrowed specifically for the teacher-facing endpoint (separate `FilterSet` or a queryset-level restriction on the choices), rather than relying only on the selector-level exclusion, so this can't regress if a selector is refactored later without someone re-discovering this rule.

**Frontend changes required:** None required for correctness (backend now the source of truth), but recommended as defense-in-depth: `BookingsPage.tsx`'s teacher "All" tab could pass an explicit status-exclusion or the frontend could just trust the backend response fully (simpler — remove any client-side assumption that "no status param" means "everything," since it will no longer mean that).

**Database changes required:** None.

**Migration requirements:** None.

**Testing requirements:**
- Test: `GET /api/bookings/teacher/` (no filter) as the booking's own teacher, with a `PENDING_PAYMENT` booking in the DB → not present in results.
- Test: `GET /api/bookings/teacher/?status=PENDING_PAYMENT` explicitly → still excluded (proves the exclusion isn't bypassable via filter param).
- Test: `GET /api/bookings/<pending_payment_booking_id>/` as the teacher → 404, not 403 (avoid existence leak).
- Test: `GET /api/bookings/<pending_payment_booking_id>/` as the **student** → still 200 with full data (regression guard — this must not break the polling UI from the audit's Scenario 2/3).
- Regression: full `tests/bookings/` suite.

**Risk level:** 🟢 Low — pure access-control tightening, no fund movement, small blast radius.

**Dependencies:** None technically, but sequenced after Phases 1-3 since those are fund-correctness/fund-safety issues that outrank a data-exposure issue in priority, per the audit's own severity ordering (Critical > High).

---

## Phase 5 — Admin Panel Hardening & Frontend Messaging Fixes *(Audit items 7, 8)*

**Objective:** Close the last write-path that can bypass the service layer, and fix the two user-facing/staff-facing messaging problems.

**Problems being solved:**
- Medium — `PaymentAdmin`/`BookingRequestAdmin` allow direct `status` field edits, bypassing `PaymentEvent`/ledger/notification side effects.
- Medium — `BookingRequestForm.tsx`'s self-contradictory payment-timing copy.
- Low — `PaymentCard.tsx`'s `provider === "INTERNAL" ? "Free" : "Stripe"` mislabel.

**Files/modules affected:**
- `apps/payments/admin.py`, `apps/bookings/admin.py`
- `afra-frontend/src/features/bookings/components/BookingRequestForm.tsx`
- `afra-frontend/src/features/payments/components/PaymentCard.tsx`

**Backend changes required:**
1. Add `"status"` to `readonly_fields` on both `PaymentAdmin` and `BookingRequestAdmin`, mirroring the existing `has_change_permission`-style discipline already applied to `PaymentEventAdmin`/`PayoutLedgerEntryAdmin`.
2. If staff genuinely need a manual override path (e.g., support resolving a stuck booking), add an explicit Django admin **action** (not a raw field edit) that calls the real service function (`refund_payment`, `release_payment_to_teacher`, etc.) so the audit trail stays intact — only build this if product/support confirms it's needed; otherwise leave it out and require engineering involvement for the rare manual-fix case.

**Frontend changes required:**
1. `BookingRequestForm.tsx`: replace the contradictory two-sentence copy with a single accurate statement (see audit's suggested fix). This should be treated as a copy review, not just a mechanical find/replace — confirm with product/design ownership if one exists.
2. `PaymentCard.tsx`: change `providerLabel` logic to key off `amount === 0` for "Free" rather than `provider === "INTERNAL"`; give `INTERNAL` its own accurate label (e.g. "Processed internally" or whatever term the product decides is user-facing-appropriate — this is a naming decision, not just a bug fix, since "INTERNAL"/"simulated" language may not be something the product wants surfaced to real users at all if/when this becomes the production path per Phase 1/2).

**Database changes required:** None.

**Migration requirements:** None.

**Testing requirements:**
- Admin test: attempt to change `Payment.status` via the Django admin change form → field is read-only, submitted value has no effect (or the form doesn't render it as editable, per Django admin test conventions).
- Same for `BookingRequest.status`.
- Frontend snapshot/unit test: `BookingRequestForm` renders the new copy (simple regression guard against the sentence drifting back out of sync if edited again later).
- Frontend unit test: `PaymentCard` with `amount=0` shows "Free"; `amount=50, provider=INTERNAL` does not show "Free".

**Risk level:** 🟢 Low — admin change is a permissions tightening with no runtime/user-facing behavior change (assuming no admin workflow currently depends on hand-editing status, which should be confirmed with whoever operates the admin panel before shipping — see Part 6). Frontend changes are copy/label-only.

**Dependencies:** None functionally, but sequenced after Phase 1 for the `PaymentCard` label decision specifically — once Phase 1 clarifies what "INTERNAL" actually means going forward (is it staying as a real second production mode, or is it purely test/dev?), the correct replacement label can be chosen deliberately rather than guessed.

---

## Phase 6 — Legacy Code Cleanup & Operational Safeguards *(Audit items 9, 10)*

**Objective:** Remove the source of the confusion that produced Phase 1's root cause, and add the operational sweeps the audit identified as missing so future regressions are caught automatically rather than by a manual audit.

**Problems being solved:**
- Low — dead `process_payment`/`_process_payment_via_stripe` Celery machinery that's never dispatched, but is the *only* correctly-`PAYMENTS_USE_STRIPE`-gated code in the file, misleading future readers.
- Operational gaps identified in the audit's "targeted answers" section: no reaper for bookings stuck `PENDING_PAYMENT` if a Celery task is lost; no reconciliation invariant enforced continuously (Phase 0 built the *tool* — this phase makes it run on a schedule and alert).

**Files/modules affected:**
- `apps/payments/services.py`, `apps/payments/tasks.py` — remove or explicitly unify `process_payment`
- New: `apps/bookings/tasks.py` addition — `reap_stale_pending_payment_bookings`
- `config/celery.py` — Beat schedule additions
- `apps/payments/management/commands/reconcile_payments.py` (from Phase 0) — promote to a scheduled task with alerting

**Backend changes required:**
1. **Decision point (needs a team call, flagged explicitly rather than assumed):** either (a) delete `process_payment`/`_process_payment_via_stripe`/`_process_payment_simulated` and the associated Celery task entirely, since `capture_payment_for_booking` is confirmed the sole live path, or (b) unify them — make `capture_payment_for_booking` internally call the now-correctly-gated `process_payment` logic instead of maintaining two separate simulated/Stripe branches. Option (b) is architecturally cleaner (single source of truth for "how do we capture a payment, real or simulated") but is a larger diff touching the code Phase 1 just modified — recommend option (a) for this phase to minimize churn, with (b) as a documented future improvement if the team wants a real unification later.
2. New Beat task: sweep `BookingRequest.status == "PENDING_PAYMENT"` older than a short threshold (minutes — configurable, e.g. `PAYMENTS_STUCK_CAPTURE_THRESHOLD_MINUTES`), with **no** associated `Payment` row at all (distinguishing "capture task never ran" from "capture is legitimately still in flight") → cancel the booking and emit an alertable log/metric, since this indicates infrastructure failure (lost Celery task), not normal business flow.
3. Promote Phase 0's reconciliation command to a scheduled Beat task with alerting (not just a manually-run management command) — e.g., daily, emitting a metric/log an on-call engineer would see.

**Frontend changes required:** None.

**Database changes required:** None.

**Migration requirements:** None (unless the decision-point cleanup removes model fields no longer used — confirm none of `process_payment`'s removed code path was the only writer of any field still read elsewhere before deleting).

**Testing requirements:**
- If deleting: confirm via `grep`/CI check that no remaining code references the removed functions/task (a simple "dead code" CI lint is sufter than trusting manual review a second time, given this is exactly how the audit found the confusion in the first place).
- New task test: a `PENDING_PAYMENT` booking older than the threshold with no `Payment` row → reaped/cancelled. A `PENDING_PAYMENT` booking younger than the threshold → left alone (not prematurely cancelled while capture may still be legitimately in flight).
- New task test: a `PENDING_PAYMENT` booking with a `Payment` row already present (capture genuinely still processing, e.g. the Stripe path from Phase 1/2 in flight) → **not** reaped, even if past the threshold — this distinction must be explicit and tested, not incidental.
- Reconciliation task test: seeded invariant violation → task surfaces it (log/metric assertion, not just "doesn't crash").

**Risk level:** 🟢 Low-to-Medium — deletion carries a small "did we miss a caller" risk (mitigated by the dead-code CI check above); the new reaper task is additive and only touches a status (`PENDING_PAYMENT`→`CANCELLED`) that's already a fully supported terminal-for-that-booking transition.

**Dependencies:** Phase 1 (needs the provider dispatch settled before deciding how to unify/delete the legacy Stripe-gating code, since deleting it prematurely could remove the reference implementation Phase 1 is modeled on). Phase 0 (reuses its reconciliation tool).

---

# PART 2 — Target Architecture

## 2.1 Payment capture

**Principle:** Capture is provider-aware from the moment `Payment` is created — `provider` is not just a label, it determines every downstream code path (release, refund-to-source, dispute resolution).

```
create_booking()
   │  (atomic: teacher-row lock closes the double-booking race — Phase 3)
   ▼
BookingRequest(status=PENDING_PAYMENT)
   │  on_commit
   ▼
capture_payment_for_booking (Celery, retryable, idempotent under row lock)
   │
   ├─ settings.PAYMENTS_USE_STRIPE == True
   │      → create real stripe.PaymentIntent, Payment(provider="STRIPE", gateway_charge_id=<intent id>)
   │
   └─ settings.PAYMENTS_USE_STRIPE == False
          → simulated capture, Payment(provider="INTERNAL", gateway_charge_id=NULL)

   Both branches converge on the same state transition:
   success → Payment.status = HELD_IN_ESCROW, Booking.status = AWAITING_TEACHER_REVIEW
   failure → Payment.status = FAILED, Booking.status = CANCELLED
```

`provider` and `gateway_charge_id` are then **load-bearing** for every later operation, not just descriptive metadata — this is the core architectural correction from the audit.

## 2.2 Escrow

Unchanged in spirit from today's design (which the audit found to be sound), corrected in mechanics:
- `HELD_IN_ESCROW` is the resting state from successful capture through session completion + hold window.
- Release is **exclusively** Beat-sweep-driven (`release_held_payments`), never user-triggered — this was already correct and stays.
- Release is blocked by: open/under-review `Dispute`, confirmed `NO_SHOW_TEACHER` outcome (both already correct), **and now also** a `provider`/`gateway_charge_id` consistency check (Phase 1) that refuses to proceed rather than silently doing the wrong thing.

## 2.3 Teacher payout

```
release_payment_to_teacher(payment)
   │
   ├─ payment.provider == "STRIPE"
   │     → require teacher_profile.is_payout_ready (real Connect onboarding,
   │       now reachable via the account.updated webhook — Phase 2)
   │     → stripe.Transfer.create(...), gated by settings.PAYMENTS_USE_STRIPE
   │       as a defense-in-depth belt even though provider already implies it
   │
   └─ payment.provider == "INTERNAL"
         → no external call, no Stripe-readiness requirement at all
         → ledger-only transition to RELEASED + PayoutLedgerEntry
```

This is the single most important structural change: **"is this teacher ready to be paid" and "did we actually collect this specific payment through Stripe" become two independent questions**, where today they're incorrectly conflated into one gate that blocks everything.

## 2.4 Stripe integration

- Platform-level events (`payment_intent.succeeded/failed`, `charge.dispute.created`) — existing webhook, unchanged, already correctly signature-verified and idempotent.
- Connect-level events (`account.updated`) — new handler (Phase 2), ideally on a **separate webhook endpoint/secret** per Stripe's own recommended practice for platform vs. connected-account events; this needs a decision from whoever owns the Stripe account configuration, not an assumption baked into code.
- Stripe is **never** called from a code path that doesn't already know, from `payment.provider`, that this specific payment went through Stripe.

## 2.5 INTERNAL/testing mode

Explicitly promoted from "an implicit side-effect of `PAYMENTS_USE_STRIPE=False`" to a **first-class, fully self-contained payment provider**:
- Capture: simulated success/failure, no external calls.
- Release: ledger-only, no external calls, no Stripe-readiness requirement.
- Refund: ledger-only (already true today — `refund_payment` doesn't call Stripe for `INTERNAL` payments in the current code either, this part was already correct).
- Frontend label: something accurate and deliberately chosen (Phase 5) — not silently implying "Free," not silently implying "Stripe."

**This is the environment separation the roadmap treats as central:**

| | Development/Test environment | Production environment |
|---|---|---|
| `PAYMENTS_USE_STRIPE` | `False` (default) | `True` |
| Capture | Simulated coin-flip, `provider=INTERNAL` | Real `stripe.PaymentIntent`, `provider=STRIPE` |
| Release | Ledger-only, Phase 1/2's `INTERNAL` branch | Real `stripe.Transfer`, gated + payout-ready-checked |
| Teacher onboarding | Not required at all | Required, driven by `account.updated` webhook (Phase 2) |
| Webhooks | Not exercised by the capture path (nothing to receive events *about*) | Fully exercised; both platform and Connect event types |
| Reconciliation (Phase 0/6) | Still runs — catches `INTERNAL` rows with an erroneous `gateway_charge_id`, which would indicate a config/branch bug | Still runs — catches drift between captured/released/refunded sums |

The critical rule this table is meant to encode: **a payment's `provider` field, once set at capture time, is what every later stage trusts — never the global `settings.PAYMENTS_USE_STRIPE` flag at release time.** Today's bug is exactly that release-time code reads the wrong signal (implicitly assumes Stripe-always, or doesn't check at all) instead of the per-row signal set at capture time. This distinction — **per-row provider vs. global settings flag** — is the one sentence that summarizes the entire Phase 1/2 fix.

## 2.6 Booking state machine (target — same as audit-documented actual, confirmed correct, restated here for completeness)

```
                 ┌──────────────────┐
                 │  PENDING_PAYMENT  │◀── create_booking()
                 └─────────┬─────────┘
              capture fails │ capture succeeds
              ┌─────────────┴─────────────┐
              ▼                           ▼
        ┌───────────┐        ┌─────────────────────────┐
        │ CANCELLED │        │ AWAITING_TEACHER_REVIEW  │
        └───────────┘        └────────────┬─────────────┘
                         teacher accepts │ teacher rejects / SLA expiry / student cancels
                    ┌────────────────────┴────────────────────┐
                    ▼                                          ▼
              ┌───────────┐                             ┌────────────┐
              │ ACCEPTED  │                             │  REJECTED  │
              └─────┬─────┘                             └────────────┘
                    │  (Session lifecycle drives completion —
                    │   Booking itself has no explicit COMPLETED
                    │   state; Session.status carries that)
                    ▼
             Session SCHEDULED → ONGOING → COMPLETED
```
**No change recommended to this state machine** — the audit found it logically sound; Phase 4's fix is purely about *who can see* a `PENDING_PAYMENT` node, not about adding/removing states.

## 2.7 Payment state machine (target)

```
        ┌─────────┐
        │ PENDING │  (capture in flight)
        └────┬────┘
    success │ failure
   ┌─────────┴─────────┐
   ▼                   ▼
┌───────────────┐  ┌────────┐
│ HELD_IN_ESCROW │  │ FAILED │  (terminal)
└───────┬────────┘  └────────┘
        │
  ┌─────┼───────────────────┬────────────────────┐
  │ hold window elapses     │ refund requested    │ dispute opened
  ▼                         ▼                     ▼
┌──────────┐          ┌───────────┐         ┌───────────┐
│ RELEASED │          │ REFUNDED  │         │ DISPUTED  │
└────┬─────┘          └───────────┘         └─────┬─────┘
     │ chargeback / late dispute on already-released funds        │ resolved
     ▼                                                              ▼
┌───────────────────────┐                              RELEASED / REFUNDED / split
│ CLAWBACK ledger entry  │                              (per resolve_dispute outcome —
│ (Payment stays RELEASED,│                              already correctly modeled today)
│  history not rewritten) │
└───────────────────────┘
```
Also unchanged from audit-documented actual — this part of the design was already assessed as sound (the "distinct `CLAWBACK` entry rather than mutating history" pattern is exactly right and should be preserved, not touched, by any of the above phases).

## 2.8 Payment provider decision flow

```
                    capture_payment_for_booking()
                              │
                    settings.PAYMENTS_USE_STRIPE?
                    ┌─────────┴─────────┐
                   True                False
                    │                    │
                    ▼                    ▼
         provider = "STRIPE"     provider = "INTERNAL"
         gateway_charge_id set   gateway_charge_id = NULL
                    │                    │
                    └─────────┬──────────┘
                              ▼
              Payment.provider is now IMMUTABLE and
              is the sole signal every later function
              (release, refund-to-source, dispute
              resolution) branches on — settings.
              PAYMENTS_USE_STRIPE is never consulted
              again after this point.
```

---

# PART 3 — Implementation Order & Rationale

```
Phase 0 → Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6
```

**Why this order is safest:**

1. **Phase 0 first, unconditionally.** You cannot safely verify any fund-movement change (Phases 1–2) without a reconciliation check already in place to catch a regression. Building the safety net before the risky work, not after, is the only way "verify Phase 1 didn't break anything" is answerable rather than assumed.

2. **Phase 1 before Phase 2.** Phase 2's entire justification ("decouple `INTERNAL` release from Stripe payout-readiness") only makes sense once Phase 1 has introduced the provider-based dispatch to decouple *into*. Doing Phase 2 first would mean patching the old unified path twice.

3. **Phase 1–2 (fund-movement correctness) before Phase 3 (double-booking).** This is a deliberate priority call, not a technical dependency: the audit rates both Critical, but a double-booking race produces a *support/business* problem (two students paid for one slot, needs manual resolution) whereas the fund-movement bugs produce a *financial-integrity* problem (money can be misdirected or permanently stuck) — the latter is fixed first because its blast radius compounds silently over time (every payment, forever, until fixed) while the former only compounds at the (much lower) rate of actual concurrent-booking collisions.

4. **Phase 3 before Phase 4.** No strict dependency, but Phase 3 involves a schema migration with a real pre-migration data-cleanliness risk (see Part 6) — sequencing it before the purely-additive Phase 4 means if Phase 3 needs to pause for a manual data-cleanup step, Phase 4 isn't blocked behind it unnecessarily... actually, reversed reasoning: keeping Phase 3 *before* Phase 4 in the numbered list, but noting they have **no hard dependency**, means a team under time pressure could parallelize Phase 4 alongside Phase 3's migration-risk investigation without violating the plan.

5. **Phase 4 before Phase 5.** Both are low-risk, but Phase 4 (data exposure) is a High-severity audit finding while Phase 5's remaining items are Medium/Low — severity ordering resumes once the two Critical, higher-priority-by-blast-radius items (1–2) are done.

6. **Phase 5 before Phase 6.** Phase 6's legacy-code decision (delete vs. unify `process_payment`) is explicitly easier to make correctly once Phase 5 has settled the `PaymentCard` labeling question, since both touch "what does `INTERNAL`/simulated mode mean to the product, going forward" — resolving the user-facing question first informs the internal-code-cleanup question.

7. **Phase 6 last.** It's pure cleanup and operational hardening with no urgent user- or fund-impact — appropriately last, and it also depends on Phase 1 being fully settled (see Phase 6's dependency note) so the legacy code isn't deleted out from under a still-in-progress Phase 1 change.

---

# PART 4 — AI Coding Prompts (one per phase)

Each prompt below is written to be handed to an AI coding assistant directly, with no additional context needed beyond repo access.

---

### 🤖 Coding Prompt — Phase 0

```
CONTEXT
An audit of this payment system found no reconciliation invariant anywhere:
nothing checks that sum(Payment.amount where status=HELD_IN_ESCROW) +
sum(released) + sum(refunded) == sum(captured), and nothing checks that a
Payment's provider field is internally consistent with whether it has a
gateway_charge_id set. Before any fund-movement code is touched (upcoming
phases), we need a read-only tool that can verify correctness before and
after each change.

CURRENT BEHAVIOR
No reconciliation tooling exists in apps/payments/ at all.

EXPECTED BEHAVIOR
A Django management command `reconcile_payments` that:
1. For every Payment row, asserts: provider == "STRIPE" implies
   gateway_charge_id is non-null; provider == "INTERNAL" implies
   gateway_charge_id IS null. Report any violating row's id.
2. Computes, per currency, the sum of Payment.amount grouped by status
   bucket, and reports the totals (this is an observability report, not
   a hard assertion yet — flag any bucket that looks anomalous, e.g.
   HELD_IN_ESCROW rows whose booking's session ended more than
   PAYMENTS_ESCROW_HOLD_WINDOW_HOURS * 3 ago, which would suggest release
   is stuck).
3. Exits non-zero if any provider/gateway_charge_id inconsistency is found
   (this is the hard-fail condition); exits zero (with a printed report)
   otherwise, even if there are stuck-looking HELD_IN_ESCROW rows (that's
   informational for now, not a hard failure, since Phase 1/2 haven't
   shipped yet to actually fix those).

FILES TO INSPECT FIRST
- apps/payments/models.py (Payment, PaymentEvent, PayoutLedgerEntry)
- apps/payments/services.py (to understand existing status semantics -
  do not modify this file in this phase)
- apps/payments/admin.py (for reference on how existing management-style
  read-only reporting is structured in this codebase, if any exists)

IMPLEMENTATION REQUIRED
- New file: apps/payments/management/commands/reconcile_payments.py
  (create apps/payments/management/ and
  apps/payments/management/commands/ dirs with __init__.py files if they
  don't exist)
- Use Django's BaseCommand, structured logging (not print, unless the
  codebase's existing convention is print for management commands -
  check other apps' management commands first if any exist)

CONSTRAINTS
- This command must be strictly read-only: no .save(), no .update(), no
  state mutation of any kind. It only queries and reports.
- Do not modify apps/payments/services.py, apps/payments/tasks.py, or any
  other existing file in this phase - this is purely additive.

TESTS TO ADD
- tests/payments/test_reconcile_command.py:
  - A payment with provider=STRIPE and gateway_charge_id set: reported
    as consistent, command exits 0.
  - A payment with provider=STRIPE and gateway_charge_id=None: reported
    as a violation, command exits non-zero.
  - A payment with provider=INTERNAL and gateway_charge_id set (should
    never happen today, but the command must catch it if it does):
    reported as a violation.
  - A mix of HELD_IN_ESCROW/RELEASED/REFUNDED payments: the per-status
    sum report is numerically correct (assert against manually
    calculated expected totals in the test fixture).

ACCEPTANCE CRITERIA
- `python manage.py reconcile_payments` runs successfully against the
  existing test fixtures/factories with no violations reported.
- Command exit code is 0 when clean, non-zero when a provider/
  gateway_charge_id inconsistency exists.
- No existing test in the suite is broken (this phase touches no
  existing file).
```

---

### 🤖 Coding Prompt — Phase 1

```
CONTEXT
An audit found that apps/payments/services.py's release_payment_to_teacher
(~line 855-1013) and resolve_dispute's RESOLVED_TEACHER/RESOLVED_SPLIT
branches (~line 1363-1411) call stripe.Transfer.create() unconditionally -
with no check on settings.PAYMENTS_USE_STRIPE and no check on the specific
Payment's `provider` field. Meanwhile, the only path that actually creates
escrow payments today, capture_payment_for_booking, always uses
provider="INTERNAL" with gateway_charge_id=None (it never calls Stripe at
all). This means release logic will either always fail
(no valid Stripe context) or, worse, attempt to move real Stripe balance
for a payment that was never actually collected through Stripe - a fund-
conservation violation.

CURRENT BEHAVIOR
release_payment_to_teacher and resolve_dispute call stripe.Transfer.create
unconditionally for every payment being released/resolved in the
teacher's favor, regardless of that payment's `provider` field.

EXPECTED BEHAVIOR
release_payment_to_teacher and resolve_dispute's teacher-favoring branches
must branch on `payment.provider` BEFORE deciding whether to call Stripe:
- provider == "STRIPE": existing Stripe Transfer.create logic, but ALSO
  now guarded by `if not settings.PAYMENTS_USE_STRIPE: raise
  ApplicationError(...)` as a defense-in-depth check (a STRIPE-provider
  payment should never exist if PAYMENTS_USE_STRIPE was ever False, but
  guard against it explicitly rather than trusting that invariant
  silently).
- provider == "INTERNAL": skip Stripe entirely. Perform only the DB-side
  effects that currently happen alongside the Stripe call today (status
  transition to RELEASED, PayoutLedgerEntry creation, commission
  calculation, notification dispatch) - inspect the existing function
  carefully to identify exactly which side effects are Stripe-specific
  (skip those) vs. which are the actual business-logic side effects that
  must happen regardless of provider (keep those, in a shared code path
  or a clearly-named internal helper called by both branches).
- Any other provider value (defensive): raise ApplicationError, do not
  silently no-op.

Additionally, before attempting a STRIPE-branch release, hard-validate:
`if payment.provider == "STRIPE" and not payment.gateway_charge_id: raise
ApplicationError("Cannot release a STRIPE payment with no
gateway_charge_id")`. This must fire before any Stripe API call is
attempted.

FILES TO INSPECT FIRST
- apps/payments/services.py in full - especially
  release_payment_to_teacher, resolve_dispute, capture_payment_for_
  booking, and process_payment/_process_payment_via_stripe (the one
  place in this file that already correctly gates Stripe calls behind
  settings.PAYMENTS_USE_STRIPE - use this as your reference pattern for
  how gating should look, even though process_payment itself is legacy/
  unused and will be addressed in a later phase - do not modify or
  remove it in this phase)
- apps/payments/models.py (Payment.provider choices, Payment.
  gateway_charge_id)
- tests/payments/test_services.py - review how existing tests currently
  mock stripe.Transfer.create (they patch it unconditionally today; your
  change means it should no longer be called for INTERNAL-provider test
  fixtures, so review whether existing tests' assertions need updating
  to explicitly assert non-invocation for INTERNAL cases)

IMPLEMENTATION REQUIRED
- Refactor release_payment_to_teacher to dispatch on payment.provider as
  described above.
- Refactor resolve_dispute's RESOLVED_TEACHER and RESOLVED_SPLIT branches
  identically.
- Extract the shared (non-Stripe-specific) side effects into a private
  helper if release_payment_to_teacher's current structure makes that
  the cleanest way to avoid duplicating logic between the STRIPE and
  INTERNAL branches - use your judgment on the cleanest structure, but
  do not duplicate the commission-calculation logic between branches.

CONSTRAINTS
- Do not change the public signature of release_payment_to_teacher or
  resolve_dispute - callers (apps/payments/tasks.py's
  release_held_payments, and the dispute-resolution view/serializer)
  must be unaffected.
- Do not change Payment.STATUS choices or any other model field in this
  phase - this is a service-layer logic change only.
- Do not touch apps/payments/tasks.py, apps/teachers/*, apps/bookings/*,
  or any frontend file in this phase.
- Preserve every existing side effect (notifications, ledger entries,
  logging) for the STRIPE branch exactly as-is - only the INTERNAL
  branch is new behavior.

TESTS TO ADD (in tests/payments/test_services.py)
- release_payment_to_teacher on a provider="INTERNAL" HELD_IN_ESCROW
  payment: assert final status is RELEASED, a PayoutLedgerEntry was
  created with correct amounts, and mock_stripe.Transfer.create was
  NEVER called.
- release_payment_to_teacher on a provider="STRIPE" HELD_IN_ESCROW
  payment with a valid gateway_charge_id and PAYMENTS_USE_STRIPE=True:
  existing behavior preserved (Stripe called, ledger entry created).
- release_payment_to_teacher on a provider="STRIPE" payment with
  gateway_charge_id=None: raises ApplicationError, Stripe never called.
- release_payment_to_teacher on a provider="STRIPE" payment while
  settings.PAYMENTS_USE_STRIPE=False (override in test): raises
  ApplicationError, Stripe never called.
- resolve_dispute RESOLVED_TEACHER on an INTERNAL-provider disputed
  payment: same INTERNAL-branch assertions as above.
- resolve_dispute RESOLVED_SPLIT on an INTERNAL-provider disputed
  payment: split ledger entries created correctly with no Stripe call.
- Full regression run of the existing tests/payments/test_services.py
  and tests/payments/test_disputes.py suites - fix any test that was
  relying on the old unconditional Stripe-mock behavior by making its
  fixture's provider explicit.

ACCEPTANCE CRITERIA
- All new tests pass.
- Full existing payments test suite passes (with any necessary fixture-
  level provider= updates, not logic changes to the tests' assertions
  about business outcomes).
- Manually run `python manage.py reconcile_payments` (from Phase 0)
  against a test database seeded with a mix of provider values after
  exercising both release branches - zero violations reported.
- grep confirms stripe.Transfer.create is never reached in any code
  path where payment.provider != "STRIPE".
```

---

### 🤖 Coding Prompt — Phase 2

```
CONTEXT
An audit found that TeacherProfile.is_payout_ready (required by
release_payment_to_teacher's STRIPE branch, per Phase 1's work) can never
become True in production: it requires stripe_onboarding_status ==
"COMPLETE", but the only function that sets that
(mark_stripe_onboarding_complete, in apps/teachers/services.py) is never
called from any reachable code path - only from its own test file. Its
docstring incorrectly claims an "onboarding-return endpoint" calls it;
no such endpoint exists (confirmed: apps/teachers/views.py only has
start_stripe_onboarding and a read-only status endpoint). There is also
no Stripe `account.updated` webhook handler in apps/payments/webhooks.py
(which currently only handles payment_intent.succeeded,
payment_intent.payment_failed, and charge.dispute.created).

CURRENT BEHAVIOR
stripe_onboarding_status can be set to "PENDING" (by
start_stripe_onboarding) but never to "COMPLETE" through any production
code path.

EXPECTED BEHAVIOR
Add an `account.updated` handler to the existing Stripe webhook view that:
1. Verifies the event the same way existing handlers do (this view
   already does correct signature verification via
   stripe.Webhook.construct_event - do not change that verification
   logic, only add a new event-type branch to what happens after
   verification succeeds).
2. Looks up TeacherProfile by the event's account id (`event["account"]`
   or `event["data"]["object"]["id"]` - inspect Stripe's actual
   account.updated payload shape before implementing, don't guess the
   field path).
3. If the account object's `charges_enabled` and `payouts_enabled` are
   both true: call mark_stripe_onboarding_complete for that
   TeacherProfile.
4. If the account object indicates the account is no longer fully
   enabled (charges_enabled or payouts_enabled is now false, e.g. via
   `requirements.disabled_reason` being set): set
   stripe_onboarding_status back to "PENDING" (add this as a new
   function in apps/teachers/services.py,
   e.g. `mark_stripe_onboarding_incomplete`, following the same pattern
   as the existing mark_stripe_onboarding_complete).
5. If no TeacherProfile matches the account id in the payload: log a
   warning and return 200 (mirror the existing pattern in
   handle_payment_intent_succeeded for an unknown payment id - do not
   error/500 on this case).

Also fix the incorrect docstring on mark_stripe_onboarding_complete once
it's actually wired up - update it to describe the real caller (the new
webhook handler) instead of the nonexistent "onboarding-return endpoint."

FILES TO INSPECT FIRST
- apps/payments/webhooks.py in full (StripeWebhookView, and how the
  three existing event types are each dispatched and handled - follow
  this exact pattern for the new event type)
- apps/teachers/services.py (start_stripe_onboarding,
  mark_stripe_onboarding_complete)
- apps/teachers/views.py and apps/teachers/urls.py (to confirm there is
  genuinely no other onboarding-completion caller before adding this one
  - do not assume, verify)
- apps/users/models.py (TeacherProfile.stripe_account_id,
  stripe_onboarding_status, is_payout_ready)
- config/settings/base.py (STRIPE_WEBHOOK_SECRET - determine whether
  this phase should introduce a second webhook secret setting for
  Connect events, e.g. STRIPE_CONNECT_WEBHOOK_SECRET, per Stripe's own
  recommendation to use separate endpoints/secrets for platform vs.
  connected-account events - if the existing single-secret setup is
  intentional and documented elsewhere, keep it single; if introducing a
  second secret, add it as a new setting with a safe default that
  doesn't break existing deployments that haven't configured it, and
  flag this as a config change needed in deployment docs/env vars)

IMPLEMENTATION REQUIRED
- New event-type branch in StripeWebhookView's dispatch logic for
  "account.updated".
- New handler function handle_account_updated(event) in
  apps/payments/webhooks.py, following the existing handlers' structure.
- New apps/teachers/services.py function
  mark_stripe_onboarding_incomplete (or an equivalent name matching
  existing naming conventions in that file).

CONSTRAINTS
- Do not modify the existing payment_intent.* or charge.dispute.*
  handlers or the signature-verification logic - additive only.
- Do not change TeacherProfile.is_payout_ready's definition in this
  phase (it should already correctly reflect stripe_account_id +
  stripe_onboarding_status=="COMPLETE" - if Phase 1's work requires this
  property to only be consulted for provider=="STRIPE" payments, that's
  a services.py concern already handled in Phase 1, not a change to this
  property itself).
- If introducing a new webhook secret setting, it must have a safe
  default (e.g. falling back to STRIPE_WEBHOOK_SECRET if unset) so
  existing deployments aren't broken by this change alone - flag in your
  output whether you did this and why.

TESTS TO ADD (in tests/payments/ - follow existing webhook test file's
patterns)
- account.updated event with charges_enabled=True, payouts_enabled=True
  for a known stripe_account_id -> TeacherProfile.
  stripe_onboarding_status == "COMPLETE" after.
- account.updated event with charges_enabled=False -> status remains/
  becomes "PENDING", not "COMPLETE".
- account.updated event for a previously-COMPLETE account that's now
  disabled (charges_enabled flips to False) -> status moves back to
  "PENDING".
- account.updated event for an unknown stripe_account_id -> 200
  response, warning logged, no exception raised.
- Invalid signature on an account.updated payload -> 400, same as
  existing invalid-signature tests for other event types.
- Integration test spanning apps/teachers and apps/payments: full flow
  from start_stripe_onboarding -> simulated account.updated webhook ->
  is_payout_ready becomes True -> release_payment_to_teacher (from
  Phase 1's STRIPE branch) succeeds end-to-end.

ACCEPTANCE CRITERIA
- All new tests pass; full existing webhook and teachers test suites
  pass unchanged.
- A STRIPE-provider payment for a teacher who has NOT completed
  onboarding still correctly raises PayoutBlockedError at release time
  (this behavior must be preserved, not accidentally removed).
- An INTERNAL-provider payment's release (per Phase 1) is completely
  unaffected by this phase's changes - confirm with a test that
  INTERNAL-provider release still works with is_payout_ready == False.
```

---

### 🤖 Coding Prompt — Phase 3

```
CONTEXT
An audit found a real, reproducible race condition: apps/bookings/
services.py's _check_for_conflicting_booking (~line 91-117) uses
select_for_update() on a query filtered to existing conflicting
BookingRequest rows. When no conflicting booking exists yet (the normal
case - the first request for a currently-open slot), select_for_update()
on a zero-row result locks nothing. Two concurrent requests for the same
teacher/time-window can both pass this check and both successfully
create a BookingRequest, resulting in two students holding escrow on the
same slot. Postgres's default isolation is READ COMMITTED (confirmed in
config/settings/base.py - no isolation override).

CURRENT BEHAVIOR
_check_for_conflicting_booking only prevents a race once a conflicting
row already exists to lock. The first booking into an open slot has no
concurrency protection.

EXPECTED BEHAVIOR
Two layers of protection, both required:

LAYER 1 (application-level, ships first, no migration needed):
Before running the conflict-existence check, acquire a
select_for_update() lock on the relevant TeacherProfile row (this row is
guaranteed to already exist, unlike a not-yet-created conflicting
booking), serializing all concurrent create_booking calls for that
specific teacher. Inspect create_booking's current transaction structure
carefully - this lock must be taken inside the same atomic block that
currently wraps the conflict check and the BookingRequest creation, and
must be released only when that transaction commits/rolls back (i.e.
standard select_for_update usage - do not hold the lock longer than
necessary, and do not lock a teacher-independent resource that would
serialize unrelated teachers' bookings against each other).

LAYER 2 (database-level, requires migration):
Add a PostgreSQL exclusion constraint on BookingRequest that makes
overlapping active bookings for the same teacher impossible at the
schema level, independent of application code:
EXCLUDE USING gist (teacher_id WITH =, tsrange(requested_start,
requested_end) WITH &&) WHERE (status IN ('PENDING_PAYMENT',
'AWAITING_TEACHER_REVIEW', 'ACCEPTED'))
This requires the btree_gist Postgres extension. Use Django's
django.contrib.postgres.operations.BtreeGistExtension and
django.contrib.postgres.constraints.ExclusionConstraint.

The API layer must translate a resulting django.db.utils.IntegrityError
(from the exclusion constraint firing) into the same ConflictError/409
response that the existing application-level check already produces -
find where ConflictError is currently raised/caught in create_booking
and apps/bookings/views.py, and wrap the BookingRequest.objects.create()
call (or the surrounding transaction) in a try/except that catches the
constraint violation specifically (inspect the IntegrityError's
attributes/message to distinguish this specific constraint violation
from any other possible IntegrityError, rather than catching
IntegrityError broadly) and re-raises as ConflictError.

FILES TO INSPECT FIRST
- apps/bookings/services.py (create_booking, _check_for_conflicting_
  booking, _ACTIVE_BOOKING_STATUSES)
- apps/bookings/models.py (BookingRequest - field names for
  requested_start, requested_end, teacher, status)
- apps/bookings/views.py (how ConflictError is currently translated to
  an HTTP response, so the new IntegrityError-to-ConflictError
  translation is consistent with existing error-handling conventions in
  this codebase)
- apps/bookings/migrations/ (current head is 0005 - your new migrations
  will be 0006 and 0007)
- config/settings/base.py (confirm DATABASES engine is postgresql -
  this feature is Postgres-specific and must fail loudly/be skipped
  appropriately if run against a different backend, e.g. sqlite in some
  test configurations - check apps/bookings/migrations/ and test
  settings for whether tests run against Postgres or sqlite, since
  ExclusionConstraint is not supported on sqlite and this could break
  the test suite if it runs on sqlite)
- tests/bookings/test_services.py (existing sequential conflict test,
  line ~57, to match conventions)

IMPLEMENTATION REQUIRED
- Add select_for_update() on TeacherProfile inside create_booking's
  existing transaction, before the conflict check.
- New migration bookings/0006: enable btree_gist extension.
- New migration bookings/0007: add the exclusion constraint described
  above.
- Update create_booking / the view layer to translate the constraint's
  IntegrityError into ConflictError.
- IMPORTANT - pre-migration data check: before writing the constraint
  migration as a straightforward AddConstraint, first write and run a
  read-only query against representative data to check whether any
  existing overlapping active BookingRequest rows already violate the
  intended constraint (this could exist if the pre-fix race has already
  fired in this environment). Report this explicitly in your output
  rather than silently assuming the migration will apply cleanly - if
  violations are found, the migration cannot be safely applied as a
  single step and this must be flagged back to the team, not resolved
  unilaterally (e.g. do not auto-cancel/auto-resolve conflicting
  bookings without explicit sign-off, since that's a business decision
  about which of two paid students keeps their slot).

CONSTRAINTS
- Do not weaken the existing _ACTIVE_BOOKING_STATUSES-based logic - the
  exclusion constraint's WHERE clause must match exactly what the
  application-level check currently treats as "active" (PENDING_PAYMENT,
  AWAITING_TEACHER_REVIEW, ACCEPTED - verify this list against the
  actual current value of _ACTIVE_BOOKING_STATUSES in the code, don't
  assume the three listed here are exactly right without checking).
- Do not change requested_start/requested_end field types or any other
  unrelated model field.
- The TeacherProfile row-lock (Layer 1) must not be held across any
  I/O-bound operation (no external API calls, no Celery dispatch) inside
  the lock's scope - keep the locked section as short as possible,
  consistent with this codebase's existing select_for_update usage
  patterns elsewhere (e.g. how capture_payment_for_booking or other
  service functions already scope their locks).

TESTS TO ADD
- New CONCURRENT/threaded test (not sequential) in
  tests/bookings/test_services.py: spawn two threads (or use pytest-
  django's transaction test case patterns appropriate for testing
  select_for_update races, e.g. two separate DB connections/threads each
  wrapped in their own transaction) both calling create_booking for the
  same teacher and an overlapping requested_start/requested_end at
  effectively the same time. Assert exactly one succeeds and the other
  raises ConflictError. This is the test that would have caught the
  original bug - it must actually exercise concurrency, not just call
  the function twice sequentially.
- Constraint-level test: bypass the service layer and attempt a raw
  BookingRequest.objects.create() (or bulk_create) for two overlapping
  active bookings for the same teacher - assert the database itself
  rejects the second one via IntegrityError, proving the constraint
  works independent of application code.
- Regression: the existing sequential conflict test at line ~57 still
  passes unchanged.
- Regression: two non-overlapping bookings for the same teacher still
  both succeed (constraint is not over-broad).
- Regression: two overlapping bookings for the same teacher where one is
  already REJECTED/CANCELLED still both succeed (constraint's WHERE
  clause correctly excludes inactive statuses).

ACCEPTANCE CRITERIA
- The new concurrent test passes reliably (run it multiple times / in a
  loop during development to confirm it's not flaky and is actually
  catching the race, not passing by luck of thread scheduling).
- Migration applies cleanly against the actual test database.
- Full existing tests/bookings/ suite passes.
- python manage.py reconcile_payments (Phase 0) still reports clean
  after running the new concurrent test's scenario.
```

---

### 🤖 Coding Prompt — Phase 4

```
CONTEXT
An audit found that although teachers correctly cannot ACT on a
PENDING_PAYMENT booking (accept_booking/reject_booking already
correctly restrict this via _TEACHER_ACTIONABLE_STATUSES), they CAN see
one: apps/bookings/selectors.py's list_teacher_bookings and
get_booking_detail apply no default status exclusion, and apps/bookings/
filters.py's BookingFilter allows an explicit ?status=PENDING_PAYMENT
query. This contradicts the system's own design intent, stated in
comments in apps/bookings/models.py and apps/bookings/services.py, that
a PENDING_PAYMENT booking is "not yet visible to the teacher."

CURRENT BEHAVIOR
GET /api/bookings/teacher/ (no filter, or ?status=PENDING_PAYMENT
explicitly) returns PENDING_PAYMENT bookings including the student's
message, requested time, and locked price. GET /api/bookings/<id>/ as
the teacher also returns a PENDING_PAYMENT booking's full detail if the
teacher knows/guesses the id.

EXPECTED BEHAVIOR
- list_teacher_bookings must unconditionally exclude PENDING_PAYMENT
  (and DRAFT, if that status is reachable/exists in BookingRequest.
  STATUS - check the model) from its base queryset, BEFORE any caller-
  supplied filter (including an explicit status= query param) is
  applied. A teacher must not be able to retrieve these rows through any
  combination of query parameters.
- get_booking_detail must raise NotFoundError (not PermissionDenied) if
  the requesting user is the booking's teacher AND the booking's status
  is PENDING_PAYMENT/DRAFT - use NotFoundError specifically, to avoid
  confirming to the teacher that a booking exists in a state they
  shouldn't know about (an information-disclosure best practice, not
  just a permissions check).
- The equivalent call from the STUDENT'S side (get_booking_detail when
  the requesting user is the booking's student) must be completely
  unaffected - a student must still see their own PENDING_PAYMENT
  booking with full detail, since the existing frontend polling UI
  (BookingDetailPage.tsx) depends on this.
- list_student_bookings (the student-facing equivalent, if separately
  implemented - check apps/bookings/selectors.py for its actual name)
  must also be completely unaffected.

FILES TO INSPECT FIRST
- apps/bookings/selectors.py in full (list_teacher_bookings,
  get_booking_detail, and whatever the student-facing list function is
  called)
- apps/bookings/filters.py (BookingFilter)
- apps/bookings/views.py (TeacherBookingListView, BookingDetailView -
  confirm exactly how these selectors are called and whether
  BookingDetailView is shared between student and teacher callers or
  is two separate view classes)
- apps/bookings/models.py (BookingRequest.STATUS - confirm the full list
  of statuses, specifically whether DRAFT exists as an actual reachable
  status or was only a documentation reference)
- apps/bookings/serializers.py (BookingRequestSerializer - confirm no
  field-level information would leak even if a row did slip through,
  as defense in depth, though the primary fix is at the selector level)

IMPLEMENTATION REQUIRED
- Modify list_teacher_bookings to exclude PENDING_PAYMENT/DRAFT
  unconditionally at the base-queryset level (i.e. .exclude(status__in=
  [...]) applied before the filterset is applied, not as something the
  filterset could override).
- Modify get_booking_detail to check status when the caller is the
  teacher, as described above.
- Consider (and implement if it fits cleanly without a large refactor)
  restricting BookingFilter's status field choices for the teacher-
  facing endpoint specifically, so the invalid choice is rejected at the
  serializer/filter level too, not just filtered out silently - use your
  judgment on whether this needs a second FilterSet class or can be done
  by parameterizing the existing one; do not over-engineer this if the
  selector-level fix alone is sufficient and well-tested.

CONSTRAINTS
- Do not change list_student_bookings or the student-facing branch of
  get_booking_detail in any way - any test failure on the student side
  is a regression, not intended behavior.
- Do not change BookingRequest.STATUS choices or add/remove any status
  value in this phase.
- Do not change accept_booking/reject_booking or
  _TEACHER_ACTIONABLE_STATUSES - those are already correct per the
  audit and out of scope here.

TESTS TO ADD (in tests/bookings/)
- GET teacher booking list, no filter, with a PENDING_PAYMENT booking
  belonging to that teacher in the DB -> booking absent from results.
- GET teacher booking list, ?status=PENDING_PAYMENT explicitly -> still
  absent (proves the exclusion isn't bypassable via the filter param).
- GET booking detail for a PENDING_PAYMENT booking, as its teacher ->
  404 NotFoundError, not 403.
- GET booking detail for a PENDING_PAYMENT booking, as its STUDENT ->
  200, full detail returned (regression guard).
- GET teacher booking list with a mix of statuses including
  PENDING_PAYMENT, AWAITING_TEACHER_REVIEW, ACCEPTED -> only the non-
  PENDING_PAYMENT ones returned, others unaffected.
- Full regression of tests/bookings/ suite.

ACCEPTANCE CRITERIA
- All new tests pass.
- Full existing bookings test suite passes with zero changes needed to
  student-side test assertions.
- Manual verification: the frontend's BookingDetailPage polling flow
  (student side) still works end-to-end against the modified backend -
  if you have a way to run the frontend against this backend, confirm a
  student can still see their own PENDING_PAYMENT booking status update
  in real time; if not, state clearly in your output that this manual
  check should be done by a human before merging.
```

---

### 🤖 Coding Prompt — Phase 5

```
CONTEXT
An audit found three separate low/medium-severity issues to fix together
in this phase:
1. apps/payments/admin.py's PaymentAdmin and apps/bookings/admin.py's
   BookingRequestAdmin both allow direct editing of the `status` field
   via Django admin, bypassing the service layer entirely (no
   PaymentEvent, no PayoutLedgerEntry, no notification, no state-machine
   validation) - inconsistent with this codebase's own discipline
   elsewhere (e.g. PaymentEventAdmin already correctly disables
   add/change permissions with an explanatory comment - use that as your
   reference pattern).
2. afra-frontend/src/features/bookings/components/BookingRequestForm.tsx
   contains self-contradictory copy: "You'll be charged X once the
   teacher accepts. Payment is captured now and held until then" - the
   first sentence contradicts the second; actual behavior is capture-at-
   submission-time.
3. afra-frontend/src/features/payments/components/PaymentCard.tsx has
   `const providerLabel = payment.provider === "INTERNAL" ? "Free" :
   "Stripe"` - this mislabels every non-zero INTERNAL-provider payment
   (which, before Phase 1/2, was every payment in the system) as "Free,"
   when INTERNAL describes the payment processing method, not the
   amount.

CURRENT BEHAVIOR
See above - three independent surface-level issues in the admin and
frontend layers.

EXPECTED BEHAVIOR
1. Add "status" to readonly_fields on both PaymentAdmin and
   BookingRequestAdmin. Do NOT build a custom admin action to replace
   this in this phase unless you find explicit existing evidence (code
   comments, docs, or an existing admin action elsewhere in this
   codebase for a similar purpose) that staff currently rely on hand-
   editing status for a legitimate operational workflow - if you find
   no such evidence, simply lock the field and note in your output that
   a follow-up admin action (calling the real service functions) should
   be added later if support/product confirms it's needed. Do not guess
   at what that workflow should look like without that confirmation.
2. Rewrite the BookingRequestForm.tsx payment-timing copy as a single,
   internally-consistent statement that accurately reflects capture-at-
   submission-time, escrow-hold-until-teacher-response behavior. Keep
   the surrounding component structure and styling unchanged - this is a
   copy-only change.
3. Change PaymentCard.tsx's providerLabel logic to be based on
   `payment.amount === 0` for the "Free" label, independent of
   `provider`. For a non-zero-amount INTERNAL-provider payment, use a
   distinct, accurate label - not "Free," not "Stripe." Pick clear,
   professional-sounding label text (e.g. "Processed internally" or
   similar) and note in your output that this specific wording is a
   product/copy decision that should be confirmed with whoever owns
   product copy in this codebase, not treated as final.

FILES TO INSPECT FIRST
- apps/payments/admin.py (PaymentEventAdmin and PayoutLedgerEntryAdmin,
  as the reference pattern for read-only/locked-down admin classes in
  this codebase)
- apps/bookings/admin.py
- afra-frontend/src/features/bookings/components/BookingRequestForm.tsx
  (full component, to understand where this copy lives and what data -
  e.g. previewPrice, previewCurrency - is available to reference in the
  rewritten copy)
- afra-frontend/src/features/payments/components/PaymentCard.tsx (full
  component)
- afra-frontend/src/features/payments/components/__tests__/
  PaymentCard.test.tsx (existing test file - review its current
  assertions before changing the label logic, since some may currently
  assert the old "Free" behavior and will need updating, not just
  additive new tests)

IMPLEMENTATION REQUIRED
- apps/payments/admin.py: add "status" to PaymentAdmin.readonly_fields.
- apps/bookings/admin.py: add "status" to
  BookingRequestAdmin.readonly_fields.
- BookingRequestForm.tsx: rewrite the identified paragraph only.
- PaymentCard.tsx: change providerLabel computation as described.

CONSTRAINTS
- Do not remove "status" from list_display or list_filter on either
  admin class - staff should still be able to VIEW and FILTER by status,
  only not directly edit it via the change form.
- Do not change any other admin.py field configuration in this phase.
- Do not change PaymentCard's or BookingRequestForm's props, data
  fetching, or any logic unrelated to the specific copy/label described
  above.

TESTS TO ADD
- Backend: an admin-form-level test (or a test against
  PaymentAdmin.get_readonly_fields / equivalent) confirming "status" is
  present in readonly_fields for both PaymentAdmin and
  BookingRequestAdmin.
- Frontend: update/add a test in PaymentCard's existing test file
  asserting: amount=0 (any provider) -> "Free" label shown; amount=50,
  provider=INTERNAL -> the new non-"Free" label shown, not "Free" and
  not "Stripe"; amount=50, provider=STRIPE -> "Stripe" label shown
  (unchanged existing behavior).
- Frontend: a rendering test (or manual visual check, noted in your
  output if automated testing isn't practical for copy content
  specifically) confirming BookingRequestForm no longer renders the
  contradictory two-sentence copy - assert the new text is present and
  the old contradictory phrasing is absent.

ACCEPTANCE CRITERIA
- All new/updated tests pass.
- Full existing admin, bookings, and payments (frontend) test suites
  pass.
- Manually confirm (or note for human confirmation) that the Django
  admin change form for both Payment and BookingRequest renders `status`
  as non-editable text rather than a dropdown/input.
```

---

### 🤖 Coding Prompt — Phase 6

```
CONTEXT
Two remaining items from the audit: (1) apps/payments/services.py's
process_payment / _process_payment_via_stripe / _process_payment_
simulated functions, and the corresponding Celery task in apps/
payments/tasks.py, are confirmed dead code (grep shows no .delay()/
.apply_async() call site anywhere outside their own definitions and
tests) - yet this is the one place in the pre-Phase-1 codebase that
correctly gated Stripe calls behind settings.PAYMENTS_USE_STRIPE, which
made it a misleading reference for anyone reading the file after Phase 1
already fixed the real path. (2) The audit's scenario analysis found two
operational gaps: no automated cleanup for a BookingRequest stuck at
PENDING_PAYMENT if its capture_payment_for_booking Celery task is lost
(broker failure, task eviction, etc. - not a normal failure path, an
infrastructure one), and Phase 0's reconciliation command (this phase
assumes Phase 0 already shipped) is currently a manually-run tool, not a
scheduled/alerting one.

CURRENT BEHAVIOR
- process_payment and its helpers exist and are fully tested but never
  invoked by any live code path.
- A BookingRequest that reaches PENDING_PAYMENT but never gets a
  capture_payment_for_booking task actually executed (e.g. Celery broker
  down at the exact moment of transaction.on_commit) stays
  PENDING_PAYMENT forever with no automated recovery.
- reconcile_payments (Phase 0) must be run manually.

EXPECTED BEHAVIOR
1. Remove apps/payments/services.py's process_payment,
   _process_payment_via_stripe, and _process_payment_simulated
   functions, and the corresponding Celery task definition in apps/
   payments/tasks.py, along with their dedicated tests - ONLY after
   confirming via grep/search across the entire repo (backend AND
   frontend) that nothing references them. If you find any reference
   you weren't expecting, STOP and report it rather than deleting
   anything - do not delete code you're not fully certain is unused.
2. Add a new scheduled Celery task,
   reap_stale_pending_payment_bookings, in apps/bookings/tasks.py (or
   apps/payments/tasks.py if that's a better fit given where similar
   sweep tasks currently live - check both files' existing conventions
   for where this kind of cross-app sweep belongs before choosing), that:
   - Finds BookingRequest rows with status == "PENDING_PAYMENT" older
     than a new setting PAYMENTS_STUCK_CAPTURE_THRESHOLD_MINUTES
     (default something reasonable like 15 minutes - this should be
     comfortably longer than normal capture latency, not a tight
     timeout).
   - CRITICALLY: only reaps bookings that have NO associated Payment row
     at all (a booking whose capture task genuinely ran and created a
     Payment, even if that Payment is still status=PENDING for some
     legitimate in-progress reason, must NOT be touched by this sweep -
     that's a different, already-idempotent, already-retryable code
     path). Inspect the Payment model's relationship to BookingRequest
     to write this query correctly (e.g. Payment.objects.filter(booking=
     booking).exists()).
   - For matching stale bookings: transition to CANCELLED (following
     the existing pattern used elsewhere for booking cancellation - do
     not invent a new status), and emit a clearly-taggable log/metric
     (e.g. logger.error with a distinct message prefix) since this
     condition indicates infrastructure failure and should be
     alertable, not silently swallowed at info level.
   - Add this task to the existing Celery Beat schedule in
     config/celery.py, following the existing schedule entries'
     conventions (interval, task name string format).
3. Promote apps/payments/management/commands/reconcile_payments.py
   (from Phase 0) into a Celery Beat scheduled task as well (wrap the
   existing command's logic in a shared function callable from both the
   management command and a new Celery task, rather than duplicating
   logic) that runs on a reasonable schedule (e.g. daily) and emits an
   alertable log/metric on any violation found - do not just print/log
   at info level for a violation, since the entire point of this task is
   that a human should be notified.

FILES TO INSPECT FIRST
- apps/payments/services.py (process_payment and helpers, full context)
- apps/payments/tasks.py (the corresponding Celery task)
- apps/bookings/tasks.py (if it exists - check apps/bookings/ structure;
  if it doesn't exist yet, apps/sessions/tasks.py is a good structural
  reference for how sweep-style Beat tasks are conventionally written in
  this codebase - follow its logging/retry-policy conventions, e.g. its
  documented distinction between "normal outcome, don't retry" vs.
  "unexpected exception, do retry")
- config/celery.py (existing beat_schedule structure)
- apps/payments/models.py (Payment - booking foreign key field name)
- apps/payments/management/commands/reconcile_payments.py (from Phase 0
  - to refactor its core logic into a shared, task-callable function)
- Full-repo grep for "process_payment" and
  "_process_payment_via_stripe" and "_process_payment_simulated" before
  touching anything, to build your own confirmation list independent of
  this prompt's claim that it's unused.

IMPLEMENTATION REQUIRED
- Deletion of the three functions + task + their dedicated tests, gated
  on your own verification step above.
- New reap_stale_pending_payment_bookings Celery task + Beat schedule
  entry + new settings.PAYMENTS_STUCK_CAPTURE_THRESHOLD_MINUTES value in
  config/settings/base.py.
- Refactor of reconcile_payments into a shared function + new scheduled
  Celery task wrapping it + Beat schedule entry.

CONSTRAINTS
- Do not delete anything you couldn't independently verify is unused via
  your own grep - if in doubt, leave it and report the ambiguity instead.
- The new reaper task must never touch a PENDING_PAYMENT booking that
  already has a Payment row, under any circumstance - this is the single
  most important correctness constraint in this phase, since getting it
  wrong would cancel bookings whose capture is legitimately still in
  progress (e.g. a slow-but-healthy Stripe API call in Phase 1/2's
  STRIPE branch).
- Do not change the reconciliation logic's actual checks from Phase 0 -
  only change how/when it runs (manual command -> also a scheduled
  task), not what it checks.

TESTS TO ADD
- Dead-code removal: a CI-style test or script assertion that grep for
  the removed function/task names across the full repo returns zero
  results after your change (belt-and-suspenders against a future
  accidental reintroduction of a stale reference).
- reap_stale_pending_payment_bookings: a PENDING_PAYMENT booking older
  than the threshold with NO Payment row -> gets cancelled, error-level
  log emitted.
- reap_stale_pending_payment_bookings: a PENDING_PAYMENT booking younger
  than the threshold with NO Payment row -> left untouched.
- reap_stale_pending_payment_bookings: a PENDING_PAYMENT booking older
  than the threshold that DOES have a Payment row (regardless of that
  Payment's own status) -> left completely untouched - this is the
  critical negative test for the constraint above.
- Scheduled reconciliation task: seeded invariant violation (reuse Phase
  0's test fixtures) -> task run results in an alertable log/metric,
  verified via assertion on the log output or mocked metric call.

ACCEPTANCE CRITERIA
- All new tests pass.
- Full existing test suite (backend-wide) passes with no unexpected
  failures from the deletion.
- Beat schedule (config/celery.py) contains both new scheduled tasks
  with sensible, documented intervals.
- Your output explicitly lists every file you deleted and every grep
  result you used to confirm safety of that deletion, for human review
  before merge.
```

---

# PART 5 — AI Verification Prompts (one per phase)

---

### ✅ Verification Prompt — Phase 0
```
Review the implementation of the Phase 0 reconciliation tooling
(apps/payments/management/commands/reconcile_payments.py and its
tests).

Confirm:
1. The command performs ZERO writes to the database - grep the file for
   .save(, .update(, .delete(, .create(, bulk_create, bulk_update. Any
   match is a failure of this phase's core constraint; report it.
2. The provider/gateway_charge_id consistency check correctly flags both
   directions of inconsistency (STRIPE without a charge id, AND INTERNAL
   with one) - re-derive this from the model fields yourself rather than
   trusting the implementation's own test descriptions, and confirm the
   test fixtures actually exercise both directions.
3. Run the full existing test suite (backend-wide) and confirm zero
   regressions - this phase should have touched no existing file, so any
   test failure outside the new test file is unexpected and must be
   explained.
4. Run `python manage.py reconcile_payments` against the current test
   database / factory-seeded data and paste the actual output.
5. Confirm the command's exit code behavior matches spec: 0 when clean,
   non-zero on a real inconsistency - actually trigger both cases and
   show the exit codes, don't just read the code and assume.

Report PASS/FAIL against each of the 5 points above individually, not
just an overall verdict.
```

---

### ✅ Verification Prompt — Phase 1
```
Review the Phase 1 changes to apps/payments/services.py
(release_payment_to_teacher and resolve_dispute's provider-dispatch
logic) against the audit's Critical Finding #1 (partial - the Stripe-
gating half specifically, not the onboarding-wiring half, which is
Phase 2).

Confirm:
1. release_payment_to_teacher's INTERNAL branch never calls
   stripe.Transfer.create under any input - write and run a test (if not
   already covered) that mocks stripe.Transfer.create and asserts
   assert_not_called() for an INTERNAL-provider payment, across every
   release_payment_to_teacher call site you can find, not just the
   happy path.
2. release_payment_to_teacher's STRIPE branch behavior is BYTE-FOR-BYTE
   equivalent to the pre-Phase-1 behavior for a valid STRIPE payment
   (same PayoutLedgerEntry fields, same notification calls, same
   PaymentEvent records) - diff the pre- and post-change side effects
   for a STRIPE-provider payment specifically; this is the highest-risk
   regression surface since it's real-money logic.
3. The new guard (STRIPE provider + missing gateway_charge_id -> raise,
   don't call Stripe) actually fires before any network-adjacent code -
   confirm by checking whether an exception is raised even when
   stripe.Transfer.create is NOT mocked in the test (i.e. it would have
   made a real network call if reached) - this proves the guard is
   genuinely first, not just first in most cases.
4. resolve_dispute's RESOLVED_TEACHER and RESOLVED_SPLIT branches have
   the identical INTERNAL/STRIPE dispatch applied - confirm this wasn't
   only done for release_payment_to_teacher and forgotten for the
   dispute-resolution code path, which was explicitly called out as
   needing the same fix.
5. Run `python manage.py reconcile_payments` (Phase 0) after running the
   full new + existing test suite against a database left in its post-
   test state (or a targeted seed matching the phase's test scenarios) -
   confirm zero violations.
6. Run the FULL existing tests/payments/ suite (test_services.py AND
   test_disputes.py) and confirm every test passes - list any test that
   needed a fixture change (e.g. explicit provider= now required) and
   confirm in each case the change was to the fixture setup, not to the
   test's actual business-outcome assertion (a business-assertion change
   here would suggest the fix altered behavior it shouldn't have).
7. Re-run the audit's original Scenario table (from the prior audit
   document) for scenarios 6 and 9 specifically (payment succeeds but
   booking stuck / webhook duplicated) and confirm they're still handled
   correctly - this phase shouldn't have touched capture logic, but
   verify that assumption rather than skipping the check.

Report PASS/FAIL against each of the 7 points above individually.
```

---

### ✅ Verification Prompt — Phase 2
```
Review the Phase 2 Stripe Connect onboarding webhook implementation
(apps/payments/webhooks.py's new account.updated handler and apps/
teachers/services.py's mark_stripe_onboarding_incomplete).

Confirm:
1. Signature verification for the new event type uses the exact same
   verified path as the three pre-existing event types - no new,
   separate, or weaker verification logic was introduced. If a second
   webhook secret setting was added, confirm it defaults safely (does
   NOT silently disable verification if unset) and that existing
   deployments' webhook configuration isn't broken by this change -
   actually check config/settings/base.py's default value logic, don't
   just read the code's intent.
2. An account.updated event for an unknown stripe_account_id returns 200
   (not 500) and logs a warning - actually send/simulate this exact
   payload in a test and confirm the response, don't infer it from code
   reading alone.
3. is_payout_ready correctly starts False, becomes True only after a
   fully-enabled account.updated event, and correctly reverts to
   requiring PENDING status again after a disabled account.updated event
   - this three-state transition (never onboarded -> pending -> complete
   -> back to pending on disablement) must be tested explicitly, not
   just the happy path to COMPLETE.
4. End-to-end integration: seed a teacher with STRIPE-provider
   HELD_IN_ESCROW payment, run start_stripe_onboarding, simulate the
   account.updated webhook reaching COMPLETE, then actually invoke
   release_held_payments (the real Beat task, or its underlying
   function) and confirm the payment reaches RELEASED - this is the
   actual proof that Critical Finding #1 is resolved end-to-end, not
   just that individual functions behave correctly in isolation.
5. Confirm Phase 1's INTERNAL-provider release path is completely
   unaffected by this phase - run an INTERNAL-provider release with
   is_payout_ready explicitly False and confirm it still succeeds (this
   is the regression check proving the two provider paths stayed
   decoupled as designed).
6. Full existing webhook and teachers test suites pass with zero
   regressions.
7. Cross-check against the original audit: is `mark_stripe_onboarding_
   complete`'s docstring now accurate (describes the real caller), or
   does stale documentation still exist anywhere referencing the old,
   nonexistent "onboarding-return endpoint"? Grep for it.

Report PASS/FAIL against each of the 7 points above individually.
```

---

### ✅ Verification Prompt — Phase 3
```
Review the Phase 3 double-booking race condition fix (TeacherProfile
row-lock in create_booking, and the new exclusion-constraint migrations
in apps/bookings/migrations/0006 and 0007).

Confirm:
1. The new concurrent/threaded test actually exercises real concurrency
   - read the test implementation and confirm it uses genuinely separate
   database connections/transactions running in parallel (e.g. separate
   threads each opening their own connection), not two sequential calls
   within the same test transaction that would never actually race. If
   the test doesn't genuinely race, it cannot prove the fix - flag this
   explicitly if found.
2. Run that concurrent test repeatedly (at least 20 iterations in a
   loop) and confirm it passes consistently, not intermittently - a
   race-condition test that only sometimes catches the race is not
   sufficient evidence the fix works; report the pass rate you observed.
3. Confirm the exclusion constraint's WHERE clause status list exactly
   matches the current value of _ACTIVE_BOOKING_STATUSES in apps/
   bookings/services.py at the time of this review (not what it was
   assumed to be during implementation) - if these have drifted, that's
   a bug in this phase's implementation.
4. Confirm the migration was tested against a scenario with pre-existing
   overlapping data (simulate the pre-fix race having already produced
   two overlapping active bookings in a test database) and that the
   implementation's output clearly reports this as a blocking issue
   requiring manual resolution, rather than the migration silently
   failing or silently resolving the conflict without human decision-
   making, per the coding prompt's explicit constraint against
   unilateral resolution.
5. Confirm the IntegrityError-to-ConflictError translation actually
   returns the same HTTP status code and response shape as the existing
   application-level conflict check (test both code paths side by side
   and diff the actual HTTP responses, not just the exception types).
6. Regression: confirm non-overlapping bookings for the same teacher,
   and overlapping bookings where one is REJECTED/CANCELLED, both still
   succeed - run these specific existing/new regression tests and
   confirm pass.
7. Re-run the original audit's Scenario 8 description manually (two
   students attempting to book the same teacher/slot simultaneously) as
   an integration-level check, not just a unit test of the internal
   locking mechanism, to confirm the fix holds at the API-request level
   too.

Report PASS/FAIL against each of the 7 points above individually.
```

---

### ✅ Verification Prompt — Phase 4
```
Review the Phase 4 teacher-visibility fix (apps/bookings/selectors.py's
list_teacher_bookings and get_booking_detail changes).

Confirm:
1. A PENDING_PAYMENT booking is unreachable via GET /api/bookings/
   teacher/ under every combination of query parameters you can think of
   - test with no filter, ?status=PENDING_PAYMENT, and any other status
   filter combined with it if the filter supports multiple values -
   don't just test the one case the coding prompt explicitly listed.
2. GET /api/bookings/<id>/ for a PENDING_PAYMENT booking as its teacher
   returns 404, and confirm the response body doesn't leak any
   information distinguishing "doesn't exist" from "exists but hidden"
   (compare the exact response body/headers against a genuinely
   nonexistent booking id's 404 response - they should be
   indistinguishable).
3. The exact same booking, requested by its STUDENT, still returns 200
   with full data, completely unchanged from pre-Phase-4 behavior - this
   is the most important regression check in this phase, since breaking
   it would break the core booking-creation polling UX the whole system
   depends on.
4. Check whether AWAITING_TEACHER_REVIEW, ACCEPTED, REJECTED, CANCELLED
   bookings are all still correctly visible to the teacher exactly as
   before - Phase 4 should have a surgical, narrow blast radius (only
   PENDING_PAYMENT/DRAFT affected), confirm this by running the FULL
   existing bookings test suite, not just the new tests.
5. If BookingFilter's choices were narrowed as an additional defense-in-
   depth measure, confirm it doesn't break any legitimate existing
   status-filter usage from the frontend (cross-check against
   BookingsPage.tsx's actual tab definitions from the original audit -
   allBookingsStatusTabs and the pending sub-tabs - confirm every status
   value the frontend actually sends is still a valid filter choice).
6. Full regression run of tests/bookings/.

Report PASS/FAIL against each of the 6 points above individually.
```

---

### ✅ Verification Prompt — Phase 5
```
Review the Phase 5 admin-hardening and frontend-copy changes.

Confirm:
1. Attempt to submit a Django admin change form for an existing Payment
   with a modified status value (simulate this via Django's admin test
   client, posting a form payload with a different status than the
   object currently has) - confirm the status field value is unchanged
   after submission (proving it's genuinely read-only, not just visually
   styled differently while still processing POSTed changes).
2. Same test for BookingRequest.
3. Confirm list_display and list_filter still include "status" on both
   admin classes (this phase should not have removed staff's ability to
   VIEW/FILTER by status, only edit it).
4. Read the new BookingRequestForm.tsx copy and confirm it contains no
   internal contradiction - specifically check it doesn't claim two
   different things about WHEN the student is charged anywhere in the
   component (search the full component file, not just the one
   paragraph that was flagged, in case similar copy exists elsewhere in
   the same file).
5. Confirm PaymentCard's new label logic: test amount=0 with
   provider=INTERNAL, amount=0 with provider=STRIPE, amount=50 with
   provider=INTERNAL, and amount=50 with provider=STRIPE as four
   distinct cases - confirm "Free" appears only for the two amount=0
   cases and the two amount=50 cases show provider-appropriate distinct
   labels (not "Free" in either case).
6. Full existing admin, bookings-admin, and payments-frontend test
   suites pass.

Report PASS/FAIL against each of the 6 points above individually, and
explicitly flag that the specific wording chosen for the new PaymentCard
INTERNAL-provider label and the new BookingRequestForm copy are product/
copy decisions this verification pass cannot rubber-stamp as
"correct" - only as "internally consistent and accurate," which is a
different, narrower claim.
```

---

### ✅ Verification Prompt — Phase 6
```
Review the Phase 6 legacy-code deletion and operational-safeguard
additions (apps/payments/services.py's process_payment removal, the new
reap_stale_pending_payment_bookings task, and the scheduled
reconciliation task).

Confirm:
1. Independently re-run the same grep/search the coding prompt required
   before deletion - across the ENTIRE repo, backend and frontend,
   including any config/deployment files that might reference a Celery
   task by string name (e.g. config/celery.py's beat_schedule, or any
   deployment YAML/env config listing task names) - confirm zero
   remaining references to process_payment,
   _process_payment_via_stripe, or _process_payment_simulated. Do not
   trust the implementation's own claimed grep output - redo it
   yourself.
2. Full backend test suite passes after the deletion, with zero
   unexpected failures (any failure here means something depended on the
   deleted code that wasn't caught).
3. For reap_stale_pending_payment_bookings: this is the single highest-
   risk new addition in this phase, since a bug here cancels real
   customer bookings. Specifically stress this negative case: create a
   PENDING_PAYMENT booking, create a Payment row for it with
   status=PENDING (simulating a slow-but-healthy in-flight Stripe
   capture from Phase 1/2's STRIPE branch), set its created_at far in
   the past (older than the reap threshold), and run the reaper task -
   confirm the booking is NOT touched. This exact scenario is the one
   most likely to be gotten wrong and must be verified directly, not
   inferred from reading the query.
4. Confirm the reaper task's threshold setting
   (PAYMENTS_STUCK_CAPTURE_THRESHOLD_MINUTES) has a sensible default
   that's comfortably longer than the actual observed capture latency in
   this codebase (cross-check against capture_payment_for_booking's
   retry policy/timeouts, if any are configured, to confirm the reaper
   threshold can't fire while a legitimate retry is still in its normal
   window).
5. Confirm the scheduled reconciliation task reuses the exact same
   checking logic as Phase 0's original management command (diff them,
   or confirm both call a shared function) - not a re-implementation
   that could drift from the original and check something subtly
   different.
6. Confirm the Beat schedule entries in config/celery.py are
   syntactically correct and would actually be picked up (if there's a
   way to validate Celery Beat schedule config without a live broker,
   use it; otherwise note this as a deployment-time check still
   required).
7. Confirm the implementation's own output (per the coding prompt's
   requirement) lists every deleted file/function and every grep result
   used - review that list yourself against point 1's independent
   re-verification and flag any discrepancy.

Report PASS/FAIL against each of the 7 points above individually.
```

---

# PART 6 — Risk Analysis

## 6.1 Changes that could break existing users

| Change | Who's affected | Risk | Mitigation |
|---|---|---|---|
| Phase 1: release logic dispatch | Any teacher with a payment currently sitting `HELD_IN_ESCROW` at deploy time | A payment captured under the *old* code (provider set, but the old unconditional-Stripe-call behavior) must still resolve correctly under the *new* dispatch logic — the new code reads the same `provider` field the old row already has, so this should be transparent, but must be explicitly tested against **pre-existing rows**, not just newly-created test fixtures | Before deploying Phase 1, run `reconcile_payments` (Phase 0) against production data to get a full inventory of existing `provider` values and any existing inconsistencies, so you know exactly what population Phase 1's dispatch will act on at cutover |
| Phase 2: `account.updated` webhook | Teachers who completed real Stripe onboarding *before* this fix shipped, if any exist | If any teacher's `stripe_onboarding_status` was manually forced to `"COMPLETE"` outside the normal flow (e.g., via Django admin, given Phase 5 hasn't locked that down yet at this point in the roadmap) before Phase 2 ships, the new webhook could receive a *stale* `account.updated` event and incorrectly move them back to `"PENDING"` if their account object doesn't currently report fully-enabled | Audit existing `TeacherProfile` rows for `stripe_onboarding_status == "COMPLETE"` before Phase 2 ships; if any exist and weren't set through a real completed Connect flow, resolve manually first |
| Phase 3: exclusion constraint | Any student/teacher with existing overlapping active bookings from before the fix | The migration itself may **fail to apply** — see Database migration risks below | See 6.2 |
| Phase 4: teacher visibility restriction | Teachers who were (knowingly or not) relying on seeing `PENDING_PAYMENT` bookings for any workflow | Low likelihood given this contradicts the system's own stated design, but confirm with product/support before shipping — if any support tooling or manual workflow depends on this visibility, it needs a replacement before Phase 4 ships, not after |
| Phase 5: admin lockdown | Staff/support users who currently hand-edit `status` for manual issue resolution | If this is an active support workflow (even undocumented), locking it down without a replacement breaks their ability to resolve stuck bookings/payments | **Must confirm with whoever operates the admin panel before shipping** — this is called out explicitly in the Phase 5 coding prompt as a gate, not an assumption |
| Phase 6: dead-code deletion | None directly (confirmed unused), but see below | Small risk of an unnoticed reference (e.g., a deployment script, a cron entry, an external monitoring config referencing the Celery task name string) that pure code-grep wouldn't catch | Also grep deployment configs/infra-as-code, not just application code, before deleting |

## 6.2 Database migration risks

- **Phase 3's exclusion constraint is the single highest-risk migration in this roadmap.** If the pre-fix race condition has already fired in production — i.e., two overlapping active `BookingRequest` rows already exist for some teacher — the `ADD CONSTRAINT` migration will fail outright (Postgres refuses to add a constraint that existing data violates). This is not a hypothetical: the entire reason Phase 3 exists is that the audit proved this race is reachable, so assume it's plausible that it already happened at least once in any environment with real concurrent traffic.
  - **Mitigation:** Run the pre-migration data check (specified explicitly in the Phase 3 coding prompt) as a **separate, earlier step** — a read-only query run well before the migration is scheduled, with results reviewed by a human. If violations exist, they need a business decision (which student keeps the slot, how the other is compensated/refunded) **before** the schema change can proceed — this cannot be automated away.
  - **Secondary mitigation:** Deploy Phase 3's Layer 1 (application-level row lock) first, let it run for a period with no further races possible, *then* deploy Layer 2 (the constraint) once you're confident no *new* violations will be created — this doesn't fix pre-existing violations but stops the count from growing while the data cleanup is planned.
- **`btree_gist` extension requirement.** Confirm the production Postgres instance/managed-database provider (check `config/settings/base.py`'s DB config, referenced as coming from a `render.yaml`-style managed config in the codebase's own comment) actually permits extension installation — some managed Postgres tiers restrict `CREATE EXTENSION` to superuser/admin-assisted operations. This must be confirmed with infrastructure ownership before Phase 3 is scheduled, not discovered at deploy time.

## 6.3 Payment data risks

- **Phase 1 changes the code path for real fund movement.** Even with full test coverage, recommend a staged rollout: deploy Phase 1 behind a check that logs-but-doesn't-yet-branch (i.e., compute what the new dispatch *would* do, log it, but still execute the old unconditional path) for a short period first if the team's deployment tooling supports this pattern, before fully cutting over — this gives a real-data comparison window without financial risk. If that staged approach isn't practical given the codebase's current tooling, at minimum: deploy to a staging environment with production-data-shaped fixtures and run Phase 0's reconciliation tool before and after.
- **No payment data is deleted or mutated by any phase in this roadmap** — every phase either adds new capability (Phase 2's webhook, Phase 3's constraint, Phase 6's reaper) or corrects *future* behavior (Phase 1's dispatch) without rewriting historical `Payment`/`PaymentEvent`/`PayoutLedgerEntry` rows. This is a deliberate constraint carried through every phase's implementation instructions and should remain non-negotiable — if any phase's implementation is found to touch historical financial records, that's a signal to stop and re-plan, not to proceed.

## 6.4 Backward compatibility issues

- **API response shapes are unchanged across all six phases** — no phase modifies a serializer's field set or a status enum's values, so no frontend contract changes are required as a hard dependency (Phase 5's frontend changes are copy/label-only, not contract changes; Phase 4's changes affect *which rows* are returned, not their shape).
- **Frontend/backend deploy ordering:** because no contract changes exist, backend and frontend for each phase can deploy independently/in either order without a compatibility window — this is a genuine advantage of scoping the phases this way, and should be preserved as a constraint on any future phase added to this roadmap.

## 6.5 Deployment risks

- **Celery Beat schedule changes (Phases 2's webhook doesn't need one, but Phase 6's two new scheduled tasks do)** require a Beat process restart/reload to pick up new schedule entries — confirm the deployment process actually restarts Beat on this kind of config-only change, not just the web/worker processes, or the new tasks will silently never run despite the code being deployed.
- **Webhook endpoint changes (Phase 2)** require coordinating with whoever manages the Stripe dashboard's webhook configuration — if a new `account.updated` event type (and possibly a new endpoint/secret, per the coding prompt's explicit flag) is being added, this is a **manual, out-of-band step in the Stripe dashboard** that must happen in lockstep with the code deploy, not before (events will 404/fail signature verification if the endpoint isn't configured yet) and not long after (onboarding completions would be silently missed in the gap).
- **Migration ordering (Phase 3)** — the two migrations (`btree_gist` extension, then the constraint) must run in that order and the data-cleanliness check must complete *before* the constraint migration is even attempted in production, which likely means Phase 3 cannot be a fully-automated CI/CD deploy step the way the other phases can — plan for a manual gate in the deployment pipeline specifically for this phase.

## 6.6 Summary mitigation checklist (to run once, before starting Phase 1)

- [ ] Run Phase 0's reconciliation tool against current production data; record the baseline report.
- [ ] Query production for any existing overlapping active `BookingRequest` rows (Phase 3's pre-check), even though Phase 3 is scheduled later — knowing this early avoids a late surprise.
- [ ] Confirm with admin-panel operators whether hand-editing `Payment.status`/`BookingRequest.status` is an active workflow (Phase 5 gate).
- [ ] Confirm with infrastructure ownership whether `CREATE EXTENSION btree_gist` is permitted on the production database (Phase 3 gate).
- [ ] Confirm with whoever owns the Stripe dashboard configuration that they're available to coordinate the `account.updated` webhook setup at Phase 2's deploy time.
- [ ] Confirm the deployment pipeline restarts Celery Beat on schedule-only config changes (Phase 6 gate).
