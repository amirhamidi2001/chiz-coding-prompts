# معماری SSR صفحه‌های عمومی هیربَد با React Router v7 (Framework Mode)

> **نحوه‌ی خواندن:** بخش‌های توضیحی فارسی‌اند. Promptهای مخصوص Agent عمداً **انگلیسی** نوشته شده‌اند، چون Agentها با دستور انگلیسی دقیق‌تر کار می‌کنند. مسیر فایل‌ها از بررسی مستقیم repository آمده است. هرجا چیزی را ندیده‌ام، با «تأییدنشده» علامت زده‌ام و Prompt 0 آن را بررسی می‌کند. هیچ‌کدام از Promptها را بدون Prompt قبلی اجرا نکنید و بعد از هر Prompt خروجی را تأیید کنید.

## 1. هدف

- HTML خامی که سرور برای هر صفحه‌ی عمومی می‌فرستد، خودش شامل محتوای صفحه، `<title>`، `description`، `canonical`، `og:*` و JSON-LD یکتا باشد (بدون اجرای جاوااسکریپت).
- ربات‌های گوگل، بینگ و پیش‌نمایش لینک تلگرام، واتس‌اپ، توییتر و لینکدین، برای هر URL داده‌ی درست ببینند.
- صفحه‌ی ناموجود، وضعیت HTTP **404 واقعی** بگیرد (نه soft 404).
- ناحیه‌ی خصوصی (داشبورد، رزرو، پرداخت و ...) مثل قبل SPA بماند، `noindex` بگیرد و داده‌ی خصوصی هیچ‌وقت در HTML سرور نیاید.
- منطق کسب‌وکار، APIها، query keyها، کلیدهای i18n و ساختار feature-folder تا حد ممکن تغییر نکنند.

خارج از محدوده: SSR برای ناحیه‌ی خصوصی، تغییر طراحی UI، تغییر منطق رزرو یا پرداخت، چندزبانه‌سازی.

## 2. معماری

### 2.1 نمای کلی

```
Browser / Googlebot / Telegram
            |
   nginx (کانتینر web)
   |-- /assets/*    -> فایل ثابت (build/client)، immutable
   |-- /media/*     -> فایل ثابت (volume)
   |-- /api/*, /sitemap*.xml -> backend (Django)
   '-- همه‌ی مسیرهای HTML  -> ssr (Node، react-router-serve :3000)
                                |  INTERNAL_API_URL=http://backend:8000
                                '--> Django (فقط خواندن داده‌ی عمومی)
```

تفاوت با طرح قبلی در گفت‌وگو: چون در Framework Mode با `ssr: true` فایل `index.html` ثابت وجود ندارد، **همه‌ی درخواست‌های HTML به Node می‌روند**. مسیرهای خصوصی در Node فقط یک پوسته‌ی سبک (`ClientOnly`) رندر می‌کنند و هیچ داده‌ای از Django نمی‌گیرند. پس بازگشت به SPA ثابت در صورت خرابی Node ممکن نیست. به جای آن از **cache قدیمی nginx** (`proxy_cache_use_stale`) برای صفحه‌های عمومی استفاده می‌کنیم.

### 2.2 تصمیم‌های کلیدی

| # | تصمیم | دلیل |
| --- | --- | --- |
| 1 | `react-router.config.ts` با `ssr: true` و `appDirectory: "src/app"` | ساختار فعلی `src/app` حفظ شود |
| 2 | اول ارتقا به v7 در library mode، بعد فعال‌کردن framework mode | ریسک ارتقا از ریسک معماری جدا می‌شود |
| 3 | `loader` فقط برای داده‌ی **عمومی**؛ هیچ توکن یا کوکی به سرور نمی‌رود | refresh token در `localStorage` است و سرور نمی‌تواند کاربر را بشناسد |
| 4 | hookهای TanStack Query می‌مانند؛ خروجی loader به‌عنوان `initialData` به آن‌ها داده می‌شود | کمترین تغییر در کامپوننت‌ها و رفتار cache |
| 5 | `meta`، canonical و JSON-LD در `meta` هر route از روی داده‌ی loader ساخته می‌شود | جایگزین `RouteSeo` سمت مرورگر |
| 6 | cache کوتاه nginx برای HTML عمومی | کاهش فشار روی Node و تحمل خرابی کوتاه |
| 7 | sitemap از Django | داده‌ی معلم‌ها و مقالات آنجاست |
| 8 | پوسته‌ی خصوصی با `ClientOnly` | جلوگیری از hydration mismatch و نشت داده |

### 2.3 جدول مسیرها

مسیرها از `src/app/routes/router.tsx` و `nginx.conf.template` استخراج شده‌اند. نام دقیق APIها «تأییدنشده» است.

| مسیر | داده‌ی SSR (loader) | ایندکس | Cache nginx |
| --- | --- | --- | --- |
| `/` | داده‌ی فعلی لندینگ (دسته‌ها و ...) | بله | ۶۰ ثانیه |
| `/teachers` | لیست با `category`، `page`، `search` | بله (قواعد canonical پایین) | ۶۰ ثانیه |
| `/teachers/:teacherId` | جزئیات عمومی معلم، مهارت‌ها | بله، 404 اگر نبود | ۶۰ ثانیه |
| `/group-sessions`، `/group-sessions/:id` | لیست و جزئیات | بله، 404 اگر نبود | ۶۰ ثانیه |
| `/articles`، `/articles/:slug` | لیست و مقاله | بله، 404 اگر نبود | ۱۲۰ ثانیه |
| `/about`، `/faq`، `/contact`، `/terms`، `/rules`، `/privacy` | صفحه‌ی ثابت از content API | بله | ۳۰۰ ثانیه |
| `/register` | ندارد | بله | ۳۰۰ ثانیه |
| `/login`، `/verify-email`، `/forgot-password`، `/reset-password` | ندارد | `noindex` | ندارد |
| داشبورد، رزرو، پرداخت، پروفایل، اعلان‌ها و سایر مسیرهای خصوصی | ندارد (`ClientOnly`) | `noindex` | ندارد |
| `*` | loader با `data(null, {status: 404})` | `noindex` | ندارد |

**قواعد canonical:** همه‌ی query stringها حذف می‌شوند، به‌جز `page` وقتی بزرگ‌تر از ۱ باشد (خودارجاع). `category`، `search` و `ordering` به آدرس پایه canonical می‌شوند. وقتی صفحه‌های دسته‌بندی مسیرمحور ساخته شدند (Prompt 11)، `?category=` با 301 به آن‌ها می‌رود. نتیجه‌ی جستجو (`search`) و لیست خالی `noindex` می‌شوند.

### 2.4 جریان داده

1. درخواست HTML به nginx می‌رسد. اگر در cache بود، همان‌جا جواب داده می‌شود.
2. Node، route را تطبیق می‌دهد و `loader` را اجرا می‌کند. loader با `fetch` به `INTERNAL_API_URL` درخواست می‌زند، با timeout کوتاه (۳ ثانیه) و بدون کوکی یا `Authorization`.
3. پاسخ 404 Django به `throw data(null, {status: 404})` تبدیل می‌شود. خطای 5xx یا شبکه به **503** تبدیل می‌شود (هرگز 200 و هرگز cache نمی‌شود).
4. `meta` از روی `data` تگ‌ها را می‌سازد. کامپوننت HTML کامل را رندر می‌کند.
5. در مرورگر، `hydrateRoot` اجرا می‌شود. کامپوننت داده‌ی loader را به‌صورت `initialData` به hook فعلی می‌دهد. ناوبری‌های بعدی با `.data` به loader می‌روند و TanStack Query مثل قبل cache و refetch را مدیریت می‌کند.

### 2.5 احراز هویت، تم، i18n و زمان

