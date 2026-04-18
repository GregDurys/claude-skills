---
name: reddit-research
description: Search Reddit and retrieve full thread content including comments and upvotes. Use this skill whenever asked to "search Reddit", "check Reddit", "find Reddit discussions", or fetch a specific Reddit thread URL. Uses tiered search (free Brave MCP and built-in web_search first, paid FireCrawl fallback) and Bright Data's structured web_data_reddit_posts tool for thread content - FireCrawl blocks Reddit scraping.
metadata:
  author: Greg Durys
  version: '1.0'
---

# Reddit Research

Search Reddit threads and retrieve their structured content (posts, comments, upvotes, archive status).

## Dependencies

| MCP | Required? | Cost / free tier | Sign-up / setup |
|-----|-----------|------------------|-----------------|
| Bright Data MCP | Yes | **5,000 requests/month free** for new MCP users. Paid pay-as-you-go from $1.50/1K results | https://brightdata.com/pricing/mcp-server - required for thread content. Run `API_TOKEN=<token> PRO_MODE=true npx -y @brightdata/mcp` locally once to provision required zones. |
| Brave MCP | Optional | Free Brave Search API, local via npx | No public remote Brave MCP. Get a free Brave Search API key at https://brave.com/search/api. Runs locally via `@modelcontextprotocol/server-brave-search` in Claude Code or Claude Desktop. Claude.ai web needs an HTTP bridge. Skip if not configuring - the skill uses built-in web_search or FireCrawl instead. |
| Built-in `web_search` | Optional | Free | Fallback for thread discovery |
| FireCrawl | Optional | **500 credits free** (one-time, no card). Paid plans from $16/mo | https://firecrawl.dev - for thread discovery only. `firecrawl_scrape` IS BLOCKED on reddit.com. |

## Why Bright Data for fetching

Reddit aggressively blocks scrapers. The only reliable way to retrieve a thread's full content (including all comments) is Bright Data's structured `web_data_reddit_posts` endpoint, which returns clean JSON. FireCrawl scrape is blocked; built-in `web_fetch` hits login walls.

---

## Step 1: Find thread URLs

### Tier 1 - Brave (free, if available)

```
Tool: brave_llm_context_search
query: site:reddit.com [keywords]
count: 5
responseMode: compact
```

One request per response (Brave's 1-req/sec rate limit). Skip silently if the MCP is unavailable.

### Tier 2 - Built-in web_search (free)

```
Tool: web_search
query: site:reddit.com [keywords]
```

URLs from built-in search are pre-approved for `web_fetch`, though `web_fetch` will fail on Reddit anyway.

### Tier 3 - FireCrawl search (paid)

```
Tool: firecrawl_search
query: site:reddit.com [keywords]
limit: 5
tbs: qdr:y           # optional time filter
```

FireCrawl search works fine on Reddit. Only FireCrawl **scrape** is blocked.

---

## Step 2: Fetch thread content

### Primary - Bright Data structured (preferred)

```
Tool: Brightdata pro:web_data_reddit_posts
url: [reddit thread URL]
```

Returns structured JSON with:

| Field | Type | Description |
|-------|------|-------------|
| `post_id` | string | Reddit post ID |
| `url` | string | Canonical thread URL |
| `user_posted` | string | Username of OP |
| `user_id` | string | OP user ID |
| `title` | string | Post title |
| `description` | string | Body text (plain) |
| `description_markdown` | string | Body text (markdown) |
| `num_comments` | integer | Comment count |
| `num_upvotes` | integer | Upvote count |
| `post_karma` | integer | OP's karma |
| `date_posted` | string | ISO datetime |
| `community_name` | string | Subreddit name |
| `community_url` | string | Subreddit URL |
| `community_description` | string | Subreddit description |
| `community_members_num` | integer | Subreddit subscribers |
| `related_posts` | array | Related thread summaries |
| `author_icon` | string | OP avatar URL |
| `subreddit_icon_image` | string | Subreddit icon URL |
| `archived` | string | "true" / "false" - whether the post is archived (locked from new comments) |

### Fallback - Bright Data markdown scrape

If `web_data_reddit_posts` errors:

```
Tool: Brightdata pro:scrape_as_markdown
url: [reddit thread URL]
```

Less reliable - often includes login walls and may miss comments. Use only as a fallback.

---

## What NOT to do with Reddit

- Do NOT try `firecrawl_scrape` on reddit.com URLs - it is blocked.
- Do NOT try built-in `web_fetch` on reddit.com - Reddit aggressively blocks crawlers.
- Do NOT rely on search snippets as thread content - they are truncated and miss comments.
- Do NOT skip Step 2 thinking Step 1 gave you enough context - Reddit thread value is almost always in the comments, which never appear in search snippets.

---

## Cost notes

- Brave MCP, built-in `web_search`: free
- FireCrawl search: 2 credits per 10 results (discovery only)
- Bright Data `web_data_reddit_posts`: per-request from balance
- Bright Data `scrape_as_markdown`: per-request from balance (fallback)

Do not mention costs unless the user asks.

---

## Troubleshooting

**Brave unavailable:** Skip silently to built-in web_search or FireCrawl search.

**`web_data_reddit_posts` errors on a specific URL:** Strip Reddit tracking parameters to the base thread URL. Format: `https://www.reddit.com/r/<subreddit>/comments/<post_id>/<slug>/`. If still failing, fall back to `scrape_as_markdown`.

**Thread is archived:** The `archived` field will be "true". You can still read content, but no new comments can be posted.

**Login wall in scrape_as_markdown output:** Reddit shows login walls on some threads. `web_data_reddit_posts` bypasses these - always prefer it when available.
