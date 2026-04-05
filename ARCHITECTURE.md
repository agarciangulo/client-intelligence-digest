# Architecture — Client Intelligence Digest Agent

This document defines the system architecture, component design, data flows, and prompt specifications for the Client Intelligence Digest Agent described in `SCOPE.md`.

---

## 1. System Overview

The agent is a **linear pipeline with a dual-path front end** that runs once weekly via GitHub Actions. Client-specific sources and general industry sources are scraped in parallel, tagged to clients, merged, and then summarized by an LLM.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                           CLIENT INTELLIGENCE DIGEST AGENT                                         │
├──────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                    │
│  ┌──────────────────────────────────────────────┐                                                │
│  │            SOURCE SCRAPING (DUAL-PATH)        │                                                │
│  │                                                │                                                │
│  │  ┌───────────────┐    ┌───────────────────┐   │                                                │
│  │  │ CLIENT-SPECIFIC│    │ GENERAL SOURCES   │   │                                                │
│  │  │ SOURCES        │    │ + KEYWORD MATCHER │   │                                                │
│  │  │ (auto-tagged)  │    │ (match to clients)│   │                                                │
│  │  └───────┬───────┘    └────────┬──────────┘   │                                                │
│  │          │                     │               │                                                │
│  │          └──────────┬──────────┘               │                                                │
│  │                     │                          │                                                │
│  │           ┌─────────▼─────────┐                │                                                │
│  │           │ MERGE & DEDUP     │                │                                                │
│  │           │ Tagged articles   │                │                                                │
│  │           └──────────────────┘                │                                                │
│  └──────────────────┬───────────────────────────┘                                                │
│                      │                                                                            │
│  ┌──────────────┐   │   ┌──────────────┐    ┌──────────────┐    ┌────────┐    ┌────────┐         │
│  │     LLM      │◀──┘   │    EMAIL     │    │    EMAIL     │    │  STATE │    │ ERROR  │         │
│  │  SUMMARIZER  │───────▶│  COMPOSER   │───▶│   SENDER     │───▶│ UPDATE │    │ HANDLER│         │
│  │  + HIGHLIGHT │        │             │    │              │    │        │    │        │         │
│  │  SELECTOR    │        │ HTML email  │    │ Gmail SMTP   │    │ JSON   │    │ Wraps  │         │
│  │  (Claude)    │        │ template    │    │              │    │ commit │    │ all    │         │
│  └──────────────┘        └──────────────┘    └──────────────┘    └────────┘    └────────┘         │
│                                                                                                    │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────┐    │
│  │                              CONFIGURATION LAYER                                           │    │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌──────────────────┐        │    │
│  │  │ Client Registry│  │ General        │  │ Subscribers    │  │ Environment      │        │    │
│  │  │ + Client       │  │ Sources        │  │ List           │  │ Variables        │        │    │
│  │  │ Sources (JSON) │  │ (JSON)         │  │ (JSON)         │  │                  │        │    │
│  │  └────────────────┘  └────────────────┘  └────────────────┘  └──────────────────┘        │    │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                                    │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Design Principles

| Principle | Implementation |
|-----------|----------------|
| **Dual-path, single output** | Two source paths produce the same `Article` objects, merge before analysis. |
| **Minimal state** | Only a JSON state file for article deduplication; no databases. |
| **Fail-safe** | Any stage failure triggers an error email; individual source failures don't kill the pipeline. |
| **Source-agnostic downstream** | After tagging, all articles look the same regardless of origin. |
| **Cost-conscious** | Keyword matching is free; LLM is only used for summarization and highlight selection. |
| **Simple** | No orchestration framework, no queues. Just Python functions called in sequence. |

---

## 2. Pipeline Flow — Detailed

### 2.1 End-to-End Sequence

