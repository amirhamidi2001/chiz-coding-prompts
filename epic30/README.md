# Epic 30 — Documentation — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next. This is the **final epic** in the master backlog's 30-epic sequence.

**Assumed prerequisites:** Epics 1–29 are fully merged.

**The single most important, and most human, finding in this entire 30-epic document series — confirmed directly from the repo, sitting in plain sight the whole time:** `README.md`'s very first line reads:

> `# ⚡ chiz — Smart Electrical Panel E-Commerce Platform`

**This project has not sold smart electrical panels at any point across this entire 29-epic transformation.** This is leftover branding from whatever generic e-commerce template this codebase originated from before being repurposed, epic by epic, into a Persian cosmetics and skincare platform — and it was never updated, not once, across every other epic's work. The README's "Features at a Glance" section is equally frozen in time: it describes a "Payment Methods" tab as a real feature (the exact concept Epic 18 Task 18.1.1.1 identified as **conceptually obsolete** and recommended repurposing), lists only the original apps (`accounts`, `blog`, `cart`, `chat`, `contact`, `core`, `dashboard`, `order`, `shop`) with **zero mention** of `payments`, `shipping`, `promotions`, or `notifications` — four entire Django apps this series built from scratch across Epics 6, 7, 9, and 16 — and never mentions Celery, Redis caching, Sentry, or any other infrastructure from Epics 20–29. **Task 30.1.1.3 below exists specifically to fix this**, and it is, without exaggeration, the most consequential documentation task in this epic — a new engineer's or new AI agent's very first impression of this codebase, right now, is confidently, completely wrong about what the product even is.

