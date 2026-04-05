# Scope — Client Intelligence Digest Agent

## Objective

Build an automated agent that monitors two types of public sources weekly:

1. **Client-specific sources** — Each tracked client's own media pages (press releases, news mentions, featured stories)
2. **General industry sources** — Broad publications (e.g., Defense News) that may mention any tracked client

The agent identifies relevant content, analyzes it with an LLM, and delivers a curated email newsletter featuring:

1. **Relevant Highlights** — The most important and noteworthy developments across all clients, with context and analysis
2. **Per-Client Sections** — Organized by client, then by project, with summaries of all mentions and direct links to original sources

The agent runs weekly via GitHub Actions with zero manual intervention.

---

## Core Pipeline

```
┌─────────────┐     ┌───────────────────────────────────────────┐     ┌──────────────────┐     ┌──────────────┐     ┌──────────────┐
│  SCHEDULER   │     │            SOURCE SCRAPING                 │     │  LLM SUMMARIZER   │     │  EMAIL        │     │  STATE        │
│  (GH Actions │────▶│                                            │────▶│  + HIGHLIGHT       │────▶│  COMPOSER &   │────▶│  UPDATE       │
│   weekly)    │     │  Client Path           General Path        │     │  SELECTOR          │     │  SENDER       │     │  (JSON file)  │
│              │     │  (auto-tagged)    (keyword/LLM matched)   │     │  (Claude)          │     │               │     │               │
└─────────────┘     └───────────────────────────────────────────┘     └──────────────────┘     └──────────────┘     └──────────────┘
```

### Stage Descriptions

| Stage | Input | Output | LLM? |
|-------|-------|--------|------|
| **Client-Specific Scraper** | Client's own media pages (HTML) | Articles auto-tagged to that client | No |
| **General Source Scraper** | Industry publications (RSS/HTML) | Untagged articles from the wider industry | No |
| **Keyword/LLM Matcher** | General-source articles + Client registry | Articles matched and tagged to clients | Optional |
| **LLM Summarizer** | All tagged articles (from both paths) per client | Per-client/project narrative summaries | Yes |
| **Highlight Selector** | All client summaries | Top highlights across all clients with brief analysis | Yes |
| **Email Composer** | Highlights + per-client summaries + metadata | Formatted HTML email with source links | No |
| **Email Sender** | Composed email + subscriber list | Delivered newsletter | No |
| **State Update** | Processed article URLs | Updated state file committed to repo | No |

---

## Dual-Path Source Model

### Why Two Paths?

Not all sources are the same. Some sources *belong to* a specific client (their own media pages), while others are general industry publications that may mention any client. These require fundamentally different handling:

```
    CLIENT-SPECIFIC SOURCES                     GENERAL SOURCES
  (client's own media pages)              (industry publications)
           │                                        │
           ▼                                        ▼
    Scrape & auto-tag                      Scrape all articles
    to known client                                 │
           │                                        ▼
           │                              Keyword / LLM match
           │                              against client registry
           │                                        │
           └──────────────┬─────────────────────────┘
                          │
                          ▼
                Tagged articles (merged)
                from both paths
                          │
                          ▼
               LLM Summarizer + Highlights
                          │
                          ▼
                     Email Digest
```

