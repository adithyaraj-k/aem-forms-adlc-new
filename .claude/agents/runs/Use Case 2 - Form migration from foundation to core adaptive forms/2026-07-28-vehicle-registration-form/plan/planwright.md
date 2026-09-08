# PLAN Phase — Vehicle Registration Form Delivery

**Delivery Date:** 2026-07-28  
**Form Name:** vehicle-registration-form  
**Form Title:** VEHICLE REGISTRATION FORM  
**Project:** aem-adaptive-forms-agents  

---

## Executive Summary

This is a **new single form delivery** to build an AEM Adaptive Form (Core Components, AEM Cloud) that is an **EXACT visual and functional replica** of a provided screenshot (reference design). The form captures vehicle owner details, vehicle information, address, documents/insurance, and a declaration; it validates all input locally and submits via PDF/Document of Record generation + an admin approval workflow.

**Key characteristics:**
- **Input type:** Reference screenshot + complete field inventory + styling specification
- **Data binding:** JSON Schema (not FDM — no external system-of-record for v1)
- **Template:** Reuse existing `blank-af-v2` (no new template)
- **Theme:** Reuse project's default `aem-adaptive-forms-agents-wknd` theme + form-specific token overrides
- **Submit:** PDF/Document of Record generation + shared assign-task-to-admin workflow
- **Prefill:** None (form starts with empty fields + defaults)
- **Workflow:** Shared reusable model (created once, reused across all forms)
- **Custom components:** None (all Core Components cover the field types)
- **Validations:** 20 field-level rules (required, format, pattern, range, date limits)
- **Accessibility:** WCAG 2.1 AA; aria-label on every field

---

## Structured Requirements

### Business Objective

Build a native AEM Adaptive Form that captures complete vehicle registration details from owners. The form must be an EXACT visual and functional replica of the provided screenshot, including all fields, validations, layout, typography, colors, buttons, and styling. The form will accept owner details, vehicle information, address, document/insurance data, and a declaration, then submit via PDF generation (Document of Record) and an approval workflow.

### Delivery Type & Scope

- **Type:** New single form (standalone, not part of a multi-form program)
- **Audience:** Public (embedded on AEM Sites page)
- **Channels:** Web (responsive, desktop + mobile)
- **Layout:** Single-page (no wizard, tabs, or accordion)

### Form Structure — 5 Sections

#### **Section 1: OWNER DETAILS** (3-column layout)
- Owner Full Name * (text, required)
- Date of Birth * (date, DD/MM/YYYY, required, no future dates, optionally 18+)
- Gender * (dropdown: Male, Female, Other; required)
- Contact Number * (tel, 10-digit numeric only; required)
- Email Address * (email, RFC 5322; required)
- Emergency Contact No. * (tel, 10-digit numeric only; required)

#### **Section 2: VEHICLE DETAILS** (3-column layout)
- Vehicle Type * (dropdown, default: "Electric Vehicle"; options: Two Wheeler, Four Wheeler, Commercial Vehicle, Electric Vehicle; required)
- Make & Model * (text; required)
- Registration Year * (text/number, 4-digit year, no future; required)
- Chassis Number * (text; required)
- Fuel Type * (dropdown: Petrol, Diesel, CNG, Electric, Hybrid; required)

#### **Section 3: ADDRESS DETAILS** (full-width address line + 3-column row)
- Address Line * (text, full-width; required)
- City * (text, 3-column; required)
- State * (text, 3-column; required)
- Pincode / ZIP Code * (text/number, 6-digit numeric; required)

#### **Section 4: DOCUMENT & INSURANCE DETAILS** (3-column layout)
- RC Book Number * (text; required)
- Insurance Policy Number * (text; required)
- PUC Certificate Number * (text; required)

#### **Section 5: DECLARATION** (2-column layout + static text)
- Static text: "I confirm that the above information is accurate."
- Submitted By * (text; required)
- Date * (date, DD/MM/YYYY, defaults to today, no future dates; required)

#### **Buttons**
- Submit Form (primary, solid green/teal background, white text)
- Reset (outlined, white background, green/teal border and text)

### Visual Specification (EXACT Replica)

Replicate the provided screenshot in ALL aspects:

