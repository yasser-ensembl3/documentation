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
| Styling | Inline styles (`style={{}}`) + styled-jsx + globals.css (no Tailwind) |
| Data layer | Notion SDK (@notionhq/client ^2.2.15) |
| AI | Anthropic SDK (@anthropic-ai/sdk ^0.74.0) — Claude Haiku 4.5 |
| GitHub | GraphQL API (Project V2) |
| Deployment | Vercel |

---

## Architecture

```
Dashboard MiniVault/
├── pages/
│   ├── index.mdx                  # Overview — command center (6 sections)
│   ├── datavault.mdx              # Research — saved articles
│   ├── contentvault.mdx           # Media — content tracker
│   ├── stockvault.mdx             # Stoic — investment data (fully static)
│   ├── tao-promotion.mdx          # Tao — book promotion KPIs
│   ├── _meta.json                 # Nextra sidebar: Overview, Research, Media, Stoic, Tao
│   ├── _app.tsx                   # App wrapper (imports globals.css)
│   └── api/
│       ├── dashboard.ts           # Aggregated KPIs from 4 Notion DBs
│       ├── contentvault.ts        # ContentVault items + metrics
│       ├── datavault.ts           # DataVault saved articles
│       ├── digest.ts              # Cross-vault activity feed (5 sources, fingerprint cache)
│       ├── ai/media-insights.ts   # Claude Haiku 4.5 media ranking + fallback
│       ├── github/issues.ts       # GitHub Project V2 board (GraphQL)
│       └── tao/
│           ├── status.ts          # KPIs: revenue, unfulfilled, goals (getLatestMetric)
│           ├── orders.ts          # Orders with Notion-side unfulfilled filter
│           ├── tasks.ts           # Tasks with client-side status filter (polymorphic)
│           └── feedback.ts        # Reviews with ?recent=true (last 3 months)
├── components/
│   ├── DashboardData.tsx          # 6 overview components (627 lines)
│   ├── TaoPromotionData.tsx       # 8 Tao components (252 lines)
│   ├── ContentVaultData.tsx       # 6 media components (212 lines)
│   ├── DataVaultData.tsx          # 5 research components (197 lines)
│   ├── VaultCard.tsx              # Generic vault card (legacy, unused)
│   ├── MetricsDisplay.tsx         # Generic metrics grid (legacy, unused)
│   ├── VaultLink.tsx              # External link button with SVG icon
│   ├── LoadingState.tsx           # 4 skeleton types + ErrorState with retry
│   └── index.ts                   # Barrel exports (25 components)
├── hooks/
│   └── useNotionData.ts           # Client fetch hooks + 5-min localStorage cache
├── lib/
│   ├── notion.ts                  # Notion SDK client + 12 property helpers
│   ├── types.ts                   # 14 TypeScript interfaces
│   └── cache.ts                   # Server-side 30-min cache + fingerprint invalidation
├── data/
│   └── vaults.json                # Static snapshot (Stoic page: 12 stocks, 21 investments)
├── styles/
│   └── globals.css                # Nextra overrides + 3 responsive breakpoints
└── theme.config.tsx               # Nextra theme: grayscale, "MetaVault" logo
```

---

## Pages

| Route | Sidebar Name | Data Source | Components |
|-------|-------------|-------------|------------|
| `/` | Overview | All Notion DBs + GitHub + Claude AI | ActionRequiredBanner, WeeklyWorkflow, MediaAndResearch, GoalsAndMetrics, ActionsToDo, DigestFeed |
| `/datavault` | Research | Notion DataVault DB | DataVaultHeader, DataVaultItems (expandable), DataVaultAbout, DataVaultLastUpdated |
| `/contentvault` | Media | Notion ContentVault DB | ContentVaultHeader, ContentVaultMetrics, ContentVaultItems, ContentVaultBySource |
| `/stockvault` | Stoic | Static JSON (`data/vaults.json`) | No components — inline MDX rendering 12 stock companies + 21 investment companies grid |
| `/tao-promotion` | Tao | Notion Tao DBs (4) | TaoHeader, TaoGoalsAndMetrics, TaoUnfulfilledOrders, TaoPendingTasks |

