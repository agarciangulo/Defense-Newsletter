# Build Summary — Client Intelligence Digest Agent

A technical summary of the project for reference when writing about the build.

---

## What It Does

An automated agent that monitors two types of public sources every week:

1. **Client-specific sources** — Each tracked client's own media pages (press releases, news mentions, feature stories). These are auto-tagged to the client.
2. **General industry sources** — Broad publications like Defense News where any tracked client might be mentioned. These are keyword-matched against a client registry.

Both paths converge into a unified set of client-tagged articles. An LLM then summarizes what each client has been up to (organized by project) and selects the week's top highlights. The result is a curated email newsletter delivered every Monday morning — fully automated, zero manual intervention.

The digest has two tiers:

- **Relevant Highlights (Top Section)**: The 3–5 most significant developments across all clients, each with a brief analytical paragraph explaining strategic implications.
- **Per-Client Sections**: Organized by client, then by project. Each project section has a 100–200 word narrative summary plus direct links to every source article. Source origin is noted (client media vs. industry publication).

---

## The Pipeline

The system is a **linear pipeline with a dual-path front end** — no databases (beyond a state file), no servers, no queues.

```
GitHub Actions (7 AM ET, every Monday)
    │
    ▼
1a. CLIENT-SPECIFIC SCRAPER
    │  For each client: scrape their media pages (HTML)
    │  Auto-tag all articles to that client
    │
1b. GENERAL SOURCE SCRAPER
    │  Scrape industry publications (RSS + HTML)
    │  Articles are untagged at this point
    ▼
2.  KEYWORD MATCHER (general sources only)
    │  Scan general articles for client/project keywords
    │  Tag matches to clients; discard unmatched
    ▼
3.  MERGE & DEDUPLICATE
    │  Combine both paths → group by client
    ▼
4a. PER-CLIENT SUMMARIZER — For each client with articles:
    │  Send all articles to Claude → per-project summaries
    │  For client-source articles: LLM categorizes by project
    │  For general-source articles: already keyword-matched
    │
4b. HIGHLIGHT SELECTOR — Select 3-5 top developments
    ▼
5.  EMAIL COMPOSER — Jinja2 HTML template
    ▼
6.  EMAIL SENDER — Gmail SMTP
    ▼
7.  STATE UPDATE — Record processed URLs → commit to repo
```

Total runtime: **~2–6 minutes**. Total weekly cost: **~$0.50–1.50**.

---

## Key Technical Decisions (and Why)

### Dual-Path Source Architecture

The most important design decision. Sources fall into two fundamentally different categories:

- **Client-specific sources** are each client's own media pages. We already know who the articles belong to — scraping them auto-tags to that client. Example: Draper's News Releases, In the News, and Featured Stories pages.
- **General industry sources** are publications like Defense News that cover the entire industry. Articles from these need to be matched against the client registry to find relevant mentions.

Both paths produce the same `Article` objects with the same fields. After tagging, everything downstream is path-agnostic. The merge point combines articles from both paths, deduplicates by URL, and groups by client.

### HTML Scraping with Per-Source Selectors

Client-specific sources are almost always HTML pages with unique structures. Rather than hard-coding selectors per source, the config file defines CSS selectors for each source. This means adding a new client source is a config change, not a code change. General sources that have RSS feeds use `feedparser` instead.

### Keyword Matching Only on General Sources

Client-specific source articles don't need matching — they're auto-tagged by definition. The keyword matcher only runs on general-source articles, using Python's `re` module with word-boundary matching. This is fast, free, and deterministic.

### LLM Does Double Duty for Client Sources

For general-source articles, keyword matching identifies which project they relate to before they reach the LLM. But client-source articles arrive without project tags. The per-client summarizer handles this: it receives the client's project definitions and categorizes articles into the right projects as part of the summarization call. One LLM call per client handles both categorization and summarization.

### Source Origin Tracking

Every article carries a `source_path` field ("client" or "general"). This flows through the entire pipeline and appears in the email. Industry publication mentions often carry more weight than self-published client content, so the highlight selector uses this signal when choosing top developments.

### Shared LLM Client with Retry Logic

All Claude API calls go through a single `call_claude()` function with exponential backoff and jitter. Identical to the ArXiv digest system's approach.

### State File for Cross-Week Deduplication

A JSON file (`state/processed.json`) committed to the repository after each run tracks processed article URLs. Entries older than 90 days are pruned automatically.

### Gmail SMTP — Simplest Thing That Works

Email delivery uses Python's built-in `smtplib` with a Gmail App Password. Zero cost, minimal code.

---

