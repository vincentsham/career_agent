# Phase 2: Form Filling Design

**Date:** 2026-05-14
**Updated:** 2026-05-19 (v2: core spec — 3-tier knowledge, fast-path replay; design rationale in form-filling-v2-design.md)
**Status:** Active — this is the v2 CORE spec (ATS knowledge lives in playbooks/<ats>.yaml)

---

## Overview

Phase 2 fills job application forms using `profile.yaml` as the data source and Playwright MCP for browser control. It uses a hand-over model — the user opens the browser and navigates to application forms manually, then hands off to Claude.

---

## Knowledge Tiers & Loading

Phase 2 knowledge is split into three tiers. A run loads only what it needs:

| Tier | File | Loaded |
|---|---|---|
| Core | this file (`playbooks/form-filling-core.md`) | always |
| ATS | `playbooks/<ats>.yaml` | when the tab URL matches that ATS's detection patterns |
| Tenant | `playbooks/<ats>/<url-host>.yaml` | only if a file for that host exists |

**Load procedure (start of every run):**

1. Read this core spec.
2. Detect the ATS from the tab URL. If `playbooks/<ats>.yaml` exists, read it.
3. Compute the tenant host = the URL host (e.g. `priceline.wd1.myworkdayjobs.com`). If `playbooks/<ats>/<host>.yaml` exists, read it.

A run never loads an ATS playbook it is not applying to. A Workday application does not load the RBC playbook.

### Tenant file schema

A tenant file records what was *confirmed* on a real run of one company's form so the next run replays it instead of rediscovering. Written only on full capture (see Stage 5). Schema:

```yaml
ats: workday
tenant_host: priceline.wd1.myworkdayjobs.com
last_verified: 2026-05-19
pages:
  - id: 2
    name: My Experience
    field_ids:
      work_experience:
        container: '[id*="workExperience"]'
        jobTitle_suffix: '--jobTitle'
        company_suffix: '--companyName'
        # one suffix per discovered field
    dropdowns:
      degree:
        options: ["M.S.", "B.S."]   # confirmed labels for THIS tenant
    custom_questions: []
quirks_encountered:
  - date_field_requires_blur          # signature compared across tenant files for graduation
```

Tenant files contain no personal data (only element IDs and public option labels) and are committed to git as shared knowledge.

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
[On full capture: write tenant file + run quirk-graduation check]
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

---

## Pre-flight (run once before the first page, any ATS)

1. **Close extra tabs**: Close all browser tabs except the target application tab (and optionally the job listing). Playwright MCP resolves button clicks by role/name globally across all open tabs — extra tabs cause misfired clicks that kill sessions.
2. **Identify the resume file**: Check `output/` for `resume_<company>_<role>.pdf`. If not found, use `resume/resume.pdf`. Resolve before Page 1.
3. **Compute start date**: Convert `availability.start_date` ("2 weeks", "immediately", etc.) to an absolute `YYYY-MM-DD` date based on today.

---

## Stage 1: Reconnaissance (Scan)

For **unknown ATSes only** (no playbook match):

1. Take a `browser_snapshot(depth=3)` to see page structure cheaply — this returns field labels and roles without expanding dropdown option lists
2. If a field needs its options read (ambiguous dropdown, custom widget), take a scoped snapshot on that container only
3. Inventory all visible form fields: label text, field type, required status (`*`)
4. Detect page structure: looks for Next / Continue / Submit buttons to determine if more pages follow
5. Flag any cover letter or free-text fields that require generation rather than a direct lookup
6. If CAPTCHA detected: stop, notify user in prompt, wait for confirmation before continuing

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
| File | `browser_file_upload` — use `output/resume_<company>_<role>.pdf`; fall back to `resume/resume.pdf` if not found |
| Cover letter (required) | Generate using job description + `profile.yaml` per cover letter rules in CLAUDE.md |
| Cover letter (optional) | Leave blank |

