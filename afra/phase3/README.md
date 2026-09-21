# Phase 3 — بازطراحی پالت رنگ

## Implementation Prompts

### Prompt 1 — محاسبه و اعمال پالت جدید در `globals.css`

```
Goal: Update the primary/secondary/muted/border/input/ring color
tokens in src/styles/globals.css to a "trust + focus" blue-based
palette suitable for a paid, transactional educational platform,
verifying WCAG AA contrast for every changed foreground/background
pair, in both light and dark mode. accent and every --status-* token
must remain completely untouched.

Before starting, read src/styles/globals.css in full (both the :root
and .dark blocks) and src/styles/themes.css in full, to understand
every existing token's current value and role.

What to build:

1. Compute the new color values. Use these exact target HSL values
   (chosen for hue in the 210-225 "trust blue" range, per this
   Phase's color-psychology rationale, with lightness/saturation
   tuned to preserve WCAG AA contrast against their paired
   foreground/background):

   :root (light mode):
     --primary: 217 76% 34%;              /* was: 194.7 83.9% 31.6% */
     --primary-foreground: 210 40% 98%;   /* was: 190 100% 97% */
     --secondary: 217 35% 93%;            /* was: 168.4 44.2% 91.6% */
     --secondary-foreground: 217 76% 34%; /* was: 194.7 83.9% 31.6% */
     --muted: 217 25% 95%;                /* was: 168.4 30% 94% */
     --muted-foreground: 217 12% 42%;     /* was: 195 15% 45% */
     --border: 217 25% 89%;               /* was: 192 30% 89% */
     --input: 217 25% 89%;                /* was: 192 30% 89% */
     --ring: 217 76% 34%;                 /* was: 194.7 83.9% 31.6% */

   .dark:
     --primary: 217 65% 55%;              /* was: 192.4 65.6% 40.2% */
     --primary-foreground: 217 40% 8%;    /* was: 195 60% 4.5% */
     --secondary: 217 30% 13%;            /* was: 195 40% 11.5% */
     --secondary-foreground: 217 20% 95%; /* was: 190 50% 96% */
     --muted: 217 25% 13%;                /* was: 195 35% 12% */
     --muted-foreground: 217 15% 68%;     /* was: 192 18% 68% */
     --border: 217 25% 13%;               /* was: 195 35% 12% */
     --input: 217 25% 13%;                /* was: 195 35% 12% */
     --ring: 217 65% 55%;                 /* was: 192.4 65.6% 40.2% */

   Leave --background, --foreground, --card, --card-foreground,
   --popover, --popover-foreground, --accent, --accent-foreground,
   --destructive, --destructive-foreground, --radius, and every single
   --status-* token in BOTH :root and .dark completely unchanged —
   copy them verbatim from the current file, do not touch a single
   character of those lines.

2. Before applying, verify contrast programmatically rather than
   trusting the numbers above blindly: write a short one-off Node or
   Python script (does not need to be committed to the repo — a
   scratch script is fine, or inline calculation) that converts each
   HSL pair to relative luminance and computes the WCAG contrast
   ratio for:
   - --primary-foreground text on --primary background (light AND dark)
   - --secondary-foreground text on --secondary background (light AND dark)
   - --muted-foreground text on --background (light AND dark, since
     muted-foreground text commonly appears directly on the page
     background, not just inside a --muted-colored box — check how
     this project's components actually use text-muted-foreground,
     e.g. grep -rn "text-muted-foreground" src/shared/components/ui/
     for a couple of real usage contexts, to confirm which background
     it's actually rendered against)
   Confirm every ratio is >= 4.5:1. If any pair computed above fails,
   adjust that specific value's lightness (not hue) slightly until it
   passes, and note the adjustment.

3. Apply the verified values to src/styles/globals.css, replacing only
   the nine token lines listed in step 1, in both the :root and .dark
   blocks — every other line in the file (including --accent and all
   --status-* tokens) must be byte-for-byte identical to before this
   change.

4. In src/styles/themes.css, add a new comment block near the top
   (above the existing "Status token reference" comment, not replacing
   it) documenting the palette choice, matching this file's existing
   documentation style:
   /*
     Brand palette (primary/secondary/muted/border/ring):
     hue ~217 (a classic "trust blue", the range used by most
     financial/fintech platforms), chosen deliberately over the
     previous teal-leaning hue (~195) because this platform is
     explicitly paid/transactional, not purely content-driven - see
     [this roadmap's Phase 3 architecture doc] for the full rationale.
     accent (~180, teal) and every --status-* token are unchanged -
     this palette pass is scoped to primary brand identity only, not
     the booking/session/payment status color system.
   */

Files affected:
- src/styles/globals.css (only the nine token lines, in both blocks)
- src/styles/themes.css (only the new comment block, nothing else)

Then write a test: src/styles/__tests__/contrast.test.ts (a plain
Vitest unit test, not a component test) that parses the actual HSL
values out of globals.css (read the file as text, regex-extract the
--primary/--primary-foreground/etc. values for both :root and .dark)
and asserts computed contrast ratios are >= 4.5 for every pair checked
in step 2 — this turns the manual verification into a permanent,
automated regression guard so a future palette tweak can't
accidentally reintroduce a contrast failure.

Acceptance Criteria:
- npx vitest run src/styles/__tests__/contrast.test.ts passes
  completely
- git diff src/styles/globals.css shows changes to exactly the nine
  token lines per block (eighteen total lines changed), nothing else
- --accent, --accent-foreground, and every --status-* token are
  confirmed byte-identical to before (diff shows zero changes to those
  lines)

Verification Steps:
1. npx vitest run src/styles/__tests__/contrast.test.ts
2. git diff src/styles/globals.css (manually confirm only the
   expected lines changed)
3. npm run build
4. git diff --stat
```

