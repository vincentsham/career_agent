# Phase 2 v2 — Form Filling Redesign

**Date:** 2026-05-19
**Status:** Design — approved, pending implementation plan
**Supersedes:** structural parts of `form-filling-design.md` (the playbook + logging + self-learning model). Core fill mechanics (DOM-first algorithm, widget quirks, snapshot discipline) carry forward unchanged.

---

## Goal

Improve Phase 2 across three axes — accuracy, speed, token cost — and add a persistent per-ATS knowledge record so a repeat application to the same company skips discovery and replays a known-good path.

The driving observation: today every Phase 2 run reads one ~640-line monolithic spec in full (loading every ATS's playbook regardless of target), and re-derives the same field IDs and dropdown options from the DOM on every single run. Nothing about a previously-seen form is retained. The logging and self-learning sections were specced but never executed once.

---

## 1. Knowledge architecture (token cost)

Replace the single monolithic spec with three load tiers:

| Tier | File | Contents | Loaded |
|---|---|---|---|
| Core | `form-filling-design.md` | Process stages, snapshot discipline, error/CAPTCHA handling, the generic DOM-first algorithm. ATS-agnostic. | Always |
| ATS | `playbooks/<ats>.yaml` | Stable across **all tenants** of that ATS: detection regex, page sequence + names, widget quirks (date-blur, custom-listbox click, skip virtual-list/Skills), discovery query patterns, batch-fill JS templates. | When URL matches that ATS |
| Tenant | `playbooks/<ats>/<tenant-host>.yaml` | Confirmed field IDs per page, confirmed dropdown option-label sets, company-specific custom questions. | Only if a file for that host exists |

CLAUDE.md's Phase 2 trigger changes from "read the whole spec" to:

> read core → detect ATS from tab URL → load `playbooks/<ats>.yaml` → if `playbooks/<ats>/<host>.yaml` exists, load it.

A Workday run no longer pays to load the RBC playbook or any other ATS.

**Tenant-host key:** the URL host (e.g. `priceline.wd1.myworkdayjobs.com`). Different companies on the same ATS get different files; the same company's runs share one file.

---

## 2. Run flow with fast-path (speed)

Per page:

1. **Probe** — one `browser_evaluate`: do all tenant-recorded element IDs for this page still exist in the DOM?
2. **All present → fast replay** — skip discovery and snapshots entirely. Batch-fill text + dates (with blur) + textareas from the recorded IDs in 1–2 evaluate calls. Custom dropdowns: replay the recorded option labels.
3. **Any missing, or no tenant file → discovery path** — run the generic DOM-first 4-phase algorithm (create entries upfront → discover IDs → batch fill → dropdown clicks via `browser_click`), then **rewrite** the tenant record for that page.
4. **Pre-Save validation** — the empty-required-fields scan runs before *every* page advance, every ATS (generalized from the current Workday-only check).

First application to a company = discovery cost (same as today) + a record write. Every subsequent application to that company = no discovery, no option-reading, minimal snapshots. ATS-level quirks are never re-derived — they live in the ATS tier.

---

## 3. Accuracy mechanisms

- **Existence probe before any replay** — a stale or reshuffled form fails the probe and falls back to full discovery instead of silently filling wrong fields.
- **Generalized pre-Save validation** — the required-field scan is mandatory before every page advance, not just Workday Page 2.
- **Post-batch read-back** — each batch-fill `browser_evaluate` returns the resulting `.value` of every field it set; any mismatch triggers a per-field retry. No extra snapshot — the data rides back on the same call.
- **Ambiguous dropdown still stops and asks** (unchanged safety rule) — but once the user confirms a choice, it is recorded to the tenant file so the same company is never ambiguous on that field again.

---

## 4. Logs — kept, with two hard mechanisms

The existing `logs/form-filling/` system stays, narrowed to one job: **the cross-tenant graduation signal**. It died before because Stage 6 ("write a trace") and Stage 7 ("review what you encountered") were soft closing suggestions with no trigger and no consumer — so they were skipped every run. v2 fixes both structurally:

**Mechanism A — mandatory writer.** The log write is a non-optional step in the run loop, gated identically to the tenant-file write (on full capture, see §5). Not a closing suggestion.

**Mechanism B — defined consumer with a trigger.** *Before* writing/updating a tenant file, scan that ATS's accumulated logs. If the same widget-quirk or error signature appears in **≥3 distinct tenants**, promote it from tenant-tier knowledge up into the stable ATS tier (`playbooks/<ats>.yaml`). Concrete rule, runs every capture, observable output.

**Why logs are still justified at all:** tenant files record *successful state only* — a failed or aborted run writes no tenant file. The logs are therefore the *only* place failure-and-recovery patterns are retained, and recurring failures across tenants are exactly the signal worth promoting into the permanent ATS tier.

**Log content unchanged:** execution trace only — widgets seen, errors, recovery, delta fields. No field values, no IDs. One file per run: `logs/form-filling/<company>_<role>_<YYYY-MM-DD>.yaml`.

---

## 5. Write-back trigger

The tenant file **and** the run log are written only after the run is **fully captured** (`jobs.yaml` status set to `applied`). A failed or aborted run records neither — the fast-path is never poisoned with broken state. During the run, per-page records are accumulated in memory and flushed together on capture.

---

## 6. Tenant file schema (sketch — finalized in the implementation plan)

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
        # ... remaining suffixes
    dropdowns:
      degree:
        options: ["M.S.", "B.S."]   # confirmed labels for THIS tenant
    custom_questions: []
```

---

## 7. Scope

- Build the framework generically: tier loader, probe → replay → record loop, generalized pre-Save validation, post-batch read-back, mandatory log writer + graduation consumer.
- Migrate **Workday** fully into `playbooks/workday.yaml` (we have the most data on it).
- **RBC stays as prose in the core spec** for now; it migrates to a tier file the next time it is run. No big-bang migration.

---

## Net effect

| | Today | v2 first run (new company) | v2 repeat run (same company) |
|---|---|---|---|
| Spec load | Full monolith every run | Core + ATS tier | Core + ATS tier + small tenant file |
| Field discovery | Every run | Once, then recorded | Skipped (probe + replay) |
| Dropdown options | Read every run | Read once, recorded | Replayed |
| Snapshots | Many | DOM-first minimum | Near-zero (probe only) |
| Accuracy guard | Workday-only pre-Save | Probe + generalized validation + read-back | Probe + generalized validation + read-back |
| Logs | Specced, never written | Mandatory write + graduation scan | Mandatory write + graduation scan |

---

## Out of scope

- Phase 1 and Phase 3 are untouched.
- No change to the hand-over model (user navigates, Claude fills) or the "never submit without explicit confirmation" rule.
- RBC playbook restructuring is deferred, not part of this work.
