# Scraping Specifications — Client Intelligence Digest Agent

This document defines the exact scraping strategy, selectors, data extraction, and link-following behavior for each configured source. It supplements `ARCHITECTURE.md` with implementation-level detail.

---

## 1. Source Inventory

| Source | Client | Strategy | List Data | Body Content | Dates | Link Domain |
|--------|--------|----------|-----------|-------------|-------|-------------|
| **News Releases** | Draper | JSON-in-HTML | Embedded `<script>` JSON | Follow → detail page | ISO dates in JSON | draper.com (internal) |
| **In the News** | Draper | HTML cards | CSS selectors | Follow → external article | ISO `datetime` attr | External sites |
| **Featured Stories** | Draper | HTML cards | CSS selectors | Follow → detail page | None available | draper.com (internal) |
| **Defense News** | General | RSS feed | feedparser | Included in feed | RSS `pubDate` | N/A (no following) |

---

## 2. Scraping Architecture

Every source goes through a two-phase process: **list extraction** (get article metadata from the index page) and **content extraction** (get full article text by following links).

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        SCRAPING PIPELINE                                │
│                                                                          │
│  ┌─────────────────┐       ┌─────────────────┐       ┌───────────────┐ │
│  │  PHASE 1         │       │  FILTER          │       │  PHASE 2      │ │
│  │  List Extraction │──────▶│  Date + State    │──────▶│  Content      │ │
│  │                  │       │                  │       │  Extraction   │ │
│  │  • JSON-in-HTML  │       │  • By date range │       │               │ │
│  │  • HTML cards    │       │  • By state file │       │  • trafilatura│ │
│  │  • RSS feed      │       │  • Dedup by URL  │       │  (all links)  │ │
│  └─────────────────┘       └─────────────────┘       └───────────────┘ │
│         ▲                                                    │          │
│         │                                                    ▼          │
│    Index page                                        Full Article       │
│    (list of articles)                                objects with body   │
└──────────────────────────────────────────────────────────────────────────┘
```

**Why two phases?**

All Draper sources provide only metadata on their list/index pages (title, URL, date). The actual article text lives on separate pages — either Draper's own detail pages or external publication sites. By filtering before following links, we avoid unnecessary HTTP requests for articles we've already processed or that fall outside the date window.

---

## 3. Content Extraction Library: `trafilatura`

All link-following uses **trafilatura** for article text extraction. This is a purpose-built library for extracting the main content from web pages, stripping navigation, ads, and boilerplate.

| Aspect | Detail |
|--------|--------|
| **Library** | `trafilatura` |
| **What it does** | Downloads a URL, extracts the main article text, strips boilerplate |
| **Why this one** | Handles arbitrary page structures without per-site selectors; excellent accuracy on news/article pages; extracts metadata (title, date, author) as fallback |
| **Used for** | Draper detail pages (News Releases, Featured Stories) AND external publication pages (In the News) |
| **Fallback** | If trafilatura fails to extract meaningful content, log a warning and use the title + any available summary as the article body |

**Why not per-site CSS selectors for detail pages?**

We could write selectors for Draper's own detail pages, but:
1. External sites (In the News) have unpredictable HTML — we need trafilatura anyway
2. Using one extraction strategy for all link-following keeps the code simpler
3. trafilatura handles Draper's own pages well (tested — see Section 4.1)
4. If a client's website redesigns, trafilatura adapts automatically

---

## 4. Per-Source Specifications

### 4.1 Draper — News Releases

**URL:** `https://www.draper.com/media-center/news-releases`

**Strategy:** JSON-in-HTML — article data is embedded in a `<script id="site-news" type="application/json">` tag on the list page, rendered by an Elastic AppSearch frontend.

#### List Extraction

The page contains a `<script>` tag with a JSON array of article objects:

```html
<script id="site-news" type="application/json">
[
  {
    "id": {"raw": "27546"},
    "title": {"raw": "Draper's LEAP Valve Demonstrates..."},
    "published_time": {"raw": "2026-03-10"},
    "url": {"raw": "https://www.draper.com/media-center/news-releases/detail/27546/..."},
    "image_url": {"raw": "https://...thumbnail.jpg"}
  },
  ...
]
</script>
```

