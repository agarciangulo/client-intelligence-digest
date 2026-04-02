# Build Summary — ArXiv AI Paper Digest Agent

A technical summary of the project for reference when writing about the build.

---

## What It Does

An AI agent that reads every paper published daily on arXiv's `cs.AI` category (~150–200 papers/day), identifies the 10 most relevant ones based on a personal interest profile, and delivers a curated email digest every morning — fully automated, zero manual intervention.

The digest has two tiers:

- **Top 5 — Deep Dives**: The agent downloads the full PDF of each paper, extracts the complete text, and generates a 1,000–2,000 word summary structured into six sections: "So What?", "Core Idea", "How It Works", "Key Results", "Limitations & Open Questions", and "Real-World Applications & Opportunities." These are written for a smart non-specialist — accessible but not dumbed down.
- **Next 5 — Quick Reads**: 100–150 word blurbs generated from abstracts only, each ending with a "Read this if..." tag.

Every entry links to the original PDF and HTML versions of the paper.

---

## The Pipeline

The system is a **linear pipeline** — no databases, no servers, no queues. Just Python functions called in sequence, triggered by a cron job.

```
GitHub Actions (6 AM ET, Tue–Sat)
    │
    ▼
1. DATA COLLECTOR — Fetch all cs.AI papers from arXiv
    │                 (~150–200 papers, deduplicated)
    ▼
2. STAGE 1: RANKER — Claude reads every title + abstract
    │                  against a personal interest profile
    │                  → returns ranked top 10 with justifications
    ▼
3. STAGE 2a: DEEP ANALYZER — For the top 5:
    │   Download full PDF → extract text → send to Claude
    │   → 1,000–2,000 word structured summary per paper
    │
   STAGE 2b: BLURB GENERATOR — For papers #6–10:
    │   Send abstracts to Claude → 100–150 word blurb each
    ▼
4. EMAIL COMPOSER — Render everything into an HTML email
    │                  via Jinja2 templates
    ▼
5. EMAIL SENDER — Deliver via Gmail SMTP
```

Total runtime: **~5–7 minutes**. Total daily cost: **~$0.30–0.60** (almost entirely Anthropic API usage).

---

## Key Technical Decisions (and Why)

### Hybrid Data Collection

arXiv has two data access patterns, and neither alone was sufficient:

- The **RSS feed** gives you the correct daily paper list (what was actually announced that day), but only provides IDs and basic metadata.
- The **arxiv Python API** gives rich metadata (full abstracts, author lists, comments, categories), but its date filtering uses submission date, not announcement date — which doesn't match the daily listing.

The solution: **use both**. The RSS feed provides the correct set of arXiv IDs for the day, then the API fetches full metadata for those specific IDs in a batch lookup. This ensures the digest covers exactly what was published, with complete information.

### Claude Over GPT-4o

Anthropic's Claude was chosen over OpenAI's GPT-4o for a specific reason: long-document summarization. The deep dive summaries require processing full research papers (often 20–40 pages) and producing nuanced, accurate, accessible writeups. Claude's 200K token context window handles this comfortably, and in testing, it produced more faithful summaries that stayed grounded in what the paper actually said rather than editorializing.

The model used is `claude-sonnet-4-6` — consistent across all three LLM calls (ranking, deep analysis, blurb generation). Using a single model simplified the system, and the cost difference between Sonnet and a cheaper model at 7 calls/day was negligible (~$0.05/day savings).

### Full PDF Extraction, Not Just Abstracts

The deep dives read the **entire paper**, not just the abstract. This is what distinguishes the summaries from something you could get by pasting an abstract into ChatGPT. The agent uses PyMuPDF to extract text from PDFs, with an HTML fallback for papers where PDF extraction fails. Text is cleaned (headers, footers, references stripped) and truncated at ~80K tokens to stay within Claude's context window while leaving room for the prompt and output.

### Interest Profile as a First-Class Concept

The ranking isn't generic "what's important in AI." It's driven by a structured JSON profile that defines:

- **Primary interests** (highest weight): LLM agents, AI-hardware co-design, proprietary dataset strategies
- **Secondary interests** (moderate weight): RAG, AI safety, novel training techniques
- **Wildcard interest**: Genuinely novel paradigms, even if outside stated interests
- **Deprioritize list**: Pure CV, narrow medical AI, incremental benchmarks
- **Prioritized sources**: OpenAI, DeepMind, MIT, Stanford, etc. — used as a tiebreaker, not a gatekeeper

This profile is passed to the LLM at every stage. The ranker uses it to select papers. The deep analyzer uses it to emphasize aspects the reader cares about. The entire system is designed so swapping the profile changes what surfaces — making it extensible to multiple subscribers with different interests.

### Stateless by Design

Every run is completely independent. No database, no state file, no memory of previous runs. The pipeline fetches fresh data, processes it, sends an email, and exits. This means:

- No infrastructure to maintain
- No state corruption bugs
- Any run can be re-triggered safely
- The entire system is a single GitHub Actions workflow

### Shared LLM Client with Retry Logic

All Claude API calls go through a single `call_claude()` function that handles:

- Exponential backoff (5s → 10s → 20s → 40s → 60s cap)
- Random jitter (0–50% of delay) to prevent thundering herd
- Automatic retry on 429 (rate limit) and 529 (overloaded) errors
- Up to 5 retry attempts before failing

This was built after encountering persistent 529 errors during development — the API would intermittently return overloaded status, especially with older model IDs.

### Gmail SMTP — Simplest Thing That Works

