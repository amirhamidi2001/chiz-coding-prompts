# Phase 4 — کامپوننت‌های صفحهٔ اصلی (TopTeachers / LatestArticles / SkillCategoryGrid)

## Implementation Prompts

### Prompt 1 — افزودن `TeacherSkill.category` + Migration

```
Goal: Add a category field to the existing TeacherSkill model, aligned
with apps.content.models.ARTICLE_CATEGORIES (from this roadmap's Phase
0) for consistency across the platform's category taxonomy. No
selector/view/frontend changes yet.

Before starting, read:
1. apps/teachers/models.py — TeacherSkill's full current definition
2. apps/content/models.py — ARTICLE_CATEGORIES's exact current list
   (from this roadmap's Phase 0) — the new SKILL_CATEGORIES list here
   must use the identical set of values for consistency, since the
   same conceptual taxonomy (subject categories) now spans both
   articles and skills

What to build:

In apps/teachers/models.py, add above TeacherSkill:

SKILL_CATEGORIES = [
    ("PROGRAMMING", "Programming"),
    ("SCIENCE", "Science"),
    ("LANGUAGE", "Language"),
    ("SPORTS", "Sports"),
    ("ART", "Art"),
    ("MUSIC", "Music"),
    ("COOKING", "Cooking"),
    ("GENERAL", "General"),
]
(Confirm this list is character-for-character identical, in both
value and order, to apps.content.models.ARTICLE_CATEGORIES — if Phase
0's actual implemented list differs even slightly from this, use
Phase 0's real list as the source of truth here, not this prompt's
assumption.)

Add to TeacherSkill:
    category = models.CharField(
        max_length=20, choices=SKILL_CATEGORIES, default="GENERAL", db_index=True,
    )
Add a docstring note on the field or class explaining: "Defaults to
GENERAL for every existing skill row at migration time — teachers with
existing skills will need to manually re-categorize them for the
homepage's category-filtered browsing (Phase 4 of this roadmap) to
surface them correctly under their true subject; this is a known,
accepted limitation, not a bug."

Run: python manage.py makemigrations teachers
Review the migration — a plain AddField with default="GENERAL" is
correct and sufficient; no data migration is needed since the default
value itself is the intended behavior for existing rows.

Files affected:
- apps/teachers/models.py
- apps/teachers/migrations/00xx_*.py (new)

Acceptance Criteria:
- python manage.py makemigrations --check --dry-run is empty
- python manage.py migrate runs cleanly
- Every existing TeacherSkill row has category="GENERAL" after
  migrating

Verification Steps:
1. python manage.py makemigrations --check --dry-run
2. python manage.py migrate
3. python manage.py shell -c "from apps.teachers.models import TeacherSkill; print(TeacherSkill.SKILL_CATEGORIES if hasattr(TeacherSkill,'SKILL_CATEGORIES') else 'field added')"
4. git diff --stat
```

### Prompt 2 — فیلتر `category` در `list_teachers` + View + Serializer

```
Goal: Extend apps.teachers.selectors.list_teachers with a category
filter parameter, wire it through the existing TeacherListView, and
expose category in TeacherSkillSerializer.

Before starting, read these files completely:
1. apps/teachers/selectors.py — list_teachers's full current
   implementation and docstring (Prompt 1's field addition doesn't
   change this file yet)
2. apps/teachers/views.py — TeacherListView's full current
   implementation, specifically how it currently reads and passes the
   existing "skill" query param to the selector — match this exact
   pattern for the new "category" param
3. apps/teachers/serializers.py — TeacherSkillSerializer's current
   fields list

What to build:

a) In apps/teachers/selectors.py, modify list_teachers:
   Add a category: str | None = None parameter (keyword-only, matching
   the existing parameters' style). Add to the docstring:
   "category: Exact match against an active skill's category
   (SKILL_CATEGORIES value, e.g. 'PROGRAMMING'). Combinable with the
   `skill` filter above — a teacher must match both if both are
   given."
   In the filtering logic, add:
   if category:
       queryset = queryset.filter(teacher_profile__skills__category=category, teacher_profile__skills__is_active=True)
   IMPORTANT: since this filters across a many-to-many-like relation
   (a teacher can have several skills), and the existing `skill` filter
   likely already has this same concern, check how the existing skill
   filter avoids duplicate rows (does it already call .distinct()
   somewhere in this function, or rely on something else?) — if
   .distinct() isn't already applied, add queryset = queryset.distinct()
   at the end of the function (after all filters are applied, not
   conditionally only when category is set, so behavior is consistent
   regardless of which filters are active) to prevent a teacher with
   multiple matching skills appearing multiple times in the results.

b) In apps/teachers/views.py, in TeacherListView (wherever it currently
   reads query_params.get("skill") and passes it to
   selectors.list_teachers), add the same pattern for "category":
   category = request.query_params.get("category")
   ... list_teachers(..., category=category, ...)

c) In apps/teachers/serializers.py, add "category" to
   TeacherSkillSerializer's Meta.fields (wherever "name", "level", etc.
   are already listed).

Files affected:
- apps/teachers/selectors.py
- apps/teachers/views.py
- apps/teachers/serializers.py

Then write tests: extend tests/teachers/test_selectors.py and
tests/teachers/test_views.py with:
- list_teachers(category="PROGRAMMING") returns only teachers with an
  active PROGRAMMING skill
- A teacher with two PROGRAMMING skills appears exactly once (the
  distinct() regression test — this is the most important test in
  this prompt)
- Combining skill= and category= together works as an AND filter
- GET /api/teachers/?category=PROGRAMMING returns the filtered,
  paginated result via the API
- TeacherSkillSerializer's output includes "category"

Acceptance Criteria:
- pytest tests/teachers/ -v passes completely, including the
  duplicate-row regression test

Verification Steps:
1. pytest tests/teachers/ -v
2. git diff --stat
```