**Extraction logic:**
1. Fetch HTML page with `requests`
2. Parse with BeautifulSoup: `soup.find('script', id='site-news')`
3. Parse the tag's text content as JSON
4. Extract from each entry: `title.raw`, `published_time.raw`, `url.raw`

**Fields extracted from list page:**

| Field | JSON Path | Example |
|-------|-----------|---------|
| Title | `entry["title"]["raw"]` | `"Draper's LEAP Valve Demonstrates..."` |
| URL | `entry["url"]["raw"]` | `"https://www.draper.com/media-center/news-releases/detail/27546/..."` |
| Date | `entry["published_time"]["raw"]` | `"2026-03-10"` (ISO format) |
| ID | `entry["id"]["raw"]` | `"27546"` |

#### Content Extraction

Follow each article URL → extract full text with trafilatura.

**Detail page structure (for reference, trafilatura handles this):**
- Title: `<h1>` heading
- Body: Main content div with full press release text
- Date: "Released {Month Day, Year}" at the bottom
- Boilerplate: "About Draper" section at the end (trafilatura strips this)

#### Pagination

The page uses Elastic AppSearch. The `<script id="site-news">` JSON on the **first page** contains 12 articles, the oldest dated **2025-11-17**. All 2026 articles (11 as of Apr 2026) fit on page 1.

| Scenario | Pages Needed | Approach |
|----------|-------------|----------|
| Weekly run (last 7 days) | 1 | First page only — newest articles are always first |
| Backfill (since Jan 1, 2026) | 1 | First page covers Jan–present |
| Deep backfill (older than Nov 2025) | Multiple | Would need Elastic API queries (out of scope) |

**For MVP, page 1 is sufficient.** If a future client's news page has more frequent releases, pagination via `?page=N` or Elastic API can be added.

#### Date Handling

Dates come as ISO strings (`"2026-03-10"`) directly from the JSON — no parsing ambiguity. Filter using `datetime.date.fromisoformat()`.

#### Volume

~3–5 new articles per month. Weekly runs will typically find 0–2 new articles.

---

### 4.2 Draper — In the News

**URL:** `https://www.draper.com/media-center/in-the-news`

**Strategy:** HTML card scraping — articles are rendered as `<article>` cards with structured metadata including dates and external source names.

#### List Extraction

```html
<article class="media card neutral-bg" data-mh="media">
    <div class="media-body">
        <div class="date">
            <time datetime="2026-02-18T00:00:00">Feb 18, 2026</time>
            <span class="news-source"> - Semiconductor Digest</span>
        </div>
        <span class="media-heading">
            <a href="https://www.semiconductor-digest.com/..." target="_blank" rel="noopener">
                Article Title
            </a>
        </span>
    </div>
</article>
```

**CSS Selectors:**

| Field | Selector | Attribute | Example |
|-------|----------|-----------|---------|
| Article container | `article.media.card` | — | Each card is one article |
| Title | `.media-heading a` | text content | `"DOD picks 10 for $25B microelectronics contract"` |
| URL | `.media-heading a` | `href` | External URL (absolute) |
| Date | `time[datetime]` | `datetime` attr | `"2026-01-05T00:00:00"` |
| Source name | `.news-source` | text content | `" - Washington Technology"` (strip leading " - ") |

**Important:** URLs on this page point to **external sites** (Semiconductor Digest, Washington Technology, Space News, etc.), not to Draper's own pages.

#### Content Extraction

Follow each external URL → extract full article text with trafilatura.

**External site considerations:**
- Every site has different HTML structure — trafilatura handles this
- Some sites may be paywalled or require cookies — trafilatura handles soft paywalls, hard paywalls will fail gracefully
- If extraction fails, use the article title as the body (with a warning logged)

#### Pagination

24 pages total, URL format: `?page=N` (1-indexed). Each page shows 12 articles.

| Scenario | Pages Needed | Approach |
|----------|-------------|----------|
| Weekly run | 1 | Page 1 — newest articles are first |
| Backfill (since Jan 1, 2026) | 1 | Only 3 articles since Jan 2026, all on page 1 |

#### Date Handling

ISO datetime in the `datetime` attribute: `"2026-02-18T00:00:00"`. Parse with `datetime.datetime.fromisoformat()`.

#### Also Available: RSS Feed

`https://www.draper.com/media-center/in-the-news/rss` — contains the same 12 entries with `pubDate` and external links. The RSS description fields are **empty** (no content). The HTML path is preferred because it additionally provides the **source publication name** (e.g., "Semiconductor Digest"), which is valuable metadata for the newsletter.