- **Auth:** `AuthProvider` با وضعیت اولیه‌ی `"unknown"` شروع می‌کند و بازیابی نشست را در `useEffect` انجام می‌دهد. تا آن موقع Navbar یک جایگاه با ابعاد ثابت برای بخش حساب کاربری نشان می‌دهد (بدون CLS). سرور و اولین رندر کلاینت یکسان می‌شوند.
- **تم:** پیش‌فرض از CSS (`prefers-color-scheme`) و یک اسکریپت inline کوچک در `<head>` که کلاس تم را قبل از اولین paint از `localStorage` می‌گذارد. `<html>` دارای `suppressHydrationWarning` است.
- **i18n:** برای هر درخواست یک نمونه‌ی جدید i18next با منابع `fa` که به‌صورت همگام راه‌اندازی می‌شود (`useSuspense: false`).
- **زمان و اعداد:** سرور معمولاً UTC است و مرورگر کاربر نه. هر قالب‌بندی تاریخ یا ساعت یا باید با `timeZone` صریح (ثابت `APP_TIMEZONE`) در هر دو طرف یکسان باشد، یا فقط داخل `ClientOnly` رندر شود. در غیر این صورت hydration mismatch رخ می‌دهد.
- **QueryClient:** به‌جای singleton سطح module، با `useState(() => createQueryClient())` در `root.tsx` ساخته می‌شود تا بین درخواست‌ها داده‌ی کاربران مخلوط نشود.

### 2.6 سیستم SEO

- پوشه‌ی `src/shared/seo/` شامل `site.ts` (نام، توضیح پیش‌فرض، `og:image` پیش‌فرض)، `buildMeta.ts` و `jsonld.ts`.
- `buildMeta` یک آرایه‌ی **کامل** از تگ‌ها می‌سازد: `title`، `description`، `robots`، `canonical`، `og:*`، `twitter:*` و `script:ld+json`. در v7 خروجی `meta` فرزند، `meta` والد را به‌طور خودکار ادغام نمی‌کند، پس هر route همه‌چیز را خودش می‌دهد.
- JSON-LD: `Organization` و `WebSite` (صفحه‌ی اصلی)، `BreadcrumbList` (همه‌ی صفحه‌های داخلی)، `Person` (معلم)، `Event`/`Course` (کلاس گروهی)، `Article` (مقاله). `aggregateRating` فقط وقتی داده‌ی واقعی و قابل مشاهده در صفحه باشد.
- مبدأ سایت از `PUBLIC_SITE_URL` (متغیر محیطی runtime در سرور) خوانده می‌شود، نه ثابت در کد.
- مسیرهای خصوصی: هم `<meta name="robots" content="noindex,nofollow">` و هم هدر `X-Robots-Tag` از طریق export `headers`.
- برای عنوان ≤ حدود ۶۰ کاراکتر و توضیح ≤ حدود ۱۵۵ کاراکتر هدف‌گذاری می‌شود.

## 3. تغییرات Backend

مسیر Django: `hirbad-backend/apps/*`. نام اپ دقیق کلاس‌های گروهی و دسته‌بندی‌ها «تأییدنشده» است.

1. **ممیزی serializerهای عمومی** (لیست و جزئیات معلم، مقاله، صفحه‌ی ثابت، کلاس گروهی، دسته‌بندی). هیچ فیلد خصوصی (ایمیل، تلفن، مدارک هویتی، اطلاعات مالی، شناسه‌های داخلی) نباید در پاسخ باشد. چون داده‌ی loader داخل HTML سریالایز و cache می‌شود.
2. **فیلدهای لازم برای سئو** (اگر موجود نیستند):
   - مقاله: `published_at`، `updated_at`، `meta_description`، `cover_image`، نام نمایشی نویسنده.
   - معلم: `updated_at`، عنوان یا بیوگرافی کوتاه، میانگین و تعداد امتیاز، مهارت‌ها.
   - کلاس گروهی: شروع و پایان، قیمت، ظرفیت و جای خالی، معلم.
3. **Sitemap** با `django.contrib.sitemaps`: `/sitemap.xml` (index) و بخش‌های `static`، `teachers`، `articles`، `group-sessions` (و بعداً `categories`) با `lastmod`. فقط معلم تأییدشده و فعال، مقاله‌ی منتشرشده و کلاس گروهی آینده و عمومی. تنظیم `SITE_URL`. حذف `public/sitemap.xml` ثابت.
4. **دسترسی داخلی SSR:**
   - `ALLOWED_HOSTS` شامل `backend`.
   - چون `production.py` مقدار `SECURE_SSL_REDIRECT=True` دارد، باید `SECURE_PROXY_SSL_HEADER` تنظیم باشد و Node هدر `X-Forwarded-Proto: https` را بفرستد. وگرنه هر درخواست داخلی Node به https ریدایرکت می‌شود.
   - throttle ناشناس DRF همه‌ی درخواست‌های Node را یک IP می‌بیند. یک هدر محرمانه‌ی `X-Internal-Key` (مقدار `INTERNAL_API_KEY`) فقط برای GETهای عمومی از throttle معاف شود.
5. اختیاری: هدر `Cache-Control: public, max-age=0, s-maxage=60` روی GETهای عمومی.
6. فاز دسته‌بندی (Prompt 11): endpoint جزئیات دسته با `slug` و متن سئو.

## 4. تغییرات Frontend

مسیر: `hirbad-frontend/`. فایل‌های موجود از بررسی مستقیم repository هستند.

### 4.1 فایل‌های جدید

| فایل | کار |
| --- | --- |
| `react-router.config.ts` | `ssr: true`، `appDirectory: "src/app"` |
| `src/app/root.tsx` | `Layout` (html، head، Meta، Links، Scripts)، `App` با providerها، `ErrorBoundary` |
| `src/app/entry.client.tsx` | `hydrateRoot` و `HydratedRouter`، راه‌اندازی analytics بعد از hydration |
| `src/app/entry.server.tsx` | رندر stream، `isbot`، timeout، i18n هر درخواست |
| `src/app/routes.ts` | تعریف همه‌ی مسیرها (جایگزین `router.tsx`) |
| `src/app/routes/*.tsx` | route moduleهای نازک که صفحه‌ی موجود feature را export می‌کنند و `loader`، `meta`، `headers` می‌گیرند |
| `src/app/routes/healthz.ts` | resource route برای healthcheck |
| `src/shared/seo/{site,buildMeta,jsonld}.ts` | سیستم SEO |
| `src/shared/api/server.ts` و `features/*/api/*.server.ts` | کلاینت سمت سرور (بدون axios و بدون توکن) |
| `src/shared/api/query-client.ts` | factory برای QueryClient |
| `src/shared/hooks/useHydrated.ts`، `src/shared/components/ClientOnly.tsx` | رندر فقط سمت کلاینت |
| `src/shared/config/env.server.ts` | اعتبارسنجی متغیرهای سرور |

### 4.2 فایل‌های تغییرپذیر

| فایل | تغییر |
| --- | --- |
| `src/app/providers/AuthProvider.tsx` | وضعیت `unknown`، بازیابی نشست در effect |
| `src/app/providers/ThemeProvider.tsx` | بدون خواندن storage در رندر |
| `src/shared/hooks/useLocalStorage.ts` | خواندن storage در effect |
| `src/shared/api/axios.instance.ts` | محافظت دسترسی‌های `localStorage` و بدون اثر جانبی در import |
| `src/shared/i18n/*` | factory برای نمونه‌ی هر درخواست |
| `src/shared/lib/analytics.ts` | فقط سمت کلاینت |
| `src/shared/components/layout/{AppLayout,AuthLayout,DashboardLayout,Navbar}.tsx` | حذف `ScrollToTop` و `RouteSeo`، جایگاه ثابت auth در Navbar |
| `features/teachers/hooks/{useTeachers,useTeacherProfile}.ts`، `features/content/hooks/{useArticles,useStaticPage}.ts` و hookهای کلاس گروهی و لندینگ | پارامتر اختیاری `initialData` |
| صفحه‌های عمومی هر feature | خواندن `useLoaderData` و پاس‌دادن به hook |
| `package.json`، `vite.config.ts`، `tsconfig*.json`، `.gitignore` | اسکریپت‌ها، plugin، `.react-router/` |
| تست‌ها (`__tests__`، `src/test/integration/*`) | جایگزینی MemoryRouter و `createMemoryRouter` با `createRoutesStub` |

### 4.3 فایل‌های حذفی

`src/main.tsx`، `index.html`، `src/app/routes/router.tsx`، `src/shared/seo/RouteSeo.tsx`، `src/shared/seo/seoConfig.ts`، `src/shared/components/layout/ScrollToTop.tsx` (با `<ScrollRestoration/>` جایگزین می‌شود)، `public/sitemap.xml`.