### Prompt 2 — تأیید عدم وجود رنگ Hardcode‌شده (Guard دائمی)

```
Goal: Add a permanent lint/test guard confirming no component
introduces a hardcoded color value that bypasses the CSS variable
token system — turning this Phase's initial audit finding ("zero
hardcoded colors today") into an enforced invariant, not just a
one-time observation.

Before starting:
1. Run: grep -rlnP '#[0-9a-fA-F]{3,8}\b|rgb\(|rgba\(' src/features
   src/shared/components --include=*.tsx --include=*.ts 2>/dev/null |
   grep -v test
   Confirm this still returns zero results (re-verify the finding
   Prompt 1's design relied on, since other Phases may have added code
   since the original audit).

What to build:

1. If the grep in step 1 is NOT empty (some hardcoded color was
   introduced by an earlier Phase of this roadmap, e.g. Phase 2's new
   content/contact components), fix each one found by replacing it
   with the appropriate Tailwind token class (e.g. bg-primary,
   text-muted-foreground) instead of a raw hex/rgb value — report
   exactly which files needed this fix.

2. Add a lint rule or a small custom script enforcing this going
   forward. Check this project's existing ESLint configuration
   (eslint.config.js or .eslintrc, whichever this project uses) for
   whether a rule like eslint-plugin-tailwindcss or a custom
   no-restricted-syntax rule could catch a hardcoded hex/rgb string
   inside a className or style prop. If a suitable existing ESLint
   plugin is already installed, configure and enable the relevant
   rule (e.g. tailwindcss/no-arbitrary-value or a no-restricted-syntax
   regex rule) scoped to src/features and src/shared/components. If no
   suitable plugin is installed and adding one is disproportionate for
   this single check, instead add a lightweight standalone script
   (e.g. scripts/check-no-hardcoded-colors.js, run via an npm script
   like "lint:colors") that runs the same grep-equivalent check and
   exits non-zero if it finds a match, and wire it into this project's
   CI workflow (.github/workflows/frontend-ci.yml) as an additional
   step alongside the existing lint/test steps — check that file's
   current structure first and add the new step in a style consistent
   with the existing ones.

Files affected:
- Any file fixed per step 1 (report the list, even if empty)
- Either an ESLint config change, or a new script +
  .github/workflows/frontend-ci.yml addition (per your choice in step 2
  — report which approach was taken and why)

Acceptance Criteria:
- The hardcoded-color check (however implemented) passes with zero
  violations
- The check is wired into CI, not just a one-off manual command
- npx vitest run passes the entire frontend test suite (no regression
  from any fix made in step 1)

Verification Steps:
1. Run the new check manually (whichever command it added, e.g.
   npm run lint:colors) and confirm it passes
2. npx vitest run
3. npm run build
4. git diff --stat
```

### Prompt 3 — بررسی بصری نهایی + مستندسازی

```
Goal: Final visual verification of the new palette across the most
color-sensitive existing pages, and a brief documentation note so
future contributors understand why the palette looks the way it does.

Before starting, review the diff from Prompts 1-2.

What to build:

1. Manual visual review (this step is inherently manual, not
   scriptable — perform it directly): run the dev server
   (npm run dev) and visually inspect, in both light and dark mode
   (toggle via whatever this project's existing theme switcher UI is
   — check src/app/providers/ThemeProvider.tsx's consumer, e.g. a
   settings page or navbar toggle, for how to switch modes manually):
   - The landing page (primary CTA buttons, hero section)
   - A form page with a submit button (e.g. the login page or the new
     Phase 2 contact form) — confirm the primary button's text is
     clearly readable
   - A page using --secondary heavily (check where secondary is most
     visibly used, e.g. badges or secondary buttons)
   - A focus state (tab to a button/input) to confirm --ring is
     visible and matches the new primary hue correctly

2. If any visual issue is found during step 1 that Prompt 1's computed
   values didn't anticipate (e.g. a specific component's own internal
   opacity/blend making a passing contrast ratio look worse in
   practice than the raw math suggested), adjust the specific token's
   lightness slightly in globals.css and re-run Prompt 1's contrast
   test to confirm it still passes after the adjustment — report any
   such adjustment made here.

3. Add a short paragraph to this project's README (or a CONTRIBUTING/
   design-system doc, if one exists — check first) under a "Design
   System" or "Theming" heading (create one if none exists and this
   project's README has no natural place for it — keep it brief, a
   few lines, not a full style guide) explaining: the palette lives
   entirely in src/styles/globals.css as HSL CSS custom properties,
   consumed via Tailwind's bg-primary/text-primary/etc. utilities;
   hardcoding a color value anywhere else is disallowed and enforced
   by Prompt 2's check; the current primary hue (~217) was chosen for
   "trust + focus" given this platform's paid/transactional nature,
   documented further in src/styles/themes.css.

Files affected:
- src/styles/globals.css (only if step 2 found an adjustment needed —
  report either way)
- README.md or an equivalent doc (a few new lines)

Acceptance Criteria:
- The manual visual review in step 1 confirms no readability issue in
  either light or dark mode across the four checkpoints listed
- If an adjustment was made in step 2, the contrast test from Prompt 1
  still passes after it
- npx vitest run passes the entire frontend test suite
- npm run build succeeds

Verification Steps:
1. npm run dev, manually walk through the four visual checkpoints in
   both light and dark mode
2. npx vitest run
3. npm run build
4. git diff --stat (final summary of this entire Phase)
```
