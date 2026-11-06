# Dutch Supermarket Price Comparison: MVP Plan

> **Status:** proposed.
>
> **Decided on 2026-11-05:** a .NET backend and a Next.js frontend on current versions (§14); the MVP collects current deals plus a watchlist, not full catalogs (§2.3).

## 1. Overview

A private app that runs on your laptop. Once a day it collects prices and promotions from **Albert Heijn** and **Jumbo** and converts them to a price per kg, litre or piece. It then shows you:

- this week's deals from both stores, ranked by what you actually pay;
- for each item on your watchlist, the store that is cheapest today;
- the price history of the products you track.

Everything runs in Docker. Nothing runs while the laptop is off; when it comes back on, the day's run catches up.

### Goals (MVP)
- AH and Jumbo. A new store is a new adapter, with no changes to the pipeline, database or UI.
- One run per store per day, whenever the laptop is on that day.
- One normalization and matching pipeline, one PostgreSQL database, one .NET backend, one Next.js frontend.
- `docker compose up -d` starts everything and `docker compose stop` stops it. Data survives both.

### Non-goals (MVP)
- Full catalogs. The MVP collects current promotions plus your watchlist (§2.3).
- Fuzzy product matching. The MVP uses your pins and EANs (§7).
- Dirk, Aldi, Lidl, Plus and other stores. They come later, as new adapters.
- Offers that need a login: AH Persoonlijke Bonus, app-only coupons, loyalty-card prices.
- Notifications, hosting anywhere but the laptop, user accounts.
- In-store price differences. The online price is the reference.

## 2. Ground rules for collecting data

Neither AH nor Jumbo has a public API. Their apps call internal JSON endpoints that can change or close without notice, and both stores' terms of use restrict automated access. This plan assumes **private, non-commercial use at low volume**. Read both stores' terms before Phase 1; this plan does not establish that the use is permitted.

### 2.1 Etiquette, enforced in code
- One request at a time per store, spaced 1–2 seconds apart with random jitter.
- A hard cap per run: 500 requests per store by default. The run stops when it reaches the cap.
- Retries only after a 429, a 5xx, a timeout or a network error: at most 3, with exponential backoff, always honouring `Retry-After`.
- After 5 failures in a row, that store's run ends as `failed`.
- One fixed set of headers, containing only what the endpoint requires. No User-Agent rotation, proxies or fingerprint spoofing: evading detection invites blocks and puts you at odds with the stores' terms.
- **If a store blocks this polite client from your home connection, stop.** Don't work around the block; decide what to do in Phase 0 (§13).

### 2.2 The repository is public
- Never commit untrimmed raw responses, database dumps or backups, `.env`, or tokens. The AH anonymous token is a bearer token, so keep it out of the logs too.
- Trim test fixtures by hand before committing them: a few products, and only the fields the parsers read.
- Don't publish the collected data, and don't expose the app beyond `localhost`.

### 2.3 What the MVP collects each day

| What | Per store | Requests |
|---|---|---|
| Current promotions | every page of the store's promotions listing | tens |
| Watchlist terms | the first page of search results for each active term (20 products by default) | one per term |
| Pinned products | a refresh by id of each pinned product | one per pin |
| EAN enrichment | a detail call, once per new product, only when listings lack the EAN; at most 100 per run | 0–100 |

In total, a few hundred requests a day. Phase 0 measures the real numbers.

## 3. Architecture

```text
      Albert Heijn app API          Jumbo app API
                ^                         ^
                +------------+------------+
                             |  HTTPS, one polite client per store
+----------------------------|----------- docker compose (laptop) ---+
|                            |                                       |
|   worker (.NET) -----------+    daily pipeline (§5)                |
|     |                                                              |
|     | reads and writes                                             |
|     v                                                              |
|   db (PostgreSQL 18) <------ migrate (applies migrations, exits)   |
|     ^                                                              |
|     | reads; writes pins and watchlist                             |
|   api (ASP.NET Core minimal API, internal port 8080)               |
|     ^                                                              |
|     | server-side fetch                                            |
|   web (Next.js, published on 127.0.0.1:3000)                       |
|                                                                    |
+--------------------------------------------------------------------+
                 ^
                 |  http://localhost:3000
            your browser
```

