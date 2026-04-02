# Tech Stack — ArXiv AI Paper Digest Agent

This document catalogs all technologies, libraries, and services required for the ArXiv AI Paper Digest Agent. It complements `ARCHITECTURE.md` by specifying the exact tools behind each component.

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
│  │ (cron)   │      │          │      │ 3.5      │      │          │            │
│  └──────────┘      └──────────┘      │ Sonnet   │      └──────────┘            │
│                          │           └──────────┘                               │
│                          │                                                      │
│                    Dependencies                                                 │
│       ┌──────────┬──────────┬──────────┬──────────┐                            │
│       │ arxiv    │ PyMuPDF  │anthropic │ Jinja2   │                            │
│       │ (API)    │ (PDF)    │ (LLM)   │ (email)  │                            │
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

---

## 3. Core Dependencies

### 3.1 arXiv Data Collection

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`arxiv`** | Fetch papers from the arXiv API | Official-ish Python wrapper; handles pagination, rate limiting, and metadata parsing out of the box |

**What it gives us:**
- Search by category (`cs.AI`) and date range
- Returns structured results: title, authors, abstract, comments, categories, PDF links
- Built-in rate limiting (respects arXiv's 3-second guideline)
- Handles pagination for large result sets (>200 papers)

**Alternative considered:** `feedparser` on the Atom feed — lighter, but requires manual date filtering and metadata parsing. Not worth the effort when `arxiv` does it cleanly.

### 3.2 PDF Processing

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`PyMuPDF`** (imported as `fitz`) | Extract text from downloaded PDFs | Fast, reliable, no external dependencies (pure C library). Best text extraction quality among Python PDF libraries. |

**What it gives us:**
- Page-by-page text extraction
- Handles multi-column layouts (common in academic papers)
- Works on all arXiv PDFs (LaTeX-generated)
- No Java/Poppler dependency (unlike `pdfplumber` or `tabula`)

**Alternative considered:** `pdfplumber` — good but slower and depends on `pdfminer.six`. `PyMuPDF` is faster and handles academic paper layouts better.

**Fallback strategy:** If a PDF is corrupted or extraction fails, we try the arXiv HTML version of the paper (`https://arxiv.org/html/{id}`). HTML extraction uses:

| Library | Purpose |
|---------|---------|
| **`requests`** | Download the HTML page |
| **`beautifulsoup4`** | Parse HTML and extract article text |

### 3.3 LLM Integration

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`anthropic`** | Claude API client | Official SDK. Clean interface, handles retries, streaming, and structured output. |

**Model Selection:**

| Stage | Model | Rationale |
|-------|-------|-----------|
| Stage 1: Ranker | Claude 3.5 Sonnet | Best judgment for ranking; large context window handles all papers in one call |
| Stage 2: Deep Analyzer | Claude 3.5 Sonnet | High-quality long-form writing; accurate summarization of full papers |
| Stage 2: Blurb Generator | Claude 3.5 Sonnet | Consistency with other stages; cost difference vs. Haiku is negligible at 1 call/day |

**Why all Sonnet?**
- Total LLM cost is ~$0.30–0.60/day — the savings from using Haiku for 1-2 calls would be ~$0.05. Not worth the quality tradeoff or the complexity of managing multiple model configs.
- We can revisit if costs rise (e.g., adding more categories or subscribers with different profiles).

### 3.4 Email Composition

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`Jinja2`** | HTML email template rendering | Industry standard for Python templating. Clean syntax, inheritance, filters. |

**What it gives us:**
- Separate HTML template file from Python logic
- Loops for rendering paper lists
- Filters for formatting dates, truncating text
- Easy to modify email design without touching code

### 3.5 Email Delivery

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`smtplib`** (stdlib) | SMTP connection to Gmail | Built into Python — zero dependencies |
| **`email.mime`** (stdlib) | Construct MIME email messages | Built into Python — handles HTML content type, headers, encoding |

No third-party library needed for email sending. Python's standard library handles SMTP + MIME natively.

### 3.6 HTTP & Networking

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`requests`** | HTTP calls (PDF downloads, HTML fallback) | De facto standard for Python HTTP. Simple API, connection pooling, retries. |

**Note:** The `arxiv` library handles its own HTTP internally. `requests` is only needed for:
1. Downloading PDF files
2. Fetching HTML fallback pages

### 3.7 Configuration & Environment

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`python-dotenv`** | Load `.env` file for local development | Standard approach; GitHub Actions uses native secrets instead |

---

## 4. Development & Testing Dependencies

| Library | Purpose | Why This One |
|---------|---------|--------------|
| **`pytest`** | Unit and integration testing | Python testing standard; clean fixtures, parametrize, assertions |
| **`pytest-mock`** | Mock LLM calls and SMTP in tests | Avoids real API calls during testing |
| **`ruff`** | Linting and formatting | Replaces flake8 + black + isort in a single tool. Fast (written in Rust). |

### Why Ruff Over Black + Flake8?

- Single tool instead of three
- 10-100x faster than black/flake8
- Handles linting AND formatting
- Growing community standard for new Python projects

---

## 5. Complete `requirements.txt`

```
# Core — arXiv data collection
arxiv>=2.1.0

# Core — PDF text extraction
PyMuPDF>=1.24.0

# Core — HTML fallback parsing
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

No heavy frameworks, no ORMs, no web servers. Minimal footprint.

---

## 6. Standard Library Usage (No Install Needed)

These Python standard library modules are used directly — no `pip install` required:

| Module | Purpose | Used In |
|--------|---------|---------|
| `smtplib` | SMTP connection and email sending | `email_sender.py` |
| `email.mime.text` | Construct HTML email body | `email_sender.py` |
| `email.mime.multipart` | Construct multi-part email message | `email_sender.py` |
| `json` | Parse/write config files and LLM responses | Multiple modules |
| `os` | Environment variable access | `main.py`, config loading |
| `pathlib` | File path handling | Multiple modules |
| `logging` | Structured logging throughout pipeline | All modules |
| `datetime` | Date calculation (previous business day) | `collector.py`, `main.py` |
| `dataclasses` | Paper and result data models | Multiple modules |
| `traceback` | Format error tracebacks for error emails | `error_handler.py` |
| `time` | Rate limiting sleeps | `collector.py`, `pdf_extractor.py` |
| `tempfile` | Temporary storage for downloaded PDFs | `pdf_extractor.py` |

---

## 7. External Services

| Service | Free Tier | Purpose | Credentials Needed |
|---------|-----------|---------|-------------------|
| **GitHub Actions** | 2,000 min/month (private repos) | Daily cron scheduling + pipeline execution | None (built into repo) |
| **Anthropic API** | Pay-as-you-go (~$0.30–0.60/day) | Claude 3.5 Sonnet for ranking and summarization | `ANTHROPIC_API_KEY` |
| **Gmail SMTP** | Free (500 emails/day limit) | Email delivery | `GMAIL_ADDRESS` + `GMAIL_APP_PASSWORD` |
| **arXiv API** | Free (rate-limited) | Paper metadata and PDFs | None |

### Monthly Cost Estimate

| Service | Usage | Cost |
|---------|-------|------|
| GitHub Actions | ~7 min/day × 22 days = ~154 min | $0 (within free tier) |
| Anthropic API | ~$0.45/day × 22 days | ~$10/month |
| Gmail SMTP | ~1 email/day | $0 |
| **Total** | | **~$10/month** |

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
| `ARXIV_CATEGORY` | `cs.AI` | arXiv category to scan |
| `DRY_RUN` | `false` | If `true`, run pipeline but don't send email (useful for testing) |

### `.env.example`

```bash
# Required — API Keys & Credentials
ANTHROPIC_API_KEY=sk-ant-api03-your-key-here
GMAIL_ADDRESS=your-email@gmail.com
GMAIL_APP_PASSWORD=your-app-password-here

# Optional — Defaults shown
LOG_LEVEL=INFO
DIGEST_TIMEZONE=US/Eastern
ARXIV_CATEGORY=cs.AI
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

The pipeline runs on GitHub's Ubuntu runners. Here's what the execution environment looks like:

| Aspect | Detail |
|--------|--------|
| **OS** | Ubuntu 22.04 (latest) |
| **Python** | 3.12 (via `setup-python` action) |
| **Memory** | 7 GB RAM |
| **Disk** | 14 GB SSD |
| **Network** | Full internet access (needed for arXiv, Anthropic, Gmail) |
| **Timeout** | 15 minutes (set in workflow) |
| **Secrets** | Injected as environment variables, masked in logs |

**No persistent storage between runs.** Each run starts fresh — downloads dependencies, runs pipeline, exits. This is by design (stateless architecture).

---

## 11. Dependency Security

| Practice | Implementation |
|----------|----------------|
| **Pinned minimum versions** | `requirements.txt` uses `>=` with known-good minimums |
| **No unnecessary dependencies** | 6 runtime deps — each one earns its place |
| **No secrets in code** | All credentials via environment variables / GitHub Secrets |
| **`.gitignore`** covers | `.env`, `__pycache__`, `*.pdf`, `.pytest_cache` |

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

# Downloaded files
*.pdf

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db
```

---

## 12. Tech Decision Log

Decisions made during planning and their rationale, for future reference:

| Decision | Chosen | Over | Why |
|----------|--------|------|-----|
| Language | Python 3.12 | Node.js, Go | Richest AI/ML ecosystem; every dependency is Python-native |
| arXiv access | `arxiv` package | `feedparser`, scraping | Cleanest API; handles rate limiting and pagination |
| PDF extraction | PyMuPDF | pdfplumber, pdfminer | Fastest; best quality on academic papers; no external deps |
| LLM provider | Anthropic Claude | OpenAI GPT-4o | Superior long-document summarization; larger context window |
| LLM model | Claude 3.5 Sonnet (all stages) | Mixed models (Haiku + Sonnet) | Cost difference negligible at this volume; simplicity wins |
| Email sending | Gmail SMTP (stdlib) | SendGrid, Resend, AWS SES | Zero cost, zero setup, sufficient for <50 subscribers |
| Email templating | Jinja2 | f-strings, mako | Industry standard; clean separation of template and code |
| Scheduling | GitHub Actions cron | AWS Lambda, local cron, Render | Free; zero infrastructure; built-in secrets management |
| Testing | pytest | unittest | Better assertions, fixtures, and parametrize support |
| Linting | ruff | black + flake8 | Single tool; faster; modern standard |
| Orchestration | Plain Python functions | LangChain, LangGraph, Celery | Pipeline is linear and simple; a framework would add complexity with no benefit |
| Database | None | SQLite, PostgreSQL | Stateless design; no data needs to persist between runs |

---

## 13. Summary

| Layer | Technology | Count |
|-------|-----------|-------|
| **Runtime** | Python 3.12 on GitHub Actions (Ubuntu) | — |
| **Runtime deps** | arxiv, PyMuPDF, requests, beautifulsoup4, anthropic, Jinja2, python-dotenv | 7 |
| **Dev deps** | pytest, pytest-mock, ruff | 3 |
| **Stdlib modules** | smtplib, email, json, os, pathlib, logging, datetime, dataclasses, traceback, time, tempfile | 11 |
| **External services** | GitHub Actions, Anthropic API, Gmail SMTP, arXiv API | 4 |
| **Monthly cost** | ~$10 (almost entirely Anthropic API) | — |
| **Credentials** | 3 secrets (Anthropic key, Gmail address, Gmail App Password) | 3 |

Minimal, focused, and every piece earns its place.
