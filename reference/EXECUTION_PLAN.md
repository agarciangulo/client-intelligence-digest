# Execution Plan — ArXiv AI Paper Digest Agent

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

Nothing advances until the current piece verifiably works. Every section below ends with a **✅ Checkpoint** block that defines exactly what "working" means.

---

## Phase 0 — Project Setup

### 0.1 Repository & Structure

- [ ] Initialize git repository (`git init`)
- [ ] Initialize the project directory structure:

```
arxiv-digest/
├── .github/workflows/
├── config/
├── src/
├── templates/
├── tests/
├── docs/
├── requirements.txt
├── .env.example
├── .gitignore
└── README.md
```

- [ ] Create `requirements.txt` with all dependencies (from TECH_STACK.md)
- [ ] Create `.env.example` with placeholder values
- [ ] Create `.gitignore` (must include `.env`, `*.pdf`, `__pycache__/`, `.dev_cache/`)
- [ ] Install dependencies in a virtual environment

**✅ Checkpoint 0.1:**
```bash
# Verify git is initialized
git status

# Verify the virtual environment and all deps install cleanly
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Verify key imports work
python -c "import arxiv; import fitz; import anthropic; import jinja2; print('All imports OK')"
```
Expected: Git repo initialized. All imports succeed, no version conflicts.

---

### 0.2 Configuration Files

- [ ] Create `config/user_profile.json` with the full profile from SCOPE.md
- [ ] Create `config/subscribers.json` with a placeholder email
- [ ] Create `.env.example` with documented variables

**✅ Checkpoint 0.2:**
```bash
# Verify config files parse correctly
python -c "
import json
profile = json.load(open('config/user_profile.json'))
subs = json.load(open('config/subscribers.json'))
print(f'Profile: {profile[\"profile_name\"]}')
print(f'Subscribers: {len(subs[\"subscribers\"])}')
print(f'Primary interests: {len(profile[\"primary_interests\"])}')
print('Config OK')
"
```
Expected: Profile and subscriber data load and print correctly.

---

### 0.3 Logging Setup

- [ ] Configure Python `logging` module in a shared utility
- [ ] Set format to include timestamp, level, and module name
- [ ] Support `LOG_LEVEL` environment variable

**✅ Checkpoint 0.3:**
```bash
# Verify logging works at different levels
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

During development, we'll need to iterate on downstream stages (ranker prompts, analyzer prompts) without re-running expensive upstream stages every time. A simple dev cache saves intermediate results to disk.

- [ ] Create `src/dev_cache.py` with `save_cache(key, data)` and `load_cache(key)` functions
- [ ] Cache location: `.dev_cache/` directory (git-ignored)
- [ ] Each cache entry is a JSON file named by key (e.g., `papers.json`, `ranked.json`)
- [ ] Cache is date-stamped — auto-invalidates if the date changes

**How it's used during development:**

| When Iterating On... | Load From Cache | Run Live |
|---|---|---|
| Ranker prompt (Phase 3) | Papers from collector | Ranker only |
| Analyzer prompt (Phase 4) | Papers + ranking results | Analyzer only |
| Blurb prompt (Phase 4) | Papers + ranking results | Blurb generator only |
| Email template (Phase 5) | Papers + rankings + summaries + blurbs | Email composer only |

This saves both time (~30-60s per skipped arXiv fetch) and money (~$0.10-0.15 per skipped Claude ranking call) across prompt iteration cycles.

**✅ Checkpoint 0.4:**
```bash
# Verify cache save and load
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

## Phase 1 — Data Collector (No LLM Needed)

This is the foundation. If we can't reliably get papers, nothing else matters.

### 1.0 arXiv API Investigation

**⚠️ Do this first, before writing any production code.**

The `arxiv` Python package wraps the arXiv Search API, which searches by **submission date**. But the daily listing page (`/list/cs.AI/recent`) shows papers by **announcement date** — and these are different. A paper submitted on Monday might not be announced until Wednesday.

We need to verify which approach gives us the same results as the daily listing page.

- [ ] Test the `arxiv` Python package: query `cs.AI` papers by date range, count results
- [ ] Compare results against the actual `/list/cs.AI/recent` page for the same date
- [ ] If they don't match, test alternatives:
  - arXiv RSS feed (`http://arxiv.org/rss/cs.AI`) — shows latest announcements
  - arXiv OAI-PMH API — supports querying by announcement set/date
  - Parsing the HTML listing page (last resort)
- [ ] Document which approach gives us the correct, complete daily listing

