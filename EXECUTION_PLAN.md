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

Nothing advances until the current piece verifiably works. Every section below ends with a **Checkpoint** block that defines exactly what "working" means.

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
- [ ] Create `.gitignore` (must include `.env`, `__pycache__/`, `.dev_cache/`)
- [ ] Install dependencies in a virtual environment

**✅ Checkpoint 0.1:**
```bash
git status

python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

python -c "import feedparser; import requests; import bs4; import anthropic; import jinja2; print('All imports OK')"
```
Expected: Git repo initialized. All imports succeed, no version conflicts.

---

### 0.2 Configuration Files

- [ ] Create `config/sources.json` with 1-2 example sources (at least one RSS, one HTML)
- [ ] Create `config/clients.json` with 1-2 example clients with projects and keywords
- [ ] Create `config/subscribers.json` with a placeholder email
- [ ] Create `state/processed.json` with an empty initial structure
- [ ] Create `.env.example` with documented variables

**✅ Checkpoint 0.2:**
```bash
python -c "
import json
sources = json.load(open('config/sources.json'))
clients = json.load(open('config/clients.json'))
subs = json.load(open('config/subscribers.json'))
state = json.load(open('state/processed.json'))
print(f'Sources: {len(sources[\"sources\"])}')
print(f'Clients: {len(clients[\"clients\"])}')
print(f'Subscribers: {len(subs[\"subscribers\"])}')
print(f'Processed articles: {len(state[\"processed_articles\"])}')
print('Config OK')
"
```
Expected: All config files load and parse correctly.

---

### 0.3 Logging Setup

- [ ] Configure Python `logging` module in a shared utility
- [ ] Set format to include timestamp, level, and module name
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
Expected: INFO and WARNING print; DEBUG is suppressed at INFO level.

---

### 0.4 Development Cache Utility

During development, we'll need to iterate on downstream stages (summarizer prompts, highlight prompts) without re-running expensive upstream stages every time.

- [ ] Create `src/dev_cache.py` with `save_cache(key, data)` and `load_cache(key)` functions
- [ ] Cache location: `.dev_cache/` directory (git-ignored)
- [ ] Each cache entry is a JSON file named by key (e.g., `articles.json`, `matches.json`)
- [ ] Cache is date-stamped — auto-invalidates if the date changes

**How it's used during development:**

| When Iterating On... | Load From Cache | Run Live |
|---|---|---|
| Keyword matcher (Phase 2) | Articles from scraper | Matcher only |
| Summarizer prompt (Phase 4) | Articles + matches | Summarizer only |
| Highlight prompt (Phase 4) | Articles + matches + summaries | Highlight selector only |
| Email template (Phase 5) | All intermediate data | Email composer only |

**✅ Checkpoint 0.4:**
```bash
python -c "
from src.dev_cache import save_cache, load_cache
save_cache('test', {'hello': 'world', 'count': 42})
result = load_cache('test')
assert result['hello'] == 'world'
assert result['count'] == 42
print('Dev cache OK')
"
```
Expected: Data round-trips through cache correctly.

---

### 0.5 First Commit

- [ ] Stage all Phase 0 files
- [ ] Commit: "Initial project setup: structure, config, logging, dev cache"

**✅ Checkpoint 0.5:**
```bash
git add -A
git commit -m "Initial project setup: structure, config, logging, dev cache"
git log --oneline
```
Expected: Clean first commit with all scaffolding.

---

## Phase 1 — Source Scraper (No LLM Needed)

This is the foundation. If we can't reliably scrape sources, nothing else matters.

### 1.1 RSS Adapter

- [ ] Implement `scraper.py` with RSS adapter using `feedparser`
- [ ] Parse feed entries into `Article` dataclass (url, url_hash, title, body, publish_date, source_name, source_type)
- [ ] Filter entries to the configured lookback window (default 7 days)
- [ ] Handle date parsing across common feed formats

**✅ Checkpoint 1.1 — Live RSS test:**
```bash
python -c "
from src.scraper import fetch_rss_articles

articles = fetch_rss_articles({
    'name': 'Test RSS Source',
    'url': 'https://rss.nytimes.com/services/xml/rss/nyt/World.xml',
    'type': 'rss'
})
print(f'Fetched {len(articles)} articles from RSS')
if articles:
    print(f'First article: {articles[0].title}')
    print(f'URL: {articles[0].url}')
    print(f'Date: {articles[0].publish_date}')
    print(f'Body length: {len(articles[0].body)} chars')
"
```
Expected: Articles returned with populated title, URL, date, and body text.