```
GitHub Actions Cron (7 AM ET / 12:00 UTC, every Monday)
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1a. CLIENT-SPECIFIC SOURCE SCRAPER                               │
│    • For each client in the registry:                            │
│      - Scrape each of the client's configured media pages (HTML) │
│      - Auto-tag every article to that client                     │
│    • Filter articles to last 7 days                              │
│    • Filter out already-processed articles (state file)          │
│    • Output: List[Article] (pre-tagged to clients)               │
│                                                                   │
│ 1b. GENERAL SOURCE SCRAPER                                       │
│    • Scrape all general industry sources (RSS/HTML)              │
│    • Filter articles to last 7 days                              │
│    • Filter out already-processed articles (state file)          │
│    • Output: List[Article] (untagged)                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. KEYWORD MATCHER (general-source articles only, no LLM)        │
│    • Load client registry (names, aliases, project keywords)     │
│    • For each general-source article, scan title + body for      │
│      client/project keywords                                     │
│    • Tag matched articles to clients/projects                    │
│    • Discard unmatched general-source articles                   │
│    • Extract match context (surrounding sentences)               │
│    • Output: List[Article] (now tagged to clients)               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. MERGE & DEDUPLICATE                                           │
│    • Combine client-source articles + matched general articles   │
│    • Deduplicate by URL (same article from both paths)           │
│    • Group all articles by client                                │
│    • Output: Dict[client → List[Article]]                        │
│                                                                   │
│    IF 0 articles after merge → send "quiet week" email → EXIT    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4a. LLM: PER-CLIENT SUMMARIZER (LLM Calls #1–N)                 │
│    • For each client with articles:                              │
│      - Send all articles (from both paths) to Claude             │
│      - Include project definitions for categorization            │
│      - Generate per-project narrative summaries (100-200 words)  │
│    • One LLM call per client                                     │
│    • For client-source articles: LLM categorizes by project      │
│    • For general-source articles: already tagged, LLM summarizes │
│    • Output: Per-client summary with project breakdowns          │
│                                                                   │
│ 4b. LLM: HIGHLIGHT SELECTOR (LLM Call #N+1)                     │
│    • Input: All client summaries + article metadata              │
│    • Select 3-5 most significant developments                    │
│    • Write a 2-3 sentence analysis for each                      │
│    • Output: Ordered highlights list                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. EMAIL COMPOSER                                                │
│    • Input: Highlights + per-client summaries + article metadata │
│    • Template: Jinja2 HTML template                               │
│    • Render highlights section, per-client sections, stats footer │
│    • Output: Complete HTML email body                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. EMAIL SENDER                                                  │
│    • Input: HTML email + subscriber list                          │
│    • Method: Gmail SMTP with App Password                         │
│    • Sends to each active subscriber                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. STATE UPDATE                                                  │
│    • Add all processed article URLs to state/processed.json      │
│    • Prune entries older than 90 days                             │
│    • Commit updated state file back to repository                │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 LLM Call Summary

| Call # | Stage | Model | Input Size | Output Size | Purpose |
|--------|-------|-------|------------|-------------|---------|
| 1–N | Per-Client Summarizer | Claude Sonnet | ~2-8K tokens per client | ~500-1500 tokens per client | Narrative summaries + project categorization |
| N+1 | Highlight Selector | Claude Sonnet | ~2-5K tokens (all summaries) | ~1-2K tokens (3-5 highlights) | Select and analyze top highlights |

**Total LLM calls per run:** Variable — depends on number of clients with content. Typical: 3–10 clients + 1 highlight call = **4–11 calls**.

**Estimated weekly cost:** ~$0.50–1.50 depending on volume.

---

## 3. Component Details

### 3.1 Source Scraper (Dual-Path, Two-Phase)

**Responsibility:** Fetch articles from both client-specific and general sources using a two-phase approach: list extraction then content extraction.

> **Full scraping details** (CSS selectors, JSON paths, per-source behavior) are in `SCRAPING_SPECS.md`.

#### Phase 1: List Extraction

Three strategies for extracting article metadata from index/list pages:

| Strategy | Library | Used By | What It Gets |
|----------|---------|---------|-------------|
| `json_script` | BeautifulSoup + `json` | News Releases | Title, URL, date from embedded `<script>` JSON |
| `html_cards` | BeautifulSoup | In the News, Featured Stories | Title, URL, date, source name from `<article>` cards |
| `rss` | feedparser | Defense News | Title, URL, date, full body text from RSS feed |

#### Phase 2: Content Extraction (Link-Following)

All Draper sources require following links to get full article text. This uses **trafilatura** — a purpose-built library for extracting article content from arbitrary web pages.

```
For each client in registry:
    For each client_source:
        ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
        │  Fetch List Page │      │  Extract Metadata│      │  Follow Links    │      │  Tag to Client   │
        │                  │      │                  │      │                  │      │                  │
        │  requests +      │─────▶│  json_script OR  │─────▶│  trafilatura     │─────▶│  article.client  │
        │  BeautifulSoup   │      │  html_cards      │      │  extracts body   │      │  = client_name   │
        │                  │      │  (per config)    │      │  from detail/    │      │  article.source  │
        │                  │      │                  │      │  external pages  │      │  _path = "client"│
        └──────────────────┘      └──────────────────┘      └──────────────────┘      └──────────────────┘