**✅ Checkpoint 1.0 — Validate data source:**
```bash
# Compare our API results against the web listing
python -c "
# Step 1: Fetch via the arxiv package
import arxiv

search = arxiv.Search(
    query='cat:cs.AI',
    max_results=300,
    sort_by=arxiv.SortCriterion.SubmittedDate,
    sort_order=arxiv.SortOrder.Descending,
)
api_papers = list(search.results())
api_ids = {r.entry_id.split('/')[-1].split('v')[0] for r in api_papers}
print(f'arxiv package returned: {len(api_papers)} papers')
print(f'Unique IDs: {len(api_ids)}')
print()
print('Sample titles:')
for p in api_papers[:5]:
    print(f'  - {p.title[:80]}')
print()
print('ACTION: Manually compare these against https://arxiv.org/list/cs.AI/recent')
print('Do the same papers appear? Is the count roughly the same?')
"
```

**Expected outcome:** Either the `arxiv` package gives us matching results (great, proceed), or we identify a gap and switch to RSS/OAI-PMH before building everything on top of the wrong data source.

**This is a blocker. Do not proceed to 1.1 until the data source is validated.**

---

### 1.1 Basic arXiv Fetch

- [ ] Implement `collector.py` using the **validated approach from 1.0**
- [ ] Parse results into `Paper` dataclass (arxiv_id, title, authors, abstract, comments, subjects, pdf_url, html_url, published_date)
- [ ] Handle the date logic (previous business day calculation)

**✅ Checkpoint 1.1 — Live API test:**
```bash
# Fetch yesterday's papers and print stats
python -c "
from src.collector import fetch_papers
papers = fetch_papers()
print(f'Fetched {len(papers)} papers')
print(f'First paper: {papers[0].title}')
print(f'Authors: {papers[0].authors[:3]}')
print(f'Abstract length: {len(papers[0].abstract)} chars')
print(f'PDF URL: {papers[0].pdf_url}')
print(f'Has comments: {papers[0].comments is not None}')
"
```
Expected: ~150-200 papers returned, matching the daily listing page. Each paper has title, authors, abstract, and a valid PDF URL.

### 1.2 Deduplication

- [ ] Add deduplication by arxiv_id
- [ ] Log count of duplicates removed

**✅ Checkpoint 1.2:**
```bash
# Verify dedup works by checking no duplicate IDs
python -c "
from src.collector import fetch_papers
papers = fetch_papers()
ids = [p.arxiv_id for p in papers]
unique_ids = set(ids)
print(f'Total: {len(ids)}, Unique: {len(unique_ids)}, Dupes removed: {len(ids) - len(unique_ids)}')
assert len(ids) == len(unique_ids), 'Duplicates found!'
print('Dedup OK')
"
```
Expected: Total == Unique (all duplicates removed).

### 1.3 Edge Cases

