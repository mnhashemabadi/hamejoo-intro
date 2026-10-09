# همه‌جو

همه‌جو را ساختم چون آگهی مسکن، خودرو، کالا و خدمات روی چند سایت پراکنده است و می‌خواستم یک عبارت برای پیدا کردنشان کافی باشد. نتیجهٔ جستجو روی همه‌جو می‌ماند و خود آگهی روی سایت منبع می‌ماند. همه‌جو فروشگاه نیست و صفحهٔ جدا برای خود آگهی نساختم.

سایت: [hamejoo.ir](https://hamejoo.ir)

## چرا فقط فراداده را نگه می‌دارم

روی صفحهٔ «نحوهٔ کار جستجو» نوشتم که خزش، صفحهٔ عمومی را می‌خواند و عنوان، دسته‌بندی، قیمت، موقعیت و زمان انتشار را برمی‌دارد. تصویر و توضیح بلند را روی همه‌جو بارگذاری نمی‌کنم. از توضیح فقط خلاصه‌ای حداکثر ۳۰۰ نویسه‌ای می‌ماند و از تصویر فقط نشانی‌اش، نه خود فایل. قیمت، متراژ، شهر، محله و تعداد اتاق کنار هم ذخیره می‌شوند تا فیلتر جستجو به متن کامل آگهی وابسته نباشد.

کلیک، کاربر را مستقیم به آگهی روی سایت منبع می‌برد و لینک `noopener noreferrer nofollow` است. ثبت کلیک را با `navigator.sendBeacon` می‌فرستم و اگر در دسترس نباشد با `fetch` و `keepalive`، تا رفتن کاربر معطل نشود. برای هر کلیک فقط آگهی، منبع و زمان را نگه می‌دارم. نشانی مقصد و صفحه را ذخیره نمی‌کنم؛ وگرنه همه‌جو به محل ثبت مسیر کاربر تبدیل می‌شد.

## سه برنامه، یک پایگاه

همه‌جو سه برنامه است: وب با Next.js و App Router، API با FastAPI روی Uvicorn، و خزنده با Playwright. API با SQLAlchemy به Postgres وصل می‌شود و اتصال قطع‌شده را پیش از اجرای کوئری جایگزین می‌کند. Redis لایهٔ کش پاسخ است. خزنده آگهی‌ها را در همان پایگاه می‌نویسد و پاسخ جستجوی عمومی را برنمی‌گرداند.

مرورگر مستقیم با API حرف نمی‌زند. درخواست به مسیری روی خود وب می‌رود و سرور Next آن را به API بازنویسی می‌کند. مبدأ API را داخل صفحه نگذاشتم تا با عوض شدنش کلاینت درگیر نشود.

## جستجو

جستجو عبارت متنی، بازهٔ قیمت و متراژ، شهر، محله و ترتیب نتیجه را می‌گیرد: مرتبط‌ترین، تازه‌ترین، ارزان‌ترین یا گران‌ترین. نتیجه صفحه‌به‌صفحه برمی‌گردد. صفحهٔ جستجو عمداً جستجوی بی‌شهر را سراسری نگه می‌دارد؛ وگرنه یک جستجوی سراسری بی‌صدا به یک شهر محدود می‌شد. اگر کادر متن پر باشد ترتیب بر اساس ارتباط است و اگر خالی باشد بر اساس تازگی.

متن با جستجوی تمام‌متن Postgres سنجیده می‌شود: `to_tsvector` روی عنوان، خلاصه، شهر و محله در برابر `plainto_tsquery`، و کنارش تطبیق جزئی عنوان و محله با `ILIKE`. پیکربندی `simple` را گذاشتم چون متن آگهی فارسی است و واژه‌نامهٔ انگلیسی Postgres به ریشهٔ کلمه کمکی نمی‌کند. در ترتیب مرتبط‌ترین، `ts_rank` همان بردار به کار می‌رود و بعد تازگی. در ترتیب قیمت، آگهی بی‌قیمت ته فهرست می‌رود. فیلتر قیمت و متراژ و شهر و محله فقط وقتی اضافه می‌شود که پارامترش آمده باشد. تعداد کل نتیجه با تابع پنجره‌ای `count` در همان کوئری حساب می‌شود.

پیش از SQL، کلید کش را با SHA-256 از پارامترها می‌سازم. همان جستجو تا ۶۰ ثانیه از Redis جواب می‌گیرد و دوباره به پایگاه نمی‌رود.

آگهی‌ای که در مبدأ حذف شود هفت روز منقضی می‌ماند و بعد به‌صورت فیزیکی پاک می‌شود. صفحهٔ عمومی نحوهٔ کار همین هفت روز را می‌گوید و آمار پاک‌سازی را هم نشان می‌دهد تا این مهلت فقط ادعا نباشد.

## پیشنهاد خودکار

عبارت‌هایی که نتیجه داشته‌اند برای پیشنهاد خودکار شمرده می‌شوند. عبارت کوتاه‌تر از ۲ نویسه یا بلندتر از ۱۲۰ را ثبت نمی‌کنم تا پیشنهادها با نویسهٔ تصادفی پر نشود. پیشنهاد از همین تاریخچه پر می‌شود، بعد از عنوان آگهی‌های تازه، و اگر هیچ‌کدام نبود از یک فهرست ثابت. تکراری‌ها حذف می‌شوند و پاسخ ۳۰ ثانیه کش می‌شود.

مسیرهای خواندنی با slowapi محدودیت نرخ دارند. فرم پیشنهاد منبع جدید یک فیلد honeypot دارد تا ارسال ماشینی ثبت نشود.

## خزنده

هر دور خزش یک مرورگر Playwright مشترک باز می‌کند و تعداد کارش سقف دارد تا یک اجرا پایگاه را قفل نکند. هر آگهی با هش محتوا شناخته می‌شود. تکراری‌ها یکی می‌شوند، آگهی تازه درج می‌شود، و آگهی‌ای که دوباره دیده شود به‌روز می‌شود و از حالت منقضی بیرون می‌آید. اگر درج دسته‌ای شکست بخورد، ردیف‌به‌ردیف دوباره می‌زنم تا یک ردیف خراب کل دور را نیندازد.

## کتابخانه‌ها

- وب: Next.js ^14.2.35، React 18.3.1، TypeScript 5.7.3، Tailwind CSS 3.4.17، PostCSS 8.4.49، Autoprefixer 10.4.20
- API: FastAPI، Uvicorn، SQLAlchemy، psycopg، Redis، Pydantic Settings، slowapi
- خزنده: Playwright، SQLAlchemy، psycopg، Pydantic Settings، HTTPX، pytest

## English

I built Hamejoo because housing, car, goods, and service listings sit on separate sites, and I wanted one phrase to be enough. The result stays on Hamejoo. The listing stays on the source site. Hamejoo is not a shop; I did not build a destination page.

Site: [hamejoo.ir](https://hamejoo.ir)

### Why I keep only metadata

On the public how-search page I wrote that a crawl reads a public page and keeps title, category, price, location, and publish time. I do not upload the image or the long description onto Hamejoo. Only a summary of at most 300 characters remains, and only the image address, not the file. Price, area, city, district, and room count sit together so search filters do not depend on the full listing text.

A click goes straight to the listing on the source site, with `rel` set to `noopener noreferrer nofollow`. I record the click with `navigator.sendBeacon`, or `fetch` with `keepalive` when beacon is missing, so leaving the page is not delayed. For each click I keep only the listing, the source, and the time. I do not store the outbound URL or the page. Otherwise Hamejoo would become a log of where the user went.

### Three programs, one database

Hamejoo is three programs: a Next.js web app on the App Router, a FastAPI API served by Uvicorn, and a Playwright crawler. The API reaches Postgres through SQLAlchemy and replaces a dead connection before running the query. Redis is the response cache. The crawler writes listings into the same database and does not serve the public search response.

The browser does not talk to the API directly. It calls a path on the web app, and the Next server rewrites that request onto the API. I did not put the API origin in the page, so changing it does not reach the client.

### Search

Search takes a text query, price and area ranges, city, district, and a sort: most relevant, newest, cheapest, or most expensive. Results come back paginated. The search page deliberately keeps a query without a city nationwide; otherwise a nationwide query would silently collapse to one city. When the text box is non-empty the sort is relevance; otherwise it is newest.

Text goes through Postgres full-text search: `to_tsvector` over title, summary, city, and district against `plainto_tsquery`, with partial `ILIKE` matches on title and district beside it. I used the `simple` configuration because listing text is Persian and Postgres's English dictionary does not help with stemming. The relevance sort uses `ts_rank` of that vector, then recency. Price sorts put listings without a price last. Price, area, city, and district filters are added only when those parameters are present. The total comes from a window `count` in the same query.

Before SQL I build a cache key with SHA-256 over the parameters. The same search is answered from Redis for 60 seconds and does not reach the database again.

A listing deleted at the source stays expired for seven days and is then physically deleted. The public how-search page states the same seven days and shows the purge stats, so the window is not only a claim.

### Suggestions

Queries that returned results are counted for autocomplete. I do not record queries shorter than 2 characters or longer than 120, so suggestions do not fill with random characters. Suggestions come from that history, then from recent listing titles, then from a fixed list when nothing else matches. Duplicates are dropped and the response is cached for 30 seconds.

Read routes are rate-limited with slowapi. The suggest-a-source form carries a honeypot field so an automated post is not recorded.

### Crawler

Each crawl round opens one shared Playwright browser, and the round has a job cap so one run does not lock the database. Each listing is identified by a content hash. Duplicates collapse, a new listing is inserted, and a listing seen again is updated and leaves the expired state. If the batch insert fails, I retry row by row so one bad row does not drop the round.

### Libraries

- Web: Next.js ^14.2.35, React 18.3.1, TypeScript 5.7.3, Tailwind CSS 3.4.17, PostCSS 8.4.49, Autoprefixer 10.4.20
- API: FastAPI, Uvicorn, SQLAlchemy, psycopg, Redis, Pydantic Settings, slowapi
- Crawler: Playwright, SQLAlchemy, psycopg, Pydantic Settings, HTTPX, pytest

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
