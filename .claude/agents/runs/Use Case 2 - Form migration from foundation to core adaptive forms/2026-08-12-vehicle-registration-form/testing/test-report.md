# Sentinel Test Report — Vehicle Registration Form

**Run:** `2026-08-12-vehicle-registration-form` · **Phase:** TEST (final delivery gate) · **Date:** 2026-08-13

---

## 1. Environment under test (mandatory)

| | |
|---|---|
| **Target** | **cloud DEV — AEMaaCS AUTHOR tier** |
| Cloud Manager program | `p185256` |
| Cloud Manager environment | `e1945105` |
| Author URL (primary surface) | `https://author-p185256-e1945105.adobeaemcloud.com` |
| **Primary test surface** | `…/content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html?wcmmode=disabled` — the form **as embedded in the "Test Adaptive Form" Sites page** |
| Standalone form (isolation cross-check only) | `…/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form.html` |
| Form path under test | `/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form` |
| Publish URL (referenced only for pre-existing defects D1/D2) | `https://publish-p185256-e1945105.adobeaemcloud.com` |
| Merged commit | `e570de76f0436cda6e8fa9d1c7f387762c505a87` |
| Merge commit on `origin/main` | `bfbd5ad` — "Merge pull request #5 from arjunvenu-cts/feature/pilotAgentChanges" |
| Pull request | [#5](https://github.com/arjunvenu-cts/AEM-Adaptive-Forms-Agent/pull/5) → `main` |
| CM DEV deploy confirmed | **yes** — all 6 primary URLs return HTTP 200 and serve the **new** form (46 `vehicle-registration-form` references on the page, **0** `event-ticket-booking`) |
| Human go-ahead | received 2026-08-13, with the explicit author URL, the `login-token` session cookie, and instruction to test now |
| Authentication | Adobe IMS **session cookie** (`login-token`) reused from the human's browser. **There is no `admin:admin` on AEMaaCS**; IMS email+password is rejected as Basic auth (401, verified). Token expiry epoch `1786639596`; it did **not** expire during this run. Never written into `runs/`. |

### Entry gate — all three human steps confirmed before testing

1. **PR merged** — `git fetch origin && git log --oneline origin/main` shows `bfbd5ad` (merge of PR #5) containing Pilot's commit `e570de7`. ✅
2. **Cloud Manager DEV pipeline deployed** — the DEV author environment answers 200 on the page, the standalone form, `guideContainer.model.json`, `guideContainer.html`, `theme.css` and `theme.js`, and serves the newly delivered form (not a stale build). ✅
3. **Explicit human prompt to start** — received. Sentinel did not self-start. ✅

### Where each check ran

| Check | Environment |
|---|---|
| UI parity (pixel + vision), functional, E2E, embedded-page, HTTP surface, rule ASTs, contrast | **cloud DEV author** |
| JUnit 5 / AEM Mocks unit suite | **LOCAL** (`mvn test -pl core`) — AEM Mocks needs no instance; this is the one check that legitimately runs locally and is labelled as such throughout |

**No localhost result is presented as a cloud DEV result anywhere in this report.**

---

## 2. Verdict & gate

# ❌ Gate: **FAIL**

**One-line reason:** UI parity passes (93.72% pixel match, 0 Critical) and the embedded page is fully
functional, but **9 of 48 DESI test cases fail** and **7 more cannot be executed at all** because the
DESI test-case artifact describes a different, earlier field inventory than the form that was
delivered — so the traceability loop cannot be closed.

| Gate criterion | Required | Actual | Verdict |
|---|---|---|---|
| Started on an explicit human prompt | yes | yes | ✅ |
| Cloud DEV deploy confirmed | yes | yes | ✅ |
| Embedded page shows exactly the new form | yes | yes — 1 container, inline, no stale/duplicate | ✅ |
| Form functional inside the page | yes | yes — validation + submit both work in-page | ✅ |
| UI parity — pixel match | ≥ 90% | **93.72%** | ✅ |
| UI parity — Critical findings | 0 | **0** | ✅ |
| Unit tests green | yes | 22/22 | ✅ |
| Coverage on Forms services | ≥ 80% | 100% and 96.4% | ✅ |
| **All test cases executed** | **`unexecuted_cases = 0`** | **7 unexecutable** | ❌ |
| **All executed cases pass** | **0 failures** | **9 failed** | ❌ |
| All user stories covered | `uncovered_stories = 0` | 10/10 covered | ✅ |

This is **not** a soft-pass. Per the non-negotiable rules, a failed case or an unexecuted case is a
gate FAIL even when the visual and functional headline results are strong — and they genuinely are
strong here. The distinction that matters for triage: **most of the failures are specification drift,
not broken software.** See §7.

---

## 3. Scores (each reflects a measurement actually taken)

| Gauge | Value | Basis |
|---|---|---|
| Test Cases | **67%** | 32 of 48 DESI cases passed |
| Story Coverage | **100%** | 10 of 10 user stories have ≥ 1 passing case |
| UI Parity | **94%** | measured pixel match 93.72% (6.285% mismatch of 2,111,250 px) |
| Functional | **78%** | 14 of 18 discrete functional checks passed (§6) |
| Unit Tests | **100%** | 22 of 22 green, 0 failures (LOCAL) |

No gauge is inferred or estimated. The pixel figure is the raw measured value from
`pixel-result.json`, not a rounded-up impression.

---

## 4. Embedded-page result

| | |
|---|---|
| Page path | `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form` |
| Page URL | `https://author-p185256-e1945105.adobeaemcloud.com/content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html?wcmmode=disabled` |
| HTTP | **200** (64,343 bytes) |
| Embeds exactly the new form | **yes** |
| Container count | **exactly 1** (`class="cmp-adaptiveform-container cmp-container"`) |
| Container target | `data-cmp-path="/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form/jcr:content/guideContainer"` |
| Rendered inline (not an iframe) | **yes** — zero `<iframe>` elements |
| Stale form present | **no** — 0 references to `event-ticket-booking` (the previously embedded form) |
| Duplicate / stacked form | **no** — 1 container, 1 `__wrapper` |
| Functional inside the page | **yes** — 36 inputs hydrated, all 5 sections render, required validation blocks submit in-page, valid submit reaches the action in-page |
| Page chrome around the form | breadcrumb, search, "Test Adaptive Form" H1, footer — correctly excluded from the parity crop |

**Assembler's embed is correct. No defect routed to Assembler.**

The standalone form URL was cross-checked and behaves identically, confirming the in-page context does
not alter validation or submit behaviour.

---

## 5. UI parity

Full detail: [`test-form-ui-report.md`](./test-form-ui-report.md)

| | |
|---|---|
| Reference | `ui.tests/test-module/cypress/results/form-ui/reference.png` (1258×2066, the Vehicle Registration source design) |
| Capture | cloud DEV author, **cropped to the embedded form region** (`.cmp-adaptiveform-container`), 1250×1689 |
| **Pixel verdict** | **PASS — 6.285% mismatch = 93.72% match** vs the ≤ 10% replica gate (132,693 / 2,111,250 px) |
| Vision findings | **0 Critical** · 3 Major · 3 Minor |
| **UI-parity gate** | **PASS** — both bars met (≥ 90% match AND zero Critical) |

The pixel figure is reliable — same form, same aspect, form-region crop on both sides. No
mismatched-aspect excuse is invoked.

**Field inventory is an exact match to the reference** — 0 missing, 0 extra, correct 3-column row
grouping, correct control types, and Email Address correctly carries **no** asterisk (it is genuinely
optional in the reference). No error state at initial load.

The residual 6.29% is **cumulative vertical spacing drift**: in `diff.png` every label appears twice
(reference position and actual position), the offset growing to ≈80 px by section 5. Nothing is
missing — the form is ~4% vertically compressed relative to the reference.

Major/Minor findings: missing section dividers (D3), missing "Select" placeholder text (D4), three
measured WCAG contrast deficiencies (D11), missing `DD / MM / YYYY` date hints (D5), spacing
compression (D10), radio sizing (D12).

**Responsive:** at 414×896 all sampled fields report `left = 32`, `width = 311` → single column below
768 px. PASS.

---

## 6. Functional findings

Driven through the **embedded page** on cloud DEV author.

### Passed (14)

| # | Check | Evidence |
|---|---|---|
| 1 | Renders + hydrates in-page | 36 inputs, 6 panels, all 5 section headings present |
| 2 | Exactly one form container | `containerCount = 1` |
| 3 | **No error state at initial load** | `errors_at_initial_load = 0` |
| 4 | **Empty submit is BLOCKED** | 23 inline errors; form still rendered; **0** submit/PDF POSTs intercepted; no thank-you |
| 5 | First invalid field focused | focus ring on Full Name in the capture |
| 6 | Authored required messages fire | "Please enter your full name.", "Please select your gender.", "Please select the fuel type." … |
| 7 | Optional fields correctly exempt | Email Address and Document Checklist raise no required error |
| 8 | Format validation blocks submit | invalid mobile + email + PIN all flagged, submit blocked |
| 9 | PIN format message correct | "Enter a valid 6-digit PIN code." |
| 10 | **DOB rejects a future date** | `15/01/2030` → "Enter a valid date (DD/MM/YYYY) that is not in the future."; a valid past date is accepted |
| 11 | `maxLength` enforced | typing 14 digits into Mobile Number yields `1234567890`; `fullName maxlength=100` |
| 12 | **Valid submit succeeds and reaches its action** | POST to `/adobe/forms/af/submit/L2NvbnRlbnQvZm9ybXMvYWYvYWVtLWFkYXB0aXZlLWZvcm0tYWdlbnRzL3ZlaGljbGUtcmVnaXN0cmF0aW9uLWZvcm0=` → "Thank you for submitting the form.", 0 residual errors |
| 13 | Reset clears the form | Full Name and City emptied, 0 errors after reset |
| 14 | Keyboard navigable + accessible names | 39/39 controls focusable; **32/32** form controls have an accessible name (23 via `label[for]`, 9 via `aria-label`) |

### Option sets and defaults (all verified against the reference)

| Field | Options | Default | Matches reference? |
|---|---|---|---|
| State | 38 (blank + 37 states/UTs, full display names) | blank | ✅ reference shows "Select" |
| Vehicle Type | 7 (blank, Two Wheeler, Three Wheeler, Four Wheeler / Car, Commercial Vehicle, Heavy Vehicle, Other) | blank | ✅ reference shows "Select" |
| Fuel Type | 8 (blank, Petrol, Diesel, CNG, LPG, Electric, Hybrid, Other) | blank | ✅ |
| Manufacturing Year | 32 (blank + 2026 → 1996) | blank | ✅ |
| Document Checklist | 6 checkboxes | none checked | ✅ |
| Declaration Date | — | **empty** | ✅ reference shows an empty `DD / MM / YYYY` |

> ⚠️ **The human's brief asked me to verify "Vehicle Type = Electric Vehicle" and "Date = today" as
> defaults. Those defaults do not exist — and the reference screenshot shows they should not.** The
> reference renders "Select" in Vehicle Type and an empty `DD / MM / YYYY` in every date field, and
> "Electric Vehicle" is not even an option in the delivered list (the closest is Fuel Type =
> Electric). I am reporting the contradiction rather than silently picking a side: **the form matches
> the reference**, and the "Electric Vehicle / today" expectation traces to the stale DESI artifact
> (TC-03-001, TC-07-003, TC-09-003). Confirm which is authoritative before any change — see D7.

### Failed (4)

| # | Check | Expected | Actual |
|---|---|---|---|
| 15 | Mobile Number validation **message** | "Enter a valid 10-digit mobile number." | **"Please match the format requested."** (generic) — D6 |
| 16 | Email validation **message** | "Enter a valid email address." | **"Specify the value in allowed format : email."** (generic) — D6 |
| 17 | Declaration Date rejects a future date | error raised | **no error** — `15/01/2030` accepted silently — D8 |
| 18 | Approval workflow fires on submit | `assign-task-to-admin` invoked | **no workflow wired at all** — D9 |

### Notes

- **Validation timing:** the authored expressions fire reliably **on submit**. On blur alone, only the
  PIN expression fired in testing — mobile and email surfaced no message until submit. Not a blocker
  (submit is the gate), but the blur-time experience is inconsistent. Minor, folded into D6.
- **PDF / Document of Record:** `dorType = "none"`, so the PDF comes from the shared
  `Custom-Submit-GeneratePDF` action rather than the DoR mechanism. The action and its servlet are
  covered by 17 green unit tests at 100% / 96.4% coverage, and the live submit reached the AF submit
  endpoint. **The PDF file itself was not captured as a download** in headless Cypress — reported as
  verified at unit + endpoint level, not as an observed file.
- **Thank-you page is unstyled** — browser-default serif on a blank page, no theme or branding. Minor
  (D13).

---

## 7. Root cause of the traceability failure — DESI artifact drift

This is the finding that actually decides the gate, so it is stated plainly.

The DESI artifacts consumed for traceability live in the **earlier** run
`2026-07-28-vehicle-registration-form` (`plan/user-stories.yaml`, `design/test-cases.yaml`). They
describe a **materially different form** from the one delivered and deployed:

| DESI expects | Delivered form (and the reference) |
|---|---|
| `emergencyContactNo` — Emergency Contact No. | **no such field** |
| `registrationYear` — 4-digit past year | Registration Number (alphanumeric) + Manufacturing Year (dropdown) |
| `makeModel` — one combined field | **two** fields: Manufacturer + Model |
| `rcBookNumber` — RC Book Number | **no such field** |
| `pucCertificateNumber` — PUC Certificate Number | PUC exists only as a **checkbox option** |
| `State` as a free-text field (max 50) | **37-option dropdown** |
| `Gender` as a dropdown | **radio group** |
| Vehicle Type default = "Electric Vehicle" | blank ("Select"); "Electric Vehicle" is not an option |
| Declaration Date default = today | empty |
| Declaration text "I confirm that the above information is accurate" | "I hereby declare that the above information is true and correct to the best of my knowledge." |
| "Submitted By" | "Signature / Full Name" |
| Address Line full-width | 1-of-3 columns |
| Section headers **green/teal**; Submit/Reset **green/teal** | **navy** `#1a2b5e` |

Every one of those "deltas" is a case where **the delivered form matches the reference screenshot and
the DESI artifact does not.** The reference — which the human designated authoritative and which the
form reproduces at 93.72% — has 5 numbered sections (Owner, Vehicle, Insurance, Document Checklist,
Declaration) that the DESI test cases never mention.

**Conclusion:** the delivered form is correct against its reference; the DESI test-case and
user-story artifacts are **stale** and were written against an earlier design. This makes 7 cases
physically unexecutable and causes a further ~9 cases to read as "fail" on cosmetic/naming grounds
that the reference actually contradicts.

**This must be resolved by re-baselining the DESI artifacts against the reference (D7), not by
changing the form to match stale test cases.** Doing the latter would regress a 93.72% replica.

---

## 8. Test-case results — all 48 DESI cases

Legend: **pass** = executed, behaviour correct · **fail** = executed, behaviour incorrect ·
**blocked** = cannot execute (the field/behaviour named does not exist in the form *or* the reference).
`*` = executed against the reference-correct equivalent; the case's own wording is stale.

| Case | Story | AC | Executor | Executed | Notes |
|---|---|---|---|---|---|
| TC-01-001 | US-01 | AC-01.1 | test-form-ui | **pass** | `fullName` required, `maxlength=100` measured |
| TC-01-002 | US-01 | AC-01.2 | test-form-ui | **pass** | future `15/01/2030` → authored not-in-future error; 18+ is "optional" in the AC, not implemented |
| TC-01-003 | US-01 | AC-01.3 | test-form-ui | **pass** | `validateDateDDMMYYYY` verified in page scope; valid date accepted |
| TC-01-004 | US-01 | AC-01.4 | test-form-ui | **pass\*** | required + Male/Female/Other confirmed; delivered as **radio group** per reference, not a dropdown |
| TC-02-001 | US-02 | AC-02.1 | test-form-ui | **pass\*** | field is `mobileNumber`; 10-digit pattern, 14 digits truncated to 10 |
| TC-02-002 | US-02 | AC-02.2 | test-form-ui | **fail** | generic "Please match the format requested." — D6 |
| TC-02-003 | US-02 | AC-02.3 | test-form-ui | **pass\*** | invalid email blocked; field is **optional** per reference (AC says required) |
| TC-02-004 | US-02 | AC-02.4 | test-form-ui | **fail** | generic "Specify the value in allowed format : email." — D6 |
| TC-02-005 | US-02 | AC-02.5 | test-form-ui | **blocked** | no Emergency Contact No. field in the form or the reference |
| TC-02-006 | US-02 | AC-02.6 | test-form-ui | **blocked** | same |
| TC-03-001 | US-03 | AC-03.1 | test-form-ui | **fail** | required ✅; default blank and "Electric Vehicle" not an option — matches reference, case stale — D7 |
| TC-03-002 | US-03 | AC-03.2 | test-form-ui | **pass\*** | delivered as `manufacturer` + `model`, both required |
| TC-03-003 | US-03 | AC-03.3 | test-form-ui | **blocked** | no Registration Year field |
| TC-03-004 | US-03 | AC-03.4 | test-form-ui | **blocked** | same |
| TC-03-005 | US-03 | AC-03.5 | test-form-ui | **pass\*** | required ✅; no `maxLength: 50` constraint present |
| TC-04-001 | US-04 | AC-04.1 | test-form-ui | **pass** | all 5 named fuel types present (+LPG, Other per reference) |
| TC-04-002 | US-04 | AC-04.2 | test-form-ui | **pass** | "Please select the fuel type." |
| TC-05-001 | US-05 | AC-05.1 | test-form-ui | **pass\*** | required ✅; 1-of-3 columns, not full-width — matches reference |
| TC-05-002 | US-05 | AC-05.2 | test-form-ui | **pass\*** | required ✅; no `maxLength` constraint |
| TC-05-003 | US-05 | AC-05.3 | test-form-ui | **pass\*** | required ✅; 37-option dropdown per reference, not free text |
| TC-05-004 | US-05 | AC-05.4 | test-form-ui | **pass** | `type=number`; `12` rejected |
| TC-05-005 | US-05 | AC-05.5 | test-form-ui | **pass** | authored "Enter a valid 6-digit PIN code." |
| TC-06-001 | US-06 | AC-06.1 | test-form-ui | **blocked** | no RC Book Number field |
| TC-06-002 | US-06 | AC-06.2 | test-form-ui | **pass\*** | `policyNumber` required ✅; no `maxLength` |
| TC-06-003 | US-06 | AC-06.3 | test-form-ui | **blocked** | no PUC Certificate Number field (PUC is a checkbox option) |
| TC-07-001 | US-07 | AC-07.1 | test-form-ui | **pass\*** | declaration text visible; wording is the reference's, not the AC's |
| TC-07-002 | US-07 | AC-07.2 | test-form-ui | **pass\*** | delivered as "Signature / Full Name", required ✅ |
| TC-07-003 | US-07 | AC-07.3 | test-form-ui | **fail** | format ✅; default empty (matches reference); future date **not** rejected — D8 |
| TC-07-004 | US-07 | AC-07.4 | test-form-ui | **fail** | no error on `15/01/2030` — D8 |
| TC-08-001 | US-08 | AC-08.1 | test-form-ui | **pass\*** | visible, solid, white label — **navy** per reference, not green/teal |
| TC-08-002 | US-08 | AC-08.2 | **sentinel** | **pass** | 23 inline errors, 0 POSTs, no PDF, first field focused |
| TC-08-003 | US-08 | AC-08.3 | create-form-tests | **pass\*** | 17 green tests, 100%/96.4% coverage (**LOCAL**) + live submit reached the action; PDF file not captured headlessly; `dorType=none` so PDF is via the custom action, not DoR |
| TC-08-004 | US-08 | AC-08.4 | create-form-tests | **fail** | no workflow model, no workflow wiring — D9 |
| TC-09-001 | US-09 | AC-09.1 | test-form-ui | **pass\*** | white fill + border + label — **navy** per reference, not green/teal |
| TC-09-002 | US-09 | AC-09.2 | test-form-ui | **pass** | fields cleared, 0 errors after reset |
| TC-09-003 | US-09 | AC-09.3 | test-form-ui | **blocked** | those defaults don't exist in the form or the reference |
| TC-10-001 | US-10 | AC-10.1 | test-form-ui | **pass** | centered, bold, `#1a2b5e` |
| TC-10-002 | US-10 | AC-10.2 | test-form-ui | **pass\*** | regular weight, `#3a4a70`; **left**-aligned in both reference and form |
| TC-10-003 | US-10 | AC-10.3 | test-form-ui | **fail** | bold/CAPS/numbered ✅, navy per reference; **6 divider rules missing** — D3 |
| TC-10-004 | US-10 | AC-10.4 | test-form-ui | **pass** | `#e02020` asterisks pixel-confirmed; none on optional Email |
| TC-10-005 | US-10 | AC-10.5 | test-form-ui | **pass** | 3-column rows match the reference row by row |
| TC-10-006 | US-10 | AC-10.6 | test-form-ui | **pass** | `#ffffff` fill, `#c9d3e6` border, small radius |
| TC-10-007 | US-10 | AC-10.7 | test-form-ui | **fail** | 93.72% match, but spacing compression + missing dividers + missing placeholders ≠ "exactly" — D3/D4/D5/D10 |
| TC-10-008 | US-10 | AC-10.8 | test-form-ui | **pass** | centered pair, correct order and spacing |
| TC-11-001 | US-10 | AC-11.1 | test-form-ui | **pass** | 414 px: all fields `left=32`, `width=311` → single column |
| TC-12-001 | US-10 | AC-12.1 | create-form-tests | **pass\*** | **32/32** controls have an accessible name (23 `label[for]`, 9 `aria-label`) → WCAG-conformant; only 9 use `aria-label` literally |
| TC-12-002 | US-10 | AC-12.2 | **sentinel** | **pass** | 39/39 focusable, `tabIndex >= 0`, none disabled |
| TC-12-003 | US-10 | AC-12.3 | create-form-tests | **fail** | **measured**: `#e02020`/`#e8eef7` = 4.09:1; placeholder 2.93:1; border 1.51:1 — D11 |

### Counts

| By result | Count |
|---|---|
| **pass** | **32** (of which 16 are `pass*` — reference-correct equivalent) |
| **fail** | **9** |
| **blocked / unexecuted** | **7** |
| **Total** | **48** |

| By executor | Total | pass | fail | blocked |
|---|---|---|---|---|
| `test-form-ui` | 42 | 28 | 7 | 7 |
| `create-form-tests` | 4 | 2 | 2 | 0 |
| `sentinel` (functional/E2E) | 2 | 2 | 0 | 0 |

`unexecuted_cases = 7` — **must be 0** for a PASS. This is the primary gate blocker.

---

## 9. User-story coverage

A story is *covered* when ≥ 1 of its cases passed.

| Story | Title | Cases | Passing cases | Covered |
|---|---|---|---|---|
| US-01 | Owner provides identifying details | TC-01-001…004 | 4 | ✅ yes |
| US-02 | Owner provides contact information | TC-02-001…006 | TC-02-001, TC-02-003 | ✅ yes |
| US-03 | Vehicle type and basic vehicle information | TC-03-001…005 | TC-03-002, TC-03-005 | ✅ yes |
| US-04 | Vehicle fuel type | TC-04-001, 002 | 2 | ✅ yes |
| US-05 | Residential address details | TC-05-001…005 | 5 | ✅ yes |
| US-06 | Document / insurance details | TC-06-001…003 | TC-06-002 | ✅ yes |
| US-07 | Declaration | TC-07-001…004 | TC-07-001, TC-07-002 | ✅ yes |
| US-08 | Submit the registration | TC-08-001…004 | TC-08-001, 002, 003 | ✅ yes |
| US-09 | Reset the form | TC-09-001…003 | TC-09-001, TC-09-002 | ✅ yes |
| US-10 | Visual design, responsive & accessibility | TC-10-001…008, TC-11-001, TC-12-001…003 | 9 | ✅ yes |

**`user_stories: 10 total / 10 covered` · `uncovered_stories = 0`** ✅

Coverage is complete, but coverage alone does not carry the gate — the 9 failures and 7 unexecutable
cases do.

---

## 10. Defects

| ID | Sev | Finding | Route to | Status |
|---|---|---|---|---|
| **D1** | Critical | **Theme 404 for anonymous on publish.** Of the 20 CSS/JS URLs the publish page requests, only `…vehicle-registration-form.theme/_default/theme.css` and `theme.js` 404. `themeRef` points at `/apps/fd/af/themes/aem-adaptive-forms-agents-vehicle-registration`, which anonymous cannot read on publish. The DAM theme asset `/content/dam/formsanddocuments-themes/aem-adaptive-forms-agents/vehicle-registration` **is** deployed and populated (15 KB `renditions/original`, 5 KB `theme-json`) but nothing references it. Publish renders in browser-default serif, unthemed. *(pre-diagnosed on publish; recorded, not re-diagnosed)* | formwright (`create-form-theme`) | open |
| **D2** | Critical | **Rules dead on publish.** The publish page inlines no form model (`"rules"` and `fieldType` both absent from the HTML), so the runtime must fetch `…/guideContainer.model.json`, which 404s for anonymous even though `guideContainer.html` returns 200. No model → no rules, no client-side validation. *(pre-diagnosed on publish; recorded, not re-diagnosed)* | groundsmith / formwright | open |
| **D3** | Major | **6 section separator/divider rules missing** — the thin rule under the title/subtitle header and between each of the 5 numbered sections. Confirmed by pixels in `diff.png`, not by CSS inspection. | formwright (`create-form-clientlib`) | open |
| **D4** | Major | **Dropdown placeholder text "Select" not rendered** on State, Vehicle Type, Fuel Type, Manufacturing Year. All four have a blank first option; it simply has no display label. | formwright (`create-adaptive-form`) | open |
| **D6** | Major | **Authored validation messages pre-empted by built-in constraints.** Mobile shows "Please match the format requested." and Email shows "Specify the value in allowed format : email." instead of the authored copy. The `pattern` / `email`-format constraint fires before the `validationExpression` and wins the message slot. Also makes blur-time validation inconsistent. | formwright (`create-adaptive-form` / `create-form-rules`) | open |
| **D7** | Major | **DESI artifacts are stale — spec conflict.** `2026-07-28-.../design/test-cases.yaml` and `plan/user-stories.yaml` describe a different field inventory than the reference and the delivered form (see §7): 7 cases unexecutable, ~9 more read as failures on wording the reference contradicts. Includes the "Vehicle Type = Electric Vehicle" / "Date = today" default expectations, which the reference explicitly shows as "Select" / empty. | **draftsmith** (re-run `design-form-tests`) + **planwright** (re-baseline `user-stories.yaml`) | open — **primary gate blocker** |
| **D8** | Major | **`declarationDate` does not reject a future date.** Only `validateDateDDMMYYYY` is wired; no not-in-future guard. Confirmed live: `15/01/2030` accepted silently on Declaration Date while the same value was correctly rejected on Date of Birth. Fails AC-07.3 / TC-07-004. | formwright (`create-form-rules`) | open |
| **D9** | Major | **Approval workflow absent.** The PLAN specifies `submit: ["dor_pdf","workflow"]` with `assign-task-to-admin` (AC-08.4). There is no workflow model anywhere in the repo, no "Invoke an AEM Workflow" wiring, and a single `actionType` (`Custom-Submit-GeneratePDF`). | **groundsmith** (`create-workflow`) | open |
| **D11** | Major | **3 measured WCAG 2.1 AA contrast failures.** Error/asterisk red `#e02020` on page bg `#e8eef7` = **4.09:1** (needs 4.5); placeholder `#8a97b5` on `#ffffff` = **2.93:1**; field border `#c9d3e6` vs `#ffffff` = **1.51:1** (needs 3.0, WCAG 1.4.11 non-text). Fails TC-12-003. Suggested: red → ≈`#c01515`, placeholder → ≈`#5d6a8a`, border → ≈`#9aa8c4`. | formwright (`create-form-theme`) | open |
| **D5** | Minor | `DD / MM / YYYY` placeholder hint missing from all 4 date inputs (they are `type="text"`, so a `placeholder` will render). | formwright (`create-form-clientlib`) | open |
| **D10** | Minor | Cumulative vertical spacing ~4% tighter than the reference, producing progressive offset (≈80 px by section 5) — the bulk of the residual 6.29% mismatch. | formwright (`create-form-clientlib`) | open |
| **D12** | Minor | Gender radio controls smaller and more tightly spaced than the reference. | formwright (`create-form-theme`) | open |
| **D13** | Minor | Thank-you page after a valid submit is **unstyled** — browser-default serif on a blank page, no theme or branding. | groundsmith (`create-submit-action`) | open |

**No defect routes to Assembler** (the page embed is correct) or **Forgemaster** (the build and deploy
are sound — `BUILD SUCCESS`, all artifacts present and served).

> **D1 and D2 do not affect the author surface tested here** — both URLs return **200 on author with
> an authenticated session**, which isolates their cause to anonymous access / publish delivery
> rather than to missing artifacts. They remain open DEV defects against the publish tier.

---

## 11. Re-test route

Nothing was patched on the DEV environment, and no PR was merged or pipeline triggered by Sentinel.

Every fix must travel the full route:

**owning lead (source fix) → forgemaster (rebuild + local-SDK deploy) → pilot (commit, push, PR) →
⏸ human merges the PR → ⏸ human runs the Cloud Manager DEV pipeline → ⏸ human prompts Sentinel again.**

Recommended sequencing:

1. **First, resolve D7** — re-baseline the DESI test cases and user stories against the reference
   screenshot. This is the gate blocker and it costs no form change. Until it is done, 7 cases remain
   unexecutable and the loop cannot close. Do **not** rewrite the form to satisfy stale cases; that
   would regress a 93.72% replica.
2. Then the genuine behaviour gaps: **D8** (declaration-date future guard), **D6** (authored
   messages), **D9** (approval workflow).
3. Then the accessibility and polish items: **D11** (contrast), **D3**, **D4**, **D5**, **D10**,
   **D12**, **D13**.
4. **D1/D2** separately, as a publish-tier delivery fix, with a publish-tier re-test.

---

## 12. Reference artifacts

| Artifact | Path |
|---|---|
| This report (canonical) | `.claude/agents/runs/2026-08-12-vehicle-registration-form/testing/test-report.md` |
| HTML companion | `.claude/agents/runs/2026-08-12-vehicle-registration-form/testing/test-report.html` |
| UI parity detail | `.claude/agents/runs/2026-08-12-vehicle-registration-form/testing/test-form-ui-report.md` |
| Unit + integration detail | `.claude/agents/runs/2026-08-12-vehicle-registration-form/testing/integration-test-report.md` |
| Pilot SCM record (commit / PR) | `.claude/agents/runs/2026-08-12-vehicle-registration-form/scm/pilot.md` |
| Forgemaster build + deploy gate | `.claude/agents/runs/2026-08-12-vehicle-registration-form/deployment/code-quality-report.md` |
| Assembler page embed | `.claude/agents/runs/2026-08-12-vehicle-registration-form/assembly/assembler.md` |
| DESI test cases (**stale** — see D7) | `.claude/agents/runs/2026-07-28-vehicle-registration-form/design/test-cases.yaml` |
| User stories (**stale** — see D7) | `.claude/agents/runs/2026-07-28-vehicle-registration-form/plan/user-stories.yaml` |
| Captures + pixel result (images stay here, never in `runs/`) | `ui.tests/test-module/cypress/results/form-ui/vrf/` |
| Pristine reference | `ui.tests/test-module/cypress/results/form-ui/reference.png` |

Text-only in `runs/` — **no image files**. The HTML companion *references* the Cypress PNGs by
relative path.

---

## 13. Handoff YAML

```yaml
agent: sentinel
phase: TEST
status: FAILED
started_on: human-prompt
environment:
  target: cloud-dev
  tier: author                    # primary surface; publish referenced only for D1/D2
  author_url: "https://author-p185256-e1945105.adobeaemcloud.com"
  publish_url: "https://publish-p185256-e1945105.adobeaemcloud.com"
  program: "p185256"
  env_name: "e1945105"
  merged_commit: "e570de76f0436cda6e8fa9d1c7f387762c505a87"
  merge_commit_on_main: "bfbd5ad"
  pull_request: "https://github.com/arjunvenu-cts/AEM-Adaptive-Forms-Agent/pull/5"
  cm_dev_deploy_confirmed: true
  auth: "adobe-ims session cookie (login-token) — no admin:admin on AEMaaCS"
  local_only_checks: ["create-form-tests (AEM Mocks — no instance)"]
test_cases: { total: 48, executed: 41, passed: 32, failed: 9 }
unexecuted_cases: 7            # MUST be 0 — blocked by stale DESI artifacts (D7)
executes_with: { create-form-tests: 4, test-form-ui: 42, functional: 2 }
coverage_pct: 100              # CustomSubmitGeneratePDFAction 100%, GeneratePDFServlet 96.4% (local)
unit_tests: { run: 22, failures: 0, errors: 0 }
user_stories: { total: 10, covered: 10 }
uncovered_stories: 0
ui_parity:
  pixel: PASS
  pixel_match_pct: 93.72
  pixel_mismatch_pct: 6.285
  critical_findings: 0
  major_findings: 3
  minor_findings: 3
  gate: PASS                   # >=90% match AND 0 Critical
  captured_from: "cloud DEV author, cropped to the embedded form region"
  responsive_single_column: true
embedded_page:
  path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form"
  url: "https://author-p185256-e1945105.adobeaemcloud.com/content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html?wcmmode=disabled"
  embeds_new_form: true
  container_count: 1
  stale_form_present: false
  duplicate_form: false
  functional_in_page: true
rule_asts: { total: 7, valid_node_names: 7, value_expression_occurrences: 0 }
submit_gating: { empty_submit_blocked: true, inline_errors: 23, pdf_on_invalid: false, valid_submit_ok: true }
defects:
  - { id: D1,  sev: critical, area: publish-theme-404,        route: formwright,  status: open }
  - { id: D2,  sev: critical, area: publish-model-404-rules,  route: groundsmith, status: open }
  - { id: D3,  sev: major,    area: missing-section-dividers, route: formwright,  status: open }
  - { id: D4,  sev: major,    area: dropdown-select-placeholder, route: formwright, status: open }
  - { id: D6,  sev: major,    area: authored-validation-messages, route: formwright, status: open }
  - { id: D7,  sev: major,    area: stale-desi-artifacts,     route: draftsmith+planwright, status: open }
  - { id: D8,  sev: major,    area: declarationdate-future-guard, route: formwright, status: open }
  - { id: D9,  sev: major,    area: approval-workflow-absent, route: groundsmith, status: open }
  - { id: D11, sev: major,    area: wcag-aa-contrast,         route: formwright,  status: open }
  - { id: D5,  sev: minor,    area: date-placeholder-hint,    route: formwright,  status: open }
  - { id: D10, sev: minor,    area: vertical-spacing-drift,   route: formwright,  status: open }
  - { id: D12, sev: minor,    area: radio-control-sizing,     route: formwright,  status: open }
  - { id: D13, sev: minor,    area: unstyled-thankyou-page,   route: groundsmith, status: open }
report: ".claude/agents/runs/2026-08-12-vehicle-registration-form/testing/test-report.md"
report_html: ".claude/agents/runs/2026-08-12-vehicle-registration-form/testing/test-report.html"
gate_result: FAIL
gate_blockers:
  - "unexecuted_cases = 7 (must be 0) — DESI test cases describe a different field inventory (D7)"
  - "9 executed cases failed (D6, D7, D8, D9, D3, D11 and the TC-10-007 aggregate)"
notes:
  - "UI parity, embedded-page, submit gating, unit tests and story coverage all PASS."
  - "Most failures are specification drift, not broken software — the form matches its reference at 93.72%."
  - "Nothing was patched on DEV; no PR merged and no pipeline triggered by sentinel."
next: >
  Re-baseline the DESI artifacts (D7) via draftsmith/planwright, then fix D8/D6/D9 and the
  accessibility/polish items through formwright/groundsmith -> forgemaster -> pilot -> human merge ->
  Cloud Manager DEV -> a new human prompt to sentinel for re-test.
```
