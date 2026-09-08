# PLAN Package — Vehicle Registration Form

**Strategist (PLAN) lead** · runId `2026-07-01-vehicle-registration-form`
**Delivery type:** new_single_form · **Data backing:** JSON Schema (no FDM, no workflow)
**Gate result:** PASS · **Gaps:** 0 · **User stories:** 13 (all with ≥1 acceptance criterion)

Authoritative source: reference screenshot `c:\Users\330073\Downloads\Vehicle Reg.jpg` (read; drives fields, labels, order, sections, required markers, layout).

Team deliverables in this folder:
- `PLAN-discover-form-requirements.yaml` — Structured Requirements + full field inventory + 13 user stories with acceptance criteria
- `PLAN-architect-form-solution.yaml` — Solution Architecture + Integration & NFR Strategy + ADLC execution plan

---

## 1. What we are building

A public-facing **Vehicle Registration Form** (title "VEHICLE REGISTRATION FORM", subtitle "Please provide the following details to register your vehicle."). Single page, **3-column responsive grid**, **5 numbered sections**, ~26 fields, red asterisks on required fields, Submit + Reset in the footer. On Submit it produces a **PDF Document of Record** of the submitted data.

### Field inventory (from the screenshot — nothing fabricated)

| # | Section | Field | Type | Required | Notes |
|---|---------|-------|------|----------|-------|
| 1 | Owner Details | Full Name | text | Yes | |
| 2 | | Date of Birth | date | Yes | DD/MM/YYYY, not future |
| 3 | | Gender | radio group | Yes | Male / Female / Other |
| 4 | | Mobile Number | tel | Yes | exactly 10 digits |
| 5 | | Email Address | email | **No** | valid email when provided |
| 6 | | Address | textarea | Yes | multiline |
| 7 | | City | text | Yes | |
| 8 | | State | dropdown | Yes | placeholder "Select" |
| 9 | | PIN Code | number | Yes | exactly 6 digits |
| 10 | Vehicle Details | Vehicle Type | dropdown | Yes | placeholder "Select" |
| 11 | | Manufacturer | text | Yes | |
| 12 | | Model | text | Yes | |
| 13 | | Registration Number | text | Yes | |
| 14 | | Chassis Number | text | Yes | |
| 15 | | Engine Number | text | Yes | |
| 16 | | Fuel Type | dropdown | Yes | placeholder "Select" |
| 17 | | Color | text | Yes | |
| 18 | | Manufacturing Year | dropdown | Yes | placeholder "Select" |
| 19 | Insurance Details | Insurance Company | text | Yes | |
| 20 | | Policy Number | text | Yes | |
| 21 | | Policy Valid Till | date | Yes | DD/MM/YYYY |
| 22 | Document Checklist | Document Checklist | checkbox group | No | multi-select; 6 options |
| 23 | Declaration | Declaration text | display text | — | static statement |
| 24 | | Place | text | Yes | |
| 25 | | Date | date | Yes | DD/MM/YYYY |
| 26 | | Signature / Full Name | text | Yes | |

Document Checklist options: Proof of Identity, Proof of Address, Vehicle Insurance, Pollution Under Control (PUC) Certificate, Road Tax Receipt, Others (if any).
Declaration text: "I hereby declare that the above information is true and correct to the best of my knowledge."

---

