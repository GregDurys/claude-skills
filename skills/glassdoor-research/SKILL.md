---
name: glassdoor-research
description: Pull company reviews, ratings breakdowns, and interview experiences from Glassdoor. Use this skill whenever asked to "check Glassdoor", "pull reviews for", "what's it like to work at", "research [company]", or "interview process at". Uses FireCrawl with stealth proxy to bypass Glassdoor's aggressive bot detection, with Bright Data as a deeper fallback.
---

# Glassdoor Research

Retrieve Glassdoor reviews, ratings, and interview data; present them in a structured format with sentiment analysis.

## Dependencies

| MCP | Required? | Cost / free tier | Sign-up / setup |
|-----|-----------|------------------|-----------------|
| FireCrawl | Yes | **500 credits free** (one-time, no card). Stealth scrape = 5 credits/page, so 500 credits covers ~100 Glassdoor pages. Paid plans from $16/mo | https://firecrawl.dev - get an API key, configure the MCP endpoint |
| Bright Data MCP | Optional | **5,000 requests/month free** for new MCP users. Paid pay-as-you-go from $1.50/1K results | https://brightdata.com/pricing/mcp-server - fallback if FireCrawl stealth is blocked. Run `API_TOKEN=<token> PRO_MODE=true npx -y @brightdata/mcp` locally once. |
| FireCrawl search or built-in `web_search` | Recommended | Same FireCrawl free tier above, or free built-in | For discovering the correct employer ID when starting from a company name |

## Why stealth proxy

Glassdoor has aggressive bot detection. Built-in `web_fetch` fails; FireCrawl basic scrape returns login walls. `proxy: "stealth"` with a `waitFor` of 5s reliably retrieves public review content.

---

## Step 1: Identify the company

Glassdoor uses a slug (the company's name as it appears in URLs) and an employer ID. Public examples:

| Company | Slug | Employer ID | Full URL pattern |
|---------|------|-------------|------------------|
| Google | `Google` | `9079` | `/Reviews/Google-Reviews-E9079.htm` |
| Microsoft | `Microsoft` | `1651` | `/Reviews/Microsoft-Reviews-E1651.htm` |
| Netflix | `Netflix` | `11891` | `/Reviews/Netflix-Reviews-E11891.htm` |

If you don't know the employer ID:

```
Tool: firecrawl_search        # or built-in web_search if available
query: site:glassdoor.co.uk [Company name] reviews
limit: 3
```

Extract the employer ID from the URL returned (the number after `-E` or `EI_IE`).

---

## Step 2: Pull the reviews page

```
Tool: firecrawl_scrape
url: https://www.glassdoor.co.uk/Reviews/<Slug>-Reviews-E<employerId>.htm
formats: ["markdown"]
onlyMainContent: true
proxy: "stealth"
waitFor: 5000
```

Use the `.co.uk` domain for UK-focused reviews, `.com` for US-focused.

### Review schema (per review after parsing)

| Field | Description |
|-------|-------------|
| `review_date` | When the review was posted |
| `job_title` | Reviewer's role |
| `location` | Office or remote |
| `rating_overall` | 1-5 |
| `summary` | Review headline |
| `pros` | Positive points |
| `cons` | Negative points |
| `advice` | Advice to management (optional) |
| `is_current_employee` | Boolean |
| `rating_work_life_balance` | 0-5 (0 = not rated) |
| `rating_compensation_and_benefits` | 0-5 |
| `rating_senior_leadership` | 0-5 |
| `rating_culture_and_values` | 0-5 |
| `rating_career_opportunities` | 0-5 |
| `rating_diversity_and_inclusion` | 0-5 |

Exclude 0-rated categories from per-category averages - 0 means "not rated by the reviewer", not a score of zero.

---

## Step 3: Pull interview data (if requested)

### Base interview page

```
Tool: firecrawl_scrape
url: https://www.glassdoor.co.uk/Interview/<Slug>-Interview-Questions-E<employerId>.htm
formats: ["markdown"]
onlyMainContent: true
proxy: "stealth"
waitFor: 5000
```

### Location-specific interviews

Glassdoor filters interviews by office location. URL pattern:

```
https://www.glassdoor.co.uk/Interview/<Slug>-<City>-Interview-Questions-EI_IE<employerId>.0,N_IL.M,P_IM<locationId>.htm
```

The `<locationId>` is Glassdoor's internal ID (visible in the URL when you click "Filter by location" on the public page). If you don't know it, fall back to the base URL and filter content by location after scraping.

### Pagination

Append `_IP2.htm`, `_IP3.htm` etc. to paginate beyond the first 5 interviews:

```
https://www.glassdoor.co.uk/Interview/<Slug>-Interview-Questions-E<employerId>_IP2.htm
```

---

## Step 4: Analyse and present

### Rating summary table

Calculate averages across retrieved reviews (excluding 0 values):

```
| Category                    | Average | Assessment |
|-----------------------------|---------|------------|
| Overall                     | X.X/5   |            |
| Work/Life Balance           | X.X/5   |            |
| Compensation & Benefits     | X.X/5   |            |
| Senior Leadership           | X.X/5   |            |
| Culture & Values            | X.X/5   |            |
| Career Opportunities        | X.X/5   |            |
| Diversity & Inclusion       | X.X/5   |            |
```

Flag any category below 2.5 as a concern.

### Sentiment analysis

Group review text into themes. Common patterns:

- Compensation complaints (below market, stagnant pay)
- Leadership issues (disconnect, lack of accountability)
- Culture shifts (post-restructure, offshoring)
- Career progression (stalled promotions, limited growth)
- Work-life balance (on-call pressure, remote flexibility)
- Positive themes (good people, learning, benefits)

### Current vs former employee split

Note the ratio. A high proportion of former employees with negative reviews suggests turnover issues.

### Location-specific insights

If reviews span multiple locations, highlight patterns by site relevant to the role being evaluated.

### Interview summary (if Step 3 ran)

- Typical stages and format
- Average duration (application to offer)
- Difficulty rating (if provided)
- Common interview questions
- Red flags or tips candidates mention

### "What this means for you" section

Tailor to the user's situation:

- Interview prep: highlight process details, common questions, red flags
- Offer evaluation: flag compensation concerns, leadership issues, culture fit
- General research: balanced summary of positives and negatives

---

## Cost notes

- FireCrawl stealth scrape: 5 credits per page. A reviews page + interview page = 10 credits per company.
- Bright Data fallback: per-request from balance.

Inform the user of approximate cost if pulling more than 5 companies in one session.

---

## Troubleshooting

**Wrong company returned:** The slug or employer ID is incorrect. Re-search via FireCrawl / web_search with `site:glassdoor.co.uk [Company]` and verify the URL.

**Empty reviews page:** The company has few or no public reviews. Try the `.com` domain if you used `.co.uk` (or vice versa) for different regional pools.

**Login wall despite stealth proxy:** Glassdoor is aggressively gating content. Fall back to Bright Data:

```
Tool: Brightdata pro:scrape_as_markdown
url: [same URL]
```

**Scraped markdown is mostly navigation:** Add `waitFor: 10000` to let JS render fully.

**Location ID unknown:** Use the base interview URL and filter manually in the output rather than constructing the location-specific URL.
