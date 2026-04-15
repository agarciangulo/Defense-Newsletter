# Client Intelligence Digest Agent

A weekly automated digest that monitors public sources for mentions of a set of tracked clients, synthesizes what actually moved, and emails a curated newsletter — no manual effort after setup.

> **Status: design-first, pre-build.** This repository currently contains the full specification, architecture, and execution plan. Code is forthcoming. I'm deliberately investing in planning before implementation — see [Why planning first](#why-planning-first) below.

## The problem

If you're tracking a portfolio of clients, staying informed means scanning the same handful of industry publications every week, cross-referencing their mentions against your own client list, and remembering which announcement connects to which engagement. It's the kind of work that *feels* like reading but is mostly lookup.

This agent automates the lookup so a human can spend their time on the read.

## What it will do

Every Monday at 7 AM ET, the agent:

1. **Scrapes client-specific sources** — each tracked client's own media pages (press releases, news, featured stories). These articles are auto-tagged to the client.
2. **Scrapes general industry sources** — broad publications like Defense News. These start untagged.
3. **Matches general articles to clients** via a keyword/LLM pass over a client registry.
4. **Merges and deduplicates** the two streams into a per-client view.
5. **Summarizes each client's week** using Claude — organized by project, with direct links to every source.
6. **Selects 3–5 top highlights** across all clients with brief strategic commentary.
7. **Sends the digest** via Gmail SMTP to a subscriber list.
8. **Commits an updated state file** back to the repo so already-processed URLs aren't reprocessed next week.

Zero servers, zero databases. A linear pipeline on GitHub Actions with a JSON state file as its only memory.

## Architecture at a glance

```
           CLIENT-SPECIFIC                GENERAL SOURCES
           (auto-tagged)                  (keyword/LLM matched)
                 │                                │
                 └────────────┬───────────────────┘
                              ▼
                       Merge & deduplicate
                              │
                              ▼
                 Claude: per-client summaries
                              │
                              ▼
                  Claude: top weekly highlights
                              │
                              ▼
                   HTML email → Gmail SMTP
                              │
                              ▼
                 State update (JSON, git-committed)
```

Full design in [ARCHITECTURE.md](ARCHITECTURE.md).

## Planning artifacts

The repo is currently a spec, not a codebase. If you want to understand the design without running anything:

| Document | What's in it |
|---|---|
| [SCOPE.md](SCOPE.md) | Goal, deliverables, constraints |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Components, data flows, prompt specs |
| [TECH_STACK.md](TECH_STACK.md) | Every library, version, and why |
| [EXECUTION_PLAN.md](EXECUTION_PLAN.md) | Phase-by-phase build sequence with verification checkpoints |
| [SCRAPING_SPECS.md](SCRAPING_SPECS.md) | Per-source scraping rules |
| [PROMPTS.md](PROMPTS.md) | Full text of the LLM prompts |
| [BUILD_SUMMARY.md](BUILD_SUMMARY.md) | Condensed technical overview |

## Planned tech stack

- **Python 3.12** — primary language
- **GitHub Actions** — weekly cron
- **Anthropic Claude (Sonnet)** — summarization and highlight selection
- **feedparser + requests + BeautifulSoup** — scraping
- **Jinja2** — HTML email templating
- **Gmail SMTP** — delivery
- **JSON files + git** — state and configuration

No databases. No services. No orchestrators. Deliberately boring.

## Why planning first

I've built enough half-finished projects to know the failure mode: the interesting architectural choices happen *after* you start writing code, at which point you have to redo work to accommodate them. This repo is an experiment in front-loading the design — getting the dual-path source model, prompt structure, and state-handling strategy pinned down before committing to an implementation that would constrain them.

The artifacts here are meant to be readable by someone who isn't me. If any of them aren't, that's the signal the design isn't ready yet.

## Status

- ✅ Scope, architecture, tech stack, execution plan, and prompts specified
- 🛠️ Implementation: not started
- 🎯 Next: Phase 0 (project setup) per [EXECUTION_PLAN.md](EXECUTION_PLAN.md)

## Collaboration

If you're building something similar, or you want to collaborate on this build, I'd be happy to hear from you — reach out via [GitHub](https://github.com/agarciangulo).

## License

MIT — see [LICENSE](LICENSE).