---

### 1.2 HTML Adapter

- [ ] Implement HTML adapter in `scraper.py` using `requests` + `BeautifulSoup`
- [ ] Accept per-source CSS selectors from config
- [ ] Extract article list, then title/link/date/body for each article
- [ ] Handle relative URLs (convert to absolute)

**✅ Checkpoint 1.2 — Live HTML test:**
```bash
python -c "
from src.scraper import fetch_html_articles

articles = fetch_html_articles({
    'name': 'Test HTML Source',
    'url': 'https://example.com/news',
    'type': 'html',
    'selectors': {
        'article_list': 'div.article',
        'title': 'h2 a',
        'link': 'h2 a@href',
        'date': 'span.date',
        'body': 'div.content'
    }
})
print(f'Fetched {len(articles)} articles from HTML')
for a in articles[:3]:
    print(f'  - {a.title[:60]}... ({a.publish_date})')
"
```
Expected: Articles extracted using CSS selectors. May need to adjust selectors for the actual test source.

**Note:** The HTML adapter will need per-source selector tuning. This is expected — each HTML source has its own page structure. The config file accommodates this.

---

### 1.3 Unified Scraper Interface

- [ ] Implement `fetch_all_sources()` that routes each source to the correct adapter based on `type`
- [ ] Add rate limiting (3s between requests to the same domain)
- [ ] Add per-source error handling (one source fails → log warning, continue with others)
- [ ] Deduplicate articles by URL across sources
- [ ] Log summary: articles per source, total, failures

**✅ Checkpoint 1.3:**
```bash
python -c "
from src.scraper import fetch_all_sources
import json

sources = json.load(open('config/sources.json'))
articles, failures = fetch_all_sources(sources['sources'])

print(f'Total articles: {len(articles)}')
print(f'Failed sources: {len(failures)}')
for f in failures:
    print(f'  ⚠ {f[\"name\"]}: {f[\"error\"]}')
"
```
Expected: Articles from all working sources. Any failures are logged but don't crash the pipeline.

---

### 1.4 State Filtering

- [ ] Implement state-based filtering: skip articles whose URL hash exists in `state/processed.json`
- [ ] Log count of articles filtered by state

**✅ Checkpoint 1.4:**
```bash
python -c "
from src.scraper import fetch_all_sources
from src.state_manager import load_state, is_processed
import json

sources = json.load(open('config/sources.json'))
articles, _ = fetch_all_sources(sources['sources'])
state = load_state()

new_articles = [a for a in articles if not is_processed(state, a.url_hash)]
print(f'Total articles: {len(articles)}')
print(f'Already processed: {len(articles) - len(new_articles)}')
print(f'New articles: {len(new_articles)}')
"
```
Expected: On first run, all articles are new. On subsequent runs, previously processed articles are filtered out.

---

### 1.5 Cache Scraped Articles (Dev Support)

- [ ] After a successful scrape, auto-save articles to dev cache
- [ ] Downstream phases can load from cache during development

**✅ Checkpoint 1.5:**
```bash
python -c "
from src.scraper import fetch_all_sources
from src.dev_cache import save_cache, load_cache
import json

sources = json.load(open('config/sources.json'))
articles, _ = fetch_all_sources(sources['sources'])
save_cache('articles', [a.__dict__ for a in articles])
cached = load_cache('articles')
print(f'Cached {len(cached)} articles to .dev_cache/articles.json')
"
```
Expected: Articles saved to cache for downstream development.

---

### 1.6 Unit Tests for Scraper

- [ ] Write `tests/test_scraper.py`
- [ ] Test Article dataclass construction
- [ ] Test RSS parsing (mock feedparser response)
- [ ] Test HTML parsing (mock HTML content with selectors)
- [ ] Test date filtering logic
- [ ] Test deduplication logic
- [ ] Test error handling (mock network failures)

**✅ Checkpoint 1.6:**
```bash
pytest tests/test_scraper.py -v
```
Expected: All tests pass.

### 1.7 Commit

- [ ] Commit: "Add source scraper: RSS + HTML adapters, rate limiting, state filtering"

---

## Phase 2 — Keyword Matcher (No LLM Needed)

### 2.1 Basic Keyword Matching