| Service | Built from | Role | Port |
|---|---|---|---|
| `db` | `postgres:18` | the only database | `127.0.0.1:5432`, for database tools |
| `migrate` | the worker image, run with `--migrate` | applies EF Core migrations, then exits | — |
| `worker` | `src/SupermarketPrices.Worker` | scheduler, collectors and pipeline | — |
| `api` | `src/SupermarketPrices.Api` | read endpoints, pins and watchlist | `8080`, inside the compose network only |
| `web` | `web/` (Next.js standalone build) | UI | `127.0.0.1:3000` |

The API isn't published to the host. Next.js Server Components call it over the compose network, so the browser never talks to it: no CORS, and one less open port. To browse `/openapi/v1.json` while developing, publish `127.0.0.1:8080` temporarily.

## 4. Code layout and design patterns

```text
nl-supermarket-price-comparer/
├─ src/
│  ├─ SupermarketPrices.Core/        shared by both hosts
│  │  ├─ Domain/                     Money, Quantity, UnitPrice, Promotion, entities
│  │  ├─ Stores/
│  │  │  ├─ IStoreAdapter.cs
│  │  │  ├─ Ah/                      AhAdapter, AH JSON models, AhOptions
│  │  │  └─ Jumbo/                   JumboAdapter, Jumbo JSON models, JumboOptions
│  │  ├─ Pipeline/                   Collector, Processor, SizeParser, PromotionRules/, Matchers/
│  │  └─ Persistence/                PricesDbContext, Migrations/
│  ├─ SupermarketPrices.Worker/      host: scheduler, CLI modes, Dockerfile
│  └─ SupermarketPrices.Api/         host: minimal API endpoints, Dockerfile
├─ tests/
│  └─ SupermarketPrices.Tests/       Unit/, Stores/ (fixtures), Integration/
├─ web/                              Next.js app, Dockerfile
├─ plan/
├─ docker-compose.yml
├─ .env.example                      committed; .env is not
├─ Directory.Packages.props          every NuGet version in one place
├─ global.json                       pins the .NET SDK
├─ .gitignore, .gitattributes
└─ README.md                         run, stop, back up
```

Shared code lives in one class library, and the two hosts stay thin. A worker, an API, shared code and tests make four projects; that's the minimum for this design.

### 4.1 Store adapters (Adapter + Registry)

Each store sits behind one interface. An adapter fetches data and translates store JSON into a common shape. It does no unit maths, no matching and no database work.

```csharp
public interface IStoreAdapter
{
    string StoreCode { get; }                                   // "ah", "jumbo"
    string StoreName { get; }                                   // "Albert Heijn", "Jumbo"

    IAsyncEnumerable<RawResponse> FetchPromotionsAsync(CancellationToken ct);
    IAsyncEnumerable<RawResponse> SearchAsync(string term, int maxItems, CancellationToken ct);
    Task<RawResponse> FetchProductAsync(string externalId, CancellationToken ct);

    IEnumerable<ScrapedItem> Parse(RawResponse response);       // store JSON -> common shape
}

public sealed record RawResponse(string RequestKey, string Json);

public sealed record ScrapedItem(
    string ExternalId, string Name, string? Brand, string? Ean,
    string? SizeLabel,                 // "6 x 330 ml", as the store shows it
    int PriceCents,                    // regular shelf price
    string? UnitPriceLabel,            // "€3,06 per liter", as the store shows it
    string? PromoLabel,                // "2e halve prijs"
    int? PromoPriceCents,              // when the store states the promotion price
    DateOnly? PromoFrom, DateOnly? PromoUntil,
    string? Category, string? Url, string? ImageUrl);
```

Each store needs one registration line. The pipeline receives `IEnumerable<IStoreAdapter>` and runs only the stores enabled in configuration:

```csharp
builder.Services.AddStoreAdapter<AhAdapter>("ah");
builder.Services.AddStoreAdapter<JumboAdapter>("jumbo");
// appsettings.json: "Stores": { "Enabled": [ "ah", "jumbo" ] }
```

`AddStoreAdapter<T>` also gives the store's typed `HttpClient` the shared handlers from §4.2, so no adapter can skip the etiquette rules. At startup the worker upserts a `stores` row for every registered adapter.

**Adding a store (Dirk, for example):**
1. Add a `Stores/Dirk/` folder with the adapter and its JSON models.
2. Add trimmed fixtures and parser tests.
3. Add one `AddStoreAdapter` line.
4. Add `"dirk"` to `Stores:Enabled`.

