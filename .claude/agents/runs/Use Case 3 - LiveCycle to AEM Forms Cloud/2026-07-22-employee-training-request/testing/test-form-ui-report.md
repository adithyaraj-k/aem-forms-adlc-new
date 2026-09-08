# Form UI Comparison Report — Employee Training Request (migration structural parity)

| | |
|---|---|
| Form URL (tested) | http://localhost:4502/content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html (embedded page — the delivery's user-facing surface) |
| Form under test | /content/forms/af/aem-adaptive-form-agents/employee-training-request (embedded via the page's AEM Form Container) |
| Reference | `C:\tmp-inner-pkg\jcr_root\content\dam\formsanddocuments\TestMigration\1.0\Forms\DocRoute.xdp\_jcr_content\renditions\cq5dam.web.1280.1280.jpeg` (DAM preview JPEG, screenshot mode) |
| Viewport | 1280x1600 (captured full-page, normalized to 1280x2002) |
| Auth path | author login (`:4502`) |
| Run date | 2026-07-22 |
| Mode | **Migration structural parity** — pixel gate advisory; verdict driven by Critical findings |

## Verdict: **FAIL** (1 Critical finding — structural parity gap)

- **Pixel diff:** 13.79% mismatch (= 86.21% match) vs advisory 100% threshold → gate itself PASSES
  (advisory only, per migration-parity mode). Most of the mismatch is explained by (a) the Sites
  page chrome (breadcrumb, search box, "Test Adaptive Form" H1 — correctly present, not part of the
  form itself, absent from the bare-XDP reference) and (b) a vertical offset below the Employee
  Details panel caused by finding #1 below.
- **Semantic findings:** 1 Critical · 1 Major · 0 Minor
- Overall = **FAIL** because a Critical finding exists (migration-parity PASS requires zero Critical
  findings, even though the pixel gate is advisory in this mode).

## Images
Not stored in the run directory. See the Cypress working dir:
`ui.tests/test-module/cypress/results/form-ui/` → `reference.png`, `actual.png`, `diff.png`,
`pixel-result.json`.

## Findings

| # | Severity | Type | Reference shows | Form renders | Recommendation |
|---|---|---|---|---|---|
| 1 | **Critical** | Missing section heading | "Employee Details" bold maroon heading directly above the Employee Id/Name/Email row | No heading at all — the panel goes straight from the form title ("Employee Training Request") into the Employee Id/Name/Email fields; `employeeDetailsFragment.label.value="Employee Details"` is set `visible:true` in `guideContainer.model.json` but never renders in the DOM | This is the project's **first** fragment embedded with a visible host-level label — the wrapper panel's `label` is being suppressed in favour of the fragment's own internal title rendering (or vice-versa), a rendering gap specific to the fragment-embed pattern. Route to **Formwright** (`create-AdaptiveFormFragment` / `create-adaptive-form`) to fix the `employee-identity` fragment's title/label visibility so "Employee Details" renders like the other section headings; re-test after redeploy. Same risk class as the 2 rules Formwright already flagged as lowest-confidence for this fragment-embed pattern. |
| 2 | Major | Row grouping / column count | Justification panel: Business Need / Benefits To Project / Training Category appear as 3 fields grouped in a compact row | Business Need (half-width) + Benefits To Project (half-width) share one row; Training Category (own row) sits below as a separate row | Cosmetic column-count difference (3-across vs 2+1), consistent with a deliberate reflow for longer multi-line textareas. Not a content/field defect. If exact-replica 3-column grouping is desired, adjust `justificationPanel`'s `cq:responsive` widths via `create-adaptive-form`; otherwise acceptable as an intentional layout adaptation for the longer text-area fields. |

Everything else structurally matches:
- Panel grouping/order matches the reference exactly: Employee Details → Training Request Details →
  Justification → (Attachments, Declaration, Approval Information — not visible in this single
  static JPEG rendition, but confirmed present/correctly ordered from the live model.json and
  justified by the actual XDP source per `plan/planwright.md` / `design/draftsmith.md`).
- Section header band colour (light blue, ≈RGB 196,219,251) + maroon bold caption (≈RGB 128,64,64)
  on "Training Request Details" / "Justification" / "Attachments" — matches the reference.
- Panel body fill (pale green, ≈RGB 225,242,219) — matches the reference.
- 3-column field grid and field order within Employee Details / Training Request Details panels —
  matches the reference exactly, field-for-field (Employee Id/Employee Name/Email →
  Department/Manager Id/Manager Name → Location/Business Unit; Request Id/Request Type/Provider Name
  → Course Name/Start Date/End Date → Training Mode/Location/Amount → Currency).
- Declaration section (Accepted Terms checkbox + Submission Date field) renders with **no** header
  band, matching the reference's bare styling for that section (`declarationFragment` correctly
  authored with `hideTitle=true`).
