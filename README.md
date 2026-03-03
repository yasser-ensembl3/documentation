# Projects Documentation

Central documentation and process guides for all projects.

| Project | Description | Doc |
|---------|-------------|-----|
| [Kindle-to-md](Kindle-to-md/) | Book-to-Markdown pipeline with AI distillation and thematic synthesis | [Process](Kindle-to-md/PROCESS.md) |
| [Transcript-pipeline](Transcript-pipeline/) | Auto-extract, summarize, and generate audio from YouTube/articles/podcasts via Notion | [Process](Transcript-pipeline/PROCESS.md) |
| [Research Vault](Research-Vault/) | Research management platform — paper search (ArXiv/Semantic Scholar), hypothesis tracking, Notion sync | [Process](Research-Vault/PROCESS.md) |
| [Stoic Vault](Stoic-Vault/) | Financial insights dashboard — quarterly reports, company comparison, AI chat, stock data pipeline | [Process](Stoic-Vault/PROCESS.md) |
| [MiniVault (Tao Promotion)](MiniVault-TaoPromotion/) | Unified project dashboard — Notion, Drive, GitHub, Shopify integration with modular sections | [Process](MiniVault-TaoPromotion/PROCESS.md) |
| [Founders Graph](Founders-Graph/) | Multi-source founder profiling pipeline — web scraping, LinkedIn, YouTube, LLM enrichment and synthesis | [Process](Founders-Graph/PROCESS.md) |
| [Options Model](Options-Model/) | Montreal Exchange options dashboard — Python scraper, PostgreSQL storage, analytics charts, AI analysis | [Process](Options-Model/PROCESS.md) |
| [Tao Bite](Tao-Bite/) | PDF knowledge base with RAG — Qdrant vector search, OpenAI embeddings, Claude AI content generation | [Process](Tao-Bite/PROCESS.md) |
| [Scraping Genius](Scraping-Genius/) | n8n workflow — Genius.com lyrics scraping with Puppeteer, Google Sheets storage | [Process](Scraping-Genius/PROCESS.md) |
| [Scraping Centris.ca](Scraping-Centris/) | Real estate scraping — n8n workflow + Puppeteer scripts for centris.ca property data | [Process](Scraping-Centris/PROCESS.md) |
| [Products Scrapers](Products-Scrapers/) | Multi-provider e-commerce scraping toolkit — Apify, Bright Data, Channel3, DataForSEO, Zyte + n8n Shopify workflow with PostgreSQL | [Process](Products-Scrapers/PROCESS.md) |
| [Founder MTL](Founder-MTL/) | MiniVault project dashboard — Notion, Google Drive, Gmail, GitHub, AI reports in a unified Next.js 14 interface | [Process](Founder-MTL/PROCESS.md) |
| [Content Vault](Content-Vault/) | Read-only content reader — Notion-backed with n8n transcript summaries, markdown preview, favorites, read/unread tracking | [Process](Content-Vault/PROCESS.md) |
| [Dashboard MiniVault](Dashboard-MiniVault/) | MetaVault command center — aggregates all vaults, 6 Notion DBs, GitHub Project V2, Claude AI media curation, digest feed | [Process](Dashboard-MiniVault/PROCESS.md) |
| [Present Agent](Present-Agent/) | MiniVault template — reusable dashboard boilerplate with setup script, 6 Notion DBs, dual AI, OAuth, schema-adaptive API | [Process](Present-Agent/PROCESS.md) |
| [Kai Claude Code Config](Kai-Claude-Code-Config/) | Personal AI Infrastructure — 12 skills, 8 hooks, 3 tools, 2 MCP servers (Notion + Shopify), 10-tier security, structured history | [Process](Kai-Claude-Code-Config/PROCESS.md) | 


# Notion Databases Documentation

| Main Page | Database | Details | Link |
|-----------|----------|---------|------|
| Transcripts | **Archives** | Repository of past transcriptions, preserving older media content for future reference and research | [Archives](https://www.notion.so/Archives-22558fe731b180738e0ed16013e0e7c3) |
| Transcripts | **n8n Transcript** | Connected to the Media Vault — paste a YouTube or article URL and the system automatically generates the transcription along with its audio version, no manual processing required | [n8n Transcript](https://www.notion.so/2cd58fe731b1808a9228df1e399d74fe?v=2cd58fe731b1807cb4e0000c6a893a82) |
| Wiki | **Emails 2025** | Output of an automated n8n workflow that retrieves all emails from the Stoic label in Gmail, generates a summary for each, and pushes them into Notion — fully automated, no manual input required, covering the entire year 2025 | [Emails 2025](https://www.notion.so/2cd58fe731b180a4aceec7c59e3bbf48?v=2cd58fe731b18096b180000c2b73a653) |
| Wiki | **Stoic Investment Company Recap** | Markdown-formatted recap of all investment company movements within Stoic — tracking transactions, changes, and key decisions across all related entities | [Stoic Investment Company Recap](https://www.notion.so/Stoic-investment-company-recap-2ce58fe731b180ff98bde1a59a18464a) |
| Delegation Station | **Delegation Station** | Database grouping processes for recurring tasks performed throughout the year. No longer updated since mid-2025 following the transition to GitHub | [Delegation Station](https://www.notion.so/18858fe731b180bebbfdc29083578bb4?v=18858fe731b180219482000c072fada8) |
| TO DO | **TO DO** | Database grouping all TO DOs for Gui and Yasser, along with a global Kanban view. No longer in use since end of 2025 following the transition to GitHub | [TO DO](https://www.notion.so/18c58fe731b180929813f5618e0e77ed?v=1cb58fe731b180919b62000c1bffb8de) |
| [MiniVault Projects](https://www.notion.so/MiniVault-Projects-28e58fe731b180438a2ffaa541202d35) | **Tao Promotion Database** | Central project database for the Tao promotion, linked to the MiniVault system. Contains 8 interconnected databases: Tasks, Goals, Milestones, Documents, Feedback, Metrics, Recurring Tasks, and Essentials. A fetch system automatically pulls data from these Notion databases to keep the MiniVault up to date in real time | [Tao Promotion Database](https://www.notion.so/Tao-Promotion-Database-29d58fe731b18165b26efba278f74602) |
| [MiniVault Projects](https://www.notion.so/MiniVault-Projects-28e58fe731b180438a2ffaa541202d35) | **Saved Articles** | Database linked to the Media Vault that automatically saves all bookmarked or favorited articles, centralizing them in Notion for easy access and reference | [Saved Articles](https://www.notion.so/2d958fe731b180d5a744d354f84db9fb) |
| Yasser December 2025 | **Yasser December 2025** | All tasks for Yasser during December 2025, including progress tracking, comments, and updates for each task | [Yasser December 2025](https://www.notion.so/Yasser-December-2025-2ca58fe731b181f29cede09f2123ce18) |
| Yasser January 2026 | **Yasser January 2026** | All tasks for Yasser during January 2026, including progress tracking, comments, and updates for each task | [Yasser January 2026](https://www.notion.so/Yasser-January-2026-2d758fe731b180e39bb9e29d30ad2216) |
| Fournisseurs | **Fournisseurs** | List of providers sourced from TaskRabbit for the various jobs and tasks requested by Gui | [Fournisseurs](https://www.notion.so/26858fe731b180138e7dec577b3243d7?v=26858fe731b180989d51000c2f29bb33) |
