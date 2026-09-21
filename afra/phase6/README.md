# Phase 6 — آمادگی دیپلوی ایرانی (حذف Render/Vercel/Cloudinary)

## Implementation Prompts

### Prompt 1 — حذف کد Render-Specific + رفع یافتهٔ حیاتی Celery Eager

```
Goal: Remove the Render-specific ALLOWED_HOSTS/CSRF_TRUSTED_ORIGINS
auto-detection block from production.py, and flip
CELERY_TASK_ALWAYS_EAGER's production default from True to False —
documenting explicitly why this default existed (Render free tier
couldn't run a persistent worker) and why a real VPS deployment
requires the opposite default (a real, always-on worker + beat will
finally exist).

Before starting, read config/settings/production.py in full,
especially the "Render platform integration" comment block and the
"Celery — free-tier fallback" comment block at the bottom — both
explain exactly why the current code exists, which this prompt must
correctly supersede, not just delete blindly.

What to build:

1. Remove the entire "Render platform integration" section: the
   RENDER_EXTERNAL_HOSTNAME config() call and the
   `if RENDER_EXTERNAL_HOSTNAME: ALLOWED_HOSTS.append(...)` block.
   ALLOWED_HOSTS and CSRF_TRUSTED_ORIGINS should now be populated
   entirely from the existing base.py mechanism (already
   provider-agnostic, reads from the ALLOWED_HOSTS/CSRF_TRUSTED_ORIGINS
   env vars directly) — no replacement auto-detection code is needed,
   since a VPS deployment's hostname is known and set explicitly via
   env vars, unlike Render's dynamically-assigned *.onrender.com
   hostname this code existed to handle.

2. Replace the entire "Celery — free-tier fallback" section:
   CELERY_TASK_ALWAYS_EAGER: bool = config(
       "CELERY_TASK_ALWAYS_EAGER", default=False, cast=bool
   )
   CELERY_TASK_EAGER_PROPAGATES: bool = False
   Rewrite the comment above it to explain the NEW reasoning:
   "Defaults to False in production — unlike the platform this project
   was previously deployed on (Render's free tier, which could not run
   a persistent Celery worker/beat service, forcing tasks to run
   eagerly/synchronously as a workaround), the VPS deployment this
   project now targets runs real, always-on celery_worker and
   celery_beat containers (see the root docker-compose.prod.yml). This
   is the first production environment where Celery actually operates
   as designed: every .delay() call is genuinely queued to Redis and
   processed asynchronously by a real worker, and every periodic sweep
   task registered in config/celery.py's beat_schedule (payment
   timeout reaping, group session capacity checks, session reminders,
   meeting room creation retries, and every other scheduled task
   across this project's entire feature set) will actually run on
   schedule for the first time. CELERY_TASK_EAGER_PROPAGATES remains
   False regardless, matching this file's pre-existing reasoning
   (documented below) for why a task failure should never surface as a
   500 to the end user." Keep the existing explanation of
   CELERY_TASK_EAGER_PROPAGATES's own reasoning (about a transient SMTP
   hiccup not becoming a request-cycle 500) verbatim below the new
   comment, since that reasoning is still fully valid regardless of
   the EAGER default.

Files affected:
- config/settings/production.py (only these two sections)

Then check whether any existing test relies on the old
RENDER_EXTERNAL_HOSTNAME behavior or the old EAGER=True default (grep
-rn "RENDER_EXTERNAL_HOSTNAME\|CELERY_TASK_ALWAYS_EAGER"
tests/ config/) and update/remove any such test to match the new
behavior — do not leave a test asserting the old Render-specific logic
still exists.

Acceptance Criteria:
- grep -rn "RENDER_EXTERNAL_HOSTNAME" apps/ config/ tests/ returns no
  results anywhere
- pytest -x (full backend suite) passes — confirm this doesn't break
  under test settings, which likely already override
  CELERY_TASK_ALWAYS_EAGER independently (check config/settings/test.py
  to confirm test settings still force EAGER=True for test isolation,
  regardless of production's new default — these are separate settings
  modules and should NOT be conflated)
- python manage.py check --settings=config.settings.production
  (with appropriate env vars stubbed) reports no errors

Verification Steps:
1. grep -rn "RENDER_EXTERNAL_HOSTNAME" apps/ config/ tests/
2. pytest -x -v
3. grep -n "CELERY_TASK_ALWAYS_EAGER" config/settings/test.py
   (confirm test settings are unaffected by production's default
   change)
4. git diff --stat
```

