# Testing

Live test results for each skill in this repo, run against the MCP tools on the test date. Tests confirm each documented tier works end-to-end.

**Test date:** 2026-04-17

---

## Results summary

| # | Skill | Tool | Test input | Expected | Result |
|---|-------|------|------------|----------|--------|
| T1 | web-research | `brave_llm_context_search` | query: `cybersecurity blog 2026`, count 8 | >= 3 results with URL + snippet | PASS |
| T2 | web-research | `firecrawl_search` | query: `python tutorial 2026`, limit 5 | >= 3 URLs | PASS (5 URLs) |
| T3 | web-research | `web_fetch` | url: https://example.com | Page content extracted | PASS |
| T4 | web-research | `firecrawl_scrape` (basic) | url: https://example.com | Markdown page content | PASS (1 credit used) |
| T5 | reddit-research | `brave_llm_context_search` | query: `site:reddit.com cybersecurity jobs`, count 5 | >= 3 reddit.com URLs | PASS |
| T6 | reddit-research | `Brightdata pro:web_data_reddit_posts` | url: reddit learnprogramming thread | Structured JSON with title, description, num_comments, `archived` field | PASS (archived field present as "false") |
| T7 | reddit-research | `firecrawl_scrape` on reddit.com URL | any reddit thread | BLOCKED (documented behaviour) | PASS (block confirmed) |
| T8 | glassdoor-research | `firecrawl_scrape` with `proxy: "stealth"` | https://www.glassdoor.co.uk/Reviews/Google-Reviews-E9079.htm | Review content in markdown | PASS (5 credits, 47K reviews, rating 4.4/5 surfaced) |
| T9 | glassdoor-research | `firecrawl_search` | query: `site:glassdoor.co.uk Google reviews`, limit 3 | Valid employer page URLs | PASS (3 employer URLs) |
| T10 | linkedin-job-search | `firecrawl_search` | query: `site:linkedin.com/jobs/view "Software Engineer" United Kingdom`, tbs: `sbd:1,qdr:w`, limit 10 | >= 3 linkedin.com/jobs/view URLs | PASS (10 URLs) |
| T11 | linkedin-job-search | `Brightdata pro:scrape_as_markdown` | one URL from T10 (Checkout.com Senior SWE) | Full JD text with role + company + requirements | PASS (full JD, including company description, job description, requirements, seniority) |
| T12 | linkedin-job-search | `firecrawl_scrape` on linkedin.com URL | same URL as T11 | BLOCKED (documented behaviour) | PASS (block confirmed - "we do not support this site") |

**Overall:** 12 of 12 tests produced the expected outcome. All four skills are operational.

---

## LinkedIn Apify-free gate

The critical gate for LinkedIn publication was that T10 (FireCrawl job URL discovery) and T11 (Bright Data JD scrape) must both pass without needing an Apify key.

- **T10:** PASS - FireCrawl returned 10 valid `linkedin.com/jobs/view/<id>` URLs with titles and snippets.
- **T11:** PASS - Bright Data returned the full JD for one of those URLs, including company description, role description, requirements, seniority level, and employment type.

The LinkedIn skill is therefore published in this repo. It operates on FireCrawl + Bright Data alone; no Apify dependency.

---

## Notes on tests that intentionally fail

**T7 (FireCrawl on reddit.com):** FireCrawl explicitly refuses to scrape reddit.com. This is expected and documented in the `reddit-research` skill. The error message from FireCrawl is:

> "We apologize for the inconvenience but we do not support this site."

The reddit-research skill steers users to `Brightdata pro:web_data_reddit_posts` (primary) or `Brightdata pro:scrape_as_markdown` (fallback) for Reddit content.

**T12 (FireCrawl on linkedin.com):** Same block, same error message. The linkedin-job-search skill documents this and uses Bright Data for all LinkedIn JD scraping.

---

## Notes on dependencies

- **Bright Data MCP:** required for reddit-research (Step 2) and linkedin-job-search (Step 3). Without it those skills have no way to retrieve the content they need. Free tier: 5,000 requests/month for new MCP users - covers a weekly LinkedIn run plus Reddit lookups easily.
- **FireCrawl:** required for glassdoor-research (Step 2), linkedin-job-search (Step 1), and recommended for web-research fallbacks. Free tier: 500 credits one-time (no card) - sufficient to run the full test suite in this doc and still have credits left over.
- **Brave MCP:** optional everywhere. All four skills degrade gracefully to built-in `web_search` or FireCrawl when Brave is unavailable.

See each SKILL.md's Dependencies table at the top for full detail.

## Test run cost

The entire test run documented above used roughly:

- **FireCrawl:** 12 credits (4 searches at 2 credits each + 5 scrapes at 1-5 credits each).
- **Bright Data:** 2 requests (1 LinkedIn JD scrape + 1 Reddit thread fetch).

Both well inside the respective free tiers.

---

## Reproducing the tests

Run each test in a fresh Claude Code session with the relevant MCPs connected. Use benign public inputs (e.g. Google for Glassdoor, "Software Engineer United Kingdom" for LinkedIn). Do not run tests against private companies or identifying queries.

Tests involving Brave may be marked SKIPPED rather than PASS or FAIL if the Brave MCP server is unreachable - this is not a skill failure, just unavailability. All skills are designed to degrade silently when Brave is absent.