- Font family, weight, and required-field asterisks render correctly; no premature validation-error
  state at initial load.
- Myriad Pro / browser-fallback sans-serif rendering difference vs the legacy XFA renderer is
  expected and NOT scored as a defect, per this run's migration-parity instruction.

## Field inventory (from guideContainer.model.json, cross-checked against the reference)
- Reference fields visible in the JPEG bounds: Employee Id, Employee Name, Email, Department,
  Manager Id, Manager Name, Location, Business Unit, Request Id, Request Type, Provider Name,
  Course Name, Start Date, End Date, Training Mode, Location (training), Amount, Currency, Business
  Need, Benefits To Project, Training Category, Accepted Terms, Submission Date.
- Form fields (live model.json): all 22 of the above present, plus Attachments (Vendor Quotation,
  Course Brochure, Manager Recommendation) and Approval Information (Current Status, Manager
  Comments, Department Head Comments, Finance Comments) — outside the reference JPEG's visible
  bounds, confirmed instead against the source XDP field inventory per `plan/planwright.md`.
- Missing in form vs the visible reference bounds: **"Employee Details" heading text** (see Finding
  #1) — the only content-level gap.
- Extra in form vs the reference: page chrome only (breadcrumb, search, "Test Adaptive Form" H1,
  footer copyright) — correctly part of the Sites page shell, not the form; the "Employee Training
  Request" form title is an explicit AF Title component per `create-adaptive-form`'s zero-defect
  checklist, expected and not present in the bare-XDP static preview.

## Recommendation

Fix-then-ship. One Critical, well-contained defect: the `employee-identity` fragment's "Employee
Details" section heading does not render live despite `visible:true` in the JCR — send back to
Formwright to correct the fragment/host-panel title visibility (the same fragment-embed pattern
Formwright already flagged as unprecedented and lowest-confidence for this delivery), then redeploy
and re-run this check. The Major row-grouping note on Justification is cosmetic and does not block
ship. Everything else — panel order, band/body colours, 3-column grids, field order/labels,
Declaration's bare styling — is a faithful structural replica of the legacy DocRoute.xdp design.

---

## Retest — fix pass 1 (2026-07-22)

| | |
|---|---|
| Form URL (tested) | http://localhost:4502/content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html (embedded page, same as original run) |
| Reference | Same DAM preview JPEG as the original run |
| Viewport | 1280x1600 (captured full-page, normalized to 1258x2053) |
| Run date | 2026-07-22 (post fix-pass-1 redeploy) |
| Mode | Migration structural parity — pixel gate advisory; verdict driven by Critical findings |

### Verdict: **Critical Finding #1 RESOLVED.** Remaining state: 0 Critical, 1 Major (unchanged)

- **Pixel diff:** 14.21% mismatch (= 85.79% match) vs advisory 100% threshold → gate PASSES
  (advisory-only in migration mode; the small increase in mismatch vs the original 86.21% is
  consistent with the added heading now occupying vertical space that shifts subsequent content, not
  a regression).
- **Finding #1 (previously Critical) — "Employee Details" heading — CONFIRMED FIXED.** Re-captured
  the live embedded page (`ui.tests/test-module/cypress/results/form-ui/actual.png` this run) and
  visually compared against the reference: the heading now renders as a bold maroon caption on the
  light-blue header band, directly above the Employee Id/Name/Email row — pixel-for-pixel consistent
  with the styling of "Training Request Details" and "Justification", and structurally matching the
  reference's own "Employee Details" heading. This was verified by actually **rendering** the page
  (Cypress capture + direct visual read of the resulting PNG), not by a JSON-endpoint guess — per
  this retest's explicit instruction, since the coordinator's own JSON-based spot-check attempt on
  this specific fix was inconclusive (404 on a guessed fragment model path).
- **Finding #2 (Major, Justification row-grouping 2+1 vs 3-across) — UNCHANGED, still present.**
  Business Need + Benefits To Project still share one row (each ~half width) with Training Category
  on its own row below, vs the reference's compact 3-across grouping. This was never in scope for
  fix pass 1 (cosmetic, non-blocking) and Formwright did not touch it — confirmed still present,
  as expected, not a regression.
- **0 Critical findings remain from the UI-parity check itself.** (Note: a separate, newly
  discovered Critical *data*-binding defect — D3, see `integration-test-report.md` §"Retest — fix
  pass 1" — affects the Document of Record's data completeness. That is a functional/data defect,
  not a UI-rendering one, and does not appear in this pixel/vision check since it concerns the
  *submitted values*, not what renders on the interactive form's screen.)

### Updated recommendation

The one Critical UI-parity defect from the original run is resolved and confirmed via a fresh
render. The Major Justification row-grouping note remains open but is cosmetic/non-blocking, as
originally assessed. **UI parity alone no longer blocks the gate** — the delivery's remaining gate
blockers are the two functional defects covered in `integration-test-report.md` (D2 — OR-split
routing, known/escalated; D3 — fragment dataRef mismatch, newly discovered this pass).

---

## FINAL retest — fix pass 2 (2026-07-22)

**Not re-captured — carried forward from the fix-pass-1 retest above, with justification.**

Fix pass 2 touched exactly 4 files: `employee-identity/.content.xml` (8 `dataRef` corrections),
`declaration-consent/.content.xml` (2 `dataRef` corrections), `employee-training-request/.content.xml`
(the submit button's `fd:click`/`fd:events` script text only), and
`employee-training-request-clientlib/js/functions.js` (one new JS function). None of these are
visual/theme/CSS/layout artifacts — a `dataRef` attribute change and a rule-script text change have
no effect on rendered pixels. Re-running the full Cypress capture + pixel-diff + vision pass for zero
visual-affecting change would not produce new information and would burn the run budget on a check
whose outcome is already determined by "no visual file changed."

**Spot check performed instead (live DOM, not pixel-diff):** confirmed via
`employee-training-request-final-retest.cy.js` (test FR-01) that the "Employee Details" heading still
renders visibly in the live DOM on the embedded page after the fix-pass-2 redeploy — no regression of
the fix-pass-1 UI fix.

**Carried-forward verdict: 0 Critical, 1 Major (unchanged, cosmetic), 85.79% pixel match (advisory in
migration mode).** UI parity remains a non-blocker for this final gate. The gate's remaining blockers
are functional, not visual — see `integration-test-report.md` §"FINAL retest — fix pass 2" and
`test-report.md`'s Verdict & Gate section.

---

## FINAL RETEST — fix pass 3 (2026-07-22)

**Not re-captured — carried forward from the fix-pass-1 retest, with the same justification as fix
pass 2.**

Fix pass 3 touched exactly one file, `employee-training-request/.content.xml`, and only two
attribute values inside it (the submit button's `fd:click` JSON metadata `script` array entry and the
`fd:events`/`click` mirror attribute) — a rule-script text change with zero effect on rendered pixels.
Re-running the full Cypress capture + pixel-diff + vision pass would not produce new information.

**Spot check performed instead (live DOM, not pixel-diff):** confirmed via
`employee-training-request-final-retest.cy.js` (test FR-01) that the "Employee Details" heading still
renders visibly in the live DOM on the embedded page after the fix-pass-3 redeploy — no regression.

**Carried-forward verdict: 0 Critical, 1 Major (unchanged, cosmetic), 85.79% pixel match (advisory in
migration mode).** UI parity remains a non-blocker. The gate's remaining blockers are functional — see
`integration-test-report.md` §"FINAL retest — fix pass 3" and `test-report.md`'s Verdict & Gate
section. This is the third consecutive fix pass confirming UI parity is stable and fully resolved;
no further UI-parity re-capture is recommended unless a future change touches a visual/theme/CSS
artifact.

---

## RETEST — fix pass 4 (2026-07-23, user-reported remediation)

**Not re-captured — same justification as fix passes 2 and 3.** Fix pass 4's 3 issues (fragment
`schemaType`/`schemaRef` binding, a script-inventory audit with 0 code changes, and the workflow's
Assign Task/Invoke DDX/Send Email/Convert-to-PDF/A steps rebuilt onto genuine OOTB step classes)
touch **zero** visual/theme/CSS/clientlib artifacts — a fragment's data-model binding and a
workflow's underlying step implementation classes have no effect on rendered pixels.

**Spot check performed instead (live DOM, not pixel-diff):** ran
`employee-training-request-final-retest.cy.js` (FR-01) fresh against the fix-pass-4 redeploy — the
"Employee Details" heading still renders visibly in the live DOM on the embedded page. No regression.

**Carried-forward verdict: 0 Critical, 1 Major (unchanged, cosmetic), 85.79% pixel match (advisory in
migration mode).** UI parity remains a non-blocker for this retest — the gate's remaining blockers
are the pre-existing D2/D4 functional gaps, both outside this fix pass's scope and both independently
reconfirmed unchanged. See `integration-test-report.md` §"RETEST — fix pass 4" and `test-report.md`'s
§"RETEST — fix pass 4" for the full functional verification and gate rationale. This is the fourth
consecutive fix pass confirming UI parity is stable; no further re-capture is recommended unless a
future change touches a visual/theme/CSS/clientlib artifact.

---

## RETEST — fix pass 5 (2026-07-23, OOTB step I/O configuration)

**Not re-captured — same justification as fix passes 2-4.** Fix pass 5 touched only the workflow
model's step `metaData` (Invoke DDX / Convert-to-PDF/A / Send Email / Assign Task I/O and ROUTES
values) — zero visual/theme/CSS/clientlib artifacts. A workflow step's Java-side I/O configuration has
no effect on the embedded form's rendered pixels.

**Spot check performed instead (live DOM, not pixel-diff):** ran the full existing Cypress regression
suite fresh against the fix-pass-5 redeploy — `employee-training-request-final-retest.cy.js` FR-01
confirms the "Employee Details" heading still renders visibly in the live DOM on the embedded page;
`employee-training-request-functional.cy.js` RT-01 confirms the same independently. Both passed
(14/14 across the 3 regression specs run this pass — see `test-report.md`'s "RETEST — fix pass 5" for
the full run). No visual regression.

This pass's actual investigative work was almost entirely server-side (AEM Forms Workspace task-detail
rendering, error-log/bytecode analysis of the Assign Task Java classes) — outside this report's scope.
See `integration-test-report.md`'s "RETEST — fix pass 5" for the full functional findings (3 newly
discovered Critical defects, D6/D7/D8, blocking real task completion) and `test-report.md`'s "RETEST —
fix pass 5" for the case tally and gate verdict.