- [ ] Implement `matcher.py` with keyword matching against client registry
- [ ] Load client registry from `config/clients.json`
- [ ] For each article, scan title and body for client names, aliases, project names, and keywords
- [ ] Use word-boundary matching (regex `\b...\b`, case-insensitive)

**✅ Checkpoint 2.1 — Match against real articles:**
```bash
python -c "
from src.scraper import Article
from src.matcher import match_articles
from src.dev_cache import load_cache
import json

articles = [Article(**a) for a in load_cache('articles')]
clients = json.load(open('config/clients.json'))

matches = match_articles(articles, clients['clients'])

print(f'Articles scanned: {len(articles)}')
print(f'Articles with matches: {len(set(m.article.url for m in matches))}')
print(f'Total matches: {len(matches)}')
print()
for m in matches[:10]:
    print(f'  [{m.client_name}] [{m.project_name or \"General\"}] {m.article.title[:60]}...')
    print(f'    Keywords: {m.matched_keywords}')
    print()
"
```
Expected: Matches found for articles that mention clients/projects. No false positives on unrelated articles.

---

### 2.2 Context Extraction

- [ ] When a keyword matches, extract the sentence(s) containing the keyword
- [ ] Store as `context_snippets` on the Match object
- [ ] These snippets are sent to the LLM summarizer for grounding

**✅ Checkpoint 2.2:**
```bash
python -c "
from src.scraper import Article
from src.matcher import match_articles
from src.dev_cache import load_cache
import json

articles = [Article(**a) for a in load_cache('articles')]
clients = json.load(open('config/clients.json'))

matches = match_articles(articles, clients['clients'])

for m in matches[:3]:
    print(f'Match: {m.client_name} / {m.project_name}')
    print(f'Keywords: {m.matched_keywords}')
    print(f'Context:')
    for snippet in m.context_snippets[:2]:
        print(f'  \"...{snippet[:120]}...\"')
    print()
"
```
Expected: Each match includes relevant sentence(s) from the article.

---

### 2.3 Match Grouping

- [ ] Group matches by client → project → list of articles
- [ ] Include a "General" bucket for client-level matches without a specific project
- [ ] Output a structured `MatchResults` object ready for the summarizer

**✅ Checkpoint 2.3:**
```bash
python -c "
from src.scraper import Article
from src.matcher import match_articles, group_matches
from src.dev_cache import load_cache, save_cache
import json

articles = [Article(**a) for a in load_cache('articles')]
clients = json.load(open('config/clients.json'))

matches = match_articles(articles, clients['clients'])
grouped = group_matches(matches)

save_cache('matches', grouped)

for client_name, projects in grouped.items():
    total = sum(len(arts) for arts in projects.values())
    print(f'{client_name} ({total} mentions)')
    for project_name, arts in projects.items():
        print(f'  {project_name}: {len(arts)} articles')
print()
print('Matches cached for downstream development')
"
```
Expected: Matches cleanly grouped by client and project.

---

### 2.4 Unit Tests for Matcher

- [ ] Write `tests/test_matcher.py`
- [ ] Test keyword matching (exact, case-insensitive, word-boundary)
- [ ] Test that "Titan" doesn't match "Titanium" (word boundary)
- [ ] Test multi-client matching (one article matches two clients)
- [ ] Test context extraction
- [ ] Test grouping logic
- [ ] Test with zero matches (returns empty structure gracefully)

**✅ Checkpoint 2.4:**
```bash
pytest tests/test_matcher.py -v
```
Expected: All tests pass.

### 2.5 Commit

- [ ] Commit: "Add keyword matcher: word-boundary matching, context extraction, grouping"

---

## Phase 3 — State Manager

### 3.1 State File Operations

- [ ] Implement `state_manager.py` with `load_state()`, `is_processed()`, `update_state()`, `prune_state()`
- [ ] Handle missing or corrupted state file gracefully (treat as empty)
- [ ] Prune entries older than 90 days

**✅ Checkpoint 3.1:**
```bash
python -c "
from src.state_manager import load_state, update_state, prune_state, is_processed

state = load_state()
print(f'Current entries: {len(state[\"processed_articles\"])}')

# Add a test entry
update_state(state, [{
    'url_hash': 'test123',
    'url': 'https://example.com/test',
    'title': 'Test Article',
    'source': 'Test Source',
    'processed_date': '2026-04-01',
    'matched_clients': ['Test Client']
}])
print(f'After update: {len(state[\"processed_articles\"])} entries')
print(f'Is test123 processed? {is_processed(state, \"test123\")}')
print(f'Is unknown processed? {is_processed(state, \"unknown\")}')

pruned = prune_state(state, max_age_days=90)
print(f'After prune: {len(pruned[\"processed_articles\"])} entries')
print('State manager OK')
"
```
Expected: State operations work correctly.

