# معماری و پرامپت‌ها: منوی عمومی هیربَد

## ۱. یافته‌های تازه از کد که طراحی را شکل داده‌اند

۱. **تست‌های فعلی Navbar عمداً لینک «کلاس‌های گروهی» را ممنوع کرده‌اند.** این لینک قبلاً در `DashboardLayout` هم برای مدرس نمایش داده می‌شد و حذف شد. `CommandPalette` همین قاعده را با `excludeRoles: ["TEACHER"]` اعمال می‌کند. اگر لینک‌های عمومی را روی `<Navbar />` بی‌قید بگذاریم، سه تست فعلی می‌شکنند و باگ قدیمی برمی‌گردد. بنابراین Navbar یک `variant` می‌گیرد و لینک‌ها فقط در `variant="public"` می‌آیند. لینک گروهی هم برای مدرس فیلتر می‌شود.

۲. **یک باگ دوم در همان ناحیه هست.** همبرگر `md:hidden` است (از ۷۶۸px پنهان می‌شود) ولی Sidebar فقط از `lg:block` (۱۰۲۴px) دیده می‌شود. بین ۷۶۸ تا ۱۰۲۳px، مثلاً در تبلت، در داشبورد هیچ ناوبری‌ای وجود ندارد. هر دو نقطه‌ی شکست را یکی می‌کنم: همبرگر `lg:hidden` و لینک‌های افقی `lg:flex`.

۳. ابزارهای پروژه، یعنی `Sheet`، `useLocale().dir` و الگوی `side={dir === "rtl" ? "right" : "left"}` در Sidebar، همه آماده‌اند. `ui/sheet.tsx` را دست نمی‌زنیم.

۴. گیت‌های CI شامل `typecheck`، `lint`، `lint:colors` (ممنوعیت رنگ hex/rgb) و `vitest` است. فقط token classهای Tailwind مجاز است.

## ۲. معماری هدف

```
src/shared/components/layout/
├── publicNav.ts            ← NEW  منبع واحد لینک‌ها + فیلتر نقش (بدون JSX)
├── PublicNavLinks.tsx      ← NEW  نوار افقی دسکتاپ (lg و بالاتر)
├── PublicNavSheet.tsx      ← NEW  کشوی موبایل/تبلت (زیر lg)
├── Navbar.tsx              ← EDIT variant + state منوی عمومی
├── AppLayout.tsx           ← EDIT <Navbar variant="public" />
├── DashboardLayout.tsx     ← EDIT <Navbar variant="dashboard" onMenuClick=… />
└── __tests__/
    ├── Navbar.test.tsx             ← EDIT (تست‌های جدید، قدیمی‌ها دست‌نخورده)
    ├── PublicNavSheet.test.tsx     ← NEW
    └── PublicNavLinks.test.tsx     ← NEW
shared/i18n/locales/fa/common.json  ← EDIT کلیدهای nav.*
test/integration/app-shell.smoke.test.tsx ← EDIT یک سناریوی «صفحه اصلی + همبرگر»
```

### ۲.۱ قرارداد `publicNav.ts`

```ts
type Role = "TEACHER" | "STUDENT";
interface PublicNavItem {
  id: string;
  to: string;
  labelKey: string;              // کلید namespace "common"
  excludeRoles?: Role[];
  placement: "primary" | "secondary";
}
// primary  → هم در نوار افقی دسکتاپ، هم در Sheet
// secondary → فقط در Sheet (و فوتر که خودش دارد)
export const PUBLIC_NAV_ITEMS: readonly PublicNavItem[];
export function getPublicNavItems(
  role: Role | undefined,
  scope: "primary" | "all"
): PublicNavItem[];
```

| id | مسیر | placement | excludeRoles |
|---|---|---|---|
| teachers | `/teachers` | primary | – |
| groupSessions | `/group-sessions` | primary | TEACHER |
| articles | `/articles` | primary | – |
| about | `/about` | secondary | – |
| faq | `/faq` | secondary | – |
| contact | `/contact` | secondary | – |

### ۲.۲ قرارداد `Navbar`

```ts
interface NavbarProps {
  variant?: "public" | "dashboard";   // default: "dashboard"
  onMenuClick?: () => void;           // فقط dashboard
}
```

مقدار پیش‌فرض عمداً `"dashboard"` است تا تست‌های فعلی که `<Navbar />` را بدون prop رندر می‌کنند بدون تغییر سبز بمانند.

