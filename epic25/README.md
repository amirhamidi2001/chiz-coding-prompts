# Epic 25 — DevOps / Docker / CI-CD — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–24 are fully merged, including Epic 22's Celery worker/beat services and Epic 23/24's new CI jobs (vulnerability scanning, E2E tests).

**A real, currently-live discrepancy confirmed directly in the repo — read this before starting Task 25.1.1.3:** `.github/workflows/ci.yml`'s own header comment block explicitly documents four jobs:
```
#   backend  — pytest with real PostgreSQL + Redis services
#   frontend — vitest
#   docker   — build both images (validates Dockerfiles)
#   deploy   — SSH into server, pull, restart prod stack (main branch only)
```
**But the actual file contains only three jobs — `backend`, `frontend`, and `docker`. There is no `deploy` job anywhere in the file.** The comment describes a deploy mechanism that either was removed at some point without updating the header, or was planned and documented but never actually implemented. Either way, **this project currently has zero automated deployment of any kind** — every merge to `main`, despite what the comment implies, does not actually get deployed anywhere by CI. This is a significant, concrete finding this epic must address, not a hypothetical gap to guess at.

---

## Phase 25.1 — Environment Maturity

### Feature 25.1.1 — Staging & Deployment Safety

---

#### Task 25.1.1.1 — Add `docker-compose.staging.yml`

```
You are working in docker-compose.staging.yml (new file). Assume
Epics 1–24 are fully merged. Base this closely on the CONFIRMED
structure of the existing docker-compose.prod.yml (Daphne ASGI
backend, nginx-served frontend, db/redis services, Celery
worker/beat services per Epic 22 Task 22.1.1.3) — staging should mirror
production's architecture as closely as possible, since the entire
POINT of a staging environment is catching production-shaped problems
before they reach production; a staging environment that's
architecturally different from prod (e.g. running the dev server
instead of Daphne) would fail to catch exactly the class of bugs
staging exists to catch.

TASK
Create a staging Docker Compose configuration that mirrors production's
service topology, pointed at staging-specific configuration (database,
external service sandbox modes, domain).

REQUIREMENTS
- Copy `docker-compose.prod.yml`'s structure as the starting point —
  SAME service list (db, redis, backend via Daphne, frontend via
  nginx, celery_worker, celery_beat per Epic 22), SAME `target: production`
  build stage (staging should run the SAME production-optimized image,
  not the dev-mode image — this is important: staging exists to
  validate the ACTUAL artifact that would be deployed to production,
  not a differently-built one).
- The key differences from prod:
  - `env_file: .env.staging` (a new, staging-specific env file — add
    `.env.staging` to `.gitignore` per the exact existing pattern
    already covering `.env.dev`/`.env.prod`).
  - Distinct container names (`_staging` suffix instead of `_prod`) and
    a distinct Docker network name, so staging and production can
    potentially run on the SAME host (e.g. during initial setup, or for
    smaller-scale deployments) without any container-name or network
    collision.
  - Distinct volume names (`postgres_data_staging`,
    `media_volume_staging`) — staging must have its OWN, separate
    database and media storage, never sharing prod's actual data.
  - External service SANDBOX modes should be the deliberate default for
    staging — `ZARINPAL_SANDBOX=True` (Epic 6), and staging's
    `.env.staging` should be documented as expected to use
    sandbox/test credentials for every third-party integration
    (Kavenegar, shipping carriers) rather than real production
    credentials, so staging testing never has real-world side effects
    (no real SMS sent to real phone numbers, no real payment processed,
    no real shipment booked with a real carrier) — this is a genuinely
    important safety property of a staging environment, not a minor
    detail; document it clearly in a comment at the top of the compose
    file.
  - `FRONTEND_URL`/`ALLOWED_HOSTS`/CORS settings pointed at whatever
    staging subdomain this project will actually use (e.g.
    `staging.chiz.com`) — document as a placeholder requiring
    real configuration, not a working default.
- Add a `.env.staging.example` template file (mirroring whatever
  `.env.example`-equivalent convention this project established, if
  any, per Epic 6 Task 6.2.1.6's note about potentially creating one) —
  documenting every required staging env var with clear placeholder
  values and comments, especially calling out which ones MUST be
  sandbox/test credentials, not real ones.

ACCEPTANCE CRITERIA / TESTS
- `docker compose -f docker-compose.staging.yml config` validates the
  file's syntax correctness with no errors.
- Manually verify (with a real, populated `.env.staging` file, not
  committed): `docker compose -f docker-compose.staging.yml up --build`
  successfully starts every service, and the resulting stack is
  reachable and functions correctly end-to-end, running against
  entirely separate staging data/volumes from any existing dev/prod
  stack that might be running on the same machine simultaneously (a
  real, concrete test — run staging alongside a dev stack and confirm
  zero interference/collision between the two).
```

