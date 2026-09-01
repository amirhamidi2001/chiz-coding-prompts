# Phase 6 — Jalali Calendar (Frontend)

## Implementation Prompts

### Prompt 1 — نصب `jalaali-js` + گسترش `shared/lib/date.ts` با توابع Jalali (بدون تغییر مصرف‌کننده‌ها)

```
Goal: Add jalaali-js as a dependency and extend src/shared/lib/date.ts
with Jalali-aware formatting and conversion functions, following this
file's own established documentation style and single-source-of-truth
contract. No consumer file is changed in this prompt — pure addition.

Before starting, read src/shared/lib/date.ts completely, especially its
module-level docstring establishing it as "the ONLY place in the
codebase allowed to know [Tehran timezone] and construct/reinterpret an
ISO datetime string by hand" — every function you add here must respect
and extend that same contract, not create a parallel one.

What to build:

1. Install jalaali-js (a small, dependency-free, well-established pure
   Gregorian<->Jalali conversion library — no UI, no timezone logic of
   its own, which is exactly why it's the right tool here: it composes
   cleanly with this file's existing Tehran-timezone handling rather
   than duplicating or conflicting with it). Add it to package.json's
   dependencies.

2. Create src/shared/lib/jalaliMonthNames.ts:
   export const JALALI_MONTH_NAMES = [
     "فروردین", "اردیبهشت", "خرداد", "تیر", "مرداد", "شهریور",
     "مهر", "آبان", "آذر", "دی", "بهمن", "اسفند",
   ] as const;
   export const JALALI_WEEKDAY_NAMES_SHORT = [
     "ش", "ی", "د", "س", "چ", "پ", "ج",
   ] as const; // Saturday-first, matching the Jalali week's actual start
   export const JALALI_WEEKDAY_NAMES_FULL = [
     "شنبه", "یکشنبه", "دوشنبه", "سه‌شنبه", "چهارشنبه", "پنجشنبه", "جمعه",
   ] as const;
   (Keep these as plain exported constants for now, not yet routed
   through i18next — Prompt 2, which builds JalaliCalendar, decides
   whether these need to go through the translation system per Phase
   5's convention or can stay as fixed constants since Jalali month
   names in Persian don't vary by any locale switch this project has;
   document that reasoning wherever it's ultimately decided.)

3. In src/shared/lib/date.ts, add (importing jalaali-js at the top,
   alongside the existing date-fns/date-fns-tz imports):

   export function gregorianDateToJalali(date: Date): { jy: number; jm: number; jd: number } {
     const { jy, jm, jd } = jalaali.toJalaali(date.getFullYear(), date.getMonth() + 1, date.getDate());
     return { jy, jm, jd };
   }
   (Note: this deliberately reads the Gregorian date via
   getFullYear()/getMonth()/getDate() — this is safe here specifically
   because, per this file's existing established pattern in
   zonedTimeToUtcIso's design and TimeSlotPicker's
   buildIsoFromDateAndTime, callers of this function are expected to
   already be working with a Date object whose Y/M/D fields represent
   pure calendar-day numbers, not a real moment in the viewing device's
   timezone — document this precondition explicitly in this function's
   docstring, cross-referencing the same caution already present
   elsewhere in this file, so future readers don't accidentally call it
   on a genuine timestamp Date.)

   export function jalaliToGregorianDate(jy: number, jm: number, jd: number): Date {
     const { gy, gm, gd } = jalaali.toGregorian(jy, jm, jd);
     return new Date(gy, gm - 1, gd);
   }
   (Constructs a local Date at midnight for the given calendar day —
   mirroring how react-day-picker's existing Calendar component already
   hands Date objects to callers today, so this is a drop-in-compatible
   return shape.)

   export function formatJalaliDate(isoString: string, timezone: string = APP_TIMEZONE): string {
     const date = safeParse(isoString);
     if (!date) return "";
     try {
       const zoned = toZonedTime(date, timezone);
       const { jy, jm, jd } = gregorianDateToJalali(zoned);
       return `${jd} ${JALALI_MONTH_NAMES[jm - 1]} ${jy}`;
     } catch {
       return "";
     }
   }

   export function formatJalaliDateTime(isoString: string, timezone: string = APP_TIMEZONE): string {
     const date = safeParse(isoString);
     if (!date) return "";
     try {
       const zoned = toZonedTime(date, timezone);
       const { jy, jm, jd } = gregorianDateToJalali(zoned);
       const weekdayName = JALALI_WEEKDAY_NAMES_FULL[<compute the
         correct Jalali weekday index for `zoned`, being careful that
         JavaScript's own Date.getDay() returns a Sunday-first index
         (0=Sunday) which must be remapped to this file's
         Saturday-first JALALI_WEEKDAY_NAMES_FULL array order — write a
         small local helper for this remapping rather than getting the
         off-by-some-days math wrong inline>];
       const time = formatTz(zoned, "HH:mm", { timeZone: timezone });
       return `${weekdayName}، ${jd} ${JALALI_MONTH_NAMES[jm - 1]} · ${time}`;
     } catch {
       return "";
     }
   }
   (Import JALALI_MONTH_NAMES/JALALI_WEEKDAY_NAMES_FULL from the new
   jalaliMonthNames.ts file created in step 2.)

4. Fix formatRelativeTime to use Persian: import { faIR } from
   "date-fns/locale" (check the exact export name/path for this
   project's installed date-fns version — it may be `faIR` or `fa-IR`
   depending on version; verify against node_modules/date-fns/locale
   before assuming), and pass { addSuffix: true, locale: faIR } to
   formatDistanceToNow.

Do NOT modify formatDate, formatDateTime, formatTime, or
zonedTimeToUtcIso — they remain exactly as they are (still correctly
used internally by the new Jalali functions above, and still the right
choice for any hypothetical non-Jalali-display need, if one exists,
which Prompt 6 will confirm one way or the other).

Files affected:
- package.json (+ package-lock.json)
- src/shared/lib/jalaliMonthNames.ts (new)
- src/shared/lib/date.ts

Then write tests: extend src/shared/lib/__tests__/date.test.ts (or
create it if this project doesn't already have one — check first) with:
- gregorianDateToJalali/jalaliToGregorianDate round-trip correctly for
  at least 5 known date pairs, including one Jalali leap year (e.g.
  1403 is a Jalali leap year — verify this against jalaali-js's own
  isLeapJalaaliYear if unsure, and pick a genuinely correct example,
  don't guess)
- formatJalaliDate produces the correct Persian month name and day/year
  for a known ISO string
- formatJalaliDateTime produces the correct weekday name (spot-check
  against a real calendar for the specific date you use, to catch any
  off-by-one in the Saturday-first remapping)
- formatRelativeTime now returns Persian text (e.g. contains "پیش" for
  "ago", matching whatever faIR's actual output format is — check the
  real output rather than assuming exact wording)

Acceptance Criteria:
- npx vitest run src/shared/lib/__tests__/date.test.ts passes
  completely, including the leap-year round-trip test
- No other file in the repository has been modified

Verification Steps:
1. npx vitest run src/shared/lib/__tests__/date.test.ts
2. npm run build (confirm the new jalaali-js dependency resolves/builds
   cleanly)
3. git status (confirm changes limited to the files listed above)
4. git diff --stat
```

