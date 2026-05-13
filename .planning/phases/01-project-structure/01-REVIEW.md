---
phase: 01-project-structure
reviewed: 2026-05-13T00:00:00Z
depth: standard
files_reviewed: 3
files_reviewed_list:
  - .gitignore
  - output/.gitkeep
  - .claude/settings.json
findings:
  critical: 1
  warning: 2
  info: 1
  total: 4
status: issues_found
---

# Phase 01: Code Review Report

**Reviewed:** 2026-05-13
**Depth:** standard
**Files Reviewed:** 3
**Status:** issues_found

## Summary

Three project-structure files were reviewed: `.gitignore`, `output/.gitkeep`, and `.claude/settings.json`. The `.gitkeep` file itself is empty and correct. The critical finding is that `jobs.yaml` — the state store for job application history — is not gitignored despite containing potentially sensitive personal data. A significant warning concerns `.claude/settings.json` containing only `WebFetch` permission while all other required agent permissions live in the machine-local (gitignored) `settings.local.json`, leaving the project non-functional on a fresh checkout without undocumented manual setup.

## Critical Issues

### CR-01: `jobs.yaml` not excluded from version control

**File:** `.gitignore`
**Issue:** `jobs.yaml` is the state store tracking seen, matched, and applied jobs (per CLAUDE.md). It will accumulate job application history, company names, URLs, and application statuses over time — the same category of personal data that justified ignoring `profile.yaml`. It is not listed in `.gitignore`, meaning it will be committed to the repository on first write. If this repo is ever made public or shared, that application history is exposed.
**Fix:**
```
# Personal data — do not commit to public repo
profile.yaml
jobs.yaml
```
Add `jobs.yaml` directly below `profile.yaml` on line 2 of `.gitignore`.

## Warnings

### WR-01: `.claude/settings.json` leaves agent non-functional on fresh checkout

**File:** `.claude/settings.json:1-7`
**Issue:** The committed `settings.json` grants only `WebFetch`. All permissions actually required for the agent to operate — `Bash(git *)`, `Bash(gh *)`, `Bash(latexmk *)`, Chrome DevTools MCP tools, filesystem MCP — live exclusively in `.claude/settings.local.json`, which is gitignored (by the global git config). A developer cloning this repository gets a Claude agent that can only fetch URLs. There is no documented setup step telling them to create `settings.local.json`. The agent will silently fail or prompt for every action.
**Fix:** Either document the required `settings.local.json` contents in a `README` or `SETUP.md` (so a fresh clone can bootstrap), or promote the non-sensitive shared permissions (e.g., `Bash(git *)`, `Bash(gh *)`, `Bash(latexmk *)`) into the committed `settings.json`, keeping only truly machine-specific entries in the local file.

### WR-02: `output/` directory ignore pattern prevents future `git add` of tracked files

**File:** `.gitignore:23`
**Issue:** The pattern `output/` ignores the entire directory. `output/.gitkeep` was force-added (`git add -f`) and is currently tracked, which works. However, any future attempt to add a non-PDF file to `output/` (e.g., a manifest, index, or log) will be silently ignored by git. Collaborators running `git status` will see no changes in `output/` regardless of what is written there. This is intentional for PDFs but the blanket directory exclusion creates a silent trap for any other file type.
**Fix:** Replace the blanket `output/` pattern with a targeted pattern that only excludes PDFs inside the directory:
```
# Rendered PDF output (Phase 2 writes here)
output/*.pdf
```
This lets `output/.gitkeep` and any future non-PDF files be tracked normally without force-adding.

## Info

### IN-01: Redundant `resume/*.log` pattern superseded by `*.log`

**File:** `.gitignore:8`
**Issue:** Line 8 (`resume/*.log`) is made entirely redundant by line 19 (`*.log`), which matches all `.log` files everywhere in the tree. The specific pattern adds no value and creates a false impression that only `resume/*.log` files are covered by the LaTeX build artifact section.
**Fix:** Remove line 8 (`resume/*.log`) from the LaTeX build artifacts block. The `*.log` global pattern on line 19 already covers it.

---

_Reviewed: 2026-05-13_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