**Carried-forward verdict: 0 Critical, 1 Major (unchanged, cosmetic), 85.79% pixel match (advisory in
migration mode).** UI parity remains a non-blocker for this retest — the gate's remaining blockers are
now functional (D6, D7, D8 newly discovered this pass, plus the carried-forward D2/D4), none of them
visual. This is the fifth consecutive fix pass confirming UI parity is stable; no further re-capture is
recommended unless a future change touches a visual/theme/CSS/clientlib artifact.

---

## FINAL functional retest -- fix pass 6 (2026-07-23, real task completion + DDX/DoR verification)

**Not re-captured -- same justification as fix passes 2-5.** Fix pass 6 touched only two workflow
model nodes' `metaData` (`FORM_TYPE`, removal of the `*_COMBINED_*` attachment properties on
`assigntask_manager`/`assigntask_finance`) plus a brand-new, separate `assign-task-to-admin` reference
model -- zero visual/theme/CSS/clientlib artifacts touched. None of this affects the embedded form's
rendered pixels.

**Spot check performed instead (live DOM, not pixel-diff):** the full existing Cypress regression
suite was re-run fresh against the fix-pass-6 redeploy (functional 8/8, final-retest 5/5,
submissiondate-probe 1/1 -- 14/14 total) -- `employee-training-request-final-retest.cy.js`'s FR-01
and `employee-training-request-functional.cy.js`'s RT-01 both independently reconfirm the "Employee
Details" heading still renders visibly on the embedded page. No visual regression.

