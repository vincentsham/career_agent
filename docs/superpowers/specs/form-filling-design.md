# Phase 2: Form Filling Design

**Date:** 2026-05-14
**Updated:** 2026-05-15
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
  Detect ATS from URL
        ↓
  Known ATS? ──yes──→ Load playbook → Execute known fields (scoped snapshots)
      │                                       ↓
      │                              Delta-scan: find unfilled/unknown fields
      │                                       ↓
      no                             Fill unknowns via standard Plan → Fill
      ↓                                       ↓
  Scan page → Plan field map → Fill page    Next/Continue
       ↑___________re-scan after each page transition___|
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

## ATS Playbook System

### How it works

Before starting any page, Claude checks the tab URL against known ATS patterns. If a match is found, Claude uses a playbook instead of reasoning from scratch:

1. **Execute playbook steps** for all known fields on this page — no inventory scan, no field-by-field reasoning
2. **Delta-scan**: take one scoped snapshot after executing the playbook steps — find any fields that are unfilled or not covered by the playbook
3. **Fill unknowns** using standard Plan → Fill logic for those fields only
4. Click Next/Continue

For unknown ATSes, or any page with no playbook entry, the standard Scan → Plan → Fill loop runs unchanged.

### Why this helps

- Known fields are executed straight through with scoped (not full-page) snapshots — token cost drops significantly
- The delta-scan catches company-specific custom fields that differ between Workday instances
- Playbooks are flexible by design: they cover what's predictably always there, not every possible variation

### Extending playbooks

When a new ATS or a novel page pattern is encountered:
- Fill using the standard Scan → Plan → Fill loop
- After the run completes, add the observed pattern to the appropriate playbook section below
- Keep playbook entries concise — field label, source, widget type, and any non-obvious notes

---

## Stage 1: Reconnaissance (Scan)

For **unknown ATSes only** (no playbook match):

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

## Snapshot Discipline

Snapshots are expensive — Workday accessibility trees are large. Follow this budget strictly:

| When | Action |
|---|---|
| Page start (Scan) | One full snapshot — inventory all fields |
| During fill — plain text, textarea, radio, checkbox | No snapshot — trust the fill, continue |
| During fill — typeahead or file upload | One scoped snapshot after the interaction to confirm selection |
| Before clicking Next/Continue | One snapshot to verify complex fields are set correctly |
| Stale ref recovery | Snapshot the **nearest stable container** (section div), not the full page |

**Never re-snapshot just to find the next field.** Plan the full field order from the initial scan and execute it straight through.

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

## Stage 6: Run Logging

After every run (success or failure), write an execution trace to `logs/form-filling/`.

**File path:** `logs/form-filling/<company>_<role>_<YYYY-MM-DD>.yaml`

**What to log — execution trace only, no field values:**

```yaml
run:
  company: <company>
  role: <role>
  ats: <detected ATS>
  date: <YYYY-MM-DD>

pages:
  - id: <page number>
    title: <page title>
    playbook_matched: true | partial | false
    procedures:
      - field: <field label>
        widget: <widget type>
        commands: [<commands used in order>]
    errors:
      - field: <field label>
        type: <error type>
        recovery: <what fixed it>
    delta_fields:
      - label: <field label>
        type: <field type>
```

Rules:
- Log every run — do not skip failed runs (failures are the most useful for pattern analysis)
- Omit pages where `playbook_matched: true`, `errors` is empty, and `delta_fields` is empty — they add no signal
- Never record values filled into fields — those come from `profile.yaml` and are irrelevant to pattern analysis
- `commands` entries describe the Playwright calls used (e.g. `browser_evaluate(scrollTop=2800)`, `browser_snapshot(scoped)`) — not the data passed to them

**Purpose:** Accumulating logs across multiple runs reveals which widgets appear consistently, which errors recur, and which custom fields different ATS instances add — all actionable improvements to the playbooks below.

---

## Stage 7: Self-Learning

After each fully captured application, Claude reviews what it encountered. If any new ATS, field type, or workaround was needed that is not already in this spec, it appends to the appropriate playbook section or Learned Patterns below.

Rules:
- Write only after `jobs.yaml` is updated (run fully captured)
- If a run fails before capture, do not write any pattern
- One entry per new observation — keep entries concise

---

## Workday Playbook

### Detection

URL matches any of: `*.workday.com`, `*.myworkdayjobs.com`, `wd*.myworkday.com`

### Pre-flight (run once before Page 1)

1. **Declare skip list**: Skills, Certifications, and Languages sections are skipped by default. Do not explore them.
2. **Check `profile.yaml` completeness**: Verify that `diversity.gender`, `diversity.ethnicity`, `diversity.disability_status`, and `how_did_you_hear_about_us` are populated. Flag any gaps to the user before starting — a mid-session pause is more disruptive than an upfront question.
3. **Identify the resume file**: Check `output/` for `resume_<company>_<role>.pdf`. If not found, use `resume/resume.pdf`. Resolve this before Page 2 (My Experience), not during.

### Widget Library

**`custom-listbox`** — Workday province/state and degree dropdowns are custom listboxes, not native `<select>` elements. `browser_select_option` will fail. Procedure:
1. Click the container to open it
2. Take a scoped snapshot of the expanded listbox to get option refs
3. Click the matching option ref directly with `browser_click`

**`virtual-list`** — Workday's Field of Study uses a virtualised list. Typing does NOT filter it. Procedure:
1. Click the typeahead container to open it (state shows `Expanded`)
2. Use `browser_evaluate` to scroll the list's scrollable parent:
   ```js
   () => {
     const listbox = document.querySelector('[role="listbox"][aria-label="Options Expanded"]');
     const scrollable = listbox.closest('[style*="overflow"]') || listbox.parentElement;
     scrollable.scrollTop = <N>;
     return `scrollTop: ${scrollable.scrollTop}`;
   }
   ```
