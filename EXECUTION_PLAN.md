# Execution Plan — Client Intelligence Digest Agent

This plan turns the scope and architecture into a step-by-step build sequence. Every step includes a **verification checkpoint** — we test constantly as we build, not after.

---

## Philosophy: Build → Verify → Advance

```
┌─────────┐     ┌──────────┐     ┌─────────┐     ┌──────────┐
│  BUILD   │────▶│  VERIFY   │────▶│  WORKS?  │────▶│  NEXT    │
│  a piece │     │  it works │     │          │     │  piece   │
└─────────┘     └──────────┘     │  YES ──────────▶│          │
                                  │  NO  ──┐       └──────────┘
                                  └────────┘
                                       │
                                       ▼
                                  ┌──────────┐
                                  │   FIX    │
                                  │  before  │
                                  │  moving  │
                                  │  on      │
                                  └──────────┘
```

Nothing advances until the current piece verifiably works.

---

## Phase 0 — Project Setup

### 0.1 Repository & Structure

- [ ] Initialize git repository
- [ ] Initialize the project directory structure:

```
client-intelligence-digest/
├── .github/workflows/
├── config/
├── state/
├── src/
├── templates/
├── tests/
├── requirements.txt
├── .env.example
├── .gitignore
└── README.md
```

- [ ] Create `requirements.txt` with all dependencies (from TECH_STACK.md)
- [ ] Create `.env.example` with placeholder values
- [ ] Create `.gitignore`
- [ ] Install dependencies in a virtual environment

**✅ Checkpoint 0.1:**
```bash
git status
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python -c "import feedparser; import requests; import bs4; import anthropic; import jinja2; print('All imports OK')"
```
Expected: All imports succeed, no version conflicts.

---

### 0.2 Configuration Files

- [ ] Create `config/clients.json` with 1 example client (Draper) including client_sources and projects
- [ ] Create `config/general_sources.json` with 1 example source (Defense News RSS)
- [ ] Create `config/subscribers.json` with a placeholder email
- [ ] Create `state/processed.json` with an empty initial structure

**✅ Checkpoint 0.2:**
```bash
python -c "
import json
clients = json.load(open('config/clients.json'))
general = json.load(open('config/general_sources.json'))
subs = json.load(open('config/subscribers.json'))
state = json.load(open('state/processed.json'))
print(f'Clients: {len(clients[\"clients\"])}')
print(f'Client sources for {clients[\"clients\"][0][\"client_name\"]}: {len(clients[\"clients\"][0][\"client_sources\"])}')
print(f'General sources: {len(general[\"general_sources\"])}')
print(f'Subscribers: {len(subs[\"subscribers\"])}')
print('Config OK')
"
```
Expected: All config files load and parse correctly.

---

### 0.3 Logging Setup

- [ ] Configure Python `logging` module in a shared utility
- [ ] Support `LOG_LEVEL` environment variable

**✅ Checkpoint 0.3:**
```bash
python -c "
from src.logger import setup_logger
log = setup_logger('test')
log.info('Info message')
log.warning('Warning message')
log.debug('Debug message (should not show at INFO level)')
"
```

---

### 0.4 Development Cache Utility

- [ ] Create `src/dev_cache.py` with `save_cache(key, data)` and `load_cache(key)` functions
- [ ] Cache location: `.dev_cache/` directory (git-ignored)
- [ ] Date-stamped auto-invalidation

**How it's used during development:**

| When Iterating On... | Load From Cache | Run Live |
|---|---|---|
| Keyword matcher (Phase 2) | Articles from scraper | Matcher only |
| Summarizer prompt (Phase 4) | Articles + matches | Summarizer only |
| Highlight prompt (Phase 4) | All intermediate data | Highlight selector only |
| Email template (Phase 5) | All intermediate data | Email composer only |

**✅ Checkpoint 0.4:**
```bash
python -c "
from src.dev_cache import save_cache, load_cache
save_cache('test', {'hello': 'world', 'count': 42})
result = load_cache('test')
assert result['hello'] == 'world'
print('Dev cache OK')
"
```