**Typography & Layout:**
- Title: "VEHICLE REGISTRATION FORM" (centered, bold, dark navy/charcoal)
- Subtitle: "Please provide the details below to complete your registration." (gray, centered, regular weight)
- Section headers: Numbered (1. 2. 3. etc.), green/teal bold color, ALL CAPS, with thin horizontal dividers below
- Required asterisks: RED color on all required field labels
- Form fields: 1px light gray borders, white/light background, 4px border radius
- Input text: dark color, 14px
- Placeholder text: light gray, 14px

**Layout & Spacing:**
- 3-column field rows for Panels 1, 2, 4
- Full-width address line for Panel 3; then 3-column for city/state/pincode
- 2-column for Submitted By / Date in Panel 5
- 24px padding between sections; 16px vertical gap between field rows; 16px horizontal gap between columns
- 8px gap between labels and inputs

**Colors:**
- Section headers: #228B22 (forest green) or #006B5B (teal)
- Primary button: #228B22 background, white text
- Secondary button: white background, #228B22 border and text
- Required asterisk: #d32f2f (red)
- Field borders: #cccccc (light gray)
- Text: #1a1a1a (dark)
- Placeholder: #999999 (gray)

**Buttons:**
- Submit Form: full-width (or responsive) solid green/teal, white text, 12px/24px padding, 4px radius
- Reset: outlined style, white/transparent background, green/teal border, green/teal text, 4px radius

---

## User Stories (Traceability Spine)

**10 user stories** cover all fields, rules, buttons, and visual/functional requirements. Each has ≥1 acceptance criterion and traces to the form implementation:

| ID | Title | Role | Goal | Coverage |
|----|-------|------|------|----------|
| US-01 | Owner provides identifying details | vehicle owner | Enter full name, DoB, gender | ownerFullName, dateOfBirth, gender |
| US-02 | Owner provides contact information | vehicle owner | Enter contact numbers, email | contactNumber, emailAddress, emergencyContactNo |
| US-03 | Owner specifies vehicle info | vehicle owner | Select vehicle type, enter make/model, year, chassis | vehicleType, makeModel, registrationYear, chassisNumber |
| US-04 | Owner specifies fuel type | vehicle owner | Select fuel type | fuelType |
| US-05 | Owner provides residential address | vehicle owner | Enter address, city, state, pincode | addressLine, city, state, pincode |
| US-06 | Owner provides doc/insurance details | vehicle owner | Enter RC, insurance, PUC numbers | rcBookNumber, insurancePolicyNumber, pucCertificateNumber |
| US-07 | Owner declares accuracy & signs | vehicle owner | Acknowledge accuracy, enter submitted-by name, date | submittedBy, declarationDate |
| US-08 | Owner submits completed form | vehicle owner | Submit + generate PDF record | Submit button, PDF/DoR generation, workflow trigger |
| US-09 | Owner resets form | vehicle owner | Clear all fields and start over | Reset button |
| US-10 | Form displays exact visual replica | user | Form matches reference screenshot in all aspects | Layout, colors, typography, spacing, buttons |

**Acceptance criteria detail:** See `user-stories.yaml` for full AC list (10–15 per story, covering every field validation, button behavior, and visual requirement).

---

## Validation Rules

**20 field-level rules** implement all requirements:

