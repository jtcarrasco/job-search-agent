---
aliases: ["Job Search Agent"]
---

# CLAUDE.md — Job Search Agent (job_agent)

## Identity

This is job_agent. Primary function: weekly job search for remote roles.
Runs two searches: W2 web development and W2 AI-focused roles. Freelance run is disabled.

---

## Working Directory

All file operations are relative to this agent's folder:
`~/dev/_pkm/3_Resources/Agents/job_agent/`

---

## Schedule

<!-- schedule: day=2, hour=08, task=W2 weekly run -->
<!-- schedule: day=4, hour=08, task=W2 weekly run -->
<!-- schedule: day=2, hour=09, task=AI weekly run -->
<!-- schedule: day=4, hour=09, task=AI weekly run -->
<!-- DISABLED: schedule: day=1, hour=08, task=Freelance run -->
<!-- DISABLED: schedule: day=3, hour=08, task=Freelance run -->
<!-- DISABLED: schedule: day=5, hour=08, task=Freelance run -->

Day reference: 1=Mon 2=Tue 3=Wed 4=Thu 5=Fri 6=Sat 7=Sun. Time is UTC.

---

## Setup — Read Before Running

1. Load API keys from `.env` in this directory:
   - `JSEARCH_KEY` — RapidAPI key for JSearch (searches LinkedIn, Indeed, ZipRecruiter, Glassdoor)
   - `ADZUNA_APP_ID` + `ADZUNA_APP_KEY` — Adzuna job board API

2. Read `knowledge/criteria-w2.md` for W2 web dev search filters.
   Read `knowledge/criteria-ai.md` for W2 AI-focused search filters.
   Read `knowledge/criteria-freelance.md` for freelance search filters.
   If a criteria file is empty, fall back to the defaults in the Tasks section below.

3. Read `knowledge/seen-postings.md` to load the dedup list. Skip any posting
   whose URL or job ID already appears in that file.

4. Read `knowledge/blacklist.md` for companies and terms to exclude entirely.

---

## Tasks

### W2 weekly run

Search for remote full-time W2 web development roles posted in the last 7 days.

**Search queries:**
- "remote web developer"
- "remote front-end developer"
- "remote WordPress developer"
- "remote full-stack developer"

**Default criteria (use if criteria-w2.md is empty):**
- Remote or remote-friendly
- Web development / front-end focus (HTML, CSS, JS, PHP, WordPress, React, etc.)
- Prefer US-based companies or roles open to US applicants
- Exclude: pure backend/DevOps, design-only, non-English postings
- Exclude: anything in `knowledge/blacklist.md`

**Sources (in order):**
1. We Work Remotely RSS feeds:
   - `https://weworkremotely.com/remote-jobs.rss`
   - `https://weworkremotely.com/categories/remote-front-end-programming-jobs.rss`
   - `https://weworkremotely.com/categories/remote-programming-jobs.rss`
2. JSearch API (OpenWeb Ninja): `https://api.openwebninja.com/jsearch/search`
   - Header: `x-api-key: $JSEARCH_KEY`
   - Query params: `query=remote web developer`, `date_posted=week`, `work_from_home=true`
   - Note: JSearch aggregates LinkedIn, Glassdoor, Indeed, and ZipRecruiter — no need to scrape those sites directly
3. Adzuna API: `https://api.adzuna.com/v1/api/jobs/us/search/1`
   - Params: `app_id=$ADZUNA_APP_ID`, `app_key=$ADZUNA_APP_KEY`, `what=remote web developer`, `max_days_old=7`, `full_time=1`
   - Note: Adzuna does not accept `where=remote` — include "remote" in the `what` query instead; filter results by checking if title/description contains "remote"
4. EdJoin — local South Bay school district postings: WebFetch each URL, scan for tech/web roles
   - Redondo Beach USD: `https://www.edjoin.org/Home/Jobs?districtID=328&catID=0`
   - Hermosa Beach City SD: `https://www.edjoin.org/HermosaBeach`
   - Manhattan Beach USD: `https://www.edjoin.org/mbusd`
   - Note: pages may be JS-rendered; if WebFetch returns sparse content, note it and skip

**Output:** `reports/w2/YYYY-MM-DD.md`

---

### Freelance run

Search for remote freelance or contract web development work posted in the last 7 days.

**Search queries:**
- "remote freelance web developer"
- "remote contract front-end developer"
- "remote contract WordPress developer"

**Default criteria (use if criteria-freelance.md is empty):**
- Remote
- Freelance, contract, or project-based (not full-time W2)
- Web development / front-end focus
- Exclude: anything in `knowledge/blacklist.md`

**Sources (in order):**
1. We Work Remotely RSS — same feeds as W2 run, filter for contract/freelance titles
2. JSearch API (OpenWeb Ninja): `https://api.openwebninja.com/jsearch/search`
   - Header: `x-api-key: $JSEARCH_KEY`
   - Query params: `query=remote freelance web developer`, `date_posted=week`, `work_from_home=true`, `employment_types=CONTRACTOR`
   - Note: JSearch aggregates LinkedIn, Glassdoor, Indeed, and ZipRecruiter — no need to scrape those sites directly