### 0.5 First Commit

- [ ] Commit: "Initial project setup: structure, config, logging, dev cache"

---

## Phase 1 — Source Scraper (Dual-Path, Two-Phase, No LLM Needed)

This is the foundation. The scraper uses a two-phase approach for each source: **list extraction** (get article metadata) then **content extraction** (follow links for full text). See `SCRAPING_SPECS.md` for detailed per-source selectors and behavior.

### 1.1 Article Data Model

- [ ] Define `Article` dataclass in `scraper.py`
- [ ] Include nullable `publish_date` (Featured Stories have no dates)
- [ ] Include `external_source` field (for In the News source names)
- [ ] Include `summary` field (separate from full `body`)

**✅ Checkpoint 1.1:**
```bash
python -c "
from src.scraper import Article
a = Article(url='https://example.com', url_hash='abc', title='Test', body='Body text',
            summary='Short summary', publish_date=None, source_name='Test Source',
            source_type='html_cards', source_path='client', external_source=None,
            client='Draper', matched_projects=[], matched_keywords=[], context_snippets=[])
print(f'{a.title} — client={a.client}, date={a.publish_date}')
print('Article dataclass OK')
"
```

---

### 1.2 JSON-in-HTML List Extractor (News Releases)

- [ ] Implement `extract_json_script_articles()` — parses the `<script id="site-news">` JSON tag
- [ ] Extract title, URL, and ISO date from each entry
- [ ] See `SCRAPING_SPECS.md` §4.1 for JSON structure

**✅ Checkpoint 1.2 — Test against live News Releases page:**
```bash
python -c "
from src.scraper import extract_json_script_articles
import requests

html = requests.get('https://www.draper.com/media-center/news-releases').text
articles = extract_json_script_articles(html, script_id='site-news',
    fields={'title': 'title.raw', 'url': 'url.raw', 'date': 'published_time.raw'})

print(f'Extracted {len(articles)} articles from JSON')
for a in articles[:3]:
    print(f'  {a[\"date\"]} — {a[\"title\"][:60]}')
    print(f'    {a[\"url\"][:80]}')
"
```
Expected: 12 articles with ISO dates and absolute URLs.

---

### 1.3 HTML Card List Extractor (In the News, Featured Stories)

- [ ] Implement `extract_html_card_articles()` — parses `<article>` card elements
- [ ] Support configurable CSS selectors from source config
- [ ] Handle optional fields (date, source_name, subtitle)
- [ ] Convert relative URLs to absolute using `base_url`
- [ ] See `SCRAPING_SPECS.md` §4.2 and §4.3 for selectors

**✅ Checkpoint 1.3a — Test against live In the News page:**
```bash
python -c "
from src.scraper import extract_html_card_articles
import requests

html = requests.get('https://www.draper.com/media-center/in-the-news').text
selectors = {
    'container': 'article.media.card',
    'title': '.media-heading a',
    'link': '.media-heading a',
    'link_attribute': 'href',
    'date': 'time',
    'date_attribute': 'datetime',
    'source_name': '.news-source'
}
articles = extract_html_card_articles(html, selectors, base_url='https://www.draper.com')

print(f'Extracted {len(articles)} articles')
for a in articles[:3]:
    print(f'  {a.get(\"date\", \"no date\")} — {a[\"title\"][:50]}')
    print(f'    Source: {a.get(\"external_source\", \"N/A\")}')
    print(f'    URL: {a[\"url\"][:60]}')
"
```
Expected: 12 articles with dates, source names, and external URLs.

