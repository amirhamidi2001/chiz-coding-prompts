# Epic 22 — Celery & Async Tasks — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next. This is a small epic (4 tasks) but an unusually high-stakes one: **a large number of tasks across prior epics in this series were explicitly written assuming Celery infrastructure already exists**, with each one carrying a caveat like *"confirm Celery/django-celery-beat is actually set up before starting; if not, this task depends on Epic 22 Phase 22.1 landing first."* This epic is where every one of those deferred assumptions finally gets a real foundation.

**Assumed prerequisites:** Epics 1–21 are fully merged, including Epic 21's documented Redis DB allocation scheme (Task 21.1.1.1's header comment: DB 0 = Channels, DB 1 = Celery broker, DB 2 = cache, DB 3 = sessions).

**Confirmed directly from the repo:** `docker-compose.dev.yml` and `docker-compose.prod.yml` **already exist** with a complete `db`/`redis`/`backend`/`frontend` service topology — `redis` is already a running service in both (image `redis:7`, healthchecked via `redis-cli ping`) with **no host port exposed in production** (only reachable from other containers) and **port 6379 exposed to the host in dev**. Production runs the backend via **Daphne ASGI** (`daphne -b 0.0.0.0 -p 8000 core.asgi:application`), confirming the Channels/WebSocket setup from Epics 2/16 is genuinely served correctly in production, not just locally. `backend/core/__init__.py` is currently **empty** — no Celery app is wired in at all yet.

**A complete inventory of every `@shared_task` already written across prior epics' prompts, which this epic's autodiscovery and beat schedule must correctly pick up** (verify each actually exists in your real codebase at this point, since exact task-writing order may have varied):
- **Periodic (need a beat schedule entry):** `shop.tasks.deactivate_expired_variants` (Epic 3, nightly), `payments.tasks.reconcile_stuck_payment_transactions` (Epic 6, every 10–15 min), `shipping.tasks.poll_shipment_tracking` (Epic 7, every 30–60 min).
- **Triggered on-demand only (no beat schedule needed, just correct autodiscovery):** `shop.tasks.notify_stock_alert_subscribers` (Epic 4), `shop.tasks.notify_price_drop_subscribers` (Epic 11), `notifications.tasks.retry_sms_notification` (Epic 16), `notifications.email.send_notification_email_task` (Epic 16).

---

## Phase 22.1 — Celery Setup

### Feature 22.1.1 — Task Queue Infrastructure

---

#### Task 22.1.1.1 — Add Celery + `django-celery-beat` to requirements

```
You are working in backend/requirements.txt. Assume Epics 1–21 are
fully merged.

CONTEXT
Neither `celery` nor `django-celery-beat` appears anywhere in
requirements.txt — confirmed by grep. Every prior epic's Celery-
dependent task has been building on an assumption this task is the
first to actually satisfy.

TASK
Add Celery and its Django-integrated periodic-task scheduler to the
project's dependencies.

REQUIREMENTS
- Add to backend/requirements.txt, pinned to current stable versions
  matching this project's fully-pinned convention, placed in correct
  alphabetical order per the file's existing convention:
  - `celery` — the task queue library itself.
  - `django-celery-beat` — provides `DatabaseScheduler`, letting
    periodic task schedules be stored in the database and managed via
    Django admin (rather than only via a static `CELERY_BEAT_SCHEDULE`
    dict in settings) — this is the RECOMMENDED approach for this
    project specifically, since it lets ops/admin staff adjust
    schedules (e.g. changing how often `poll_shipment_tracking` runs)
    WITHOUT a code deploy, which is valuable for a project that's
    already established a strong pattern of admin-configurable
    operational parameters (Epic 6 Task 6.3.1.3's `PaymentGatewayConfig`,
    Epic 7's `ShippingCarrier.is_active` toggles) — use the SAME
    admin-configurability philosophy here rather than defaulting to
    hardcoded settings-file schedules.
- Confirm the already-installed `redis==7.4.0` package version is
  compatible with whatever Celery version you pin — Celery's Redis
  broker/backend support (`celery[redis]` extra, or a compatible
  standalone `redis` version) has real version-compatibility
  constraints worth verifying rather than assuming any two versions
  work together.
- Add `"django_celery_beat"` to `INSTALLED_APPS` in
  backend/core/settings/base.py (required for the `DatabaseScheduler`
  approach — it needs its own database tables for storing schedules),
  and generate/apply its migrations.

ACCEPTANCE CRITERIA / TESTS
- `pip install -r backend/requirements.txt` (or this environment's
  established install command) resolves with no dependency conflicts.
- `python manage.py migrate` successfully creates
  `django_celery_beat`'s tables.
- `python manage.py check` passes with the new app installed.
- No functional Celery tests yet — this task is dependency
  installation only; Task 22.1.1.2 is where the actual app
  configuration and functional testing begins.
```