Nothing in `Pipeline/`, the database schema, the API or the UI changes.

### 4.2 One polite HTTP client (Decorator)

Every store's `HttpClient` gets the same two handlers:

- **Outside:** `AddStandardResilienceHandler()` from `Microsoft.Extensions.Http.Resilience`. It retries with backoff, honours `Retry-After` and applies timeouts, set to the limits in §2.1.
- **Inside:** `PoliteHandler`, a small custom `DelegatingHandler`. It spaces requests and adds jitter, enforces the per-run cap, counts requests for `scrape_runs`, and never logs authorization headers.

Because `PoliteHandler` sits inside the retry handler, every attempt, retries included, is spaced and counted. The "5 failures in a row" rule lives in the collector, not in the circuit breaker. The circuit breaker's defaults are tuned for busy services and would never trip at one request every second or two.

### 4.3 Promotion rules (Strategy)

Each promotion mechanism is one class. The rules are tried in order, and the first one that recognises the label wins.

```csharp
public interface IPromotionRule
{
    bool TryApply(string label, int regularPriceCents, out PromotionTerms terms);
}

public sealed record PromotionTerms(
    PromotionType Type,          // PercentOff, XPlusYFree, NthHalfPrice, NForPrice, FixedPrice
    int MinQuantity,             // how many you must buy to get the deal
    int EffectivePriceCents);    // price per item when you buy MinQuantity
```

When no rule recognises a label, the promotion is stored as `Unknown` with the raw text, shown as-is in the UI, and listed in the run log so you know to add a rule. It is never dropped.

### 4.4 Matchers (Chain of Responsibility)

```csharp
public interface IProductMatcher
{
    // Returns the canonical product id, or null to hand over to the next matcher.
    Task<long?> MatchAsync(StoreProduct item, CancellationToken ct);
}
```

The MVP runs `ManualLinkMatcher` first, then `EanMatcher`. Your pins always win, and nothing overwrites them. A trigram suggestion matcher can be added to the end later without touching the others (§7).

### 4.5 Deliberately not used
- Generic repositories or a unit of work on top of EF Core: `DbContext` already is both.
- MediatR, CQRS, message queues, or a service per store: there's one daily job and a handful of endpoints.
- A plugin system with assembly scanning: one registration line per store is simpler and easier to follow.

## 5. Daily pipeline

### 5.1 Scheduler with catch-up

A cron job at 04:00 rarely fires on a laptop. Instead, the worker runs a `BackgroundService` with a `PeriodicTimer`:

