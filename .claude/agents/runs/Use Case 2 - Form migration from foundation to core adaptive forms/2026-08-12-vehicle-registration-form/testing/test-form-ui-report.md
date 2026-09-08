# Form UI Comparison Report — Vehicle Registration Form

| | |
|---|---|
| **Environment under test** | **cloud DEV — AEMaaCS AUTHOR tier** (program `p185256`, environment `e1945105`) |
| Form URL (capture source) | `https://author-p185256-e1945105.adobeaemcloud.com/content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html?wcmmode=disabled` |
| Capture scope | `form` — cropped to `.cmp-adaptiveform-container` (the embedded form region only, excluding the Sites page nav/search/title/footer chrome) |
| Form under test | `/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form` |
| Reference | `ui.tests/test-module/cypress/results/form-ui/reference.png` (screenshot, 1258×2066 — the Vehicle Registration source design) |
| Viewport | 1280×1024 (desktop) · 414×896 (mobile) |
| Normalised diff size | 1250×1689 |
| Auth path | Adobe IMS **session cookie** (`login-token`) reused from the human's browser session. `admin:admin` does not exist on AEMaaCS; `cy.AEMLogin()` cannot complete the IMS SSO flow and was therefore not used. |
| Gate | Replica delivery → `MAX_MISMATCH_PCT = 10` (≥ 90% pixel match required) |
| Run date | 2026-08-13 |

> Not localhost. Every pixel in this report was captured from the cloud DEV author tier at the URL above.

---

## Verdict: **PASS** (UI parity gate)

- **Pixel diff:** **6.29% mismatch = 93.72% match** (132,693 mismatched of 2,111,250 px) vs the 10% threshold → **PASS**
- **Semantic findings:** **0 Critical** · 3 Major · 3 Minor
- **Overall UI-parity gate = PASS** — both bars are met: ≥ 90% pixel match **and** zero Critical findings.

The pixel percentage is **reliable** here: the reference and the capture are the same form at the same
aspect, and the capture was cropped to the form region, so no page chrome pollutes the diff. No
appeal to "mismatched-aspect reference" is being made.

The residual 6.29% is dominated by **cumulative vertical spacing drift**, not by content. In `diff.png`
every label appears **twice** — once at its reference position and once at its (slightly higher) actual
position — with the offset growing down the page to roughly 80 px by the DECLARATION section. No field
appears in only one of the two images.

## Field inventory — exact parity with the reference

Cross-checked against `guideContainer.model.json` (37 nodes) rather than guessed from pixels.

| Section | Reference fields | Form fields | Delta |
|---|---|---|---|
| 1. OWNER DETAILS | Full Name\*, Date of Birth\*, Gender\* (Male/Female/Other), Mobile Number\*, Email Address (no \*), Address\*, City\*, State\* (Select), PIN Code\* | identical | none |
| 2. VEHICLE DETAILS | Vehicle Type\* (Select), Manufacturer\*, Model\*, Registration Number\*, Chassis Number\*, Engine Number\*, Fuel Type\* (Select), Color\*, Manufacturing Year\* (Select) | identical | none |
| 3. INSURANCE DETAILS | Insurance Company\*, Policy Number\*, Policy Valid Till\* | identical | none |
| 4. DOCUMENT CHECKLIST | 6 checkboxes: Proof of Identity, Proof of Address, Vehicle Insurance, Pollution Under Control (PUC) Certificate, Road Tax Receipt, Others (if any) | identical, 6 rendered | none |
| 5. DECLARATION | declaration text, Place\*, Date\*, Signature / Full Name\* | identical | none |
| Actions | Submit (navy solid), Reset (outlined) | identical | none |

- **Missing in form:** none · **Extra in form:** none
- **Email Address correctly carries NO red asterisk** in both reference and form (it is genuinely optional — `required: false`).
- Row-by-row column grouping matches the reference (3 fields per row; Address is a 1-column textarea, not full-width — as in the reference).
- **No error state at initial load** — 0 visible error messages on a clean render (verified programmatically), matching the reference.

## Findings