---

#### Task 22.1.1.2 — Create `core/celery.py` app config

```
You are working in backend/core/celery.py (new file),
backend/core/__init__.py, backend/core/settings/base.py. Assume Task
22.1.1.1 is already merged.

CONTEXT — READ THIS DOCUMENT'S HEADER BEFORE STARTING
Every `@shared_task`-decorated function across the entire codebase
(the full inventory listed in this document's header, spanning `shop`,
`payments`, `shipping`, and `notifications`) currently exists as dead
code with no Celery app instance to register itself against — none of
these tasks can actually run, be discovered, or be scheduled until
this task wires up the real Celery application.

TASK
Create the standard Django-Celery bootstrap (`core/celery.py`), with
autodiscovery correctly finding every existing `tasks.py` module across
every app, using the Redis broker on the DB index Epic 21 already
reserved for this purpose (DB 1, per that epic's documented allocation
scheme), and register the beat schedule for the three periodic tasks
identified in this document's header.

REQUIREMENTS
- Create backend/core/celery.py, following Celery's standard, well-
  established Django integration pattern:
  ```python
  import os
  from celery import Celery

  os.environ.setdefault("DJANGO_SETTINGS_MODULE", "core.settings.development")

  app = Celery("chiz")
  app.config_from_object("django.conf:settings", namespace="CELERY")
  app.autodiscover_tasks()


  @app.task(bind=True, ignore_result=True)
  def debug_task(self):
      print(f"Request: {self.request!r}")
  ```
  Note the `DJANGO_SETTINGS_MODULE` default here (`core.settings.development`)
  MUST be overridden correctly for production — verify this project's
  actual settings-module-selection mechanism (check `manage.py`/`wsgi.py`/
  `asgi.py` for how they currently select between
  `core.settings.development`/`core.settings.production` — likely via a
  `DJANGO_SETTINGS_MODULE` environment variable already set differently
  per environment, per the `.env.dev`/`.env.prod` files referenced in
  the confirmed docker-compose setup) and make sure `celery.py`'s
  `setdefault` call doesn't accidentally FORCE development settings in
  production if the env var is somehow unset — `setdefault` only
  applies if the variable ISN'T already set, so as long as the
  container's environment correctly sets `DJANGO_SETTINGS_MODULE`
  before this module loads (which it should, given the existing
  `env_file: .env.prod`/`.env.dev` pattern), this default is just a
  safe fallback, not an override — confirm this reasoning holds for
  this project's actual startup sequence rather than assuming.
- Update backend/core/__init__.py (currently empty, confirmed) to
  ensure the Celery app is loaded when Django starts:
  ```python
  from .celery import app as celery_app

  __all__ = ("celery_app",)
  ```
  (this is the standard, required Django-Celery integration pattern —
  ensures `@shared_task`-decorated functions across the project
  correctly bind to this specific Celery app instance).
- Add Celery-specific settings to backend/core/settings/base.py, using
  the `REDIS_DB_CACHE`-style pattern already established in Epic 21
  Task 21.1.1.1 for consistency, and Epic 21's documented DB 1
  allocation for Celery:
  ```python
  REDIS_DB_CELERY = config("REDIS_DB_CELERY", default=1, cast=int)
  CELERY_BROKER_URL = f"redis://{config('REDIS_HOST', default='127.0.0.1')}:{config('REDIS_PORT', default=6379)}/{REDIS_DB_CELERY}"
  CELERY_RESULT_BACKEND = CELERY_BROKER_URL  # same Redis instance/DB serves as both broker and result store — acceptable for this project's scale; revisit if result-storage volume ever becomes a real operational concern
  CELERY_ACCEPT_CONTENT = ["json"]
  CELERY_TASK_SERIALIZER = "json"
  CELERY_RESULT_SERIALIZER = "json"
  CELERY_TIMEZONE = TIME_ZONE  # reuse the project's already-configured Asia/Tehran timezone (Epic 14 Task 14.1.1.2), so scheduled task times are interpreted consistently with the rest of the application
  CELERY_BEAT_SCHEDULER = "django_celery_beat.schedulers.DatabaseScheduler"
  ```
  Using `CELERY_TIMEZONE = TIME_ZONE` (reusing Epic 14's `Asia/Tehran`
  setting rather than leaving Celery on its own default UTC
  interpretation) is a genuinely important detail — without it, a
  periodic task an admin schedules via the database scheduler
  expecting "run at 2 AM" could run at 2 AM UTC instead of 2 AM Iran
  time, a confusing, easy-to-miss mismatch.
- Register the beat schedule for the THREE periodic tasks identified in
  this document's header. Since `CELERY_BEAT_SCHEDULER` is set to the
  `DatabaseScheduler` (per the admin-configurability rationale in Task
  22.1.1.1), the PRIMARY way to register these is via
  `django_celery_beat`'s admin-manageable `PeriodicTask`/
  `IntervalSchedule`/`CrontabSchedule` models, NOT a static
  `CELERY_BEAT_SCHEDULE` dict in settings (which the `DatabaseScheduler`
  effectively ignores/supersedes once the database has entries) — add
  a data migration (in whichever app makes most sense — `core` doesn't
  have its own migrations directory typically; consider adding this to
  one of the actual task-owning apps, e.g. a new
  `shop/migrations/00XX_seed_periodic_tasks.py`, or centralize all
  three seed entries in one place if that reads more cleanly) seeding
  the three known periodic tasks as real `PeriodicTask` database rows:
  ```python
  from django.db import migrations

  def seed_periodic_tasks(apps, schema_editor):
      IntervalSchedule = apps.get_model("django_celery_beat", "IntervalSchedule")
      CrontabSchedule = apps.get_model("django_celery_beat", "CrontabSchedule")
      PeriodicTask = apps.get_model("django_celery_beat", "PeriodicTask")

      nightly, _ = CrontabSchedule.objects.get_or_create(minute="0", hour="2", day_of_week="*", day_of_month="*", month_of_year="*")
      PeriodicTask.objects.get_or_create(
          name="Deactivate expired product variants",
          task="shop.tasks.deactivate_expired_variants",
          crontab=nightly,
      )

      every_15min, _ = IntervalSchedule.objects.get_or_create(every=15, period="minutes")
      PeriodicTask.objects.get_or_create(
          name="Reconcile stuck payment transactions",
          task="payments.tasks.reconcile_stuck_payment_transactions",
          interval=every_15min,
      )

      every_30min, _ = IntervalSchedule.objects.get_or_create(every=30, period="minutes")
      PeriodicTask.objects.get_or_create(
          name="Poll shipment tracking status",
          task="shipping.tasks.poll_shipment_tracking",
          interval=every_30min,
      )

  def unseed_periodic_tasks(apps, schema_editor):
      PeriodicTask = apps.get_model("django_celery_beat", "PeriodicTask")
      PeriodicTask.objects.filter(
          task__in=[
              "shop.tasks.deactivate_expired_variants",
              "payments.tasks.reconcile_stuck_payment_transactions",
              "shipping.tasks.poll_shipment_tracking",
          ]
      ).delete()

  class Migration(migrations.Migration):
      dependencies = [
          ("django_celery_beat", "0001_initial"),  # adjust to the actual installed migration name
          ("shop", "0XXX_previous_migration"),  # adjust to the actual latest shop migration
      ]
      operations = [migrations.RunPython(seed_periodic_tasks, unseed_periodic_tasks)]
  ```
  Verify the EXACT task import paths (`shop.tasks.deactivate_expired_variants`
  etc.) match reality in your actual codebase before finalizing this
  migration — a typo'd task path here would silently schedule a task
  that doesn't exist, which Celery would only surface as a runtime
  error when the schedule actually fires, not at migration time.

ACCEPTANCE CRITERIA / TESTS
- `python manage.py shell -c "from core.celery import app; print(app.tasks.keys())"`
  (or equivalent) lists EVERY task from this document's header
  inventory — the direct, concrete proof that autodiscovery correctly
  found every app's `tasks.py` module, not just a subset.
- Add a test confirming the three periodic tasks were correctly seeded
  as `PeriodicTask` rows with the correct schedule types/intervals
  after running migrations.
- Add a test confirming `CELERY_TIMEZONE` matches `settings.TIME_ZONE`
  (`Asia/Tehran`) — a simple but worthwhile regression guard against
  this specific, easy-to-silently-break detail.
```