- It ticks once at startup, then every 15 minutes.
- On each tick, a store runs when all of these hold:
  - it has no `succeeded` run for today's **Europe/Amsterdam** date (a `suspect` run is retried while attempts remain);
  - it's 06:00 or later, because promotions switch overnight and a run just after midnight can catch the old week (configurable);
  - it has had fewer than 3 attempts today, and none of them was blocked (a 403 or a challenge page ends that store's attempts for the day; §2.1).
- Stores run one after the other, independently: a failed Jumbo run doesn't stop AH.
- At startup, a run still marked `running` whose store lock (§5.2) is free stopped mid-run, so it's marked `failed`.

Whether a store should run now depends only on its run history, the current time and the configuration, so it's a pure function with table-driven tests. The loop around it is tested with `FakeTimeProvider`.

Set Docker Desktop to start when you sign in, and give the services `restart: unless-stopped`. Then opening the laptop is enough to get today's data.

### 5.2 What one store run does
1. **Start:** take a per-store Postgres advisory lock, so the scheduler and a `--run-once` never collect the same store at the same time (if the lock is taken, skip the store). Then insert a `scrape_runs` row with status `running`.
2. **Collect:** call the adapter for promotions, for each active watchlist term and for each pinned product. Then fetch details for new products whose listing has no EAN (at most 100 per run; §2.3). Store every response body in `raw_responses`, unparsed.
3. **Process:** read this run's raw responses, then:
   - `Parse` them (adapter);
   - **Normalize** (§6);
   - **Match** (§7);
   - **Load**: upsert `store_products` and `price_snapshots`, and replace the day's `watchlist_hits` for each term and store, in one transaction.
4. **Check:** compare the promotion count with the store's previous good run. If it dropped by more than 50%, the run is marked `suspect`. Its data is kept but shown with a warning, and nothing is inferred from the missing products.
5. **Finish:** set the status to `succeeded`, `suspect` or `failed`, with the counts and any error.
6. **Clean up:** delete raw responses older than 30 days.

Process always reads stored raw JSON, so a fresh run and a reprocess use the same code path.

### 5.3 Worker modes (one image)

| Arguments | What it does |
|---|---|
| *(none)* | the scheduler loop: the normal mode |
| `--migrate` | applies EF Core migrations and exits (used by the `migrate` service) |
| `--run-once [--store ah]` | runs today's collection now, ignoring the schedule |
| `--reprocess <runId>` | re-runs Process on stored raw JSON, without network calls |

For example: `docker compose run --rm worker --run-once --store jumbo`.

## 6. Normalization and promotions

### 6.1 Sizes and unit prices
- There are three base units: `kg`, `l` and `piece`. Prices stay integers, in cents.
- **Use the store's own unit price first.** EU price-indication rules require Dutch shops to show a price per kilo or litre, and both apps appear to send it. Parse that label. Compute a unit price from the size only as a fallback, and log differences above 2% as a check on the parser.
- The size parser understands Dutch labels. Its test table starts with these and grows with every new label Phase 0 finds:

| Label | Quantity | Unit |
|---|---|---|
| `500 g` | 0.5 | kg |
| `1,5 l` | 1.5 | l |
| `6 x 330 ml` | 1.98 | l |
| `25 cl` | 0.25 | l |
| `ca. 500 g` | 0.5 | kg (approximate) |
| `per stuk` / `4 stuks` | 1 / 4 | piece |
| `per kilo` | 1 | kg |

When a label can't be parsed, the raw text is kept and the quantity is null. The product still appears, just without a unit price.

### 6.2 Promotions
Each promotion label becomes a type, a minimum quantity and an effective price per item:

| Mechanism | Example labels | Minimum quantity | Effective price per item |
|---|---|---|---|
| Percent off | `25% korting` | 1 | price × (100 − p) ÷ 100 |
| X + Y free | `1+1 gratis`, `2+1 gratis`, `3 voor 2` | X + Y | price × X ÷ (X + Y) |
| Second at half price | `2e halve prijs` | 2 | price × 0.75 |
| N for a fixed amount | `2 voor €3,00`, `3 voor 5,-` | N | amount ÷ N |
| Fixed price | the store sends a promotion price | 1 | the promotion price |
| Unknown | anything else | — | none; the label is shown as-is |

The effective unit price is the effective price divided by the quantity, and Deals sort on it.

**Example:** a 650 ml jar of mayonnaise costs €1.99, or €3.06/L. With `2e halve prijs` it costs €1.49 each when you buy two, so Deals ranks it at €2.30/L.

Promotion start and end dates come from the store, if it sends them; Phase 0 finds out. If it doesn't, the UI shows "on offer since <first day seen>" and drops the "ending soonest" sort.

## 7. Matching

The MVP has two ways to say "these products from different stores are the same thing". They run in this order:

1. **Your pins (manual, permanent).** On a watchlist result, you pin the AH and Jumbo products you consider equivalent, and they become one canonical product. The matcher never overwrites a manual link. Pins are the only way to compare store brands such as "AH Halfvolle melk" and "Jumbo Halfvolle melk", because store brands never share an EAN.
2. **EAN.** If both stores expose EANs, products with the same EAN are linked automatically. Phase 0 checks AH; a third-party Jumbo scraper reports an `ean` field. This covers branded products, for example a Coca-Cola deal at Jumbo next to AH's price for the same bottle.

A **watchlist term** is a third, looser grouping: everything the term returns from both stores, ranked by effective unit price. That alone answers "where is halfvolle melk cheapest today?" without any matching.

**Later, only if needed:** trigram suggestions with Postgres `pg_trgm`, using `similarity()` on normalized names. Hard gates would apply: same base unit, size within ±10%, and the same brand when both products have one. Suggestions go to a review list, and nothing is linked until you click.

## 8. Data model

EF Core migrations are the source of truth; the SQL below shows the intended shape. Conventions:
- Internal ids are `bigint` identity columns, and text columns are `text`.
- Money is integer cents.
- Timestamps are `timestamptz`, stored in UTC.
- "The day" is a `date` in Europe/Amsterdam.

```sql
CREATE TABLE stores (
    code text PRIMARY KEY,                    -- 'ah', 'jumbo'; upserted at worker startup
    name text NOT NULL
);

CREATE TABLE scrape_runs (
    id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    store_code    text NOT NULL REFERENCES stores(code),
    run_date      date NOT NULL,              -- Europe/Amsterdam
    started_at    timestamptz NOT NULL DEFAULT now(),
    finished_at   timestamptz,
    status        text NOT NULL CHECK (status IN ('running', 'succeeded', 'suspect', 'failed')),
    request_count int NOT NULL DEFAULT 0,
    item_count    int NOT NULL DEFAULT 0,
    error         text
);
CREATE INDEX ON scrape_runs (store_code, run_date);

CREATE TABLE raw_responses (                  -- kept 30 days
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    run_id      bigint NOT NULL REFERENCES scrape_runs(id) ON DELETE CASCADE,
    request_key text NOT NULL,                -- 'promotions?page=3', 'search:halfvolle melk'
    payload     jsonb NOT NULL,
    fetched_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON raw_responses (run_id);

CREATE TABLE products (                       -- canonical: "the thing you buy"
    id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name      text NOT NULL,
    base_unit text NOT NULL CHECK (base_unit IN ('kg', 'l', 'piece'))
);

CREATE TABLE store_products (                 -- one row per product per store
    id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    store_code    text NOT NULL REFERENCES stores(code),
    external_id   text NOT NULL,
    ean           text,                       -- GTIN, when the store exposes it
    name          text NOT NULL,
    brand         text,
    size_label    text,                       -- '6 x 330 ml'
    size_quantity numeric(12, 3),             -- 1.980
    base_unit     text CHECK (base_unit IN ('kg', 'l', 'piece')),
    category      text,                       -- the store's own category path
    url           text,
    image_url     text,
    product_id    bigint REFERENCES products(id) ON DELETE SET NULL,
    match_method  text CHECK (match_method IN ('manual', 'ean')),
    first_seen_at timestamptz NOT NULL DEFAULT now(),
    last_seen_at  timestamptz NOT NULL DEFAULT now(),
    UNIQUE (store_code, external_id)
);
CREATE INDEX ON store_products (ean) WHERE ean IS NOT NULL;
CREATE INDEX ON store_products (product_id);

CREATE TABLE price_snapshots (                -- one row per product per day; re-runs overwrite it
    store_product_id                 bigint   NOT NULL REFERENCES store_products(id) ON DELETE CASCADE,
    observed_on                      date     NOT NULL,
    run_id                           bigint   NOT NULL REFERENCES scrape_runs(id),
    price_cents                      int      NOT NULL,
    unit_price_cents                 int,
    promo_type                       text,    -- null when there is no promotion
    promo_label                      text,
    promo_min_quantity               smallint,
    promo_effective_price_cents      int,
    promo_effective_unit_price_cents int,
    promo_valid_from                 date,
    promo_valid_until                date,
    PRIMARY KEY (store_product_id, observed_on)
);

CREATE TABLE watchlist_entries (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    term       text NOT NULL UNIQUE,          -- 'halfvolle melk'
    max_items  smallint NOT NULL DEFAULT 20,
    active     boolean NOT NULL DEFAULT true,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE watchlist_hits (                 -- what a term returned, per store, per day
    watchlist_entry_id bigint   NOT NULL REFERENCES watchlist_entries(id) ON DELETE CASCADE,
    store_product_id   bigint   NOT NULL REFERENCES store_products(id) ON DELETE CASCADE,
    observed_on        date     NOT NULL,
    rank               smallint NOT NULL,
    PRIMARY KEY (watchlist_entry_id, store_product_id, observed_on)
);
```

A *pinned* product is a store product with `match_method = 'manual'`; the worker refreshes those by id every day.

At deals-plus-watchlist volume (several thousand snapshot rows a day, plus 30 days of raw JSON), the database should grow by less than 1 GB a year; Phase 0 gives real numbers. It needs no partitioning and no compression scheme.

## 9. API (ASP.NET Core minimal API)

| Endpoint | Returns or does |
|---|---|
| `GET /api/status` | the last run per store: status, finish time, counts. Drives the freshness banner |
| `GET /api/deals?store=&category=&q=&sort=unit_price\|discount\|ends&page=` | current promotions from both stores |
| `GET /api/watchlist` | each term with today's cheapest option per store, and whether it's on promotion |
| `POST /api/watchlist` | adds a term. If today's run is already done, the term is first fetched tomorrow, unless you run `--run-once` |
| `DELETE /api/watchlist/{id}` | removes a term |
| `GET /api/watchlist/{id}/results` | today's results for a term from both stores, ranked by effective unit price |
| `POST /api/products/pins` | links the given store products into one canonical product (new or existing) |
| `DELETE /api/products/pins/{storeProductId}` | unlinks a store product |
| `GET /api/products/{id}` | a canonical product: today's price per store and 90 days of history |

- OpenAPI is built in (`AddOpenApi()` and `MapOpenApi()`). The frontend generates its TypeScript types from `/openapi/v1.json` with `openapi-typescript`, so the API and the UI can't drift apart.
- Errors use the built-in Problem Details responses.
- There's no authentication: the API isn't published to the host, and the web app only listens on `127.0.0.1`. Add authentication before exposing either one anywhere else.

## 10. Frontend (Next.js)

- **App Router.** Server Components fetch from the API at `API_BASE_URL=http://api:8080`.
- **Mutations** (adding a term, pinning) are Server Actions that call the API and revalidate the page. The browser never calls the API directly, so no CORS is needed.
- **No caching layer:** the data changes once a day and there's one user.
- **Libraries:** Tailwind CSS (offered by `create-next-app`) and Recharts for the history chart. Add others only when needed.

| Page | Shows |
|---|---|
| `/` (Today) | the watchlist summary (cheapest store per term, promotion badges, change since last week) and deals that weren't there yesterday |
| `/deals` | every current promotion. Filter by store, category or text; sort by effective unit price, discount, or ending soonest. A matched deal links to its product page; an unmatched one links to the store's site |
| `/watchlist/[id]` | a term's results from both stores, ranked by effective unit price, with pin buttons |
| `/products/[id]` | today's price at each store and a 90-day history chart |
| every page | a freshness banner, for example "AH updated today at 08:12 · Jumbo failed; last good data is from yesterday" |

"The latest and the cheapest" maps to two things: today's prices, with the banner showing how fresh they are, and deals that are new since yesterday.

## 11. Running it

### 11.1 `docker-compose.yml` (sketch)

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_DB: prices
      POSTGRES_USER: prices
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}       # from .env, which is gitignored
    volumes:
      - pgdata:/var/lib/postgresql                  # 18+: not /var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432"                       # local database tools only
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U prices -d prices"]
      interval: 5s
      retries: 10
    restart: unless-stopped

  migrate:
    image: supermarket-prices-worker               # shared with worker, so it's built once
    build: { context: ., dockerfile: src/SupermarketPrices.Worker/Dockerfile }
    command: ["--migrate"]
    environment: &db-connection
      ConnectionStrings__Prices: "Host=db;Database=prices;Username=prices;Password=${POSTGRES_PASSWORD}"
    depends_on:
      db: { condition: service_healthy }
    restart: "no"

  worker:
    image: supermarket-prices-worker
    build: { context: ., dockerfile: src/SupermarketPrices.Worker/Dockerfile }
    environment: *db-connection
    depends_on:
      migrate: { condition: service_completed_successfully }
    restart: unless-stopped

  api:
    build: { context: ., dockerfile: src/SupermarketPrices.Api/Dockerfile }
    environment: *db-connection                     # listens on 8080 inside the network only
    depends_on:
      migrate: { condition: service_completed_successfully }
    restart: unless-stopped

  web:
    build: { context: ./web }
    environment:
      API_BASE_URL: http://api:8080
    ports:
      - "127.0.0.1:3000:3000"
    depends_on: [api]
    restart: unless-stopped