| # | Severity | Type | Reference shows | Form renders | Recommendation | Route |
|---|---|---|---|---|---|---|
| 1 | **Major** | Missing separator | A thin horizontal divider under the title/subtitle header **and** between each of the 5 numbered sections (6 rules total) | No divider rules at all | Draw the rules as a `border-bottom` on the served section-panel classes (`.cmp-adaptiveform-panelcontainer`) / an `::after` rule under the header block. Confirm by pixels, not by CSS presence. | `create-form-clientlib` (formwright) |
| 2 | **Major** | Missing placeholder text | The word **"Select"** inside State, Vehicle Type, Fuel Type and Manufacturing Year | Those 4 dropdowns render a **blank** first option | Give each dropdown an explicit empty-value first option whose display text is `Select` (all four already have a blank option — it just has no label). | `create-adaptive-form` (formwright) |
| 3 | **Major** | Contrast (a11y, measured) | — | Red `#e02020` on page bg `#e8eef7` = **4.09:1** (< AA 4.5); placeholder `#8a97b5` on `#ffffff` = **2.93:1**; field border `#c9d3e6` vs `#ffffff` = **1.51:1** (< 3.0, WCAG 1.4.11) | Darken the error/asterisk red to ≈ `#c01515`, the placeholder to ≈ `#5d6a8a`, and the field border to ≈ `#9aa8c4`. | `create-form-theme` (formwright) |
| 4 | Minor | Missing placeholder text | `DD / MM / YYYY` hint inside all 4 date inputs | Date inputs render empty (calendar icon only) | Add a `placeholder="DD / MM / YYYY"` to the datepicker widgets (they are `type="text"`, so a placeholder will render). | `create-form-clientlib` (formwright) |
| 5 | Minor | Spacing | Looser row rhythm — reference content is ~2066 px tall for the same 5 sections | Actual is ~4% vertically compressed, producing progressive offset (≈80 px by section 5) | Increase per-row / per-section bottom margin so the cumulative height matches the reference. | `create-form-clientlib` (formwright) |
| 6 | Minor | Control size | Gender radio circles larger with wider inter-option gaps | Radios smaller and more tightly packed | Increase radio control size and option gap in the theme. | `create-form-theme` (formwright) |

### Explicitly verified visuals (not just fields)

Per the "UI parity checks EVERY visual" rule, each non-field visual was checked **by pixels**:

| Visual element | Reference | Form | Verdict |
|---|---|---|---|
| Title "VEHICLE REGISTRATION FORM" — centered, bold, navy | yes | yes, `#1a2b5e` | **PASS** |
| Subtitle line, regular weight, left-aligned | yes (left) | yes (left), `#3a4a70` | **PASS** |
| Section headings — bold, ALL CAPS, numbered 1–5, navy | yes | yes | **PASS** |
| **Section separator / divider rules** | **6** | **0** | **FAIL** (finding 1) |
| Red required asterisks on every required label | yes | yes, `#e02020`, rendered | **PASS** |
| No asterisk on optional Email Address | yes | yes | **PASS** |
| Light-blue card / page background band | `#e8eef7` | `#e8eef7` | **PASS** |
| Field styling — white fill, thin border, small radius | yes | `#ffffff` on `#c9d3e6` | **PASS** |
| Submit — solid navy, white label | yes | yes, `#1a2b5e` / `#ffffff` | **PASS** |
| Reset — white fill, navy border + navy label | yes | yes | **PASS** |
| Button pair centered, Submit then Reset | yes | yes | **PASS** |
| Declaration static text visible | yes | yes (wording matches reference exactly) | **PASS** |
| Images / logos / icons / background bands | none besides the calendar icons | calendar icons render | **PASS** |

No icon, logo or background band exists in the reference beyond the date-field calendar glyphs, which render.

## Responsive / mobile

`form-ui-mobile.cy.js` at **414×896**: all sampled fields (`fullName`, `dateOfBirth`, `mobileNumber`,
`city`) report `left = 32` and `width = 311` → the 3-column grid **collapses to a single column** below
768 px. **PASS.** (`cypress/results/form-ui/vrf/mobile-layout.json`)

## Submit-gating cross-check (functional, required alongside the visual pass)

| Check | Result |
|---|---|
| Empty/invalid form BLOCKS submission | **PASS** — 23 inline errors, form still rendered |
| Inline errors shown | **PASS** — e.g. "Please enter your full name." |
| First invalid field focused | **PASS** — focus ring on Full Name (visible in the capture) |
| NO PDF / no submit POST on invalid | **PASS** — 0 intercepted POSTs |
| Valid form submits AND reaches its action | **PASS** — POST to `/adobe/forms/af/submit/…`, "Thank you for submitting the form." |

