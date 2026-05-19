# Phase 1: Resume Tailoring — Design Spec

**Date:** 2026-05-13
**Branch:** phase-1-resume-tailoring
**Status:** Approved, ready for implementation planning

---

## Goal

Given a job posting (URL or pasted text), produce a tailored `.tex` resume and compile it to PDF. The master resume is never modified. Every application gets its own source file and PDF.

---

## Workflow

```
Input: job posting URL or pasted text
  │
  ▼
[Fetch] URL → WebFetch; pasted text → use as-is
  │
  ▼
[Extract] from posting:
  company name, role title, required skills,
  preferred skills, top keywords, role focus
  │
  ▼
[PAUSE 1 — Tailoring Brief]
  Show: company, role, top keywords, role focus,
  planned bullet reorders + phrase changes
  Wait: user says "go", redirects, or aborts
  │ (on "go")
  ▼
[Write] copy resume/resume.tex
     → output/resume_<company>_<role>.tex
  Apply: Skills to Highlight + experience bullets only
  (slight rephrase + reorder, no fabrication)
  │
  ▼
[PAUSE 2 — Diff]
  Show: diff -u resume/resume.tex output/resume_<company>_<role>.tex
  Wait: user chooses compile / edit / discard
  │
  ├─ compile → pdflatex (single pass) → output/resume_<company>_<role>.pdf → git commit
  ├─ edit    → user edits .tex manually → re-show diff (loop back)
  └─ discard → delete output/resume_<company>_<role>.tex → done
```

---

## File Structure

```
resume/
  resume.tex                          ← master, never modified
  twentysecondcv.cls                  ← style, never modified

output/
  resume_<company>_<role>.tex         ← tailored source, one per application
  resume_<company>_<role>.pdf         ← compiled output
```

### Naming convention

Lowercase, spaces → underscores, punctuation stripped.

| Company | Role | Filename prefix |
|---|---|---|
| Google | ML Engineer | `resume_google_ml_engineer` |
| Shopify | Senior Data Scientist | `resume_shopify_senior_data_scientist` |

---

## Compilation

The `.cls` lives in `resume/` but the tailored `.tex` is in `output/`. Use `TEXINPUTS` to bridge them:

```bash
cd output && TEXINPUTS=../resume: pdflatex -interaction=nonstopmode resume_<company>_<role>.tex
rm -f output/resume_<company>_<role>.{aux,log,out,fls,fdb_latexmk,synctex.gz}
```

Use `pdflatex` directly (not `latexmk`) — a resume needs only one pass and `latexmk` will auto-rerun unnecessarily when cross-references change.

If compilation fails, run `grep -A3 "^!" output/resume_<company>_<role>.log` to find the error, fix the `.tex`, and retry. Delete the log after fixing.

After a successful compile, `output/` contains only `.tex` and `.pdf` files.

---

## Tailoring Brief Format (Pause 1)

Shown before any file is written. Should be readable in 30 seconds.

```
Company: Shopify
Role: Senior Data Scientist

Top keywords: Python, causal inference, experimentation, A/B testing, MLOps
Role focus: Experimentation platform and causal ML — not pure engineering

Planned changes:
Skills to Highlight
  • Move "AI & GenAI" bullet to position 1
  • Add "experimentation" to the Data Engineering bullet

Experience
  Vector Institute
    • Lead with the model optimization bullet (most relevant to MLOps framing)
    • Add "A/B testing" to the stakeholder demo bullet
  First North Consulting (Team Lead)
    • Add "experimentation pipeline" framing to the ETL bullet
  Silver Team
    • De-emphasize — move its bullets to last within the role

No changes to: sidebar skills, certifications, education, projects, summary
```

The role focus line is where misinterpretation gets caught before any file is written.

---

## Tailoring Scope

### In scope
- `\section{Skills to Highlight}` bullets: reorder + light rephrase
- `\section{Experience}` bullets: reorder within each role + light rephrase

### Out of scope (never touched)
- `\aboutme{}` — left blank
- `\cvjobtitle{}` — left blank
- Sidebar skills list (`\skills{}`)
- Certifications (`\cert{}`)
- Education section
- Featured Product / Academic Projects sections

### What "light rephrase" means
- Insert a keyword into an existing phrase: `"ETL workflows"` → `"ETL and orchestration workflows"`
- Shift emphasis by reordering bullets within a role
- Use the posting's language where the meaning is identical

### What it does not mean
- Adding technologies not present in the original bullet
- Inventing metrics, outcomes, or achievements
- Changing company names, titles, or dates

---

## Guardrails

| Rule | Detail |
|---|---|
| No fabrication | Never add companies, titles, dates, metrics, or technologies not in the original |
| No new skills | If the posting requires a skill with no evidence in the resume, flag it in the brief — do not add it |
| No `.cls` edits | The style file is never touched |
| Minimal changes | If a bullet already matches the posting, leave it as-is |
| Structure locked | Section names and section count cannot change without explicit user instruction |

---

## Git Commit Format

```
[Company] [Role] - tailored resume
```

Example: `[Shopify] Senior Data Scientist - tailored resume`

Files staged in the commit:

```bash
git add output/resume_<company>_<role>.tex output/resume_<company>_<role>.pdf
git commit -m "[Company] [Role] - tailored resume"
```

---

## User Decision Points

| Pause | What user sees | Options |
|---|---|---|
| Pause 1 — Brief | Keyword extraction + planned changes | go / redirect / abort |
| Pause 2 — Diff | `diff -u` of master vs. tailored .tex | compile / edit / discard |

On **edit**: user modifies the `.tex` manually, then types ready — Claude re-shows the diff and waits again.
On **discard**: tailored `.tex` is deleted, nothing is committed.