## 5. تغییرات Database

برای فاز اصلی **migration اجباری وجود ندارد**، مگر اینکه فیلدهای سئو در Prompt 0 غایب باشند. همه‌ی migrationها باید backward-compatible باشند (nullable یا دارای default) تا استقرار بدون downtime ممکن باشد.

| مورد | فاز | توضیح |
| --- | --- | --- |
| `Article.meta_description` و `cover_image` | فقط اگر نبودند | برای snippet و `og:image` |
| `SkillCategory.slug` (unique) | فقط اگر نبود | برای مسیر دسته‌بندی |
| `SkillCategory.seo_title`، `seo_description`، `intro_text` | Prompt 11 | متن یکتا برای صفحه‌های دسته |
| ایندکس روی `updated_at` و فلگ‌های انتشار (`is_verified`، `is_published`) | Prompt 3 | سرعت sitemap |
| `TeacherProfile.slug` | اختیاری، فاز بعد | آدرس خوانا؛ فعلاً UUID می‌ماند |

## 6. تغییرات Infrastructure

فایل‌ها: `hirbad-frontend/Dockerfile`، `hirbad-frontend/nginx/nginx.conf.template`، `docker-compose.prod.yml` (ریشه‌ی repository)، نمونه‌ی متغیرهای محیطی.

1. **Dockerfile چندمرحله‌ای:**
   - `build`: `npm ci` و `react-router build` با خروجی `build/client` و `build/server`.
   - `ssr` (ایمیج اجرایی Node): با `node:20-alpine`، `npm ci --omit=dev`، کپی `build`، اجرا با کاربر غیر root، `HEALTHCHECK` روی `/healthz` و فرمان `react-router-serve ./build/server/index.js`.
   - `web` (nginx): با `nginx:alpine` و کپی `build/client` در `/usr/share/nginx/html`.
   - هر دو ایمیج از یک commit ساخته و با هم استقرار می‌شوند.
2. **docker-compose.prod.yml:** سرویس جدید `ssr` با `restart: unless-stopped`، محدودیت حافظه (تخمینی ۵۱۲ مگابایت؛ با اندازه‌گیری واقعی تنظیم شود)، `healthcheck`، متغیرهای `NODE_ENV=production`، `PORT=3000`، `INTERNAL_API_URL=http://backend:8000`، `INTERNAL_API_KEY`، `PUBLIC_SITE_URL` و `depends_on: backend`. سرویس nginx به `ssr` وابسته می‌شود. هیچ پورت به بیرون expose نمی‌شود.
3. **nginx:**
   - حذف whitelist مسیرها و `try_files /index.html`. وضعیت 404 حالا از خود اپلیکیشن می‌آید.
   - `/assets/` از فایل ثابت با `immutable`، `/media/` مثل قبل، `/api/` مثل قبل به backend.
   - `/sitemap.xml` و `/sitemap-*.xml` به backend. `robots.txt` فایل ثابت (با خط `Sitemap:`).
   - `location /` با `proxy_pass http://ssr:3000` و هدرهای `Host` و `X-Forwarded-*`. هدر `Cookie` به Node نرود (`proxy_set_header Cookie ""`) تا cache با کوکی‌های تبلیغاتی مثل `_ga` خراب نشود.
   - **توضیح اینکه دقیقاً چه چیزی cache می‌شود:** `proxy_cache_path` و zone با کلید `$scheme$host$request_uri`، `proxy_cache_valid 200 60s`، `404 10s`، هرگز 5xx، `proxy_cache_lock on`، `proxy_cache_background_update on` و `proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504`. مسیرهای خصوصی با `proxy_cache_bypass` و `proxy_no_cache` از cache خارج می‌شوند.
   - هدرهای امنیتی فعلی می‌مانند. نکته: TLS هنوز در این stack تنظیم نشده (طبق کامنت خود compose)، و این بخش با آن تداخل ندارد.
4. **مانیتورینگ:** لاگ JSON از stdout، زمان هر loader، خطاهای 5xx و `cache status` (`add_header X-Cache-Status $upstream_cache_status`).
5. **Rollback:** نگه داشتن tag ایمیج SPA قبلی و یک فایل nginx قبلی برای برگشت سریع (در `RUNBOOK.md` توضیح داده می‌شود).

## 7. Dependencies

| اقدام | بسته | توضیح |
| --- | --- | --- |
| اضافه | `react-router@^7` | هسته‌ی جدید (جایگزین `react-router-dom`) |
| حذف | `react-router-dom` | در v7 فقط re-export است |
| اضافه | `@react-router/node`، `@react-router/serve`، `isbot` | سرور و entry.server |
| اضافه (dev) | `@react-router/dev` | pluginهای Vite و typegen |
| احتمالی | `vite-tsconfig-paths` | فقط اگر alias فعلی کافی نباشد |
| احتمالی (dev) | `@playwright/test` | فقط در Prompt 13 |
| Node | ⋥20 | مطابق Dockerfile فعلی |

**تأییدنشده:** سازگاری نسخه‌ی Vite موجود در پروژه با `@react-router/dev`. این را Prompt 0 باید بررسی کند. اگر ناسازگار بود، قبل از ادامه متوقف شوید و صدا بزنید.

## 8. Risks

| # | ریسک | شدت | کاهش ریسک |
| --- | --- | --- | --- |
| 1 | **Hydration mismatch** (احراز هویت، تم، `Date`، `Intl`، منطقه‌ی زمانی، `matchMedia`) | بالا | Prompt 2 قبل از هر چیزی؛ `ClientOnly`؛ تست console در Prompt 13 |
| 2 | **نشت داده‌ی خصوصی**، چون خروجی loader داخل HTML و cache می‌ماند | بالا | ممیزی serializerها؛ تست کلیدهای مجاز؛ grep در HTML خام |
| 3 | ریدایرکت https در ارتباط داخلی Node→Django به‌خاطر `SECURE_SSL_REDIRECT` | بالا | هدر `X-Forwarded-Proto` و تست در Prompt 3 و 5 |
| 4 | throttle ناشناس DRF همه‌ی SSR را یک کاربر می‌بیند | بالا | کلید داخلی و معافیت محدود |
| 5 | باده شدن نگهبانی: صفحه‌ی خطا با وضعیت 200 یا cache شدن خطای 503 | متوسط | تبدیل خطاها به 4xx/5xx واقعی؛ تست nginx |
| 6 | مهاجرت تست‌ها: تست‌های فعلی با `MemoryRouter` و `createMemoryRouter` لایه‌ها را مستقیم مونت می‌کنند، و plugin فریم‌ورک در Vitest نباید فعال باشد | متوسط | `!process.env.VITEST` روی plugin و `createRoutesStub` |
| 7 | تغییرات شکننده‌ی v7 (حذف `json()` و `defer`، نام پکیج، `RouterProvider` از `react-router/dom`) | متوسط | Prompt 1 مستقل است و قبل framework mode اجرا می‌شود |
| 8 | ناسازگاری Vite با plugin | متوسط | بررسی در Prompt 0 |
| 9 | XSS در JSON-LD و HTML مقاله (مثلاً DOMPurify به `window` نیاز دارد و در سرور شکست می‌خورد) | متوسط | escape کردن `<`؛ sanitize در backend یا کتابخانه‌ی isomorphic |
| 10 | فشار CPU و حافظه‌ی Node در یک VPS | متوسط | cache ۶۰ ثانیه‌ای، timeout، محدودیت حافظه، تست بار سبک |
| 11 | version skew در deploy: HTML قدیمی به assetهای قدیمی اشاره می‌کند | متوسط | استقرار هم‌زمان `ssr` و `web`؛ نگهداشت assetهای بیلد قبلی یک چرخه |
| 12 | سئو: محتوای تکراری (فیلترها و `?page=`)، صفحه‌های کم‌محتوا، تغییر URL بدون 301 | متوسط | قواعد canonical؛ `noindex` برای صفحه‌های خالی؛ 301 برای URLهای قدیمی |
| 13 | تحویل `og:image` پیش‌فرض | کم | کار دستی: تصویر ۱۲۰۰×۶۳۰ برای `public/` تهیه شود |
| 14 | اسکریپت analytics و رضایت کاربر | کم | بارگذاری فقط بعد از hydration در کلاینت |

