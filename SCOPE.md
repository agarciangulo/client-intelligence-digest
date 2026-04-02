# Scope — Client Intelligence Digest Agent

## Objective

Build an automated agent that monitors a fixed set of public sources (publications, newsletters, websites) on a weekly basis, identifies any mentions of tracked clients and their projects via keyword matching, and delivers a curated email newsletter featuring:

1. **Relevant Highlights** — The most important and noteworthy mentions across all clients, with context and analysis
2. **Per-Client Sections** — Organized by client, then by project, with summaries of all mentions and direct links to original sources

The agent runs weekly via GitHub Actions with zero manual intervention.

---

## Core Pipeline

```
┌─────────────┐     ┌──────────────────┐     ┌───────────────────┐     ┌──────────────────┐     ┌──────────────┐     ┌──────────────┐
│  SCHEDULER   │────▶│  SOURCE SCRAPER   │────▶│  KEYWORD MATCHER   │────▶│  LLM SUMMARIZER   │────▶│  EMAIL        │────▶│  STATE        │
│  (GH Actions │     │  (RSS + HTML)     │     │  (client registry) │     │  + HIGHLIGHT       │     │  COMPOSER &   │     │  UPDATE       │
│   weekly)    │     │  fixed source list│     │  keyword-based     │     │  SELECTOR          │     │  SENDER       │     │  (JSON file)  │
└─────────────┘     └──────────────────┘     └───────────────────┘     └──────────────────┘     └──────────────┘     └──────────────┘
```

### Stage Descriptions

| Stage | Input | Output | LLM? |
|-------|-------|--------|------|
| **Source Scraper** | Fixed source list (RSS feeds + HTML pages) | Articles with title, URL, body text, publish date, source name | No |
| **Keyword Matcher** | All scraped articles + Client registry | Matched articles grouped by client → project, with matched keywords | No |
| **LLM Summarizer** | Matched articles per client | Per-client/project narrative summaries | Yes |
| **Highlight Selector** | All client summaries | Top highlights across all clients with brief analysis | Yes |
| **Email Composer** | Highlights + per-client summaries + metadata | Formatted HTML email with source links | No |
| **Email Sender** | Composed email + subscriber list | Delivered newsletter | No |
| **State Update** | Processed article URLs | Updated state file committed to repo | No |

---

## Client Registry

### Purpose

The client registry defines *what the agent is looking for*. Every matched article traces back to a client and project in this registry. Without it, the keyword matcher has nothing to match against.

### Registry Structure

```json
{
  "clients": [
    {
      "client_name": "Acme Defense Systems",
      "aliases": ["Acme Defense", "Acme DS", "ADS"],
      "industry": "Defense & Aerospace",
      "projects": [
        {
          "project_name": "Project Falcon",
          "keywords": ["Project Falcon", "Falcon UAV", "Falcon drone program"],
          "description": "Next-generation autonomous UAV platform"
        },
        {
          "project_name": "Shield Network",
          "keywords": ["Shield Network", "ShieldNet", "Acme cybersecurity platform"],
          "description": "Enterprise cybersecurity monitoring system"
        }
      ]
    },
    {
      "client_name": "Northgate Industries",
      "aliases": ["Northgate", "NGI"],
      "industry": "Defense Contracting",
      "projects": [
        {
          "project_name": "Titan Upgrade Program",
          "keywords": ["Titan Upgrade", "Titan modernization", "Titan program"],
          "description": "Vehicle fleet modernization contract"
        }
      ]
    }
  ]
}
```

### Matching Behavior

- **Client-level match**: Article mentions the client name or any alias
- **Project-level match**: Article mentions any project keyword
- Both are case-insensitive
- A single article can match multiple clients or multiple projects within the same client
- Client-level matches (name/alias but no specific project keyword) are grouped under a "General" category for that client

---

## Source Configuration

### Purpose

The source list defines *where the agent looks*. Each source is a publicly accessible URL that the agent scrapes on a fixed schedule.

### Source Structure

