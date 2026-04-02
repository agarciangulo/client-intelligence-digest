# Tech Stack — Client Intelligence Digest Agent

This document catalogs all technologies, libraries, and services required for the Client Intelligence Digest Agent. It complements `ARCHITECTURE.md` by specifying the exact tools behind each component.

---

## 1. Architecture → Tech Mapping

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              TECH STACK OVERVIEW                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  Scheduling         Pipeline            LLM                Email                │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐            │
│  │ GitHub   │─────▶│ Python   │─────▶│ Anthropic│─────▶│ Gmail    │            │
│  │ Actions  │      │ 3.12     │      │ Claude   │      │ SMTP     │            │
│  │ (cron)   │      │          │      │ Sonnet   │      │          │            │
│  └──────────┘      └──────────┘      └──────────┘      └──────────┘            │
│                          │                                                      │
│                    Dependencies                                                 │
│       ┌──────────┬──────────┬──────────┬──────────┐                            │
│       │feedparser│ requests │anthropic │ Jinja2   │                            │
│       │ (RSS)    │ + BS4    │ (LLM)   │ (email)  │                            │
│       └──────────┘──────────┘──────────┘──────────┘                            │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Runtime & Language

| Component | Version | Purpose |
|-----------|---------|---------|
| **Python** | 3.12+ | Primary language for all pipeline components |
| **pip** | Latest | Dependency management via `requirements.txt` |

### Why Python 3.12?

- GitHub Actions ships with 3.12 by default — zero setup friction
- f-strings, dataclasses, type hints, `|` union syntax all available
- Every dependency we need has mature Python support
- Consistent with the ArXiv digest system (shared knowledge base)

---

## 3. Core Dependencies

### 3.1 RSS Feed Parsing

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`feedparser`** | Parse RSS and Atom feeds from configured sources | De facto standard for feed parsing in Python. Handles RSS 2.0, Atom, RDF. Normalizes dates, handles encoding quirks, and is tolerant of malformed feeds. |

**What it gives us:**
- Automatic detection of feed format (RSS, Atom, etc.)
- Normalized entry structure (title, link, summary, published date)
- Tolerant of malformed XML (common in real-world feeds)
- Date parsing across multiple formats

### 3.2 HTML Scraping

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`requests`** | Fetch HTML pages from configured sources | De facto standard for Python HTTP. Simple API, connection pooling, retries. |
| **`beautifulsoup4`** | Parse HTML and extract article content using CSS selectors | Industry standard for HTML parsing. Tolerant of messy HTML, excellent CSS selector support. |

**What it gives us:**
- Fetch any public webpage
- Extract article content using per-source CSS selectors (title, body, date, links)
- Handle malformed HTML gracefully
- Simple, readable extraction code

**Alternative considered:** `Playwright` / `Selenium` for JavaScript-rendered pages. These are heavier (require a browser binary) and slower. Deferred unless specific sources require JS rendering.

### 3.3 LLM Integration

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`anthropic`** | Claude API client | Official SDK. Clean interface, handles retries, streaming, and structured output. |

**Model Selection:**

| Stage | Model | Rationale |
|-------|-------|-----------|
| Per-client Summarizer | Claude Sonnet | Good balance of quality and cost for narrative summaries |
| Highlight Selector | Claude Sonnet | Needs judgment to pick the most significant mentions |

**Why Sonnet for both?**
- Weekly cadence means fewer total runs than the daily ArXiv digest
- Cost difference between Sonnet and a cheaper model is negligible at weekly frequency
- Consistency across stages simplifies the system
- Quality matters — the account manager relies on these summaries for client interactions

### 3.4 Email Composition

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`Jinja2`** | HTML email template rendering | Industry standard for Python templating. Clean syntax, inheritance, filters. |

**What it gives us:**
- Separate HTML template file from Python logic
- Loops for rendering per-client sections and project lists
- Conditional rendering (highlights present or not, quiet week notice)
- Filters for formatting dates, truncating text

### 3.5 Email Delivery

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`smtplib`** (stdlib) | SMTP connection to Gmail | Built into Python — zero dependencies |
| **`email.mime`** (stdlib) | Construct MIME email messages | Built into Python — handles HTML content type, headers, encoding |

No third-party library needed for email sending.

### 3.6 Configuration & Environment

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`python-dotenv`** | Load `.env` file for local development | Standard approach; GitHub Actions uses native secrets instead |

---

