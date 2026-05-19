# Phase 2 v2 Form-Filling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure Phase 2 from a single monolithic spec into a 3-tier knowledge model (core spec + per-ATS playbook + per-tenant record) with a probe→replay→record fast-path, so repeat applications to the same company skip discovery.

**Architecture:** This is a documentation/spec restructuring, not an application-code change. Phase 2 "runs" by Claude reading spec/playbook files and driving Playwright MCP. There is no test framework — verification is YAML-parse checks (`python3 -c "import yaml; ..."`) and `grep` assertions that references are consistent and dangling references are gone. Tasks are ordered so the system never breaks mid-transition: the new ATS-tier file is created first (additive), core-spec tier-loading is added before old sections are removed, and CLAUDE.md's entrypoint is switched only after core + ATS tier are both in place.

**Tech Stack:** Markdown specs, YAML playbooks, git. Verification via `python3` (PyYAML) and `grep`. No build step.

---

## File Structure

| Path | Action | Responsibility |
|---|---|---|
| `playbooks/workday.yaml` | Create | Workday ATS tier — stable knowledge true for all Workday tenants: detection, page sequence, widget library, quirks, skip lists. No tenant-specific IDs. |
| `docs/superpowers/specs/form-filling-design.md` | Modify | The lean **core** operational spec: tier loading, generic multi-entry DOM-first algorithm, tenant schema, probe→replay→record loop, generalized pre-Save validation, post-batch read-back, quirk-graduation. RBC prose retained as legacy pending migration. |
| `CLAUDE.md` | Modify | Phase 2 trigger switched from "read one spec" to tiered load (core → ATS tier → tenant tier). |
| `logs/form-filling/.gitkeep` | Delete | Logs subsystem dropped per v2 design (§4). |
| `playbooks/workday/<host>.yaml` | (runtime) | Tenant tier — written by Claude at run capture, not created by this plan. Schema defined in core spec (Task 2). |

Source of truth for migrated Workday content: the current prose at `form-filling-design.md:276-413` (Workday Playbook) and `form-filling-design.md:586-623` (Workday Learned Patterns), as read at plan-writing time and reproduced verbatim in Task 1.

---

### Task 1: Create the Workday ATS-tier playbook

**Files:**
- Create: `playbooks/workday.yaml`

- [ ] **Step 1: Create `playbooks/workday.yaml` with the full content below**

