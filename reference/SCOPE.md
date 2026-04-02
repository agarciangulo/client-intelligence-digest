# Scope — ArXiv AI Paper Digest Agent

## Objective

Build an automated AI agent that scans daily arXiv publications in the `cs.AI` category, identifies the most relevant and impactful papers, and delivers a curated email digest featuring:

1. **Deep Dives (Top 5)** — Full paper analysis with 1,000–2,000 word accessible summaries
2. **Quick Reads (Next 5)** — Short blurbs with context on why they made the list

All entries include direct links to PDFs for further reading.

The agent runs daily via GitHub Actions with zero manual intervention.

---

## Core Pipeline

```
┌─────────────┐     ┌──────────────────┐     ┌───────────────────┐     ┌──────────────────┐     ┌──────────────┐
│  SCHEDULER   │────▶│  DATA COLLECTOR   │────▶│  STAGE 1: RANKER   │────▶│  STAGE 2: DEEP    │────▶│  EMAIL        │
│  (GH Actions │     │  (arXiv API)      │     │  (titles+abstracts)│     │  ANALYZER          │     │  COMPOSER &   │
│   daily cron)│     │  ~150-200/day     │     │  → top 10+         │     │  (full PDFs × 5)  │     │  SENDER       │
└─────────────┘     └──────────────────┘     └───────────────────┘     └──────────────────┘     └──────────────┘
```

### Stage Descriptions

| Stage | Input | Output | LLM? |
|-------|-------|--------|------|
| **Data Collector** | arXiv `cs.AI` daily listing | Titles, authors, abstracts, IDs, comments for all papers | No |
| **Stage 1: Ranker** | All papers (titles + abstracts) | Ranked top ~10–15 candidates with justifications | Yes |
| **Stage 2: Deep Analyzer** | Full PDF text for top 5 | 1,000–2,000 word lay-audience summaries per paper | Yes |
| **Email Composer** | Top 5 summaries + next 5 blurbs | Formatted HTML email with PDF links | No |
| **Email Sender** | Composed email + subscriber list | Delivered digest | No |

---

## User Profile & Interest Configuration

### Purpose

The user profile defines *what makes a paper "relevant"* to the subscriber(s). Without this, the LLM falls back to generic heuristics (novelty, citation potential, venue prestige), which may not reflect what the reader actually cares about.

### Profile Structure

```json
{
  "profile_name": "Andres — AI Research Digest",
  "role_context": "Technology professional interested in staying current with AI research, especially as it applies to real-world products, business problems, and the evolving competitive landscape of AI.",
  "primary_interests": [
    "LLM agents and agentic architectures",
    "AI-hardware co-design and integration (custom silicon, neuromorphic computing, on-device AI, AI accelerators)",
    "Proprietary and unique dataset strategies as competitive moats",
    "AI applied to business processes and productivity",
    "Multi-agent systems and coordination",
    "AI reasoning and chain-of-thought improvements"
  ],
  "secondary_interests": [
    "Retrieval-Augmented Generation (RAG)",
    "AI safety and alignment",
    "Novel training techniques and fine-tuning methods",
    "AI benchmarks and evaluation methodologies",
    "Knowledge graphs and structured knowledge"
  ],
  "wildcard_interest": "Any paper that proposes a genuinely novel paradigm, unconventional approach, or potential breakthrough — even if it falls outside the listed interests. Flag these with a note explaining why they're worth attention despite not matching the profile.",
  "deprioritize": [
    "Pure computer vision with no AI/LLM connection",
    "Narrow medical/biology AI applications",
    "Non-English language-specific NLP",
    "Incremental benchmark improvements without methodological novelty"
  ],
  "prioritized_sources": {
    "industry_labs": [
      "OpenAI",
      "Google DeepMind",
      "Anthropic",
      "Meta AI (FAIR)",
      "Microsoft Research",
      "NVIDIA Research",
      "Mistral AI",
      "xAI"
    ],
    "universities": [
      "MIT",
      "Stanford",
      "Harvard",
      "Carnegie Mellon (CMU)",
      "UC Berkeley",
      "Princeton",
      "Caltech",
      "University of Toronto",
      "Tsinghua University"
    ],
    "research_organizations": [
      "Allen Institute for AI (AI2)",
      "MILA",
      "EleutherAI"
    ],
    "usage_note": "Papers from these sources get a ranking boost, but a brilliant paper from an unknown lab should still surface. Source is a tiebreaker, not a gatekeeper."
  },
  "preference_signals": {
    "favor_accepted_venues": true,
    "venues_of_interest": ["ICLR", "NeurIPS", "ICML", "AAAI", "ACL", "EMNLP"],
    "favor_practical_applications": true,
    "favor_novel_methods_over_incremental": true,
    "accessibility_preference": "Prefer papers that introduce concepts clearly, even if technically deep"
  }
}
```