نکات مهم مهاجرت:

- هر Prompt یک branch و حداقل یک commit مستقل بگیرد تا revert ممکن باشد.
- تا Prompt 12 نباید چیزی در production تغییر کند. فازهای 0 تا 11 فقط روی branch و staging هستند.
- اجرای هم‌زمان چند Prompt روی یک branch توصیه نمی‌شود. Promptها به ترتیب وابسته‌اند.
- صفحه‌های مهم (لندینگ، پروفایل معلم) قبل از استقرار با `curl` و View Source بررسی شوند.

## 9. Acceptance Criteria (کلی پروژه)

**خروجی HTML خام**

1. `curl -sA "Googlebot" <URL>` برای ۱۲ مسیر عمومی کلیدی وضعیت 200 می‌دهد و HTML شامل `<h1>`، محتوای اصلی، `<title>` و `description` یکتا، `canonical` خودارجاع، `og:*` و JSON-LD معتبر است.
2. معلم، مقاله، کلاس گروهی و صفحه‌ی ناموجود و URL ناشناخته همه **404 واقعی** می‌دهند و `noindex` دارند.
3. همه‌ی مسیرهای خصوصی هدر `X-Robots-Tag: noindex` دارند و هیچ داده‌ی کاربری در HTML نیست.
4. در HTML خام هیچ کلید غیرمجاز (ایمیل، تلفن، نشانه‌ی توکن) وجود ندارد.
5. پیش‌نمایش لینک (`og:title`، `og:description`، `og:image`) برای هر صفحه یکتا است (با ابزار رسمی تلگرام و Facebook Debugger بررسی شود).

**رفتار برنامه**

6. هیچ خطای یا هشدار hydration در console برای ۱۲ مسیر کلیدی وجود ندارد.
7. ورود، refresh token، خروج، رزرو و پرداخت مثل قبل کار می‌کنند و تست‌های موجود پاس هستند.
8. در لاگ سرور هیچ `window is not defined` یا `localStorage is not defined` دیده نمی‌شود.

**سئو و sitemap**

9. `/sitemap.xml` XML معتبر است، همه‌ی معلم‌های تأییدشده، مقالات منتشرشده و کلاس‌های آینده را دارد و هیچ صفحه‌ی `noindex` در آن نیست.
10. Rich Results Test برای پروفایل معلم، کلاس گروهی و مقاله خطای بحرانی نمی‌دهد.
11. Lighthouse SEO ≥ 95 و CLS < 0.1 روی صفحه‌ی اصلی و لیست معلم‌ها (مقادیر هدف؛ با اندازه‌گیری واقعی اصلاح شود).

**عملکرد و تاب‌آوری**

12. TTFB صفحه‌های cacheشده کمتر از ۲۰۰ میلی‌ثانیه و cacheنشده کمتر از ۸۰۰ میلی‌ثانیه (p95، هدف اولیه).
13. با خاموش کردن کانتینر `ssr`، صفحه‌های عمومی قبلاً بازدیدشده حداقل ۵ دقیقه از cache نیامده سرو می‌شوند، و برای صفحه‌های بدون cache وضعیت 502 یا 503 برمی‌گردد.
14. همه‌ی `typecheck`، `lint`، `test`، `build` و `docker compose build` موفق‌اند.
15. `RUNBOOK.md` شامل مراحل استقرار، مانیتورینگ و rollback است و rollback یک‌بار روی staging امتحان شده است.

## 10. Implementation Prompts

قواعد استفاده: Promptها را **به ترتیب** و هرکدام در یک نشست جدید به Agent بدهید. قبل از هر Prompt، «Preamble» زیر را بالای آن paste کنید. بعد از هر Prompt خروجی (diff، نتیجه‌ی تست‌ها، چک‌لیست Acceptance) را بررسی کنید و فقط در صورت قبولی سراغ بعدی بروید.

### Preamble (قبل از هر Prompt paste شود)

```
You are working in the Hirbad monorepo: hirbad-frontend (React 19, Vite, TypeScript, TanStack Query, i18next, Tailwind, Vitest), hirbad-backend (Django + DRF), and docker-compose.prod.yml at the repo root. We are migrating the frontend to React Router v7 framework mode with SSR for PUBLIC pages only. Private pages stay client-rendered.

Global rules for every task:
1. Read before you write. Open every file you will change, plus its tests and callers. Write a 5-10 line summary of what you found before editing.
2. Minimal diff. Change only what the task requires. No drive-by refactors, renames, formatting sweeps, dependency upgrades or style changes.
3. Preserve the existing architecture (feature folders, hooks, API layer, query keys, i18n keys) unless the task explicitly says to refactor it.
4. Never put secrets, tokens or personal data into loader data, logs, tests or committed files.
5. Add or adjust tests for every behavior change. Run typecheck, lint and tests before and after, and report the real results. If something fails and the fix is outside scope, STOP and report instead of working around it.
6. Work on a branch named ssr/<prompt-number>-<slug>. Make small focused commits. Do not push or merge.
7. If an assumption in the task does not match the repo, STOP and report the mismatch.
8. Finish with: files changed, tests added, commands run with results, and the Acceptance Criteria checklist marked pass/fail.
```

### Prompt 0 — Baseline و inventory (فقط مستندسازی)

```
PROMPT 0 - Baseline and inventory (documentation only, NO source changes)

CONTEXT
We will migrate hirbad-frontend to React Router v7 framework mode with SSR for public pages. Before any change we need a verified baseline.

TASK
Create docs/ssr-migration/BASELINE.md with file:line references covering:
1. Versions and baseline command results. Versions: node, npm, react, react-dom, react-router-dom, vite, @vitejs/plugin-react, tailwind, vitest, i18next, @tanstack/react-query. Commands in hirbad-frontend: npm ci, typecheck, lint, test, build. Record pass/fail and test counts. If anything already fails, say so.
2. Route inventory from src/app/routes/router.tsx: path, layout (AppLayout/AuthLayout/DashboardLayout), guard components, how pages are loaded (Suspense/lazy). Classify each route as PUBLIC-INDEXABLE (/, /teachers, /teachers/:id, /group-sessions, /group-sessions/:id, /articles, /articles/:slug, /about, /faq, /contact, /terms, /rules, /privacy, /register), PUBLIC-NOINDEX (/login, /verify-email, /forgot-password, /reset-password) or PRIVATE (everything else).
3. Every use of window, document, navigator, localStorage, sessionStorage, matchMedia, Intl.*, Date and Math.random in non-test files under src/. Classify each as module-top-level, render-time, effect-time or event-handler.
4. How auth works (AuthProvider, src/shared/api/axios.instance.ts: where access and refresh tokens live, refresh flow), theme handling, i18n initialization (sync or async, Suspense), analytics init, every import.meta.env.VITE_* usage, presence of manifest or service worker.
5. How article body HTML is rendered and sanitized (library used; does it need window?).
6. Backend (hirbad-backend): URL patterns and serializer fields of the public endpoints used by public pages (teachers list/detail, articles list/detail, content pages, group sessions, skill categories); DRF throttle settings; ALLOWED_HOSTS; SECURE_SSL_REDIRECT and SECURE_PROXY_SSL_HEADER; which Django app owns group sessions; which of updated_at, published_at, meta_description, cover_image, slug exist on each public model.
7. All tests that mount the router or layouts.
8. Compatibility of the installed Vite and Node with react-router@7 and @react-router/dev (use npm view if network is available; otherwise say it could not be verified).
9. A 'Mismatches' section listing every place where the assumptions above differ from reality.

CONSTRAINTS: read-only except the new document. Do not install packages or run formatters.

ACCEPTANCE CRITERIA
- docs/ssr-migration/BASELINE.md exists with all 9 sections.
- git diff --stat shows only that file.
VERIFICATION: open the file and spot-check 5 file:line references.
```

### Prompt 1 — ارتقا به React Router v7 (Library Mode)

