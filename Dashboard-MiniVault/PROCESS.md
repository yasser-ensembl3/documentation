# Dashboard MiniVault (MetaVault) — Process Documentation

## Overview

Aggregated command-center dashboard that unifies data from all vault apps (Media Vault, Research Vault, Stoic Vault, Tao Promotion) into a single Nextra-powered interface. Pulls live data from 6 Notion databases, GitHub Project V2 board, and uses Claude AI (Haiku 4.5) for media curation. Features a cross-vault digest feed, three-layer caching, and mobile-responsive inline-styled components.

**Repo**: [yasser-ensembl3/dashboard_app](https://github.com/yasser-ensembl3/dashboard_app)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Framework | Next.js 14 (Pages Router) |
| Docs engine | Nextra 2.13 (docs theme, grayscale) |
| Language | TypeScript |
| Styling | Inline styles (dark gray palette, no Tailwind) |
| Data layer | Notion API SDK (@notionhq/client) |
| AI | Anthropic Claude Haiku 4.5 |
| GitHub | GraphQL API (Project V2) |
| Deployment | Vercel |

---

## Architecture

```
Dashboard MiniVault/
├── pages/
│   ├── index.mdx                  # Overview — command center
│   ├── datavault.mdx              # Research — saved articles
│   ├── contentvault.mdx           # Media — content tracker
│   ├── stockvault.mdx             # Stoic — investment data (static)
│   ├── tao-promotion.mdx          # Tao — book promotion KPIs
│   ├── _meta.json                 # Nextra sidebar navigation
│   ├── _app.tsx                   # App wrapper
│   └── api/
│       ├── dashboard.ts           # Aggregated dashboard data
│       ├── contentvault.ts        # ContentVault items
│       ├── datavault.ts           # DataVault/saved articles
│       ├── digest.ts              # Cross-vault activity feed (5 sources)
│       ├── ai/media-insights.ts   # AI-ranked media via Claude Haiku 4.5
│       ├── github/issues.ts       # GitHub Project V2 board items
│       └── tao/
│           ├── status.ts          # KPIs and goals
│           ├── orders.ts          # Orders (filter: unfulfilled)
│           ├── tasks.ts           # Tasks (filter: status)
│           └── feedback.ts        # Reviews
├── components/
│   ├── DashboardData.tsx          # Overview sections (6 components)
│   ├── TaoPromotionData.tsx       # Tao sections (8 components)
│   ├── ContentVaultData.tsx       # Media sections (6 components)
│   ├── DataVaultData.tsx          # Research sections (5 components)
│   ├── VaultCard.tsx              # Generic vault card (legacy)
│   ├── MetricsDisplay.tsx         # Generic metrics grid (legacy)
│   ├── VaultLink.tsx              # External link button
│   ├── LoadingState.tsx           # Loading skeletons + error state
│   └── index.ts                   # Barrel exports
├── hooks/
│   └── useNotionData.ts           # Client fetch with 5-min localStorage cache
├── lib/
│   ├── notion.ts                  # Notion client + 12 property helpers
│   ├── types.ts                   # All TypeScript interfaces
│   └── cache.ts                   # Server-side 30-min cache with fingerprint invalidation
├── data/
│   └── vaults.json                # Static data (Stoic page)
├── styles/
│   └── globals.css                # Nextra overrides + mobile responsive
└── theme.config.tsx               # Nextra theme (grayscale, "MetaVault" logo)
```

---

## Pages

| Route | Sidebar Name | Data Sources | Description |
|-------|-------------|--------------|-------------|
| `/` | Overview | All Notion DBs + GitHub + Claude AI | Command center with action banner, workflow, curated media, goals, to-do, digest |
| `/datavault` | Research | Notion DataVault DB | Saved articles list with authors, subjects, PDF links |
| `/contentvault` | Media | Notion ContentVault DB | Content tracker with metrics by type and source |
| `/stockvault` | Stoic | Static JSON (`data/vaults.json`) | Investment companies and stock analysis (static) |
| `/tao-promotion` | Tao | Notion Tao DBs (4) | Book promotion: unfulfilled orders, pending tasks, goals, reviews |

---

## Overview Page — Command Center

The main dashboard (`index.mdx`) renders 6 sections in priority order:

### 1. Action Required Banner

**Component**: `ActionRequiredBanner`
**API**: `GET /api/tao/orders?filter=unfulfilled`

Red alert banner showing unfulfilled Tao orders that need attention. Hidden when no unfulfilled orders exist.

### 2. Weekly Workflow

**Component**: `WeeklyWorkflow`
**API**: none (client-side)

Placeholder section displaying a visual weekly calendar. Shows "Coming soon — weekly workflow tracking".

### 3. Media + Research

**Component**: `MediaAndResearch`
**API**: `GET /api/ai/media-insights` (primary), `GET /api/contentvault` + `GET /api/datavault` (fallback)

AI-curated content recommendations. Claude Haiku 4.5 ranks ContentVault + DataVault items by relevance to active Tao tasks. Falls back to recency-sorted list if AI is unavailable.

### 4. Goals & Metrics

**Component**: `GoalsAndMetrics`
**API**: `GET /api/dashboard`

Aggregated KPIs from Tao: total orders, total revenue, unfulfilled count, plus goals from Tao Goals DB.

### 5. Actions / To Do

**Component**: `ActionsToDo`
**API**: `GET /api/github/issues`

GitHub Project V2 board items from `GuillaumeRacine/ensemble_prototypes` project #4. Shows issues with status labels.

### 6. Digest

**Component**: `DigestFeed`
**API**: `GET /api/digest`

Cross-vault activity timeline merging 5 sources chronologically:
- Tao Orders (recent orders with customer/total)
- Tao Tasks (recently modified tasks with status)
- ContentVault (recently added media items)
- DataVault (recently added articles)
- GitHub (recent commits)

Uses `Promise.allSettled` so partial failures don't break the feed.

---

## API Routes (10 endpoints)

### Core Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /api/dashboard` | Aggregated data: Tao orders (count, revenue, unfulfilled), goals, ContentVault/DataVault item counts |
| `GET /api/contentvault` | All ContentVault items with type/source breakdown metrics |
| `GET /api/datavault` | All saved articles from DataVault DB |
| `GET /api/digest` | Cross-vault activity feed from 5 sources, merged chronologically |

### AI Endpoint

| Endpoint | Description |
|----------|-------------|
| `GET /api/ai/media-insights` | Claude Haiku 4.5 ranks media items by relevance to active tasks. Falls back to recency sort. |

**AI Flow**:
1. Fetch active tasks from Tao Tasks DB
2. Fetch recent items from ContentVault + DataVault
3. Send to Claude with ranking prompt
4. Return AI-ranked list with relevance scores
5. Graceful fallback to recency-sorted list

### GitHub Endpoint

| Endpoint | Description |
|----------|-------------|
| `GET /api/github/issues` | GitHub Project V2 board items via GraphQL (user: GuillaumeRacine, project #4) |

### Tao Promotion Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /api/tao/status` | KPIs: total orders, revenue, unfulfilled count + goals |
| `GET /api/tao/orders` | Orders list. `?filter=unfulfilled` for unfulfilled only |
| `GET /api/tao/tasks` | Tasks list. `?status=To Do` to filter by status |
| `GET /api/tao/feedback` | Reviews. `?limit=5&recent=true` for recent subset |

All routes enforce `GET` only (return 405 for other methods).

---

## Three-Layer Caching

```
Layer 1: Server in-memory (lib/cache.ts)
├── TTL: 30 minutes
├── Fingerprint-based invalidation
│   (hash of item IDs + last-edited timestamps)
└── Used by: /api/digest, /api/ai/media-insights

Layer 2: CDN / Vercel Edge
├── Cache-Control: s-maxage=300 (5 min)
├── stale-while-revalidate=600 (10 min)
└── Set on all API routes

Layer 3: Client localStorage (hooks/useNotionData.ts)
├── TTL: 5 minutes per endpoint
├── useNotionData<T>(endpoint) — single fetch
└── useMultipleNotionData<T>(endpoints) — parallel fetch
```

---

## Notion Databases (6)

| Database | Env Var | Properties | Used By |
|----------|---------|------------|---------|
| ContentVault | `NOTION_CONTENTVAULT_DB` | Link (title), type (select), Status (select), Date, Audio Link (url), Audio/Text summary (files) | Media, AI, digest |
| DataVault | `NOTION_DATAVAULT_DB` | Title (title), Authors (rich_text), Subject (rich_text), Description (rich_text), Submission (date), pdf Link (url) | Research, AI, digest |
| Tao Orders | `NOTION_TAO_ORDERS_DB` | Order (title), Customer (rich_text), Total $ (number), Date (date), Payment (select), Fulfillment (select) | Tao, dashboard, digest |
| Tao Tasks | `NOTION_TAO_TASKS_DB` | Name (title), Status (status/select/rich_text), Priority (select), Due Date (date) | Tao, AI, digest |
| Tao Feedback | `NOTION_TAO_FEEDBACK_DB` | Title (title), Feedback (rich_text), User Name (rich_text) | Tao reviews |
| Tao Goals | `NOTION_TAO_GOALS_DB` | Metric Name (title), Number (number), Last Updated (date) | Tao, dashboard |

### Notion Property Helpers (`lib/notion.ts`)

12 typed extraction functions: `getTitle`, `getRichText`, `getNumber`, `getSelect`, `getMultiSelect`, `getDate`, `getUrl`, `getFileUrl`, `getPerson`, `getAnyText`, `formatDate`, plus `queryDatabase` wrapper using @notionhq/client SDK.

---

## Component Architecture

### Per-Page Component Files

| File | Components | Page |
|------|-----------|------|
| `DashboardData.tsx` | ActionRequiredBanner, WeeklyWorkflow, MediaAndResearch, GoalsAndMetrics, ActionsToDo, DigestFeed | Overview |
| `TaoPromotionData.tsx` | TaoHeader, TaoGoalsAndMetrics, TaoMetrics, TaoUnfulfilledOrders, TaoPendingTasks, TaoGoals, TaoRecentReviews, TaoLastUpdated | Tao |
| `ContentVaultData.tsx` | ContentVaultHeader, ContentVaultMetrics, ContentVaultBySource, ContentVaultByType, ContentVaultItems, ContentVaultLastUpdated | Media |
| `DataVaultData.tsx` | DataVaultHeader, DataVaultStatus, DataVaultItems, DataVaultAbout, DataVaultLastUpdated | Research |

### Shared Components

| Component | Purpose |
|-----------|---------|
| `VaultLink` | External link button with SVG icon |
| `LoadingState` | 4 skeleton types (spinner, skeleton, card, metrics) + error state with retry |
| `VaultCard` | Generic vault card with metrics (legacy, unused) |
| `MetricsDisplay` | Generic metrics grid (legacy, unused) |

### Styling Pattern

All components use **inline `style={{}}` objects** — no Tailwind or CSS modules. Dark gray palette:
- Backgrounds: `#1f2937`, `#374151`, `#4b5563`
- Text: `#6b7280`, `#9ca3af`, `#d1d5db`, `#f9fafb`
- Hover via `onMouseEnter`/`onMouseLeave` changing inline colors

---

## External Vault Links

Each page links to its corresponding standalone app:

| Page | Standalone App |
|------|---------------|
| Research | datavault-rust.vercel.app |
| Media | media-minivault.vercel.app |
| Stoic | stock-vault.vercel.app |
| Tao | tao-promotion.vercel.app |

---

## Environment Variables

```env
# Notion
NOTION_API_KEY=ntn_xxx
NOTION_CONTENTVAULT_DB=xxx
NOTION_DATAVAULT_DB=xxx
NOTION_TAO_ORDERS_DB=xxx
NOTION_TAO_TASKS_DB=xxx
NOTION_TAO_FEEDBACK_DB=xxx
NOTION_TAO_GOALS_DB=xxx

# AI
ANTHROPIC_API_KEY=sk-ant-xxx

# GitHub
GITHUB_TOKEN=ghp_xxx
```

---

## Setup

### Prerequisites

- Node.js 18+
- Notion integration with access to all 6 databases
- Anthropic API key (for AI media curation)
- GitHub PAT with `read:project` scope

### Installation

```bash
npm install
# Create .env.local with all variables above
npm run dev
# Open http://localhost:3000
```

### Deployment (Vercel)

1. Push to GitHub
2. Import in Vercel
3. Add all environment variables
4. Deploy (framework auto-detected as Next.js)

---

## Credentials

| Service | Type | Purpose |
|---------|------|---------|
| Notion | Internal integration token | Read all 6 databases |
| Anthropic | API key | Claude Haiku 4.5 for media curation |
| GitHub | Personal access token | Project V2 board + commits |

---

## Evolution

17 commits tracking the project's transformation:

1. **Initial dashboard** with 3 apps, then 4
2. **Nextra migration** — converted to docs theme with real vault data
3. **Compact refactor** — grayscale theme, live Notion data, CDN caching
4. **MetaVault rename** — from MiniVault to MetaVault
5. **Command center** — added workflow, media feed, and digest
6. **Page rename** — datavault→Research, contentvault→Media, stockvault→Stoic
7. **AI integration** — Claude Haiku 4.5 for media curation
8. **Digest feed** — cross-vault activity timeline from 5 sources
9. **Mobile responsive** — responsive CSS, card overflow fixes
10. **Security** — CVE-2026-0969 fix (next-mdx-remote override)

---

## Design Decisions

1. **Nextra docs theme** — sidebar navigation, search, grayscale palette, MDX pages with embedded React components
2. **Inline styles over Tailwind** — all components use `style={{}}` objects for dark theme consistency
3. **Three-layer caching** — server (30 min) + CDN (5 min) + client (5 min) for fast loads without hammering Notion
4. **Fingerprint invalidation** — server cache hashes item IDs + timestamps to detect Notion changes
5. **AI graceful degradation** — media insights fall back to recency sort if Claude is unavailable
6. **Promise.allSettled** — digest feed survives partial source failures
7. **Notion type polymorphism** — Status handled as `status`, `select`, or `rich_text` property types
8. **All GET only** — API routes enforce GET, return 405 for other methods

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Empty dashboard | Check all `NOTION_*` env vars in `.env.local` |
| AI insights showing fallback | Verify `ANTHROPIC_API_KEY` is set and has credits |
| GitHub section empty | Check `GITHUB_TOKEN` has `read:project` scope |
| Stale data | Clear localStorage (`clearCache()`) or wait for CDN TTL (5 min) |
| Unfulfilled orders not showing | Verify Tao Orders DB has "Fulfillment" select property |
| Digest feed partial | Expected — `Promise.allSettled` allows partial failures |