**✅ Checkpoint 1.3b — Test against live Featured Stories page (no dates):**
```bash
python -c "
from src.scraper import extract_html_card_articles
import requests

html = requests.get('https://www.draper.com/media-center/featured-stories').text
selectors = {
    'container': 'article.media.card',
    'title': '.media-heading a',
    'link': '.media-heading a',
    'link_attribute': 'href',
    'subtitle': '.media-heading p'
}
articles = extract_html_card_articles(html, selectors, base_url='https://www.draper.com')

print(f'Extracted {len(articles)} articles')
for a in articles[:3]:
    print(f'  {a[\"title\"][:50]}')
    print(f'    Date: {a.get(\"date\", \"NONE (expected)\")}')
    print(f'    URL: {a[\"url\"][:60]}')
"
```
Expected: 12 articles with titles and absolute URLs, no dates.

---

### 1.4 RSS Adapter (Defense News)

- [ ] Implement RSS parsing in `scraper.py` using `feedparser`
- [ ] Extract full article text from `content:encoded` (Defense News includes it)
- [ ] Fall back to `description` if full content not available
- [ ] Filter entries to configured lookback window (default 7 days)

**✅ Checkpoint 1.4 — Test RSS scraping against Defense News:**
```bash
python -c "
from src.scraper import fetch_rss_articles

articles = fetch_rss_articles({
    'name': 'Defense News',
    'url': 'https://www.defensenews.com/arc/outboundfeeds/rss/?outputType=xml',
    'type': 'rss'
})
print(f'Fetched {len(articles)} articles from RSS')
if articles:
    print(f'First: {articles[0].title}')
    print(f'Date: {articles[0].publish_date}')
    print(f'Body length: {len(articles[0].body)} chars')
    print(f'Has full text: {len(articles[0].body) > 500}')
"
```
Expected: Articles with full body text (500+ chars), not just summaries.

---

### 1.5 Content Extraction via Link-Following (trafilatura)

- [ ] Implement `extract_article_content(url)` using trafilatura
- [ ] Return body text + any extracted metadata (date, author)
- [ ] Handle failures gracefully (return empty body with warning)
- [ ] Apply rate limiting (3s same domain, 1s different domains)

**✅ Checkpoint 1.5a — Test on a Draper detail page:**
```bash
python -c "
from src.scraper import extract_article_content

body, date = extract_article_content(
    'https://www.draper.com/media-center/news-releases/detail/27523/draper-and-the-northeast-microelectronics-coalition-nemc-announce-chip-design-and-packaging-services-to-accelerate-technology-commercialization'
)
print(f'Body length: {len(body)} chars')
print(f'Extracted date: {date}')
print(f'First 200 chars: {body[:200]}')
"
```

**✅ Checkpoint 1.5b — Test on an external article (In the News link):**
```bash
python -c "
from src.scraper import extract_article_content

body, date = extract_article_content(
    'https://www.semiconductor-digest.com/draper-and-the-northeast-microelectronics-coalition-announce-chip-design-and-packaging-services/'
)
print(f'Body length: {len(body)} chars')
print(f'Extracted date: {date}')
print(f'First 200 chars: {body[:200]}')
"
```
Expected: Non-empty body text from both Draper-hosted and external pages.

---

### 1.6 Client-Specific Source Scraping (Combined)

- [ ] Implement `scrape_client_sources()` — iterates over each client's sources
- [ ] Calls the correct list extractor based on `type` (json_script or html_cards)
- [ ] Follows links (Phase 2) for sources where `follow_links: true`
- [ ] Auto-tags every article with `client = client_name` and `source_path = "client"`
- [ ] Applies date filtering (skip articles outside LOOKBACK_DAYS window)
- [ ] Handle per-source failures (log, skip, continue)

**✅ Checkpoint 1.6 — Full end-to-end for Draper:**
```bash
python -c "
from src.scraper import scrape_client_sources
import json

clients = json.load(open('config/clients.json'))
draper = clients['clients'][0]

articles, failures = scrape_client_sources(draper, lookback_days=95)

print(f'Client: {draper[\"client_name\"]}')
print(f'Total articles: {len(articles)}')
print(f'Failed sources: {len(failures)}')
for a in articles[:5]:
    print(f'  [{a.source_name}] {a.title[:50]}...')
    print(f'    date={a.publish_date}, body_len={len(a.body)}')
"
```
Expected: Articles from all 3 Draper sources, pre-tagged, with body text filled.

