# Scraping Centris.ca — Process Documentation

## Overview

Real estate property scraping tools for centris.ca (Quebec's main real estate listing platform). Three scripts with different approaches: an n8n workflow for scheduled property detail extraction, a verbose debug Puppeteer script for condo searches, and a stealth-optimized Puppeteer script for commercial property link collection with pagination.

**Repo**: [yasser-ensembl3/Centris.ca](https://github.com/yasser-ensembl3/Centris.ca)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Workflow engine | n8n (self-hosted) |
| Browser automation | Puppeteer (headless Chrome) |
| Runtime | Node.js |
| HTML parsing | n8n HTML Extractor (CSS selectors) |
| Storage | Google Sheets API (OAuth2) |
| Target site | centris.ca |

---

## Architecture

```
Scraping centris.ca/
├── Scraping centris.ca.json     # n8n workflow — detail page extraction (most recent)
├── Scraping_centris.js          # Puppeteer debug script — condo search (oldest)
└── centris_scraper.js           # Puppeteer stealth script — commercial link collection
```

---

## Script 1: n8n Workflow — Scraping centris.ca.json

**Purpose**: Scheduled automation that reads property URLs from Google Sheets, fetches each detail page, extracts structured data via CSS selectors, and writes results back.

**Most recent version** — this is the production workflow.

### Pipeline

```
Schedule Trigger (hourly)
    → Read Google Sheets (filter by Statut column)
    → Limit to 20 items per run
    → Loop Over Items
        → HTTP Request (fetch property URL)
        → HTML Extractor (CSS selectors)
        → Code node (parse image URLs from embedded JS)
        → Append/Update Google Sheets row
```

### Data Extracted

| Field | CSS Selector | Description |
|-------|-------------|-------------|
| Titre | `[data-id="PageTitle"]` | Property title/heading |
| Address | `[itemprop="address"]` | Structured address |
| Price | `#BuyPrice` | Listed sale price |
| Caracteristiques | `.row.teaser > div[class*="col-"]` | Property features (array) |
| Details | `.carac-container` | Specifications — type, age, lot size, etc. (array) |
| Evaluation financiere | `.financial-details-table table` | Municipal assessment values (array) |
| Images | `script:contains('window.MosaicPhotoUrls')` | Photo URLs from embedded JavaScript |

### Image URL Extraction

The Code node parses photo URLs from the JavaScript variable `window.MosaicPhotoUrls` embedded in the page source:

```javascript
const match = imageScript.match(/window\.MosaicPhotoUrls\s*=\s*(\[.*?\]);/);
imageUrls = JSON.parse(match[1]); // Array of up to 20 image URLs
```

### Google Sheets Structure

| Column | Content |
|--------|---------|
| Links | Property URL (input + matching key for upsert) |
| Titre | Extracted title |
| Addresse | Extracted address |
| Prix | Extracted price |
| Details | Property specifications (newline-separated) |
| Evaluation municipale | Municipal assessment (formatted) |
| Images | Up to 20 photo URLs (newline-separated) |
| Statut | Set to "OK" after processing |

Uses `appendOrUpdate` mode with `Links` as matching column — re-running updates existing rows.

### Workflow Logic

1. **Schedule Trigger** fires hourly
2. **Read rows** filtered by `Statut` column (empty = not yet processed)
3. **Limit** caps at 20 per run to avoid timeouts
4. **Loop** processes each item sequentially:
   - Fetch page HTML via HTTP Request
   - Extract fields via CSS selectors
   - Parse image URLs from JavaScript
   - Write back to Google Sheets with `Statut = "OK"`

### Credentials

- Google Sheets OAuth2 (yasser@ensembl3.xyz)
- Notion API (disabled — was for writing to Notion database "Proprietes Immobilieres")

---

## Script 2: Scraping_centris.js — Debug/Development

**Purpose**: Visual development script with comprehensive logging, screenshots, and verbose output. Searches for condos in Montreal Rive-Nord.

**Oldest version** — used for initial development and debugging.

### Pipeline

```
Launch browser (visible, slowMo: 250ms)
    → Navigate to centris.ca/fr/
    → Search "Montreal Rive-Nord" (.select2-search__field)
    → Open filters → Select "Condo" (#PropertyType-SellCondo-input)
    → Click search → Wait for results
    → Extract property data from .property-thumbnail-item elements
    → Save HTML + JSON + 11 screenshots
```

### Data Extracted Per Listing

| Field | Selector | Description |
|-------|----------|-------------|
| Price | `.price span` | Listed price |
| Address | `.address` | Property address |
| Category | `.category` | Property type |
| MLS number | `[id^="MlsNumberNoStealth"] p` | MLS listing number |
| Bedrooms | `.features .cac` | Number of bedrooms |
| Bathrooms | `.features .sdb` | Number of bathrooms |
| Link | `.a-more-detail` href | Detail page URL |

### Output Files

- `centris_results_[timestamp].html` — full page HTML
- `centris_properties_[timestamp].json` — structured data
- `screenshot_[step]_[timestamp].png` — 11 screenshots at each step

### Features

- Color-coded console logging (cyan, green, yellow, red, magenta)
- Timestamps and emojis for every action
- Event listeners for console messages, page errors, HTTP responses
- n8n compatible via `runForN8n()` export (returns first 10 properties)

```bash
node Scraping_centris.js
```

---

## Script 3: centris_scraper.js — Stealth Production

**Purpose**: Headless Puppeteer script optimized for anti-detection. Searches commercial multi-family properties in specific Montreal areas with full pagination.

**Middle version** — refined approach with human-like behavior simulation.

### Pipeline

```
Launch headless browser (anti-detection enabled)
    → Navigate to centris.ca/fr/propriete-commerciale~a-vendre
    → Accept cookie consent (#didomi-notice-agree-button)
    → Search "Montreal (Saint-Leonard)" + "Montreal (Montreal-Nord)"
    → Open filters → Select "Multi-Family" (#MultiFamily-input)
    → Click search → Wait for results
    → Paginate through all result pages (li.next a)
    → Collect all property links (a.a-more-detail)
    → Output deduplicated URLs
```

### Human-Like Behavior Functions

| Function | What It Simulates |
|----------|-------------------|
| `randomDelay(min, max)` | Random waits between actions |
| `humanScroll(page)` | Variable speed/distance scrolling |
| `scrollToTop(page)` | Smooth scroll back to top |
| `humanMouseMovement(page)` | Random mouse moves with steps |
| `humanClick(page, selector)` | Natural click with position variance and pre-movement |
| `humanType(page, selector, text)` | Typing with delays, occasional typos + backspace correction |
| `readingPause()` | Simulates reading time (1.5-4 seconds) |
| `exploratoryScroll(page)` | Multi-step scrolling with pauses |
| `occasionalScroll(page)` | Random scrolls during navigation |

### Anti-Detection Measures

- `navigator.webdriver` property removed
- Chrome runtime object simulated
- French User-Agent + language headers (`fr-FR, fr, en`)
- Puppeteer flags: `--disable-blink-features=AutomationControlled`, `--start-maximized`
- Full HTTP header spoofing (Sec-Fetch-*, Cache-Control, Accept-Language)

### Pagination

Follows `li.next a` across all result pages. Stops when `li.next` has class `inactive`. Extra 5-8 second delay every 3 pages to avoid rate limiting.

### Output

URLs printed to console (one per line). Returns `{ links: [], totalPages: n }`.

```bash
node centris_scraper.js
```

---

## Comparison

| Feature | n8n Workflow | Scraping_centris.js | centris_scraper.js |
|---------|-------------|--------------------|--------------------|
| Mode | Scheduled (hourly) | Manual (visible browser) | Manual (headless) |
| Anti-detection | None (HTTP requests) | Low | High (human simulation) |
| Property type | From input URLs | Condo | Commercial Multi-Family |
| Location | Via Google Sheets | Montreal Rive-Nord | Saint-Leonard, Montreal-Nord |
| Data extracted | Full details + images | Listing data + screenshots | Links only |
| Pagination | No (fixed URLs) | No | Yes (all pages) |
| Output | Google Sheets (upsert) | JSON + HTML + PNG files | Console (URLs) |
| Frequency | Automated hourly | One-time manual | One-time manual |

---

## Setup

### Prerequisites

- Node.js 18+
- Puppeteer (`npm install puppeteer`)
- n8n instance (for the workflow)

### Installation

```bash
npm init -y
npm install puppeteer
```

For the n8n workflow:
1. Import `Scraping centris.ca.json` into n8n
2. Configure Google Sheets OAuth2 credentials
3. Create a Google Sheet with columns: Links, Titre, Addresse, Prix, Details, Evaluation municipale, Images, Statut
4. Add property URLs to the Links column
5. Activate the workflow

### Credentials

| Service | Type | Used By |
|---------|------|---------|
| Google Sheets | OAuth2 (n8n) | n8n workflow |
| Notion | API token (n8n) | n8n workflow (disabled) |
| Centris.ca | None | All scripts (anonymous) |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Puppeteer launch fails | Ensure Chromium is installed: `npx puppeteer browsers install chrome` |
| Cookie consent blocks interaction | Script handles `#didomi-notice-agree-button` — may need selector update |
| Search dropdown not appearing | centris.ca may have changed `.select2-search__field` selector |
| Anti-bot blocking | Use `centris_scraper.js` (stealth mode) or add delays between runs |
| n8n workflow returns empty | Check Google Sheet has URLs in Links column with empty Statut |
| Images field empty | `window.MosaicPhotoUrls` regex may need updating if page structure changed |