---

## Ranking Criteria

The Stage 1 Ranker uses the user profile plus these general quality signals to score papers:

### Quality Signals (in priority order)

| Signal | Weight | Source | Description |
|--------|--------|--------|-------------|
| **Topic relevance** | Highest | User profile | Does it match primary or secondary interests? |
| **Novelty** | High | Abstract analysis | New method/framework vs. incremental improvement? |
| **Practical applicability** | High | Abstract analysis | Can this be applied in industry, or is it purely theoretical? |
| **Venue acceptance** | Medium | Comments field | Accepted at a top conference (ICLR, NeurIPS, etc.)? |
| **Breadth of impact** | Medium | Abstract analysis | Applicable across domains, or very narrow? |
| **Clarity of contribution** | Low | Abstract analysis | Is it clear what the paper achieves? |
| **Author/institution signal** | Medium | Author field | From a prioritized source? (See profile — used as a boost, not a gatekeeper) |
| **Wildcard / paradigm shift** | Medium | Abstract analysis | Does it propose something genuinely new, even if outside listed interests? |

### Scoring Approach

The LLM receives:
- The full user profile (interests, deprioritizations, preferences)
- All paper titles + abstracts for the day
- Instructions to return a ranked top 10–15 with brief justifications

The ranking is qualitative (LLM judgment), not a numeric scoring formula. The justifications serve as an audit trail for why each paper was selected.

---

## Summary Format

### Top 5 — Deep Dive Summaries (1,000–2,000 words each)

Each summary should:

1. **Open with the "So What?"** — One paragraph explaining why this paper matters, in plain language. What problem does it solve? Why should a busy professional care?

2. **Explain the Core Idea** — The key insight or method, described accessibly. Use analogies where helpful. Assume the reader is intelligent but not a specialist in this sub-field.

3. **How It Works** — A clear walkthrough of the approach. Not a full technical deep dive, but enough that the reader understands the mechanism. Skip heavy math; explain the intuition.

4. **Key Results** — What did they find? How does it compare to previous work? Include specific numbers where they tell a compelling story.

5. **Limitations & Open Questions** — Honest assessment. What doesn't the paper address? What would need to happen for this to be practically useful?

6. **Real-World Applications & Opportunities** — Concrete examples of how this research could be applied in industry, products, or business. What opportunities does it open up? Who should be paying attention to this, and why?

### Next 5 — Quick Blurbs (~100–150 words each)

Each blurb should:
- State the core contribution in 1–2 sentences
- Explain why it's noteworthy
- Include a "Read this if you care about: [topic]" tag

### All 10 Entries Include

- Paper title
- Authors (first 3 + "et al." if more)
- Link to PDF (`https://arxiv.org/pdf/{arxiv_id}`)
- Link to HTML version (`https://arxiv.org/html/{arxiv_id}`)
- Venue acceptance note (if applicable, from comments field)

---

## Email Format

### Structure

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📰 ArXiv AI Digest — {Date}
   {N} papers scanned · Top 10 curated for you
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📌 TODAY'S TOP 5 — DEEP DIVES

1. {Paper Title}
   {Authors} | {Venue if applicable}
   📄 PDF | 🌐 HTML

   {1,000–2,000 word summary}

   ─────────────────────────────────

