# CLAUDE.md

Behavioral guidelines merged with career agent project context.

---

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:

* State your assumptions explicitly. If uncertain, ask.
* If multiple interpretations exist, present them - don't pick silently.
* If a simpler approach exists, say so. Push back when warranted.
* If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

* No features beyond what was asked.
* No abstractions for single-use code.
* No "flexibility" or "configurability" that wasn't requested.
* No error handling for impossible scenarios.
* If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:

* Don't "improve" adjacent code, comments, or formatting.
* Don't refactor things that aren't broken.
* Match existing style, even if you'd do it differently.
* If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:

* Remove imports/variables/functions that YOUR changes made unused.
* Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:

* "Tailor resume" → "Match keywords from job posting, render PDF, verify it compiles cleanly"
* "Fill form" → "Map all profile.yaml fields to form fields, verify before submitting"
* "Find jobs" → "Return only jobs matching criteria, confirm before applying"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

## Project: Career Agent

### Overview
A 3-phase career agent that tailors resumes, fills job applications, and scrapes job boards.

### Phases
- **Phase 1** — Tailor LaTeX resume to job posting → render to PDF
- **Phase 2** — Fill job application forms in browser using personal profile, generate cover letter if needed
- **Phase 3** — Scrape job boards, find suitable jobs, run Phase 1 + 2 automatically

### Tech Stack
- Mixed JS/Python
- latexmk for LaTeX → PDF rendering
- Playwright MCP for programmatic browser control
- gh for version controlling tailored resumes
- context7 MCP for live documentation

### Key Files
- `profile.yaml` → form-first personal data (work auth, visa, salary, start date, etc.)
- `criteria.yaml` → job search filters (title, keywords, location, salary range)
- `jobs.yaml` → state store tracking seen/matched/applied jobs
- `resume/` → LaTeX source files (.tex and .cls)
- `output/` → rendered PDFs per application

### Tools Available
- `latexmk` → compile resume to PDF
- `gh` → version control, one commit per tailored resume
- Playwright MCP → programmatic browser control (click, fill, navigate, screenshot)
- context7 MCP → live docs

### Critical Rules
- Never submit a form without explicit user confirmation
- Always pause at CAPTCHAs for manual solving
- Never modify .cls file unless explicitly asked
- Never fabricate experience or skills in resume tailoring
- One git commit per tailored resume with job title and company in commit message
- Always verify PDF compiles cleanly before Phase 1 is complete
- Phase 3 pauses and asks before triggering Phase 1 + 2 for any job

## Phase 1: Resume Tailoring

**Trigger:** Any prompt about tailoring, writing, or updating a resume for a role or company — exact wording doesn't matter.

**Instructions:** Read `playbooks/resume-tailoring.md` and follow it exactly. Read the file at the start of every Phase 1 run — do not rely on memory of its contents.

## Phase 2: Form Filling

**Trigger:** Any prompt to fill a job application form, or hand-over from Phase 3.

**Instructions:** Read `playbooks/form-filling-core.md` and follow it exactly. Read the file at the start of every Phase 2 run — do not rely on memory of its contents.

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
   - Location: is it in preferred_locations, or does work_arrangement allow it?
   - Salary: if shown, is it ≥ salary.minimum?
2. Set matching jobs to status `matched` in `jobs.yaml`
3. Present each matched job one at a time:
   "**[Title]** at **[Company]** — [Location] — [Salary or 'not listed']
   URL: [URL]
   Apply? (yes / no / skip)"
4. On `yes`: update status to `approved` → run Phase 1 for this job → run Phase 2 for this job
5. On `no`: update status to `rejected`
6. On `skip`: leave as `matched`, move to next