```
PROMPT 1 - Upgrade react-router-dom v6 to react-router v7 (library mode, NO framework mode yet)

CONTEXT
The app uses createBrowserRouter from react-router-dom ^6.30 in src/app/routes/router.tsx and src/main.tsx. Goal: move to v7 with zero behavior change. Framework mode comes in Prompt 4.

READ FIRST
package.json, src/main.tsx, src/app/routes/router.tsx, every file importing react-router-dom (rg 'react-router-dom'), every test that mocks it, docs/ssr-migration/BASELINE.md.

STEPS
1. Enable all v7_* future flags available in 6.30 on createBrowserRouter and RouterProvider. Run tests, fix warnings and regressions, commit.
2. Follow the official v6 to v7 upgrade guide: add react-router@^7, remove react-router-dom, change imports to 'react-router' (RouterProvider comes from 'react-router/dom').
3. If json() or defer() are used anywhere, replace them with plain returned objects.
4. Update tests and vi.mock paths accordingly.

CONSTRAINTS: no other dependency upgrades, no route changes, no formatting sweep.

TESTS: the existing suite must pass. Add one test that the router resolves '/' and a nonsense path to the not-found page.

ACCEPTANCE CRITERIA
- package.json has react-router@7 and no react-router-dom.
- rg 'react-router-dom' in src returns nothing.
- typecheck, lint, test and build pass.
VERIFICATION: npm run dev, then manually open /, /teachers, /login, /dashboard (redirects when logged out).
```

### Prompt 2 — اصلاح کد برای SSR-safe شدن (هنوز SPA)

```
PROMPT 2 - Make the client code SSR-safe (still a pure SPA after this prompt)

CONTEXT
Server rendering runs code in Node where window, document and localStorage do not exist, and the first client render must produce the same HTML as the server. Known browser-API users: src/app/providers/AuthProvider.tsx, src/app/providers/ThemeProvider.tsx, src/shared/hooks/useLocalStorage.ts, src/shared/api/axios.instance.ts (refresh token in localStorage key 'hirbad_refresh'), src/shared/i18n/*, src/shared/lib/analytics.ts, Navbar. Use docs/ssr-migration/BASELINE.md for the complete list.

READ FIRST
The files above, every item the baseline classifies as module-top-level or render-time, and their tests.

TASK
1. No module may touch window/document/localStorage at import time. Guard or move to effects.
2. AuthProvider: initial status is 'unknown' (same on server and first client render). Restore the session in useEffect with the existing logic. Navbar: while status is 'unknown' render a fixed-size placeholder for the account area so layout does not shift. Keep all behavior after restore identical.
3. ThemeProvider: do not read storage during render. Default comes from CSS prefers-color-scheme; apply the stored theme in an effect. Export a tiny inline script string (themeInitScript) that sets the theme class before first paint; it will be used by root.tsx in Prompt 4.
4. useLocalStorage: return the initial value on the first render, read storage in an effect.
5. axios.instance.ts: guard localStorage with typeof window checks; no side effects at import.
6. i18n: export createI18n() returning a NEW i18next instance with bundled fa resources initialized synchronously, useSuspense false. Keep the existing key structure. main.tsx uses it.
7. Add src/shared/api/query-client.ts exporting createQueryClient() with the current defaults, and use it in main.tsx (no module-level singleton).
8. Add src/shared/hooks/useHydrated.ts and src/shared/components/ClientOnly.tsx (renders fallback until mounted).
9. analytics: initialize only in the browser, after first render.
10. Any date or number formatting rendered in markup must use an explicit timeZone and locale from one shared constant (APP_TIMEZONE in src/shared/config); otherwise wrap in ClientOnly.

CONSTRAINTS: no visual change; no routing change; do not introduce framework-mode code.

TESTS
- A test file with '// @vitest-environment node' that imports each touched module and asserts no throw.
- Tests for AuthProvider initial 'unknown' then restored state, Navbar placeholder, useLocalStorage first-render value, ClientOnly, createI18n and createQueryClient.

ACCEPTANCE CRITERIA
- All touched modules import cleanly in a node environment.
- Existing tests pass; the app behaves identically in the browser (login, refresh, logout, theme toggle, language strings).
- No hydration-sensitive reads remain at render time in the touched files.
VERIFICATION: typecheck, lint, test; manual login/logout/theme check.
```

### Prompt 3 — Backend: audit داده‌ی عمومی، دسترسی داخلی SSR، sitemap

```
PROMPT 3 - Backend: public data audit, internal SSR access, sitemap

CONTEXT
The Node SSR server will call Django over the internal Docker network (http://backend:8000) and embed the returned JSON in the HTML, which is then cached. So public serializers must not contain private fields, and Django must accept internal requests. production.py sets SECURE_SSL_REDIRECT=True. DRF anonymous throttling would see all SSR traffic as one IP.

READ FIRST
hirbad-backend/config/settings/*.py, apps/teachers, apps/content, the app that owns group sessions, skill/category models, their urls.py, serializers, views, permissions and tests; docs/ssr-migration/BASELINE.md section 6.

TASK
1. Audit the public serializers (teacher list/detail, articles, content pages, group sessions, categories). Add tests asserting the EXACT set of keys returned to anonymous users. If a private field (email, phone, identity documents, financial data, internal ids) is exposed, remove it from the public serializer only if the frontend does not use it; if the frontend uses it, STOP and report.
2. Add missing read-only SEO fields only if absent and only where needed (see design section 3). Any migration must be additive and nullable or defaulted.
3. Settings: add 'backend' to ALLOWED_HOSTS; confirm SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https') is set; add SITE_URL and INTERNAL_API_KEY settings read from env, documented in .env.prod.example.
4. Throttle exemption: for safe GET requests on public endpoints, a request carrying header X-Internal-Key equal to INTERNAL_API_KEY (constant-time compare) skips anonymous throttling. Without the key, behavior is unchanged.
5. Sitemaps with django.contrib.sitemaps at /sitemap.xml (index) with sections: static pages, teachers (verified AND active), articles (published), group sessions (future AND public). Include lastmod. Wire URLs in config/urls.py. Add select_related/prefetch_related to avoid N+1.
6. Do not delete public/sitemap.xml yet (Prompt 12 does).

CONSTRAINTS: do not change business logic, permissions of private endpoints, or response shapes the frontend relies on.

TESTS (use the repo's existing runner)
- Exact-key tests for each public serializer.
- Sitemap contains verified teachers and published articles, and excludes unverified, draft and past items.
- Throttle exemption works only with the correct key.
- Request with Host: backend and X-Forwarded-Proto: https is not redirected.

ACCEPTANCE CRITERIA
- /sitemap.xml returns valid XML in the test client.
- No private field appears in any public response.
- All backend tests pass; new migrations (if any) are backward-compatible.
VERIFICATION: run the backend test suite; run manage.py makemigrations --check and report.
```

### Prompt 4 — فعال‌سازی Framework Mode (SSR روشن، بدون loader)

```
PROMPT 4 - Enable React Router framework mode with SSR (no loaders yet)

CONTEXT
Prompts 1 and 2 are done. Now convert the app shell to framework mode. All existing pages must render unchanged. Only structure changes. Private routes render a ClientOnly shell on the server.

READ FIRST
package.json, vite.config.ts, tsconfig*.json, index.html, src/main.tsx, src/app/routes/router.tsx, src/app/providers/*, layouts under src/shared/components/layout/, src/shared/seo/RouteSeo.tsx, all tests that mount the router or layouts, docs/ssr-migration/BASELINE.md.

STEPS
1. Add deps: @react-router/dev (dev), @react-router/node, @react-router/serve, isbot. Verify compatibility with the installed Vite first; if incompatible, STOP and report.
2. react-router.config.ts: { appDirectory: 'src/app', ssr: true }.
3. vite.config.ts: add the reactRouter() plugin only when process.env.VITEST is not set; keep aliases, Tailwind and test config.
4. src/app/root.tsx: export Layout (html lang='fa' dir='rtl' suppressHydrationWarning; head with Meta, Links and the themeInitScript inline; body with children, ScrollRestoration, Scripts), default App (QueryClientProvider via useState(() => createQueryClient()), I18nextProvider, ThemeProvider, AuthProvider, existing global providers, Outlet), and ErrorBoundary. Move everything static from index.html (favicons, manifest, theme-color, font preloads, Organization JSON-LD) into links()/meta() in root.tsx.
5. src/app/entry.client.tsx: hydrateRoot with HydratedRouter in startTransition; initialize analytics after hydration. src/app/entry.server.tsx: based on the framework default, using renderToPipeableStream, isbot, a 5s abort timeout, and a fresh createI18n() instance per request.
6. src/app/routes.ts: reproduce EVERY route of the old router.tsx 1:1 using route/index/layout. Create thin route modules in src/app/routes/ that default-export the existing page components. Layout routes use the existing AppLayout, AuthLayout, DashboardLayout. The private DashboardLayout subtree wraps its content in ClientOnly. Keep existing guard components. Add a splat route '*' for NotFound. Add src/app/routes/healthz.ts as a resource route returning 'ok'.
7. Replace ScrollToTop with ScrollRestoration (root), remove the RouteSeo mount from layouts (leave the files until Prompt 6).
8. Delete src/main.tsx, index.html and the old router.tsx.
9. package.json scripts: dev = react-router dev; build = react-router build; start = react-router-serve ./build/server/index.js; typecheck = react-router typegen && tsc --noEmit. Add .react-router/ to .gitignore and include .react-router/types/**/* in tsconfig (follow the official template).
10. Update tests: use createRoutesStub or MemoryRouter for component tests; keep Vitest working without the framework plugin.

CONSTRAINTS: do not change page components, hooks or API modules; do not add loaders; do not touch backend or infra files.

ACCEPTANCE CRITERIA
- npm run typecheck, lint, test and build pass; npm start serves the app on :3000.
- curl http://localhost:3000/ returns HTML that already contains the landing hero text (not an empty root div).
- Private routes return 200 HTML shell without errors in the server log; unknown paths return the NotFound page with HTTP 404.
- No hydration warnings in the browser console on /, /teachers, /login.
- Login, refresh, logout and theme toggle still work.
VERIFICATION: commands above plus a manual click-through of landing, teachers, login, dashboard.
```

