# Architecture — ArXiv AI Paper Digest Agent

This document defines the system architecture, component design, data flows, and prompt specifications for the ArXiv AI Paper Digest Agent described in `docs/SCOPE.md`.

---

## 1. System Overview

The agent is a **linear pipeline** that runs once daily via GitHub Actions. There are no persistent servers, no databases, and no user-facing APIs — just a scheduled job that collects, analyzes, and emails.

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                              ARXIV AI PAPER DIGEST AGENT                                     │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌────────┐ │
│  │    DATA       │    │   STAGE 1    │    │   STAGE 2    │    │    EMAIL     │    │  EMAIL │ │
│  │  COLLECTOR    │───▶│   RANKER     │───▶│    DEEP      │───▶│  COMPOSER   │───▶│ SENDER │ │
│  │              │    │              │    │  ANALYZER    │    │             │    │        │ │
│  │ arXiv API    │    │ All titles + │    │ Full PDFs    │    │ HTML email  │    │ Gmail  │ │
│  │ ~200 papers  │    │ abstracts    │    │ for top 5    │    │ template    │    │ SMTP   │ │
│  │              │    │ → top 10     │    │ + blurbs 6-10│    │             │    │        │ │
│  └──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘    └────────┘ │
│         │                   │                   │                   │               │        │
│         └───────────────────┴───────────────────┴───────────────────┴───────────────┘        │
│                                              │                                               │
│                                    ┌─────────▼─────────┐                                    │
│                                    │   ERROR HANDLER    │                                    │
│                                    │                    │                                    │
│                                    │ Catches failures   │                                    │
│                                    │ at any stage →     │                                    │
│                                    │ sends error email  │                                    │
│                                    └────────────────────┘                                    │
│                                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────────┐    │
│  │                              CONFIGURATION LAYER                                      │    │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌──────────────────┐   │    │
│  │  │ User Profile   │  │ Subscribers    │  │ Environment    │  │ Email Template   │   │    │
│  │  │ (JSON)         │  │ (JSON)         │  │ Variables      │  │ (Jinja2/HTML)    │   │    │
│  │  └────────────────┘  └────────────────┘  └────────────────┘  └──────────────────┘   │    │
│  └──────────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                              │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Design Principles

| Principle | Implementation |
|-----------|----------------|
| **Stateless** | No database, no persistent state between runs. Each run is independent. |
| **Fail-safe** | Any stage failure triggers an error email; never fails silently. |
| **Idempotent** | Re-running the same day produces the same results (same input papers → same output). |
| **Cost-conscious** | Use cheaper models where quality allows; full Claude only for deep analysis. |
| **Simple** | No orchestration framework, no queues, no microservices. Just Python functions called in sequence. |

---

## 2. Pipeline Flow — Detailed

### 2.1 End-to-End Sequence