---

### 1.7 General Source Scraping

- [ ] Implement `scrape_general_sources()` — iterates over general sources
- [ ] Tag articles with `source_path = "general"` and `client = None`
- [ ] RSS sources with full content skip link-following

**✅ Checkpoint 1.7 — Scrape general sources:**
```bash
python -c "
from src.scraper import scrape_general_sources
import json

general = json.load(open('config/general_sources.json'))
articles, failures = scrape_general_sources(general['general_sources'])

print(f'General articles: {len(articles)}')
print(f'Failed sources: {len(failures)}')
for a in articles[:3]:
    print(f'  [{a.source_name}] {a.title[:50]}...')
    print(f'    body_len={len(a.body)}, client={a.client}')
"
```
Expected: Articles with full body text, all with `client=None`.

---

### 1.8 State Filtering & Deduplication

- [ ] Filter out articles whose URL hash exists in `state/processed.json`
- [ ] Deduplicate by URL across all sources
- [ ] Log counts: total scraped, already processed, new

**✅ Checkpoint 1.8:**
```bash
python -c "
from src.scraper import scrape_client_sources, scrape_general_sources
from src.state_manager import load_state, is_processed
import json

clients = json.load(open('config/clients.json'))
general = json.load(open('config/general_sources.json'))
state = load_state()

client_articles, _ = scrape_client_sources(clients['clients'][0])
general_articles, _ = scrape_general_sources(general['general_sources'])

all_articles = client_articles + general_articles
new_articles = [a for a in all_articles if not is_processed(state, a.url_hash)]

print(f'Total scraped: {len(all_articles)}')
print(f'Already processed: {len(all_articles) - len(new_articles)}')
print(f'New articles: {len(new_articles)}')
"
```

---

### 1.9 Cache Scraped Articles (Dev Support)

- [ ] Save scraped articles to dev cache for downstream development

### 1.10 Unit Tests for Scraper

- [ ] Write `tests/test_scraper.py`
- [ ] Test Article dataclass construction (including nullable date)
- [ ] Test JSON-in-HTML extraction (mock News Releases HTML)
- [ ] Test HTML card extraction (mock In the News HTML with dates)
- [ ] Test HTML card extraction (mock Featured Stories HTML without dates)
- [ ] Test RSS parsing (mock Defense News feed with full content)
- [ ] Test trafilatura content extraction (mock detail page)
- [ ] Test relative URL resolution
- [ ] Test date filtering with nullable dates
- [ ] Test client auto-tagging
- [ ] Test error handling (mock network failures)

**✅ Checkpoint 1.10:**
```bash
pytest tests/test_scraper.py -v
```

### 1.11 Commit

- [ ] Commit: "Add two-phase source scraper: list extraction (JSON/HTML/RSS) + content following (trafilatura)"

---

## Phase 2 — Keyword Matcher (General Sources Only, No LLM)

### 2.1 Keyword Matching

- [ ] Implement `matcher.py` with keyword matching against client registry
- [ ] Only operates on articles where `source_path == "general"`
- [ ] Word-boundary matching, case-insensitive
- [ ] Populate `client`, `matched_projects`, `matched_keywords` on matched articles
- [ ] Discard unmatched general-source articles

**✅ Checkpoint 2.1:**
```bash
python -c "
from src.scraper import Article
from src.matcher import match_general_articles
from src.dev_cache import load_cache
import json

# Load cached general-source articles
articles = [Article(**a) for a in load_cache('general_articles')]
clients = json.load(open('config/clients.json'))

matched, unmatched = match_general_articles(articles, clients['clients'])

print(f'General articles scanned: {len(articles)}')
print(f'Matched: {len(matched)}')
print(f'Unmatched (discarded): {len(unmatched)}')
for m in matched[:5]:
    print(f'  [{m.client}] {m.title[:50]}...')
    print(f'    Keywords: {m.matched_keywords}')
"
```

---