```yaml
# Workday ATS tier — stable across ALL Workday tenants.
# Tenant-specific field IDs and confirmed dropdown labels live in
# playbooks/workday/<host>.yaml (written at run capture, not here).
ats: workday

detection:
  url_matches:
    - "*.workday.com"
    - "*.myworkdayjobs.com"
    - "wd*.myworkday.com"

pages:
  - id: 1
    name: My Information
    fields:
      - { label: First Name, source: personal.first_name, widget: text }
      - { label: Last Name, source: personal.last_name, widget: text }
      - { label: Address Line 1, source: personal.address.street, widget: text }
      - { label: City, source: personal.address.city, widget: text }
      - { label: State/Province, source: personal.address.state, widget: custom-listbox, note: "e.g. ON -> Ontario" }
      - { label: Postal Code, source: personal.address.zip, widget: text }
      - { label: Country, source: personal.address.country, widget: custom-listbox, note: may be pre-filled }
      - { label: Phone, source: personal.phone, widget: text }
      - { label: Email, source: personal.email, widget: text, note: verify if pre-filled }
      - { label: How did you hear?, source: how_did_you_hear_about_us, widget: radio-or-dropdown, note: "options vary by instance — read them, never assume" }
      - { label: Previously employed here?, source: "No", widget: radio }
    delta_scan: catch company-specific additions (middle name, preferred name, pronoun)
  - id: 2
    name: My Experience
    algorithm: core-multi-entry-dom-first
    sections:
      resume_upload: { widget: browser_file_upload, source: pre-flight resume path }
      websites:
        - { source: personal.linkedin }
        - { source: personal.github }
      work_experience:
        repeat_over: work_history
        fields:
          - { label: Job Title, source: title, widget: text }
          - { label: Company, source: company, widget: text }
          - { label: Location, source: location, widget: text, optional: true }
          - { label: Start Month/Year, source: start_date, widget: spinbutton-date }
          - { label: End Month/Year, source: end_date, widget: spinbutton-date, note: or "currently work here" checkbox }
          - { label: Description, source: description, widget: textarea }
      education:
        repeat_over: education
        fields:
          - { label: School, source: school, widget: text }
          - { label: Degree, source: degree, widget: custom-listbox, note: "labels differ by instance — read them, never assume" }
          - { label: Field of Study, widget: virtual-list, skip: true, note: not required in any observed instance }
          - { label: Start Year, source: start_year, widget: text }
          - { label: End Year, source: end_year, widget: text }
          - { label: GPA, source: gpa, widget: text, optional: true }
    delta_scan: catch portfolio URL, cover letter upload, open-ended questions
  - id: 3
    name: Application Questions
    note: company-specific — execute common patterns then delta-scan
    common_questions:
      - { pattern: Non-compete / restrictive covenant, answer: "No" }
      - { pattern: Legally authorized to work in [country], answer: "Yes", source: work_authorization.authorized }
      - { pattern: Require visa sponsorship, answer: "No", source: work_authorization.requires_sponsorship }
      - { pattern: Relatives or connections at company, answer: "No" }
      - { pattern: Eligible to work in Canada / UK, answer: "Yes" }
      - { pattern: Financial licenses or registrations, answer: "No" }
      - { pattern: Compensation expectation, answer: "$[salary_min] - $[salary_max] [currency]", source: compensation }
      - { pattern: Prior relationship with / history at company, answer: "Not Applicable" }
  - id: 4
    name: Voluntary Disclosures
    fields:
      - { label: Gender, source: diversity.gender, widget: radio-or-dropdown }
      - { label: Race / Ethnicity, source: diversity.ethnicity, widget: radio-or-dropdown }
      - { label: Disability, source: diversity.disability_status, widget: typeahead, note: "-> No (Canada)" }
      - { label: Terms & Conditions, widget: checkbox, always: check }
    note: if gender/ethnicity blank in profile, select "I do not wish to disclose" — do not stop unless no opt-out exists
    delta_scan: catch veteran status, LGBTQ2+
  - id: 5
    name: Review
    note: no fields — notify user, wait for explicit "submit"

pre_flight:
  - Declare skip list — Skills, Certifications, Languages are skipped; do not explore them
  - Check profile.yaml has diversity.gender, diversity.ethnicity, diversity.disability_status, how_did_you_hear_about_us — flag gaps before starting
  - Resolve resume file before Page 2

widget_library:
  custom-listbox:
    note: Workday province/state and degree dropdowns are custom listboxes, not native <select>. browser_select_option fails.
    procedure:
      - Click the container to open it
      - Take a scoped snapshot of the expanded listbox to get option refs
      - Click the matching option ref with browser_click (JS .click() does NOT register)
  spinbutton-date:
    note: Date inputs are spinbuttons. Without a blur event the display div stays empty and Save shows "From is required" despite a value being set.
    fill_pattern: |
      const fillDate = (el, value) => {
        const setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
        el.focus();
        setter.call(el, value);
        el.dispatchEvent(new InputEvent('input', { bubbles: true, data: String(value) }));
        el.dispatchEvent(new Event('change', { bubbles: true }));
        el.dispatchEvent(new Event('blur', { bubbles: true }));
      };
  textarea:
    note: HTMLInputElement.prototype.value setter throws "Illegal invocation" on <textarea>. Use direct assignment.
    fill_pattern: |
      el.value = text;
      el.dispatchEvent(new Event('input', { bubbles: true }));
      el.dispatchEvent(new Event('change', { bubbles: true }));
  virtual-list:
    note: Field of Study uses a virtualised list and is not required in any observed instance — SKIP it.

quirks:
  - id: js_click_no_register_on_options
    text: "JS .click() does not register on Workday option elements. Always use browser_click with the accessibility-tree ref from a fresh snapshot."
  - id: skills_typeahead_no_languages
    text: "Skills typeahead returns 'No Items.' for Python/SQL etc. Skip the Skills section; do not search."
  - id: hdyhau_options_vary
    text: "'How Did You Hear About Us?' options are instance-specific. Open the dropdown and read options before selecting. Observed: Manulife=radio buttons; Edelman=Corporate Website/Glassdoor/Handshake/Indeed/LinkedIn/LinkedIn-Alumni/Other."
  - id: date_field_requires_blur
    text: "Date spinbuttons require a blur event to commit (see widget_library.spinbutton-date)."
  - id: textarea_direct_assignment
    text: "Textareas need direct value assignment, not the HTMLInputElement setter (see widget_library.textarea)."
```