| Aspect | Client-Specific Path | General Source Path |
|--------|---------------------|--------------------|
| **Example** | Draper's news releases, "In the News" page | Defense News RSS feed |
| **Client known?** | Yes — by definition | No — needs matching |
| **Matching needed?** | No — auto-tagged to client | Yes — keyword and/or LLM |
| **Typical format** | HTML (each client's website) | RSS (industry pubs tend to have feeds) |
| **What the LLM does** | Categorize by project, assess significance | (Optionally) catch semantic mentions missed by keywords |

---

## Client Registry

### Purpose

The client registry defines *who we're tracking* and *where to look*. Each client includes their own media pages (client-specific sources) plus project definitions that drive both the keyword matcher and the LLM summarizer.

### Registry Structure

**`config/clients.json`** — Client registry with client-specific sources and projects:

```json
{
  "clients": [
    {
      "client_name": "Draper",
      "aliases": ["Draper Laboratory", "Charles Stark Draper Laboratory", "Draper Lab"],
      "industry": "Defense & Aerospace",
      "client_sources": [
        {
          "name": "News Releases",
          "url": "https://www.draper.com/media-center/news-releases",
          "type": "json_script",
          "follow_links": true
        },
        {
          "name": "In the News",
          "url": "https://www.draper.com/media-center/in-the-news",
          "type": "html_cards",
          "follow_links": true
        },
        {
          "name": "Featured Stories",
          "url": "https://www.draper.com/media-center/featured-stories",
          "type": "html_cards",
          "follow_links": true
        }
      ],
      "projects": [
        {
          "project_name": "Navy Strategic Systems",
          "keywords": ["Navy Strategic", "submarine guidance", "inertial sensor"],
          "description": "Guidance and navigation systems for Navy submarine-launched ballistic missiles"
        },
        {
          "project_name": "Microelectronics",
          "keywords": ["microelectronics", "chip design", "NEMC", "Northeast Microelectronics Coalition"],
          "description": "Chip design, advanced packaging, and microelectronics initiatives"
        },
        {
          "project_name": "Space Systems",
          "keywords": ["space navigation", "VLEO", "lunar", "ISAM", "spacecraft navigation"],
          "description": "Space navigation, in-space servicing, and exploration systems"
        }
      ]
    }
  ]
}
```

**`config/general_sources.json`** — Industry-wide publications (separate file):

```json
{
  "general_sources": [
    {
      "name": "Defense News",
      "url": "https://www.defensenews.com/arc/outboundfeeds/rss/?outputType=xml",
      "type": "rss",
      "active": true
    }
  ]
}
```

### Matching Behavior (General Sources Only)

- **Client-level match**: Article mentions the client name or any alias
- **Project-level match**: Article mentions any project keyword
- Both are case-insensitive with word-boundary matching
- A single article can match multiple clients or multiple projects
- Client-level matches (name/alias but no specific project keyword) are grouped under a "General" category

### Client-Specific Source Behavior

- All articles scraped from a client's own pages are auto-tagged to that client
- The LLM categorizes them by project during the summarization stage
- No keyword matching is needed — the client association is inherent

---

## General Source Configuration

### Purpose

General sources are industry-wide publications that may mention any tracked client. These are scraped broadly, then matched against the client registry.

### Source Types

| Type | Library | How It Works | Best For |
|------|---------|--------------|----------|
| **RSS** | `feedparser` | Parses structured RSS/Atom feed; articles come with title, link, date, and sometimes full text | General industry publications (e.g., Defense News — includes full article text) |
| **JSON-in-HTML** | `requests` + `BeautifulSoup` + `json` | Extracts article metadata from embedded `<script>` JSON tags | Pages using search-based frontends (e.g., Draper News Releases) |
| **HTML cards** | `requests` + `BeautifulSoup` | Extracts article metadata from `<article>` card elements using CSS selectors | Structured listing pages (e.g., Draper In the News, Featured Stories) |

All client-specific sources require **link-following** after list extraction. Article body text is extracted from detail/external pages using **trafilatura**, a generic article extraction library.

All types produce the same `Article` output. Everything downstream is source-type-agnostic. Full CSS selectors and extraction details are in `SCRAPING_SPECS.md`.

---

## Matching Approach (General Sources)

### MVP: Keyword Matching

The Keyword Matcher runs only on articles from general sources. It scans each article's title and body text for keywords from the client registry.

| Matching Rule | Description |
|---------------|-------------|
| **Case-insensitive** | "Draper" matches "draper" and "DRAPER" |
| **Whole-word by default** | "Draper" should not match "Draperidge" — use word boundary matching |
| **Multi-match allowed** | One article can match multiple clients/projects |
| **Client-level fallback** | If an article matches a client name but no specific project, it's filed under "General" |
| **Match context** | Store the sentence(s) where the keyword was found |

### Future Enhancement: LLM-Assisted Matching

A future upgrade could add LLM-powered semantic matching on general sources to catch indirect references (e.g., "the Cambridge-based defense lab" → Draper). This is out of scope for MVP but the architecture supports it — the Matcher stage can be augmented without affecting other stages.

---

## Newsletter Structure

### Relevant Highlights (Top Section)

The LLM selects the 3–5 most significant mentions from across all clients and writes a brief analytical paragraph for each. Selection criteria:

- Strategic significance (contract wins, partnerships, major milestones)
- Competitive implications (new entrants, lost bids, market shifts)
- Timeliness (breaking news over routine updates)
- Cross-client relevance (industry trends affecting multiple clients)

Each highlight includes:
- A 2–3 sentence analysis explaining why it matters
- The client and project it relates to
- Link to the original source

### Per-Client Sections

For each client with content that week:

- **Client header** with the client name and total number of articles
- **Per-project subsections** with:
  - Project name
  - Summary of what was found (LLM-generated, 100–200 words per project)
  - List of source articles with titles and links
  - Source origin noted (client media page vs. industry publication)
- **General mentions** (articles that don't map to a specific project)

### Quiet Week Handling

If no content is found in a given week, send a brief "quiet week" notice confirming the system ran successfully.

---

## Email Format

### Structure

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Client Intelligence Digest — Week of {Date Range}
   {N} sources scanned · {M} articles found · {C} clients covered
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🔔 THIS WEEK'S HIGHLIGHTS

1. {Highlight Title}
   {Client Name} · {Project Name}
   {2-3 sentence analysis}
   🔗 Source

   ─────────────────────────────────

2. {Highlight Title}
   ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 CLIENT DETAILS

┌─ DRAPER (6 articles)
│
│  📁 Navy Strategic Systems (2 articles)
│     {100-200 word summary}
│     • Article Title — Source Name 🔗
│     • Article Title — Source Name 🔗
│
│  📁 Microelectronics (1 article)
│     {100-200 word summary}
│     • Article Title — Source Name 🔗
│
│  📁 General (3 articles)
│     {Brief summary}
│     • Article Title — Source Name 🔗
│     • Article Title — Source Name 🔗
│     • Article Title — Source Name 🔗
│
└─────────────────────────────────

┌─ {NEXT CLIENT} ({N} articles)
│  ...
└─────────────────────────────────

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 This Week's Stats
   Client sources scraped: {N}
   General sources scraped: {N}
   Total articles processed: {A}
   Clients covered: {C}
   Date range: {start} – {end}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Subscriber Management

### MVP (Now)

- Subscriber list stored in a JSON config file
- Single newsletter shared across all subscribers
- Adding a subscriber = editing the config file

### Future Enhancements

- Per-subscriber client filtering (only show clients relevant to them)
- Unsubscribe link in emails
- Frequency preferences (weekly, biweekly, monthly)
- Role-based views (executive summary vs. detailed)

### Subscriber Schema

```json
{
  "subscribers": [
    {
      "email": "accountmanager@example.com",
      "name": "Account Manager",
      "active": true
    }
  ]
}
```

---

## State Management

### Purpose

The system tracks which articles have already been processed to avoid re-surfacing the same content week after week.

### Approach

A lightweight JSON file committed to the repository after each run.

```json
{
  "last_run": "2026-04-01T12:00:00Z",
  "processed_articles": [
    {
      "url": "https://www.draper.com/news/article-123",
      "url_hash": "a1b2c3d4",
      "title": "Draper Wins Navy Contract",
      "source": "Draper — News Releases",
      "source_path": "client",
      "processed_date": "2026-04-01",
      "client": "Draper"
    }
  ]
}
```

### Behavior

| Scenario | Action |
|----------|--------|
| New article not in state | Process normally |
| Article URL already in state | Skip — already reported in a previous digest |
| State file missing or corrupted | Treat as fresh run (process everything in date range) |
| Manual re-run within the same week | State prevents duplicate reporting |

### Retention

Keep the last 90 days of processed articles in state. Older entries are pruned on each run.

---

## Success Criteria

| Category | Criteria |
|----------|----------|
| **Source Scraping** | All configured sources (client-specific and general) are fetched reliably |
| **Client Coverage** | Client-specific source articles are correctly auto-tagged |
| **Matching Quality** | Keyword matcher on general sources accurately identifies client mentions |
| **Summary Quality** | Per-client summaries are accurate, concise, and capture key information |
| **Highlight Quality** | Highlights reflect genuinely important developments |
| **Email Delivery** | Newsletter arrives weekly at a consistent time; formatting renders correctly |
| **State Management** | No duplicate articles across consecutive weeks; state file stays healthy |
| **Reliability** | Agent runs unattended via GitHub Actions without failures for 4+ consecutive weeks |
| **Cost** | Total weekly LLM cost under $2.00 |
| **Latency** | Full pipeline completes within 15 minutes |

---

## Constraints & Assumptions

| Constraint | Details |
|------------|---------|
| **Client sources** | HTML pages on client websites; no login or paywall required |
| **General sources** | Public RSS feeds and HTML pages; no paywalled content |
| **Source access** | Rate-limit-friendly fetching (~3s between requests) |
| **Matching** | Keyword-based for MVP on general sources; LLM-assisted matching deferred |
| **LLM provider** | Anthropic (Claude) for summarization and highlight selection |
| **Email delivery** | Gmail SMTP with App Password |
| **Scheduling** | GitHub Actions cron (weekly) |
| **Subscribers** | Shared newsletter across all subscribers |
| **Language** | English-language sources only |
| **State storage** | JSON file committed to repository; no external database |

---

## Out of Scope (Deferred)

- LLM-powered semantic matching on general sources
- Per-subscriber client filtering
- Web UI or dashboard
- Paywalled source access
- JavaScript-rendered page scraping (would need Playwright)
- Slack/Teams integration
- Sentiment analysis on mentions
- Historical trend tracking or charts
- Competitor monitoring
- Automatic source discovery
- Non-English source support
- Real-time alerts

---

## Resolved Design Decisions

### Backfill Mode

The first run will use an extended lookback window to catch up on content since January 1, 2026. After the initial backfill, the system switches to weekly mode.

| Mode | `LOOKBACK_DAYS` | Coverage |
|------|-----------------|----------|
| **Backfill** | `95` (adjusted for actual date) | Jan 1, 2026 – present |
| **Weekly** | `7` (default) | Previous 7 days |

Featured Stories (which have no reliable dates) are always included regardless of the lookback window — state deduplication prevents re-processing.

The Defense News RSS feed only retains ~50 recent articles (~3-5 days), so the backfill cannot recover older general source matches. This is acceptable since the Draper-specific sources cover the primary content.

### Scheduling & Timing

**Newsletter schedule:** Run every Monday at **7:00 AM ET**, covering the previous 7 days.

| Period Covered | Newsletter Runs |
|----------------|-----------------|
| Mon Mar 24 – Sun Mar 30 | Monday Mar 31, 7:00 AM ET |
| Mon Mar 31 – Sun Apr 6 | Monday Apr 7, 7:00 AM ET |

**GitHub Actions cron (UTC):** `0 12 * * 1` (12:00 UTC = 7:00 AM ET, every Monday)

### Dual-Path Architecture

Client-specific sources and general sources serve fundamentally different purposes:
- Client sources tell us "what is this client publishing about themselves?"
- General sources tell us "what is the industry saying about this client?"

Keeping them as two distinct paths with a merge point is cleaner than trying to force them into one unified scraping model. The merge happens naturally when all articles are tagged to clients and ready for summarization.

### Keyword Matching on General Sources Only

Client-specific source articles don't need matching — they're auto-tagged by definition. Keyword matching only runs on general source articles, keeping the matcher focused and efficient.

### State File in Repository

Storing state as a JSON file committed to the repo is the simplest approach with zero infrastructure. The GitHub Actions workflow needs write access to push the state commit.

### Error Handling

- If individual sources fail, the pipeline continues with remaining sources and notes failures in the email footer
- If the pipeline itself fails, an error notification email is sent
- If no content is found, a "quiet week" notice is sent

---

## Next Steps

1. ✅ Scope document finalized
2. Create `ARCHITECTURE.md` with detailed component design and prompt specifications
3. Create `TECH_STACK.md` with all dependencies and environment variables
4. Create `EXECUTION_PLAN.md` with step-by-step build plan
5. Begin implementation
