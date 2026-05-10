---
name: linkedin-job-search
description: Search LinkedIn for job listings, retrieve full job descriptions, and triage results against a configured profile. Use this skill whenever asked to "run my job search", "find LinkedIn jobs", "search for roles", or similar. Uses FireCrawl search + Bright Data Discover for dual-tool URL discovery, then Bright Data web_data_linkedin_job_listings for structured JD content with application status. Customise the Profile section below with target roles, locations, and triage rules before first use.
metadata:
  author: Greg Durys
  version: '1.5'
---

# LinkedIn Job Search

Search LinkedIn job listings, retrieve full JDs, and present a triaged shortlist.

## Changelog

- **v1.5:** Dual-tool search in Step 1. FireCrawl switched from `site:` operator to `includeDomains` (catches more results per coverage testing). Added Bright Data `discover` as Tier 2.5 supplementary sweep. Deduplicate across tools before triage. Cost impact: +5 Bright Data requests/week (negligible vs 5,000/month free tier). Coverage test confirmed `filter_keywords` parameter is useless for domain restriction.
- **v1.4:** Replaced `scrape_as_markdown` with `web_data_linkedin_job_listings` for Step 3 validation. Returns structured JSON including `application_availability` (true/false) - auto-skip closed roles. Also returns reliable `job_num_applicants` and `job_posted_date`. One call per URL (no batch mode), but data quality is significantly better. Added mandatory "No longer accepting applications" check.
- **v1.3:** Result limit increased from 10 to 50 across all groups.
- **v1.2:** Added cert-keyword supplementary group (Group E).
- **v1.1:** Added senior management group (Group D). Expanded triage for ambiguous consultant titles.
- **v1.0:** Initial three-group setup.

## Dependencies