### Batch fill via `browser_evaluate`

When a page has **3 or more native `<select>` dropdowns** and their field names/IDs are known from the playbook, fill them all in a single `browser_evaluate` call instead of individual `browser_select_option` calls:

```js
() => {
  const set = (selector, value) => {
    const el = document.querySelector(selector);
    if (!el) return;
    const setter = Object.getOwnPropertyDescriptor(window.HTMLSelectElement.prototype, 'value').set;
    setter.call(el, value);
    el.dispatchEvent(new Event('change', { bubbles: true }));
  };
  set('[aria-label="Field label one"]', 'Yes');
  set('[aria-label="Field label two"]', 'No');
  // ... remaining fields
}
```

Use `browser_select_option` for individual fields or when the selector is uncertain. Use batch fill only when selectors are confirmed from a previous scan or playbook.

After filling all fields on the current page, Claude clicks Next/Continue and re-enters Scan → Plan → Fill on the new page. This repeats until the submit button is the only remaining action.

---

## Snapshot Discipline

Snapshots are expensive — option-heavy dropdowns (country lists, province lists) can add 3,000–5,000 tokens per snapshot. Follow this budget strictly:

| When | Action |
|---|---|
| Page start (Scan) | `browser_snapshot(depth=3)` — structure only, no option lists |
| Need to read dropdown options | Scoped snapshot on that container only |
| Page contains country/province dropdowns | Save to `filename=".playwright-mcp/page-scan.yml"` then `Read` only the relevant section |
| During fill — plain text, textarea, radio, checkbox | No snapshot — trust the fill, continue |
| During fill — typeahead or file upload | One scoped snapshot after the interaction to confirm selection |
| Before clicking Next/Continue | `browser_snapshot(depth=3)` to verify required fields are set |
| Stale ref recovery | Snapshot the **nearest stable container** (section div), not the full page |

**Never re-snapshot just to find the next field.** Plan the full field order from the initial scan and execute it straight through.

**Never use a full snapshot on a page with country/state dropdowns.** Those lists bloat every snapshot by thousands of tokens.

---

## Per-Page Fast-Path: Probe → Replay → Record

For every page, before filling:

1. **Probe** — if a tenant file has recorded element IDs for this page, run one `browser_evaluate` that checks every recorded ID still exists in the DOM. Return the list of missing IDs.
2. **All present → replay** — skip discovery and snapshots. Batch-fill text + dates (blur) + textareas from the recorded IDs in 1–2 evaluate calls. Replay recorded dropdown option labels.
3. **Any missing, or no tenant file → discovery** — run the Multi-Entry DOM-First Algorithm below. After full capture, (re)write the tenant record for this page.
4. **Pre-Save validation** (below) before advancing — every page, every ATS.

### Multi-Entry DOM-First Algorithm

For any page with repeating entry groups (work experience, education, etc.). Uses ~2 evaluate calls + N dropdown clicks total — never a snapshot per entry.

**Phase 1 — Create all entries upfront (1 evaluate).** N = count of entries in the source list. First entry already exists; click Add Another N−1 times:

```js
async () => {
  const delay = ms => new Promise(r => setTimeout(r, ms));
  for (let i = 0; i < N - 1; i++) {
    const btn = Array.from(document.querySelectorAll('button'))
      .find(b => /Add Another/i.test(b.textContent));
    btn?.click();
    await delay(600);
  }
}
```

**Phase 2 — Discover field IDs (1 evaluate).** Query all entry containers; return a map `{ 0: { jobTitle: id, company: id, ... }, 1: {...} }`. Fields absent from the DOM simply don't appear — skip them without error. The container query is ATS-specific (see the ATS playbook; Workday uses `[id*="workExperience"]`).