### Prompt 2 — جایگزینی Cloudinary با `FileSystemStorage` محلی

```
Goal: Remove Cloudinary from production media storage, replacing it
with local FileSystemStorage backed by the existing media_data Docker
volume (already provisioned in the current backend
docker-compose.prod.yml), and remove the now-unused Cloudinary
packages from requirements.

Before starting, read:
1. config/settings/production.py — the full "Media storage
   (Cloudinary)" section, including its detailed comment explaining
   WHY Cloudinary was chosen (Render's ephemeral disk) — this reasoning
   no longer applies once media lives on a real Docker volume, which
   persists across container restarts/redeploys exactly the way
   Cloudinary's persistence was needed for
2. config/settings/base.py — confirm the exact current MEDIA_URL/
   MEDIA_ROOT values and whether base.py already implicitly defaults
   to FileSystemStorage (Django's own default) when STORAGES isn't
   overridden — if so, production.py may not need to explicitly set
   STORAGES["default"] at all, just remove the override entirely
3. requirements/production.txt — confirm the exact two Cloudinary
   package lines to remove
4. apps/common/views.py — MediaServeView's full docstring and
   implementation (read in full before deciding whether to remove it
   in this prompt or a later one — this prompt's scope is storage
   backend only; MediaServeView's removal is more naturally scoped to
   the nginx same-origin work in a later prompt of this Phase, since
   removing it only makes sense once nginx actually serves /media/
   directly — for THIS prompt, leave MediaServeView untouched, just
   note this deferral)

What to build:

1. In config/settings/production.py, remove the entire "Media storage
   (Cloudinary)" section: the INSTALLED_APPS extension
   (cloudinary_storage, cloudinary), the CLOUDINARY_STORAGE dict, and
   the STORAGES dict entirely IF base.py's own default is plain
   FileSystemStorage for "default" (confirm this first per your
   reading above) — if STORAGES["staticfiles"] (WhiteNoise) still
   needs to be set here (it does, since that's unrelated to Cloudinary
   and still needed in production), keep that key, just remove the
   "default" key so it falls through to Django's built-in default
   (django.core.files.storage.FileSystemStorage), OR set it explicitly
   for clarity:
   STORAGES: dict[str, dict[str, str]] = {
       "default": {
           "BACKEND": "django.core.files.storage.FileSystemStorage",
       },
       "staticfiles": {
           "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
       },
   }
   Add a comment explaining the new reasoning: "Media now persists on
   the media_data Docker volume mounted into this container (see the
   root docker-compose.prod.yml) — the exact same volume the original
   backend-only docker-compose.prod.yml already provisioned for this
   purpose. Previously this project used Cloudinary specifically
   because Render's disk was ephemeral (wiped on every redeploy); a
   VPS with a real, persistent Docker volume doesn't have that
   problem, so local storage is now both simpler and sufficient. If
   media volume grows large enough to need a CDN or the VPS's disk
   becomes a bottleneck, migrating to a regional S3-compatible object
   storage (e.g. via django-storages) is a natural future upgrade —
   deliberately not implemented now, to avoid a dependency this
   project's current scale doesn't need."

2. In requirements/production.txt, remove the lines:
   cloudinary==1.44.2
   django-cloudinary-storage==0.3.0

Files affected:
- config/settings/production.py
- requirements/production.txt

Then check whether any existing test references CLOUDINARY_* settings
or cloudinary_storage/cloudinary in INSTALLED_APPS (grep -rn
"cloudinary\|CLOUDINARY" apps/ tests/ config/ | grep -v migrations) and
remove/update any such reference.

Acceptance Criteria:
- grep -rn "cloudinary\|CLOUDINARY" apps/ tests/ config/ requirements/
  returns no results anywhere
- pytest -x (full backend suite) passes
- python manage.py check --settings=config.settings.production
  (with appropriate env vars stubbed) reports no errors

Verification Steps:
1. grep -rn "cloudinary\|CLOUDINARY" apps/ tests/ config/ requirements/
2. pytest -x -v
3. git diff --stat
```

### Prompt 3 — ادغام `docker-compose.prod.yml` در یک فایل واحد ریشه