- [ ] Handle "no papers found" (returns empty list gracefully)
- [ ] Handle arXiv API errors (retry 3x with backoff)
- [ ] Handle date edge cases (Monday → fetch Friday's papers)

**✅ Checkpoint 1.3:**
```bash
# Test date logic
python -c "
from src.collector import get_previous_business_day
from datetime import date

# Test each day of the week
test_cases = [
    ('Tuesday', date(2026, 2, 24), date(2026, 2, 23)),    # Tue → Mon
    ('Monday', date(2026, 2, 23), date(2026, 2, 20)),      # Mon → Fri
    ('Saturday', date(2026, 2, 21), date(2026, 2, 20)),    # Sat → Fri
]
for name, input_date, expected in test_cases:
    result = get_previous_business_day(input_date)
    status = '✓' if result == expected else '✗'
    print(f'{status} {name}: {input_date} → {result} (expected {expected})')
"
```
Expected: All date calculations correct, especially Monday→Friday and Saturday→Friday.

### 1.4 Cache Collected Papers (Dev Support)

- [ ] After a successful fetch, auto-save papers to dev cache
- [ ] Downstream phases (ranker, analyzer) can load from cache during development

**✅ Checkpoint 1.4:**
```bash
# Verify papers are cached and loadable
python -c "
from src.collector import fetch_papers
from src.dev_cache import save_cache, load_cache

papers = fetch_papers()
save_cache('papers', [p.__dict__ for p in papers])
cached = load_cache('papers')
print(f'Cached {len(cached)} papers to .dev_cache/papers.json')
print('Downstream phases can now load from cache instead of re-fetching')
"
```
Expected: Papers saved to cache. Future phases can skip the arXiv fetch during development.

### 1.5 Unit Tests for Collector

- [ ] Write `tests/test_collector.py`
- [ ] Test Paper dataclass construction
- [ ] Test deduplication logic (mock data)
- [ ] Test date calculation logic
- [ ] Test error retry behavior (mock API failures)

**✅ Checkpoint 1.5:**
```bash
pytest tests/test_collector.py -v
```
Expected: All tests pass.

### 1.6 Commit

- [ ] Commit: "Add data collector: arXiv fetch, dedup, date logic"

---

## Phase 2 — PDF Extraction (No LLM Needed)

### 2.1 PDF Download

- [ ] Implement `pdf_extractor.py` with PDF download function
- [ ] Rate-limit downloads (3s between requests)
- [ ] Save to temp directory, clean up after extraction

**✅ Checkpoint 2.1 — Download a real paper:**
```bash
# Download a known paper and verify file
python -c "
from src.pdf_extractor import download_pdf
path = download_pdf('2602.17663')
print(f'Downloaded to: {path}')
import os
size = os.path.getsize(path)
print(f'File size: {size:,} bytes')
assert size > 10000, 'PDF too small — likely a download error'
print('Download OK')
os.unlink(path)
"
```
Expected: PDF downloads successfully, file size is reasonable (>10KB).

### 2.2 Text Extraction

- [ ] Implement text extraction from PDF using PyMuPDF
- [ ] Clean up extracted text (remove headers/footers, page numbers)
- [ ] Optionally strip references section

**✅ Checkpoint 2.2 — Extract and inspect text:**
```bash
# Extract text from a real paper and check quality
python -c "
from src.pdf_extractor import download_and_extract
text = download_and_extract('2602.17663')
print(f'Extracted text length: {len(text)} chars ({len(text.split())} words)')
print(f'Approximate tokens: {len(text) // 4}')
print()
print('--- First 500 chars ---')
print(text[:500])
print()
print('--- Last 500 chars ---')
print(text[-500:])
"
```
Expected: Clean readable text. Word count should be 4,000–15,000 for a typical paper. No garbled characters or repeated headers.

### 2.3 HTML Fallback

- [ ] Implement HTML fallback extraction (BeautifulSoup on arxiv HTML)
- [ ] Use HTML when PDF extraction fails or produces garbage

**✅ Checkpoint 2.3 — HTML fallback works:**
```bash
# Test HTML extraction for a paper known to have an HTML version
python -c "
from src.pdf_extractor import extract_from_html
text = extract_from_html('2602.17663')
if text:
    print(f'HTML text length: {len(text)} chars ({len(text.split())} words)')
    print('HTML fallback OK')
else:
    print('No HTML version available (expected for some papers)')
"
```
Expected: Either returns clean text, or gracefully reports no HTML version.

### 2.4 Token Safety Check

- [ ] Implement token count estimation (chars / 4 as rough heuristic)
- [ ] Add truncation logic for papers exceeding 80K tokens
- [ ] Log a warning when truncation occurs

**✅ Checkpoint 2.4:**
```bash
# Verify truncation logic with a mock long text
python -c "
from src.pdf_extractor import truncate_if_needed
short_text = 'word ' * 10000    # ~10K words ≈ ~13K tokens
long_text = 'word ' * 200000    # ~200K words ≈ ~267K tokens

short_result = truncate_if_needed(short_text)
long_result = truncate_if_needed(long_text)

print(f'Short text: {len(short_text.split())} words → {len(short_result.split())} words (should be same)')
print(f'Long text: {len(long_text.split())} words → {len(long_result.split())} words (should be truncated)')
assert len(short_result.split()) == len(short_text.split()), 'Short text should not be truncated'
assert len(long_result.split()) < len(long_text.split()), 'Long text should be truncated'
print('Truncation OK')
"
```
Expected: Short text passes through untouched. Long text is truncated.

### 2.5 Unit Tests for PDF Extractor

- [ ] Write `tests/test_pdf_extractor.py`
- [ ] Test text cleanup logic (mock PDF text with headers/footers)
- [ ] Test truncation logic
- [ ] Test fallback behavior (mock PDF failure → HTML attempt)

**✅ Checkpoint 2.5:**
```bash
pytest tests/test_pdf_extractor.py -v
```
Expected: All tests pass.

### 2.6 Commit

- [ ] Commit: "Add PDF extractor: download, text extraction, HTML fallback"

---

## Phase 3 — Stage 1 Ranker (First LLM Integration)

This is where we first connect to Claude. Critical to get right.

### 3.1 Prompt Builder

- [ ] Implement `ranker.py` with function to build the ranking prompt
- [ ] Load user profile from config
- [ ] Format all papers into the prompt template from ARCHITECTURE.md

**✅ Checkpoint 3.1 — Inspect the assembled prompt:**
```bash
# Build the prompt with real data (or cached papers) and inspect it
python -c "
from src.collector import fetch_papers
from src.dev_cache import load_cache, save_cache
from src.ranker import build_ranking_prompt

# Use cached papers if available, otherwise fetch live
cached = load_cache('papers')
if cached:
    from src.collector import Paper
    papers = [Paper(**p) for p in cached]
    print(f'Loaded {len(papers)} papers from cache')
else:
    papers = fetch_papers()
    save_cache('papers', [p.__dict__ for p in papers])
    print(f'Fetched {len(papers)} papers (saved to cache)')

system_prompt, user_prompt = build_ranking_prompt(papers)

print(f'System prompt: {len(system_prompt)} chars')
print(f'User prompt: {len(user_prompt)} chars')
print(f'Estimated tokens: ~{len(user_prompt) // 4}')
print()
print('--- System prompt (first 200 chars) ---')
print(system_prompt[:200])
print()
print('--- User prompt (first 500 chars) ---')
print(user_prompt[:500])
print()
print('--- User prompt (last 500 chars) ---')
print(user_prompt[-500:])
"
```
Expected: Well-formatted prompt with all papers listed. Token count should be ~50K-80K. Profile is embedded at the top.

### 3.2 LLM Call + Response Parsing

- [ ] Implement Claude API call for ranking
- [ ] Parse JSON response into structured `RankedResults`
- [ ] Handle malformed JSON (retry once with stricter prompt)
- [ ] Validate that response contains 10 papers with required fields

**✅ Checkpoint 3.2 — Live ranking with real papers (THE KEY TEST):**
```bash
# Run the ranker (load papers from cache to avoid redundant arXiv fetch)
python -c "
from src.collector import fetch_papers, Paper
from src.dev_cache import load_cache, save_cache
from src.ranker import rank_papers

# Load papers from cache
cached = load_cache('papers')
if cached:
    papers = [Paper(**p) for p in cached]
    print(f'Loaded {len(papers)} papers from cache')
else:
    papers = fetch_papers()
    save_cache('papers', [p.__dict__ for p in papers])
    print(f'Fetched {len(papers)} papers')

print('Sending to ranker...')
print()

results = rank_papers(papers)

# Cache ranking results so Phase 4 can use them without re-ranking
save_cache('ranked', results)
print(f'Ranked {results[\"total_papers_evaluated\"]} papers (saved to cache)')
print()

for paper in results['top_papers']:
    tier_icon = '📌' if paper['tier'] == 'deep_dive' else '📋'
    wildcard = ' 🃏 WILDCARD' if paper['is_wildcard'] else ''
    print(f'{tier_icon} #{paper[\"rank\"]}: {paper[\"title\"][:80]}...')
    print(f'   Justification: {paper[\"justification\"][:120]}...')
    print(f'   Tags: {paper[\"relevance_tags\"]}')
    print(f'   Source: {paper[\"source_match\"]}{wildcard}')
    print()
"
```
**⚡ THIS IS THE MOST IMPORTANT CHECKPOINT IN THE ENTIRE BUILD.**

Expected:
- 10 papers returned, ranked 1-10
- Top 5 tagged as `deep_dive`, #6-10 as `blurb`
- Justifications reference the user profile interests
- At least some papers match primary interests (agents, AI-hardware, etc.)
- Papers in the `deprioritize` list are absent

**Review the output carefully.** If the ranking doesn't match expectations:
- Adjust the prompt
- Tweak the user profile
- Re-run until we're satisfied with ranking quality

### 3.3 Unit Tests for Ranker

- [ ] Write `tests/test_ranker.py`
- [ ] Test prompt assembly (mock papers)
- [ ] Test JSON response parsing (mock LLM responses — valid and malformed)
- [ ] Test validation (missing fields, wrong number of papers)

**✅ Checkpoint 3.3:**
```bash
pytest tests/test_ranker.py -v
```
Expected: All tests pass. No real API calls in unit tests (all mocked).

### 3.4 Commit

- [ ] Commit: "Add Stage 1 ranker: prompt, Claude integration, JSON parsing"

---

## Phase 4 — Stage 2: Deep Analyzer + Blurbs (Heavy LLM Work)

### 4.1 Deep Analyzer — Single Paper

- [ ] Implement `analyzer.py` with the deep analysis prompt from ARCHITECTURE.md
- [ ] Take a single paper's full text + metadata and return a structured summary
- [ ] Validate summary length (1,000–2,000 words)

**✅ Checkpoint 4.1 — Analyze ONE real paper:**
```bash
# Pick the #1 ranked paper and generate a deep summary
# Uses cached papers + rankings from Phase 3 (no redundant API calls)
python -c "
from src.collector import Paper
from src.dev_cache import load_cache
from src.pdf_extractor import download_and_extract
from src.analyzer import analyze_paper

papers = [Paper(**p) for p in load_cache('papers')]
ranked = load_cache('ranked')

top_paper_id = ranked['top_papers'][0]['arxiv_id']
top_paper = next(p for p in papers if p.arxiv_id == top_paper_id)

print(f'Analyzing: {top_paper.title}')
print(f'Downloading PDF...')
full_text = download_and_extract(top_paper_id)
print(f'Extracted {len(full_text.split())} words from paper')
print()

summary = analyze_paper(
    paper=top_paper,
    full_text=full_text,
    rank_info=ranked['top_papers'][0]
)
word_count = len(summary.split())
print(f'Summary length: {word_count} words')
print()
print('=' * 60)
print(summary)
print('=' * 60)
"
```
**⚡ READ THE FULL SUMMARY.** This is the core product — the thing subscribers will read every day.

Expected:
- 1,000–2,000 words
- Six clear sections (So What → Core Idea → How It Works → Results → Limitations → Applications)
- Accessible to a non-specialist
- Specific — references actual results from the paper, not generic filler
- The "Applications & Opportunities" section has concrete, actionable ideas

**If the summary quality isn't right, iterate on the prompt before moving on.**

### 4.2 Deep Analyzer — All Top 5

- [ ] Implement loop to analyze all 5 deep-dive papers sequentially
- [ ] Handle individual paper failures (skip, promote #6)
- [ ] Log progress (paper 1/5, 2/5, etc.)

**✅ Checkpoint 4.2 — Full top-5 analysis:**
```bash
# Run deep analysis on all 5 top papers (loads cached papers + rankings)
python -c "
from src.collector import Paper
from src.dev_cache import load_cache, save_cache
from src.analyzer import analyze_top_papers

papers = [Paper(**p) for p in load_cache('papers')]
ranked = load_cache('ranked')

print('Analyzing top 5 papers (this will take 2-4 minutes)...')
print()

summaries = analyze_top_papers(papers, ranked)

# Cache summaries for email template iteration in Phase 5
save_cache('summaries', summaries)

for i, (paper_id, summary) in enumerate(summaries.items(), 1):
    word_count = len(summary.split())
    print(f'Paper {i}/5: {paper_id} — {word_count} words ✓')

total_words = sum(len(s.split()) for s in summaries.values())
print(f'\nTotal summary output: {total_words} words across 5 papers')
assert len(summaries) >= 4, f'Expected at least 4 summaries, got {len(summaries)}'
print('All deep analyses complete ✓ (saved to cache)')
"
```
Expected: 5 summaries generated (or 4 if one PDF failed, with a logged warning). Total output ~5,000–10,000 words. Each summary in the 1,000–2,000 word range.

### 4.3 Blurb Generator

- [ ] Implement `blurb_generator.py` with the blurb prompt from ARCHITECTURE.md
- [ ] Single LLM call for all 5 blurbs
- [ ] Parse JSON response, validate 100–150 words each

**✅ Checkpoint 4.3 — Generate blurbs:**
```bash
# Generate blurbs for papers #6-10 (loads cached papers + rankings)
python -c "
from src.collector import Paper
from src.dev_cache import load_cache, save_cache
from src.blurb_generator import generate_blurbs

papers = [Paper(**p) for p in load_cache('papers')]
ranked = load_cache('ranked')

blurbs = generate_blurbs(papers, ranked)

# Cache blurbs for email template iteration in Phase 5
save_cache('blurbs', blurbs)
print(f'Generated {len(blurbs)} blurbs (saved to cache)\n')

for blurb in blurbs:
    word_count = len(blurb['blurb'].split())
    print(f'#{blurb[\"rank\"]}: {blurb[\"arxiv_id\"]} — {word_count} words')
    print(f'   {blurb[\"blurb\"][:150]}...')
    print(f'   Read this if: {blurb[\"read_this_if\"]}')
    print()
"
```
Expected: 5 blurbs, each 100–150 words. Each has a "Read this if" tag. Concise and informative.

### 4.4 Unit Tests for Analyzer & Blurbs

- [ ] Write `tests/test_analyzer.py`
- [ ] Write `tests/test_blurb_generator.py`
- [ ] Test prompt assembly (mock data)
- [ ] Test response parsing (mock LLM responses)
- [ ] Test word count validation
- [ ] Test graceful failure handling (one paper fails → others still work)

**✅ Checkpoint 4.4:**
```bash
pytest tests/test_analyzer.py tests/test_blurb_generator.py -v
```
Expected: All tests pass.

### 4.5 Commit

- [ ] Commit: "Add Stage 2: deep analyzer, blurb generator, PDF extraction pipeline"

---

## Phase 5 — Email Composition & Delivery

### 5.1 Email Template

- [ ] Create `templates/digest_email.html` (Jinja2 template)
- [ ] Inline CSS styling (single-column, mobile-friendly)
- [ ] Sections: header, deep dives (1-5), quick reads (6-10), footer
- [ ] Include PDF and HTML links for every paper
- [ ] Include venue badges where applicable

**✅ Checkpoint 5.1 — Render template with mock data:**
```bash
# Render the email template with fake data and save to file for inspection
python -c "
from src.email_composer import compose_email

# Use mock data to test template rendering
html = compose_email(
    deep_summaries=[
        {
            'rank': i,
            'title': f'Test Paper Title #{i}',
            'authors': ['Author A', 'Author B', 'Author C'],
            'arxiv_id': f'2602.{17660+i}',
            'summary': f'This is a test summary for paper {i}. ' * 50,
            'venue': 'ICLR 2026' if i == 1 else None,
            'relevance_tags': ['LLM agents', 'AI reasoning'],
            'is_wildcard': i == 4,
        }
        for i in range(1, 6)
    ],
    blurbs=[
        {
            'rank': i,
            'title': f'Blurb Paper Title #{i}',
            'authors': ['Author X', 'Author Y'],
            'arxiv_id': f'2602.{17670+i}',
            'blurb': f'This paper proposes a novel approach to problem {i}. ' * 10,
            'read_this_if': 'you care about multi-agent coordination',
        }
        for i in range(6, 11)
    ],
    stats={'total_papers': 187, 'date': '2026-02-20', 'profile_name': 'Andres — AI Research Digest'},
)

with open('/tmp/test_digest.html', 'w') as f:
    f.write(html)
print(f'Email rendered: {len(html)} chars')
print('Saved to /tmp/test_digest.html — open in a browser to inspect')
"
```
**⚡ Open `/tmp/test_digest.html` in a browser.** Check:
- Layout renders correctly
- Links work
- Sections are clearly separated
- Mobile-friendly (resize browser window to test)
- No broken formatting or missing data

### 5.2 Email Sender

- [ ] Implement `email_sender.py` with Gmail SMTP logic
- [ ] Support sending to multiple subscribers
- [ ] Log delivery status per recipient

**✅ Checkpoint 5.2 — Send a real test email:**
```bash
# Send a test email to yourself
python -c "
from src.email_sender import send_email

html = '<h1>ArXiv Digest Test</h1><p>If you can read this, email sending works.</p>'

send_email(
    subject='🧪 ArXiv Digest — Test Email',
    html_body=html,
    recipients=[{'email': 'YOUR_EMAIL_HERE', 'name': 'Test'}]
)
print('Test email sent — check your inbox (and spam folder)')
"
```
**⚡ Check your inbox.** Verify:
- Email arrives
- Subject line renders correctly (with emoji)
- HTML content displays properly
- Sender address is correct
- Not in spam folder

### 5.3 Error Email

- [ ] Implement `error_handler.py` with error email formatting
- [ ] Include stage name, error message, truncated traceback, timestamp

**✅ Checkpoint 5.3 — Trigger an error email:**
```bash
# Simulate a pipeline failure and send error notification
python -c "
from src.error_handler import send_error_email

try:
    raise ValueError('Simulated failure in Stage 1: Ranker — LLM returned invalid JSON')
except Exception as e:
    send_error_email(
        stage='Stage 1: Ranker',
        error=e,
        context={'papers_fetched': 187, 'date': '2026-02-20'}
    )
    print('Error email sent — check your inbox')
"
```
Expected: Error email arrives with clear description of what went wrong, which stage, and when.

### 5.4 Quiet Day Email

- [ ] Implement quiet day notice email (no papers found)

**✅ Checkpoint 5.4:**
```bash
# Send a quiet day notice
python -c "
from src.email_composer import compose_quiet_day_email
from src.email_sender import send_email

html = compose_quiet_day_email(date='2026-02-20')
send_email(
    subject='📰 ArXiv AI Digest — Quiet Day (2026-02-20)',
    html_body=html,
    recipients=[{'email': 'YOUR_EMAIL_HERE', 'name': 'Test'}]
)
print('Quiet day email sent')
"
```
Expected: Clean, brief email confirming the system is running but no papers were published.

### 5.5 Unit Tests for Email

- [ ] Write `tests/test_email.py`
- [ ] Test template rendering with various data shapes (0 blurbs, missing venue, long titles)
- [ ] Test SMTP connection logic (mocked)
- [ ] Test error email formatting

**✅ Checkpoint 5.5:**
```bash
pytest tests/test_email.py -v
```
Expected: All tests pass.

### 5.6 Commit

- [ ] Commit: "Add email system: HTML template, SMTP sender, error/quiet day emails"

---

## Phase 6 — Pipeline Orchestration (Wiring It All Together)

### 6.1 Main Orchestrator

- [ ] Implement `main.py` that calls each stage in sequence
- [ ] Wrap entire pipeline in try/except for error handling
- [ ] Add logging at each stage transition
- [ ] Support `DRY_RUN` mode (runs everything but doesn't send email — prints to console instead)

**Note:** `main.py` does NOT use the dev cache. It runs the full pipeline live every time — collector → ranker → analyzer → email. The dev cache is only for interactive development in earlier phases.

**✅ Checkpoint 6.1 — Dry run the full pipeline:**
```bash
# Run the entire pipeline end-to-end in dry run mode
DRY_RUN=true python src/main.py
```
**⚡ THIS IS THE FULL INTEGRATION TEST.**

Expected output (approximately):
```
[INFO] Starting ArXiv AI Digest pipeline
[INFO] Date: 2026-02-20 (fetching papers from 2026-02-19)
[INFO] ─── Stage: Data Collector ───
[INFO] Fetched 187 papers from cs.AI
[INFO] After dedup: 185 unique papers
[INFO] ─── Stage 1: Ranker ───
[INFO] Sending 185 papers to Claude for ranking...
[INFO] Ranked top 10 papers (5 deep dive, 5 blurb)
[INFO] #1: [Paper Title] — LLM agents
[INFO] #2: [Paper Title] — AI-hardware
[INFO] ...
[INFO] ─── Stage 2: PDF Download & Extraction ───
[INFO] Downloading paper 1/5: 2602.17560...
[INFO] Extracted 8,234 words
[INFO] Downloading paper 2/5: 2602.17612...
[INFO] ...
[INFO] ─── Stage 2: Deep Analysis ───
[INFO] Analyzing paper 1/5: 2602.17560...
[INFO] Summary: 1,847 words ✓
[INFO] Analyzing paper 2/5: 2602.17612...
[INFO] ...
[INFO] ─── Stage 2: Blurb Generation ───
[INFO] Generating blurbs for papers #6-10...
[INFO] 5 blurbs generated ✓
[INFO] ─── Email Composition ───
[INFO] Email composed: 45,230 chars
[INFO] ─── DRY RUN — Email not sent ───
[INFO] Would send to: andres@example.com
[INFO] Subject: 📰 ArXiv AI Digest — 2026-02-20
[INFO] Pipeline complete in 4m 23s
```

Verify:
- Every stage completes without errors
- Paper counts are reasonable
- Summaries are the right length
- Total runtime is under 15 minutes

### 6.2 Live Run (Real Email)

- [ ] Run the pipeline without DRY_RUN
- [ ] Send the actual digest email to yourself

**✅ Checkpoint 6.2 — The real thing:**
```bash
# Run for real — this will send an actual email
python src/main.py
```
**⚡ Check your inbox. Read the entire digest.** This is the moment of truth.

Evaluate:
- [ ] Email arrives promptly
- [ ] Formatting looks good on desktop
- [ ] Formatting looks good on mobile (check on phone)
- [ ] Top 5 summaries are genuinely useful and accessible
- [ ] Blurbs are concise and informative
- [ ] All PDF/HTML links work (click each one)
- [ ] Venue badges show where applicable
- [ ] Stats footer is accurate
- [ ] Overall: would you want to receive this every morning?

**If anything isn't right, iterate before moving to Phase 7.**

### 6.3 Commit

- [ ] Commit: "Add pipeline orchestrator with DRY_RUN support"

---

## Phase 7 — GitHub Actions Deployment

### 7.1 Create GitHub Repository

- [ ] Create a new repository on GitHub (public or private)
- [ ] Add the remote: `git remote add origin git@github.com:<user>/arxiv-digest.git`
- [ ] Push all existing commits: `git push -u origin main`

**✅ Checkpoint 7.1:**
```bash
git remote -v
git log --oneline
# Verify all commits from Phases 0-6 are present
# Verify remote points to your GitHub repo
```

### 7.2 Workflow File

- [ ] Create `.github/workflows/daily_digest.yml` per ARCHITECTURE.md
- [ ] Configure cron schedule: `0 11 * * 2-6` (6 AM ET, Tue-Sat)
- [ ] Add `workflow_dispatch` for manual trigger
- [ ] Set 15-minute timeout
- [ ] Commit: "Add GitHub Actions workflow for daily digest"

**✅ Checkpoint 7.2 — Validate workflow syntax:**
```bash
# Verify YAML is valid (if you have actionlint installed)
# Otherwise, just push and check GitHub's validation
cat .github/workflows/daily_digest.yml
```

### 7.3 GitHub Secrets

- [ ] Add `ANTHROPIC_API_KEY` to repository secrets
- [ ] Add `GMAIL_ADDRESS` to repository secrets
- [ ] Add `GMAIL_APP_PASSWORD` to repository secrets

**✅ Checkpoint 7.3:**
Navigate to GitHub → Repository → Settings → Secrets and variables → Actions.
Verify all 3 secrets are listed (values are hidden, just confirm they exist).

### 7.4 Manual Trigger Test

- [ ] Push workflow file to GitHub
- [ ] Trigger the workflow manually via GitHub UI (Actions → Run workflow)
- [ ] Monitor the run in real time

**✅ Checkpoint 7.4 — GitHub Actions works:**
- [ ] Go to Actions tab → click the workflow run
- [ ] Watch each step complete (Install deps → Run pipeline)
- [ ] Verify email arrives in your inbox
- [ ] Check logs for any warnings or errors
- [ ] Confirm total runtime is under 15 minutes

### 7.5 Cron Verification

- [ ] Leave the cron running overnight
- [ ] Next morning: verify the digest arrived at ~6 AM ET

**✅ Checkpoint 7.5 — Automated delivery:**
- [ ] Email arrived without manual intervention
- [ ] Content is correct for the previous day's papers
- [ ] No errors in GitHub Actions logs

---

## Phase 8 — Hardening & Polish

### 8.1 Error Scenario Testing

Deliberately break things to verify error handling works:

- [ ] **Test: Invalid API key** — Set a bad `ANTHROPIC_API_KEY` → verify error email arrives
- [ ] **Test: arXiv down** — Mock a network failure → verify retry + error email
- [ ] **Test: PDF download fails** — Mock one PDF failure → verify #6 gets promoted and digest still sends
- [ ] **Test: LLM returns garbage** — Mock invalid JSON → verify retry logic works

**✅ Checkpoint 8.1:**
Each error scenario results in either graceful degradation (digest still sends) or a clear error notification email. No silent failures.

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
| 1 | **arXiv API investigation** + live test + unit tests | Yes (arXiv only) | Validate data source, verify collection works |
| 2 | Live PDF download + unit tests | Yes (arXiv only) | Verify extraction works |
| 3 | **Live ranking** + unit tests | **Yes (Claude)** — cached papers | Verify ranking quality |
| 4 | **Live analysis** + unit tests | **Yes (Claude)** — cached rankings | Verify summary quality |
| 5 | Live email send + unit tests | Yes (Gmail) — cached summaries | Verify delivery works |
| 6 | **Full pipeline dry run + live run** | **Yes (all, live)** | End-to-end integration |
| 7 | GitHub Actions run | Yes (all, live) | Verify CI/CD works |
| 8 | Deliberate failure tests | Mixed | Verify error handling |

### Unit Tests (Mocked — Fast, Free)

Every module has unit tests that mock external dependencies:

| Test File | Mocks | Tests |
|-----------|-------|-------|
| `test_collector.py` | arXiv API | Paper parsing, dedup, date logic |
| `test_pdf_extractor.py` | PDF files | Text cleanup, truncation, fallback |
| `test_ranker.py` | Claude API | Prompt building, JSON parsing, validation |
| `test_analyzer.py` | Claude API | Prompt building, summary parsing, word count |
| `test_blurb_generator.py` | Claude API | Prompt building, JSON parsing, word count |
| `test_email.py` | SMTP | Template rendering, email construction |

### Live Tests (Real APIs — Manual, Costs Money)

The checkpoint scripts above serve as live tests. They hit real APIs and cost real money (small amounts). Run them during development, not in CI.

---

## Estimated Timeline

| Phase | Description | Estimated Time | Cumulative |
|-------|-------------|---------------|------------|
| **Phase 0** | Project setup | ~30 min | 30 min |
| **Phase 1** | Data Collector | ~1.5 hours | 2 hours |
| **Phase 2** | PDF Extraction | ~1.5 hours | 3.5 hours |
| **Phase 3** | Stage 1 Ranker | ~2 hours (prompt iteration) | 5.5 hours |
| **Phase 4** | Deep Analyzer + Blurbs | ~2 hours (prompt iteration) | 7.5 hours |
| **Phase 5** | Email Composition & Delivery | ~1.5 hours | 9 hours |
| **Phase 6** | Pipeline Orchestration | ~1 hour | 10 hours |
| **Phase 7** | GitHub Actions Deployment | ~30 min | 10.5 hours |
| **Phase 8** | Hardening & Polish | ~1.5 hours | 12 hours |

**Total estimated effort: ~12 hours**

Phases 3 and 4 are the most variable — prompt iteration can take longer if the output quality needs tuning. Budget extra time there.

---

## Contingency Notes

### If Running Behind

| Shortcut | Impact | When to Use |
|----------|--------|-------------|
| Skip HTML fallback (Phase 2.3) | Rare papers won't extract | If no PDF failures encountered in testing |
| Simplify email template (Phase 5.1) | Less polished look | If formatting is taking too long; function over form |
| Skip error scenario testing (Phase 8.1) | Less robust | If pipeline is working reliably in testing |

### If Running Ahead

| Extension | Benefit | Effort |
|-----------|---------|--------|
| Add paper history file (avoid re-ranking across days) | Prevents cross-day duplicates | ~1 hour |
| Add weekly digest mode (aggregate Mon-Fri) | Weekend catchup email | ~2 hours |
| Polish email CSS further | Better visual design | ~1 hour |

---

## Definition of Done

The project is **complete** when:

- [ ] Pipeline runs daily via GitHub Actions without manual intervention
- [ ] Digest email arrives by ~6:15 AM ET with correct content
- [ ] Top 5 summaries are 1,000–2,000 words, accessible, and accurate
- [ ] Blurbs are 100–150 words with "Read this if" tags
- [ ] All PDF/HTML links work
- [ ] Error emails arrive when the pipeline fails
- [ ] Quiet day notices arrive on holidays
- [ ] Pipeline completes in under 15 minutes
- [ ] Daily cost is under $1.00
- [ ] All unit tests pass
- [ ] Code passes ruff linting
- [ ] Ran successfully for 3+ consecutive days unattended