## Architecture at a Glance

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Language | Python 3.12 | Richest scraping/AI ecosystem |
| LLM | Claude Sonnet | Good summarization quality at reasonable cost |
| RSS parsing | feedparser | Handles all feed formats, tolerant of malformed XML |
| HTML scraping | requests + BeautifulSoup | Lightweight, config-driven selectors |
| Keyword matching | Python `re` (word-boundary) | Fast, free, deterministic |
| Email | Gmail SMTP (stdlib) | Zero cost, minimal code |
| Scheduling | GitHub Actions cron | Free, zero infrastructure |
| State | JSON file in repo | Zero infrastructure; git-native |
| Orchestration | Plain Python functions | Pipeline is linear — no framework needed |
| Config | JSON files | Client registry + sources + subscribers |

---

## Numbers

| Metric | Value |
|--------|-------|
| Source types | 2 (client-specific HTML + general RSS/HTML) |
| LLM calls per run | Variable (1 per client + 1 highlights) |
| Per-project summary length | 100–200 words |
| Highlight analysis length | 2–3 sentences each |
| Pipeline runtime | ~2–6 minutes |
| Weekly cost | ~$0.50–1.50 |
| Monthly cost | ~$2–6 |
| Lines of Python | ~990 (excluding tests and templates) |
| Runtime dependencies | 6 |
| External services | 3 (GitHub Actions, Anthropic, Gmail) |
| Infrastructure to maintain | None |

---

## The Build Process

8 phases following "Build → Verify → Advance":

1. **Project setup** — Structure, config files, logging, dev cache
2. **Source scraper** — Dual-path: client-specific HTML + general RSS/HTML, rate limiting, state filtering
3. **Keyword matcher** — Word-boundary matching on general sources only + merge logic
4. **State manager** — Load, update, prune, save, git commit
5. **LLM stages** — Per-client summarizer (with project categorization), highlight selector
6. **Email system** — Jinja2 templates with source origin badges, SMTP sender
7. **Pipeline orchestrator** — Dual-path wiring, DRY_RUN mode, error handling
8. **GitHub Actions deployment** — Weekly cron, state commit step, secrets

---

## Interesting Design Challenges

### Two Paths, One Pipeline

The dual-path model is the defining architectural feature. The key insight: client-specific sources and general sources are fundamentally different at the scraping stage (auto-tagged vs. needs matching) but produce identical outputs (tagged articles). The merge point is clean — after it, the pipeline doesn't know or care where an article came from.

### The "In the News" Hybrid

Some client pages (like Draper's "In the News") are curated lists of third-party articles about the client. These are client-specific sources by configuration, but they link to external content. The scraper handles this by extracting the article metadata (title, date, external URL) from the client's page. If the same article also appears in a general source RSS feed, the deduplication step catches it.

### Per-Source CSS Selectors

Every HTML source has a different page structure. The config-driven selector approach means each source defines its own CSS selectors (article list container, title, link, date, body). Adding a new client source requires inspecting the HTML and adding selectors — but no code changes.

### Project Categorization Without Keyword Matching

Client-source articles arrive without project tags. Rather than running keyword matching on them (which would be redundant — we already know the client), the LLM categorizer assigns them to projects during the summarization call. This is more accurate than keyword matching because the LLM understands context, and it doesn't add an extra API call.

---

## What's Not Built (Yet)

- LLM-powered semantic matching on general sources
- Per-subscriber client filtering
- Sentiment analysis on mentions
- Historical trend tracking or dashboards
- Slack/Teams delivery
- Real-time alerts for high-priority mentions
- Competitor monitoring
- JavaScript-rendered page scraping (would need Playwright)
- Source health monitoring

The architecture supports all of these as incremental additions. The dual-path model, in particular, makes it easy to add new source types without affecting the downstream pipeline.

---

## Project Structure

```
client-intelligence-digest/
├── .github/workflows/
│   └── weekly_digest.yml          # GitHub Actions cron job
├── config/
│   ├── clients.json               # Client registry + client sources + projects
│   ├── general_sources.json       # Industry-wide sources
│   └── subscribers.json           # Email recipient list
├── state/
│   └── processed.json             # Tracked processed articles
├── src/
│   ├── main.py                    # Pipeline orchestrator
│   ├── scraper.py                 # Dual-path: client-specific + general scraping
│   ├── matcher.py                 # Keyword Matcher (general sources only)
│   ├── summarizer.py              # LLM: Per-client summaries + project categorization
│   ├── highlights.py              # LLM: Highlight selection
│   ├── llm.py                     # Shared Claude client with retry logic
│   ├── email_composer.py          # Jinja2 template rendering
│   ├── email_sender.py            # Gmail SMTP delivery
│   ├── state_manager.py           # State file management
│   ├── error_handler.py           # Error capture + notification
│   ├── logger.py                  # Centralized logging
│   └── dev_cache.py               # Development caching utility
├── templates/
│   ├── digest_email.html          # Main digest email template
│   └── quiet_week_email.html      # "No content this week" template
├── tests/                         # Unit tests for all modules
├── requirements.txt
└── .env.example
```