| | `public` | `dashboard` |
|---|---|---|
| همبرگر (`lg:hidden`) | همیشه هست و `PublicNavSheet` را با state داخلی باز می‌کند | فقط اگر `onMenuClick` داده شده باشد |
| لینک افقی (`hidden lg:flex`) | `PublicNavLinks` | ندارد |
| `PublicNavSheet` | رندر می‌شود | رندر نمی‌شود |

### ۲.۳ رفتار `PublicNavSheet` به‌تفکیک کاربر

| وضعیت | لینک‌های وسط | پایین Sheet |
|---|---|---|
| مهمان | همه ۶ لینک | دکمه‌ی «ورود» و دکمه‌ی «ثبت‌نام رایگان» |
| دانش‌آموز | همه ۶ لینک | لینک «رفتن به داشبورد» |
| مدرس | ۵ لینک (بدون کلاس‌های گروهی) | لینک «رفتن به داشبورد» |

جزئیات رفتار:

- با کلیک روی هر لینک، Sheet بسته می‌شود (مثل `onNavigate` در Sidebar).
- لبه‌ی Sheet از `dir` گرفته می‌شود.
- از `SheetTitle` و `SheetDescription` برای a11y استفاده می‌شود (Radix بدون آن‌ها هشدار می‌دهد).
- لینک فعال با `NavLink` و کلاس‌های `bg-primary/10 text-primary` مشخص می‌شود.
- فقط از logical propertyها (`ms-`، `ps-`، `start-`، …) و token classها استفاده می‌شود.

### ۲.۴ کلیدهای i18n جدید (`nav.*`)

| کلید | متن |
|---|---|
| teachers | معلم‌ها |
| articles | مقالات |
| about | درباره ما |
| faq | سوالات متداول |
| contact | تماس با ما |
| mainNav | ناوبری اصلی |
| publicMenuTitle | منوی هیربَد |
| publicMenuDescription | دسترسی سریع به بخش‌های عمومی هیربَد |
| goToDashboard | رفتن به داشبورد |
| signupFree | ثبت‌نام رایگان |

کلیدهای `nav.groupSessions`، `nav.login`، `nav.signup` و `nav.openMenu` از قبل هستند.

### ۲.۵ تصمیم‌ها و ریسک‌ها

- **D1:** دکمه‌های ورود/ثبت‌نام هدر موبایل بدون تغییر می‌مانند و در Sheet هم تکرار می‌شوند. ریسک: در ۳۲۰px هدر مهمان شلوغ می‌شود (همبرگر، لوگو، جستجو، تم، ورود، عضویت). ایجنت باید در ۳۲۰ و ۳۶۰px بررسی کند. اگر سرریز دید، لینک «ورود» هدر را در `variant="public"` زیر `sm` پنهان می‌کنیم، چون در Sheet هست.
- **D2:** `CommandPalette`، `Footer` و `Sidebar` دست نمی‌خورند.
- **D3:** تغییر breakpoint همبرگر داشبورد از `md` به `lg` در همین کار انجام می‌شود (یافته‌ی ۲) و در توضیح PR جدا ذکر می‌شود.

## ۳. ترتیب اجرا

```
T1 (config + i18n) → T2 (کامپوننت‌ها) → T3 (سیم‌کشی Navbar/Layoutها)
                      ↘ T4 (تست‌ها) می‌تواند با T2 و T3 موازی شروع شود (قرارداد بخش ۲ ثابت است)
                                           → T5 (بازبینی مستقل)
```

### Preamble (به ابتدای هر پرامپت اضافه کن)

```
You are working in the repo `hirbad-platform`, directory `hirbad-frontend/`
(React 19, Vite, TypeScript, Tailwind + shadcn/ui, react-router-dom v6,
react-i18next with a single Persian/RTL locale, Vitest + Testing Library).

Project conventions you MUST follow:
- Every source file starts with a path comment, e.g. `// src/shared/components/layout/X.tsx`.
- Path alias `@/` maps to `src/`.
- RTL-first: use logical Tailwind utilities (ms-/me-/ps-/pe-/start-/end-), never left/right,
  except where an existing component's API is physical (ui/sheet.tsx `side`).
- No hardcoded colors (no hex/rgb/rgba). Use Tailwind token classes only
  (bg-background, text-muted-foreground, bg-primary/10, ...). `npm run lint:colors` enforces this.