**Also confirmed directly from the repo:** `backend/requirements.txt` has **both** `drf-spectacular==0.29.0` **and** `drf-yasg==1.21.14` installed simultaneously — two competing OpenAPI schema-generation libraries doing the same job, with `SpectacularAPIView` registered in `core/urls.py` but **zero `SPECTACULAR_SETTINGS` customization anywhere** (no project title, description, or version configured — the schema currently ships under drf-spectacular's bare defaults).

---

## Phase 30.1 — Team-Readiness Docs

### Feature 30.1.1 — Engineering Documentation

---

#### Task 30.1.1.1 — Architecture Decision Records (ADR) folder + backfill key decisions

```
You are working in backend/docs/adr/. Assume Epics 1–29 are fully
merged, including Epic 14 Task 14.1.2.1's currency-storage ADR
(0001-currency-storage.md) and Epic 14 Task 14.4.1.1's slug-strategy
ADR (0002-slug-strategy.md) — both already established the ADR
folder/numbering convention this task extends.

CONTEXT
This entire 30-epic series made a genuinely large number of
significant, non-obvious architectural decisions — most documented
only inline, scattered across individual task prompts, never
consolidated anywhere a future engineer (human or AI) could discover
them without re-reading dozens of historical task descriptions. Only
two ADRs currently exist (currency storage, slug strategy). This task
backfills the REST of the decisions that genuinely warrant a permanent,
discoverable record.

TASK
Write ADRs for every other genuinely significant architectural
decision made across this series, following the exact format/style
already established by ADR 0001/0002.

REQUIREMENTS
Write, at minimum, ADRs for the following decisions (numbered
sequentially continuing from 0002; review the full history and add any
other genuinely significant decision you find that isn't listed here —
this list is a floor, not a ceiling):

- **0003-guest-cart-and-session-strategy.md**: Epic 5's decision to
  support anonymous, session-based carts (nullable `Cart.user` +
  `session_key` with a DB-enforced exactly-one-owner constraint) rather
  than requiring authentication for cart usage, and the merge-on-login
  behavior.
- **0004-payment-gateway-abstraction.md**: Epic 6's `PaymentGateway`
  ABC + settings-driven factory pattern, the decision to create orders
  as `PENDING` before payment rather than `PROCESSING`, and the
  stock-reservation-then-release-on-failure design this implies.
- **0005-notification-channel-architecture.md**: Epic 16's unified
  `notify(user, event, context)` entry point, the decision to route
  ALL notification-triggering call sites through one function rather
  than ad-hoc per-feature sending, and the deliberately-broad exception
  handling boundary at that specific layer.
- **0006-coupon-restriction-and-guest-exclusion.md**: Epic 9's decision
  to scope coupon redemption to authenticated users only (and the
  reasoning: `uses_per_user` is unenforceable for a session-only
  identity), and the category/product-restriction design that forced
  `validate_coupon()`'s signature to accept a full cart rather than a
  bare subtotal.
- **0007-redis-multi-tenancy-allocation.md**: Epic 21's documented
  Redis DB-index allocation scheme (Channels=0, Celery=1, cache=2,
  sessions=3) and the reasoning for keeping each subsystem isolated on
  one shared Redis instance rather than provisioning separate instances.
- **0008-analytics-tool-selection.md**: Epic 27's choice of self-hosted
  Plausible over GA4, specifically the Iran-market Google-service-
  reliability reasoning that drove it — this is exactly the kind of
  decision whose REASONING is far more valuable to preserve than the
  choice itself, since a future engineer unaware of the reasoning could
  easily "simplify" this back to GA4 without understanding why that
  would be a regression for this platform's specific audience.
- **0009-celery-should-have-been-built-earlier.md** (or fold this into
  whichever ADR fits best): document the master backlog's own explicit
  guidance that Celery infrastructure (and, similarly, several Epic 23
  security items) were flagged as "cheap early, expensive to retrofit"
  and should ideally have been pulled forward rather than built at
  their numbered epic position — a genuinely useful RETROSPECTIVE
  record for whoever plans the NEXT project's epic sequencing, distinct
  from a decision about THIS project's own architecture.
- **0010-frontend-refactor-deferred-to-epic-29.md**: document WHY
  code-splitting/React-Query/form-library adoption were deferred to
  the very end (Epic 29) rather than being foundational from Epic 1 —
  an honest record that this was a real, accumulated cost (every
  earlier epic's frontend work was built WITHOUT these tools, meaning
  Epic 29 had to migrate a large amount of existing code rather than
  establishing conventions from a clean slate) worth acknowledging
  explicitly rather than glossing over.
- Follow the SAME format as ADRs 0001/0002 for every new one: Context
  → Decision → Consequences/Tradeoffs, in plain, direct prose, with
  concrete references to the specific epic/task that made the decision
  so a reader can trace back to the full original reasoning if needed.

ACCEPTANCE CRITERIA
The completed set of ADR documents in backend/docs/adr/ is this task's
deliverable. Add an `backend/docs/adr/README.md` index listing every
ADR with a one-line summary, so the full set is discoverable at a
glance rather than requiring someone to open every file to find the
one relevant to their question.
```

---

#### Task 30.1.1.2 — API documentation review (drf-spectacular schema completeness)

```
You are working in backend/requirements.txt, core/settings/base.py.
Assume Epics 1–29 are fully merged. RE-READ THIS DOCUMENT'S HEADER —
the redundant drf-spectacular/drf-yasg installation is a real,
confirmed finding this task must resolve, not a hypothetical one.

CONTEXT
Two competing OpenAPI schema generators are installed simultaneously,
with the actually-registered `SpectacularAPIView` endpoint running
under completely unconfigured defaults — no project title/description/
version, meaning the API schema currently identifies itself (if at
all) as whatever drf-spectacular's generic default is, not as
"chiz API" or anything meaningful. Beyond the redundant
dependency itself, given this project has grown to encompass roughly
a dozen Django apps across this entire series (`accounts`, `blog`,
`cart`, `chat`, `contact`, `core`, `dashboard`, `order`, `shop`,
`payments`, `shipping`, `promotions`, `notifications`), there's a real
open question whether the schema this tool GENERATES actually reflects
every one of those apps' endpoints correctly and completely, or
whether some views/serializers produce incomplete/incorrect schema
entries that have simply never been reviewed.

TASK
Remove the redundant `drf-yasg` dependency, configure
`SPECTACULAR_SETTINGS` properly, and audit real schema completeness/
correctness across the full API surface.

REQUIREMENTS
- Confirm `drf-yasg` is genuinely UNUSED anywhere in the codebase
  (search for any `yasg`-specific imports/decorators/URL registrations
  across the whole backend) before removing it — if it turns out to be
  actively used somewhere this grounding pass didn't surface, DON'T
  remove it blindly; investigate and make a deliberate choice between
  the two tools instead, documenting that choice as a new ADR (per
  Task 30.1.1.1's established convention) rather than silently keeping
  both. Assuming it's genuinely unused (the more likely finding, per
  this project's confirmed `SpectacularAPIView`-based URL registration
  being the only one actually wired into `core/urls.py`): remove
  `drf-yasg` from `requirements.txt` entirely.
- Add `SPECTACULAR_SETTINGS` to `core/settings/base.py`:
  ```python
  SPECTACULAR_SETTINGS = {
      "TITLE": "chiz API",
      "DESCRIPTION": "REST API for the chiz Persian cosmetics and skincare e-commerce platform.",
      "VERSION": "1.0.0",
      "SERVE_INCLUDE_SCHEMA": False,
      "COMPONENT_SPLIT_REQUEST": True,
      "SORT_OPERATIONS": False,
  }
  ```
  (adjust the exact settings to reflect real current drf-spectacular
  configuration options and this project's actual needs — verify
  against the installed version's current documentation).
- Generate the FULL schema (`python manage.py spectacular --file schema.yaml`,
  or via the live `/api/schema/` endpoint) and REVIEW it systematically
  against every app's real endpoint list — specifically look for:
  1. Endpoints missing entirely from the generated schema (a common
     drf-spectacular gap for non-standard views — e.g. custom
     `APIView`s without proper type hints/serializer declarations, which
     several epics in this series legitimately used, e.g. Epic 6's
     `PaymentCallbackView`, Epic 8's `OrderReorderView`).
  2. Endpoints present but with INCORRECT or MISSING request/response
     schemas (e.g. a plain `Response({...})` dict return with no
     declared serializer, which drf-spectacular can't correctly infer
     the shape of without help).
  3. Missing operation descriptions/summaries on endpoints that would
     benefit from one (particularly the more UNUSUAL, non-CRUD
     endpoints built across this series — Epic 6's payment callback,
     Epic 9's coupon validation, Epic 27's conversion-funnel endpoint —
     these are exactly the ones a future API consumer would most need
     a clear description for, unlike a standard CRUD endpoint whose
     purpose is usually self-evident from its URL/method alone).
  For every gap found, add `@extend_schema` decorators (drf-spectacular's
  standard mechanism for manually annotating a view when automatic
  inference falls short) with correct request/response serializers and
  clear descriptions.
- Given the REAL scale of this audit (a dozen apps' worth of endpoints
  accumulated across 29 epics), treat this as a legitimately multi-
  session task, matching the same honest scoping already established
  for large sweeps elsewhere in this series (Epic 14's translation-
  string wrapping, Epic 29's fetch-site migration) — prioritize the
  HIGHEST-VALUE, most-likely-to-be-consumed-by-a-real-API-client
  endpoints first (checkout, payments, cart, product listing) over
  lower-traffic admin-only endpoints, and document remaining work
  clearly if not fully completed in one pass.

ACCEPTANCE CRITERIA / TESTS
- `python manage.py spectacular --file schema.yaml --validate` runs
  with no schema-generation ERRORS (warnings about specific
  under-documented endpoints are expected and fine at this stage;
  outright generation FAILURES are not).
- Manually review the rendered schema (via `/api/schema/swagger-ui/`
  or `/api/schema/redoc/`, whichever this project's drf-spectacular
  setup exposes) for the highest-priority endpoints identified above,
  confirming they now show accurate, complete request/response shapes
  and clear descriptions.
- Confirm `python manage.py check` and the full backend test suite
  still pass after removing `drf-yasg` (no lingering import/reference
  anywhere breaks as a result of the removal).
```

---

#### Task 30.1.1.3 — Local dev setup guide update (OTP dev mode, Celery, new env vars)

```
You are working in README.md. Assume Epics 1–29 are fully merged. THIS
IS THE MOST IMPORTANT TASK IN THIS EPIC — RE-READ THIS DOCUMENT'S
HEADER BEFORE STARTING. The README's very first line currently
describes this project as a "Smart Electrical Panel E-Commerce
Platform." Fix that first, before anything else in this task.

CONTEXT
`README.md` is frozen at essentially this project's pre-transformation
state — wrong product description, a features list describing
functionality (a literal "Payment Methods" saved-cards concept) that
Epic 18's own grounding identified as conceptually obsolete since Epic
6 replaced it with real gateway redirects, a project-structure section
listing only the ORIGINAL apps with zero mention of `payments`,
`shipping`, `promotions`, or `notifications` (four entire apps this
series built), and a "Getting Started" section that almost certainly
predates Celery, Redis caching, Sentry, and every environment variable
introduced across Epics 6 through 29.

TASK
Rewrite `README.md` to accurately, completely describe what this
project actually is and how to actually run it today.

REQUIREMENTS
- **Fix the title and description first**: replace "Smart Electrical
  Panel E-Commerce Platform" with an accurate description — a Persian
  (RTL) cosmetics and skincare e-commerce platform for the Iranian
  market, built on Django REST Framework + React/Vite, with real
  Iranian payment gateway (ZarinPal/Zibal/IDPay) and shipping carrier
  integrations, SMS/email/in-app notifications, and a full admin
  dashboard.
- **Rewrite "Features at a Glance"** to reflect what this platform
  ACTUALLY does today, organized by real capability: real payment
  processing (Epic 6, not the old fake card-entry theater), real
  shipping carrier integration (Epic 7), coupons/promotions/flash sales
  (Epic 9), Persian localization including RTL layout, Jalali calendar,
  and Toman currency display (Epic 14), full-text Persian-aware search
  (Epic 12), SMS/email/in-app notifications (Epic 16), the wishlist/
  reviews/recommendation features (Epics 10/11/13), and the admin
  bulk-operations/analytics tooling (Epics 17/27) — remove the
  now-fictional "Payment Methods" saved-cards feature bullet entirely,
  replacing it with an accurate description of whatever Epic 18 Task
  18.1.1.1 actually decided to repurpose that account tab into (payment
  HISTORY, per that task's recommendation — confirm what was actually
  implemented and describe THAT).
- **Update the Tech Stack table** to include everything actually added
  since the original list: Celery + Redis (task queue, Epic 22), Redis
  caching (Epic 21, distinct from its Channels usage), Sentry (error
  tracking, Epic 26), django-axes/django-csp (security, Epic 23),
  Plausible (analytics, Epic 27), react-hook-form + zod (Epic 29),
  TanStack Query (Epic 29) — the CURRENT real stack, not the day-one
  stack.
- **Update the Project Structure section** to include `payments/`,
  `shipping/`, `promotions/`, `notifications/` alongside the original
  app list, each with an accurate one-line description matching what
  they actually do (not what the ORIGINAL apps' descriptions imply by
  parallel structure — write real, specific descriptions for each new
  app).
- **Update "Getting Started"**: this needs REAL verification, not just
  a docs edit — actually walk through setting up this project FRESH
  (or as close to fresh as practical) following the CURRENT instructions,
  and fix every point where they're stale or incomplete. At minimum,
  document: every environment variable introduced across Epics 6–29
  (payment gateway credentials, Kavenegar SMS credentials, Sentry DSN,
  Plausible config, `FRONTEND_URL`, `ADMIN_IP_ALLOWLIST`, Redis DB
  allocation — cross-reference every `.env.example`/`.env.staging.example`
  established across this series rather than re-deriving this list from
  scratch), how to run the Celery worker/beat services locally (Epic
  22 Task 22.1.1.3's docker-compose services — confirm they're
  correctly included in whatever `docker compose up` command the
  README currently documents), and — given Epic 2's OTP-based auth is
  now a PRIMARY authentication path for this platform — explicitly
  document the DEV-MODE OTP flow: `ConsoleSMSProvider` (Epic 2 Task
  2.2.1.2) prints the OTP code to the backend container's console/logs
  rather than sending a real SMS in development, and a new developer
  needs to know to look there to actually log in via the OTP flow
  locally — this is exactly the kind of non-obvious, easy-to-get-stuck-
  on local-dev detail that belongs prominently in a setup guide, not
  buried in a task prompt from many epics ago.
- **Update the SEO section**: it currently correctly mentions
  `robots.txt`, but should be updated per Epic 15's actual work — the
  domain-resolution fix, the environment-aware crawling rules, the
  sitemap coverage now including `BrandSitemap`.
- **Add a "Further Documentation" section** linking to the ADR folder
  (Task 30.1.1.1) and the runbooks (Tasks 30.1.1.4/30.1.1.5), so this
  README becomes a genuine entry point into the rest of this project's
  documentation, not an isolated, disconnected document.

ACCEPTANCE CRITERIA / TESTS
- The updated README's title/description/features list is verified
  ACCURATE against the real, current state of this codebase — not a
  copy-edit of the old text, an actual factual correction.
- Perform an ACTUAL fresh local setup following the updated
  instructions (or as close to a genuinely fresh environment as your
  context allows) and confirm every step works as documented, including
  successfully completing an OTP-based login using the documented
  console-log lookup method — the concrete, real verification this
  task's central deliverable (a setup guide that actually works)
  genuinely does.
```

---

#### Task 30.1.1.4 — Runbook: payment gateway outage procedure

```
You are working in backend/docs/runbooks/payment-gateway-outage.md
(new file, alongside whatever runbook location Epic 25 Task 25.1.1.6's
backup-restore-drill runbook established — match that same directory
convention). Assume Epics 1–29 are fully merged, including Epic 6's
full payment infrastructure, Epic 6 Task 6.3.1.3's gateway fallback
mechanism, and Epic 26 Task 26.1.1.4's payment-failure-rate alerting.

CONTEXT
Epic 6 Task 6.2.1.6 already flagged, at the time it was written, that
this exact runbook was future work: *"Add a runbook: payment gateway
outage procedure — what to do if ZarinPal is down — fallback gateway,
customer comms."* Epic 26's alerting work (Task 26.1.1.4) now means
someone WILL actually get paged when this happens — but they need
somewhere authoritative to turn to for what to DO next, written before
the incident, not improvised during it.

TASK
Write the payment-gateway-outage runbook.

REQUIREMENTS
Document, clearly and actionably:
- **How this incident is first detected**: the Epic 26 Task 26.1.1.4
  alert that fires when payment failures cross the configured
  threshold within a 10-minute window — what it actually looks like
  (a Sentry `error`-level event, per that task), and how to distinguish
  it from ordinary, expected background payment-decline noise (a
  single failure is normal; the ALERT firing at all means the
  threshold was already crossed, so if you're reading this runbook
  because of a real alert, treat it as genuinely actionable by
  definition).
- **Immediate triage steps**: check the specific gateway named in the
  alert (Epic 26 Task 26.1.1.4's context includes which gateway) —
  verify via the gateway's OWN status page/dashboard (if the vendor
  provides one) whether this is a confirmed vendor-side outage vs. a
  problem on THIS platform's own side (credentials expired,
  configuration error, network issue reaching the gateway) — document
  the specific URLs/access needed to check ZarinPal/Zibal/IDPay's own
  status, if such resources are known/available.
- **Immediate mitigation**: per Epic 6 Task 6.3.1.3's ALREADY-BUILT
  fallback mechanism — confirm whether `PaymentGatewayConfig.fallback_order`
  is ALREADY configured with a working secondary gateway (if so, the
  system may already be automatically routing around the outage with
  no manual action needed — check the Task 6.3.1.3 test/logging output
  confirming which gateway is ACTUALLY being used for new transactions
  right now) — if NOT already configured, or if the configured
  fallback is ALSO failing, document the exact admin steps to manually
  change `PaymentGatewayConfig.active_gateway` to a working alternative
  (via Django admin, per that task's editable admin registration) —
  this is the single most important, concrete action this runbook
  needs to make trivially easy to execute under pressure.
- **Customer communications**: what to do about customers CURRENTLY
  mid-checkout when the outage is discovered — since Epic 6's flow
  already correctly handles a failed/timed-out payment gracefully
  (order cancelled, stock released, customer sees Epic 6 Task 6.4.1.2's
  failure page with a "try again" option), most affected customers
  will organically retry and succeed once the gateway (or the fallback)
  is working again, WITHOUT needing direct outreach — but for a
  LONGER outage, consider whether a temporary, visible SITE-WIDE
  banner/notice (check whether this project has any existing mechanism
  for a site-wide announcement banner; if not, note that building one
  is a reasonable, small follow-up rather than something to invent
  ad-hoc during a real incident) is warranted to proactively set
  customer expectations rather than letting them discover the problem
  independently at checkout.
- **Verification the outage is resolved**: don't just wait for the
  alert to stop firing (which only means failures dropped BELOW the
  threshold, not that they've stopped entirely) — document an explicit
  verification step: attempt a real, small test transaction against
  the PRIMARY gateway specifically (using its sandbox/test mode if
  available in production per Epic 6 Task 6.2.1.6's sandbox-toggle
  design, to avoid a real charge) before switching `active_gateway`
  back from a fallback to the recovered primary.
- **Post-incident**: note what to check afterward — Epic 4's
  `StockMovement` audit log for any variants whose stock might need
  manual reconciliation if an unusually large number of orders were
  cancelled during the outage window, and confirming Epic 6 Task
  6.4.2.1's reconciliation task correctly cleaned up any transactions
  left in an ambiguous state during the outage.

ACCEPTANCE CRITERIA
The completed runbook document is this task's deliverable. Have
someone UNFAMILIAR with the payment system's internals (or, if working
solo, deliberately re-read it after a delay, trying to follow it
literally rather than from memory of having just written it) attempt
to follow the documented steps against a SIMULATED outage in staging
(e.g. deliberately misconfigure the primary gateway's credentials) and
confirm the runbook's steps are genuinely clear and correct enough to
resolve it without needing to consult anything beyond the runbook
itself.
```

---

#### Task 30.1.1.5 — Runbook: incident response (Sentry alert → triage → resolve)

```
You are working in backend/docs/runbooks/incident-response.md (new
file). Assume Epic 26 (Sentry integration) is fully merged, and Task
30.1.1.4 is already merged (this general runbook complements, rather
than duplicates, that payment-specific one).

CONTEXT
Epic 26 Task 26.1.1.4's own closing note explicitly flagged this exact
document as future work: *"a standard on-call procedure doc."* Sentry
(Epic 26 Task 26.1.1.1) now captures errors across the ENTIRE
application — not just payments — and there's no general procedure
documented for what to actually do when ANY alert fires, beyond the
payment-specific case Task 30.1.1.4 already covers.

TASK
Write a general incident-response runbook covering the full lifecycle
from alert to resolution, for any Sentry-captured issue.

REQUIREMENTS
Document, clearly and actionably:
- **Triage — assessing severity FIRST, before doing anything else**:
  a simple, explicit severity framework appropriate to THIS platform's
  actual risk areas, built from what this series' epics have already
  identified as genuinely high-stakes vs. lower-stakes failure modes:
  - **Critical (immediate action)**: anything touching checkout/
    payment (Epic 6), authentication/OTP (Epic 2), or a complete
    application outage (Epic 26 Task 26.1.1.3's uptime-monitor
    alerting territory, distinct from a Sentry error but related).
  - **High (same-day action)**: a Celery task repeatedly failing (Epic
    22's tasks — expiry sweep, payment reconciliation, shipment
    polling, notification sends — a persistent failure here degrades
    real functionality even if not immediately customer-visible),
    a spike in `NotificationLog` `FAILED` entries (Epic 16).
  - **Medium/Low**: an isolated, non-repeating error in a non-critical
    path (e.g. a single failed image-thumbnail generation, per Epic
    20) — worth fixing, not worth interrupting anything for.
- **Where to actually LOOK once an alert fires**: the real, specific
  tools this project now has, per this whole series' work — the
  Sentry event itself (Epic 26 Task 26.1.1.1, including its attached
  `request_id`, per Epic 26 Task 26.1.1.2's structured logging, which
  lets you correlate the Sentry event with the FULL structured JSON
  log line for that exact request), the `/api/health/` and
  `/api/health/live/` endpoints (Epic 22/26) for infrastructure-level
  checks, the relevant admin audit-log models built across this series
  when the issue is DATA-related rather than purely code-related
  (`StockMovement` for inventory questions, `NotificationLog` for
  delivery questions, `PaymentTransaction`/`PaymentAdminOverride` for
  payment questions) — a triager who knows WHERE to look for each
  category of problem resolves incidents dramatically faster than one
  starting from scratch every time.
- **Common false-positive patterns worth knowing about UP FRONT**
  (built from this series' own explicit design decisions) — e.g. a
  single payment failure is NORMAL (per Task 30.1.1.4's own framing;
  only the Epic 26 Task 26.1.1.4 THRESHOLD alert is genuinely
  actionable), a `SearchQueryLog` entry with zero results is expected/
  routine data (Epic 12), a `NotificationLog` `SKIPPED` entry (as
  opposed to `FAILED`) usually reflects a correct, intentional
  preference-based skip (Epic 16), not a bug — documenting these
  KNOWN-BENIGN patterns up front saves real triage time compared to
  re-investigating the same non-issue repeatedly across different
  on-call rotations.
- **Escalation**: when/how to escalate beyond whoever's initially
  triaging (a business-hours-only concern vs. a genuine 24/7 page-
  worthy issue — this depends on real staffing/on-call structure this
  document can't fully specify in the abstract, but should provide a
  clear TEMPLATE for whoever owns this project to fill in their actual
  contact/escalation chain).
- **Resolution and follow-up**: once resolved, document in the Sentry
  issue itself what was found/fixed (Sentry supports issue comments/
  resolution notes — use them), and — for anything that revealed a
  GENUINE gap (not just a one-off transient blip) — the expectation
  that a real follow-up task gets created, not just a "resolved" click
  with no lasting record of what was learned.

ACCEPTANCE CRITERIA
The completed runbook document is this task's deliverable. Cross-link
it with Task 30.1.1.4's payment-specific runbook (each should reference
the other where relevant — this general runbook's severity framework
should point to the payment-specific runbook for that category's
detailed procedure, rather than duplicating it) and with Epic 25 Task
25.1.1.6's backup-restore-drill runbook (a genuinely catastrophic
incident might require invoking THAT runbook too — this document should
make that connection explicit, not leave it to a stressed responder to
independently realize).
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 30.1.1.1 | ADR folder + backfill key decisions | ☐ |
| 30.1.1.2 | API documentation review (remove redundant drf-yasg, configure spectacular) | ☐ |
| 30.1.1.3 | Local dev setup guide update (fix README's wrong product description) | ☐ |
| 30.1.1.4 | Runbook: payment gateway outage procedure | ☐ |
| 30.1.1.5 | Runbook: incident response (Sentry alert → triage → resolve) | ☐ |

---

## This is the final epic in the master backlog.

Across Epics 1 through 30, this document series has taken the
chiz repository from its original, generic e-commerce-template
state — grounded in a real architecture review that identified fake
checkout, no localization, no real payment processing, and a dozen
other production-readiness gaps — through a complete, sequenced
transformation into a production-grade Persian cosmetics and skincare
platform for the Iranian market. Along the way, this series
consistently found that the codebase was **neither uniformly broken
nor uniformly complete**: some backlog items (Wishlist, the Address
model, order search/filtering, the entire admin analytics layer, the
blog app) turned out to already be substantially built and only needed
auditing or extending; others (real payment processing, Persian
localization, the entire notification system, code-splitting) were
genuinely absent and needed building from nothing. The single common
thread across all 30 epics has been the same discipline applied
consistently throughout: **ground every task in what the code actually
does, not what the backlog assumed it does** — and this final epic's
own central finding, a README that still describes a smart electrical
panel store, is as good a closing reminder as any of why that
discipline matters even for the parts of a project that feel like pure
formality.
