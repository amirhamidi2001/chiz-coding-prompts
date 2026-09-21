# Phase 2 — صفحات عمومی Frontend

## Implementation Prompts

### Prompt 1 — لایهٔ API + Types برای `content` و `contact`

```
Goal: Build the API client layer and TypeScript types for the new
content and contact features, following this project's exact existing
API-layer conventions. No pages/components/routes yet.

Before starting, read these files completely:
1. src/features/reviews/api/reviews.api.ts — the exact call convention
   (AxiosResponse wrapper return type, api.get/api.post usage,
   PaginatedResponse<T> import) to replicate precisely
2. src/features/teachers/api/teachers.api.ts — the closest existing
   precedent for a paginated list endpoint with query params
3. src/shared/types/api.types.ts — PaginatedResponse<T>'s exact shape
4. apps/content/serializers.py and apps/contact/serializers.py
   (backend, Phases 0-1) — the exact field names each serializer
   returns, to build matching TypeScript types precisely (do not
   guess field names)

What to build:

a) Create src/features/content/types/content.types.ts:
   export interface StaticPage {
     slug: string;
     title: string;
     body: string;
     updated_at: string;
   }
   export interface ArticleListItem {
     slug: string;
     title: string;
     excerpt: string;
     cover_image: string | null;
     category: string;
     author_name: string | null;
     published_at: string;
   }
   export interface Article extends Omit<ArticleListItem, "excerpt"> {
     body: string;
   }
   export type ArticleCategory =
     | "PROGRAMMING" | "SCIENCE" | "LANGUAGE" | "SPORTS"
     | "ART" | "MUSIC" | "COOKING" | "GENERAL";
   (Confirm this category list exactly matches
   apps.content.models.ARTICLE_CATEGORIES from Phase 0's backend — do
   not diverge from it.)

b) Create src/features/content/api/content.api.ts:
   export const contentApi = {
     getStaticPage(slug: string): Promise<AxiosResponse<StaticPage>> {
       return api.get<StaticPage>(`/api/content/pages/${slug}/`);
     },
     listArticles(params?: { category?: string; page?: number }): Promise<AxiosResponse<PaginatedResponse<ArticleListItem>>> {
       return api.get<PaginatedResponse<ArticleListItem>>("/api/content/articles/", { params });
     },
     getArticleBySlug(slug: string): Promise<AxiosResponse<Article>> {
       return api.get<Article>(`/api/content/articles/${slug}/`);
     },
   };
   (Match reviews.api.ts's exact object-literal-with-methods export
   style, not a class or standalone functions, for consistency.)

c) Create src/features/contact/types/contact.types.ts:
   export interface ContactFormData {
     name: string;
     email: string;
     subject: string;
     message: string;
   }

d) Create src/features/contact/api/contact.api.ts:
   export const contactApi = {
     submit(data: ContactFormData): Promise<AxiosResponse<void>> {
       return api.post<void>("/api/contact/", data);
     },
   };

Files affected:
- src/features/content/types/content.types.ts (new)
- src/features/content/api/content.api.ts (new)
- src/features/contact/types/contact.types.ts (new)
- src/features/contact/api/contact.api.ts (new)

Then add MSW handlers (check src/test/mocks/handlers/ for this
project's exact handler-file convention, e.g. reviews.handlers.ts, and
match it):
- src/test/mocks/handlers/content.handlers.ts: handlers for the three
  content.api.ts calls, with a couple of fixture StaticPage/Article
  objects
- src/test/mocks/handlers/contact.handlers.ts: a handler for POST
  /api/contact/ returning 204, and confirm this project's MSW setup
  file (wherever handlers are aggregated/registered) includes both new
  handler files

Files affected (continued):
- src/test/mocks/handlers/content.handlers.ts, contact.handlers.ts (new)
- src/test/mocks/fixtures/content.fixtures.ts, contact.fixtures.ts (new,
  if this project's convention separates fixtures from handlers — check
  reviews' equivalent files first)
- wherever handlers are aggregated (e.g. src/test/mocks/handlers.ts or
  similar) — add the two new handler arrays

Acceptance Criteria:
- npx vitest run passes (no test yet references these files, so this
  should trivially pass — the real check is that nothing else broke)
- npm run build succeeds (TypeScript types compile cleanly)

Verification Steps:
1. npx vitest run
2. npm run build
3. git diff --stat
```

### Prompt 2 — صفحات محتوایی ثابت (`StaticPageView`)