3. Take a scoped snapshot of `role=listbox[name="Options Expanded"]` to see visible options
4. If target not visible, adjust `scrollTop` and re-snapshot
5. Click the option ref with `browser_click` — JS `.click()` does **not** register in Workday
6. Confirm: container should show `1 item selected, <value>`

Approximate scrollTop values (Manulife Workday): A=0, B=1500–2000, C=2800, binary search 0–6000 for other letters.

---

### Page 1 — My Information

Snapshot scope: full page (small)

| Field | Source | Widget |
|---|---|---|
| First Name | `personal.first_name` | text |
| Last Name | `personal.last_name` | text |
| Address Line 1 | `personal.address.street` | text |
| City | `personal.address.city` | text |
| State/Province | `personal.address.state` (e.g. "ON" → "Ontario") | `custom-listbox` |
| Postal Code | `personal.address.zip` | text |
| Country | `personal.address.country` | `custom-listbox` or pre-filled |
| Phone | `personal.phone` | text |
| Email | `personal.email` | text (verify if pre-filled) |
| How did you hear? | `how_did_you_hear_about_us.[source]` | radio or dropdown |
| Previously employed here? | No | radio |

Delta-scan after: catch any company-specific additions (e.g. middle name, preferred name, gender pronoun).

---

### Page 2 — My Experience

Snapshot scope: scoped containers only — never full page.
- Resume section: `role=group[name="Resume"]`
- Work entries: `role=group[name="Work Experience N"]` (where N is the entry number)
- Education entries: `role=group[name="Education N"]`

**Step 1 — Resume upload**
- File path from pre-flight
- Widget: `browser_file_upload`

**Step 2 — LinkedIn URL**
- Source: `personal.linkedin`
- Widget: text

**Step 3 — Work Experience** (repeat for each entry in `work_history`, in order)

| Field | Source | Widget |
|---|---|---|
| Job Title | `title` | text |
| Company | `company` | text |
| Location | `location` | text |
| Start Month | month part of `start_date` | dropdown |
| Start Year | year part of `start_date` | text |
| End Month | month part of `end_date` (or check "I currently work here") | dropdown / checkbox |
| End Year | year part of `end_date` | text |
| Description | `description` | textarea |

After each entry: click "Add Another Work Experience", take scoped snapshot on `role=group[name="Work Experience N"]` for fresh refs before filling the next entry.

**Step 4 — Education** (repeat for each entry in `education`, in order)

| Field | Source | Widget | Notes |
|---|---|---|---|
| School | `school` | text | |
| Degree | `degree` | `custom-listbox` | Map: MSc → "Master of Science", HBSc → "Bachelor of Science" |
| Field of Study | `field` | `virtual-list` | Use JS scroll method |
| Start Year | `start_year` | text | |
| End Year | `end_year` | text | |
| GPA | `gpa` | text | Only if field is shown |

After each entry: click "Add Another Education", take scoped snapshot on `role=group[name="Education N"]` for fresh refs.

**Step 5 — Skip list**
Skills → SKIP. Certifications → SKIP. Languages → SKIP.

Delta-scan after: catch custom fields (e.g. portfolio URL, cover letter upload, open-ended text questions placed on this page).

---

### Page 3 — Application Questions

This page is company-specific. The playbook covers common question patterns only — execute these, then delta-scan for anything else.

| Question pattern | Answer | Source |
|---|---|---|
| Non-compete / restrictive covenant agreement | No | — |
| Legally authorized to work in [country] | Yes | `work_authorization.authorized` |
| Require visa sponsorship | No | `work_authorization.requires_sponsorship` |
| Relatives or connections at the company | No | — |
| Eligible to work in Canada / UK | Yes | — |
| Financial licenses or registrations | No | — |
| Compensation expectation | `"$[salary_min] – $[salary_max] [currency]"` | `compensation.salary_min/max/currency` |
| Prior relationship with / history at [company] | Not Applicable | — |

Delta-scan after: fill any unanswered questions using standard Plan → Fill.

---

### Page 4 — Voluntary Disclosures

Snapshot scope: full page (short)

| Field | Source | Widget |
|---|---|---|
| Gender | `diversity.gender` | radio or dropdown |
| Race / Ethnicity | `diversity.ethnicity` | radio or dropdown |
| Disability | `diversity.disability_status` → "No (Canada)" | typeahead |
| Terms & Conditions | always check | checkbox |

If gender/ethnicity fields are blank in `profile.yaml`, select "I do not wish to disclose" — do not stop and ask unless the field has no opt-out option.

Delta-scan after: catch any additional voluntary fields (veteran status, LGBTQ2+, etc.).

---

### Page 5 — Review

No fields. Verify the form is complete, then notify the user:
> "Form filled for **[Company] — [Role]**. Review it in the browser and type 'submit' to confirm."

---

## Learned Patterns

Use this section for new ATS behaviors and workarounds that don't yet have a full playbook. Once enough patterns accumulate for an ATS, graduate them into a named playbook section.

### "How Did You Hear About Us?" — infer from source

Map the job source to the answer:
- Applied via Indeed → **"Indeed"**
- Applied via LinkedIn → **"LinkedIn"**

Source is stored in `profile.yaml` under `how_did_you_hear_about_us`.

### Workday: Skills typeahead does not list programming languages

Common skills like "Python", "SQL" return "No Items." in Workday's skill catalogue. Skip the Skills section unless the ATS has a known compatible list. Do not waste cycles searching.

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
