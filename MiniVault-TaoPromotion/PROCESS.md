# MiniVault (Tao Promotion) — Process Documentation

## Overview

E-commerce dashboard for Shopify store management. Consolidates orders, sales analytics, web traffic, customer feedback, and operational tasks into a single interface. All data lives in Notion databases. Built with Next.js 14.

Follows a **one repo = one store** model — each deployment manages a single Shopify store configured via environment variables.

**Repo**: [yasser-ensembl3/Tao-promotion](https://github.com/yasser-ensembl3/Tao-promotion)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Framework | Next.js 14 (App Router), React 18 |
| Language | TypeScript |
| Styling | Tailwind CSS 4 (dark mode only), shadcn/ui, Radix UI |
| Auth | NextAuth.js (Google + GitHub OAuth) |
| Data fetching | SWR (60-second cache) |
| Charts | Recharts (line, bar, pie) |
| Database | Notion API |
| Deployment | Vercel |

---

## Architecture

```
MiniVault-TaoPromotion/
├── app/
│   ├── layout.tsx                      # Root layout (SessionProvider, SWR)
│   ├── page.tsx                        # Redirects to /dashboard
│   ├── dashboard/page.tsx              # Main dashboard (all sections)
│   ├── orders/page.tsx                 # Orders with filters
│   ├── analytics/page.tsx              # Goals + Metrics + Sales + Web Analytics
│   ├── tasks/page.tsx                  # Kanban task board
│   ├── recurring-tasks/page.tsx        # Recurring task tracking
│   ├── feedback/page.tsx               # Feedback with time filters
│   ├── essentials/page.tsx             # Tools & resources
│   ├── guides/page.tsx                 # Documentation links
│   ├── overview/page.tsx               # Project description
│   ├── reports/page.tsx                # AI weekly reports
│   ├── auth/signin/page.tsx            # OAuth sign-in
│   └── api/notion/                     # All Notion CRUD routes
├── components/
│   ├── sidebar.tsx                     # Navigation (desktop + mobile)
│   ├── dashboard/
│   │   ├── main-dashboard.tsx          # Section orchestrator
│   │   ├── dashboard-section.tsx       # Reusable collapsible wrapper
│   │   └── [12 feature sections]       # One component per section
│   └── ui/                            # shadcn/ui components
├── lib/
│   ├── auth.ts                        # NextAuth config
│   ├── project-config.ts              # Env-based config
│   ├── use-cached-fetch.ts            # SWR hooks
│   ├── swr-config.tsx                 # SWR provider (60s cache)
│   └── utils.ts                       # Helpers
└── mcp-server/                        # MCP server for Claude integration
```

---

## Dashboard Sections — How They Work

The dashboard is modular. Each section is collapsible (key metrics when collapsed, full detail when expanded). Sections are enabled/disabled by setting the corresponding `NEXT_PUBLIC_NOTION_DB_*` env var.

### 1. Orders

**Route**: `/api/notion/sales`

Displays 4 metric cards:
- Unfulfilled count
- Total orders
- Total revenue
- Refunded count

Table with: Order ID, Date, Items, Total $, Payment Status (Paid/Pending/Refunded), Fulfillment Status (Fulfilled/Unfulfilled). Calculates unfulfilled revenue and fulfillment rate.

### 2. Goals (Output Metrics)

**Route**: `/api/notion/metrics`

Tracks results: sales count, subscribers, Amazon reviews, etc. UI pattern: clickable metric cards → line chart with date range → recent entries table. Can create new entries via dialog form.

### 3. Sales Tracking

**Route**: `/api/notion/shopify?type=sales`

5 metric cards: Total Sales, Net Sales, Paid Orders, Average Order Value, Returning Customer Rate. Breakdown grid: Gross Sales, Discounts, Returns, Shipping+Tax. Line chart with period-based trend.

Period format: "Jan 1-31 2025" — parsed and sorted chronologically.

### 4. Web Analytics

**Route**: `/api/notion/shopify?type=analytics`

4 metric cards: Sessions, Conversion Rate, Add to Cart Rate, Checkout Rate.

Visualizations:
- **Traffic Sources Pie Chart**: Direct, Google, Facebook, Twitter, LinkedIn, Other
- **Device Breakdown Bar Chart**: Desktop vs Mobile
- **Conversion Funnel**: Sessions → Add to Cart % → Checkout % → Conversion %
- **Recent Periods Table**: Last 5 periods

### 5. Metrics (Input Metrics)

**Route**: `/api/notion/metrics`

Same structure as Goals but tracks actions/efforts: posts, interactions, marketing ROI. Blue color scheme vs green for Goals.

### 6. Essentials

**Route**: `/api/notion/essentials`

Grid of cards organized by type (Tool, Milestone, Strategy, Resource, Partnership, Achievement). Color-coded by priority (Critical, High, Medium). CRUD operations.

### 7. Tasks

**Route**: `/api/notion/tasks`

Kanban-style board with status flow: To Do → In Progress → Review → Done. Cards show status, priority, assignee, tags, due date. Sorted: In Progress first, then To Do, then by due date.

### 8. Recurring Tasks

**Route**: `/api/notion/recurring-tasks`

Tracked by frequency: Daily, Weekly, Monthly, Quarterly. Shows next scheduled run and assignee.

### 9. Feedback

**Route**: `/api/notion/feedback`

Customer/user feedback with CRUD. Filter cards: Total, This Week, This Month. Compact when empty (single-line with "Add Feedback" button). Full card list when populated.

### 10. Guides & Docs

**Route**: `/api/notion/documents`

Card grid of documentation links with type badges (Notion, Google Drive, GitHub, etc.). CRUD operations.

---

## Data Flow

```
Client Components
    → useProjectConfig() gets database IDs from env vars
    → useNotionData(endpoint, databaseId) via SWR hooks
    → /api/notion/* routes query Notion API with NOTION_TOKEN
    → Parse Notion response → formatted JSON
    → Cached 60 seconds by SWR
    → Components render with data
```

---

## Notion Database Schemas

### Orders

| Property | Type | Values |
|----------|------|--------|
| Order | Text | Order ID |
| Date | Date | Order date |
| Items | Number | Item count |
| Total $ | Number | Order total |
| Payment | Select | Paid, Pending, Refunded |
| Fulfillment | Select | Fulfilled, Unfulfilled |
| Customer | Text | Customer name |

### Sales Tracking

| Property | Type |
|----------|------|
| Period | Text ("Jan 1-31 2025" format) |
| Gross Sales, Net Sales, Total Sales | Number |
| Discounts, Returns, Taxes, Shipping | Number |
| Paid Orders, Orders Fulfilled, Average Order Value | Number |

### Web Analytics

| Property | Type |
|----------|------|
| Period | Text |
| Sessions | Number |
| Conversion Rate, Add to Cart Rate, Checkout Rate | Number |
| Direct, Google, Facebook, Twitter, LinkedIn, Other | Number (traffic sources) |
| Desktop, Mobile | Number (device split) |

### Goals / Metrics

| Property | Type |
|----------|------|
| Metric Name | Title |
| Number | Number |
| Last Updated | Date |

### Tasks

| Property | Type | Values |
|----------|------|--------|
| Name | Title | Task name |
| Status | Select | To Do, In Progress, Review, Done |
| Priority | Select | Optional |
| Assignee | Text | Optional |
| Tags | Multi-select | Optional |
| Due Date | Date | Optional |

### Essentials

| Property | Type | Values |
|----------|------|--------|
| Title | Title | Item name |
| Description | Text | Optional |
| Type | Select | Tool, Milestone, Strategy, Resource, Partnership, Achievement |
| Priority | Select | Critical, High, Medium |
| URL | URL | Optional link |

---

## Authentication

NextAuth.js with Google and GitHub OAuth. JWT-based token storage.

**Google scopes**: openid, email, profile, gmail.readonly, drive.readonly, documents.readonly
**GitHub scopes**: read:user, user:email, repo

Flow: OAuth provider → NextAuth token exchange → JWT storage → session exposes tokens to client → API routes use tokens for authenticated requests.

---

## Installation & Setup

### Prerequisites

- Node.js 18+
- Notion workspace with API integration
- Google Cloud project (OAuth credentials)

### Installation

```bash
npm install
cp .env.example .env.local
# Edit .env.local
npm run dev    # localhost:3000
```

### Environment Variables

**Auth (required):**

| Variable | Description |
|----------|-------------|
| `NEXTAUTH_URL` | App URL |
| `NEXTAUTH_SECRET` | Session secret |
| `GOOGLE_CLIENT_ID` / `SECRET` | Google OAuth |
| `GITHUB_ID` / `SECRET` | GitHub OAuth (optional) |
| `NOTION_TOKEN` | Notion integration token |

**Notion Databases (set to enable section):**

| Variable | Section |
|----------|---------|
| `NEXT_PUBLIC_NOTION_DB_ORDERS` | Orders |
| `NEXT_PUBLIC_NOTION_DB_GOALS` | Goals |
| `NEXT_PUBLIC_NOTION_DB_SALES_TRACKING` | Sales tracking |
| `NEXT_PUBLIC_NOTION_DB_WEB_ANALYTICS` | Web analytics |
| `NEXT_PUBLIC_NOTION_DB_METRICS` | Metrics |
| `NEXT_PUBLIC_NOTION_DB_ESSENTIALS` | Essentials |
| `NEXT_PUBLIC_NOTION_DB_TASKS` | Tasks |
| `NEXT_PUBLIC_NOTION_DB_RECURRING_TASKS` | Recurring tasks |
| `NEXT_PUBLIC_NOTION_DB_FEEDBACK` | Feedback |
| `NEXT_PUBLIC_NOTION_DB_DOCUMENTS` | Guides & docs |

### Notion Setup

1. Create integration at [notion.so/my-integrations](https://www.notion.so/my-integrations)
2. Grant read/write access
3. Create databases matching the schemas above
4. Share each database with the integration
5. Copy database IDs → set in `.env.local`

### Deployment (Vercel)

Add all env vars to Vercel dashboard. Build and deploy:

```bash
npm run build && npm start
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Section not showing | Set the corresponding `NEXT_PUBLIC_NOTION_DB_*` env var |
| Notion returns empty | Check `NOTION_TOKEN` and ensure database is shared with integration |
| OAuth 403 | Sign out and re-authenticate to refresh scopes |
| Charts empty | Ensure Notion entries have proper date values in "Last Updated" |
| Sales periods not sorting | Use "Jan 1-31 2025" format in Notion Period property |
| Build fails | Run `npm run lint`, check env vars are single-line |
