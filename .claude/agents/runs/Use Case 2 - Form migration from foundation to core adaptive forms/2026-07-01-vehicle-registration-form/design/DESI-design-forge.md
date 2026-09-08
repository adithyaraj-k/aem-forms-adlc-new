# DESI Package — Vehicle Registration Form

**Design forge (DESI) lead** · runId `2026-07-01-vehicle-registration-form`
**Upstream:** Strategist PLAN (PASS, 0 gaps, 13 user stories) · **Data backing:** JSON Schema (no FDM, no workflow)
**Gate result:** PASS · **Custom components:** 0 · **Theme:** build · **Test cases:** 45 · **Uncovered US/AC:** 0

Authoritative UX source: reference screenshot `c:\Users\330073\Downloads\Vehicle Reg.jpg` (read; drives labels, order, sections, required markers, layout, palette). All dropdown enums and theme tokens were locked in the DESI brief — no UX/brand input was missing, so no clarification was needed.

Team deliverables in this folder:
- `DESI-design-form-components.yaml` — Component Inventory & Specs + Design Specifications + Authoring Guideline
- `DESI-design-form-tests.yaml` — 45 traceable Test Cases (every US + every AC covered)

---

## 1. What gets built (Component Inventory)

Single-page form, 5 numbered panels, **3-column responsive grid**, **26 logical fields** (25 value-bound + 1 static declaration text), Submit + Reset footer. Every field maps to an AEM Forms **Core Component** — proxied under `{project}/components/adaptiveForm/...` (super-type `core/fd/components/form/...`). Data binds via JSONPath into 5 schema objects: `owner`, `vehicle`, `insurance`, `documents`, `declaration`.

| Panel | Fields | Core Component(s) |
|---|---|---|
| **1. OWNER DETAILS** | Full Name, Date of Birth, Gender, Mobile Number, Email Address, Address, City, State, PIN Code | textinput, datepicker, radiobutton, telephoneinput, emailinput, textinput(multiline), textinput, dropdown, numberinput |
| **2. VEHICLE DETAILS** | Vehicle Type, Manufacturer, Model, Registration Number, Chassis Number, Engine Number, Fuel Type, Color, Manufacturing Year | dropdown, textinput ×5, dropdown, textinput, dropdown |
| **3. INSURANCE DETAILS** | Insurance Company, Policy Number, Policy Valid Till | textinput, textinput, datepicker |
| **4. DOCUMENT CHECKLIST** | Document Checklist (multi-select, 6 options) | checkboxgroup (array-of-enum) |
| **5. DECLARATION** | Declaration text (static), Place, Date, Signature / Full Name | text(plain), textinput, datepicker, textinput |
| **Footer** | Submit, Reset | button (submit), button (reset) |
| **Header** | Title, Subtitle | title, text(plain) |

**Counts:** 25 value-bound fields · 23 required · 2 optional (Email, Document Checklist) · 1 display-only (declaration) · 4 dropdowns · 3 date pickers · 5 panels · 2 buttons · **0 custom components**.

### Dropdown enums (finalized — resolves PLAN risk #1)
- **State** — 28 Indian states + 8 UTs (36 options) + `Select` placeholder, via enum/enumNames.
- **Vehicle Type** — Two Wheeler, Three Wheeler, Four Wheeler / Car, Commercial Vehicle, Heavy Vehicle, Other.
- **Fuel Type** — Petrol, Diesel, CNG, LPG, Electric, Hybrid, Other.
- **Manufacturing Year** — descending 2026 → 1996 (31 entries); regenerate top-of-range at build time so the current year is always first.

All choice lists (incl. Gender radio + Document Checklist) use **enum/enumNames arrays, never `<items>` child nodes** (per project memory — `<items>` render empty in Core Components).

---

## 2. Against which tokens (Design Specifications)

**Theme decision: BUILD** a dedicated theme `{project}-vehicle-registration` (all six stock themes rejected — none match the navy/light-blue palette).

| Token | Value |
|---|---|
| Heading / label navy | `#1a2b5e` |
| Page background | `#e8eef7` (light blue) |
| Field background / border | `#ffffff` / `#c9d3e6` |
| Required asterisk | `#e02020` (red) |
| Submit button | navy `#1a2b5e` filled, white text |
| Reset button | white, navy outline/text |
| Section rule | subtle `#c9d3e6` horizontal rule between panels |
| Grid | 3-column responsive (targets the live `.aem-Grid--12` float grid) |