```
Goal: Build the shared StaticPageView component serving all five
static pages (about/terms/rules/privacy/faq), and wire its five routes.

Before starting, read:
1. src/features/content/api/content.api.ts, types/content.types.ts
   (Prompt 1)
2. src/features/teachers/pages/TeacherPublicProfilePage.tsx — the
   closest existing precedent for "a page fetching a single object by
   a route param via useQuery, with loading/error/not-found states" —
   match its exact useQuery/loading-skeleton/error-handling structure
3. src/app/routes/router.tsx, lazyPages.ts — the exact route
   registration pattern (Suspense + PageLoader wrapping, lazy() import)

What to build:

1. Create src/features/content/pages/StaticPageView.tsx:
   - Reads slug from useParams<{ slug: string }>()
   - useQuery(["static-page", slug], () => contentApi.getStaticPage(slug))
     with a longer staleTime than this project's default (e.g. 10
     minutes — document with a comment: "content changes rarely;
     avoid refetching on every navigation")
   - Loading state: a simple skeleton (reuse whatever loading
     component pattern TeacherPublicProfilePage already uses)
   - Error state: if the request 404s, render this project's existing
     NotFoundPage content inline or redirect to it (check how other
     pages in this project already handle a 404 from a detail-fetch
     query — follow that exact precedent rather than inventing a new
     approach)
   - Success: render page.title as an <h1>, and page.body with
     white-space: pre-wrap (via a Tailwind class, e.g. `whitespace-pre-wrap`)
     — deliberately NOT rendering any HTML/Markdown, since
     apps.content.models.StaticPage.body is a plain TextField on the
     backend (Phase 0's deliberate design choice) — document this
     with a comment so a future contributor doesn't assume this needs
     a markdown renderer

2. In src/app/routes/lazyPages.ts, add:
   export const StaticPageView = lazy(() => import("@/features/content/pages/StaticPageView"));

3. In src/app/routes/router.tsx, under the existing AppLayout children
   array, add five routes, each rendering the same StaticPageView
   component with a different fixed slug — since the component reads
   slug from useParams, the cleanest approach matching this project's
   routing style is a single dynamic route:
   { path: "/about", element: <Suspense fallback={<PageLoader />}><StaticPageView /></Suspense> }
   (repeated for /terms, /rules, /privacy, /faq)
   Rather than a single /:slug catch-all route, use five explicit
   static paths as shown — this keeps each URL explicit and
   independently linkable/crawlable (better for SEO than a generic
   dynamic segment), and StaticPageView can derive its slug from
   `useLocation().pathname.replace("/", "")` OR, cleaner, each route
   entry can pass a fixed slug prop instead of relying on useParams at
   all — reconsider the component's API: since there's no dynamic
   segment in the URL, StaticPageView should accept slug as a prop
   (element: <StaticPageView slug="about" />), not read it from
   useParams(). Revise Prompt 2's component design accordingly before
   implementing: StaticPageView(props: { slug: string }) is the
   correct, simpler API given five fixed static routes rather than one
   dynamic one.

Files affected:
- src/features/content/pages/StaticPageView.tsx (new)
- src/app/routes/lazyPages.ts
- src/app/routes/router.tsx

Then write tests: src/features/content/pages/__tests__/StaticPageView.test.tsx
covering:
- Renders the page title and body from a mocked API response
- Shows a loading state initially
- Shows a not-found state for a 404 response (mock the MSW handler to
  return 404 for one specific slug)

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds
- Manually visiting /about, /terms, /rules, /privacy, /faq in the dev
  server each renders correctly

Verification Steps:
1. npx vitest run
2. npm run build
3. Manual check in dev server for all five routes
4. git diff --stat
```

### Prompt 3 — صفحات مقالات (List + Detail)

