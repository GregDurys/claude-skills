# claude-skills

A collection of Claude skills for everyday tasks. Each skill is self-contained so you can cherry-pick what you need - adopt one, a few, or all of them. Where applicable, skills follow a free-first cost principle (use free tools and free tiers before paid ones). More skills will be added over time.

## Skills

| Skill | Purpose | One-line summary |
|-------|---------|------------------|
| [web-research](skills/web-research/SKILL.md) | Generic baseline | Tiered web search and page fetching with free-first cost logic |
| [reddit-research](skills/reddit-research/SKILL.md) | Reddit | Find threads and retrieve structured thread content including comments |
| [glassdoor-research](skills/glassdoor-research/SKILL.md) | Company research | Pull reviews, ratings, and interview experiences from Glassdoor |
| [linkedin-job-search](skills/linkedin-job-search/SKILL.md) | Job hunting | Search LinkedIn listings, scrape JDs, and triage against a configurable profile |
| [cve-researcher](skills/cve-researcher/SKILL.md) | Security research | Pull and analyse CVEs for any vendor or product via NIST NVD + CISA KEV, with optional VulnCheck cross-reference |

See [docs/DESIGN.md](docs/DESIGN.md) for the design rationale and [docs/TESTING.md](docs/TESTING.md) for live test results.

---

## Dependencies

All three paid MCPs have meaningful free tiers. For typical personal usage you can run every skill in this repo without paying anything.

### Pricing at a glance

