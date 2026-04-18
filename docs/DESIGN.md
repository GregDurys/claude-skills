# Design

Design rationale, history, and tool-selection notes for the skills in this repo. Read this after the README to understand *why* each skill is shaped the way it is.

---

## Purpose

Four Claude skills covering everyday web research tasks:

1. **web-research** - generic tiered search and fetch, the baseline anyone needs.
2. **reddit-research** - finding and reading Reddit threads (including comments) despite Reddit's aggressive anti-scraping.
3. **glassdoor-research** - pulling company reviews, ratings, and interview experiences despite Glassdoor's aggressive bot detection.
4. **linkedin-job-search** - searching LinkedIn job listings, scraping JDs, and triaging results against a configurable profile - despite LinkedIn being blocked by most general-purpose scrapers.

All four are built on the same tiered, free-first, cost-aware philosophy.

---

## History

The skills went through two iterations:

### Iteration 1 (early 2026): three separate Apify-based skills

The original set was:

- `linkedin-job-search` - Apify `harvestapi/linkedin-job-search` actor
- `fetch-with-fallback` - Brave MCP + built-in web_fetch + Bright Data
- `glassdoor-research` - Apify `getdataforme/glassdoor-reviews-scraper` actor

Problems:

- Apify was paid per-result with startup fees on every run.
- Brave tooling was unreliable (self-hosted, rate-limited, evolving tool set).
- Bright Data was used as fallback in multiple places with slightly different logic.

### Iteration 2 (mid-2026): unified web-research skill

All three were merged into one `web-research` skill, replacing Apify with FireCrawl + Bright Data:

- FireCrawl replaced Apify for anything outside LinkedIn/Reddit (cheaper per-result, no startup fee, returns cleaner URL structures, supports Google search operators).
- Bright Data retained for LinkedIn and Reddit because FireCrawl explicitly blocks both.
- Brave kept as the free Tier 1 for search.

This worked but hit a discoverability problem: a single 442-line skill is hard to share on GitHub. People adopting individual workflows couldn't cherry-pick.

### Iteration 3 (this repo): split back into four skills, sanitised for publication

The current set splits the merged skill back into four purpose-specific skills:

