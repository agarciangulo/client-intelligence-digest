# Build Summary — Client Intelligence Digest Agent

A technical summary of the project for reference when writing about the build.

---

## What It Does

An automated agent that monitors two types of public sources every week:

1. **Client-specific sources** — Each tracked client's own media pages (press releases, news mentions, feature stories). These are auto-tagged to the client.
2. **General industry sources** — Broad publications like Defense News where any tracked client might be mentioned. These are keyword-matched against a client registry.

Both paths converge into a unified set of client-tagged articles. An LLM then summarizes what each client has been up to (organized by project) and selects the week's top highlights. The result is a curated email newsletter delivered every Monday morning — fully automated, zero manual intervention.

The digest has two tiers:

- **Relevant Highlights (Top Section)**: The 3–5 most significant developments across all clients, each with a brief analytical paragraph explaining strategic implications.
- **Per-Client Sections**: Organized by client, then by project. Each project section has a 100–200 word narrative summary plus direct links to every source article. Source origin is noted (client media vs. industry publication).

---

## The Pipeline

The system is a **linear pipeline with a dual-path front end** — no databases (beyond a state file), no servers, no queues.

```
GitHub Actions (7 AM ET, every Monday)
    │
    ▼
1a. CLIENT-SPECIFIC SCRAPER
    │  For each client: scrape their media pages (HTML)
    │  Auto-tag all articles to that client
    │
1b. GENERAL SOURCE SCRAPER
    │  Scrape industry publications (RSS + HTML)
    │  Articles are untagged at this point
    ▼
2.  KEYWORD MATCHER (general sources only)
    │  Scan general articles for client/project keywords
    │  Tag matches to clients; discard unmatched
    ▼
3.  MERGE & DEDUPLICATE
    │  Combine both paths → group by client
    ▼
4a. PER-CLIENT SUMMARIZER — For each client with articles:
    │  Send all articles to Claude → per-project summaries
    │  For client-source articles: LLM categorizes by project
    │  For general-source articles: already keyword-matched
    │
4b. HIGHLIGHT SELECTOR — Select 3-5 top developments
    ▼
5.  EMAIL COMPOSER — Jinja2 HTML template
    ▼
6.  EMAIL SENDER — Gmail SMTP
    ▼
7.  STATE UPDATE — Record processed URLs → commit to repo
```

Total runtime: **~2–6 minutes**. Total weekly cost: **~$0.50–1.50**.

---

## Key Technical Decisions (and Why)

### Dual-Path Source Architecture

The most important design decision. Sources fall into two fundamentally different categories:

- **Client-specific sources** are each client's own media pages. We already know who the articles belong to — scraping them auto-tags to that client. Example: Draper's News Releases, In the News, and Featured Stories pages.
- **General industry sources** are publications like Defense News that cover the entire industry. Articles from these need to be matched against the client registry to find relevant mentions.

Both paths produce the same `Article` objects with the same fields. After tagging, everything downstream is path-agnostic. The merge point combines articles from both paths, deduplicates by URL, and groups by client.

### Two-Phase Scraping with Link-Following

Client-specific sources are HTML pages with unique structures, but none of them include article body text on the list page itself. Scraping works in two phases:

1. **List extraction** — Get article metadata (title, URL, date) from the index page using one of three strategies: JSON-in-HTML (`<script>` tag parsing), HTML card elements (CSS selectors), or RSS feeds.
2. **Content extraction** — Follow each article's link to get the full text using `trafilatura`, a generic article extraction library that works on any web page.

This two-phase approach is necessary because Draper's "In the News" links to external sites (Semiconductor Digest, Washington Technology, etc.) with unpredictable HTML structures. trafilatura handles all of them without per-site configuration. See `SCRAPING_SPECS.md` for full details.

### Keyword Matching Only on General Sources

Client-specific source articles don't need matching — they're auto-tagged by definition. The keyword matcher only runs on general-source articles, using Python's `re` module with word-boundary matching. This is fast, free, and deterministic.

### LLM Does Double Duty for Client Sources

For general-source articles, keyword matching identifies which project they relate to before they reach the LLM. But client-source articles arrive without project tags. The per-client summarizer handles this: it receives the client's project definitions and categorizes articles into the right projects as part of the summarization call. One LLM call per client handles both categorization and summarization.

### Source Origin Tracking

Every article carries a `source_path` field ("client" or "general"). This flows through the entire pipeline and appears in the email. Industry publication mentions often carry more weight than self-published client content, so the highlight selector uses this signal when choosing top developments.

### Shared LLM Client with Retry Logic

All Claude API calls go through a single `call_claude()` function with exponential backoff and jitter. Identical to the ArXiv digest system's approach.

### State File for Cross-Week Deduplication

A JSON file (`state/processed.json`) committed to the repository after each run tracks processed article URLs. Entries older than 90 days are pruned automatically.

### Gmail SMTP — Simplest Thing That Works

Email delivery uses Python's built-in `smtplib` with a Gmail App Password. Zero cost, minimal code.

---