### Prompt 2 — ساخت کامپوننت `JalaliCalendar`

```
Goal: Build a new JalaliCalendar component that renders a correct
Jalali calendar grid (using the Prompt 1 conversion functions for the
underlying date math, not just relabeling), while visually and
structurally mirroring the existing shared/components/ui/calendar.tsx
as closely as possible — same styling classes, same RTL handling
pattern, same external Date-in/Date-out API contract — so every
existing consumer can eventually swap to it with an import-only change.

Before starting, read these files completely:
1. src/shared/components/ui/calendar.tsx — the full file, in complete
   detail: its exact Tailwind classNames for every part
   (root/months/month/nav/button_previous/button_next/month_caption/
   month_grid/weekdays/weekday/week/day/range_*/today/outside/disabled),
   its RTL chevron-rotation classes
   (rtl:**:[.rdp-button_next>svg]:rotate-180 and the previous
   equivalent), and its exact external props shape (value/onChange
   pattern — check how react-day-picker's DayPicker consumes/produces
   selected dates, likely via a `selected`/`onSelect` prop pair, to
   know exactly what shape JalaliCalendar's own props need to match for
   drop-in compatibility with existing consumers)
2. src/features/bookings/components/TimeSlotPicker.tsx — the full
   file, to understand exactly how the existing Calendar component is
   currently used (selected/onSelect wiring, the disabled prop function
   isDateDisabled) so JalaliCalendar's API can support the exact same
   usage pattern
3. src/shared/lib/date.ts and jalaliMonthNames.ts (from Prompt 1) — the
   exact functions/constants you'll use
4. src/shared/components/ui/button.tsx, button-variants.ts — since
   calendar.tsx reuses this project's Button component for day cells,
   follow the same reuse pattern

What to build:

Create src/shared/components/ui/JalaliCalendar.tsx:

1. Define the props interface to match calendar.tsx's existing
   external contract as closely as possible for the "single date
   selection" use case (the only mode this project's existing Calendar
   usages appear to need — confirm this by checking every actual call
   site found via grep -rln "Calendar" src/features/, and if any use
   `mode="range"` or multi-select, note this as a finding and extend
   the scope accordingly; otherwise build only single-select support to
   avoid over-engineering for an unused case):
   interface JalaliCalendarProps {
     selected?: Date;
     onSelect?: (date: Date | undefined) => void;
     disabled?: (date: Date) => boolean;
     className?: string;
   }

2. Internal state: track the currently-displayed Jalali month/year
   (jy, jm) as component state, initialized from `selected` (converted
   via gregorianDateToJalali) if provided, otherwise from today's date
   converted the same way.

3. Grid generation: for the displayed (jy, jm), compute:
   - The Jalali month's length via jalaali.jalaaliMonthLength(jy, jm)
     (never hardcode 29/30/31 — always call this, since it correctly
     accounts for Esfand's leap-year variability)
   - The weekday of the 1st of the month, converted to this project's
     Saturday-first index (reuse or extract the same remapping helper
     built in Prompt 1's formatJalaliDateTime, rather than
     re-implementing it — if it's not already exported from date.ts,
     go back and export it as a small shared helper, e.g.
     `gregorianDayToJalaliWeekdayIndex`, rather than duplicating the
     math here)
   - Build a grid of day cells (including leading blank cells for the
     days-of-week offset before the 1st, matching how
     react-day-picker's showOutsideDays visually pads a month grid,
     though outside-month days can be simple blanks here rather than
     full outside-day cells, unless the existing Calendar's visual
     design specifically relies on showing real outside-month days —
     check calendar.tsx's `outside` className usage and decide whether
     to replicate that level of polish or simplify; prefer replicating
     it for visual consistency with the rest of the app, converting the
     adjacent Jalali months' trailing/leading days the same way)

4. For each day cell, convert its (jy, jm, jd) to a Gregorian Date via
   jalaliToGregorianDate (from Prompt 1) to: (a) compare against
   `selected` (for highlighting), (b) call the `disabled` prop function
   (which, per the existing TimeSlotPicker contract, operates on
   Gregorian Date objects — do not change this contract), and (c) pass
   to `onSelect` when clicked.

5. Month navigation (previous/next buttons): decrement/increment jm
   (rolling jy at the Farvardin/Esfand boundary), re-render the grid —
   do NOT use any Gregorian month-arithmetic here (no date-fns
   addMonths/subMonths, which operate on the Gregorian calendar and
   would produce incorrect Jalali month boundaries).

6. Visual structure and classNames: copy calendar.tsx's exact
   className strings for each structural part (root, months, month,
   nav, button_previous, button_next, month_caption, month_grid,
   weekdays, weekday, week, day, today, outside, disabled), including
   its exact RTL chevron-rotation utility classes
   (String.raw`rtl:**:[...]:rotate-180`) — reuse this project's
   existing Button/buttonVariants for the nav buttons and day cells,
   exactly as calendar.tsx does, so JalaliCalendar looks visually
   identical to the existing Calendar in every way except its actual
   date system.

7. Month caption / weekday header labels: use JALALI_MONTH_NAMES and
   JALALI_WEEKDAY_NAMES_SHORT from Prompt 1's jalaliMonthNames.ts.

8. "Today" detection: compare each cell's Jalali (jy, jm, jd) against
   today's date converted via gregorianDateToJalali(new Date()), not a
   Gregorian comparison.

Files affected:
- src/shared/components/ui/JalaliCalendar.tsx (new)
- src/shared/lib/date.ts (only if the weekday-remapping helper needs to
  be extracted/exported per step 3 above — report whether this was
  needed)

Then write tests: src/shared/components/ui/__tests__/JalaliCalendar.test.tsx
covering:
- Renders the correct Jalali month name and year in the caption for a
  given `selected` date
- Renders the correct number of day cells for a month with 29 days
  (Esfand in a non-leap year) vs 31 days (a first-half-of-year month)
  vs 30 days (a second-half month) — use jalaali.jalaaliMonthLength
  directly in the test to compute the expected count, don't hardcode it
- Clicking a day cell calls onSelect with the correct Gregorian Date
  (cross-check against a known Jalali-to-Gregorian conversion pair)
- The disabled prop correctly prevents a day's click from firing
  onSelect
- Clicking the "next month" navigation button correctly advances to the
  next Jalali month, correctly rolling over the year at the
  Esfand->Farvardin boundary (test this specific boundary case
  explicitly, since it's the most error-prone part of manual
  month-increment logic)
- The RTL chevron rotation classes are present on the navigation
  buttons (a simple className-presence assertion, matching how
  calendar.tsx's own existing RTL fix would be tested, if it has an
  equivalent test — check for a precedent first)

Acceptance Criteria:
- npx vitest run src/shared/components/ui/__tests__/JalaliCalendar.test.tsx
  passes completely, including the leap-month-length and
  year-rollover tests
- JalaliCalendar's props shape (selected/onSelect/disabled) is
  confirmed compatible with TimeSlotPicker's existing Calendar usage
  (verify by reading TimeSlotPicker.tsx once more after building this
  component and confirming no prop mismatch — don't wire it in yet,
  that's the next prompt, just confirm compatibility on paper/by
  inspection)

Verification Steps:
1. npx vitest run src/shared/components/ui/__tests__/JalaliCalendar.test.tsx
2. npm run build
3. git diff --stat
```