- All user-facing strings go through `useTranslation("common")` with keys in
  `src/shared/i18n/locales/fa/common.json`. No inline Persian strings in components
  (the brand name "هیربَد" in the Navbar logo is the existing exception).
- eslint-plugin-react-refresh is on: component files export only components;
  put constants/helpers in separate .ts files.
- Do NOT modify: src/shared/components/ui/** (incl. sheet.tsx), CommandPalette.tsx,
  Footer.tsx, Sidebar.tsx, any backend file.
- Quality gates (run from hirbad-frontend/): `npm run typecheck`, `npm run lint`,
  `npm run lint:colors`, `npm test`. All must pass before you report done.
- Keep the diff minimal and scoped to the task below. Do not refactor unrelated code.

Background: the mobile hamburger in Navbar is dead on public pages because AppLayout renders
<Navbar /> without `onMenuClick` and there is no public menu. We are adding a public navigation
(desktop inline links + mobile/tablet Sheet), driven by one shared config.
```

### T1: کانفیگ و ترجمه

```
TASK T1 — shared public-nav config + i18n keys

1. Create `src/shared/components/layout/publicNav.ts` (no JSX) exporting:
   - `type PublicNavRole = "TEACHER" | "STUDENT"`
   - `interface PublicNavItem { id: string; to: string; labelKey: string;
       excludeRoles?: PublicNavRole[]; placement: "primary" | "secondary" }`
   - `const PUBLIC_NAV_ITEMS: readonly PublicNavItem[]` with exactly these entries, in this order:
       teachers      -> /teachers       nav.teachers      primary
       groupSessions -> /group-sessions nav.groupSessions primary  excludeRoles ["TEACHER"]
       articles      -> /articles       nav.articles      primary
       about         -> /about          nav.about         secondary
       faq           -> /faq            nav.faq           secondary
       contact       -> /contact        nav.contact       secondary
   - `function getPublicNavItems(role: PublicNavRole | undefined, scope: "primary" | "all"): PublicNavItem[]`
     Returns items filtered by (a) scope: "primary" => only placement "primary", "all" => every item;
     (b) role: drop items whose excludeRoles includes the given role. `undefined` role (guest) drops nothing.
   Add a short comment explaining the group-sessions rule mirrors CommandPalette's `excludeRoles`
   and that the marketplace link must never show for teachers.

2. Edit `src/shared/i18n/locales/fa/common.json`, inside the existing `"nav"` object, ADD (do not rename
   or remove existing keys):
     "teachers": "معلم‌ها",
     "articles": "مقالات",
     "about": "درباره ما",
     "faq": "سوالات متداول",
     "contact": "تماس با ما",
     "mainNav": "ناوبری اصلی",
     "publicMenuTitle": "منوی هیربَد",
     "publicMenuDescription": "دسترسی سریع به بخش‌های عمومی هیربَد",
     "goToDashboard": "رفتن به داشبورد",
     "signupFree": "ثبت‌نام رایگان"
   Keep valid JSON and the file's existing indentation/ordering style. Check
   `src/shared/i18n/__tests__` for any key-parity or snapshot test and keep it green.

3. Add `src/shared/components/layout/__tests__/publicNav.test.ts` (Vitest, pure unit test):
   - guest + "all" -> 6 items; guest + "primary" -> 3 items
   - role TEACHER -> no `groupSessions` in either scope
   - role STUDENT -> includes `groupSessions`
   - order of items is stable (teachers, groupSessions, articles, about, faq, contact)

Acceptance: all four quality gates pass. Report the files changed.
```

### T2: کامپوننت‌ها

```
TASK T2 — PublicNavLinks and PublicNavSheet (depends on T1 being merged)

First read: Sidebar.tsx (how it uses Sheet, `useLocale().dir`, NavLink active classes, `onNavigate`),
ui/sheet.tsx (exports available, e.g. SheetHeader/SheetTitle/SheetDescription), ui/button.tsx,
and Navbar.tsx `navLinkClass` for the existing link styling.

1. Create `src/shared/components/layout/PublicNavLinks.tsx`
   - Renders `<nav aria-label={t("nav.mainNav")} className="hidden items-center gap-1 lg:flex">`.
   - Items: `getPublicNavItems(user?.role, "primary")`; role from `useAuth()`
     (`@/app/providers/useAuth`). Each item is a react-router `NavLink` using the same visual
     classes as Navbar's `navLinkClass` (active: `text-primary bg-primary/10`; idle:
     `text-muted-foreground hover:text-foreground hover:bg-muted`), including focus-visible ring.
   - Named export `PublicNavLinks` + default export.

2. Create `src/shared/components/layout/PublicNavSheet.tsx`
   - Props: `{ open: boolean; onOpenChange: (open: boolean) => void }`.
   - Uses `Sheet`/`SheetContent`; `side={dir === "rtl" ? "right" : "left"}` with `dir` from
     `useLocale()` (copy the explanatory-comment approach from Sidebar.tsx); `className="w-72 p-0"`.
   - Accessible header: `SheetTitle` = t("nav.publicMenuTitle"), `SheetDescription` =
     t("nav.publicMenuDescription") (description may be `sr-only`). No Radix a11y warnings in test output.
   - Body: `<nav aria-label={t("nav.mainNav")}>` listing `getPublicNavItems(user?.role, "all")`
     as NavLinks styled like Sidebar items (`flex items-center rounded-md px-3 py-2 text-sm ...`),
     each with `onClick={() => onOpenChange(false)}`.
   - Footer area (border-t, p-4, space-y-2):
       * guest (`!isAuthenticated`): full-width primary link-button to `/login` (t("nav.login")) styled as an
         outline/secondary button, and full-width primary button to `/register` (t("nav.signupFree")).
         Use the existing Button + `asChild` + `Link` pattern if ui/button supports it; otherwise
         style Links with the same classes Navbar uses for its signup link.
       * authenticated: a single link to `/dashboard` with t("nav.goToDashboard").
     All footer links also close the sheet on click.
   - Do NOT render a logout control here (the Navbar user dropdown already has it).
   - Named export + default export.

Constraints: no new dependencies, no hardcoded colors, logical properties only, no edits to ui/*.
Acceptance: typecheck/lint/lint:colors/test pass (existing tests untouched and green). Components are
not wired into Navbar yet — that is T3.
```

### T3: سیم‌کشی

```
TASK T3 — wire variants into Navbar and layouts (depends on T2)

1. Edit `src/shared/components/layout/Navbar.tsx`:
   - Props become `{ variant?: "public" | "dashboard"; onMenuClick?: () => void }`,
     default `variant = "dashboard"`. Add a doc comment describing both variants and why the
     default is "dashboard" (existing tests render <Navbar /> bare and assert no group-sessions link).
   - Add `const [publicMenuOpen, setPublicMenuOpen] = useState(false)`.
   - Hamburger button:
       * change its visibility class from `md:hidden` to `lg:hidden` (it must align with Sidebar's
         `lg:block` so there is no 768–1023px dead zone);
       * `variant === "public"`: always rendered, onClick -> `setPublicMenuOpen(true)`;
       * `variant === "dashboard"`: rendered only when `onMenuClick` is provided; onClick -> onMenuClick.
       * add `aria-haspopup="dialog"` and `aria-expanded={publicMenuOpen}` only in the public variant.
   - In the left/start cluster after the logo link, when `variant === "public"` render
     `<PublicNavLinks />` (give it `ms-4` via a wrapper or className so spacing is logical).
   - Next to `<CommandPalette />` render `<PublicNavSheet open={publicMenuOpen} onOpenChange={setPublicMenuOpen} />`
     only when `variant === "public"`.
   - Do not change anything else (search button, ModeToggle, bell, user dropdown, auth links).
   - Check layout at 320px and 360px widths mentally/with a quick Playwright or devtools check if
     available: guest header (hamburger, logo, search, theme, login, signup) must not overflow
     horizontally. If it does, hide ONLY the header "login" NavLink below `sm` when
     `variant === "public"` (the Sheet contains it) and note this in your report.

2. Edit `AppLayout.tsx`: `<Navbar variant="public" />`.
3. Edit `DashboardLayout.tsx`: `<Navbar variant="dashboard" onMenuClick={() => setSidebarOpen(true)} />`.

4. Search for any other `<Navbar` usages (`grep -rn "<Navbar" src`) — AdminLayout may have its own
   header; only touch usages that clearly belong to the public/dashboard shells, and list what you found.

Acceptance: the three existing "does not render a Group Sessions link" tests and the Admin-link
tests pass unchanged. All quality gates pass. Provide a short manual test script (guest on `/`,
teacher on `/`, student on `/`, dashboard at 800px and 1100px widths).
```

### T4: تست‌ها

```
TASK T4 — tests (can start once the contracts in T1–T3 are fixed)

Use `@/test/utils` helpers (`render`, `renderAsTeacher`, `renderAsStudent`, `renderWithAuth`) and
`userEvent`. jsdom does not evaluate Tailwind breakpoints, so responsive-hidden elements are still in
the DOM: scope queries with `within(...)` and role/name to avoid ambiguity (the desktop
`PublicNavLinks` and the Sheet render the same link names).

A. Extend `layout/__tests__/Navbar.test.tsx` (KEEP all existing tests unchanged; add a new
   `describe("public variant")` and `describe("dashboard variant")`):
   public variant
   - guest: clicking button named "باز کردن منوی ناوبری" opens a `dialog`; inside it (within) the links
     معلم‌ها, کلاس‌های گروهی, مقالات, درباره ما, سوالات متداول, تماس با ما, ورود, ثبت‌نام رایگان exist with
     correct hrefs.
   - teacher: dialog has NO "کلاس‌های گروهی" link, has "رفتن به داشبورد" (href /dashboard), no ورود/ثبت‌نام رایگان.
   - student: dialog HAS "کلاس‌های گروهی" and "رفتن به داشبورد".
   - clicking a link inside the dialog closes it.
   - the desktop nav (`navigation` named "ناوبری اصلی", outside the dialog) lists exactly 3 primary
     items for a guest and 2 for a teacher.
   dashboard variant
   - `<Navbar variant="dashboard" onMenuClick={fn} />`: click calls fn once and no dialog opens.
   - `<Navbar variant="dashboard" />` (no handler): the hamburger button is not rendered.
   - default `<Navbar />` renders no "ناوبری اصلی" navigation landmark.

B. `layout/__tests__/PublicNavSheet.test.tsx`: controlled `open` prop; role filtering; guest vs
   authenticated footer; every link calls `onOpenChange(false)`; dialog has an accessible name
   ("منوی هیربَد") and no console.error from Radix about missing Title/Description (spy on console.error).

C. `layout/__tests__/PublicNavLinks.test.tsx`: renders 3/2 items by role; active link gets the active
   classes when routed to `/teachers` (use MemoryRouter initialEntries via the util's options if supported).

D. In `src/test/integration/app-shell.smoke.test.tsx`: add a scenario that mounts the real AppLayout route
   at `/` (follow the file's existing pattern), clicks the hamburger, and asserts the dialog appears
   and that navigating via "معلم‌ها" lands on the teachers page and closes the menu. This is the
   regression test for the original bug (dead hamburger on the landing page).

Acceptance: new tests fail if you revert the AppLayout change (verify by temporarily removing
`variant="public"` locally, confirm red, then restore). All gates green.
```

### T5: بازبینی مستقل (read-only)

```
TASK T5 — independent review (do NOT edit files; output a report)

Review the diff of the public-nav change in hirbad-frontend. Check and report PASS/FAIL with evidence
(file:line) for each:
1. Bug fixed: AppLayout's Navbar hamburger opens PublicNavSheet; no code path leaves a visible,
   inert hamburger (public always wired; dashboard hidden without handler).
2. Breakpoint consistency: hamburger `lg:hidden`, PublicNavLinks `lg:flex`, Sidebar `lg:block` —
   no width range with zero navigation in either shell.
3. Role rule: "group sessions" is never shown to TEACHER in desktop nav or Sheet; single source in publicNav.ts.
4. Existing Navbar tests unchanged and passing; default Navbar variant is "dashboard".
5. RTL: no physical left/right utilities introduced; Sheet side derived from `dir`.
6. No hardcoded colors; no inline Persian strings outside common.json; all new keys exist.
7. A11y: nav landmarks have distinct/unique labels per context, Sheet has title+description,
   hamburger has aria-label (+aria-expanded in public), focus returns to the hamburger on close,
   links are keyboard reachable, Esc closes.
8. No edits to forbidden files (ui/**, CommandPalette, Footer, Sidebar, backend).
9. Run all four quality gates and paste the summaries.
List any residual risks (e.g., 320px header crowding) and suggest follow-ups, without implementing them.
```