- **Layout:** single page (not wizard/tabs). Address is a multiline textarea; Document Checklist stacks its 6 checkboxes vertically; declaration text is full-width above a 3-column Place/Date/Signature row; buttons centered in footer.
- **Responsive:** desktop ≥1024px = 3 columns → mobile <768px = single column, full-width fields; date-picker overlay must not be clipped.
- **Accessibility (WCAG 2.1 AA):** aria-label on every field; keyboard-navigable in DOM order; visible focus; required state conveyed by attribute/aria (not asterisk colour alone); contrast ≥4.5:1 (navy-on-light-blue, white-on-navy verified).
- **Icon assets:** **none** — the screenshot is a pure text/field form with no logo/icon/illustration, so nothing is stored as a DAM asset.

---

## 3. Authoring Guideline (summary)

- **Template:** reuse built-in `af-page-v2` wiring (no governed editable template — create-editable-template skipped).
- **Allowed:** textinput, datepicker, radiobutton, telephoneinput, emailinput, numberinput, dropdown, checkboxgroup, text, title, panelcontainer, button.
- **Disallowed:** Foundation guide resource types; CAPTCHA (screenshot parity — see decisions); file/attachment; custom components.
- **Content policies:** lock the 5 panels/headings/order, title/subtitle, and the Submit+Reset footer.
- **Dos/don'ts:** enum/enumNames for all choices; aria-label on every field; validations as fd:rules AST via Rule Editor (never improvised `<validate>` nodes or inline `fd:click='fn()'`); Document Checklist bound to a schema array (not a repeatable panel); use the SHARED Custom-Submit-GeneratePDF action; render the declaration as an AF text(plain) display component (not a checkbox); do NOT make Email or Document Checklist required.

---

## 4. Tested by which cases (Test Design)

**45 test cases**, every one traced to a user story + acceptance criterion, mapped to its executor:

| Executor | Count | What |
|---|---|---|
| **create-form-tests** (JUnit unit) | 5 | clientlib validation fns (mobile-10, PIN-6, email-when-present, date DD/MM/YYYY + DOB-not-future) + shared Custom-Submit-GeneratePDF action/servlet |
| **test-form-ui** (visual parity) | 8 | title/subtitle, 5 section headings, 3-col grid + red asterisks, Submit/Reset styling, declaration text, overall parity gate, responsive 3-col→1-col |
| **sentinel** (e2e / functional / a11y) | 33 (of 45; 32 sentinel-only + shared) | required-blocks-submit per field, 10-digit mobile, 6-digit PIN, email optional-but-valid, dropdown placeholders/enums, checkbox multi-select, dates, Submit gate, PDF DoR delivery, Reset, aria-label sweep, WCAG 2.1 AA keyboard/contrast |

**Categories:** field_validation 27 · ui_parity 8 · accessibility 3 · submit 3 · business_rule 1 (Reset) · responsive 1. No performance case (no SLA/volume in NFRs). Each behaviour covers happy + empty/invalid + error paths where applicable.

---

## 5. Traceability & gate

- **13/13 user stories** have ≥1 test case; **33/33 acceptance criteria** have ≥1 test case. `uncovered_stories` and `uncovered_acceptance_criteria` are **empty**.
- Every component traces to a requirement/AC (see `traceability:` in the components YAML); nothing invented.
- PLAN risks resolved at DESI: **#1 dropdown enums finalized**; **#4 DD/MM/YYYY** enforced in both the date-picker display format and the clientlib validation fns (parity verified by TC-045/TC-042 in Phase 13). **#2 CAPTCHA** — excluded for screenshot parity (locked decision). **#3 PII retention** — business/go-live concern, outside DESI scope; flagged forward.

---

## 6. Hand off to implementation

The DESI package is complete and gated PASS. It hands to `aem-forms-program-agent`, which executes:
- **Build/integration phases** 1 (schema) → 3 (form) → 4 (rules) → 6 (submit wiring) → 8 (theme) → 9 (clientlib) against these **component & design specs**.
- **Test phases** 10 (create-form-tests) and 13 (test-form-ui), plus Sentinel functional, against these **45 test cases**.