Each page includes a `VaultLink` button linking to its standalone app.

---

## Overview Page — Command Center

The main dashboard (`index.mdx`) renders 6 sections in priority order:

### 1. Action Required Banner

**Component**: `ActionRequiredBanner`
**API**: `GET /api/tao/orders?filter=unfulfilled`

Red-background alert showing unfulfilled Tao orders count. Hidden when count is 0. Links to tao-promotion.vercel.app.

### 2. Weekly Workflow

**Component**: `WeeklyWorkflow`
**API**: none (client-side)

7-column day grid highlighting today's day of week. Displays "Coming soon — weekly workflow tracking."

### 3. Media + Research

**Component**: `MediaAndResearch`
**API**: `GET /api/ai/media-insights` (primary), `GET /api/contentvault` + `GET /api/datavault` (fallback)

Two rendering modes:

- **AI view** (primary): Items ranked by Claude with colored left borders based on `relevanceScore`:
  - Green (`#22c55e`) — score ≥ 80
  - Yellow (`#eab308`) — score ≥ 50
  - Gray (`#6b7280`) — score < 50

  Each item shows vault badge (CV=purple `#7c3aed`, DV=blue `#2563eb`), summary, and relevance reason.

- **Fallback view** (`MediaFallbackView`): Fetches ContentVault + DataVault separately, merges by `lastEdited` descending, shows top 5 items.

### 4. Goals & Metrics

**Component**: `GoalsAndMetrics`
**API**: `GET /api/dashboard`

4 KPI cards in a `.goals-grid`, each linking to tao-promotion.vercel.app:
- Amazon Sales (count from goals DB)
- Amazon Reviews (combined .com + .ca)
- Subscribers (count from goals DB)
- Shopify Revenue (total revenue from orders DB, formatted as `$X`)

### 5. Actions / To Do

**Component**: `ActionsToDo`
**API**: `GET /api/github/issues`

GitHub Project V2 board items grouped by status with colored headers:
- "Up next" — yellow `#f59e0b`
- "In progress" — blue `#3b82f6`
- "For Review" — purple `#8b5cf6`

Each status section is expandable (click to show/hide issues). Shows count per status.

### 6. Digest

**Component**: `DigestFeed`
**API**: `GET /api/digest`

Cross-vault activity timeline with source badges:
- ContentVault (CV) — purple `#7c3aed`
- DataVault (DV) — blue `#2563eb`
- Tao Promotion (TP) — amber `#d97706`
- GitHub (GH) — gray `#6b7280`

Each entry is expandable to show structured details (DigestDetail label/value pairs). Displays relative time ("2h ago", "3d ago"). CSS timeline with dot markers and vertical line.

---

## API Routes (10 endpoints)

All routes enforce `GET` only (405 for other methods). All set `Cache-Control: s-maxage=300, stale-while-revalidate=600`.

### Core Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /api/dashboard` | Fetches 4 Notion DBs in parallel via `Promise.allSettled`. Returns taoStatus (unfulfilled count, total revenue, latest amazon sales/reviews/subscribers from goals DB using `getLatestMetric()`) + ContentVault/DataVault item counts. |
| `GET /api/contentvault` | All items with URL priority: Audio summary (files) > Audio Link (url) > Text summary (files). Returns `{ metrics, bySource, byType, items }`. |
| `GET /api/datavault` | Saved articles: Title, Authors, Subject, Description, Submission date, pdf Link. Returns `{ metrics, status, items }`. |
| `GET /api/digest` | 5 fetchers run via `Promise.allSettled`: ContentVault (5 items), DataVault (5), Tao Orders (5), Tao Tasks (10 → filter completed → take 5), GitHub (Project V2 issues + last 3 repo commits with additions/deletions/changedFiles stats). Merges chronologically, returns top 10. Server-side fingerprint cache. |

### AI Endpoint