```json
{
  "sources": [
    {
      "name": "Defense News",
      "url": "https://www.defensenews.com/rss/",
      "type": "rss",
      "active": true
    },
    {
      "name": "Industry Weekly Blog",
      "url": "https://www.example.com/news",
      "type": "html",
      "selectors": {
        "article_list": "div.article-card",
        "title": "h2.title a",
        "link": "h2.title a@href",
        "date": "span.date",
        "body": "div.article-body"
      },
      "active": true
    }
  ]
}
```

### Source Types

| Type | Library | How It Works | Best For |
|------|---------|--------------|----------|
| **RSS** | `feedparser` | Parses structured RSS/Atom feed; articles come with title, link, summary, date | Publications with RSS feeds (most news sites) |
| **HTML** | `requests` + `BeautifulSoup` | Fetches the page, extracts articles using CSS selectors defined per source | Sites without RSS feeds; blogs, press release pages |

Both types produce the same output: a list of `Article` objects with normalized fields. Everything downstream is source-type-agnostic.

---

## Matching Approach

### MVP: Keyword Matching

The Keyword Matcher is a deterministic, non-LLM stage. It scans each article's title and body text for keywords from the client registry.

| Matching Rule | Description |
|---------------|-------------|
| **Case-insensitive** | "Project Falcon" matches "project falcon" and "PROJECT FALCON" |
| **Whole-word by default** | "Titan" should not match "Titanium" — use word boundary matching |
| **Multi-match allowed** | One article can match multiple clients/projects |
| **Client-level fallback** | If an article matches a client name but no specific project, it's filed under "General" |
| **Match context** | Store the sentence(s) where the keyword was found — useful for the LLM summarizer |

### Future Enhancement: Semantic Matching

A future upgrade could add LLM-powered semantic matching to catch indirect references (e.g., "the aerospace giant's latest drone program" → Acme Defense / Project Falcon). This is out of scope for MVP but the architecture supports it — the Keyword Matcher stage can be swapped or augmented without affecting downstream stages.

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

For each client with matches that week:

- **Client header** with the client name and total number of mentions
- **Per-project subsections** with:
  - Project name
  - Summary of what was found (LLM-generated, 100–200 words per project)
  - List of source articles with titles and links
