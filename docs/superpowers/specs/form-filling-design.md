# Phase 2: Form Filling Design

**Date:** 2026-05-14
**Status:** Active

---

## Overview

Phase 2 fills job application forms using `profile.yaml` as the data source and Playwright MCP for browser control. It uses a hand-over model — the user opens the browser and navigates to application forms manually, then hands off to Claude.

---

## Architecture

```
[User opens headed browser + navigates to application forms in tabs]
        ↓
[Hand-over command: "apply to all tabs" OR "apply to <specific tab>"]
        ↓
[Claude: detect which open tabs are job application forms]
        ↓
[For each target tab:]
  Scan page → Plan field map → Fill page → Next/Continue
       ↑_____________re-scan after each page transition___|
        ↓
  [Submit page reached → notify user in prompt → wait for "submit"]
        ↓
  [User reviews in browser → types "submit" → Claude clicks submit]
        ↓
  [Capture confirmation ID → write to jobs.yaml]
        ↓
[Append any new learned patterns to this file]
```

---

## Browser Setup (one-time)

Initialize the persistent session profile:

```bash
npx playwright codegen --save-storage=auth.json --user-data-dir="./job_search_profile"
```

Use the opened window to log into job boards (LinkedIn, Workday, RBC, etc.) manually. Session cookies are saved to `./job_search_profile` and persist across all future runs. The Playwright MCP server in `.mcp.json` is already configured to use this directory with `--headed`.

---

## Trigger Modes

Claude infers intent from any natural prompt — exact wording is not required:

- **Batch**: any prompt meaning "do all of them" (e.g. "apply to all tabs", "submit everything open", "go through all the forms") — Claude queues every tab it identifies as a job application form
- **Specific**: any prompt naming a company or tab (e.g. "do the RBC one", "apply to Shopify", "fill the form I have open") — Claude focuses on that tab only

---

## Stage 1: Reconnaissance (Scan)

At the start of each page, Claude:

1. Takes a snapshot of the current page state via Playwright
2. Inventories all visible form fields: label text, field type, required status (`*`)
3. Detects page structure: looks for Next / Continue / Submit buttons to determine if more pages follow
4. Flags any cover letter or free-text fields that require generation rather than a direct lookup
5. If CAPTCHA detected: stop, notify user in prompt, wait for confirmation before continuing

After each Next/Continue transition, Claude re-runs this scan on the new page before filling anything.

---

## Stage 2: Field Mapping (Plan)

Claude builds an explicit field map before touching any field:

| Scanned label | Field type | Source | Action |
|---|---|---|---|
| Standard field | any | `profile.yaml` matching key | fill |
| Resume upload | file | `output/resume_<company>_<role>.pdf` → fallback: `resume/resume.pdf` | upload |
| Cover letter | textarea | job description + `profile.yaml` | generate (required only) |
| Required field, no match | any | — | stop, ask user |
| Optional field, no match | any | — | leave blank |

**Dropdown ambiguity**: if the closest option is genuinely unclear, stop and ask before selecting.

---

## Stage 3: Fill Loop

Claude fills each field based on type:

| Type | Behavior |
|---|---|
| Text | Type value directly |
| Textarea | Type value directly |
| Dropdown / select | `browser_select_option` with closest matching option |
| Checkbox | Check if profile value is true/yes; uncheck otherwise |
| Radio | Select option matching profile value |
| Date | Format to match what the field expects (MM/DD/YYYY, YYYY-MM-DD, etc.) |
| File | `browser_upload_file` — use `output/resume_<company>_<role>.pdf`; fall back to `resume/resume.pdf` if not found |
| Cover letter (required) | Generate using job description + `profile.yaml` per cover letter rules in CLAUDE.md |
| Cover letter (optional) | Leave blank |

After filling all fields on the current page, Claude clicks Next/Continue and re-enters Scan → Plan → Fill on the new page. This repeats until the submit button is the only remaining action.

---

## Stage 4: Confirmation & Submission

When the submit button is reached:

1. Claude notifies the user in the prompt:
   > "Form filled for **[Company] — [Role]**. Review it in the browser and type 'submit' to confirm."
2. User reviews the form directly in the headed browser window
3. User types "submit" to confirm
4. Claude clicks the submit button

Claude never clicks submit without receiving explicit "submit" confirmation from the user.

---

## Stage 5: Post-Submission Capture

After submission:

1. Wait for the confirmation/success page to load
2. Extract the confirmation ID (application reference number, if shown)
3. Find the `jobs.yaml` entry matching this company + role. If no entry exists (Phase 2 triggered directly, not via Phase 3), create one. Set:
   - `status`: `applied`
   - `applied_at`: today's date
   - `confirmation_id`: extracted ID (or `null` if not shown)

---

## Stage 6: Self-Learning

After each fully captured application, Claude reviews what it encountered. If any new field type, ATS behavior, or workaround was needed that is not already in this spec, it appends to the Learned Patterns section below.

Rules:
- Write only after `jobs.yaml` is updated (run fully captured)
- If a run fails before capture, do not write any pattern
- One entry per new observation — keep entries concise

---

## Learned Patterns

*(This section grows over time as Phase 2 runs encounter new behaviors.)*

---

## CAPTCHA Handling

If a CAPTCHA is encountered at any point during the run:

1. Stop immediately — do not attempt to fill or advance the form
2. Notify user in prompt: "CAPTCHA detected on [Company] form — please solve it in the browser, then type 'continue'."
3. Wait for user confirmation before resuming

---

## Error Handling

| Situation | Action |
|---|---|
| Required field with no `profile.yaml` match | Stop, name the field, ask user for the value |
| Dropdown with no clear match | Stop, show options, ask user which to pick |
| Page fails to load after Next/Continue | Stop, notify user, wait for instruction |
| Login wall encountered | Stop, notify user to log in manually, wait for "continue" |
| CAPTCHA | See CAPTCHA Handling above |