```
Goal: Merge the two existing standalone production Docker Compose
files (afra-backend/docker-compose.prod.yml and
afra-frontend/docker-compose.prod.yml) into one unified
docker-compose.prod.yml at the repository root, sharing one Docker
network, adding celery_worker and celery_beat as genuinely persistent
services (the first time this project will run them for real), and
applying the resource limits from this roadmap's earlier architecture
audit.

Before starting, read both existing files completely:
1. afra-backend/docker-compose.prod.yml — every service definition in
   full (db, redis, backend, celery_worker, celery_beat), their exact
   healthchecks, volumes, environment variables, and the extensive
   comments explaining each design choice
2. afra-frontend/docker-compose.prod.yml — the frontend service
   definition in full, its build args, and its comment explaining
   VITE_API_BASE_URL vs API_ORIGIN's build-time vs runtime distinction

What to build:

Create docker-compose.prod.yml at the repository root, merging both
files:

1. Keep every service from the backend file (db, redis, backend,
   celery_worker, celery_beat) essentially as-is, with these specific
   changes:
   - Remove the backend service's `ports: ["8000:8000"]` mapping
     entirely — it's no longer directly exposed to the host; only
     nginx (added below) is externally reachable. It stays reachable
     to nginx via Docker's internal DNS (service name "backend") on
     the shared network.
   - Rename the shared network from afra_backend_net to something
     neutral like afra_net (since it's no longer backend-only), and
     use this same network name for every service in this merged file,
     including the frontend/nginx service.
   - Add explicit resource limits to each service, matching this
     roadmap's earlier architecture-audit sizing table (adjust the
     exact syntax to this project's Docker Compose version — v2's
     `deploy.resources.limits` requires `docker compose` in swarm mode
     to be enforced by default with plain `docker compose up`; if this
     project's Compose version doesn't enforce deploy.resources
     without swarm, use the simpler `mem_limit`/`cpus` top-level keys
     instead, which plain `docker compose up` does respect — check
     which mechanism actually takes effect for a non-swarm
     `docker compose up -d` deployment and use that one, not the one
     that silently does nothing):
     - backend (gunicorn): mem_limit 1.5g, cpus 1.0
     - celery_worker: mem_limit 512m
     - celery_beat: mem_limit 128m
     - db (postgres): mem_limit 2g, cpus 1.0
     - redis: mem_limit 256m
     - frontend/nginx (added below): mem_limit 128m

2. Add the frontend service, adapted from
   afra-frontend/docker-compose.prod.yml:
   - Same build context/args (context: ./afra-frontend, dockerfile:
     Dockerfile, the same three build args)
   - Change VITE_API_BASE_URL's default to a relative path, e.g. "/api"
     (since this Phase's architecture makes frontend/backend
     same-origin — no more cross-origin absolute URL needed; this is a
     build-time arg, so document that changing it requires a rebuild,
     matching the original file's own existing documented caveat)
   - Change the published port mapping from "8080:80" to "80:80" (and
     "443:443" if TLS termination happens in this same nginx container
     — decide based on whether this project wants Let's Encrypt/
     certbot integration in this same container or expects TLS to
     terminate at a layer outside Docker entirely, e.g. a
     provider-level load balancer; for a self-managed VPS, TLS
     termination INSIDE this compose stack is the more realistic,
     self-contained choice — note this decision explicitly and, if TLS
     is handled inside this stack, add a certbot/companion container OR
     document that this is deliberately deferred to a follow-up
     infrastructure task outside this Phase's code-only scope, since
     TLS certificate provisioning requires a real domain name that
     doesn't exist in this development environment — DO NOT attempt to
     fabricate certificate files or guess a domain)
   - Add depends_on: backend (so nginx doesn't start proxying before
     the backend container exists, even though Docker Compose's
     depends_on only waits for the container to start, not for
     Django's own readiness — the backend service's existing
     healthcheck plus nginx's own retry-on-connection-refused behavior
     is sufficient; don't over-engineer a startup-ordering solution
     beyond what this project's existing services already use
     elsewhere)
   - Mount the media_data volume (read-only) into the frontend/nginx
     container too, e.g. at /usr/share/nginx/media, so nginx's new
     /media/ location block (built in the next prompt) can serve files
     directly from the same volume the backend writes to

3. Keep all volumes (postgres_data, redis_data, media_data) defined
   once at the bottom, shared correctly across services that need them.

4. Delete afra-backend/docker-compose.prod.yml and
   afra-frontend/docker-compose.prod.yml once the merged root file is
   confirmed working (do this as the last step, after Prompt 4's nginx
   changes and a successful local `docker compose -f
   docker-compose.prod.yml up --build` test — don't delete the
   originals until the replacement is proven).

Files affected:
- docker-compose.prod.yml (new, root)
- afra-backend/docker-compose.prod.yml,
  afra-frontend/docker-compose.prod.yml (deleted, at the end of this
  prompt only after verification)

Acceptance Criteria:
- docker compose -f docker-compose.prod.yml config (validates syntax
  without starting anything) succeeds with no errors
- Every service from both original files is present in the merged file
  with no functional regression in its own configuration (healthchecks,
  env vars, volumes all preserved)

Verification Steps:
1. docker compose -f docker-compose.prod.yml config
2. (If Docker is available in this environment) docker compose -f
   docker-compose.prod.yml up --build -d, then docker compose -f
   docker-compose.prod.yml ps to confirm every service reaches a
   healthy/running state, then docker compose -f docker-compose.prod.yml
   down
3. git diff --stat
```