---

#### Task 25.1.1.2 — CI: run migrations check (`makemigrations --check`)

```
You are working in .github/workflows/ci.yml. Assume Epics 1–24 are
fully merged. This is a small, quick task — the SMALLEST in this epic
— but a genuinely valuable, cheap guard given how many migrations this
project has accumulated across 24 epics.

CONTEXT
Nothing in CI currently catches the case where a developer changes a
model (adds/removes/alters a field) but forgets to run
`makemigrations` — the change would pass tests locally (SQLite/Postgres
test databases are typically created fresh from CURRENT model state
via `migrate`, not necessarily requiring the corresponding migration
FILE to exist, depending on exact test-runner configuration) but break
`migrate` in any REAL environment (staging/production) applying
migrations sequentially against an EXISTING database, where the
missing migration file would leave the database schema silently
out of sync with the actual Django models.

TASK
Add a CI step failing the build if any model change lacks a
corresponding migration file.

REQUIREMENTS
- Add to the EXISTING `backend` job in `.github/workflows/ci.yml`
  (not a new job — this is a fast, cheap check that belongs alongside
  the existing pytest run, sharing that job's already-configured
  Postgres/Redis services and Python environment setup, rather than
  duplicating that setup cost in a separate job):
  ```yaml
  - name: Check for missing migrations
    run: python manage.py makemigrations --check --dry-run
  ```
  Positioned BEFORE the `Run pytest` step (fail fast on this cheap
  check before spending time on the full, slower test suite) — check
  the existing job's step ordering and insert appropriately.
  `--check` makes the command exit non-zero if any migrations WOULD be
  created (i.e. model state and migration files are out of sync);
  `--dry-run` ensures it doesn't actually WRITE any migration files
  during this check (which would be a strange, unwanted side effect of
  a CI check step) — both flags together are the standard, correct
  combination for this exact purpose.
- Add a clear, actionable failure message via a follow-up step (or a
  comment in the workflow file) so a developer hitting this failure
  immediately understands the fix: run `python manage.py makemigrations`
  locally, commit the generated migration file(s), and push again.

ACCEPTANCE CRITERIA / TESTS
- Manually verify: intentionally modify a model (e.g. add a throwaway
  field) WITHOUT running `makemigrations`, push to a test branch, and
  confirm the new CI step correctly fails with a clear message; then
  run `makemigrations`, commit the generated file, push again, and
  confirm the step now passes.
- Revert the throwaway test change before considering this task done —
  this verification step should not leave any actual schema change
  behind.
```

---

#### Task 25.1.1.3 — CI: deploy-to-staging job on `develop` branch

