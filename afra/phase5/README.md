# Phase 5 — Persian i18n + RTL (Frontend)

## Implementation Prompts

### Prompt 1 — زیرساخت i18next + `LocaleProvider` + تنظیمات پایهٔ RTL/فونت

```
Goal: Install and configure i18next/react-i18next, build a LocaleProvider
mirroring the existing ThemeProvider's exact structure, set dir="rtl"
and lang="fa" globally, and load the Vazirmatn font. No feature-level
text translation or Tailwind class changes yet — this prompt is purely
foundational plumbing.

Before starting, read these files completely:
1. src/app/providers/ThemeProvider.tsx, ThemeContext.ts, useTheme.ts —
   the full existing theme provider implementation, to copy its exact
   structural pattern (Context + localStorage persistence +
   useSyncExternalStore for external-source values + a single
   DOM-syncing useEffect)
2. src/App.tsx — to see exactly how ThemeProvider is currently wired
   in, so LocaleProvider is added the same way
3. index.html — the full current file
4. src/styles/globals.css — the full current file, to see the existing
   font-family setup and where to add Vazirmatn
5. package.json — confirm exact current React/TypeScript versions for
   compatibility with the i18next/react-i18next versions you'll add

What to build:

1. Install dependencies: i18next, react-i18next (latest stable
   versions compatible with this project's React version). Add them to
   package.json's dependencies (not devDependencies — they're runtime
   dependencies).

2. Create src/shared/i18n/locales/fa/common.json with an initial small
   set of genuinely common keys (not yet the full app's copy — that's
   built out namespace-by-namespace in later prompts):
   {
     "app": { "name": "افرا" },
     "actions": { "save": "ذخیره", "cancel": "انصراف", "submit": "ثبت", "retry": "تلاش مجدد", "close": "بستن" },
     "loading": "در حال بارگذاری…",
     "errors": { "generic": "خطایی رخ داد. لطفاً دوباره تلاش کنید." }
   }
   (Keep this intentionally minimal for now — enough to prove the
   plumbing works end to end without pre-writing all of the app's
   copy in this foundational prompt.)

3. Create src/shared/i18n/index.ts:
   Initialize i18next with react-i18next, configured with:
   - lng: "fa", fallbackLng: "fa" (single-locale for now, per this
     Phase's design — structured so a second locale could be added
     later without an architecture change, but not building any
     language-switcher UI now)
   - ns: ["common"] initially (grows as later prompts add
     auth/bookings/payments/etc. namespaces — document this in a
     comment: "Add new namespaces here as each feature's translations
     are added in later prompts of this Phase")
   - resources: { fa: { common: <imported common.json> } }
   - interpolation: { escapeValue: false } (React already escapes)
   Export the configured i18next instance.

4. Create src/app/providers/LocaleContext.ts, mirroring
   ThemeContext.ts's exact shape:
   export interface LocaleContextValue {
     locale: "fa";
     dir: "rtl";
   }
   export const LocaleContext = createContext<LocaleContextValue | undefined>(undefined);
   (Kept deliberately simple/fixed for now since only "fa"/"rtl" exist
   — this is intentionally less complex than ThemeContext's
   light/dark/system three-way state, since there's no equivalent
   "auto-detect" concept for locale in this Phase's scope. Document
   this asymmetry with a comment referencing ThemeContext as the
   pattern this was modeled on, explaining why it's simpler.)

5. Create src/app/providers/LocaleProvider.tsx, mirroring
   ThemeProvider.tsx's exact structure and code style (its function
   naming conventions, its comment style, its default export):
   - Imports and initializes src/shared/i18n/index.ts (side-effect
     import, or explicit init call — match whatever's cleanest)
   - A useEffect that applies dir="rtl" and lang="fa" to
     document.documentElement (document.documentElement.dir = "rtl";
     document.documentElement.lang = "fa";) — mirroring
     applyResolvedTheme's DOM-syncing effect pattern exactly
   - Wraps children in both LocaleContext.Provider (value={{ locale:
     "fa", dir: "rtl" }}) and react-i18next's I18nextProvider
   - Also create a small useLocale() hook (mirroring useTheme.ts's
     exact shape) that reads LocaleContext via useContext with the
     same "throw if used outside provider" guard pattern useTheme.ts
     uses

6. In index.html:
   - Change <html lang="en"> to <html lang="fa" dir="rtl"> (a correct
     static default before JS even runs, so there's no flash of
     LTR-then-RTL on first paint — the LocaleProvider's effect in step
     5 then just confirms/maintains this at runtime)
   - Add a font preload/link for Vazirmatn (use a CDN link, e.g. from
     Google Fonts if Vazirmatn is available there, or note in a
     comment that self-hosting is preferable for production and this
     is a starting point — check what's actually available and use the
     most reliable option; prefer a preconnect + stylesheet link
     pattern matching standard web font loading best practice)

7. In src/styles/globals.css:
   Set Vazirmatn as the primary font-family on body (with the same
   fallback chain already used in the email templates —
   'Vazirmatn', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto,
   Helvetica, Arial, sans-serif — for visual consistency between the
   app and the transactional emails from Phase 4).

8. In src/App.tsx:
   Wrap the existing provider tree with <LocaleProvider> — placed
   outermost or wherever makes sense relative to ThemeProvider (check
   whether locale should wrap theme or vice versa; since they're
   independent concerns, order likely doesn't matter functionally, but
   pick one and keep it consistent — document your choice).

Files affected:
- package.json (+ package-lock.json regenerated by npm install)
- src/shared/i18n/index.ts (new)
- src/shared/i18n/locales/fa/common.json (new)
- src/app/providers/LocaleContext.ts (new)
- src/app/providers/LocaleProvider.tsx (new)
- src/app/providers/useLocale.ts (new)
- index.html
- src/styles/globals.css
- src/App.tsx

Then write tests:
- src/app/providers/__tests__/LocaleProvider.test.tsx: rendering
  LocaleProvider sets document.documentElement.dir === "rtl" and
  lang === "fa"; useLocale() throws when used outside the provider
  (mirroring however ThemeProvider's equivalent guard is already
  tested, if it is — check tests/app/providers/ThemeProvider.test.tsx
  or similar for the exact testing convention to follow)
- A minimal smoke test rendering a component that calls useTranslation("common")
  and confirms t("actions.save") resolves to "ذخیره"

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds
- Manually running the dev server shows the page rendered
  right-to-left with the Vazirmatn font applied

Verification Steps:
1. npx vitest run
2. npm run build
3. npm run dev, open the app in a browser, confirm (a) text direction
   is RTL, (b) the font is Vazirmatn (check via browser devtools'
   computed font-family), (c) no console errors about missing i18next
   resources
4. git diff --stat
```

