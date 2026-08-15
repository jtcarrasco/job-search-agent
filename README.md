# job_agent — Autonomous Job Search Agent

An autonomous Claude Code agent that searches multiple job boards, filters against
personal criteria, deduplicates results, and writes structured Markdown reports
directly to an Obsidian vault via Syncthing.

Built to solve a specific problem: manually checking job boards every day doesn't scale, and
most job-alert emails are noise. This runs unattended on a cron schedule, does the filtering
work a human would do (dedup, relevance, red flags), and only surfaces what's actually worth
looking at — in a plain Markdown report, not another dashboard to check.

**What's notable about the implementation:** there's no traditional application code here. The
entire agent — search strategy, filtering criteria, report formatting, dedup logic — is defined
in a single Markdown file (`CLAUDE.md`) that Claude Code reads and executes headlessly on every
run. No Python scraper, no scheduler framework, no database. The "code" is instructions, and the
runtime is an LLM with tool access. See [Architecture](#architecture) below for how that holds
together in practice.

---

## What It Does

Runs on a cron schedule (no manual trigger needed). Each run:

1. Reads search criteria and filters from Markdown knowledge files
2. Skips any posting already seen (rolling 90-day dedup list)
3. Queries job board APIs in sequence
4. Writes a dated Markdown report to the Obsidian vault
5. Updates its own task list to log completion

Three search types run on independent schedules:
- **W2 web dev** — Tue/Thu 8am UTC
- **AI** — Tue/Thu 9am UTC
- **Freelance/contract** — disabled (Mon/Wed/Fri 8am UTC schedule exists but is turned off)

Each run queries multiple sources, deduplicates against a rolling 90-day list, and writes
a dated Markdown report to the vault.

---

## Architecture

```
cron dispatcher (VPS)
    └── reads task-list.md for [trigger] or scheduled tasks
    └── invokes: claude --print --allowedTools "Read,Write,WebFetch,Bash" < CLAUDE.md

CLAUDE.md (agent instructions)
    ├── loads .env (API keys)
    ├── reads knowledge/ (criteria, dedup list, blacklist)
    ├── queries job board APIs
    ├── writes reports/[type]/YYYY-MM-DD.md
    └── updates task-list.md + knowledge/seen-postings.md

Syncthing
    └── syncs reports/ to MacBook Obsidian vault in real time
```

**Key design principle:** The agent is fully defined in a single Markdown file (`CLAUDE.md`).
No Python, no Node, no build step — just Claude reading instructions and calling APIs.

---

## Job Board Sources

| Source | Method | Coverage |
|---|---|---|
| We Work Remotely | RSS feed | Remote-first roles |
| JSearch (OpenWeb Ninja) | REST API (`api.openwebninja.com`) | LinkedIn, Indeed, ZipRecruiter, Glassdoor |
| Adzuna | REST API | Aggregated US job boards |
| EdJoin | WebFetch | Local South Bay school district postings |

---

## Report Output

Each report is a Markdown table with company, title, salary range, remote status,
date posted, and apply link — plus per-posting notes and a summary of hiring trends.

Example output (abbreviated):

```markdown
# W2 Job Search Report — 2026-03-05

| Company | Job Title | Remote | Salary Range | Date Posted | Apply Link |
|---------|-----------|--------|--------------|-------------|------------|
| Acme Co | Front-End Developer | Remote | $90k–$110k | 2026-03-03 | [link] |
...

## Summary
**Hiring trends:** Strong demand for WordPress + React hybrid roles...
**Standout roles:** Acme Co — strong culture signal, matches all criteria
```

---

## Tech Stack

- **Claude Code** — agent runtime (claude CLI, `--print` mode)
- **JSearch API (OpenWeb Ninja)** — job board aggregation (LinkedIn, Indeed, ZipRecruiter, Glassdoor)
- **Adzuna API** — supplementary job board data
- **We Work Remotely RSS** — remote-first listings
- **Syncthing** — vault sync (VPS → MacBook)
- **Cron** — scheduling on self-hosted VPS

---

## Files

```
job_agent/
├── CLAUDE.md                     # Agent instructions (source of truth)
├── task-list.md                  # Trigger and run log
├── .gitignore
├── knowledge/
│   ├── criteria-w2.md            # W2 web dev search filters
│   ├── criteria-freelance.md     # Freelance search filters
│   └── blacklist.md              # Excluded companies/terms
└── reports/                      # gitignored — generated output
    ├── w2/
    ├── ai/
    └── freelance/
```

---

## Engineering Notes

A few real problems this agent has had to handle in production, not just the happy path:

- **Rolling dedup with automatic pruning.** Every run prunes `seen-postings.md` entries older
  than 90 days before checking for duplicates, so the dedup list doesn't grow unbounded across
  months of daily/weekly runs.
- **Stuck-run recovery.** The dispatcher treats an `[in-progress]` task older than 6 hours as a
  failed run and re-triggers it automatically, rather than silently blocking the next scheduled
  run forever — this specifically fixed a real incident where a Claude CLI auth failure exited
  0 but never updated the task file, silently killing the weekly schedule.
- **Recruiter/aggregator noise filtering.** The agent recognizes patterns like a single staffing
  agency posting 6 near-identical "openings" with suspiciously precise salary figures across
  multiple runs, and flags it as one opportunity to verify rather than 6 distinct leads —
  learned from watching real recruiter behavior over months of runs, not a hardcoded rule.
- **Dead-source detection over false positives.** Sources that return zero results run after run
  (a defunct RSS feed, a permanently 403-blocking site) get dropped rather than silently retried
  forever — verified live (DNS resolution, HTTP status) before removal, not assumed.

## About This Repo

This is a curated excerpt of a private agent running on my own infrastructure. Included:
`CLAUDE.md` (the actual agent instructions, unmodified), this README, and two sanitized example
reports showing the real output format (`reports/`). Not included: the live search criteria,
dedup history, and API credentials — those are personal and stay private, but nothing about the
agent's logic or architecture is hidden here.

## Running Locally

Requires the [Claude CLI](https://docs.anthropic.com/en/docs/claude-code) and API keys
in a `.env` file (not included).

```bash
# Trigger a manual run
echo "[ ] W2 weekly run [trigger]" >> task-list.md

# Or invoke directly
claude --print --allowedTools "Read,Write,WebFetch,Bash" "$(cat CLAUDE.md)"
```