volumes:
  pgdata:
```

`.env.example` (committed) holds `POSTGRES_PASSWORD=change-me`. Copy it to `.env` and pick a real password.

### 11.2 Everyday commands (they go into the README in Phase 1)

```bash
docker compose up -d --build                  # start everything (first time, or after code changes)
docker compose stop                           # stop; all data stays
docker compose start                          # start again
docker compose logs -f worker                 # watch today's run
docker compose run --rm worker --run-once     # collect now instead of waiting for the next tick
docker compose down                           # remove the containers; the database volume stays
```

`docker compose down -v` also deletes the database volume, and your price history with it. That history can't be scraped again.

### 11.3 Backups
The price history is the one thing you can't recreate. Back it up weekly:

```bash
docker compose exec db pg_dump -U prices -Fc -f /tmp/prices.dump prices
docker compose cp db:/tmp/prices.dump ./backups/prices.dump
```

To restore, stop the apps first, then:

```bash
docker compose stop worker api web
docker compose cp ./backups/prices.dump db:/tmp/prices.dump
docker compose exec db pg_restore -U prices -d prices --clean --if-exists /tmp/prices.dump
docker compose start worker api web
```

`backups/` is gitignored. Don't capture `pg_dump` output with `>` in Windows PowerShell: it re-encodes the bytes, and the dump won't restore. The same redirection is the likely reason the current `README.md` is UTF-16.

### 11.4 Windows notes
- In Docker Desktop, turn on "Start Docker Desktop when you sign in to your computer". With `restart: unless-stopped`, the stack comes back after a reboot, and the startup tick catches up on the day.
- For development, run only the database in Docker (`docker compose up -d db`) and run the apps on the host (`dotnet watch`, `npm run dev`) for fast reloads. Use the full stack for everyday use.
- Add a `.gitattributes` with `* text=auto eol=lf` so line endings stay consistent between Windows and the Linux containers.

## 12. Testing

| Layer | What | How |
|---|---|---|
| Unit | size parser, promotion rules, unit-price maths, matchers, the "should this store run now?" decision | xUnit v3, table-driven; `FakeTimeProvider` for the clock |
| Store adapters | `Parse()` on recorded responses | trimmed fixtures under `tests/SupermarketPrices.Tests/Stores/<store>/`; tests make no network calls |
| Integration | the whole Process step against real Postgres, and that running it twice changes nothing | Testcontainers with `postgres:18` |
| Frontend | types, lint, production build | `tsc --noEmit`, `eslint`, `next build` |

Nothing tests the live endpoints automatically. A run that parses zero products, or turns up many `Unknown` promotion labels, shows in `scrape_runs` and in the banner. That's your alarm that a store changed its API.

## 13. Roadmap

Each phase is one branch and one pull request, and ends with its exit check.

### Phase 0: data-source spike (throwaway code)
From your home connection, use `.http` files (Visual Studio, Rider, or VS Code with the REST Client extension) or a scratch console app to answer, per store:

| Question | AH | Jumbo |
|---|---|---|
| How do you list all current promotions? Paging? Page size? | | |
| Search by term: endpoint and maximum page size? | | |
| Product by id? | | |
| EAN in listings, only in detail, or not at all? | | |
| Unit price sent? In what format? | | |
| Promotion label, promotion price, validity dates? | | |
| Authentication: anonymous token? How long does it last? | | |
| Does a plain, slow client get through from home? | | |

Leads to start from (third-party, not verified):
- **AH:** the community clients [`python-appie`](https://pypi.org/project/python-appie/) (0.2.2, March 2026) and [`supermarktconnector`](https://github.com/bartmachielsen/SupermarktConnector) use an anonymous token from `/mobile-auth/v1/auth/token/anonymous` and search at `/mobile-services/product/search/v2`. `python-appie` also implements product detail and is the best reference for the required headers.
- **Jumbo:** a [commercial scraper's documentation](https://apify.com/studio-amba/jumbo-scraper) lists `ean`, `unitPrice`, `promotion` and `promotionPrice` fields from Jumbo's search backend, with no promotion dates. It also says the API sits behind Cloudflare and only answers Dutch IPs, so it won't work through a non-Dutch VPN.

**Exit:** the table is filled in and copied into §15, each endpoint has 3–5 trimmed fixtures, and each store has a go/no-go. A no-go for Jumbo means no workaround (§2.1). Instead, choose between waiting, another store, or another source, such as the open [checkjebon](https://github.com/supermarkt/checkjebon) dataset if it's still updated (its repository was last updated in November 2024).

### Phase 1: skeleton, and AH deals end to end
- **Repo hygiene:** `.gitignore` (dotnet, node, `.env`, `backups/`), `.gitattributes`, and `README.md` re-saved as UTF-8 (it's UTF-16 today, so git treats it as binary).
- **Setup:** the solution, `global.json`, `Directory.Packages.props` and `.env.example`; `db`, `migrate` and `worker` in compose.
- **Pipeline:** the EF Core model and first migration; `IStoreAdapter`, `PoliteHandler`, collection of AH promotions, raw storage, the size parser and Load.
- **README:** how to run, stop, run once and back up.

**Exit:** `docker compose run --rm worker --run-once --store ah` fills in today's AH deals. Running it a second time changes no row counts, and the tests pass.

### Phase 2: Jumbo, promotions, scheduler
- The Jumbo adapter. It's the test of the pattern: nothing outside `Stores/Jumbo/` and its registration line changes.
- Promotion rules and effective prices.
- The catch-up scheduler, `scrape_runs` statuses and the sanity check.

**Exit:** after the laptop has been off overnight, both stores have today's data within 15 minutes of turning it on. When one store fails, the other still loads.

### Phase 3: API and deals UI
- The minimal API (`status`, `deals`) and the OpenAPI-generated TypeScript types.
- The Next.js app with the Deals page and the freshness banner.
- `api` and `web` added to compose.

**Exit:** from a clean clone, `docker compose up -d` shows today's deals at `http://localhost:3000`, and `stop` followed by `start` keeps everything.

