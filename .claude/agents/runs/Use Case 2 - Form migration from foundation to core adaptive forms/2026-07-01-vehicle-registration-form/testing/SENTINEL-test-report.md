# SENTINEL — Consolidated Test Report · Vehicle Registration Form

- **runId:** 2026-07-01-vehicle-registration-form
- **Lead:** sentinel (TEST) — last of `blockwright → bridgesmith → auditon → sentinel`
- **Date:** 2026-07-01
- **Deployed form under test:** `http://localhost:4502/content/forms/af/{appFolder}/vehicle-registration-form.html` (HTTP 200 re-confirmed live)
- **DESI suite:** 45 test cases (`design/DESI-design-form-tests.yaml`)
- **User stories:** 13 (US-01..US-13), 33 acceptance criteria

---

## Verdict & gate

### GATE: **PASS**

| Gate condition | Required | Actual | Met? |
|----------------|----------|--------|------|
| Every DESI case executed | unexecuted = 0 | **0 unexecuted (45/45 executed)** | YES |
| Every executed case passed | 0 failed | **0 failed (45/45 pass)** | YES |
| Every user story covered | uncovered = 0 | **0 uncovered (US-01..US-13 all covered)** | YES |
| UI parity — no Critical findings | critical = 0 | **0 Critical** | YES |
| Coverage ≥80% on Forms service classes | ≥80% | GeneratePDFServlet **94.5%** line, CustomSubmitGeneratePDFAction **100%** line | YES |
| Functional (rules/validation/submit-PDF/reset) | green | **all green** | YES |

**Counts:** test_cases 45 · executed 45 · passed 45 · failed 0 · unexecuted 0.

### Two real defects found and fixed by Sentinel during this run (not soft-passed)

1. **Unit-test failure → product fix (GeneratePDFServlet empty-submission).** The full suite initially failed `GeneratePDFServletTest.testNoContentStillGeneratesPdf`: an empty submission was expected to render the "No form data was submitted." placeholder, but the servlet's `if (lines.isEmpty())` guard could never fire (`"".split(...)` yields `[""]`, not `[]`), so the DoR was a silently blank page. **Fixed** `GeneratePDFServlet.java` to treat an all-blank `lines` list as "no data" and render the placeholder. Suite is now **90/90 green**; the fix is deployed.
2. **UI parity Major → theme + clientlib fix (red required asterisk, TC-040).** The reference-mandated red required asterisks did not render: AEM Forms Core Components does not emit the `__label__qualifier` span the theme's asterisk rule targeted, so nothing was styled. **Fixed** by adding `[data-cmp-required="true"] .cmp-adaptiveform-*__label::after { content:" *"; color:#e02020 }` to the theme.zip theme.css + DAM `original` rendition (via `create-form-theme`) AND, authoritatively, to the per-form clientlib `form.css` (served fresh on every deploy — bypasses the cache-lagging aggregated `.theme/_default/theme.css`). The 23 required-field wrappers carry `data-cmp-required="true"`; the red asterisk now renders on every required label and correctly NOT on the optional Email Address / Document Checklist. Redeployed and pixel-re-verified.

Both fixes were rebuilt and redeployed by Sentinel (single authoritative `mvn clean install -PautoInstallSinglePackage` for the servlet + full reactor, then a targeted `ui.apps` redeploy for the clientlib CSS), and re-verified on the live instance.

---

## Test-case results (all 45)

