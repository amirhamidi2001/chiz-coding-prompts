# Phase 7 — Meeting Provider Abstraction + Skyroom Integration

## Implementation Prompts

### Prompt 1 — `MeetingProvider` Interface + `InternalSimulatedMeetingProvider`

```
Goal: Build the abstract meeting-provider layer for Afra, following
apps.payments.gateways' exact architectural pattern from Phase 1 (ABC +
dataclass results + an internal simulated implementation for dev/test).
No wiring into sessions yet.

Before starting, read these files completely:
1. apps/payments/gateways/base.py and internal.py (from Phase 1) — the
   exact structural pattern to replicate: ABC with dataclass results,
   a GatewayError-equivalent exception, an internal/simulated
   implementation usable in tests without any network call
2. apps/sessions/models.py — the Session model, to know what fields
   (id, teacher, start_time, end_time, session_type) are available as
   create_room's input
3. apps/sessions/services.py — create_session_from_booking's current
   fake meet_link line, to understand exactly what this abstraction
   replaces

What to build:

1. Create apps/meetings/ app structure: __init__.py, apps.py (matching
   this project's existing AppConfig style), models.py (empty for now),
   migrations/__init__.py, providers/__init__.py.

2. Create apps/meetings/providers/base.py:
   - dataclass RoomCreateResult(external_room_id: str, status: str)
   - exception MeetingProviderError(Exception)
   - abstract class MeetingProvider(ABC):
     def create_room(self, *, session: "Session") -> RoomCreateResult: ...
     def get_join_url(self, *, room_reference: str, user: "User", role: str) -> str: ...
     def end_room(self, *, room_reference: str) -> None: ...
   Use TYPE_CHECKING imports for Session/User to avoid circular imports,
   matching Phase 1's exact pattern.

3. Create apps/meetings/providers/internal.py:
   class InternalSimulatedMeetingProvider(MeetingProvider):
   - create_room: no network call, generates
     f"internal-room-{session.id}", returns
     RoomCreateResult(external_room_id=..., status="CREATED")
   - get_join_url: returns a deterministic fake URL
     f"https://meet.afra.internal.test/{room_reference}?user={user.id}&role={role}"
     (clearly a test/dev placeholder, never meant to be a real
     clickable link in production)
   - end_room: no-op
   - Add an optional constructor parameter fail_create: bool = False
     (mirroring Phase 1's force_result pattern) so tests can
     deterministically exercise the failure path of create_room by
     having it raise MeetingProviderError when set.

Files affected:
- apps/meetings/__init__.py, apps.py, migrations/__init__.py (new)
- apps/meetings/providers/__init__.py, base.py, internal.py (new)

Then write tests: tests/meetings/test_providers_internal.py covering
create_room's normal and fail_create=True paths, get_join_url's output
shape, and end_room's no-op success.

Acceptance Criteria:
- pytest tests/meetings/test_providers_internal.py -v passes completely
- git status shows only the new files listed

Verification Steps:
1. pytest tests/meetings/test_providers_internal.py -v
2. python -c "from apps.meetings.providers.base import MeetingProvider; print(MeetingProvider.__abstractmethods__)"
3. git diff --stat
```

### Prompt 2 — پیاده‌سازی `SkyroomMeetingProvider`