- [ ] **Step 2: Verify the YAML parses**

Run: `cd "$(git rev-parse --show-toplevel)" && python3 -c "import yaml,sys; d=yaml.safe_load(open('playbooks/workday.yaml')); print('OK pages:', [p['id'] for p in d['pages']])"`
Expected: `OK pages: [1, 2, 3, 4, 5]`

- [ ] **Step 3: Verify the migrated quirks are all present**

Run: `cd "$(git rev-parse --show-toplevel)" && grep -c -E "js_click_no_register_on_options|skills_typeahead_no_languages|hdyhau_options_vary|date_field_requires_blur|textarea_direct_assignment" playbooks/workday.yaml`
Expected: `5`

- [ ] **Step 4: Commit**

```bash
cd "$(git rev-parse --show-toplevel)"
git add playbooks/workday.yaml
git commit -m "feat(phase-2): add Workday ATS-tier playbook (playbooks/workday.yaml)

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

### Task 2: Add core-spec tier-loading, generic algorithm, tenant schema, and the fast-path loop

This task is purely additive to `form-filling-design.md`. Old sections stay until Task 3 removes them, so the spec is internally consistent at every commit.

**Files:**
- Modify: `docs/superpowers/specs/form-filling-design.md` (insert new sections; no deletions in this task)

- [ ] **Step 1: Update the header line to mark the restructure**

Replace (at `form-filling-design.md:3-5`):

```markdown
**Date:** 2026-05-14
**Updated:** 2026-05-19 (DOM-first Workday Page 2 algorithm + date blur fix + textarea fix + pre-Save validation)
**Status:** Active
```

with:

```markdown
**Date:** 2026-05-14
**Updated:** 2026-05-19 (v2: core spec — 3-tier knowledge, fast-path replay; design rationale in form-filling-design-v2.md)
**Status:** Active — this is the v2 CORE spec (ATS knowledge lives in playbooks/<ats>.yaml)
```

- [ ] **Step 2: Insert the "Knowledge Tiers & Loading" section immediately after the `## Overview` block**

Insert after the Overview section's closing `---` (currently `form-filling-design.md:13`), before `## Architecture`:

```markdown
## Knowledge Tiers & Loading

Phase 2 knowledge is split into three tiers. A run loads only what it needs:

| Tier | File | Loaded |
|---|---|---|
| Core | this file (`form-filling-design.md`) | always |
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
```

- [ ] **Step 3: Insert the generic "Multi-Entry DOM-First Algorithm" section and the fast-path loop after `## Snapshot Discipline`**

Insert immediately after the Snapshot Discipline section's closing `---` (currently `form-filling-design.md:194`), before `## Stage 4: Confirmation & Submission`:

````markdown
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
````

- [ ] **Step 4: Insert the generalized "Pre-Save Validation" and "Quirk Graduation" sections after Stage 5 (Post-Submission Capture)**

Insert immediately after the Stage 5 section's closing `---` (currently `form-filling-design.md:221`), before `## Stage 6: Run Logging`:

```markdown
## Pre-Save Validation (every page, every ATS)

Before clicking Save/Next/Continue on any page, run:

```js
() => Array.from(document.querySelectorAll('p, span, div'))
  .filter(el => el.childElementCount === 0 && el.textContent.includes('required and must'))
  .map(el => el.textContent.trim())
```

Non-empty array → diagnose and fill the missing fields before advancing. Most common cause: a date spinbutton blur was not fired — re-fill those dates with the blur pattern.

## Quirk Graduation (replaces run logging + self-learning)

There is no separate logging subsystem. When the discovery path hits a quirk and recovers from it (e.g. a date field needed a blur, a dropdown label differed), append the quirk's signature to the tenant file's `quirks_encountered` list (written on full capture, Stage 5).

**Graduation rule:** when writing/updating any tenant file, scan the other tenant files for the same ATS. If the same quirk signature appears in **≥3 tenant files**, promote it into that ATS's playbook (`playbooks/<ats>.yaml` under `quirks:`) so future tenants never rediscover it. Runs on every capture; no extra files.

Rationale: tenant files record successful state only. The actionable signal is recurring, recoverable quirks — those occur on captured runs. Patterns from runs that fail before capture are intentionally not retained.
```