```
GitHub Actions Cron (6 AM ET / 11:00 UTC, Tue-Sat)
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. DATA COLLECTOR                                                │
│    • Fetch all cs.AI papers from previous day via arXiv API      │
│    • Extract: arxiv_id, title, authors, abstract, comments,      │
│      subjects, pdf_url, html_url                                 │
│    • Deduplicate by arxiv_id                                     │
│    • Output: List[Paper] (~150-200 papers)                       │
│                                                                   │
│    IF 0 papers found → send "quiet day" email → EXIT              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. STAGE 1: RANKER (LLM Call #1)                                 │
│    • Input: All papers (titles + abstracts + authors + comments)  │
│             + User Profile                                        │
│    • Model: Claude 3.5 Sonnet (balance of cost and quality)       │
│    • Output: Ranked top 10-15 papers with justifications          │
│    • Split: Top 5 → deep analysis, #6-10 → blurbs                │
└────────────────────────┬────────────────────────────────────────┘
                         │
                    ┌────┴────┐
                    │         │
              Top 5 ▼    #6-10 ▼
┌────────────────────┐  ┌────────────────────────────────────────┐
│ 3a. PDF FETCHER    │  │ 3c. BLURB GENERATOR (LLM Call #7)       │
│  • Download PDFs   │  │  • Input: 5 abstracts + user profile    │
│    for top 5       │  │  • Model: Claude 3.5 Sonnet              │
│  • Extract text    │  │  • Output: 100-150 word blurb per paper  │
│  • Rate-limited    │  │  • Single call for all 5 blurbs          │
│    (3s between)    │  └─────────────────────┬──────────────────┘
└─────────┬──────────┘                        │
          │                                   │
          ▼                                   │
┌─────────────────────────────────────────┐   │
│ 3b. DEEP ANALYZER (LLM Calls #2-6)      │   │
│  • Input: Full paper text + user profile │   │
│  • Model: Claude 3.5 Sonnet              │   │
│  • Output: 1,000-2,000 word summary      │   │
│  • One call per paper (5 calls total)    │   │
│  • Summary follows format spec from      │   │
│    SCOPE.md (So What → Core Idea →       │   │
│    How It Works → Results → Limitations  │   │
│    → Applications & Opportunities)       │   │
└─────────────────────┬───────────────────┘   │
                      │                       │
                      └───────────┬───────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. EMAIL COMPOSER                                                │
│    • Input: 5 deep summaries + 5 blurbs + paper metadata         │
│    • Template: Jinja2 HTML template                               │
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
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 LLM Call Summary

| Call # | Stage | Model | Input Size | Output Size | Purpose |
|--------|-------|-------|------------|-------------|---------|
| 1 | Stage 1: Ranker | Claude 3.5 Sonnet | ~60-80K tokens (all papers) | ~2K tokens (ranked list) | Rank all papers, select top 10 |
| 2-6 | Stage 2: Deep Analyzer | Claude 3.5 Sonnet | ~15-30K tokens each (full paper) | ~2-3K tokens each (summary) | Deep 1,000-2,000 word summaries |
| 7 | Stage 2: Blurb Generator | Claude 3.5 Sonnet | ~3K tokens (5 abstracts) | ~1K tokens (5 blurbs) | Short blurbs for #6-10 |

**Total LLM calls per run:** 7
**Estimated daily cost:** ~$0.30–0.60

---

## 3. Component Details

### 3.1 Data Collector

**Responsibility:** Fetch all `cs.AI` papers published on arXiv the previous day.

**Input:** Date (previous business day)
**Output:** `List[Paper]` — structured paper objects

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│   arXiv API      │      │  Paper Parser    │      │   Deduplicator   │
│                  │      │                  │      │                  │
│ Fetch cs.AI      │─────▶│ Extract fields:  │─────▶│ Remove duplicate │
│ papers for       │      │ • arxiv_id       │      │ arxiv_ids        │
│ target date      │      │ • title          │      │ (cross-listings) │
│                  │      │ • authors        │      │                  │
│ Rate-limited:    │      │ • abstract       │      │ Output:          │
│ 3s between       │      │ • comments       │      │ List[Paper]      │
│ requests         │      │ • subjects       │      │                  │
│                  │      │ • pdf_url        │      │                  │
│                  │      │ • html_url       │      │                  │
└──────────────────┘      └──────────────────┘      └──────────────────┘
```

**Paper Data Model:**

```python
@dataclass
class Paper:
    arxiv_id: str           # e.g., "2602.17663"
    title: str              # Paper title
    authors: list[str]      # List of author names
    abstract: str           # Full abstract text
    comments: str | None    # e.g., "Accepted at ICLR 2026"
    subjects: list[str]     # e.g., ["cs.AI", "cs.CL"]
    pdf_url: str            # https://arxiv.org/pdf/2602.17663
    html_url: str           # https://arxiv.org/html/2602.17663
    published_date: str     # ISO date
```

**API Strategy:**

| Approach | Pros | Cons | Recommendation |
|----------|------|------|----------------|
| `arxiv` Python package | Clean API, handles pagination | May need date filtering logic | ✅ Primary |
| Atom/RSS feed | Real-time, lightweight | Limited metadata | Fallback |
| Scraping the HTML listing | Gets exactly what the page shows | Fragile, against ToS spirit | ❌ Avoid |

**Date Handling:**
- The agent runs Tuesday–Saturday
- It always fetches **the previous business day's** papers
- Friday's run fetches Thursday's papers; Saturday's run fetches Friday's papers
- If no papers are found (holiday), the pipeline sends a quiet day notice and exits

---

### 3.2 Stage 1: Ranker

**Responsibility:** Evaluate all papers against the user profile and return a ranked top 10–15.

