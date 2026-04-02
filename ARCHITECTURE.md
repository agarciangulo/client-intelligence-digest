# Architecture — Client Intelligence Digest Agent

This document defines the system architecture, component design, data flows, and prompt specifications for the Client Intelligence Digest Agent described in `SCOPE.md`.

---

## 1. System Overview

The agent is a **linear pipeline** that runs once weekly via GitHub Actions. There are no persistent servers, no databases (beyond a state file), and no user-facing APIs — just a scheduled job that scrapes, matches, summarizes, and emails.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                           CLIENT INTELLIGENCE DIGEST AGENT                                         │
├──────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────┐  ┌────────┐ │
│  │   SOURCE      │  │   KEYWORD    │  │     LLM      │  │    EMAIL     │  │  EMAIL │  │  STATE │ │
│  │   SCRAPER     │─▶│   MATCHER    │─▶│  SUMMARIZER  │─▶│  COMPOSER   │─▶│ SENDER │─▶│ UPDATE │ │
│  │              │  │              │  │  + HIGHLIGHT  │  │             │  │        │  │        │ │
│  │ RSS + HTML   │  │ Client       │  │  SELECTOR    │  │ HTML email  │  │ Gmail  │  │ JSON   │ │
│  │ adapters     │  │ registry     │  │ (Claude)     │  │ template    │  │ SMTP   │  │ commit │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  └────────┘  └────────┘ │
│         │                 │                 │                 │               │            │      │
│         └─────────────────┴─────────────────┴─────────────────┴───────────────┴────────────┘      │
│                                              │                                                    │
│                                    ┌─────────▼─────────┐                                         │
│                                    │   ERROR HANDLER    │                                         │
│                                    │                    │                                         │
│                                    │ Catches failures   │                                         │
│                                    │ at any stage →     │                                         │
│                                    │ sends error email  │                                         │
│                                    └────────────────────┘                                         │
│                                                                                                    │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────┐    │
│  │                              CONFIGURATION LAYER                                           │    │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌──────────────────┐        │    │
│  │  │ Client         │  │ Sources        │  │ Subscribers    │  │ Environment      │        │    │
│  │  │ Registry       │  │ Config         │  │ List           │  │ Variables        │        │    │
│  │  │ (JSON)         │  │ (JSON)         │  │ (JSON)         │  │                  │        │    │
│  │  └────────────────┘  └────────────────┘  └────────────────┘  └──────────────────┘        │    │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                                    │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Design Principles

| Principle | Implementation |
|-----------|----------------|
| **Minimal state** | Only a JSON state file for article deduplication; no databases. |
| **Fail-safe** | Any stage failure triggers an error email; individual source failures don't kill the pipeline. |
| **Source-agnostic** | RSS and HTML sources produce identical `Article` objects; everything downstream is type-agnostic. |
| **Cost-conscious** | Keyword matching is free (no LLM); LLM is only used for summarization and highlight selection. |
| **Simple** | No orchestration framework, no queues, no microservices. Just Python functions called in sequence. |

---

## 2. Pipeline Flow — Detailed

### 2.1 End-to-End Sequence

