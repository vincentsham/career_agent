---
phase: 01-project-structure
plan: 01
subsystem: infra
tags: [gitignore, git, mcp, permissions, claude-settings]

requires: []
provides:
  - output/ directory tracked via .gitkeep placeholder
  - .gitignore excludes output/, *.pdf, and profile.yaml from version control
  - .claude/settings.json grants unscoped WebFetch permission for job posting URL fetching
affects: [02-resume-source-data, 03-claude-md-playbooks]

tech-stack:
  added: []
  patterns:
    - "Force-add .gitkeep to track an otherwise-ignored directory in git"
    - "Separate shared settings.json from user-local settings.local.json for project-level Claude permissions"

key-files:
  created:
    - output/.gitkeep
    - .claude/settings.json
  modified:
    - .gitignore

key-decisions:
  - "Used unscoped WebFetch (not WebFetch(domain:...)) to satisfy SETUP-03 which requires Claude to fetch any URL"
  - "Appended new .gitignore section rather than rewriting to preserve all 11 existing rules"
  - "settings.local.json left completely untouched — 17 local Chrome/git/gh permissions retained"

patterns-established:
  - "output/ is an ignored directory whose .gitkeep is force-added so latexmk has a stable write target"

requirements-completed: [SETUP-01, SETUP-02, SETUP-03]

duration: 8min
completed: 2026-05-13
---

# Phase 01 Plan 01: Project Structure Foundation Summary

**.gitignore extended with output/ and *.pdf exclusions, tracked output/ placeholder created, and .claude/settings.json wired with unscoped WebFetch permission**

## Performance

- **Duration:** 8 min
- **Started:** 2026-05-13T09:00:00Z
- **Completed:** 2026-05-13T09:08:00Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments

- Added `output/` and `*.pdf` exclusion rules to `.gitignore` (preserving all 11 pre-existing rules including `profile.yaml` and 9 LaTeX artifact patterns)
- Created `output/.gitkeep` as a force-added tracked placeholder so the directory exists in the repo and latexmk has a stable write target in Phase 2
- Created `.claude/settings.json` with `{ "permissions": { "allow": ["WebFetch"] } }` enabling Claude to fetch job posting URLs

## Task Commits

Each task was committed atomically:

1. **Task 1: Add output/ and *.pdf gitignore rules and create tracked output dir** - `7b05a31` (chore)
2. **Task 2: Create .claude/settings.json with WebFetch permission** - `088f5ed` (chore)

**Plan metadata:** (see final metadata commit)

## Files Created/Modified

- `.gitignore` - Added `# Rendered PDF output (Phase 2 writes here)` section with `output/` and `*.pdf` rules appended at end
- `output/.gitkeep` - Empty tracked placeholder; force-added with `git add -f` to bypass gitignore exclusion
- `.claude/settings.json` - New file with minimal WebFetch permission grant; distinct from `.claude/settings.local.json` (untouched)

## Decisions Made

- Used unscoped `"WebFetch"` rather than domain-scoped `"WebFetch(domain:...)"` — SETUP-03 requires Claude to fetch any URL (job postings can be on any domain)
- Appended a clearly labeled section to `.gitignore` rather than rewriting, to minimize diff and preserve all existing exclusion rules
- `.claude/settings.local.json` deliberately left untouched — it holds 17 user-local Chrome DevTools/git/gh permissions that must not be clobbered

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- `output/` directory is tracked and ready for latexmk PDF output in Phase 2
- `profile.yaml` and any `*.pdf` files will remain out of version control
- Claude can fetch job posting URLs via WebFetch when Phase 1 resume tailoring runs
- Phase 2 (resume source + data files) and Phase 3 (CLAUDE.md playbooks) can proceed

---
*Phase: 01-project-structure*
*Completed: 2026-05-13*