```
You are working in .github/workflows/ci.yml. Assume Task 25.1.1.1
(docker-compose.staging.yml) is already merged. RE-READ THIS
DOCUMENT'S HEADER BEFORE STARTING — this task is addressing a REAL,
confirmed gap: the workflow file's own header comment describes a
"deploy" job that does not actually exist anywhere in the file. This
task builds the FIRST real deployment automation this project has ever
had.

CONTEXT
`ci.yml` currently triggers on pushes to BOTH `main` and `develop`
(confirmed from the file's `on:` block), but does nothing deployment-
related for either branch — every merge, regardless of branch, only
runs tests and validates Docker builds. Given the header comment's
description implies a PRODUCTION deploy job (SSH-based, main-branch-
only) was intended to exist, and this epic's task is specifically
scoped to STAGING deploy on `develop` — implementing BOTH is a
reasonable, connected piece of work, since they share nearly identical
mechanics and leaving one implemented without the other would be an
odd, incomplete outcome given how closely related they are.

TASK
Add a real `deploy-staging` job triggered on pushes to `develop`
(after tests pass), and — given the confirmed gap in the file's own
documented intent — also add the `deploy-production` job the header
comment already describes but which was never actually built,
triggered on `main`. Update the header comment to accurately reflect
whatever's ACTUALLY in the file once this task is done.

REQUIREMENTS
- Add `deploy-staging`, depending on `backend`/`frontend`/`docker`
  jobs succeeding first, and gated to only run on the `develop` branch:
  ```yaml
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [backend, frontend, docker]
    if: github.ref == 'refs/heads/develop' && github.event_name == 'push'
    environment: staging  # GitHub Environments — enables environment-specific secrets and optional manual-approval gates
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_SSH_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /opt/chiz
            git pull origin develop
            docker compose -f docker-compose.staging.yml pull
            docker compose -f docker-compose.staging.yml up -d --build
            docker compose -f docker-compose.staging.yml exec -T backend python manage.py migrate
  ```
  Using GitHub's `environment: staging` feature (rather than a plain
  job with no environment binding) is a deliberate, valuable addition —
  it lets this project's repo settings configure environment-SCOPED
  secrets (staging SSH credentials distinct from production ones,
  preventing a staging-deploy credential leak from also compromising
  production access) and OPTIONALLY require manual approval before a
  staging deploy proceeds, if that's ever desired — even if no approval
  gate is configured initially, establishing the environment binding
  now costs nothing and enables that option cleanly later.
- Add `deploy-production`, mirroring the same structure but gated to
  `main` and using `docker-compose.prod.yml`/production secrets, with
  `environment: production` — and, given this is a MORE consequential
  deploy target than staging, seriously consider WHETHER a manual
  approval gate should be configured (via GitHub's environment
  protection rules, configured in repo settings, not purely in this
  YAML file) — RECOMMEND requiring at least one manual approval for
  production deploys, given this project handles real customer
  payment/order data once actually live; document this recommendation
  clearly even if you can't directly configure the repo-settings side
  of it from within this YAML file alone.
- Fix the workflow file's header comment to accurately describe the
  ACTUAL current job list (now five: `backend`, `frontend`, `docker`,
  `deploy-staging`, `deploy-production`) — closing the exact
  documentation-vs-reality gap that motivated this task in the first
  place, so a future reader never again encounters a comment describing
  functionality that doesn't actually exist.
- Note the deploy commands assume an EXISTING checkout at
  `/opt/chiz` on the target server that gets `git pull`ed — this
  mirrors what the header comment's ORIGINAL description ("SSH into
  server, pull, restart prod stack") already implied was the intended
  mechanism; if this project's actual server provisioning/directory
  layout differs, adjust accordingly, but this is a reasonable,
  standard baseline for a simple SSH-based deploy approach (as opposed
  to a more sophisticated container-registry-push-then-pull approach,
  which Task 25.1.1.4's blue-green/rolling-restart work may push this
  toward evolving into — but that's explicitly that task's concern,
  not this one's; keep this task's scope to "get SOME real, working
  automated deploy in place," not the more sophisticated zero-downtime
  mechanism yet).

ACCEPTANCE CRITERIA / TESTS
- Manually verify (requires real staging server access/secrets
  configured in the repo, which is inherently outside what's directly
  testable from within the YAML file alone): pushing to `develop`
  successfully triggers `deploy-staging`, which correctly connects via
  SSH, pulls the latest code, rebuilds/restarts the staging stack, and
  runs migrations — confirm by checking the staging environment
  reflects the deployed change afterward.
- Confirm `deploy-staging`/`deploy-production` correctly do NOT run on
  pull_request events (only on actual pushes to their respective
  branches) — a deploy job accidentally triggering on every PR against
  `develop`/`main` would be a real, undesirable behavior; verify the
  `if:` condition correctly excludes this.
- Confirm the header comment now accurately lists all five real jobs.
```