**Input:** `List[Paper]` (all papers) + User Profile (from config)
**Output:** `RankedResults` — top 10-15 papers with justifications, split into "deep dive" (top 5) and "blurb" (next 5) tiers

```
┌──────────────────────────────────────────────────────────────────┐
│                        STAGE 1: RANKER                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Input Assembly:                                                  │
│  ┌─────────────────────┐  ┌──────────────────────┐                │
│  │  User Profile       │  │  All Papers          │                │
│  │  (from config)      │  │  (from collector)    │                │
│  │                     │  │                      │                │
│  │  • Interests        │  │  • Title             │                │
│  │  • Deprioritize     │  │  • Authors           │                │
│  │  • Sources          │  │  • Abstract          │                │
│  │  • Preferences      │  │  • Comments          │                │
│  │  • Wildcard note    │  │  • Subjects          │                │
│  └──────────┬──────────┘  └──────────┬───────────┘                │
│             │                        │                            │
│             └────────────┬───────────┘                            │
│                          │                                        │
│                          ▼                                        │
│                ┌──────────────────┐                               │
│                │   LLM Call #1    │                               │
│                │   Claude Sonnet  │                               │
│                │                  │                               │
│                │  "Rank these     │                               │
│                │   papers by      │                               │
│                │   relevance to   │                               │
│                │   this profile"  │                               │
│                └────────┬─────────┘                               │
│                         │                                        │
│                         ▼                                        │
│              ┌────────────────────┐                              │
│              │  Ranked Output     │                              │
│              │                    │                              │
│              │  Top 5:  → Deep    │                              │
│              │  #6-10:  → Blurbs  │                              │
│              │  Each with:        │                              │
│              │  • Rank            │                              │
│              │  • Justification   │                              │
│              │  • Relevance tier  │                              │
│              └────────────────────┘                              │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

**Ranking Output Schema:**

```json
{
  "total_papers_evaluated": 187,
  "ranking_date": "2026-02-20",
  "top_papers": [
    {
      "rank": 1,
      "arxiv_id": "2602.17560",
      "title": "ODESteer: A Unified ODE-Based Steering Framework...",
      "tier": "deep_dive",
      "justification": "Directly addresses LLM alignment through a novel ODE-based framework...",
      "relevance_tags": ["LLM agents", "AI reasoning"],
      "source_match": "ICLR 2026 accepted",
      "is_wildcard": false
    }
  ]
}
```

---

### 3.3 Stage 2: PDF Fetcher & Text Extractor

**Responsibility:** Download and extract readable text from the top 5 papers' PDFs.

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│   PDF Download   │      │  Text Extraction │      │  Text Cleanup    │
│                  │      │                  │      │                  │
│ Download PDF     │─────▶│ Extract text     │─────▶│ Remove:          │
│ from arXiv       │      │ from PDF using   │      │ • Page headers   │
│                  │      │ pymupdf          │      │ • References     │
│ Rate-limited:    │      │                  │      │   section (opt.) │
│ 3s between       │      │ Fallback:        │      │ • Figure/table   │
│ downloads        │      │ Try HTML version │      │   captions (opt.)│
│                  │      │ if PDF fails     │      │                  │
│                  │      │                  │      │ Truncate if      │
│                  │      │                  │      │ > 80K tokens     │
└──────────────────┘      └──────────────────┘      └──────────────────┘
```

**Text Extraction Strategy:**

| Method | When to Use | Pros | Cons |
|--------|-------------|------|------|
| **PyMuPDF (fitz)** | Primary — works on all PDFs | Fast, reliable, good text extraction | Some formatting artifacts |
| **arXiv HTML** | Fallback / preferred when available | Clean text, proper structure | Not all papers have HTML versions |

**Text Cleanup Rules:**
1. Remove repeated headers/footers (page numbers, running titles)
2. Optionally trim the References section (saves tokens, not needed for summary)
3. If extracted text exceeds ~80K tokens (~60K words), truncate from the end (keeping abstract, introduction, methodology, results)
4. Preserve section headings for structural context

**Token Budget per Paper:**

| Component | Estimated Tokens |
|-----------|-----------------|
| Full paper text | 15,000–30,000 |
| System prompt + user profile | ~2,000 |
| Summary format instructions | ~500 |
| **Total input** | **~17,500–32,500** |
| Generated summary (1,000-2,000 words) | ~1,500–3,000 |
| **Headroom in 200K context** | ✅ Plenty |