- [ ] **Step 5: Verify all new sections exist and the file still parses as one document**

Run: `cd "$(git rev-parse --show-toplevel)" && grep -nE "^## (Knowledge Tiers & Loading|Per-Page Fast-Path|Pre-Save Validation|Quirk Graduation)$" docs/superpowers/specs/form-filling-design.md && grep -nE "^### (Tenant file schema|Multi-Entry DOM-First Algorithm|Post-batch read-back)$" docs/superpowers/specs/form-filling-design.md`
Expected: 7 matching lines (4 `##` headers + 3 `###` headers), in document order.

- [ ] **Step 6: Commit**

```bash
cd "$(git rev-parse --show-toplevel)"
git add docs/superpowers/specs/form-filling-design.md
git commit -m "feat(phase-2): add core-spec tier loading, generic DOM-first algorithm, fast-path

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

### Task 3: Remove superseded sections from the core spec

Now that `playbooks/workday.yaml` exists (Task 1) and core has tier-loading + the generic algorithm (Task 2), strip the migrated/dropped prose.

**Files:**
- Modify: `docs/superpowers/specs/form-filling-design.md` (deletions + one diagram edit)

- [ ] **Step 1: Remove the architecture diagram's stale self-learning line**

In the `## Architecture` ASCII block, replace the line:

```
[Append any new learned patterns to this file]
```

with:

```
[On full capture: write tenant file + run quirk-graduation check]
```

- [ ] **Step 2: Remove the "Extending playbooks" subsection**

Delete the entire `### Extending playbooks` subsection under `## ATS Playbook System` (currently `form-filling-design.md:87-93`, from the `### Extending playbooks` header through the blank line before `---`). It is replaced by Quirk Graduation.

- [ ] **Step 3: Delete Stage 6 (Run Logging) and Stage 7 (Self-Learning) entirely**

Delete from the `## Stage 6: Run Logging` header through the end of the Stage 7 section (currently `form-filling-design.md:223-274`), inclusive of the trailing `---`. The replacements (Pre-Save Validation, Quirk Graduation) were added in Task 2.

- [ ] **Step 4: Delete the embedded Workday Playbook prose block**

Delete from the `## Workday Playbook` header through the end of `### Page 5 — Review` and its trailing `---` (currently `form-filling-design.md:276-475`). This content now lives in `playbooks/workday.yaml`.

- [ ] **Step 5: Delete the Workday-specific Learned Patterns entries**

In `## Learned Patterns`, delete these four subsections (now in `playbooks/workday.yaml`): `### Workday: Skills typeahead does not list programming languages`, `### Workday: "How Did You Hear About Us?" — options vary by instance`, `### Workday: JS .click() does not register on options`, `### Workday: date spinbutton requires blur to commit value`, `### Workday: textarea fill — use direct assignment, not HTMLInputElement setter`. Keep `### "How Did You Hear About Us?" — infer from source` (it is ATS-agnostic). If `## Learned Patterns` now has only that one entry, leave the section; it is still valid for non-Workday ATSes.

- [ ] **Step 6: Mark the RBC Playbook as legacy prose pending migration**

Directly under the `## RBC Playbook` header, insert this line:

```markdown
> **Legacy prose.** RBC is not yet migrated to a tier file. On the next RBC run, after capture, migrate this section to `playbooks/rbc.yaml` (mirror `playbooks/workday.yaml`'s structure) and a `playbooks/rbc/<host>.yaml` tenant file, then delete this section. Until then, this prose is authoritative for RBC.
```

- [ ] **Step 7: Verify superseded content is gone and nothing dangling remains**

Run: `cd "$(git rev-parse --show-toplevel)" && ! grep -nE "^## (Stage 6: Run Logging|Stage 7: Self-Learning|Workday Playbook)$|^### Extending playbooks$" docs/superpowers/specs/form-filling-design.md && ! grep -n "logs/form-filling" docs/superpowers/specs/form-filling-design.md && ! grep -n "Append any new learned patterns to this file" docs/superpowers/specs/form-filling-design.md && echo "CLEAN"`
Expected: `CLEAN`