---

#### Task 25.1.1.4 — Blue-green or rolling restart strategy for backend container

```
You are working in docker-compose.prod.yml, docker-compose.staging.yml,
and the deploy job(s) from Task 25.1.1.3. Assume that task is already
merged.

CONTEXT
Task 25.1.1.3's deploy mechanism (`docker compose up -d --build`) has a
real downtime characteristic worth being explicit about: Docker
Compose's default behavior when rebuilding/restarting a service is to
STOP the old container before STARTING the new one — meaning every
deploy currently produces a real, if brief, window where the backend
is completely unavailable (requests failing/timing out) between the
old container stopping and the new one becoming ready to serve
traffic. For a real e-commerce platform processing live orders/
payments, even a brief unavailability window during every deploy is a
real, avoidable cost.

TASK
Implement a rolling-restart or blue-green deployment strategy for the
backend service, eliminating (or substantially minimizing) deploy-time
downtime.

REQUIREMENTS
- Evaluate the two realistic approaches given this project's actual
  infrastructure (a single-host Docker Compose setup, confirmed
  throughout this project's grounding — NOT a Kubernetes/orchestrated
  multi-host setup, which would have different, often simpler native
  tooling for this exact problem):
  1. **Rolling restart via multiple backend replicas + Docker Compose
     scaling**: run 2+ replicas of the `backend` service
     simultaneously, restart them ONE AT A TIME (never taking the
     LAST healthy replica down until its replacement is confirmed
     ready), with nginx/a reverse proxy load-balancing across whichever
     replicas are currently healthy. This requires: (a) nginx
     configuration supporting multiple upstream backend addresses
     (Docker Compose's internal DNS resolves a scaled service name to
     ALL running replica IPs, which nginx can round-robin across if
     configured with the service name as an upstream), and (b) a
     deploy script that restarts replicas sequentially with health
     checks between each, rather than Compose's default all-at-once
     restart behavior.
  2. **Blue-green via two full parallel stacks**: maintain TWO
     complete backend deployments ("blue" and "green"), deploy the NEW
     version to whichever is currently INACTIVE, verify it's healthy,
     then atomically switch the reverse proxy's upstream target from
     the old to the new stack, then tear down the now-inactive old one.
     This gives a cleaner, more absolute cutover (and easy instant
     ROLLBACK — just switch the proxy back) at the cost of needing
     roughly double the resources during the transition window.
  RECOMMEND approach 1 (rolling restart with 2 replicas) as the more
  PROPORTIONATE choice for this project's actual scale (per the
  original architecture review's own production-readiness discussion,
  this platform is not yet operating at a scale where blue-green's
  extra complexity/resource cost is clearly justified) — but implement
  whichever you choose FULLY and correctly, rather than a half-measure
  that doesn't actually eliminate the downtime window it's meant to
  address.
- For the rolling-restart approach: update `docker-compose.prod.yml`
  (and staging) so `backend` can run multiple replicas — Docker Compose
  V2's `deploy.replicas` key works for THIS purpose even outside
  Swarm mode for basic replica-count management (verify current Compose
  version behavior/support for this, since exact replica-management
  capabilities outside Swarm mode have evolved across Compose versions)
  — OR, if that proves unreliable outside Swarm, achieve the same
  effect via `docker compose up -d --scale backend=2 --no-recreate`
  combined with a custom deploy script explicitly restarting one
  container at a time.
  Update the nginx configuration (check this project's actual nginx
  config file, confirmed to exist per the `frontend` service's
  described role in `docker-compose.prod.yml`) to proxy to the
  `backend` service by its Compose DNS name (which resolves to ALL
  healthy replica IPs when scaled), rather than a single fixed
  container reference.
  Write a deploy script (e.g. `scripts/rolling-deploy.sh`) that: builds
  the new image, starts ONE additional new-version replica alongside
  the existing old-version ones, waits for it to pass a health check
  (confirm Epic 22 Task 22.1.1.4's `/api/health/` endpoint, or an
  equivalent, is suitable for this), THEN stops one old-version
  replica, repeats until all replicas are running the new version —
  at every point during this process, AT LEAST ONE healthy replica
  (old or new version) is serving traffic, achieving genuinely
  zero-downtime deployment.
  Update Task 25.1.1.3's deploy job(s) to invoke this script instead of
  the simpler `docker compose up -d --build`.
- Document the chosen strategy, its actual mechanics, and its
  operational tradeoffs (e.g. a brief window during rolling restart
  where OLD and NEW code versions are BOTH simultaneously serving live
  traffic — confirm this is safe given this project's actual deploy
  patterns; a database-migration-heavy deploy where the new code
  expects a schema change the old code doesn't understand could be
  genuinely UNSAFE to roll out this way if migrations aren't
  backward-compatible with the previous code version for the brief
  overlap window — this is a real, non-trivial operational
  consideration worth explicitly flagging in the documentation rather
  than silently assuming every future deploy will always be safely
  backward-compatible).

ACCEPTANCE CRITERIA / TESTS
- Manually verify against a real staging deployment: while continuously
  polling the application (e.g. a simple loop hitting the health-check
  or homepage endpoint every 100ms) DURING a deploy triggered via the
  new rolling-deploy mechanism, confirm ZERO failed requests occur
  throughout the entire deploy process — the actual, concrete proof
  this task achieves its stated goal, not just that the scripts run
  without error.
- Confirm a deploy correctly results in ALL replicas eventually running
  the NEW version (not silently leaving a stale old-version replica
  running indefinitely if the script has a bug in its replica-count
  bookkeeping).
```