### Prompt 5 — لایه‌ی داده‌ی سرور و تنظیمات محیطی

```
PROMPT 5 - Server data layer and environment

CONTEXT
Loaders run in Node, where axios with a relative base URL ('/') does not work and where tokens are not available. We need a small server-only API client that talks to Django on the internal network and never forwards user credentials.

READ FIRST
src/shared/api/axios.instance.ts, every features/*/api/*.api.ts used by public pages (teachers, content, group sessions, landing, categories), their types, .env.example, docs/ssr-migration/BASELINE.md.

TASK
1. src/shared/config/env.server.ts: validate (with the validation library already used in the repo, otherwise plain checks) INTERNAL_API_URL (default http://localhost:8000 only when NODE_ENV is not production), INTERNAL_API_KEY, PUBLIC_SITE_URL. Throw clear errors at startup in production when missing.
2. src/shared/api/server.ts: fetchJson<T>(path, { request, searchParams }) using fetch. Headers: Accept: application/json, Accept-Language: fa, X-Forwarded-Proto: https, X-Internal-Key. NO cookies and NO Authorization. Timeout 3s with AbortSignal. Status mapping: 404 -> throw data(null, {status: 404}); other 4xx -> 404 for detail endpoints; 5xx, timeout or network error -> throw data(null, {status: 503}). Never return a 200 for an upstream failure.
3. Server functions per feature as features/<feature>/api/*.server.ts that mirror the existing API modules and REUSE their response types: for example getTeacherById, listTeachers, getArticleBySlug, listArticles, getStaticPage, listGroupSessions, getGroupSession, listCategories (names should follow existing naming).
4. Update .env.example and docs with the new variables. Make sure *.server.ts files are never imported from client code.

CONSTRAINTS: do not modify existing axios API modules; do not duplicate types.

TESTS: with a fetch mock, test header forwarding (and the absence of cookie/authorization), status mapping (404, 400, 500, timeout), query-string building, and env validation.

ACCEPTANCE CRITERIA
- Server API module compiles and its tests pass; no client bundle imports a .server file (build passes).
- Upstream 500 and timeouts become 503, never 200.
VERIFICATION: typecheck, lint, test, build.
```

### Prompt 6 — لایه‌ی SEO و جایگزینی RouteSeo

```
PROMPT 6 - SEO helper layer and removal of client-side RouteSeo

CONTEXT
SEO tags are currently set in the browser by src/shared/seo/RouteSeo.tsx from src/shared/seo/seoConfig.ts. With SSR, each route must emit its own tags from its route module. In React Router v7, a child route's meta() does NOT merge with its parent, so each route returns the complete set.

READ FIRST
src/shared/seo/*, src/app/root.tsx, route modules from Prompt 4, docs/ssr-migration/BASELINE.md.

TASK
1. src/shared/seo/site.ts: site name, default description, default og:image path (/og-default.png), locale fa_IR, siteUrl() that reads PUBLIC_SITE_URL on the server (passed to the client via root loader data).
2. src/shared/seo/buildMeta.ts: buildMeta({ title, description, path, noindex, image, type, jsonLd }) returns a COMPLETE array of meta descriptors: title, description, robots, canonical link ({ tagName: 'link', rel: 'canonical', href }), og:title, og:description, og:url, og:type, og:image, og:locale, og:site_name, twitter:card/title/description/image, and one { 'script:ld+json': ... } per JSON-LD object. canonicalFor(url) keeps only the page param (when > 1) and drops every other query parameter and any trailing slash.
3. src/shared/seo/jsonld.ts: builders for Organization, WebSite, BreadcrumbList, Person (teacher), Event or Course (group session), Article. Output must be safe inside a script tag (escape '<' as \u003c).
4. src/shared/seo/headers.ts: noindexHeaders() and publicCacheHeaders({ sMaxAge, swr }) returning Headers (X-Robots-Tag: noindex, nofollow; Cache-Control: public, max-age=0, s-maxage=N, stale-while-revalidate=M).
5. root.tsx meta(): site-wide defaults only (og:site_name, og:locale, twitter:card, Organization and WebSite JSON-LD on the home route only).
6. Port titles/descriptions from seoConfig.ts into the route modules as static meta for routes without loaders (/register, /login, etc.). Private routes and auth pages use noindex meta plus noindexHeaders in headers().
7. Delete RouteSeo.tsx and seoConfig.ts once nothing imports them.

CONSTRAINTS: no loaders in this prompt; no visual change.

TESTS: unit tests for buildMeta (complete array, noindex, canonical rules incl. ?page=2 and ?category=x), JSON-LD escaping of </script>, header helpers, and a route-module test for the /login and /dashboard meta.

ACCEPTANCE CRITERIA
- curl of / and /register shows distinct title, description and canonical in raw HTML; /login and /dashboard are noindex in meta AND the X-Robots-Tag header.
- RouteSeo.tsx and seoConfig.ts are gone; typecheck, lint, test and build pass.
VERIFICATION: npm run build && npm start; curl -s localhost:3000/register | rg -i '<title>|canonical|robots'.
```

### Prompt 7 — صفحه‌ی اصلی و صفحه‌های ثابت

```
PROMPT 7 - Landing page and static content pages (loaders + meta)

CONTEXT
Prompts 0-6 are done. Now the first real SSR data. Routes: /, /about, /faq, /contact, /terms, /rules, /privacy, /register. Data hooks live in features/landing and features/content (for example useStaticPage in src/features/content/hooks/useStaticPage.ts).

READ FIRST
The route modules for these paths, the page components and the hooks they use, src/features/landing/**, src/features/content/**, the server API functions from Prompt 5, src/shared/seo/*.

TASK
1. For each route add loader() that fetches the same data the page's hook fetches today, using the server API functions. Unknown static slug or missing content -> 404 via data(null, {status: 404}).
2. Pass loader data to the existing hook as initialData (add an optional options argument to the hook; keep default behavior unchanged for other callers). Use useLoaderData in the route module and pass props down; do not rewrite the page components beyond that.
3. meta(): use buildMeta. Home: title 'معلم خصوصی آنلاین | کلاس خصوصی و گروهی با معلم متخصص - هیربَد', Organization + WebSite JSON-LD, h1 stays the current landing H1. Static pages: title from the page content with the site suffix, description from the first ~155 characters of plain text, BreadcrumbList JSON-LD. If the page data is missing, return noindex meta.
4. headers(): publicCacheHeaders (home 60s, static pages 300s, stale-while-revalidate 300s).
5. /register: no loader, indexable meta, no-cache default headers.
6. The landing hero animations (animate-slide-up and fade) must not cause hydration warnings; keep the existing CSS.

CONSTRAINTS: no copy or style changes; do not change the API modules.

TESTS: loader unit tests with a mocked server API (success, 404, 503); meta unit tests; render test with createRoutesStub confirming the page renders from loader data.

ACCEPTANCE CRITERIA
- curl -sA Googlebot localhost:3000/ and /about contain the page text, a unique title/description, canonical and JSON-LD.
- /about-does-not-exist returns 404; stopping the backend makes /faq return 503 (not 200).
- No hydration warnings; tests, typecheck, lint, build pass.
VERIFICATION: curl commands against npm start with the backend running, then stopped.
```