### Prompt 3 — سیم‌کشی `JalaliCalendar` در جریان انتخاب تاریخ (Booking/Availability)

```
Goal: Swap every date-picker consumer (TimeSlotPicker,
AvailabilityCalendar, and any other component using the Gregorian
Calendar for a user-facing date-selection UI) from
shared/components/ui/calendar.tsx's Calendar to Prompt 2's
JalaliCalendar, verifying the swap doesn't change any downstream
booking/availability logic (which continues operating on Gregorian
Date objects exactly as before, per JalaliCalendar's drop-in-compatible
contract).

Before starting, read these files completely:
1. src/features/bookings/components/TimeSlotPicker.tsx — the full
   file, especially isDateDisabled and buildIsoFromDateAndTime, to
   confirm exactly nothing about them needs to change
2. src/features/teachers/components/AvailabilityCalendar.tsx — the
   full file
3. Check src/features/teachers/pages/TeacherAvailabilityPage.tsx and
   src/features/bookings/components/BookingRequestForm.tsx for any
   direct Calendar usage (grep -n "from \"@/shared/components/ui/calendar\""
   src/features/ to get the definitive, complete list of every file
   importing the Gregorian Calendar component — do not rely on the
   file list in this prompt's description alone, confirm it fresh)
4. src/shared/components/ui/JalaliCalendar.tsx (from Prompt 2)

What to build:

For every file found by the grep in step 3 above (the complete,
authoritative list — work through all of them, not just the ones named
in this prompt's goal):

1. Change the import from
   `import { Calendar } from "@/shared/components/ui/calendar"` to
   `import { JalaliCalendar } from "@/shared/components/ui/JalaliCalendar"`
2. Change the JSX usage from `<Calendar ... />` to
   `<JalaliCalendar ... />`, keeping every prop the same (selected,
   onSelect/onChange — check the exact prop name each call site
   currently uses and confirm JalaliCalendar's prop names match; if
   there's a naming mismatch between calendar.tsx's actual prop names
   and JalaliCalendar's Prompt 2 design, fix the mismatch in
   JalaliCalendar itself right now rather than papering over it at each
   call site, so the drop-in-replacement claim is actually true).
3. Do NOT touch any surrounding logic in these files (isDateDisabled,
   buildIsoFromDateAndTime, weekday-mapping logic, MAX_DAYS_AHEAD) —
   this prompt is exclusively the component swap, since Prompt 2's
   design already guarantees the Date objects flowing in/out are
   unchanged in shape and meaning.

Files affected: exactly the list produced by step 3's grep (confirm and
report the final list in your response).

Then run the existing tests for each swapped file and fix only what
directly broke due to the swap (e.g. a test that specifically queried
for calendar.tsx's Gregorian month-name text like "March" now correctly
expects a Persian Jalali month name instead — update these assertions
to the correct new expected text, verified against a real
Gregorian-to-Jalali conversion for whatever fixed test date the test
uses, not guessed).

Also add one integration-level test (in
src/features/bookings/components/__tests__/TimeSlotPicker.test.tsx or
wherever this component's existing tests live) that specifically
confirms: selecting a Jalali calendar day and then a time slot produces
the exact same final ISO datetime string that was produced by the
equivalent Gregorian-Calendar-based selection before this Phase — pick
a specific known date/time pair, compute the expected ISO string
independently (using the Prompt 1 conversion functions directly in the
test, not by trusting the component under test), and assert the
TimeSlotPicker's onChange callback receives exactly that string. This
is the single most important correctness test in this entire Phase,
since it's the one that proves this Phase's UI-only claim is actually
true end to end.

Acceptance Criteria:
- pytest is not applicable (frontend-only); npx vitest run passes the
  entire frontend test suite
- The critical ISO-equivalence integration test passes
- grep -n "from \"@/shared/components/ui/calendar\"" src/features/
  shows zero remaining results in date-selection contexts (report any
  legitimate non-date-selection usage found, if any exists, and leave
  it untouched with a note)
- npm run build succeeds

Verification Steps:
1. npx vitest run
2. grep -n "from \"@/shared/components/ui/calendar\"" src/features/
3. npm run build
4. Manual check: run the dev server, open the booking flow's date/time
   picker, confirm the calendar shows Jalali months/days correctly and
   that selecting a slot and completing a booking still produces a
   correctly-scheduled session (cross-check the resulting
   requested_start against what you clicked)
5. git diff --stat
```

