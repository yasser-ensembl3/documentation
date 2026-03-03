# Founders Graph — Process Documentation

## Overview

Multi-source web scraping and enrichment pipeline for building comprehensive founder profiles. Aggregates data from articles, blogs, podcasts, YouTube videos, LinkedIn, and press mentions into structured Markdown profiles with LLM-powered relevance filtering and synthesis.

**Repo**: [yasser-ensembl3/Social-graph](https://github.com/yasser-ensembl3/Social-graph)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Python 3.10+ |
| Data models | Pydantic |
| Semantic search | Exa.ai |
| Content extraction | Jina Reader API |
| Video search | YouTube Data API v3 |
| Web search | Google Custom Search API |
| Podcast search | Listen Notes + Podcast Index |
| LinkedIn scraping | Phantombuster API |
| LLM (filtering) | OpenAI GPT-4o-mini |
| LLM (synthesis) | OpenAI GPT-4o / Anthropic Claude Sonnet |

---

## Architecture

```
founders-graph/
├── scripts/
│   ├── scrapers/
│   │   ├── apis/
│   │   │   ├── founder_scraper.py      # Unified scraper (orchestrates all sources)
│   │   │   ├── exa.py                  # Exa.ai semantic search client
│   │   │   ├── jina.py                 # Jina URL-to-Markdown reader
│   │   │   ├── youtube.py              # YouTube Data API v3 client
│   │   │   ├── google_search.py        # Google Custom Search client
│   │   │   ├── podcasts.py             # Listen Notes + Podcast Index clients
│   │   │   └── content_scraper.py      # Exa + Jina combined pipeline
│   │   └── linkedin/
│   │       └── phantombuster.py        # LinkedIn scraper via Phantombuster API
│   ├── parsers/
│   │   ├── models.py                   # Pydantic data models (FounderProfile, Position, etc.)
│   │   └── linkedin_parser.py          # Parse LinkedIn .md exports into FounderProfile
│   ├── enrichment/
│   │   ├── enrichment_pipeline.py      # Main enrichment orchestrator
│   │   └── relevance_filter.py         # LLM-based content relevance scoring
│   ├── synthesis/
│   │   └── llm_synthesizer.py          # LLM profile generation (OpenAI or Anthropic)
│   └── batch_scrape.py                 # CSV batch processing with resume support
├── data/
│   ├── input/                          # Input data (Phantombuster JSON exports)
│   ├── cache/                          # Raw JSON outputs from scrapers
│   ├── output/                         # Generated Markdown profiles
│   └── enriched/                       # LLM-enriched profiles
├── requirements.txt
└── .env.example
```

---

## Pipeline — How It Works

### Full Flow

```
Input: Founder name + company (or CSV batch)
    │
    ├── 1. Multi-source search (Exa, YouTube, Google, Podcasts)
    ├── 2. Content extraction (Jina Reader)
    ├── 3. LinkedIn scraping (Phantombuster, optional)
    ├── 4. LLM relevance filtering (GPT-4o-mini)
    └── 5. LLM profile synthesis (GPT-4o / Claude Sonnet)
    │
    ▼
Output: Structured Markdown profile + JSON cache
```

### Step 1: Multi-Source Search

The unified scraper (`founder_scraper.py`) orchestrates searches across all configured sources:

| Source | Client | What It Finds |
|--------|--------|---------------|
| Exa.ai | `exa.py` | Articles, blog posts, podcast mentions via semantic search. Auto-excludes social media, categorizes results |
| YouTube | `youtube.py` | Video appearances — runs 4 search queries per founder: interview, podcast, talk, general |
| Google | `google_search.py` | Press mentions, media articles, interviews. Excludes social media, categorizes by type |
| Podcasts | `podcasts.py` | Podcast episodes where founder is guest, host, or mentioned (Listen Notes + Podcast Index) |

### Step 2: Content Extraction

**Module**: `jina.py`

Converts discovered URLs to clean Markdown via the Jina Reader API. Extracts: title, full content, word count, author, publication date. Combined with Exa in `content_scraper.py` for a search-then-extract pipeline.

### Step 3: LinkedIn Enrichment (Optional)

**Module**: `phantombuster.py`

Scrapes LinkedIn profiles and activity (posts with engagement metrics) via Phantombuster API. LinkedIn `.md` exports are parsed into `FounderProfile` objects by `linkedin_parser.py`.

### Step 4: LLM Relevance Filtering

**Module**: `relevance_filter.py`

Each search result is scored by GPT-4o-mini on a 0-100 scale:

| Score | Meaning |
|-------|---------|
| 90-100 | Direct interview or podcast appearance |
| 70-89 | Content created by the person |
| 50-69 | Significant mention |
| Below 50 | Filtered out |

### Step 5: LLM Profile Synthesis

**Module**: `llm_synthesizer.py`

Generates a structured Markdown profile from all collected data. Supports two providers with automatic fallback:

1. **OpenAI GPT-4o** — primary
2. **Anthropic Claude Sonnet** — fallback

Generated sections: executive summary, current position, professional journey, education, expertise areas, media appearances, LinkedIn activity, contact info.

---

## Enrichment Pipeline (Alternative Flow)

The full enrichment pipeline (`enrichment_pipeline.py`) starts from an existing LinkedIn Markdown export:

1. **Parse** LinkedIn .md file → FounderProfile (Pydantic)
2. **Scrape** full LinkedIn profile via Phantombuster
3. **Scrape** LinkedIn activity (posts)
4. **Search** YouTube for video appearances
5. **Search** Google for media coverage
6. **Search** podcast appearances (Listen Notes)
7. **Filter** results via LLM relevance scoring
8. **Synthesize** enriched Markdown profile via LLM

---

## Batch Processing

Process a CSV of founders with resume support:

```bash
python scripts/batch_scrape.py /path/to/founders.csv --max=5 --start=0
```

**CSV format:** `firstName`, `lastName`, `companyName`

Features:
- Skips already-generated profiles
- Resumable with `--start=N`
- macOS notifications every 10 founders
- Progress tracking with ETA
- 1-second delay between requests

---

## Data Model

```python
FounderProfile
├── id, name                           # Identity
├── current_position: Position         # Title, company, duration
├── location, industry                 # Demographics
├── summary, role_description          # Bio
├── linkedin_url                       # LinkedIn
├── experiences: list[Position]        # Career history
├── education: list[Education]         # Schools, degrees
├── skills: list[str]                  # Skill tags
├── linkedin_posts: list[Activity]     # Posts with engagement metrics
├── media_mentions: list[MediaMention] # Press mentions
├── video_appearances: list[Video]     # YouTube/podcast appearances
├── executive_summary: str             # LLM-generated
├── expertise_areas: list[str]         # LLM-generated
└── enriched_at, sources_used          # Metadata
```

---

## Output

Each founder generates:

1. **Markdown profile** (`data/output/{name}.md`):
   - Summary table with source counts
   - Articles & blog posts (links, dates, categories)
   - Full scraped content (text, word counts)
   - YouTube videos (titles, channels, dates)
   - Press mentions

2. **JSON cache** (`data/cache/{name}_raw.json`): raw API responses for reprocessing

3. **Enriched profile** (`data/enriched/{name}.md`): LLM-synthesized version with executive summary and structured sections

---

## Installation & Setup

### Prerequisites

- Python 3.10+
- API keys for Exa.ai, YouTube, Google Custom Search, OpenAI (minimum)

### Installation

```bash
pip install -r requirements.txt
cp .env.example .env.local
# Edit .env.local with your API keys
```

### Environment Variables

| Variable | Required | Purpose |
|----------|----------|---------|
| `EXA_API_KEY` | Yes | Exa.ai semantic search |
| `YOUTUBE_API_KEY` | Yes | YouTube video search |
| `GOOGLE_API_KEY` | Yes | Google Custom Search |
| `GOOGLE_SEARCH_ENGINE_ID` | Yes | Google CSE ID |
| `OPENAI_API_KEY` | Yes | LLM synthesis + relevance filtering |
| `JINA_API_KEY` | No | Jina Reader (works without, with limits) |
| `ANTHROPIC_API_KEY` | No | Alternative LLM provider |
| `PHANTOMBUSTER_API_KEY` | No | LinkedIn scraping |
| `LISTENNOTES_API_KEY` | No | Podcast search |

### API Rate Limits

| API | Free Tier |
|-----|-----------|
| Exa.ai | 1,000 req/month |
| Jina Reader | 1M credits/month |
| YouTube Data | 10,000 req/day |
| Google Custom Search | 100 req/day |
| Phantombuster | Paid only |

---

## Usage

```bash
# Single founder
python -m scripts.scrapers.apis.founder_scraper "Sam Altman" "OpenAI" --max=5

# Batch from CSV
python scripts/batch_scrape.py founders.csv --max=5

# Full enrichment (from LinkedIn .md)
python -m scripts.enrichment.enrichment_pipeline input.md output.md
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Exa returns no results | Check `EXA_API_KEY` validity, verify monthly quota |
| YouTube quota exceeded | Free tier is 10,000 req/day — wait or use a new key |
| Google Search 429 | Free tier is 100 req/day — very limited, batch accordingly |
| Jina extraction fails | Some sites block Jina — content will be skipped |
| Phantombuster errors | Ensure LinkedIn agent is configured in Phantombuster dashboard |
| LLM synthesis empty | Check `OPENAI_API_KEY`, both providers down triggers skip |
| Batch stalls | Check API rate limits, use `--start=N` to resume |