---

#### Task 25.1.1.5 — Automated DB backup job (scheduled `pg_dump` to storage)

```
You are working in docker-compose.prod.yml (and staging), a new
scripts/backup-db.sh, and — since Epic 22's Celery infrastructure now
exists — potentially a Celery Beat periodic task as the scheduling
mechanism. Assume Epics 1–24 are fully merged.

CONTEXT
Confirmed from `docker-compose.prod.yml`'s grounding: the Postgres data
lives in a named Docker volume (`postgres_data_prod`) with NO backup
mechanism of any kind configured — if that volume is ever lost
(host disk failure, accidental `docker volume rm`, a botched migration)
every order, customer account, payment record, and everything else
this entire 25-epic backlog has built is gone, unrecoverably, with
zero recovery path. This is one of the most consequential gaps
remaining in the whole project at this point.

TASK
Add an automated, scheduled database backup mechanism, storing dumps
somewhere genuinely separate from the primary database's own storage
(a backup living on the SAME volume/disk as the data it's backing up
provides no real protection against the most common failure modes).

REQUIREMENTS
- Decide the STORAGE DESTINATION deliberately — a local backup (even
  to a different disk/path on the same host) protects against database-
  corruption/accidental-deletion scenarios but NOT against whole-host
  failure; genuine disaster-recovery-grade protection needs an
  OFF-HOST destination (cloud object storage — S3-compatible storage is
  the most common, widely-supported choice). RECOMMEND: implement
  support for BOTH — a local backup directory (simple, fast, good
  first line of defense) AND upload to S3-compatible object storage
  (the genuine disaster-recovery layer) — since this project has
  confirmed NO cloud storage integration anywhere yet (Epic 20's
  media-storage grounding confirmed local-filesystem-only storage),
  this task is ALSO the first point cloud storage credentials/tooling
  enter this project; scope this narrowly to backup storage specifically
  (don't scope-creep into migrating MEDIA file storage to S3 too, which
  is a separate, larger concern outside this task).
- Write `scripts/backup-db.sh`:
  ```bash
  #!/bin/bash
  set -euo pipefail

  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
  BACKUP_DIR="/backups"
  BACKUP_FILE="${BACKUP_DIR}/chiz_${TIMESTAMP}.sql.gz"

  mkdir -p "${BACKUP_DIR}"

  pg_dump \
    -h "${DATABASE_HOST}" -U "${DATABASE_USER}" -d "${DATABASE_NAME}" \
    --no-owner --no-privileges \
    | gzip > "${BACKUP_FILE}"

  echo "Backup created: ${BACKUP_FILE} ($(du -h "${BACKUP_FILE}" | cut -f1))"

  if [ -n "${S3_BACKUP_BUCKET:-}" ]; then
    aws s3 cp "${BACKUP_FILE}" "s3://${S3_BACKUP_BUCKET}/db-backups/$(basename "${BACKUP_FILE}")"
    echo "Uploaded to s3://${S3_BACKUP_BUCKET}/db-backups/"
  fi

  # Retention: delete local backups older than 7 days (S3 lifecycle
  # rules, configured separately in the bucket itself, handle
  # longer-term retention/expiry for the off-host copies — don't try
  # to manage S3 retention from this script too, that's what S3
  # lifecycle policies are specifically designed for).
  find "${BACKUP_DIR}" -name "chiz_*.sql.gz" -mtime +7 -delete
  ```
  Using `PGPASSWORD` via environment (already how this project's
  `DATABASE_PASSWORD` env var likely gets passed to `pg_dump` given the
  existing connection-config pattern — confirm and wire consistently)
  rather than a `.pgpass` file or inline password, matching whatever
  this project's existing DB-credential-handling convention already
  is.
  `--no-owner --no-privileges` produces a more PORTABLE dump (not tied
  to the exact role/ownership setup of the source database), which
  matters for Task 25.1.1.6's restore-drill testing against a
  DIFFERENT environment than where the backup was taken.
- Add the AWS CLI (or whichever S3-compatible client, if using a
  non-AWS provider) to the backend Docker image, OR run this script
  from a DEDICATED lightweight backup container/sidecar rather than
  bloating the main application image with backup-specific tooling it
  doesn't otherwise need — RECOMMEND a separate, minimal `backup`
  service in `docker-compose.prod.yml` specifically for this, keeping
  the main `backend` image focused on what it actually needs to run
  the application:
  ```yaml
  backup:
    image: postgres:15  # includes pg_dump; add awscli via a small custom Dockerfile if S3 upload is needed
    container_name: backend_backup_prod
    restart: unless-stopped
    volumes:
      - ./scripts/backup-db.sh:/backup-db.sh:ro
      - backup_storage_prod:/backups
    env_file:
      - .env.prod
    entrypoint: ["/bin/sh", "-c", "while true; do sleep 86400; done"]  # idle container; actual backup triggered via `docker compose exec` on a schedule, see below
  ```
- Schedule the actual backup EXECUTION — given Epic 22's Celery
  infrastructure now exists, consider whether a Celery Beat periodic
  task (calling this script via a subprocess, or reimplementing the
  same `pg_dump`/S3-upload logic as a proper Celery task in Python
  rather than shelling out to a bash script) is a CLEANER mechanism
  than a host-level cron job — RECOMMEND the Celery Beat approach for
  consistency with how EVERY OTHER scheduled operation in this project
  (Epic 3's expiry sweep, Epic 6's payment reconciliation, Epic 7's
  shipment polling) is already managed, rather than introducing a
  SEPARATE, parallel scheduling mechanism (host cron) that ops staff
  would need to remember exists and manage independently from the
  `django_celery_beat`-managed schedule everything else uses. Implement
  as a proper Celery task:
  ```python
  # core/tasks.py, alongside Epic 22's celery_health_check
  @shared_task
  def backup_database():
      import subprocess
      result = subprocess.run(["/backup-db.sh"], capture_output=True, text=True, timeout=600)
      if result.returncode != 0:
          raise RuntimeError(f"Database backup failed: {result.stderr}")
      return result.stdout
  ```
  Add a seeded `PeriodicTask` entry (mirroring Epic 22 Task 22.1.1.2's
  exact seeding-migration pattern) scheduling this DAILY, at a low-
  traffic hour (e.g. 3 AM Iran time, per Epic 14's `TIME_ZONE`
  configuration already correctly propagating to `CELERY_TIMEZONE`
  per Epic 22 Task 22.1.1.2).
- Add BACKUP FAILURE ALERTING — a backup job that silently fails and
  nobody notices until the moment a real restore is needed is barely
  better than having no backup at all. Wire `backup_database()`'s
  failure path into whatever alerting/monitoring this project has by
  this point (Epic 26's future monitoring work, if landed — if not
  yet, at minimum ensure the task's exception is genuinely visible via
  Celery's own error logging/whatever error-tracking exists, and flag
  this as something Epic 26 should specifically also wire into real
  alerting once that epic lands).

ACCEPTANCE CRITERIA / TESTS
- Add a test for `backup_database()` (mocking the subprocess call)
  confirming it correctly raises on a non-zero exit code from the
  backup script, and correctly succeeds/returns output on success.
- Manually verify against a real staging environment: trigger the
  backup task manually, confirm a real `.sql.gz` file is produced
  locally, confirm (if S3 credentials are configured) it's correctly
  uploaded, and confirm the retention `find ... -delete` logic
  correctly removes only files older than 7 days (construct a test
  file with an old modification time and confirm it's removed, while a
  recent one is preserved).
```

