---
name: linkedin-job-search
description: Search LinkedIn for job listings, retrieve full job descriptions, and triage results against a configured profile. Use this skill whenever asked to "run my job search", "find LinkedIn jobs", "search for roles", or similar. Uses two Apify actors on free tier for dual-actor job discovery, plus Bright Data web_data_linkedin_job_listings for structured JD validation with application status. Customise the Profile section below with target roles, locations, and triage rules before first use.
metadata:
  author: Greg Durys
  version: '1.6'
---

# LinkedIn Job Search

Search LinkedIn job listings, retrieve full JDs, and present a triaged shortlist.

## Changelog

- **v1.6:** Dual-actor architecture on Apify free tier ($0/mo subscription, ~$0.29/week from $5 monthly credits). Two actors, each doing what it does best:
  - **HarvestAPI** (`harvestapi/linkedin-job-search`, $0.001/job) for title-based search via LinkedIn's internal search API. Good for standard role titles. Bad for cert keywords (returns zero) and creative/non-standard titles.
  - **JobSpy** (`openclawai/linkedin-jobs-scraper-jobspy`, $0.005/job) for keyword/JD body search via LinkedIn's public search page. Genuinely searches JD body text, not just titles. Finds roles invisible to title-based search.
  - FireCrawl removed from job search entirely (still used by glassdoor-research and web-research).
  - Bright Data `web_data_linkedin_job_listings` unchanged for Step 3 validation.
- **v1.5:** Dual-tool search (FireCrawl `includeDomains` + Bright Data Discover). Coverage test showed `filter_keywords` parameter is useless for domain restriction.
- **v1.4:** Replaced `scrape_as_markdown` with `web_data_linkedin_job_listings` for Step 3 validation. Added `application_availability` auto-skip. One call per URL (no batch mode), better data quality.
- **v1.3:** Result limit increased to 50 across all groups.
- **v1.2:** Added cert-keyword supplementary group.
- **v1.1:** Added senior management group. Expanded triage for ambiguous consultant titles.
- **v1.0:** Initial three-group setup.

## Dependencies