### Prompt 8 — فهرست و پروفایل معلم‌ها

```
PROMPT 8 - Teachers list and teacher profile (loaders, meta, JSON-LD)

CONTEXT
Routes /teachers (query params category, page, search, ordering as used by useTeachers) and /teachers/:teacherId (UUID today). Hooks: src/features/teachers/hooks/useTeachers.ts and useTeacherProfile.ts. These are the most valuable SEO pages.

READ FIRST
The route modules, features/teachers/** (pages, hooks, api, types), BASELINE sections 2 and 6, the booking/availability components and anything that depends on the logged-in user or on time zones.

TASK
1. List loader: parse the same search params the hook uses, call listTeachers, return { items, count, page, params }. Pass as initialData to useTeachers(params, { initialData }); keep keepPreviousData behavior for client-side filtering.
2. Detail loader: validate teacherId as a UUID (otherwise 404 immediately), call getTeacherById; 404 or unverified/hidden teacher -> 404.
3. List meta: title includes the category name when present and the page number when page > 1; canonical per canonicalFor (only ?page=N survives); noindex when search is present or the result list is empty. Add BreadcrumbList.
4. Detail meta: title '<name> - معلم خصوصی <main skill> آنلاین | هیربَد' (max ~60 chars, truncate the skill part first), description from the headline/bio truncated to ~155 chars, og:image only if the avatar is a public absolute URL. JSON-LD Person (name, description, image, knowsAbout from skills) and BreadcrumbList. Add aggregateRating ONLY if the page visibly shows a rating and count >= 1.
5. User-specific or time-dependent widgets (booking, availability, favorites, anything reading auth state or local time) render inside ClientOnly, with a fixed-height skeleton fallback so layout does not shift. A guest clicking Book must still be redirected to login as today.
6. headers(): publicCacheHeaders(60s). Error and 404 responses must not be cacheable.
7. ErrorBoundary on the detail route renders the existing NotFound/ErrorState components with the real status code.

CONSTRAINTS: keep UUID URLs (slugs are a later phase); do not change the booking flow.

TESTS: loader tests (valid, invalid uuid, 404, 503), meta/JSON-LD tests (no aggregateRating without rating), canonical tests, render tests with createRoutesStub for list and detail, a test that the raw loader data contains no keys outside the public serializer set.

ACCEPTANCE CRITERIA
- curl -sA Googlebot /teachers/<real-uuid> shows the teacher name in <h1>, unique title/description, canonical, valid JSON-LD; /teachers/<random-uuid> and /teachers/not-a-uuid return 404.
- /teachers?category=x&page=2 has canonical ...?page=2 (or the base URL for page 1); /teachers?search=abc is noindex.
- No hydration warnings with a logged-in and a logged-out browser.
VERIFICATION: curl commands; browser check with and without login; typecheck, lint, test, build.
```

### Prompt 9 — کلاس‌های گروهی

```
PROMPT 9 - Group sessions list and detail

CONTEXT
Routes /group-sessions and /group-sessions/:id. Session times are time-zone sensitive and the server runs in UTC, so formatting must be deterministic.

READ FIRST
Route modules, the group sessions feature folder (find it with rg 'group-sessions'), its hooks, API module, types, and any date/time formatting helpers.

TASK
1. Loaders for list and detail using server API functions from Prompt 5 (add them in the group sessions feature as *.server.ts if missing). Detail: 404 for unknown, cancelled or non-public sessions.
2. initialData wiring into the existing hooks exactly like Prompt 8.
3. Formatting: use one shared formatter with explicit timeZone (APP_TIMEZONE) and locale for both server and client. If any output still differs between server and client, wrap that part in ClientOnly.
4. Detail meta: title '<session title> - کلاس گروهی آنلاین | هیربَد', description from the session description, canonical, BreadcrumbList. JSON-LD Event: name, description, startDate and endDate as ISO 8601 with offset, eventAttendanceMode OnlineEventAttendanceMode, eventStatus, location as VirtualLocation (url = the page URL), organizer/performer (teacher as Person), offers (price, priceCurrency using the ISO code the backend uses, availability based on remaining seats, url). Only include fields that are visible on the page.
5. Ended sessions: 200 with noindex (they must not appear in the sitemap). List: noindex when empty; canonical rules as Prompt 8.
6. headers(): publicCacheHeaders(60s).

CONSTRAINTS: no booking-flow changes; no new currency conversion.

TESTS: loader tests, JSON-LD tests (online mode, offers, ISO dates with offset), formatter test under TZ=UTC and TZ=Asia/Tehran proving identical output, render tests.

ACCEPTANCE CRITERIA
- Raw HTML of a detail page shows the title, time, price and valid Event JSON-LD; unknown id returns 404; ended session is noindex.
- No hydration warnings under both TZ settings.
VERIFICATION: run the dev server with TZ=UTC and TZ=Asia/Tehran and compare the HTML; typecheck, lint, test, build.
```

### Prompt 10 — مقالات

```
PROMPT 10 - Articles list and detail

CONTEXT
Routes /articles (category, page) and /articles/:slug. Hooks: src/features/content/hooks/useArticles.ts (useArticles, useArticleBySlug). Article HTML must be identical on server and client and safe from XSS.

READ FIRST
Route modules, features/content/**, how the article body is rendered and sanitized (BASELINE section 5), article model/serializer fields (published_at, updated_at, cover_image, meta_description, author).

TASK
1. Loaders for list and detail; 404 for unknown or unpublished slugs. initialData wiring into the hooks like Prompt 8.
2. If body sanitization depends on window (for example DOMPurify), fix it so server and client output match: prefer sanitizing on the backend; otherwise use an isomorphic sanitizer already approved in the repo. Never render unsanitized HTML on the server.
3. Detail meta: title '<article title> | هیربَد', description = meta_description or the first ~155 chars of plain text, og:type article, og:image = cover image when it is a public absolute URL. JSON-LD Article (headline, datePublished, dateModified, image, author as Person or Organization) and BreadcrumbList.
4. List meta and canonical like Prompt 8 (noindex for empty lists).
5. Internal linking: related articles or a link back to /teachers must be real <a> links inside the HTML.
6. headers(): publicCacheHeaders(120s).

CONSTRAINTS: no editor or CMS changes; keep the current article layout.

TESTS: loader tests, sanitizer test proving a script tag and an inline event handler are stripped on the server path, JSON-LD tests, render tests.

ACCEPTANCE CRITERIA
- Raw HTML of an article contains the full body text, a valid Article JSON-LD and a unique title/description; unknown slug returns 404.
- A script payload in an article body never reaches the HTML.
VERIFICATION: curl a seeded article; typecheck, lint, test, build.
```

### Prompt 11 — صفحه‌های دسته‌بندی مسیرمحور (اختیاری، پرارزش)