### Prompt 2 — RTL: کامپوننت‌های پایهٔ `shared/components/ui`

```
Goal: Convert every directional (LTR-hardcoded) Tailwind utility class
in src/shared/components/ui/ (the shadcn-style base component library
used throughout the entire app) to its logical-property equivalent, so
every higher-level feature component that uses these primitives is
automatically RTL-correct without further changes. No text translation
in this prompt — pure CSS-class conversion.

Before starting:
1. Run: grep -rln "ml-\|mr-\|pl-\|pr-\|left-\|right-\|text-left\|text-right\|border-l\|border-r\|rounded-l\|rounded-r\|space-x-" src/shared/components/ui/
   and get the exact file list to work through.
2. Read the Tailwind CSS documentation's logical properties utility
   list (or infer from Tailwind's known utility-class naming: ml-N ->
   ms-N, mr-N -> me-N, pl-N -> ps-N, pr-N -> pe-N, left-N -> start-N,
   right-N -> end-N, text-left -> text-start, text-right -> text-end,
   border-l -> border-s, border-r -> border-e, rounded-l -> rounded-s,
   rounded-r -> rounded-e, space-x-N needs special handling — see
   below) to build your exact mechanical mapping before editing files.
3. Read src/shared/components/ui/dialog.tsx, dropdown-menu.tsx,
   select.tsx, popover.tsx, sheet.tsx, tooltip.tsx in full — these wrap
   Radix UI primitives, so check specifically whether any hardcoded
   `side="left"`/`side="right"` or `align="start"/"end"` props (Radix's
   own positioning props, separate from Tailwind classes) need
   adjustment. Radix's `align="start"/"end"` values are already
   direction-aware (they follow `dir` automatically per Radix's docs),
   so these likely need NO change — but `side="left"`/`side="right"`
   are NOT direction-aware in Radix (they're physical, not logical) and
   DO need review: if a component hardcodes side="right" intending "the
   side where it visually makes sense in LTR," that assumption may now
   be wrong in RTL and should become side="end" if Radix supports it
   for that primitive, or be explicitly reconsidered per-component.

What to build:

For each file found in step 1:
1. Replace every directional margin/padding/position/text-align/
   border-radius/border-width utility with its logical equivalent per
   the mapping above.
2. For `space-x-N` (Tailwind's non-logical horizontal-spacing
   shorthand, which has no direct RTL-safe equivalent in Tailwind 3
   without a plugin): replace with explicit `gap-N` on a flex/grid
   parent where the layout is already flex/grid (gap is direction-
   agnostic and RTL-safe by default), which is almost always possible
   for the kinds of layouts in a component library like this. If a
   specific case genuinely can't be converted to gap (rare), leave a
   comment explaining why and flag it in your final summary rather
   than silently leaving a broken RTL case.
3. For any Radix `side="left"`/`side="right"` found per your reading
   above, evaluate case by case whether it should become "start"/"end"
   (if that primitive's API supports logical sides — check each
   component's actual Radix primitive docs/types) or needs a
   dir-aware conditional (using the useLocale() hook from Prompt 1) if
   the primitive genuinely only accepts physical sides. Document each
   decision with a one-line comment at the call site.

Do NOT touch any text content, any component's exported API/props
shape, or any file outside src/shared/components/ui/ in this prompt.

Files affected: exactly the list produced by step 1's grep (confirm
the final list in your response).

Then run the existing component tests for this directory
(npx vitest run src/shared/components/ui/) and fix any test that
specifically asserted a physical class name (e.g. checking for
"ml-2" in a className string) — update those assertions to the new
logical class name; do not change what the test is actually verifying.

Acceptance Criteria:
- grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right\|border-l-\|border-r-\|rounded-l-\|rounded-r-\|space-x-" src/shared/components/ui/
  returns no results, except any explicitly documented/flagged
  exceptions from step 3 above
- npx vitest run src/shared/components/ui/ passes completely
- npm run build succeeds

Verification Steps:
1. grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right\|border-l-\|border-r-\|rounded-l-\|rounded-r-\|space-x-" src/shared/components/ui/
2. npx vitest run src/shared/components/ui/
3. npm run build
4. Manual check: run the dev server (with dir="rtl" already active
   from Prompt 1), open a page using Dialog/DropdownMenu/Select (check
   router.tsx for a page that uses one, e.g. a form page), and confirm
   it opens/aligns correctly in RTL, not mirrored incorrectly
5. git diff --stat
```