### 2.2 Context Extraction

- [ ] Extract sentence(s) containing matched keywords
- [ ] Store as `context_snippets` on the Article

**✅ Checkpoint 2.2:**
```bash
python -c "
from src.matcher import match_general_articles
from src.dev_cache import load_cache
import json

# Use cached articles
# Verify context snippets are populated
print('Context extraction verified')
"
```

---

### 2.3 Merge Both Paths

- [ ] Implement merge logic: combine client-source articles + matched general articles
- [ ] Deduplicate by URL (same article from both paths)
- [ ] Group by client
- [ ] Cache merged results for downstream development

**✅ Checkpoint 2.3:**
```bash
python -c "
from src.matcher import merge_articles
from src.dev_cache import load_cache, save_cache

client_articles = load_cache('client_articles')
matched_general = load_cache('matched_general')

merged = merge_articles(client_articles, matched_general)

save_cache('merged', merged)

for client_name, articles in merged.items():
    client_count = sum(1 for a in articles if a['source_path'] == 'client')
    general_count = sum(1 for a in articles if a['source_path'] == 'general')
    print(f'{client_name}: {len(articles)} total ({client_count} client, {general_count} general)')
"
```

### 2.4 Unit Tests

- [ ] Write `tests/test_matcher.py`
- [ ] Test keyword matching (exact, case-insensitive, word-boundary)
- [ ] Test that "Draper" doesn't match "Draperstown" (word boundary)
- [ ] Test multi-client matching
- [ ] Test context extraction
- [ ] Test merge and deduplication
- [ ] Test that client-source articles pass through unmatched
- [ ] Test with zero general-source matches

**✅ Checkpoint 2.4:**
```bash
pytest tests/test_matcher.py -v
```

### 2.5 Commit

- [ ] Commit: "Add keyword matcher for general sources + merge logic"

---

## Phase 3 — State Manager

### 3.1 State File Operations

- [ ] Implement `state_manager.py` with `load_state()`, `is_processed()`, `update_state()`, `prune_state()`
- [ ] Handle missing or corrupted state file (treat as empty)
- [ ] Prune entries older than 90 days

**✅ Checkpoint 3.1:**
```bash
python -c "
from src.state_manager import load_state, update_state, prune_state, is_processed

state = load_state()
update_state(state, [{'url_hash': 'test123', 'url': 'https://example.com', 'title': 'Test', 'source': 'Test', 'source_path': 'client', 'processed_date': '2026-04-01', 'client': 'Test'}])
print(f'Is test123 processed? {is_processed(state, \"test123\")}')
print(f'Is unknown processed? {is_processed(state, \"unknown\")}')
print('State manager OK')
"
```

### 3.2 Git Commit Logic

- [ ] Implement `commit_state()` using `subprocess` for git commands
- [ ] Handle no-changes case gracefully

### 3.3 Unit Tests

- [ ] Write `tests/test_state_manager.py`

**✅ Checkpoint 3.3:**
```bash
pytest tests/test_state_manager.py -v
```

### 3.4 Commit

- [ ] Commit: "Add state manager: load, update, prune, save state file"

---

## Phase 4 — LLM Summarizer + Highlights (Core LLM Work)

### 4.1 Shared LLM Client

- [ ] Implement `llm.py` with shared `call_claude()` function
- [ ] Exponential backoff + jitter
- [ ] Retry on 429/529 errors

**✅ Checkpoint 4.1:**
```bash
python -c "
from src.llm import call_claude
response = call_claude(
    system_prompt='You are a helpful assistant.',
    user_prompt='Respond with exactly: {\"status\": \"ok\"}',
    temperature=0.3
)
print(f'Response: {response[:100]}')
print('LLM client OK')
"
```

---

### 4.2 Per-Client Summarizer — Single Client

- [ ] Implement `summarizer.py` with the per-client prompt from ARCHITECTURE.md
- [ ] Handle articles from both paths (client-source needs project categorization, general-source already matched)
- [ ] Parse JSON response, validate structure

