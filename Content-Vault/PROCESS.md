# Content Vault (Media Vault) — Process Documentation

## Overview

Read-only content consumption frontend backed by a single Notion database. Displays AI-generated text and audio summaries from an external n8n Transcripts pipeline. Features read/unread tracking, favorites system, full-screen markdown preview with react-markdown, and a Nextra documentation site. Forked from MiniVault-TaoPromotion and progressively simplified into a focused content reader.

**Repo**: [yasser-ensembl3/Media-Minivault](https://github.com/yasser-ensembl3/Media-Minivault)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Framework | Next.js 14 (App Router) |
| Language | TypeScript |
| UI | shadcn/ui + Tailwind CSS v4 (dark mode locked) |
| Documentation | Nextra v4 (MDX pages at `/docs`) |
| Markdown rendering | react-markdown + @tailwindcss/typography |
| Icons | lucide-react |
| Data layer | Notion API (raw fetch, not SDK) |
| Deployment | Vercel |

---

## Architecture

```
ContentVault/
├── app/
│   ├── api/
│   │   ├── content/route.ts       # Notion CRUD (GET/POST/PATCH/DELETE)
│   │   └── markdown/route.ts      # Markdown file proxy (Google Drive URL conversion)
│   ├── vault/
│   │   ├── page.tsx               # Main page — unread content
│   │   ├── archive/page.tsx       # Read/completed content
│   │   └── favorites/page.tsx     # Favorited content
│   ├── docs/
│   │   ├── layout.tsx             # Nextra docs layout
│   │   └── [[...mdxPath]]/page.tsx # Dynamic MDX page renderer
│   ├── layout.tsx                 # Root layout (dark mode, Inter font)
│   ├── page.tsx                   # Redirects / → /vault
│   ├── globals.css                # Tailwind v4 + CSS custom properties
│   └── _meta.ts                   # Nextra navigation config
├── components/
│   ├── content-list.tsx           # Main state manager — fetch, filter, actions
│   ├── content-item.tsx           # Content card — markdown modal, audio, favorites
│   ├── filter-bar.tsx             # Type/Status/Source filters (unused, kept)
│   ├── search-input.tsx           # Search input (unused, kept)
│   ├── add-content-form.tsx       # Content creation form (unused, kept)
│   └── ui/                        # shadcn/ui primitives (badge, button, card, input, select)
├── content/
│   ├── index.mdx                  # Docs — introduction
│   ├── setup.mdx                  # Docs — setup guide
│   ├── api-reference.mdx          # Docs — API endpoints
│   ├── notion-schema.mdx          # Docs — database schema
│   └── _meta.ts                   # Docs navigation order
├── lib/
│   └── utils.ts                   # cn() utility + color generator
├── mdx-components.tsx             # Nextra MDX bridge
└── next.config.mjs                # Next.js wrapped with Nextra
```

---

## Pages & Routing

| Route | Page | Description |
|-------|------|-------------|
| `/` | `app/page.tsx` | Redirects to `/vault` |
| `/vault` | `app/vault/page.tsx` | Unread content (status != "Done") |
| `/vault/archive` | `app/vault/archive/page.tsx` | Read content (status == "Done") |
| `/vault/favorites` | `app/vault/favorites/page.tsx` | Favorited content |
| `/docs` | Nextra MDX | Documentation home |
| `/docs/setup` | Nextra MDX | Setup guide |
| `/docs/notion-schema` | Nextra MDX | Database schema reference |
| `/docs/api-reference` | Nextra MDX | API documentation |

---

## Data Flow

```
[n8n Transcripts Pipeline (external)]
    → Processes articles, podcasts, videos
    → Generates AI text summaries (.md files)
    → Generates audio summaries
    → Writes to Notion database
           │
           ▼
[Notion Database]
    ← Notion REST API (2022-06-28)
           │
           ▼
[/api/content/route.ts]
    GET:    Query all items, return simplified JSON
    POST:   Create new page
    PATCH:  Toggle status/favorite
    DELETE: Archive page
           │
           ▼
[content-list.tsx]
    Fetches all items on mount
    Client-side filtering by mode:
      - "unread": status !== "Done"
      - "read": status === "Done"
      - "favorites": favorite === true
           │
           ▼
[content-item.tsx]
    Renders card with type badge, date, actions
    Click behavior:
      1. Has .md file → markdown preview modal
      2. Has audio file → open audio URL
      3. Otherwise → open Notion URL
           │
           ▼
[/api/markdown/route.ts]  (if markdown preview)
    Proxies .md file download
    Converts Google Drive URLs to direct download
    Returns { content: "..." }
```

---

## Content Card Behavior

### Type Badges

| Type | Color | Icon |
|------|-------|------|
| Podcast | Purple | Headphones |
| Youtube video | Red | Video |
| Audio | Amber | Headphones |
| Talk Show | Cyan | MessageSquare |
| Post | Blue | FileText |
| Article/News | Green | FileText |

### Click Priority

1. If `.md` text summary exists → opens **full-screen markdown modal**
2. If audio summary exists → opens **audio URL** in new tab
3. Otherwise → opens **Notion URL** in new tab

### Markdown Preview Modal

- Full-screen dark overlay
- Fetches `.md` content via `/api/markdown?url=...`
- Rendered with `react-markdown` + Tailwind typography prose classes
- Download button (saves as `.md` file)
- "Listen" button (if audio URL also exists)
- Mobile-responsive with footer action buttons

### Actions Per Card

- **Heart icon** — toggle favorite (PATCH `{ id, favorite: true/false }`)
- **Check icon** — mark as read/done (PATCH `{ id, status: "Done" }`)
- **Undo icon** — mark as unread (PATCH `{ id, status: "Inbox" }`)

---

## Read/Unread Workflow

```
/vault (unread)                    /vault/archive (read)
┌─────────────┐    mark done      ┌─────────────┐
│ Inbox       │ ──────────────▶  │ Done        │
│ To Read     │                   │             │
│ Reading     │  ◀──────────────  │             │
└─────────────┘    mark unread    └─────────────┘

/vault/favorites — shows all favorited items regardless of read status
```

---

## API Routes

### /api/content (route.ts)

**GET** — Query Notion database

- Sorts by `Date` descending, page size 100, `cache: "no-store"`
- Optional filters: `type` (select match), `search` (title contains)
- Returns `{ items: [...], types: ["Article", "Podcast", ...] }`
- Each item: `id`, `title`, `type`, `status`, `favorite`, `dateAdded`, `notionUrl`, `mdFileUrl`, `audioUrl`
- Extracts `.md` URLs from "Text summary" files property
- Extracts audio URLs from "Audio summary" files property

**POST** — Create new page

- Body: `{ title, url, type?, source?, status? }`
- Status defaults to "Inbox", auto-sets "Date Added" to today

**PATCH** — Update page

- Body: `{ id, status?, favorite? }`
- Used for read/unread toggle and favorite toggle

**DELETE** — Archive page

- Query: `?id=xxx`
- Notion API archives (not true delete)

### /api/markdown (route.ts)

**GET** — Proxy markdown file download

- Query: `?url=xxx`
- Converts Google Drive view URLs (`drive.google.com/file/d/{id}`) to direct download URLs
- Returns `{ content: "markdown text..." }`

---

## Notion Database Schema

| Property | Notion Type | Required | Description |
|----------|-------------|----------|-------------|
| Link | Title | Yes | Content title (GET reads from this) |
| URL | URL | Yes | Source link |
| type | Select | Yes | Article, Podcast, Youtube video, Talk Show, Post, Audio |
| Status | Select | Yes | Inbox, To Read, Reading, Done, Archived |
| Favorite | Checkbox | No | Favorite flag for heart toggle |
| Date | Date | No | Content publication date |
| Date Added | Date | Yes | When added (auto-set on POST) |
| Source | Select | No | YouTube, Twitter, Substack, etc. |
| Text summary | Files | No | .md summary file (populated by n8n pipeline) |
| Audio summary | Files | No | Audio summary file/Drive link (populated by n8n pipeline) |

**Note**: The "Text summary" and "Audio summary" fields are populated by an external **n8n Transcripts pipeline** (documented separately in the Transcript-pipeline project) that processes content, generates AI summaries, and writes them to this Notion database.

---

## n8n Integration

This app is the **consumer** side of the Transcripts pipeline:

1. **n8n pipeline** (external) processes content sources (YouTube, articles, podcasts)
2. Pipeline generates text summaries (stored as `.md` files in Notion "Text summary" field)
3. Pipeline generates audio summaries (stored as files/Drive links in "Audio summary" field)
4. **Media Vault** reads these summaries and displays them in the UI
5. Users read summaries via markdown preview modal or listen via audio links

---

## Evolution History

The project went through significant changes (18 commits):

1. **Fork** from MiniVault-TaoPromotion
2. **Transform** into read-only content hub (removed auth, dashboards, multi-database)
3. **Add CRUD** — content creation, custom types/sources
4. **Embedded previews** — Notion and Google Docs iframe support (later simplified)
5. **Migrate to Nextra** — added documentation site with Tailwind v4
6. **Simplify** — removed search bar, filters, and add content form from UI
7. **n8n integration** — switched to Transcripts database with text/audio summary support
8. **Favorites system** — heart icon, favorites page, read/unread toggle
9. **Rename** — ContentVault → Media Vault

---

## Unused Components (kept in codebase)

| Component | File | Original Purpose |
|-----------|------|-----------------|
| FilterBar | `filter-bar.tsx` | Type/Status/Source dropdown filters |
| SearchInput | `search-input.tsx` | Text search with clear button |
| AddContentForm | `add-content-form.tsx` | Content creation form (POST endpoint still works) |

These were removed from the UI during simplification but the files and the POST API endpoint remain functional.

---

## Environment Variables

```env
# Required
NOTION_TOKEN=secret_xxx
NEXT_PUBLIC_NOTION_DATABASE_ID=xxx

# Optional
NEXT_PUBLIC_SITE_NAME=Media Vault
```

---

## Setup

### Prerequisites

- Node.js 18+
- Notion integration with database access

### Installation

```bash
npm install
cp .env.example .env.local
# Add NOTION_TOKEN and NEXT_PUBLIC_NOTION_DATABASE_ID
npm run dev
# Open http://localhost:3000
```

### Notion Setup

1. Create integration at notion.so/my-integrations
2. Share your content database with the integration
3. Copy the database ID from the URL
4. Add token + database ID to `.env.local`

### Deployment (Vercel)

1. Push to GitHub
2. Import in Vercel
3. Add environment variables
4. Deploy

---

## Credentials

| Service | Type | Purpose |
|---------|------|---------|
| Notion | Internal integration token | Read/write content database |

No authentication required for end users — the app is intentionally open/read-only (Notion token is server-side only).

---

## Design Decisions

1. **No auth** — intentionally open, server-side Notion token only
2. **Single database** — one Notion DB is the only data source
3. **Read-first** — optimized for consuming content, not managing it
4. **Dark mode locked** — `<html className="dark">`, no toggle
5. **Client-side filtering** — all items fetched once, filtered in the browser by mode
6. **Nextra docs** — built-in documentation site at `/docs` for self-reference
7. **Raw fetch over SDK** — `@notionhq/client` installed but unused, all Notion calls use raw `fetch()`

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Empty content list | Check `NOTION_TOKEN` and `NEXT_PUBLIC_NOTION_DATABASE_ID` in `.env.local` |
| Markdown preview empty | Check "Text summary" field has a `.md` file attached in Notion |
| Audio not playing | Verify "Audio summary" contains a valid URL/file; Google Drive links are converted automatically |
| Notion 401 error | Ensure integration has access to the database (Share → invite integration) |
| Types not showing | The `type` property must be a Select type in Notion |
