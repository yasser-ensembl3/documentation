# Options Model — Process Documentation

## Overview

Montreal Exchange options trading dashboard. Scrapes all listed options from m-x.ca via a Python async scraper (`mx_scraper.py`), stores them in PostgreSQL with daily history, and displays them in a dark mode Next.js dashboard with interactive charts, real-time filters, and AI-powered investment analysis via OpenAI GPT-4o.

**Repo**: [yasser-ensembl3/Options](https://github.com/yasser-ensembl3/Options)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Framework | Next.js 14 (App Router), React 18 |
| Language | TypeScript |
| Styling | TailwindCSS 3.4 |
| Charts | Recharts (LineChart, BarChart, ComposedChart) |
| PDF Export | jsPDF + html2canvas |
| Database | PostgreSQL (pg driver with connection pooling) |
| AI | OpenAI GPT-4o |
| Scraper | Python 3.10+ (asyncio, httpx, BeautifulSoup, psycopg2) |

---

## Architecture

```
Options model/
├── mx_scraper.py                      # Python scraper (Montreal Exchange → PostgreSQL)
├── app/
│   ├── api/
│   │   ├── options/
│   │   │   ├── route.ts               # Main API — query PostgreSQL for options
│   │   │   ├── symbols/route.ts       # Dynamic symbol list from DB
│   │   │   └── export-today/route.ts  # Export all data created today
│   │   ├── stock-price/route.ts       # Fetch stock price from DB
│   │   ├── analyze/route.ts           # AI analysis via OpenAI GPT-4o
│   │   └── scrape/route.ts            # Legacy n8n webhook proxy (unused)
│   ├── dark/
│   │   ├── page.tsx                   # Main dashboard (dark mode)
│   │   └── print-preview/page.tsx     # PDF export preview
│   ├── page.tsx                       # Legacy page
│   ├── layout.tsx                     # Root layout
│   └── globals.css                    # TailwindCSS styles
├── components/
│   ├── DataFilters.tsx                # Filter UI (legacy, built into dark mode)
│   ├── OptionsTable.tsx               # Table display (legacy)
│   └── AIAnalysis.tsx                 # AI analysis display (legacy)
├── backup_n8n_config/                 # Archived n8n config (replaced by PostgreSQL)
├── package.json
├── tailwind.config.js
└── tsconfig.json
```

---

## Pipeline — How It Works

### Full Flow

```
mx_scraper.py (Python, daily)
    │
    ├── 1. Fetch company list from m-x.ca (~200+ companies)
    ├── 2. Scrape options data (Calls + Puts) — 10 concurrent
    ├── 3. Scrape underlying stock prices
    └── 4. Upsert into PostgreSQL (batch every 10 companies)
    │
    ▼
PostgreSQL (mondb_test)
    ├── db_option — options contracts with daily history
    └── db_stock_price — underlying stock prices
    │
    ▼
Next.js API Routes
    │
    ├── /api/options — query by symbol or date range
    ├── /api/options/symbols — dynamic symbol list
    ├── /api/stock-price — latest stock price
    └── /api/analyze — send to OpenAI GPT-4o
    │
    ▼
Dark Mode Dashboard (/dark)
    ├── 4 interactive charts
    ├── Real-time filters & sorting
    ├── Tab system for multi-symbol comparison
    └── Export: PDF + JSON
```

### Step 1: Data Scraping — mx_scraper.py

The Python scraper is the data collection engine. Run it daily (or as needed) to populate PostgreSQL.

**Process:**

1. **Fetch companies** — GET `m-x.ca/en/trading/data/options-list#etf`, parse HTML table to extract company names and option page links (~200+ companies)

2. **Scrape options** (async, 10 concurrent requests) — For each company page, parse `<tr class="parent" data-row>` elements containing JSON option data. Extract both Call and Put options with 20+ fields each.

3. **Scrape stock prices** — From the same page, parse `<div class="quote-info">` to extract last_price, bid_price, ask_price for the underlying stock.

4. **Batch insert** — Every 10 companies, upsert into PostgreSQL. Uses `ON CONFLICT (symbol, scrape_date) DO UPDATE` to handle same-day re-runs.

**Fields per option:** symbol, quotes (call/put), expiration_date, strike_price, bid/ask price, bid/ask size, open/high/low price, last_close_price, net_change, settlement_price, volatility, open_interest, nb_trades, is_option, is_weekly, scrape_date.

**Performance:** ~145 seconds for 200+ companies, ~12,000+ options per run.

```bash
python mx_scraper.py
```

### Step 2: API Layer

Next.js API routes query PostgreSQL via connection pooling (`pg` library).

| Route | Method | Description |
|-------|--------|-------------|
| `/api/options` | GET | Fetch options with 90-day expiration filter, or `?export=all` for everything |
| `/api/options` | POST | Fetch options for a specific symbol (finds closest scrape_date) |
| `/api/options/symbols` | GET | List unique symbol codes from database |
| `/api/options/export-today` | GET | Export all options scraped today |
| `/api/stock-price` | GET | Fetch most recent stock price for a symbol |
| `/api/analyze` | POST | Send options data to OpenAI GPT-4o for analysis |

Data is transformed from PostgreSQL column format to the frontend's expected format (legacy n8n-compatible structure) via `transformDataToN8nFormat()`.

### Step 3: Dashboard Visualization

The primary interface is `/dark` — a dark mode professional dashboard.

**Charts (Recharts):**

| Chart | What It Shows |
|-------|---------------|
| Volatility Smile | Implied volatility plotted against strike price |
| Volume by Strike | Top 15 most liquid strikes by open interest |
| IV Term Structure | How implied volatility changes across expiration dates |
| Call/Put Ratio | Ratio of calls to puts — market sentiment indicator |

**Filters (8 types):**
- Type: Call / Put
- Date range: min / max expiration
- Strike range: min / max
- Volatility range: min / max
- Weekly: Weekly / Standard flag

**Sorting:** 6 columns (Date, Strike, Volatility, Open Interest, Bid, Ask) with asc/desc toggle.

**Tab system:** Each symbol search creates a new tab. Multiple symbols can be loaded simultaneously for side-by-side comparison.

### Step 4: AI Analysis (Optional)

POST `/api/analyze` sends the current options dataset to OpenAI GPT-4o.

**Prompt structure:**
- System: Options trading expert persona
- User: Structured data summary + request for market overview, buy/avoid recommendations, strategic advice, risk warnings
- Temperature: 0.7, Max tokens: 2000

**Output:** Markdown-formatted analysis with risk levels (green/yellow/red), rendered via `react-markdown` with `remark-gfm`.

### Step 5: Export

Three export options:
- **PDF**: Captures charts via `html2canvas`, generates PDF via `jsPDF` (accessible at `/dark/print-preview`)
- **JSON (current view)**: Filtered and sorted data as currently displayed
- **JSON (all data)**: All options scraped today, fetched from `/api/options/export-today`

---

## Database Setup

### Prerequisites

- PostgreSQL installed and running on localhost:5432

### Create Database and Tables

```sql
-- Create database
CREATE DATABASE mondb_test;

-- Connect to the database
\c mondb_test

-- Create options table
CREATE TABLE db_option (
    id SERIAL PRIMARY KEY,
    symbol VARCHAR NOT NULL,
    quotes VARCHAR,
    expiration_date DATE,
    strike_price NUMERIC,
    bid_price NUMERIC,
    ask_price NUMERIC,
    bid_size NUMERIC,
    ask_size NUMERIC,
    open_price NUMERIC,
    high_price NUMERIC,
    low_price NUMERIC,
    last_close_price NUMERIC,
    net_change NUMERIC,
    settlement_price NUMERIC,
    volatility NUMERIC,
    open_interest NUMERIC,
    nb_trades NUMERIC,
    is_option BOOLEAN,
    is_weekly BOOLEAN,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP,
    scrape_date DATE NOT NULL,
    UNIQUE (symbol, scrape_date)
);

-- Create stock price table
CREATE TABLE db_stock_price (
    id SERIAL PRIMARY KEY,
    symbol VARCHAR NOT NULL,
    last_price NUMERIC,
    bid_price NUMERIC,
    ask_price NUMERIC,
    scrape_date DATE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE (symbol, scrape_date)
);
```

### Scraper Configuration

The scraper connects to PostgreSQL with these defaults (hardcoded in `mx_scraper.py`):

```python
DB_CONFIG = {
    "host": "localhost",
    "port": 5432,
    "database": "mondb_test",
    "user": "mac",
    "password": ""
}
```

Update these values in `mx_scraper.py` if your PostgreSQL setup differs.

---

## Installation & Setup

### Prerequisites

- Node.js 18+
- Python 3.10+
- PostgreSQL running locally

### Installation

```bash
# Frontend dependencies
npm install

# Python scraper dependencies
pip install httpx beautifulsoup4 psycopg2-binary

# Environment
cp .env.example .env.local
# Edit .env.local with your credentials
```

### Environment Variables

| Variable | Required | Purpose |
|----------|----------|---------|
| `POSTGRES_HOST` | Yes | PostgreSQL host (default: localhost) |
| `POSTGRES_PORT` | Yes | PostgreSQL port (default: 5432) |
| `POSTGRES_USER` | Yes | PostgreSQL user |
| `POSTGRES_PASSWORD` | No | PostgreSQL password (empty for local) |
| `POSTGRES_DB` | Yes | Database name (default: mondb_test) |
| `POSTGRES_TABLE` | No | Options table name (default: db_option) |
| `OPENAI_API_KEY` | No | OpenAI API key for AI analysis feature |

### Running

```bash
# 1. Create database and tables (see Database Setup above)

# 2. Scrape data into PostgreSQL
python mx_scraper.py

# 3. Start the dashboard
npm run dev
# Open http://localhost:3000/dark
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Scraper "table db_option n'existe pas" | Run the CREATE TABLE SQL statements above |
| Scraper connection error | Verify PostgreSQL is running on localhost:5432 |
| No symbols in dropdown | Run `mx_scraper.py` first to populate the database |
| Charts empty | Ensure options exist for the selected symbol in db_option |
| AI analysis fails | Check `OPENAI_API_KEY` in .env.local |
| PDF export blank | Access via `/dark/print-preview` for proper chart capture |
| m-x.ca scraping fails | Website structure may have changed — check HTML selectors |