```
Goal: Real implementation of MeetingProvider for Skyroom, behind the
same interface from Prompt 1.

Before starting:
1. Read apps/meetings/providers/base.py, internal.py
2. Read apps/payments/gateways/zarinpal.py (Phase 1) as the closest
   structural reference for "a real third-party API-backed provider
   implementation with explicit timeouts, structured error handling,
   and logging without leaking sensitive data"
3. Consult Skyroom's actual API documentation for room creation and
   per-user join-token/URL issuance endpoints (Skyroom's API
   typically involves creating a room and then issuing a
   participant-specific access token or URL — confirm the exact
   current endpoint shapes rather than assuming; if you cannot access
   live documentation, implement against the most standard/likely
   shape — room creation returns a room id, and a separate
   token-issuance call takes the room id + a participant identifier
   and role, returning a signed join URL — and clearly mark any
   assumption with a comment for a human to verify against Skyroom's
   actual current docs before this goes to production)

What to build:

Create apps/meetings/providers/skyroom.py:

class SkyroomMeetingProvider(MeetingProvider):
- Constructor takes api_key: str, api_base_url: str (from
  settings.SKYROOM_API_KEY / settings.SKYROOM_API_BASE_URL, passed as
  constructor params, not read directly from settings inside the
  class, for testability)
- create_room: POST to Skyroom's room-creation endpoint with a room
  name derived from the session (e.g. f"afra-session-{session.id}"),
  explicit timeout, on success return RoomCreateResult with Skyroom's
  returned room id; on any HTTP/network error, raise
  MeetingProviderError (never let requests.RequestException leak
  directly), logging the attempt (room id being created, HTTP status)
  without logging the API key or other secrets
- get_join_url: POST to Skyroom's token/join-url issuance endpoint
  with the room_reference, a participant identifier (user.id or
  user.email — pick whichever Skyroom's API actually expects) and
  role (map this project's "teacher"/"student" role strings to
  whatever role concept Skyroom's API uses, e.g.
  "presenter"/"viewer" — document this mapping explicitly since it's
  an assumption-prone integration point), explicit timeout, returns
  the join URL string on success, raises MeetingProviderError on
  failure
- end_room: if Skyroom's API has an explicit room-closing endpoint,
  call it; if not, implement this as a documented no-op (check
  Skyroom's docs/your best available information and note the
  decision clearly in the method's docstring either way, since this
  was flagged as a real risk in this Phase's architecture: leaving
  rooms open unnecessarily has cost/security implications if the API
  does support explicit closing and this method silently no-ops when
  it shouldn't)

Files affected:
- apps/meetings/providers/skyroom.py (new)
- config/settings/base.py (add SKYROOM_API_KEY, SKYROOM_API_BASE_URL,
  following the exact pattern of ZARINPAL_* settings from Phase 1)
- requirements/base.txt (only if a new HTTP library is genuinely
  needed — check what's already available first, per Phase 1's
  precedent)

Then write tests: tests/meetings/test_providers_skyroom.py, mocking
every HTTP call (no real network access allowed), covering:
- create_room success and HTTP-failure paths
- get_join_url success and HTTP-failure paths, confirming the
  role-mapping is applied correctly
- end_room's behavior (whichever was chosen)
- Confirm no API key or secret value ever appears in a logged message
  (assert on captured log output)

Acceptance Criteria:
- pytest tests/meetings/test_providers_skyroom.py -v passes completely,
  no real network calls
- Every HTTP call has an explicit timeout (manual grep confirmation)

Verification Steps:
1. pytest tests/meetings/test_providers_skyroom.py -v
2. grep -n "requests\.\(get\|post\)" apps/meetings/providers/skyroom.py
   (manually confirm every call has timeout=...)
3. pytest tests/meetings/ -v
4. git diff --stat
```

### Prompt 3 — Factory + تنظیمات

```
Goal: A single selection point for the active MeetingProvider based on
settings.MEETING_PROVIDER, mirroring Phase 1's gateway factory exactly.

Before starting, read apps/payments/gateways/factory.py (Phase 1) and
apps/meetings/providers/base.py, internal.py, skyroom.py.

What to build:

Create apps/meetings/providers/factory.py:
def get_meeting_provider() -> MeetingProvider:
    Reads settings.MEETING_PROVIDER ("internal" or "skyroom"), returns
    the corresponding instance, raises ImproperlyConfigured otherwise —
    exact structural mirror of get_gateway_provider from Phase 1.

In config/settings/base.py, add:
MEETING_PROVIDER: str = config("MEETING_PROVIDER", default="internal")
MEETINGS_JOIN_WINDOW_MINUTES_BEFORE: int = config(
    "MEETINGS_JOIN_WINDOW_MINUTES_BEFORE", default=15, cast=int
)
With a comment cross-referencing apps.sessions.services._REMINDER_LEAD_TIME
(30 minutes) and explicitly noting this value should not exceed it, so
the reminder email doesn't arrive before the join window opens.

Files affected:
- apps/meetings/providers/factory.py (new)
- config/settings/base.py

Then write tests: tests/meetings/test_provider_factory.py, mirroring
Phase 1's test_gateway_factory.py exactly (internal/skyroom/invalid
cases).

Acceptance Criteria:
- pytest tests/meetings/test_provider_factory.py -v passes completely

Verification Steps:
1. pytest tests/meetings/test_provider_factory.py -v
2. python manage.py check
3. git diff --stat
```