### Prompt 3 — کامپوننت `SkillCategoryGrid`

```
Goal: Build the icon-based skill category grid for the landing page,
linking each category icon to the filtered teacher browsing page.

Before starting, read these files completely:
1. src/features/landing/components/FeaturesSection.tsx — the closest
   existing precedent for "a grid of icon + label cards" in this
   project's landing feature — match its exact structural/styling
   pattern (it already uses lucide-react icons in a card grid, per
   LandingPage.tsx's FeaturesSection usage)
2. src/features/teachers/pages/BrowseTeachersPage.tsx — the full file,
   to confirm exactly how it currently reads "skill" from
   useSearchParams, so this prompt's second part (extending it for
   category) matches precisely
3. src/app/providers/useLocale.ts — for the RTL-aware forward-arrow
   pattern already established in LandingPage.tsx, if this component
   needs any directional icon (unlikely here, but check)

What to build:

a) Create src/features/landing/components/SkillCategoryGrid.tsx:
   - Accepts no required props (self-contained, matching
     FeaturesSection's pattern of taking translated content as props
     from LandingPage — decide whether categories are passed as a prop
     array from LandingPage.tsx, matching the `features`/`steps` array
     pattern already used there, or defined inline in this component;
     prefer matching the existing pattern: LandingPage.tsx builds the
     data array with icons + translated labels + hrefs, and passes it
     as a prop, exactly like `features`/`steps` are currently built
     and passed to FeaturesSection/HowItWorksSection)
   - Each grid item: an icon (lucide-react — pick one per category:
     Code2 for PROGRAMMING, FlaskConical for SCIENCE, Languages for
     LANGUAGE, Dumbbell for SPORTS, Palette for ART, Music for MUSIC,
     ChefHat for COOKING — omit GENERAL from the grid entirely, since
     it's a catch-all fallback category, not something a user would
     intentionally browse for), a translated label, wrapped in a
     React Router <Link to={`/teachers?category=${category}`}>
   - A responsive grid layout (reuse whatever grid/card Tailwind
     classes FeaturesSection already establishes for consistency, e.g.
     grid-cols-2 sm:grid-cols-4 or similar — match its exact responsive
     breakpoints rather than inventing new ones)
   - Each item should have a hover state (matching this project's
     existing interactive-card hover pattern elsewhere, e.g. TeacherCard)

b) In src/shared/i18n/locales/fa/landing.json, add a "skillCategories"
   section with one label per category (e.g. "برنامه‌نویسی", "علمی",
   "زبان خارجه", "ورزشی", "هنر", "موسیقی", "آشپزی") plus a section
   heading/subheading (e.g. "بر اساس زمینهٔ موردعلاقه‌تان جستجو کنید").

c) In src/features/teachers/pages/BrowseTeachersPage.tsx, extend the
   existing useSearchParams reading (the same block that currently
   reads "skill") to also read "category" the identical way, and pass
   it into whatever hook/query call already consumes the "skill"
   value (matching teachers.api.ts's Prompt 2 updated listTeachers
   signature from the backend side of this Phase). Also add a visible
   UI indicator on this page when a category filter is active (e.g. a
   small "Showing: Programming ×" chip the user can clear) — check
   whether an equivalent "active filter" indicator already exists for
   the "skill" filter on this page, and if so, replicate its exact
   pattern for "category" rather than inventing a new UI element.

Files affected:
- src/features/landing/components/SkillCategoryGrid.tsx (new)
- src/shared/i18n/locales/fa/landing.json
- src/features/teachers/pages/BrowseTeachersPage.tsx
- src/features/teachers/api/teachers.api.ts (only if listTeachers's
  call signature needs a category param added, matching backend
  Prompt 2 — check first)
- src/features/teachers/types/teacher.types.ts (add category to the
  TeacherSkill type, matching backend Prompt 2's serializer change)

Then write tests:
- SkillCategoryGrid.test.tsx: renders all seven category items (not
  GENERAL); clicking one navigates to the correct
  /teachers?category=X URL
- Extend BrowseTeachersPage.test.tsx: a category query param in the
  URL correctly filters the displayed results (using the MSW handler)
  and shows the active-filter indicator

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds
- Manual check: clicking a category icon on the landing page (once
  wired into LandingPage in a later prompt) correctly filters
  /teachers

Verification Steps:
1. npx vitest run
2. npm run build
3. git diff --stat
```