Submit is correctly gated on validation — no Critical defect here.

## Images

Not stored in the run directory (AGENTS.md forbids image files under `runs/`). See the Cypress
working dir `ui.tests/test-module/cypress/results/form-ui/vrf/`:

| File | What |
|---|---|
| `reference.png` | the source design, normalised to the capture size |
| `actual.png` | the cloud DEV author capture, cropped to the form region |
| `diff.png` | pixel diff (red = mismatch) |
| `pixel-result.json` | `{ mismatched: 132693, total: 2111250, mismatchPercentage: 6.285, width: 1250, height: 1689 }` |
| `evidence-embedded-page-clean.png` | the whole "Test Adaptive Form" page, clean render |
| `evidence-empty-submit-blocked.png` | empty submit → 23 inline errors |
| `evidence-format-and-date-errors.png` | format + future-date errors |
| `evidence-valid-submit-accepted.png` | thank-you after a valid submit |
| `evidence-mobile-414.png` | single-column mobile render |
| `mobile-layout.json` | measured per-field left/width at 414 px |

The pristine 1258×2066 reference remains untouched at
`ui.tests/test-module/cypress/results/form-ui/reference.png` — the diff task wrote its normalised
copy into the `vrf/` subdirectory instead of overwriting it.

## Recommendation

**Ship-worthy on parity, fix-then-ship on polish.** The form is a faithful replica: 93.72% pixel match,
a byte-for-byte field inventory match against the reference, correct 3-column layout, correct colours,
correct button treatment, no premature error state, and a correctly gated submit. There are **no
Critical findings**, so the parity gate passes.

Top three actions, in order:

1. **Add the 6 missing separator/divider rules** (finding 1) — the single most visible delta, and the
   one the reference most clearly defines. Route to `create-form-clientlib`.
2. **Fix the 3 measured WCAG AA contrast deficiencies** (finding 3) — the error-message/asterisk red at
   4.09:1 is a genuine accessibility failure on the light-blue band. Route to `create-form-theme`.
3. **Restore the "Select" and "DD / MM / YYYY" placeholder text** (findings 2 and 4) — cheap, and it
   removes most of the remaining perceived difference.

Findings 5 and 6 (spacing rhythm, radio sizing) are cosmetic and can ride along with the clientlib fix;
together they would close most of the residual 6.29%.

All fixes go the full route — owning lead → forgemaster → pilot → human merge → Cloud Manager DEV
pipeline → a new human prompt to Sentinel. **Nothing was patched on the DEV environment.**

## Run mechanics (for reproducibility)

```bash
cd ui.tests/test-module
export CYPRESS_FORM_URL="https://author-p185256-e1945105.adobeaemcloud.com/content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html?wcmmode=disabled"
export CYPRESS_AEM_LOGIN_TOKEN='<login-token cookie value>'   # never written into runs/
export CYPRESS_REFERENCE_IMAGE="<scratchpad>/vrf-reference.png"
export CYPRESS_CAPTURE_SCOPE="form"
export CYPRESS_FORM_SELECTOR=".cmp-adaptiveform-container"
export CYPRESS_VIEWPORT="1280x1024"
export CYPRESS_MAX_MISMATCH_PCT="10"
export CYPRESS_OUT_DIR="cypress/results/form-ui/vrf"
export CYPRESS_SHOT_PREFIX="form-ui/vrf"
export REPORTS_PATH="cypress/results"
npx cypress run --browser chrome --spec "cypress/e2e/form-ui-compare.cy.js"
```

Two harness notes worth keeping:

- Every spec var **must** carry the `CYPRESS_` prefix or `Cypress.env()` returns `undefined` and the
  `before` hook dies with "Cannot read properties of undefined (reading 'includes')".
- `form-ui-compare.cy.js` and `form-ui-mobile.cy.js` were updated to authenticate to cloud author via
  the `login-token` cookie instead of `cy.AEMForceLogout()` / `cy.AEMLogin()`, which cannot complete
  Adobe IMS SSO. The container gate also asserts `.should('exist')` rather than `be.visible`.
- Cypress `trashAssetsBeforeRuns` defaults to **true** and deletes the screenshots folder between
  spec runs — the evidence captures were re-run with `--config trashAssetsBeforeRuns=false`.