---

### 3.4 Stage 2: Deep Analyzer

**Responsibility:** Generate a detailed, accessible, 1,000–2,000 word summary of each top-5 paper.

**Input:** Full paper text + User Profile + Summary format spec
**Output:** Structured summary following the 6-section format from SCOPE.md
**Model:** Claude 3.5 Sonnet
**Calls:** 5 (one per paper, sequential)

```
For each of the top 5 papers:
    ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
    │  Full Paper Text │      │   LLM Call       │      │  Structured      │
    │  + User Profile  │─────▶│   Claude Sonnet  │─────▶│  Summary         │
    │  + Format Spec   │      │                  │      │                  │
    │                  │      │  Generate        │      │  • So What?      │
    │                  │      │  1,000-2,000     │      │  • Core Idea     │
    │                  │      │  word summary    │      │  • How It Works  │
    │                  │      │                  │      │  • Key Results   │
    │                  │      │                  │      │  • Limitations   │
    │                  │      │                  │      │  • Applications  │
    └──────────────────┘      └──────────────────┘      └──────────────────┘
```

**Why Sequential, Not Parallel?**
- arXiv rate limiting means PDFs are downloaded sequentially anyway
- Sequential calls are simpler to debug and log
- Total time for 5 calls ≈ 2–3 minutes — well within our 15-minute budget
- Parallel calls can be added later if needed

---

### 3.5 Stage 2: Blurb Generator

**Responsibility:** Generate short blurbs for papers ranked #6–10.

**Input:** 5 abstracts + User Profile
**Output:** 100–150 word blurb per paper
**Model:** Claude 3.5 Sonnet
**Calls:** 1 (all 5 blurbs in a single call)

This is a single, cheap LLM call. No PDF download needed — abstracts are sufficient for short blurbs.

---

### 3.6 Email Composer

**Responsibility:** Assemble the final HTML email from summaries, blurbs, and metadata.

**Input:** 5 deep summaries + 5 blurbs + paper metadata + daily stats
**Output:** Complete HTML email string
**LLM:** None — pure template rendering

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  Template Data   │      │  Jinja2 Engine   │      │  HTML Email      │
│                  │      │                  │      │                  │
│  • Deep summaries│─────▶│ Render template  │─────▶│ Ready to send    │
│  • Blurbs        │      │ with data        │      │                  │
│  • Paper metadata│      │                  │      │ Includes:        │
│  • Date          │      │ Apply styling    │      │ • Inline CSS     │
│  • Stats         │      │                  │      │ • PDF/HTML links │
│  • Profile name  │      │                  │      │ • Stats footer   │
└──────────────────┘      └──────────────────┘      └──────────────────┘
```

**Template Sections:**

| Section | Content | Data Source |
|---------|---------|-------------|
| Header | Digest title, date, paper count | Pipeline metadata |
| Deep Dives (1-5) | Title, authors, venue, links, full summary | Deep Analyzer output |
| Quick Reads (6-10) | Title, authors, links, blurb | Blurb Generator output |
| Footer | Stats, profile name, update instructions | Config + metadata |

**Email Styling Notes:**
- Use **inline CSS** (many email clients strip `<style>` blocks)
- Keep layout single-column for mobile compatibility
- Use web-safe fonts (Georgia, Arial, system fonts)
- Test with Gmail rendering (primary target)

---

### 3.7 Email Sender

**Responsibility:** Deliver the composed email to all active subscribers.

**Input:** HTML email string + subscriber list
**Output:** Delivery status per recipient

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  Subscriber List │      │  Gmail SMTP      │      │  Delivery Log    │
│  (from config)   │      │                  │      │                  │
│                  │─────▶│ For each active  │─────▶│ Log per          │
│  Filter: active  │      │ subscriber:      │      │ recipient:       │
│  only            │      │ • Connect SMTP   │      │ • email          │
│                  │      │ • Send email     │      │ • status         │
│                  │      │ • Close          │      │ • timestamp      │
└──────────────────┘      └──────────────────┘      └──────────────────┘
```

**SMTP Configuration:**