```
Goal: Build ArticleListPage (paginated, category-filterable) and
ArticleDetailPage, and wire their routes.

Before starting, read:
1. src/features/content/api/content.api.ts, types/content.types.ts
2. src/features/teachers/pages/BrowseTeachersPage.tsx — the closest
   existing precedent for "a paginated, filterable list page" (its
   exact pagination-control usage, filter UI pattern, useQuery with
   query-params-as-query-key convention) — match its structure closely
3. src/features/reviews/components/ReviewCard.tsx — a small
   presentational card component precedent, for ArticleCard's
   structure

What to build:

1. Create src/features/content/components/ArticleCard.tsx: a card
   showing cover_image (with a sensible placeholder/fallback if null —
   check how TeacherCard or similar already handles a nullable image),
   title, excerpt, category (as a small badge/tag), author_name,
   published_at (formatted via formatJalaliDate from
   src/shared/lib/date.ts — the established Jalali-date function from
   this project's earlier roadmap, not a raw date string), linking to
   /articles/{slug}.

2. Create src/features/content/pages/ArticleListPage.tsx:
   - useQuery keyed on [category, page], calling
     contentApi.listArticles({ category, page })
   - A category filter control (simple select/tabs — reuse whatever UI
     primitive this project's other filter UIs already use, e.g.
     check BrowseTeachersPage's skill filter for the exact component)
   - Renders a grid of ArticleCard
   - Pagination controls matching this project's existing paginated-list
     UI pattern exactly (reuse the same pagination component
     BrowseTeachersPage already uses, don't build a new one)
   - Empty state: "No articles yet" (translated)

3. Create src/features/content/pages/ArticleDetailPage.tsx:
   - useQuery on slug (from useParams, since /articles/:slug IS a
     genuine dynamic route, unlike the five fixed static pages in
     Prompt 2)
   - Renders cover_image, title, category, author_name, published_at
     (formatJalaliDate), and body with white-space: pre-wrap (same
     plain-text rendering decision as StaticPageView, since
     Article.body is also a plain TextField per Phase 0 — do not
     introduce markdown rendering here either)
   - 404 handling identical to StaticPageView's approach from Prompt 2

4. In lazyPages.ts and router.tsx, add:
   /articles -> ArticleListPage
   /articles/:slug -> ArticleDetailPage

Files affected:
- src/features/content/components/ArticleCard.tsx (new)
- src/features/content/pages/ArticleListPage.tsx (new)
- src/features/content/pages/ArticleDetailPage.tsx (new)
- src/app/routes/lazyPages.ts
- src/app/routes/router.tsx

Then write tests:
- ArticleCard.test.tsx: renders all fields correctly, handles a null
  cover_image gracefully
- ArticleListPage.test.tsx: renders a list from mocked data; category
  filter triggers a refetch with the right query param; empty state
  renders correctly
- ArticleDetailPage.test.tsx: renders full article content; 404 case
  handled

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds
- Manual check: /articles lists mocked/seeded articles, clicking one
  navigates to /articles/:slug with full content

Verification Steps:
1. npx vitest run
2. npm run build
3. Manual check in dev server (with backend Phase 0 seeded with a
   published article)
4. git diff --stat
```

### Prompt 4 — فرم و صفحهٔ Contact

```
Goal: Build the ContactForm component and its page, following this
project's exact established form-with-mutation pattern.

Before starting, read:
1. src/features/reviews/components/SubmitReviewForm.tsx — the exact
   structural pattern to replicate: Zod schema with Persian validation
   messages, react-hook-form + zodResolver, useMutation, success/error
   toast handling via getApiErrorMessage from
   src/shared/lib/apiErrorMessage.ts (the established backend-error-code
   mapping layer from this project's earlier roadmap)
2. src/features/contact/api/contact.api.ts, types/contact.types.ts
   (Prompt 1)
3. src/shared/i18n/locales/fa/reviews.json — the closest existing
   precedent for a validation-message-containing namespace JSON file's
   structure

What to build:

1. Create src/shared/i18n/locales/fa/contact.json with keys for: page
   title/intro text, field labels (name/email/subject/message),
   placeholder text, validation messages (required, invalid email,
   max length), submit button, success message, and a specific message
   for the rate-limit case (HTTP 429 — check whether
   apps.contact.views.SubmitContactMessageView's throttle failure
   surfaces through getApiErrorMessage's existing code-mapping
   mechanism, or whether a 429 needs distinct handling since it may
   not carry an ApplicationError `code` field the way other backend
   errors do — check this project's apiErrorMessage.ts implementation
   for how it currently handles a throttled response, if at all; if it
   doesn't yet, add a specific case there mapping HTTP 429 to a
   Persian "too many requests, try again later" message, since this is
   the first Phase to actually expose an endpoint where hitting this
   limit is a realistic, expected user-facing scenario rather than a
   rare edge case).

2. Register the "contact" namespace in src/shared/i18n/index.ts.

3. Create src/features/contact/components/ContactForm.tsx:
   const contactSchema = z.object({
     name: z.string().min(1, t("contact:validation.nameRequired")).max(150),
     email: z.string().email(t("contact:validation.emailInvalid")),
     subject: z.string().min(1, t("contact:validation.subjectRequired")).max(200),
     message: z.string().min(1, t("contact:validation.messageRequired")).max(5000),
   });
   (Confirm the max lengths exactly match
   apps.contact.serializers.SubmitContactMessageSerializer's field
   constraints from Phase 1's backend — keep them in sync.)
   - useMutation calling contactApi.submit
   - On success: show a success message inline (not just a toast —
     replace the form with a confirmation state, since this is a
     one-time action the user won't repeat immediately) via
     t("contact:success")
   - On error: use getApiErrorMessage(error) for the displayed message,
     matching SubmitReviewForm's exact error-handling pattern

4. Create src/features/contact/pages/ContactPage.tsx: a thin page
   wrapper with a title/intro and <ContactForm />.

5. In lazyPages.ts and router.tsx, add /contact -> ContactPage.

Files affected:
- src/shared/i18n/locales/fa/contact.json (new)
- src/shared/i18n/index.ts
- src/shared/lib/apiErrorMessage.ts (only if the 429 case needed adding)
- src/features/contact/components/ContactForm.tsx (new)
- src/features/contact/pages/ContactPage.tsx (new)
- src/app/routes/lazyPages.ts
- src/app/routes/router.tsx

Then write tests: ContactForm.test.tsx covering:
- Submitting valid data calls contactApi.submit with the right payload
  and shows the success state
- Client-side validation errors (empty required field, invalid email)
  show the correct Persian messages without calling the API
- A mocked 429 response shows the rate-limit-specific message
- A mocked generic error shows the fallback message

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds
- Manual check: submitting the form against a running backend (Phase 1)
  creates a real ContactMessage row

Verification Steps:
1. npx vitest run
2. npm run build
3. Manual check against a running backend
4. git diff --stat
```