**✅ Checkpoint 4.2 — Summarize ONE client:**
```bash
python -c "
from src.summarizer import summarize_client
from src.dev_cache import load_cache
import json

merged = load_cache('merged')
clients = json.load(open('config/clients.json'))

client_name = list(merged.keys())[0]
client_config = next(c for c in clients['clients'] if c['client_name'] == client_name)

print(f'Summarizing: {client_name}')
summary = summarize_client(client_name, client_config, merged[client_name])

print(f'Total articles: {summary[\"total_articles\"]}')
for ps in summary['project_summaries']:
    print(f'  {ps[\"project_name\"]} ({ps[\"mention_count\"]} articles, {ps[\"significance\"]})')
    print(f'  {ps[\"summary\"][:150]}...')
    print()
"
```
**Review the output.** Summaries should be factual, concise, and correctly categorize articles by project.

---

### 4.3 Per-Client Summarizer — All Clients

- [ ] Loop over all clients with content
- [ ] Handle individual client failures gracefully
- [ ] Cache summaries for downstream development

**✅ Checkpoint 4.3:**
```bash
python -c "
from src.summarizer import summarize_all_clients
from src.dev_cache import load_cache, save_cache
import json

merged = load_cache('merged')
clients = json.load(open('config/clients.json'))

summaries = summarize_all_clients(merged, clients['clients'])
save_cache('summaries', summaries)

for client_name, summary in summaries.items():
    print(f'{client_name}: {summary[\"total_articles\"]} articles, {len(summary[\"project_summaries\"])} projects')
print('All summaries complete (saved to cache)')
"
```

---

### 4.4 Highlight Selector

- [ ] Implement `highlights.py` with the highlight prompt from ARCHITECTURE.md
- [ ] Select 3-5 top highlights across all clients
- [ ] Note source origin (client media vs. industry publication)

**✅ Checkpoint 4.4:**
```bash
python -c "
from src.highlights import select_highlights
from src.dev_cache import load_cache, save_cache

summaries = load_cache('summaries')
highlights = select_highlights(summaries)
save_cache('highlights', highlights)

print(f'Selected {len(highlights[\"highlights\"])} highlights:')
for i, h in enumerate(highlights['highlights'], 1):
    print(f'{i}. {h[\"title\"]}')
    print(f'   {h[\"client_name\"]} / {h[\"project_name\"]}')
    print(f'   Source: {h[\"source_name\"]} ({h[\"source_path\"]})')
    print()
"
```

### 4.5 Unit Tests

- [ ] Write `tests/test_summarizer.py` and `tests/test_highlights.py`
- [ ] Mock LLM responses, test parsing and validation

**✅ Checkpoint 4.5:**
```bash
pytest tests/test_summarizer.py tests/test_highlights.py -v
```

### 4.6 Commit

- [ ] Commit: "Add LLM stages: per-client summarizer with project categorization, highlight selector"

---

## Phase 5 — Email Composition & Delivery

### 5.1 Email Template

- [ ] Create `templates/digest_email.html` (Jinja2)
- [ ] Sections: header, highlights, per-client/project sections, footer
- [ ] Include source origin badges (client media vs. industry publication)
- [ ] Inline CSS, mobile-friendly

**✅ Checkpoint 5.1 — Render with mock data, inspect in browser.**

### 5.2 Email Sender

- [ ] Implement Gmail SMTP logic
- [ ] Send test email to verify delivery

### 5.3 Error Email + Quiet Week Email

- [ ] Error notification email
- [ ] Quiet week notice

### 5.4 Unit Tests

**✅ Checkpoint 5.4:**
```bash
pytest tests/test_email.py -v
```

### 5.5 Commit

- [ ] Commit: "Add email system: HTML templates, SMTP sender, error/quiet week emails"

---

## Phase 6 — Pipeline Orchestration

### 6.1 Main Orchestrator

- [ ] Implement `main.py` — dual-path scraping → merge → summarize → email → state update
- [ ] Support `DRY_RUN` mode
- [ ] Wrap in try/except for error handling