```

For general sources with full-content RSS feeds (like Defense News), Phase 2 is skipped — the body text comes directly from the feed.

```
For each general source:
    ┌──────────────────┐      ┌──────────────────┐
    │  RSS Adapter     │      │  Normalize to    │
    │  (feedparser)    │      │  Article objects  │
    │                  │      │                  │
    │  Full body text  │─────▶│  article.client  │
    │  in feed content │      │  = None (untagged)│
    │                  │      │  article.source  │
    │                  │      │  _path = "general"│
    └──────────────────┘      └──────────────────┘
```

#### Common: Article Data Model

```python
@dataclass
class Article:
    url: str                    # Canonical URL of the article
    url_hash: str               # SHA-256 hash of URL (for state tracking)
    title: str                  # Article title
    body: str                   # Full text extracted via trafilatura or RSS content
    summary: str                # Short summary (RSS description or first ~200 chars)
    publish_date: str | None    # ISO date string, or None (Featured Stories)
    source_name: str            # e.g., "Draper — News Releases" or "Defense News"
    source_type: str            # "json_script", "html_cards", or "rss"
    source_path: str            # "client" or "general"
    external_source: str | None # Publication name for In the News articles
    client: str | None          # Pre-tagged client (for client sources) or None
    matched_projects: list[str] # Filled by matcher (general) or summarizer (client)
    matched_keywords: list[str] # Filled by matcher (general sources only)
    context_snippets: list[str] # Sentences containing matched keywords
```

**Rate Limiting:**
- 3 seconds between requests to the same domain
- 1 second between requests to different domains
- Individual source failures are caught and logged — the pipeline continues
- User-Agent: `ClientIntelligenceDigest/1.0`

---

### 3.2 Keyword Matcher

**Responsibility:** Match general-source articles against the client registry. Does NOT run on client-specific source articles (they're already tagged).

**Input:** `List[Article]` (general sources only, where `source_path == "general"`) + Client registry
**Output:** `List[Article]` with `client`, `matched_projects`, `matched_keywords`, and `context_snippets` populated

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  Client Registry │      │  For each article │      │  Tagged Articles │
│                  │      │  × each client:   │      │                  │
│  • Client names  │─────▶│                   │─────▶│  article.client  │
│  • Aliases       │      │  Scan title +     │      │  = matched client│
│  • Project names │      │  body for keywords│      │  article.matched │
│  • Keywords      │      │  (word-boundary)  │      │  _projects = [...│
│                  │      │                   │      │  ]               │
│                  │      │  Discard unmatched │      │                  │
└──────────────────┘      └──────────────────┘      └──────────────────┘
```