Email delivery uses Python's built-in `smtplib` with a Gmail App Password. No SendGrid, no Resend, no AWS SES. For a single subscriber (or a small mailing list), this is zero-cost and takes about 10 lines of code. The tradeoff: no open/click analytics, no fancy unsubscribe links, 500 emails/day cap. All acceptable at this scale.

---

## Architecture at a Glance

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Language | Python 3.12 | Richest AI/ML ecosystem |
| LLM | Claude Sonnet 4.6 | Best long-document summarization |
| PDF extraction | PyMuPDF | Fast, no external deps, handles academic layouts |
| Email | Gmail SMTP (stdlib) | Zero cost, minimal code |
| Scheduling | GitHub Actions cron | Free, zero infrastructure |
| Orchestration | Plain Python functions | Pipeline is linear — a framework would add complexity with no benefit |
| Database | None | Stateless design; nothing needs to persist |
| Total dependencies | 7 runtime, 3 dev-only | Minimal footprint |

---

## Numbers

| Metric | Value |
|--------|-------|
| Papers scanned per day | ~150–200 |
| LLM calls per run | 7 (1 ranking + 5 deep analysis + 1 blurb batch) |
| Deep dive word count | 1,000–2,000 words each |
| Blurb word count | 100–150 words each |
| Pipeline runtime | ~5–7 minutes |
| Daily cost | ~$0.30–0.60 |
| Monthly cost | ~$10 |
| Lines of Python | ~700 (excluding tests and templates) |
| Runtime dependencies | 7 |
| External services | 4 (GitHub Actions, Anthropic, Gmail, arXiv) |
| Infrastructure to maintain | None |

---

## The Build Process

The project was built in 8 phases following a "Build → Verify → Advance" philosophy, with checkpoints after every meaningful piece of work:

1. **Project setup** — Structure, config files, logging, dev caching utilities
2. **Data collector** — Hybrid RSS + API approach with deduplication
3. **PDF extractor** — Download, text extraction, cleaning, HTML fallback
4. **Stage 1 ranker** — LLM-powered paper ranking against user profile
5. **Stage 2 analyzer + blurbs** — Deep summaries and short blurbs, shared LLM client
6. **Email system** — Jinja2 templates, SMTP sender, markdown-to-HTML conversion
7. **Pipeline orchestrator** — Wire everything together, DRY_RUN mode, error handling
8. **GitHub Actions deployment** — Cron workflow, secrets configuration, manual trigger testing

Each phase included unit tests (mocked, fast, free) and live checkpoint tests (real APIs, manual verification). The dev caching system allowed replaying intermediate results during development without re-running expensive API calls.

---

## Interesting Problems Encountered

### arXiv's Two Date Systems

arXiv papers have a **submission date** and an **announcement date**, and they're not the same thing. The API filters by submission date, but the daily listing page shows papers by announcement date. This meant a naive API query would return the wrong set of papers. The hybrid approach (RSS for IDs, API for metadata) solved this cleanly.

### PDF Text Quality

Academic PDFs are notoriously messy to extract text from. Multi-column layouts, LaTeX-generated ligatures, headers/footers on every page, inline math, references sections that bloat the token count. The pipeline includes a cleaning step that strips page numbers, running headers, and reference sections, then truncates to ~80K tokens. For papers where PDF extraction fails entirely, it falls back to arXiv's HTML rendering.

### LLM API Reliability

During development, Claude's API returned 529 (overloaded) errors intermittently. The fix was threefold: update to the current model ID (`claude-sonnet-4-6`), implement exponential backoff with jitter, and centralize all API calls through a shared client. The jitter is important — without it, retries from multiple calls would all hit the API at the same intervals, making congestion worse.

### Secret Scanning on Push

GitHub's push protection caught the Anthropic API key that had been committed in `.env.example` during early development. Even though it was later replaced with a placeholder, the real key persisted in git history. The fix required rewriting all commits with `git filter-branch` to scrub the secret, then rotating the key.

---

## What's Not Built (Yet)

- Per-subscriber interest profiles (currently shared)
- Web UI or dashboard
- Historical paper tracking / cross-day deduplication
- Reading analytics (which papers get clicked)
- Slack/Teams delivery
- Multiple arXiv category support
- Weekly digest mode

The architecture is intentionally simple enough that any of these could be added without a rewrite. The pipeline is linear, the components are decoupled, and the configuration is externalized.

---

## Project Structure

```
Research-Paper-Parser/
├── .github/workflows/
│   └── daily_digest.yml        # GitHub Actions cron job
├── config/
│   ├── user_profile.json       # Interest profile for ranking
│   └── subscribers.json        # Email recipient list
├── src/
│   ├── main.py                 # Pipeline orchestrator
│   ├── collector.py            # arXiv data collection (RSS + API)
│   ├── ranker.py               # Stage 1: LLM ranking
│   ├── pdf_extractor.py        # PDF download + text extraction
│   ├── analyzer.py             # Stage 2: Deep dive summaries
│   ├── blurb_generator.py      # Stage 2: Short blurbs
│   ├── llm.py                  # Shared Claude client with retry logic
│   ├── email_composer.py       # Jinja2 template rendering
│   ├── email_sender.py         # Gmail SMTP delivery
│   ├── logger.py               # Centralized logging
│   └── dev_cache.py            # Development caching utility
├── templates/
│   ├── digest_email.html       # Main digest email template
│   └── quiet_day_email.html    # "No papers today" template
├── tests/                      # Unit tests for all modules
├── docs/                       # SCOPE, ARCHITECTURE, TECH_STACK, EXECUTION_PLAN
├── requirements.txt
└── .env.example
```
