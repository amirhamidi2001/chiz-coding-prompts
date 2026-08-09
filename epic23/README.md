# Epic 23 — Security Hardening — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–22 are fully merged.

**Important grounded discovery for this epic — read before starting Task 23.1.1.3:** Epic 20 Task 20.1.1.5 **already built and deployed exactly the kind of content-based (not extension/header-based) file validation** this epic's backlog describes — a shared `validate_uploaded_image()` function using Pillow to inspect actual file bytes, applied across all 7 `ImageField`s in the project, explicitly replacing a confirmed real vulnerability in `AvatarUploadSerializer` that trusted the client-supplied `Content-Type` header. **Task 23.1.1.3 below is therefore an audit-and-extend task**, not a rebuild — its real remaining scope is confirming Epic 20's coverage is complete and extending the SAME rigor to **non-image** file uploads that Epic 20 didn't cover (specifically Epic 17 Task 17.2.1.2's CSV/Excel bulk product import, the only other file-upload surface in this codebase).

**Confirmed directly from the repo, relevant across several tasks:** `.github/workflows/ci.yml` already has `backend` (pytest against real Postgres+Redis), `frontend` (vitest), and `docker` (image build) jobs — Task 23.1.1.4 adds to this existing pipeline, not a new one. The Django admin path is **already configurable** via `ADMIN_URL = config("ADMIN_URL", default="admin/")` in `core/urls.py` — a baseline "don't use the default /admin/ path" mitigation already exists; Task 23.1.1.6 builds real access control on top of this existing foundation, not from zero.

---

## Phase 23.1 — Application Security

### Feature 23.1.1 — Header & Policy Hardening

---

#### Task 23.1.1.1 — Add Content-Security-Policy header (`django-csp`)

```
You are working in backend/requirements.txt, core/settings/base.py
(and production.py/development.py as needed). Assume Epics 1–22 are
fully merged.

CONTEXT
No Content-Security-Policy header is set anywhere in this project —
by this point the frontend legitimately loads resources from several
distinct origins that a CSP needs to explicitly account for: this
project's own API/media origin, a CDN for Bootstrap Icons (confirmed
via a CDN `<link>` in `index.html` per earlier grounding), the
self-hosted Vazirmatn font (Epic 14 Task 14.3.1.4 — same-origin, no
CSP concern), and the WebSocket connections established for chat/
notifications (Epic 2/16). A CSP that's too strict will silently break
real, working functionality; one that's too loose provides little real
protection — getting the actual directive values right for THIS
project's real resource origins matters more than just enabling the
feature.

TASK
Add `django-csp` and configure a CSP that's genuinely correct for this
project's actual resource-loading footprint.

REQUIREMENTS
- Add `django-csp` to backend/requirements.txt, pinned to a current
  stable version. Add `"csp.middleware.CSPMiddleware"` to `MIDDLEWARE`
  in backend/core/settings/base.py, positioned per the package's
  current documented placement recommendation (typically near the top
  of the middleware stack, after security-related middleware like
  `SecurityMiddleware` — verify against current `django-csp` docs for
  the specific version installed).
- Configure directives based on this project's ACTUAL confirmed
  resource origins — audit rather than guess:
  ```python
  CONTENT_SECURITY_POLICY = {
      "DIRECTIVES": {
          "default-src": ["'self'"],
          "img-src": ["'self'", "data:"],  # 'data:' for any inline/base64 images if used anywhere (e.g. small icons/placeholders)
          "font-src": ["'self'"],  # self-hosted Vazirmatn (Epic 14), no external font CDN
          "style-src": ["'self'", "https://cdn.jsdelivr.net"],  # VERIFY against the actual Bootstrap Icons CDN URL used in index.html — adjust to the real, confirmed domain
          "script-src": ["'self'"],
          "connect-src": ["'self'", "wss://" + config("PRODUCTION_DOMAIN", default="chiz.com")],  # WebSocket connections for chat/notifications (Epics 2/16)
          "frame-ancestors": ["'none'"],  # this platform should never be embedded in an iframe elsewhere — clickjacking protection
          "form-action": ["'self'"],
      },
  }
  ```
  Before finalizing, ACTUALLY INSPECT `frontend/index.html` and every
  place external resources are loaded (CDN links, any third-party
  embeds/widgets from prior epics — check whether Epic 6/7's payment/
  shipping gateway integrations ever embed anything CLIENT-SIDE, e.g.
  a hosted payment widget rather than a pure redirect, which per Epic
  6's actual design is a full-page redirect, not an embed — confirm
  this holds, since a CSP would need a very different `frame-src`/
  `connect-src` treatment if any gateway integration turns out to load
  content client-side rather than redirecting) to build an accurate
  directive list, not a generic template copied from elsewhere.
- Set `CSP_REPORT_ONLY` mode (via whatever mechanism the installed
  `django-csp` version provides for this) for an initial ROLLOUT
  PERIOD in production — a CSP deployed directly in enforcing mode
  risks silently breaking real functionality that wasn't correctly
  accounted for in the directive list above; report-only mode logs
  violations without actually blocking anything, letting you observe
  real-world violation reports (if a reporting endpoint is configured)
  or browser console warnings before flipping to enforcing mode. Add a
  settings flag: `CSP_REPORT_ONLY = config("CSP_REPORT_ONLY", default=True, cast=bool)`
  in production.py specifically, with a clear comment instructing
  ops/whoever manages the production deploy to monitor for violations
  for some period before setting this to `False`.
- Confirm `SECURE_SSL_REDIRECT`/HSTS settings (already confirmed
  present in production.py from Epic 1's original grounding) aren't
  duplicated/conflicting with anything `django-csp` might also try to
  manage — these are separate, complementary security headers, not
  overlapping ones, so this should just be a confirmation, not a
  conflict to resolve.

ACCEPTANCE CRITERIA / TESTS
- Add a test confirming the CSP header is present on a real response
  with the expected directive values.
- Manually verify (running the real app locally with the CSP active):
  every page loads without CSP-related console errors/warnings — the
  homepage, PDP (images/font), checkout (payment gateway redirect
  flow), and the chat/notification WebSocket connection all function
  correctly with the CSP active — this is a genuine, necessary manual
  verification pass given how easy it is for a CSP to silently break
  something a purely automated test wouldn't catch.
- If report-only mode is used initially, confirm violation reports (if
  a reporting endpoint is configured) or browser console warnings are
  actually visible/observable, so the rollout process described above
  is genuinely actionable, not just theoretical.
```