| Setting | Value |
|---------|-------|
| Server | `smtp.gmail.com` |
| Port | `587` (TLS) |
| Auth | Gmail App Password |
| From | Configured sender address |
| Subject | `📰 ArXiv AI Digest — {Date}` |
| Content-Type | `text/html` |

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
│  │  • Timestamp     │      │  ArXiv Digest    │                  │
│  │  • Papers found  │      │  Pipeline Error  │                  │
│  │    (if any)      │      │                  │                  │
│  └──────────────────┘      └──────────────────┘                  │
│                                                                    │
│  Error email contains:                                            │
│  • Which stage failed (Data Collector / Ranker / Analyzer / etc.) │
│  • Error message and type                                         │
│  • Truncated traceback (last 20 lines)                            │
│  • Timestamp (UTC + ET)                                           │
│  • Note: "Will retry on next scheduled run"                       │
│                                                                    │
│  Also: log full error to GitHub Actions console for debugging     │
│                                                                    │
└─────────────────────────────────────────────────────────────────┘
```

**Failure Modes & Handling:**

| Failure | Stage | Handling | User Impact |
|---------|-------|----------|-------------|
| arXiv API unreachable | Data Collector | Retry 3x with 30s backoff → error email | No digest today |
| 0 papers returned (holiday) | Data Collector | Send quiet day notice → exit cleanly | Gets notice |
| LLM API error | Ranker or Analyzer | Retry 2x → error email | No digest today |
| LLM returns malformed JSON | Ranker | Retry with stricter prompt → error email | No digest today |
| PDF download fails (1 paper) | PDF Fetcher | Skip paper, promote #6 to deep dive | Digest with 4 deep + 1 promoted |
| PDF text extraction fails | PDF Fetcher | Try HTML fallback → if both fail, treat as blurb | Graceful degradation |
| Gmail SMTP error | Email Sender | Retry 3x → log error (can't email about email failure) | No digest; visible in GH Actions logs |
| Summary too short (<800 words) | Deep Analyzer | Re-prompt once with "please expand" → accept if improved | Slightly shorter summary |

---

## 4. Configuration Files

### 4.1 File Structure

```
config/
├── user_profile.json       # Interests, priorities, sources
├── subscribers.json         # Email recipient list
└── .env                     # API keys, SMTP credentials (not in repo)
```

### 4.2 User Profile (`config/user_profile.json`)

This is the full profile structure defined in `SCOPE.md`. The Ranker and Deep Analyzer both receive this as context.

### 4.3 Subscribers (`config/subscribers.json`)

```json
{
  "subscribers": [
    {
      "email": "andres@example.com",
      "name": "Andres",
      "active": true
    }
  ]
}
```

### 4.4 Environment Variables

| Variable | Purpose | Stored In |
|----------|---------|-----------|
| `ANTHROPIC_API_KEY` | Claude API authentication | GitHub Secrets |
| `GMAIL_ADDRESS` | Sender email address | GitHub Secrets |
| `GMAIL_APP_PASSWORD` | Gmail App Password for SMTP | GitHub Secrets |

---

## 5. Prompt Specifications

This section defines the exact prompt structure for each LLM-calling stage. These are the contracts the model must fulfill.

---

### 5.1 Stage 1: Ranker Prompt

**Purpose:** Evaluate all daily papers against the user profile, return a ranked top 10.

**Temperature:** 0.3 (structured, consistent ranking)

**System Prompt:**

```
You are an expert AI research curator. Your job is to read through a day's
worth of arXiv papers and identify the ones most relevant, impactful, and
interesting to a specific reader based on their profile.

You are thorough, fair, and intellectually curious. You don't just match
keywords — you understand the significance of research contributions and
can identify genuinely important work even when it doesn't perfectly match
the reader's stated interests.
```

**User Prompt:**

```
## Reader Profile

{user_profile_json}

## Today's Papers ({paper_count} total)

{for each paper:}
---
[{index}] arxiv_id: {arxiv_id}
Title: {title}
Authors: {authors}
Abstract: {abstract}
Comments: {comments or "None"}
Subjects: {subjects}
---

## Instructions

Evaluate ALL papers above against the reader's profile and return the top 10
most relevant papers, ranked from most to least important.

For each selected paper, provide:
1. The arxiv_id
2. A justification (2-3 sentences) explaining why this paper matters
   to this specific reader