| MCP | Required? | Cost / free tier | Sign-up / setup |
|-----|-----------|------------------|-----------------|
| [FireCrawl MCP](https://github.com/firecrawl/firecrawl-mcp-server) | Yes | **500 credits free** (one-time, no card). Search = 2 credits per 10 results. At limit 50 that is 10 credits per query. Paid plans from $16/mo | https://firecrawl.dev - for job URL discovery. LinkedIn scrape is BLOCKED on FireCrawl, but search is fine. |
| [Bright Data MCP](https://github.com/brightdata/brightdata-mcp) | Yes | **5,000 requests/month free** for new MCP users. Paid pay-as-you-go from $1.50/1K results | https://brightdata.com/pricing/mcp-server - required for JD validation AND Discover supplementary search. Run `API_TOKEN=<token> PRO_MODE=true npx -y @brightdata/mcp` locally once. |
| [Brave MCP](https://github.com/brave/brave-search-mcp-server) | Optional | Free Brave Search API, local via npx | Optional Tier 1 search. Skip if not configuring. |
| Built-in `web_search` | Optional | Free | Last-resort search fallback |

Both required MCPs have meaningful free tiers - a weekly job search at limit 50 across 5 groups costs ~50 FireCrawl credits + ~10-20 Bright Data requests (5 discover + 8-15 validation).

---

## Before first use: customise your profile

This skill is a **framework**. Edit the placeholder markers below with your own targets before running.

### Your profile (EDIT THIS SECTION)

- **Target roles:** `<TARGET_ROLES>` - e.g. "Data Engineer", "Backend Developer", "Penetration Tester", "Head of Engineering"
- **Target locations:** `<TARGET_LOCATIONS>` - e.g. "United Kingdom", "Remote EMEA", "Berlin"
- **Workplace preference:** `<WORKPLACE_TYPE>` - e.g. "Remote or hybrid, max 2 days office"
- **Salary floor:** `<SALARY_FLOOR>` - e.g. "GBP 70K+ permanent", "GBP 400+/day contract", "USD 150K+"
- **Known gaps / exclusions (optional):** `<KNOWN_GAPS>` - certifications not held, clearances not available, technologies outside your experience. Roles requiring these as mandatory will be auto-skipped during triage. Leave blank if not applicable.
- **Always exclude (role types):** `<EXCLUDED_ROLE_TYPES>` - e.g. "SOC, GRC, IAM, pre-sales" for an offensive security practitioner; "sales, marketing, product management" for an engineer; etc.

### Your search queries (EDIT THESE)

Define 3-5 query groups that together cover your target roles. Use multiple groups to separate core roles, specialist variants, career-pivot titles, and optional supplementary queries.

**Example shape** (for an offensive-security practitioner - replace with your own terms):

1. **Group A (core roles):** `"Penetration Tester" OR "Red Team" OR "Offensive Security"`
2. **Group B (specialist roles):** `"Vulnerability Researcher" OR "Threat Hunter" OR "Purple Team"`
3. **Group C (career pivot):** `"Product Security" OR "Vulnerability Management"`
4. **Group D (senior management):** `"Head of [Your Domain]" OR "Director of [Your Domain]"`
5. **Group E (cert-keyword supplementary, weekly only):** `"[CERT_1]" OR "[CERT_2]"` - catches roles where your target certs are listed. **Requires filter-on-content triage** - a cert appearing in a JD does not mean the role matches your target domain. Run Group E on weekly full passes only, not daily catch-ups.

### Choosing search terms (notes)

Prefer **specific, descriptive role titles** over generic ones. Common mistakes to avoid:

- **Overly broad titles** - e.g. "Security Manager", "Engineering Manager", "Developer" - match hundreds of roles spanning disciplines outside the target.
- **Single-word certifications** - e.g. bare cert names like "AWS", "Kubernetes" - match any JD that lists them as nice-to-have, dragging in unrelated roles. Use certs only in a dedicated Group E with strict filter-on-content triage.
- **Acronyms with multiple meanings** - e.g. "TTPs", "DAST" - too ambiguous for a job title search.
- **Generic consultant titles** - e.g. "Data Consultant", "Security Consultant" - capture advisory, strategy, pre-sales, and compliance alongside technical work. If using, add a filter-on-content rule (see Group A example above).

Start narrow. Add broader terms only after narrow ones return too few results.

---

## Step 1: Search for job URLs (dual-tool, v1.5)

Run each query group through TWO tools, then deduplicate by LinkedIn job ID before Step 2.

### Tier 1 - Brave (free, optional)

If Brave MCP is available, use for the FIRST query only (1 req/sec rate limit):

```
Tool: brave_llm_context_search
query: site:linkedin.com/jobs/view [GROUP A KEYWORDS] <TARGET_LOCATIONS>
count: 20
responseMode: compact
```

### Tier 2 - FireCrawl (paid, primary)

Use `includeDomains` instead of `site:` operator. This catches more results per coverage testing because `includeDomains` is FireCrawl's native domain filter rather than a Google search syntax hint.

```
Tool: firecrawl_search
query: [GROUP KEYWORDS] <TARGET_LOCATIONS> jobs
includeDomains: ["linkedin.com"]
tbs: sbd:1,qdr:w
location: <TARGET_COUNTRY>
limit: 50
```

Run for each group.

**Note:** Without `site:linkedin.com/jobs/view` in the query, some results may be LinkedIn company pages, user profiles, or non-job URLs. This is acceptable noise - Step 2 triage filters by URL pattern. Only process URLs matching `linkedin.com/jobs/view/`.

**`tbs` cheat-sheet:**

- `sbd:1,qdr:w` - last week, sorted newest (default for weekly runs)
- `sbd:1,qdr:d` - last 24h (daily catch-up, omit Group E, drop limit to 20-30)
- `qdr:m` - last month (catch-up after gap > 1 week)

### Tier 2.5 - Bright Data Discover (paid, supplementary, new in v1.5)

After FireCrawl completes for each group, run a supplementary sweep with Discover:

```
Tool: Brightdata pro:discover
query: [GROUP KEYWORDS] <TARGET_LOCATIONS> site:linkedin.com/jobs/view
country: <TARGET_COUNTRY_CODE>
intent: LinkedIn job postings for [group description] roles in <TARGET_LOCATIONS>
num_results: 20
start_date: [ISO date 7 days ago]
```

**CRITICAL:** Always include `site:linkedin.com/jobs/view` in the `query` parameter. Without it, Discover returns 40-60% non-LinkedIn results (itjobswatch, indeed, totaljobs, etc.) which are pure noise for this workflow.

**NEVER use `filter_keywords`** for domain restriction. Coverage testing showed `filter_keywords: ["linkedin.com/jobs/view"]` reduced LinkedIn precision from 100% to 20%. It is a post-filter, not a search modifier.

**Deduplication:** Extract LinkedIn job IDs from URLs returned by both FireCrawl and Discover. Merge into a single set before proceeding to Step 2.

### Tier 3 - Built-in web_search (free, last resort)

```
Tool: web_search
query: site:linkedin.com/jobs/view [terms] <TARGET_LOCATIONS>
```

Less precise. Use only if both FireCrawl and Discover are unavailable.

### When to skip Discover (cost optimisation)

- **Daily catch-ups:** skip Discover, run FireCrawl only at limit 20-30
- **Credit pressure:** Discover is the first to drop (FireCrawl's broader result set is more valuable)
- **If FireCrawl returned 40+ results for a group:** Discover unlikely to find actionable additions

---

## Step 2: Triage on title and snippet

Apply keep/drop rules to the URLs returned by Step 1. No scraping needed yet - just reading titles and snippets.

### Always-exclude (framework)

Exclude roles that don't match the target focus. Customise from `<EXCLUDED_ROLE_TYPES>`. Generic examples:

- Adjacent-but-different specialisms (e.g. SOC analysts for an offensive-security role; product managers for an engineering role)
- Pre-sales / solutions engineer roles at vendors selling into the target space
- Sales / marketing / recruitment roles at companies in the target industry (tangential keyword matches)
- Physical security / guard / officer roles (LinkedIn matches these on "security")
- Roles outside the target geographic scope
- Roles with salary explicitly below `<SALARY_FLOOR>`
- On-site-only roles (verify JD body as LinkedIn metadata tags can be misleading)
- Hybrid consultancies requiring weekly client-site travel
- RLHF / AI training gig platforms (DataAnnotation, Mercor, Alignerr, Scale AI, Outlier, Crossing Hurdles, Sourced) - contractor piecework, not employment
- Roles where a cert or clearance listed in `<KNOWN_GAPS>` is a **mandatory requirement** (not desirable)

### Always-include (framework)

Keep roles that clearly match the target focus. Customise from `<TARGET_ROLES>`. Generic examples:

- Exact title matches from the query groups
- Close-variant titles (e.g. "Red Team Operator" if targeting Red Team)
- Adjacent-but-compatible roles (e.g. "Exploit Developer" if targeting offensive security)
- Senior/head/director variants of target role titles (management-pivot roles)

### Borderline - include only if the JD confirms focus

Some titles are ambiguous and depend on the actual role content. Fetch the JD in Step 3 before deciding:

- **"[Domain] Consultant"** - include only if the listing mentions the target work as core responsibility, not if wrapped in advisory / strategy / pre-sales.
- **"[Domain] Architect"** - include only if hands-on / practitioner-led, not governance-led.
- **"[Domain] Engineer"** - include only if it involves the target practice, not adjacent ops or hardening.
- **"[Domain] Manager"** / **"Head of [Domain]"** - include only if a player-coach role with operational mandate over the target function.
- **Roles from Group E (cert-keyword)** - the cert appearing in a JD does NOT mean the role matches the target domain. Read the role responsibilities.

### Salary filtering rules

- **Exclude** roles with a salary explicitly listed below `<SALARY_FLOOR>`.
- **Keep** roles with no salary listed - most good roles don't advertise.
- **Keep** roles at companies known for paying well even if unlisted.

### Consultancy filter

- **Keep** consultancy roles if fully remote.
- **Exclude** consultancy roles that are hybrid or on-site. The issue is client-site travel, not consultancies as such.

### Recruitment agencies

- **Always keep** agency-posted roles - they source for real companies.
- **Duplicate-agency flag:** if two agencies post suspiciously similar shapes in the same region and salary band, verify with one recruiter before applying through both.

---

## Step 3: Validate using structured LinkedIn data (v1.4+)

For roles that passed Step 2 triage, retrieve structured data:

```
Tool: Brightdata pro:web_data_linkedin_job_listings
url: https://www.linkedin.com/jobs/view/<jobId>/
```

**Strip LinkedIn tracking parameters** to base URL before calling.

### What this tool returns (structured JSON)

| Field | Use |
|-------|-----|
| `application_availability` | **true/false** - CRITICAL. If false, role is closed. Auto-skip. |
| `job_num_applicants` | Exact applicant count (reliable, not text-parsed) |
| `job_posted_date` | ISO date (reliable) |
| `job_title` | Exact title |
| `company_name` | Employer name |
| `job_location` | Location |
| `job_summary` | Full JD text |
| `job_seniority_level` | LinkedIn seniority tag |
| `is_easy_apply` | Whether Easy Apply is enabled |

### Mandatory checks before presenting

For each role returned by this tool, check:

1. **`application_availability` must be `true`** - if false, skip immediately. Do NOT present closed roles.
2. **Read `job_summary`** for content triage (role shape vs target, defensive vs target domain)
3. **Check for certs/clearances from `<KNOWN_GAPS>`** in `job_summary` - if they appear as **required**, skip.

### When to fall back to scrape_as_markdown

`web_data_linkedin_job_listings` may occasionally return empty or fail for very new listings not yet in Bright Data's index. If a role passed Step 2 triage and `web_data_linkedin_job_listings` returns nothing:

1. Try `Brightdata pro:scrape_as_markdown` as fallback
2. Flag the role as **"application status unverified"** in the presentation
3. Note that the user should check LinkedIn directly before applying

### One call per URL (no batch mode)

Unlike `scrape_batch` (up to 10 URLs per call), `web_data_linkedin_job_listings` is one call per URL. For a typical weekly run with 8-15 shortlisted roles, this means 8-15 individual calls. This is acceptable given the data quality improvement.

For efficiency: triage aggressively in Step 2 so fewer roles reach Step 3. Only validate roles that genuinely look promising from title + snippet.

---

## Validation discipline (apply before Step 4)

**A listing only exists as a real opportunity if ALL THREE are confirmed:**

1. The actual `linkedin.com/jobs/view/<jobId>/` URL from Step 1.
2. The full JD validated via `web_data_linkedin_job_listings` in Step 3.
3. **`application_availability: true`** confirmed in Step 3.

**Do NOT present:**

- Listings where `application_availability` is false (closed)
- Job titles or companies from search snippets only (not validated)
- Roles whose JD has not been retrieved
- Roles flagged as "application status unverified" without explicit warning

**Location tag vs JD body:** LinkedIn's workplace-type tag (Remote / Hybrid / On-site) is set by the recruiter and often mislabelled. Trust the JD body text, not the tag.

---

## Step 4: Present

Group results by role type for scanability. Suggested category structure (customise to match `<TARGET_ROLES>`):

- **Category 1:** Core target roles (exact matches from Groups A/B)
- **Category 2:** Specialist / research roles (Group B variants)
- **Category 3:** Leadership / management pivot roles (Group D)
- **Category 4:** Contract roles (if a contract-only variant ran)
- **Category 5:** Supplementary (cert-caught from Group E, not fitting Categories 1-3)

Each category is a table:

| Role | Company | Location | Type | Salary | Applicants | Status | Link |

The **Status** column shows "Open" (application_availability: true) or "Unverified" (fallback scrape was used).

### Quality check before presenting

For each candidate role, ask:

1. Does this role involve the work the user actually wants to do day-to-day?
2. Would someone with the user's background genuinely want this job, or is it matching on keywords only?
3. Is `application_availability` confirmed true?
4. If any answer is no, remove it regardless of title.

Only present roles that pass all questions.

### Search run discipline

After every full run, record:

> **SEARCH COMPLETED: [ISO date]**

Check this timestamp before claiming a search was recent. Weekly cadence assumes previous run within 7 days; if longer, widen `tbs` from `qdr:w` to `qdr:m`.

---

## Cost notes

- FireCrawl search: 2 credits per 10 results, limit 50 = 10 credits per query. Full weekly run (5 groups) = ~50 credits.
- Discover supplementary: 1 Bright Data request per group. Full weekly run (5 groups) = 5 requests.
- `web_data_linkedin_job_listings`: 1 Bright Data request per URL. Typical weekly run validates 8-15 roles = 8-15 requests.
- **Total weekly cost:** ~50 FireCrawl credits + ~13-20 Bright Data requests. Well within free tiers.
- Daily catch-up runs (Groups A-D only, FireCrawl at limit 20-30, skip Discover, ~5-8 validations) = ~24-32 FC credits + 5-8 BD requests.

---

## Troubleshooting

**FireCrawl returns aggregate pages** instead of job view URLs: add `inurl:view` to query or filter Step 2 more strictly on URL pattern.

**Discover returns non-LinkedIn results:** `site:linkedin.com/jobs/view` is missing from query string. Add it. Never use `filter_keywords` for domain restriction (tested: reduces precision from 100% to 20%).

**`web_data_linkedin_job_listings` returns empty:** listing may be too new for Bright Data's index. Fall back to `scrape_as_markdown` and flag as "application status unverified".

**LinkedIn says role is closed:** `application_availability: false` auto-skips. Note in table if useful for context ("X was hiring recently").

**Too many results from Group E:** apply filter-on-content more strictly. Skip roles outside the target domain even if certs are listed. Drop limit to 20 if consistently noisy.

**Too few results:** relax `tbs` from `qdr:w` to `qdr:m`.

**Hidden employer via agency:** never guess or speculate. Present as "Employer unknown" and move on.

**FireCrawl credit pressure:** drop Discover first (FireCrawl has broader coverage). Then drop limits from 50 to 20-30. Or run only core groups weekly, others fortnightly.

**Duplicate job IDs across search tools:** expected. Deduplicate before Step 2. LinkedIn itself sometimes posts the same role under multiple IDs (e.g. different location tags).

**Role appears open but was recently closed:** `web_data_linkedin_job_listings` data can lag by hours. If a role is reported closed after being shown as open, acknowledge the lag and skip.
