# Build Summary — Client Intelligence Digest Agent

A technical summary of the project for reference when writing about the build.

---

## What It Does

An automated agent that monitors a fixed set of public sources (RSS feeds, newsletters, websites) every week, identifies mentions of tracked clients and their projects via keyword matching, and delivers a curated email digest every Monday morning — fully automated, zero manual intervention.

The digest has two tiers:

- **Relevant Highlights (Top Section)**: The 3–5 most significant developments across all clients, each with a brief analytical paragraph explaining why it matters and what the account manager should consider doing about it.
- **Per-Client Sections**: Organized by client, then by project. Each project section has a 100–200 word narrative summary of what sources reported that week, plus direct links to every original article.

Every mention traces back to a specific source article with a clickable link.

---

## The Pipeline

The system is a **linear pipeline** — no databases (beyond a state file), no servers, no queues. Just Python functions called in sequence, triggered by a cron job.

```
GitHub Actions (7 AM ET, every Monday)
    │
    ▼
1. SOURCE SCRAPER — Fetch articles from all configured sources
    │                 (RSS via feedparser, HTML via requests + BeautifulSoup)
    │                 Filter to last 7 days, dedup, skip already-processed
    ▼
2. KEYWORD MATCHER — Scan articles for client/project mentions
    │                  (case-insensitive, word-boundary matching)
    │                  → matched articles grouped by client → project
    ▼
3a. PER-CLIENT SUMMARIZER — For each client with matches:
    │   Send matched articles to Claude → 100-200 word narrative
    │   summary per project
    │
3b. HIGHLIGHT SELECTOR — Given all client summaries:
    │   Select 3-5 most significant developments → 2-3 sentence
    │   analysis per highlight
    ▼
4. EMAIL COMPOSER — Render everything into an HTML email
    │                  via Jinja2 templates
    ▼
5. EMAIL SENDER — Deliver via Gmail SMTP
    │
    ▼
6. STATE UPDATE — Record processed article URLs → commit state
                   file back to repository
```

Total runtime: **~2–5 minutes**. Total weekly cost: **~$0.50–1.50** (almost entirely Anthropic API usage).

---

## Key Technical Decisions (and Why)

### Two-Adapter Source Scraper

Sources come in two flavors, and each gets its own clean adapter:

- **RSS feeds** are parsed via `feedparser`, which handles RSS 2.0, Atom, and RDF formats with automatic date normalization and encoding tolerance.
- **HTML pages** are scraped via `requests` + `BeautifulSoup`, with per-source CSS selectors defined in the config file (so adding a new HTML source is a config change, not a code change).

Both adapters produce identical `Article` objects. Everything downstream is source-type-agnostic.

### Keyword Matching Over LLM Matching

The Keyword Matcher is a deterministic, non-LLM stage. It uses Python's `re` module with word-boundary matching (`\b...\b`, case-insensitive) to scan article titles and body text against client names, aliases, project names, and keywords.

This was chosen over LLM-based semantic matching because:
- It's free (no API cost)
- It's fast (~1-5 seconds for hundreds of articles)
- It's deterministic (same input → same output)
- It's debuggable (you can see exactly which keyword triggered a match)
- For known client and project names, keyword matching catches the vast majority of relevant mentions

The tradeoff: it misses indirect references ("the aerospace giant" instead of the client name). Semantic matching can be layered on later without changing the pipeline structure.

### Claude for Summarization, Not for Matching

The LLM is used where it adds the most value: turning matched articles into concise, useful narratives. The per-client summarizer writes 100–200 word briefings per project. The highlight selector identifies the 3–5 most strategically significant developments and writes brief analyses.

The model used is Claude Sonnet — consistent across both LLM stages. The number of LLM calls varies per run based on how many clients have matches (one call per client + one for highlights).

### Client Registry as a First-Class Concept

The matching isn't generic keyword search. It's driven by a structured JSON registry that defines:

- **Clients** with names and aliases (catches "Acme Defense", "Acme DS", "ADS")
- **Projects** with keywords and descriptions (catches "Project Falcon", "Falcon UAV")
- **Grouping**: matches are organized by client → project → articles, which drives the email structure

This registry is the single source of truth for "what to look for." Adding a new client or project is a config change.

### State File for Cross-Week Deduplication

Unlike the ArXiv digest (which is fully stateless), this system needs to remember which articles it has already processed. The state is a JSON file (`state/processed.json`) committed to the repository after each run. It stores URL hashes, titles, sources, and matched clients for the last 90 days.

This approach:
- Requires zero infrastructure (no database)
- Is git-native (history is tracked automatically)
- Survives system failures (just re-run; state protects against duplicates)
- Self-prunes (entries older than 90 days are cleaned up each run)

The tradeoff: the GitHub Actions workflow needs write access to push the state commit. This is a small addition to the workflow configuration.

### Shared LLM Client with Retry Logic

All Claude API calls go through a single `call_claude()` function that handles:

- Exponential backoff (5s → 10s → 20s → 40s → 60s cap)
- Random jitter (0–50% of delay) to prevent thundering herd
- Automatic retry on 429 (rate limit) and 529 (overloaded) errors
- Up to 5 retry attempts before failing

### Gmail SMTP — Simplest Thing That Works

Email delivery uses Python's built-in `smtplib` with a Gmail App Password. No SendGrid, no Resend, no AWS SES. For a small subscriber list, this is zero-cost and minimal code. The tradeoff: no analytics, no fancy unsubscribe links, 500 emails/day cap. All acceptable at this scale.