---

#### Task 22.1.1.3 — Add Celery worker + beat services to `docker-compose.dev.yml`/`prod.yml`

```
You are working in docker-compose.dev.yml, docker-compose.prod.yml.
Assume Task 22.1.1.2 is already merged. RE-READ this document's header
— both compose files' EXACT existing structure (service naming,
healthcheck patterns, env_file usage, volume conventions) is
confirmed and must be matched precisely, not approximated.

CONTEXT
Both compose files already run `db` and `redis` services with
healthchecks, and a `backend` service depending on both being healthy
before starting. Neither file has ANY Celery worker or beat service —
even after Task 22.1.1.2's code-level wiring, nothing would actually
EXECUTE any queued task without a running worker process, and nothing
would EVER FIRE the three periodic tasks without a running beat
process.

TASK
Add `celery_worker` and `celery_beat` services to both compose files,
matching each file's exact existing conventions precisely.

REQUIREMENTS — docker-compose.dev.yml
- Add, following the EXACT structural pattern of the existing
  `backend` service (same build context/target, same `env_file`, same
  `depends_on` health-condition pattern):
  ```yaml
  # ── Celery worker — processes queued background tasks ──────────────────────
  celery_worker:
    build:
      context: ./backend
      target: development
    container_name: backend_celery_worker_dev
    restart: unless-stopped
    command: celery -A core worker --loglevel=info
    volumes:
      - ./backend:/app
      - media_volume_dev:/app/media
    env_file:
      - .env.dev
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  # ── Celery beat — schedules periodic tasks ──────────────────────────────────
  celery_beat:
    build:
      context: ./backend
      target: development
    container_name: backend_celery_beat_dev
    restart: unless-stopped
    command: celery -A core beat --loglevel=info --scheduler django_celery_beat.schedulers:DatabaseScheduler
    volumes:
      - ./backend:/app
    env_file:
      - .env.dev
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
  ```
  Note `celery_worker` mounts the `media_volume_dev` volume (matching
  the `backend` service) since some queued tasks may need to read/
  write media files (e.g. Epic 8's invoice PDF generation, if it's
  ever queued rather than synchronous — check whether it currently is;
  even if not, other future media-touching tasks might be, so this is
  a reasonable, low-cost inclusion) — `celery_beat` does NOT need this,
  since beat only SCHEDULES tasks, it never executes their actual
  logic or touches files itself.
  Both new services correctly depend on `db`/`redis` being HEALTHY
  before starting, matching the EXACT dependency pattern already used
  by the `backend` service — a worker starting before Postgres/Redis
  are ready would crash-loop uselessly.
  Consider whether these new services should also `depends_on: backend`
  specifically (not just `db`/`redis`) — since `backend`'s startup
  command runs `python manage.py migrate` BEFORE starting the dev
  server, and Task 22.1.1.2's periodic-task-seeding migration needs to
  have RUN before `celery_beat` can correctly find the seeded
  `PeriodicTask` rows — add `depends_on: backend: condition: service_started`
  (not `service_healthy`, since the `backend` service in THIS compose
  file has no explicit healthcheck defined — check whether adding one
  is warranted, or whether `service_started` is an acceptable, simpler
  dependency condition given this project's existing patterns) to
  both new services to ensure migrations have at least been INITIATED
  before Celery services start.

REQUIREMENTS — docker-compose.prod.yml
- Add the equivalent services, adapted to match the PRODUCTION file's
  distinct conventions (no source-code volume mount, `target: production`,
  `.env.prod`, no host port exposure needed since these services expose
  no network ports at all):
  ```yaml
  celery_worker:
    build:
      context: ./backend
      target: production
    container_name: backend_celery_worker_prod
    restart: unless-stopped
    command: celery -A core worker --loglevel=info --concurrency=4
    volumes:
      - media_volume_prod:/app/media
    env_file:
      - .env.prod
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  celery_beat:
    build:
      context: ./backend
      target: production
    container_name: backend_celery_beat_prod
    restart: unless-stopped
    command: celery -A core beat --loglevel=info --scheduler django_celery_beat.schedulers:DatabaseScheduler
    env_file:
      - .env.prod
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
  ```
  Note the added `--concurrency=4` on the PRODUCTION worker
  specifically (a deliberate, explicit worker-process-count choice for
  real traffic, vs. dev's default single-process behavior being
  perfectly adequate for local development) — adjust the exact number
  based on this project's actual expected task volume/server resources
  if you have more specific guidance; `4` is a reasonable, moderate
  starting point worth documenting as a tunable rather than treating
  as a permanently-correct fixed value.
  Confirm the PRODUCTION Dockerfile's `production` build target
  actually includes everything Celery needs to run (the SAME image
  used for the `backend` service, per the shared `context: ./backend`/
  `target: production` — since `celery_worker`/`celery_beat` build from
  the EXACT SAME image as `backend`, no separate Dockerfile/target is
  needed; this is the standard, correct approach for a Django+Celery
  project — the image contains the full Django project either way,
  and which PROCESS runs inside the container is determined purely by
  the `command:` override, not by needing a different image).

ACCEPTANCE CRITERIA / TESTS
- Manually verify (this is fundamentally an infrastructure/ops task,
  not something unit-testable in the traditional sense): running
  `docker compose -f docker-compose.dev.yml up --build` starts all
  services including the two new ones without errors; the worker log
  output shows it successfully connecting to the Redis broker and
  discovering the expected task list (cross-check against this
  document's header inventory); the beat log output shows it correctly
  loading the `DatabaseScheduler` and the three seeded periodic
  schedules.
- Trigger one on-demand task manually (e.g. via
  `docker compose exec backend python manage.py shell` and calling
  `shop.tasks.deactivate_expired_variants.delay()`) and confirm the
  `celery_worker` container's logs show it picking up and executing
  the task — the real, concrete end-to-end proof the whole pipeline
  (Django → Redis broker → worker process) actually works, not just
  that the YAML is syntactically valid.
- Add this manual verification procedure to whatever local-dev-setup
  documentation this project maintains (per Epic 30's future
  documentation work, if landed, or just a README update now) so
  future developers have a known-good way to confirm Celery is working
  correctly in their own environment.
```