### Prompt 4 — nginx: Reverse-Proxy هم‌مبدأ برای `/api/` و `/media/`

```
Goal: Update the frontend's nginx config template to reverse-proxy
/api/ to the backend service and serve /media/ directly from the
shared volume, switching this project's frontend-backend
communication from cross-origin to same-origin, and update the CSP
accordingly.

Before starting, read afra-frontend/nginx/nginx.conf.template in full,
especially its top comment block explicitly documenting the CURRENT
cross-origin architecture — this prompt directly supersedes that
documented design, so the comment must be rewritten, not left stale
and contradicting the new config below it.

What to build:

1. Rewrite the top comment block to describe the new architecture:
   frontend and backend now run on the SAME server/domain, with this
   nginx container reverse-proxying /api/ to the backend container by
   its Docker Compose service name ("backend", port 8000) over the
   shared afra_net network (from Prompt 3). CORS is no longer needed
   for the browser-to-API path (same-origin requests don't trigger
   CORS at all) — CORS_ALLOWED_ORIGINS/CSRF_TRUSTED_ORIGINS on the
   Django side can be simplified accordingly (note this, but the
   actual Django-side settings simplification is out of this prompt's
   scope — nginx config only here).

2. Add a new location block for the API, placed before the catch-all
   `location /` block (nginx matches more specific locations first
   regardless of order, but place it logically near the top for
   readability):
   location /api/ {
       proxy_pass http://backend:8000/api/;
       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
       proxy_connect_timeout 10s;
       proxy_read_timeout 60s;
       client_max_body_size 10M;
   }
   (proxy_read_timeout 60s is deliberately generous — some admin
   actions, e.g. an article cover_image upload or a Skyroom API call
   during session creation, may take longer than a typical API
   response; don't set this so low that a legitimate slow request gets
   cut off.)

3. Add a location block for media, serving directly from the mounted
   volume (per Prompt 3's read-only mount at /usr/share/nginx/media):
   location /media/ {
       alias /usr/share/nginx/media/;
       expires 30d;
       add_header Cache-Control "public, max-age=2592000" always;
       add_header Cross-Origin-Resource-Policy "same-origin" always;
       access_log off;
       try_files $uri =404;
   }
   (Cross-Origin-Resource-Policy is now "same-origin", not the
   cross-origin-permissive value MediaServeView previously needed to
   set on the Django side — since this is genuinely same-origin now,
   the stricter value is both correct and more secure; this is
   precisely the ORB-related concern MediaServeView's own docstring
   described, now resolved architecturally rather than by a Django
   workaround view.)

4. Simplify the Content-Security-Policy's connect-src directive: since
   the API is now same-origin, remove the ${API_ORIGIN} substitution
   from connect-src (it can just be 'self') — but check whether
   ${API_ORIGIN} is used ANYWHERE else in this template (e.g. if
   Google Analytics collection domains also relied on that same
   variable's presence check, verify carefully) before removing it
   entirely; if API_ORIGIN was ONLY ever used for connect-src, this
   variable and its docker-entrypoint.sh substitution step become
   dead — note this finding explicitly, but leave the actual
   entrypoint.sh cleanup for Prompt 6, which handles env-var file
   updates broadly, to avoid scattering related cleanup across two
   prompts.

Files affected:
- afra-frontend/nginx/nginx.conf.template

Acceptance Criteria:
- nginx -t (or an equivalent config syntax check, if nginx is
  available in this environment; otherwise a careful manual review
  substitutes) reports no syntax errors on the rendered template
- The /api/ and /media/ location blocks are present with the exact
  proxy_pass/alias targets specified above

Verification Steps:
1. (If Docker/nginx is available) build the frontend image and run
   `docker run --rm <image> nginx -t` after entrypoint substitution, or
   manually substitute ${API_ORIGIN} and run `nginx -t` against the
   resulting file
2. Manual review of the diff for correctness
3. git diff --stat
```