```
GitHub Actions Cron (7 AM ET / 12:00 UTC, every Monday)
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. SOURCE SCRAPER                                                │
│    • Load source list from config/sources.json                   │
│    • For each active source:                                     │
│      - RSS → feedparser                                          │
│      - HTML → requests + BeautifulSoup with per-source selectors │
│    • Filter articles to last 7 days                              │
│    • Deduplicate by URL                                          │
│    • Filter out already-processed articles (state file)          │
│    • Output: List[Article] (new articles only)                   │
│                                                                   │
│    IF 0 new articles found → send "quiet week" email → EXIT      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. KEYWORD MATCHER (No LLM)                                      │
│    • Load client registry from config/clients.json               │
│    • For each article, scan title + body for:                    │
│      - Client names and aliases                                  │
│      - Project names and keywords                                │
│    • Case-insensitive, word-boundary matching                    │
│    • Extract match context (surrounding sentences)               │
│    • Output: List[Match] grouped by client → project             │
│                                                                   │
│    IF 0 matches found → send "quiet week" email → EXIT           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3a. LLM: PER-CLIENT SUMMARIZER (LLM Calls #1–N)                 │
│    • For each client with matches:                               │
│      - Batch all matched articles for that client                │
│      - Send to Claude with client context + match details        │
│      - Generate per-project narrative summaries (100-200 words)  │
│    • One LLM call per client (batches all projects together)     │
│    • Output: Per-client summary with project breakdowns          │
│                                                                   │
│ 3b. LLM: HIGHLIGHT SELECTOR (LLM Call #N+1)                     │
│    • Input: All client summaries + matched article metadata      │
│    • Select 3-5 most significant mentions across all clients     │
│    • Write a 2-3 sentence analysis for each highlight            │
│    • Output: Ordered highlights list                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. EMAIL COMPOSER                                                │
│    • Input: Highlights + per-client summaries + article metadata │
│    • Template: Jinja2 HTML template                               │
│    • Render highlights section, per-client sections, stats footer │
│    • Output: Complete HTML email body                             │
│    • No LLM needed — pure template rendering                     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. EMAIL SENDER                                                  │
│    • Input: HTML email + subscriber list                          │
│    • Method: Gmail SMTP with App Password                         │
│    • Sends to each active subscriber                              │
│    • Logs success/failure per recipient                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. STATE UPDATE                                                  │
│    • Add all processed article URLs to state/processed.json      │
│    • Prune entries older than 90 days                             │
│    • Commit updated state file back to repository                │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 LLM Call Summary

| Call # | Stage | Model | Input Size | Output Size | Purpose |
|--------|-------|-------|------------|-------------|---------|
| 1–N | Per-Client Summarizer | Claude Sonnet | ~2-8K tokens per client (matched articles + context) | ~500-1500 tokens per client (project summaries) | Narrative summaries per client/project |
| N+1 | Highlight Selector | Claude Sonnet | ~2-5K tokens (all client summaries) | ~1-2K tokens (3-5 highlights with analysis) | Select and analyze top highlights |

**Total LLM calls per run:** Variable — depends on number of clients with matches. Typical: 3–10 clients + 1 highlight call = **4–11 calls**.

**Estimated weekly cost:** ~$0.50–1.50 depending on match volume.

---

## 3. Component Details

### 3.1 Source Scraper

**Responsibility:** Fetch articles from all configured sources for the past 7 days.

**Input:** Source config (from `config/sources.json`) + State file (for deduplication)
**Output:** `List[Article]` — new, deduplicated articles from the past week

```
                    ┌──────────────────┐
                    │  Source Config    │
                    │  (JSON)          │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  For each source: │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │                             │
     ┌────────▼────────┐          ┌────────▼────────┐
     │  RSS Adapter    │          │  HTML Adapter   │
     │                 │          │                 │
     │  feedparser     │          │  requests +     │
     │  Parse feed     │          │  BeautifulSoup  │
     │  Extract entries│          │  CSS selectors  │
     └────────┬────────┘          └────────┬────────┘
              │                             │
              └──────────────┬──────────────┘
                             │
                    ┌────────▼─────────┐
                    │  Normalize to    │
                    │  Article objects │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  Filter:         │
                    │  • Last 7 days   │
                    │  • Dedup by URL  │
                    │  • Not in state  │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  Output:         │
                    │  List[Article]   │
                    └──────────────────┘
```

**Article Data Model:**

```python
@dataclass
class Article:
    url: str                # Canonical URL of the article
    url_hash: str           # SHA-256 hash of URL (for state tracking)
    title: str              # Article title
    body: str               # Full article text (or summary if full text unavailable)
    publish_date: str       # ISO date string
    source_name: str        # Name of the source (from config)
    source_type: str        # "rss" or "html"
