# claude-skills

A collection of Claude skills for everyday tasks. Each skill is self-contained, allowing individual adoption - one, a few, or all of them. Where applicable, skills follow a free-first cost principle (use free tools and free tiers before paid ones). More skills will be added over time.

## Skills

| Skill | Purpose | One-line summary |
|-------|---------|------------------|
| [web-research](plugins/web-research/skills/web-research/SKILL.md) | Generic baseline | Tiered web search and page fetching with free-first cost logic |
| [reddit-research](plugins/reddit-research/skills/reddit-research/SKILL.md) | Reddit | Find threads and retrieve structured thread content including comments |
| [glassdoor-research](plugins/glassdoor-research/skills/glassdoor-research/SKILL.md) | Company research | Pull reviews, ratings, and interview experiences from Glassdoor |
| [linkedin-job-search](plugins/linkedin-job-search/skills/linkedin-job-search/SKILL.md) | Job hunting | Search LinkedIn listings, scrape JDs, and triage against a configurable profile |
| [cve-researcher](plugins/cve-researcher/skills/cve-researcher/SKILL.md) | Security research | Pull and analyse CVEs for any vendor or product via NIST NVD + CISA KEV, with optional VulnCheck cross-reference |

See [docs/DESIGN.md](docs/DESIGN.md) for the design rationale and [docs/TESTING.md](docs/TESTING.md) for live test results.

---

## Dependencies

All three paid MCPs have meaningful free tiers. For typical personal usage, every skill in this repo can run without paying anything.

### Pricing at a glance