---

#### Task 22.1.1.4 — Health check / smoke-test task

```
You are working in backend/core/tasks.py (new file), and wherever this
project's existing health-check mechanism lives (check for a
`check_health` management command or health-check endpoint from any
prior epic's DevOps-adjacent work — the master backlog references one
existing from earlier project history; confirm its current location
and pattern before adding to it). Assume Task 22.1.1.3 is already
merged.

CONTEXT
Task 22.1.1.3's manual verification procedure (triggering a task via
Django shell and eyeballing worker logs) proves the pipeline works
ONCE, by hand — there's no AUTOMATED, repeatable way to confirm
"Celery is currently healthy and processing tasks" as part of this
project's ongoing operations (e.g. as part of a deploy-verification
step, or an admin-visible health dashboard).

TASK
Add a trivial Celery smoke-test task, and wire it into whatever
health-check mechanism this project already has (or add a minimal new
one if none exists), providing an automated, repeatable way to confirm
the full pipeline is actually functioning.

REQUIREMENTS
- Add backend/core/tasks.py:
  ```python
  from celery import shared_task
  from django.utils import timezone


  @shared_task
  def celery_health_check() -> str:
      return f"Celery is healthy as of {timezone.now().isoformat()}"
  ```
- Add a health-check endpoint/management-command integration that
  ACTUALLY EXERCISES this task end-to-end (queues it, waits briefly for
  a result, confirms it completed) rather than just confirming the
  Celery APP OBJECT is importable (which would prove far less — an
  app object can exist and be correctly configured while the actual
  broker connection or worker process is down):
  ```python
  # core/views.py or wherever a health-check view already exists
  from core.tasks import celery_health_check

  class HealthCheckView(APIView):
      permission_classes = [AllowAny]

      def get(self, request):
          checks = {"database": self._check_database(), "celery": self._check_celery()}
          all_healthy = all(checks.values())
          return Response(
              {"status": "healthy" if all_healthy else "unhealthy", "checks": checks},
              status=status.HTTP_200_OK if all_healthy else status.HTTP_503_SERVICE_UNAVAILABLE,
          )

      def _check_database(self) -> bool:
          from django.db import connection
          try:
              with connection.cursor() as cursor:
                  cursor.execute("SELECT 1")
              return True
          except Exception:
              return False

      def _check_celery(self) -> bool:
          try:
              result = celery_health_check.apply_async()
              result.get(timeout=5)  # block briefly waiting for a real worker to pick it up and complete it
              return True
          except Exception:
              return False
  ```
  The `result.get(timeout=5)` call is the crux of this task — it
  BLOCKS waiting for an actual worker process to pick up and complete
  the task within 5 seconds, genuinely proving the full pipeline
  (broker connectivity + at least one live worker process) is
  functioning, not just that Django CAN construct a task message. A
  short timeout is deliberate: a health-check endpoint itself
  shouldn't hang indefinitely if Celery IS down — 5 seconds is a
  reasonable, bounded wait that clearly distinguishes "healthy and
  fast" from "broken or badly degraded," without making the health
  endpoint itself a slow, unpleasant thing to poll regularly.
  If this project already has an EXISTING `check_health` management
  command (per the master backlog's own reference to one from earlier,
  broader project history) rather than/in addition to a REST endpoint,
  integrate the SAME Celery-check logic into that command too, so
  BOTH the CLI-based and HTTP-based health checks agree and stay in
  sync — extract the actual check logic into a shared function
  (`core/health.py`) both the view and the management command call,
  rather than duplicating the `apply_async()`/`.get()` logic in two
  places.
- Register the health-check URL (if adding a new endpoint) at a
  conventional, memorable path — `/api/health/` or `/health/` — check
  whether this project's existing infra/monitoring conventions (if any
  established by this point) expect a specific path, and match it;
  otherwise, `/api/health/` is a reasonable default consistent with
  this project's existing `/api/`-prefixed URL structure.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/core/tests/test_health.py (new file):
1. With a real, running Celery worker available (this test needs
   Celery configured to run tasks EAGERLY in the test environment —
   check whether `CELERY_TASK_ALWAYS_EAGER = True` is already set for
   tests via this project's test settings, or add it; running eagerly
   means the task executes synchronously in-process rather than
   needing a real separate worker process, which is the standard,
   correct way to test Celery-dependent code without spinning up real
   infrastructure in a test suite), the health-check endpoint returns
   `200` with `checks.celery: true`.
2. With the database check DELIBERATELY forced to fail (mock
   `connection.cursor` to raise), the endpoint returns `503` with
   `checks.database: false`, `checks.celery` still evaluated
   independently and correctly reported (proving the two checks are
   genuinely independent, not one failure masking the other's real
   status).
3. With Celery deliberately made to fail (mock `celery_health_check.apply_async`
   to raise, or mock `.get()` to raise a timeout), confirm the endpoint
   correctly returns `503` with `checks.celery: false` and does NOT
   hang for longer than the configured timeout.
Manually verify against the REAL Docker Compose stack from Task
22.1.1.3 (not just the eager-mode test suite): stop the `celery_worker`
container specifically (leaving `redis`/`db`/`backend` running), hit
the health endpoint, and confirm it correctly reports `checks.celery: false`
within the expected ~5 second timeout window — the actual, real-world
proof this health check would catch a genuine "worker process died"
incident in production, not just a mocked unit-test approximation of
one.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 22.1.1.1 | Add Celery + django-celery-beat to requirements | ☐ |
| 22.1.1.2 | Create core/celery.py app config + seed periodic tasks | ☐ |
| 22.1.1.3 | Add Celery worker + beat services to docker-compose | ☐ |
| 22.1.1.4 | Health check / smoke-test task | ☐ |

**A final note specific to this epic's place in the overall backlog:** per the master backlog's own execution-order guidance, Celery infrastructure was flagged as one of the "late" epics with components that are "cheap to do early and expensive to retrofit" — meaning a real engineering team would likely have pulled this epic's 4 tasks forward, much earlier than its numbered position, rather than leaving every dependent task across Epics 3/4/6/16 sitting on an unmet assumption for this long. If you're generating these prompts to actually guide sequential implementation rather than following this document series' fixed numbering, strongly consider building Epic 22 immediately after Epic 1, before any of the epics that depend on it.

Once Epic 22 is fully merged, the next epic to generate prompts for is
**Epic 23 — Security Hardening**, which the master backlog similarly
flags as having P0 items (throttling, file validation, login lockout)
that ideally should have been pulled forward alongside Epic 1/2 rather
than saved for their numbered position — worth the same consideration
before starting it.