### Prompt 5 — حذف `render.yaml`، `vercel.json` + بازنویسی `deploy.yml`

```
Goal: Remove the Render Blueprint and Vercel config files entirely,
and rewrite the GitHub Actions deploy workflow to deploy to a VPS over
SSH instead of Render/Vercel deploy hooks.

Before starting, read:
1. render.yaml (root) and afra-frontend/vercel.json in full, one more
   time, to confirm nothing in them documents information (like a
   resource sizing decision) that should be preserved elsewhere before
   deletion — if you find such information, copy it into this Phase's
   documentation (Prompt 6) before deleting the file it currently lives
   in
2. .github/workflows/deploy.yml — the full current file, as the
   structural starting point (its trigger condition — workflow_run
   after Backend CI / Frontend CI succeed on main — is still exactly
   correct and should be preserved; only the actual deployment steps
   change)
3. .github/workflows/backend-ci.yml, frontend-ci.yml — to confirm the
   exact workflow names deploy.yml's workflow_run trigger references
   ("Backend CI", "Frontend CI") remain accurate

What to build:

1. Delete render.yaml (repository root) and afra-frontend/vercel.json.

2. Rewrite .github/workflows/deploy.yml:
   - Keep the exact same `on: workflow_run:` trigger block, permissions
     block, and the two job names (deploy-backend, deploy-frontend) —
     or, since deployment is now a single unified Docker Compose stack
     rather than two independently-deployed platforms, consider
     whether this should collapse into ONE job (deploy) that builds
     and ships both images together, matching the new
     docker-compose.prod.yml architecture from Prompt 3, rather than
     keeping the old two-separate-platforms job split that no longer
     reflects reality. Prefer collapsing to one job — the old split
     existed specifically because Render and Vercel were two unrelated
     platforms; that reason no longer applies.
   - The new single job should:
     a. Checkout the exact commit that was tested (matching the
        existing pattern: `ref: ${{ github.event.workflow_run.head_sha }}`)
     b. SSH into the VPS (use an established action like
        appleboy/ssh-action, referencing new repository secrets:
        VPS_HOST, VPS_USER, VPS_SSH_PRIVATE_KEY — document these as
        new required secrets in a comment)
     c. On the VPS, pull the latest code (git pull, assuming the repo
        is cloned there) and run:
        docker compose -f docker-compose.prod.yml up -d --build --remove-orphans
     d. Run database migrations as part of the same SSH session (or as
        a separate step): docker compose -f docker-compose.prod.yml
        exec -T backend python manage.py migrate --noinput
     e. A basic post-deploy health check: curl the VPS's public
        /api/health/ endpoint and fail the job if it doesn't return
        200 within a reasonable retry window (a few retries with a
        short sleep, since the backend container may take a few
        seconds to become ready after restart)
   - Preserve the existing `concurrency:` group settings (adapted to
     the single new job name) to prevent overlapping deploys.
   - Add a comment explaining this is a simple, direct SSH-based
     deploy appropriate for a single-VPS setup — not a zero-downtime
     blue-green deployment (a brief window of unavailability during
     `docker compose up -d --build` is expected and acceptable at this
     project's current scale; note this as a known, accepted
     limitation, not an oversight, and suggest that a future
     zero-downtime setup would need a load balancer + multiple app
     instances, deliberately out of scope for this Phase).

Files affected:
- render.yaml (deleted)
- afra-frontend/vercel.json (deleted)
- .github/workflows/deploy.yml (rewritten)

Acceptance Criteria:
- The new deploy.yml is valid YAML (verify with a YAML parser/linter,
  e.g. `python -c "import yaml; yaml.safe_load(open('.github/workflows/deploy.yml'))"`)
- No file in the repository references render.yaml or vercel.json
  content anymore (grep -rln "render.yaml\|vercel.json" . | grep -v
  node_modules | grep -v .git — review and update any remaining
  reference found, e.g. in README.md or docs/)

Verification Steps:
1. python -c "import yaml; yaml.safe_load(open('.github/workflows/deploy.yml'))"
2. grep -rln "render.yaml\|vercel.json\|RENDER_BACKEND_DEPLOY_HOOK\|VERCEL_TOKEN" . --include=*.md --include=*.yml --include=*.py | grep -v node_modules
3. git diff --stat
```