### Phase 4: watchlist, pins, history
- Watchlist terms and their results, and pins (manual links).
- The daily refresh of pinned products by id.
- The EAN matcher, if Phase 0 found EANs at both stores.
- The Today, Watchlist and Product pages, with history.

**Exit:** add "halfvolle melk", run `--run-once`, and see both stores ranked by €/L. Then pin one product per store and see a single product with both prices and its history.

### Later (not in the MVP)
- More stores (Dirk, Plus, …), each a new adapter.
- Trigram match suggestions (§7).
- A daily digest when a watched product goes on promotion (ntfy, Telegram or email, for example).
- Automated weekly backups.
- Full catalogs, if the watchlist turns out too narrow.

## 14. Versions

Checked on 2026-11-05 against each project's release feed:

| Component | Use | Notes |
|---|---|---|
| .NET (SDK, runtime, ASP.NET Core, EF Core) | **10.0** (runtime 10.0.14): LTS, supported until 2028-11-14 | 11.0 is at RC3 (go-live); see the rule below |
| Npgsql.EntityFrameworkCore.PostgreSQL | **10.0.3** | 11.0.0-rc.3 is out |
| PostgreSQL | **18** (18.6), image `postgres:18` | 19 is at RC1. Major upgrades need a dump and restore or `pg_upgrade`, so stay on 18 until there's a reason to move |
| Node.js (for the `web` image) | **26 LTS** (26.13.0), image `node:26-alpine` | |
| Next.js | **16.4.5** | needs Node 20.9 or later |
| React | **19.3.1** | |