---

### 3.2 Git Commit Logic

- [ ] Implement `commit_state()` that writes the state file and commits it
- [ ] Use `subprocess` to run git commands
- [ ] Handle the case where there are no changes to commit (no-op)

**✅ Checkpoint 3.2:**
```bash
python -c "
from src.state_manager import save_state, commit_state

# This test only verifies the save; actual git commit is tested in integration
save_state({'last_run': '2026-04-01T12:00:00Z', 'processed_articles': []})
print('State file saved to state/processed.json')
print('Git commit will be tested in Phase 6 integration')
"
```
Expected: State file writes correctly. Git commit tested during integration.

---

### 3.3 Unit Tests for State Manager

- [ ] Write `tests/test_state_manager.py`
- [ ] Test load from empty/missing file
- [ ] Test is_processed lookup
- [ ] Test update with new entries
- [ ] Test pruning old entries
- [ ] Test save/load round-trip

**✅ Checkpoint 3.3:**
```bash
pytest tests/test_state_manager.py -v
```
Expected: All tests pass.

### 3.4 Commit

- [ ] Commit: "Add state manager: load, update, prune, save state file"

---

## Phase 4 — LLM Summarizer + Highlights (Core LLM Work)

### 4.1 Shared LLM Client

- [ ] Implement `llm.py` with a shared `call_claude()` function
- [ ] Exponential backoff (5s → 10s → 20s → 40s → 60s cap)
- [ ] Random jitter (0–50% of delay)
- [ ] Retry on 429 (rate limit) and 529 (overloaded) errors
- [ ] Up to 5 retry attempts before failing

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
Expected: Claude responds successfully. Retry logic tested implicitly over time.

---

### 4.2 Per-Client Summarizer — Single Client

- [ ] Implement `summarizer.py` with the per-client summarizer prompt from ARCHITECTURE.md
- [ ] Take a single client's grouped matches and return structured summaries
- [ ] Parse JSON response, validate structure

**✅ Checkpoint 4.2 — Summarize ONE client:**
```bash
python -c "
from src.summarizer import summarize_client
from src.dev_cache import load_cache
import json

matches = load_cache('matches')
clients = json.load(open('config/clients.json'))

# Pick the first client with matches
client_name = list(matches.keys())[0]
client_config = next(c for c in clients['clients'] if c['client_name'] == client_name)

print(f'Summarizing: {client_name}')
print()

summary = summarize_client(client_name, client_config, matches[client_name])

print(f'Total mentions: {summary[\"total_mentions\"]}')
for ps in summary['project_summaries']:
    print(f'  {ps[\"project_name\"]} ({ps[\"mention_count\"]} mentions, {ps[\"significance\"]})')
    print(f'  {ps[\"summary\"][:150]}...')
    print()
"
```
**Review the output carefully.** The summaries should be factual, concise, and reference what the sources actually reported.

---

### 4.3 Per-Client Summarizer — All Clients

- [ ] Implement loop to summarize all clients with matches
- [ ] Handle individual client failures (log warning, skip, continue)
- [ ] Log progress (client 1/N, 2/N, etc.)

**✅ Checkpoint 4.3 — Full summarization:**
```bash
python -c "
from src.summarizer import summarize_all_clients
from src.dev_cache import load_cache, save_cache
import json

matches = load_cache('matches')
clients = json.load(open('config/clients.json'))

print(f'Summarizing {len(matches)} clients...')
print()

summaries = summarize_all_clients(matches, clients['clients'])

save_cache('summaries', summaries)

for client_name, summary in summaries.items():
    print(f'{client_name}: {summary[\"total_mentions\"]} mentions, {len(summary[\"project_summaries\"])} projects')

print()
print(f'All summaries complete (saved to cache)')
"
```
Expected: Summaries generated for all clients with matches.

---

### 4.4 Highlight Selector

- [ ] Implement `highlights.py` with the highlight selector prompt from ARCHITECTURE.md
- [ ] Take all client summaries and return 3-5 ordered highlights
- [ ] Parse JSON response, validate structure