### Prompt 6 — به‌روزرسانی متغیرهای محیطی نمونه + مستندات دیپلوی

```
Goal: Update every .env example file to remove Render/Cloudinary
variables and reflect the new same-origin/local-storage architecture,
and rewrite this project's deployment documentation for the VPS-based
setup, consolidating the previously Render/Vercel-specific docs.

Before starting, read:
1. .env.example (root) and afra-backend/.env.example — every current
   variable
2. afra-backend/.env.prod.example (if it exists separately from
   .env.example — check) and afra-frontend/.env.prod.example
3. docs/DEPLOYMENT.md, DEPLOYMENT_RENDER_VERCEL.md, CI_CD_SETUP.md (or
   wherever these live — confirm exact paths first) in full

What to build:

1. In every env example file, remove: RENDER_EXTERNAL_HOSTNAME,
   CLOUDINARY_CLOUD_NAME, CLOUDINARY_API_KEY, CLOUDINARY_API_SECRET.
   Add (if not already present) any new variables Prompt 5's deploy
   workflow references being set as VPS-side environment (these live
   in the VPS's own .env.prod file, not GitHub Secrets, so confirm
   they're documented in afra-backend's or the root's .env.prod.example
   rather than assuming deploy.yml itself needs them as CI secrets
   beyond the SSH credentials).
   Update afra-frontend's VITE_API_BASE_URL example to the new
   relative-path default ("/api"), matching Prompt 3's compose change,
   with a comment explaining why (same-origin nginx proxy).
   If Prompt 4 found API_ORIGIN is now fully unused (dead), remove it
   from the frontend's env example and from the nginx
   docker-entrypoint.sh substitution step (check that script's content
   first) — this is the deferred cleanup Prompt 4 flagged.

2. Rewrite docs/DEPLOYMENT.md as the single, authoritative deployment
   doc for this project going forward, covering:
   - Prerequisites: a VPS (mention Iranian providers generically, not
     endorsing a specific one, per this roadmap's earlier audit
     wording — e.g. "an Iranian VPS provider with adequate regional
     network performance"), Docker + Docker Compose installed, a
     domain pointed at the VPS's IP
   - The resource sizing table from this roadmap's original audit
     (RAM/CPU per service)
   - Initial setup steps: clone the repo, copy .env.prod.example to
     .env.prod and fill in real values, run
     `docker compose -f docker-compose.prod.yml up -d --build`, run
     migrations, create a superuser
   - How ongoing deploys work (referencing the new deploy.yml)
   - TLS/certificate setup (referencing whatever decision Prompt 3
     made about where TLS terminates — if deferred, say so explicitly
     here with next steps for a human to complete)
   - Backup guidance: at minimum, a note that postgres_data and
     media_data are the two volumes requiring backup, with a simple
     `pg_dump`-based example cron job (a full backup automation system
     is out of this Phase's scope, but the guidance must exist)
   - Monitoring guidance: reference this project's existing
     /api/health/ endpoint and structured JSON logging
     (apps.common.logging) as the primary observability tools now that
     Sentry may not be reliably reachable from Iran (per this
     roadmap's original audit finding) — suggest self-hosted log
     aggregation as a future improvement, not implemented here

3. Delete or fully replace docs/DEPLOYMENT_RENDER_VERCEL.md and
   docs/CI_CD_SETUP.md content — if other docs reference them by name
   (check via grep), update those references to point at the
   consolidated DEPLOYMENT.md instead.

Files affected:
- .env.example, afra-backend/.env.example,
  afra-backend/.env.prod.example (or equivalent),
  afra-frontend/.env.prod.example
- afra-frontend/docker/entrypoint.sh (or wherever the nginx
  entrypoint substitution script lives — only if API_ORIGIN cleanup
  applies)
- docs/DEPLOYMENT.md (rewritten)
- docs/DEPLOYMENT_RENDER_VERCEL.md, docs/CI_CD_SETUP.md
  (deleted/replaced)
- README.md (only if it references the deleted docs — check first)

Acceptance Criteria:
- grep -rln "CLOUDINARY\|RENDER_EXTERNAL_HOSTNAME\|vercel" . --include=*.example --include=*.md | grep -v node_modules
  returns no results (except, if intentionally kept, a historical note
  in a CHANGELOG-style doc explicitly marked as "previously used,
  removed in this roadmap's Phase 6" — report any such intentional
  exception)
- docs/DEPLOYMENT.md is self-contained and doesn't reference
  Render/Vercel-specific steps anywhere

Verification Steps:
1. grep -rln "CLOUDINARY\|RENDER_EXTERNAL_HOSTNAME\|vercel" . --include=*.example --include=*.md | grep -v node_modules
2. Manual read-through of the rewritten docs/DEPLOYMENT.md for
   completeness against the checklist above
3. git diff --stat
```