---

#### Task 23.1.1.2 — Add `django-axes` for login lockout after repeated failures

```
You are working in backend/requirements.txt, core/settings/base.py.
Assume Epics 1–22 are fully merged, including Epic 2 Task 2.4.1.4's
`AuthSensitiveRateThrottle` (a request-RATE limit on login/register/
password-reset endpoints, distinct from what this task adds).

CONTEXT
Epic 2 Task 2.4.1.4 already added rate-LIMITING to auth endpoints (a
DRF throttle capping requests per time window) — but rate limiting and
account LOCKOUT are different, complementary protections: a rate limit
slows down a brute-force attempt but doesn't stop it from eventually
succeeding given enough time/distributed IPs; account lockout
specifically responds to REPEATED FAILURES AGAINST ONE ACCOUNT by
locking that account out entirely for a period, which is a stronger,
more targeted protection against credential-stuffing/brute-force
attacks on a SPECIFIC user's credentials. Neither replaces the other —
this task adds the missing second layer.

TASK
Add `django-axes`, configured for both this project's email/password
`LoginView` AND Epic 2's OTP verification flow (both are
"authentication attempt" surfaces that can be brute-forced).

REQUIREMENTS
- Add `django-axes` to backend/requirements.txt, pinned to a current
  stable version. Add `"axes"` to `INSTALLED_APPS`, and
  `"axes.backends.AxesStandaloneBackend"` as the FIRST entry in
  `AUTHENTICATION_BACKENDS` in backend/core/settings/base.py (axes
  requires being checked before Django's default `ModelBackend` — set
  `AUTHENTICATION_BACKENDS = ["axes.backends.AxesStandaloneBackend", "django.contrib.auth.backends.ModelBackend"]`
  if this setting doesn't already exist with a custom value; if it
  DOES already have custom entries from a prior epic, insert axes'
  backend FIRST in the existing list rather than overwriting it).
  Add `"axes.middleware.AxesMiddleware"` to `MIDDLEWARE`, positioned
  per the package's current documented placement requirement (typically
  near the end of the middleware stack — verify against current docs
  for the installed version).
  Generate/apply `axes`'s migrations.
- Configure lockout behavior:
  ```python
  AXES_FAILURE_LIMIT = 5
  AXES_COOLOFF_TIME = 1  # hours
  AXES_LOCKOUT_PARAMETERS = ["username"]  # lock by account (email), not by IP alone — a distributed attacker rotating IPs against ONE account should still be caught
  AXES_RESET_ON_SUCCESS = True  # a successful login clears the failure count
  ```
  The `AXES_LOCKOUT_PARAMETERS = ["username"]` choice (locking by the
  ATTEMPTED ACCOUNT identifier, not by source IP) is a deliberate,
  important decision: IP-based lockout alone is both easily bypassed
  (an attacker with access to multiple IPs/a botnet) AND has a real
  denial-of-service risk of its own (a shared corporate/NAT IP with
  many legitimate users could get one user's failed attempts lock out
  everyone behind that IP) — locking by the account being attacked is
  the more correct, standard mitigation for credential-stuffing
  specifically. Document this choice.
- CRITICAL: `django-axes` out of the box hooks into Django's standard
  `django.contrib.auth.authenticate()` flow — but THIS project's real
  login surface is `LoginView(TokenObtainPairView)` (SimpleJWT-based,
  per Epic 2's grounding), and its OTP verification flow
  (`OTPVerifyView`, a plain `APIView` with entirely custom logic, NOT
  going through Django's standard `authenticate()` call at all, per
  Epic 2 Task 2.3.1.2's design). Verify axes' integration ACTUALLY
  covers `LoginView`'s SimpleJWT-based auth path correctly (SimpleJWT's
  `TokenObtainPairSerializer` DOES internally call
  `authenticate()`, so axes' standard hook likely works correctly here
  — confirm with a real test rather than assuming) — but axes' standard
  hooks will NOT automatically cover `OTPVerifyView`, since that view
  never calls Django's `authenticate()` at all. For OTP specifically,
  either (a) manually integrate axes' lower-level API
  (`axes.utils.get_client_username`/`AxesBackend`-adjacent helper
  functions, or the `axes.signals` module) directly into
  `OTPVerifyView` to record failed OTP verification attempts and check
  lockout status before processing, or (b) rely on the ALREADY-EXISTING,
  separate `OTP_MAX_VERIFICATION_ATTEMPTS` mechanism (Epic 2 Task
  2.1.2.3) as functionally equivalent protection for OTP specifically
  (it already permanently invalidates a code after too many wrong
  guesses, which is arguably MORE targeted protection than axes would
  add, since it's scoped to the specific code rather than a time-based
  account lockout) — RECOMMEND option (b): rely on the existing,
  already-built OTP-specific protection rather than force-fitting axes
  onto a flow it wasn't designed for, and scope THIS task's axes
  integration to the email/password `LoginView` specifically, where
  axes' standard `authenticate()`-based hooks apply cleanly and
  correctly. Document this scoping decision clearly.
- Confirm axes' lockout response (a 403, by default, when a locked-out
  account attempts to authenticate) integrates sensibly with this
  project's existing error-response conventions — check whether the
  frontend's login-error handling (per Epic 2 Task 2.3.2.1's
  established error-display pattern) needs any adjustment to show a
  clear "too many failed attempts, try again in X" message rather than
  a generic/confusing error.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/accounts/tests/test_views.py:
1. 5 consecutive failed login attempts against the SAME account (via
   `LoginView`) result in the 6th attempt (even with the CORRECT
   password) being rejected due to lockout, not proceeding to normal
   authentication.
2. A DIFFERENT account is NOT locked out by failed attempts against
   the first account (confirming per-account, not global, lockout).
3. A successful login resets the failure counter (subsequent failed
   attempts start counting from zero again, per
   `AXES_RESET_ON_SUCCESS`).
4. After the configured `AXES_COOLOFF_TIME`, a previously-locked-out
   account can attempt login again (test via manipulating time/using
   axes' own testing utilities if the package provides them, matching
   whatever pattern its documentation recommends for testing lockout
   expiry).
5. Confirm `OTPVerifyView`'s EXISTING `OTP_MAX_VERIFICATION_ATTEMPTS`
   protection (Epic 2 Task 2.1.2.3) still works correctly and is
   UNAFFECTED by this task's changes (a regression check, given this
   task deliberately did NOT modify that flow, per the scoping decision
   above — confirm that decision didn't accidentally leave OTP
   completely unprotected due to some unexpected interaction with the
   newly-added axes middleware).
```