**✅ Checkpoint 4.4 — Select highlights:**
```bash
python -c "
from src.highlights import select_highlights
from src.dev_cache import load_cache, save_cache

summaries = load_cache('summaries')

highlights = select_highlights(summaries)

save_cache('highlights', highlights)

print(f'Selected {len(highlights[\"highlights\"])} highlights:')
print()
for i, h in enumerate(highlights['highlights'], 1):
    print(f'{i}. {h[\"title\"]}')
    print(f'   Client: {h[\"client_name\"]} / {h[\"project_name\"]}')
    print(f'   {h[\"analysis\"][:150]}...')
    print(f'   Source: {h[\"source_name\"]}')
    print()
"
```
**Review the highlights carefully.** They should represent genuinely significant developments, not routine mentions.

---

### 4.5 Unit Tests for LLM Stages

- [ ] Write `tests/test_summarizer.py`
- [ ] Write `tests/test_highlights.py`
- [ ] Test prompt assembly (mock data)
- [ ] Test JSON response parsing (mock LLM responses — valid and malformed)
- [ ] Test validation logic
- [ ] Test graceful failure handling

**✅ Checkpoint 4.5:**
```bash
pytest tests/test_summarizer.py tests/test_highlights.py -v
```
Expected: All tests pass. No real API calls (all mocked).

### 4.6 Commit

- [ ] Commit: "Add LLM stages: per-client summarizer, highlight selector, shared client"

---

## Phase 5 — Email Composition & Delivery

### 5.1 Email Template

- [ ] Create `templates/digest_email.html` (Jinja2 template)
- [ ] Inline CSS styling (single-column, mobile-friendly)
- [ ] Sections: header with stats, highlights, per-client sections, footer
- [ ] Include links to original sources for every mention

**✅ Checkpoint 5.1 — Render template with mock data:**
```bash
python -c "
from src.email_composer import compose_email

html = compose_email(
    highlights=[
        {
            'title': 'Acme Wins Major Contract',
            'client_name': 'Acme Defense',
            'project_name': 'Project Falcon',
            'analysis': 'Acme Defense secured a \$500M contract for the next phase of Project Falcon. This represents a significant expansion of the program.',
            'source_url': 'https://example.com/article/1',
            'source_name': 'Defense News'
        }
    ],
    client_summaries={
        'Acme Defense': {
            'total_mentions': 3,
            'project_summaries': [
                {
                    'project_name': 'Project Falcon',
                    'summary': 'Multiple sources reported on the Project Falcon contract win this week. The \$500M award covers Phase 3 development.',
                    'mention_count': 2,
                    'significance': 'high',
                    'articles': [
                        {'title': 'Acme Wins Contract', 'url': 'https://example.com/1', 'source': 'Defense News'},
                        {'title': 'Falcon Phase 3 Details', 'url': 'https://example.com/2', 'source': 'Industry Weekly'}
                    ]
                }
            ],
            'general_mentions': {'summary': 'Acme mentioned in year-end industry review.', 'mention_count': 1}
        }
    },
    stats={
        'sources_scanned': 12,
        'articles_processed': 245,
        'articles_matched': 8,
        'clients_mentioned': 3,
        'date_range': 'Mar 24 – Mar 30, 2026'
    }
)

with open('/tmp/test_digest.html', 'w') as f:
    f.write(html)
print(f'Email rendered: {len(html)} chars')
print('Saved to /tmp/test_digest.html — open in a browser to inspect')
"
```
**Open `/tmp/test_digest.html` in a browser.** Check layout, links, mobile rendering.

---

### 5.2 Email Sender

- [ ] Implement `email_sender.py` with Gmail SMTP logic
- [ ] Support sending to multiple subscribers
- [ ] Log delivery status per recipient

**✅ Checkpoint 5.2 — Send a real test email:**
```bash
python -c "
from src.email_sender import send_email

html = '<h1>Client Intelligence Digest — Test</h1><p>If you can read this, email sending works.</p>'

send_email(
    subject='🧪 Client Digest — Test Email',
    html_body=html,
    recipients=[{'email': 'YOUR_EMAIL_HERE', 'name': 'Test'}]
)
print('Test email sent — check your inbox (and spam folder)')
"
```
**Check your inbox.** Verify arrival, subject line, HTML rendering.

---

### 5.3 Error Email