### Prompt 4 — مدل `Meeting` + سیم‌کشی در `create_session_from_booking` + Retry Task

```
Goal: Add the Meeting/MeetingEvent models, replace the fake meet_link
assignment in create_session_from_booking with a real
create_meeting_for_session call, and add a Celery retry task for
failed room creation — ensuring a room-creation failure never blocks
booking acceptance.

Before starting, read these files completely:
1. apps/sessions/services.py — create_session_from_booking in full
2. apps/payments/models.py — PaymentEvent, as the append-only pattern
   MeetingEvent mirrors
3. apps/meetings/providers/base.py, factory.py (from previous prompts)
4. apps/bookings/tasks.py — for this project's exact lazy-import +
   try/except + logger.exception Celery task convention

What to build:

a) apps/meetings/models.py:

   class Meeting(models.Model):
       id = UUIDField(primary_key=True, default=uuid4, editable=False)
       session = OneToOneField("sessions.Session", on_delete=CASCADE, related_name="meeting")
       provider = CharField(max_length=15, choices=[("INTERNAL", "Internal"), ("SKYROOM", "Skyroom")])
       external_room_id = CharField(max_length=255, blank=True)
       STATUS = [("CREATED", "Created"), ("FAILED", "Failed"), ("ENDED", "Ended")]
       status = CharField(max_length=10, choices=STATUS, default="FAILED", db_index=True)
       failure_reason = TextField(blank=True)
       created_at = DateTimeField(auto_now_add=True)
       ended_at = DateTimeField(null=True, blank=True)
       (Deliberately defaults to "FAILED" rather than "CREATED" — a
       Meeting row only ever gets an explicit "CREATED" status after a
       genuinely successful provider.create_room() call; document this
       "pessimistic default" choice in the class docstring, mirroring
       how Payment/PaymentEvent's defaults are chosen deliberately
       elsewhere in this project.)

   class MeetingEvent(models.Model):
       (Append-only audit log, exact structural mirror of PaymentEvent:
       meeting FK, from_status, to_status, reason, created_at.)

   Run python manage.py makemigrations meetings, review/rename cleanly.

b) apps/meetings/services.py:

   def create_meeting_for_session(*, session: "Session") -> "Meeting":
       """Create the Meeting row for a just-created Session, attempting
       a real room via the configured provider. A provider failure
       here is deliberately non-fatal to the caller — see this
       function's docstring cross-reference to
       apps.sessions.services.create_session_from_booking, which must
       keep succeeding even if this fails, since a booking's
       acceptance and payment flow shouldn't be gated on third-party
       video infrastructure being momentarily available."""
       provider = get_meeting_provider()
       meeting = Meeting.objects.create(session=session, provider=settings.MEETING_PROVIDER.upper())
       try:
           result = provider.create_room(session=session)
           meeting.external_room_id = result.external_room_id
           meeting.status = "CREATED"
           meeting.save(update_fields=["external_room_id", "status"])
       except MeetingProviderError as exc:
           meeting.failure_reason = str(exc)
           meeting.save(update_fields=["failure_reason"])
           logger.warning("create_meeting_for_session failed for session=%s: %s", session.id, exc)
           from apps.meetings.tasks import retry_failed_room_creation
           transaction.on_commit(
               lambda: retry_failed_room_creation.apply_async(
                   args=[str(meeting.id)], countdown=60
               )
           )
       return meeting

c) apps/meetings/tasks.py:

   @shared_task(bind=True, max_retries=5, default_retry_delay=60)
   def retry_failed_room_creation(self, meeting_id: str) -> None:
       Fetch the Meeting (return if not found, logged); if status is
       already "CREATED" (a previous retry or manual admin action
       already fixed it), return early (idempotent). Otherwise attempt
       provider.create_room again using the session it's attached to,
       following the exact same success/failure handling as
       create_meeting_for_session's try/except (extract this into a
       small shared helper if it avoids duplicating the same 6-8 lines
       twice — your call, but don't let the two diverge silently). On
       repeated failure past max_retries, log a final error (this is
       now something requiring human/admin attention — no further
       automated action in this Phase; Phase 8's admin work could later
       surface this, but don't build that here).

d) apps/sessions/services.py, in create_session_from_booking:
   Remove entirely:
       session.meet_link = f"https://meet.afra.example.com/{session.id}"
       session.save(update_fields=["enrolled_count", "meet_link"])
   Replace with:
       session.save(update_fields=["enrolled_count"])
       from apps.meetings.services import create_meeting_for_session
       create_meeting_for_session(session=session)
   (Keep this call inside the same transaction.atomic block that
   already wraps session creation — since create_meeting_for_session
   itself is designed to never raise on a provider failure, per its
   docstring above, this is safe to call synchronously here without
   risking the whole booking-acceptance transaction failing due to a
   third-party outage.)

e) config/settings/base.py: add "apps.meetings" to INSTALLED_APPS.

Files affected:
- apps/meetings/models.py, migrations/0001_initial.py (new)
- apps/meetings/services.py, tasks.py (new)
- apps/sessions/services.py
- config/settings/base.py

Then write tests:
- tests/meetings/test_services.py: create_meeting_for_session happy
  path (status="CREATED"); failure path (mock provider to raise,
  confirm Meeting.status stays "FAILED", failure_reason is set, and
  retry_failed_room_creation is dispatched via on_commit)
- tests/meetings/test_tasks.py: retry succeeding after a prior
  failure updates status to "CREATED"; retry that's already CREATED is
  a no-op (idempotency)
- tests/sessions/test_services.py: create_session_from_booking still
  succeeds and returns a valid Session even when the meeting provider
  is mocked to fail entirely (this is the single most important test
  in this prompt — it proves the non-fatal design actually holds)

Acceptance Criteria:
- pytest tests/meetings/ tests/sessions/ -v passes completely
- The non-fatal-failure test for create_session_from_booking passes
- python manage.py makemigrations --check --dry-run is empty

Verification Steps:
1. pytest tests/meetings/ tests/sessions/ -v
2. python manage.py makemigrations --check --dry-run
3. python manage.py migrate
4. git diff --stat
```