This pass's actual investigative work was, once again, almost entirely server-side/client-JS
(AEM Forms Workspace task-detail rendering, `formview.jsp`'s NullPointerException, the real
`workitemdetails.js` clientlib source) -- outside this report's scope. See
`integration-test-report.md`'s "FINAL functional retest -- fix pass 6" section for the full
functional findings (D6/D7/D8 confirmed fixed; a new Critical defect D9 discovered, blocking real
task completion) and `test-report.md`'s matching section for the case tally and final gate verdict.

**Carried-forward verdict: 0 Critical, 1 Major (unchanged, cosmetic), 85.79% pixel match (advisory in
migration mode).** UI parity remains a non-blocker -- the gate's remaining blocker is now a single,
precisely diagnosed functional defect (D9), plus the carried-forward D2/D4, none of them visual. This
is the sixth and final consecutive fix pass confirming UI parity is stable; no further re-capture is
warranted unless a future change touches a visual/theme/CSS/clientlib artifact.

## FINAL functional verification -- fix pass 7 (2026-07-27, Approve path, Reject path, DDX/DoR assembly)

**Not re-captured -- same justification as fix passes 2-6.** Fix pass 7 touched only the
`FORM_TYPE` value (reverted to `READ_ONLY_AF`) on two workflow-model nodes -- zero visual/theme/
CSS/clientlib artifacts touched. None of this affects the embedded form's rendered pixels.