### Prompt 4 — سیم‌کشی توابع نمایش Jalali در فایل‌های Display-only (۱۲ فایل)

```
Goal: Switch every read-only date-display consumer (booking cards,
payment cards, session cards, detail pages, and any other place a date
is shown to the user but not selected via a calendar) from the existing
Gregorian formatDate/formatDateTime to Prompt 1's
formatJalaliDate/formatJalaliDateTime.

Before starting:
1. Run: grep -rln "formatDate\|formatDateTime" src/features/ src/shared/
   (excluding date.ts itself and its tests) to get the complete,
   authoritative list of display consumers — do not rely solely on any
   list named in this prompt's goal, confirm it fresh, since Phase 5's
   changes might have touched some of these files too and altered
   their exact line numbers/structure.
2. Read each file found in full before editing it.
3. Read src/shared/lib/date.ts (post-Prompt 1) to confirm the exact
   formatJalaliDate/formatJalaliDateTime signatures.

What to build:

For every file found in step 1:
1. Change the import from formatDate/formatDateTime to
   formatJalaliDate/formatJalaliDateTime (keep both old and new imports
   only if a single file genuinely needs both — e.g. displaying one
   Gregorian-context date and one Jalali-context date, which should be
   extremely rare or nonexistent in a Jalali-only product; if you find
   such a case, flag it explicitly rather than silently converting or
   silently leaving it Gregorian).
2. Replace every call site: formatDate(x) -> formatJalaliDate(x),
   formatDateTime(x) -> formatJalaliDateTime(x).
3. Do not touch formatTime or formatRelativeTime call sites (formatTime
   is calendar-agnostic and correct as-is; formatRelativeTime was
   already fixed at its source in Prompt 1, so its call sites need no
   change).

Files affected: exactly the list from step 1 (confirm and report the
final list, expected to be around 12 files based on this Phase's
architecture doc, but verify the real count).

Then update every affected test: any test asserting a Gregorian-format
date string (e.g. "March 15, 2024" or a regex matching that shape)
needs its expected value replaced with the correct Jalali equivalent
for that same underlying date — compute this independently using the
Prompt 1 conversion functions in the test itself (or by direct
calculation, documented in a comment) rather than just copying whatever
the component under test happens to output, to make sure the test is
actually verifying correctness and not just echoing implementation
behavior.

Acceptance Criteria:
- grep -rln "formatDate\b\|formatDateTime\b" src/features/ src/shared/
  (excluding date.ts and its tests) returns zero results — every
  display consumer now uses the Jalali variants
- npx vitest run passes the entire frontend test suite
- npm run build succeeds

Verification Steps:
1. grep -rln "formatDate\b\|formatDateTime\b" src/features/ src/shared/
2. npx vitest run
3. npm run build
4. Manual check: run the dev server, view a booking detail page, a
   payment card, and a session card — confirm every displayed date
   shows the correct Jalali month name/day/year, and time-of-day still
   displays correctly (unaffected, since formatTime wasn't touched)
5. git diff --stat
```