### Prompt 3 — RTL: Layout، Navigation، و کامپوننت‌های `feedback`/`guards`

```
Goal: Fix RTL for the app's structural layout components (Navbar,
Sidebar, Footer, AppLayout, DashboardLayout, Breadcrumbs) and the
feedback/guard components, including any icon-direction issues
(e.g. a "next" chevron that must visually point the opposite way in
RTL).

Before starting:
1. Run: grep -rln "ml-\|mr-\|pl-\|pr-\|left-\|right-\|text-left\|text-right\|border-l\|border-r\|rounded-l\|rounded-r\|space-x-" src/shared/components/layout/ src/shared/components/feedback/ src/shared/components/guards/
2. Read src/shared/components/layout/Navbar.tsx, Sidebar.tsx,
   Breadcrumbs.tsx in full — these are the highest-risk files for
   directional icons (a breadcrumb separator chevron, a sidebar
   collapse/expand arrow, a "back" button)
3. grep -rn "ChevronRight\|ChevronLeft\|ArrowRight\|ArrowLeft" src/shared/components/
   to find every directional icon import from lucide-react in this
   scope — each one needs manual review: does this icon represent a
   PHYSICAL direction (e.g. a literal left-pointing arrow in a diagram)
   or a LOGICAL direction (e.g. "next"/"forward" in a breadcrumb or
   pagination, which must flip in RTL since "forward" now points
   visually left)? Only the logical ones need to change.

What to build:

1. Apply the same mechanical logical-class conversion from Prompt 2's
   mapping to every file found in step 1.

2. For every logical-direction icon found in step 3 (e.g. a breadcrumb
   separator that's currently always ChevronRight, or a "next page"
   icon), make it direction-aware: import useLocale from Prompt 1 (or,
   simpler, since dir is currently always "rtl" for the whole app, you
   could hardcode the RTL-correct icon directly — but prefer the
   direction-aware approach using useLocale()'s dir value with a
   conditional, e.g. dir === "rtl" ? ChevronLeft : ChevronRight, since
   this keeps the component correct if a second locale/LTR mode is
   ever added later, matching this Phase's stated "structured for
   future multi-locale, not building the toggle now" design principle
   from the architecture doc).

3. Specifically review Sidebar.tsx and Navbar.tsx for any layout that
   assumes a fixed physical side (e.g. "sidebar is always on the
   left," a common LTR assumption) — in RTL, a sidebar conventionally
   moves to the right. Check how this project's Sidebar is currently
   positioned (a fixed left offset via Tailwind classes, or flex
   ordering) and correct it to use logical start/end so it correctly
   appears on the right in RTL without any special-casing beyond the
   logical-class conversion already done.

4. Review Breadcrumbs.tsx's separator rendering logic specifically —
   confirm the separator icon direction is now correct per step 2, and
   confirm the breadcrumb items themselves read in the correct visual
   order for RTL reading (this should happen automatically from
   dir="rtl" plus a flex layout with no hardcoded flex-direction, but
   verify this isn't overridden by an explicit flex-row that would need
   to become direction-agnostic).

Files affected: the list from step 1, plus any file with a directional
icon identified in step 3.

Then run the relevant existing tests and fix any broken assertions,
following the same principle as Prompt 2 (fix the assertion to match
the new correct behavior, don't change what's being tested).

Acceptance Criteria:
- grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right\|border-l-\|border-r-\|rounded-l-\|rounded-r-\|space-x-" src/shared/components/layout/ src/shared/components/feedback/ src/shared/components/guards/
  returns no results
- npx vitest run passes for these directories
- npm run build succeeds

Verification Steps:
1. grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right\|border-l-\|border-r-\|rounded-l-\|rounded-r-\|space-x-" src/shared/components/layout/ src/shared/components/feedback/ src/shared/components/guards/
2. npx vitest run src/shared/components/
3. npm run build
4. Manual visual check: run the dev server, confirm the Sidebar (if
   present on any page) renders on the right, the breadcrumb separator
   points the correct direction, and the Navbar's items read
   right-to-left correctly
5. git diff --stat
```

### Prompt 4 — فارسی‌سازی + RTL: دامنهٔ Auth

