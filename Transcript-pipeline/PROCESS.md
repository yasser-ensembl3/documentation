# Transcript Pipeline — Process Documentation

## Overview

Automated content processing pipeline that extracts transcripts/articles from various sources (YouTube, podcasts, web articles), generates AI summaries via Claude, converts them to audio via Edge TTS, and uploads everything to Google Drive — with Notion as the central database.

**Repo**: [yasser-ensembl3/Audio-transcriber](https://github.com/yasser-ensembl3/Audio-transcriber)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Python 3.10+ |
| LLM | Claude Code CLI (`claude -p`) |
| Text-to-Speech | Edge TTS (free, no API key) |
| Database | Notion API |
| File storage | Google Drive API v3 (OAuth2) |
| Web framework | Flask (webhook server) |
| Article extraction | Trafilatura |
| YouTube transcripts | youtube-transcript-api + yt-dlp |
| Podcast transcription | faster-whisper (local) |

---

## Architecture

```
transcript-pipeline/
├── process_all.py           # Batch processing — all pending Notion entries
├── process_single.py        # Single entry processing (webhook/manual)
├── notion_reader.py         # Notion DB queries, URL type auto-detection
├── notion_updater.py        # Update Notion pages with result links
├── youtube_transcript.py    # YouTube transcript extraction + metadata
├── article_extractor.py     # Web article extraction (Trafilatura)
├── podcast_transcript.py    # Podcast download + Whisper transcription (disabled)
├── summarizer.py            # Claude Code CLI summarization with chunking
├── audio_generator.py       # Edge TTS text-to-speech (async)
├── drive_uploader.py        # Google Drive OAuth2 upload + sharing
├── watcher.py               # Polling daemon (checks Notion every 2 min)
├── webhook_server.py        # Flask webhook server (port 5050)
├── requirements.txt
├── .env.example
└── INSTALL.md
```

---

## Pipeline — How It Works

### Full Flow

```
Notion Database (URL added)
    │
    ▼
Auto-detect content type (YouTube / Article / Podcast)
    │
    ├── 1. Extract transcript or article text
    ├── 2. Summarize with Claude Code CLI
    ├── 3. Generate audio (Edge TTS)
    └── 4. Upload to Google Drive
    │
    ▼
Notion updated with links to summary + audio
```

### Step 1: Content Extraction

Three extraction modules, selected automatically based on URL pattern:

| Source | Module | Method |
|--------|--------|--------|
| YouTube | `youtube_transcript.py` | youtube-transcript-api for transcript (prefers French → English → any), yt-dlp for metadata (title, channel, duration) |
| Web articles | `article_extractor.py` | Trafilatura library for clean text extraction, outputs Markdown with title, author, date |
| Podcasts | `podcast_transcript.py` | yt-dlp to download audio, faster-whisper for local transcription with auto language detection. **Currently disabled in pipeline** |

**Output**: `*_transcript.md` saved in `output/`

### Step 2: Summarization

**Module**: `summarizer.py`

- Calls Claude Code CLI (`claude -p`) as a subprocess — uses existing subscription, zero API cost
- Respects 90K token limit (~360K characters) — automatically chunks oversized content
- Summary length adapts to content type:
  - YouTube/Podcast: ~25% of original (1,500–3,000 words target)
  - Articles: proportional to length (150–1,500 words range)
- Multi-chunk workflow: each chunk summarized independently, then merged into a cohesive final summary via a second Claude call
- Timeout: 300 seconds per chunk

**Output**: `*_summary.md` with metadata header (word count, estimated audio duration)

### Step 3: Audio Generation

**Module**: `audio_generator.py`

- Uses Microsoft Edge TTS — free service, no API key required
- Strips Markdown formatting (headers, metadata) before converting
- Default voice: `en-US-AriaNeural` (female, conversational)
- Other available voices: Guy (male), Jenny (female, clear), Davis (male, calm)
- Async processing via `edge-tts` library

**Output**: `*_audio.mp3`

### Step 4: Upload & Update

**Upload** (`drive_uploader.py`):
- OAuth2 authentication with token caching (`token.pickle`)
- Uploads summary (.md) and audio (.mp3) to configured Drive folder
- Sets public sharing permissions automatically
- Returns shareable web links

**Notion update** (`notion_updater.py`):
- Sets `Text summary` field with Drive link to the .md file
- Sets `Audio summary` field with Drive link to the .mp3 file
- Renames the `Link` field to the content's actual title (auto-extracted)
- Filenames truncated to 100 characters

---

## Notion Database Schema

| Field | Type | Purpose |
|-------|------|---------|
| Link | Title | URL or content title (auto-renamed after processing) |
| Audio Link | URL | Alternative URL field |
| type | Select | Content type: YouTube, Podcast, Article (auto-detected) |
| Text summary | Files | Drive link to generated summary |
| Audio summary | Files | Drive link to generated audio |

---

## Trigger Methods

### 1. Watcher (polling daemon)

Polls Notion every 2 minutes for new entries without summaries. Maintains a `.processed_ids` cache file to avoid reprocessing across restarts.

```bash
# Foreground
python watcher.py

# Background
nohup python -u watcher.py > watcher.log 2>&1 &
```

Can also be configured as a macOS LaunchAgent for auto-start (see INSTALL.md).

### 2. Webhook Server

Flask server on port 5050 for external triggers (n8n, manual API calls). Runs pipeline in background threads (non-blocking).

```bash
python webhook_server.py
```

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/health` | Health check |
| POST | `/webhook/process` | Process specific entry or all pending |
| POST | `/webhook/process-all` | Batch process all pending entries |

**Request format:**
```json
{
  "secret": "your-webhook-secret",
  "page_id": "notion-page-id",
  "url": "content-url"
}
```

Authentication via `secret` field in body or `X-Webhook-Secret` header.

### 3. Manual

```bash
# Process all pending entries
python process_all.py

# Process a single entry
python process_single.py <page_id> <url>
```

---

## Installation & Setup

### Prerequisites

- Python 3.10+
- Node.js (for Claude Code CLI)
- Claude Code CLI installed and authenticated (`claude -p "test"` must work)
- Notion integration token
- Google Drive OAuth credentials

### Installation

```bash
cd transcript-pipeline
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# Edit .env with your credentials
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `NOTION_TOKEN` | Notion integration token (`ntn_xxxxx`) |
| `NOTION_DATABASE_ID` | Target Notion database UUID |
| `GDRIVE_FOLDER_ID` | Google Drive folder ID for uploads |
| `GDRIVE_CREDENTIALS_PATH` | Path to Google OAuth credentials JSON (default: `credentials.json`) |
| `WEBHOOK_SECRET` | Authentication token for webhook endpoints |

### Google Drive OAuth Setup

1. Google Cloud Console → create or select a project
2. Enable Google Drive API
3. Create OAuth 2.0 credentials (Desktop type)
4. Download JSON → save as `credentials.json` at project root
5. First run opens browser for authentication → token cached in `token.pickle`

### Notion Integration Setup

1. Go to [Notion Integrations](https://www.notion.so/my-integrations)
2. Create a new integration with read/write permissions
3. Copy the token → set as `NOTION_TOKEN` in `.env`
4. Share your target database with the integration
5. Copy the database ID from the URL → set as `NOTION_DATABASE_ID`

### Claude Code Path

The `summarizer.py` calls Claude Code at a hardcoded path (`/Users/mac/.local/bin/claude`). Update this path if your installation differs.

---

## Output Structure

```
output/
├── video_title_transcript.md    # Extracted transcript/article
├── video_title_summary.md       # AI-generated summary
└── video_title_audio.mp3        # TTS audio of summary
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Claude CLI timeout | Increase timeout in `summarizer.py` (default: 300s) |
| YouTube transcript not found | Video may not have captions — check available languages |
| Google Drive auth fails | Delete `token.pickle` and retry |
| Notion 401 error | Check `NOTION_TOKEN` and ensure database is shared with integration |
| Edge TTS fails | Check internet connection — service requires network access |
| Watcher reprocessing entries | Check `.processed_ids` file — ensure page IDs are cached correctly |
| Webhook returns 403 | Verify `WEBHOOK_SECRET` matches between sender and `.env` |
