# Tao Bite — Process Documentation

## Overview

PDF-based knowledge management system with RAG (Retrieval Augmented Generation). Upload PDFs or Markdown documents, build vector embeddings in Qdrant, and generate custom content (quotes, posts, summaries, insights) using Claude AI with strict anti-hallucination controls.

All source documents are stored as `.md` files in Google Drive:
https://drive.google.com/drive/u/0/folders/1m874HrsS4DO7v9Y4pI4ES1sF2g4ktUlq

**Repo**: [yasser-ensembl3/TaoBite](https://github.com/yasser-ensembl3/TaoBite)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Web framework | Flask 3.1 (Python 3.11) |
| PDF extraction | pdfplumber (primary) + LlamaParse (fallback) |
| Text chunking | LangChain RecursiveCharacterTextSplitter |
| Token counting | tiktoken |
| Embeddings | OpenAI text-embedding-3-small (1536 dim) |
| Vector database | Qdrant (local SQLite or Qdrant Cloud) |
| AI generation | Anthropic Claude 3 Haiku |
| Production server | Gunicorn |

---

## Architecture

```
Tao Bite/
├── app.py                             # Main Flask application (30+ endpoints)
├── qdrant_store.py                    # Standalone Qdrant wrapper class
├── migrate_to_qdrant_cloud.py         # Local → Qdrant Cloud migration
├── obsidian_pdf_converter.py          # PDF → Obsidian Markdown converter
├── pdf_to_markdown.py                 # Marker-based PDF converter
├── templates/
│   ├── index.html                     # Main UI (content generator + DB overview)
│   ├── admin.html                     # Document upload page (drag & drop)
│   ├── database_overview.html         # Database viewer
│   ├── draft_generator.html           # Substack draft generator
│   ├── qdrant_viewer.html             # Qdrant collection viewer
│   └── obsidian_converter.html        # Obsidian converter UI
├── static/
│   ├── css/                           # Stylesheets
│   └── js/                            # Client-side scripts
├── uploads/                           # Uploaded PDFs (gitignored)
├── outputs/                           # Converted Markdown (gitignored)
├── qdrant_storage/                    # Local vector DB (gitignored)
├── requirements.txt                   # 132 packages
└── .env.example
```

---

## Pipeline — How It Works

### Full Flow

```
Documents (.md from Google Drive or .pdf uploads)
    │
    ├── 1. PDF extraction (pdfplumber → LlamaParse fallback)
    ├── 2. Chunking (1000 tokens, 200 overlap)
    ├── 3. Embedding (OpenAI text-embedding-3-small)
    └── 4. Vector storage (Qdrant, collection: pdf_documents)
    │
    ▼
User query (keywords + instructions)
    │
    ├── 5. Semantic search (query embedding → Qdrant cosine similarity)
    ├── 6. Relevance filtering (score threshold ≥ 30%)
    ├── 7. Context assembly (top-k passages with source attribution)
    └── 8. Generation (Claude 3 Haiku, anti-hallucination prompting)
    │
    ▼
Output: Generated content + source chunks with relevance scores
```

### Step 1: Document Ingestion

Two ingestion paths:

**From Google Drive (primary):**
1. Download `.md` files from the [Drive folder](https://drive.google.com/drive/u/0/folders/1m874HrsS4DO7v9Y4pI4ES1sF2g4ktUlq)
2. Upload via the `/admin` page
3. Auto-pipeline processes them

**From PDF upload:**
1. Upload via `/admin` (drag & drop) or `POST /upload`
2. Dual extraction strategy:
   - **pdfplumber** — local, fast, free, works for most text-based PDFs
   - **LlamaParse** — cloud fallback for scanned/complex PDFs with OCR
3. Quality gate: minimum 100 characters extracted
4. Markdown saved to `outputs/`

### Step 2: Chunking

**Module**: `app.py` → `chunk_markdown()`

Uses LangChain's `RecursiveCharacterTextSplitter`:
- **Chunk size**: 1000 tokens
- **Overlap**: 200 tokens (context preservation)
- **Separators**: `["\n\n", "\n", " ", ""]` — paragraphs first
- **Token counting**: tiktoken (`cl100k_base` encoding)

Each chunk gets metadata: `chunk_id`, `token_count`, `char_count`.

### Step 3: Embedding

**Model**: OpenAI `text-embedding-3-small` (1536 dimensions)

- Batch processing: 100 texts per API call
- OpenAI limit: 2048 texts per request

### Step 4: Vector Storage

**Database**: Qdrant (collection `pdf_documents`)

- **Distance metric**: Cosine similarity
- **Vector size**: 1536
- **Payload per point**:
  ```json
  {
    "text": "chunk content",
    "chunk_id": 1,
    "token_count": 845,
    "char_count": 5234,
    "filename": "document.pdf",
    "job_id": "uuid",
    "source": "pdfplumber"
  }
  ```

Auto-pipeline endpoint `POST /auto-pipeline/<job_id>` runs steps 2-4 automatically after upload.

### Step 5-6: Semantic Search & Filtering

1. User query embedded via OpenAI (same model)
2. Qdrant similarity search returns top_k results (configurable 5-20)
3. Results filtered by minimum relevance score (default 30%)

### Step 7-8: Content Generation (RAG)

**Model**: Claude 3 Haiku (Anthropic)

Anti-hallucination controls:
- Word-for-word extraction only — no paraphrasing
- Source citations required for every claim
- Refuses if insufficient relevant content found
- Substantive quotes only (15+ words)
- No section headers, survey items, or filler

**Templates available**: Quotes, LinkedIn Post, Summary, Key Terms, Insights — or custom instructions.

Response includes: generated content + full source chunks with relevance scores + token usage tracking.

---

## Qdrant Database Setup

The Qdrant vector database stores all document embeddings. The collection is auto-created on first injection, but the Qdrant instance must be available.

### Option A: Local Qdrant (development)

No setup needed — when `QDRANT_URL` is not set, the app uses local SQLite-based storage in `./qdrant_storage/`.

### Option B: Qdrant Cloud (production)

1. Create a free cluster at [cloud.qdrant.io](https://cloud.qdrant.io/)
2. Get the cluster URL and API key from the dashboard
3. Set in `.env`:
   ```bash
   QDRANT_URL=https://your-cluster-id.region.qdrant.io
   QDRANT_API_KEY=your-api-key
   ```
4. The app auto-detects cloud configuration and connects to it

### Recreating the Database (after deletion)

The Qdrant database was deleted. To recreate it:

1. **Start the app** — `python app.py`
2. **Download source documents** — Get all `.md` files from the [Google Drive folder](https://drive.google.com/drive/u/0/folders/1m874HrsS4DO7v9Y4pI4ES1sF2g4ktUlq)
3. **Upload each document** via the `/admin` page (http://localhost:8080/admin)
4. **Auto-pipeline runs automatically** for each upload:
   - Chunks the content (1000 tokens, 200 overlap)
   - Generates OpenAI embeddings (1536 dimensions)
   - Creates the `pdf_documents` collection if it doesn't exist
   - Injects all vectors with metadata

The collection is auto-created by `ensure_qdrant_collection()`:
```python
VectorParams(size=1536, distance=Distance.COSINE)
```

### Migrating Local → Cloud

After populating locally, migrate to production:

```bash
python migrate_to_qdrant_cloud.py
```

Batch-transfers all vectors from `./qdrant_storage/` to Qdrant Cloud with metadata preserved.

---

## API Routes

### Content Generation

| Route | Method | Description |
|-------|--------|-------------|
| `/generate-content` | POST | Generate content with keywords + custom instructions + top_k + min_score |
| `/extract-quotes` | POST | Extract quotes (legacy) |
| `/generate-draft` | POST | Generate Substack drafts |

### Document Management

| Route | Method | Description |
|-------|--------|-------------|
| `/upload` | POST | Upload PDF (returns job_id) |
| `/auto-pipeline/<job_id>` | POST | Auto-process: chunk → embed → inject |
| `/status/<job_id>` | GET | Check processing status |

### Database

| Route | Method | Description |
|-------|--------|-------------|
| `/api/database/stats` | GET | Collection statistics (total points, documents) |
| `/api/database/documents` | GET | List all documents with chunk counts and token stats |
| `/qdrant/search` | POST | Semantic search with query, top_k, min_score |

---

## Installation & Setup

### Prerequisites

- Python 3.11+
- API keys: OpenAI + Anthropic (required), LlamaParse (optional)

### Installation

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# Edit .env with your API keys
```

### Environment Variables

| Variable | Required | Purpose |
|----------|----------|---------|
| `OPENAI_API_KEY` | Yes | Embeddings (text-embedding-3-small) |
| `ANTHROPIC_API_KEY` | Yes | Claude 3 Haiku content generation |
| `LLAMA_CLOUD_API_KEY` | No | LlamaParse PDF fallback extraction |
| `QDRANT_URL` | No | Qdrant Cloud URL (local storage if unset) |
| `QDRANT_API_KEY` | No | Qdrant Cloud authentication |
| `PORT` | No | Server port (default: 8080) |

### Running

```bash
python app.py
# Open http://localhost:8080
# Admin upload: http://localhost:8080/admin
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Missing API keys error on startup | Copy `.env.example` to `.env` and fill in OpenAI + Anthropic keys |
| PDF extraction returns empty | Try LlamaParse fallback — set `LLAMA_CLOUD_API_KEY` |
| Qdrant collection not found | Upload a document — collection is auto-created on first injection |
| Low relevance scores | Try broader keywords, or lower min_score threshold |
| Claude refuses to generate | Not enough relevant content found — add more documents to the database |
| Local Qdrant SQLite threading error | App handles this with `force_disable_check_same_thread=True` |
| Migration fails | Ensure both `QDRANT_URL` and `QDRANT_API_KEY` are set, and local storage exists |