## 2. Key architecture decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Data backing** | **JSON Schema** (generate-schema) | Input is a screenshot only — no integration contract, endpoint, or system of record. FDM is enabled project-wide but deliberately not used here. |
| **FDM / data source** | None | No external service in scope. |
| **Submit** | **Shared generic `Custom-Submit-GeneratePDF`** (reused, not per-form) | Satisfies the Document-of-Record NFR (PDF of submitted data). No REST write-back, no email. |
| **Prefill** | None | Anonymous public form, no source of truth. |
| **Workflow** | None | No approval / review / routing / e-signature. |
| **Custom components** | None | All 26 fields + declaration map to Core Components. |
| **Editable template** | Reuse built-in af-page-v2 wiring | Single one-off form; no governed reusable template requested. |
| **Theme** | **Build a dedicated theme** (`{project}-vehicle-registration`) | Navy headings (~#1a2b5e), light-blue background, red asterisks, navy filled Submit, outlined Reset, 3-column grid — distinct from all stock themes. |
| **Clientlib** | `vehicle-registration-form-clientlib` | 5 validation functions (mobile 10-digit, PIN 6-digit, email-when-present, date DD/MM/YYYY, DOB-not-future) + Rule-Editor custom functions (with js.txt manifest). |
| **Rules** | create-form-rules (validate only) | No show/hide/calculate/cascade needed. |
| **Fragments** | None | No shared reuse across forms. |

### JCR paths
- Form page: `/content/forms/af/vehicle-registration-form`
- DAM guide asset: `/content/dam/formsanddocuments/{appFolder}/vehicle-registration-form`
- Conf context: `/conf/{project}`
- Schema: `/content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json`
- Theme (apps): `/apps/fd/af/themes/{project}-vehicle-registration/theme.zip`
- Theme (DAM): `/content/dam/formsanddocuments/{appFolder}/themes/{project}-vehicle-registration`
- Clientlib: `/apps/clientlibs/vehicle-registration-form-clientlib`
- Submit action: `/apps/{project}/fd/af/submitactions/Custom-Submit-GeneratePDF` (shared, reused)

---

## 3. Integration & NFR strategy

- **Accessibility (WCAG 2.1 AA):** aria-label on every field (radio group, all 4 dropdowns, checkbox group, all 3 date fields); keyboard navigable; contrast enforced in the theme and verified in the Phase 13 UI parity check.
- **Localization:** en at launch; i18n-ready (externalized labels + validation messages).
- **Performance:** dispatcher-cache theme + clientlib; keep clientlib lightweight (5 functions); single-page form, no heavy client logic.
- **Security / PII:** owner PII (name, DOB, mobile, email, address) — HTTPS only, no PII in logs, restrict access to submissions + generated PDFs; retention TBD by business.
- **CAPTCHA:** **Recommended** for this public/anonymous form (reCAPTCHA v3 / Turnstile / hCaptcha). Not confirmed by business — added via the Core Components CAPTCHA field during form build if approved (no new phase).
- **Document of Record:** PDF-on-submit via the shared `Custom-Submit-GeneratePDF` action.
- **Environments:** AEMaaCS author + publish + dispatcher; static assets cached; no per-environment secrets.

---

## 4. ADLC Execution Plan

Ordered phases the program agent will execute:

| Phase | Skill | Produces | Covers (user stories) |
|-------|-------|----------|------------------------|
| 0 | ensure-forms-agents-md | config / AGENTS.md | bootstrap — **already done** (verify only) |
| 1 | generate-schema | JSON Schema + fd:formDataRef bindings (~26 props) | data backing for all fields (US-01..US-09) |
| 3 | create-adaptive-form | form page + DAM guide asset + conf context + filter + validation clientlib scaffold; 5 panels, 3-col grid, Submit/Reset, aria-labels, title/subtitle/headings, declaration text; CAPTCHA if approved | US-01..US-09, US-11, US-12, US-13 |
| 4 | create-form-rules | fd:rules validation AST (mobile 10-digit, PIN 6-digit, email-when-present, DOB not-future, date DD/MM/YYYY, required-blocks-submit) | US-01/02/03/07/09/10 |
| 6 | create-submit-action | ensure shared Custom-Submit-GeneratePDF exists + wire form to it | US-10 (DoR PDF) |
| 8 | create-form-theme | dedicated navy/light-blue theme (apps theme.zip + DAM theme-json) | US-12, US-13 (contrast), branding |
| 9 | create-form-clientlib | 5 validation functions + Rule-Editor custom functions (js.txt manifest), wired via clientLibRef | backs Phase 4 validations |
| 10 | create-form-tests | JUnit 5 tests for shared submit action / PDF servlet + Sling Models | US-10, service coverage |
| 13 | test-form-ui | pixel diff + vision report vs Vehicle Reg.jpg | US-12, overall UI parity |

**Parallelizable:** phases 4, 6, 8, 9 after Phase 3; phase 10 after 6; phase 13 after 3+8+9.

### Phases skipped (with reasons)

| Phase | Skill | Reason |
|-------|-------|--------|
| 2 | create-editable-template | Single one-off form — built-in af-page-v2 wiring suffices (optional). |
| 5 | create-form-component | Every field maps to a Core Component. |
| 7 | create-prefill-service | Anonymous public form, nothing to prefill. |
| 11 | migrate-form | New build, not a migration. |
| 12 | create-workflow | No approval/routing/e-signature. |
| — (fragment) | create-AdaptiveFormFragment | No shared section across forms. |
| — (data) | create-fdm | Screenshot-only input, no data source — JSON Schema instead. |

---

## 5. User stories (traceability spine)

13 stories, each with ≥1 acceptance criterion; together they cover every field, every validation rule, the multi-select checklist, the declaration text, Submit(with PDF)/Reset, layout, and accessibility. Full detail in `PLAN-discover-form-requirements.yaml`.

| Story | Coverage |
|-------|----------|
| US-01 | Full Name, Date of Birth, Gender |
| US-02 | Mobile Number (10-digit), Email (optional/valid) |
| US-03 | Address, City, State, PIN Code (6-digit) |
| US-04 | Vehicle Type, Manufacturer, Model |
| US-05 | Registration / Chassis / Engine Number |
| US-06 | Fuel Type, Color, Manufacturing Year |
| US-07 | Insurance Company, Policy Number, Policy Valid Till |
| US-08 | Document Checklist (multi-select, 6 options) |
| US-09 | Declaration text, Place, Date, Signature/Full Name |
| US-10 | Submit blocked until valid → PDF Document of Record |
| US-11 | Reset clears the form |
| US-12 | Title, subtitle, 5 numbered sections, 3-col grid, red asterisks, Submit/Reset |
| US-13 | Accessibility — aria-label on every field, WCAG 2.1 AA |

---

## 6. Risks / open decisions (resolve before/at DESI)

1. **Dropdown enum lists** (State, Vehicle Type, Fuel Type, Manufacturing Year) not enumerated in the screenshot — finalize domain lists at design time; use enum/enumNames (NOT `<items>` child nodes).
2. **CAPTCHA** recommended for a public form but not business-confirmed — resolve before Phase 3 so it can be built in if approved.
3. **PII retention / access-restriction** for submissions + generated PDFs — confirm with business before go-live.
4. **Date format DD/MM/YYYY** must be enforced in both the date-picker display and the validation clientlib — verify parity in Phase 13.

---

## 7. Go-ahead

**User confirmed the plan (2026-07-01).** Handed to **`aem-forms-program-agent`**, which executes phases 1 → 3 → 4 → 6 → 8 → 9 → 10 → 13 against this plan (through the DESI design-forge and IMPL/DEPLOY/TEST leads). Open decisions carried into DESI: CAPTCHA inclusion (recommended) and dropdown enum lists to be finalized at design time.