#### Volume

~1–3 new articles per month. Low volume.

---

### 4.3 Draper — Featured Stories

**URL:** `https://www.draper.com/media-center/featured-stories`

**Strategy:** HTML card scraping — articles are rendered as `<article>` cards with images and titles but **no dates**.

#### List Extraction

```html
<article class="media card neutral-bg" data-mh="media">
    <img src="https://...image.png" alt="..." class="media-object">
    <div class="media-body">
        <span class="media-heading">
            <a href="/media-center/featured-stories/detail/27477/drapers-role-in-the-past-and-future-of-vleo">
                Draper's Role in the Past and Future of VLEO
            </a>
            <p>Optional subtitle text</p>
        </span>
    </div>
</article>
```

**CSS Selectors:**

| Field | Selector | Attribute | Example |
|-------|----------|-----------|---------|
| Article container | `article.media.card` | — | Each card is one article |
| Title | `.media-heading a` | text content | `"Draper's Role in the Past and Future of VLEO"` |
| URL | `.media-heading a` | `href` | Relative path (needs base URL) |
| Subtitle | `.media-heading p` | text content | `"How Draper Ensures Reliable Antenna Performance"` (optional) |
| Date | **Not available** | — | — |

**Important:** URLs are **relative** (e.g., `/media-center/featured-stories/detail/27477/...`). Must prepend `https://www.draper.com`.

#### Content Extraction

Follow each detail page URL → extract full article text with trafilatura.

**Detail pages are Draper-hosted long-form articles** (e.g., the VLEO article is ~1,000 words of thought leadership content). trafilatura extracts these well.

#### The Date Problem

This source has **no reliable dates**:

| Source | Date Available? | Issue |
|--------|----------------|-------|
| List page HTML | No | No date elements in the cards |
| RSS feed (`/featured-stories/rss`) | Broken | All entries show the feed generation timestamp, not actual publish dates |
| Detail page HTML | No visible date | Pages don't display a publication date |
| trafilatura metadata | Maybe | trafilatura may extract a date from meta tags if present |

**Workaround:**

1. **Try trafilatura metadata** — it checks `<meta>` tags and schema.org markup for dates
2. **If no date found,** use the article's position in the list as a rough ordering signal (newest first)
3. **For date filtering,** Featured Stories without dates are **always included** in the scrape — state deduplication prevents re-processing
4. **For the newsletter,** display "Date not available" or omit the date field

This is acceptable because Featured Stories are **infrequent** (12 total over several years) and are thought leadership pieces rather than time-sensitive news.

#### Pagination

3 pages total, URL format: `?page=N`. Each page shows 12 articles.

| Scenario | Pages Needed | Approach |
|----------|-------------|----------|
| Weekly run | 1 | Page 1 — check for any new entries (infrequent) |
| Backfill | 1–3 | Process all pages; state dedup handles the rest |

#### Volume

Very low — ~2–4 new articles per year. Most runs will find 0 new articles.

---

### 4.4 Defense News (General Source)

**URL:** `https://www.defensenews.com/arc/outboundfeeds/rss/?outputType=xml`

> **Note:** The previously documented URL (`https://www.defensenews.com/rss/`) returns 404. The correct feed URL uses the Arc Publishing outbound feeds path.

**Strategy:** RSS feed with full article content — no link-following needed.

#### List + Content Extraction (Single Phase)

Defense News includes **full article text** in the RSS feed's CDATA section, making this the simplest source to scrape.

```xml
<item>
    <title><![CDATA[ US special forces rescue second F-15 airman from Iran ]]></title>
    <link>https://www.defensenews.com/news/your-military/2026/04/05/...</link>
    <dc:creator><![CDATA[ Phil Stewart and Menna Alaa El-Din, Reuters ]]></dc:creator>
    <description><![CDATA[ The airman was wounded but "will be just fine"... ]]></description>
    <pubDate>Sun, 05 Apr 2026 13:40:03 +0000</pubDate>
    <content:encoded><![CDATA[
        (Full article text, multiple paragraphs, often 500-2000 words)
    ]]></content:encoded>
</item>
```

**Fields from feedparser:**