3. Relevance tags (which profile interests it matches)
4. Whether this is a "wildcard" pick (innovative/revolutionary but
   outside the reader's stated interests)

RANKING CRITERIA (in priority order):
1. Topic relevance to primary interests (highest weight)
2. Novelty — new paradigm or method vs. incremental improvement
3. Practical applicability — can it be used in industry?
4. Source credibility — from a prioritized institution/lab? (boost, not gatekeeper)
5. Venue acceptance — accepted at a top conference? (noted in comments)
6. Breadth of impact — relevant across domains?
7. Wildcard potential — genuinely revolutionary even if outside stated interests?

IMPORTANT RULES:
- Primary interests get MUCH higher weight than secondary
- Papers matching "deprioritize" topics should be excluded unless
  they are truly exceptional
- Include at least 1 wildcard pick if anything qualifies
- If a paper is from a prioritized source, note it but don't rank it
  higher JUST for that — quality and relevance come first
- A great paper from an unknown lab should absolutely make the list

RETURN THIS EXACT JSON STRUCTURE:
{
  "total_papers_evaluated": <number>,
  "ranking_date": "<YYYY-MM-DD>",
  "top_papers": [
    {
      "rank": <1-10>,
      "arxiv_id": "<id>",
      "title": "<paper title>",
      "tier": "deep_dive" | "blurb",
      "justification": "<2-3 sentences>",
      "relevance_tags": ["<matching interest 1>", "<matching interest 2>"],
      "source_match": "<institution/venue note or null>",
      "is_wildcard": <true|false>
    }
  ]
}

Papers ranked 1-5 should have tier "deep_dive".
Papers ranked 6-10 should have tier "blurb".

Return ONLY valid JSON. No markdown, no commentary outside the JSON.
```

---

### 5.2 Stage 2: Deep Analyzer Prompt

**Purpose:** Generate a detailed, accessible summary of a single research paper.

**Temperature:** 0.5 (balanced — creative enough for good writing, grounded enough for accuracy)

**System Prompt:**

```
You are a world-class science communicator who makes cutting-edge AI research
accessible to smart professionals who aren't necessarily researchers. Think of
your audience as a tech-savvy executive or senior engineer who wants to
understand what's happening in AI without reading full papers.

Your writing style is:
- Clear and direct, never condescending
- Uses analogies and examples to explain complex ideas
- Includes specific results and numbers when they tell a story
- Honest about limitations — you don't overhype
- Engaging — the reader should want to finish the summary

You write in a professional but warm tone. No jargon without explanation.
No hand-waving. Every claim is grounded in what the paper actually says.
```

**User Prompt:**

```
## Reader Profile (for context on what matters to them)

{user_profile_json}

## Paper Metadata

Title: {title}
Authors: {authors}
arXiv ID: {arxiv_id}
Subjects: {subjects}
Comments: {comments or "None"}
Venue: {extracted venue or "Not specified"}

## Ranking Context

This paper was ranked #{rank} out of {total_papers} papers today.
Ranking justification: {justification_from_stage1}
Relevance tags: {relevance_tags}

## Full Paper Text

{full_paper_text}

## Instructions

Write a detailed summary of this paper following this EXACT structure.
Target length: 1,000-2,000 words total across all sections.

### 1. The "So What?" (1 paragraph)
Open with why this paper matters in plain language. What problem does it
solve? Why should a busy professional care? Lead with impact, not
technical details.

### 2. The Core Idea (2-3 paragraphs)
Explain the key insight or method in accessible terms. Use analogies
where they genuinely help. Assume the reader is intelligent but not a
specialist in this specific sub-field.

### 3. How It Works (2-3 paragraphs)
Walk through the approach clearly. Not a full technical deep dive, but
enough that the reader understands the mechanism. Skip heavy math —
explain the intuition behind the math. If there's a novel architecture
or pipeline, describe it step by step.

### 4. Key Results (1-2 paragraphs)
What did they find? How does it compare to previous work? Include
specific numbers, percentages, or benchmarks when they tell a
compelling story. Don't list every result — highlight the ones that
matter most.

### 5. Limitations & Open Questions (1 paragraph)
Be honest. What doesn't the paper address? What assumptions does it
make? What would need to happen for this to be practically useful at
scale? This section builds trust with the reader.

### 6. Real-World Applications & Opportunities (1-2 paragraphs)
Concrete examples of how this research could be applied in industry,
products, or businesses. What opportunities does it create? Who should
be paying attention — and what could they build with this? Connect it
to the reader's interests where relevant.

FORMAT RULES:
- Use markdown headers (### ) for each section
- Write in flowing prose, not bullet points (except where a short list
  genuinely aids clarity)
- Do NOT include the paper title or authors in the summary body — those
  are in the email template
- Do NOT start with "This paper..." — lead with the problem or insight
- Aim for the high end of the word range (closer to 2,000 than 1,000)
  when the paper warrants it
```

---

### 5.3 Blurb Generator Prompt

**Purpose:** Generate short blurbs for papers ranked #6–10.

**Temperature:** 0.4 (concise, focused)

**System Prompt:**

```
You are a concise AI research curator. You write sharp, informative blurbs
that help busy professionals decide whether to read a paper. Every word earns
its place.
```

**User Prompt:**

```
## Reader Profile

{user_profile_json}

## Papers to Summarize

{for each paper #6-10:}
---
[{rank}] arxiv_id: {arxiv_id}
Title: {title}
Authors: {authors}
Abstract: {abstract}
Comments: {comments or "None"}
Ranking justification: {justification_from_stage1}
---

## Instructions

For EACH paper above, write a blurb of 100-150 words that:

1. States the core contribution in 1-2 crisp sentences
2. Explains why it's noteworthy (what's new or different)
3. Ends with a "Read this if:" tag pointing to who should care

RETURN THIS EXACT JSON STRUCTURE:
{
  "blurbs": [
    {
      "arxiv_id": "<id>",
      "rank": <number>,
      "blurb": "<100-150 word blurb text>",
      "read_this_if": "<one-line tag, e.g., 'you're building multi-agent systems'>"
    }
  ]
}

Return ONLY valid JSON. No markdown, no commentary outside the JSON.
```

---

### 5.4 Temperature Guidelines

| Temperature | Rationale | Stages |
|-------------|-----------|--------|
| **0.3** | Consistent, structured output; minimal creative variance | Stage 1: Ranker |
| **0.4** | Concise and focused; slight creative flexibility | Blurb Generator |
| **0.5** | Balanced — good writing quality with factual grounding | Deep Analyzer |

---

### 5.5 Prompt Engineering Principles

| Principle | Application |
|-----------|-------------|
| **Structured output** | Always request JSON with explicit schema; easier to parse and validate |
| **Role-based system prompts** | Each stage has a distinct persona (curator vs. communicator vs. concise writer) |
| **Reader context** | Always include the user profile so the LLM tailors its output |
| **Explicit format rules** | Word counts, section structures, and anti-patterns ("Don't start with...") |
| **Grounding** | Deep Analyzer sees the full paper, not just abstract — summaries are grounded in actual content |
| **Audit trail** | Ranker justifications explain why each paper was selected — useful for debugging and trust |

---

## 6. Pipeline Timing Estimate

| Stage | Duration | Notes |
|-------|----------|-------|
| Data Collector | ~30-60s | arXiv API fetch + parsing, rate-limited |
| Stage 1: Ranker | ~15-30s | Single LLM call, large input |
| PDF Download (5 papers) | ~15-20s | 3s rate limit × 5 papers |
| Text Extraction | ~5-10s | Local processing, fast |
| Deep Analyzer (5 papers) | ~2-4 min | 5 sequential LLM calls, ~30-45s each |
| Blurb Generator | ~10-15s | Single LLM call, small input |
| Email Composer | ~1s | Template rendering, no LLM |
| Email Sender | ~5-10s | SMTP connection + send |
| **Total** | **~4-7 minutes** | Well within 15-minute budget |

---

## 7. Directory Structure

```
arxiv-digest/
├── .github/
│   └── workflows/
│       └── daily_digest.yml       # GitHub Actions cron job
│
├── config/
│   ├── user_profile.json          # Reader interests & preferences
│   └── subscribers.json           # Email recipient list
│
├── src/
│   ├── __init__.py
│   ├── main.py                    # Pipeline orchestrator (entry point)
│   ├── collector.py               # Data Collector — arXiv API
│   ├── ranker.py                  # Stage 1 — LLM ranking
│   ├── pdf_extractor.py           # PDF download + text extraction
│   ├── analyzer.py                # Stage 2 — Deep summaries
│   ├── blurb_generator.py         # Stage 2 — Short blurbs
│   ├── email_composer.py          # HTML email assembly
│   ├── email_sender.py            # Gmail SMTP delivery
│   └── error_handler.py           # Error capture + notification
│
├── templates/
│   └── digest_email.html          # Jinja2 email template
│
├── tests/
│   ├── test_collector.py
│   ├── test_ranker.py
│   ├── test_analyzer.py
│   └── test_email.py
│
├── docs/
│   ├── SCOPE.md
│   ├── ARCHITECTURE.md            # This document
│   ├── TECH_STACK.md
│   └── EXECUTION_PLAN.md
│
├── requirements.txt
├── .env.example                    # Template for environment variables
└── .gitignore
```

**Module Responsibilities:**

| Module | Lines (est.) | LLM? | Description |
|--------|-------------|------|-------------|
| `main.py` | ~80 | No | Orchestrates the pipeline; calls each stage in sequence |
| `collector.py` | ~100 | No | Fetches papers from arXiv API, deduplicates |
| `ranker.py` | ~80 | Yes | Builds ranking prompt, calls Claude, parses response |
| `pdf_extractor.py` | ~120 | No | Downloads PDFs, extracts text, cleans up |
| `analyzer.py` | ~80 | Yes | Builds deep analysis prompt, calls Claude per paper |
| `blurb_generator.py` | ~60 | Yes | Builds blurb prompt, calls Claude once for all 5 |
| `email_composer.py` | ~60 | No | Loads Jinja2 template, renders with data |
| `email_sender.py` | ~60 | No | SMTP connection, sends to subscriber list |
| `error_handler.py` | ~50 | No | Try/except wrapper, error email formatting |

**Estimated total:** ~700 lines of Python (excluding tests and templates)

---

## 8. GitHub Actions Workflow

```yaml
# .github/workflows/daily_digest.yml
name: ArXiv AI Digest

on:
  schedule:
    # 11:00 UTC = 6:00 AM ET, Tuesday through Saturday
    - cron: '0 11 * * 2-6'
  workflow_dispatch:  # Allow manual trigger from GitHub UI

jobs:
  digest:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - uses: actions/checkout@v4

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
```

**Key Features:**
- `schedule` runs automatically on the cron
- `workflow_dispatch` allows manual runs from the GitHub UI (useful for testing)
- `timeout-minutes: 15` kills the job if it hangs
- Secrets are injected as environment variables — never in code

---

## 9. Future Extension Points

The architecture is intentionally simple, but these are the natural extension seams:

| Extension | Where It Plugs In | Complexity |
|-----------|-------------------|------------|
| **Additional arXiv categories** | `collector.py` — add categories to fetch list | Low |
| **Per-subscriber profiles** | `ranker.py` / `analyzer.py` — loop per profile | Medium |
| **Weekly digest mode** | `main.py` — aggregate multiple days before ranking | Low |
| **SendGrid/Resend** | `email_sender.py` — swap SMTP for API client | Low |
| **Paper history / dedup across days** | Add simple JSON or SQLite file | Low |
| **Slack/Teams delivery** | Add `slack_sender.py` alongside email sender | Medium |
| **Reading history feedback** | Track which papers users click → refine profile | High |

---

## 10. Summary

| Aspect | Design Choice | Rationale |
|--------|---------------|-----------|
| **Architecture** | Linear pipeline, no framework | Simplest thing that works; easy to debug |
| **State** | Stateless between runs | No database to maintain; each run is independent |
| **LLM** | Claude 3.5 Sonnet for all calls | Best balance of quality, context window, and cost |
| **LLM Calls** | 7 per run | 1 ranking + 5 deep + 1 blurb batch |
| **PDF Extraction** | PyMuPDF primary, HTML fallback | Reliable text from any paper format |
| **Email** | Jinja2 template + Gmail SMTP | Simple, free, good enough for <50 subscribers |
| **Scheduling** | GitHub Actions cron | Free, zero-infrastructure, auto-retry |
| **Error Handling** | Error email + GH Actions logs | User always knows if something went wrong |
| **Estimated Cost** | ~$0.30–0.60/day | Well under $1/day budget |
| **Estimated Runtime** | ~4-7 minutes | Well under 15-minute budget |