### Graceful Partial Failures

The pipeline is designed to keep going when individual pieces fail:

- One source is down? Skip it, process the rest, note it in the footer.
- One client's summarization fails? Skip that client, send the digest without it.
- No matches at all? Send a "quiet week" notice instead of nothing.

Only pipeline-level failures (LLM API completely unreachable, SMTP broken) trigger an abort with an error email.

---

## Architecture at a Glance

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Language | Python 3.12 | Richest scraping/AI ecosystem |
| LLM | Claude Sonnet | Good summarization quality at reasonable cost |
| RSS parsing | feedparser | Handles all feed formats, tolerant of malformed XML |
| HTML scraping | requests + BeautifulSoup | Lightweight, no browser binary needed |
| Email | Gmail SMTP (stdlib) | Zero cost, minimal code |
| Scheduling | GitHub Actions cron | Free, zero infrastructure |
| Orchestration | Plain Python functions | Pipeline is linear — a framework would add complexity with no benefit |
| State | JSON file in repo | Zero infrastructure; git-native |
| Matching | Keyword (regex) | Fast, free, deterministic |
| Total dependencies | 6 runtime, 3 dev-only | Minimal footprint |

---

## Numbers

| Metric | Value |
|--------|-------|
| Sources monitored | Configurable (RSS + HTML mix) |
| LLM calls per run | Variable (1 per matched client + 1 highlights) |
| Per-project summary length | 100–200 words |
| Highlight analysis length | 2–3 sentences each |
| Pipeline runtime | ~2–5 minutes |
| Weekly cost | ~$0.50–1.50 |
| Monthly cost | ~$2–6 |
| Lines of Python | ~930 (excluding tests and templates) |
| Runtime dependencies | 6 |
| External services | 3 (GitHub Actions, Anthropic, Gmail) |
| Infrastructure to maintain | None (state file is self-managing) |

---

## The Build Process

The project is built in 8 phases following a "Build → Verify → Advance" philosophy, with checkpoints after every meaningful piece of work:

1. **Project setup** — Structure, config files, logging, dev caching utilities
2. **Source scraper** — RSS adapter, HTML adapter, rate limiting, state filtering
3. **Keyword matcher** — Word-boundary matching, context extraction, grouping by client/project
4. **State manager** — Load, update, prune, save, and git-commit state file
5. **LLM stages** — Per-client summarizer, highlight selector, shared Claude client
6. **Email system** — Jinja2 templates, SMTP sender, error/quiet week emails
7. **Pipeline orchestrator** — Wire everything together, DRY_RUN mode, error handling
8. **GitHub Actions deployment** — Cron workflow, secrets configuration, manual trigger testing

Each phase includes unit tests (mocked, fast, free) and live checkpoint tests (real APIs, manual verification). The dev caching system allows replaying intermediate results during development without re-running expensive stages.

---

## Interesting Design Challenges

### Heterogeneous Source Scraping

Unlike the ArXiv system (one API, one data format), this system scrapes multiple sources with different structures. RSS feeds are well-structured, but HTML pages vary wildly. The solution is per-source CSS selectors in the config file, making the HTML adapter a generic extraction engine driven by configuration rather than hard-coded selectors.

The tradeoff: each new HTML source requires some upfront work to identify the right selectors. But once configured, it runs hands-off.

### Word Boundary Matching

Naive keyword matching ("does the text contain this string?") produces false positives: "Titan" matching "Titanium", "AI" matching "FAIR" or "email". Word-boundary regex (`\bTitan\b`) solves this cleanly. The matcher also handles multi-word phrases and aliases, so "Acme Defense Systems" and "ADS" both trigger a match for the same client.

### State File Growth

Left unchecked, the state file would grow indefinitely. The 90-day pruning ensures the file stays manageable (~2-3 KB per week of entries) while retaining enough history to prevent duplicates across any reasonable re-processing window.

### Cross-Client Article Matching

A single article can mention multiple clients or multiple projects for the same client. The matcher handles this by creating separate Match objects for each client/project pair. The grouping logic then presents the article under every client/project it mentions, with appropriate context snippets for each.

---

## What's Not Built (Yet)

- Semantic / LLM-powered matching (beyond keyword matching)
- Per-subscriber client filtering
- Sentiment analysis on mentions
- Historical trend tracking or dashboards
- Slack/Teams delivery
- Real-time alerts for high-priority mentions
- Competitor monitoring
- Source health monitoring (detecting when a source goes stale)
- JavaScript-rendered page support (would need Playwright)

The architecture is intentionally simple enough that any of these could be added without a rewrite. The pipeline is linear, the components are decoupled, and the configuration is externalized.

---

## Project Structure

```
client-intelligence-digest/
├── .github/workflows/
│   └── weekly_digest.yml          # GitHub Actions cron job
├── config/
│   ├── sources.json               # RSS feeds and HTML scraping configs
│   ├── clients.json               # Client registry with projects/keywords
│   └── subscribers.json           # Email recipient list
├── state/
│   └── processed.json             # Tracked processed articles
├── src/
│   ├── main.py                    # Pipeline orchestrator
│   ├── scraper.py                 # Source Scraper (RSS + HTML adapters)
│   ├── matcher.py                 # Keyword Matcher
│   ├── summarizer.py              # LLM: Per-client summaries
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
│   └── quiet_week_email.html      # "No matches this week" template
├── tests/                         # Unit tests for all modules
├── requirements.txt
└── .env.example
```