---

#### Task 23.1.1.3 — File upload validation (MIME sniffing, not just extension)

```
You are working in backend/dashboard/serializers.py (or wherever Epic
17 Task 17.2.1.2's bulk-import file-handling logic lives) and
backend/core/validators.py (Epic 20 Task 20.1.1.5's shared image
validator, established there). Assume Epic 20 Task 20.1.1.5 is
already merged. RE-READ THIS DOCUMENT'S HEADER — this task's real scope
is narrower than its title suggests.

CONTEXT — READ THIS CAREFULLY BEFORE STARTING
Epic 20 Task 20.1.1.5 already built `validate_uploaded_image()` — a
content-based (Pillow `Image.open().format`, not
extension/`Content-Type`-header-based) validator — and applied it
across ALL SEVEN `ImageField`s in this codebase. That work directly
satisfies this task's backlog description for every IMAGE upload
surface already. This task's ACTUAL remaining scope: (1) confirm that
coverage is genuinely complete (a verification pass, given how many
epics have touched image fields since), and (2) extend the SAME
content-based validation rigor to the ONE OTHER file-upload surface in
this codebase that Epic 20 didn't cover at all — Epic 17 Task
17.2.1.2's CSV/Excel bulk product import, which currently accepts
`request.FILES.get("file")` with NO content validation whatsoever
(only a `ValueError` catch around the actual PARSING attempt, which is
a very different, much weaker guarantee than validating the file's
real type/structure BEFORE attempting to process it as trusted data).

TASK
1. Audit every `ImageField`-accepting serializer to confirm Epic 20's
   validator is genuinely applied everywhere (close any gap found).
2. Add content-based validation to the CSV/Excel bulk-import upload
   path specifically.

REQUIREMENTS — Part 1: audit
- Grep the ENTIRE backend for every `ImageField(` declaration and
  every serializer field of type `serializers.ImageField`, and trace
  each one back to confirm it genuinely routes through
  `validate_uploaded_image()` (or an equivalent explicit `validate_<field>()`
  method calling it) — don't assume Epic 20's coverage was complete
  just because that task claimed to cover "all 7 fields"; a NEW
  ImageField could theoretically have been introduced by a LATER epic
  (check whether any epic after Epic 20 added a new image-bearing
  model or field — review the full epic sequence for this) that never
  got this validation applied. Fix any gap found.

REQUIREMENTS — Part 2: bulk-import file validation
- Add content-based validation for the CSV/Excel upload specifically,
  BEFORE attempting to parse it as trusted spreadsheet data:
  ```python
  # backend/core/validators.py — alongside validate_uploaded_image()
  import csv
  import io
  from openpyxl import load_workbook
  from openpyxl.utils.exceptions import InvalidFileException

  ALLOWED_IMPORT_EXTENSIONS = {".csv", ".xlsx"}
  MAX_IMPORT_FILE_SIZE_MB = 10


  def validate_uploaded_spreadsheet(value):
      """
      Validate a bulk-import file by ACTUAL FILE STRUCTURE, not just
      its filename extension or client-supplied Content-Type.
      """
      if value.size > MAX_IMPORT_FILE_SIZE_MB * 1024 * 1024:
          raise serializers.ValidationError(f"File must be under {MAX_IMPORT_FILE_SIZE_MB} MB.")

      filename = value.name.lower()
      if filename.endswith(".xlsx"):
          try:
              value.seek(0)
              load_workbook(value, read_only=True)  # will raise if not a genuine, well-formed XLSX
          except InvalidFileException:
              raise serializers.ValidationError("The uploaded file is not a valid Excel (.xlsx) file.")
          finally:
              value.seek(0)
      elif filename.endswith(".csv"):
          try:
              value.seek(0)
              sample = value.read(8192).decode("utf-8-sig")
              csv.Sniffer().sniff(sample)  # raises csv.Error if the content doesn't genuinely look like delimited CSV data
          except (UnicodeDecodeError, csv.Error):
              raise serializers.ValidationError("The uploaded file is not a valid CSV file.")
          finally:
              value.seek(0)
      else:
          raise serializers.ValidationError("Only .csv and .xlsx files are supported.")
      return value
  ```
  Note this is a GENUINELY WEAKER content-validation guarantee than
  Pillow's magic-byte image checking — CSV in particular is just
  delimited TEXT with no strong binary format signature to verify
  against, so `csv.Sniffer()`'s heuristic ("does this look like
  legitimate CSV structure") is the best practically-available check,
  not a hard cryptographic guarantee the way `Image.open()` provides
  for binary image formats — document this honestly rather than
  overstating the strength of CSV validation specifically. XLSX, being
  a genuine binary ZIP-based format, gets a stronger guarantee via
  `openpyxl`'s own parser correctly rejecting malformed files.
  A file that is NEITHER a valid image NOR a valid spreadsheet but
  disguises itself with a `.csv`/`.xlsx` extension AND happens to pass
  the loose `csv.Sniffer()` heuristic (e.g. a crafted text file that
  happens to look CSV-shaped) is a genuine, acknowledged residual risk
  of text-based formats that binary-format validation doesn't have —
  the REAL, primary defense against this residual risk is what Epic
  17 Task 17.2.1.2's import logic ALREADY does downstream: it processes
  each row through normal Django ORM operations with normal field
  validation (a malicious "product name" field, however it got there,
  still just becomes a `CharField` value subject to normal length/
  content handling, not executable code) — this validator's job is
  catching the OBVIOUSLY malformed/wrong-type case early with a clear
  error, not serving as the sole line of defense against arbitrary
  malicious content, which the existing per-row processing/validation
  already provides.
- Wire this into Epic 17 Task 17.2.1.2's `BulkProductImportView.post()`:
  ```python
  def post(self, request):
      file = request.FILES.get("file")
      if not file:
          return Response({"file": "No file provided."}, status=400)
      try:
          validate_uploaded_spreadsheet(file)
      except serializers.ValidationError as exc:
          return Response({"file": exc.detail}, status=400)
      ...  # existing parsing/processing logic continues unchanged
  ```

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/dashboard/tests/test_views.py (extending Epic
17 Task 17.2.1.2's existing import tests):
1. A genuinely valid CSV file passes validation.
2. A genuinely valid XLSX file passes validation.
3. An oversized file (either format) is rejected before any parsing
   attempt.
4. A file with a `.csv` extension but genuinely non-CSV-shaped content
   (e.g. random binary data) is rejected by the sniffer check.
5. A file with a `.xlsx` extension but that is NOT actually a valid
   ZIP-based Excel file (e.g. a renamed plain text file) is rejected
   by `openpyxl`'s parse failure.
6. A file with an unsupported extension (e.g. `.txt`, `.pdf`) is
   rejected regardless of its actual content.
Add a brief audit summary (in your task response, not necessarily a
committed file) confirming which specific serializers/fields you
verified already correctly use `validate_uploaded_image()`, and
explicitly note if you found and fixed any gap.
```

---

#### Task 23.1.1.4 — Dependency vulnerability scanning in CI (`pip-audit`, `npm audit`)

```
You are working in .github/workflows/ci.yml. Assume Epics 1–22 are
fully merged. RE-READ THIS DOCUMENT'S HEADER — the existing workflow's
exact job structure (`backend`, `frontend`, `docker`) is confirmed and
must be extended to match its existing conventions precisely.

CONTEXT
The CI pipeline currently runs tests but never checks whether any
installed dependency (Python or npm) has a KNOWN published
vulnerability — across 23 epics' worth of dependency additions
(Celery, django-redis, django-axes, django-csp, weasyprint, openpyxl,
react-helmet-async, dayjs plugins, and everything else added along the
way), there's real, accumulated surface area this check has never
covered even once.

TASK
Add dependency vulnerability scanning to both the `backend` and
`frontend` CI jobs, failing the build on high-severity findings.

REQUIREMENTS
- In the `backend` job (`.github/workflows/ci.yml`), add a step AFTER
  `Install dependencies` and BEFORE (or alongside, order doesn't
  strictly matter as long as it runs) `Run pytest`:
  ```yaml
  - name: Audit Python dependencies for known vulnerabilities
    run: |
      pip install pip-audit
      pip-audit --requirement requirements.txt --strict --desc
  ```
  `--strict` makes `pip-audit` exit non-zero (failing the CI step, and
  therefore the job) on ANY finding — consider whether this is too
  aggressive for a project with 23+ epics of dependencies (a LOW-
  severity finding with no realistic exploit path might be an
  unreasonable reason to block every single PR) versus whether
  filtering to only HIGH/CRITICAL severity findings is more
  practical — check `pip-audit`'s current documented options for
  severity-based filtering (its vulnerability data source and exact
  filtering flags may have evolved; verify against current docs rather
  than assuming a specific flag name) and use whichever approach
  balances "catch real problems" against "don't create constant
  false-alarm friction that trains developers to ignore CI failures
  entirely" — a security check that's SO noisy nobody takes it
  seriously is often worse than a more targeted one; use judgment
  here, documenting whichever choice you make.
- In the `frontend` job, add a step:
  ```yaml
  - name: Audit npm dependencies for known vulnerabilities
    run: npm audit --audit-level=high
  ```
  `--audit-level=high` (only failing on high/critical findings, not
  every low-severity advisory) is a REASONABLE default starting point
  for the exact same "don't create excessive false-alarm friction"
  reasoning as the backend step above — adjust if this project's
  actual security posture/risk tolerance warrants a stricter or looser
  threshold, but start here rather than the maximally strict
  `--audit-level=low`, which would likely generate an unmanageable
  volume of low-value findings across a real `node_modules` tree's
  full transitive dependency graph.
- Consider whether these should be BLOCKING (failing the PR/build) or
  NON-BLOCKING (reporting findings without failing, matching the
  existing `continue-on-error: true` pattern already used for the
  frontend coverage step in this same workflow file) for an INITIAL
  rollout period — introducing a new, potentially-noisy CI gate that
  immediately blocks all future PRs the moment it's added (if the
  CURRENT dependency tree already has some pre-existing findings that
  haven't been triaged yet) could be disruptive. RECOMMEND: run BOTH
  new steps in `continue-on-error: true` mode INITIALLY (matching the
  existing precedent already established in this exact file for the
  coverage step), giving whoever owns this project a chance to review
  and address any findings the FIRST run surfaces, then flip to
  blocking (`continue-on-error: false`, the default) in a FOLLOW-UP
  change once the current dependency tree is confirmed clean — don't
  silently make this permanently non-blocking and call the task done,
  since that defeats the actual security purpose; document this as an
  explicit two-phase rollout with the follow-up step clearly flagged.
- Add a step to upload the audit results as a build artifact (mirroring
  the EXISTING `Upload coverage report` step's pattern in this same
  workflow file) so findings are reviewable even when running in
  non-blocking mode:
  ```yaml
  - name: Save pip-audit results
    if: always()
    run: pip-audit --requirement requirements.txt --format json --output pip-audit-results.json || true
  - name: Upload pip-audit results
    uses: actions/upload-artifact@v4
    if: always()
    with:
      name: pip-audit-results
      path: backend/pip-audit-results.json
  ```

ACCEPTANCE CRITERIA / TESTS
- This is a CI-configuration task, verified by actually running the
  workflow (push to a branch/open a PR, or run the job locally via
  `act` or equivalent if available in your environment) and confirming
  both new steps execute, produce readable output, and correctly
  succeed/fail based on real findings against the CURRENT dependency
  tree.
- Review whatever findings the FIRST real run surfaces — if any
  genuine high/critical vulnerabilities are found in currently-pinned
  dependencies, address them (bump the affected package to a patched
  version, re-run, confirm clean) as part of completing this task,
  rather than merging a security-scanning task that immediately starts
  its life failing/ignoring known findings.
```

---

#### Task 23.1.1.5 — Secrets audit — confirm no credentials in repo/history

```
You are working with the project's actual GIT REPOSITORY (not just the
current file tree — this task specifically requires inspecting
commit HISTORY, which is meaningfully different from auditing the
current working directory's contents). Assume Epics 1–22 are fully
merged.

CONTEXT
Across 23 epics' worth of commits, there's a real, non-zero risk that
a real secret (an API key, a database password, a `SECRET_KEY` value,
a payment gateway credential) was accidentally committed at some point
— even briefly, even if later removed in a subsequent commit — and
git's fundamental design means "removed in a later commit" does NOT
mean "gone from history"; the secret remains permanently retrievable
from any earlier commit unless the history itself is rewritten. This
task is a genuine audit, not a formality — confirmed from THIS specific
extracted snapshot: no `.env`/`.env.dev`/`.env.prod` files exist in the
current working tree, and `.gitignore` correctly excludes them going
forward — but this only confirms the CURRENT state is clean, not that
history is.

TASK
Run an actual secrets-scanning tool against the full git history (not
just the current working tree), triage any findings, and rotate any
genuinely-real credential found.

REQUIREMENTS
- Run a dedicated git-history secrets scanner against the FULL commit
  history — `gitleaks` and `trufflehog` are both well-established,
  actively-maintained options for this specific purpose (verify
  current maintenance status/recommended usage before choosing,
  ecosystem tooling can shift); EITHER is reasonable, pick one:
  ```bash
  # example using gitleaks
  gitleaks detect --source . --report-format json --report-path gitleaks-report.json --log-opts="--all"
  ```
  The `--log-opts="--all"` (or equivalent full-history flag for
  whichever tool you choose) is important — a scan of only the CURRENT
  HEAD commit would miss exactly the "committed then later removed"
  scenario this task exists to catch.
- Triage every finding: for EACH one, determine (a) is this a genuine,
  real secret (an actual working API key/password), or a FALSE
  POSITIVE (test/placeholder values like the CI workflow's confirmed
  `SECRET_KEY: ci-secret-key-not-used-in-production` and
  `DATABASE_PASSWORD: postgres` — these are intentional, non-sensitive
  placeholder values used ONLY in the CI environment, not real
  credentials, and scanning tools frequently flag such patterns even
  though they pose no actual risk)? Document each finding's
  disposition clearly.
- For any GENUINE real secret found (even in a since-removed/rewritten
  commit): 
  1. IMMEDIATELY rotate/invalidate that credential at its source (the
     actual service it belongs to — e.g. if a real ZarinPal merchant
     key, Kavenegar API key, or database password was ever genuinely
     exposed, generating/requesting a NEW one and updating the actual
     deployed environment's configuration is the ONLY real fix — simply
     removing it from a future commit does NOT undo the exposure, since
     anyone who cloned the repo at any point had access to the old
     history).
  2. Consider whether REWRITING git history to scrub the secret
     entirely (via `git filter-repo` or similar) is warranted — this is
     a DISRUPTIVE operation (rewrites commit hashes, requires every
     collaborator to re-clone, breaks any existing PRs/branches based
     on the old history) and should only be done if there's a specific
     reason beyond "it would be nice to remove" — since ROTATING the
     credential already neutralizes the actual risk (the exposed OLD
     value becomes worthless once rotated), history rewriting is often
     NOT necessary purely for security purposes once rotation has
     happened; weigh this tradeoff explicitly rather than reflexively
     rewriting history for every finding.
- Produce a written summary of the audit: what was scanned, what tool
  was used, every finding and its disposition (genuine secret +
  rotated, vs. confirmed false positive), and any remaining action
  items.
- As a preventive measure alongside the audit itself, add a
  `.pre-commit-config.yaml` hook (or equivalent, if this project has
  any pre-commit-hook convention already — check for one) running the
  SAME secrets-scanning tool on every future commit BEFORE it's made,
  catching new secret-commits before they ever enter history at all —
  a genuinely more valuable long-term control than a one-time
  retrospective audit alone. This is optional/nice-to-have relative to
  the audit itself, but a natural, cheap addition while already set up
  to run the scanning tool.

ACCEPTANCE CRITERIA
The completed, written audit summary (findings + dispositions + any
rotation actions taken) is this task's primary deliverable. If ANY
genuine secret was found, the corresponding rotation MUST be confirmed
complete (the old credential invalidated at its source, the new one
correctly deployed) before this task can be considered done — a
documented finding with no actual remediation is not an acceptable
outcome for a genuine secret exposure.
```

---

#### Task 23.1.1.6 — Admin panel IP allowlist or extra-auth layer

```
You are working in backend/core/settings/production.py, and
potentially backend/core/middleware.py (new file) or nginx/reverse-
proxy configuration if this project's deployment architecture makes
that the more appropriate layer. Assume Epics 1–22 are fully merged.
RE-READ THIS DOCUMENT'S HEADER — `ADMIN_URL` is already
env-configurable, an existing baseline this task builds on top of, not
starting from zero.

CONTEXT
The Django admin currently has TWO layers of protection: Django's own
authentication (only staff/superuser accounts can log in at all) and
the confirmed-existing `ADMIN_URL` env-configurability (obscuring the
default `/admin/` path). Neither of these is a REAL access-control
layer independent of credentials — if an admin credential is ever
compromised (phished, leaked, weak password despite Epic 23.1.1.2's
lockout protection), the attacker has full admin access from anywhere
in the world with no additional barrier. This task adds a genuine
SECOND factor of access control independent of credentials: restricting
WHO can even REACH the admin login page at all, by network origin.

TASK
Add IP-allowlist-based access restriction to the Django admin,
configurable per environment (since a real production deployment's
legitimate admin IPs — office/VPN egress addresses — differ entirely
from what's needed in development/CI).

REQUIREMENTS
- Decide the enforcement LAYER: this can be implemented at the
  Django/application layer (a middleware checking `request.META["REMOTE_ADDR"]`
  against an allowlist before permitting any `/admin/`-prefixed
  request through) OR at the reverse-proxy/infrastructure layer (nginx
  `allow`/`deny` directives, or equivalent at whatever sits in front of
  this project's Daphne ASGI server in production, per the confirmed
  `docker-compose.prod.yml` architecture). RECOMMEND the
  infrastructure/reverse-proxy layer as the STRONGER choice when
  available — blocking a request before it ever reaches the Django
  application process is more robust than an application-layer check
  (which still consumes a Django worker/process cycle to even reject
  the request, and depends on `REMOTE_ADDR` being correctly set, which
  requires correct `X-Forwarded-For` handling if there's any
  intermediate proxy/load balancer — a real, easy-to-get-wrong detail
  at the application layer). HOWEVER: implement a Django-layer
  middleware as well, regardless of whether infra-layer blocking is
  ALSO configured, as defense-in-depth — don't rely on infrastructure
  configuration alone for a control this important, since infra config
  can drift/be misconfigured without the application code itself
  reflecting that risk.
- Add the Django-layer middleware:
  ```python
  # core/middleware.py
  from django.conf import settings
  from django.http import HttpResponseForbidden


  class AdminIPAllowlistMiddleware:
      def __init__(self, get_response):
          self.get_response = get_response

      def __call__(self, request):
          if request.path.startswith(f"/{settings.ADMIN_URL}") and settings.ADMIN_IP_ALLOWLIST:
              client_ip = self._get_client_ip(request)
              if client_ip not in settings.ADMIN_IP_ALLOWLIST:
                  return HttpResponseForbidden("Access denied.")
          return self.get_response(request)

      def _get_client_ip(self, request):
          # Correctly handle a reverse-proxy setup: prefer the
          # left-most X-Forwarded-For entry (the original client) if
          # present and this deployment is confirmed to sit behind a
          # trusted proxy that sets this header correctly — using
          # X-Forwarded-For WITHOUT confirming a trusted proxy sits in
          # front is itself a spoofable vector (a client could set
          # this header directly if no trusted proxy strips/overwrites
          # it), so this check is only meaningful/safe in a deployment
          # where the reverse proxy is confirmed to be the ONLY path
          # to the Django app and correctly manages this header.
          xff = request.META.get("HTTP_X_FORWARDED_FOR")
          if xff and settings.TRUST_X_FORWARDED_FOR:
              return xff.split(",")[0].strip()
          return request.META.get("REMOTE_ADDR")
  ```
  Add `ADMIN_IP_ALLOWLIST = config("ADMIN_IP_ALLOWLIST", default="", cast=lambda v: [ip.strip() for ip in v.split(",") if ip.strip()])`
  and `TRUST_X_FORWARDED_FOR = config("TRUST_X_FORWARDED_FOR", default=False, cast=bool)`
  to backend/core/settings/base.py. Leave `ADMIN_IP_ALLOWLIST` EMPTY by
  default (an empty list means the middleware's `if ... and settings.ADMIN_IP_ALLOWLIST:`
  check is falsy, so it does NOTHING by default) — this is a
  deliberate, safe default: an accidentally-misconfigured EMPTY
  allowlist in production should NOT lock out every admin including
  legitimate ones with no way to fix it remotely; the feature must be
  explicitly OPTED INTO via a real, populated env var, not
  accidentally enforced by omission.
  Add `AdminIPAllowlistMiddleware` to `MIDDLEWARE` in base.py,
  positioned EARLY in the stack (before most other middleware, so a
  rejected admin request is short-circuited as cheaply/early as
  possible).
- Add infra-layer nginx configuration too (check this project's actual
  nginx setup — confirmed via `docker-compose.prod.yml`'s `frontend`
  service description mentioning it serves static/media files; verify
  whether nginx ALSO proxies `/admin/`-path requests to the backend, or
  whether the backend's port 8000 is reached some other way in the real
  production topology) — if nginx does sit in front of admin traffic,
  add commented-out EXAMPLE `allow`/`deny` directives to whatever nginx
  config file this project maintains, clearly documented as something
  ops must populate with real, actual office/VPN IP ranges before
  enabling, rather than shipping with fake placeholder IPs that look
  functional but aren't.
- As an ADDITIONAL, complementary layer beyond IP allowlisting (which
  requires knowing static admin IPs in advance — awkward for a
  distributed/remote team without a VPN), consider whether TOTP-based
  two-factor authentication for admin/staff accounts specifically
  (`django-otp` or similar) is a more practical fit for this project's
  actual team structure — RECOMMEND flagging this as a reasonable
  ALTERNATIVE or ADDITIONAL follow-up task rather than building it as
  part of THIS task (which is explicitly scoped to "IP allowlist OR
  extra-auth layer" per the backlog — IP allowlisting is a complete,
  valid implementation of this task on its own; 2FA is a legitimate
  alternative approach worth noting for whoever owns ongoing security
  hardening decisions, not something to build in addition without
  being asked).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/core/tests/test_middleware.py:
1. With `ADMIN_IP_ALLOWLIST` EMPTY (the default), a request to the
   admin path from ANY IP is NOT blocked by this middleware (proving
   the safe-default behavior).
2. With `ADMIN_IP_ALLOWLIST` populated, a request from an ALLOWED IP
   succeeds; a request from a NON-allowed IP is rejected with 403.
3. Non-admin-path requests are NEVER affected by this middleware
   regardless of allowlist configuration (the storefront/API must
   remain fully accessible from any IP).
4. `TRUST_X_FORWARDED_FOR=False` (the default) correctly ignores any
   client-supplied `X-Forwarded-For` header, using `REMOTE_ADDR`
   directly — confirming the spoofing-risk guard is genuinely active
   by default.
Manually verify against a real deployment (or a close local
approximation): setting `ADMIN_IP_ALLOWLIST` to exclude your own
current IP correctly blocks admin access; setting it to include your
own IP correctly allows it.
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 23.1.1.1 | Content-Security-Policy header | ☐ |
| 23.1.1.2 | django-axes login lockout | ☐ |
| 23.1.1.3 | File upload validation (audit + extend to bulk-import) | ☐ |
| 23.1.1.4 | Dependency vulnerability scanning in CI | ☐ |
| 23.1.1.5 | Secrets audit (git history) | ☐ |
| 23.1.1.6 | Admin panel IP allowlist / extra-auth layer | ☐ |

**A note on this epic's place in the backlog, echoing Epic 22's closing note:** the master backlog itself flags several of THESE tasks specifically (throttling — already pulled forward into Epic 2 Task 2.4.1.3/2.4.1.4 — plus file validation and login lockout) as P0 items that ideally belong alongside Epic 1/2, not saved for their numbered position here. Task 23.1.1.3's real remaining scope turned out to be small precisely because Epic 20 already did the heavy lifting on file validation without waiting for this epic's number to come up — a good illustration of that exact principle in practice across this document series.

Once Epic 23 is fully merged, the next epic to generate prompts for is
**Epic 24 — Testing**, which focuses on critical-path test coverage
that should ideally have been written incrementally alongside each
dependent epic (payments, coupons, OTP) as it shipped, per the master
backlog's own guidance — much of that coverage already exists
throughout this document series' individual task acceptance criteria,
so Epic 24's real remaining scope is likely narrower than its
backlog description suggests, similar to this epic's Task 23.1.1.3.
