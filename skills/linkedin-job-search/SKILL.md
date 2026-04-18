---
name: linkedin-job-search
description: Search LinkedIn for job listings, retrieve full job descriptions, and triage results against a configured profile. Use this skill whenever asked to "run my job search", "find LinkedIn jobs", "search for roles", or similar. Uses FireCrawl search to discover job URLs (free Brave MCP as optional Tier 1) and Bright Data scrape_as_markdown for JD content - FireCrawl blocks LinkedIn scraping. Customise the Profile section below with your own target roles, locations, and triage rules before first use.
metadata:
  author: Greg Durys
  version: '1.0'
---

# LinkedIn Job Search

Search LinkedIn job listings, retrieve full JDs, and present a triaged shortlist.

## Dependencies

| MCP | Required? | Cost / free tier | Sign-up / setup |
|-----|-----------|------------------|-----------------|
| [FireCrawl MCP](https://github.com/firecrawl/firecrawl-mcp-server) | Yes | **500 credits free** (one-time, no card). Search = 2 credits per 10 results, so 500 credits covers ~2,500 search results. Paid plans from $16/mo | https://firecrawl.dev - for job URL discovery. LinkedIn scrape is BLOCKED on FireCrawl, but search is fine. |
| [Bright Data MCP](https://github.com/brightdata/brightdata-mcp) | Yes | **5,000 requests/month free** for new MCP users. Paid pay-as-you-go from $1.50/1K results | https://brightdata.com/pricing/mcp-server - required for JD scraping (no free alternative for LinkedIn). Run `API_TOKEN=<token> PRO_MODE=true npx -y @brightdata/mcp` locally once. |
| [Brave MCP](https://github.com/brave/brave-search-mcp-server) | Optional | Free Brave Search API, local via npx | No public remote Brave MCP. Get a free Brave Search API key at https://brave.com/search/api. Runs locally via `@brave/brave-search-mcp-server` in Claude Code or Claude Desktop - becomes Tier 1 free search. Claude.ai web needs an HTTP bridge. Otherwise skip. |
| Built-in `web_search` | Optional | Free | Last-resort search fallback |

Both required MCPs have meaningful free tiers - a weekly job search runs at zero cost.

---

## Before first use: customise your profile

This skill is a **framework**. Edit the placeholder markers below with your own targets before running. The skill triages LinkedIn results against your personalised rules.

### Your profile (EDIT THIS SECTION)

- **Target roles:** `<TARGET_ROLES>` - e.g. "Data Engineer", "Backend Developer", "Penetration Tester"
- **Target locations:** `<TARGET_LOCATIONS>` - e.g. "United Kingdom", "Remote EMEA", "London"
- **Workplace preference:** `<WORKPLACE_TYPE>` - e.g. "Remote or hybrid, max 2 days office"
- **Salary floor:** `<SALARY_FLOOR>` - e.g. "GBP 70K+ permanent", "GBP 400+/day contract"
- **Always exclude (role types):** `<EXCLUDED_ROLE_TYPES>` - e.g. "SOC, GRC, IAM, pre-sales" for an offensive security practitioner; "sales, marketing, product management" for an engineer; etc.
- **Always include (role types):** `<INCLUDED_ROLE_TYPES>` - e.g. "pen testing, red team, exploit development"; "distributed systems, backend infrastructure"; etc.

### Your search queries (EDIT THESE)

Define 2-4 query groups that together cover your target roles. Example shape for an offensive-security practitioner (replace with your own terms):

1. **Group A (core roles):** `"Penetration Tester" OR "Red Team" OR "Offensive Security"`
2. **Group B (specialist roles):** `"Vulnerability Researcher" OR "Threat Hunter" OR "Purple Team"`
3. **Group C (career pivot):** `"Product Security" OR "Vulnerability Management"`

Include only terms that genuinely match roles you'd take. See "Choosing search terms" below for what to avoid.

### Choosing search terms (notes)

Prefer **specific, descriptive role titles** over generic ones. Common mistakes to avoid:

- **Overly broad titles** - e.g. "Security Manager", "Engineering Manager", "Developer" - match hundreds of roles spanning disciplines you don't want.
- **Single-word certifications or skills** - e.g. "OSCP", "AWS", "Kubernetes" - match any JD that lists them as a nice-to-have, dragging in unrelated roles.
- **Acronyms with multiple meanings** - e.g. "TTPs", "DAST" - too ambiguous for a job title search.
- **Generic consultant titles** - e.g. "Cybersecurity Consultant", "Data Consultant" - vendor-neutral and usually capture GRC, strategy, pre-sales, and compliance alongside technical work.

Start narrow. Add broader terms only after narrow ones return too few results.

---

## Step 1: Search for job URLs

Run one query per search group.

### Tier 1 - Brave (free, optional)

If Brave MCP is available, use it for the FIRST query only (1 req/sec rate limit):

```
Tool: brave_llm_context_search
query: site:linkedin.com/jobs/view [Group A terms] <TARGET_LOCATIONS>
count: 10
responseMode: compact
```

### Tier 2 - FireCrawl (paid, primary)

For remaining queries, use FireCrawl:

```
Tool: firecrawl_search
query: site:linkedin.com/jobs/view [Group B terms] <TARGET_LOCATIONS>
tbs: sbd:1,qdr:w              # sort by date, last week
location: <TARGET_COUNTRY>    # e.g., "United Kingdom"
limit: 10
```

```
Tool: firecrawl_search
query: site:linkedin.com/jobs/view [Group C terms] <TARGET_LOCATIONS>
tbs: sbd:1,qdr:w
location: <TARGET_COUNTRY>
limit: 10
```

**`tbs` cheat-sheet:**

- `sbd:1,qdr:w` - last week, sorted newest
- `qdr:m` - last month (catch-up after a gap)
- `qdr:y` - last year (deeper history)

### Tier 3 - Built-in web_search (free, last resort)

```
Tool: web_search
query: site:linkedin.com/jobs/view [terms] <TARGET_LOCATIONS>
```

Less precise than FireCrawl. Use only if both Brave and FireCrawl are unavailable.

### Contract-only variant (optional)

If you want a separate contract-focused sweep, run an extra query biased toward contract listings:

```
Tool: firecrawl_search
query: site:linkedin.com/jobs/view [Group A/B terms] contract <TARGET_LOCATIONS>
tbs: sbd:1,qdr:w
location: <TARGET_COUNTRY>
limit: 10
```

---

## Step 2: Triage

Apply your exclude / include rules to the URLs returned by Step 1. Make a keep / drop decision based on the title and snippet - no need to scrape every JD.

### Always-exclude (framework)

Exclude roles that don't match your target focus. Customise from your `<EXCLUDED_ROLE_TYPES>`. Generic examples:

- Adjacent-but-different specialisms (e.g., SOC analysts for an offensive-security role; product managers for an engineering role)
- Pre-sales / solutions engineer roles at vendors selling into your space
- Sales / marketing / recruitment roles at companies in your industry (tangential keyword matches)
- Physical security / guard / officer roles (LinkedIn matches these on "security")
- Roles outside your geographic scope
- Roles explicitly below your `<SALARY_FLOOR>`

### Always-include (framework)

Keep roles that clearly match your target focus. Customise from your `<INCLUDED_ROLE_TYPES>`. Generic examples:

- Exact title matches (e.g., "Penetration Tester", "Staff Software Engineer")
- Close-variant titles (e.g., "Red Team Operator" if targeting Red Team)
- Adjacent-but-compatible roles (e.g., "Exploit Developer" if targeting Offensive Security)

### Borderline - include only if the JD confirms focus

Some titles are ambiguous and depend on the actual role content. Examples:

- **"[Domain] Consultant"** - include only if the listing mentions your target work (not GRC, strategy, or pre-sales wrapping).
- **"[Domain] Architect"** - include only if it is hands-on / practitioner-led, not governance-led.
- **"[Domain] Engineer"** - include only if it involves the practice you want, not adjacent ops or hardening.
- **"[Domain] Manager"** - include only if it is a player-coach role over a team doing your target work.

For these, fetch the JD in Step 3 before deciding.

### Salary filtering rules

- **Exclude** roles with a salary explicitly listed below your `<SALARY_FLOOR>`.
- **Keep** roles with no salary listed - most good roles don't advertise.
- **Keep** roles at companies known for paying well even if unlisted.

### Consultancy filter

- **Keep** consultancy roles if they are fully remote.
- **Exclude** consultancy roles that are hybrid or on-site. The issue is client-site travel, not consultancies as such.

### Recruitment agencies

- **Always keep** roles posted by recruitment agencies - they are sourcing on behalf of real companies. Don't discard based on the poster.

---

## Step 3: Validate interesting roles (Bright Data)

For roles that passed Step 2 triage and need JD content to decide (borderline titles, standout candidates), scrape the full JD:

```
Tool: Brightdata pro:scrape_as_markdown
url: https://www.linkedin.com/jobs/view/<jobId>/
```

**Strip LinkedIn tracking parameters** to the base URL before scraping. The core form is always `linkedin.com/jobs/view/<jobId>/`.

FireCrawl is BLOCKED on LinkedIn - do not attempt `firecrawl_scrape`.

---

## Validation discipline (apply before Step 4)

**A listing only exists as a real opportunity if you have BOTH:**

1. The actual `linkedin.com/jobs/view/<jobId>/` URL from Step 1.
2. The full JD validated via Bright Data in Step 3.

**Do NOT present as confirmed or verified:**

- Job titles or companies mentioned only in FireCrawl / Brave / web_search snippets
- Roles inferred from cached index pages or aggregator previews
- Listings whose JD has not been retrieved via Bright Data

**If you spot something promising in a snippet but can't retrieve the listing URL** (the URL is an aggregate search page, the JD scrape fails, the role is already closed, etc.): say so explicitly in your presentation rather than listing it as an opportunity:

> "Saw [company] / [role] mentioned in search snippets but couldn't retrieve the individual listing - treat as unconfirmed."

When summarising the run, separate "found" from "validated":

> "X listings discovered via FireCrawl search, Y validated via Bright Data JD retrieval, Z presented below."

Step 2's triage (keep/drop on title + snippet) is fine because no claim is being made yet. The hard rule is at Step 4: anything presented as a real opportunity in the final table must have passed Step 3 validation, or be flagged as unconfirmed.

---

## Step 4: Present

Group results by role type for scanability. Suggested 4-category split (customise category names to match your `<INCLUDED_ROLE_TYPES>`):

- **Category 1:** Core target roles (exact matches)
- **Category 2:** Specialist / research roles
- **Category 3:** Leadership / management pivot roles (if applicable)
- **Category 4:** Contract roles (if the contract-only variant ran)

Each category is a table:

| Role | Company | Location | Type | Salary | Applicants | Link |

After the tables, highlight 2-3 standout roles with a one-sentence rationale: low applicant count, strong product company fit, remote, good salary band, particularly relevant to your profile, etc.

### Quality check before presenting

For each candidate role, ask:

1. Does this role involve the work I actually want to do day-to-day?
2. Would someone with my background genuinely want this job, or am I matching on keywords only?
3. If either answer is no, remove it regardless of title.

Only present roles that pass both questions.

### Search run discipline

After every full run, record:

> **SEARCH COMPLETED: [ISO date]**

Check this timestamp before claiming a search was recent.

---

## Cost notes

- FireCrawl search: 2 credits per 10 results. Three groups x 10 results = 6 credits per run.
- Bright Data `scrape_as_markdown`: per-request from balance. Scrape only roles worth investigating.
- Brave (via free Brave API key + local npx), built-in `web_search`: free.

A full weekly run is typically a few dozen credits plus a handful of per-request scrape calls.

---

## Troubleshooting

**FireCrawl search returns aggregate pages** (e.g., `linkedin.com/jobs/search/...`) instead of individual job view URLs: tighten the query with `inurl:view` or use more specific title terms.

**Bright Data JD scrape returns thin content:** strip all tracking parameters and retry with just the base URL `linkedin.com/jobs/view/<jobId>/`.

**LinkedIn says the role is closed / no longer accepting:** note this on the table row rather than removing it - timing info is useful context.

**Too many irrelevant results:** your search terms are too broad (see "Choosing search terms"). Narrow to more specific titles.

**Too few results:** widen query groups, or relax `tbs` from `qdr:w` to `qdr:m`.

**Brave MCP unavailable:** skip silently to FireCrawl. Brave is optional.
