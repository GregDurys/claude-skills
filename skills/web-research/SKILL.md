---
name: web-research
description: Tiered web search and page fetching using free tools first, then paid fallbacks. Use this skill whenever asked to fetch a URL, scrape a page, search the web, or "read this page". Uses Brave MCP and built-in web_search/web_fetch first (free), then FireCrawl and Bright Data as paid escalation. Handles JavaScript-rendered pages, bot-protected sites, and the URL approval boundary for built-in web_fetch.
metadata:
  author: Greg Durys
  version: '1.0'
---

# Web Research

Generic web search and fetch workflow with free-first cost logic. A baseline for any workflow that needs to discover URLs or retrieve page content.

## Dependencies

| MCP | Required? | Cost / free tier | Sign-up / setup |
|-----|-----------|------------------|-----------------|
| Built-in `web_search` / `web_fetch` | Recommended | Free | Provided by Claude - no setup |
| FireCrawl | Optional | **500 credits free** (one-time, no card). Paid plans from $16/mo | https://firecrawl.dev - get an API key, configure the MCP endpoint |
| Bright Data MCP | Optional | **5,000 requests/month free** for new MCP users. Paid pay-as-you-go from $1.50/1K results | https://brightdata.com/pricing/mcp-server - sign up, run `API_TOKEN=<token> PRO_MODE=true npx -y @brightdata/mcp` locally once to provision `mcp_unlocker` and `mcp_browser` zones |
| Brave MCP | Optional | Free Brave Search API, local via npx | No public remote Brave MCP. Get a free Brave Search API key at https://brave.com/search/api. Claude Code: `claude mcp add brave -e BRAVE_API_KEY=... -- npx -y @modelcontextprotocol/server-brave-search`. Claude Desktop: same package via `claude_desktop_config.json`. Claude.ai web: needs an HTTP bridge. Skip if not configuring - skill degrades to built-in web_search. |

If only the built-in tools are available, the skill still works. Both paid tiers are optional fallbacks.

## Cost principle

Always try free tools before paid ones. Tier 1 is free (Brave, built-in); Tier 2/3/4 are paid. Don't skip tiers - a 429 or failure at Tier 1 falls through to Tier 2, not straight to Tier 3.

---

## Part A: Searching (find URLs, research a topic)

### Search Tier 1 - Brave MCP (free, optional)

If Brave MCP is available (runs locally via npx in Claude Code / Desktop; needs an HTTP bridge on Claude.ai web):

```
Tool: brave_llm_context_search
query: [search terms, supports site: intitle: inurl: "..." operators]
count: 10
responseMode: compact
```

**Available Brave tools on a typical server:**

- `brave_llm_context_search` - web search + content extraction (returns page snippets, not just URLs)
- `brave_local_search` - location-based search
- `brave_image_search` - image search (param: `searchTerm`, not `query`)
- `brave_video_search` - video search (has `freshness` filter: `pd/pw/pm/py`)

**Note:** `brave_llm_context_search` does NOT have a `freshness` filter. Add time context to queries manually (e.g. include "2026" or "this week"). Only `brave_video_search` retains `freshness`.

**Rate limit:** 1 request per second, hardcoded. Only fire ONE Brave search per response. If you need multiple searches in one response, use Brave for the first and fall back to Tier 2 for the rest. Most errors are wrapped 429s - do not retry in the same response.

**Failure handling:** try Brave once. If it errors or the server is disconnected, skip silently to Tier 2. Do not warn the user unless they ask.

**URL approval caveat:** URLs found via Brave CANNOT be fetched with built-in `web_fetch`. See the Fetching section for which tools to use on Brave-discovered URLs.

### Search Tier 2 - Built-in web_search (free, always available)

```
Tool: web_search
query: [search terms]
```

Returns citation-tagged results. URLs found here ARE approved for built-in `web_fetch` (Fetch Tier 1).

Use this when Brave is unavailable or for second+ searches in the same response after Brave's 1-per-second limit has been used.

### Search Tier 3 - FireCrawl (paid, richest)

```
Tool: firecrawl_search
query: [search terms, supports operators]
limit: 10
location: [country name if geo-relevant]
tbs: [time filter if needed]
```

**Useful `tbs` time filters:**

- `qdr:h` / `qdr:d` / `qdr:w` / `qdr:m` / `qdr:y` - past hour/day/week/month/year
- `sbd:1` - sort by date (newest first)
- `sbd:1,qdr:w` - last week, sorted newest first
- `cdr:1,cd_min:MM/DD/YYYY,cd_max:MM/DD/YYYY` - custom date range

**Useful operators** (work on Brave, built-in web_search, and FireCrawl):

- `site:example.com` - restrict to a domain
- `"exact phrase"` - exact match
- `-keyword` - exclude
- `intitle:keyword` - must appear in title
- `inurl:keyword` - must appear in URL

### Search Tier 4 - Bright Data search_engine (paid, last resort)

Use `Brightdata pro:search_engine` when earlier tiers can't deliver:

- Geo-targeted results (`geo_location` param, e.g. "uk", "us")
- Deep pagination (cursor-based, beyond 50 results)
- Google / Bing / Yandex engine selection

Returns JSON with an `organic[]` array of `{ link, title, description }`.

