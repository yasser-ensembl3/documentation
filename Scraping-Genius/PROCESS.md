# Scraping Genius — Process Documentation

## Overview

n8n automation workflow that scrapes song lyrics from Genius.com. Uses Puppeteer-based Node.js scripts for headless browser automation, extracts and cleans lyrics via HTML parsing and JavaScript processing, and stores the results in a Google Sheet.

**Repo**: [yasser-ensembl3/Scraping-genius](https://github.com/yasser-ensembl3/Scraping-genius)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Workflow engine | n8n (self-hosted) |
| Browser automation | Puppeteer (headless Chrome) |
| Runtime | Node.js |
| HTML parsing | n8n HTML Extractor node (CSS selectors) |
| Data processing | JavaScript (n8n Code node) |
| Storage | Google Sheets API (OAuth2) |
| Target site | Genius.com |

---

## Architecture

```
Scraping genius/
├── Scraping genius.com.json     # n8n workflow definition (import into n8n)
├── scraping_genius.js           # Puppeteer: search Genius.com → return URL
├── scrape-single-page.js        # Puppeteer: fetch page HTML content
└── .gitignore
```

---

## Pipeline — How It Works

### Full Flow

```
n8n Form (Song title + Artist)
    │
    ├── 1. scraping_genius.js → Search Genius.com → first result URL
    ├── 2. scrape-single-page.js → Fetch full page HTML
    ├── 3. HTML Extractor → CSS selectors extract lyrics + title
    ├── 4. Code node → Clean text, filter song sections
    └── 5. Google Sheets → Append (Title, Lyrics, URL)
```

### n8n Workflow Nodes

```
Form Trigger → Execute Command → Execute Command1 → HTML Extract → Code → Google Sheets
```

| Node | Type | Input | Output |
|------|------|-------|--------|
| **On form submission** | Form Trigger | User enters song title + artist | `{ Titre, Artist }` |
| **Execute Command** | Execute Command | Runs `scraping_genius.js "Artist Titre"` | Genius URL (stdout) |
| **Execute Command1** | Execute Command | Runs `scrape-single-page.js <url>` | Full HTML (stdout) |
| **HTML1** | HTML Extractor | Parses HTML with CSS selectors | `{ Lyrics: [...], Titre }` |
| **Code** | JavaScript Code | Cleans and filters lyrics sections | `{ url, lyrics, sectionsCount, hasLyrics }` |
| **Genius** | Google Sheets | Appends row to spreadsheet | Row with Title, Lyrics, URL |

### Step 1: Search — scraping_genius.js

Headless Puppeteer script that searches Genius.com and returns the first result URL.

**Process:**
1. Launch headless Chrome with anti-detection flags
2. Navigate to `https://genius.com/`
3. Wait 3 seconds for page stabilization
4. Find and fill search input (`input[name="q"]`)
5. Press Enter to submit search
6. Wait for results (selector: `a.mini_card`)
7. Extract first result's `href`
8. Output URL via `console.log` (captured by n8n as stdout)

**Anti-detection measures:**
- Removes `navigator.webdriver` property
- Simulates Chrome runtime object
- Realistic User-Agent (Chrome 120, macOS)
- Full HTTP header spoofing (Accept-Language, Sec-Fetch-*, Cache-Control)
- Puppeteer flags: `--disable-blink-features=AutomationControlled`, `--no-sandbox`, `--disable-web-security`

### Step 2: Fetch Page — scrape-single-page.js

Headless Puppeteer script that fetches the complete HTML of a Genius song page.

**Process:**
1. Launch headless Chrome
2. Set realistic User-Agent and HTTP headers
3. Enable request interception — block CSS and images (performance)
4. Navigate to the song URL (`domcontentloaded` strategy, 60s timeout)
5. Wait 3 seconds for client-side JavaScript execution
6. Detect captcha/blocking (checks URL for "captcha" or "blocked")
7. Output full page HTML via `console.log` (captured by n8n as stdout)

### Step 3: HTML Extraction

n8n HTML Extractor node parses the stdout HTML using CSS selectors:

| Key | CSS Selector | Returns |
|-----|-------------|---------|
| `Lyrics` | `[data-lyrics-container="true"]` | Array of lyrics container text |
| `Titre` | `[class*="SongHeader-desktop__HiddenMask-"]` | Song title string |

### Step 4: Lyrics Cleaning

n8n Code node (JavaScript) processes the raw extracted text:

1. **Remove metadata**: "Read More" text, navigation, translation links
2. **Remove references**: Internal links (`[/digit/...]`)
3. **Normalize whitespace**: Max 3 newlines → 2 newlines, collapse spaces
4. **Filter sections**: Keep only recognized song structure markers:
   - `[Verse]`, `[Chorus]`, `[Pre-Chorus]`, `[Post-Chorus]`
   - `[Bridge]`, `[Outro]`, `[Intro]`
5. **Combine**: Join valid sections with double newline separators

Output:
```json
{
  "url": "https://genius.com/...",
  "lyrics": "cleaned full lyrics text",
  "sectionsCount": 8,
  "hasLyrics": true
}
```

### Step 5: Google Sheets Storage

Appends a row to a Google Sheet with three columns:

| Column | Source |
|--------|--------|
| Titre | Song title from HTML extraction |
| Lyrics | Cleaned lyrics from Code node |
| url | Genius URL from search step |

---

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- Node.js 18+
- Puppeteer installed
- Google Sheets OAuth2 credentials configured in n8n

### Installation

1. **Install Puppeteer scripts:**
   ```bash
   mkdir -p ~/mon-projet-puppeteer
   cp scraping_genius.js scrape-single-page.js ~/mon-projet-puppeteer/
   cd ~/mon-projet-puppeteer && npm init -y && npm install puppeteer
   ```

2. **Import n8n workflow:**
   - Open n8n → Workflows → Import from file
   - Select `Scraping genius.com.json`

3. **Update script paths** in the Execute Command nodes:
   - Node "Execute Command": path to `scraping_genius.js`
   - Node "Execute Command1": path to `scrape-single-page.js`

4. **Configure Google Sheets:**
   - Create Google Sheets OAuth2 credential in n8n
   - Create a spreadsheet with columns: `Titre`, `Lyrics`, `url`
   - Update the document ID in the "Genius" node

5. **Activate the workflow** — the form becomes available at the n8n webhook URL.

### Credentials Required

| Service | Type | Purpose |
|---------|------|---------|
| Google Sheets | OAuth2 (configured in n8n) | Store extracted lyrics |
| Genius.com | None (anonymous scraping) | Source of lyrics |

---

## Usage

1. Open the n8n form URL (provided after workflow activation)
2. Enter song title (required) and artist (optional)
3. Submit — workflow runs the full pipeline automatically
4. Check the Google Sheet for the new row with title, lyrics, and URL

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Search returns no results | Check if Genius.com changed the `a.mini_card` selector |
| Page fetch timeout | Genius may be blocking — try running with `headless: false` to debug |
| Captcha detected | IP may be rate-limited — wait or use a different network |
| Empty lyrics | CSS selector `[data-lyrics-container="true"]` may have changed |
| Script path not found | Update absolute paths in Execute Command nodes |
| Google Sheets auth error | Re-authorize the OAuth2 credential in n8n |