- [ ] Implement error email formatting in `error_handler.py`
- [ ] Include stage name, error message, truncated traceback, timestamp
- [ ] Note which sources were attempted

**✅ Checkpoint 5.3:**
```bash
python -c "
from src.error_handler import send_error_email

try:
    raise ValueError('Simulated failure in Keyword Matcher — invalid regex pattern')
except Exception as e:
    send_error_email(
        stage='Keyword Matcher',
        error=e,
        context={'sources_scraped': 12, 'articles_fetched': 245}
    )
    print('Error email sent — check your inbox')
"
```
Expected: Error email arrives with clear failure description.

---

### 5.4 Quiet Week Email

- [ ] Create `templates/quiet_week_email.html`
- [ ] Implement quiet week notice (no matches found)

**✅ Checkpoint 5.4:**
```bash
python -c "
from src.email_composer import compose_quiet_week_email
from src.email_sender import send_email

html = compose_quiet_week_email(
    date_range='Mar 24 – Mar 30, 2026',
    sources_scanned=12,
    articles_processed=245
)
send_email(
    subject='📊 Client Intelligence Digest — Quiet Week (Mar 24–30)',
    html_body=html,
    recipients=[{'email': 'YOUR_EMAIL_HERE', 'name': 'Test'}]
)
print('Quiet week email sent')
"
```
Expected: Clean email confirming the system ran but found no matches.

---

### 5.5 Unit Tests for Email

- [ ] Write `tests/test_email.py`
- [ ] Test template rendering with various data shapes (0 highlights, many clients, missing fields)
- [ ] Test SMTP connection logic (mocked)
- [ ] Test error email formatting
- [ ] Test quiet week email

**✅ Checkpoint 5.5:**
```bash
pytest tests/test_email.py -v
```
Expected: All tests pass.

### 5.6 Commit

- [ ] Commit: "Add email system: HTML templates, SMTP sender, error/quiet week emails"

---

## Phase 6 — Pipeline Orchestration (Wiring It All Together)

### 6.1 Main Orchestrator