## 4. Development & Testing Dependencies

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`pytest`** | Unit and integration testing | Python testing standard; clean fixtures, parametrize, assertions |
| **`pytest-mock`** | Mock LLM calls, SMTP, and HTTP requests in tests | Avoids real API calls during testing |
| **`ruff`** | Linting and formatting | Replaces flake8 + black + isort in a single tool. Fast (written in Rust). |

---

## 5. Complete `requirements.txt`

```
# Core — RSS feed parsing
feedparser>=6.0.0

# Core — HTML scraping
requests>=2.31.0
beautifulsoup4>=4.12.0

# Core — LLM integration
anthropic>=0.40.0

# Core — Email templating
Jinja2>=3.1.0

# Config — Environment variable loading
python-dotenv>=1.0.0

# Dev — Testing
pytest>=8.0.0
pytest-mock>=3.14.0

# Dev — Linting & formatting
ruff>=0.8.0
```

**Total direct dependencies: 9** (6 runtime + 3 dev-only)

No heavy frameworks, no browser automation, no databases. Minimal footprint.

---

## 6. Standard Library Usage (No Install Needed)

| Module | Purpose | Used In |
|--------|---------|---------|
| `smtplib` | SMTP connection and email sending | `email_sender.py` |
| `email.mime.text` | Construct HTML email body | `email_sender.py` |
| `email.mime.multipart` | Construct multi-part email message | `email_sender.py` |
| `json` | Parse/write config files, state file, and LLM responses | Multiple modules |
| `os` | Environment variable access | `main.py`, config loading |
| `pathlib` | File path handling | Multiple modules |
| `logging` | Structured logging throughout pipeline | All modules |
| `datetime` | Date range calculations for weekly window | `collector.py`, `main.py` |
| `dataclasses` | Article, Client, Match data models | Multiple modules |
| `re` | Keyword matching with word boundaries | `matcher.py` |
| `hashlib` | URL hashing for state deduplication | `state_manager.py` |
| `traceback` | Format error tracebacks for error emails | `error_handler.py` |
| `time` | Rate limiting sleeps between requests | `scraper.py` |

---

## 7. External Services

| Service | Free Tier | Purpose | Credentials Needed |
|---------|-----------|---------|-------------------|
| **GitHub Actions** | 2,000 min/month (private repos) | Weekly cron scheduling + pipeline execution | None (built into repo) |
| **Anthropic API** | Pay-as-you-go | Claude Sonnet for summarization and highlight selection | `ANTHROPIC_API_KEY` |
| **Gmail SMTP** | Free (500 emails/day limit) | Email delivery | `GMAIL_ADDRESS` + `GMAIL_APP_PASSWORD` |

### Monthly Cost Estimate

| Service | Usage | Cost |
|---------|-------|------|
| GitHub Actions | ~10 min/week × 4 weeks = ~40 min | $0 (within free tier) |
| Anthropic API | ~$0.50–1.50/week × 4 weeks | ~$2–6/month |
| Gmail SMTP | ~1 email/week | $0 |
| **Total** | | **~$2–6/month** |

LLM cost varies with the number of clients that have matches in a given week. More clients with more mentions = more summarization calls.

---

## 8. Environment Variables

### Required (Must Be Set)

| Variable | Example Value | Purpose | Where It's Set |
|----------|---------------|---------|----------------|
| `ANTHROPIC_API_KEY` | `sk-ant-api03-...` | Authenticates with Claude API | GitHub Secrets |
| `GMAIL_ADDRESS` | `mydigest@gmail.com` | Sender email for SMTP | GitHub Secrets |
| `GMAIL_APP_PASSWORD` | `xxxx xxxx xxxx xxxx` | Gmail App Password (not regular password) | GitHub Secrets |

### Optional (Have Defaults)