### Prompt 4 — کامپوننت `TopTeachersSection`

```
Goal: Build the top-teachers showcase section for the landing page,
reusing the existing teachers API (already sorted by average_rating
descending, per apps.teachers.selectors.list_teachers's established
default ordering — no backend change needed for this prompt).

Before starting, read:
1. src/features/teachers/api/teachers.api.ts — the exact listTeachers
   call signature and response shape
2. src/features/teachers/components/TeacherCard.tsx (or wherever the
   existing teacher-summary card component lives — check
   BrowseTeachersPage.tsx's imports to find it) — reuse this exact
   component, do not build a new teacher card from scratch
3. src/features/landing/components/StatsSection.tsx — a closest
   existing precedent for "a horizontally-arranged section pulling
   from small, fixed data" in the landing feature, for general styling
   consistency

What to build:

Create src/features/landing/components/TopTeachersSection.tsx:
- useQuery calling teachersApi.listTeachers({ page: 1 }) (relying on
  the API's existing default average_rating-descending order — do not
  add a new backend sort parameter, since the default already does
  exactly what's needed) — request only what's needed for display
  (check whether listTeachers's pagination supports a page_size
  override via query params; if StandardResultsPagination supports a
  page_size param, request e.g. page_size=6 to avoid over-fetching a
  full page just to show 6 cards — check this project's
  StandardResultsPagination implementation for whether it honors a
  client-supplied page_size before assuming it does)
- Renders a heading (translated, e.g. "مدرس‌های برتر") + a responsive
  grid of TeacherCard (reusing the existing component exactly, not a
  simplified variant) for the first 4-6 results
- A "View all teachers" link/button at the end, linking to /teachers
  (no filter — the general browse page)
- Loading state: a skeleton grid (reuse whatever skeleton pattern
  BrowseTeachersPage already uses for its own loading state, for
  visual consistency between this landing section and the full listing
  page)
- Empty state: if there are truly zero approved teachers yet (a
  realistic scenario for a freshly-launched platform), render nothing
  at all rather than an empty section with just a heading — document
  this with a comment, since an empty "Top Teachers" section with no
  cards looks broken/unfinished on a live site, whereas quietly
  omitting the whole section is graceful for a pre-launch or
  very-early-stage marketplace

Files affected:
- src/features/landing/components/TopTeachersSection.tsx (new)
- src/shared/i18n/locales/fa/landing.json (add topTeachers.heading,
  topTeachers.viewAll keys)

Then write tests: TopTeachersSection.test.tsx covering:
- Renders teacher cards from mocked API data
- Renders nothing when the API returns zero results (the empty-state
  decision above)
- The "View all" link points to /teachers

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds

Verification Steps:
1. npx vitest run
2. npm run build
3. git diff --stat
```

### Prompt 5 — کامپوننت `LatestArticlesSection`