```

**Adapter Interface:**

Both RSS and HTML adapters implement the same interface:

```python
def fetch_articles(source: SourceConfig) -> list[Article]
```

This means the rest of the pipeline never knows or cares whether an article came from an RSS feed or an HTML page.

**Rate Limiting:**
- 3 seconds between requests to the same domain
- Sources are processed sequentially to respect rate limits
- Individual source failures are caught and logged — the pipeline continues with remaining sources

---

### 3.2 Keyword Matcher

**Responsibility:** Scan all articles against the client registry and identify mentions.

**Input:** `List[Article]` + Client registry (from `config/clients.json`)
**Output:** `List[Match]` — articles matched to specific clients and projects

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  Client Registry │      │  For each article │      │  Match Results   │
│                  │      │  × each client:   │      │                  │
│  • Client names  │─────▶│                   │─────▶│  Grouped by:     │
│  • Aliases       │      │  Search title +   │      │  Client →        │
│  • Project names │      │  body for keywords│      │    Project →     │
│  • Keywords      │      │  (case-insensitive│      │      [Articles]  │
│                  │      │   word-boundary)  │      │                  │
└──────────────────┘      └──────────────────┘      └──────────────────┘
```

**Match Data Model:**

```python
@dataclass
class Match:
    article: Article            # The matched article
    client_name: str            # Which client was matched
    project_name: str | None    # Which project (None = general client mention)
    matched_keywords: list[str] # Which keywords triggered the match
    context_snippets: list[str] # Sentences containing the matched keywords
```

**Matching Logic:**

1. For each article, check title and body against all clients
2. Use `re.compile(r'\b' + re.escape(keyword) + r'\b', re.IGNORECASE)` for word-boundary matching
3. If a keyword matches, extract the surrounding sentence(s) as context
4. Group matches: client → project → list of matched articles
5. Articles matching a client name/alias but no specific project keyword go under "General"

**Why Not LLM-Based Matching?**

| Approach | Pros | Cons |
|----------|------|------|
| **Keyword (chosen)** | Free, fast, deterministic, easy to debug | Misses indirect references |
| **LLM-based** | Catches semantic matches ("the aerospace giant") | Expensive, slower, non-deterministic |

Keyword matching is the right choice for MVP. The architecture supports swapping in LLM-based matching later — the Matcher stage can be replaced without affecting upstream or downstream stages.

---

### 3.3 LLM Stage: Per-Client Summarizer

**Responsibility:** For each client with matches, generate narrative summaries of what was found, organized by project.

**Input:** Matched articles for one client + client context
**Output:** Structured summary with per-project narratives (100–200 words each)
**Model:** Claude Sonnet
**Calls:** One per client with matches (variable per run)

```
For each client with matches:
    ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
    │  Matched Articles│      │   LLM Call       │      │  Client Summary  │
    │  for this client │      │   Claude Sonnet  │      │                  │
    │  + Client context│─────▶│                  │─────▶│  Per-project     │
    │  + Match snippets│      │  Summarize what  │      │  narratives      │
    │                  │      │  was found per   │      │  (100-200 words  │
    │                  │      │  project         │      │   each)          │
    └──────────────────┘      └──────────────────┘      └──────────────────┘
```

**Why One Call Per Client (Not Per Project)?**