### Prompt 7 — بررسی نهایی End-to-End کامل (کل Roadmap)

```
Goal: Final comprehensive verification of this Phase and, since this
is the last Phase of the entire roadmap, a full regression pass across
the whole project — backend and frontend — to confirm the Iranian VPS
deployment architecture works end to end with zero functional
regression from every prior Phase of both roadmaps.

Before starting, review the diffs from Prompts 1-6.

What to build:

1. (If Docker is available in this environment) Perform a full local
   simulation of the target deployment:
   cp .env.prod.example .env.prod   # fill with test values
   docker compose -f docker-compose.prod.yml up -d --build
   Wait for all services to report healthy (docker compose ps), then:
   - curl the health endpoint through nginx:
     curl http://localhost/api/health/ — confirm 200 with
     {"database": "ok", "redis": "ok", "celery": "ok"}
   - Confirm celery_worker and celery_beat containers are genuinely
     running (docker compose logs celery_worker | tail, docker compose
     logs celery_beat | tail) and show real startup log lines, not an
     eager-mode no-op
   - Trigger any simple async-task-producing action available (e.g. a
     registration email send, if test data/a test user can be created
     against this running stack) and confirm, via celery_worker's
     logs, that it was picked up asynchronously by the worker — not
     executed inline in the web request (this is the definitive proof
     that Prompt 1's EAGER=False change actually took effect end to
     end, not just in settings)
   - Upload a test file (e.g. via the API, if a quick authenticated
     request can be scripted) and confirm it's retrievable via
     http://localhost/media/... served by nginx directly (check
     response headers show nginx, not Django, served it)
   docker compose -f docker-compose.prod.yml down (clean up)

2. If Docker isn't available in this environment, perform the
   equivalent verification via static analysis and configuration
   review instead, and explicitly note in your final summary that live
   verification could not be performed here and should be done by a
   human against a real or containerized environment before genuine
   production cutover.

3. Run the full backend and frontent test suites one final time as an
   overall regression check for the ENTIRE roadmap (not just this
   Phase): pytest -x (backend), npx vitest run && npm run build
   (frontend).

4. Final project-wide grep sweep:
   grep -rln "render\.com\|vercel\.com\|cloudinary" . --include=*.py --include=*.ts --include=*.tsx --include=*.yml --include=*.yaml --include=*.md --include=*.json | grep -v node_modules | grep -v .git
   (report the complete output — this should be empty or contain only
   deliberately-preserved historical references, per Prompt 6's
   exception clause)

Files affected:
- None expected — this is a verification-only prompt; report and fix
  anything unexpectedly found

Acceptance Criteria:
- The full docker compose stack (if testable in this environment)
  starts healthy, serves /api/health/ through nginx, runs Celery
  asynchronously (not eagerly), and serves media directly via nginx
- pytest -x (backend) passes with zero failures
- npx vitest run and npm run build (frontend) both succeed
- The final grep sweep is clean (or exceptions are explicitly
  documented)

Verification Steps:
1. (If Docker available) full docker compose up/down cycle as
   described in step 1
2. pytest -x -v
3. npx vitest run
4. npm run build
5. grep -rln "render\.com\|vercel\.com\|cloudinary" . --include=*.py --include=*.ts --include=*.tsx --include=*.yml --include=*.yaml --include=*.md --include=*.json | grep -v node_modules | grep -v .git
6. git diff --stat (final summary of this Phase, and — since this is
   the last Phase of the entire roadmap — a closing summary of the
   full transformation: content, contact, admin dashboard, color
   palette, homepage components, and Iranian deployment readiness, all
   built incrementally on the already-solid 9-phase marketplace
   foundation from the previous roadmap, with zero unnecessary
   rewrites throughout)
```