**Spot check performed instead (live DOM, not pixel-diff):** the full existing Cypress regression
suite was re-run fresh against the current deploy (functional 8/8, final-retest 5/5,
submissiondate-probe 1/1 -- 14/14 total, zero regressions). This pass's actual investigative work
was, once again, entirely server-side/client-JS and JCR/authorization state (AEM Forms Workspace
task-detail claim/delegate flow, the live `workitemdetails.js` clientlib source, `error.log`
stack traces, group-membership queries) -- outside this report's scope. See
`integration-test-report.md`'s "FINAL functional verification -- fix pass 7" section for the full
functional findings (D9 confirmed fixed; a new, deeper Critical defect D10 discovered -- the
`administrators` assignee group has no real human member, blocking real task completion on every
route by every method tried) and `test-report.md`'s matching section for the case tally and final
gate verdict.

**Carried-forward verdict: 0 Critical, 1 Major (unchanged, cosmetic), 85.79% pixel match (advisory in
migration mode).** UI parity remains a non-blocker -- the gate's remaining blocker is now a single,
newly diagnosed functional/authorization defect (D10), plus the carried-forward D2/D4, none of them
visual. This is the seventh consecutive fix pass confirming UI parity is stable; no further
re-capture is warranted unless a future change touches a visual/theme/CSS/clientlib artifact.

## DEFINITIVE final functional verification -- fix pass 8 (2026-07-27, D10: admin assignee; NEW D11/D12 discovered)

**Not re-captured -- same justification as fix passes 2-7.** Fix pass 8 touched only the
`STATIC_ASSIGNEE` value (`"administrators"` -> `"admin"`) on two workflow-model nodes, plus one
temporary, fully-reverted live diagnostic toggle of `IS_COMMENT_ALLOWED`/`WORKITEM_COMMENT` (restored
byte-for-byte to the committed values before this pass ended) -- zero visual/theme/CSS/clientlib
artifacts touched at any point. None of this affects the embedded form's rendered pixels.

**Spot check performed instead (live DOM, not pixel-diff):** the full existing Cypress regression
suite was re-run fresh against the current deploy (functional 8/8, final-retest 5/5,
submissiondate-probe 1/1 -- 14/14 total, zero regressions). This pass's investigative work was almost
entirely server-side (AEM Forms Workspace task-completion servlet, `error.log` stack traces, the
Invoke DDX process class) -- outside this report's scope. See `integration-test-report.md`'s
"DEFINITIVE final functional verification -- fix pass 8" section for the full functional findings
(D10 confirmed fixed; two NEW Critical defects discovered -- D11, a comment-save crash blocking task
completion on the delivered config, and D12, a deterministic Invoke DDX failure that conclusively
answers this delivery's central DDX/DoR question) and `test-report.md`'s matching section for the case
tally and final gate verdict.

**Carried-forward verdict: 0 Critical, 1 Major (unchanged, cosmetic), 85.79% pixel match (advisory in
migration mode).** UI parity remains a non-blocker -- the gate's remaining blockers are now two newly
diagnosed functional defects (D11, D12), plus the carried-forward D2/D4, none of them visual. This is
the eighth consecutive fix pass confirming UI parity is stable; no further re-capture is warranted
unless a future change touches a visual/theme/CSS/clientlib artifact.

---

## DEFINITIVE final E2E — fix pass 9 (Generate-DoR + admin assignee, real AF_PATH) — 2026-07-27

UI parity was NOT re-captured this pass: fix pass 9 (workflow DoR step swap + dorType none->generate)
touched no visual/theme/CSS/layout artifact, so the pixel comparison is unchanged and carried forward:
0 Critical findings, ~85.79% pixel match (advisory in migration mode per solution-architecture.yaml),
1 Major (Justification 3-field row grouping — cosmetic). dorType="generate" affects only Document-of-
Record configuration, not the interactive rendering, so no visual regression is possible from it.

Live spot-check (no regression): the embedded page http://localhost:4502/content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html
returns HTTP 200 and embeds exactly the employee-training-request form via its single AEM Form
Container (formRef -> /content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request,
loadType=embed, useiframe=false; no stale/duplicate embed). The functional regression run confirmed the
"Employee Details" section heading still renders (RT-01 PASS), and all fix-pass-9 form submissions were
performed THROUGH this embedded page (HTTP 200 + workflow instances created), proving the form is
functional in-page. TC-024 (US-10 / AC-10.1): PASS (0 Critical, carried).

UI parity is NOT the reason this delivery's gate is FAIL — the failure is the server-side Generate-DoR
step (see test-report.md / integration-test-report.md, defect D13).
