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
- Chrome DevTools MCP for programmatic browser control
- Claude in Chrome for page-aware browsing
- filesystem MCP for reading local files
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
- Chrome DevTools MCP → programmatic browser control (click, fill, navigate)
- Claude in Chrome → page-aware browsing and interaction
- filesystem MCP → read local files
- context7 MCP → live docs

### Critical Rules
- Never submit a form without explicit user confirmation
- Always pause at CAPTCHAs for manual solving
- Never modify .cls file unless explicitly asked
- Never fabricate experience or skills in resume tailoring
- One git commit per tailored resume with job title and company in commit message
- Always verify PDF compiles cleanly before Phase 1 is complete
- Phase 3 pauses and asks before triggering Phase 1 + 2 for any job