| Field | feedparser Path | Notes |
|-------|----------------|-------|
| Title | `entry.title` | Plain text (CDATA unwrapped) |
| URL | `entry.link` | Absolute URL |
| Author | `entry.get('dc_creator', '')` or `entry.get('author', '')` | May need fallback |
| Summary | `entry.summary` | Short description |
| Full text | `entry.get('content', [{}])[0].get('value', '')` | Full article in `content:encoded` |
| Date | `entry.published_parsed` | `time.struct_time`, convert to `datetime` |

**Body text priority:**
1. Use `content:encoded` (full article text) if available
2. Fall back to `description` (summary) if not
3. feedparser normalizes both automatically

#### Pagination

None — RSS feeds are a single document. The feed contains the most recent ~50 articles.

#### Date Handling

Standard RSS `pubDate` format: `"Sun, 05 Apr 2026 13:40:03 +0000"`. feedparser parses this into `time.struct_time` automatically.

#### Category Feeds

Defense News offers category-specific feeds for more targeted scraping:

| Category | Feed URL |
|----------|----------|
| Home (all) | `https://www.defensenews.com/arc/outboundfeeds/rss/?outputType=xml` |
| Air | `.../rss/category/air/?outputType=xml` |
| Land | `.../rss/category/land/?outputType=xml` |
| Naval | `.../rss/category/naval/?outputType=xml` |
| Pentagon | `.../rss/category/pentagon/?outputType=xml` |
| Congress | `.../rss/category/congress/?outputType=xml` |
| Space | `.../rss/category/space/?outputType=xml` |
| Industry | `.../rss/category/industry/?outputType=xml` |

For MVP, use the **Home (all)** feed. Category feeds can be added later if the all-articles volume is too high for keyword matching.

#### Volume

~10–20 new articles per day. Weekly run will process ~70–140 articles, most of which will be discarded by the keyword matcher (only articles mentioning tracked clients are kept).

---

## 5. Backfill vs. Weekly Mode

The system supports two run modes controlled by the `LOOKBACK_DAYS` environment variable:

| Mode | `LOOKBACK_DAYS` | Date Cutoff | When Used |
|------|-----------------|-------------|-----------|
| **Weekly** | `7` (default) | Last 7 days | Regular weekly cron runs |
| **Backfill** | `95` | Since Jan 1, 2026 (~95 days from Apr 5) | One-time initial run |

### Backfill Behavior by Source

| Source | Backfill Scope | Pages Needed | Expected Articles |
|--------|---------------|-------------|-------------------|
| News Releases | Jan 1, 2026 – present | 1 (JSON covers all 2026) | ~11 articles |
| In the News | Jan 1, 2026 – present | 1 (only 3 since Jan 2026) | 3 articles |
| Featured Stories | All (no dates) | 1–3 (process all, state dedup) | 12 articles (all time) |
| Defense News | Jan 1, 2026 – present | 1 (RSS has ~50 recent) | ~50 articles from RSS; older not available |

### Backfill Limitation

The Defense News RSS feed only contains the most recent ~50 articles (~3-5 days of content). For a backfill to January, we **cannot** get older Defense News articles from the RSS feed alone. Options:

1. **Accept the gap** — Run the backfill now and capture whatever is in the current RSS feed. Older Defense News mentions are lost. This is the pragmatic approach.
2. **Category feeds** — Check if category-specific feeds have longer retention (unlikely but worth testing).
3. **HTML scraping** — Scrape Defense News article pages directly (complex, not worth it for MVP).

**Recommendation:** Accept the gap. The Draper-specific sources cover the important content. Defense News general mentions are supplementary.

### Running a Backfill

```bash
LOOKBACK_DAYS=95 DRY_RUN=true python src/main.py
```

After verifying output, run without `DRY_RUN` to send the newsletter and update state. Subsequent weekly runs use the default `LOOKBACK_DAYS=7`.

---

## 6. Rate Limiting & Politeness

| Rule | Value | Rationale |
|------|-------|-----------|
| Delay between requests to same domain | 3 seconds | Respectful; avoids rate limiting |
| Delay between different domains | 1 second | Lighter touch for unrelated sites |
| User-Agent header | `ClientIntelligenceDigest/1.0 (weekly newsletter bot)` | Identifies the bot; some sites block default python-requests UA |
| Timeout per request | 30 seconds | Prevents hanging on unresponsive sites |
| Max retries per URL | 2 | With exponential backoff (3s, 9s) |