| Dependency | Free tier | Source |
|------------|-----------|--------|
| [FireCrawl](https://firecrawl.dev) (MCP) | 500 credits one-time, no card | [firecrawl.dev/pricing](https://firecrawl.dev/pricing) |
| [Bright Data MCP](https://brightdata.com/pricing/mcp-server) | 5,000 requests/month (new MCP users) | [brightdata.com/pricing/mcp-server](https://brightdata.com/pricing/mcp-server) |
| Brave MCP | Brave Search API free tier + you self-host the MCP wrapper | [brave.com/search/api](https://brave.com/search/api) |
| Built-in `web_search` / `web_fetch` | Free, unlimited | Provided by Claude |
| NIST NVD API 2.0 (used by cve-researcher) | Free, no key required (optional key for higher rate limits) | [nvd.nist.gov/developers](https://nvd.nist.gov/developers/request-an-api-key) |
| CISA KEV (used by cve-researcher) | Free, no key | [cisa.gov/known-exploited-vulnerabilities-catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) |
| VulnCheck NVD++ (used by cve-researcher) | Free community tier, key required | [vulncheck.com/community](https://vulncheck.com/community) |

Each skill lists its own dependencies at the top of its `SKILL.md`. The full matrix across all four:

| MCP | Free tier | Paid pricing | Sign-up / setup | Used by |
|-----|-----------|--------------|-----------------|---------|
| Built-in `web_search` / `web_fetch` | Free, unlimited | N/A | No setup - provided by Claude | web-research, reddit-research, glassdoor-research, linkedin-job-search (all as fallback) |
| [FireCrawl](https://firecrawl.dev) | **500 credits one-time** (no card required) | Hobby $16/mo (3K credits), Standard, Growth, Scale | Sign up at firecrawl.dev, get API key, configure the MCP endpoint. Search: 2 credits/10 results. Basic scrape: 1 credit/page. Stealth scrape: 5 credits/page. | web-research (optional), reddit-research (optional, discovery only), glassdoor-research (required), linkedin-job-search (required) |
| [Bright Data MCP](https://brightdata.com/pricing/mcp-server) | **5,000 requests/month** (new MCP users) | Pay-as-you-go $1.50/1K results, or $499/mo for 380K | Sign up at brightdata.com. One-time setup: run `API_TOKEN=<token> PRO_MODE=true npx -y @brightdata/mcp` locally to provision `mcp_unlocker` and `mcp_browser` zones. | web-research (optional), reddit-research (required), linkedin-job-search (required) |
| Brave MCP | Free (Brave Search API free tier) | Self-hosted - you run the MCP wrapper. Brave Search API itself has a free tier at brave.com/search/api. | No public MCP exists. Get a Brave Search API key at https://brave.com/search/api and run your own MCP wrapper (see https://github.com/brave/brave-search-mcp or a community fork). Skip if you don't self-host. | All skills as optional Tier 1 free search |

### Try everything for free

Both FireCrawl and Bright Data free tiers cover typical personal use:

- **FireCrawl 500 credits** = ~500 basic page scrapes, ~100 Glassdoor stealth scrapes, ~2,500 search results, or any mix.
- **Bright Data 5K requests/month** = enough for a weekly LinkedIn job run plus ongoing Reddit fetching with room to spare.

For a regular weekly LinkedIn job search + occasional Glassdoor / Reddit lookups, these two free tiers together are sufficient without ever paying.

### Minimum set for each skill

If you want only one skill, here's the minimum working dependency set (all have free tiers):

| Skill | Minimum dependencies | Free-tier viable? |
|-------|----------------------|-------------------|
| web-research | none required (degrades to built-in tools) | yes - no sign-up needed |
| reddit-research | Bright Data MCP | yes - 5K requests/month free |
| glassdoor-research | FireCrawl | yes - 500 credits one-time free |
| linkedin-job-search | FireCrawl + Bright Data MCP | yes - both have free tiers |
| cve-researcher | Python 3 (no MCP needed - calls NVD and CISA public APIs directly) | yes - no sign-up needed for base script; VulnCheck cross-reference needs free key |

Brave is always optional.

### Brave MCP caveat

There is no public Brave MCP. You would need to self-host one (using the Brave Search API and a community or official MCP wrapper) and configure Claude to reach it. If you self-host on a home machine behind a tunnel (e.g., Tailscale Funnel), Brave is only reachable while that machine is online - all skills treat Brave failures as silent skips and degrade to paid tiers.

If you don't intend to self-host Brave, skip it entirely. All skills work on the other MCPs alone.

---

## Configuring MCPs

Connect the MCP servers before installing the skills. Each MCP needs to be configured once in your Claude client.

### Claude Code (CLI)

Add each MCP via `claude mcp add`. Remote endpoints are the simplest; local via `npx` gives you more control.

**FireCrawl** - sign up at https://firecrawl.dev/app to get an API key:

```bash
# Remote hosted (recommended)
claude mcp add firecrawl --url https://mcp.firecrawl.dev/YOUR_API_KEY/v2/mcp

# Or local via npx
claude mcp add firecrawl -e FIRECRAWL_API_KEY=YOUR_API_KEY -- npx -y firecrawl-mcp
```

**Bright Data** - sign up at https://brightdata.com to get an API token (free tier: 5,000 requests/month):

```bash
# Remote hosted (recommended)
claude mcp add brightdata --url https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN

# Or local via npx (auto-provisions required zones on first run)
claude mcp add brightdata -e API_TOKEN=YOUR_API_TOKEN -e PRO_MODE=true -- npx -y @brightdata/mcp
```

**Brave** (optional, self-hosted only) - no public Brave MCP exists. Self-host a wrapper around the Brave Search API (https://brave.com/search/api) and point Claude Code at your endpoint:

```bash
claude mcp add brave --url https://your-brave-mcp-endpoint
```

See https://github.com/brave/brave-search-mcp for self-hosting options. Skip Brave entirely if you're not self-hosting - the skills degrade gracefully to built-in `web_search` and FireCrawl.

### Claude.ai (web)

Add MCP connectors via `Settings - Connectors`. Use the same endpoints:

| MCP | Endpoint |
|-----|----------|
| FireCrawl | `https://mcp.firecrawl.dev/YOUR_API_KEY/v2/mcp` |
| Bright Data | `https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN` |
| Brave | your self-hosted wrapper endpoint (optional) |

Verify each MCP shows as active in the connectors list before loading skills that depend on it.

---

## Installation

Claude skills are activated by placing `SKILL.md` files in a Claude-readable skills directory.

### Claude.ai (Projects)

Add each skill through the web UI - you don't place files on disk manually. In your Claude.ai Project settings, find the Skills / Capabilities section and upload each `SKILL.md`. The skill's identity comes from the `name:` field in its frontmatter, so the filename you upload doesn't matter - Claude uses the frontmatter value.

Claude handles storage internally.

### Claude Code

Place each skill's folder at `~/.claude/skills/<skill-name>/SKILL.md`:

```
~/.claude/skills/web-research/SKILL.md
~/.claude/skills/reddit-research/SKILL.md
~/.claude/skills/glassdoor-research/SKILL.md
~/.claude/skills/linkedin-job-search/SKILL.md
```

The directory name must match the `name:` field in each skill's frontmatter. Consult the current Claude Code documentation if this location has changed.

### Important caveat

Skills only take effect from the **next conversation** after you add them. The `<available_skills>` list in Claude's system prompt is set at conversation start - editing files mid-conversation won't cause them to load. Start a fresh chat after installing.

### Preset content (optional)

These skills are self-contained. Their frontmatter `description` fields handle auto-triggering, and each skill's body contains all the logic it needs (tier selection, validation rules, presentation format). You do **not** need any Claude.ai preset content to make them work.

If you previously used a preset that referenced an older unified `web-research` skill, or contained a LinkedIn job-listing validation rule, you can safely remove those preset lines - the four split skills auto-trigger, and the LinkedIn validation rule is now baked into `linkedin-job-search/SKILL.md` under "Validation discipline".

Cross-cutting preset rules that don't relate to these skills (formatting preferences, search-before-stating-facts, regulations-and-deadlines requirements, etc.) are independent and should stay in your preset.

---

## Before first use: customising linkedin-job-search

Unlike the other three skills, `linkedin-job-search` is a **framework** that needs customisation with your own profile before it produces useful output. Edit these placeholder markers in the SKILL.md:

| Marker | What it is | Example values |
|--------|-----------|----------------|
| `<TARGET_ROLES>` | Your target role titles | "Data Engineer", "Backend Developer", "Penetration Tester", "Product Designer" |
| `<TARGET_LOCATIONS>` | Where you want to work | "United Kingdom", "Remote EMEA", "London", "Berlin", "Anywhere in EU" |
| `<WORKPLACE_TYPE>` | Remote / hybrid / office preferences | "Fully remote only", "Remote or hybrid, max 2 days office", "Hybrid 3 days London" |
| `<SALARY_FLOOR>` | Minimum salary you'd accept | "GBP 70K+ permanent", "GBP 400+/day contract", "USD 150K+", "EUR 80K+" |
| `<EXCLUDED_ROLE_TYPES>` | Role types to always filter out | For an offensive-security practitioner: "SOC, GRC, IAM, pre-sales, AppSec without pen testing". For a backend engineer: "sales, marketing, product management, frontend-only". For a data scientist: "BI reporting, data entry, pure analytics roles". |
| `<INCLUDED_ROLE_TYPES>` | Role types to always keep | For an offensive-security practitioner: "pen testing, red team, exploit development, vulnerability research". For a backend engineer: "distributed systems, backend infrastructure, platform engineering". For a data scientist: "ML engineering, applied ML research, recommender systems". |

The other three skills need no customisation.

---

## Design principles

- **Free-first cost tiering.** Every skill orders its tools from free to paid. Skip-ahead is forbidden.
- **Self-contained skills.** Each `SKILL.md` works standalone. Grab one without adopting the others.
- **Site-specific fallbacks.** FireCrawl blocks LinkedIn and Reddit; Bright Data is the documented alternative.
- **Graceful degradation.** Skills degrade silently when optional MCPs (particularly Brave) are unavailable.

See [docs/DESIGN.md](docs/DESIGN.md) for the full rationale.

---

## Repository layout

```
claude-skills/
  README.md                              this file
  .gitignore
  docs/
    DESIGN.md                            design rationale and history
    TESTING.md                           live test cases and results
  skills/
    web-research/SKILL.md
    reddit-research/SKILL.md
    glassdoor-research/SKILL.md
    linkedin-job-search/SKILL.md
```

---

## License

The skills in this repo are documentation - prompts and workflows, not executable code. Use them as you see fit. No warranty is offered; MCP providers may change their APIs, pricing, and availability at any time.