---

#### Task 25.1.1.6 — Backup restore drill documentation + test

```
You are working in scripts/restore-db.sh (new file) and a new
docs/runbooks/backup-restore-drill.md. Assume Task 25.1.1.5 is already
merged.

CONTEXT
Task 25.1.1.5 produces backups — but an UNTESTED backup provides false
confidence, not real disaster-recovery capability; the single most
common way organizations discover their backups don't actually work is
during a REAL emergency, which is exactly the worst possible time to
find out. This task proves the backups Task 25.1.1.5 produces are
GENUINELY restorable, and documents the procedure clearly enough that
someone under real incident-response pressure (not calmly reading
documentation at leisure) can follow it correctly.

TASK
Write a restore script, and actually PERFORM a real restore drill
against a genuine backup file, documenting the full procedure as a
runbook.

REQUIREMENTS
- Write `scripts/restore-db.sh`:
  ```bash
  #!/bin/bash
  set -euo pipefail

  if [ -z "${1:-}" ]; then
    echo "Usage: $0 <backup_file.sql.gz>"
    exit 1
  fi

  BACKUP_FILE="$1"
  RESTORE_DB_NAME="${RESTORE_DB_NAME:-chiz_restore_test}"

  echo "WARNING: this will DROP and recreate database '${RESTORE_DB_NAME}'."
  read -p "Continue? (yes/no) " confirm
  if [ "${confirm}" != "yes" ]; then
    echo "Aborted."
    exit 1
  fi

  dropdb -h "${DATABASE_HOST}" -U "${DATABASE_USER}" --if-exists "${RESTORE_DB_NAME}"
  createdb -h "${DATABASE_HOST}" -U "${DATABASE_USER}" "${RESTORE_DB_NAME}"
  gunzip -c "${BACKUP_FILE}" | psql -h "${DATABASE_HOST}" -U "${DATABASE_USER}" -d "${RESTORE_DB_NAME}"

  echo "Restore complete into '${RESTORE_DB_NAME}'."
  ```
  Note the DEFAULT target database name is a DEDICATED, clearly-named
  `_restore_test` database, NOT the live application database — this
  is a deliberate, important safety default: an accidental invocation
  of this script should never be able to silently clobber a real,
  in-use database; a real PRODUCTION restore (as opposed to a drill/
  verification) would require EXPLICITLY overriding
  `RESTORE_DB_NAME` to the real database name, a conscious, deliberate
  action, not an accidental default.
  Include the interactive confirmation prompt for the SAME reason —
  this script is destructive by nature (it drops a database), and a
  moment of explicit confirmation is a reasonable, low-cost safety
  measure even during a real, time-pressured incident.
- ACTUALLY PERFORM a real restore drill: trigger Task 25.1.1.5's
  backup task against a real staging (never production) database with
  representative data, obtain the resulting backup file, run
  `restore-db.sh` against it targeting the default `_restore_test`
  database name, and VERIFY the restored data is genuinely complete
  and correct — spot-check specific known records (a specific order,
  a specific product, a specific user) exist in the restored database
  with the expected data, not just that the restore command exited
  successfully with no errors (a restore command can complete "
  successfully" while still producing an incomplete/corrupted result
  if something upstream in the backup itself was subtly wrong — the
  ACTUAL data verification is the real test, not just the command's
  exit code).
- Write `docs/runbooks/backup-restore-drill.md` documenting: (1) the
  full restore procedure step by step, written for someone under real
  incident-response pressure (clear, numbered, unambiguous steps — not
  a casual description), (2) how to identify the CORRECT backup file to
  restore from (e.g. how to determine the most recent good backup,
  where backups are stored/named, per Task 25.1.1.5's naming/storage
  conventions), (3) the DATA-VERIFICATION checklist to run after any
  real restore (the same spot-check approach used during this task's
  own drill), (4) a note on RECOVERY TIME — roughly how long a real
  restore actually took during this drill, giving whoever responds to
  a real incident a realistic expectation rather than an unknown
  unknown, and (5) — importantly — a note on RECOVERY POINT: since
  backups run DAILY (per Task 25.1.1.5's schedule), any real restore
  means accepting up to 24 hours of data loss (every order/account/
  change made since the last successful backup before the incident) —
  state this limitation explicitly and prominently, since it's a real,
  important operational fact whoever owns this project should be
  consciously aware of (and could motivate a future decision to
  increase backup FREQUENCY if 24 hours of potential data loss is
  judged unacceptable — flag this as a legitimate follow-up
  consideration, not something to silently decide either way within
  this task).
- Schedule this drill as a RECURRING exercise, not a one-time proof —
  add a calendar/process note (in the runbook itself, or wherever this
  project tracks recurring operational tasks) recommending this drill
  be re-run periodically (e.g. quarterly) — a backup/restore mechanism
  that worked once, months ago, before numerous subsequent schema
  migrations and infrastructure changes, is not the same as a
  CURRENTLY-verified-working one; ongoing verification matters, not
  just this one initial proof.

ACCEPTANCE CRITERIA / TESTS
- The actual, real restore drill described above must be genuinely
  PERFORMED (not just scripted and left untested) as part of completing
  this task — a documented procedure that's never actually been
  executed end-to-end is not an acceptable substitute for a real,
  verified drill, given this task's entire purpose is proving the
  backup/restore mechanism genuinely works.
- The completed runbook document, reflecting the ACTUAL results of the
  real drill performed (real timing numbers, real verification
  checklist derived from what was actually checked), is this task's
  primary deliverable alongside the restore script itself.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 25.1.1.1 | Add docker-compose.staging.yml | ☐ |
| 25.1.1.2 | CI: migrations check (makemigrations --check) | ☐ |
| 25.1.1.3 | CI: deploy-to-staging job (+ fix missing deploy job entirely) | ☐ |
| 25.1.1.4 | Blue-green/rolling restart strategy | ☐ |
| 25.1.1.5 | Automated DB backup job | ☐ |
| 25.1.1.6 | Backup restore drill documentation + test | ☐ |

Once Epic 25 is fully merged, the next epic to generate prompts for is
**Epic 26 — Monitoring & Logging**, which the master backlog flags
(alongside this epic) as needing to finish before any real production
launch — and which Task 25.1.1.5 above already explicitly flagged a
dependency on, for real backup-failure alerting once that epic's
error-tracking/alerting infrastructure exists.