```
Goal: Translate all user-visible text in the auth feature
(login/register/forgot-password/reset-password/verify-email) to
Persian via i18next, add Persian Zod validation messages, and fix any
remaining directional Tailwind classes specific to this feature's own
components (beyond what Prompts 2-3 already fixed in shared
components).

Before starting, read these files completely:
1. Every file under src/features/auth/ (pages and any
   feature-specific components) — the full current English copy and
   Zod schemas in each
2. src/shared/i18n/index.ts (from Prompt 1) — to see the exact
   namespace-registration pattern for adding a new "auth" namespace
3. src/app/providers/useLocale.ts / useTranslation usage pattern
   established in Prompt 1

What to build:

1. Create src/shared/i18n/locales/fa/auth.json with keys for every
   piece of user-visible text in this feature: page titles, labels,
   placeholders, button text, validation error messages, success/error
   toast messages, links ("Don't have an account? Register", etc.).
   Organize keys by page for readability, e.g.:
   {
     "login": { "title": "...", "emailLabel": "...", "passwordLabel": "...", "submit": "...", "noAccount": "..." },
     "register": { ... },
     "forgotPassword": { ... },
     "resetPassword": { ... },
     "verifyEmail": { ... },
     "validation": { "emailRequired": "...", "emailInvalid": "...", "passwordMinLength": "...", "passwordsDoNotMatch": "..." }
   }

2. Register the "auth" namespace in src/shared/i18n/index.ts's
   resources/ns config (extending, not replacing, the existing
   "common" registration from Prompt 1).

3. In every page/component under src/features/auth/:
   - Replace every hardcoded English string in JSX with
     t("auth:login.title") style calls via useTranslation("auth")
     (or however this project's i18next setup expects namespaced key
     access — confirm the exact calling convention against your
     Prompt 1 config)
   - Update every Zod schema's .min()/.email()/.refine() error message
     strings to pull from the same translation keys (e.g.
     z.string().email(t("auth:validation.emailInvalid"))) — note that
     Zod schemas are often defined outside the component's render
     scope (e.g. module-level), which won't have access to a
     component-scoped t() from useTranslation; if that's the case
     here, either (a) move schema definition inside the component/a
     hook so it can call useTranslation, or (b) use i18next's
     standalone t function (imported directly from the i18next
     instance in Prompt 1, not the React hook) for schemas defined at
     module scope — check how each file in this feature currently
     defines its schema (inside vs outside the component) and use
     whichever approach fits without restructuring the component
     unnecessarily
   - Reuse t("common:actions.save")-style calls from the "common"
     namespace (Prompt 1) wherever a generic action button already
     has a common-namespace equivalent (e.g. a generic "Cancel"
     button), rather than duplicating that string into auth.json

4. grep -rln "ml-\|mr-\|pl-\|pr-\|left-\|right-\|text-left\|text-right"
   src/features/auth/ and fix any remaining directional classes local
   to this feature's own JSX (not already covered by Prompts 2-3's
   shared-component fixes).

Files affected:
- src/shared/i18n/locales/fa/auth.json (new)
- src/shared/i18n/index.ts (namespace registration)
- every file under src/features/auth/pages/ and any
  src/features/auth/components/

Then update every existing test in src/features/auth/**/__tests__/:
replace any getByText("English string") query with either
getByRole(...) (preferred, per this Phase's testing-quality
improvement goal noted in the architecture doc) or
getByText(/فارسی متن مربوطه/) matching the new Persian copy — prefer
role-based queries wherever the test doesn't specifically need to
assert exact copy, and reserve text-based queries for tests that are
actually about verifying specific message content (e.g. confirming a
validation error shows the right message).

Acceptance Criteria:
- grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right" src/features/auth/
  returns no results
- No hardcoded English JSX string remains in src/features/auth/ (spot
  check manually — this is hard to grep reliably, so do a careful
  read-through of every file after editing)
- npx vitest run src/features/auth/ passes completely
- npm run build succeeds

Verification Steps:
1. grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right" src/features/auth/
2. npx vitest run src/features/auth/
3. npm run build
4. Manual check: run the dev server, visit /login, /register,
   /forgot-password — confirm all text is Persian and the layout reads
   correctly right-to-left
5. git diff --stat
```

### Prompt 5 — فارسی‌سازی + RTL: دامنهٔ Teachers (Marketplace، Dashboard، Verification، Payout)