```
PROMPT 11 - Category landing pages (optional but high SEO value)

CONTEXT
Categories are currently a ?category= filter on /teachers, which canonicalizes to /teachers. Category pages like /teachers/category/<slug> can rank for queries such as 'معلم خصوصی برنامه نویسی'. This prompt touches backend, database and frontend.

READ FIRST
Skill/category models and serializers, the landing category grid component, the teachers list route module, BASELINE sections 2 and 6.

TASK
1. Backend: ensure SkillCategory has a unique slug; add nullable seo_title, seo_description and intro_text (additive migration). Add a public endpoint for category detail by slug (404 when inactive) and include category slugs in the sitemap. Exact-key serializer test.
2. Frontend: route /teachers/category/:slug reusing the teachers list page and loader logic (with the category fixed by the path). Unique h1 and intro_text above the list, title from seo_title or '<name> - معلم خصوصی آنلاین | هیربَد', BreadcrumbList. noindex when the category has zero teachers.
3. Redirects: /teachers?category=<slug> -> 301 /teachers/category/<slug> (preserving page when > 1) when the category exists. The landing category grid links to the new paths.
4. Make sure the UUID validator on /teachers/:teacherId rejects the literal 'category'.

CONSTRAINTS: migrations must be backward-compatible; do not rename existing endpoints.

TESTS: backend tests for the endpoint and sitemap; loader tests for 200, 404 and 301; meta tests; a route-ranking test that /teachers/category/x does not hit the teacher detail route.

ACCEPTANCE CRITERIA
- curl -sI .../teachers?category=programming returns 301 to the path URL; the path URL shows unique title, h1, intro text and the teacher list in raw HTML.
- Sitemap includes active categories; empty categories are noindex.
VERIFICATION: backend tests + makemigrations --check; frontend typecheck, lint, test, build; curl checks.
```

### Prompt 12 — Infrastructure: Docker، nginx، compose

```
PROMPT 12 - Infrastructure: Docker images, nginx, compose

CONTEXT
Until now only code changed. Now production topology changes: nginx serves static files and proxies HTML to a Node SSR container. TLS is intentionally still not handled here (see the TLS note inside docker-compose.prod.yml). Do not change TLS.

READ FIRST
hirbad-frontend/Dockerfile, hirbad-frontend/nginx/nginx.conf.template (including the route whitelist and the /assets, /media, /api, robots.txt, sitemap.xml, ads.txt blocks), docker-compose.prod.yml, public/robots.txt, public/sitemap.xml, hirbad-frontend/.env.example.

TASK
1. Dockerfile stages: build (npm ci, react-router build), ssr (node:20-alpine, npm ci --omit=dev, copy build, non-root user, HEALTHCHECK on /healthz, CMD react-router-serve ./build/server/index.js, NODE_OPTIONS=--max-old-space-size=384), web (nginx:alpine with build/client at /usr/share/nginx/html and the nginx template). Keep the VITE_* build args that are still needed.
2. docker-compose.prod.yml: add service ssr (restart unless-stopped, healthcheck, mem_limit 512m as a starting value, env NODE_ENV=production, PORT=3000, INTERNAL_API_URL=http://backend:8000, INTERNAL_API_KEY, PUBLIC_SITE_URL; depends_on backend; no published ports). nginx service depends_on ssr. Add the new variables to .env.prod.example (backend) without real values.
3. nginx: remove the route whitelist and try_files /index.html fallbacks. Keep /assets (immutable), /media, /api, ads.txt. Proxy /sitemap.xml and /sitemap-*.xml to backend; keep robots.txt static (update its Sitemap: line; remove Disallow entries that would hide pages from crawling that we want noindexed via header, keep /api/). location / proxies to http://ssr:3000 with Host and X-Forwarded-* headers and proxy_set_header Cookie ''.
4. Cache: proxy_cache_path zone ssr_cache; key $scheme$host$request_uri; proxy_cache_valid 200 60s and 404 10s (never 5xx); proxy_cache_lock on; proxy_cache_background_update on; proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504; bypass and no_cache for private path prefixes and for requests that are not GET/HEAD; add X-Cache-Status header. Verify how nginx treats the Cache-Control that Node sends and make the config explicit instead of relying on defaults.
5. Delete public/sitemap.xml (Django serves it now).
6. Preserve existing security headers and the HSTS behavior.

CONSTRAINTS: no TLS work, no host-level nginx changes, no changes to application code.

TESTS / CHECKS: docker compose -f docker-compose.prod.yml config passes; nginx -t passes inside the web image; build both images; start the stack locally with the dev backend and run curl checks.

ACCEPTANCE CRITERIA
- docker compose build succeeds; all services become healthy.
- Second request to a public page shows X-Cache-Status: HIT; private paths never HIT; 5xx is never cached.
- With the ssr container stopped, a previously cached public page is still served and an uncached one returns 502/503.
- /sitemap.xml comes from Django; unknown paths return 404 from the app.
VERIFICATION: the curl and docker commands above, with output pasted in the report.
```

### Prompt 13 — تست E2E، Runbook و Cutover

```
PROMPT 13 - End-to-end checks, runbook and cutover plan

CONTEXT
Everything is implemented. We need automated proof of the Acceptance Criteria and operational docs.

READ FIRST
The design document section 9, existing test setup, docker-compose.prod.yml, nginx config, package.json scripts.

TASK
1. Add Playwright (devDependency) with e2e/ tests runnable via npm run e2e with BASE_URL. Tests: (a) for each key public URL (/, /teachers, one seeded teacher, /group-sessions, one session, /articles, one article, /about, /register) fetch raw HTML with a Googlebot user agent and assert: status 200, one h1, unique title and description, canonical equals the URL without extra params, valid JSON-LD where expected; (b) 404 for unknown teacher, article, session and path; (c) private routes have X-Robots-Tag noindex and the HTML contains no email, phone or token patterns; (d) open each key URL in a real browser and fail on any console error or hydration warning, once logged out and once logged in; (e) login, logout, theme toggle still work.
2. Add scripts/seo-smoke.sh that runs the curl-based checks (status, title, canonical, robots header, X-Cache-Status) against any BASE_URL, for use after each deploy.
3. Write docs/ssr-migration/RUNBOOK.md: build and deploy order (ssr and web together), required env variables, how to read the logs, how to flush the nginx cache, rollback to the previous SPA images and nginx config (and how to test that rollback on staging), and the post-launch checklist (Search Console sitemap submit, URL Inspection of 5 URLs, coverage checks at 1, 7 and 28 days).
4. Add a short CI job (or document the commands if CI config is not in the repo) running typecheck, lint, test, build and the smoke script against a started stack.

CONSTRAINTS: do not modify application code except to fix bugs the tests reveal, and report each such fix separately.

ACCEPTANCE CRITERIA
- npm run e2e passes on staging; scripts/seo-smoke.sh exits 0.
- RUNBOOK.md exists and its rollback section was executed once on staging (report the result).
VERIFICATION: paste the e2e and smoke outputs.
```

## 11. Verification / Testing

این بخش برای شماست، نه برای Agent. بعد از اجرای همه‌ی Promptها (و بعد از هر Prompt مربوطه) انجام دهید.

**خروجی خام (بدون جاوااسکریپت)**

```
curl -sA "Googlebot" https://STAGING/teachers/<UUID> | head -c 4000
curl -sI https://STAGING/teachers/00000000-0000-0000-0000-000000000000   # باید 404 باشد
curl -sI https://STAGING/dashboard                                        # باید X-Robots-Tag: noindex داشته باشد
curl -sI https://STAGING/teachers | rg -i 'x-cache-status|cache-control'
```

**ابزارهای بیرونی**

- Rich Results Test و Schema Markup Validator برای JSON-LD معلم، کلاس و مقاله.
- ابزار رسمی پیش‌نمایش تلگرام (`@WebpageBot`) و Facebook Sharing Debugger برای `og:*`.
- Search Console: ثبت `sitemap.xml`، سپس URL Inspection روی ۵ آدرس و مقایسه‌ی «HTML خام» با «HTML رندرشده».
- Lighthouse (موبایل) برای صفحه‌ی اصلی، لیست معلم‌ها، پروفایل معلم؛ هدف CLS < 0.1 و SEO ≥ 95.

**پایداری و بار**

- یک تست بار سبک (مثلاً `autocannon` روی ۳ صفحه‌ی عمومی) برای دیدن مصرف CPU و حافظه‌ی `ssr`.
- کانتینر `ssr` را متوقف کنید: صفحه‌های cacheشده باید سرو شوند و صفحه‌های بدون cache باید 502/503 بدهند.
- backend را متوقف کنید: صفحه‌های loaderدار باید 503 بدهند (نه 200) و در cache نمانند.

**پس از راه‌اندازی**

- روز ۱، ۷ و ۲۸: Coverage و Pages در Search Console، تعداد «Crawled - currently not indexed» و «Soft 404».
- لاگ ssr: خطاهای 5xx، زمان loader و نرخ HIT در cache.
- اگر مشکلی بود، طبق `RUNBOOK.md` به SPA قبلی برگردید، سپس علت را روی staging بررسی کنید.
