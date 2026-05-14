# Career Agent — Skill Matrix

**Date:** 2026-05-13  
**Scope:** All skills needed to complete the 3-phase career agent — what exists, what needs configuring, what needs to be built.

---

## Orchestration Model

Claude Code acts as the sole orchestrator. No separate runner script. Custom skills are encoded as CLAUDE.md instruction sets and YAML config/state files.

---

## Skills Already Available

| Skill | Tool | Used In |
|---|---|---|
| Compile LaTeX → PDF | `latexmk` | Phase 1 |
| Version control tailored resumes | `gh` | Phase 1 |
| Read .tex / .cls / YAML files | Claude Code native `Read` tool | All |
| Programmatic browser control (click, fill, navigate, screenshot) | `Playwright MCP` — `@playwright/mcp` with `--user-data-dir ./job_search_profile --headed`; persists login sessions across runs | Phase 2, 3 |
| Fetch job posting from URL | `WebFetch` (Claude Code built-in) | Phase 1, 3 |
| AI tailoring, matching, form mapping | Claude Code itself | All |
| Live library docs | `context7 MCP` | As needed |

---

## Skills to Install / Configure

| Skill | Action |
|---|---|
| Allow `WebFetch` in permissions | Add to `.claude/settings.json` |
| Playwright MCP permissions | Add `mcp__playwright__browser_*` entries to `.claude/settings.local.json` |
| Log in to job boards via Playwright | One-time manual login per board (LinkedIn, Indeed) — sessions persist in `./job_search_profile` user data dir |

---

## Skills to Build Custom

| Skill | What it is | Phase |
|---|---|---|
| `profile.yaml` | Form-first schema: fields defined by what job applications ask for (work auth, visa status, start date, salary expectations, remote preference, LinkedIn URL, etc.) — populated by extracting from .tex where possible, user fills the rest | 2, 3 |
| `criteria.yaml` | Job search filters: title, keywords, location, salary range | 3 |
| `jobs.yaml` | State store: tracks seen / matched / applied jobs to prevent duplicates | 3 |
| Resume tailoring guardrails | CLAUDE.md rules: no fabrication, never modify .cls, only rephrase/highlight/reorder existing content | 1 |
| Form-filling instructions | CLAUDE.md playbook: scan fields → map profile.yaml → fill by type (text, dropdown, checkbox, date) → upload resume PDF via file input field → handle cover letter field → detect CAPTCHA → confirm before submit → capture confirmation → write to jobs.yaml | 2 |
| Cover letter generator | CLAUDE.md instructions: when a form has a cover letter field, generate a tailored cover letter from profile.yaml + job description, respecting any character/word limit | 2 |
| Job board scraper instructions | CLAUDE.md playbook for job boards: LinkedIn + Indeed as primary, extensible to others — search query format per platform, pagination, human-like timing, CAPTCHA pause, platform-specific selectors | 3 |

---

## Key Constraints

- Never submit a form without explicit user confirmation
- Always pause at CAPTCHAs for manual solving
- Never modify .cls file unless explicitly asked
- One git commit per tailored resume with job title and company in commit message
- Always verify PDF compiles cleanly before Phase 1 is complete
- Phase 3 pauses and asks before triggering Phase 1 + 2 for any job