### Request Order

```
1. Draper News Releases (list page)
   ├─ [3s delay] Detail page 1
   ├─ [3s delay] Detail page 2
   └─ ... (same domain)

2. [1s delay]

3. Draper In the News (list page)
   ├─ [1s delay] External site A
   ├─ [1s delay] External site B
   └─ ... (different domains, 1s between)

4. [1s delay]

5. Draper Featured Stories (list page)
   ├─ [3s delay] Detail page 1
   ├─ [3s delay] Detail page 2
   └─ ... (same domain)

6. [1s delay]

7. Defense News (RSS — single request, no following)
```

---

## 7. Error Handling per Source

| Error | Source | Handling | Impact |
|-------|--------|----------|--------|
| List page returns non-200 | Any Draper source | Log warning, skip source | Partial coverage |
| JSON parse fails for `<script>` tag | News Releases | Log error, skip source | No press releases this week |
| RSS feed unreachable | Defense News | Log warning, skip source | No general source matches |
| External link returns 403/404 | In the News | Log warning, use title as body | Article included with limited content |
| trafilatura extraction returns empty | Any followed link | Log warning, use title + available metadata | Article included with limited content |
| Timeout on external site | In the News | Skip that article | One fewer article |
| Page structure changed | Any HTML source | trafilatura still works (generic); list extraction may fail | Depends on what changed |

---

## 8. Updated Article Data Model

Based on the scraping investigation, the `Article` dataclass gains a few fields:

```python
@dataclass
class Article:
    url: str                    # Canonical URL of the article
    url_hash: str               # SHA-256 hash of URL (for state tracking)
    title: str                  # Article title
    body: str                   # Full text extracted via trafilatura or RSS content
    summary: str                # Short summary (RSS description or first ~200 chars of body)
    publish_date: str | None    # ISO date string, or None if unavailable (Featured Stories)
    source_name: str            # e.g., "Draper — News Releases" or "Defense News"
    source_type: str            # "json_script", "html_cards", or "rss"
    source_path: str            # "client" or "general"
    external_source: str | None # Publication name for In the News (e.g., "Semiconductor Digest")
    client: str | None          # Pre-tagged client (for client sources) or None
    matched_projects: list[str] # Filled by matcher (general) or summarizer (client)
    matched_keywords: list[str] # Filled by matcher (general sources only)
    context_snippets: list[str] # Sentences containing matched keywords
```

**New/changed fields:**
- `summary` — Short summary for email previews (separate from full `body`)
- `publish_date` — Now `str | None` to handle Featured Stories which have no dates
- `source_type` — Updated enum values: `"json_script"`, `"html_cards"`, `"rss"`
- `external_source` — Publication name for articles sourced from "In the News" (extracted from the `.news-source` span)

---

## 9. Updated Configuration Format

### Client Sources (`config/clients.json`)

Based on the actual page structures discovered:

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

### General Sources (`config/general_sources.json`)

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

> **Corrected URL:** The previous `https://www.defensenews.com/rss/` returns 404. The correct URL is `https://www.defensenews.com/arc/outboundfeeds/rss/?outputType=xml`.

---

## 10. Implementation Notes

### trafilatura Usage Pattern

```python
import trafilatura

def extract_article_content(url: str) -> tuple[str, str | None]:
    """Follow a URL and extract the main article text.
    
    Returns (body_text, extracted_date_or_none).
    """
    downloaded = trafilatura.fetch_url(url)
    if not downloaded:
        return ("", None)
    
    result = trafilatura.extract(
        downloaded,
        include_comments=False,
        include_tables=False,
        favor_precision=True,
        output_format="txt",
    )
    
    # Also try to get metadata (date, author, etc.)
    metadata = trafilatura.extract_metadata(downloaded)
    extracted_date = metadata.date if metadata else None
    
    return (result or "", extracted_date)
```

### JSON-in-HTML Extraction Pattern

```python
from bs4 import BeautifulSoup
import json

def extract_news_releases(html: str) -> list[dict]:
    """Extract articles from the embedded JSON script tag."""
    soup = BeautifulSoup(html, "html.parser")
    script_tag = soup.find("script", id="site-news")
    if not script_tag:
        return []
    
    data = json.loads(script_tag.string)
    return [
        {
            "title": entry["title"]["raw"],
            "url": entry["url"]["raw"],
            "date": entry["published_time"]["raw"],
        }
        for entry in data
    ]
```