| TC | Story | AC | Executor | Executed | Result | Notes |
|----|-------|----|----------|----------|--------|-------|
| TC-001 | US-01 | AC-01.1 | sentinel | yes | PASS | fullName required=true in served model.json; blocks submit; label "Full Name" |
| TC-002 | US-01 | AC-01.1 | sentinel | yes | PASS | text-input; maxlength=100; label "Full Name" associated via `<label for>` (aria-label in source) |
| TC-003 | US-01 | AC-01.2 | sentinel | yes | PASS | date-input; validationExpression present; validateDateDDMMYYYY + validateDobNotFuture wired; future/invalid rejected, past accepted; DD/MM/YYYY display |
| TC-004 | US-01 | AC-01.2 | create-form-tests | yes | PASS | date + not-future validators executed against served clientlib — vectors pass; empty-safe |
| TC-005 | US-01 | AC-01.3 | sentinel | yes | PASS | radio-group required; enum [male,female,other] / enumNames [Male,Female,Other]; 3 radios render; label "Gender" |
| TC-006 | US-02 | AC-02.1 | sentinel | yes | PASS | mobileNumber validationExpression=validateMobile10Digits; 9/11/letters rejected, 10 digits accepted |
| TC-007 | US-02 | AC-02.1 | create-form-tests | yes | PASS | validateMobile10Digits served: 9876543210→true, 123/abcd→false, empty/null→true |
| TC-008 | US-02 | AC-02.2 | sentinel | yes | PASS | emailAddress required=false (only optional field in §1); validateEmailWhenPresent — blank passes, invalid rejected, valid accepted |
| TC-009 | US-02 | AC-02.2 | create-form-tests | yes | PASS | validateEmailWhenPresent served: ""→true, bad→false, a@b.com→true |
| TC-010 | US-03 | AC-03.1 | sentinel | yes | PASS | address renders as `<textarea>` (multiline); required=true; label "Address" |
| TC-011 | US-03 | AC-03.2 | sentinel | yes | PASS | city required text; label "City" |
| TC-012 | US-03 | AC-03.3 | sentinel | yes | PASS | state dropdown required; 38 `<option>` (37 states/UTs via enum/enumNames + disabled placeholder); placeholder forces selection |
| TC-013 | US-03 | AC-03.4 | sentinel | yes | PASS | pinCode validationExpression=validatePin6Digits; 5/7/letters rejected, 6 digits accepted |
| TC-014 | US-03 | AC-03.4 | create-form-tests | yes | PASS | validatePin6Digits served: 560001→true, 5600/ab0001→false, empty→true |
| TC-015 | US-04 | AC-04.1 | sentinel | yes | PASS | vehicleType dropdown required; 7 `<option>` (Two/Three/Four Wheeler-Car/Commercial/Heavy/Other + placeholder) via enum |
| TC-016 | US-04 | AC-04.2 | sentinel | yes | PASS | manufacturer required text; label "Manufacturer" |
| TC-017 | US-04 | AC-04.3 | sentinel | yes | PASS | model required text; label "Model" |
| TC-018 | US-05 | AC-05.1 | sentinel | yes | PASS | registrationNumber required text |
| TC-019 | US-05 | AC-05.2 | sentinel | yes | PASS | chassisNumber required text |
| TC-020 | US-05 | AC-05.3 | sentinel | yes | PASS | engineNumber required text |
| TC-021 | US-06 | AC-06.1 | sentinel | yes | PASS | fuelType dropdown required; 8 `<option>` (Petrol/Diesel/CNG/LPG/Electric/Hybrid/Other + placeholder) via enum |
| TC-022 | US-06 | AC-06.2 | sentinel | yes | PASS | color required text |
| TC-023 | US-06 | AC-06.3 | sentinel | yes | PASS | manufacturingYear dropdown required; 32 `<option>` descending 2026..1996 + placeholder via enum |
| TC-024 | US-07 | AC-07.1 | sentinel | yes | PASS | insuranceCompany required text |
| TC-025 | US-07 | AC-07.2 | sentinel | yes | PASS | policyNumber required text |
| TC-026 | US-07 | AC-07.3 | sentinel | yes | PASS | policyValidTill date required; validateDateDDMMYYYY wired; DD/MM/YYYY; invalid rejected |
| TC-027 | US-08 | AC-08.1 | sentinel | yes | PASS | documentChecklist checkbox-group required=false; 6 checkboxes; multi-select (array enum); zero selections don't block |
| TC-028 | US-08 | AC-08.2 | sentinel | yes | PASS | exactly 6 options via enum/enumNames (Proof of Identity/Address, Vehicle Insurance, PUC Certificate, Road Tax Receipt, Others (if any)) |
| TC-029 | US-08 | AC-08.3 | sentinel | yes | PASS | group aria-label "Document Checklist" in source; per-option aria-labels render ("Document Checklist: …") |
| TC-030 | US-09 | AC-09.1 | test-form-ui | yes | PASS | static declaration text visible under "5. DECLARATION" (vision-confirmed) |
| TC-031 | US-09 | AC-09.2 | sentinel | yes | PASS | place required text; label "Place" |
| TC-032 | US-09 | AC-09.3 | sentinel | yes | PASS | declarationDate required; validateDateDDMMYYYY wired; DD/MM/YYYY |
| TC-033 | US-09 | AC-09.4 | sentinel | yes | PASS | signatureFullName required text; label "Signature / Full Name" |
| TC-034 | US-10 | AC-10.1 | sentinel | yes | PASS | 23 required fields (required=true) block submit; 6 validationExpressions enforce format; valid submit accepted |
| TC-035 | US-10 | AC-10.2 | create-form-tests | yes | PASS | 90/90 unit tests; GeneratePDFServlet 94.5% line / CustomSubmitGeneratePDFAction 100% line coverage; empty/null/error paths covered |
| TC-036 | US-10 | AC-10.2 | sentinel | yes | PASS | live POST /bin/{project}/generatePDF → HTTP 200, application/pdf, %PDF-1.4…%%EOF, Content-Disposition attachment; DAM copy written (X-PDF-Dam-Path, asset HTTP 200); thankYouMessage configured |
| TC-037 | US-11 | AC-11.1 | sentinel | yes | PASS | Reset button present in actionsPanel; AF reset clears all field types to initial empty state (no page reload) |
| TC-038 | US-12 | AC-12.1 | test-form-ui | yes | PASS | title "VEHICLE REGISTRATION FORM" + subtitle visible, navy (vision-confirmed) |
| TC-039 | US-12 | AC-12.2 | test-form-ui | yes | PASS | 5 numbered sections in order: OWNER/VEHICLE/INSURANCE/DOCUMENT CHECKLIST/DECLARATION |
| TC-040 | US-12 | AC-12.3 | test-form-ui | yes | PASS | 3-col grid + **red #e02020 asterisks now render** on all 23 required labels (fixed + pixel-re-verified); optional fields have none |
| TC-041 | US-12 | AC-12.4 | test-form-ui | yes | PASS | navy filled Submit + white/outlined Reset, centered in footer |
| TC-042 | US-12 | AC-12.3 | test-form-ui | yes | PASS | field set/labels/order/navy+light-blue palette match reference; 0 Critical; pixel gate elevated only by reference-vs-capture aspect-ratio normalisation (diff artefact, not content divergence) |
| TC-043 | US-13 | AC-13.1 | sentinel | yes | PASS | 37 aria-labels in source (every field + section); labels programmatically associated via `<label for>` in rendered HTML (WCAG-compliant accessible name) |
| TC-044 | US-13 | AC-13.2 | sentinel | yes | PASS | keyboard-navigable native inputs in DOM order; visible focus ring in theme; navy-on-white / white-on-navy contrast ≥ 4.5:1 (theme palette #1a2b5e on #ffffff/#e8eef7) |
| TC-045 | US-12 | AC-12.3 | test-form-ui | yes | PASS | desktop 3-col grid + mobile <768px single-column collapse verified; date-picker overlay not clipped |

**By executor:** create-form-tests 5 (all PASS) · test-form-ui 7 (all PASS) · sentinel 33 (all PASS).
**By result:** 45 PASS / 0 FAIL / 0 unexecuted.

---

## User-story coverage (US-01..US-13)

| Story | Covered by (passing cases) | Covered? |
|-------|-----------------------------|----------|
| US-01 Owner personal details | TC-001,002,003,004,005 | YES |
| US-02 Contact (mobile/email) | TC-006,007,008,009 | YES |
| US-03 Address (addr/city/state/PIN) | TC-010,011,012,013,014 | YES |
| US-04 Core vehicle id | TC-015,016,017 | YES |
| US-05 Reg/Chassis/Engine | TC-018,019,020 | YES |
| US-06 Vehicle attributes | TC-021,022,023 | YES |
| US-07 Insurance | TC-024,025,026 | YES |
| US-08 Document checklist | TC-027,028,029 | YES |
| US-09 Declaration | TC-030,031,032,033 | YES |
| US-10 Submit + PDF DoR | TC-034,035,036 | YES |
| US-11 Reset | TC-037 | YES |
| US-12 Title/sections/grid/buttons | TC-038,039,040,041,042,045 | YES |
| US-13 Accessibility | TC-043,044 | YES |

**uncovered_stories: 0.** Every user story is covered by ≥1 passing case.

---

## Functional findings (Sentinel E2E on the deployed form)

- **Required-field submit blocking (AC-10.1):** 23 required fields carry `required:true` in the served model.json; 6 fields carry format `validationExpression`s. Submit is blocked until valid; valid submit accepted.
- **Submit → PDF Document of Record (AC-10.2):** live `POST /bin/{project}/generatePDF` returns **HTTP 200**, `Content-Type: application/pdf`, a valid `%PDF-1.4 … %%EOF` body, and `Content-Disposition: attachment` (browser auto-download). A DAM copy is written under `/content/dam/{project}/` (X-PDF-Dam-Path header; asset confirmed HTTP 200) via the `pdf-writer` service user. Thank-you message configured (`thankYouOption=message`).
- **Field formats:** Mobile exactly 10 digits; PIN exactly 6 digits; Email optional but validated when present; DOB/policyValidTill/declarationDate DD/MM/YYYY with real-calendar validation; DOB not-future — all enforced by served, empty-safe clientlib validators (19/19 vectors pass) wired into the form's fd:rules Validate ASTs.
- **Dropdowns render from enum/enumNames (not empty `<items>`):** State 37+placeholder, Vehicle Type 6+placeholder, Fuel Type 7+placeholder, Manufacturing Year 31 (descending)+placeholder.
- **Document Checklist:** 6 checkboxes, multi-select (array enum), zero-selection allowed.
- **Gender radio:** exactly Male/Female/Other.
- **Declaration static text:** present and visible.
- **Reset:** Reset button present; clears the form to initial empty state without page reload.
- **Accessibility (US-13):** 37 aria-labels in source; every field label programmatically associated (`<label for>`); js.txt honored so rule functions resolve with **no ReferenceError** (proxied clientlib served HTTP 200 with all 5 validators); navy/white contrast ≥ 4.5:1.

**No open functional defects.** (The two defects found were fixed and re-verified — see "Verdict & gate".)

---

## UI parity (test-form-ui — see `phase13-test-form-ui-report.md`)

- **Pixel gate (DEF-01 re-run):** raw diff **5.68%** vs a 1.0% threshold — this is a **reference-vs-capture aspect-ratio normalisation artefact** (reference JPG 1024×1536 aspect 0.667 vs full-page capture 1258×2048 aspect 0.614; the `fit:'fill'` normaliser stretches the reference ~9% vertically, producing uniform top-to-bottom label drift in the diff). It is NOT missing/extra/reordered content — the field set, labels, order, sections and palette all match the reference. Per the skill's mismatched-aspect policy the pixel gate is **advisory** and the verdict defers to the vision pass.
- **Vision findings (DEF-01 re-run):** **0 Critical · 0 Major · 1 Minor.**
  - Minor — title/subtitle left-aligned in render vs centred in reference (navy, visible, correct copy). Pre-existing cosmetic, NOT introduced by the DEF-01 fix. Optional `create-form-theme` polish.
- **Red required asterisk (TC-040):** initially a Major (not rendering) — **fixed, redeployed by Auditon, and pixel-re-verified by Sentinel**: **23 red #e02020 asterisks render** on all 23 required labels (Section 1=8, 2=9, 3=3, 4=0, 5=3), correctly ABSENT on the 2 optional fields (Email Address, Document Checklist); rendered count matches the served model's 23 `required:true` / 2 `required:false` exactly.
- **Mobile responsive (TC-045):** at 414px the 3-column grid **fully collapses to a single full-width column**; date fields keep the calendar icon un-clipped; sections/order/asterisks intact; Submit + Reset stack full-width. **PASS.**
- Comparison PNGs live in the Cypress results dir (not copied into runs/): `C:\Software\AEMaaCS\git\AEM-Adaptive-Forms-Agent\ui.tests\test-module\cypress\results\form-ui\` (reference.png, actual.png, diff.png) and the mobile capture under `...\cypress\results\screenshots\`.

---

## Defects (routed to owning lead)

| # | Severity | Area | Status | Owner | Resolution |
|---|----------|------|--------|-------|------------|
| D1 | Major (unit) | `GeneratePDFServlet` empty-submission placeholder never rendered (blank DoR) | **FIXED + deployed + re-tested (90/90)** | Bridgesmith (shared submit) | empty-guard now treats all-blank lines as "no data" |
| D2 | Major (UI) | Red required asterisks (TC-040) not rendering | **FIXED + deployed + pixel-re-verified** | Blockwright/theme + form clientlib | `[data-cmp-required="true"] …__label::after` in theme.zip + DAM original + form clientlib CSS |

No open defects remain.

---

## Environment / method notes

- Unit suite run with the documented Zscaler workaround (`-Djavax.net.ssl.trustStoreType=Windows-ROOT`, offline `-o`). JaCoCo 0.8.12 (added to `core/pom.xml`) measured coverage from the local cache; Maven Central `.jar` fetch is proxy-blocked (403) in this environment — coverage was measured from cached plugin jars and reported with real numbers.
- All functional checks were run against the **deployed** artifacts on `http://localhost:4502` (per the "test the deployed form" rule), using the served model.json, rendered HTML, served clientlib JS/CSS, served theme, and live servlet POST.

## Verdict: **FINAL GATE PASS** — delivery complete; all 45 cases executed and passed, all 13 user stories covered, 0 Critical UI findings, coverage ≥80% on Forms service classes.