### Prompt 5 — Join-URL Endpoint با کنترل دسترسی زمانی/نقشی + حذف `meet_link` از سریالایزر

```
Goal: Build the on-demand join-URL endpoint with strict time-window and
participant-role access control, and remove the static meet_link field
from the public API surface entirely (closing the security gap
identified in this Phase's architecture: a permanent, unguarded
meeting link).

Before starting, read these files completely:
1. apps/sessions/serializers.py — SessionSerializer's current fields
   list including meet_link
2. apps/sessions/models.py — Session, SessionEnrollment (for
   determining "is this user a real participant")
3. apps/sessions/services.py — report_no_show, as the closest existing
   precedent for a time-window + role-validated action on a Session
4. apps/meetings/services.py, models.py (from Prompt 4)
5. config/settings/base.py — MEETINGS_JOIN_WINDOW_MINUTES_BEFORE
   (Prompt 3)

What to build:

a) apps/sessions/serializers.py: remove "meet_link" from
   SessionSerializer's Meta.fields (and read_only_fields, which
   currently mirrors fields).

b) apps/meetings/services.py, add:

   def get_join_url_for_session(*, session_id: str, user: "User") -> str:
       """Issue a fresh, participant-specific join URL — never reads
       or returns a stored URL, always calls the provider live.

       Eligibility (all must hold):
       - session exists and has a Meeting with status in
         ("CREATED", "FAILED") — FAILED is included deliberately, so a
         best-effort retry (see below) can still succeed right before
         a session if the earlier attempt failed transiently
       - user is either session.teacher, or has an active
         SessionEnrollment for this session
       - session.status is SCHEDULED or ONGOING
       - now is within
         [session.start_time - MEETINGS_JOIN_WINDOW_MINUTES_BEFORE, session.end_time]
       """
       Fetch session with select_related("meeting"); NotFoundError if
       missing. Determine role: "teacher" if user == session.teacher,
       else check SessionEnrollment.objects.filter(session=session,
       student=user).exists() -> role="student", else
       PermissionDeniedError. Check session.status (ApplicationError
       code "session_not_joinable" otherwise). Check the time window
       (ApplicationError code "join_window_not_open" if too early,
       "join_window_closed" if too late — two distinct codes, since
       the user-facing guidance differs).

       If meeting.status == "FAILED": attempt one best-effort
       synchronous retry of provider.create_room(session=session)
       right here (not via Celery, since the user is actively waiting
       — a synchronous attempt with the provider's own request timeout
       is appropriate); on success, update the Meeting row inline and
       proceed; on failure, raise ApplicationError(code=
       "meeting_unavailable", message explaining support should be
       contacted) rather than trying to hand back a broken URL.

       On success, call
       provider.get_join_url(room_reference=meeting.external_room_id,
       user=user, role=role) and return it.

c) apps/meetings/views.py:

   class SessionJoinUrlView(APIView):
       permission_classes = [IsAuthenticated]
       def get(self, request, session_id):
           join_url = services.get_join_url_for_session(
               session_id=session_id, user=request.user,
           )
           return Response({"join_url": join_url})

d) apps/meetings/urls.py + wire into the project's root URL config
   under something like /api/sessions/<uuid:session_id>/meeting/join-url/
   (confirm whether this should live under apps/sessions/urls.py
   instead, for URL-namespace consistency with how the rest of the
   session-related API is organized — check the existing convention
   and follow it, matching whichever cross-app URL-registration
   approach Phase 3's Prompt 5 already decided on for a similar
   cross-app URL case, for consistency across the project).

Files affected:
- apps/sessions/serializers.py
- apps/meetings/services.py
- apps/meetings/views.py, urls.py (new)
- config/urls.py (or apps/sessions/urls.py, per your URL-organization
  decision above)

Then write tests: tests/meetings/test_join_url.py covering:
- Teacher and enrolled student both get a valid join_url within the
  window
- An unrelated authenticated user gets PermissionDeniedError
- Too early (before the window opens) gets ApplicationError code
  "join_window_not_open"
- After session.end_time gets "join_window_closed"
- A session with a FAILED meeting successfully retries inline and
  returns a valid URL (mock the provider to fail once then succeed)
- A session with a FAILED meeting that keeps failing returns
  ApplicationError code "meeting_unavailable", not a broken URL
- GET /api/sessions/{id}/ (the existing session detail endpoint) no
  longer includes "meet_link" in its response body at all

Acceptance Criteria:
- pytest tests/meetings/ tests/sessions/ -v passes completely
- The response shape of GET /api/sessions/{id}/ is confirmed to no
  longer contain meet_link

Verification Steps:
1. pytest tests/meetings/test_join_url.py -v
2. pytest tests/sessions/ -v
3. python manage.py runserver, curl a session detail endpoint,
   manually confirm meet_link is absent from the JSON
4. git diff --stat
```