| Variable | Default | Purpose |
|----------|---------|---------|
| `LOG_LEVEL` | `INFO` | Logging verbosity (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |
| `DIGEST_TIMEZONE` | `US/Eastern` | Timezone for date calculations and display |
| `LOOKBACK_DAYS` | `7` | Number of days to look back when scraping sources |
| `DRY_RUN` | `false` | If `true`, run pipeline but don't send email or update state |

### `.env.example`

```bash
# Required — API Keys & Credentials
ANTHROPIC_API_KEY=sk-ant-api03-your-key-here
GMAIL_ADDRESS=your-email@gmail.com
GMAIL_APP_PASSWORD=your-app-password-here

# Optional — Defaults shown
LOG_LEVEL=INFO
DIGEST_TIMEZONE=US/Eastern
LOOKBACK_DAYS=7
DRY_RUN=false
```

---

## 9. Gmail App Password Setup

Since Gmail blocks "less secure apps" by default, the sender account needs an App Password. One-time setup:

1. Go to [Google Account → Security](https://myaccount.google.com/security)
2. Enable **2-Step Verification** (required for App Passwords)
3. Go to [App Passwords](https://myaccount.google.com/apppasswords)
4. Select **Mail** as the app
5. Generate — Google gives you a 16-character password
6. Store this as `GMAIL_APP_PASSWORD` in GitHub Secrets

**Important:** This is NOT your regular Gmail password. It's a separate credential that only works for SMTP. If compromised, revoke it without affecting your main account.

---

## 10. GitHub Actions Runtime Environment

| Aspect | Detail |
|--------|--------|
| **OS** | Ubuntu 22.04 (latest) |
| **Python** | 3.12 (via `setup-python` action) |
| **Memory** | 7 GB RAM |
| **Disk** | 14 GB SSD |
| **Network** | Full internet access (needed for source scraping, Anthropic, Gmail) |
| **Timeout** | 15 minutes (set in workflow) |
| **Secrets** | Injected as environment variables, masked in logs |
| **Git Access** | Needs write access to push state file updates back to repo |

---

## 11. Dependency Security

| Practice | Implementation |
|----------|----------------|
| **Pinned minimum versions** | `requirements.txt` uses `>=` with known-good minimums |
| **No unnecessary dependencies** | 6 runtime deps — each one earns its place |
| **No secrets in code** | All credentials via environment variables / GitHub Secrets |
| **`.gitignore`** covers | `.env`, `__pycache__`, `.pytest_cache` |

### `.gitignore`

```
# Environment
.env
.venv/
venv/

# Python
__pycache__/
*.pyc
*.pyo
.pytest_cache/

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# Dev cache (if used during development)
.dev_cache/
```

---

## 12. Tech Decision Log

| Decision | Chosen | Over | Why |
|----------|--------|------|-----|
| Language | Python 3.12 | Node.js, Go | Richest scraping/AI ecosystem; every dependency is Python-native |
| RSS parsing | feedparser | Raw XML parsing, scrapy | Cleanest API; handles edge cases and format variations |
| HTML scraping | requests + BeautifulSoup | Playwright, Selenium, scrapy | Lightweight; no browser binary needed; sufficient for static pages |
| LLM provider | Anthropic Claude | OpenAI GPT-4o | Consistent with existing ArXiv digest; good summarization quality |
| LLM model | Claude Sonnet (all stages) | Mixed models | Cost difference negligible at weekly frequency; simplicity wins |
| Email sending | Gmail SMTP (stdlib) | SendGrid, Resend, AWS SES | Zero cost, zero setup, sufficient for small subscriber list |
| Email templating | Jinja2 | f-strings, mako | Industry standard; clean separation of template and code |
| Scheduling | GitHub Actions cron | AWS Lambda, local cron, Render | Free; zero infrastructure; built-in secrets management |
| State storage | JSON file in repo | SQLite, PostgreSQL, Redis | Zero infrastructure; git-native; sufficient for weekly URL tracking |
| Keyword matching | Python `re` module | spaCy, LLM-based | Fast, deterministic, free; good enough for known client/project names |
| Testing | pytest | unittest | Better assertions, fixtures, and parametrize support |
| Linting | ruff | black + flake8 | Single tool; faster; modern standard |

---

## 13. Summary

| Layer | Technology | Count |
|-------|-----------|-------|
| **Runtime** | Python 3.12 on GitHub Actions (Ubuntu) | — |
| **Runtime deps** | feedparser, requests, beautifulsoup4, anthropic, Jinja2, python-dotenv | 6 |
| **Dev deps** | pytest, pytest-mock, ruff | 3 |
| **Stdlib modules** | smtplib, email, json, os, pathlib, logging, datetime, dataclasses, re, hashlib, traceback, time | 12 |
| **External services** | GitHub Actions, Anthropic API, Gmail SMTP | 3 |
| **Monthly cost** | ~$2–6 (mostly Anthropic API) | — |
| **Credentials** | 3 secrets (Anthropic key, Gmail address, Gmail App Password) | 3 |

Minimal, focused, and every piece earns its place.
