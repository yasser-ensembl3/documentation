# DataVault — Process Documentation

## Overview

Research management platform for organizing academic papers, tracking research hypotheses, and exploring data sources. Built with Next.js 14, integrated with Notion for persistent storage, and featuring search across ArXiv and Semantic Scholar (200M+ papers).

**Repo**: [yasser-ensembl3/datavault](https://github.com/yasser-ensembl3/datavault)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Framework | Next.js 14, React 18 |
| Language | TypeScript |
| Styling | Tailwind CSS 4, Radix UI, shadcn/ui |
| Database | Notion API |
| Search APIs | ArXiv XML API, Semantic Scholar REST API |
| Docs engine | Nextra 4.6 (MDX) |
| MCP server | Model Context Protocol for Claude integration |
| Automation | n8n webhook (optional) |

---

## Architecture

```
DataVault/
├── app/
│   ├── layout.tsx                    # Root layout (sidebar, dark theme)
│   ├── page.tsx                      # Home (redirects to /areas)
│   ├── areas/page.tsx                # Research areas + paper search
│   ├── saved/page.tsx                # Saved papers management
│   ├── assumptions/page.tsx          # Hypothesis tracking + annexes
│   ├── apis-dataset/page.tsx         # API catalog (23+ sources)
│   ├── docs/                         # Nextra documentation
│   └── api/
│       ├── areas/
│       │   ├── search-hybrid/        # ArXiv + Semantic Scholar enrichment
│       │   ├── search-advanced/      # Semantic Scholar with filters
│       │   ├── search/               # Basic ArXiv search
│       │   ├── enrich/               # Batch paper enrichment
│       │   ├── papers/               # n8n webhook integration
│       │   └── download/             # PDF download proxy
│       ├── saved/                    # CRUD saved papers (Notion)
│       ├── assumptions/              # CRUD hypotheses (Notion)
│       └── content/                  # Content management
├── components/
│   ├── sidebar.tsx                   # Fixed sidebar navigation
│   ├── mobile-nav.tsx                # Bottom nav for mobile
│   ├── areas/area-badge.tsx          # Topic selection badges
│   └── ui/                          # shadcn/ui (button, card, badge, input, select)
├── lib/
│   ├── apis/
│   │   ├── arxiv.ts                 # ArXiv XML API client + parser
│   │   └── semantic-scholar.ts      # Semantic Scholar client + rate limit retry
│   └── utils.ts                     # Color generation, class merge
├── content/                          # Nextra MDX documentation source
├── data/                             # Static data
└── mcp-server/
    └── src/index.ts                  # MCP server — full CRUD on 10+ Notion databases
```

---

## Features — How They Work

### 1. Research Areas (`/areas`)

The main search interface. Users create topic areas (e.g. "ADHD", "LLM Training") and search for papers within each.

**Two search modes:**

| Mode | How it works |
|------|-------------|
| ArXiv Hybrid | Queries ArXiv XML API → extracts arXiv IDs → batch enriches via Semantic Scholar (citations, venue, TL;DR) → merges results |
| Semantic Scholar Direct | Queries Semantic Scholar (200M+ papers) with advanced filters: year, citation count, publication type, fields of study, venue, open access |

**Filtering & sorting:**
- Year range, minimum citations, publication type, fields of study, venue, open access only
- Sort by citations, date, or title

**Caching:**
- Client-side localStorage with 24-hour TTL
- Cache key is a hash of (source, query, filters)

**Rate limiting (Semantic Scholar):**
- Auto-retry on 429 with exponential backoff (3s, 6s, 9s)
- Optional API key in env for higher rate limits

### 2. Saved Papers (`/saved`)

Bookmark papers to a Notion database for later reference.

- Save/remove/clear all operations
- Modal view with full paper metadata: authors, abstract, citations, venue, publication type
- PDF download via proxy route (`/api/areas/download`)
- All data persisted in Notion (`NOTION_SAVED_DATABASE_ID`)

### 3. Research Hypotheses (`/assumptions`)

Track research hypotheses through a validation workflow.

**Status workflow:**
```
Hypothesis → Testing → Validated
                    → Invalidated
                    → Revised
```

**Fields:**

| Field | Required | Values |
|-------|----------|--------|
| Title | Yes | The hypothesis statement |
| Description | No | Detailed explanation |
| Status | No | Hypothesis (default), Testing, Validated, Invalidated, Revised |
| Confidence | No | Low, Medium, High |
| Area | No | Research domain (e.g. "ADHD") |
| Evidence | No | Summary of supporting evidence |
| Sources | No | URLs of supporting papers/articles |
| Notes | No | Additional notes |

**Annexes:** Each hypothesis can have attached annexes (links, documents, free-text notes). Stored as Notion page children blocks.

**Visual indicators:** Status and confidence levels are color-coded in the UI.

All data synced to Notion (`NOTION_ASSUMPTIONS_DATABASE_ID`).

### 4. APIs/Dataset Catalog (`/apis-dataset`)

Curated list of 23+ research APIs and data sources, organized by category:

| Category | Examples |
|----------|----------|
| Health | PubMed, ClinicalTrials.gov, openFDA, Infermedica, Lexigram |
| Academic | Semantic Scholar, OpenAlex, CrossRef, CORE, Unpaywall |
| Machine Learning | HuggingFace, Papers With Code, Kaggle |
| Science | NASA ADS, Nobel Prize data, Open Science Framework |
| Knowledge | Wikidata, Wikipedia API, Wolfram Alpha |

Each entry shows: description, category, API docs link, active/inactive toggle.

### 5. Documentation (`/docs`)

Built-in documentation via Nextra 4.6 (MDX-based). Covers setup guides, Notion schema, and API reference. Rendered with sidebar navigation.

---

## API Routes Reference

| Route | Method | Description |
|-------|--------|-------------|
| `/api/areas/search-hybrid` | GET | ArXiv search + Semantic Scholar enrichment |
| `/api/areas/search-advanced` | GET | Semantic Scholar with advanced filters |
| `/api/areas/search` | GET | Basic ArXiv search |
| `/api/areas/enrich` | POST | Batch enrich papers with citation data |
| `/api/areas/papers` | GET | Fetch papers via n8n webhook |
| `/api/areas/download` | GET | PDF download proxy |
| `/api/saved` | GET/POST/DELETE | Manage saved papers |
| `/api/saved` | PATCH | Clear all saved papers |
| `/api/assumptions` | GET/POST/PATCH/DELETE | Full CRUD on hypotheses |
| `/api/content` | POST/GET | Content management |

---

## MCP Server

Located in `mcp-server/src/index.ts`. Provides a Model Context Protocol server enabling Claude to interact with all Notion databases directly.

**Capabilities:** 23 tools for CRUD operations on tasks, orders, essentials, metrics, goals, feedback, assumptions, and more.

**Transport:** stdio (configured in `.mcp.json`).

**Usage with Claude:** Allows saving hypotheses, querying databases, and updating records directly from conversation context.

---

## Notion Integration

### Databases Used

| Database | Env Variable | Purpose |
|----------|-------------|---------|
| Saved Papers | `NOTION_SAVED_DATABASE_ID` | Bookmarked research papers |
| Assumptions | `NOTION_ASSUMPTIONS_DATABASE_ID` | Research hypotheses |
| Content | `NEXT_PUBLIC_NOTION_DATABASE_ID` | General content |

### Operations
- Query with filters and pagination (100 max per page)
- Create pages with structured properties
- Update properties (title, status, dates, etc.)
- Archive (soft delete)
- Batch operations for children blocks (annexes)

---

## Installation & Setup

### Prerequisites

- Node.js 18+
- Notion workspace with API access
- (Optional) Semantic Scholar API key for higher rate limits

### Installation

```bash
cd DataVault
npm install
cp .env.example .env.local
# Edit .env.local with your credentials
npm run dev    # Starts on localhost:3000
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `NOTION_TOKEN` | Notion API integration token |
| `NEXT_PUBLIC_NOTION_DATABASE_ID` | Content database ID |
| `NOTION_SAVED_DATABASE_ID` | Saved papers database ID |
| `NOTION_ASSUMPTIONS_DATABASE_ID` | Hypotheses database ID |
| `SEMANTIC_SCHOLAR_API_KEY` | Optional — higher rate limits for Semantic Scholar |
| `N8N_WEBHOOK_URL` | Optional — n8n automation webhook |
| `NEXT_PUBLIC_SITE_NAME` | Browser title (default: DataVault) |

### Notion Setup

1. Create a Notion integration at [notion.so/my-integrations](https://www.notion.so/my-integrations)
2. Grant read/write permissions
3. Create databases for saved papers and assumptions
4. Share each database with the integration
5. Copy database IDs from URLs → set in `.env.local`

---

## Data Models

### Paper

```typescript
{
  id: string
  title: string
  abstract: string
  authors: string
  pdfLink: string
  year?: number
  citationCount: number
  influentialCitationCount: number
  venue: string
  tldr?: string | null
  arxivId?: string
  publicationTypes?: string[]
  categories?: string[]
}
```

### Assumption

```typescript
{
  id: string
  title: string
  description?: string | null
  status: "Hypothesis" | "Testing" | "Validated" | "Invalidated" | "Revised"
  confidence: "Low" | "Medium" | "High"
  area?: string | null
  evidence?: string | null
  sources?: string | null
  notes?: string | null
  createdAt: string
  notionUrl: string
  annexes: { type: "link" | "note" | "document", title: string, url?: string, content?: string }[]
}
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Semantic Scholar rate limited (429) | Auto-retries 3 times with backoff. Add `SEMANTIC_SCHOLAR_API_KEY` for higher limits |
| Notion 401/403 | Check `NOTION_TOKEN` and ensure databases are shared with integration |
| ArXiv returns empty results | Try broader search terms — ArXiv search is keyword-based |
| PDF download fails | Some papers don't have open access PDFs — check `pdfLink` availability |
| Cache stale results | Client cache has 24h TTL — clear localStorage to force refresh |
| MCP server not connecting | Check `.mcp.json` config and ensure `mcp-server/` is built |
