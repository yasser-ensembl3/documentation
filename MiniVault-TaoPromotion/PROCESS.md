# MiniVault (Tao Promotion) — Process Documentation

## Overview

Unified project management dashboard integrating Notion, Google Drive, Gmail, GitHub, and Shopify into a single interface. Built with Next.js 14. Follows a **one repo = one project** model — each deployment is a single project configured entirely via environment variables.

**Repo**: [yasser-ensembl3/Tao-promotion](https://github.com/yasser-ensembl3/Tao-promotion)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Framework | Next.js 14 (App Router), React 18 |
| Language | TypeScript |
| Styling | Tailwind CSS 4 (dark mode only), Radix UI, shadcn/ui |
| Auth | NextAuth.js (Google + GitHub OAuth) |
| Data fetching | SWR 2.3.8 (60-second cache) |
| Charts | Recharts |
| Database | Notion API (v2022-06-28) |
| MCP server | Model Context Protocol for Claude integration |
| Docs | Nextra |
| Deployment | Vercel |

---

## Architecture

```
MiniVault-TaoPromotion/
├── app/
│   ├── layout.tsx                          # Root layout with providers
│   ├── page.tsx                            # Home (redirects to /dashboard)
│   ├── auth/signin/page.tsx                # OAuth sign-in
│   ├── dashboard/page.tsx                  # Main dashboard
│   ├── tasks/page.tsx                      # Tasks page
│   ├── recurring-tasks/page.tsx            # Recurring tasks
│   ├── orders/page.tsx                     # Orders
│   ├── overview/page.tsx                   # Project overview
│   ├── essentials/page.tsx                 # Essentials
│   ├── guides/page.tsx                     # Guides & docs
│   ├── feedback/page.tsx                   # Feedback
│   ├── reports/page.tsx                    # Reports
│   ├── analytics/page.tsx                  # Analytics
│   ├── docs/[[...mdxPath]]/               # Nextra docs
│   └── api/
│       ├── auth/[...nextauth]/             # NextAuth handler
│       ├── notion/                         # 13+ Notion database routes
│       ├── github/repo/                    # GitHub proxy
│       ├── drive/files/                    # Google Drive proxy
│       ├── google/gmail/                   # Gmail
│       ├── ai/chat/                        # AI chat
│       └── feedback/save/                  # Feedback file save
├── components/
│   ├── sidebar.tsx                         # Navigation (desktop + mobile)
│   ├── dashboard/
│   │   ├── main-dashboard.tsx              # Master orchestrator
│   │   ├── dashboard-section.tsx           # Reusable collapsible wrapper
│   │   └── [12 feature sections]           # One component per section
│   └── ui/                                # shadcn/ui components
├── lib/
│   ├── auth.ts                            # NextAuth config
│   ├── project-config.ts                  # Env-based project config
│   ├── use-cached-fetch.ts                # SWR hooks
│   ├── swr-config.tsx                     # SWR provider (60s cache)
│   ├── api-auth.ts                        # API auth utilities
│   └── utils.ts                           # Helpers
├── mcp-server/                            # MCP server for Claude
├── content/docs/                          # Nextra MDX docs
└── types/next-auth.d.ts                   # NextAuth type augmentation
```

---

## How It Works

### One Repo = One Project

Each MiniVault deployment is dedicated to a single project. No project switching in the UI. All configuration (Notion databases, GitHub repo, Drive folder) is set via `.env.local`. To manage a different project, fork/clone the repo and configure a new `.env.local`.

### Dashboard — Modular Sections

The dashboard is composed of independent, collapsible sections. Each section:
- Shows key metrics when collapsed
- Reveals detailed content when expanded
- Is independently connected to its Notion database
- Can be enabled/disabled by setting or omitting the corresponding `NEXT_PUBLIC_NOTION_DB_*` env var

**Section order (priority):**

| # | Section | Data Source | Purpose |
|---|---------|------------|---------|
| 1 | Orders | Notion | Order fulfillment tracking |
| 2 | Goals | Notion | Output metrics (sales, subscribers, reviews) with charts |
| 3 | Sales Tracking | Notion | Revenue and sales analytics |
| 4 | Web Analytics | Notion | Traffic and conversion data |
| 5 | Metrics | Notion | Input metrics (posts, interactions) with charts |
| 6 | Essentials | Notion | Critical tools, milestones, resources, partnerships |
| 7 | Guides & Docs | Notion | Documentation links in card grid |
| 8 | Overview | Notion | Project description, vision, milestones |
| 9 | Recurring Tasks | Notion | Repeating task tracking |
| 10 | Tasks | Notion | One-time tasks (Kanban board) |
| 11 | Reports | Local | AI-generated weekly summaries |
| 12 | User Feedback | Notion | Feedback collection and display |

### Metrics System

Dual approach with interactive charts (Recharts):

**Goals (Outputs)** — Track results:
- Examples: # of sales, # of subscribers, # of reviews
- Color scheme: green
- Clickable metric cards switch chart display

**Metrics (Inputs)** — Track actions:
- Examples: # of posts, # of interactions, marketing ROI
- Color scheme: blue
- Same card-to-chart interaction pattern

Both use date-based tracking with normalized metric names (lowercase, no accents).

### Authentication Flow

1. User visits protected page → redirected to `/auth/signin`
2. Signs in with Google or GitHub OAuth
3. NextAuth exchanges code for access/refresh tokens
4. Tokens stored in JWT (server-side)
5. Session callback exposes tokens to client
6. API routes use `getServerSession(authOptions)` to retrieve tokens for authenticated requests

**Google OAuth scopes:** openid, email, profile, gmail.readonly, drive.readonly, documents.readonly
**GitHub OAuth scopes:** read:user, user:email, repo

### Data Fetching

SWR (Stale-While-Revalidate) with 60-second cache:
- `useCachedFetch<T>(url)` — generic data fetching
- `useNotionData<T>(endpoint, databaseId)` — Notion-specific
- `useGitHubData<T>(owner, repo)` — GitHub-specific
- `useDriveData<T>(folderId)` — Drive-specific

Error retry: 2 attempts with 5-second intervals. Keeps previous data while fetching.

---

## API Routes Reference

### Notion CRUD

All Notion routes use `NOTION_TOKEN` from env. Support GET (fetch), POST (create), PATCH (update), DELETE (remove).

| Route | Purpose |
|-------|---------|
| `/api/notion/metrics` | Input metrics |
| `/api/notion/goals` | Output metrics |
| `/api/notion/tasks` | One-time tasks |
| `/api/notion/recurring-tasks` | Recurring tasks |
| `/api/notion/orders` | Orders |
| `/api/notion/documents` | Documentation links |
| `/api/notion/feedback` | User feedback |
| `/api/notion/essentials` | Tools, milestones, resources |
| `/api/notion/sales` | Sales data |
| `/api/notion/milestones` | Project milestones |
| `/api/notion/project-overview` | Project description |
| `/api/notion/shopify` | Shopify integration |
| `/api/notion/page-content` | Page content fetch |

### External Services

| Route | Auth | Purpose |
|-------|------|---------|
| `/api/github/repo` | GitHub OAuth | Repo metadata, commits, issues, PRs |
| `/api/drive/files` | Google OAuth | Drive folder contents |
| `/api/google/gmail` | Google OAuth | Gmail messages |
| `/api/ai/chat` | — | AI chat functionality |

---

## MCP Server

Located in `mcp-server/`. Model Context Protocol server enabling Claude to interact with all Notion databases directly.

**Dependencies:** @anthropic-ai/sdk, @modelcontextprotocol/sdk, @notionhq/client

**Transport:** stdio

---

## Installation & Setup

### Prerequisites

- Node.js 18+
- Notion workspace with API integration
- Google Cloud project (OAuth + Drive API)
- GitHub OAuth app

### Installation

```bash
cd MiniVault-TaoPromotion
npm install
cp .env.example .env.local
# Edit .env.local with credentials
npm run dev    # localhost:3000
```

### Environment Variables

**Authentication (required):**

| Variable | Description |
|----------|-------------|
| `NEXTAUTH_URL` | App URL |
| `NEXTAUTH_SECRET` | Session secret |
| `GOOGLE_CLIENT_ID` | Google OAuth client ID |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret |
| `GITHUB_ID` | GitHub OAuth app ID |
| `GITHUB_SECRET` | GitHub OAuth app secret |
| `NOTION_TOKEN` | Notion integration token |

**Project config:**

| Variable | Description |
|----------|-------------|
| `NEXT_PUBLIC_PROJECT_NAME` | Display name |
| `NEXT_PUBLIC_PROJECT_DESCRIPTION` | Project description |
| `NEXT_PUBLIC_GITHUB_OWNER` | GitHub owner |
| `NEXT_PUBLIC_GITHUB_REPO` | GitHub repo name |
| `NEXT_PUBLIC_GOOGLE_DRIVE_FOLDER_ID` | Drive folder ID |

**Notion database IDs (enable/disable sections):**

| Variable | Section |
|----------|---------|
| `NEXT_PUBLIC_NOTION_DB_TASKS` | Tasks |
| `NEXT_PUBLIC_NOTION_DB_RECURRING_TASKS` | Recurring tasks |
| `NEXT_PUBLIC_NOTION_DB_GOALS` | Goals |
| `NEXT_PUBLIC_NOTION_DB_METRICS` | Metrics |
| `NEXT_PUBLIC_NOTION_DB_MILESTONES` | Milestones |
| `NEXT_PUBLIC_NOTION_DB_DOCUMENTS` | Docs |
| `NEXT_PUBLIC_NOTION_DB_FEEDBACK` | Feedback |
| `NEXT_PUBLIC_NOTION_DB_ORDERS` | Orders |
| `NEXT_PUBLIC_NOTION_DB_ESSENTIALS` | Essentials |
| `NEXT_PUBLIC_NOTION_DB_SALES` | Sales |
| `NEXT_PUBLIC_NOTION_DB_SALES_TRACKING` | Sales tracking |
| `NEXT_PUBLIC_NOTION_DB_WEB_ANALYTICS` | Web analytics |

### Notion Setup

1. Create a Notion integration at [notion.so/my-integrations](https://www.notion.so/my-integrations)
2. Grant read/write permissions
3. Create databases for each section you want to enable
4. Share each database with the integration
5. Copy database IDs → set as `NEXT_PUBLIC_NOTION_DB_*` in `.env.local`

### Deployment (Vercel)

```bash
npm run build
npm start
```

Add all environment variables to Vercel dashboard. Uses `vercel.json` for config.

---

## Key Design Patterns

### DashboardSection Component
Reusable collapsible card wrapper. Props: title, description, icon, keyMetrics (collapsed view), detailedContent (expanded view). All sections use this wrapper.

### Flexible Notion Property Detection
API routes support multiple naming conventions for Notion properties. Smart type detection handles title, rich_text, number, select, multi_select, date, and status properties automatically.

### Mobile-First Responsive
All components use Tailwind responsive prefixes. Adaptive padding, text sizing, and grid layouts. Sidebar collapses to hamburger menu on mobile.

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| OAuth 403 on API calls | Sign out and sign back in to re-trigger consent with updated scopes |
| Notion returns empty | Check `NOTION_TOKEN` and ensure database is shared with integration |
| Section not showing | Verify the corresponding `NEXT_PUBLIC_NOTION_DB_*` env var is set |
| GitHub section empty | Must be signed in with GitHub (not Google) |
| Drive section empty | Must be signed in with Google (not GitHub), verify folder ID |
| Build fails | Run `npm run lint`, check env vars are single-line formatted |
| Metrics chart empty | Check date format in Notion database, ensure entries have date values |