**Matching Logic:**
1. Only runs on articles where `source_path == "general"`
2. Uses `re.compile(r'\b' + re.escape(keyword) + r'\b', re.IGNORECASE)`
3. Extracts surrounding sentences as context snippets
4. Unmatched articles are discarded
5. Matched articles get their `client` field populated

---

### 3.3 Merge & Deduplicate

**Responsibility:** Combine articles from both paths into a single collection grouped by client.

**Input:** Client-source articles (pre-tagged) + Matched general-source articles (now tagged)
**Output:** `Dict[str, List[Article]]` — articles grouped by client name

**Deduplication:** If the same article URL appears in both paths (e.g., Draper's "In the News" page links to a Defense News article that was also scraped from the Defense News RSS feed), keep one copy and note both source origins.

---

### 3.4 LLM Stage: Per-Client Summarizer

**Responsibility:** For each client with articles, generate narrative summaries organized by project.

**Input:** All articles for one client (from both paths) + client project definitions
**Output:** Structured summary with per-project narratives (100–200 words each)
**Model:** Claude Sonnet
**Calls:** One per client with content

The summarizer has an additional job for client-source articles: **project categorization**. Since client-source articles aren't keyword-matched to projects, the LLM determines which project(s) each article relates to based on the project definitions in the registry.

```
For each client with articles:
    ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
    │  All articles    │      │   LLM Call       │      │  Client Summary  │
    │  for this client │      │   Claude Sonnet  │      │                  │
    │  (both paths)    │─────▶│                  │─────▶│  Per-project     │
    │  + Project defs  │      │  Categorize +    │      │  narratives      │
    │  + Source origin  │      │  Summarize       │      │  (100-200 words  │
    │                  │      │                  │      │   each)          │
    └──────────────────┘      └──────────────────┘      └──────────────────┘
```

---

### 3.5 LLM Stage: Highlight Selector

**Responsibility:** Review all client summaries and select the 3–5 most important findings.

**Input:** All per-client summaries + article metadata
**Output:** Ordered list of 3–5 highlights with brief analysis
**Model:** Claude Sonnet
**Calls:** 1 (always exactly one call)

---

### 3.6 Email Composer

**Responsibility:** Assemble the final HTML email from highlights, client summaries, and metadata.

**Input:** Highlights + per-client summaries + article metadata + weekly stats
**Output:** Complete HTML email string
**LLM:** None — pure template rendering

**Template Sections:**

| Section | Content | Data Source |
|---------|---------|-------------|
| Header | Digest title, date range, stats summary | Pipeline metadata |
| Highlights (3-5) | Title, client, analysis, source link | Highlight Selector output |
| Client Sections | Client header, per-project summaries, article links | Per-Client Summarizer output |
| Footer | Stats (sources scanned, articles found, etc.) | Pipeline metadata |

**Email Styling Notes:**
- Use **inline CSS** (many email clients strip `<style>` blocks)
- Single-column layout for mobile compatibility
- Web-safe fonts (Georgia, Arial, system fonts)
- Source origin badges (client media vs. industry publication)

---

### 3.7 Email Sender

**Responsibility:** Deliver the composed email to all active subscribers.

**SMTP Configuration:**

| Setting | Value |
|---------|-------|
| Server | `smtp.gmail.com` |
| Port | `587` (TLS) |
| Auth | Gmail App Password |
| From | Configured sender address |
| Subject | `📊 Client Intelligence Digest — Week of {Date Range}` |
| Content-Type | `text/html` |

---

### 3.8 State Manager

**Responsibility:** Track processed articles across runs to prevent duplicates.

**State File Schema:**

```json
{
  "last_run": "2026-04-01T12:00:00Z",
  "processed_articles": [
    {
      "url_hash": "a1b2c3d4e5f6...",
      "url": "https://www.draper.com/news/article-123",
      "title": "Article Title",
      "source": "Draper — News Releases",
      "source_path": "client",
      "processed_date": "2026-04-01",
      "client": "Draper"
    }
  ]
}
```

**State Operations:**

| Operation | When | Purpose |
|-----------|------|---------|
| **Load** | Start of pipeline | Get list of already-processed URLs |
| **Check** | During scraping | Skip articles whose URL hash exists in state |
| **Update** | After email sent | Add newly processed articles |
| **Prune** | During update | Remove entries older than 90 days |
| **Commit** | End of pipeline | Push updated state file to repository |

---

### 3.9 Error Handler

**Failure Modes & Handling:**

| Failure | Stage | Handling | User Impact |
|---------|-------|----------|-------------|
| Client source URL unreachable | Client Scraper | Log warning, skip source, continue | Partial coverage noted in footer |
| General source unreachable | General Scraper | Log warning, skip source, continue | Partial coverage |
| 0 articles after merge | Merge | Send quiet week email → exit | Gets notice |
| LLM API error | Summarizer or Highlights | Retry 3x with backoff → error email | No digest this week |
| LLM returns malformed JSON | Summarizer | Retry with stricter prompt → skip client | Partial digest |
| Gmail SMTP error | Email Sender | Retry 3x → log error | No digest; visible in GH Actions logs |
| State file corrupted | State Manager | Treat as fresh run | May re-report some articles |
| Git push fails | State Manager | Log warning, continue | Next run may re-process articles |

---

## 4. Configuration Files

### 4.1 File Structure

```
config/
├── clients.json            # Client registry + client-specific sources + projects
├── general_sources.json    # Industry-wide sources to scrape
├── subscribers.json        # Email recipient list
└── .env                    # API keys, SMTP credentials (not in repo)

state/
└── processed.json          # Tracked article URLs (committed to repo)
```

### 4.2 Clients (`config/clients.json`)

> **Full selector/extraction details** for each source are in `SCRAPING_SPECS.md`.

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
          "config": {
            "script_id": "site-news",
            "fields": {
              "title": "title.raw",
              "url": "url.raw",
              "date": "published_time.raw"
            }
          },
          "follow_links": true,
          "active": true
        },
        {
          "name": "In the News",
          "url": "https://www.draper.com/media-center/in-the-news",
          "type": "html_cards",
          "config": {
            "selectors": {
              "container": "article.media.card",
              "title": ".media-heading a",
              "link": ".media-heading a",
              "link_attribute": "href",
              "date": "time",
              "date_attribute": "datetime",
              "source_name": ".news-source"
            },
            "base_url": "https://www.draper.com"
          },
          "follow_links": true,
          "active": true
        },
        {
          "name": "Featured Stories",
          "url": "https://www.draper.com/media-center/featured-stories",
          "type": "html_cards",
          "config": {
            "selectors": {
              "container": "article.media.card",
              "title": ".media-heading a",
              "link": ".media-heading a",
              "link_attribute": "href",
              "subtitle": ".media-heading p"
            },
            "base_url": "https://www.draper.com"
          },
          "follow_links": true,
          "active": true
        }
      ],
      "projects": [
        {
          "project_name": "Navy Strategic Systems",
          "keywords": ["Navy Strategic", "submarine guidance", "inertial sensor", "ballistic missile"],
          "description": "Guidance and navigation systems for Navy submarine-launched ballistic missiles"
        },
        {
          "project_name": "Microelectronics",
          "keywords": ["microelectronics", "chip design", "NEMC", "Northeast Microelectronics Coalition", "advanced packaging"],
          "description": "Chip design, advanced packaging, and microelectronics initiatives"
        },
        {
          "project_name": "Space Systems",
          "keywords": ["space navigation", "VLEO", "lunar", "ISAM", "spacecraft", "Parker Solar"],
          "description": "Space navigation, in-space servicing, and exploration systems"
        }
      ]
    }
  ]
}
```

### 4.3 General Sources (`config/general_sources.json`)

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

### 4.4 Subscribers (`config/subscribers.json`)

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

### 4.5 Environment Variables

| Variable | Purpose | Stored In |
|----------|---------|-----------|
| `ANTHROPIC_API_KEY` | Claude API authentication | GitHub Secrets |
| `GMAIL_ADDRESS` | Sender email address | GitHub Secrets |
| `GMAIL_APP_PASSWORD` | Gmail App Password for SMTP | GitHub Secrets |

---

## 5. Prompt Specifications

Full prompt text, output schemas, and design rationale are in **`PROMPTS.md`**.

### 5.1 Summary

| Prompt | Purpose | Model | Temp | Calls |
|--------|---------|-------|------|-------|
| **Per-Client Summarizer** | Categorize articles, write project updates (refresher + new developments), flag new activity | Claude Sonnet | 0.4 | 1 per client |
| **Highlight Selector** | Select 3-5 top items across all clients for the newsletter header | Claude Sonnet | 0.3 | 1 total |

### 5.2 Key Design Decisions

- **Progress over background:** The summarizer leads with what's NEW this week, not project descriptions. A one-sentence "refresher" provides context without restating known facts.
- **New Activity as a first-class signal:** Articles that don't fit known projects are flagged as "New Activity" with an LLM-generated name and "why it matters" analysis — not dumped into a generic bucket.
- **Highlight priority order:** New/emerging activity > project progress > industry coverage > strategic developments > risks.
- **Significance levels:** High (actionable), Medium (noteworthy), Low (routine). See `PROMPTS.md` §1.4 for criteria.

---

## 6. Pipeline Timing Estimate

| Stage | Duration | Notes |
|-------|----------|-------|
| Client-Specific Scraper (list + follow) | ~60-180s | List pages + trafilatura on each article; 3s same-domain rate limit |
| General Source Scraper | ~15-60s | RSS parsing (Defense News includes full text, no link-following) |
| Keyword Matcher | ~1-5s | Local string matching, fast |
| Merge & Dedup | ~1s | In-memory operation |
| Per-Client Summarizer | ~30-90s | ~10-15s per client with content |
| Highlight Selector | ~10-15s | Single LLM call, small input |
| Email Composer | ~1s | Template rendering |
| Email Sender | ~5-10s | SMTP connection + send |
| State Update | ~10-15s | Git add + commit + push |
| **Total** | **~2-6 minutes** | Well within 15-minute budget |

---

## 7. Directory Structure

```
client-intelligence-digest/
├── .github/
│   └── workflows/
│       └── weekly_digest.yml        # GitHub Actions cron job
│
├── config/
│   ├── clients.json                 # Client registry + client sources + projects
│   ├── general_sources.json         # Industry-wide sources
│   └── subscribers.json             # Email recipient list
│
├── state/
│   └── processed.json               # Tracked processed articles
│
├── src/
│   ├── __init__.py
│   ├── main.py                      # Pipeline orchestrator (entry point)
│   ├── scraper.py                   # Source Scraper — RSS + HTML adapters
│   ├── matcher.py                   # Keyword Matcher (general sources only)
│   ├── summarizer.py                # LLM: Per-client summaries + project categorization
│   ├── highlights.py                # LLM: Highlight selection
│   ├── llm.py                       # Shared Claude client with retry logic
│   ├── email_composer.py            # HTML email assembly via Jinja2
│   ├── email_sender.py              # Gmail SMTP delivery
│   ├── state_manager.py             # State file load/save/prune/commit
│   ├── error_handler.py             # Error capture + notification
│   ├── logger.py                    # Centralized logging
│   └── dev_cache.py                 # Development caching utility
│
├── templates/
│   ├── digest_email.html            # Main digest email template
│   └── quiet_week_email.html        # "No content this week" template
│
├── tests/
│   ├── test_scraper.py
│   ├── test_matcher.py
│   ├── test_summarizer.py
│   ├── test_highlights.py
│   ├── test_email.py
│   └── test_state_manager.py
│
├── requirements.txt
├── .env.example
└── .gitignore
```

**Module Responsibilities:**

| Module | Lines (est.) | LLM? | Description |
|--------|-------------|------|-------------|
| `main.py` | ~120 | No | Orchestrates dual-path pipeline |
| `scraper.py` | ~250 | No | Two-phase scraping: list extraction (JSON/HTML/RSS) + content extraction (trafilatura) |
| `matcher.py` | ~120 | No | Keyword matching on general-source articles only |
| `summarizer.py` | ~90 | Yes | Per-client summaries with project categorization |
| `highlights.py` | ~70 | Yes | Highlight selection across all clients |
| `llm.py` | ~60 | — | Shared Claude client with exponential backoff + jitter |
| `email_composer.py` | ~70 | No | Jinja2 template rendering with source origin badges |
| `email_sender.py` | ~60 | No | SMTP connection, sends to subscriber list |
| `state_manager.py` | ~80 | No | Load, check, update, prune, commit state file |
| `error_handler.py` | ~50 | No | Try/except wrapper, error email formatting |
| `logger.py` | ~30 | No | Logging configuration |
| `dev_cache.py` | ~40 | No | Development caching for prompt iteration |

**Estimated total:** ~990 lines of Python (excluding tests and templates)

---

## 8. GitHub Actions Workflow

```yaml
# .github/workflows/weekly_digest.yml
name: Client Intelligence Digest