| Rule | Field | Type | Validation | Error Message |
|------|-------|------|-----------|---------------|
| R01 | ownerFullName | required, length | not empty, max 100 chars | Name required, max 100 chars |
| R02 | dateOfBirth | date, range | no future, optionally 18+ | Cannot be future, must be 18+ |
| R03 | gender | required, enum | one of Male/Female/Other | Please select a gender |
| R04 | contactNumber | required, pattern | 10-digit numeric only | Must be 10 digits |
| R05 | emailAddress | required, format | valid RFC 5322 | Invalid email format |
| R06 | emergencyContactNo | required, pattern | 10-digit numeric only | Must be 10 digits |
| R07 | vehicleType | required, enum, default | one of {options}, defaults to Electric Vehicle | Please select vehicle type |
| R08 | makeModel | required, length | not empty, max 100 chars | Required, max 100 chars |
| R09 | registrationYear | required, pattern, range | 4-digit, past year only | Must be valid past year |
| R10 | chassisNumber | required, length | not empty, max 50 chars | Required, max 50 chars |
| R11 | fuelType | required, enum | one of {options} | Please select fuel type |
| R12 | addressLine | required, length | not empty, max 200 chars | Required, max 200 chars |
| R13 | city | required, length | not empty, max 50 chars | Required, max 50 chars |
| R14 | state | required, length | not empty, max 50 chars | Required, max 50 chars |
| R15 | pincode | required, pattern | 6-digit numeric only | Must be 6 digits |
| R16 | rcBookNumber | required, length | not empty, max 50 chars | Required, max 50 chars |
| R17 | insurancePolicyNumber | required, length | not empty, max 50 chars | Required, max 50 chars |
| R18 | pucCertificateNumber | required, length | not empty, max 50 chars | Required, max 50 chars |
| R19 | submittedBy | required, length | not empty, max 100 chars | Required, max 100 chars |
| R20 | declarationDate | required, default, range | defaults to today, no future | Date cannot be in future |

---

## Solution Architecture

### Data Binding Decision

**Schema (JSON Schema), not FDM**

**Reasoning:**
- Input type is reference screenshot + field inventory + styling spec — **NOT** a requirement document with integration contract.
- Per architect rule: screenshot/image input → default to schema, not FDM.
- Form is standalone collect-and-submit with PDF/DoR output; no backend system-of-record needed for v1.
- If future CRM/backend integration required (Phase 2+), FDM can be added via `create-fdm` skill.

**Artifact:** `/content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json`  
**Bindings:** Field name → schema property (1:1 mapping)

### Template Decision

**Reuse existing `blank-af-v2` template (SKIP `create-editable-template`)**

**Reasoning:**
- Existing template `/conf/aem-adaptive-forms-agents/settings/wcm/templates/blank-af-v2` is af-page-v2 Core Components type with generic locked header, unlocked form container, footer.
- Fits form's single-page layout; allows all required Core Components fields.
- Broadly reusable across forms; no distinct structure/policy justifies a new template.
- Per-form styling (green/teal colors) is handled via form-specific theme tokens + clientlib, NOT template.
- **Decision: REUSE — no new template build.**

### Theme Decision

**Reuse project default `aem-adaptive-forms-agents-wknd` theme + form clientlib token override (SKIP `create-form-theme`)**

