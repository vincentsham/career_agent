# Requirements: Career Agent

**Defined:** 2026-05-13
**Core Value:** Eliminate the manual, repetitive work of applying to jobs while keeping the human in the loop for quality control — every submission is reviewed and confirmed before it goes out.

## v1.0 Requirements — Foundation

Requirements for milestone v1.0. Each maps to roadmap phases.

### Project Setup

- [x] **SETUP-01**: .gitignore excludes output/, *.pdf, and any sensitive YAML from version control
- [x] **SETUP-02**: output/ directory exists with .gitkeep so git tracks the directory structure
- [x] **SETUP-03**: WebFetch MCP permission is configured in .claude/settings.json

### Resume Source

- [x] **RESUME-01**: LaTeX resume source files (.tex and .cls) are present in resume/ directory

### Data Files

- [x] **DATA-01**: profile.yaml exists with form-first schema and placeholder values for all fields job applications ask for (name, email, phone, address, work authorization, visa status, salary expectation, remote preference, start date, LinkedIn URL, GitHub URL)
- [x] **DATA-02**: criteria.yaml exists with job search filter schema (title keywords, location, salary min/max, job type, company blacklist, required skills)
- [x] **DATA-03**: jobs.yaml exists as an initialized-empty application state store with schema documented in comments (id, company, role, url, status, applied_date, confirmation_id)

### CLAUDE.md Playbooks

- [x] **PLAY-01**: Phase 1 resume tailoring playbook is in CLAUDE.md — keyword extraction rules, no-fabrication guardrail, latexmk compile verification, git commit format ([Company] [Role] - tailored resume)
- [x] **PLAY-02**: Phase 2 form-filling playbook is in CLAUDE.md — profile.yaml field mapping, CAPTCHA pause behavior, screenshot-before-submit confirmation, cover letter generation when field present
- [x] **PLAY-03**: Phase 3 job board scraping playbook is in CLAUDE.md — criteria.yaml usage, deduplication against jobs.yaml, scoring against criteria, pause-and-confirm before triggering Phase 1+2

## Future Requirements

### Phase 1 — Resume Tailoring

- **TAIL-01**: Given a job posting URL or pasted text, Claude tailors the LaTeX resume to match keywords and requirements
- **TAIL-02**: Tailored resume compiles to PDF via latexmk without errors
- **TAIL-03**: PDF saved to output/<Company>-<Role>/resume.pdf
- **TAIL-04**: Commit created with [Company] [Role] - tailored resume message

### Phase 2 — Form Filling

- **FORM-01**: Claude opens application URL in Chrome, scans all fields, maps to profile.yaml
- **FORM-02**: Fills all field types: text, dropdown, checkbox, date, file upload (resume PDF)
- **FORM-03**: Generates cover letter when form includes a cover letter field
- **FORM-04**: Pauses at CAPTCHA and waits for manual solve before continuing
- **FORM-05**: Shows completed form screenshot to user and waits for explicit confirmation before submitting
- **FORM-06**: Captures confirmation ID after submission and writes to jobs.yaml

### Phase 3 — Job Board Scraping

- **SCRP-01**: Scrapes LinkedIn and Indeed using criteria.yaml search parameters
- **SCRP-02**: Navigates with human-like timing to avoid bot detection
- **SCRP-03**: Deduplicates listings against jobs.yaml before adding
- **SCRP-04**: Scores each listing against criteria.yaml (title, location, salary)
- **SCRP-05**: Presents matched jobs to user one at a time with apply/no/skip prompt
- **SCRP-06**: On approval: runs Phase 1 then Phase 2 for that job

## Out of Scope

| Feature | Reason |
|---------|--------|
| Multi-user support | Personal tool only — no auth, profiles, or user management |
| Custom runner script or orchestration code | Claude Code IS the orchestrator (Approach A) |
| Paid scraping APIs (Apify, ScrapingBee) | Chrome MCP with human-like timing only |
| Playwright/Puppeteer | Fingerprinted too easily; using Chrome DevTools MCP |
| Automatic submission without user confirmation | Always pause-and-confirm |
| Parsing or modifying .cls files | Never touched by automation |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| SETUP-01 | Phase 1: Project Structure | Complete |
| SETUP-02 | Phase 1: Project Structure | Complete |
| SETUP-03 | Phase 1: Project Structure | Complete |
| RESUME-01 | Phase 2: Resume Source and Data Files | Complete |
| DATA-01 | Phase 2: Resume Source and Data Files | Complete |
| DATA-02 | Phase 2: Resume Source and Data Files | Complete |
| DATA-03 | Phase 2: Resume Source and Data Files | Complete |
| PLAY-01 | Phase 3: CLAUDE.md Playbooks | Complete |
| PLAY-02 | Phase 3: CLAUDE.md Playbooks | Complete |
| PLAY-03 | Phase 3: CLAUDE.md Playbooks | Complete |

**Coverage:**
- v1.0 requirements: 10 total
- Mapped to phases: 10
- Unmapped: 0 ✓

---
*Requirements defined: 2026-05-13*
*Last updated: 2026-05-13 — traceability updated after roadmap creation*