```
Goal: Build the latest-articles showcase section for the landing page,
reusing this roadmap's Phase 0/2 content API and ArticleCard component
entirely.

Before starting, read:
1. src/features/content/api/content.api.ts — listArticles's exact
   call signature (from this roadmap's Phase 2)
2. src/features/content/components/ArticleCard.tsx — reuse this exact
   component (from Phase 2), do not build a new article card

What to build:

Create src/features/landing/components/LatestArticlesSection.tsx:
- useQuery calling contentApi.listArticles({ page: 1 }) (relying on
  Article's default ordering, already -published_at descending per
  Phase 0's Meta.ordering — "latest" is already the default order, no
  new backend parameter needed), requesting a small page_size (e.g. 3)
  the same way Prompt 4 did for teachers, if supported
- Renders a heading (translated, "آخرین مقالات") + a grid of
  ArticleCard (reused exactly from Phase 2) for the first 3 results
- A "View all articles" link to /articles
- Loading state: skeleton grid (reuse ArticleListPage's existing
  loading pattern from Phase 2, for consistency)
- Empty state: same reasoning as Prompt 4 — if there are zero
  published articles yet, render nothing rather than an empty section
  with just a heading (a realistic scenario before any content is
  published)

Files affected:
- src/features/landing/components/LatestArticlesSection.tsx (new)
- src/shared/i18n/locales/fa/landing.json (add latestArticles.heading,
  latestArticles.viewAll keys)

Then write tests: LatestArticlesSection.test.tsx covering:
- Renders article cards from mocked API data
- Renders nothing when the API returns zero results
- The "View all" link points to /articles

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds

Verification Steps:
1. npx vitest run
2. npm run build
3. git diff --stat
```

### Prompt 6 — ترکیب در `LandingPage` + بررسی نهایی End-to-End

```
Goal: Compose the three new sections into LandingPage.tsx in a
sensible order, and do a final comprehensive review of this Phase.

Before starting, read src/features/landing/pages/LandingPage.tsx in
full (its current exact composition order:
HeroSection -> StatsSection -> FeaturesSection -> HowItWorksSection ->
FinalCtaSection), to decide the most natural insertion points for the
three new sections without disrupting the page's existing narrative
flow.

What to build:

1. In LandingPage.tsx, insert the three new sections in this order
   (rationale: SkillCategoryGrid right after the hero gives visitors
   an immediate, concrete way to start browsing by interest before
   reading marketing copy; TopTeachersSection provides social proof
   after HowItWorksSection explains the process; LatestArticlesSection
   goes last, before the final CTA, as a lower-priority
   content/engagement touchpoint):

   HeroSection
   SkillCategoryGrid          <- NEW, right after Hero
   StatsSection
   FeaturesSection
   HowItWorksSection
   TopTeachersSection          <- NEW, after HowItWorksSection
   LatestArticlesSection       <- NEW, before FinalCta
   FinalCtaSection

   Import all three new components and render them with no required
   props beyond what Prompts 3-5 already defined (if SkillCategoryGrid
   ended up needing a data-array prop per Prompt 3's design decision,
   build that array here in LandingPage.tsx matching the exact
   `features`/`steps` array-building pattern already used for the
   existing sections).

2. Run the full frontend test suite and fix anything LandingPage's
   own existing test broke (e.g. a snapshot or section-count
   assertion).

3. Final grep sweep across this Phase's new files:
   grep -rln "ml-[0-9]\|mr-[0-9]\|text-left\|text-right" src/features/landing/components/SkillCategoryGrid.tsx src/features/landing/components/TopTeachersSection.tsx src/features/landing/components/LatestArticlesSection.tsx
   (should return nothing, per this project's established RTL
   convention)
   grep -rlnP '#[0-9a-fA-F]{3,8}\b|rgb\(' src/features/landing/components/SkillCategoryGrid.tsx src/features/landing/components/TopTeachersSection.tsx src/features/landing/components/LatestArticlesSection.tsx
   (should return nothing, per Phase 3's hardcoded-color guard)

4. Add one integration-style test rendering LandingPage in full (with
   mocked teacher/article API data) confirming all three new sections
   render together without error, in the correct order.

Files affected:
- src/features/landing/pages/LandingPage.tsx
- one new/extended integration test

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds
- Both grep sweeps in step 3 return no results
- Manual check: the landing page in the dev server shows all three new
  sections in the correct order, each populated with real data from a
  running backend (Phases 0/1/4's backend pieces)

Verification Steps:
1. npx vitest run
2. npm run build
3. grep -rln "ml-[0-9]\|mr-[0-9]\|text-left\|text-right" src/features/landing/components/SkillCategoryGrid.tsx src/features/landing/components/TopTeachersSection.tsx src/features/landing/components/LatestArticlesSection.tsx
4. grep -rlnP '#[0-9a-fA-F]{3,8}\b|rgb\(' src/features/landing/components/SkillCategoryGrid.tsx src/features/landing/components/TopTeachersSection.tsx src/features/landing/components/LatestArticlesSection.tsx
5. Manual full walkthrough in the dev server
6. git diff --stat (final summary of this entire Phase)
```