2. {Paper Title}
   ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 ALSO WORTH YOUR ATTENTION (#6–10)

6. {Paper Title}
   {Authors} | 📄 PDF | 🌐 HTML
   {100–150 word blurb}

7. ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 Today's Stats
   Papers scanned: {N}
   Categories covered: cs.AI + cross-listings
   Profile: {profile_name}

🔧 To update your interests, reply to this email.
```

---

## Subscriber Management

### MVP (Now)

- Subscriber list stored in a JSON config file
- Single profile (shared across all subscribers)
- Adding a subscriber = editing the config file

### Future Enhancements

- Per-subscriber interest profiles
- Unsubscribe link in emails
- Frequency preferences (daily, weekly digest)
- Multiple arXiv category support

### Subscriber Schema

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

---

## Success Criteria

| Category | Criteria |
|----------|----------|
| **Data Collection** | All daily `cs.AI` papers (including cross-listings) are fetched reliably |
| **Ranking Quality** | Top 10 papers consistently match user interests; irrelevant papers are filtered out |
| **Summary Quality** | Deep dive summaries are accurate, accessible, and capture the paper's core contribution |
| **Summary Length** | Top 5 summaries are 1,000–2,000 words; blurbs are 100–150 words |
| **Email Delivery** | Digest arrives daily at a consistent time; formatting renders correctly |
| **Reliability** | Agent runs unattended via GitHub Actions without failures for 7+ consecutive days |
| **Cost** | Total daily LLM cost under $1.00 |
| **Latency** | Full pipeline completes within 15 minutes |

---

## Constraints & Assumptions

| Constraint | Details |
|------------|---------|
| **Data source** | arXiv `cs.AI` category only (expandable later) |
| **PDF access** | arXiv PDFs are freely available; rate-limit-friendly fetching (~3s between requests) |
| **LLM provider** | Anthropic (Claude) for deep analysis; optionally cheaper model for Stage 1 ranking |
| **Email delivery** | Gmail SMTP with App Password |
| **Scheduling** | GitHub Actions cron (daily) |
| **Subscribers** | Shared interest profile across all subscribers (per-subscriber profiles are future) |
| **Language** | English-language papers only |
| **Volume** | ~150–200 papers/day typical; system should handle up to ~500 without architectural changes |

---

## Out of Scope (Deferred)

- Web UI or dashboard
- Per-subscriber interest profiles
- Historical paper database or search
- Paper recommendation based on reading history
- Slack/Teams integration
- Non-`cs.AI` categories (though architecture should make this easy to add)
- Citation network analysis
- Full-text search across past digests

---

## Resolved Design Decisions

### Scheduling & Timing

arXiv publishes new papers at **8:00 PM US Eastern Time (ET)**, Monday through Friday only. No publications on weekends or US holidays.

**Digest schedule:** Run daily at **6:00 AM ET the following morning** (Tuesday through Saturday), covering the previous evening's complete publication.

| arXiv publishes... | Digest runs... | Covers... |
|---|---|---|
| Monday 8 PM ET | Tuesday 6 AM ET | Monday's papers |
| Tuesday 8 PM ET | Wednesday 6 AM ET | Tuesday's papers |
| Wednesday 8 PM ET | Thursday 6 AM ET | Wednesday's papers |
| Thursday 8 PM ET | Friday 6 AM ET | Thursday's papers |
| Friday 8 PM ET | Saturday 6 AM ET | Friday's papers |

**GitHub Actions cron (UTC):** `0 11 * * 2-6` (11:00 UTC = 6:00 AM ET, Tuesday–Saturday)

### No-Paper Days (Holidays)

If the pipeline runs but finds no new papers (e.g., US holiday), send a brief **"quiet day" notice** rather than skipping the email entirely. This confirms the system is running and sets expectations.

### Duplicate Handling

Papers that appear in multiple daily listings (cross-listings) are **deduplicated by arXiv ID**. Each paper is processed only once regardless of how many categories it appears in.

### Error Handling

If the pipeline fails at any stage, the system sends an **error notification email** to the subscriber list with:
- Which stage failed
- Error message / traceback summary
- Timestamp
- Note that the digest will retry on the next scheduled run

---

## Next Steps

1. ✅ Scope document finalized
2. Create `ARCHITECTURE.md` with detailed component design and prompt specifications
3. Create `TECH_STACK.md` with all dependencies and environment variables
4. Create `EXECUTION_PLAN.md` with step-by-step build plan
5. Begin implementation

