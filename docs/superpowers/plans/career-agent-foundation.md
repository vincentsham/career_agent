# Career Agent Foundation Setup — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create all config files, YAML schemas, and CLAUDE.md instruction sets needed before Phase 1 (resume tailoring), Phase 2 (form filling), or Phase 3 (job scraping) can run.

**Architecture:** No application code. All custom skills are files: YAML schemas for data/state, CLAUDE.md sections for behavioral instructions. Phase 1, 2, 3 implementation plans are written separately after this foundation is in place.

**Tech Stack:** YAML, Markdown, latexmk (PDF verification), Python3 (YAML syntax validation)

---

## File Map

| File | Responsibility |
|---|---|
| `.claude/settings.json` | Grant WebFetch permission |
| `.gitignore` | Exclude LaTeX build artifacts; keep PDFs |
| `resume/` | LaTeX source files (.tex, .cls) |
| `output/` | Compiled PDFs, one subdirectory per application |
| `profile.yaml` | Form-first personal data (Phase 2 + 3) |
| `criteria.yaml` | Job search filters (Phase 3) |
| `jobs.yaml` | Application state store (Phase 3) |
| `CLAUDE.md` | Append Phase 1, 2, 3 behavioral instruction sets |

---

## Prerequisites

**Log in to Chrome for each job board before running Phase 2 or Phase 3.** Sessions must be active; the scraper does not handle login flows.

1. Open Chrome and go to `https://www.linkedin.com` — log in if not already
2. Open `https://www.indeed.com` — log in if not already
3. For other job boards you plan to scrape, log in to each one
4. Keep Chrome open between sessions so cookies persist

This is a one-time manual step per board (until the session expires).

---

### Task 1: Project structure, .gitignore, and WebFetch permission

**Files:**
- Create: `resume/.gitkeep`
- Create: `output/.gitkeep`
- Create: `.gitignore`
- Create: `.claude/settings.json`

- [ ] **Step 1: Create directories**

```bash
mkdir -p resume output .claude && touch resume/.gitkeep output/.gitkeep
```

Expected: No errors. `ls` shows `resume/` and `output/` directories.

- [ ] **Step 2: Create .gitignore**

Create `.gitignore`:

```
# LaTeX build artifacts
resume/*.aux
resume/*.log
resume/*.fls
resume/*.fdb_latexmk
resume/*.synctex.gz
resume/*.out
resume/*.toc
resume/*.bbl
resume/*.blg

# Machine-specific MCP config
.mcp.json
```

- [ ] **Step 3: Create .claude/settings.json**

Create `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "WebFetch"
    ]
  }
}
```

- [ ] **Step 4: Verify JSON is valid**

```bash
python3 -c "import json; json.load(open('.claude/settings.json')); print('Valid JSON')"
```

Expected: `Valid JSON`

- [ ] **Step 5: Commit**

```bash
git add resume/.gitkeep output/.gitkeep .gitignore .claude/settings.json
git commit -m "chore: project structure, gitignore, and WebFetch permission"
```

---

### Task 2: Copy resume files and verify compilation

**Files:**
- Create: `resume/<your-resume>.tex`
- Create: `resume/<your-style>.cls`

