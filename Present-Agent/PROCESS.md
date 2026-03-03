# Present Agent (MiniVault Template) — Process Documentation

## Overview

Reusable project management dashboard boilerplate built with Next.js 14. Designed to be cloned for each new project — a `setup-project.py` script auto-creates 6 Notion databases and generates `.env.local` with all IDs and credentials. Integrates Notion databases (tasks, goals, metrics, milestones, documents, feedback), Google Drive, Gmail, GitHub, and dual AI providers (OpenAI + Anthropic) into a single dark-themed dashboard. Follows a **one repo = one project** architecture where each deployment serves a single project, configured entirely via environment variables.

**Repo**: [yasser-ensembl3/Present-Agent](https://github.com/yasser-ensembl3/Present-Agent)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Framework | Next.js 14 (App Router) |
| Language | TypeScript |
| UI | shadcn/ui + Tailwind CSS (dark mode locked, class strategy) |
| Authentication | NextAuth.js (Google + GitHub OAuth) |
| Data layer | Notion API (raw fetch, not SDK — `@notionhq/client` installed but unused) |
| AI | OpenAI (`gpt-4-turbo-preview`) + Anthropic (`claude-3-5-sonnet-20241022`) |
| Google APIs | Gmail + Drive via service account JWT (`googleapis`) + Drive via OAuth |
| Charts | Recharts (line charts) |
| Calendar | react-day-picker + date-fns |
| CSS Variables | HSL-based design tokens via `globals.css` |
| Animations | tailwindcss-animate + Radix UI data attributes |
| Deployment | Vercel (iad1 region) |
| Setup | Python 3 script (`setup-project.py`) using `urllib.request` |

---

## Architecture

```
Present-Agent/
├── app/
│   ├── api/
│   │   ├── auth/[...nextauth]/route.ts    # NextAuth handler (imports from lib/auth)
│   │   ├── ai/chat/route.ts              # Dual AI — OpenAI + Anthropic
│   │   ├── notion/
│   │   │   ├── tasks/route.ts            # GET/POST — schema-adaptive task CRUD
│   │   │   ├── metrics/route.ts          # GET/POST — shared by goals + metrics
│   │   │   ├── goals/route.ts            # GET/POST — output goals CRUD
│   │   │   ├── documents/route.ts        # GET/POST/PATCH/DELETE — docs links
│   │   │   ├── feedback/route.ts         # GET/POST/PATCH/DELETE — user feedback
│   │   │   ├── milestones/route.ts       # GET/POST/PATCH/DELETE — milestones
│   │   │   └── project-overview/route.ts # GET/POST — page block content
│   │   ├── drive/files/route.ts          # Google Drive (OAuth session token)
│   │   ├── github/repo/route.ts          # GitHub repo, commits, PRs, issues
│   │   ├── google/
│   │   │   ├── drive/route.ts            # Google Drive (service account JWT)
│   │   │   └── gmail/route.ts            # Gmail (service account JWT)
│   │   └── feedback/save/route.ts        # Save feedback as .txt to disk
│   ├── auth/signin/page.tsx               # OAuth sign-in page
│   ├── layout.tsx                         # Root layout (dark mode, Inter, AuthProvider)
│   ├── page.tsx                           # Renders MainDashboard
│   └── globals.css                        # Tailwind base + HSL CSS variables
├── components/
│   ├── auth/session-provider.tsx           # NextAuth SessionProvider wrapper
│   ├── dashboard/
│   │   ├── main-dashboard.tsx             # Orchestrator — session check + 7 sections
│   │   ├── header.tsx                     # Project name, user info, sign out
│   │   ├── dashboard-section.tsx          # Reusable collapsible card (Radix)
│   │   ├── goals-metrics-section.tsx      # 470 lines — green output metrics
│   │   ├── metrics-section.tsx            # 470 lines — blue input metrics
│   │   ├── guides-docs-section.tsx        # 635 lines — 13 link types + auto-sync
│   │   ├── overview-section.tsx           # 662 lines — description/vision + milestones
│   │   ├── project-tracking-section.tsx   # 455 lines — 5-column Kanban
│   │   ├── reports-section.tsx            # 267 lines — AI weekly reports
│   │   ├── user-feedback-section.tsx      # 465 lines — feedback CRUD
│   │   ├── drive-section.tsx              # 220 lines — NOT mounted
│   │   ├── github-section.tsx             # 302 lines — NOT mounted
│   │   └── knowledge-section.tsx          # 73 lines — NOT mounted
│   └── ui/                                # 11 shadcn/ui primitives
│       ├── badge.tsx                      # cva variants (default/secondary/destructive/outline)
│       ├── button.tsx                     # 6 variants + 4 sizes, asChild via Radix Slot
│       ├── calendar.tsx                   # react-day-picker wrapper with custom DayButton
│       ├── card.tsx                       # Card/Header/Title/Description/Content/Footer
│       ├── collapsible.tsx                # Radix Collapsible re-export
│       ├── dialog.tsx                     # Radix Dialog with animated overlay + close button
│       ├── input.tsx                      # Standard text input
│       ├── label.tsx                      # Radix Label with cva
│       ├── popover.tsx                    # Radix Popover with transform-origin
│       ├── select.tsx                     # Radix Select with scroll buttons
│       └── textarea.tsx                   # Textarea with min-h-[80px]
├── lib/
│   ├── auth.ts                            # NextAuth config (Google + GitHub)
│   ├── project-config.ts                  # Env-based ProjectConfig interface + hook
│   └── utils.ts                           # cn() utility (clsx + tailwind-merge)
├── types/
│   └── next-auth.d.ts                     # Session extension (accessToken, refreshToken, provider)
├── data/feedbacks/                        # .txt feedback files (gitignored)
├── contexts/                              # Empty — reserved for future React contexts
├── setup-project.py                       # Automated Notion setup + .env.local generator
├── CLAUDE.md                              # 351 lines — internal project docs for AI assistants
├── PROJECT.md                             # 63.4KB — exhaustive project documentation
├── API.md                                 # 500 lines — API route documentation
├── next.config.js                         # swcMinify, removeConsole, optimizePackageImports
├── tailwind.config.ts                     # shadcn/ui theme with HSL vars + tailwindcss-animate
├── vercel.json                            # region: iad1
└── components.json                        # shadcn/ui config (slate base, cssVariables)
```

---

## Dashboard Sections

Rendered in priority order in `main-dashboard.tsx`. All use the `DashboardSection` wrapper with `keyMetrics` (collapsed summary) and `detailedContent` (expanded view) slots.

| # | Section | Component | Data Source | Theme | Default State |
|---|---------|-----------|------------|-------|---------------|
| 1 | Goals | GoalsMetricsSection | Notion goals DB via `/api/notion/metrics` | Green | Collapsed |
| 2 | Metrics | MetricsSection | Notion metrics DB via `/api/notion/metrics` | Blue | Collapsed |
| 3 | Guides & Docs | GuidesDocsSection | Notion documents DB | Default | Open |
| 4 | Overview | OverviewSection | Notion project page + milestones DB | Default | Open |
| 5 | Projects & Tasks | ProjectTrackingSection | Notion tasks DB | Default | Open |
| 6 | Weekly Reports | ReportsSection | AI → localStorage | Default | Compact when empty |
| 7 | User Feedback | UserFeedbackSection | Notion feedback DB | Default | Compact when empty |

### Section Pattern

Every section wraps with `DashboardSection`:
- Collapsible card using `@radix-ui/react-collapsible`
- Icon, title, description in trigger
- `keyMetrics` slot — summary stats visible when collapsed
- `detailedContent` slot — full data view when expanded
- `defaultOpen` prop controls initial state
- Mobile-responsive padding: `p-4 sm:p-6`

### Unmounted Components

3 components exist in code but are NOT imported in `main-dashboard.tsx`:

| Component | File | Lines | Purpose |
|-----------|------|-------|---------|
| DriveSection | `drive-section.tsx` | 220 | Google Drive file browser via OAuth session token |
| GitHubSection | `github-section.tsx` | 302 | Repo viewer: commits (5), PRs (10), issues (10) |
| KnowledgeSection | `knowledge-section.tsx` | 73 | Static "Coming soon" placeholder |

---

## Dual Metrics System

Two tracking dimensions share the same API route (`/api/notion/metrics`) but query different databases:

**Goals (outputs)** — results achieved:
- Database: `NEXT_PUBLIC_NOTION_DB_GOALS`
- Green color scheme (`bg-green-900/30`, `border-green-500/50`, `ring-green-400`)
- Examples: # of sales, # of subscribers, # of Amazon reviews

**Metrics (inputs)** — actions taken:
- Database: `NEXT_PUBLIC_NOTION_DB_METRICS`
- Blue color scheme (`bg-blue-900/30`, `border-blue-500/50`, `ring-blue-400`)
- Examples: # of posts, # of interactions, marketing spend

Both sections share identical architecture:
- `normalizeMetricType()` — lowercase, strips accents via `normalize("NFD").replace()`
- Clickable metric cards with selected state (ring highlight + darker background)
- Recharts `LineChart` with `ResponsiveContainer`
- Group by type, sort by date, display latest value per type
- Add dialog with type, value, date fields

---

## Kanban Board

`ProjectTrackingSection` provides 5-column task management:

```
To Do → In Progress → Review → Done → Completed
```

- Fetches from Notion tasks DB via `/api/notion/tasks`
- Status filter buttons for each column
- Task creation dialog with:
  - Title (required)
  - Assignee (text)
  - Status (select)
  - Priority (select: Low/Medium/High/Urgent)
  - Due date (react-day-picker in Popover)
  - Tags (text, comma-separated)
- Priority colors: urgent=red, high=orange, medium=yellow, low=blue
- `keyMetrics` shows count of in-progress tasks

---

## AI Report Generation

`ReportsSection` generates weekly progress reports:

1. Fetches current tasks from Notion
2. Builds a task summary message
3. POSTs to `/api/ai/chat` with `provider: "openai"`
4. Receives markdown report
5. Stores in `localStorage` under key `'minivault_weekly_reports'`
6. Delete with confirmation dialog

Reports are **not persisted** to any database — localStorage only.

---

## Guides & Docs

`GuidesDocsSection` manages documentation links:

- Full CRUD via Notion documents DB
- 13 supported link types: notion, drive, github, slack, figma, jira, confluence, trello, asana, miro, docs, api, other
- Auto-syncs on initialization:
  - Fetches Google Drive files → creates Notion entries for new URLs
  - Fetches GitHub repo info → creates Notion entry if URL not yet tracked
  - Deduplicates by URL, deletes Notion duplicates
- Card grid: 2-5 columns responsive (`grid-cols-2 md:grid-cols-3 lg:grid-cols-4 xl:grid-cols-5`)
- Edit/Delete per card

---

## Overview Section

`OverviewSection` combines two data sources:

1. **Project description + vision** — reads Notion page blocks via `/api/notion/project-overview`:
   - Scans for `heading_2` or `heading_3` blocks titled "Description" and "Vision"
   - Reads the `paragraph` block immediately following each heading
   - POST updates existing paragraph or creates new heading + paragraph pair
2. **Milestones** — full CRUD from Notion milestones DB:
   - Name, due date (react-day-picker + date-fns), description, percentage (number)
   - Percentage field discovery: 11 name variants (`percentage`, `taux`, `completion`, `pourcentage`, `taux de completion`, `avancement`, `progress`, `progression`, `complete`, `done`, `%`) searched via case-insensitive exact match then partial match
   - Progress bar visualization

---

## API Routes

### Notion Routes

All routes use raw `fetch()` with Notion REST API v2022-06-28. All mark `force-dynamic` and `revalidate = 0`. Every POST/PATCH route queries the database schema first to discover property names and types.

| Route | Methods | Schema Discovery | Key Details |
|-------|---------|-----------------|-------------|
| `/api/notion/tasks` | GET, POST | Title: 4 variants, Assignee: 4 variants (incl. "Assignée"), Status: 2, Due Date: 3, Priority: 2, Tags: 2 | Status reads `status.name` → `select.name` → `rich_text`. Priority written as `rich_text`. Tags as comma-separated `rich_text`. Fallback: `NOTION_DATABASE_ID` env var for GET. |
| `/api/notion/metrics` | GET, POST | Name: 3 variants ("Metric Name", "Name", "name"), Number: 2 ("Number", "number") | Number property adapts: if schema type is `multi_select` → `multi_select`, if `rich_text` → `rich_text`, else → `number`. Date written as "Last Updated". |
| `/api/notion/goals` | GET, POST | Name: 4 variants, Category: 2, Current Progress: 3, Deadline: 3, Status: 2, Target: 2 | All non-title fields written as `rich_text`. |
| `/api/notion/documents` | GET, POST, PATCH, DELETE | Title: 2 ("Title", "Name"), URL: 2 ("URL", "Link"), Type: 2 ("Type", "Category") | Type adapts: if schema type is `rich_text` → write as `rich_text`, else → `select`. PATCH fetches page to find parent database then queries schema. DELETE archives. |
| `/api/notion/feedback` | GET, POST, PATCH, DELETE | Title: 4, Date: 3, Feedback: 4 ("Feedback", "feedback", "Message", "message"), User Name: 4 | PATCH uses page property introspection — finds each property by reference equality with `Object.keys().find()`. |
| `/api/notion/milestones` | GET, POST, PATCH, DELETE | Percentage: 11 case-insensitive variants | Sorted by "Due Date" ascending. PATCH also queries parent database schema. DELETE archives. |
| `/api/notion/project-overview` | GET, POST | N/A (page blocks, not database) | Reads children blocks of a page. Looks for heading_2/3 "Description"/"Vision" + following paragraph. POST updates existing paragraph or appends heading_3 + paragraph. |

### AI Route

**`/api/ai/chat`** — POST

- Body: `{ message, provider?, model? }`
- Provider `"openai"` (default): `gpt-4-turbo-preview`, `openai.chat.completions.create()`
- Provider `"anthropic"`: `claude-3-5-sonnet-20241022`, `anthropic.messages.create()` with `max_tokens: 1024`
- Returns `{ response, provider, model }`

### Google Routes

| Route | Auth | Details |
|-------|------|---------|
| `/api/drive/files` | OAuth session token | Lists files in a folder. Returns: folder metadata + files with type detection (`isFolder`, `isDocument`, `isSpreadsheet`, `isPresentation`). 50 results, sorted by `modifiedTime desc`. |
| `/api/google/drive` | Service account JWT | Alternative Drive access via `googleapis`. Supports `maxResults` and `query` params. |
| `/api/google/gmail` | Service account JWT | Lists messages via `gmail.users.messages.list()`. Fetches full details per message: subject, from, date, snippet. |

### Other Routes

| Route | Details |
|-------|---------|
| `/api/auth/[...nextauth]` | Imports `authOptions` from `lib/auth.ts`. Logs env var presence at startup. |
| `/api/github/repo` | Requires GitHub OAuth session. Fetches: repo metadata + 5 latest commits + 10 open issues + 10 open PRs. |
| `/api/feedback/save` | Saves feedback as `.txt` files to `data/feedbacks/`. Filename: `YYYY-MM-DD_sanitized_title.txt`. Content: Title, Date, User, Feedback, Created timestamp. |

---

## Authentication Flow

```
User → /auth/signin → Click Google or GitHub button
  → NextAuth redirects to OAuth provider
  → Provider returns authorization code
  → JWT callback: stores access_token, refresh_token, provider in JWT
  → Session callback: exposes tokens to client via session object
  → MainDashboard: checks session → redirects to /auth/signin if absent
  → API routes: use getServerSession(authOptions) for authenticated requests
```

**Google OAuth scopes**: openid, email, profile, gmail.readonly, drive.readonly, documents.readonly
**GitHub OAuth scopes**: read:user, user:email, repo

**Session extension** (`types/next-auth.d.ts`):
```typescript
interface Session {
  accessToken?: string
  refreshToken?: string
  provider?: string
}
```

**Two Google Drive implementations coexist**:
1. OAuth-based (`/api/drive/files`) — uses user's access token from NextAuth session
2. Service-account-based (`/api/google/drive`) — uses `GOOGLE_SERVICE_ACCOUNT_KEY` JWT

---

## Setup Script (`setup-project.py`)

Python 3 script that bootstraps a new project with zero manual Notion setup:

### How It Works

1. Reads embedded credentials (NOTION_TOKEN, GOOGLE_*, GITHUB_*, OPENAI_API_KEY, ANTHROPIC_API_KEY, NEXTAUTH_SECRET)
2. Creates a parent Notion page with the project name as title
3. Creates 6 child databases with predefined schemas:

| Database | Properties |
|----------|-----------|
| Tasks | Name (title), Status (select: To Do/In Progress/Review/Done/Completed), Priority (select: Low/Medium/High/Urgent), Assignée (rich_text), Due Date (date), Tags (multi_select) |
| Goals | Name (title), Category (select), Current Progress (number), Target (number), Deadline (date), Status (select: Not Started/In Progress/Completed) |
| Metrics | Metric Name (title), Number (number), Last Updated (date) |
| Milestones | Name (title), Due Date (date), Description (rich_text), Taux de completion (number, format: percent) |
| Documents | Name (title), URL (url), Description (rich_text), Type (select: Notion/Google Drive/GitHub/Slack/Figma/API/Docs/Other) |
| Feedback | Title (title), Date (date), Feedback (rich_text), User Name (rich_text) |

4. Generates `.env.local` with all database IDs and credentials
5. Uses only `urllib.request` (no pip dependencies)

### Usage

```bash
python3 setup-project.py "Project Name"
```

---

## Project Configuration

All settings via environment variables. `lib/project-config.ts` provides:

- **`ProjectConfig`** interface — typed access to all `NEXT_PUBLIC_*` vars
- **`getProjectConfig()`** — returns config object (works server + client)
- **`useProjectConfig()`** — React hook wrapper

Supported databases in config: tasks, goals, metrics, milestones, documents, feedback, **sales**, **customMetrics** (last two are template extras not present in all forks).

---

## Notion Database Schemas

Expected property types (routes handle multiple naming variants):

**Tasks DB**

| Property | Type | Variants Checked |
|----------|------|-----------------|
| Name | title | Name, Title, title, name |
| Assignee | rich_text | Assignée, Assignee, Assigned, assignee |
| Status | status/select/rich_text | Status, status |
| Priority | select/rich_text | Priority, priority |
| Due Date | date | Due Date, DueDate, dueDate |
| Tags | multi_select/rich_text | Tags, tags |

**Metrics DB**

| Property | Type | Variants Checked |
|----------|------|-----------------|
| Metric Name | title | Metric Name, Name, name |
| Number | number/multi_select/rich_text | Number, number |
| Last Updated | date | Last Updated, Date, date |

**Goals DB**

| Property | Type | Variants Checked |
|----------|------|-----------------|
| Name | title | Name, Title, name, title |
| Category | rich_text/select | Category, category |
| Current Progress | rich_text/number | Current Progress, Current, current |
| Deadline | date | Deadline, deadline, Due Date |
| Status | status/select/rich_text | Status, status |
| Target | rich_text/number | Target, target |

**Documents DB**

| Property | Type | Variants Checked |
|----------|------|-----------------|
| Name | title | Name, Title |
| URL | url | URL, Link |
| Description | rich_text | Description |
| Type | select/rich_text | Type, Category |

**Feedback DB**

| Property | Type | Variants Checked |
|----------|------|-----------------|
| Title | title | Title, title, Name, name |
| Date | date | Date, date, Created Date |
| Feedback | rich_text | Feedback, feedback, Message, message |
| User Name | rich_text | User Name, userName, User, user |

**Milestones DB**

| Property | Type | Variants Checked |
|----------|------|-----------------|
| Name | title | Name |
| Due Date | date | Due Date |
| Description | rich_text | Description |
| Percentage | number | 11 variants: percentage, taux, completion, pourcentage, taux de completion, avancement, progress, progression, complete, done, % — searched case-insensitive (exact match then partial match) |

---

## UI Components

11 shadcn/ui primitives in `components/ui/`:

| Component | Base | Key Features |
|-----------|------|-------------|
| Badge | `class-variance-authority` | 4 variants: default, secondary, destructive, outline. Rounded-full. |
| Button | `@radix-ui/react-slot` + cva | 6 variants (default/destructive/outline/secondary/ghost/link), 4 sizes (default/sm/lg/icon). `asChild` prop via Slot. |
| Calendar | `react-day-picker` | Custom `CalendarDayButton` with range selection support. `--cell-size: 2rem` CSS var. RTL support. |
| Card | Native div | Card/Header/Title/Description/Content/Footer sub-components. |
| Collapsible | `@radix-ui/react-collapsible` | Direct re-export of Root, Trigger, Content. |
| Dialog | `@radix-ui/react-dialog` | Animated overlay (fade + zoom), close button (X icon), Header/Footer/Title/Description. |
| Input | Native input | Standard styled input with ring focus. |
| Label | `@radix-ui/react-label` + cva | Peer-disabled styling. |
| Popover | `@radix-ui/react-popover` | Portal-based, transform-origin aware. |
| Select | `@radix-ui/react-select` | Full select with scroll up/down buttons, check indicator, separator. |
| Textarea | Native textarea | min-h-[80px], responsive text (text-base → md:text-sm). |

---

## Internal Documentation Files

The template ships with extensive internal documentation:

| File | Size | Purpose |
|------|------|---------|
| `CLAUDE.md` | 351 lines | AI assistant reference — architecture, auth flow, API routes, component patterns, UX patterns, troubleshooting |
| `PROJECT.md` | 63.4 KB | Exhaustive project documentation — table of contents, data flow diagrams, every component and route |
| `API.md` | 500 lines | API route documentation — endpoints, auth methods, error handling, rate limiting |

---

## Relationship to Other Projects

This template is the **source boilerplate** from which project-specific dashboards are cloned:

| Project | Repo | Relationship |
|---------|------|-------------|
| **Present Agent** (this) | `Present-Agent` | Template/boilerplate with setup script |
| **Founder MTL** | `FounderMTL` | Fork/clone deployed for a specific project |

Key differences between template and forks:
- Template has `setup-project.py` for automated Notion bootstrapping
- Template has extra database support (`sales`, `customMetrics` in ProjectConfig)
- Template ships with `CLAUDE.md`, `PROJECT.md`, `API.md` internal docs
- Template has `data/feedbacks/` directory with README explaining file format
- Template has `vercel.json` (iad1 region)
- Forks may have custom section configurations or additional mounted components

---

## Environment Variables

```env
# Authentication
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=

# OAuth Providers
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GITHUB_ID=
GITHUB_SECRET=

# APIs
NOTION_TOKEN=
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
GOOGLE_SERVICE_ACCOUNT_KEY=  # single-line JSON

# Project (NEXT_PUBLIC_ = client-accessible)
NEXT_PUBLIC_PROJECT_NAME=
NEXT_PUBLIC_NOTION_DB_TASKS=
NEXT_PUBLIC_NOTION_DB_GOALS=
NEXT_PUBLIC_NOTION_DB_METRICS=
NEXT_PUBLIC_NOTION_DB_MILESTONES=
NEXT_PUBLIC_NOTION_DB_DOCUMENTS=
NEXT_PUBLIC_NOTION_DB_FEEDBACK=
NEXT_PUBLIC_NOTION_DB_SALES=           # Template extra
NEXT_PUBLIC_NOTION_DB_CUSTOM_METRICS=  # Template extra
NEXT_PUBLIC_NOTION_PROJECT_PAGE_ID=
NEXT_PUBLIC_GITHUB_OWNER=
NEXT_PUBLIC_GITHUB_REPO=
NEXT_PUBLIC_GOOGLE_DRIVE_FOLDER_ID=
```

---

## Setup

### Prerequisites

- Node.js 18+
- Python 3 (for setup script)
- Notion integration with workspace access
- Google Cloud project (Gmail, Drive APIs enabled)
- GitHub OAuth App
- OpenAI API key (for reports)

### Installation

```bash
# Clone template
git clone git@github.com:yasser-ensembl3/Present-Agent.git my-project
cd my-project

# Run automated setup
python3 setup-project.py "My Project"

# Install and start
npm install
npm run dev
# Open http://localhost:3000
```

### OAuth Callback URLs

| Provider | URL |
|----------|-----|
| Google | `http://localhost:3000/api/auth/callback/google` |
| GitHub | `http://localhost:3000/api/auth/callback/github` |

### Vercel Deployment

1. Push to GitHub
2. Import in Vercel
3. Add all environment variables
4. Update `NEXTAUTH_URL` to production URL
5. Update OAuth callback URLs to production domain

---

## Build Configuration

**`next.config.js`**:
- `swcMinify: true`
- `removeConsole: true` (production only — strips all `console.log` calls)
- `optimizePackageImports: ["lucide-react", "@radix-ui/react-icons"]` — reduces first compilation from ~45s to ~1-2s

**`tailwind.config.ts`**:
- `darkMode: ["class"]`
- HSL CSS variable-based color system
- `tailwindcss-animate` plugin
- Container: centered, `2rem` padding, max `1400px`

**`components.json`** (shadcn/ui):
- Style: default
- Base color: slate
- CSS variables: enabled
- RSC: true

**`vercel.json`**: region `iad1`

**`tsconfig.json`**: ES2017 target, bundler module resolution, `@/*` path alias

---

## Design Decisions

1. **Template-first** — designed to be cloned with automated setup, not used as a single instance.
2. **One repo = one project** — no multi-project support, no project switching UI. Each clone is dedicated to a single project.
3. **Schema-adaptive Notion** — every POST route queries the database schema before writing. Supports English and French property name variants. Adapts write format to actual property type (select vs rich_text vs number vs multi_select).
4. **Raw fetch over SDK** — `@notionhq/client` is installed but unused. All Notion calls use raw `fetch()` for maximum control.
5. **Dark mode locked** — `<html className="dark">` in layout, no toggle.
6. **No database** — all persistent data in Notion databases or localStorage (reports). No SQL/Prisma.
7. **Dual Google Drive** — OAuth-based (user token from session) and service-account-based (JWT) implementations coexist.
8. **Console logging stripped in prod** — extensive `[ComponentName]` prefixed debug logs removed by `next.config.js` `removeConsole: true`.
9. **Feedback to disk** — in addition to Notion, feedback can be saved as `.txt` files via `/api/feedback/save`.
10. **Internal docs** — ships with CLAUDE.md, PROJECT.md, API.md for AI assistant and developer reference.

---

## Credentials

| Service | Type | Purpose |
|---------|------|---------|
| Google | OAuth2 (client ID + secret) | User auth, Drive (OAuth), Gmail (OAuth) |
| Google | Service account (JSON key) | Gmail + Drive API (server-side, no user auth) |
| GitHub | OAuth App (ID + secret) | User auth, repo access |
| Notion | Internal integration token | All database operations + page block access |
| OpenAI | API key | AI report generation (gpt-4-turbo-preview) |
| Anthropic | API key | AI chat alternative (claude-3-5-sonnet) |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "No access token" | Sign in with the correct provider (Google for Drive/Gmail, GitHub for repos) |
| Notion API 401 | Check `NOTION_TOKEN`, ensure integration has database access |
| Empty sections | Verify the corresponding `NEXT_PUBLIC_NOTION_DB_*` env var is set |
| OAuth callback error | Check callback URLs match exactly in OAuth app settings |
| Milestones percentage missing | Percentage property name may not match any of the 11 searched variants |
| AI reports fail | Check `OPENAI_API_KEY` is set and has credits |
| GOOGLE_SERVICE_ACCOUNT_KEY error | Must be single-line JSON (no unescaped newlines) |
| Slow first compilation | `optimizePackageImports` in next.config.js should handle this automatically |
| setup-project.py fails | Ensure `NOTION_TOKEN` has workspace access and Python 3 is installed |