### Prompt 6 — Django Admin برای `Meeting`

```
Goal: Add Django Admin visibility into Meeting status, with a manual
"retry room creation" action for support staff to use when a Meeting
is stuck in FAILED status.

Before starting, read apps/payments/admin.py (for this project's
established single-object admin-action-with-ApplicationError-catching
pattern) and apps/meetings/models.py, services.py.

What to build:

In apps/meetings/admin.py:

@admin.register(Meeting)
class MeetingAdmin(admin.ModelAdmin):
    list_display = ["id", "session", "provider", "status", "created_at"]
    list_filter = ["provider", "status"]
    search_fields = ["session__id", "external_room_id"]
    readonly_fields = ["session", "provider", "external_room_id", "created_at", "ended_at"]
    actions = ["retry_room_creation"]

    @admin.action(description="Retry room creation for selected meetings")
    def retry_room_creation(self, request, queryset):
        for meeting in queryset.filter(status="FAILED"):
            from apps.meetings.tasks import retry_failed_room_creation
            retry_failed_room_creation.delay(str(meeting.id))
        self.message_user(request, "Retry queued for selected failed meetings.")

Files affected:
- apps/meetings/admin.py (new)

Acceptance Criteria:
- python manage.py check reports no errors
- Manual admin check: a FAILED meeting can be selected and retried via
  the admin action

Verification Steps:
1. python manage.py check
2. python manage.py runserver, verify the admin action in /admin/
3. git diff --stat
```