3. Adzuna API: `https://api.adzuna.com/v1/api/jobs/us/search/1`
   - Params: `app_id=$ADZUNA_APP_ID`, `app_key=$ADZUNA_APP_KEY`, `what=remote freelance web developer`, `max_days_old=7`, `contract=1`
   - Note: do not use `where=remote` — it causes an error; use "remote" in the `what` query instead
4. EdJoin — local South Bay school district postings: WebFetch each URL, scan for contract/tech roles
   - Redondo Beach USD: `https://www.edjoin.org/Home/Jobs?districtID=328&catID=0`
   - Hermosa Beach City SD: `https://www.edjoin.org/HermosaBeach`
   - Manhattan Beach USD: `https://www.edjoin.org/mbusd`
   - Note: pages may be JS-rendered; if WebFetch returns sparse content, note it and skip

**Output:** `reports/freelance/YYYY-MM-DD.md`

---

### AI weekly run

Search for remote full-time W2 AI-focused roles posted in the last 7 days.

**Search queries:**
- "remote AI engineer"
- "remote prompt engineer"
- "remote AI product manager"
- "remote AI developer"
- "remote machine learning engineer"
- "remote AI solutions engineer"

**Default criteria (use if criteria-ai.md is empty):**
- Remote or remote-friendly
- AI/ML engineering, prompt engineering, AI product management, or AI integration roles
- Prefer roles where web dev or automation background is relevant (SaaS, agency, edtech, platform teams)
- Exclude: pure research/PhD-required roles, non-English postings, on-site only
- Exclude: anything in `knowledge/blacklist.md`

**Sources (in order):**
1. We Work Remotely RSS feeds:
   - `https://weworkremotely.com/remote-jobs.rss`
   - `https://weworkremotely.com/categories/remote-programming-jobs.rss`
   - `https://weworkremotely.com/categories/remote-management-finance-jobs.rss`
2. JSearch API (OpenWeb Ninja): `https://api.openwebninja.com/jsearch/search`
   - Header: `x-api-key: $JSEARCH_KEY`
   - Query params: `query=remote AI engineer`, `date_posted=week`, `work_from_home=true`
   - Run a second query: `query=remote prompt engineer`, `date_posted=week`, `work_from_home=true`
   - Note: JSearch aggregates LinkedIn, Glassdoor, Indeed, and ZipRecruiter — no need to scrape those sites directly
3. Adzuna API: `https://api.adzuna.com/v1/api/jobs/us/search/1`
   - Params: `app_id=$ADZUNA_APP_ID`, `app_key=$ADZUNA_APP_KEY`, `what=remote AI engineer`, `max_days_old=7`, `full_time=1`
   - Note: do not use `where=remote` — it causes an error; use "remote" in the `what` query instead

**Output:** `reports/ai/YYYY-MM-DD.md`

---

## Report Format

Each report must include:

```
# [W2 / Freelance] Job Search Report — YYYY-MM-DD

**Run date:** YYYY-MM-DD
**Search window:** Last 7 days (YYYY-MM-DD – YYYY-MM-DD)
**Sources checked:** [list sources checked, note any skipped and why]

> [Note if fewer than 10 results found, or anything notable about the run]

---

## Results

| Company | Job Title | Industry | Remote/Hybrid/Onsite | Location | Experience Level | Salary Range | Date Posted | Apply Link |
|---------|-----------|----------|----------------------|----------|------------------|--------------|-------------|------------|
...

---

## Posting Notes

[For each result: 2–4 sentences on company, role fit, stack, employment type, and any red flags.
Mark standout roles as **STANDOUT**.]

---

## Summary

**Hiring trends this week:** [1–2 sentences]
**Most commonly requested skills:** [comma list]
**Standout companies or roles worth prioritizing:** [bullet list]
**Action recommended:** [anything the operator should know or do]
```

---

## Dedup Rules

**Format:** one entry per line: `YYYY-MM-DD https://...`

**At the start of every run:**
1. Read `knowledge/seen-postings.md`.
2. Remove any lines where the date is older than 90 days from today.
3. Write the pruned list back to the file.

**During the run:**
4. Skip any posting whose URL already appears in the (pruned) list.

**After writing a report:**
5. Append all new posting URLs to `knowledge/seen-postings.md` in `YYYY-MM-DD URL` format, using today's date.
6. Delete any temp RSS files left in `~/` matching `wwr_*.rss`.

---

## Output Rules

- Always write reports as dated markdown files: `YYYY-MM-DD.md`
- Never overwrite a previous report — if today's file exists, append a `-2` suffix
- Keep tables clean and scannable in Obsidian
- If fewer than 5 results found, note it at the top of the report
- If an API or source fails, log the error and continue with remaining sources — never fail silently
- If rate limited, note which source was skipped and continue
- Update `task-list.md` after every run — mark tasks `[done]` with timestamp and result count

---

## Error Handling

- Missing `.env` or missing API keys: skip that source, note in report header
- HTTP errors from any source: log status code, skip that source, continue
- Empty result from a source: note it, continue
- If all sources fail: write a report with zero results and a clear error summary

---

## Notes

<!-- Add any persistent notes or context here -->