## Architecture at a Glance

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Language | Python 3.12 | Richest scraping/AI ecosystem |
| LLM | Claude Sonnet | Good summarization quality at reasonable cost |
| RSS parsing | feedparser | Handles all feed formats, tolerant of malformed XML |
| HTML scraping | requests + BeautifulSoup | Lightweight; list page extraction (JSON/HTML cards) |
| Article extraction | trafilatura | Generic content extraction for link-following |
| Keyword matching | Python `re` (word-boundary) | Fast, free, deterministic |
| Email | Gmail SMTP (stdlib) | Zero cost, minimal code |
| Scheduling | GitHub Actions cron | Free, zero infrastructure |
| State | JSON file in repo | Zero infrastructure; git-native |
| Orchestration | Plain Python functions | Pipeline is linear — no framework needed |
| Config | JSON files | Client registry + sources + subscribers |

---

## Numbers

| Metric | Value |
|--------|-------|
| Source types | 3 extraction strategies (json_script, html_cards, rss) + trafilatura for link-following |
| LLM calls per run | Variable (1 per client + 1 highlights) |
| Per-project summary length | 100–200 words |
| Highlight analysis length | 2–3 sentences each |
| Pipeline runtime | ~2–6 minutes |
| Weekly cost | ~$0.50–1.50 |
| Monthly cost | ~$2–6 |
| Lines of Python | ~990 (excluding tests and templates) |
| Runtime dependencies | 7 |
| External services | 3 (GitHub Actions, Anthropic, Gmail) |
| Infrastructure to maintain | None |

---

## The Build Process

8 phases following "Build → Verify → Advance":

1. **Project setup** — Structure, config files, logging, dev cache
2. **Source scraper** — Two-phase (list extraction + link-following), three strategies (JSON/HTML/RSS), trafilatura for content
3. **Keyword matcher** — Word-boundary matching on general sources only + merge logic
4. **State manager** — Load, update, prune, save, git commit
5. **LLM stages** — Per-client summarizer (with project categorization), highlight selector
6. **Email system** — Jinja2 templates with source origin badges, SMTP sender
7. **Pipeline orchestrator** — Dual-path wiring, DRY_RUN mode, error handling
8. **GitHub Actions deployment** — Weekly cron, state commit step, secrets

---

## Interesting Design Challenges

### Two Paths, One Pipeline

The dual-path model is the defining architectural feature. The key insight: client-specific sources and general sources are fundamentally different at the scraping stage (auto-tagged vs. needs matching) but produce identical outputs (tagged articles). The merge point is clean — after it, the pipeline doesn't know or care where an article came from.

### The "In the News" Hybrid

Draper's "In the News" page is a curated list of third-party articles about Draper. It's a client-specific source by configuration, but every link points to an external site (Semiconductor Digest, Washington Technology, etc.). The scraper extracts metadata from Draper's page (title, date, source name, external URL), then follows each link using trafilatura to get the actual article content. If the same article also appears in a general source RSS feed, the deduplication step catches it.

### Three Extraction Strategies, One Interface

Each source has a different page structure requiring a different extraction approach:
- **News Releases** embed article data as JSON inside a `<script>` tag (Elastic AppSearch frontend)
- **In the News** uses HTML card elements with dates, source names, and external links
- **Featured Stories** uses similar HTML cards but with no dates anywhere

All three produce the same output: a list of `(title, url, date)` tuples, normalized before content extraction. The config file defines which strategy and selectors to use per source.

### The Featured Stories Date Problem

Featured Stories have no reliable dates — not on the list page, not in the RSS feed (all dates are the feed generation timestamp), and not on the detail pages. The workaround: articles without dates always pass the date filter, and state deduplication prevents re-processing. Since Featured Stories are infrequent (~2-4/year), this is acceptable.

### Project Categorization Without Keyword Matching

Client-source articles arrive without project tags. Rather than running keyword matching on them (which would be redundant — we already know the client), the LLM categorizer assigns them to projects during the summarization call. This is more accurate than keyword matching because the LLM understands context, and it doesn't add an extra API call.

---

## What's Not Built (Yet)

- LLM-powered semantic matching on general sources
- Per-subscriber client filtering
- Sentiment analysis on mentions
- Historical trend tracking or dashboards
- Slack/Teams delivery
- Real-time alerts for high-priority mentions
- Competitor monitoring
- JavaScript-rendered page scraping (would need Playwright)
- Source health monitoring

The architecture supports all of these as incremental additions. The dual-path model, in particular, makes it easy to add new source types without affecting the downstream pipeline.

---

## Project Structure

```
client-intelligence-digest/
├── .github/workflows/
│   └── weekly_digest.yml          # GitHub Actions cron job
├── config/
│   ├── clients.json               # Client registry + client sources + projects
│   ├── general_sources.json       # Industry-wide sources
│   └── subscribers.json           # Email recipient list
├── state/
│   └── processed.json             # Tracked processed articles
├── src/
│   ├── main.py                    # Pipeline orchestrator
│   ├── scraper.py                 # Two-phase scraping: list extraction + content following
│   ├── matcher.py                 # Keyword Matcher (general sources only)
│   ├── summarizer.py              # LLM: Per-client summaries + project categorization
│   ├── highlights.py              # LLM: Highlight selection
│   ├── llm.py                     # Shared Claude client with retry logic
│   ├── email_composer.py          # Jinja2 template rendering
│   ├── email_sender.py            # Gmail SMTP delivery
│   ├── state_manager.py           # State file management
│   ├── error_handler.py           # Error capture + notification
│   ├── logger.py                  # Centralized logging
│   └── dev_cache.py               # Development caching utility
├── templates/
│   ├── digest_email.html          # Main digest email template
│   └── quiet_week_email.html      # "No content this week" template
├── tests/                         # Unit tests for all modules
├── requirements.txt
└── .env.example
```