| Endpoint | Description |
|----------|-------------|
| `GET /api/ai/media-insights` | 1. Fetches ContentVault + DataVault + active Tao Tasks (In Progress/To Do) in parallel. 2. Checks fingerprint cache. 3. If no `ANTHROPIC_API_KEY`: returns recency-sorted fallback. 4. Calls Claude Haiku 4.5 with ranking prompt (recency, status, task alignment, diversity, novelty). 5. **Validates** AI response IDs against real Notion page IDs — drops hallucinated items. 6. Returns max 5 items with `relevanceScore` + `relevanceReason`. |

**AI Flow**:
```
ContentVault items ──┐
DataVault items ─────┼──→ Claude Haiku 4.5 ──→ Validate IDs ──→ Max 5 ranked items
Active Tao Tasks ────┘    (ranking prompt)     (vs real IDs)
```

### GitHub Endpoint

| Endpoint | Description |
|----------|-------------|
| `GET /api/github/issues` | GraphQL query for user `GuillaumeRacine`, project #4. Returns items grouped by status with counts. |

### Tao Promotion Endpoints

| Endpoint | Details |
|----------|---------|
| `GET /api/tao/status` | Queries Orders DB for unfulfilled count + total revenue. Queries Goals DB — `getLatestMetric()` sorts by "Last Updated" date descending, finds most recent value per metric. Metric names: "number of sales", "amazoncom reviews", "amazonca reviews", "number of subscribers". Returns `TaoStatus` interface. |
| `GET /api/tao/orders` | Sorted by Date descending. `?filter=unfulfilled` applies **Notion-side** select filter on Fulfillment="Unfulfilled". Properties: Order (title), Customer/Name (rich_text), Total $ (number), Date (date), Payment (select), Fulfillment (select). Each order includes Notion URL. |
| `GET /api/tao/tasks` | Fetches all tasks, **client-side** status filtering because Status can be `rich_text` (not filterable in Notion API). Polymorphic Status: tries `status.name` → `select.name` → `rich_text[0].plain_text`. Polymorphic Priority: tries `select.name` → `rich_text[0].plain_text`. `?status=To Do` to filter. |
| `GET /api/tao/feedback` | Sorted by created_time descending. `?recent=true` adds Notion filter: `created_time.on_or_after` = 3 months ago. `?limit=5` for page size. Properties: Title/Name (title), Feedback/Content (rich_text), User Name (rich_text). |

---

## Three-Layer Caching

```
Layer 1: Server in-memory (lib/cache.ts)
├── TTL: 30 minutes (DEFAULT_TTL = 30 * 60 * 1000)
├── Fingerprint-based invalidation
│   buildFingerprint(): joins sorted "id:lastEdited" pairs with "|"
│   getCached(): returns null if expired OR fingerprint mismatch
│   setCache(): stores data + fingerprint + expiresAt
├── Store: Map<string, CacheEntry<unknown>>
└── Used by: /api/digest, /api/ai/media-insights

Layer 2: CDN / Vercel Edge
├── Cache-Control: s-maxage=300 (5 min)
├── stale-while-revalidate=600 (10 min)
└── Set on all 10 API routes

Layer 3: Client localStorage (hooks/useNotionData.ts)
├── TTL: 5 minutes (CACHE_TTL = 5 * 60 * 1000)
├── useNotionData<T>(endpoint, cacheKey?) — single fetch with cache
├── useMultipleNotionData<T>(endpoints) — parallel fetch with cache
├── refetch() — bypasses cache (skipCache=true)
├── clearCache(key?) — clears specific or all "notion-*" entries
└── SSR-safe: all cache ops check typeof window !== 'undefined'
```

---

## Notion Databases (6)

| Database | Env Var | Properties Used in Code |
|----------|---------|------------------------|
| ContentVault | `NOTION_CONTENTVAULT_DB` | Link (title), type (select), Status (select), Date (date), Audio Link (url), Audio summary (files), Text summary (files) |
| DataVault | `NOTION_DATAVAULT_DB` | Title (title), Authors (rich_text), Subject (rich_text), Description (rich_text), Submission (date), pdf Link (url) |
| Tao Orders | `NOTION_TAO_ORDERS_DB` | Order (title), Customer/Name (rich_text), Total $ (number), Date (date), Payment (select), Fulfillment (select) |
| Tao Tasks | `NOTION_TAO_TASKS_DB` | Name (title), Status (status/select/rich_text — polymorphic), Priority (select/rich_text), Due Date (date) |
| Tao Feedback | `NOTION_TAO_FEEDBACK_DB` | Title/Name (title), Feedback/Content (rich_text), User Name (rich_text) |
| Tao Goals | `NOTION_TAO_GOALS_DB` | Metric Name (title), Number (number), Last Updated (date) |

