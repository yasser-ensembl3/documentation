# Products Scrapers — Process Documentation

## Overview

Multi-provider e-commerce product scraping toolkit with seven scrapers across five API providers (Apify, Bright Data, Channel3, DataForSEO, Zyte), a full-stack web application (Vercel + local server), and an n8n workflow for automated Shopify product scraping with PostgreSQL storage.

**Repo**: [yasser-ensembl3/Products-scrapers](https://github.com/yasser-ensembl3/Products-scrapers)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Python 3 |
| Scraping APIs | Apify, Bright Data, Channel3 SDK, DataForSEO, Zyte |
| Workflow engine | n8n (self-hosted) |
| Database | PostgreSQL |
| Frontend | Vanilla HTML/CSS/JS |
| Deployment | Vercel (serverless) |
| Storage | Google Sheets, Google Drive (OAuth2), PostgreSQL |

---

## Architecture

```
Products-scrapers/
├── Scrapers/
│   ├── apify/
│   │   └── apify_ecommerce_scraper.py      # Apify — any e-commerce site
│   ├── brightdata/
│   │   ├── brightdata_shopify_scraper.py    # Bright Data — Shopify stores
│   │   ├── brightdata_airbnb_scraper.py     # Bright Data — Airbnb Experiences
│   │   ├── api/scrape.py                    # Vercel serverless API
│   │   ├── server.py                        # Local dev server + Google Drive OAuth
│   │   └── public/index.html               # Frontend UI
│   ├── channel3/
│   │   └── channel3_gift_scraper.py         # Channel3 — gift product search
│   ├── dataforseo/
│   │   └── amazon_gift_scraper.py           # DataForSEO — Amazon products
│   └── zyte/
│       └── zyte_ecommerce_scraper.py        # Zyte — AI-powered extraction
├── n8n Workflow/
│   ├── Scraping produits (1).json           # n8n workflow definition
│   └── shopify_scraper_step1.py             # Shopify fetcher called by n8n
└── README.md
```

---

## Standalone Scrapers

### Apify — E-commerce Scraper

**File:** `Scrapers/apify/apify_ecommerce_scraper.py`

Uses Apify's "E-commerce Scraping Tool" actor to scrape any e-commerce site.

**How it works:**
1. `ApifyClient` sends URLs to the Apify actor (`aYG0l9s7dbB7j3gbS`)
2. Synchronous execution with 10-minute timeout (or async with 5s polling)
3. `normalize_product()` standardizes raw data into a uniform schema
4. Outputs JSON file with `metadata` + `products`

**Features:**
- Supports listing pages, product pages, and search result pages
- Scrape modes: `BROWSER` (full rendering) or `HTTP` (faster)
- Handles `offers` as dict or list, `brand` as dict or string, `image` as list or single
- Discount calculation: `(1 - price / price_compare) * 100`

**Auth:** `APIFY_API_TOKEN` (query parameter)

### Bright Data — Shopify Scraper

**File:** `Scrapers/brightdata/brightdata_shopify_scraper.py`

Scrapes Shopify stores via `/products.json` through Bright Data's Web Unlocker proxy.

**How it works:**
1. Takes a Shopify collection URL
2. Appends `/products.json?limit=250&page=N`
3. Routes through Bright Data's `web_unlocker1` zone
4. Paginates until empty response or max reached
5. `extract_product_data()` normalizes all variants, images, options

**Features:**
- Full pagination (250 products per page)
- Variant extraction with option1/option2/option3
- Image dimensions extraction
- Product URL construction from handle
- HTML description cleaning

**Auth:** `BRIGHTDATA_API_KEY` from `.env.local`

### Bright Data — Airbnb Experiences Scraper

**File:** `Scrapers/brightdata/brightdata_airbnb_scraper.py`

Scrapes Airbnb Experiences from search result pages.

**How it works:**
1. Fetches full HTML of Airbnb search page via Bright Data
2. BeautifulSoup scans `<script>` tags for `niobeClientData`
3. Deep JSON traversal: `data.presentation.experiencesSearch.results.searchResults`
4. Filters by `__typename == 'ExperienceSearchResult'`
5. Price extraction via regex from accessibility labels

**Features:**
- Single page scrape (no pagination)
- Constructs French Airbnb URLs: `fr.airbnb.com/experiences/{id}`
- Duration in minutes, rating, review count

**Auth:** `BRIGHTDATA_API_KEY` from `.env.local`

### Bright Data — Web Application

**Files:** `api/scrape.py`, `server.py`, `public/index.html`

Full-stack scraping application combining Airbnb + Shopify scrapers with a web UI and Google Drive integration.

**Vercel API** (`api/scrape.py`):
- POST endpoint: `{"url": "...", "scraper": "airbnb"|"shopify"}`
- Returns: `{"success": true, "scraper": "...", "count": N, "data": [...]}`
- CORS enabled, error handling returns HTTP 500

**Local Server** (`server.py`):
- `localhost:8000` development server
- Google OAuth2 flow (authorization code grant, offline access)
- Google Drive API v3 multipart file upload
- In-memory token storage (dev only)

**Frontend** (`public/index.html`):
- Dark theme UI with radio buttons for scraper selection
- Card grid for results display (images, titles, prices)
- Google Drive connect + save buttons
- Toast notifications, loading spinner
- Pure vanilla HTML/CSS/JS

**Auth:** `BRIGHTDATA_API_KEY`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_DRIVE_FOLDER_ID`

### Channel3 — Gift Product Scraper

**File:** `Scrapers/channel3/channel3_gift_scraper.py`

Scrapes gift products from Channel3's product discovery API.

**How it works:**
1. Iterates through 50 gift-related keywords (by relation, occasion, interest, type, gender/age)
2. Calls `client.search.perform()` from Channel3 SDK for each keyword
3. Deduplicates by product ID, keeping highest relevance score
4. Filters: min score 50, excludes gift cards and digital products
5. Sorts by relevance score descending

**Features:**
- 50 seed keywords organized by category
- Early stopping when `all_products >= max_products * 2`
- Uses `getattr()` for safe SDK object access

**Auth:** `CHANNEL3_API_KEY` (hardcoded placeholder)

### DataForSEO — Amazon Gift Scraper

**File:** `Scrapers/dataforseo/amazon_gift_scraper.py`

Scrapes Amazon product listings via DataForSEO Merchant API.

**How it works:**
1. **Create tasks:** POST keyword searches to DataForSEO (0.1s between tasks)
2. **Poll:** Check `get_tasks_ready()` every 2 seconds (60s max wait)
3. **Fetch results:** GET from "advanced" endpoint per completed task
4. Deduplicates by ASIN, sorts by composite popularity score

**Popularity Score (0-100):**
- Rating: max 25 points
- Review count: max 35 points (logarithmic scale)
- Amazon's Choice badge: 15 points
- Best Seller badge: 15 points
- Prime eligibility: 10 points

**Quality filters:** Min 100 reviews, min 4.0 rating

**Locations:** US (2840), CA (2124)

**Auth:** `DATAFORSEO_LOGIN` + `DATAFORSEO_PASSWORD` (HTTP Basic Auth, base64-encoded)

### Zyte — AI E-commerce Scraper

**File:** `Scrapers/zyte/zyte_ecommerce_scraper.py`

Scrapes any e-commerce site using Zyte's ML-powered extraction API.

**How it works:**
1. `extract_product_list(url)` — AI identifies product data on collection page
2. `extract_product_navigation(url)` — AI finds pagination `nextPage.url`
3. Follows pagination automatically, 1s delay between pages
4. Deduplicates by product URL
5. Preserves Zyte's `probability` confidence score

**Features:**
- No site-specific CSS selectors needed — ML-based
- Three extraction modes: list, navigation, individual product
- Rate limiting: 1s between pages

**Auth:** `ZYTE_API_KEY` (HTTP Basic Auth, key as username, empty password)

---

## n8n Workflow — Scraping produits

**Files:** `n8n Workflow/Scraping produits (1).json`, `n8n Workflow/shopify_scraper_step1.py`

### Pipeline

```
Manual Trigger
    → Google Sheets (read rows where Statut = "KO")
    → Limit (75 items per run)
    → Loop Over Items
        → Execute Command (python3 shopify_scraper_step1.py <domain_url>)
        → Code in JavaScript (parse JSON stdout → product array)
        → PostgreSQL Upsert (insert or update by url column)
    → Loop back
```

### Workflow Nodes

| Node | Type | Description |
|------|------|-------------|
| Manual Trigger | manualTrigger | Click to execute |
| Get row(s) in sheet1 | googleSheets | Read from "weddingUS" sheet, filter `Statut = "KO"` |
| Limit1 | limit | Cap at 75 items per run |
| Loop Over Items | splitInBatches | Process one domain at a time |
| Execute Command1 | executeCommand | `python3 shopify_scraper_step1.py {{ $json.domain_url }}` |
| Code in JavaScript | code | Parse stdout JSON, map to n8n items |
| Insert or update rows | postgres | Upsert into `public.products`, match on `url` |

### shopify_scraper_step1.py

Lightweight Shopify scraper designed for n8n Execute Command integration:

1. Takes domain URL from `sys.argv[1]` (auto-prepends `https://` if missing)
2. Fetches `{domain}/products.json?limit=125&sort_by=created_at&order=desc`
3. Formats each product for n8n DataTable: Website, handle, sku, title, Description, vendor, price, image, url
4. Prints JSON array to stdout (consumed by n8n)
5. Exits with code 1 on failure

No proxy needed — accesses Shopify's public `/products.json` endpoint directly.

### PostgreSQL Database Setup

Create the `products` table before running the workflow:

```sql
CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    website VARCHAR(255),
    handle VARCHAR(255),
    sku VARCHAR(255),
    title VARCHAR(255),
    description TEXT,
    vendor VARCHAR(255),
    price NUMERIC(10, 2),
    image VARCHAR(1000),
    availability BOOLEAN,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    url VARCHAR(1000) UNIQUE
);
```

The workflow upserts on the `url` column — existing products are updated with new data, new products are inserted.

### Column Mapping (n8n → PostgreSQL)

| n8n Field | PostgreSQL Column | Type |
|-----------|------------------|------|
| `$json.Website` | website | VARCHAR(255) |
| `$json.handle` | handle | VARCHAR(255) |
| `$json.sku` | sku | VARCHAR(255) |
| `$json.title` | title | VARCHAR(255) |
| `$json.Description` | description | TEXT |
| `$json.vendor` | vendor | VARCHAR(255) |
| `$json.price` | price | NUMERIC(10,2) |
| `$json.image` | image | VARCHAR(1000) |
| `$json.url` | url | VARCHAR(1000) UNIQUE |
| `$now` | created_at | TIMESTAMP |
| `$now` | updated_at | TIMESTAMP |

### Google Sheets Structure

| Column | Description |
|--------|-------------|
| domain_url | Shopify store URL (input) |
| Statut | "KO" = pending, updated after processing |

---

## Scraper Comparison

| Scraper | Provider | Target | Pagination | Auth Method | Output |
|---------|----------|--------|-----------|-------------|--------|
| Apify | Apify | Any e-commerce | Via actor | Token param | JSON file |
| Bright Data Shopify | Bright Data | Shopify stores | `?page=N` (250/page) | Bearer (env) | JSON file |
| Bright Data Airbnb | Bright Data | Airbnb Experiences | None | Bearer (env) | JSON file |
| Channel3 | Channel3 SDK | Gift products | Keyword iteration | API key | JSON file |
| DataForSEO | DataForSEO | Amazon | Async tasks | Basic Auth | JSON file |
| Zyte | Zyte | Any e-commerce | AI-detected | Basic Auth | JSON file |
| n8n Workflow | Direct (public API) | Shopify stores | None (125/req) | None | PostgreSQL |

---

## Setup

### Prerequisites

- Python 3.8+
- n8n instance (for the workflow)
- PostgreSQL (for the workflow)

### Installation

```bash
pip install requests python-dotenv beautifulsoup4
```

For the Bright Data web app:
```bash
cd Scrapers/brightdata
pip install -r requirements.txt
```

For the n8n workflow:
1. Import `n8n Workflow/Scraping produits (1).json` into n8n
2. Create the PostgreSQL `products` table (see SQL above)
3. Configure Google Sheets OAuth2 credentials in n8n
4. Configure PostgreSQL connection in n8n
5. Update the script path in the Execute Command node
6. Add Shopify store URLs to the Google Sheet with `Statut = "KO"`
7. Click "Execute workflow"

### Credentials

| Service | Type | Used By |
|---------|------|---------|
| Apify | API token | Apify scraper |
| Bright Data | API key (env) | Shopify + Airbnb scrapers |
| Channel3 | SDK API key | Channel3 scraper |
| DataForSEO | Login + password | Amazon scraper |
| Zyte | API key | Zyte scraper |
| Google Sheets | OAuth2 (n8n) | n8n workflow |
| Google Drive | OAuth2 (server) | Bright Data web app |
| PostgreSQL | Connection (n8n) | n8n workflow |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Bright Data 403/blocked | Check API key in `.env.local`, verify `web_unlocker1` zone is active |
| Shopify `/products.json` empty | Store may block public API — try Bright Data proxy scraper instead |
| DataForSEO tasks not completing | Check polling timeout (60s default), verify credentials |
| Zyte extraction inaccurate | Zyte uses AI — results vary by site, check `probability` scores |
| n8n workflow no results | Verify Google Sheet has URLs with `Statut = "KO"` |
| PostgreSQL upsert fails | Ensure `products` table exists with `url UNIQUE` constraint |
| Channel3 SDK import error | Install SDK: `pip install channel3-sdk` |
