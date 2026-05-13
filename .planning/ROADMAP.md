# Roadmap: Career Agent

**Milestone:** v1.0 Foundation
**Granularity:** Coarse
**Coverage:** 10/10 v1.0 requirements mapped

## Phases

- [x] **Phase 1: Project Structure** - Git, output directory, and WebFetch MCP configured
- [x] **Phase 2: Resume Source and Data Files** - LaTeX files in place and all YAML files created
- [x] **Phase 3: CLAUDE.md Playbooks** - All three phase playbooks written into CLAUDE.md

## Phase Details

### Phase 1: Project Structure
**Goal**: The project directory is correctly wired for version control, PDF output, and browser access
**Depends on**: Nothing (first phase)
**Requirements**: SETUP-01, SETUP-02, SETUP-03
**Success Criteria** (what must be TRUE):
  1. `git status` does not surface output/, any .pdf file, or sensitive YAML as tracked
  2. `output/` directory exists in the repo with a .gitkeep file
  3. `.claude/settings.json` contains a WebFetch permission entry that allows Claude to fetch URLs
**Plans**: 1 plan
  - [x] 01-01-PLAN.md — Wire .gitignore (output/, *.pdf), create tracked output/.gitkeep, and add .claude/settings.json WebFetch permission

### Phase 2: Resume Source and Data Files
**Goal**: All source material and data files Claude needs to run the pipeline are present and correctly structured
**Depends on**: Phase 1
**Requirements**: RESUME-01, DATA-01, DATA-02, DATA-03
**Success Criteria** (what must be TRUE):
  1. `resume/` contains a .tex file and a .cls file that compile without errors via `latexmk`
  2. `profile.yaml` exists with labeled placeholder values for every field job applications ask for (name, email, phone, address, work auth, visa, salary, remote pref, start date, LinkedIn, GitHub)
  3. `criteria.yaml` exists with all job search filter fields present (title keywords, location, salary min/max, job type, company blacklist, required skills)
  4. `jobs.yaml` exists, is initialized empty, and has schema documented in comments
**Plans**: TBD

### Phase 3: CLAUDE.md Playbooks
**Goal**: CLAUDE.md contains complete, actionable playbooks for all three pipeline phases so Claude can execute each phase without ambiguity
**Depends on**: Phase 2
**Requirements**: PLAY-01, PLAY-02, PLAY-03
**Success Criteria** (what must be TRUE):
  1. CLAUDE.md contains a Phase 1 section with keyword extraction rules, a no-fabrication guardrail, latexmk compile verification step, and git commit format
  2. CLAUDE.md contains a Phase 2 section covering profile.yaml field mapping, CAPTCHA pause behavior, and an explicit screenshot-before-submit confirmation step
  3. CLAUDE.md contains a Phase 2 cover letter subsection with generation rules for when a cover letter field is present
  4. CLAUDE.md contains a Phase 3 section covering criteria.yaml usage, deduplication against jobs.yaml, scoring logic, and pause-before-triggering-Phase-1+2 behavior
**Plans**: TBD

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Project Structure | 1/1 | Complete | 2026-05-13 |
| 2. Resume Source and Data Files | 1/1 | Complete | 2026-05-13 |
| 3. CLAUDE.md Playbooks | 1/1 | Complete | 2026-05-13 |

---
*Roadmap created: 2026-05-13 — Milestone v1.0 Foundation*
*Phase 1 planned: 2026-05-13 — 1 plan, 2 tasks*