- `web-research` - generic baseline (search + fetch tier logic from the merged skill's Parts B + E)
- `reddit-research` - extracted from Part D, now standalone since Bright Data's `web_data_reddit_posts` is the clear primary tool
- `glassdoor-research` - extracted from Part C
- `linkedin-job-search` - extracted from Part A, rebuilt as a profile-agnostic framework for public consumption

All skills were sanitised to remove personal profile data from the original author's private version. The `linkedin-job-search` skill ships as a template that users customise with their own role targets, exclusion rules, and salary thresholds.

---

## Design principles

### Cost-tiered tool selection

Every skill orders its tools free-to-paid. Tier 1 is free (Brave MCP, built-in `web_search`/`web_fetch`); Tier 2/3 are paid (FireCrawl, Bright Data). Skip-ahead is forbidden. A 429 at Tier 1 falls to Tier 2, not straight to Tier 3.

The cost principle matters: a typical weekly LinkedIn job run at Tier 1 + Tier 2 is a few dozen FireCrawl credits plus a handful of Bright Data per-request calls. Skipping tiers would multiply that unnecessarily.

### Site-specific fallbacks

- **FireCrawl blocks LinkedIn and Reddit.** This is hardcoded in FireCrawl - no amount of proxy tuning works. Bright Data is the documented workaround. Every skill that touches those sites documents this and routes accordingly.
- **Bright Data's `web_data_reddit_posts` returns structured JSON** and is preferred over `scrape_as_markdown` for Reddit because it surfaces the `archived` field, numeric counts, and clean author info.
- **Glassdoor works with FireCrawl under `proxy: "stealth"`** but falls to Bright Data if that fails.

### Self-contained skills

Each SKILL.md is standalone. A user who wants only the Glassdoor workflow should be able to grab `skills/glassdoor-research/SKILL.md` and not need the others. Some tier logic is therefore duplicated across skills - this is deliberate, not an oversight.

### Free by default, paid by exception

All four skills degrade gracefully when paid MCPs are absent:

- `web-research` still fetches and searches using only built-in tools.
- `reddit-research` requires Bright Data for thread content (no free alternative exists). Search works free via built-in web_search if Brave is absent.
- `glassdoor-research` requires FireCrawl for reliable content (basic `web_fetch` always hits a login wall). One paid dependency is unavoidable here.
- `linkedin-job-search` requires both FireCrawl (for search) and Bright Data (for scraping). Two paid dependencies is the minimum viable combination.

Both paid MCPs offer meaningful free tiers, so "required" does not mean "requires payment":

- **FireCrawl:** 500 credits free one-time (no card required). Enough for ~500 basic scrapes, ~100 Glassdoor stealth scrapes, or ~2,500 search results.
- **Bright Data MCP:** 5,000 requests per month free for new MCP users. Covers search, scrape, web unlock, and structured data extraction.

For typical personal use (a weekly LinkedIn job search + occasional Glassdoor and Reddit lookups), the combined free tiers are sufficient indefinitely.

### Rate-limit awareness

Brave MCP has a hardcoded 1 req/sec rate limit. Every skill that uses Brave fires at most one call per response and falls back to paid tiers for further searches within the same turn.

---

## Tool selection notes

### Why FireCrawl over alternatives

- Supports the full Google search operator set (`site:`, `intitle:`, `inurl:`, `"..."`, `-`, etc.) AND the `tbs` time filter (`sbd:1,qdr:w` etc.) - essential for recent-only job searches.
- Cheap enough at scale (2 credits per 10 search results, 1 credit per basic scrape) to use as a default rather than a fallback.
- Stealth proxy mode (+4 credits) clears Glassdoor's bot detection reliably.
- JSON extraction mode exists for structured output if needed, though the skills use markdown by default for flexibility.
- Has a generous free tier (500 credits one-time, no card) sufficient to trial every skill in this repo without paying.

### Why Bright Data for LinkedIn and Reddit

- Structured tools like `web_data_reddit_posts` return clean JSON schemas rather than messy markdown.
- `scrape_as_markdown` bypasses login walls and bot detection on sites where FireCrawl is blocked.
- Per-request billing means no idle cost between uses.
- Bright Data MCP free tier (5,000 requests per month for new users) covers typical personal use - a weekly LinkedIn run plus Reddit lookups easily fits.

### Why Brave as optional free tier

- Truly free at the Brave Search API layer (https://brave.com/search/api has a free tier).
- Good quality search results with useful snippets.
- There is no public remote Brave MCP, but the official `@brave/brave-search-mcp-server` package runs locally via npx - fine for Claude Code and Claude Desktop without any hosting. Claude.ai web needs an HTTP bridge, which is more work but doable via tunnels like Tailscale Funnel.
- Rate-limit (1 req/sec) caps throughput but is not a blocker for typical usage.

### Why built-in web_fetch / web_search remain in the mix

- Free, provided by Claude, no setup.
- `web_search` results carry URL-approval metadata that lets `web_fetch` operate on them directly.
- Both have a critical constraint documented in every skill: `web_fetch` rejects URLs that came from any other MCP, so discovery tool choice determines which fetch tool is usable downstream.

---

## URL approval constraint (explained)

Claude's built-in `web_fetch` enforces a security boundary. It will only fetch URLs that arrived via:

1. The user's own message.
2. The built-in `web_search` tool.

URLs found via Brave MCP, FireCrawl search, Bright Data search, or any other MCP are rejected. This shapes every skill:

- Discovery via Brave / FireCrawl / Bright Data -> fetch via FireCrawl or Bright Data.
- Discovery via built-in `web_search` or user input -> `web_fetch` is viable.

Skills document this inline so Claude doesn't burn attempts on rejected URL combinations.

---

## Blocked sites (confirmed by testing)

FireCrawl returns an explicit "we do not support this site" error on:

- linkedin.com
- reddit.com

No workaround exists inside FireCrawl. Bright Data is the documented alternative for both. See `docs/TESTING.md` for the actual test outputs.

---

## Skill customisation

- **web-research, reddit-research, glassdoor-research:** no customisation required. They work against any company, subreddit, or URL.
- **linkedin-job-search:** requires customisation of the Profile and Search Queries sections with your own target roles, locations, salary floor, and triage rules. The skill is a framework, not a fixed workflow. Users in different domains (software engineering, marketing, data science, security, etc.) should replace the offensive-security example terms with their own.

The `linkedin-job-search/SKILL.md` ships with placeholder markers (`<TARGET_ROLES>`, `<TARGET_LOCATIONS>`, `<SALARY_FLOOR>` etc.) that users edit before first use.

---

## Known limitations

- **Brave rate limit:** 1 req/sec hardcoded, affects multi-query workflows. Mitigated by falling back to FireCrawl for second+ queries in a turn.
- **FireCrawl blocks:** LinkedIn, Reddit. Other sites might also be blocked silently - try the scrape and fall back to Bright Data on failure.
- **Brave availability:** depends on your setup - Claude Code and Claude Desktop run it locally via npx so it's as reliable as the Brave API itself. Claude.ai web relies on an HTTP bridge / tunnel that can be offline. All skills degrade gracefully when Brave is unavailable.
- **Glassdoor region:** skills default to `.co.uk`. For US-pool reviews, use `.com`. Both use the same URL structure.
- **LinkedIn URL format:** always strip tracking parameters before scraping. The `linkedin.com/jobs/view/<jobId>/` base is canonical.

---

## Testing

See `docs/TESTING.md` for the live test suite, test cases, and results. Every tier documented in the skills has been confirmed working end-to-end.