### Prompt 7 — فرانت‌اند: `MeetLinkButton` غیرهمزمان

```
Goal: Convert MeetLinkButton from a static-prop link into an async
fetch-on-click component matching the new backend join-url endpoint,
preserving its existing enabled/disabled gating behavior.

Before starting, read these files completely:
1. src/features/sessions/components/MeetLinkButton.tsx — the full
   current implementation
2. src/features/sessions/pages/SessionDetailPage.tsx — both call sites
   of MeetLinkButton
3. src/features/sessions/types/session.types.ts — the Session type's
   current meet_link field
4. src/features/sessions/api/sessions.api.ts — the existing API call
   pattern to match

What to build:

1. In session.types.ts: remove meet_link from the Session interface
   (it's no longer in the backend's response, per Phase 7's backend
   changes).

2. In sessions.api.ts, add:
   getSessionJoinUrl(sessionId: string): Promise<{ join_url: string }>
   (GET /api/sessions/{sessionId}/meeting/join-url/ or wherever Prompt
   5's backend actually registered the route — confirm the exact path)

3. Rewrite MeetLinkButton.tsx:
   interface MeetLinkButtonProps {
     sessionId: string;
     sessionStatus: SessionStatus;
   }
   (meetLink prop removed entirely — the component now fetches on
   demand)
   - isStatusReady logic unchanged (SCHEDULED or ONGOING)
   - On click: call getSessionJoinUrl via a mutation (matching this
     project's established useMutation pattern from earlier phases),
     show a loading state on the button while the request is in
     flight, then window.open(result.join_url, "_blank",
     "noopener,noreferrer") on success
   - On error: surface the backend's error via
     src/shared/lib/apiErrorMessage.ts (from Phase 5's Prompt 9 — this
     is exactly the kind of ApplicationError-code-driven message this
     helper was built for: "join_window_not_open",
     "join_window_closed", "meeting_unavailable" all need Persian
     user-facing text; add these three codes to
     src/shared/i18n/locales/fa/errors.json if they aren't already
     there from Phase 5, since they're new codes introduced in this
     Phase) via a toast, and re-enable the button (don't leave it
     stuck in a loading state after a failure)
   - The button itself stays enabled/clickable whenever isStatusReady
     is true (unlike before, we no longer know client-side whether a
     link "exists" — that's now determined server-side at click time,
     including the time-window check) — update the disabled logic and
     tooltip copy accordingly: instead of "Meeting link not available
     yet" as a static pre-computed disabled state, the button is
     always enabled when isStatusReady, and an unavailable/too-early
     state is now communicated via the error toast after a click
     attempt, not a disabled button beforehand. (This is a deliberate
     UX shift — document it in a comment: the button no longer
     pre-emptively knows the exact join window boundaries client-side,
     since that logic now correctly lives only on the backend.)

4. Update both call sites in SessionDetailPage.tsx: replace
   meetLink={session.meet_link} with sessionId={session.id} for both
   MeetLinkButton usages.

Files affected:
- src/features/sessions/types/session.types.ts
- src/features/sessions/api/sessions.api.ts
- src/features/sessions/components/MeetLinkButton.tsx
- src/features/sessions/pages/SessionDetailPage.tsx
- src/shared/i18n/locales/fa/errors.json (only if the three new error
  codes need adding — check first)

Then update src/features/sessions/components/__tests__/MeetLinkButton.test.tsx
completely (this component's contract changed fundamentally):
- Clicking the button when isStatusReady calls getSessionJoinUrl and
  opens the returned URL (mock window.open)
- Clicking shows a loading state while the request is in flight
- A failed request (e.g. mocked "join_window_not_open" error) shows
  the correct Persian toast message and the button returns to a
  clickable (not stuck-loading) state
- The button is disabled entirely when isStatusReady is false
  (unchanged from before)

Also update src/test/mocks/handlers/sessions.handlers.ts with a handler
for the new join-url endpoint (both success and error-case variants).

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds
- No remaining reference to session.meet_link anywhere in the frontend
  (grep -rn "meet_link" src/ returns nothing)

Verification Steps:
1. npx vitest run
2. grep -rn "meet_link" src/
3. npm run build
4. Manual check: with backend MEETING_PROVIDER=internal, view a
   SCHEDULED session's detail page within the join window, click "Join
   Session," confirm a new tab opens with the internal simulated URL
5. git diff --stat
```