- A client typically has 2-5 projects with matches in a given week
- Batching all projects for a client into one call is more efficient
- The LLM gets cross-project context (can note patterns, avoid redundancy)
- Input size per client is small (a few articles' worth of text) — fits easily in context

---

### 3.4 LLM Stage: Highlight Selector

**Responsibility:** Review all client summaries and select the 3–5 most important findings for the highlights section.

**Input:** All per-client summaries + article metadata
**Output:** Ordered list of 3–5 highlights with brief analysis
**Model:** Claude Sonnet
**Calls:** 1 (always exactly one call)

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  All Client      │      │   LLM Call       │      │  Highlights      │
│  Summaries       │      │   Claude Sonnet  │      │                  │
│  + Article       │─────▶│                  │─────▶│  3-5 top picks   │
│    metadata      │      │  Select most     │      │  with 2-3 sentence│
│                  │      │  significant     │      │  analysis each   │
│                  │      │  mentions        │      │                  │
└──────────────────┘      └──────────────────┘      └──────────────────┘
```

---

### 3.5 Email Composer

**Responsibility:** Assemble the final HTML email from highlights, client summaries, and metadata.

**Input:** Highlights + per-client summaries + article metadata + weekly stats
**Output:** Complete HTML email string
**LLM:** None — pure template rendering

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  Template Data   │      │  Jinja2 Engine   │      │  HTML Email      │
│                  │      │                  │      │                  │
│  • Highlights    │─────▶│ Render template  │─────▶│ Ready to send    │
│  • Client sums.  │      │ with data        │      │                  │
│  • Article links │      │                  │      │ Includes:        │
│  • Date range    │      │ Apply styling    │      │ • Inline CSS     │
│  • Stats         │      │                  │      │ • Source links   │
│                  │      │                  │      │ • Stats footer   │
└──────────────────┘      └──────────────────┘      └──────────────────┘
```

**Template Sections:**

| Section | Content | Data Source |
|---------|---------|-------------|
| Header | Digest title, date range, stats summary | Pipeline metadata |
| Highlights (3-5) | Title, client, analysis, source link | Highlight Selector output |
| Client Sections | Client header, per-project summaries, article links | Per-Client Summarizer output |
| Footer | Stats (sources scanned, articles matched, etc.) | Pipeline metadata |

**Email Styling Notes:**
- Use **inline CSS** (many email clients strip `<style>` blocks)
- Keep layout single-column for mobile compatibility
- Use web-safe fonts (Georgia, Arial, system fonts)
- Test with Gmail rendering (primary target)

---

### 3.6 Email Sender

**Responsibility:** Deliver the composed email to all active subscribers.

**Input:** HTML email string + subscriber list
**Output:** Delivery status per recipient

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

### 3.7 State Manager

**Responsibility:** Track processed articles across runs to prevent duplicates.

**Input:** List of articles processed in this run
**Output:** Updated `state/processed.json` committed to repository

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  Load State      │      │  Update State    │      │  Commit State    │
│                  │      │                  │      │                  │
│  Read processed  │─────▶│ Add new article  │─────▶│ git add + commit │
│  .json from repo │      │ URLs/hashes      │      │ + push           │
│                  │      │ Prune > 90 days  │      │                  │
│  If missing:     │      │                  │      │ Via GitHub        │
│  start fresh     │      │                  │      │ Actions bot       │
└──────────────────┘      └──────────────────┘      └──────────────────┘
```

**State File Schema:**

```json
{
  "last_run": "2026-04-01T12:00:00Z",
  "processed_articles": [
    {
      "url_hash": "a1b2c3d4e5f6...",
      "url": "https://example.com/article/12345",
      "title": "Article Title",
      "source": "Source Name",
      "processed_date": "2026-04-01",
      "matched_clients": ["Client A", "Client B"]
    }
  ]
}
```

**State Operations:**

| Operation | When | Purpose |
|-----------|------|---------|
| **Load** | Start of pipeline | Get list of already-processed URLs to filter out |
| **Check** | During scraping | Skip articles whose URL hash exists in state |
| **Update** | After email sent | Add newly processed articles to state |
| **Prune** | During update | Remove entries older than 90 days |
| **Commit** | End of pipeline | Push updated state file to repository |

---

### 3.8 Error Handler

**Responsibility:** Catch failures at any pipeline stage and send an error notification.

```
┌─────────────────────────────────────────────────────────────────┐
│                        ERROR HANDLER                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Wraps entire pipeline in try/except                              │
│                                                                    │
│  On failure:                                                      │
│  ┌──────────────────┐      ┌──────────────────┐                  │
│  │  Capture Error   │      │  Send Error      │                  │
│  │                  │      │  Email           │                  │
│  │  • Stage name    │─────▶│                  │                  │
│  │  • Error message │      │  To: subscribers │                  │
│  │  • Traceback     │      │  Subject: ⚠️     │                  │
│  │  • Timestamp     │      │  Pipeline Error  │                  │
│  │  • Sources       │      │                  │                  │
│  │    attempted     │      │                  │                  │
│  └──────────────────┘      └──────────────────┘                  │
│                                                                    │
│  Partial failure handling:                                        │
│  • Individual source fails → log warning, continue with others   │
│  • Individual client summarization fails → skip client, continue │
│  • Pipeline-level failure → send error email, abort              │
│                                                                    │
└─────────────────────────────────────────────────────────────────┘
```

**Failure Modes & Handling:**

| Failure | Stage | Handling | User Impact |
|---------|-------|----------|-------------|
| Source URL unreachable | Source Scraper | Log warning, skip source, continue with others | Partial coverage noted in footer |
| RSS feed malformed | Source Scraper | Log warning, skip source | Partial coverage |
| HTML selectors return nothing | Source Scraper | Log warning, skip source | Partial coverage |
| 0 articles after scraping | Source Scraper | Send quiet week email → exit cleanly | Gets notice |
| 0 matches after keyword matching | Keyword Matcher | Send quiet week email → exit cleanly | Gets notice |
| LLM API error | Summarizer or Highlights | Retry 3x with backoff → error email | No digest this week |
| LLM returns malformed JSON | Summarizer | Retry with stricter prompt → skip client if still fails | Partial digest |
| Gmail SMTP error | Email Sender | Retry 3x → log error | No digest; visible in GH Actions logs |
| State file corrupted | State Manager | Log warning, treat as fresh run | May re-report some articles |
| Git push fails (state update) | State Manager | Log warning, continue | Next run may re-process some articles |

---

## 4. Configuration Files

### 4.1 File Structure

```
config/
├── sources.json          # URLs and feed configs to scrape
├── clients.json          # Client registry with projects and keywords
├── subscribers.json      # Email recipient list
└── .env                  # API keys, SMTP credentials (not in repo)

state/
└── processed.json        # Tracked article URLs (committed to repo)
```

### 4.2 Sources (`config/sources.json`)

```json
{
  "sources": [
    {
      "name": "Defense News",
      "url": "https://www.defensenews.com/arc/outboundfeeds/rss/",
      "type": "rss",
      "active": true
    },
    {
      "name": "Company Press Releases",
      "url": "https://www.example.com/press-releases",
      "type": "html",
      "selectors": {
        "article_list": "div.press-release",
        "title": "h3.title a",
        "link": "h3.title a@href",
        "date": "span.date",
        "body": "div.content"
      },
      "active": true
    }
  ]
}
```

### 4.3 Clients (`config/clients.json`)

Full structure defined in SCOPE.md. The Keyword Matcher and LLM Summarizer both reference this file.

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

### 5.1 Per-Client Summarizer Prompt

**Purpose:** Summarize all matched articles for a single client, organized by project.

**Temperature:** 0.4 (factual and concise)

**System Prompt:**

```
You are a professional intelligence analyst who writes clear, concise
briefings for account managers. Your job is to summarize what public
sources are saying about a specific client and their projects.

Your writing style is:
- Professional and direct
- Factual — only state what the sources actually say
- Concise — every sentence earns its place
- Organized by project for easy scanning
- Highlights the "so what" — why should the account manager care?

You never speculate beyond what the sources report. If something is
unclear or ambiguous in the source material, say so.
```

**User Prompt:**

```
## Client

Name: {client_name}
Industry: {industry}
Aliases: {aliases}

## Projects

{for each project:}
### {project_name}
Description: {project_description}
Keywords: {keywords}

## Matched Articles

{for each matched article:}
---
Source: {source_name}
Title: {article_title}
Date: {publish_date}
URL: {article_url}
Matched Project: {project_name or "General"}
Matched Keywords: {matched_keywords}
Context Snippets: {context_snippets}

Full Text Excerpt:
{article_body_excerpt (first ~2000 chars)}
---

## Instructions

Write a briefing for this client covering the past week's mentions.

For EACH project that has matches, write a 100-200 word summary that:
1. States what was reported (factual)
2. Notes the source and date
3. Highlights why the account manager should care (implications, action items)

For "General" mentions (client name but no specific project), write a
brief 50-100 word note.

RETURN THIS EXACT JSON STRUCTURE:
{
  "client_name": "<name>",
  "project_summaries": [
    {
      "project_name": "<name>",
      "summary": "<100-200 word narrative summary>",
      "mention_count": <number>,
      "significance": "high" | "medium" | "low"
    }
  ],
  "general_mentions": {
    "summary": "<50-100 word summary or null if no general mentions>",
    "mention_count": <number>
  },
  "total_mentions": <number>
}

Return ONLY valid JSON. No markdown, no commentary outside the JSON.
```

---

### 5.2 Highlight Selector Prompt

**Purpose:** Select the 3–5 most significant findings across all clients for the highlights section.

**Temperature:** 0.3 (consistent, structured selection)

**System Prompt:**

```
You are a senior intelligence analyst who identifies the most significant
developments from a weekly monitoring report. You have excellent judgment
about what matters most to an account manager — contract wins, strategic
shifts, competitive moves, and emerging risks.

You prioritize:
1. Strategic developments (new contracts, partnerships, leadership changes)
2. Competitive signals (lost bids, market entries, pivots)
3. Project milestones (major deliveries, test results, timelines)
4. Emerging risks or opportunities
```

**User Prompt:**

```
## This Week's Client Summaries

{for each client:}
---
Client: {client_name}
Industry: {industry}
Total Mentions: {total_mentions}

{for each project_summary:}
Project: {project_name} ({mention_count} mentions, significance: {significance})
Summary: {summary}

{general_mentions if any}

Source Articles:
{list of article titles + URLs}
---

## Instructions

From ALL the client summaries above, select the 3-5 most significant
developments that the account manager should see first.

For each highlight:
1. Write a clear, specific title (not generic)
2. Identify which client and project it relates to
3. Write 2-3 sentences explaining why this matters and what action
   the account manager might consider
4. Reference the source

RETURN THIS EXACT JSON STRUCTURE:
{
  "highlights": [
    {
      "title": "<specific, descriptive title>",
      "client_name": "<client>",
      "project_name": "<project or General>",
      "analysis": "<2-3 sentence analysis>",
      "source_url": "<url>",
      "source_name": "<source>"
    }
  ]
}

Return ONLY valid JSON. No markdown, no commentary outside the JSON.
```

---

### 5.3 Temperature Guidelines

| Temperature | Rationale | Stage |
|-------------|-----------|-------|
| **0.3** | Consistent, structured selection; minimal creative variance | Highlight Selector |
| **0.4** | Factual and concise; slight flexibility for natural writing | Per-Client Summarizer |

---

### 5.4 Prompt Engineering Principles

| Principle | Application |
|-----------|-------------|
| **Structured output** | Always request JSON with explicit schema; easier to parse and validate |
| **Role-based system prompts** | Each stage has a distinct persona (analyst vs. senior analyst) |
| **Client context** | Always include client/project details so the LLM tailors its output |
| **Explicit format rules** | Word counts, section structures, significance levels |
| **Grounding** | Summaries reference actual source content, not speculation |
| **Audit trail** | Match context snippets show exactly why an article was flagged |

---

## 6. Pipeline Timing Estimate

| Stage | Duration | Notes |
|-------|----------|-------|
| Source Scraper | ~30-120s | Depends on number of sources; 3s rate limit per request |
| Keyword Matcher | ~1-5s | Local string matching, fast |
| Per-Client Summarizer | ~30-90s | Variable: ~10-15s per client with matches |
| Highlight Selector | ~10-15s | Single LLM call, small input |
| Email Composer | ~1s | Template rendering, no LLM |
| Email Sender | ~5-10s | SMTP connection + send |
| State Update | ~10-15s | Git add + commit + push |
| **Total** | **~2-5 minutes** | Well within 15-minute budget |

---

## 7. Directory Structure

```
client-intelligence-digest/
├── .github/
│   └── workflows/
│       └── weekly_digest.yml        # GitHub Actions cron job
│
├── config/
│   ├── sources.json                 # RSS feeds and HTML scraping configs
│   ├── clients.json                 # Client registry with projects/keywords
│   └── subscribers.json             # Email recipient list
│
├── state/
│   └── processed.json               # Tracked processed articles (committed to repo)
│
├── src/
│   ├── __init__.py
│   ├── main.py                      # Pipeline orchestrator (entry point)
│   ├── scraper.py                   # Source Scraper — RSS + HTML adapters
│   ├── matcher.py                   # Keyword Matcher — client/project matching
│   ├── summarizer.py                # LLM: Per-client narrative summaries
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
│   └── quiet_week_email.html        # "No matches this week" template
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
| `main.py` | ~100 | No | Orchestrates the pipeline; calls each stage in sequence |
| `scraper.py` | ~180 | No | RSS + HTML adapters, rate limiting, date filtering |
| `matcher.py` | ~120 | No | Keyword matching against client registry, context extraction |
| `summarizer.py` | ~80 | Yes | Builds per-client prompt, calls Claude, parses response |
| `highlights.py` | ~70 | Yes | Builds highlight prompt, calls Claude, parses response |
| `llm.py` | ~60 | — | Shared Claude client with exponential backoff + jitter |
| `email_composer.py` | ~60 | No | Loads Jinja2 template, renders with data |
| `email_sender.py` | ~60 | No | SMTP connection, sends to subscriber list |
| `state_manager.py` | ~80 | No | Load, check, update, prune, commit state file |
| `error_handler.py` | ~50 | No | Try/except wrapper, error email formatting |
| `logger.py` | ~30 | No | Logging configuration |
| `dev_cache.py` | ~40 | No | Development caching for iterating on prompts |

**Estimated total:** ~930 lines of Python (excluding tests and templates)

---

## 8. GitHub Actions Workflow

```yaml
# .github/workflows/weekly_digest.yml
name: Client Intelligence Digest

on:
  schedule:
    # 12:00 UTC = 7:00 AM ET, every Monday
    - cron: '0 12 * * 1'
  workflow_dispatch:  # Allow manual trigger from GitHub UI

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

**Key Features:**
- `schedule` runs automatically on the cron (every Monday at 7 AM ET)
- `workflow_dispatch` allows manual runs from the GitHub UI (useful for testing)
- `timeout-minutes: 15` kills the job if it hangs
- Secrets are injected as environment variables — never in code
- State commit step uses `git diff --quiet --cached || git commit` to avoid empty commits
- Checkout uses `GITHUB_TOKEN` to enable pushing back to the repo

---

## 9. Future Extension Points

| Extension | Where It Plugs In | Complexity |
|-----------|-------------------|------------|
| **Semantic matching (LLM)** | `matcher.py` — add LLM call after keyword pass | Medium |
| **Additional source types** | `scraper.py` — add new adapter (e.g., API-based) | Low |
| **Per-subscriber client filtering** | `email_composer.py` — filter sections per recipient | Medium |
| **Sentiment analysis** | `summarizer.py` — add sentiment to project summaries | Low |
| **Historical trend tracking** | `state_manager.py` — aggregate mention counts over time | Medium |
| **Slack/Teams delivery** | Add `slack_sender.py` alongside email sender | Medium |
| **Real-time alerts** | Add separate workflow for high-priority keyword matches | High |
| **Dashboard / web UI** | Separate frontend consuming state data | High |
| **Competitor monitoring** | `clients.json` — add competitor section per client | Low |

---

## 10. Summary

| Aspect | Design Choice | Rationale |
|--------|---------------|-----------|
| **Architecture** | Linear pipeline, no framework | Simplest thing that works; easy to debug |
| **State** | JSON file committed to repo | Minimal infrastructure; git-native |
| **Source scraping** | Two adapters (RSS + HTML) behind common interface | Clean separation; easy to add new types |
| **Matching** | Keyword-based, no LLM | Fast, free, deterministic; good enough for known names |
| **LLM** | Claude Sonnet for summarization + highlights | Good balance of quality and cost |
| **LLM Calls** | Variable (1 per client + 1 highlight) | Scales with actual match volume |
| **Email** | Jinja2 template + Gmail SMTP | Simple, free, good enough for small subscriber list |
| **Scheduling** | GitHub Actions cron (weekly) | Free, zero-infrastructure, auto-retry |
| **Error Handling** | Error email + graceful degradation for partial failures | Robust without being complex |
| **Estimated Cost** | ~$0.50–1.50/week | Well under budget |
| **Estimated Runtime** | ~2-5 minutes | Well under 15-minute budget |
