# Kindle-to-MD — Process Documentation

## Overview

Automated pipeline that converts books (PDF or Markdown) into structured Markdown, distills each chapter through 3 analytical lenses, and synthesizes thematic insights — all powered by Claude.

**Repo**: [yasser-ensembl3/Kindle-to-pdf](https://github.com/yasser-ensembl3/Kindle-to-pdf)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Python 3.10+ |
| CLI | Typer + Rich |
| PDF extraction | pdfplumber |
| Data models | Pydantic v2 |
| LLM calls | Claude Code CLI (`claude -p`) |
| Google Drive | google-api-python-client + OAuth2 |
| Parallelism | concurrent.futures.ThreadPoolExecutor (10 workers) |

---

## Architecture

```
Kindle-to-md/
├── src/
│   ├── cli/commands.py            # CLI entry point — 5 commands
│   ├── extractors/
│   │   ├── pdf.py                 # PDF extraction (pdfplumber)
│   │   ├── markdown_parser.py     # .md parser (3 supported formats)
│   │   └── chapter_detector.py    # Chapter/part detection (regex)
│   ├── converters/
│   │   └── book_markdown.py       # Book → Obsidian Markdown
│   ├── models/
│   │   └── book.py                # Pydantic models (Book, Chapter, Part)
│   ├── prompts/
│   │   └── generator.py           # Prompt generation from templates
│   ├── drive/
│   │   └── client.py              # Google Drive API v3 client
│   └── config.py                  # Paths and default config
├── templates/
│   ├── distillation_chapter.md    # Per-chapter 3-lens prompt
│   ├── distillation_assembly.md   # Assembly prompt
│   └── insights_synthesis.md      # Thematic synthesis prompt
├── watch.sh                       # launchd watcher for inbox/
├── com.kindle2md.watcher.plist    # macOS launchd config
├── pyproject.toml
├── requirements.txt
└── INSTALL.md
```

---

## Pipeline — How It Works

### Step 1: Extraction

**Command**: `kindle2md extract <file>`

- **PDF**: pdfplumber extracts text page by page → detects and removes recurring headers/footers (>30% frequency) → fixes hyphenated word breaks at line endings → detects chapter/part boundaries via regex
- **Markdown**: Parser handles 3 formats:
  - Epub-converted: `C[HAPTER]{.small}`
  - Standard: `## Chapter N - Title`
  - Numbered: `## 1. Title`
- Auto-detects parts, strips backmatter (notes, appendix, etc.)

**Output**: `book.md` (structured Markdown with YAML frontmatter + Obsidian TOC) + individual chapter files in `chapters/`

### Step 2: Distillation

**Command**: `kindle2md distill <file>`

Each chapter is sent to Claude with a structured prompt (template `distillation_chapter.md`). Analysis through 3 lenses:

1. **Phenomenology** (4-6 bullets) — what it *feels* like from the inside. Quotes, metaphors, lived experiences
2. **Deep Facts** (4-6 bullets) — 3rd-to-5th layer insights. Counter-intuitive truths, hidden mechanisms
3. **Action Items** (3-5 checkboxes) — concrete, actionable recommendations

**Parallelism**: 10 concurrent Claude calls. Large chapters (>15,000 words) are automatically chunked.

**Assembly**: Final assembly is done locally (string concatenation, no LLM call).

**Output**: `book_distillation.md`

### Step 3: Synthesis

**Command**: `kindle2md synthesize <file>`

The full distillation is sent to Claude to reorganize **by theme** (not by chapter). Produces 6-10 thematic sections numbered with roman numerals, plus a 4-6 sentence core synthesis.

If the distillation is too large (>15,000 words), it's chunked, synthesized in parallel, then merged.

**Output**: `book_insights.md`

### Full Pipeline

**Command**: `kindle2md pipeline <file>` — runs all 3 steps end-to-end.

```bash
kindle2md pipeline book.pdf --model haiku
```

---

## Drive Sync

**Command**: `kindle2md drive-sync <folder_url>`

Workflow:
1. OAuth2 Google authentication (opens browser on first run, token cached afterwards)
2. Recursive scan of the Drive folder for `.md` files
3. Filters out already-generated files (`_distillation.md`, `_insights.md`)
4. For each book:
   - Downloads the `.md` file
   - Parses chapters (auto-detects format)
   - Distills in parallel (10 workers)
   - Synthesizes insights
   - Uploads `_distillation.md` and `_insights.md` to the same Drive subfolder

**Performance**: ~2.5 min for a ~60k word / 20 chapter book.

---

## Installation & Setup

### Prerequisites

- Python 3.10+
- Claude Code CLI installed and authenticated (`claude -p "test"` must work)
- Google OAuth credentials (only for Drive sync)

### Installation

```bash
cd Kindle-to-md
python3 -m venv venv && source venv/bin/activate
pip install -e .
kindle2md --help
```

### Google Drive (optional)

1. Google Cloud Console → create a project
2. Enable Google Drive API
3. Create OAuth 2.0 credentials (Desktop or Web)
4. Download the JSON → save as `credentials.json` at the project root
5. If Web type: add `http://localhost:8080/` to redirect URIs

### launchd Watcher (optional)

Watches `inbox/` and automatically runs the pipeline on dropped PDF files.

```bash
# Edit paths in the plist
nano com.kindle2md.watcher.plist

# Install
cp com.kindle2md.watcher.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.kindle2md.watcher.plist
```

---

## Command Reference

| Command | Description |
|---------|-------------|
| `kindle2md extract <pdf>` | Extract PDF → book.md + chapters |
| `kindle2md distill <pdf>` | Distill each chapter (3 lenses) |
| `kindle2md synthesize <pdf>` | Synthesize by themes |
| `kindle2md pipeline <pdf>` | All 3 steps end-to-end |
| `kindle2md drive-sync <url>` | Sync from Google Drive |
| `kindle2md version` | Show tool version |

### Common Options

| Option | Description |
|--------|-------------|
| `--model, -m` | Claude model: `haiku` (fast), `sonnet` (balanced), `opus` (best) |
| `--title, -t` | Override book title |
| `--author, -a` | Override author |
| `--output-dir, -o` | Custom output directory |

---

## Key Data Models

### Book (Pydantic)

```python
Book
├── metadata: BookMetadata (title, author, published, tags, aliases, related)
├── introduction: str
└── parts: list[Part]
    └── chapters: list[Chapter]
        ├── number, title, text
        ├── part_number, part_title
        └── start_page, end_page
```

- JSON serializable (`book.to_json()` / `Book.from_json()`)
- Computed properties: `all_chapters`, `total_chapters`, `total_words`

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `claude -p` not working | Verify Claude Code CLI is installed and authenticated |
| No chapters detected | Book format not recognized — check patterns in `chapter_detector.py` |
| Drive auth fails | Check `credentials.json`, delete `token.json` and retry |
| Timeout on large books | Increase timeout in `_call_claude()` (default: 300s) |
| Chapter too large | Automatically chunked if >15,000 words — no action needed |