**✅ Checkpoint 6.1 — Dry run:**
```bash
DRY_RUN=true python src/main.py
```

Expected output:
```
[INFO] Starting Client Intelligence Digest pipeline
[INFO] ─── Stage: Client-Specific Source Scraping ───
[INFO] Scraping Draper (3 sources)...
[INFO]   News Releases: 5 articles
[INFO]   In the News: 4 articles
[INFO]   Featured Stories: 3 articles
[INFO] ─── Stage: General Source Scraping ───
[INFO] Scraping 1 general sources...
[INFO]   Defense News (RSS): 28 articles
[INFO] ─── Stage: Keyword Matcher (general sources) ───
[INFO] Matching 28 general articles against 1 clients...
[INFO] Matched: 2 articles
[INFO] ─── Stage: Merge ───
[INFO] Client articles: 12, Matched general: 2, Total: 14
[INFO] Grouped by 1 clients
[INFO] ─── Stage: LLM Summarizer ───
[INFO] Summarizing client 1/1: Draper...
[INFO] ─── Stage: Highlight Selector ───
[INFO] Selected 3 highlights
[INFO] ─── DRY RUN — Email not sent, state not updated ───
[INFO] Pipeline complete in 1m 45s
```

### 6.2 Live Run

- [ ] Run without DRY_RUN, verify email arrives

### 6.3 Commit

- [ ] Commit: "Add pipeline orchestrator with dual-path scraping and DRY_RUN support"

---

## Phase 7 — GitHub Actions Deployment

### 7.1 Create GitHub Repository (private recommended)
### 7.2 Workflow File with state commit step
### 7.3 GitHub Secrets (3 secrets)
### 7.4 Manual Trigger Test
### 7.5 Cron Verification (wait one week)

---

## Phase 8 — Hardening & Polish

### 8.1 Error Scenario Testing
### 8.2 Linting & Code Quality (`ruff check src/ tests/`)
### 8.3 Full Test Suite (`pytest tests/ -v`)

---

## Estimated Timeline

| Phase | Description | Estimated Time | Cumulative |
|-------|-------------|---------------|------------|
| **Phase 0** | Project setup | ~30 min | 30 min |
| **Phase 1** | Source Scraper (two-phase, dual-path) | ~4 hours | 4.5 hours |
| **Phase 2** | Keyword Matcher + Merge | ~1.5 hours | 6 hours |
| **Phase 3** | State Manager | ~1 hour | 7 hours |
| **Phase 4** | LLM Summarizer + Highlights | ~2 hours | 9 hours |
| **Phase 5** | Email Composition & Delivery | ~1.5 hours | 10.5 hours |
| **Phase 6** | Pipeline Orchestration | ~1 hour | 11.5 hours |
| **Phase 7** | GitHub Actions Deployment | ~30 min | 12 hours |
| **Phase 8** | Hardening & Polish | ~1 hour | 13 hours |

**Total estimated effort: ~13 hours**

Phase 1 is the most variable — it now involves three extraction strategies plus trafilatura integration. Budget extra time there. Phase 4 (LLM stages) may also take longer for prompt iteration.

---

## Definition of Done

The project is **complete** when:

- [ ] Dual-path scraping works (client-specific + general sources)
- [ ] Pipeline runs weekly via GitHub Actions without manual intervention
- [ ] Digest email arrives Monday morning with correct content
- [ ] Highlights reflect genuinely significant developments
- [ ] Per-client sections correctly categorize articles by project
- [ ] Articles from both paths (client media + industry publications) are represented
- [ ] All source links work
- [ ] No duplicate articles across consecutive weeks
- [ ] Error emails arrive when the pipeline fails
- [ ] Quiet week notices arrive when no content is found
- [ ] Pipeline completes in under 15 minutes
- [ ] Weekly cost is under $2.00
- [ ] All unit tests pass
- [ ] Code passes ruff linting
- [ ] Ran successfully for 2+ consecutive weeks unattended
