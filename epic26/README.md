# Epic 26 — Monitoring & Logging — AI Coding Prompts

Repo: `tablogenix-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–25 are fully merged.

**Confirmed directly from the repo:** there is **no `LOGGING` setting anywhere** in any settings file (meaning Django's bare default logging config is active — console output only, no structured format, no file output, no external capture) and **no Sentry SDK, or any error-tracking tool, anywhere in `requirements.txt`**. This epic is genuinely greenfield — but it is also where a *lot* of previously-built, deliberately-broad `except Exception` blocks with `logging.getLogger(...).exception(...)` calls finally get somewhere real to report to. Across this document series, several tasks were explicitly written with this future dependency in mind:
- Epic 6's `ZarinPalGateway`/`ZibalGateway`/`IDPayGateway` implementations catch and log every gateway HTTP failure.
- Epic 16 Task 16.1.1.4's `notify()` service has a deliberately broad `except Exception` specifically because a notification failure must never break the business operation that triggered it — its `logger.getLogger("notifications").exception(...)` call is currently the *only* record of a failure, visible only in raw console output.
- Epic 17 Task 17.2.1.2's bulk product importer catches per-row exceptions to avoid aborting an entire batch import over one bad row.
- Epic 25 Task 25.1.1.5's `backup_database()` Celery task explicitly noted: *"ensure the task's exception is genuinely visible via Celery's own error logging/whatever error-tracking exists, and flag this as something Epic 26 should specifically also wire into real alerting once that epic lands."*

None of these currently go anywhere durable or actionable — this epic is where that changes.

---

## Phase 26.1 — Observability

### Feature 26.1.1 — Error Tracking & Logs

---

#### Task 26.1.1.1 — Integrate Sentry (backend + frontend)

```
You are working in backend/requirements.txt, core/settings/base.py
(and production.py), and frontend/package.json, main.jsx. Assume
Epics 1–25 are fully merged.