**The .NET 11 rule:** if .NET 11.0.0 and Npgsql 11.0.0 are both released (GA) when Phase 1 starts, use 11.
- .NET 9 got 24 months of STS support (until 2026-11-10 in the release feed), so 11 should be supported until about November 2028, the same as 10.
- Otherwise, start on 10. Moving to 11 later means changing `global.json`, `Directory.Packages.props` and the Dockerfile base images.

**Version policy:**
- **Pin exact versions:** NuGet packages in `Directory.Packages.props`, npm packages through `package-lock.json`, the SDK in `global.json`.
- **Docker images pin only the major version** (`postgres:18`, `node:26-alpine`, `mcr.microsoft.com/dotnet/aspnet:10.0`), so `docker compose build --pull` picks up security patches.
- **Smaller libraries** (xUnit v3, Testcontainers, Tailwind, Recharts, `openapi-typescript`) take the latest stable version when they're added.
- **Dependabot** for NuGet, npm and Docker keeps everything current.

## 15. Risks and data-source facts

| Risk | Impact | Mitigation |
|---|---|---|
| A store changes or closes its app API | that store's run fails, or parses nothing | raw responses and fixtures make the fix quick; failed and zero-product runs show in the banner; the other store keeps working |
| Jumbo blocks a plain client (Cloudflare) | no Jumbo data | stop rather than evade (§2.1); decide in Phase 0 |
| AH listings have no EAN | no automatic matching of branded products at AH | one detail call per new product (capped), or rely on pins |
| No promotion validity dates | no "ending soonest" | show "on offer since"; drop that sort |
| Terms of use | a store can object to automated access | private, low volume, no republishing; read the terms before Phase 1 |
| Laptop off for days | gaps in the history | expected; the UI shows the gaps instead of inventing prices |

Phase 0 copies its findings into this section as the reference for the adapters.