### Prompt 8 — بررسی نهایی: تست یکپارچه End-to-End + پاکسازی

```
Goal: Final review confirming the complete meeting lifecycle works end
to end, and that Session.meet_link is fully retired from active use
(while its column remains for safe rollback, per this project's
established soft-deprecation pattern).

Before starting, review the diffs from Prompts 1-7.

What to build:

1. grep -rn "meet_link" afra-backend afra-frontend --include=*.py
   --include=*.ts --include=*.tsx | grep -v migrations
   Confirm every remaining hit is either: the deprecated model field
   definition itself (with its DEPRECATED comment from Prompt 4), or a
   test fixture/factory that still references it harmlessly. Anything
   else is a forgotten call site — fix it.

2. Add tests/meetings/test_e2e_full_lifecycle.py covering, using real
   service functions:
   - Create a booking, accept it (via apps.bookings.services,
     following this project's established test fixture pattern) →
     confirm a Session and a Meeting (status="CREATED", using the
     internal provider) both exist
   - Attempt get_join_url_for_session before the join window opens →
     ApplicationError "join_window_not_open"
   - Advance time (freeze time or adjust the session's start_time in
     the test, matching how this project's other time-window tests —
     e.g. Phase 3's review-window test — already handle this) into the
     valid join window → get_join_url_for_session succeeds for both
     teacher and student
   - Advance time past session.end_time → "join_window_closed"
   - Simulate a FAILED meeting (mock the provider) and confirm the
     inline best-effort retry path in get_join_url_for_session works
     as designed

3. Confirm CI configuration: check .github/workflows/backend-ci.yml
   for any need to set MEETING_PROVIDER=internal explicitly for test
   runs (matching how PAYMENT_GATEWAY=internal was handled in Phase 1,
   if it was) — fix if missing.

Files affected:
- tests/meetings/test_e2e_full_lifecycle.py (new)
- .github/workflows/backend-ci.yml (only if a fix was needed)
- any forgotten meet_link reference found in step 1

Acceptance Criteria:
- pytest -x (full backend suite) passes, including the new e2e test
- npx vitest run (full frontend suite) passes
- The grep from step 1 shows only the deliberately-kept deprecated
  field definition and harmless test fixtures — report the full list

Verification Steps:
1. pytest tests/meetings/test_e2e_full_lifecycle.py -v
2. pytest -x -v (full backend suite)
3. npx vitest run (full frontend suite)
4. grep -rn "meet_link" afra-backend afra-frontend --include=*.py
   --include=*.ts --include=*.tsx | grep -v migrations
   (paste this output in your final summary)
5. git diff --stat (final summary of this entire Phase)
```