### Prompt 5 — بررسی جامع: هیچ تاریخ میلادی/تبدیل دستی باقی‌مانده‌ای وجود ندارد

```
Goal: Comprehensive project-wide audit confirming no remaining
user-facing Gregorian date display exists anywhere, and no code outside
src/shared/lib/date.ts constructs or reinterprets a datetime by hand
(the exact invariant date.ts's own docstring already establishes for
this project — this prompt verifies it's still true after this Phase's
changes, and fixes anything found).

Before starting, review the diffs from Prompts 1-4 for the full
picture.

What to build:

1. Run a comprehensive search for any remaining raw date-fns `format()`
   calls with a Gregorian format string, anywhere outside
   src/shared/lib/date.ts itself:
   grep -rn "import.*format.*from \"date-fns\"" src/features/ src/shared/components/
   grep -rn "\.toLocaleDateString\(\)\|\.toLocaleString\(\)\|\.getMonth\(\)\|\.getFullYear\(\)\|\.getDate\(\)" src/features/
   For every match found, determine whether it's: (a) a genuine
   remaining Gregorian date-display bug that needs fixing (route it
   through formatJalaliDate/formatJalaliDateTime or the appropriate
   Prompt 1 helper instead), (b) a legitimate calendar-day-number
   extraction that's part of this Phase's own established, documented
   pattern (e.g. inside JalaliCalendar.tsx itself, which is expected
   and correct per its own design), or (c) something unrelated to
   calendar display entirely (e.g. a non-date use of a similarly-named
   method — verify carefully rather than assuming). Fix (a), leave (b)
   alone with a one-line comment confirming it's intentional if one
   isn't already present, and report (c) findings without changing
   them.

2. Check src/shared/components/ui/calendar.tsx's continued usage
   project-wide: grep -rn "from \"@/shared/components/ui/calendar\""
   src/. If this returns zero results anywhere in src/features/ (i.e.
   the original Gregorian Calendar component is now completely unused
   in the actual product, only existing as dead code potentially still
   referenced by Storybook/other tooling, or simply nowhere), report
   this finding explicitly — do not delete calendar.tsx or
   react-day-picker as a dependency in this prompt (that's a separate,
   deliberate cleanup decision outside this Phase's stated scope of
   "add Jalali support," not "remove Gregorian support"), just report
   whether it's now fully unused so the project maintainers can decide
   later.

3. Add one final end-to-end smoke test (in a location matching this
   project's existing integration-test conventions, e.g.
   src/test/integration/, following whatever precedent exists there)
   that walks through: viewing a teacher's availability calendar,
   selecting a Jalali date and time slot, completing a booking, and
   then viewing that booking's detail page — confirming the displayed
   date on the detail page matches, in Jalali, the date that was
   selected in the calendar (i.e. a full round-trip: Jalali selection
   -> Gregorian storage -> Jalali display, all correctly consistent).

4. Confirm the backend is completely untouched: run (if you have shell
   access to the monorepo root) git diff --stat afra-backend/ and
   confirm it's empty — report this explicitly in your final summary,
   since this Phase's architecture doc makes an explicit promise that
   no backend change is needed, and this is the step that verifies that
   promise was kept.

Files affected: any file fixed per step 1's findings (report the exact
list, even if empty), plus the new end-to-end test file.

Acceptance Criteria:
- The grep searches in step 1 return either zero results or only
  category (b)/(c) findings, explicitly reported
- The end-to-end round-trip smoke test passes
- git diff --stat afra-backend/ is confirmed empty
- npx vitest run (full frontend suite) passes with zero failures
- npm run build succeeds

Verification Steps:
1. grep -rn "import.*format.*from \"date-fns\"" src/features/ src/shared/components/
2. grep -rn "\.toLocaleDateString\(\)\|\.toLocaleString\(\)\|\.getMonth\(\)\|\.getFullYear\(\)\|\.getDate\(\)" src/features/
3. grep -rn "from \"@/shared/components/ui/calendar\"" src/
4. npx vitest run
5. npm run build
6. git diff --stat afra-backend/ (must be empty)
7. git diff --stat (final summary of this entire Phase, frontend only)
```