- **General mentions** (client-level matches that don't map to a specific project)

### Quiet Week Handling

If no matches are found in a given week, send a brief "quiet week" notice confirming the system ran successfully. This distinguishes "nothing found" from "the system broke."

---

## Email Format

### Structure

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Client Intelligence Digest — Week of {Date Range}
   {N} sources scanned · {M} articles matched · {C} clients mentioned
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

┌─ ACME DEFENSE SYSTEMS (4 mentions)
│
│  📁 Project Falcon (2 mentions)
│     {100-200 word summary}
│     • Source Article Title — 🔗 Link
│     • Source Article Title — 🔗 Link
│
│  📁 Shield Network (1 mention)
│     {100-200 word summary}
│     • Source Article Title — 🔗 Link
│
│  📁 General (1 mention)
│     {Brief summary}
│     • Source Article Title — 🔗 Link
│
└─────────────────────────────────

┌─ NORTHGATE INDUSTRIES (2 mentions)
│  ...
└─────────────────────────────────

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 This Week's Stats
   Sources scanned: {N}
   Articles processed: {A}
   Articles matched: {M}
   Clients mentioned: {C}
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

Unlike the ArXiv digest (which is fully stateless), this system needs to track which articles have already been processed to avoid re-surfacing the same content week after week.

### Approach

A lightweight JSON file committed to the repository after each run.

```json
{
  "last_run": "2026-04-01T11:00:00Z",
  "processed_articles": [
    {
      "url": "https://www.defensenews.com/article/12345",
      "url_hash": "a1b2c3d4",
      "title": "Acme Defense Wins $500M Contract",
      "source": "Defense News",
      "processed_date": "2026-04-01",
      "matched_clients": ["Acme Defense Systems"]
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

Keep the last 90 days of processed articles in state. Older entries are pruned on each run to prevent the file from growing indefinitely.

---

## Success Criteria

| Category | Criteria |
|----------|----------|
| **Source Scraping** | All configured sources are fetched reliably; both RSS and HTML types work |
| **Matching Quality** | Client/project mentions are accurately identified with low false positives |
| **Summary Quality** | Per-client summaries are accurate, concise, and capture the key information |
| **Highlight Quality** | Highlights reflect genuinely important developments, not routine mentions |
| **Email Delivery** | Newsletter arrives weekly at a consistent time; formatting renders correctly |
| **State Management** | No duplicate articles across consecutive weeks; state file stays healthy |
| **Reliability** | Agent runs unattended via GitHub Actions without failures for 4+ consecutive weeks |
| **Cost** | Total weekly LLM cost under $2.00 |
| **Latency** | Full pipeline completes within 15 minutes |

---

## Constraints & Assumptions

| Constraint | Details |
|------------|---------|
| **Data sources** | Fixed list of public URLs (RSS feeds and HTML pages); no paywalled content |
| **Source access** | All sources are publicly accessible; rate-limit-friendly fetching (~3s between requests) |
| **Matching** | Keyword-based for MVP; semantic matching deferred |
| **LLM provider** | Anthropic (Claude) for summarization and highlight selection |
| **Email delivery** | Gmail SMTP with App Password |
| **Scheduling** | GitHub Actions cron (weekly) |
| **Subscribers** | Shared newsletter across all subscribers (per-subscriber filtering is future) |
| **Language** | English-language sources only |
| **State storage** | JSON file committed to repository; no external database |

---

## Out of Scope (Deferred)

- Semantic / LLM-powered matching (beyond keyword matching)
- Per-subscriber client filtering
- Web UI or dashboard
- Paywalled source access
- Slack/Teams integration
- Sentiment analysis on mentions
- Historical trend tracking or charts
- Competitor monitoring (clients' competitors)
- Automatic source discovery
- Non-English source support
- Real-time alerts (immediate notification on high-priority mentions)

---

## Resolved Design Decisions

### Scheduling & Timing

**Newsletter schedule:** Run every Monday at **7:00 AM ET**, covering the previous 7 days of publications.

| Period Covered | Newsletter Runs |
|----------------|-----------------|
| Mon Mar 24 – Sun Mar 30 | Monday Mar 31, 7:00 AM ET |
| Mon Mar 31 – Sun Apr 6 | Monday Apr 7, 7:00 AM ET |

**GitHub Actions cron (UTC):** `0 12 * * 1` (12:00 UTC = 7:00 AM ET, every Monday)

### Why Monday Morning

The account manager starts the week with a full picture of what happened with clients over the past week. This gives them context before any Monday meetings or calls.

### Keyword Matching Over LLM Matching (MVP)

Keyword matching is fast, deterministic, free (no LLM cost), and easy to debug. For a known set of clients and projects with specific names, keyword matching captures the vast majority of relevant mentions. Semantic matching can be layered on later without changing the pipeline structure.

### State File in Repository

Storing state as a JSON file committed to the repo is the simplest approach with zero infrastructure. The tradeoff is that the GitHub Actions workflow needs write access to push commits. This is acceptable for a single-user automation tool and avoids external databases or cloud storage.

### Separate RSS and HTML Adapters

Rather than building one scraper that handles everything, the system uses two clean adapters (RSS and HTML) behind a common interface. This keeps each adapter simple and makes it easy to add new source types later (e.g., API-based sources).

### Duplicate Handling

Articles are deduplicated by URL. If the same article appears across multiple sources (e.g., syndicated content), it is processed only once but credits all sources.

### Error Handling

If the pipeline fails at any stage, the system sends an **error notification email** to the subscriber list with:
- Which stage failed
- Error message / traceback summary
- Timestamp
- Note that the digest will retry on the next scheduled run

If individual sources fail (e.g., a website is temporarily down), the pipeline continues with the remaining sources and notes which sources were unreachable in the email footer.

---

## Next Steps

1. ✅ Scope document finalized
2. Create `ARCHITECTURE.md` with detailed component design and prompt specifications
3. Create `TECH_STACK.md` with all dependencies and environment variables
4. Create `EXECUTION_PLAN.md` with step-by-step build plan
5. Begin implementation