```
Goal: Translate all user-visible text in the teachers feature — the
largest and most feature-rich domain in this project, spanning the
public marketplace listing/profile, the teacher dashboard, the Phase 1
payout-account UI, and the Phase 2 verification UI — and fix any
remaining directional classes local to this feature.

Before starting:
1. Read every file under src/features/teachers/pages/ and
   src/features/teachers/components/ in full
2. Read src/shared/i18n/locales/fa/auth.json and index.ts (from
   Prompt 4) for the established namespace/key-organization convention
   to follow consistently

What to build:

1. Create src/shared/i18n/locales/fa/teachers.json, organized by
   page/component (marketplace list, teacher card, public profile,
   dashboard, verification status banner + submission form, payout
   account banner + form), covering every piece of user-visible text:
   labels, empty states ("No teachers found"), button text, status
   labels (PENDING/UNDER_REVIEW/APPROVED/REJECTED/SUSPENDED — these
   enum values from Phase 2's backend need a Persian display-label
   mapping, e.g. { "status": { "PENDING": "در انتظار بررسی",
   "UNDER_REVIEW": "در حال بررسی", "APPROVED": "تأیید شده",
   "REJECTED": "رد شده", "SUSPENDED": "معلق شده" } }), validation
   messages for the payout-account shaba-number form and the
   verification-document upload form.

2. Register the "teachers" namespace in src/shared/i18n/index.ts.

3. Apply t("teachers:...") translation across every file under
   src/features/teachers/, following Prompt 4's exact same approach
   (useTranslation hook, module-scope Zod schemas using the standalone
   t function where needed).

4. Specifically for status-enum display (verification status, and any
   other enum-like value rendered as a badge/label): build a small
   shared helper (e.g. src/features/teachers/lib/statusLabels.ts) that
   maps the raw API enum string to its translated label via the
   status.* keys from step 1, and use this helper everywhere a status
   is displayed (VerificationStatusBanner, any admin-facing status
   list if present) — rather than inlining a switch/ternary per
   component, so there's exactly one place that needs updating if a
   status enum value's label copy changes.

5. grep -rln "ml-\|mr-\|pl-\|pr-\|left-\|right-\|text-left\|text-right"
   src/features/teachers/ and fix any remaining directional classes.

6. Review any star-rating or numeric-badge display (e.g.
   average_rating "(23 reviews)" text from Phase 3's
   TeacherPublicProfilePage) and translate its surrounding copy
   (e.g. "{{count}} reviews" -> a Persian pluralized string — use
   i18next's built-in pluralization support, i18next-icu-style, or
   i18next's default count-based key suffixing (_one/_other, though
   Persian doesn't distinguish singular/plural the way English does —
   check i18next's Persian plural rule support; if i18next doesn't
   have a built-in CLDR plural rule for "fa", a single count-agnostic
   phrasing like "{{count}} نظر" is linguistically correct for Persian
   anyway, since Persian doesn't pluralize nouns after numbers the way
   English does — use this simpler correct-for-Persian phrasing rather
   than forcing an English-style plural/singular split that Persian
   grammar doesn't need).

Files affected:
- src/shared/i18n/locales/fa/teachers.json (new)
- src/shared/i18n/index.ts
- src/features/teachers/lib/statusLabels.ts (new)
- every file under src/features/teachers/pages/ and
  src/features/teachers/components/

Then update every existing test under src/features/teachers/, following
the exact same role-based-query-preferred approach from Prompt 4.

Acceptance Criteria:
- grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right" src/features/teachers/
  returns no results
- No hardcoded English JSX string remains (manual read-through
  confirmation)
- npx vitest run src/features/teachers/ passes completely
- npm run build succeeds

Verification Steps:
1. grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right" src/features/teachers/
2. npx vitest run src/features/teachers/
3. npm run build
4. Manual check: visit the teacher marketplace list, a public teacher
   profile, the teacher dashboard, the verification page, and the
   payout-account page — confirm all text is Persian and status badges
   show translated labels
5. git diff --stat
```

### Prompt 6 — فارسی‌سازی + RTL: دامنه‌های Bookings و Payments

```
Goal: Translate all user-visible text in the bookings and payments
features, including the Phase 1 checkout/callback flow, and fix
remaining directional classes in both.

Before starting, read every file under src/features/bookings/ and
src/features/payments/ in full, plus
src/shared/i18n/locales/fa/teachers.json (from Prompt 5) as the
continued example of the established convention.

What to build:

1. Create src/shared/i18n/locales/fa/bookings.json and
   src/shared/i18n/locales/fa/payments.json (two separate namespaces,
   matching the app-level domain split), covering:
   - Booking request form (skill selection, date/time selection labels
     — NOT the actual calendar rendering, which is Phase 6's concern;
     just the surrounding form labels/buttons)
   - Booking status labels (PENDING_PAYMENT, AWAITING_TEACHER_REVIEW,
     ACCEPTED, REJECTED, CANCELLED, COMPLETED, etc. — build a
     statusLabels.ts helper for bookings mirroring Prompt 5's pattern
     exactly)
   - Payment status labels (PENDING, HELD_IN_ESCROW, RELEASED, FAILED,
     REFUNDED, DISPUTED — same helper pattern)
   - The BookingPaymentPage's "Pay" button and loading/redirect states
     from Phase 1
   - PaymentCallbackPage's success/failure messages from Phase 1
   - PaymentCard, PaymentSummary, TeacherPaymentSummary display labels
   - DisputeDialog's form labels and validation messages (found via
     the earlier zod-schema grep)

2. Register both namespaces in src/shared/i18n/index.ts.

3. Apply translation across every file in both features, following the
   established pattern from Prompts 4-5 exactly (useTranslation hook,
   standalone t for module-scope schemas, status-label helper
   functions rather than inline switches).

4. Money display check: confirm src/shared/lib/money.ts's formatToman
   (from Phase 0) is being used consistently across all payment/
   booking price displays in these two features — if any file still
   shows a raw number without the Toman formatter (a regression risk
   worth checking now that you're touching every file in this domain
   anyway), fix it and note this finding explicitly in your summary
   even though it's technically a Phase 0 concern, since you're already
   reviewing every relevant file here.

5. grep -rln "ml-\|mr-\|pl-\|pr-\|left-\|right-\|text-left\|text-right"
   src/features/bookings/ src/features/payments/ and fix remaining
   directional classes.

Files affected:
- src/shared/i18n/locales/fa/bookings.json (new)
- src/shared/i18n/locales/fa/payments.json (new)
- src/shared/i18n/index.ts
- src/features/bookings/lib/statusLabels.ts (new)
- src/features/payments/lib/statusLabels.ts (new)
- every file under src/features/bookings/ and src/features/payments/

Then update every existing test under both feature directories,
following the established role-based-query-preferred convention.

Acceptance Criteria:
- grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right" src/features/bookings/ src/features/payments/
  returns no results
- No hardcoded English JSX string remains in either feature
- npx vitest run src/features/bookings/ src/features/payments/ passes
  completely
- npm run build succeeds
- Every price display in these two features uses formatToman
  consistently (report any fix made here explicitly)

Verification Steps:
1. grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right" src/features/bookings/ src/features/payments/
2. npx vitest run src/features/bookings/ src/features/payments/
3. npm run build
4. Manual check: walk through creating a booking, reaching the
   checkout page, and viewing a payment card — confirm all text is
   Persian, prices show as Toman, status badges are translated
5. git diff --stat
```