**Reasoning:**
- Base theme owns standard element styling + `--af-*` token defaults (per mandatory "Theme as token override — Option A" rule).
- Form-specific styling (green/teal #228B22 for buttons/headers) is a THIN `:root` token override, NOT a new full theme.
- Form clientlib (Phase 9) provides: `--af-primary: #228B22` override + grid-width rules for multi-column layout.
- **Decision: REUSE base theme; form clientlib handles brand overrides.**

### Submit Action

**Custom submit action: PDF/Document of Record + Workflow**

**Components:**
1. **OSGi service:** `com.aem.forms.agents.forms.submit.VehicleRegistrationPdfSubmit`  
   - Generates PDF from form data.
   - Triggers "Invoke an AEM Workflow" step to route to assign-task-to-admin workflow.
   - Handles errors (missing fields → block submit).

2. **JCR node:** `/apps/aem-adaptive-forms-agents/fd/af/submitactions/vehicle-registration-pdf-submit`  
   - Wires the service + workflow trigger.

**Phase:** Phase 6 (`create-submit-action`)

### Workflow

**Shared reusable model: `/var/workflow/models/assign-task-to-admin`**

**Characteristics:**
- Created ONCE; reused by all forms in the project.
- Assigns Medium-priority task to admin user on form submission.
- Includes email notification on task assignment.
- Optional Document of Record (PDF) generation on approval.

**Phase:** Phase 12 (`create-workflow`)  
**groundsmith responsibility:** Ensure model exists (create if absent, reuse if present).

### Prefill

**None for v1**

- Form starts with empty fields (except defaults: Vehicle Type = "Electric Vehicle", Declaration Date = today).
- If future prefill from user profile/CRM needed → Phase 7 (`create-prefill-service`) in v2.

### Custom Components

**None needed**

- All field types (text, email, tel, date, dropdown) are covered by Core Components.
- **Phase 5 skipped.**

### Form Validation & Rules

**20 client-side validation rules** (R01–R20) implemented via:
- Form field native validators (required, pattern, min/max).
- `create-form-rules` (Phase 4) for complex cross-field rules (if any).
- Form clientlib custom functions (Phase 9) for shared validation logic.

### Form Clientlib

**Two-clientlib architecture (mandatory rule):**

1. **Shared base clientlib:** `/apps/aem-adaptive-forms-agents/clientlibs/aem-adaptive-forms-agents.forms.base`  
   - Generic scripts/CSS reused by all forms.
   - Standard element styling + `--af-*` token defaults.

2. **Form-specific clientlib:** `/apps/aem-adaptive-forms-agents/clientlibs/vehicle-registration-form-clientlib`  
   - Form-specific validation functions (if any).
   - `--af-primary: #228B22` (green/teal) token override.
   - Grid-width rules (`.af-col-3 { flex-basis: 33% }`, etc.) for multi-column layout on embedded form.
   - Runtime aria-label injection (workaround for Core Components dropping aria-label with visible label).

**guideContainer clientLibRef:** `"aem-adaptive-forms-agents.forms.base,vehicle-registration-form-clientlib"` (comma-separated)

### Accessibility

**WCAG 2.1 AA target**
- Every field has `aria-label` (injected at runtime via clientlib if Core Components drops it).
- Keyboard navigation enabled (AF Core Components default).
- Color contrast: green/teal #228B22 on white ✓; dark text on light backgrounds ✓.
- Required asterisk styling via CSS + label content.

### Tests

1. **Unit + Integration Tests (Phase 10):** `VehicleRegistrationPdfSubmitTest`
   - PDF generation with sample data.
   - Workflow trigger verification.
   - Error handling (required fields block submit).
   - Coverage target: ≥80%.

2. **UI Parity Test (Phase 13):** `test-form-ui`
   - Compare rendered form against reference screenshot.
   - Pixel diff + vision analysis.
   - Verdict: PASS only if layout, spacing, colors, fonts match exactly.

---

## ADLC Execution Plan

Ordered phases respecting dependencies:

| Phase | Skill | Status | Duration (est.) | Parallel |
|-------|-------|--------|-----------------|----------|
| 0 | ensure-forms-agents-md | Verify | <1m | — |
| 1 | generate-schema | Execute | ~15m | — |
| 2 | create-editable-template | **SKIP** (reuse blank-af-v2) | — | — |
| 3 | create-adaptive-form | Execute | ~45m | Phase 8 |
| 4 | create-form-rules | Execute | ~30m | Phase 6, 7 |
| 5 | create-form-component | **SKIP** (no custom components) | — | — |
| 6 | create-submit-action | Execute | ~30m | Phase 7 |
| 7 | create-prefill-service | **SKIP** (no prefill) | — | — |
| 8 | create-form-theme | **SKIP** (reuse wknd + clientlib override) | — | Phase 3 |
| 9 | create-form-clientlib | Execute | ~20m | — |
| 10 | create-form-tests | Execute | ~30m | — |
| 11 | migrate-form | **SKIP** (new form, not migration) | — | — |
| 12 | create-workflow | Execute or Verify | ~20m | — |
| 13 | test-form-ui | Execute | ~30m | (after DEPLOY) |
| 14 | composer | Execute | ~15m | (before DEPLOY) |
| **DEPLOY** | **forgemaster** mvn | Execute | ~10m | (after composer) |
| **TEST** | **sentinel** | Execute | ~30m | (after DEPLOY) |

**Total estimated duration:** ~4–5 hours (sequential, without parallelism optimizations)

**Parallelizable phases:** 3 + 8; 4 + 6 + 7 (after Phase 3)

**Execution order summary:**
```
Phase 0 (config)
  ↓
Phase 1 (schema)
  ↓
Phase 3 (form) + Phase 8 (theme)
  ↓
Phase 4 (rules) + Phase 6 (submit) + Phase 7 (prefill skip)
  ↓
Phase 9 (clientlib)
  ↓
Phase 10 (tests) + Phase 12 (workflow verify)
  ↓
Phase 14 (composer embed)
  ↓
DEPLOY (forgemaster)
  ↓
TEST (sentinel) + Phase 13 (UI test)
```

---

## Key Decisions & Enforcement Rules

### Mandatory Rules (enforced in every phase)

1. ✅ **No Foundation types** — All artifacts use Core Components resource types (fd/af/components → REJECT).
2. ✅ **No hardcoded paths** — All paths derived from `.aem-forms-config.yaml` tokens (`{project}`, `{damContentRoot}`, etc.).
3. ✅ **No hardcoded secrets** — All credentials use `$[secret:keyName]` in `.cfg.json`.
4. ✅ **ResourceResolver try-with-resources** — Only try-with-resources pattern; no finally blocks.
5. ✅ **OSGi configs `.cfg.json` only** — No XML sling:OsgiConfig.
6. ✅ **aria-label on every field** — Enforced by form design + clientlib runtime injection.
7. ✅ **Submit action: OSGi service + JCR node** — Both required (VehicleRegistrationPdfSubmit service + node).
8. ✅ **Shared workflow model (not per-form)** — One `/var/workflow/models/assign-task-to-admin`, reused by all forms.
9. ✅ **Template reuse-first** — Existing `blank-af-v2` reused; no new template.
10. ✅ **Clientlib split (base + form-specific)** — Generic code in shared base; form-specific in per-form.
11. ✅ **Theme as token override (Option A)** — Base theme owns defaults; form clientlib overrides brand tokens only.
12. ✅ **{project} single namespace** — `aem-adaptive-forms-agents` used as folder segment in ALL paths.

### Reuse-First Decisions (recorded)

- **Template:** ✅ Reuse `/conf/aem-adaptive-forms-agents/settings/wcm/templates/blank-af-v2` (skip Phase 2).
- **Theme:** ✅ Reuse `/apps/fd/af/themes/aem-adaptive-forms-agents-wknd` base (skip Phase 8; override via clientlib).
- **Workflow:** ✅ Reuse or create (once) `/var/workflow/models/assign-task-to-admin` (Phase 12).
- **Clientlib (base):** ✅ Reuse `/apps/aem-adaptive-forms-agents/clientlibs/aem-adaptive-forms-agents.forms.base` (existing).

### Input Type & Data Binding Rationale

| Factor | Decision | Rationale |
|--------|----------|-----------|
| Input type | Reference screenshot | Visual + functional replica requirement |
| → Data backing | JSON Schema (not FDM) | Screenshot carries no system-of-record; architect rule: screenshot → schema default |
| → No external API | None for v1 | If CRM/backend integration needed later, add `create-fdm` Phase 2+ |
| → Prefill | None | Form starts empty (except defaults); future prefill via Phase 7 if needed |
| → Template | Reuse blank-af-v2 | Generic, broadly reusable; fits form's single-page layout |
| → Theme | Reuse wknd + clientlib override | Base theme + form-specific brand tokens (green/teal) |

---

## Non-Functional Requirements Strategy

| NFR | Strategy | Owner |
|-----|----------|-------|
| **Accessibility** | WCAG 2.1 AA; aria-label on all fields; keyboard nav; color contrast | formwright (Phase 3) + clientlib (Phase 9) |
| **Localization** | Primary: en; prepare for future FR/DE (translateable static text) | formwright (Phase 3) design |
| **Performance** | Dispatcher cache theme/clientlib (24h TTL); form load <2s on 4G | forgemaster (DEPLOY) + dispatcher config |
| **Security/PII** | HTTPS transport; no PII in logs; secrets via `$[secret:]`; service user via repoinit | groundsmith (Phase 6, 12) + core services |
| **Record Copy** | PDF/DoR generated on submit (Phase 6 custom action) | groundsmith (Phase 6) submit action |
| **Environments** | Author + publish (AEM Cloud); secrets per environment | forgemaster (DEPLOY) |
| **Bot Protection** | Not required (internal/trusted channel); can add CAPTCHA Phase 2+ if public | N/A for v1 |

---

## Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|-----------|
| Multi-column layout collapses to single column on embedded form | High | Phase 9 clientlib includes grid-width rules (`.af-col-3 { flex-basis: 33% }`); sentinel UI test verifies layout parity |
| Core Components drops aria-label when visible label exists | Medium | Phase 9 clientlib runtime JS injects aria-label; sentinel a11y audit confirms |
| Shared workflow model missing when form submits | High | Phase 12 `create-workflow` ensures model present (create if absent, reuse if exists) |
| PDF generation fails on invalid form data | Medium | Phase 6 submit action includes validation gate (blocks submit if required fields empty); Phase 10 tests cover error cases |
| Form visual doesn't match reference screenshot | High | Phase 13 `test-form-ui` pixel diff + vision analysis against reference; gate: PASS only if visual match + no Critical findings |

---

## Acceptance Criteria (Overall Delivery)

The form delivery is DONE when:

- ✅ All 10 user stories (US-01 through US-10) are implemented and verified
- ✅ All 20 validation rules (R01 through R20) working as specified
- ✅ Form visually matches reference screenshot in ALL aspects (layout, colors, typography, spacing, buttons)
- ✅ Form is accessible (WCAG 2.1 AA) with aria-labels, keyboard nav, color contrast
- ✅ Submit generates PDF/Document of Record with all data
- ✅ Submission triggers assign-task-to-admin workflow + creates admin task
- ✅ Reset button clears all fields and restores defaults
- ✅ Form is responsive (mobile/tablet)
- ✅ Required fields marked with red asterisks
- ✅ Form embedded on Test Adaptive Form Sites page via AEM Form Container
- ✅ Build SUCCESS via `mvn clean install -PautoInstallSinglePackage`
- ✅ Deployment confirmed to AEM Cloud
- ✅ Sentinel test suite passes: all cases executed + all user stories covered + UI parity PASS + functional validation PASS

---

## Open Questions & Assumptions

### Assumptions (documented, no blocking)

1. Form is for public/anonymous users (no login required at submission).
2. No prefill data source specified; form starts empty except defaults (Vehicle Type = "Electric Vehicle", Date = today).
3. Data stored as JSON in JCR (AEM Cloud Forms standard).
4. Assign-task-to-admin workflow is a shared, reusable model (not per-form).
5. Email/SMS notifications handled by AEM workflow capabilities.
6. Form does NOT require CAPTCHA (internal/trusted channel).
7. Mobile responsiveness expected (multi-column layouts collapse to single column on small screens).
8. Form embedded on AEM Sites page (Test Adaptive Form) for end-user access.

### Open Questions (for clarification)

1. Should Date of Birth enforce hard 18+ age validation, or is "no future date" sufficient for v1?
2. Are there additional sections/fields planned for Phase 2?
3. Should form data be saveable as draft before final submission?
4. What is the expected post-submission confirmation/response message to the user?
5. Should admins receive real-time email or daily digest of submitted forms?
6. Is prefill from previous submissions or user profile needed for Phase 2?

---

## Handoff to Next Phase (DESI)

This PLAN artifact provides:

1. **Structured Requirements** (`user-stories.yaml`): Complete field inventory, all 10 user stories with acceptance criteria, all 20 validation rules, visual/styling specs, NFRs.
2. **Solution Architecture** (`solution-architecture.yaml`): Data binding (schema), template/theme reuse decisions, custom component assessment, submit/workflow/prefill/clientlib strategy, full ADLC execution plan with skipped phases and parallelization notes.
3. **This summary** (`planwright.md`): Consolidated overview for stakeholder confirmation.

**Next phase:** **DESI (Design)** — `draftsmith` agent runs:
- `design-form-components` — turn the solution architecture + UX/brand into component inventory & specs, design specs, authoring guideline
- `design-form-tests` — turn requirements + design into traceable test cases (every user story + acceptance criterion gets ≥1 test case)

**Dependencies for DESI:**
- Complete PLAN (structured requirements + solution architecture) ✅
- Reference screenshot (visual source of truth) ✅
- Field inventory + validation rules ✅
- Styling specification ✅

---

**Status:** ✅ PLAN PHASE COMPLETE — Ready for stakeholder review and DESI handoff.

**Next Action:** User confirms the plan; proceed to DESI phase.
