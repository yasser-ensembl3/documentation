# Stoic Vault — Process Documentation

## Overview

Financial insights dashboard for analyzing quarterly reports from public companies. Provides company comparison, quarterly trends, AI-powered chat, stock data pipeline, and investment reports. Uses Google Drive as the primary data store, OpenAI for chat, and Alpha Vantage for stock data.

**Repo**: [yasser-ensembl3/StockVault](https://github.com/yasser-ensembl3/StockVault)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Framework | Next.js 14, React 18 |
| Language | TypeScript |
| Styling | Tailwind CSS 4, Radix UI, shadcn/ui |
| Charts | Recharts |
| Auth | NextAuth.js (Google + GitHub OAuth) |
| AI | OpenAI API (gpt-4o-mini) |
| Data storage | Google Drive API v3 |
| Stock data | Alpha Vantage API |
| Docs | Nextra |
| Caching | In-memory (60-second TTL) |

---

## Architecture

```
quarterly-vault/
├── app/
│   ├── layout.tsx                      # Root layout
│   ├── page.tsx                        # Home (redirects to /compare)
│   ├── pages/
│   │   ├── dashboard/page.tsx          # Main dashboard
│   │   ├── compare/page.tsx            # Company comparison (2-4)
│   │   ├── trends/page.tsx             # Single company trends
│   │   ├── chat/page.tsx               # AI financial assistant
│   │   ├── company/[name]/page.tsx     # Company detail (7 tabs)
│   │   ├── investment/[name]/page.tsx  # Investment report detail
│   │   ├── stock/page.tsx              # Stock search & fetch
│   │   └── stock/[symbol]/page.tsx     # Stock detail
│   └── api/
│       ├── chat/route.ts               # OpenAI chat endpoint
│       ├── drive/                      # Google Drive operations
│       ├── investment/                 # Investment report routes
│       └── stock/                      # Stock data routes
├── components/
│   ├── sidebar.tsx                     # Collapsible sidebar navigation
│   ├── company-card.tsx                # Company display card
│   ├── quarter-selector.tsx            # Quarter dropdown
│   ├── layout-wrapper.tsx              # Responsive layout with sidebar
│   ├── dashboard/                      # Dashboard-specific components
│   ├── stock/                          # Stock components
│   └── ui/                            # shadcn/ui components
├── lib/
│   ├── google-drive.ts                # Google Drive API client
│   ├── cache.ts                       # In-memory cache (60s TTL)
│   ├── auth.ts                        # NextAuth config
│   ├── alpha-vantage.ts               # Alpha Vantage API client
│   ├── investment-reports.ts          # Investment report parsing
│   ├── project-config.ts             # Config from env
│   └── utils.ts                       # Helpers
├── content/docs/                      # Nextra MDX documentation
└── types/next-auth.d.ts               # NextAuth type augmentation
```

---

## Features — How They Work

### 1. Dashboard (`/dashboard`)

Overview of selected quarter with all tracked companies.

- Quarter selector dropdown loads companies from Google Drive
- Statistics cards: total companies, financial data available, strategic data available
- Company search/filter
- Grid of company cards linking to detail pages

### 2. Company Comparison (`/compare`)

Compare 2-4 companies side-by-side with visualizations.

- Bar charts: revenue, net income, growth rates (Recharts)
- Radar chart for multi-dimensional analysis
- 14 financial metrics comparison table
- Color-coded company indicators
- Data fetched in parallel via `Promise.all()`

### 3. Quarterly Trends (`/trends`)

Track a single company across multiple quarters.

- Line charts: revenue & net income trends
- Bar charts: EPS trend, YoY growth rates
- Quarter-over-quarter comparison table
- Chronological quarter sorting

### 4. Company Detail (`/company/[name]`)

7-tab deep dive into a single company:

| Tab | Content |
|-----|---------|
| Overview | Executive summary, key metrics, key takeaways |
| Financial | Income statement, balance sheet, cash flow, revenue breakdown (by segment + geography) |
| Strategic | Initiatives, management commentary, partnerships, M&A |
| Products | R&D focus, new launches, pipeline |
| Competitive | Market position, advantages, threats, industry trends |
| Risks | Risk factors with severity badges and mitigation strategies |
| Investor | Bull/bear cases, catalysts, key questions, ESG data |

Data comes from two JSON files per company on Google Drive:
- `{Company}_financial.json` — income statement, balance sheet, cash flow, revenue breakdown, guidance
- `{Company}_strategic.json` — executive summary, initiatives, commentary, competitive landscape, risks, investor highlights

### 5. AI Chat (`/chat`)

Natural language financial queries powered by GPT-4o-mini.

**Flow:**
1. User types a question
2. POST to `/api/chat` with message history (last 6 messages retained)
3. API extracts context — companies and quarters mentioned in the message
4. Fetches relevant financial + strategic data from Google Drive
5. Builds a system prompt with extracted key metrics and strategic highlights
6. Sends to OpenAI (gpt-4o-mini)
7. Returns markdown response with context badges showing which companies/quarters were referenced

Features: suggested questions, real-time markdown rendering, typing indicator.

### 6. Stock Data (`/stock`)

Pipeline for fetching and storing stock data.

**Flow:**
1. Search stock symbols via Alpha Vantage API
2. Fetch historical data: overview, income statement, balance sheet, cash flow
3. Upload JSON files to Google Drive under `Stock Data/{SYMBOL}/`
4. View results with success/error counts

**Alpha Vantage rate limit:** 5 calls/min on free tier.

### 7. Investment Reports (`/investment/[name]`)

Detailed investment analysis pages parsed from Google Drive markdown documents.

---

## Data Flow

```
Google Drive (folder structure)
├── Root Folder/
│   ├── Q3 2024/
│   │   ├── Amazon/Insights/
│   │   │   ├── Amazon_financial.json
│   │   │   └── Amazon_strategic.json
│   │   ├── Coinbase/Insights/...
│   │   └── ...
│   ├── Q2 2024/...
│   └── Stock Data/
│       ├── AAPL/
│       │   ├── overview.json
│       │   ├── income_statement.json
│       │   ├── balance_sheet.json
│       │   ├── cash_flow.json
│       │   └── _meta.json
│       └── ...
         ↓
    Google Drive API Client (lib/google-drive.ts)
         ↓
    In-Memory Cache (60s TTL)
         ↓
    Next.js API Routes
         ↓
    React Components
```

### Caching

- All Drive API calls cached for 60 seconds via `withCache(key, fetcher)` wrapper
- Cache keys: `quarters`, `companies:{quarterId}`, `insights:{quarterId}:{company}:{type}`
- Automatic expiration, console logging for hits/misses

### Fallback

When Google Drive is unavailable, the app falls back to mock data with sample companies (Amazon, Coinbase, Shopify, NVIDIA, eBay, Etsy, LVMH, Circle) so the UI remains functional.

---

## API Routes Reference

| Route | Method | Description |
|-------|--------|-------------|
| `/api/drive/quarters` | GET | List all available quarters |
| `/api/drive/companies` | GET | List companies in a quarter |
| `/api/drive/insights` | GET | Fetch financial or strategic JSON |
| `/api/drive/files` | GET | List files in a Drive folder (auth required) |
| `/api/chat` | POST | AI chat with financial context |
| `/api/investment/companies` | GET | List investment companies |
| `/api/investment/report` | GET | Get parsed investment report |
| `/api/stock/search` | GET | Search stock symbols (Alpha Vantage) |
| `/api/stock/pipeline` | POST | Fetch stock data + upload to Drive |
| `/api/stock/symbols` | GET | List cached stock symbols |

---

## Data Structures

### Financial JSON

```
company_info: { name, ticker, quarter, year, report_date }
income_statement: { revenue, operating_income, net_income, eps, ebitda }
balance_sheet: { total_cash, total_debt, total_assets, shareholders_equity }
cash_flow: { operating, investing, financing, free_cash_flow }
revenue_breakdown: { by_segment: [...], by_geography: [...] }
guidance: { next_quarter: { metrics: [...] } }
```

### Strategic JSON

```
executive_summary: { one_liner, management_tone, key_narrative }
key_takeaways: [...]
strategic_initiatives: [{ initiative, status, progress, expected_impact }]
management_commentary: { ceo_message, cfo_message, priorities }
competitive_landscape: { market_position, advantages, threats, trends }
risks_and_challenges: [{ risk, category, severity, management_response }]
investor_highlights: { bull_case, bear_case, catalysts, key_questions }
esg_and_sustainability: { environmental, social, governance }
```

---

## Installation & Setup

### Prerequisites

- Node.js 18+
- Google Cloud project with Drive API enabled
- OpenAI API key
- Alpha Vantage API key (free tier available)

### Installation

```bash
cd quarterly-vault
npm install
cp .env.example .env.local
# Edit .env.local with your credentials
npm run dev    # Starts on localhost:3000
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `GOOGLE_CLIENT_ID` | Google OAuth client ID |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret |
| `GOOGLE_REFRESH_TOKEN` | Google OAuth refresh token |
| `GDRIVE_ROOT_FOLDER_ID` | Root folder ID for quarterly data |
| `GDRIVE_STOCK_FOLDER_ID` | Folder ID for stock data (optional) |
| `OPENAI_API_KEY` | OpenAI API key (gpt-4o-mini) |
| `ALPHA_VANTAGE_KEY` | Alpha Vantage API key |
| `NEXTAUTH_SECRET` | NextAuth session secret |
| `NEXTAUTH_URL` | App URL (default: http://localhost:3000) |
| `NEXT_PUBLIC_PROJECT_NAME` | Display name (default: Stoic Vault) |

### Google Drive Setup

1. Google Cloud Console → create or select a project
2. Enable Google Drive API
3. Create OAuth 2.0 credentials
4. Get a refresh token via OAuth playground or manual flow
5. Create the folder structure on Drive (Root → Quarter folders → Company folders → Insights/)

### Populating Data

The financial and strategic JSON files in Google Drive need to be created manually or via an external pipeline. Each company's quarter folder should contain an `Insights/` subfolder with:
- `{CompanyName}_financial.json`
- `{CompanyName}_strategic.json`

---

## Deployment

Optimized for Vercel (Next.js native):

```bash
npm run build
npm start
```

Add all environment variables to the Vercel dashboard.

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Drive API 403 | Check OAuth credentials and refresh token validity |
| No companies showing | Verify `GDRIVE_ROOT_FOLDER_ID` points to correct folder |
| Chat returns generic answers | Ensure financial/strategic JSON files exist for referenced companies |
| Alpha Vantage rate limit | Free tier allows 5 calls/min — wait or upgrade |
| Auth redirect loop | Verify `NEXTAUTH_URL` matches your deployment URL |
| Mock data showing | Drive is unavailable or misconfigured — check env vars |