CONTEXT — CONFIRMED DIRECTLY FROM THE REPO
No error-tracking tool of any kind exists in this project. Every
unhandled exception across 25 epics of accumulated backend/frontend
code currently either crashes silently (frontend) or produces a raw
500 response with a traceback visible only in server console
output/logs that nobody is actively watching (backend) — and every
deliberately-CAUGHT exception (per this document's header inventory)
is currently invisible beyond whatever's printed to console.

TASK
Integrate Sentry across both the Django backend and the React
frontend.

REQUIREMENTS — backend
- Add `sentry-sdk` to backend/requirements.txt, pinned to a current
  stable version, with the `django` and `celery` extras (Sentry's SDK
  provides framework-specific integrations for both, and this project
  needs BOTH given Epic 22's Celery infrastructure — an error inside a
  Celery task, per this document's header inventory, is just as
  important to capture as one inside a Django view):
  `sentry-sdk[django,celery]==<version>`.
- Configure in backend/core/settings/production.py SPECIFICALLY (NOT
  base.py, and NOT development.py) — Sentry should never be active
  during local development by default, since dev-environment noise
  (deliberate test errors, half-finished feature work, local-only
  misconfigurations) would pollute a real error-tracking project with
  irrelevant signal, actively harming the tool's usefulness for
  genuine production issues:
  ```python
  import sentry_sdk
  from sentry_sdk.integrations.django import DjangoIntegration
  from sentry_sdk.integrations.celery import CeleryIntegration
  from sentry_sdk.integrations.logging import LoggingIntegration

  SENTRY_DSN = config("SENTRY_DSN", default="")
  if SENTRY_DSN:
      sentry_sdk.init(
          dsn=SENTRY_DSN,
          integrations=[
              DjangoIntegration(),
              CeleryIntegration(),
              LoggingIntegration(level=None, event_level="ERROR"),  # capture logger.error()/.exception() calls as Sentry events — see note below
          ],
          traces_sample_rate=config("SENTRY_TRACES_SAMPLE_RATE", default=0.1, cast=float),
          environment=config("ENVIRONMENT", default="production"),
          send_default_pii=False,  # do not automatically attach request user data/PII to error events — see security note below
      )
  ```
  The `LoggingIntegration` with `event_level="ERROR"` is the SPECIFIC,
  important detail that closes the loop on this document's header
  inventory — every `logging.getLogger(...).exception(...)` /
  `.error(...)` call already written across Epics 6/16/17/25 (and
  likely others) will, once this integration is active, AUTOMATICALLY
  become a real, tracked Sentry event with no further code changes
  needed at any of those individual call sites — this single
  integration retroactively makes a large amount of already-written
  error-handling code actually actionable for the first time.
  `send_default_pii=False` is a deliberate SECURITY choice, not an
  oversight — this platform handles real customer names, addresses,
  phone numbers, order/payment data (per Epics 1–9); automatically
  attaching full request/user PII to every error event sent to a
  THIRD-PARTY service (Sentry) by default would be a real data-
  handling/privacy concern worth being deliberate about, not a
  default to accept passively. If specific, NON-PII debugging context
  is valuable on certain events (e.g. an order number, NOT a customer's
  name/address), add it explicitly and narrowly via
  `sentry_sdk.set_context()`/`set_tag()` at specific call sites that
  need it, rather than blanket-enabling automatic PII attachment.
  `SENTRY_DSN` defaulting to an empty string, with the `if SENTRY_DSN:`
  guard, means Sentry is SAFELY INERT if the env var isn't configured
  (e.g. during initial staging setup before a real Sentry project
  exists) rather than crashing the app at startup — a safe, sensible
  default matching the same "safe when unconfigured" pattern already
  established for other optional integrations across this project
  (e.g. Epic 6's `ZARINPAL_SANDBOX` defaulting safe).
  Add `SENTRY_DSN`/`SENTRY_TRACES_SAMPLE_RATE`/`ENVIRONMENT` to
  whatever `.env.example`/staging-env documentation Epic 25 Task
  25.1.1.1 established.

REQUIREMENTS — frontend
- Add `@sentry/react` to frontend/package.json, pinned to a current
  stable version.
- Initialize in frontend/src/main.jsx, guarded the SAME way as the
  backend (only active when a DSN is actually configured, and only
  intended for production/staging builds, not local dev):
  ```javascript
  import * as Sentry from '@sentry/react';

  if (import.meta.env.VITE_SENTRY_DSN) {
    Sentry.init({
      dsn: import.meta.env.VITE_SENTRY_DSN,
      environment: import.meta.env.VITE_ENVIRONMENT || 'production',
      tracesSampleRate: 0.1,
      // Scrub PII from error reports for the same reasons as the
      // backend configuration — see beforeSend below.
      beforeSend(event) {
        if (event.request) {
          delete event.request.cookies;
        }
        return event;
      },
    });
  }
  ```
- Wrap the app's root component in Sentry's `ErrorBoundary` (React's
  own error-boundary mechanism, which Sentry's SDK wraps to also
  report the caught error) so an unexpected RENDER error anywhere in
  the component tree is captured and reported, rather than silently
  producing a blank/broken page with no record anywhere of what
  happened:
  ```jsx
  <Sentry.ErrorBoundary fallback={<ErrorFallbackPage />}>
    <App />
  </Sentry.ErrorBoundary>
  ```
  Build a minimal `ErrorFallbackPage` component (Persian-language,
  matching Epic 14's localization conventions) shown to a real customer
  if this ever triggers — a generic, calm "something went wrong,
  please refresh" message, NOT a raw stack trace or technical error
  detail (which should go to Sentry, not to the customer's screen).

ACCEPTANCE CRITERIA / TESTS
- Add a backend test confirming an intentionally-raised test exception,
  when `SENTRY_DSN` is NOT configured (the test-environment default),
  does not crash/error the test setup itself (Sentry being inert when
  unconfigured shouldn't break anything).
- Manually verify against a REAL Sentry project (create a free/trial
  project if none exists for this purpose): trigger a genuine test
  error in a staging deployment (e.g. a deliberately-broken debug
  endpoint, removed after verification) and confirm it actually
  appears in the real Sentry dashboard, with the `LoggingIntegration`
  ALSO confirmed working by triggering one of the EXISTING
  `logger.exception()` call sites (e.g. Epic 6's gateway failure
  logging, via a mocked gateway failure in staging) and confirming
  THAT also appears as a real Sentry event — the concrete proof this
  task's central claim (existing logging calls become actionable) is
  actually true, not just theoretically wired up correctly.
- Manually verify the frontend `ErrorBoundary`: deliberately trigger a
  render error in a staging build (e.g. a temporary broken component)
  and confirm both the fallback UI renders correctly for the user AND
  the error appears in Sentry.
```

---

#### Task 26.1.1.2 — Structured JSON logging config

```
You are working in backend/core/settings/base.py (and production.py).
Assume Task 26.1.1.1 is already merged.

CONTEXT
Django's bare default `LOGGING` config (confirmed absent/unconfigured
in this project) produces unstructured, plain-text console output with
no consistent format — fine for local development, genuinely difficult
to work with at scale in production, where log AGGREGATION tools
(whatever log-shipping/aggregation this project's actual hosting
provider offers, or a self-hosted ELK/Loki-style stack) work far
better with structured, parseable (typically JSON) log lines than with
free-text output, and where correlating a specific log line back to
the REQUEST that produced it (across possibly-concurrent requests
being handled by the same worker process) is difficult without a
consistent, machine-parseable format carrying request-identifying
context.

TASK
Add a structured JSON logging configuration for production (keeping
development's simpler, human-readable console output as-is, since JSON
logs are objectively worse for a developer actively reading terminal
output during local work).

REQUIREMENTS
- Add `python-json-logger` to backend/requirements.txt, pinned to a
  current stable version.
- Configure `LOGGING` in backend/core/settings/base.py, with the
  format varying by environment (structured JSON for production,
  plain/readable for development — check whether `base.py` can
  correctly conditionalize this given Django's settings-file
  inheritance structure, or whether this belongs in
  production.py/development.py SEPARATELY instead, overriding a
  shared base — follow whichever pattern this project's existing
  settings-file split already uses for environment-conditional config,
  matching precedent rather than introducing a new conditionalization
  style):
  ```python
  # core/settings/production.py
  LOGGING = {
      "version": 1,
      "disable_existing_loggers": False,
      "formatters": {
          "json": {
              "()": "pythonjsonlogger.jsonlogger.JsonFormatter",
              "format": "%(asctime)s %(levelname)s %(name)s %(message)s %(request_id)s",
          },
      },
      "handlers": {
          "console": {
              "class": "logging.StreamHandler",
              "formatter": "json",
          },
      },
      "root": {
          "handlers": ["console"],
          "level": "INFO",
      },
      "loggers": {
          "django": {"handlers": ["console"], "level": "INFO", "propagate": False},
          "django.request": {"handlers": ["console"], "level": "ERROR", "propagate": False},
          # every custom logger already established across this project's
          # prior epics — payments, notifications, shop.sms, etc. — will
          # correctly inherit this root config automatically without
          # needing individual entries here, UNLESS a specific one needs
          # a DIFFERENT level than the default INFO; add specific
          # overrides only where actually warranted.
      },
  }
  ```
  Note the `%(request_id)s` format field — this REQUIRES a
  `request_id` attribute to actually be present on every log record,
  which doesn't happen automatically; this is exactly what
  `RequestIdMiddleware` (the master backlog's own next task,
  25.1.1.3... actually check the master backlog: this project's
  broader backlog document referenced a `RequestIdMiddleware` and a
  `log_service_call` decorator as part of this project's EARLIER,
  pre-this-series observability work, per this document series'
  original grounding context — VERIFY whether these already exist
  somewhere in the codebase from context outside this specific prompt
  series' 26 epics, since the master backlog's own historical summary
  mentions them as already-completed work; if they exist, this task
  should USE them rather than rebuilding; if this specific document
  series' epics never actually built them despite the master backlog
  referencing them as already done, build a minimal
  `RequestIdMiddleware` now as this task's own scope, since the JSON
  log format genuinely needs SOMETHING supplying `request_id` for the
  format string above to work correctly rather than raising a
  KeyError/producing malformed log lines).
  If building `RequestIdMiddleware` fresh:
  ```python
  # core/middleware.py — alongside Epic 23 Task 23.1.1.6's AdminIPAllowlistMiddleware
  import uuid
  import logging

  _local = threading.local()

  class RequestIdMiddleware:
      def __init__(self, get_response):
          self.get_response = get_response

      def __call__(self, request):
          request_id = str(uuid.uuid4())
          _local.request_id = request_id
          request.request_id = request_id
          response = self.get_response(request)
          response["X-Request-ID"] = request_id
          return response


  class RequestIdLogFilter(logging.Filter):
      def filter(self, record):
          record.request_id = getattr(_local, "request_id", "-")
          return True
  ```
  Wire `RequestIdLogFilter` into the `LOGGING` config's `filters` and
  apply it to the `console` handler, and add `RequestIdMiddleware` to
  `MIDDLEWARE` (positioned EARLY, before most other middleware, so the
  request ID is available for the entire remainder of request
  processing). Also set `response["X-Request-ID"]` so the FRONTEND
  (or an external caller debugging an issue) can correlate a specific
  API response back to its exact server-side log entries — genuinely
  useful for real production debugging.
- Confirm this new structured logging doesn't conflict with or
  duplicate Sentry's OWN logging capture (Task 26.1.1.1's
  `LoggingIntegration`) — the two are complementary, not redundant:
  Sentry captures ERROR-level+ events as trackable, alertable issues
  with full context/grouping; structured JSON console logs capture
  EVERYTHING at INFO+ for general-purpose log aggregation/searching —
  both should remain active simultaneously.

ACCEPTANCE CRITERIA / TESTS
- Add a test confirming a log statement produces valid, parseable JSON
  output in the PRODUCTION logging configuration (capture stdout/use
  `override_settings` to activate the production LOGGING config within
  a test, emit a log line, and confirm `json.loads()` on the captured
  output succeeds and contains the expected fields including
  `request_id`).
- Add a test confirming `RequestIdMiddleware` correctly generates a
  request ID, attaches it to the response as `X-Request-ID`, and that
  log lines emitted DURING that request's processing correctly include
  the SAME request ID (proving the thread-local propagation actually
  works end-to-end, not just that the middleware runs).
- Confirm DEVELOPMENT logging remains simple/readable (unchanged from
  before this task, or explicitly configured to stay human-friendly)
  — a regression check that this task didn't accidentally make local
  development console output worse.
```

---

#### Task 26.1.1.3 — Uptime monitoring for API + frontend (external ping service)

```
You are working with an EXTERNAL, third-party uptime-monitoring
service (not code within this repository) and, potentially, Epic 22
Task 22.1.1.4's existing `/api/health/` endpoint. Assume Task 26.1.1.1
is already merged.

CONTEXT
Nothing currently monitors this application's ACTUAL LIVE
AVAILABILITY from an outside-in perspective — Sentry (Task 26.1.1.1)
captures errors that occur WITHIN a running application, but tells you
nothing if the application is simply DOWN entirely (server crashed,
Docker container failed to restart, DNS misconfigured, SSL certificate
expired) — a fundamentally different, complementary failure mode that
needs external, outside-the-infrastructure monitoring to catch, since
an application that's completely down can't report its own errors to
Sentry from inside itself.

TASK
Configure external uptime monitoring against both the backend API and
the frontend, using Epic 22's existing health-check endpoint as the
backend monitoring target.

REQUIREMENTS
- This task is PRIMARILY external-service configuration, not code —
  select an uptime-monitoring service (options in this space include
  various third-party uptime-ping services with free/low-cost tiers
  suitable for a project at this stage — verify current, actively-
  available options rather than assuming any specific historical
  service name is still operating/relevant, since this specific
  category of tooling changes over time) and configure:
  1. A check against the BACKEND's `/api/health/` endpoint (Epic 22
     Task 22.1.1.4 — which already checks BOTH database AND Celery
     connectivity, making it a genuinely meaningful availability signal,
     not just "did the process respond to any request at all") — at a
     reasonable interval (every 1–5 minutes is typical for this
     category of service).
  2. A check against the FRONTEND's homepage URL (a simple HTTP
     200-check confirming the static site/SPA shell is being served
     correctly — this doesn't need the richer health-check logic the
     backend endpoint has, just basic reachability).
  3. Alert routing — configure the monitoring service to notify via
     whatever channel is actually appropriate for this project's real
     operational setup (email, SMS via a service the monitoring tool
     supports, a webhook into a team chat tool) — a monitoring check
     with no one actually receiving its alerts provides zero real
     value; this step is not optional polish.
- Consider whether the health-check endpoint itself needs any
  adjustment for this specific use case: Epic 22 Task 22.1.1.4's
  `/api/health/` endpoint BLOCKS for up to 5 seconds waiting on a real
  Celery round-trip — for an EXTERNAL uptime monitor pinging this
  endpoint frequently (e.g. every minute), this adds real, repeated
  load (a queued Celery task, every single check interval, forever) —
  weigh whether this is an acceptable ongoing cost, or whether the
  EXTERNAL uptime check should hit a LIGHTER-WEIGHT endpoint (e.g. a
  simple `/api/health/ping/` checking only that the process is
  responding + a fast DB connectivity check, WITHOUT the Celery
  round-trip) reserving the FULLER `/api/health/` check for less-
  frequent, more thorough verification (e.g. manually, or as part of a
  deploy-verification step per Epic 25's work) — RECOMMEND adding a
  lightweight `/api/health/live/` endpoint SPECIFICALLY for high-
  frequency external ping monitoring (checking only process
  responsiveness + a fast DB `SELECT 1`, no Celery round-trip), keeping
  the fuller `/api/health/` (with its Celery check) for lower-frequency/
  manual verification use — this is a genuine, worthwhile distinction
  between "is the process alive and reachable" (needed frequently,
  cheaply) and "is the FULL pipeline including background task
  processing healthy" (valuable but more expensive to check, needed
  less often).
  ```python
  # core/views.py, alongside the existing HealthCheckView
  class LivenessCheckView(APIView):
      permission_classes = [AllowAny]

      def get(self, request):
          from django.db import connection
          try:
              with connection.cursor() as cursor:
                  cursor.execute("SELECT 1")
              return Response({"status": "ok"}, status=status.HTTP_200_OK)
          except Exception:
              return Response({"status": "error"}, status=status.HTTP_503_SERVICE_UNAVAILABLE)
  ```
  Register at `/api/health/live/`, and point the external uptime
  monitor's high-frequency check at THIS endpoint rather than the
  fuller, Celery-touching one.

ACCEPTANCE CRITERIA / TESTS
- Add a test for the new `LivenessCheckView` (fast-path health check,
  DB-only, no Celery dependency).
- Manually verify: the external monitoring service, once configured,
  successfully reports the application as "up" under normal operation;
  deliberately stopping the backend container (in staging) and
  confirming the monitoring service correctly detects and alerts on
  the outage within its configured check interval — the actual,
  concrete proof this monitoring provides real value, not just that
  it's configured to look correct.
```

---

#### Task 26.1.1.4 — Payment-gateway callback failure alerting

```
You are working in backend/payments/views.py, tasks.py. Assume Task
26.1.1.1 (Sentry) is already merged, and Epic 6's full payment
callback/reconciliation infrastructure (Tasks 6.2.1.4, 6.2.1.5,
6.4.2.1) is in place.

CONTEXT
Per this document's header, payment-gateway failures throughout Epic 6
are already CAUGHT and LOGGED (never left to crash the application) —
but "logged" and "someone gets alerted in a timely way" are different
things, and payment failures specifically deserve the LATTER, not just
the former: a repeated pattern of payment VERIFICATION failures could
indicate a genuine gateway outage (per Epic 6 Task 6.2.1.6's own
sandbox/production-mode discussion) actively costing this business real
completed sales, and is exactly the kind of issue that should surface
proactively, not be discovered hours later by someone happening to
notice a revenue dip.

TASK
Add explicit, elevated alerting specifically for payment-gateway
callback failures, distinct from (and more urgent than) this project's
general error logging.

REQUIREMENTS
- Add explicit Sentry event capture with ELEVATED severity/context at
  the SPECIFIC point Epic 6 Task 6.2.1.4's `PaymentCallbackView`
  handles a verification failure (`_mark_failed()`), and Epic 6 Task
  6.4.2.1's `reconcile_stuck_payment_transactions` task, going BEYOND
  the existing general exception logging with something more
  deliberately actionable:
  ```python
  # payments/views.py, inside PaymentCallbackView._mark_failed()
  import sentry_sdk

  def _mark_failed(self, txn):
      with transaction.atomic():
          ...  # existing logic unchanged
      sentry_sdk.capture_message(
          f"Payment verification failed for order {txn.order.order_number} via {txn.gateway}",
          level="warning",
          # attach non-PII context: gateway name, transaction id, amount —
          # NOT customer name/address/contact info, per Task 26.1.1.1's
          # PII-handling principle established for this whole project
          scope=lambda scope: scope.set_context("payment", {
              "gateway": txn.gateway, "transaction_id": txn.id,
              "amount": txn.amount, "order_number": txn.order.order_number,
          }),
      )
  ```
- Add PATTERN-based alerting specifically — a SINGLE payment failure
  is normal, expected background noise (a customer's card genuinely
  declined, a customer abandoning checkout) and should NOT page anyone;
  a CLUSTER of failures in a short window is the genuinely actionable
  signal (a real gateway outage). Implement a simple rate-based check:
  ```python
  # payments/services.py or wherever the shared finalize functions
  # from Epic 6 Task 6.4.2.1's refactor live
  from django.core.cache import cache

  def check_payment_failure_rate(gateway: str):
      """
      Track recent payment failures per gateway in a short rolling
      window; escalate to a more urgent Sentry alert if the failure
      COUNT crosses a threshold suggesting a genuine outage rather than
      normal, expected individual declines.
      """
      cache_key = f"payment_failures:{gateway}"
      count = cache.get(cache_key, 0) + 1
      cache.set(cache_key, count, timeout=600)  # 10-minute rolling window
      FAILURE_THRESHOLD = 5
      if count == FAILURE_THRESHOLD:  # fire exactly once when crossing the threshold, not on every subsequent failure
          sentry_sdk.capture_message(
              f"Payment gateway '{gateway}' has had {FAILURE_THRESHOLD}+ failures in the last 10 minutes — possible outage.",
              level="error",  # elevated from "warning" — this is the genuinely urgent signal
          )
  ```
  Call `check_payment_failure_rate(txn.gateway)` from the SAME
  `_mark_failed()` call site above, and from the reconciliation task's
  failure path (Epic 6 Task 6.4.2.1). Using Epic 21's now-established
  Redis cache (Task 21.1.1.1) as the rolling-counter storage is a
  reasonable, lightweight mechanism — no new infrastructure needed,
  reusing what this project already has.
  Note the `count == FAILURE_THRESHOLD` (exact equality, not `>=`) is
  deliberate — firing the ESCALATED alert exactly ONCE when the
  threshold is first crossed, not on every subsequent failure past it
  (which would spam the SAME alert repeatedly for the duration of an
  ongoing outage, adding noise rather than useful signal once the
  initial alert has already informed someone) — a real, worthwhile
  alerting-hygiene detail.
- Consider whether Epic 6 Task 6.3.1.3's gateway FALLBACK mechanism
  (automatically trying a secondary gateway if the primary fails)
  should also be reflected in this alerting — a failure of the PRIMARY
  gateway that's SUCCESSFULLY recovered via fallback is a meaningfully
  DIFFERENT, less urgent situation than a failure with NO successful
  fallback — if straightforward given the existing fallback code's
  structure, distinguish these two cases in the alert (e.g. "primary
  gateway X failed but fallback to Y succeeded" as a lower-severity
  informational note, vs. "ALL configured gateways failed" as the
  genuinely urgent case) — implement this distinction if it fits
  cleanly into the existing Task 6.3.1.3 code without a disproportionate
  rewrite; otherwise note it as a reasonable future refinement rather
  than force-fitting it into this task.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/payments/tests/:
1. A single payment failure captures a `warning`-level Sentry message
   with the correct non-PII context (mock `sentry_sdk.capture_message`
   and assert correct call arguments).
2. Failures below the threshold (e.g. the 4th failure in the window)
   do NOT trigger the elevated `error`-level alert.
3. The 5th failure within the 10-minute window correctly triggers the
   elevated alert EXACTLY ONCE — confirm a 6th, 7th, etc. failure
   within the SAME window does NOT re-trigger it (proving the
   exact-equality threshold check works as intended).
4. After the 10-minute window expires (advance time past the cache
   TTL), a NEW failure correctly restarts the count from a fresh
   window rather than continuing to accumulate indefinitely.
5. Confirm NO PII (customer name, email, phone, address) appears
   anywhere in the captured Sentry context for either alert type —
   a direct, explicit test of the privacy principle established
   throughout this epic, not just an assumption.
Also close Epic 25 Task 25.1.1.5's explicitly-flagged dependency: wire
`backup_database()`'s failure path (that task's own TODO) into this
SAME Sentry-capture pattern now that it's genuinely available —
```python
# core/tasks.py, backup_database()
except Exception as exc:
    sentry_sdk.capture_exception(exc)
    raise
```
Add a test confirming a failed backup task correctly captures a Sentry
exception event, closing that cross-epic loop explicitly.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 26.1.1.1 | Integrate Sentry (backend + frontend) | ☐ |
| 26.1.1.2 | Structured JSON logging config | ☐ |
| 26.1.1.3 | Uptime monitoring (external ping service) | ☐ |
| 26.1.1.4 | Payment-gateway callback failure alerting (+ close Epic 25's backup-alerting TODO) | ☐ |

**This epic closes the last explicitly-flagged forward-reference in this document series** — Epic 25 Task 25.1.1.5's backup-failure alerting note is resolved directly in Task 26.1.1.4 above, the same way Epic 16 closed out Epics 4/8/11's notification TODOs, and Epic 14 closed the Persian-slug TODO Epic 3's migration left behind. With Epics 1–26 of this backlog now covered, the remaining epics in the master backlog (27 — Analytics, 28 — Marketing, 29 — Frontend Refactor, 30 — Documentation) are the ones the master backlog's own execution-order notes describe as either post-launch growth work or continuous, ongoing concerns rather than epics with the same kind of hard sequencing dependencies this first block of 26 has had throughout.