URLs found via Bright Data search ARE NOT approved for built-in `web_fetch`.

---

## Part B: Fetching (retrieve full page content)

### Decision tree

```
1. Is the URL on a known-blocked site (LinkedIn, Reddit)?
   YES -> Skip FireCrawl entirely. Use Bright Data scrape_as_markdown
          (or web_data_reddit_posts for Reddit). See the dedicated skills.
   NO  -> Continue.

2. Where did the URL come from?
   - User's message or built-in web_search -> Fetch Tier 1 (free)
   - Brave MCP, FireCrawl search, Bright Data search, any other MCP
     -> Skip Tier 1 (web_fetch will reject); start at Tier 2.

3. Try the appropriate tier. Escalate on failure.
```

### Fetch Tier 1 - Built-in web_fetch (free)

Only works for URLs the user typed OR URLs from built-in `web_search`.

```
Tool: web_fetch
url: [URL]
```

**Declare failure and escalate if any of:**

- Tool returns an error, permissions error, or HTTP 4xx/5xx
- Response contains CAPTCHA or "Please verify you are human"
- Response contains "Access Denied" / "403 Forbidden" / "Enable JavaScript" / "checking your browser"
- Body is under 200 characters
- Body is pure nav/footer HTML with no meaningful text

### Fetch Tier 2 - FireCrawl scrape (paid)

```
Tool: firecrawl_scrape
url: [URL]
formats: ["markdown"]
onlyMainContent: true
```

**Proxy selection:**

| Site type | `proxy` setting |
|-----------|-----------------|
| Normal sites (blogs, docs, news) | Don't set (uses `basic`, cheapest) |
| Bot-protected (Glassdoor, some news) | `"stealth"` (+4 credits) |
| Heavily protected | `"enhanced"` (expensive, last resort) |
| Unknown | `"auto"` (FireCrawl decides) |

**JavaScript-rendered pages:** add `waitFor: 5000` (or `10000`) to let JS execute before extraction.

### Fetch Tier 3 - Bright Data scrape_as_markdown (paid, last resort)

```
Tool: Brightdata pro:scrape_as_markdown
url: [URL]
```

Bright Data handles:

- JavaScript rendering
- Anti-bot measures and CAPTCHAs
- Rate-limited or geo-restricted content
- Sites that block datacenter IPs

For multiple URLs, use `Brightdata pro:scrape_batch` (max 10 URLs per call) instead of multiple `scrape_as_markdown` invocations.

### Everything failed

Tell the user honestly: "The URL could not be fetched. Try opening it manually in your browser."

---

## URL approval boundary (important)

Built-in `web_fetch` has a security boundary. It ONLY accepts URLs from:

1. URLs the user typed directly in their message
2. URLs returned by built-in `web_search`

URLs discovered via Brave MCP, FireCrawl search, Bright Data search, or any other MCP tool are REJECTED by `web_fetch` with a permissions error.

Quick reference:

| URL came from | web_fetch? | firecrawl_scrape? | Bright Data? |
|---------------|-----------|-------------------|--------------|
| User typed / pasted it | Yes (try first) | Yes (fallback) | Yes (last) |
| Built-in web_search | Yes (try first) | Yes (fallback) | Yes (last) |
| Brave MCP | No (rejected) | Yes | Yes (fallback) |
| FireCrawl search | No (rejected) | Yes | Yes (fallback) |
| Bright Data search | No (rejected) | Yes | Yes (fallback) |
| Any other MCP | No (rejected) | Yes | Yes (fallback) |

---

## Output convention

Prepend responses with a single line noting which tier succeeded:

```
[Fetched via: web_fetch]
[Fetched via: FireCrawl]
[Fetched via: Bright Data]
[Searched via: Brave]
[Searched via: web_search]
[Searched via: FireCrawl]
```

Then the content. Omit the prefix only if the user asked for a summary or analysis rather than raw content.

---

## Cost notes

**Free:**

- Brave MCP (local via npx, needs a free Brave Search API key)
- Built-in `web_search` / `web_fetch`

**Paid:**

- FireCrawl search: 2 credits per 10 results
- FireCrawl scrape basic: 1 credit per page
- FireCrawl scrape stealth proxy: 5 credits per page
- FireCrawl JSON extraction: +4 credits per page
- Bright Data `scrape_as_markdown`: per-request from balance
- Bright Data `search_engine`: per-request from balance

Do not mention costs unless the user asks or paid tiers are triggered repeatedly in a session.

---

## Troubleshooting

**Brave MCP unavailable:** Brave has no public MCP, so the server runs on a machine you control (often exposed via a tunnel like Tailscale Funnel). If `tool_search` returns no Brave tools or the endpoint times out, skip silently to Tier 2.

**Brave returns a generic error:** Almost always a 429 rate-limit (1 req/sec hardcoded). Do not retry in the same response.

**FireCrawl returns empty or minimal content:** Add `waitFor: 5000` for JS-heavy pages, or try `firecrawl_map` to find the correct page URL first.

**FireCrawl blocks a known site:** LinkedIn and Reddit are confirmed blocked. Use Bright Data for those - see the dedicated `linkedin-job-search` and `reddit-research` skills.

**Nothing works:** Tell the user honestly and suggest manual browser access.