- [ ] **Step 8: Verify the spec still has its essential surviving sections**

Run: `cd "$(git rev-parse --show-toplevel)" && grep -cE "^## (Overview|Knowledge Tiers & Loading|Architecture|Trigger Modes|Pre-flight|Stage 1: Reconnaissance|Per-Page Fast-Path|Pre-Save Validation|Quirk Graduation|RBC Playbook|CAPTCHA Handling|Error Handling)" docs/superpowers/specs/form-filling-design.md`
Expected: `12`

- [ ] **Step 9: Commit**

```bash
cd "$(git rev-parse --show-toplevel)"
git add docs/superpowers/specs/form-filling-design.md
git commit -m "refactor(phase-2): strip migrated Workday prose + dropped logging from core spec

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

### Task 4: Switch the CLAUDE.md Phase 2 entrypoint to tiered loading

Done after Tasks 1–3 so the entrypoint only points at the new model once it fully exists.

**Files:**
- Modify: `CLAUDE.md` (the `## Phase 2: Form Filling` section)

- [ ] **Step 1: Replace the Phase 2 Instructions block**

Find the `## Phase 2: Form Filling` section and replace its body:

```markdown
## Phase 2: Form Filling

**Trigger:** Any prompt to fill a job application form, or hand-over from Phase 3.

**Instructions:** Read `docs/superpowers/specs/form-filling-design.md` and follow it exactly. Read the file at the start of every Phase 2 run — do not rely on memory of its contents.
```

with:

```markdown
## Phase 2: Form Filling

**Trigger:** Any prompt to fill a job application form, or hand-over from Phase 3.

**Instructions:** Phase 2 uses a 3-tier knowledge model. At the start of every run, do not rely on memory:

1. Read `docs/superpowers/specs/form-filling-design.md` (the core spec) and follow it exactly.
2. Detect the ATS from the tab URL. If `playbooks/<ats>.yaml` exists, read it and follow it.
3. Compute the tenant host (URL host). If `playbooks/<ats>/<host>.yaml` exists, read it and replay it via the core spec's Probe → Replay → Record loop.

Design rationale (not operational): `docs/superpowers/specs/form-filling-design-v2.md`.
```

- [ ] **Step 2: Verify the new instruction is in place and the old single-file wording is gone**

Run: `cd "$(git rev-parse --show-toplevel)" && grep -n "playbooks/<ats>.yaml" CLAUDE.md && grep -n "3-tier knowledge model" CLAUDE.md && ! grep -n "follow it exactly. Read the file at the start of every Phase 2 run" CLAUDE.md && echo "OK"`
Expected: two matching `grep` lines then `OK`.

- [ ] **Step 3: Commit**

```bash
cd "$(git rev-parse --show-toplevel)"
git add CLAUDE.md
git commit -m "feat(phase-2): switch CLAUDE.md Phase 2 entrypoint to 3-tier tiered load

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

### Task 5: Remove the dropped logs subsystem

**Files:**
- Delete: `logs/form-filling/.gitkeep` (and the now-empty `logs/form-filling/` directory)

- [ ] **Step 1: Confirm no operational file still references the logs path**

Run: `cd "$(git rev-parse --show-toplevel)" && grep -rn "logs/form-filling" CLAUDE.md docs/superpowers/specs/form-filling-design.md playbooks/ ; echo "exit:$?"`
Expected: no matches; `exit:1` (grep found nothing). (A reference in `form-filling-design-v2.md` is fine — it is the design rationale explaining the removal.)

- [ ] **Step 2: Remove the logs subsystem from git**

```bash
cd "$(git rev-parse --show-toplevel)"
git rm logs/form-filling/.gitkeep
rmdir logs/form-filling 2>/dev/null; rmdir logs 2>/dev/null; true
```

- [ ] **Step 3: Verify it is gone**

Run: `cd "$(git rev-parse --show-toplevel)" && ! test -e logs/form-filling && git status --porcelain logs/ && echo "REMOVED"`
Expected: ends with `REMOVED` (deletion staged).

- [ ] **Step 4: Commit**

```bash
cd "$(git rev-parse --show-toplevel)"
git commit -m "chore(phase-2): drop unused logs/form-filling subsystem (v2 §4)

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