### Prompt 7 — فارسی‌سازی + RTL: دامنه‌های Sessions و Reviews

```
Goal: Translate all user-visible text in the sessions and reviews
features (including Phase 3's review submission/display UI), and fix
remaining directional classes in both.

Before starting, read every file under src/features/sessions/ and
src/features/reviews/ in full.

What to build:

1. Create src/shared/i18n/locales/fa/sessions.json and
   src/shared/i18n/locales/fa/reviews.json, covering:
   - Session status/outcome labels (SCHEDULED, ONGOING, COMPLETED,
     CANCELLED; outcome COMPLETED/NO_SHOW_TEACHER/NO_SHOW_STUDENT) —
     another statusLabels.ts helper, matching the established pattern
   - The "Leave a Review" call-to-action copy (from Phase 3)
   - SubmitReviewForm's labels and validation messages (e.g. "Please
     select a rating" for a missing star selection)
   - ReviewCard's relative-date and rating display copy
   - ReviewList's empty state ("No reviews yet")
   - Any no-show reporting UI text, if present in this feature

2. Register both namespaces in src/shared/i18n/index.ts.

3. Apply translation across every file in both features, following the
   exact established pattern from previous prompts.

4. Star rating text: ensure any numeric rating display (e.g. "4.8 out
   of 5") uses Persian-appropriate phrasing, and confirm number
   formatting for the rating value itself uses a locale-aware format
   consistent with this Phase's approach (Intl.NumberFormat("fa-IR")
   or i18next's built-in number formatting — check what Prompt 6 used
   for consistency and match it here, rather than introducing a third
   formatting approach).

5. grep -rln "ml-\|mr-\|pl-\|pr-\|left-\|right-\|text-left\|text-right"
   src/features/sessions/ src/features/reviews/ and fix remaining
   directional classes.

Files affected:
- src/shared/i18n/locales/fa/sessions.json (new)
- src/shared/i18n/locales/fa/reviews.json (new)
- src/shared/i18n/index.ts
- src/features/sessions/lib/statusLabels.ts (new)
- every file under src/features/sessions/ and src/features/reviews/

Then update every existing test under both feature directories,
following the established convention.

Acceptance Criteria:
- grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right" src/features/sessions/ src/features/reviews/
  returns no results
- No hardcoded English JSX string remains in either feature
- npx vitest run src/features/sessions/ src/features/reviews/ passes
  completely
- npm run build succeeds

Verification Steps:
1. grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right" src/features/sessions/ src/features/reviews/
2. npx vitest run src/features/sessions/ src/features/reviews/
3. npm run build
4. Manual check: visit a session detail page and the review submission
   flow — confirm all text is Persian
5. git diff --stat
```

### Prompt 8 — فارسی‌سازی + RTL: Students، Profile، Notifications، Landing، و باقیماندهٔ Shared

