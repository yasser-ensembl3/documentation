# Founder MTL (MiniVault) — Process Documentation

## Overview

Unified project management dashboard built with Next.js 14. Integrates Notion databases (tasks, goals, metrics, milestones, documents, feedback), Google Drive, Gmail, GitHub, and AI-powered report generation into a single dark-themed interface. Follows a **one repo = one project** architecture — each deployment is dedicated to a single project, configured entirely via environment variables.

**Repo**: [yasser-ensembl3/FounderMTL](https://github.com/yasser-ensembl3/FounderMTL)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Framework | Next.js 14 (App Router) |
| Language | TypeScript |
| UI | shadcn/ui + Tailwind CSS (dark mode locked) |
| Authentication | NextAuth.js (Google + GitHub OAuth) |
| Data layer | Notion API (raw fetch, not SDK) |
| AI | OpenAI (gpt-4-turbo-preview) + Anthropic (claude-3-5-sonnet) |
| Google APIs | Gmail + Drive via service account JWT |
| Charts | Recharts |
| Deployment | Vercel (iad1 region) |

---

## Architecture

```
Founder MTL/
├── app/
│   ├── api/
│   │   ├── auth/[...nextauth]/route.ts    # NextAuth handler
│   │   ├── notion/
│   │   │   ├── tasks/route.ts             # Tasks CRUD
│   │   │   ├── metrics/route.ts           # Input metrics CRUD
│   │   │   ├── goals/route.ts             # Output goals CRUD
│   │   │   ├── milestones/route.ts        # Milestones CRUD
│   │   │   ├── documents/route.ts         # Documentation links CRUD
│   │   │   ├── feedback/route.ts          # User feedback CRUD
│   │   │   └── project-overview/route.ts  # Project description + vision
│   │   ├── drive/files/route.ts           # Google Drive (OAuth)
│   │   ├── github/repo/route.ts           # GitHub repo info
│   │   ├── google/
│   │   │   ├── drive/route.ts             # Google Drive (service account)
│   │   │   └── gmail/route.ts             # Gmail (service account)
│   │   ├── ai/chat/route.ts              # AI chat (OpenAI / Anthropic)
│   │   └── feedback/save/route.ts         # Save feedback to disk
│   ├── auth/signin/page.tsx               # Sign-in page
│   ├── layout.tsx                         # Root layout (dark mode, AuthProvider)
│   ├── page.tsx                           # Home → MainDashboard
│   └── globals.css                        # CSS variables
├── components/
│   ├── auth/session-provider.tsx           # NextAuth SessionProvider wrapper
│   ├── dashboard/
│   │   ├── main-dashboard.tsx             # Orchestrator — renders all sections
│   │   ├── header.tsx                     # Project name, user info, sign out
│   │   ├── dashboard-section.tsx          # Reusable collapsible card wrapper
│   │   ├── goals-metrics-section.tsx      # Output metrics (green) + charts
│   │   ├── metrics-section.tsx            # Input metrics (blue) + charts
│   │   ├── guides-docs-section.tsx        # Documentation links + auto-sync
│   │   ├── overview-section.tsx           # Description, vision, milestones
│   │   ├── project-tracking-section.tsx   # Kanban board (tasks)
│   │   ├── reports-section.tsx            # AI weekly reports
│   │   ├── user-feedback-section.tsx      # Feedback CRUD
│   │   ├── drive-section.tsx              # Google Drive browser (not mounted)
│   │   ├── github-section.tsx             # GitHub viewer (not mounted)
│   │   └── knowledge-section.tsx          # Placeholder (not mounted)
│   └── ui/                                # shadcn/ui primitives
├── lib/
│   ├── auth.ts                            # NextAuth config
│   ├── project-config.ts                  # Env-based project config
│   └── utils.ts                           # cn() utility
└── types/
    └── next-auth.d.ts                     # Session type extensions
```

---

## Dashboard Sections

Rendered in priority order in `main-dashboard.tsx`:

| # | Section | Component | Data Source | Theme | Default |
|---|---------|-----------|------------|-------|---------|
| 1 | Goals | GoalsMetricsSection | Notion goals DB | Green | Collapsed |
| 2 | Metrics | MetricsSection | Notion metrics DB | Blue | Collapsed |
| 3 | Guides & Docs | GuidesDocsSection | Notion documents DB | Default | Open |
| 4 | Overview | OverviewSection | Notion project page + milestones DB | Default | Open |
| 5 | Projects & Tasks | ProjectTrackingSection | Notion tasks DB | Default | Open |
| 6 | Weekly Reports | ReportsSection | AI → localStorage | Default | Compact when empty |
| 7 | User Feedback | UserFeedbackSection | Notion feedback DB | Default | Compact when empty |

### Section Pattern

Every section uses the `DashboardSection` wrapper component:
- Collapsible card with icon, title, description
- `keyMetrics` slot — summary visible when collapsed
- `detailedContent` slot — expanded view with full data
- Mobile-responsive padding and text sizing

---

## Dual Metrics System

The app separates tracking into inputs and outputs:

**Goals (outputs)** — results achieved:
- Examples: sales count, subscribers, Amazon reviews
- Green color scheme
- Database: `NEXT_PUBLIC_NOTION_DB_GOALS`

**Metrics (inputs)** — actions taken:
- Examples: posts published, interactions, marketing spend
- Blue color scheme
- Database: `NEXT_PUBLIC_NOTION_DB_METRICS`

Both share the same API endpoint (`/api/notion/metrics`) but query different Notion databases. Both feature:
- Clickable metric cards with selected state (ring highlight)
- Line charts via Recharts
- Date-based tracking with most recent values
- Normalized metric names (lowercase, no accents)
- Add dialog for new entries

---

## Kanban Board

`ProjectTrackingSection` provides 5-column task management:

```
To Do → In Progress → Review → Done → Completed
```

- Fetches from Notion tasks DB via `/api/notion/tasks`
- Status filter dropdown
- Task creation dialog with calendar date picker
- Priority colors (urgent, high, medium, low)

---

## AI Report Generation

`ReportsSection` generates weekly reports:

1. Fetches current tasks from Notion
2. POSTs task summary to `/api/ai/chat`
3. API supports dual providers: OpenAI (`gpt-4-turbo-preview`) or Anthropic (`claude-3-5-sonnet`)
4. Returns markdown report
5. Stored in localStorage (not persisted to any database)

---

## Guides & Docs

`GuidesDocsSection` manages documentation links:

- Full CRUD via Notion documents DB
- Auto-syncs Google Drive and GitHub links to Notion (deduplicates by URL)
- Card grid layout with type badges
- Each card: title, description, type, edit/delete actions

---

## Overview Section

`OverviewSection` combines two data sources:

1. **Project description + vision** — read/write from Notion page blocks (heading_2/3 "Description" and "Vision" followed by paragraph blocks)
2. **Milestones** — CRUD from Notion milestones DB with percentage tracking and calendar picker

---

## API Routes

### Notion Routes

All use `NOTION_TOKEN` from env. All use raw `fetch()` with Notion API v2022-06-28 (not @notionhq/client SDK). Every route dynamically discovers property names by querying the database schema first.

| Route | Methods | Purpose |
|-------|---------|---------|
| `/api/notion/tasks` | GET, POST | Task CRUD — title, assignee, status, priority, due date, tags |
| `/api/notion/metrics` | GET, POST | Metric entries — type (name), value (number), date |
| `/api/notion/goals` | GET, POST | Goal entries — name, category, progress, deadline, status, target |
| `/api/notion/documents` | GET, POST, PATCH, DELETE | Documentation links — title, URL, description, type |
| `/api/notion/feedback` | GET, POST, PATCH, DELETE | User feedback — title, date, content, user name |
| `/api/notion/milestones` | GET, POST, PATCH, DELETE | Milestones — name, due date, description, percentage |
| `/api/notion/project-overview` | GET, POST | Project page blocks — description + vision text |

**Schema-adaptive**: Each Notion route checks multiple property name variants. For example, the milestones route searches 12+ possible names for the percentage field: "percentage", "Percentage", "taux", "Taux de completion", "completion", "progress", etc.

### Google Routes

| Route | Auth Method | Purpose |
|-------|-------------|---------|
| `/api/drive/files` | OAuth session token | Browse Google Drive folder |
| `/api/google/drive` | Service account JWT | Alternative Drive access |
| `/api/google/gmail` | Service account JWT | Read Gmail messages |

Two independent Google Drive implementations exist — OAuth-based (user's token from NextAuth session) and service-account-based (GOOGLE_SERVICE_ACCOUNT_KEY).

### Other Routes

| Route | Purpose |
|-------|---------|
| `/api/auth/[...nextauth]` | NextAuth OAuth handler (Google + GitHub) |
| `/api/github/repo` | Fetch repo info, commits (5), issues (10), PRs (10) |
| `/api/ai/chat` | Dual AI provider — OpenAI or Anthropic |
| `/api/feedback/save` | Save feedback as text files to `data/feedbacks/` on disk |

---

## Authentication Flow

```
User → /auth/signin → Click Google or GitHub
  → NextAuth redirects to OAuth provider
  → Provider returns authorization code
  → NextAuth exchanges for access_token + refresh_token
  → Tokens stored in JWT (server-side)
  → Session callback exposes tokens to client
  → MainDashboard checks session → redirects to /auth/signin if absent
  → API routes use getServerSession(authOptions) for authenticated requests
```

**Google OAuth scopes**: openid, email, profile, gmail.readonly, drive.readonly, documents.readonly

**GitHub OAuth scopes**: read:user, user:email, repo

---

## Notion Database Schemas

Expected property types (the API routes handle multiple naming variants):

**Tasks DB**

| Property | Type | Variants Checked |
|----------|------|-----------------|
| Name | title | Name, Title, Nom |
| Assignee | rich_text | Assignee, Assigned, assignee |
| Status | status/select | Status, Statut |
| Priority | select | Priority, Priorité |
| Due Date | date | Due Date, Date, Deadline |
| Tags | multi_select | Tags, Labels |

**Metrics / Goals DB**

| Property | Type |
|----------|------|
| Metric Name | title |
| Number | number |
| Date | date |

**Milestones DB**

| Property | Type |
|----------|------|
| Name | title |
| Due Date | date |
| Description | rich_text |
| Percentage | number (12+ name variants) |

**Documents DB**

| Property | Type |
|----------|------|
| Name | title |
| URL | url |
| Description | rich_text |
| Type | select/rich_text |

**Feedback DB**

| Property | Type |
|----------|------|
| Title | title |
| Date | date |
| Feedback | rich_text |
| User Name | rich_text |

---

## Project Configuration

All settings via environment variables in `.env.local`. The `lib/project-config.ts` module reads all `NEXT_PUBLIC_*` vars into a typed `ProjectConfig` object, accessible via `useProjectConfig()` hook or `getProjectConfig()` function.

```env
# Authentication
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=

# OAuth
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
NEXT_PUBLIC_PROJECT_NAME=Founder MTL
NEXT_PUBLIC_NOTION_DB_TASKS=
NEXT_PUBLIC_NOTION_DB_GOALS=
NEXT_PUBLIC_NOTION_DB_METRICS=
NEXT_PUBLIC_NOTION_DB_MILESTONES=
NEXT_PUBLIC_NOTION_DB_DOCUMENTS=
NEXT_PUBLIC_NOTION_DB_FEEDBACK=
NEXT_PUBLIC_NOTION_PROJECT_PAGE_ID=
NEXT_PUBLIC_GITHUB_OWNER=
NEXT_PUBLIC_GITHUB_REPO=
NEXT_PUBLIC_GOOGLE_DRIVE_FOLDER_ID=
```

---

## Design Decisions

1. **One repo = one project** — no multi-project support, no project switching UI. Each deployment serves one project.
2. **Schema-adaptive Notion** — dynamically discovers property names, handles French and English variants, tolerates schema differences across databases.
3. **Raw fetch over SDK** — `@notionhq/client` is installed but unused. All Notion calls use raw `fetch()`.
4. **Dark mode locked** — `<html className="dark">` in layout, no toggle.
5. **No database** — all persistent data in Notion databases or localStorage (reports). No SQL/Prisma.
6. **Dual Google Drive access** — OAuth-based (user token) and service-account-based (JWT) coexist.
7. **Console logging removed in prod** — extensive `[ComponentName]` prefixed logs, stripped by `next.config.js` `removeConsole: true`.

---

## Unmounted Components

Three components exist but are NOT imported in `main-dashboard.tsx`:

| Component | File | Purpose |
|-----------|------|---------|
| DriveSection | `drive-section.tsx` | Google Drive file browser (OAuth) |
| GitHubSection | `github-section.tsx` | GitHub repo viewer (commits, PRs, issues) |
| KnowledgeSection | `knowledge-section.tsx` | Static placeholder |

To enable: import and add to the sections array in `main-dashboard.tsx`.

---

## Setup

### Prerequisites

- Node.js 18+
- Google Cloud project (Gmail, Drive, Docs APIs enabled)
- GitHub OAuth App
- Notion integration with 6 databases shared
- OpenAI API key (for reports)

### Installation

```bash
npm install
cp .env.example .env.local
# Fill all credentials
npm run dev
# Open http://localhost:3000
```

### OAuth Callback URLs

| Provider | URL |
|----------|-----|
| Google | `http://localhost:3000/api/auth/callback/google` |
| GitHub | `http://localhost:3000/api/auth/callback/github` |

### Notion Setup

1. Create integration at notion.so/my-integrations
2. Share each of the 6 databases with the integration
3. Copy database IDs from URLs: `notion.so/<workspace>/<DATABASE_ID>?v=...`
4. Add to `.env.local`

### Vercel Deployment

1. Push to GitHub
2. Import in Vercel
3. Add all environment variables
4. Update OAuth callback URLs to production domain

---

## Credentials

| Service | Type | Purpose |
|---------|------|---------|
| Google | OAuth2 (client ID + secret) | User authentication, Drive, Gmail |
| Google | Service account (JSON key) | Gmail + Drive API (server-side) |
| GitHub | OAuth App (ID + secret) | User authentication, repo access |
| Notion | Internal integration token | All database operations |
| OpenAI | API key | AI report generation |
| Anthropic | API key | AI chat (alternative provider) |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "No access token" | Sign in with the correct provider (Google for Drive, GitHub for repos) |
| Notion API 401 | Check `NOTION_TOKEN`, ensure integration has database access |
| Empty sections | Verify the corresponding `NEXT_PUBLIC_NOTION_DB_*` env var is set |
| OAuth callback error | Check callback URLs match exactly in OAuth app settings |
| Milestones percentage missing | Notion property name may not match any of the 12+ checked variants |
| AI reports fail | Check `OPENAI_API_KEY` is set and has credits |
| GOOGLE_SERVICE_ACCOUNT_KEY error | Must be single-line JSON (no unescaped newlines) |