### Notion Property Helpers (`lib/notion.ts`)

Uses `@notionhq/client` SDK with typed `QueryDatabaseParameters`. 12 extraction functions:

| Helper | Returns | Source |
|--------|---------|--------|
| `getTitle(prop)` | string | `prop.title[].plain_text` joined |
| `getRichText(prop)` | string | `prop.rich_text[].plain_text` joined |
| `getNumber(prop)` | number | `prop.number` (default 0) |
| `getSelect(prop)` | string | `prop.select.name` |
| `getMultiSelect(prop)` | string[] | `prop.multi_select[].name` |
| `getDate(prop)` | string | `prop.date.start` |
| `getUrl(prop)` | string | `prop.url` |
| `getFileUrl(prop)` | string | First file: `external.url` → `file.url` → `name` |
| `getPerson(prop)` | string | `prop.people[].name` or `person.email`, joined |
| `getAnyText(prop)` | string | Tries: rich_text → title → people → select → email |
| `formatDate(str)` | string | Formats to "Mon DD, YYYY" (en-US) |
| `queryDatabase(params)` | QueryDatabaseResponse | Notion SDK `databases.query()` |

---

## Component Architecture

### Per-Page Component Files

| File | Exports | Description |
|------|---------|-------------|
| `DashboardData.tsx` (627 lines) | ActionRequiredBanner, WeeklyWorkflow, MediaAndResearch, GoalsAndMetrics, ActionsToDo, DigestFeed | Overview page. AI + fallback views for media. GitHub 3-column board with expandable statuses. Digest timeline with expandable details + relative time. |
| `TaoPromotionData.tsx` (252 lines) | TaoHeader, TaoGoalsAndMetrics, TaoMetrics, TaoUnfulfilledOrders, TaoPendingTasks, TaoGoals, TaoRecentReviews, TaoLastUpdated | Tao page. 5 KPI cards: Amazon Sales, Shopify Revenue, Subscribers, Amazon.com Reviews, Amazon.ca Reviews. Orders table (Order/Customer/Amount/Date). Task list with priority color badges. |
| `ContentVaultData.tsx` (212 lines) | ContentVaultHeader, ContentVaultMetrics, ContentVaultBySource, ContentVaultByType, ContentVaultItems, ContentVaultLastUpdated | Media page. 3 metrics (Total Items, To Read, Inbox). Source grid (pill badges). Type list. Items with status color coding: yellow (To Read/Inbox), green (Done). Show more/less toggle. |
| `DataVaultData.tsx` (197 lines) | DataVaultHeader, DataVaultStatus, DataVaultItems, DataVaultAbout, DataVaultLastUpdated | Research page. Expandable article list with rotating ▶ arrow. Expanded view: Description, Authors, Subject, Submitted, "View PDF ↗" link. Static About section describing Research Vault features. |

### Shared Components