| Dependency | Free tier | Source |
|------------|-----------|--------|
| [FireCrawl MCP](https://github.com/firecrawl/firecrawl-mcp-server) | 500 credits one-time, no card | [firecrawl.dev/pricing](https://firecrawl.dev/pricing) |
| [Bright Data MCP](https://github.com/brightdata/brightdata-mcp) | 5,000 requests/month (new MCP users) | [brightdata.com/pricing/mcp-server](https://brightdata.com/pricing/mcp-server) |
| [Brave MCP](https://github.com/brave/brave-search-mcp-server) | Brave Search API free tier + runs locally via npx (Claude Code / Desktop) | [brave.com/search/api](https://brave.com/search/api) |
| Built-in `web_search` / `web_fetch` | Free, unlimited | Provided by Claude |
| NIST NVD API 2.0 (used by cve-researcher) | Free, no key required (optional key for higher rate limits) | [nvd.nist.gov/developers](https://nvd.nist.gov/developers/request-an-api-key) |
| CISA KEV (used by cve-researcher) | Free, no key | [cisa.gov/known-exploited-vulnerabilities-catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) |
| VulnCheck NVD++ (used by cve-researcher) | Free community tier, key required | [vulncheck.com/community](https://vulncheck.com/community) |

Each skill lists its own dependencies at the top of its `SKILL.md`. The full matrix across all four:

| MCP | Free tier | Paid pricing | Sign-up / setup | Used by |
|-----|-----------|--------------|-----------------|---------|
| Built-in `web_search` / `web_fetch` | Free, unlimited | N/A | No setup - provided by Claude | web-research, reddit-research, glassdoor-research, linkedin-job-search (all as fallback) |
| [FireCrawl MCP](https://github.com/firecrawl/firecrawl-mcp-server) | **500 credits one-time** (no card required) | Hobby $16/mo (3K credits), Standard, Growth, Scale | Sign up at firecrawl.dev, get API key, configure the MCP endpoint. Search: 2 credits/10 results. Basic scrape: 1 credit/page. Stealth scrape: 5 credits/page. | web-research (optional), reddit-research (optional, discovery only), glassdoor-research (required), linkedin-job-search (required) |
| [Bright Data MCP](https://github.com/brightdata/brightdata-mcp) | **5,000 requests/month** (new MCP users) | Pay-as-you-go $1.50/1K results, or $499/mo for 380K | Sign up at brightdata.com. One-time setup: run `API_TOKEN=<token> PRO_MODE=true npx -y @brightdata/mcp` locally to provision `mcp_unlocker` and `mcp_browser` zones. | web-research (optional), reddit-research (required), linkedin-job-search (required) |
| [Brave MCP](https://github.com/brave/brave-search-mcp-server) | Free (Brave Search API free tier) | Runs locally via npx (`@brave/brave-search-mcp-server`) in Claude Code and Claude Desktop - no hosting required. Claude.ai web would need an HTTP bridge. | Get a Brave Search API key at https://brave.com/search/api. See "Configuring MCPs" below for the exact Claude Code / Claude Desktop / Claude.ai setup. | All skills as optional Tier 1 free search |

### Try everything for free

Both FireCrawl and Bright Data free tiers cover typical personal use:

- **FireCrawl 500 credits** = ~500 basic page scrapes, ~100 Glassdoor stealth scrapes, ~2,500 search results, or any mix.
- **Bright Data 5K requests/month** = enough for a weekly LinkedIn job run plus ongoing Reddit fetching with room to spare.

For a regular weekly LinkedIn job search + occasional Glassdoor / Reddit lookups, these two free tiers together are sufficient without ever paying.

### Minimum set for each skill

For a single skill only, here is the minimum working dependency set (all have free tiers):

| Skill | Minimum dependencies | Free-tier viable? |
|-------|----------------------|-------------------|
| web-research | none required (degrades to built-in tools) | yes - no sign-up needed |
| reddit-research | Bright Data MCP | yes - 5K requests/month free |
| glassdoor-research | FireCrawl MCP | yes - 500 credits one-time free |
| linkedin-job-search | FireCrawl MCP + Bright Data MCP | yes - both have free tiers |
| cve-researcher | Python 3 (no MCP needed - calls NVD and CISA public APIs directly) | yes - no sign-up needed for base script; VulnCheck cross-reference needs free key |

Brave is always optional.

### Brave MCP caveat

There is no public remote Brave MCP. For **Claude Code** and **Claude Desktop** the official `@brave/brave-search-mcp-server` package runs locally via npx - no hosting required, just an API key. For **Claude.ai web** the MCP has to be reachable over HTTPS. One helper that bridges a local stdio MCP server to Streamable HTTP with OAuth / JWT is [`@ilities/local-ctx`](https://github.com/Ilities/local-ctx). TLS and public reachability then come from a tunnelling service of choice - e.g., [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/), [Tailscale Funnel](https://tailscale.com/kb/1223/funnel), and many others. Skip Brave on Claude.ai web if none of that is set up.

If the Brave API key is unavailable or not configured, skip Brave entirely. All skills treat Brave failures as silent skips and degrade to built-in `web_search` or FireCrawl. The skills work on the other MCPs alone.

---

## Configuring MCPs

Connect the MCP servers before installing the skills. Both Claude Code and Claude Desktop run MCPs locally via stdio; Claude.ai web uses HTTP / SSE connectors instead.

### Claude Code (CLI)

Add each MCP via `claude mcp add`. Remote endpoints are simplest where they exist; local via `npx` gives more control and is required for Brave (no public remote endpoint).

**FireCrawl** - sign up at https://firecrawl.dev/app for an API key:

```bash
# Remote hosted (recommended)
claude mcp add --transport http --scope user firecrawl https://mcp.firecrawl.dev/YOUR_API_KEY/v2/mcp

# Or local via npx
claude mcp add --scope user -e FIRECRAWL_API_KEY=YOUR_API_KEY firecrawl -- npx -y firecrawl-mcp
```

**Bright Data** - sign up at https://brightdata.com for an API token (free tier: 5,000 requests/month):

```bash
# Remote hosted (recommended) - quote the URL because of the ?token= query string
claude mcp add --transport http --scope user brightdata "https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN"

# Or local via npx (auto-provisions required zones on first run)
claude mcp add --scope user -e API_TOKEN=YOUR_API_TOKEN -e PRO_MODE=true brightdata -- npx -y @brightdata/mcp
```

**Brave** (optional) - no public remote Brave MCP. The official `@brave/brave-search-mcp-server` package runs locally via npx; get a free API key at https://brave.com/search/api:

```bash
claude mcp add --scope user -e BRAVE_API_KEY=YOUR_API_KEY brave -- npx -y @brave/brave-search-mcp-server
```

Skip Brave if the API key is not configured - the skills degrade to built-in `web_search` and FireCrawl.

### Claude Desktop (macOS / Windows)

Edit the config file and add entries under `mcpServers`:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "firecrawl": {
      "command": "npx",
      "args": ["-y", "firecrawl-mcp"],
      "env": {"FIRECRAWL_API_KEY": "YOUR_API_KEY"}
    },
    "brightdata": {
      "command": "npx",
      "args": ["-y", "@brightdata/mcp"],
      "env": {"API_TOKEN": "YOUR_API_TOKEN", "PRO_MODE": "true"}
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@brave/brave-search-mcp-server"],
      "env": {"BRAVE_API_KEY": "YOUR_API_KEY"}
    }
  }
}
```

Restart Claude Desktop after editing. Include only the required MCPs; remove entries for services not in use. Claude Desktop's native config is stdio-based - HTTP / SSE endpoints go through `Settings > Integrations` or a local bridge; stdio above is usually the simplest path.

### Claude.ai (web)

Add MCP connectors via `Settings > Connectors`. This route uses HTTP / SSE endpoints only:

| MCP | Endpoint |
|-----|----------|
| [FireCrawl MCP](https://github.com/firecrawl/firecrawl-mcp-server) | `https://mcp.firecrawl.dev/YOUR_API_KEY/v2/mcp` |
| [Bright Data MCP](https://github.com/brightdata/brightdata-mcp) | `https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN` |
| [Brave MCP](https://github.com/brave/brave-search-mcp-server) | No public HTTP endpoint. Reaching a local Brave MCP from Claude.ai web requires a stdio → HTTPS bridge (e.g. [`@ilities/local-ctx`](https://github.com/Ilities/local-ctx)) plus a tunnelling service of choice (e.g. [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/), [Tailscale Funnel](https://tailscale.com/kb/1223/funnel), and many others) for TLS and reachability. Skip if not self-hosting. |

Verify each MCP shows as active in the connectors list before loading skills that depend on it.

---

## Installation

This repo is structured as a Claude Code **plugin marketplace** (each skill is its own installable plugin). Three install paths are supported; pick by client.

### Claude Code (plugin marketplace, recommended)

Install individual skills from the marketplace:

```
/plugin marketplace add GregDurys/claude-skills
/plugin install web-research@claude-skills
/plugin install cve-researcher@claude-skills
```

Each skill installs independently. Run `/plugin install <name>@claude-skills` for each one needed. Updates arrive via `/plugin marketplace update`. Available plugin names: `web-research`, `reddit-research`, `glassdoor-research`, `linkedin-job-search`, `cve-researcher`.

### Claude Code (manual copy)

If the plugin marketplace path is unavailable, copy each skill folder into `~/.claude/skills/<skill-name>/`:

```
~/.claude/skills/web-research/SKILL.md
~/.claude/skills/reddit-research/SKILL.md
~/.claude/skills/glassdoor-research/SKILL.md
~/.claude/skills/linkedin-job-search/SKILL.md
~/.claude/skills/cve-researcher/SKILL.md
```

Copy from `plugins/<name>/skills/<name>/` in this repo. The directory name must match the `name:` field in each skill's frontmatter.

### Claude Desktop and Claude.ai (Projects)

Neither currently supports Claude Code's plugin marketplace protocol directly. For those clients:

- **Claude Desktop**: uses local stdio MCPs from `claude_desktop_config.json`; skills themselves are added through the web UI, since Desktop shares the claude.ai skill store.
- **Claude.ai (Projects)**: add each skill through the web UI. In Project settings, find the Skills / Capabilities section and upload each `SKILL.md` from this repo. The skill's identity comes from the `name:` field in its frontmatter, so the uploaded filename is irrelevant - Claude uses the frontmatter value. Claude handles storage internally.

### Important caveat

Skills only take effect from the **next conversation** after you add them. The `<available_skills>` list in Claude's system prompt is set at conversation start - editing files mid-conversation won't cause them to load. Start a fresh chat after installing.

### Preset content (optional)

These skills are self-contained. Their frontmatter `description` fields handle auto-triggering, and each skill's body contains all the logic it needs (tier selection, validation rules, presentation format). You do **not** need any Claude.ai preset content to make them work.

Any preset that previously referenced an older unified `web-research` skill or contained a LinkedIn job-listing validation rule can be safely removed - the four split skills auto-trigger, and the LinkedIn validation rule is now baked into `linkedin-job-search/SKILL.md` under "Validation discipline".

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
  README.md                                           this file
  .gitignore
  .claude-plugin/
    marketplace.json                                  plugin marketplace catalogue
  docs/
    DESIGN.md                                         design rationale and history
    TESTING.md                                        live test cases and results
  plugins/
    web-research/
      .claude-plugin/plugin.json                      per-plugin manifest
      skills/web-research/SKILL.md
    reddit-research/
      .claude-plugin/plugin.json
      skills/reddit-research/SKILL.md
    glassdoor-research/
      .claude-plugin/plugin.json
      skills/glassdoor-research/SKILL.md
    linkedin-job-search/
      .claude-plugin/plugin.json
      skills/linkedin-job-search/SKILL.md
    cve-researcher/
      .claude-plugin/plugin.json
      skills/cve-researcher/
        SKILL.md
        scripts/cve_pull.py                           NVD + CISA KEV pull
        scripts/vulncheck_pull.py                     VulnCheck cross-reference
        docs/TESTING.md                               skill-level test cases
```

---

## License

The skills in this repo are documentation - prompts and workflows, not executable code. Use them as you see fit. No warranty is offered; MCP providers may change their APIs, pricing, and availability at any time.