| MCP | Required? | Cost | Role in skill |
|-----|-----------|------|---------------|
| [Apify MCP](https://mcp.apify.com) | Yes | **Free plan: $0/mo, $5 credits/month.** HarvestAPI: $0.001/job. JobSpy: $0.005/job. Weekly run ~$0.29, monthly ~$1.16. No card required. | Step 1: Primary job discovery via two actors |
| [Bright Data MCP](https://github.com/brightdata/brightdata-mcp) | Yes | **5,000 requests/month free** | Step 3: JD validation via `web_data_linkedin_job_listings` |
| [FireCrawl MCP](https://github.com/firecrawl/firecrawl-mcp-server) | No (for job search) | $16/mo or free credits | NOT used for job search. Retained for Glassdoor stealth scraping and general web research. |

### Apify account notes

- Free plan: $5/month credits, no card, no subscription. Credits reset monthly, don't roll over.
- Both actors are pay-per-result (not rental). Free plan can run Store actors.
- Each actor requires one-time permissions approval on first use - click the link in the error message.
- If credits exhausted mid-month: wait for reset. No overage charges on free plan.

---

## Before first use: customise your profile

This skill is a **framework**. Edit the placeholder markers below with your own targets before running.

### Your profile (EDIT THIS SECTION)

- **Target roles:** `<TARGET_ROLES>` - e.g. "Data Engineer", "Backend Developer", "Penetration Tester", "Head of Engineering"
- **Target locations:** `<TARGET_LOCATIONS>` - e.g. "United Kingdom", "Remote EMEA", "Berlin"
- **Workplace preference:** `<WORKPLACE_TYPE>` - e.g. "Remote or hybrid, max 2 days office"
- **Salary floor:** `<SALARY_FLOOR>` - e.g. "GBP 70K+ permanent", "GBP 400+/day contract", "USD 150K+"
- **Known gaps / exclusions (optional):** `<KNOWN_GAPS>` - certifications not held, clearances not available, technologies outside experience. Roles requiring these as mandatory will be auto-skipped during triage. Leave blank if not applicable.
- **Always exclude (role types):** `<EXCLUDED_ROLE_TYPES>` - e.g. "SOC, GRC, IAM, pre-sales" for an offensive security practitioner; "sales, marketing, product management" for an engineer; etc.

### Search groups (EDIT THESE)

Define 3-4 groups split across two actors based on what each tool is good at:

- **HarvestAPI** (`harvestapi/linkedin-job-search`) - best for standard role titles. Uses LinkedIn's internal search API. Cheap ($0.001/job). Bad for cert keywords or creative titles (returns zero).
- **JobSpy** (`openclawai/linkedin-jobs-scraper-jobspy`) - best for keywords that appear in JD body text, cert names, non-standard titles. Uses LinkedIn's public search page. Costlier ($0.005/job). Genuinely searches JD body text, not just titles.

**Example shape** (for an offensive-security practitioner - replace with your own terms):

**Every-run groups** (weekly full runs AND daily catch-ups):

1. **Group A (core roles) - HarvestAPI:** `"Penetration Tester"`, `"Red Team"`, `"Offensive Security"`
2. **Group B (specialist) - HarvestAPI:** `"Vulnerability Researcher"`, `"Threat Hunter"`, `"Purple Team"`
3. **Group C (high-precision keyword) - JobSpy:** `"Adversary Simulation"` - terms with very high precision tested in coverage experiments

**Weekly-only groups** (full passes only, not daily):

4. **Group D (cert-keyword + creative titles) - JobSpy:** `"OSCP"`, `"CRTO"`, `"Proactive Threat Defence"` - keywords that find roles invisible to title-based search but have higher noise rates

**Deciding which actor for each group:**

| If the search term is... | Use | Why |
|--------------------------|-----|-----|
| A standard job title ("Data Engineer", "Penetration Tester") | HarvestAPI | Cheap, good at title matching |
| A certification or skill keyword ("OSCP", "Kubernetes") | JobSpy | HarvestAPI returns zero for cert keywords |
| A creative/non-standard title ("Proactive Threat Defence") | JobSpy | HarvestAPI only matches standard LinkedIn title fields |
| A broad term with expected noise ("Security Consultant") | Either, but add filter-on-content rule | Both tools return noise for broad terms |

### Terms to avoid (and why)

- **Overly broad titles** - match hundreds of roles spanning unrelated disciplines. Test by running once and checking the noise rate. If >70% irrelevant, drop the term.
- **Bare certification names in HarvestAPI** - HarvestAPI treats cert names as job titles and returns zero results. Use JobSpy for cert-keyword search.
- **Acronyms with multiple meanings** - too ambiguous for job search.

### Filter-on-content rules

For groups that use cert keywords or broad terms (Groups C-D in the example), a filter-on-content rule is CRITICAL:

The keyword appearing in a JD does NOT mean the role matches the target domain. Read the role responsibilities. Keep only if the core day-to-day work matches `<TARGET_ROLES>`. Skip if the role is in a different domain that happens to list the cert as desirable.

Run high-noise groups on **weekly full passes only**, not daily catch-ups.

---

## Step 1: Search for jobs (dual-actor)

### HarvestAPI groups (title-based)

**CRITICAL: Max 3 job titles per call, max 20 maxItems. Sequential calls to avoid 45-second MCP timeout.**

```
Tool: Ampify:call-actor
actor: harvestapi/linkedin-job-search
previewOutput: false
input: {
  "jobTitles": [GROUP KEYWORDS AS ARRAY],
  "locations": ["<TARGET_LOCATIONS>"],
  "workplaceType": ["remote", "hybrid"],
  "employmentType": ["full-time", "part-time", "contract"],
  "experienceLevel": ["mid-senior", "director"],
  "postedLimit": "week",
  "sortBy": "date",
  "maxItems": 20
}
```

Then retrieve results:

```
Tool: Ampify:get-actor-output
datasetId: [from call result]
fields: title,company.name,location.linkedinText,postedDate,applicants,salary.text,linkedinUrl,workplaceType
```

### JobSpy groups (keyword/body search)

Run each keyword as a separate call (one `searchTerm` per call):

```
Tool: Ampify:call-actor
actor: openclawai/linkedin-jobs-scraper-jobspy
previewOutput: false
input: {
  "searchTerm": "[KEYWORD]",
  "location": "<TARGET_LOCATIONS>",
  "maxResults": 10,
  "hoursOld": 168,
  "linkedinFetchDescription": false
}
```

Then retrieve results:

```
Tool: Ampify:get-actor-output
datasetId: [from call result]
fields: title,company_name,location,job_url
```

**JobSpy parameter reference:**

| Parameter | Type | Notes |
|-----------|------|-------|
| `searchTerm` | string | Single keyword/phrase |
| `searchTerms` | array | Multiple terms (merged, deduplicated). Max 5. |
| `location` | string | e.g. "United Kingdom" |
| `maxResults` | integer | Max jobs per term. Default 50, use 10 to save credits. |
| `hoursOld` | integer | Posted within N hours. 168 = last week, 720 = last month. |
| `linkedinFetchDescription` | boolean | Fetch full JD text. Slower. Set false for discovery, true if needed. |
| `linkedinCompanyIds` | array | Company IDs for targeted search. |
| `isRemote` | boolean | Remote only filter. |
| `jobType` | string | "fulltime", "parttime", "contract", "internship", "temporary" |

**Do NOT use the salary filter** on either actor - it excludes jobs without listed salary. Filter at presentation time.

### Deduplication

Extract LinkedIn job IDs from URLs returned by both actors. Merge into a single set before Step 2. HarvestAPI returns `linkedinUrl`, JobSpy returns `job_url`.

### Timeout prevention (HarvestAPI)

1. Max 3 job titles per call, max 20 maxItems.
2. Run calls sequentially.
3. If timeout, retry with fewer titles or lower maxItems.
4. The MCP connector hard-limits at 45 seconds regardless of `callOptions.timeout`.

### Cost management

| Actor | Cost/job | Weekly usage (est.) | Monthly (est.) |
|-------|---------|-------------------|---------------|
| HarvestAPI (title groups) | $0.001 | 2 calls x 20 results = 40 jobs = $0.04 | $0.16 |
| JobSpy (every-run keywords) | $0.005 | 1 call x 10 results = $0.05 | $0.20 |
| JobSpy (weekly-only keywords) | $0.005 | 4 calls x 10 results = $0.20 | $0.80 |
| **Total** | | **$0.29/week** | **$1.16/month** |

Well within $5/month free credits. For daily catch-ups, run every-run groups only (~$0.09).

---

## Step 2: Triage

Apply keep/drop rules to results from Step 1. HarvestAPI results have title, company, location, workplace type. JobSpy results have title, company_name, location, job_url. Triage on these before Step 3.

### Always-exclude (framework)

Customise from `<EXCLUDED_ROLE_TYPES>`. Generic examples:

- Adjacent-but-different specialisms (e.g. SOC analysts for an offensive-security role; product managers for an engineering role)
- Pre-sales / solutions engineer roles at vendors selling into the target space
- Sales / marketing / recruitment roles at companies in the target industry
- Physical security / guard / officer roles
- Roles outside the target geographic scope
- Roles with salary explicitly below `<SALARY_FLOOR>`
- On-site-only roles
- Hybrid consultancies requiring weekly client-site travel
- RLHF / AI training gig platforms (DataAnnotation, Mercor, Alignerr, Scale AI, Outlier, Crossing Hurdles, Sourced, CyberOxx) - contractor piecework, not employment
- Roles where a cert or clearance listed in `<KNOWN_GAPS>` is a **mandatory requirement** (not desirable)
- **Completely non-security/non-domain roles** that appear due to LinkedIn's poor search relevance (Video Editors, Benefits Specialists, Payroll Managers, etc.) - HarvestAPI returns these frequently, skip on sight

### Always-include (framework)

Customise from `<TARGET_ROLES>`. Generic examples:

- Exact title matches from the query groups
- Close-variant titles
- Adjacent-but-compatible roles
- Senior / head / director variants (management-pivot roles)

### Borderline - include only if JD confirms focus

Fetch the JD in Step 3 before deciding:

- **"[Domain] Consultant"** - include only if the listing mentions the target work as core responsibility
- **"[Domain] Architect"** - include only if hands-on / practitioner-led
- **"[Domain] Engineer"** - include only if it involves the target practice
- **"[Domain] Manager"** / **"Head of [Domain]"** - include only if a player-coach role with operational mandate
- **Roles from cert-keyword groups** - the cert appearing in a JD does NOT mean the role matches. Read the title.

### Recruitment agencies

- **Always keep** agency-posted roles
- **Duplicate-agency flag:** if two agencies post suspiciously similar shapes, verify before applying through both

---

## Step 3: Validate using structured LinkedIn data

For roles that passed Step 2 triage, retrieve structured data:

```
Tool: Brightdata pro:web_data_linkedin_job_listings
url: https://www.linkedin.com/jobs/view/<jobId>/
```

**Strip tracking parameters** to base URL before calling.

### Mandatory checks before presenting

1. **`application_availability` must be `true`** - if false, skip. Do NOT present closed roles.
2. **Read `job_summary`** for content triage (role shape, domain match)
3. **Check for certs/clearances from `<KNOWN_GAPS>`** - if listed as **required**, skip.

### Fallback

If `web_data_linkedin_job_listings` returns empty, try `scrape_as_markdown` and flag as **"application status unverified"**.

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

Group results by role type. Suggested category structure (customise to match `<TARGET_ROLES>`):

- **Category 1:** Core target roles (title-match groups)
- **Category 2:** Specialist / research roles
- **Category 3:** Leadership / management pivot roles
- **Category 4:** Contract roles
- **Category 5:** Supplementary (cert-caught / creative-title from keyword groups)

Each category is a table:

| Role | Company | Location | Type | Salary | Applicants | Status | Link |

The **Status** column shows "Open" (application_availability: true) or "Unverified" (fallback scrape was used).

### Quality check before presenting

1. Does this role involve the work the user actually wants to do day-to-day?
2. Would someone with the user's background genuinely want this job?
3. Is `application_availability` confirmed true?
4. If any answer is no, remove it.

### Search run discipline

After every full run, record:

> **SEARCH COMPLETED: [ISO date]**

---

## Troubleshooting

**HarvestAPI timeout:** Max 3 job titles, max 20 maxItems per call. Sequential calls.

**HarvestAPI returns non-domain noise:** Expected. Title-based search on LinkedIn has poor precision for some queries (~50% irrelevant). Triage aggressively. This is a LinkedIn search quality problem, not an actor problem.

**HarvestAPI duplicates:** ~39% duplicate rate observed. Deduplicate by job ID before triage.

**HarvestAPI returns 0 for cert keywords:** Expected. HarvestAPI treats cert names as job titles and finds nothing. Use JobSpy for cert-keyword groups.

**JobSpy permissions error:** First-time use requires approval. Click the URL in the error message.

**JobSpy returns off-domain roles for cert groups:** Expected noise. Certs appear in JDs across multiple domains. Apply filter-on-content rule.

**Apify credits exhausted:** Free plan blocks until next billing cycle. No overage. Wait for reset or skip weekly-only groups. Every-run groups are higher priority.

**`web_data_linkedin_job_listings` returns empty:** Listing too new for Bright Data index. Fall back to `scrape_as_markdown`, flag as "unverified".

**Hidden employer via agency:** Never guess. "Employer unknown" and move on.

**Completely unrelated roles (Video Editors, Benefits Specialists):** HarvestAPI returns these due to LinkedIn's semantic search. Skip on sight; not worth investigating.

**Duplicate job IDs across actors:** Expected. Deduplicate before Step 2.