- [ ] Implement `main.py` that calls each stage in sequence
- [ ] Wrap entire pipeline in try/except for error handling
- [ ] Add logging at each stage transition
- [ ] Support `DRY_RUN` mode (runs everything but doesn't send email or update state)
- [ ] Handle partial source failures gracefully

**Note:** `main.py` does NOT use the dev cache. It runs the full pipeline live every time. The dev cache is only for interactive development in earlier phases.

**✅ Checkpoint 6.1 — Dry run the full pipeline:**
```bash
DRY_RUN=true python src/main.py
```
**This is the full integration test.**

Expected output (approximately):
```
[INFO] Starting Client Intelligence Digest pipeline
[INFO] Date range: 2026-03-24 to 2026-03-30
[INFO] ─── Stage: Source Scraper ───
[INFO] Scraping 12 sources (8 RSS, 4 HTML)...
[INFO] Source: Defense News — 34 articles
[INFO] Source: Industry Weekly — 12 articles
[INFO] ...
[INFO] Total: 245 articles (230 new, 15 already processed)
[INFO] ─── Stage: Keyword Matcher ───
[INFO] Matching 230 articles against 8 clients...
[INFO] Matched 12 articles across 4 clients
[INFO]   Acme Defense: 4 mentions (2 projects)
[INFO]   Northgate Industries: 3 mentions (1 project)
[INFO]   ...
[INFO] ─── Stage: LLM Summarizer ───
[INFO] Summarizing client 1/4: Acme Defense...
[INFO] Summarizing client 2/4: Northgate Industries...
[INFO] ...
[INFO] ─── Stage: Highlight Selector ───
[INFO] Selecting highlights from 4 client summaries...
[INFO] Selected 4 highlights
[INFO] ─── Stage: Email Composer ───
[INFO] Email composed: 28,450 chars
[INFO] ─── DRY RUN — Email not sent, state not updated ───
[INFO] Would send to: accountmanager@example.com
[INFO] Subject: 📊 Client Intelligence Digest — Week of Mar 24–30, 2026
[INFO] Pipeline complete in 2m 15s
```

Verify:
- Every stage completes without errors
- Matches are reasonable
- Summaries are accurate
- Total runtime is under 15 minutes

---

### 6.2 Live Run (Real Email)

- [ ] Run the pipeline without DRY_RUN
- [ ] Send the actual digest email to yourself

**✅ Checkpoint 6.2 — The real thing:**
```bash
python src/main.py
```
**Check your inbox. Read the entire digest.**

Evaluate:
- [ ] Email arrives promptly
- [ ] Formatting looks good on desktop
- [ ] Formatting looks good on mobile (check on phone)
- [ ] Highlights are genuinely significant, not routine mentions
- [ ] Per-client sections are organized and informative
- [ ] All source links work (click each one)
- [ ] Stats footer is accurate
- [ ] State file was updated correctly
- [ ] Overall: would this be useful for the account manager's weekly review?

**If anything isn't right, iterate before moving to Phase 7.**

### 6.3 Commit

- [ ] Commit: "Add pipeline orchestrator with DRY_RUN support"

---

## Phase 7 — GitHub Actions Deployment

### 7.1 Create GitHub Repository

- [ ] Create a new repository on GitHub (private recommended — client names in config)
- [ ] Add the remote: `git remote add origin git@github.com:<user>/client-intelligence-digest.git`
- [ ] Push all existing commits: `git push -u origin main`

**✅ Checkpoint 7.1:**
```bash
git remote -v
git log --oneline
```
Verify: All commits from Phases 0–6 are present. Remote points to GitHub repo.

---

### 7.2 Workflow File

- [ ] Create `.github/workflows/weekly_digest.yml` per ARCHITECTURE.md
- [ ] Configure cron schedule: `0 12 * * 1` (7 AM ET, every Monday)
- [ ] Add `workflow_dispatch` for manual trigger
- [ ] Add state commit step
- [ ] Set 15-minute timeout
- [ ] Commit: "Add GitHub Actions workflow for weekly digest"

**✅ Checkpoint 7.2:**
```bash
cat .github/workflows/weekly_digest.yml
```
Verify: YAML syntax is valid, cron is correct, state commit step is present.

---

### 7.3 GitHub Secrets

- [ ] Add `ANTHROPIC_API_KEY` to repository secrets
- [ ] Add `GMAIL_ADDRESS` to repository secrets
- [ ] Add `GMAIL_APP_PASSWORD` to repository secrets

**✅ Checkpoint 7.3:**
Navigate to GitHub → Repository → Settings → Secrets and variables → Actions.
Verify all 3 secrets are listed.

---

### 7.4 Manual Trigger Test

- [ ] Push workflow file to GitHub
- [ ] Trigger the workflow manually via GitHub UI (Actions → Run workflow)
- [ ] Monitor the run in real time

**✅ Checkpoint 7.4:**
- [ ] Go to Actions tab → click the workflow run
- [ ] Watch each step complete
- [ ] Verify email arrives in inbox
- [ ] Verify state file was committed back to repo
- [ ] Check logs for any warnings or errors
- [ ] Confirm total runtime is under 15 minutes

---

### 7.5 Cron Verification

- [ ] Leave the cron running for one week
- [ ] Next Monday morning: verify the digest arrived at ~7 AM ET

**✅ Checkpoint 7.5:**
- [ ] Email arrived without manual intervention
- [ ] Content covers the previous week's articles correctly
- [ ] State file was updated (check git log)
- [ ] No errors in GitHub Actions logs

---

## Phase 8 — Hardening & Polish

### 8.1 Error Scenario Testing

Deliberately break things to verify error handling works:

- [ ] **Test: Invalid API key** — Set a bad `ANTHROPIC_API_KEY` → verify error email arrives
- [ ] **Test: Source URL down** — Point a source at a dead URL → verify pipeline continues with others
- [ ] **Test: No matches** — Use a client registry with impossible keywords → verify quiet week email
- [ ] **Test: LLM returns garbage** — Mock invalid JSON → verify retry logic works
- [ ] **Test: State file corrupted** — Corrupt `state/processed.json` → verify fresh-start behavior

**✅ Checkpoint 8.1:**
Each error scenario results in either graceful degradation or a clear error notification. No silent failures.

---

### 8.2 Linting & Code Quality

- [ ] Run ruff on all source files
- [ ] Fix any linting issues
- [ ] Verify type hints are consistent

**✅ Checkpoint 8.2:**
```bash
ruff check src/ tests/
ruff format --check src/ tests/
```
Expected: No errors, no formatting changes needed.

---

### 8.3 Full Test Suite

- [ ] Run all unit tests together
- [ ] Verify all pass

**✅ Checkpoint 8.3:**
```bash
pytest tests/ -v --tb=short
```
Expected: All tests pass. No warnings.

---

## Testing Strategy Summary

### Test Types by Phase

| Phase | Test Type | Real API Calls? | Purpose |
|-------|-----------|-----------------|---------|
| 0 | Import & config checks | No | Verify setup |
| 1 | Live source scraping + unit tests | Yes (HTTP only) | Verify scraping works |
| 2 | Keyword matching + unit tests | No | Verify matching logic |
| 3 | State file operations + unit tests | No | Verify state management |
| 4 | **Live LLM calls** + unit tests | **Yes (Claude)** — cached articles/matches | Verify summary quality |
| 5 | Live email send + unit tests | Yes (Gmail) — cached summaries | Verify delivery works |
| 6 | **Full pipeline dry run + live run** | **Yes (all)** | End-to-end integration |
| 7 | GitHub Actions run | Yes (all) | Verify CI/CD works |
| 8 | Deliberate failure tests | Mixed | Verify error handling |

### Unit Tests (Mocked — Fast, Free)

| Test File | Mocks | Tests |
|-----------|-------|-------|
| `test_scraper.py` | HTTP requests, feedparser | RSS/HTML parsing, date filtering, dedup |
| `test_matcher.py` | None (pure logic) | Keyword matching, context extraction, grouping |
| `test_state_manager.py` | File I/O | State load, update, prune, is_processed |
| `test_summarizer.py` | Claude API | Prompt building, JSON parsing, validation |
| `test_highlights.py` | Claude API | Prompt building, JSON parsing, validation |
| `test_email.py` | SMTP | Template rendering, email construction |

---

## Estimated Timeline

| Phase | Description | Estimated Time | Cumulative |
|-------|-------------|---------------|------------|
| **Phase 0** | Project setup | ~30 min | 30 min |
| **Phase 1** | Source Scraper | ~2.5 hours | 3 hours |
| **Phase 2** | Keyword Matcher | ~1.5 hours | 4.5 hours |
| **Phase 3** | State Manager | ~1 hour | 5.5 hours |
| **Phase 4** | LLM Summarizer + Highlights | ~2 hours (prompt iteration) | 7.5 hours |
| **Phase 5** | Email Composition & Delivery | ~1.5 hours | 9 hours |
| **Phase 6** | Pipeline Orchestration | ~1 hour | 10 hours |
| **Phase 7** | GitHub Actions Deployment | ~30 min | 10.5 hours |
| **Phase 8** | Hardening & Polish | ~1.5 hours | 12 hours |

**Total estimated effort: ~12 hours**

Phase 1 (Source Scraper) is the most variable — HTML adapter development depends heavily on the specific source sites and their page structures. Budget extra time there.

Phase 4 (LLM stages) may take longer if prompt quality needs tuning.

---

## Contingency Notes

### If Running Behind

| Shortcut | Impact | When to Use |
|----------|--------|-------------|
| Skip HTML adapter (Phase 1.2) | RSS-only sources | If all sources have RSS feeds |
| Skip context extraction (Phase 2.2) | Less grounded summaries | If match context isn't adding value |
| Simplify email template (Phase 5.1) | Less polished look | If formatting is taking too long |
| Skip error scenario testing (Phase 8.1) | Less robust | If pipeline is working reliably |

### If Running Ahead

| Extension | Benefit | Effort |
|-----------|---------|--------|
| Add mention count tracking over time | Trend visibility | ~1 hour |
| Add sentiment analysis to summaries | Richer briefings | ~2 hours |
| Polish email CSS further | Better visual design | ~1 hour |
| Add source health monitoring | Know when a source goes stale | ~1 hour |

---

## Definition of Done

The project is **complete** when:

- [ ] Pipeline runs weekly via GitHub Actions without manual intervention
- [ ] Digest email arrives Monday morning with correct content
- [ ] Highlights reflect genuinely significant developments
- [ ] Per-client sections are accurate and well-organized
- [ ] All source links work
- [ ] No duplicate articles across consecutive weeks
- [ ] Error emails arrive when the pipeline fails
- [ ] Quiet week notices arrive when no matches are found
- [ ] Pipeline completes in under 15 minutes
- [ ] Weekly cost is under $2.00
- [ ] All unit tests pass
- [ ] Code passes ruff linting
- [ ] Ran successfully for 2+ consecutive weeks unattended