- [ ] **Step 1: Copy your .tex and .cls files into resume/**

Replace the paths below with your actual file locations:

```bash
cp /path/to/your/resume.tex resume/
cp /path/to/your/resume.cls resume/
```

Expected: `ls resume/` shows your .tex and .cls files (plus .gitkeep).

- [ ] **Step 2: Test compilation**

```bash
cd resume && latexmk -pdf -interaction=nonstopmode *.tex 2>&1 | tail -10
```

Expected output ends with:
```
Latexmk: All targets (resume.pdf) are up-to-date.
```

If you see errors starting with `!`, run `grep -A3 "^!" resume/*.log` to see what failed and fix the .tex file before continuing.

- [ ] **Step 3: Verify PDF was created**

```bash
ls -lh resume/*.pdf
```

Expected: At least one `.pdf` file listed with a non-zero size.

- [ ] **Step 4: Remove compiled PDF (source-controlled separately in output/)**

```bash
cd resume && latexmk -C
```

Expected: Cleans build artifacts. `ls resume/` shows only .tex, .cls, .gitkeep.

- [ ] **Step 5: Commit**

```bash
git add resume/
git commit -m "chore: add LaTeX resume source files"
```

---

### Task 3: Create profile.yaml

**Files:**
- Create: `profile.yaml`

- [ ] **Step 1: Create profile.yaml with the form-first schema**

Create `profile.yaml` — fill every field with your real data before using Phase 2 or 3:

```yaml
# profile.yaml — Form-first personal data
# Fill all fields before using Phase 2 or Phase 3.

personal:
  first_name: "Your First Name"
  last_name: "Your Last Name"
  email: "your@email.com"
  phone: "+1-555-000-0000"
  linkedin: "https://linkedin.com/in/yourhandle"
  github: "https://github.com/yourhandle"   # leave empty string if none
  website: ""                                 # leave empty string if none
  address:
    street: "123 Main St"
    city: "San Francisco"
    state: "CA"
    zip: "94105"
    country: "United States"

headline: "Software Engineer | 5 Years Experience | Python, TypeScript"

work_authorization:
  country: "United States"
  authorized: true             # true = no sponsorship needed
  requires_sponsorship: false
  visa_type: "Citizen"         # Citizen, Green Card, H1B, OPT, TN, etc.
  sponsorship_notes: ""

availability:
  start_date: "2 weeks"        # "Immediately", "2 weeks", "1 month", or "YYYY-MM-DD"
  job_type: "full-time"        # full-time, part-time, contract
  remote_preference: "hybrid"  # remote, hybrid, onsite, flexible
  willing_to_relocate: false
  preferred_locations:
    - "San Francisco, CA"
    - "Remote"

compensation:
  salary_min: 120000
  salary_max: 160000
  currency: "USD"
  open_to_equity: true
  open_to_bonus: true

experience:
  years_total: 5
  current_title: "Software Engineer"
  current_company: "Current Company Inc."

work_history:
  - company: "Current Company Inc."
    title: "Software Engineer"
    location: "San Francisco, CA"
    start_date: "2022-01"
    end_date: "Present"
    description: "One sentence describing your role and key impact"
  - company: "Previous Company LLC"
    title: "Junior Software Engineer"
    location: "New York, NY"
    start_date: "2019-06"
    end_date: "2021-12"
    description: "One sentence describing your role and key impact"

education:
  - school: "University of California, Berkeley"
    degree: "B.S."
    field: "Computer Science"
    start_year: 2015
    end_year: 2019
    gpa: ""   # leave empty string if not sharing

skills:
  programming_languages:
    - Python
    - TypeScript
  frameworks:
    - React
    - FastAPI
  tools:
    - Docker
    - AWS
    - PostgreSQL
  certifications: []

background:
  willing_background_check: true
  felony_record: false

diversity:   # All optional — only fill what you choose to disclose
  gender: ""
  ethnicity: ""
  veteran_status: "Not a veteran"
  disability_status: "No disability"

references:
  available_on_request: true
```

- [ ] **Step 2: Fill in your real data**

Replace every placeholder value with your actual information. Pay special attention to:
- `work_authorization` — visa_type and requires_sponsorship
- `availability` — start_date and remote_preference
- `compensation` — salary_min and salary_max
- `work_history` — must match your .tex resume exactly (no fabrication)

- [ ] **Step 3: Validate YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('profile.yaml')); print('Valid YAML')"
```

Expected: `Valid YAML`

If you see a `yaml.scanner.ScannerError`, check the line number in the error and fix the indentation or quotes on that line.

- [ ] **Step 4: Commit**

```bash
git add profile.yaml
git commit -m "chore: add profile.yaml with personal data schema"
```

---

### Task 4: Create criteria.yaml

**Files:**
- Create: `criteria.yaml`

- [ ] **Step 1: Create criteria.yaml**

Create `criteria.yaml` — fill with your actual job search preferences:

```yaml
# criteria.yaml — Job search filters for Phase 3

search:
  titles:
    - "Software Engineer"
    - "Senior Software Engineer"
    - "Full Stack Engineer"
  keywords:
    - "Python"
    - "TypeScript"
  locations:
    - "San Francisco, CA"
    - "New York, NY"
  include_remote: true

salary:
  minimum: 120000
  currency: "USD"

experience_level:
  - "mid"
  - "senior"
  # Options: entry, junior, mid, senior, staff, principal

date_posted_days: 30   # Only jobs posted within this many days

companies:
  exclude: []          # Companies to never apply to (e.g. ["Company A", "Company B"])
  prefer: []           # Companies to prioritize (optional)

easy_apply_only: false # true = only scrape LinkedIn Easy Apply jobs
```

- [ ] **Step 2: Fill in your actual criteria**

Update titles, keywords, locations, and salary_minimum to match your search.

- [ ] **Step 3: Validate YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('criteria.yaml')); print('Valid YAML')"
```

Expected: `Valid YAML`

- [ ] **Step 4: Commit**

```bash
git add criteria.yaml
git commit -m "chore: add criteria.yaml with job search filters"
```

---

### Task 5: Initialize jobs.yaml

**Files:**
- Create: `jobs.yaml`

- [ ] **Step 1: Create jobs.yaml**

Create `jobs.yaml`:

```yaml
# jobs.yaml — Application state store
# Managed automatically by Phase 3. Do not edit manually during a session.
#
# Statuses: seen | matched | approved | applied | rejected
#
# Schema for each entry (auto-populated by Phase 3):
# - id: "<company>-<title>-<hash>"     # deduplication key
#   title: ""
#   company: ""
#   url: ""
#   board: ""                           # linkedin, indeed, etc.
#   location: ""
#   salary: ""                          # as shown on listing, or ""
#   status: "seen"
#   seen_at: ""                         # ISO 8601 datetime
#   applied_at: ""                      # ISO 8601 datetime, or null
#   resume_path: ""                     # path to tailored PDF, or ""
#   confirmation_id: ""                 # application confirmation ID, or ""
#   notes: ""

jobs: []
```

- [ ] **Step 2: Validate YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('jobs.yaml')); print('Valid YAML')"
```

Expected: `Valid YAML`

- [ ] **Step 3: Commit**

```bash
git add jobs.yaml
git commit -m "chore: initialize jobs.yaml state store"
```

---

### Task 6: Add Phase 1 (resume tailoring) instructions to CLAUDE.md

**Files:**
- Modify: `CLAUDE.md` (append at end)

- [ ] **Step 1: Append Phase 1 section to CLAUDE.md**

Add the following to the end of `CLAUDE.md`:

```markdown
## Phase 1: Resume Tailoring

### Process
1. Get job posting: if URL, fetch with WebFetch; if pasted text, use as-is
2. Read resume source files from `resume/`
3. Identify in the posting: required skills, preferred skills, keywords, role focus
4. Tailor the resume:
   - Rephrase bullet points to include keywords from the posting
   - Reorder bullets within each role to put most relevant work first
   - Update the professional summary to match the role
   - Surface skills from work history that match the posting but aren't currently listed
5. Compile: `cd resume && latexmk -pdf -interaction=nonstopmode *.tex`
6. If compilation fails: run `grep -A3 "^!" resume/*.log` to find the error, fix it, retry
7. Copy compiled PDF to: `output/<Company>-<Role>/resume.pdf`
8. Commit: `git add output/<Company>-<Role>/resume.pdf resume/*.tex && git commit -m "[Company] [Role] - tailored resume"`

### Guardrails — NEVER do these
- Fabricate companies, titles, dates, metrics, technologies, or achievements
- Modify any `.cls` file
- Add skills or experience the user has not listed in their history
- Change document structure (section names, section count) without asking
```

- [ ] **Step 2: Verify the section was added**

```bash
grep -c "Phase 1: Resume Tailoring" CLAUDE.md
```

Expected: `1`

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add Phase 1 resume tailoring instructions to CLAUDE.md"
```

---

### Task 7: Add Phase 2 (form filling + cover letter) instructions to CLAUDE.md

**Files:**
- Modify: `CLAUDE.md` (append at end)

- [ ] **Step 1: Append Phase 2 sections to CLAUDE.md**

Add the following to the end of `CLAUDE.md`:

```markdown
## Phase 2: Form Filling

### Before filling
1. Open the application URL in Chrome
2. Take a screenshot to confirm the page loaded correctly
3. Scroll through the entire form before touching any field
4. Note all required fields (marked with *)

### Filling order
1. Personal info → use `profile.yaml` personal section
2. Work authorization → use `profile.yaml` work_authorization section
3. Work history → use `profile.yaml` work_history (most recent first)
4. Education → use `profile.yaml` education section
5. Skills → use `profile.yaml` skills section
6. Resume upload → locate `input[type="file"]` → upload `output/<Company>-<Role>/resume.pdf`
7. Cover letter → if field present, follow Phase 2: Cover Letter section below
8. Voluntary disclosures → use `profile.yaml` diversity section

### Rules
- Dropdowns: pick the closest matching option; if genuinely ambiguous, stop and ask
- Required field with no profile.yaml match: STOP → take screenshot → ask user
- Optional field with no match: leave blank
- Multi-page forms: fill page → scroll to top to check for red errors → click Next

### CAPTCHA
If CAPTCHA detected (image challenge, checkbox, Cloudflare wall): STOP → take screenshot → tell user "CAPTCHA at [URL] — please solve it" → wait for user confirmation → continue

### Submitting
1. Take a full-page screenshot of the completed form
2. Show user: "Ready to submit to [Company] for [Role]. Review the screenshot and type 'submit' to confirm."
3. Submit ONLY after user explicitly confirms
4. After submit: take screenshot of confirmation page → extract confirmation ID → update `jobs.yaml` entry: set status to `applied`, set applied_at, set confirmation_id

## Phase 2: Cover Letter Generator

When a form has a cover letter text field:
1. Check the field for a character or word limit — if found, note it before writing
2. Read the full job description and `profile.yaml`
3. Write 3–4 paragraphs (~300 words unless a limit applies):
   - Opening: one specific, genuine reason this company or role is interesting
   - Body paragraph 1: most relevant experience with a concrete result or metric from work_history
   - Body paragraph 2: second most relevant skill, project, or quality
   - Closing: one sentence of enthusiasm + "I'd welcome the opportunity to discuss further"
4. Tone: direct, specific, first person — no filler phrases ("I am writing to express my interest...")
5. Never copy resume bullet points verbatim; add narrative context instead
6. Never fabricate metrics, outcomes, or experiences not in profile.yaml
```

- [ ] **Step 2: Verify both sections were added**

```bash
grep -c "Phase 2:" CLAUDE.md
```

Expected: `2` (Form Filling and Cover Letter Generator)

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add Phase 2 form filling and cover letter instructions to CLAUDE.md"
```

---

### Task 8: Add Phase 3 (job board scraper) instructions to CLAUDE.md

**Files:**
- Modify: `CLAUDE.md` (append at end)

- [ ] **Step 1: Append Phase 3 section to CLAUDE.md**

Add the following to the end of `CLAUDE.md`:

```markdown
## Phase 3: Job Board Scraping

### Before starting
1. Read `criteria.yaml` for search parameters
2. Read `jobs.yaml` to get the list of already-seen companies+titles (for deduplication)

### Search URLs
Construct search URLs using criteria.yaml values (URL-encode spaces as `+`):

- LinkedIn: `https://www.linkedin.com/jobs/search/?keywords=<title>&location=<location>&f_TPR=r2592000`
- Indeed: `https://www.indeed.com/jobs?q=<title>+<keywords>&l=<location>&fromage=30`
- Other boards: construct equivalent search URLs using the same title/location/date parameters

### Per-listing behavior
1. Wait 2–4 seconds (random) between page loads — vary the delay each time
2. Scroll the page slowly for 1–2 seconds before clicking anything
3. Extract from listing card: title, company, location, salary (if shown), URL
4. Deduplication check: if an entry in `jobs.yaml` already has the same company + title, skip this listing
5. Click into full listing → wait 1–2 seconds → extract full job description
6. Write a new entry to `jobs.yaml` with status `seen`

### Anti-bot rules
- Maximum 25 listings per search session
- After every 15 listings: pause 45–90 seconds before continuing
- If redirected to a login page: STOP → ask user to log in manually → wait for confirmation
- If CAPTCHA appears: STOP → take screenshot → ask user to solve manually
- Never open more than 2 browser tabs at once

### Matching and presenting to user
After collecting all listings for the session:
1. For each `seen` job, score against `criteria.yaml`:
   - Title: does it match any of the target titles (exact or close)?
   - Location: is it in preferred_locations, or remote if include_remote is true?
   - Salary: if shown, is it ≥ salary.minimum?
2. Set matching jobs to status `matched` in `jobs.yaml`
3. Present each matched job one at a time:
   "**[Title]** at **[Company]** — [Location] — [Salary or 'not listed']
   URL: [URL]
   Apply? (yes / no / skip)"
4. On `yes`: update status to `approved` → run Phase 1 for this job → run Phase 2 for this job
5. On `no`: update status to `rejected`
6. On `skip`: leave as `matched`, move to next
```

- [ ] **Step 2: Verify the section was added**

```bash
grep -c "Phase 3: Job Board Scraping" CLAUDE.md
```

Expected: `1`

- [ ] **Step 3: Confirm CLAUDE.md has all three phase sections**

```bash
grep "^## Phase" CLAUDE.md
```

Expected output:
```
## Phase 1: Resume Tailoring
## Phase 2: Form Filling
## Phase 2: Cover Letter Generator
## Phase 3: Job Board Scraping
```

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add Phase 3 job board scraper instructions to CLAUDE.md"
```

---

## Done

After all 8 tasks complete, the foundation is in place:

- `profile.yaml` populated with your personal data
- `criteria.yaml` configured with your job search filters
- `jobs.yaml` initialized and ready to track applications
- `CLAUDE.md` contains behavioral instructions for all three phases
- `resume/` contains your LaTeX source files
- WebFetch is permitted in `.claude/settings.json`

**Next plans (in order):**
1. Phase 1 implementation — resume tailoring + PDF render pipeline
2. Phase 2 implementation — browser form filling end-to-end
3. Phase 3 implementation — job board scraping + matching loop