**Phase 3 — Batch fill (1–2 evaluates).** Fill all discovered text inputs, spinbutton dates (with the blur pattern from the ATS playbook's widget library), and textareas (direct assignment). The same evaluate returns each field's resulting `.value` for post-batch read-back.

**Phase 4 — Custom dropdowns (snapshot + browser_click per option).** For each custom-listbox field: scoped snapshot for option refs, then `browser_click` the match. Never JS `.click()` on options.

### Post-batch read-back

Every batch-fill `browser_evaluate` returns the resulting `.value` of each field it set. Compare against intended values; any mismatch triggers a per-field retry. No extra snapshot — the data rides back on the same call.

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

## Pre-Save Validation (every page, every ATS)

Before clicking Save/Next/Continue on any page, run:

```js
() => Array.from(document.querySelectorAll('p, span, div'))
  .filter(el => el.childElementCount === 0 && el.textContent.includes('required and must'))
  .map(el => el.textContent.trim())
```

Non-empty array → diagnose and fill the missing fields before advancing. Most common cause: a date spinbutton blur was not fired — re-fill those dates with the blur pattern.

---

## Quirk Graduation (replaces run logging + self-learning)

There is no separate logging subsystem. When the discovery path hits a quirk and recovers from it (e.g. a date field needed a blur, a dropdown label differed), append the quirk's signature to the tenant file's `quirks_encountered` list (written on full capture, Stage 5).

**Graduation rule:** when writing/updating any tenant file, scan the other tenant files for the same ATS. If the same quirk signature appears in **≥3 tenant files**, promote it into that ATS's playbook (`playbooks/<ats>.yaml` under `quirks:`) so future tenants never rediscover it. Runs on every capture; no extra files.

Rationale: tenant files record successful state only. The actionable signal is recurring, recoverable quirks — those occur on captured runs. Patterns from runs that fail before capture are intentionally not retained.

---

## RBC Playbook

> **Legacy prose.** RBC is not yet migrated to a tier file. On the next RBC run, after capture, migrate this section to `playbooks/rbc.yaml` (mirror `playbooks/workday.yaml`'s structure) and a `playbooks/rbc/<host>.yaml` tenant file, then delete this section. Until then, this prose is authoritative for RBC.

### Detection

URL matches: `jobs.rbc.com`

### Pre-flight (run once before Page 1)

1. **Close extra tabs** — Playwright MCP can misfire onto open tabs (e.g. Indeed search). Close all tabs except the RBC application.
2. **Identify resume file**: Check `output/` for tailored PDF; fall back to `resume/resume.pdf`.
3. **Upload resume first** — Resume upload is mandatory and must be done before any other field on Page 1. The file chooser is triggered by clicking the "Upload resume" button, then `browser_file_upload`. A success alert and a "Check out these tips" modal appear — dismiss the modal with "No thanks, I'll keep applying".
4. **Expect PDF parser errors on Page 2** — RBC's parser merges work entries, introduces wrong company names (from client mentions), and creates duplicate education entries. Always re-scan Page 2 after upload and fix before advancing.
5. **`isPreppedSubscribed` hidden field** — RBC embeds a Prepped.ai career coaching widget on Page 1 that may fail to load. If the Next button is blocked by a "should be string" validation error on `#isPreppedSubscribed`, run:
   ```js
   () => {
     const el = document.querySelector('#isPreppedSubscribed');
     const setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
     setter.call(el, 'false');
     el.dispatchEvent(new Event('input', { bubbles: true }));
     el.dispatchEvent(new Event('change', { bubbles: true }));
   }
   ```
   Then click Next again.

### Page 1 — My Information

Snapshot: `browser_snapshot(depth=3)` then scoped on `group[name="cntryFields"]` — avoid full snapshot (country dropdown has 200+ options).

| Field | Source | Widget | Notes |
|---|---|---|---|
| Country | `personal.address.country` | native select | Pre-filled as Canada |
| First Name | `personal.first_name` | text | |
| Last Name | `personal.last_name` | text | |
| Address Line 1 | `personal.address.street` | text | |
| City | `personal.address.city` | text | |
| Province or Territory | `personal.address.state` → "Ontario" | native select | |
| Postal Code | `personal.address.zip` | text | |
| Email | `personal.email` | text | |
| Phone Device Type | "Mobile" | native select | |
| Country Phone Code | "Canada (+1)" | native select | |
| Phone number | strip `+1-` from `personal.phone` → digits only | text | Use `browser_click` then `browser_type(slowly=true)` |
| How did you hear about us? | "Job Board" | native select | |
| Source | "Indeed" (not "Indeed Organic") | native select | |
| Have you worked for RBC? | "No" | native select | |

Delta-scan after: check for `isPreppedSubscribed` blocker before clicking Next.

### Page 2 — My Experience (`workAndEducation`)

Snapshot: save to `filename` — page is large due to multiple work/education entries.

1. Resume already uploaded in pre-flight.
2. After upload, re-scan the full page. The PDF parser will have auto-populated fields — verify and fix:
   - Check each work entry: job title, company name, dates, description
   - Check education count: should match `profile.yaml` entries exactly — remove extras using Remove buttons
   - Check degree values for each education entry
3. Language section: add English → Fluent (if shown).

### Page 3 — Application Questions (`jobSpecificQuestions`)

All native `<select>` — use batch fill via `browser_evaluate`:

| Field | Value | Source |
|---|---|---|
| Legally eligible to work in Canada? | Yes | `work_authorization.authorized` |
| Require sponsorship? | No | `work_authorization.requires_sponsorship` |
| Language preference | English | default |
| Currently a student? | No | — |
| Available to start | computed from `availability.start_date` | YYYY-MM-DD format |
| Family members at RBC? | No | — |
| Government official / PEP? | No | — |
| Referred by govt official? | No | — |
| Ever employed by PwC? | No | — |
| Registered with financial regulator? | No | SOA/CFA exams are credentials, not registrations |
| Consent to retain application? | I consent | — |

### Page 4 — Voluntary Disclosures + Terms & Conditions (`applicantAcknowledgment`)

| Field | Source | Widget | Notes |
|---|---|---|---|
| Sex | "Male" | native select | Maps from `diversity.gender = "Man"` |
| Gender Identity | `diversity.gender` → "Man" | native select | |
| LGBTQ+ Community Member | `diversity.lgbtq2plus` → "No" | native select | |
| Person with disability? | "No" | native select | `diversity.disability_status = "No disability"` |
| Race/Ethnicity | see note | native select | Required (*). Profile value "Racialized/ East Asian (Canada)" → ask user for specific option (Chinese, Japanese, Korean, etc.) if not already recorded |
| Veteran/Military Status | "I do not have military experience..." | native select | |
| I accept (Terms & Conditions) | check | checkbox | Required — check before clicking Next |

Race/Ethnicity for Vincent: **Chinese**. Update `profile.yaml` with `diversity.ethnicity_rbc: "Chinese"` to avoid asking again.

### Page 5 — Review (`applicationReview`)

No fields. Notify user:
> "Form filled for **RBC — [Role]**. Review it in the browser and type 'submit' to confirm."

---

## Learned Patterns

Use this section for new ATS behaviors and workarounds that don't yet have a full playbook. Once enough patterns accumulate for an ATS, graduate them into a named playbook section.

### "How Did You Hear About Us?" — infer from source

Map the job source to the answer:
- Applied via Indeed → **"Indeed"**
- Applied via LinkedIn → **"LinkedIn"**

Source is stored in `profile.yaml` under `how_did_you_hear_about_us`.

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
| Next button blocked by hidden field validation (e.g. `isPreppedSubscribed`) | Use `browser_evaluate` with React native setter + `input`/`change` events to set the value. See RBC playbook for exact snippet. |
| Cross-tab click misfire (button click resolves to wrong tab) | Close all non-target tabs before starting. If a misfire already occurred, switch back to the application tab with `browser_tabs(action=select)` and re-snapshot to verify state. |