### Task 6: End-to-end consistency dry-run

No code — a documentation integration check. Walk a hypothetical Workday run against the restructured files and confirm every step has a home and names are consistent.

**Files:**
- Read-only: `CLAUDE.md`, `docs/superpowers/specs/form-filling-design.md`, `playbooks/workday.yaml`, `docs/superpowers/specs/form-filling-design-v2.md`

- [ ] **Step 1: Trace the load path**

Confirm: CLAUDE.md step 1→3 names the same files the core spec's "Knowledge Tiers & Loading" table names, using the same tenant-host key wording (`<url-host>` / `priceline.wd1.myworkdayjobs.com` example). Fix any name drift inline.

- [ ] **Step 2: Trace a first-ever Workday run (no tenant file)**

Confirm a path exists end to end: tiered load → Workday pre_flight (workday.yaml) → Page 2 hits Per-Page Fast-Path → no tenant file → Multi-Entry DOM-First Algorithm → Pre-Save Validation → Stage 4 confirmation → Stage 5 capture → tenant file written per the schema → Quirk Graduation check. Confirm the algorithm's Phase 2 container query (`[id*="workExperience"]`) matches what `playbooks/workday.yaml` page 2 implies. Fix gaps inline.

- [ ] **Step 3: Trace a repeat Workday run (tenant file exists)**

Confirm: probe reads `field_ids` whose key names (`work_experience`, `*_suffix`, `dropdowns.degree.options`) are identical between the core-spec tenant schema and Task 1's runtime example. A mismatch (e.g. `jobTitle_suffix` vs `jobTitleSuffix`) is a bug — fix inline so both files use one spelling.

- [ ] **Step 4: Verify quirk-signature consistency**

Confirm the `quirks_encountered` example signature (`date_field_requires_blur`) exactly equals a `quirks[].id` in `playbooks/workday.yaml`. Graduation matches on this string — any spelling difference breaks it. Fix inline.

- [ ] **Step 5: Spec-coverage check against the v2 design**

Open `form-filling-design-v2.md`. For each of §1–§7, point to where it is implemented (core spec section, workday.yaml, or CLAUDE.md). List any uncovered requirement; if found, add and complete a remediation step here before finishing.

- [ ] **Step 6: Commit any fixes**

```bash
cd "$(git rev-parse --show-toplevel)"
git add -A
git commit -m "fix(phase-2): consistency fixes from v2 dry-run review

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>" || echo "no fixes needed — nothing to commit"
```

---

## Self-Review

**1. Spec coverage (v2 §1–§7):**
- §1 Knowledge architecture → Task 1 (workday.yaml), Task 2 Step 2 (tier table + load procedure), Task 4 (CLAUDE.md).
- §2 Run flow / fast-path → Task 2 Step 3 (Probe→Replay→Record + Multi-Entry algorithm).
- §3 Accuracy → Task 2 Step 3 (post-batch read-back), Step 4 (generalized Pre-Save Validation); probe in Step 3.
- §4 Quirk graduation, no logs → Task 2 Step 4 (Quirk Graduation), Task 5 (remove logs).
- §5 Write-back trigger → Task 2 Step 4 ("written on full capture, Stage 5") + tenant schema note.
- §6 Tenant file schema → Task 2 Step 2 (schema) + Task 1 runtime example.
- §7 Scope → Task 1 (Workday fully migrated), Task 3 Step 6 (RBC legacy marker).
All covered.

**2. Placeholder scan:** No "TBD/TODO/handle appropriately". The runtime tenant file is intentionally not created by the plan (documented in File Structure). The `# one suffix per discovered field` YAML comment is illustrative schema, not a plan placeholder.

**3. Type/name consistency:** Tenant schema key names (`field_ids`, `work_experience`, `jobTitle_suffix`, `company_suffix`, `dropdowns`, `degree`, `options`, `quirks_encountered`) are spelled identically in Task 2 Step 2 and Task 1 Step 1. Quirk id `date_field_requires_blur` matches between Task 1 (`quirks[].id`) and Task 2 (schema example) and Task 6 Step 4 enforces this. ATS detection patterns in workday.yaml match the prose source. Section header strings in verification greps match the headers written in the insert steps.

No issues found.
