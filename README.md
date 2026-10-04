# همه‌جو

همه‌جو را ساختم چون آگهی مسکن، خودرو، کالا و خدمات روی چند سایت پراکنده است و می‌خواستم یک عبارت برای پیدا کردنشان کافی باشد. نتیجهٔ جستجو روی همه‌جو می‌ماند و خود آگهی روی سایت منبع می‌ماند. همه‌جو فروشگاه نیست و صفحهٔ جدا برای خود آگهی نساختم.

سایت: [hamejoo.ir](https://hamejoo.ir)

## چرا فقط فراداده را نگه می‌دارم

روی صفحهٔ «نحوهٔ کار جستجو» نوشتم که خزش، صفحهٔ عمومی را می‌خواند و عنوان، دسته‌بندی، قیمت، موقعیت و زمان انتشار را برمی‌دارد. تصویر و توضیح بلند را روی همه‌جو بارگذاری نمی‌کنم. برای همین در جدول `listings` مقدار `snippet` را حداکثر ۳۰۰ نویسه گذاشتم و `image_url` فقط نشانی است، نه فایل تصویر. عنوان تا ۵۱۲ نویسه است. قیمت، متراژ، شهر، محله و تعداد اتاق در همان ردیف ذخیره می‌شوند تا فیلتر جستجو به متن کامل آگهی وابسته نباشد.

کلیک، کاربر را به `source_url` می‌برد. در `apps/frontend/app/_lib/outbound.ts` روی لینک `noopener noreferrer nofollow` گذاشتم. ثبت کلیک را با `navigator.sendBeacon` می‌فرستم و اگر در دسترس نباشد با `fetch` و `keepalive`. `POST /listing-clicks` فقط ۲۰۴ برمی‌گرداند. هر ردیف `listing_clicks` شناسهٔ آگهی، نام منبع و زمان است. نشانی مقصد و صفحه را ذخیره نمی‌کنم؛ وگرنه همه‌جو به محل ثبت مسیر کاربر تبدیل می‌شد.

## سه برنامه، یک جدول

مخزن خصوصی سه برنامه دارد. وب `apps/frontend` است و نام بسته `hamejoo-frontend`. صفحهٔ اصلی `app/page.tsx` است و جستجو `app/search/page.tsx`. بقیهٔ صفحه‌هایی که در App Router گذاشتم: تنظیمات، پیشنهاد منبع، نحوهٔ کار جستجو، حریم خصوصی، شرایط، تماس.

API را در `apps/backend` با FastAPI نوشتم؛ فایلش `app/main.py` است و با Uvicorn بالا می‌آید. Postgres را در `app/db.py` با SQLAlchemy و `create_engine(..., pool_pre_ping=True)` باز می‌کنم تا اتصال قطع‌شده پیش از کوئری جایگزین شود. Redis را در `app/cache.py` با `Redis.from_url` باز می‌کنم.

خزنده `apps/crawler` است. همان جدول `listings` را می‌نویسد و پاسخ جستجوی عمومی را برنمی‌گرداند. نام سایت‌های منبع را اینجا تکرار نمی‌کنم.

## چرا مرورگر مبدأ API را نمی‌بیند

در `apps/frontend/app/_lib/api-base.ts` مسیر را `/api/hamejoo` گذاشتم. `next.config.mjs` پیشوند `/api/hamejoo/:path*` را روی سرور Next به API بازنویسی می‌کند. مرورگر `GET /api/hamejoo/search` می‌زند و API همان را به‌صورت `GET /search` می‌بیند. مبدأ API را داخل صفحه نگذاشتم تا با عوض شدنش کلاینت درگیر نشود.

## شکل جستجو

`GET /search` این پارامترها را می‌گیرد: `q` از ۱ تا ۱۰۰ نویسه، `min_price` و `max_price`، `min_area` و `max_area`، `city` حداکثر ۶۴ نویسه، `district` حداکثر ۱۲۸، `sort` یکی از `relevance` و `newest` و `price_asc` و `price_desc`، `limit` از ۱ تا ۵۰ با پیش‌فرض ۲۰، و `offset`. اگر `city` حذف شود پیش‌فرض API تهران است. صفحهٔ جستجو عمداً `city` را رشتهٔ خالی می‌فرستد تا این پیش‌فرض اعمال نشود؛ وگرنه یک جستجوی سراسری بی‌صدا به تهران محدود می‌شد. اگر کادر متن پر باشد `sort` را `relevance` می‌گذارم وگرنه `newest`.

قبل از SQL کلید کش را از JSON پارامترها با SHA-256 می‌سازم (`make_cache_key` در `cache.py`). اگر کلید در کش باشد، `SearchResponse` از Redis برمی‌گردد. اگر نباشد، کوئری با SQLAlchemy اجرا می‌شود. `q` غیرخالی با `to_tsvector('simple', title || snippet || city || district)` در برابر `plainto_tsquery('simple', q)` سنجیده می‌شود، یا `title ILIKE`، یا `district ILIKE`. پیکربندی `simple` را گذاشتم چون متن آگهی فارسی است و واژه‌نامهٔ انگلیسی Postgres به ریشهٔ کلمه کمکی نمی‌کند. فیلتر قیمت و متراژ و شهر و محله فقط وقتی اضافه می‌شود که پارامتر آمده باشد.

ردیف باید در نتیجه دیده شود: `availability = active`، یا `expired` که `expired_at` هنوز داخل مهلت است. `DEFAULT_EXPIRED_GRACE_DAYS` در `listing_constants.py` برابر ۷ است و تنظیم `listing_expired_grace_days` همان مقدار را دارد. صفحهٔ عمومی نحوهٔ کار هم همین هفت روز را می‌گوید: بعد از حذف در مبدأ، آگهی منقضی می‌ماند و بعد ردیف به‌صورت فیزیکی پاک می‌شود.

ترتیب را این‌طور گذاشتم. اول ردیف‌های آلور و آزادچی را جلوتر از بقیه می‌آورم، چون آن دو بازار خودم‌اند و در نتیجهٔ یک جستجوی مشترک نباید پشت منابع دیگر گم شوند. بعد، اگر `sort` برابر `relevance` باشد و `q` پر باشد، اول `ts_rank` همان بردار می‌آید و سپس `created_at` نزولی. `price_asc` و `price_desc` قیمت تهی را آخر می‌گذارند. هر حالت دیگر، از جمله `newest`، با `created_at` نزولی است. تعداد کل از تابع پنجره‌ای `count` روی همان فیلتر می‌آید و اگر صفحه خالی باشد یک `count` جدا زده می‌شود. بدنه `SearchResponse` است: `total`، `limit`، `offset`، `items` از نوع `ListingOut`. این JSON را ۶۰ ثانیه در Redis نگه می‌دارم. جستجوی تکراری در این فاصله نباید دوباره همان SQL را بزند.

اگر `q` باشد و `offset` صفر و `total` بیشتر از صفر، `_log_search_query` روی `search_queries` upsert می‌کند: `q` کلید اصلی است، `count` یکی زیاد می‌شود و `last_seen` زمان همین لحظه است. عبارت کوتاه‌تر از ۲ نویسه یا بلندتر از ۱۲۰ را ثبت نمی‌کنم تا پیشنهادها با نویسهٔ تصادفی پر نشود.

`search_listings` در `crud.py` مسیر SQL دوم است، با همان شرط دیده شدن و همان تطبیق متن. مسیری که بازنویسی Next صدا می‌زند هندلر `GET /search` در `main.py` است.

روی مسیرهای خواندنی، از جمله `/search` و `/suggest` و `/latest` و `/listing/{listing_id}` و `/stats/catalog-listed`، محدودیت نرخ را با slowapi و `get_remote_address` گذاشتم.

## پیشنهاد، تازه‌ها، و پیشنهاد منبع

`GET /suggest` پارامتر `q` را حداکثر ۸۰ نویسه و `limit` را از ۱ تا ۱۵ (پیش‌فرض ۸) می‌گیرد و ۳۰ ثانیه کش می‌شود. آیتم‌ها را به ترتیب پر می‌کنم و بعد تکراری‌ها را حذف می‌کنم: پیشوند و شامل‌بودن روی `search_queries` با ترتیب پیشوند، سپس `count`، سپس `last_seen` (نوع `history`)؛ بعد عنوان آگهی‌های تازهٔ قابل‌دیدن (نوع `listing`)؛ اگر هیچ‌کدام نبود یک فهرست ثابت (نوع `fallback`). روی `SuggestionItem` نوع‌های `history` و `listing` و `trending` و `fallback` هست.

`GET /latest` آگهی‌های قابل‌دیدن را با `created_at` نزولی برمی‌گرداند، ۶۰ ثانیه کش می‌شود و همان شکل `SearchResponse` را دارد. `GET /listing/{listing_id}` یک `ListingOut` برمی‌گرداند، به شرطی که ردیف هنوز قابل‌دیدن باشد.

`POST /source-requests` نام سایت و نشانی را، و اگر آمده باشد دسته و توضیح و راه تماس را، در `source_requests` با وضعیت پیش‌فرض `pending` ذخیره می‌کند. فیلد `website` را honeypot گذاشتم تا ارسال ماشینی وارد جدول نشود.

`GET /health` وضعیت کوتاه است. `GET /stats/catalog-listed` تعداد فعال، منقضی داخل مهلت، و حذف‌شدهٔ تجمعی را می‌دهد. کش آمار کاتالوگ را ۵۰ ثانیه گذاشتم (`catalog_stats_cache_ttl_seconds`). `GET /listing-lifecycle-stats` وضعیت لحظه‌ای پاک‌سازی است: حذف فیزیکی تجمعی، زمان آخرین اجرا، شمارش آخرین اجرا، فعال‌های جاری، منقضی‌های داخل مهلت، و طول مهلت. همان مسیر را روی صفحهٔ عمومی نحوهٔ کار نام بردم تا مهلت هفت‌روزه فقط ادعا نباشد.

## مسیر نوشتن خزنده

`CrawledListing` در `apps/crawler/app/types.py` عنوان، اسنیپت، قیمت، متراژ، شهر، محله، اتاق، منبع، نشانی منبع، نشانی تصویر و هش را دارد. `save_listings` ردیف بی‌هش را دور می‌ریزد، تکراری‌های همان هش را یکی می‌کند، `availability` را `active` و `last_verified_at` را زمان UTC الان می‌گذارد و در `listings` درج می‌کند. اگر `hash` از قبل باشد، عنوان و اسنیپت و قیمت و متراژ و شهر و محله و اتاق و نشانی‌ها و وضعیت و `last_verified_at` را تازه می‌کند و `expired_at` را خالی می‌کند؛ آگهی‌ای که دوباره دیده شد نباید منقضی بماند. اگر درج دسته‌ای شکست بخورد، ردیف‌به‌ردیف دوباره می‌زنم تا یک ردیف خراب کل دور را نیندازد.

`run_once` اول طرح پایگاه را برقرار می‌کند، بعد برای همان دور یک مرورگر Playwright را از `launch_shared_browser_context` باز می‌کند. تعداد کار هر دور را با `max_jobs_per_run` محدود کردم تا یک اجرا جدول را قفل نکند. آگهی‌های جمع‌شده از `save_listings` رد می‌شوند.

## کتابخانه‌ها، همان‌طور که در فایل‌ها آمده

در `apps/frontend/package.json`: next ^14.2.35، react 18.3.1، react-dom 18.3.1، typescript 5.7.3، tailwindcss 3.4.17، postcss 8.4.49، autoprefixer 10.4.20، و نوع‌های node و react و react-dom.

در `apps/backend/requirements.txt` نسخه را پین نکرده‌ام: fastapi، uvicorn[standard]، sqlalchemy، psycopg[binary]، redis، pydantic-settings، slowapi.

در `apps/crawler/requirements.txt` نسخه را پین نکرده‌ام: playwright، sqlalchemy، psycopg[binary]، pydantic-settings، httpx، pytest.

میزبانی و رمزها را اینجا نیاوردم.

## English

I built Hamejoo because housing, car, goods, and service listings sit on separate sites, and I wanted one phrase to be enough. The result stays on Hamejoo. The listing stays on the source site. Hamejoo is not a shop; I did not build a destination page.

Site: [hamejoo.ir](https://hamejoo.ir)

### Why I keep only metadata

On the public how-search page I wrote that a crawl reads a public page and keeps title, category, price, location, and publish time. I do not upload the image or the long description onto Hamejoo. That is why `snippet` on `listings` is at most 300 characters and `image_url` is an address, not an image file. Title is at most 512 characters. Price, area, city, district, and room count sit on the same row so search filters do not depend on the full listing text.

A click goes to `source_url`. In `apps/frontend/app/_lib/outbound.ts` I set `rel` to `noopener noreferrer nofollow` and record the click with `navigator.sendBeacon`, or `fetch` with `keepalive` when beacon is missing. `POST /listing-clicks` returns 204. A `listing_clicks` row is the listing id, the source name, and the time. I do not store the outbound URL or the page. Otherwise Hamejoo would become a log of where the user went.

### Three programs, one table

The private tree has three apps. The web app is `apps/frontend`, package name `hamejoo-frontend`. Home is `app/page.tsx`. Search is `app/search/page.tsx`. The other App Router pages I shipped are settings, submit-source, how-search-works, privacy, terms, and contact.

The API is `apps/backend`, FastAPI in `app/main.py`, served by Uvicorn. I open Postgres in `app/db.py` with SQLAlchemy `create_engine(..., pool_pre_ping=True)` so a dead connection is replaced before the query. I open Redis in `app/cache.py` with `Redis.from_url`.

The crawler is `apps/crawler`. It writes the same `listings` table and does not serve the public search response. I am not repeating source site names here.

### Why the browser never sees the API origin

In `apps/frontend/app/_lib/api-base.ts` I set the path to `/api/hamejoo`. `next.config.mjs` rewrites `/api/hamejoo/:path*` on the Next server onto the API. The browser calls `GET /api/hamejoo/search`. The API sees `GET /search`. I did not put the API origin in the page, so changing it does not reach the client.

### The search shape

`GET /search` takes `q` (1–100 characters), `min_price`, `max_price`, `min_area`, `max_area`, `city` (max 64), `district` (max 128), `sort` (`relevance`, `newest`, `price_asc`, `price_desc`), `limit` (1–50, default 20), and `offset`. If `city` is omitted, the API default is Tehran. The search page deliberately sends `city` as an empty string so that default is not applied. Otherwise a nationwide query would silently collapse to Tehran. When the text box is non-empty I set `sort` to `relevance`; otherwise `newest`.

Before SQL I build a cache key by SHA-256 of the JSON parameters (`make_cache_key` in `cache.py`). A hit returns `SearchResponse` from Redis. A miss runs SQLAlchemy. A non-empty `q` matches `to_tsvector('simple', title || snippet || city || district)` against `plainto_tsquery('simple', q)`, or `title ILIKE`, or `district ILIKE`. I used the `simple` configuration because listing text is Persian and Postgres's English dictionary does not help with stemming. Price, area, city, and district filters are added only when those parameters are present.

A row must be visible: `availability = active`, or `expired` with `expired_at` still inside the grace window. `DEFAULT_EXPIRED_GRACE_DAYS` in `listing_constants.py` is 7, and `listing_expired_grace_days` is the same. The public how-search page says the same seven days: after the source deletes the listing, Hamejoo marks it expired, then physically deletes the row.

I closed the sort like this. First I put Alwer and Azadchi rows ahead of the rest, because those two markets are mine and they should not disappear behind other sources in a shared result. Then, when `sort` is `relevance` and `q` is set, I order by `ts_rank` of that vector and `created_at` descending. `price_asc` and `price_desc` put null prices last. Any other sort, including `newest`, orders by `created_at` descending. The total is a window count on the filtered statement, or a separate count when the page is empty. The body is `SearchResponse`: `total`, `limit`, `offset`, and `items` of `ListingOut`. I keep that JSON in Redis for 60 seconds. A repeated search in that window should not run the same SQL again.

When `q` is present, `offset` is 0, and `total` is greater than 0, `_log_search_query` upserts `search_queries`: `q` is the primary key, `count` increments, `last_seen` is now. I do not log queries shorter than 2 characters or longer than 120, so suggestions do not fill with random characters.

`search_listings` in `crud.py` is a second SQL path with the same visibility rule and the same text match. The path the Next rewrite calls is the `GET /search` handler in `main.py`.

On the read routes, including `/search`, `/suggest`, `/latest`, `/listing/{listing_id}`, and `/stats/catalog-listed`, I rate-limit with slowapi and `get_remote_address`.

### Suggest, latest, and a source suggestion

`GET /suggest` takes `q` (max 80) and `limit` (1–15, default 8) and is cached for 30 seconds. I fill items in order, then drop duplicates: prefix and contains matches on `search_queries`, ordered by prefix match, then `count`, then `last_seen` (`kind` `history`); then recent visible `listings.title` values (`kind` `listing`); then a fixed list when nothing else matches (`kind` `fallback`). `SuggestionItem` kinds are `history`, `listing`, `trending`, and `fallback`.

`GET /latest` returns visible listings by `created_at` descending, cached 60 seconds, same `SearchResponse` shape. `GET /listing/{listing_id}` returns one `ListingOut` when the row is still visible.

`POST /source-requests` stores site name, URL, and optional category, description, and contact on `source_requests`, status default `pending`. I added a honeypot field `website` so an automated post does not land in that table.

`GET /health` is a short status. `GET /stats/catalog-listed` returns active, expired-in-grace, and lifetime-removed counts. I cache catalog stats for 50 seconds (`catalog_stats_cache_ttl_seconds`). `GET /listing-lifecycle-stats` is the purge snapshot: cumulative physical removals, last purge time, last-run counts, current active and expired-in-grace counts, and the grace length. I named that route on the public how-search page so the seven-day window is not only a claim.

### Crawler write path

`CrawledListing` in `apps/crawler/app/types.py` carries title, snippet, price, area, city, district, rooms, source, source URL, image URL, and hash. `save_listings` drops rows with an empty hash, collapses duplicates on hash, sets `availability` to `active` and `last_verified_at` to the current UTC time, and inserts into `listings`. On conflict of `hash` it updates title, snippet, price, area, city, district, rooms, URLs, availability, and `last_verified_at`, and clears `expired_at`. A listing seen again must not stay expired. If the batch insert fails, I retry row by row so one bad row does not drop the round.

`run_once` ensures the schema, then opens one Playwright browser through `launch_shared_browser_context` for that round. I cap the round with `max_jobs_per_run` so one run does not lock the table. Collected items go through `save_listings`.

### Libraries, as the files declare them

In `apps/frontend/package.json`: next ^14.2.35, react 18.3.1, react-dom 18.3.1, typescript 5.7.3, tailwindcss 3.4.17, postcss 8.4.49, autoprefixer 10.4.20, and the node, react, and react-dom type packages.

In `apps/backend/requirements.txt`, versions not pinned: fastapi, uvicorn[standard], sqlalchemy, psycopg[binary], redis, pydantic-settings, slowapi.

In `apps/crawler/requirements.txt`, versions not pinned: playwright, sqlalchemy, psycopg[binary], pydantic-settings, httpx, pytest.

I am not putting hosting or credentials here.

## پروژه‌های مرتبط

- [alwer-intro](https://github.com/mnhashemabadi/alwer-intro): آلور را برای خریداری ساختم که درخواست بنویسد و پیشنهاد قیمت‌ها را کنار هم ببیند؛ آگهی فروش هم روی همان بازار است.
- [kasbafzar-intro](https://github.com/mnhashemabadi/kasbafzar-intro): کسب‌افزار را برای کسی ساختم که فروش و مشتری و هزینه را ثبت کند و فاکتور را با لینک پرداخت برای مشتری بفرستد.
- [azadchi-intro](https://github.com/mnhashemabadi/azadchi-intro): آزادچی را برای آگهی و جستجو در مناطق آزاد ساختم؛ گفتگو با طرف معامله داخل خود آزادچی می‌ماند.
- [afzi-intro](https://github.com/mnhashemabadi/afzi-intro): افزی را برای کوتاه کردن یک نشانی http یا https ساختم؛ باز کردن لینک کوتاه همان صفحه را باز می‌کند.
- [alweryar-intro](https://github.com/mnhashemabadi/alweryar-intro): آلوریار را برای همکاری در بررسی آگهی و همکاری در فروش آلور ساختم.
- [alwerchi-intro](https://github.com/mnhashemabadi/alwerchi-intro): آلورچی را برای فروشگاهی ساختم که کالا و موجودی‌اش در بازار آلور دیده شود و خریدار در آلور بماند.

## Related

- [alwer-intro](https://github.com/mnhashemabadi/alwer-intro): I built Alwer for a buyer who posts a request and compares price offers side by side; a sale listing sits on the same market.
- [kasbafzar-intro](https://github.com/mnhashemabadi/kasbafzar-intro): I built Kasbafzar for someone who records sales, customers, and expenses, and sends the customer an invoice with a payment link.
- [azadchi-intro](https://github.com/mnhashemabadi/azadchi-intro): I built Azadchi for listings and search inside free zones; the conversation with the other party stays in Azadchi.
- [afzi-intro](https://github.com/mnhashemabadi/afzi-intro): I built Afzi to shorten an http or https address; opening the short link opens that same page.
- [alweryar-intro](https://github.com/mnhashemabadi/alweryar-intro): I built Alweryar for collaboration on listing review and on sales for Alwer.
- [alwerchi-intro](https://github.com/mnhashemabadi/alwerchi-intro): I built Alwerchi for a shop whose goods and stock show on the Alwer market while the buyer stays on Alwer.

یادداشت مهندسی کوتاه‌تر: [docs/engineering.md](docs/engineering.md)

Shorter engineering note: [docs/engineering.md](docs/engineering.md)