on:
  schedule:
    # 12:00 UTC = 7:00 AM ET, every Monday
    - cron: '0 12 * * 1'
  workflow_dispatch:

jobs:
  digest:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run digest pipeline
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GMAIL_ADDRESS: ${{ secrets.GMAIL_ADDRESS }}
          GMAIL_APP_PASSWORD: ${{ secrets.GMAIL_APP_PASSWORD }}
        run: python src/main.py

      - name: Commit state updates
        run: |
          git config user.name 'github-actions[bot]'
          git config user.email 'github-actions[bot]@users.noreply.github.com'
          git add state/processed.json
          git diff --quiet --cached || git commit -m "Update processed articles state"
          git push
```

---

## 9. Future Extension Points

| Extension | Where It Plugs In | Complexity |
|-----------|-------------------|------------|
| **LLM-assisted matching** | `matcher.py` — add LLM pass after keyword matching | Medium |
| **Additional source adapters** | `scraper.py` — new source types per `SCRAPING_SPECS.md` checklist | Low–Medium |
| **Per-subscriber client filtering** | `email_composer.py` — filter sections per recipient | Medium |
| **Sentiment analysis** | `summarizer.py` — add sentiment to project summaries | Low |
| **Historical trend tracking** | `state_manager.py` — aggregate mention counts over time | Medium |
| **Slack/Teams delivery** | Add `slack_sender.py` alongside email sender | Medium |
| **Real-time alerts** | Separate workflow for high-priority matches | High |
| **Competitor monitoring** | `clients.json` — add competitor section per client | Low |

---

## 10. Summary

| Aspect | Design Choice | Rationale |
|--------|---------------|-----------|
| **Architecture** | Dual-path scraping → merge → linear pipeline | Clean handling of two fundamentally different source types |
| **State** | JSON file committed to repo | Minimal infrastructure; git-native |
| **Client sources** | Two-phase: list extraction (JSON/HTML) + link-following (trafilatura) | Each client's website has unique structure; per-source config; trafilatura handles content from any page |
| **General sources** | RSS with full content (Defense News) | Full article text in feed; no link-following needed |
| **Matching** | Keyword-based on general sources only | Client sources don't need matching; keywords are free and fast |
| **LLM** | Claude Sonnet for summarization + highlights | Summarizer also handles project categorization for client sources |
| **LLM Calls** | Variable (1 per client + 1 highlight) | Scales with actual content volume |
| **Email** | Jinja2 template + Gmail SMTP | Simple, free, sufficient |
| **Scheduling** | GitHub Actions cron (weekly) | Free, zero-infrastructure |
| **Estimated Cost** | ~$0.50–1.50/week | Well under budget |
| **Estimated Runtime** | ~2-6 minutes | Well under 15-minute budget |
