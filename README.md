# همه‌جو

همه‌جو جای جستجوی آگهی است. یک عبارت نوشته می‌شود و آگهی‌های مرتبط یک‌جا دیده می‌شوند؛ از مسکن و خودرو تا کالا و خدمات. نتیجه روی همه‌جو است و جزئیات آگهی روی منبع اصلی همان آگهی می‌ماند.

سایت: [hamejoo.ir](https://hamejoo.ir)

## رفتار عمومی

صفحهٔ اصلی سایت به جستجو، تنظیمات، حریم خصوصی، شرایط استفاده، تماس، نحوهٔ کار جستجو، و پیشنهاد منبع پیوند می‌دهد. صفحهٔ «نحوهٔ کار جستجو» فرآیند را این‌طور شرح می‌دهد: خزش صفحهٔ عمومی و استخراج فراداده (عنوان، دسته‌بندی، قیمت، موقعیت، زمان انتشار)، نمایه‌سازی بعد از نرمال‌سازی، علامت «منقضی» وقتی آگهی در مبدأ حذف شده باشد و ماندن آن حدود هفت روز در نتایج، سپس حذف فیزیکی ردیف، و هدایت کلیک به صفحهٔ منبع. همان صفحهٔ عمومی می‌گوید تصاویر و توضیح بلند آگهی روی همه‌جو بارگذاری نمی‌شوند.

کد با این شرح هم‌خوان است، بدون آنکه نام منابع خزش در این معرفی تکرار شود.

## English

Hamejoo is a place to search listings across housing, cars, goods, and services. The result is on Hamejoo. The listing detail stays on the original source.

Site: [hamejoo.ir](https://hamejoo.ir)

The public site links search, settings, privacy, terms, contact, how search works, and a source suggestion form. The public how-search page describes crawl of public pages, metadata extraction, indexing after normalization, an expired mark for about seven days after the source listing is gone, then physical deletion, and a click through to the source page. It also says listing images and long descriptions are not uploaded onto Hamejoo.

## Parts

Three applications sit in the private product tree. Only what those files declare is named here.

The web app is `apps/frontend` (package name `hamejoo-frontend`). `apps/frontend/app/page.tsx` is the home page. Search UI is `apps/frontend/app/search/page.tsx`. Other App Router pages in that tree include settings, submit-source, how-search-works, privacy, terms, and contact.

The browser does not call the API host directly. `apps/frontend/app/_lib/api-base.ts` sets `API_BASE_PATH` to `/api/hamejoo`. `apps/frontend/next.config.mjs` rewrites `/api/hamejoo/:path*` onto the backend. A search request from the page is therefore `GET /api/hamejoo/search` in the browser and `GET /search` on the API.

The API is `apps/backend`, a FastAPI application in `apps/backend/app/main.py`, served with Uvicorn. PostgreSQL is opened in `apps/backend/app/db.py` with SQLAlchemy `create_engine` and `pool_pre_ping=True`. Redis is opened in `apps/backend/app/cache.py` with `Redis.from_url`.

The crawler is `apps/crawler`. It writes the same `listings` table. It does not serve the public search response.

## Search request

`GET /search` in `apps/backend/app/main.py` accepts `q` (1–100 characters), `min_price`, `max_price`, `min_area`, `max_area`, `city` (default `تهران` when the parameter is omitted, maximum 64 characters), `district`, `sort` (`relevance`, `newest`, `price_asc`, `price_desc`), `limit` (1–50, default 20), and `offset`.

The search page builds the query itself. When the text box is non-empty, `sort` is `relevance`; otherwise it is `newest`. The page sets `city` to an empty string so the API default of Tehran is not applied. Results are fetched from `/api/hamejoo/search`.

Before SQL, the handler builds a cache key. `make_cache_key` in `cache.py` hashes the JSON payload with SHA-256 and prefixes it. A hit returns `SearchResponse` from Redis. A miss runs SQLAlchemy filters in `_apply_filters`. A non-empty `q` matches `to_tsvector('simple', title || snippet || city || district)` against `plainto_tsquery('simple', q)`, or `title ILIKE`, or `district ILIKE`. Price, area, city, and district filters are added only when those parameters are present. Rows must be publicly visible: `availability = active`, or `availability = expired` with `expired_at` still inside the grace window (`listing_constants.py` sets `DEFAULT_EXPIRED_GRACE_DAYS` to 7).

Sort `relevance` with a query orders by `ts_rank` of that same `to_tsvector` expression, then `created_at` descending. `price_asc` and `price_desc` put null prices last. Any other sort, including `newest`, orders by `created_at` descending. The page size is `limit` and `offset`. The total uses a window count on the filtered statement, or a separate count when the page is empty. The JSON body is `SearchResponse`: `total`, `limit`, `offset`, and `items` of `ListingOut`. The handler stores that JSON in Redis for 60 seconds. When `q` is present, `offset` is 0, and `total` is greater than 0, `_log_search_query` upserts `search_queries` (`q` primary key, `count` incremented, `last_seen` set to now). Queries shorter than 2 characters or longer than 120 characters are not logged.

`apps/backend/app/crud.py` has a second SQL path, `search_listings`, with the same visibility rule and the same full-text plus `ILIKE` match. The HTTP handler above is the path the Next.js rewrite calls.

Rate limits use `slowapi` (`Limiter` with `get_remote_address`) on the read routes, including `/search`, `/suggest`, `/latest`, `/listing/{listing_id}`, and `/stats/catalog-listed`.

## Suggest, latest, and click

`GET /suggest` takes `q` (max 80) and `limit` (1–15, default 8). The response is cached for 30 seconds. Items are filled in order, then de-duplicated: prefix and contains matches on `search_queries` ordered by prefix match, then `count`, then `last_seen` (`kind` `history`); then recent visible `listings.title` values (`kind` `listing`); then a fixed fallback list when nothing else matches (`kind` `fallback`). Kinds on `SuggestionItem` are `history`, `listing`, `trending`, and `fallback`.

`GET /latest` returns visible listings ordered by `created_at` descending, cached for 60 seconds, with the same `SearchResponse` shape.

`GET /listing/{listing_id}` returns one `ListingOut` when the row is publicly visible.

`POST /listing-clicks` with body `{"listing_id": <int>}` returns 204. The row written to `listing_clicks` stores `listing_id`, `source`, and `clicked_at`. The model comment states that the outbound URL and the page are not stored. The web client in `apps/frontend/app/_lib/outbound.ts` sends that POST with `navigator.sendBeacon` when available, otherwise `fetch` with `keepalive`. The anchor `href` stays `source_url`. `rel` is `noopener noreferrer nofollow`.

`POST /source-requests` accepts `site_name`, `site_url`, optional `category`, `description`, and `contact`, and stores a `source_requests` row. The schema includes a honeypot field `website`.

`GET /health` returns a short status object. `GET /stats/catalog-listed` returns active, expired, and lifetime-removed counts. `GET /listing-lifecycle-stats` returns the purge snapshot: cumulative physically removed rows, last purge time, last-run counts, current active and expired-in-grace counts, and the grace length. The public how-search page names this stats route.

## Data

SQLAlchemy models in `apps/backend/app/models.py`:

- `listings`: `id`, `title` (max 512), `snippet` (max 300), `price`, `area`, `city`, `district`, `rooms`, `source`, `source_url`, `image_url`, unique `hash`, `availability` (`active` or `expired`), `last_verified_at`, `expired_at`, `created_at`, `updated_at`. Indexes include `city`, `district`, `source`, `hash`, `availability`, `created_at`, `price`, and `area`.
- `search_queries`: one row per query string, with `count` and `last_seen`.
- `listing_clicks`: click id, listing id, source, time.
- `listing_lifecycle_stats`: a single aggregate row (`id` 1) for physical removals and the last purge snapshot.
- `listing_purge_run_log`: one row per purge run (`active_count`, `expired_in_grace_count`, `purged_count`).
- `source_requests`: suggested site name and URL, optional category, description, contact, status default `pending`.

## Crawler write path

`apps/crawler/app/types.py` defines `CrawledListing` with title, snippet, price, area, city, district, rooms, source, source URL, image URL, and hash. `save_listings` in `apps/crawler/app/main.py` drops rows with an empty hash, collapses duplicates on hash, sets `availability` to `active` and `last_verified_at` to the current UTC time, and inserts into `listings`. On conflict of `hash` it updates title, snippet, price, area, city, district, rooms, source URL, image URL, availability, and `last_verified_at`, and clears `expired_at`. If the batch insert fails, it retries row by row.

`run_once` ensures the schema, then opens one Playwright browser through `launch_shared_browser_context` in `apps/crawler/app/playwright_shared.py` for that round. Each job is bounded by `max_jobs_per_run`. Collected items are saved through `save_listings`.

## Libraries

Declared in `apps/frontend/package.json`:

- next ^14.2.35
- react 18.3.1
- react-dom 18.3.1
- typescript 5.7.3
- tailwindcss 3.4.17
- postcss 8.4.49
- autoprefixer 10.4.20
- @types/node 22.10.7
- @types/react 18.3.18
- @types/react-dom 18.3.5

Declared in `apps/backend/requirements.txt` with no pinned versions: fastapi, uvicorn[standard], sqlalchemy, psycopg[binary], redis, pydantic-settings, slowapi.

Declared in `apps/crawler/requirements.txt` with no pinned versions: playwright, sqlalchemy, psycopg[binary], pydantic-settings, httpx, pytest.

## Boundaries

Hosting and credentials are omitted. Crawled site names are omitted.

## پروژه‌های مرتبط

- [alwer-intro](https://github.com/mnhashemabadi/alwer-intro): آلور بازار نیازمندی است؛ درخواست خرید ثبت می‌شود، فروشنده پیشنهاد قیمت می‌فرستد، و آگهی فروش هم در همان بازار است.
- [kasbafzar-intro](https://github.com/mnhashemabadi/kasbafzar-intro): کسب‌افزار پیشخوان فروش و گزارش مالی است؛ فروش، مشتری و هزینه ثبت می‌شود و فاکتور می‌تواند لینک پرداخت داشته باشد.
- [azadchi-intro](https://github.com/mnhashemabadi/azadchi-intro): آزادچی بازار نیازمندی مناطق آزاد و ویژهٔ اقتصادی است؛ آگهی در همان محدوده جستجو می‌شود و گفتگو داخل همان محصول است.
- [afzi-intro](https://github.com/mnhashemabadi/afzi-intro): افزی یک نشانی http یا https را به لینک کوتاه تبدیل می‌کند و باز کردن آن لینک به همان صفحه می‌رود.
- [alweryar-intro](https://github.com/mnhashemabadi/alweryar-intro): آلوریار شبکهٔ همکاران آلور برای بررسی آگهی و همکاری در فروش است.
- [alwerchi-intro](https://github.com/mnhashemabadi/alwerchi-intro): آلورچی خانهٔ فروشگاه‌هایی است که کالا و موجودی‌شان در بازار آلور دیده می‌شود و خریدار در آلور می‌ماند.

## Related

- [alwer-intro](https://github.com/mnhashemabadi/alwer-intro): Alwer is a classifieds marketplace: a buyer posts a request, sellers send price offers, and a sale listing can be posted on the same market.
- [kasbafzar-intro](https://github.com/mnhashemabadi/kasbafzar-intro): Kasbafzar is a sales desk and a financial report: sales, customers, and expenses are recorded, and an invoice can carry a payment link.
- [azadchi-intro](https://github.com/mnhashemabadi/azadchi-intro): Azadchi is a classifieds market for free zones and special economic zones: listings are searched in that area, and the conversation stays in the product.
- [afzi-intro](https://github.com/mnhashemabadi/afzi-intro): Afzi turns an http or https address into a short link, and opening that link goes to the same page.
- [alweryar-intro](https://github.com/mnhashemabadi/alweryar-intro): Alweryar is Alwer's collaborator network for listing review and for sales collaboration.
- [alwerchi-intro](https://github.com/mnhashemabadi/alwerchi-intro): Alwerchi is the home of shops whose goods and stock appear on the Alwer marketplace while the buyer stays on Alwer.

یادداشت مهندسی کوتاه‌تر: [docs/engineering.md](docs/engineering.md)

Shorter engineering note: [docs/engineering.md](docs/engineering.md)