```
Goal: Translate all remaining user-visible text across the students,
profile, notifications, and landing features, plus any remaining
top-level shared pages (error pages, empty states, etc.), and fix any
remaining directional classes anywhere left untouched by Prompts 2-7.

Before starting:
1. Read every file under src/features/students/, src/features/profile/,
   src/features/notifications/, src/features/landing/, and
   src/shared/pages/ in full
2. Run a comprehensive grep across the ENTIRE src/features and
   src/shared trees (not just these four features) for any remaining
   directional classes, since this is the last prompt with dedicated
   scope before Prompt 9's final audit:
   grep -rln "ml-\|mr-\|pl-\|pr-\|left-\|right-\|text-left\|text-right\|space-x-" src/features/ src/shared/

What to build:

1. Create src/shared/i18n/locales/fa/students.json,
   src/shared/i18n/locales/fa/profile.json,
   src/shared/i18n/locales/fa/notifications.json,
   src/shared/i18n/locales/fa/landing.json, covering every piece of
   user-visible text in their respective features (profile edit form
   labels/validation, change-password form, notification list items
   and empty state, landing page hero/marketing copy, students feature
   pages).

2. Register all four namespaces in src/shared/i18n/index.ts.

3. Apply translation across every file in these four features,
   following the exact established pattern.

4. For any remaining shared/top-level page (e.g. a 404 page, a generic
   error page, ErrorFallback.tsx) not yet translated, add its copy to
   common.json (from Prompt 1) since these are truly app-wide, not
   feature-specific.

5. Fix every remaining directional class found by step 2's
   comprehensive grep, across the ENTIRE src tree — this prompt is
   responsible for catching anything Prompts 2-7 missed in their
   feature-scoped passes, in addition to this prompt's own four
   features.

Files affected:
- src/shared/i18n/locales/fa/students.json, profile.json,
  notifications.json, landing.json (new)
- src/shared/i18n/locales/fa/common.json (extended, for shared
  top-level pages)
- src/shared/i18n/index.ts
- every file under the four features listed, plus any leftover file
  identified by step 2's project-wide grep

Then update every remaining test with hardcoded English assertions,
following the established convention, across whatever files this
prompt touches.

Acceptance Criteria:
- grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right\|space-x-" src/features/ src/shared/
  returns no results anywhere in the project (this is the last
  feature-scoped prompt, so this must be fully clean before Prompt 9)
- No hardcoded English JSX string remains anywhere in these four
  features
- npx vitest run passes the entire frontend test suite
- npm run build succeeds

Verification Steps:
1. grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|text-left\|text-right\|space-x-" src/features/ src/shared/
2. npx vitest run
3. npm run build
4. Manual check: visit the landing page, profile page, and
   notifications list — confirm all text is Persian
5. git diff --stat
```

### Prompt 9 — لایهٔ نگاشت کد خطای Backend به پیام فارسی

```
Goal: Build a mapping layer that translates known backend
ApplicationError `code` values (established across Phases 1-4's
services.py functions) into Persian user-facing messages, and wire it
into wherever this project currently surfaces API error messages to
the user (toasts, inline form errors).

Before starting, read these files completely:
1. src/shared/api/axios.instance.ts — the full file, to see the
   current error-handling/interceptor structure and where a response
   error's body is currently accessed
2. Search across src/features/ for how a failed mutation's error is
   currently displayed to the user (grep -rn "onError\|isError\|error.message"
   src/features/ — look for the dominant pattern: is it a toast
   library call like sonner's toast.error(...), an inline form field
   error, or both, and where does the error text currently come from —
   likely directly from the API's raw English error message)
3. Compile (from your own reading of every services.py file across
   Phases 1-4 covered in this project, or by grepping the backend if
   it's accessible: grep -rn 'code="' afra-backend/apps/*/services.py)
   the complete list of ApplicationError code values currently defined
   across the project. This is the authoritative list this prompt's
   mapping must cover — do not guess at codes that don't actually
   exist in the backend.

What to build:

1. Create src/shared/i18n/locales/fa/errors.json with a flat
   code-to-message map covering every ApplicationError code found in
   your backend search, e.g.:
   {
     "gateway_refund_failed": "بازپرداخت با خطا مواجه شد. لطفاً دوباره تلاش کنید یا با پشتیبانی تماس بگیرید.",
     "invalid_verification_status": "این عملیات در وضعیت فعلی مدرس مجاز نیست.",
     "review_window_expired": "مهلت ثبت نظر برای این جلسه به پایان رسیده است.",
     "session_not_reviewable": "این جلسه هنوز قابل نظردهی نیست.",
     "group_review_not_supported": "ثبت نظر برای جلسات گروهی هنوز پشتیبانی نمی‌شود.",
     "payout_account_not_ready": "حساب بانکی مدرس هنوز تأیید نشده است.",
     ... (continue for every code found)
   }
   Also add a "generic" fallback key (reuse common.json's existing
   "errors.generic" from Prompt 1 rather than duplicating it here) for
   any code not present in this map.

2. Register the "errors" namespace in src/shared/i18n/index.ts.

3. Create src/shared/lib/apiErrorMessage.ts:
   export function getApiErrorMessage(error: unknown): string {
     // Extract the backend's error `code` field from an AxiosError's
     // response body (confirm the exact shape the backend actually
     // returns — check a real error response shape from any existing
     // test's mock fixtures, e.g. src/test/mocks/handlers/*.ts for an
     // error-case handler, to get the precise field name/nesting
     // right rather than guessing)
     // If a code is found and has a mapped translation in errors.json,
     // return i18next.t(`errors:${code}`)
     // Otherwise, return the generic fallback message
   }
   This function must not throw — any unexpected error shape (network
   error, non-Axios error, malformed response) should safely fall
   through to the generic fallback message rather than crashing the
   error-display path itself.

4. Wire getApiErrorMessage into wherever this project currently
   displays API errors (identified in your reading of step 2) —
   replace direct usage of error.message or error.response.data.detail
   (or whatever the current raw-message access pattern is) with
   getApiErrorMessage(error) at each of those call sites. Do this
   consistently across every feature that currently shows raw backend
   error text, not just a couple of examples — this is a project-wide
   sweep, similar in spirit to the directional-class sweeps in
   Prompts 2-8, but for error-message display instead of CSS classes.

Files affected:
- src/shared/i18n/locales/fa/errors.json (new)
- src/shared/i18n/index.ts
- src/shared/lib/apiErrorMessage.ts (new)
- every file identified in step 2/4 that currently displays a raw
  backend error message (exact list depends on your findings — report
  it in your final summary)

Then write tests:
- src/shared/lib/__tests__/apiErrorMessage.test.ts: a known code maps
  to its Persian message; an unknown code falls back to the generic
  message; a malformed/non-Axios error also falls back gracefully
  without throwing
- Update at least 2-3 existing feature tests (spot-checking, not
  necessarily every single call site) that specifically test an
  error-display path, to confirm they now show the mapped Persian
  message instead of a raw backend string

Acceptance Criteria:
- pytest (backend, read-only verification — no backend changes made in
  this prompt) confirms the code list gathered in your investigation
  is accurate (no backend changes needed, this is just a
  cross-reference check)
- npx vitest run src/shared/lib/__tests__/apiErrorMessage.test.ts
  passes completely
- npx vitest run (full suite) passes
- npm run build succeeds

Verification Steps:
1. npx vitest run src/shared/lib/__tests__/apiErrorMessage.test.ts
2. npx vitest run
3. npm run build
4. grep -rn "error.response.data\|error.message" src/features/
   (review remaining matches — confirm each one is either already
   routed through getApiErrorMessage or is a legitimately different
   concern, e.g. logging rather than user-facing display; report
   findings)
5. git diff --stat
```

