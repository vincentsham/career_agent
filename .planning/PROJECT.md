# Career Agent

## Current Milestone: v1.0 Foundation

**Goal:** Set up all data files, project structure, and CLAUDE.md playbooks needed for the three-phase career agent pipeline.

**Target features:**
- Project structure (.gitignore, output/ dir, WebFetch permission)
- LaTeX resume source files in resume/
- profile.yaml with form-first schema and placeholder values
- criteria.yaml with job search filter schema
- jobs.yaml initialized as application state store
- CLAUDE.md Phase 1 playbook: resume tailoring guardrails
- CLAUDE.md Phase 2 playbook: form-filling + cover letter generator
- CLAUDE.md Phase 3 playbook: job board scraping instructions

## What This Is

A personal AI agent that automates the job search pipeline. Given a job posting (URL or pasted text), it tailors a LaTeX resume to the role, compiles it to PDF, fills the online application form using personal profile data, and — in fully autonomous mode — scrapes LinkedIn, Indeed, and other job boards to find matching roles and trigger the full pipeline with user approval at each step.

## Core Value

Eliminate the manual, repetitive work of applying to jobs while keeping the human in the loop for quality control — every submission is reviewed and confirmed before it goes out.

## Requirements

### Validated

(None yet — ship to validate)

### Active

**Foundation**
- [ ] Project structure, .gitignore, and WebFetch permission configured
- [ ] LaTeX resume source files added to `resume/`
- [ ] `profile.yaml` schema created and populated (form-first: work auth, visa, salary, remote pref)
- [ ] `criteria.yaml` created with job search filters
- [ ] `jobs.yaml` initialized as application state store
- [ ] Phase 1 resume tailoring guardrails added to CLAUDE.md
- [ ] Phase 2 form-filling playbook added to CLAUDE.md
- [ ] Phase 2 cover letter generator added to CLAUDE.md
- [ ] Phase 3 job board scraper playbook added to CLAUDE.md

**Phase 1 — Resume Tailoring**
- [ ] Given a job posting (URL or paste), Claude tailors the LaTeX resume to match keywords and requirements
- [ ] Tailored resume compiles to PDF via latexmk without errors
- [ ] PDF saved to `output/<Company>-<Role>/resume.pdf`
- [ ] Commit created with `[Company] [Role] - tailored resume` message

**Phase 2 — Form Filling**
- [ ] Claude opens application URL in Chrome, scans all fields, maps to profile.yaml
- [ ] Fills all field types: text, dropdown, checkbox, date, file upload (resume PDF)
- [ ] Generates cover letter when form includes a cover letter field
- [ ] Pauses at CAPTCHA and waits for manual solve before continuing
- [ ] Shows completed form screenshot to user and waits for explicit confirmation before submitting
- [ ] Captures confirmation ID after submission and writes to jobs.yaml

**Phase 3 — Job Board Scraping**
- [ ] Scrapes LinkedIn and Indeed using criteria.yaml search parameters
- [ ] Navigates with human-like timing to avoid bot detection
- [ ] Deduplicates listings against jobs.yaml before adding
- [ ] Scores each listing against criteria.yaml (title, location, salary)
- [ ] Presents matched jobs to user one at a time with apply/no/skip prompt
- [ ] On approval: runs Phase 1 then Phase 2 for that job

### Out of Scope

- Multi-user support — personal tool only; no auth, profiles, or user management
- Custom runner script or orchestration code — Claude Code IS the orchestrator (Approach A)
- Paid scraping APIs (Apify, ScrapingBee) — Chrome MCP with human-like timing only
- Playwright/Puppeteer — fingerprinted too easily; using Chrome DevTools MCP
- Automatic submission without user confirmation — always pause-and-confirm
- Parsing or modifying `.sty` files — never touched by automation

## Context

- Resume exists as LaTeX source (.tex + .sty) — not yet in the repo; will be copied to `resume/`
- `profile.yaml` doesn't exist yet and must be built from scratch using a form-first schema (what job applications ask for, not what the resume contains)
- Chrome DevTools MCP (programmatic CDP control) and Claude in Chrome (page-aware extension) are both available and serve different roles — CDP for automation, extension for context
- LinkedIn and Indeed are primary scraping targets; both have aggressive bot detection requiring human-like behavior
- All "skills" in this project are CLAUDE.md instruction sets and YAML files — no application code is written

## Constraints

- **Safety**: Never submit a form without explicit user confirmation — Claude pauses and shows screenshot first
- **Accuracy**: Never fabricate resume content — only rephrase/reorder/highlight existing content
- **Orchestration**: Claude Code as sole orchestrator — no separate runner scripts or daemons
- **State**: File-based only (YAML) — no database, no external services
- **Anti-bot**: Chrome MCP + human-like timing only — no paid proxy services
- **Version control**: One git commit per tailored resume, message includes company + role

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Claude Code as sole orchestrator (Approach A) | Zero infra to build; conversational pause-confirm loop is native to Claude Code | — Pending |
| Chrome DevTools MCP over Playwright | Less fingerprinted; real browser instance harder to detect | — Pending |
| File-based state (YAML) over SQLite | Simpler, human-readable, no dependency | — Pending |
| LinkedIn + Indeed as primary boards | Highest volume; others extensible | — Pending |
| form-first profile.yaml schema | Job applications ask for fields not in resume (visa, salary, remote pref) | — Pending |
| Pause-before-apply in Phase 3 | Never apply without user seeing the job first | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-13 — Milestone v1.0 Foundation started*