### Prompt 5 — لینک‌های Footer + بررسی نهایی End-to-End

```
Goal: Add navigation links to all seven new pages in the existing
Footer component, and do a final comprehensive review of this Phase.

Before starting, read src/shared/components/layout/Footer.tsx in full,
to match its existing structure/styling exactly (this project's Phase 5
roadmap already made this component RTL-correct and translated — any
new links added here must follow those same established i18n/RTL
conventions, not reintroduce hardcoded English text or physical
Tailwind directional classes).

What to build:

1. Add a "content" (or reuse "common") translation namespace section
   for footer link labels if Footer.tsx doesn't already have a
   suitable namespace/keys for this — check first, extend
   common.json's existing footer-related keys if present, or add new
   ones to content.json (Prompt 2) if that's a better fit given this
   project's existing key-organization convention.

2. In Footer.tsx, add a new links column/section (matching whatever
   grouping structure Footer.tsx already has — e.g. if it already
   groups links into "Product" / "Company" / "Legal" columns, put
   About under "Company", Terms/Rules/Privacy under "Legal", FAQ and
   Articles wherever fits best, and Contact under whichever column
   makes sense) linking to all seven pages: /about, /terms, /rules,
   /privacy, /faq, /articles, /contact. Use React Router's <Link>,
   matching how every other Footer link is already implemented (not a
   raw <a> tag).

Files affected:
- src/shared/i18n/locales/fa/common.json or content.json (only the new
  link-label keys)
- src/shared/components/layout/Footer.tsx

Then:
3. Run the full frontend test suite and fix anything Footer.tsx's
   change broke (e.g. an existing Footer snapshot/link-count test).

4. Do a final grep sweep:
   grep -rn "meet_link\|formatDate\b" src/features/content/ src/features/contact/
   (should return nothing — confirms Prompts 2-4 correctly used
   formatJalaliDate, not a leftover from copy-pasting an older
   component)
   grep -rln "ml-[0-9]\|mr-[0-9]\|text-left\|text-right" src/features/content/ src/features/contact/
   (should return nothing, per this project's established RTL
   convention from its earlier roadmap)

5. Add one end-to-end-style test (using this project's existing
   integration/router-level test conventions, if any exist — check
   src/test/integration/ first) that renders the full router and
   navigates through /about -> /articles -> clicking an article card
   -> /articles/:slug -> back to home -> /contact -> submitting the
   form, confirming each step renders the expected content without
   crashing.

Files affected (continued):
- one new integration test file (location per this project's existing
  convention)

Acceptance Criteria:
- npx vitest run passes the entire frontend test suite
- npm run build succeeds
- Both grep sweeps in step 4 return no results
- The new integration test passes

Verification Steps:
1. npx vitest run
2. npm run build
3. grep -rn "meet_link\|formatDate\b" src/features/content/ src/features/contact/
4. grep -rln "ml-[0-9]\|mr-[0-9]\|text-left\|text-right" src/features/content/ src/features/contact/
5. Manual full walkthrough in the dev server: Footer links -> each of
   the 7 new pages -> back to home
6. git diff --stat (final summary of this entire Phase)
```