| Component | File | Details |
|-----------|------|---------|
| `VaultLink` | `VaultLink.tsx` | "Open {name}" button with SVG arrow icon. White bg (#ffffff), styled-jsx. Hover: translateY(-1px) + bg change. |
| `LoadingState` | `LoadingState.tsx` | 4 types: `spinner` (rotating circle, 40px), `skeleton` (row bars), `card` (2-col grid with icon+title+metrics placeholders), `metrics` (metric boxes). All with 1.5s pulse animation. Dark mode via `:global(.dark)`. |
| `ErrorState` | `LoadingState.tsx` | Error icon + message + optional "Retry" button. Dark bg (#1f2937), border #6b7280. |
| `VaultCard` | `VaultCard.tsx` | Card with gradient bg, MetricsDisplay, VaultLink. Legacy — not imported by any current page. |
| `MetricsDisplay` | `MetricsDisplay.tsx` | Auto-fit grid of metric items. 35 label mappings (totalDocs→"Documents", toRead→"To Read", etc.). Legacy — not imported by any current page. |

### Barrel Exports (`components/index.ts`)

Exports all 25 components from 8 files: VaultCard, MetricsDisplay, VaultLink, LoadingState, ErrorState, 6 Dashboard, 8 Tao, 6 ContentVault, 5 DataVault.

---

## Styling

### Inline Styles (Page Components)

All 4 page component files use `style={{}}` objects exclusively. Dark gray palette:
- Backgrounds: `#1f2937`, `#374151`, `#4b5563`
- Text: `#6b7280`, `#9ca3af`, `#d1d5db`, `#f9fafb`
- Accent colors per context: red (alerts), green (goals/scores), yellow (pending), blue (info), purple (content), amber (Tao)
- Hover: `onMouseEnter`/`onMouseLeave` handlers changing inline `backgroundColor`
- Conditional tags: items with URLs render as `<a>`, others as `<div>`

### Styled-jsx (Shared Components)

VaultLink, LoadingState, ErrorState, VaultCard, MetricsDisplay use `<style jsx>` blocks with dark mode via `:global(.dark)` selectors.

### globals.css (Nextra Overrides)

Compact heading sizes (h1: 1.25rem, h2: 0.9375rem, h3: 0.8125rem), reduced spacing, smaller paragraphs (0.75rem). Named CSS classes for component grids:
- `.goals-grid` — auto-fit minmax(70px, 1fr)
- `.actions-grid` — 3 columns
- `.workflow-grid` — 7 columns
- `.digest-timeline` — left padding + line
- `.vault-cards-grid` — 2 columns

3 responsive breakpoints:
- `768px` — hide TOC, single-column cards, tighter digest, 2×2 goals, text wrapping
- `480px` — smaller headings (h1: 1rem), reduced padding
- `380px` — extra-tight digest timeline, 3-col actions with smaller gap

---

## TypeScript Interfaces (`lib/types.ts`)

14 interfaces covering all data shapes:

| Interface | Fields |
|-----------|--------|
| `TaoOrder` | id, name, total, date, payment, fulfillment, url? |
| `TaoTask` | id, title, status, priority, due, url? |
| `TaoReview` | id, title, content, userName? |
| `TaoStatus` | metrics (unfulfilled, amazonSales, amazonReviews, shopifySales), goals (amazonSales, amazonComReviews, amazonCaReviews, subscribers) |
| `ContentItem` | id, title, source, type, status, url?, lastEdited? |
| `ContentVaultData` | metrics (totalItems, toRead, inbox), bySource[], byType[], items[] |
| `DataItem` | id, title, authors, subject, description?, submission?, url?, lastEdited? |
| `DataVaultData` | metrics (totalItems), status, items[] |
| `MediaInsightItem` | id, title, vault ('CV'/'DV'), summary, relevanceScore, relevanceReason, url?, lastEdited?, source?, type?, status?, authors?, subject? |
| `MediaInsightsResponse` | items[], generatedAt, cached |
| `DigestDetail` | label, value |
| `DigestItem` | id, source (5 union types), icon, text, details?, timestamp, url?, color? |
| `DigestResponse` | items[], generatedAt, cached |
| `VaultMetrics` / `VaultData` | Legacy card interfaces |

---

## Static Data (`data/vaults.json`)

Snapshot data used by the Stoic page (fully static, no Notion API calls):

- **12 stock companies**: Amazon, Circle, Coinbase, Constellation Software, Ebay, Etsy, FIGS, LVMH, nVidia, Shopify, Wayfair, Yeti
- **21 investment companies**: Abra Promotions, Builder Io, Ceremonia, Constructor, Daily Blends, Design Stripe, Helika, Hookdeck, Immune Biosolution, Kotn, Nolk, Pivohub, Ripple AI, Screenloop, Shakepay, Three Ships, Vasco, Waverly, Wavyy, Wonderment, Zeffy
- **3 quarters** of financial analysis
- Last updated: 2026-01-30

Also contains snapshot Tao metrics (118 Amazon sales, 97 reviews, $292 Shopify) and ContentVault metrics (19 items, 18 to read, 1 inbox) used as static fallback.

---

## Nextra Theme Configuration

```tsx
// theme.config.tsx
{
  logo: <span style={{ fontWeight: 700 }}>MetaVault</span>,
  primaryHue: 0,            // grayscale
  primarySaturation: 0,     // grayscale
  footer: { text: 'MetaVault - Your vault ecosystem hub' },
  sidebar: { defaultMenuCollapseLevel: 1, toggleButton: true },
  toc: { backToTop: true },
  useNextSeoProps: () => ({ titleTemplate: '%s – MetaVault' })
}
```

---

## External Vault Links

| Page | Standalone App |
|------|---------------|
| Research | datavault-rust.vercel.app/areas |
| Media | media-minivault.vercel.app/vault |
| Stoic | stock-vault.vercel.app |
| Tao | tao-promotion.vercel.app/dashboard |

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

## Security

- **CVE-2026-0969 fix**: `next-mdx-remote` overridden to `^6.0.0` in `package.json` overrides section
- All API keys are server-side only (no `NEXT_PUBLIC_` prefix)
- No authentication — dashboard is intentionally read-only

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
7. **AI integration** — Claude Haiku 4.5 for media curation with ID validation
8. **Digest feed** — cross-vault activity timeline from 5 sources with fingerprint cache
9. **Mobile responsive** — responsive CSS with 3 breakpoints, card overflow fixes
10. **Security** — CVE-2026-0969 fix (next-mdx-remote ^6.0.0 override)

---

## Design Decisions

1. **Nextra docs theme** — sidebar navigation, search, MDX pages with embedded React components
2. **Grayscale palette** — `primaryHue: 0`, `primarySaturation: 0` for neutral UI
3. **Inline styles over Tailwind** — page components use `style={{}}` for dark theme consistency without framework dependency
4. **Styled-jsx for shared** — LoadingState, VaultLink, VaultCard use `<style jsx>` for scoped CSS with dark mode via `:global(.dark)`
5. **Three-layer caching** — server (30 min, fingerprint invalidation) + CDN (5 min) + client (5 min) for fast loads without hammering Notion
6. **AI ID validation** — media-insights validates Claude's response IDs against real Notion page IDs, drops hallucinated items to prevent broken links
7. **Notion type polymorphism** — Status handled as `status`, `select`, or `rich_text` property types for compatibility with different Notion configurations
8. **Client-side task filtering** — tasks filtered in JavaScript because Status can be `rich_text` (not filterable via Notion API query)
9. **Promise.allSettled everywhere** — dashboard, digest, AI endpoints all survive partial source failures
10. **ContentVault URL priority** — Audio summary > Audio Link > Text summary (prefers audio files over text summaries)
11. **All GET only** — API routes enforce GET, return 405 for other methods
12. **Digest GitHub dual fetch** — fetches both Project V2 issues AND repository commits via two parallel GraphQL queries, with commit stats (additions/deletions/changedFiles)

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Empty dashboard | Check all `NOTION_*` env vars in `.env.local` |
| AI insights showing fallback list | Verify `ANTHROPIC_API_KEY` is set and has credits |
| GitHub section empty | Check `GITHUB_TOKEN` has `read:project` scope |
| Stale data | Clear localStorage (`clearCache()`) or wait for CDN TTL (5 min) |
| Unfulfilled orders not showing | Verify Tao Orders DB has "Fulfillment" select property with "Unfulfilled" option |
| Digest feed partial | Expected — `Promise.allSettled` allows partial source failures gracefully |
| Task status not matching | Status field may be `rich_text` instead of `status`/`select` — filtering happens client-side |
| Goals showing 0 | Check Goals DB has "Last Updated" date and "Number" number properties populated |
| Stoic page empty | Check `data/vaults.json` exists (fully static, no API) |