### HTML Card Extraction Pattern

```python
def extract_html_cards(html: str, selectors: dict, base_url: str = "") -> list[dict]:
    """Extract articles from HTML card elements."""
    soup = BeautifulSoup(html, "html.parser")
    articles = []
    
    for card in soup.select(selectors["container"]):
        title_el = card.select_one(selectors["title"])
        link_el = card.select_one(selectors["link"])
        
        if not title_el or not link_el:
            continue
        
        href = link_el.get(selectors.get("link_attribute", "href"), "")
        if href.startswith("/"):
            href = base_url + href
        
        article = {
            "title": title_el.get_text(strip=True),
            "url": href,
        }
        
        # Optional fields
        if "date" in selectors:
            date_el = card.select_one(selectors["date"])
            if date_el:
                article["date"] = date_el.get(selectors.get("date_attribute", ""), "")
        
        if "source_name" in selectors:
            source_el = card.select_one(selectors["source_name"])
            if source_el:
                article["external_source"] = source_el.get_text(strip=True).lstrip("- ")
        
        if "subtitle" in selectors:
            sub_el = card.select_one(selectors["subtitle"])
            if sub_el:
                article["subtitle"] = sub_el.get_text(strip=True)
        
        articles.append(article)
    
    return articles
```

---

## 11. Testing Strategy

### Source-Level Smoke Tests

Each source should have a live smoke test (run manually, not in CI) that verifies:

1. The page is reachable
2. The expected structure is present (JSON tag, HTML cards, RSS entries)
3. Articles can be extracted with correct fields
4. Link-following returns non-empty content

### Mock Tests (CI-Safe)

Unit tests use saved HTML/RSS snapshots:

| Test | What It Verifies |
|------|-----------------|
| `test_json_script_extraction` | Parses a saved News Releases HTML page → extracts correct articles |
| `test_html_card_extraction` | Parses saved In the News HTML → correct titles, URLs, dates, sources |
| `test_html_card_no_dates` | Parses saved Featured Stories HTML → handles missing dates |
| `test_rss_extraction` | Parses saved Defense News XML → correct fields, full article text |
| `test_trafilatura_extraction` | Extracts content from a saved detail page HTML |
| `test_relative_url_resolution` | Featured Stories relative URLs become absolute |
| `test_date_filtering` | Articles outside LOOKBACK_DAYS window are excluded |
| `test_no_date_always_included` | Articles without dates pass the date filter |

---

## 12. Adding New Sources

When a new client or source is added, follow this checklist:

1. **Identify the page type:** Does it embed JSON? Use HTML cards? Have an RSS feed?
2. **Check for RSS first:** RSS is simpler and more stable. If available, prefer it.
3. **Inspect the HTML:** Use browser DevTools to identify selectors for article containers, titles, links, and dates.
4. **Test link-following:** Verify trafilatura can extract content from the target detail/article pages.
5. **Check date availability:** Are dates on the list page, in the detail page, or neither?
6. **Add to config:** Create a new entry in `clients.json` or `general_sources.json` with the appropriate `type` and `config`.
7. **Test with DRY_RUN:** Run the pipeline in dry-run mode to verify extraction before going live.

---

## 13. Summary of Architectural Changes

This investigation revealed several important changes from the original architecture:

| Original Assumption | Actual Finding | Impact |
|---------------------|---------------|--------|
| Client sources use CSS selectors | News Releases use embedded JSON; In the News use HTML cards | Three extraction strategies, not one |
| Source types are "rss" or "html" | Source types are "json_script", "html_cards", or "rss" | Config format updated |
| Body text available from list pages | All Draper sources need link-following for content | New Phase 2 (content extraction) in scraping |
| trafilatura not in original plan | Essential for external link following (In the News) | New dependency |
| Defense News URL was `.../rss/` | Correct URL is `.../arc/outboundfeeds/rss/?outputType=xml` | Config corrected |
| All sources have dates | Featured Stories have no reliable dates | `publish_date` is now nullable |
| Backfill is simple date filtering | RSS feeds have limited retention; backfill gaps for general sources | Documented limitation |
| HTML scraping is uniform | Each page has unique structure (JSON, cards with/without dates) | Per-source extraction logic |