### Prompt 10 — بررسی نهایی جامع: کلیدهای گم‌شده، RTL باقیمانده، و تست End-to-End

```
Goal: Final comprehensive audit of this entire Phase — detect any
missing translation keys, confirm zero remaining directional Tailwind
classes anywhere in the project, and add an automated check that
prevents future regressions of both.

Before starting, review the diffs from Prompts 1-9 for the full
picture of this Phase's changes.

What to build:

1. Project-wide final grep (must return zero results anywhere in
   src/, not just the features covered by name in earlier prompts):
   grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|left-[0-9]\|right-[0-9]\|text-left\|text-right\|border-l-\|border-r-\|rounded-l-\|rounded-r-\|space-x-" src/
   Fix anything still found.

2. Missing-translation-key detection: configure i18next's built-in
   missing-key handling (the `saveMissing` + a custom `missingKeyHandler`
   option, or i18next's `parseMissingKeyHandler`) to, in development/
   test mode only, throw or console.error loudly when a component
   requests a key that doesn't exist in fa's resources — rather than
   i18next's default behavior of silently rendering the raw key
   string. Wire this into the Vitest test setup (e.g.
   src/test/setup.ts, if that's where global test configuration
   lives) so that ANY test rendering a component with a missing
   translation key causes that test to fail loudly, rather than
   passing with visibly-broken output that a human reviewer might miss
   in a snapshot.

3. With the missing-key detection from step 2 now active, run the full
   test suite:
   npx vitest run
   and fix every newly-surfaced missing-key failure (these are real
   gaps left by Prompts 4-8 — a key referenced in a component but
   never added to the corresponding namespace JSON file). This is the
   step most likely to surface concrete, previously-invisible bugs
   from this Phase's large surface area.

4. Add one end-to-end smoke test (using this project's existing
   integration-test setup, if src/test/integration/ has established
   conventions — check there first) that renders the full app shell
   (App.tsx with all providers) and confirms: document.documentElement.dir
   === "rtl", at least one piece of known Persian text (e.g. a Navbar
   label) renders correctly, and no console errors/warnings about
   missing translation keys occur during this render.

5. Final documentation check: if this project has a README or CONTRIBUTING
   doc describing how to add a new page/feature, add a short section
   (a few lines, not a rewrite) noting the i18n convention now in
   place — new namespaces go in src/shared/i18n/locales/fa/, registered
   in src/shared/i18n/index.ts, and every new user-facing string must
   use useTranslation() rather than hardcoded JSX text, with the
   missing-key test guard from step 2 as the enforcement mechanism.

Files affected:
- Any file fixed by step 1's final grep (report the exact list, even
  if empty)
- src/shared/i18n/index.ts or src/test/setup.ts (for the missing-key
  handler configuration)
- Any file with a genuinely missing key surfaced in step 3
- A new end-to-end smoke test file (location per this project's
  existing integration-test conventions)
- README/CONTRIBUTING doc (only if such a file exists — report either
  way)

Acceptance Criteria:
- grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|left-[0-9]\|right-[0-9]\|text-left\|text-right\|border-l-\|border-r-\|rounded-l-\|rounded-r-\|space-x-" src/
  returns zero results project-wide
- npx vitest run (full suite, with missing-key detection now active)
  passes with zero failures
- npm run build succeeds
- The new end-to-end smoke test passes

Verification Steps:
1. grep -rn "ml-[0-9]\|mr-[0-9]\|pl-[0-9]\|pr-[0-9]\|left-[0-9]\|right-[0-9]\|text-left\|text-right\|border-l-\|border-r-\|rounded-l-\|rounded-r-\|space-x-" src/
2. npx vitest run
3. npm run build
4. Manual full walkthrough: run the dev server and click through every
   major flow (marketplace -> teacher profile -> booking -> checkout
   -> callback -> session -> review, plus auth pages and the teacher
   dashboard) confirming Persian text and correct RTL layout throughout
5. git diff --stat (final summary of this entire Phase)
```
