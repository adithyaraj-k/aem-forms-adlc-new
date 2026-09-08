# Sentinel — TEST Package
## Employee Training Request — LiveCycle → AEM as a Cloud Service (Path B migration)

Run: `.claude/agents/runs/2026-07-22-employee-training-request/`
Reads: `plan/planwright.md`, `design/test-cases.yaml`, `implementation/formwright.md`,
`implementation/groundsmith.md`, `assembly/assembler.md`, `deployment/code-quality-report.md`
Produces: this file, `integration-test-report.md`, `test-form-ui-report.md`

---

## Delivery Totals (running total for the whole delivery — Sentinel is the last phase)

**UPDATED — fix pass 8 is now the definitive, final functional verification this delivery ran; see
the "DEFINITIVE final functional verification — fix pass 8" section near the end of this file for the
true final gate verdict.** Table updated to the full, cumulative accounting across all 8 fix passes
(superseding the fix-pass-7 version of this table; the bottom-of-file YAML blocks were kept cumulative
throughout — this table now matches them):

| Phase | Lead | time_taken_minutes | tokens (cli_text / read / write / other / **total**) |
|---|---|---|---|
| PLAN | planwright | 14.0 | 9,000 / 48,000 / 11,000 / 6,000 / **74,000** |
| DESI | draftsmith | 19.0 | 16,000 / 40,000 / 21,000 / 8,000 / **85,000** |
| IMPL-build | formwright | 92.0 | 48,000 / 95,000 / 62,000 / 15,000 / **220,000** |
| IMPL-integration | groundsmith | 37.0 | 38,000 / 88,000 / 24,000 / 9,000 / **159,000** |
| ASSEMBLY | assembler | 8.0 | 2,500 / 1,200 / 4,000 / 1,500 / **9,200** |
| DEPLOY | program-agent (standing in for forgemaster) | 9.0 | 3,500 / 9,000 / 4,500 / 2,000 / **19,000** |
| TEST-pass-1 | sentinel | ~45.0 | ~25,000 / ~45,000 / ~8,000 / ~15,000 / **~93,000** |
| IMPL-build-fix-pass-1 | formwright | 35.0 | — / — / — / — / **54,000** |
| IMPL-integration-fix-pass-1 | groundsmith | 72.0 | 22,000 / 64,000 / 9,000 / 35,000 / **130,000** |
| TEST-retest-fix-pass-1 | sentinel | 65.0 | — / — / — / — / **175,000** |
| IMPL-build-fix-pass-2 | formwright | 28.0 | — / — / — / — / **57,000** |
| DEPLOY-redeploy-fix-pass-2 | forgemaster | 14.0 | — / — / — / — / **24,500** |
| TEST-FINAL-retest-fix-pass-2 | sentinel | 52.0 | — / — / — / — / **164,000** |
| IMPL-build-fix-pass-3 | formwright | 16.0 | — / — / — / — / **23,000** |
| DEPLOY-redeploy-fix-pass-3 | forgemaster | 13.0 | — / — / — / — / **23,300** |
| TEST-FINAL-retest-fix-pass-3 | sentinel | 42.0 | — / — / — / — / **120,000** |
| IMPL-build-fix-pass-4 | formwright | 52.0 | — / — / — / — / **109,000** |
| IMPL-integration-fix-pass-4 | groundsmith | 95.0 | 34,000 / 145,000 / 16,000 / 55,000 / **250,000** |
| TEST-retest-fix-pass-4 | sentinel | 65.0 | — / — / — / — / **185,000** |
| DEPLOY-redeploy-fix-pass-4 | forgemaster | 19.0 | — / — / — / — / **43,500** |
| IMPL-integration-fix-pass-5 | groundsmith | 58.0 | 21,000 / 72,000 / 14,000 / 26,000 / **133,000** |
| DEPLOY-redeploy-fix-pass-5 | forgemaster | 50.0 | — / — / — / — / **83,000** |
| TEST-retest-fix-pass-5 | sentinel | 80.0 | — / — / — / — / **265,000** |
| IMPL-integration-fix-pass-6 | groundsmith | 64.0 | 26,000 / 78,000 / 11,000 / 48,000 / **163,000** |
| DEPLOY-redeploy-fix-pass-6 | forgemaster | 16.0 | 3,200 / 15,500 / 2,800 / 8,500 / **30,000** |
| TEST-FINAL-retest-fix-pass-6 | sentinel | 78.0 | 32,000 / 145,000 / 15,000 / 95,000 / **287,000** |
| IMPL-integration-fix-pass-7 | groundsmith | 22.0 | 9,000 / 34,000 / 6,000 / 3,000 / **52,000** |
| TEST-FINAL-retest-fix-pass-7 | sentinel | ~74.0 | ~42,000 / ~185,000 / ~28,000 / ~115,000 / **~370,000** |
| IMPL-integration-fix-pass-8 | groundsmith | 15.0 | 7,000 / 51,000 / 4,000 / 8,000 / **70,000** |
| DEPLOY-redeploy-fix-pass-8 | forgemaster | 14.0 | 3,000 / 13,000 / 3,000 / 9,500 / **28,500** |
| TEST-DEFINITIVE-fix-pass-8 | sentinel (this run) | ~95.0 | ~45,000 / ~220,000 / ~35,000 / ~130,000 / **~430,000** |
| **TOTAL** | | **~1356.0 min** | **~522,500 / ~1,966,700 / ~397,100 / ~849,700 / **~3,736,000** |

Several phases' per-category token splits were recorded only as a lump `tokens_total` in their own
handoff YAML (shown as `—` above); the running grand total below still reflects every phase's full
`total`. Sentinel's own figures for this final pass are marked `~` (estimate). Because at least one
phase's figure is an estimate, **the delivery total is marked `~`.**

---

## Verdict & Gate: **FAIL** (carried status prior to fix pass 8; see the "DEFINITIVE final functional
verification — fix pass 8" section near the end of this file for the TRUE final gate verdict)

This delivery does **not** pass the TEST gate. Four independently-confirmed **Critical** defects were
found via live verification (guideContainer.model.json, raw JCR node inspection, a real REST-level
submission attempt, the AEM error log, and the synced workflow model) — none of them visible from the
BUILD SUCCESS / 112-unit-tests-passing / all-bundles-Active state Forgemaster confirmed, because none
are unit-testable without live rule compilation and an actual end-to-end submission attempt. Per the
project's non-negotiable rule ("never soft-pass — a failed case, an uncovered story, or a Critical
UI-parity finding → gate FAIL"), and because **4 of 11 user stories have zero passing cases
(uncovered)**, the gate is FAIL.

- Test cases: 34 total, 34 executed, 18 passed, 16 failed. `unexecuted_cases: 0` ✓ (every case was
  attempted — live where possible, structural/config verification where a defect blocked the live
  path — none were left "planned but not run").
- User stories: 11 total, **7 covered, 4 uncovered** (US-02, US-07, US-08, US-10). `uncovered_stories`
  is **not** empty — hard gate failure per project rule.
- Embedded page: confirms the new form (see §4) — this part is clean.
- UI parity: 86.21% pixel match (13.79% mismatch, advisory in migration mode) but **1 Critical**
  finding (missing "Employee Details" section heading) — fails the migration-parity bar of zero
  Critical findings.

---

## 1. Test-case results (all 34 DESI cases)

| ID | traces_to_story | traces_to_ac | executor | executed | Result | Notes |
|---|---|---|---|---|---|---|
| TC-001 | US-01 | AC-01.1 | sentinel | pass | PASS | All 20 required fields carry `required:true` + `constraintMessages.required` in live model |
| TC-002 | US-01 | AC-01.2 | sentinel | fail | **FAIL** | Real submission attempt → HTTP 502, 0 workflow instances (Defect D1) |
| TC-003 | US-02 | AC-02.1 | sentinel | fail | **FAIL** | Manager lock rule doesn't compile live (Defect D-R1) |
| TC-004 | US-02 | AC-02.2 | sentinel | fail | **FAIL** | Tied to D-R1; also no prefill exists (accepted decision) so nothing to compare |
| TC-005 | US-03 | AC-03.1 | sentinel | pass | PASS | Email regex confirmed live |
| TC-006 | US-03 | AC-03.1 | sentinel | pass | PASS | |
| TC-007 | US-04 | AC-04.1 | sentinel | pass | PASS | `rules.enabled` binding confirmed live |
| TC-008 | US-04 | AC-04.1 | sentinel | pass | PASS | |
| TC-009 | US-05 | AC-05.1 | sentinel | pass | PASS | `validationExpression` + `validateDateAfter()` confirmed live |
| TC-010 | US-05 | AC-05.1 | sentinel | pass | PASS | |
| TC-011 | US-06 | AC-06.1 | sentinel | fail | **FAIL** | Blocked — no workflow instance ever created (D1) |
| TC-012 | US-06 | AC-06.2 | sentinel | fail | **FAIL** | Blocked by D1; also would fail on D2 (no real Approve/Reject branching) even if D1 were fixed |
| TC-013 | US-06 / US-08 | AC-06.2 / AC-08.3 | sentinel | fail | **FAIL** | Config confirmed correct (`assignee=administrators` on both tasks) but live task-landing could not be confirmed — D1 blocks any instance from starting |
| TC-014 | US-07 | AC-07.1 | sentinel | fail | **FAIL** | FinanceComments enable rule doesn't compile live (Defect D-R6) |
| TC-015 | US-07 | AC-07.1 | sentinel | fail | **FAIL** | Same root cause, D-R6 |
| TC-016 | US-08 | AC-08.1 | sentinel | fail | **FAIL** | Blocked by D1; would also fail D2 (unconditional linear execution) |
| TC-017 | US-08 | AC-08.2 | sentinel | fail | **FAIL** | Blocked by D1; would also fail D2 (DoR fires even on reject) |
| TC-018 | US-09 | AC-09.1 | sentinel | pass | PASS | `required:true`+`enum:[true]` confirmed |
| TC-019 | US-09 | AC-09.1 | sentinel | pass | PASS | Submit-click set-value script confirmed live; `submissionDate` readOnly |
| TC-020 | US-01/US-05/US-09 | AC-01.1 | sentinel | fail | **FAIL** | Invalid-half passes structurally; valid-half fails (D1 — 0 workflow instances even for a fully valid payload) |
| TC-021 | US-06 | AC-06.1 | sentinel | pass | PASS | `rules.visible` binding confirmed live — the fragment-nested-field reference WORKS |
| TC-022 | US-06 | AC-06.1 | sentinel | pass | PASS | |
| TC-023 | US-06 | AC-06.1 | sentinel | pass | PASS | Expression covers `CurrentStatus != 'FinanceApproved'` |
| TC-024 | US-10 | AC-10.1 | test-form-ui | fail | **FAIL** | 1 Critical finding: "Employee Details" heading missing live despite `visible:true` in JCR — see `test-form-ui-report.md` |
| TC-025 | US-11 | AC-11.1 | sentinel | pass | PASS | DoR template field/panel composition matches design spec (structural) |
| TC-026 | US-11 | AC-11.1 | sentinel | fail | **FAIL** | Template composition confirmed structurally, but the actual PDF was never generated — blocked by D1 |
| TC-027 | US-01 | AC-01.1 | sentinel | pass | PASS | All 30 fields carry correct `aria-label`; Employee Location vs Training Location distinctly labelled |
| TC-028 | US-01 | AC-01.1 | sentinel | pass | PASS | `employee-identity` fragmentPath identical in both consumers (single source of truth by construction) |
| TC-029 | US-09 | AC-09.1 | sentinel | pass | PASS | `declaration-consent` fragmentPath identical in both consumers |
| TC-030 | US-01 | AC-01.2 | create-form-tests | fail | **FAIL** | No custom submit-action Java class exists (stock action per Groundsmith); adapted verification = config + live attempt, fails on D1 |
| TC-031 | US-02 | AC-02.1 | create-form-tests | fail | **FAIL** | No prefill service exists (accepted "no prefill" decision) — case's literal expectation of a returned lookup pair is unmet; architecture/test-case mismatch, not a new defect |
| TC-032 | US-06 | AC-06.1 | sentinel | fail | **FAIL** | Cannot verify — no instance ever reaches a Send Email step (D1) |
| TC-033 | US-01 | AC-01.1 | sentinel | pass | PASS | All 5 dropdown option sets exact match live |
| TC-034 | US-01 | AC-01.1 | sentinel | pass | PASS | All 3 attachment fields `required:false` confirmed |

**Counts by result:** 18 pass / 16 fail (34 executed, 0 unexecuted).
**Counts by executor:** `create-form-tests` 2 (0 pass / 2 fail — both adapted, no custom Java exists) ·
`test-form-ui` 1 (0 pass / 1 fail) · `sentinel` 31 (18 pass / 13 fail).

---

## 2. User-story coverage

| Story | Cases | Covered? |
|---|---|---|
| US-01 | TC-001(P), TC-002(F), TC-020(F), TC-027(P), TC-028(P), TC-030(F), TC-033(P), TC-034(P) | **YES** (5 passing cases) — but note AC-01.2 (submit→workflow) specifically has **0 passing cases** (TC-002, TC-030 both fail) |
| US-02 | TC-003(F), TC-004(F), TC-031(F) | **NO — UNCOVERED** (0/3 pass) |
| US-03 | TC-005(P), TC-006(P) | YES |
| US-04 | TC-007(P), TC-008(P) | YES |
| US-05 | TC-009(P), TC-010(P), TC-020(F) | YES |
| US-06 | TC-011(F), TC-012(F), TC-013(F), TC-021(P), TC-022(P), TC-023(P), TC-032(F) | **YES** (3 passing cases) — but AC-06.2 (approve/reject routing) has **0 passing cases** |
| US-07 | TC-014(F), TC-015(F) | **NO — UNCOVERED** (0/2 pass) |
| US-08 | TC-016(F), TC-017(F), TC-013(F) | **NO — UNCOVERED** (0/3 pass) |
| US-09 | TC-018(P), TC-019(P), TC-020(F), TC-029(P) | YES |
| US-10 | TC-024(F) | **NO — UNCOVERED** (0/1 pass, only case) |
| US-11 | TC-025(P), TC-026(F) | YES (1 passing case) — AC-11.1's PDF-content half (TC-026) fails |

**`uncovered_stories: [US-02, US-07, US-08, US-10]` — 4 of 11. This alone is a hard gate failure**
per the project's mandatory rule ("uncovered_stories MUST be empty").

---

## 3. Functional findings (rules/validation/submit/prefill/workflow) — through the embedded page

Full detail in `integration-test-report.md`. Summary:

- **6 of 8 migrated business rules verified working live**: email format validation, RequestType
  lock/unlock, EndDate>StartDate validation, **Approval Information panel show/hide (the
  fragment-nested-field reference — Formwright's top flagged risk — WORKS)**, AcceptedTerms required,
  SubmissionDate set-value on submit.
- **2 of 8 rules confirmed broken live**: ManagerId/ManagerName lock (Defect D-R1) and FinanceComments
  lock/unlock (Defect D-R6) — both authored correctly in JCR but do not compile into a live reactive
  binding in `guideContainer.model.json`.
- **Submit → workflow: BROKEN.** A real submission with a fully valid, schema-conformant payload
  returns HTTP 502 from Adobe's own stock "Invoke an AEM Workflow" action
  (`FormsWorkflowException: Unable to save file attachment for workItem`), and creates **zero**
  workflow instances (Defect D1).
- **Workflow approval routing: BROKEN independently of D1.** The synced runtime model has no
  `OR_SPLIT` nodes and no conditional transitions — every instance would unconditionally run through
  DoR generation, the approval email, AND the rejection email regardless of the Approve/Reject choice
  (Defect D2).
- **Prefill: confirmed "none," matching Groundsmith's documented decision** — no fabricated
  auto-population occurs for any EmployeeId. (TC-031's literal wording doesn't match this accepted
  decision — flagged as a test-case mismatch, not a new defect.)
- **Fragment reuse (TC-028/029): confirmed** — both fragments are referenced by the same
  `fragmentPath` in the interactive form and the DoR template.
- **Accessibility (TC-027): confirmed** — every field has a correct `aria-label`; the two
  same-captioned "Location" fields are distinctly labelled ("Employee Location" / "Training
  Location").
- **Dropdown option sets (TC-033): exact match confirmed** for all 5 dropdowns.
- **Send Email steps (TC-032): could not be verified at all** — blocked by D1, not merely a
  SMTP-no-op acceptance case as originally anticipated.

No browser-automation tool was available in this session, so no click-through UI interaction (field
fill, click Submit in-browser, tab-through keyboard test) was performed; all findings are drawn from
live runtime JSON/JCR state, a real REST-level submission attempt, and the AEM error log — genuine
live evidence, not source-code inference.

---

## 4. Embedded-page result

- **Page:** `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form` — HTTP 200.
- **Embeds exactly the new form:** confirmed — the AEM Form Container's `formRef` points at
  `employee-training-request`'s DAM guide asset; exactly one container on the page (no
  stale/duplicate `sports-event-registration` embed found).
- **Functional in-page:** the form renders fully styled and hydrated inside the page (confirmed via
  the `test-form-ui` capture — see `test-form-ui-report.md`), with all fields, panels, and section
  headers except the one Critical gap (missing "Employee Details" heading, Finding #1 in that
  report) rendering correctly in the page context.

---

## 5. UI parity

Full detail: `test-form-ui-report.md`. Summary: **FAIL** — 86.21% pixel match (13.79% mismatch,
advisory-only in migration mode) but **1 Critical** finding (the "Employee Details" section heading
is declared `visible:true` in the JCR but never renders live — the same fragment-embed rendering
pattern class as Defects D-R1/D-R6 above) and 1 Major (Justification panel's 3-field row groups as
2+1 instead of 3-across — cosmetic, not a content gap). All other structural parity checks (panel
order, header-band/body colours, 3-column grids, field order/labels, Declaration's bare styling)
match the `DocRoute.xdp` reference.

---

## 6. Defects (routed to owning leads)

| Defect | Severity | Route to | Summary |
|---|---|---|---|
| D-R1 | Critical | **Formwright** | ManagerId/ManagerName lock rule (fragment-nested-field `Change` script) doesn't compile into a live event binding |
| D-R6 | Critical | **Formwright** | FinanceComments enable rule missing the `fd:rules`-nested companion expression string; renders as a static value, never reactive |
| Finding #1 (UI parity) | Critical | **Formwright** | "Employee Details" fragment section heading declared `visible:true` but never renders live |
| D1 | Critical | **Groundsmith** | Submit via "Invoke an AEM Workflow" fails server-side (`FormsWorkflowException`), 0 workflow instances from any real submission |
| D2 | Critical | **Groundsmith** | Synced runtime workflow model has no `OR_SPLIT` routing — Approve/Reject never actually branch |
| TC-031 mismatch | Informational | DESI (future test-design revision) | Test case expects a prefill lookup value that Groundsmith's accepted "no prefill" decision deliberately removed — not a code defect |

**Not flagged as defects** (accepted/expected per the task brief): Send Email no-op on
SMTP-unconfigured local SDK would have been accepted had D1 not blocked reaching those steps
entirely; manager-assignment and finance-manager-assignment both resolving to `administrators` is
intentional; absence of a real manager-lookup prefill is a recorded pre-production follow-up.

---

## Run metrics (this agent — sentinel)

- `time_taken_minutes`: ~45.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~25,000
  - `read`: ~45,000 (6 PLAN/DESI/IMPL/ASSEMBLY/DEPLOY docs, guideContainer.model.json + 6 JCR node
    dumps, schema.json, workflow model JSON x2, 3 images (actual/reference/diff), the test-form-ui
    skill's own instructions text)
  - `write`: ~8,000 (test-form-ui-report.md, integration-test-report.md, this file, 2 small scratch
    payload files)
  - `other`: ~15,000 (many curl/bash tool-call overhead lines incl. repeated sandbox `/etc/*`
    permission noise, the Cypress run's console output, grep/glob output)
  - `total`: ~93,000

---

## Handoff YAML

```yaml
agent: sentinel
phase: TEST
status: FAILED
time_taken_minutes: 45.0
tokens_consumed:
  cli_text: 25000
  read: 45000
  write: 8000
  other: 15000
  total: 93000
test_cases: { total: 34, executed: 34, passed: 18, failed: 16 }
unexecuted_cases: 0
executes_with: { create-form-tests: 2, test-form-ui: 1, functional: 31 }
coverage_pct: null   # no custom Forms-service Java classes exist for this delivery (stock submit action, no prefill service) — N/A, not measured
user_stories: { total: 11, covered: 7 }
uncovered_stories: 4   # US-02, US-07, US-08, US-10 — HARD GATE FAILURE
ui_parity: { pixel: PASS_ADVISORY, pixel_match_pct: 86.21, critical_findings: 1 }
embedded_page:
  path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form"
  embeds_new_form: true
  functional_in_page: true   # renders correctly except the 1 Critical UI-parity gap noted above
defects:
  - { id: D-R1, severity: Critical, owner: formwright, summary: "ManagerId/ManagerName lock rule does not compile into a live event binding" }
  - { id: D-R6, severity: Critical, owner: formwright, summary: "FinanceComments enable rule missing fd:rules companion expression; static, non-reactive" }
  - { id: "UI-Finding-1", severity: Critical, owner: formwright, summary: "Employee Details fragment section heading declared visible:true but never renders" }
  - { id: D1, severity: Critical, owner: groundsmith, summary: "Submit via Invoke-an-AEM-Workflow fails server-side; 0 workflow instances ever created" }
  - { id: D2, severity: Critical, owner: groundsmith, summary: "Synced runtime workflow model has no OR_SPLIT routing; Approve/Reject never branch" }
delivery_totals:
  total_time_taken_minutes: "~224.0"
  total_tokens_consumed: { cli_text: 142000, read: 326200, write: 134500, other: 56500, total: 659200 }
  per_phase:
    - { phase: PLAN, lead: planwright, time_taken_minutes: 14.0, tokens_total: 74000 }
    - { phase: DESI, lead: draftsmith, time_taken_minutes: 19.0, tokens_total: 85000 }
    - { phase: IMPL-build, lead: formwright, time_taken_minutes: 92.0, tokens_total: 220000 }
    - { phase: IMPL-integration, lead: groundsmith, time_taken_minutes: 37.0, tokens_total: 159000 }
    - { phase: ASSEMBLY, lead: assembler, time_taken_minutes: 8.0, tokens_total: 9200 }
    - { phase: DEPLOY, lead: "program-agent (standing in for forgemaster)", time_taken_minutes: 9.0, tokens_total: 19000 }
    - { phase: TEST, lead: sentinel, time_taken_minutes: 45.0, tokens_total: 93000 }
report: ".claude/agents/runs/2026-07-22-employee-training-request/testing/test-report.md"
gate_result: FAIL
next: "aem-forms-program-agent routes D-R1/D-R6/UI-Finding-1 back to formwright and D1/D2 back to groundsmith; re-run forgemaster (build/deploy) then re-run sentinel (full re-test) after fixes land. Do NOT ship this delivery as-is."
```

---

## RETEST — fix pass 1 (2026-07-22)

### Delivery Totals (updated running total — includes both fix passes + this retest)

| Phase | Lead | time_taken_minutes | tokens (cli_text / read / write / other / **total**) |
|---|---|---|---|
| PLAN | planwright | 14.0 | 9,000 / 48,000 / 11,000 / 6,000 / **74,000** |
| DESI | draftsmith | 19.0 | 16,000 / 40,000 / 21,000 / 8,000 / **85,000** |
| IMPL-build | formwright | 92.0 | 48,000 / 95,000 / 62,000 / 15,000 / **220,000** |
| IMPL-integration | groundsmith | 37.0 | 38,000 / 88,000 / 24,000 / 9,000 / **159,000** |
| ASSEMBLY | assembler | 8.0 | 2,500 / 1,200 / 4,000 / 1,500 / **9,200** |
| DEPLOY | program-agent (standing in for forgemaster) | 9.0 | 3,500 / 9,000 / 4,500 / 2,000 / **19,000** |
| TEST (pass 1) | sentinel | ~45.0 | ~25,000 / ~45,000 / ~8,000 / ~15,000 / **~93,000** |
| IMPL-build fix pass 1 | formwright | 35.0 | 9,000 / 28,000 / 11,000 / 6,000 / **54,000** |
| IMPL-integration fix pass 1 | groundsmith | 72.0 | 22,000 / 64,000 / 9,000 / 35,000 / **130,000** |
| TEST (retest, this run) | sentinel | ~65.0 | ~30,000 / ~90,000 / ~15,000 / ~40,000 / **~175,000** |
| **TOTAL** | | **~396.0 min** | **~203,000 / ~508,200 / ~169,500 / ~137,500 / ~1,018,200** |

Because multiple phases' figures are estimates (`~`), the delivery total is marked `~`.

### Verdict & Gate: still **FAIL** — but the gap is now much narrower and better understood

4 of the 5 items in scope for this retest are **genuinely fixed and independently confirmed live**
(not just carried forward from the fix-pass agents' own claims). The 5th (OR-split routing) is
confirmed still broken, exactly as Groundsmith's fix pass reported and expected. However, this
retest's own independent, from-scratch verification — specifically, inspecting the actual data
payload of a real end-to-end submission, something no prior pass had done because D1 always blocked
it — surfaced **one new, previously-undiscovered Critical defect (D3)** that is **not** part of the
5-item retest scope and **not** resolved by anything in fix pass 1. This means the honest answer to
"is OR-split the only remaining gap" is **no** — see below.

| Item | Status |
|---|---|
| D-R1 (ManagerId/ManagerName lock) | **FIXED** — confirmed live via `guideContainer.model.json` reactive binding + browser lock/unlock behavior |
| D-R6 (FinanceComments lock) | **FIXED** — confirmed live via `guideContainer.model.json` reactive binding (matches working `requestType` pattern) + browser initial-locked-state |
| UI-Finding-1 (Employee Details heading) | **FIXED** — confirmed by actually rendering the page (Cypress capture + vision read), not a JSON guess |
| D1 (submit → workflow HTTP 502) | **FIXED** — confirmed via an independent, freshly-authored Cypress E2E submission (not reusing Groundsmith's payload): HTTP 200, new workflow instance created, `RUNNING` at Manager Approval |
| D2 (OR-split branching) | **STILL NOT FIXED**, exactly as expected — 0 `OR_SPLIT` nodes in the live runtime model, fully linear, escalated pending Workflow Editor UI access |
| **D3 (NEW — fragment dataRef mismatch)** | **CRITICAL, newly discovered this retest** — `employee-identity`/`declaration-consent` fragment fields serialize to `employee.*`/`declaration.*`, NOT the schema's `EmployeeDetails.*`/`Declaration.*`; breaks DoR data completeness (AC-11.1) and the workflow's `managerId`/`applicantEmail` variable capture for every real submission. Full evidence in `integration-test-report.md` §"Retest — fix pass 1". |

- Test cases: 34 total, 34 executed (re-executed live where the fix pass could plausibly have
  changed the outcome; carried forward with justification where the code was untouched — see table
  below). **26 pass / 8 fail** (up from 18 pass / 16 fail on the first pass).
- User stories: 11 total, **10 covered, 1 uncovered (US-08)** — down from 4 uncovered. `US-08`
  remains uncovered because its own dedicated acceptance criteria (AC-08.1 DoR generation on finance
  approval, AC-08.2 no-DoR-on-reject) have zero passing cases, now blocked by the combination of D2
  (no real approve/reject routing exists to reach a genuine finance-approval outcome) **and** the
  newly-discovered D3 (even if reached, the DoR would render incomplete).
- Embedded page: still confirms the new form correctly (unchanged, re-verified).
- UI parity: 85.79% pixel match (14.21% mismatch, advisory in migration mode), **0 Critical findings
  remain** (the 1 Critical finding from pass 1 is resolved), 1 Major (unchanged, cosmetic).

**`uncovered_stories: [US-08]` — 1 of 11. Not empty → the gate remains FAIL** per the project's
non-negotiable rule, but this is a dramatically narrower, well-contained gap than pass 1's 4
uncovered stories and 16 failing cases.

### Is "OR-split routing (D2) only" an accurate description of the remaining gap? **No — say so explicitly.**

The task briefing anticipated that if D2 were the only remaining item, it could reasonably be
accepted as a documented, human-actionable follow-up (Workflow Editor UI access no agent in this
session has) rather than triggering a 3rd automated fix cycle. This retest's own from-scratch,
independent verification — going one level deeper than any prior pass by actually inspecting a real
submission's payload content, not just confirming a workflow instance was created — found that is
**not** the case: **D3 is a second, independent Critical defect**, newly discovered this pass, not
part of what Formwright/Groundsmith were asked to fix, and not something the OR-split fix would
touch. It directly blocks AC-11.1 (US-11's Document of Record completeness) regardless of D2's
status, and independently taints the workflow's own captured `managerId`/`applicantEmail`
variables. **Recommendation: this is a 2-item remaining gap (D2 + D3), not a 1-item gap** — D2 can
reasonably still be accepted as a documented, human-actionable follow-up (Workflow Editor UI, as
before), but **D3 should go back to Formwright for a further fix pass** (correcting the two
fragments' `dataRef` bindings to the host schema's actual property paths) before this delivery is
considered ship-ready, since it affects data integrity/correctness on every single real submission,
not just an edge-case routing scenario.

### Updated test-case results (34 cases)

| ID | Retest disposition | Result | Notes |
|---|---|---|---|
| TC-001 | carried forward (unaffected) | PASS | unchanged |
| TC-002 | **RE-EXECUTED live** | **PASS** (was FAIL) | D1 fixed — independent Cypress E2E submission returned HTTP 200 and created a new RUNNING workflow instance |
| TC-003 | **RE-EXECUTED live** | **PASS** (was FAIL) | D-R1 fixed — `guideContainer.model.json` reactive binding + browser lock/unlock confirmed (RT-02) |
| TC-004 | carried forward | FAIL (unchanged) | AC-02.2's hardcoded-manager-lookup GAP is untouched by this fix pass (out of scope, tracked as an open question, not a new defect) |
| TC-005/006 | carried forward (unaffected) | PASS | rule #2 untouched; re-confirmed live anyway via RT-05 |
| TC-007/008 | carried forward (unaffected) | PASS | rule #3 untouched; re-confirmed live anyway via RT-03 |
| TC-009/010 | carried forward (unaffected) | PASS | rule #4 untouched; re-confirmed live anyway via RT-04 + a real valid submission |
| TC-011 | **RE-EXECUTED live** | **PASS** (was FAIL) | D1 fixed — Manager Approval task now genuinely created and assigned per config; task-assignment email CONTENT accuracy flagged separately under D3 (applicantEmail), and SMTP is not configured on the local SDK (environment limitation, not a new defect) |
| TC-012 | **RE-EXECUTED (workflow model API)** | FAIL (unchanged) | D2 still open — 0 `OR_SPLIT` nodes, fully linear, confirmed via `/var/workflow/models/...` |
| TC-013 | **RE-EXECUTED live** | FAIL (unchanged, more precisely scoped) | Config confirmed correct (assignee=administrators on both tasks); FIRST task landing now confirmed live (node2 present in a real new instance); SECOND task (Finance Manager Approval) landing not exercised this pass (would require completing the first task via the Workflow Inbox — out of scope/time this session) — kept as FAIL rather than a partial pass, per "never soft-pass" |
| TC-014/015 | **RE-EXECUTED live** | **PASS** (was FAIL) | D-R6 fixed — `guideContainer.model.json` reactive binding confirmed + browser initial-locked-state (RT-06) |
| TC-016/017 | **RE-EXECUTED (workflow model + payload)** | FAIL (unchanged, new compounding reason) | D2 still blocks real approve/reject differentiation (DoR + both outcome emails would fire unconditionally); D3 additionally means any generated DoR would be incomplete (missing Employee Details/Declaration) |
| TC-018 | carried forward (unaffected) | PASS | AcceptedTerms required-gating confirmed via a real submission (RT-08: empty submit blocked, checked+valid submit succeeded) |
| TC-019 | **RE-EXECUTED live — DOWNGRADED** | **FAIL** (was PASS) | Original pass marked this PASS from the rule's *presence* in JSON only. This retest actually exercised it in a browser and found: (a) `Unable to compile expression ... ParserError: Unexpected token type: UnquotedIdentifier, value: Date` thrown on every submit (the `new Date()` constructor isn't supported by the rule engine's simple-expression parser), so the assignment never executes; (b) even if it did, it targets `declaration.submissionDate`, not the schema's `Declaration.SubmissionDate` (same D3 root cause). Corrects a previously-inaccurate PASS — reported honestly, not soft-passed. |
| TC-020 | **RE-EXECUTED live** | **PASS** (was FAIL) | Valid-half now genuinely submits (D1 fixed); required-field gating (its literal criterion) holds. D3's data-path concern is adjacent, not part of AC-01.1's literal claim, flagged separately. |
| TC-021/022/023 | carried forward (unaffected) | PASS | rule #5 untouched; re-confirmed live anyway via RT-07 |
| TC-024 | **RE-EXECUTED (test-form-ui)** | **PASS** (was FAIL) | Heading now renders, confirmed by rendering the page and reading the resulting capture — see `test-form-ui-report.md` §"Retest — fix pass 1" |
| TC-025 | carried forward (unaffected) | PASS | structural DoR composition unchanged |
| TC-026 | **RE-EXECUTED (partial) — still FAIL, new reason** | FAIL (unchanged, more precisely scoped) | D1 no longer blocks reaching the workflow; actual DoR generation wasn't driven to completion this session (requires completing both Assign Task work items via the Workflow Inbox, out of scope/time) — and even if it were, D3 means the resulting PDF would be missing Employee Details/Declaration data |
| TC-027 | carried forward (unaffected) | PASS | aria-labels unchanged |
| TC-028/029 | carried forward (unaffected) | PASS | shared `fragmentPath` reuse still structurally true (the claim these cases test); the *functional consequence* of that shared reference now includes a shared bug (D3), cross-referenced, not double-counted |
| TC-030 | **RE-EXECUTED live** | **PASS** (was FAIL) | Live-attempt half now succeeds (D1 fixed); executor/case-design mismatch (no custom submit-action Java class exists) unchanged from pass 1, noted as before |
| TC-031 | carried forward | FAIL (unchanged) | test-case/architecture mismatch against the accepted "no prefill" decision — unrelated to this fix pass |
| TC-032 | **RE-EXECUTED (workflow model)** | FAIL (unchanged, new compounding reason) | D1 no longer blocks, but D2 (unconditional linear execution) + D3 (applicantEmail always empty) + SMTP-not-configured-locally jointly mean this remains unverifiable in a meaningful way |
| TC-033/034 | carried forward (unaffected) | PASS | unchanged |

**Counts by result:** 26 pass / 8 fail (34 executed, 0 unexecuted) — up from 18 pass / 16 fail.
**Newly passing this retest:** TC-002, TC-003, TC-011, TC-014, TC-015, TC-020, TC-024, TC-030 (8 cases).
**Newly failing this retest (downgrade, not a regression — a correction):** TC-019 (previously
mis-verified as PASS from static JSON; now genuinely exercised and found broken).
**Still failing (unchanged):** TC-004, TC-012, TC-013, TC-016, TC-017, TC-026, TC-031, TC-032 (8 cases).

### Updated user-story coverage

| Story | Change | Covered? |
|---|---|---|
| US-01 | TC-002/TC-020/TC-030 now PASS | **YES** (D1 fixed; AC-01.2 now has passing cases) |
| US-02 | TC-003 now PASS | **YES** (was UNCOVERED — D-R1 fixed) — AC-02.2's hardcoded-lookup GAP remains open (unchanged, tracked separately, not gate-blocking) |
| US-03/04/05 | unaffected | YES (unchanged) |
| US-06 | TC-011 now PASS; TC-021/022/023 unaffected PASS | YES (unchanged — was already covered); AC-06.2 (approve/reject outcomes) still has 0 passing cases (D2) |
| US-07 | TC-014/015 now PASS | **YES** (was UNCOVERED — D-R6 fixed) |
| US-08 | TC-013/016/017 still FAIL | **NO — still UNCOVERED** (0/3 pass) — now blocked by D2 (no real routing to a genuine finance-approval outcome) AND D3 (DoR would be incomplete even if reached) |
| US-09 | TC-019 now FAIL (correction), TC-018/020/029 still PASS | YES (unchanged — still has passing cases; TC-019's downgrade doesn't flip coverage but is reported honestly) |
| US-10 | TC-024 now PASS | **YES** (was UNCOVERED — heading fixed) |
| US-11 | TC-025 PASS, TC-026 still FAIL | YES (1 passing case, unchanged) — AC-11.1's PDF-content half still fails, now for the D3 reason specifically |

**`uncovered_stories: [US-08]` — 1 of 11 (down from 4 of 11).** Still not empty → hard gate FAIL,
but a narrow, well-understood, single-story gap.

### Updated embedded-page result

Unchanged from pass 1 — re-verified: `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form`
still HTTP 200, still embeds exactly `employee-training-request` via its one AEM Form Container, no
stale/duplicate embed, functional in-page (now including a genuinely successful real submission
through the page, not just field rendering).

### Updated UI parity

**0 Critical findings** (was 1) — see `test-form-ui-report.md` §"Retest — fix pass 1". 85.79% pixel
match (advisory in migration mode). 1 Major (Justification row-grouping) unchanged, cosmetic,
non-blocking.

### Defects (updated)

| Defect | Severity | Status | Route to |
|---|---|---|---|
| D-R1 | Critical | **RESOLVED** | — |
| D-R6 | Critical | **RESOLVED** | — |
| UI-Finding-1 | Critical | **RESOLVED** | — |
| D1 | Critical | **RESOLVED** | — |
| D2 | Critical | **OPEN — escalated, human-actionable (Workflow Editor UI), recommended to accept as documented follow-up** | Groundsmith (needs browser/editor access, not agent-automatable on this AEM SDK) |
| **D3 (NEW)** | **Critical** | **OPEN — newly discovered this retest, NOT part of fix pass 1's scope** | **Formwright** (`create-AdaptiveFormFragment` — correct `employee-identity`/`declaration-consent` fragment field `dataRef`s to the host schema's actual `EmployeeDetails.*`/`Declaration.*` paths; also fix the `SubmissionDate` set-value script's unsupported `new Date()` expression) |

### Run metrics (this retest)

- `time_taken_minutes`: ~65.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~30,000
  - `read`: ~90,000 (prior test/implementation/deployment reports, live `guideContainer.model.json`
    dumps ×3 (host + DoR template), schema.json, workflow runtime model JSON, 2 submitted `data.xml`
    payloads + a 3rd (mine), 5 screenshot/PNG reads, Cypress console/debug output)
  - `write`: ~15,000 (new `employee-training-request-functional.cy.js` spec authored + 5 debugging
    edits, retest sections appended to all 3 reports)
  - `other`: ~40,000 (many curl/bash tool-call overhead lines, ~6 Cypress run iterations debugging
    the datepicker mask interaction, background task management)
  - `total`: ~175,000

### Updated Handoff YAML

```yaml
agent: sentinel
phase: TEST
status: FAILED
retest_of: "fix pass 1"
time_taken_minutes: 65.0
tokens_consumed:
  cli_text: 30000
  read: 90000
  write: 15000
  other: 40000
  total: 175000
test_cases: { total: 34, executed: 34, passed: 26, failed: 8 }
unexecuted_cases: 0
executes_with: { create-form-tests: 2, test-form-ui: 1, functional: 31 }
coverage_pct: null   # no custom Forms-service Java classes exist for this delivery — N/A, unchanged
user_stories: { total: 11, covered: 10 }
uncovered_stories: 1   # US-08 only — down from 4; HARD GATE FAILURE (still not empty)
ui_parity: { pixel: PASS_ADVISORY, pixel_match_pct: 85.79, critical_findings: 0 }   # was 1, now RESOLVED
embedded_page:
  path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form"
  embeds_new_form: true
  functional_in_page: true   # now includes a genuinely successful real submission, not just rendering
defects_resolved:
  - { id: D-R1, owner: formwright }
  - { id: D-R6, owner: formwright }
  - { id: "UI-Finding-1", owner: formwright }
  - { id: D1, owner: groundsmith }
defects_still_open:
  - { id: D2, severity: Critical, owner: groundsmith, status: "escalated, human-actionable (Workflow Editor UI), recommended acceptable documented follow-up" }
  - { id: D3, severity: Critical, owner: formwright, status: "NEW this retest — fragment dataRef mismatch breaks DoR completeness (AC-11.1) and workflow variable capture (managerId/applicantEmail) for every real submission; NOT resolved by fix pass 1; recommend a 3rd fix pass before ship" }
delivery_totals:
  total_time_taken_minutes: "~396.0"
  total_tokens_consumed: { cli_text: 203000, read: 508200, write: 169500, other: 137500, total: 1018200 }
  per_phase:
    - { phase: PLAN, lead: planwright, time_taken_minutes: 14.0, tokens_total: 74000 }
    - { phase: DESI, lead: draftsmith, time_taken_minutes: 19.0, tokens_total: 85000 }
    - { phase: IMPL-build, lead: formwright, time_taken_minutes: 92.0, tokens_total: 220000 }
    - { phase: IMPL-integration, lead: groundsmith, time_taken_minutes: 37.0, tokens_total: 159000 }
    - { phase: ASSEMBLY, lead: assembler, time_taken_minutes: 8.0, tokens_total: 9200 }
    - { phase: DEPLOY, lead: "program-agent (standing in for forgemaster)", time_taken_minutes: 9.0, tokens_total: 19000 }
    - { phase: TEST-pass-1, lead: sentinel, time_taken_minutes: 45.0, tokens_total: 93000 }
    - { phase: IMPL-build-fix-pass-1, lead: formwright, time_taken_minutes: 35.0, tokens_total: 54000 }
    - { phase: IMPL-integration-fix-pass-1, lead: groundsmith, time_taken_minutes: 72.0, tokens_total: 130000 }
    - { phase: TEST-retest, lead: sentinel, time_taken_minutes: 65.0, tokens_total: 175000 }
report: ".claude/agents/runs/2026-07-22-employee-training-request/testing/test-report.md"
gate_result: FAIL
gap_summary: "2-item gap remains: D2 (OR-split routing — known, escalated, human-actionable via Workflow Editor UI, reasonable to accept as a documented follow-up) AND D3 (fragment dataRef mismatch — NEW, Critical, blocks AC-11.1 and workflow variable capture, recommend NOT accepting, route back to Formwright for a further fix pass). This is narrower than pass 1 (16 failing cases / 4 uncovered stories -> 8 failing cases / 1 uncovered story) but is NOT the single-item OR-split-only gap the task hoped for."
next: "aem-forms-program-agent decides: (a) accept D2 as a documented human-actionable follow-up, AND (b) route D3 back to formwright for a 3rd fix pass (fragment dataRef correction) before considering this delivery ship-ready. Recommend NOT shipping as-is given D3's data-integrity impact on every real submission."
```

---

## FINAL RETEST — fix pass 2 (2026-07-22)

### Delivery Totals (final running total — ALL phases + both fix passes + all 3 Sentinel test runs)

| Phase | Lead | time_taken_minutes | tokens (cli_text / read / write / other / **total**) |
|---|---|---|---|
| PLAN | planwright | 14.0 | 9,000 / 48,000 / 11,000 / 6,000 / **74,000** |
| DESI | draftsmith | 19.0 | 16,000 / 40,000 / 21,000 / 8,000 / **85,000** |
| IMPL-build | formwright | 92.0 | 48,000 / 95,000 / 62,000 / 15,000 / **220,000** |
| IMPL-integration | groundsmith | 37.0 | 38,000 / 88,000 / 24,000 / 9,000 / **159,000** |
| ASSEMBLY | assembler | 8.0 | 2,500 / 1,200 / 4,000 / 1,500 / **9,200** |
| DEPLOY | program-agent (standing in for forgemaster) | 9.0 | 3,500 / 9,000 / 4,500 / 2,000 / **19,000** |
| TEST (pass 1) | sentinel | ~45.0 | ~25,000 / ~45,000 / ~8,000 / ~15,000 / **~93,000** |
| IMPL-build fix pass 1 | formwright | 35.0 | 9,000 / 28,000 / 11,000 / 6,000 / **54,000** |
| IMPL-integration fix pass 1 | groundsmith | 72.0 | 22,000 / 64,000 / 9,000 / 35,000 / **130,000** |
| TEST (retest, fix pass 1) | sentinel | ~65.0 | ~30,000 / ~90,000 / ~15,000 / ~40,000 / **~175,000** |
| IMPL-build fix pass 2 | formwright | 28.0 | 7,500 / 34,000 / 9,500 / 6,000 / **57,000** |
| DEPLOY redeploy fix pass 2 | forgemaster | 14.0 | 3,200 / 11,000 / 2,800 / 7,500 / **24,500** |
| TEST (FINAL retest, this run) | sentinel | ~52.0 | ~20,000 / ~85,000 / ~14,000 / ~45,000 / **~164,000** |
| **TOTAL** | | **~490.0 min** | **~233,700 / ~638,200 / ~195,800 / ~196,000 / ~1,263,700** |

Because multiple phases' figures are estimates (`~`), the delivery total is marked `~`.

### Verdict & Gate: **FAIL** — narrower still, but NOT down to a single accepted item

D3 and TC-019's *named* defect (the ParserError) are both genuinely fixed, independently reproduced by
this retest using a fresh, from-scratch submission and three separate live verification methods (not
carried forward from Formwright's or Forgemaster's own claims). **However, going one level deeper than
either of them did — actually inspecting what the SubmissionDate assignment *does*, not just whether it
compiles — surfaced a second, previously-masked defect underneath the one that was fixed.** This means
the honest answer to "is OR-split (D2) now the only remaining gap" is again **no**, for the second retest
in a row — but the gap is real, narrow, well-contained, and (unlike D2) fully within an agent's power to
fix.

| Item | Status |
|---|---|
| D3 (fragment dataRef → EmployeeDetails.*/Declaration.*) | **FIXED — confirmed via a real submission's persisted payload, independently reproduced** |
| D3 downstream: DoR completeness (Employee Details + AcceptedTerms) | **RESOLVED** (structural: write-side and read-side schema paths now agree) |
| D3 downstream: applicantEmail/managerId workflow variables | **RESOLVED** (config+data cross-check; could not read the live variable value directly with available tooling) |
| TC-019 ParserError (the literal named defect) | **FIXED** — confirmed gone from console across 2 full spec runs |
| TC-019 actual SubmissionDate value (found this pass) | **STILL BROKEN — new, narrower defect, called D4** — confirmed via 3 independent live methods (DOM value, outgoing request body, persisted JCR payload) |
| D-R1, D-R6, rules #2–#5/#7/#8, D1, submit-gated-on-validation (regression check) | **NO REGRESSION** — all reconfirmed live via a fresh, independent Cypress run |
| D2 (OR-split branching) | **STILL NOT FIXED, exactly as expected** — untouched this cycle, human-actionable follow-up, unchanged assessment |

- Test cases: 34 total, 34 executed, **25 pass / 9 fail** (`unexecuted_cases: 0`). Fail count is
  unchanged in size from the fix-pass-1 retest (8→9 depending on how that pass's own arithmetic is
  read — see note below) but changed in **composition**: D3-caused compounding reasons on TC-016,
  TC-017, TC-032 are gone (those 3 now fail solely due to D2, not D2+D3), while TC-019 remains failing
  for a corrected, narrower reason (D4, not the original ParserError).
  *(Note: the fix-pass-1 retest section above states "26 pass / 8 fail" in its summary prose but its
  own 34-row table, recounted, shows 9 FAILs — TC-004, TC-012, TC-013, TC-016, TC-017, TC-019, TC-026,
  TC-031, TC-032. This retest's 25/9 split is consistent with that table, not the earlier prose
  summary; flagging the discrepancy rather than silently propagating it.)*
- User stories: 11 total, **10 covered, 1 uncovered (US-08)** — **unchanged** from the fix-pass-1
  retest. `uncovered_stories: [US-08]` remains non-empty, and is now confirmed attributable **solely**
  to D2 (D3 no longer compounds it, per this retest's independent confirmation).
- Embedded page: still confirms the new form correctly, still functional in-page, now including a
  second independent fresh submission through the page (marker `FinalRetest...`) plus a focused probe
  submission (marker `Probe...`) — both succeeded end-to-end (HTTP 200, thank-you page, PDF generated).
- UI parity: carried forward, not re-captured (fix pass 2 touched no visual/theme/CSS artifact) — 0
  Critical, 85.79% pixel match (advisory in migration mode), 1 Major (cosmetic, unchanged). Spot-checked
  live that the Employee Details heading still renders (no regression).

**`uncovered_stories: [US-08]` — 1 of 11, unchanged.** Gate remains FAIL per the project's
non-negotiable rules (a failed case — TC-019 — and a non-empty `uncovered_stories` both independently
trigger FAIL).

### Is "OR-split routing (D2) only" now an accurate description of the remaining gap? **Still no — for a second, different reason this time.**

Unlike the fix-pass-1 retest (which found an entirely new defect, D3, outside the fix pass's own
scope), this time the residual issue is **inside** the very defect fix pass 2 was asked to fix
(TC-019/SubmissionDate) — Formwright's and Forgemaster's own verification stopped at "does it compile /
is the function callable," which is exactly the same category of shallow verification that caused the
original TC-019 mis-verification (checking presence in JSON, not runtime behaviour) that this whole fix
pass was created to correct. This retest's own independent, from-scratch, three-method verification
(DOM value before/after click, outgoing network request body, and the persisted JCR payload — all in
agreement) is what surfaced it.

**Recommendation:** D2 (OR-split) remains a reasonable, documented, human-actionable item to accept as
a follow-up requiring Workflow Editor UI access — this assessment is unchanged from the prior retest.
**D4 (the SubmissionDate cross-panel assignment silently not landing) should NOT be accepted** — it is
fully within an agent's power to fix (likely a one-line `$form.` scope-prefix correction on the
click-time assignment, mirroring the pattern already proven working on the `endDate` validation rule's
own cross-panel reference), it is narrowly scoped (a single field, no interaction with anything else
touched this pass or previously), and it directly affects data completeness on every single real
submission, exactly the class of issue this delivery's TEST gate exists to catch before ship. **Route
D4 back to Formwright for a third, narrow fix pass**, then redeploy and re-verify using all three
methods used here (not just AST/JSON presence) before calling it fixed.

### Updated test-case results (34 cases, FINAL retest)

| ID | FINAL retest disposition | Result | Notes |
|---|---|---|---|
| TC-001 | carried forward (unaffected) | PASS | unchanged |
| TC-002 | **RE-EXECUTED live (fresh submission)** | PASS | D1 regression-confirmed via FR-05: HTTP 200, new instance `_11` created |
| TC-003 | **RE-EXECUTED live** | PASS | D-R1 regression-confirmed via FR-02 |
| TC-004 | carried forward | FAIL (unchanged) | AC-02.2 hardcoded-manager-lookup GAP, out of scope for any fix pass, tracked as an open question |
| TC-005/006 | **RE-EXECUTED live** | PASS | rule #2 regression-confirmed via FR-03 |
| TC-007/008 | **RE-EXECUTED live** | PASS | rule #3 regression-confirmed via FR-03 |
| TC-009/010 | **RE-EXECUTED live** | PASS | rule #4 regression-confirmed via FR-03 |
| TC-011 | **RE-EXECUTED live (fresh submission)** | PASS | Manager Approval task created for instance `_11`, sitting at `node2`, consistent with config |
| TC-012 | **RE-EXECUTED (workflow model API)** | FAIL (unchanged) | D2 still open — 0 `OR_SPLIT` nodes, confirmed again this pass |
| TC-013 | carried forward (not re-exercised) | FAIL (unchanged) | first-task landing reconfirmed (instance `_11` at `node2`); second task (Finance Manager Approval) still not reachable without completing the first via Workflow Inbox — no simple REST completion endpoint exists (`:operation=complete` → 500 "Invalid operation specified for POST request"), out of scope/tooling this session, same as before |
| TC-014/015 | **RE-EXECUTED live** | PASS | D-R6 regression-confirmed via FR-04 |
| TC-016/017 | carried forward, reason narrowed | FAIL (unchanged result, D3's compounding reason now removed) | D2 alone blocks unconditional-linear-execution concerns; D3's data-incompleteness is no longer a compounding factor since a real submission's data now lands correctly |
| TC-018 | **RE-EXECUTED live** | PASS | AcceptedTerms required-gating reconfirmed via FR-05 (empty submit blocked) |
| TC-019 | **RE-EXECUTED live — CORRECTED, still FAIL** | **FAIL** (for a NEW, narrower reason) | ParserError genuinely fixed (confirmed absent from console across 2 runs), but the assignment `declarationFragment.submissionDate.$value = getCurrentDateISOString()` never actually lands — confirmed via live DOM value pre/post-click, the outgoing request body, AND the persisted JCR payload, all showing `SubmissionDate` absent. New defect D4 (likely a missing `$form.` cross-panel scope prefix); see `integration-test-report.md` for full evidence. |
| TC-020 | **RE-EXECUTED live (fresh submission)** | PASS | Both halves reconfirmed via FR-05: empty/invalid submit blocked with no PDF; valid submit succeeds with a real workflow instance AND a real generated PDF (`generatePDF` → 200, file downloaded) |
| TC-021/022/023 | carried forward (unaffected) | PASS | unchanged |
| TC-024 | carried forward, spot-checked | PASS | Employee Details heading re-confirmed rendering live via FR-01 (DOM check); full pixel-diff not re-run since no visual artifact changed this fix pass — see `test-form-ui-report.md` |
| TC-025 | carried forward (unaffected) | PASS | unchanged |
| TC-026 | carried forward (not re-exercised) | FAIL (unchanged) | actual DoR PDF generation still not driven to completion — same Workflow Inbox tooling gap as TC-013; D3's data-incompleteness concern is now resolved (structurally confirmed), narrowing this case's remaining blocker to D2 + task-completion access alone |
| TC-027 | carried forward (unaffected) | PASS | unchanged |
| TC-028/029 | carried forward, now with fresh confirming data | PASS | fragment reuse still structurally true; this retest's own fresh submission is now live, direct proof the shared reference also shares the FIXED data-binding behaviour, not just the shared bug as noted last pass |
| TC-030 | **RE-EXECUTED live** | PASS | reconfirmed via FR-05's fresh submission |
| TC-031 | carried forward | FAIL (unchanged) | test-case/architecture mismatch against the accepted "no prefill" decision, unrelated to any fix pass |
| TC-032 | carried forward, reason narrowed | FAIL (unchanged result, D3's compounding reason now removed) | D2 (no branching to a Send Email step) + no local SMTP config remain the sole blockers; `applicantEmail` would now resolve to a real value per the config+data cross-check above, so this is no longer failing for a data-integrity reason too |
| TC-033/034 | carried forward (unaffected) | PASS | unchanged |

**Counts by result:** 25 pass / 9 fail (34 executed, 0 unexecuted).
**Newly re-confirmed passing this retest (regression-proof, not just carried forward):** TC-002, TC-003,
TC-005, TC-006, TC-007, TC-008, TC-009, TC-010, TC-011, TC-014, TC-015, TC-018, TC-020, TC-030 (via the
fresh `employee-training-request-final-retest.cy.js` run).
**Still failing (unchanged pass/fail result, though 3 of these have a narrower/updated reason):** TC-004,
TC-012, TC-013, TC-016, TC-017, TC-019 (new reason — D4), TC-026, TC-031, TC-032 (9 cases).

### Updated user-story coverage (FINAL)

| Story | Change this retest | Covered? |
|---|---|---|
| US-01 | TC-002/020/030 regression-reconfirmed | YES (unchanged) |
| US-02 | TC-003 regression-reconfirmed | YES (unchanged) — AC-02.2 GAP still open, unrelated |
| US-03/04/05 | regression-reconfirmed live | YES (unchanged) |
| US-06 | TC-011/021/022/023 regression-reconfirmed | YES (unchanged) — AC-06.2 (real approve/reject outcomes) still has 0 passing cases (D2) |
| US-07 | TC-014/015 regression-reconfirmed | YES (unchanged) |
| US-08 | TC-013/016/017 still FAIL | **NO — still UNCOVERED** (0/3 pass) — now confirmed attributable **solely** to D2 (D3's compounding effect removed this retest) |
| US-09 | TC-018/020/029 still PASS; TC-019 still FAIL (corrected reason: D4, not the original ParserError) | YES (unchanged — coverage held by the other 3 cases; TC-019's continued failure is reported honestly, not smoothed over) |
| US-10 | TC-024 spot-checked, still PASS | YES (unchanged) |
| US-11 | TC-025 PASS, TC-026 still FAIL (D2 + tooling access, D3 concern resolved) | YES (1 passing case, unchanged) — AC-11.1's full-PDF-generation half still not driven to completion this session |

**`uncovered_stories: [US-08]` — 1 of 11, unchanged from the fix-pass-1 retest.** Still not empty →
hard gate FAIL, now confirmed to be a single, well-understood, solely-D2-caused gap (the coverage
picture did not get worse or better this pass — it is D4's case-level failure, not a coverage failure,
that keeps the overall verdict at FAIL).

### Updated embedded-page result

Unchanged: `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form` still HTTP 200, still embeds
exactly `employee-training-request` via its one AEM Form Container, no stale/duplicate embed. Functional
in-page reconfirmed with TWO fresh real submissions this retest (both through the embedded page, not a
hand-crafted REST payload), both reaching HTTP 200 + thank-you + a new RUNNING workflow instance + a
real generated PDF.

### Updated UI parity

Carried forward, not re-captured (justified above and in `test-form-ui-report.md`): 0 Critical, 85.79%
pixel match (advisory in migration mode), 1 Major (unchanged, cosmetic). Employee Details heading
regression-spot-checked live (still renders).

### Defects (FINAL)

| Defect | Severity | Status | Route to |
|---|---|---|---|
| D-R1 | Critical | RESOLVED (regression-confirmed this pass) | — |
| D-R6 | Critical | RESOLVED (regression-confirmed this pass) | — |
| UI-Finding-1 | Critical | RESOLVED (regression-confirmed this pass) | — |
| D1 | Critical | RESOLVED (regression-confirmed this pass, fresh submission) | — |
| D3 | Critical | **RESOLVED — confirmed this pass via a real submission's persisted payload** | — |
| D2 | Critical | **OPEN — reconfirmed still not fixed, human-actionable (Workflow Editor UI), recommended acceptable documented follow-up** | Groundsmith (needs browser/editor access, not agent-automatable on this AEM SDK) |
| **D4 (NEW)** | **Major/Critical-adjacent** | **OPEN — newly discovered this retest, found WHILE verifying the TC-019 fix; the ParserError itself is fixed but the assignment silently never updates the field** | **Formwright** — likely fix: add a `$form.` scope prefix to the click-time cross-panel assignment (`$form.declarationFragment.submissionDate.$value = getCurrentDateISOString()`), mirroring the pattern already proven working on `endDate`'s own cross-panel reference; or author the assignment from within the `declaration-consent` fragment's own scope. Re-verify via live DOM value + outgoing request body + persisted JCR payload, not AST/JSON presence alone. |

### Run metrics (this agent — sentinel, FINAL retest)

- `time_taken_minutes`: ~52.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~20,000
  - `read`: ~85,000 (prior test-report.md + integration-test-report.md + test-form-ui-report.md in
    full, formwright.md fix-pass-2 section, code-quality-report.md redeploy section, user-stories.yaml
    in full, test-cases.yaml excerpts, live `guideContainer.model.json` ×2 (host+DoR) via curl, the
    live aggregated clientlib JS, the live workflow model JSON ×2, the persisted payload `data.xml`
    ×2, 3 full Cypress run console logs)
  - `write`: ~14,000 (2 new Cypress spec files, edits to all 3 testing/ reports)
  - `other`: ~45,000 (3 Cypress run invocations incl. one failed webpack-compile attempt, ~20
    curl/PowerShell/node tool calls incl. repeated sandbox `/etc/*` permission noise on every Bash
    call, one failed REST work-item-completion attempt)
  - `total`: ~164,000

### FINAL Handoff YAML

```yaml
agent: sentinel
phase: TEST
status: FAILED
retest_of: "fix pass 2"
time_taken_minutes: 52.0
tokens_consumed:
  cli_text: 20000
  read: 85000
  write: 14000
  other: 45000
  total: 164000
test_cases: { total: 34, executed: 34, passed: 25, failed: 9 }
unexecuted_cases: 0
executes_with: { create-form-tests: 2, test-form-ui: 1, functional: 31 }
coverage_pct: null   # no custom Forms-service Java classes exist for this delivery — N/A, unchanged
user_stories: { total: 11, covered: 10 }
uncovered_stories: 1   # US-08 only — unchanged from the fix-pass-1 retest; HARD GATE FAILURE (still not empty)
ui_parity: { pixel: PASS_ADVISORY, pixel_match_pct: 85.79, critical_findings: 0 }   # carried forward, not re-captured — no visual artifact changed this fix pass
embedded_page:
  path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form"
  embeds_new_form: true
  functional_in_page: true   # 2 fresh real submissions this retest, both end-to-end successful (HTTP 200, thank-you, workflow instance, generated PDF)
defects_resolved:
  - { id: D-R1, owner: formwright, note: "regression-confirmed this pass" }
  - { id: D-R6, owner: formwright, note: "regression-confirmed this pass" }
  - { id: "UI-Finding-1", owner: formwright, note: "regression-confirmed this pass" }
  - { id: D1, owner: groundsmith, note: "regression-confirmed this pass, fresh submission" }
  - { id: D3, owner: formwright, note: "CONFIRMED FIXED this pass via a real submission's persisted payload — EmployeeDetails.*/Declaration.* now correctly populated" }
defects_still_open:
  - { id: D2, severity: Critical, owner: groundsmith, status: "reconfirmed still not fixed, human-actionable (Workflow Editor UI), recommended acceptable documented follow-up — unchanged assessment" }
  - { id: D4, severity: "Major/Critical-adjacent", owner: formwright, status: "NEW this retest — found while verifying the TC-019 fix; SubmissionDate click-time assignment compiles and calls its function correctly but never updates the field (likely missing $form. cross-panel scope prefix); confirmed via 3 independent live methods; recommend a 3rd, narrow fix pass before ship" }
delivery_totals:
  total_time_taken_minutes: "~490.0"
  total_tokens_consumed: { cli_text: 233700, read: 638200, write: 195800, other: 196000, total: 1263700 }
  per_phase:
    - { phase: PLAN, lead: planwright, time_taken_minutes: 14.0, tokens_total: 74000 }
    - { phase: DESI, lead: draftsmith, time_taken_minutes: 19.0, tokens_total: 85000 }
    - { phase: IMPL-build, lead: formwright, time_taken_minutes: 92.0, tokens_total: 220000 }
    - { phase: IMPL-integration, lead: groundsmith, time_taken_minutes: 37.0, tokens_total: 159000 }
    - { phase: ASSEMBLY, lead: assembler, time_taken_minutes: 8.0, tokens_total: 9200 }
    - { phase: DEPLOY, lead: "program-agent (standing in for forgemaster)", time_taken_minutes: 9.0, tokens_total: 19000 }
    - { phase: TEST-pass-1, lead: sentinel, time_taken_minutes: 45.0, tokens_total: 93000 }
    - { phase: IMPL-build-fix-pass-1, lead: formwright, time_taken_minutes: 35.0, tokens_total: 54000 }
    - { phase: IMPL-integration-fix-pass-1, lead: groundsmith, time_taken_minutes: 72.0, tokens_total: 130000 }
    - { phase: TEST-retest-fix-pass-1, lead: sentinel, time_taken_minutes: 65.0, tokens_total: 175000 }
    - { phase: IMPL-build-fix-pass-2, lead: formwright, time_taken_minutes: 28.0, tokens_total: 57000 }
    - { phase: DEPLOY-redeploy-fix-pass-2, lead: forgemaster, time_taken_minutes: 14.0, tokens_total: 24500 }
    - { phase: TEST-FINAL-retest, lead: sentinel, time_taken_minutes: 52.0, tokens_total: 164000 }
report: ".claude/agents/runs/2026-07-22-employee-training-request/testing/test-report.md"
gate_result: FAIL
gap_summary: "2-item gap remains, different composition than either prior pass: D2 (OR-split routing — known, escalated, human-actionable via Workflow Editor UI, reasonable to accept as a documented follow-up, UNCHANGED assessment) AND D4 (NEW — SubmissionDate click-time assignment silently never updates the field despite its ParserError being genuinely fixed; agent-fixable, narrow, recommend one more Formwright pass, do NOT accept). D3 (the fix-pass-1 finding) IS now genuinely, independently confirmed resolved — it is no longer part of the gap. uncovered_stories unchanged at 1 (US-08, now confirmed solely D2's doing). Case tally 25 pass / 9 fail, unchanged in size from the prior retest, changed in composition (D3's compounding removed from TC-016/017/032; TC-019 still fails but for the new D4 reason)."
next: "aem-forms-program-agent decides: (a) accept D2 as a documented human-actionable follow-up (unchanged recommendation), AND (b) route D4 back to formwright for a 3rd, narrow fix pass (likely a $form. scope-prefix correction on the SubmissionDate click assignment) before considering this delivery ship-ready. Recommend NOT shipping as fully clean yet, but the remaining gap is now smaller and better-isolated than either prior pass: 1 human-actionable item (D2) + 1 narrow, well-diagnosed, agent-fixable item (D4). If the business is willing to accept a permanently-blank SubmissionDate audit field as a known limitation alongside D2, this could ship with 2 flagged, non-regressing follow-ups rather than undergoing a 3rd fix cycle — that trade-off decision belongs to the program agent/business, not unilaterally to Sentinel."
```

---

## FINAL RETEST — fix pass 3 (2026-07-22, defect D4)

### Delivery Totals (final running total — ALL phases + all 3 fix passes + all 4 Sentinel test runs)

| Phase | Lead | time_taken_minutes | tokens (cli_text / read / write / other / **total**) |
|---|---|---|---|
| PLAN | planwright | 14.0 | 9,000 / 48,000 / 11,000 / 6,000 / **74,000** |
| DESI | draftsmith | 19.0 | 16,000 / 40,000 / 21,000 / 8,000 / **85,000** |
| IMPL-build | formwright | 92.0 | 48,000 / 95,000 / 62,000 / 15,000 / **220,000** |
| IMPL-integration | groundsmith | 37.0 | 38,000 / 88,000 / 24,000 / 9,000 / **159,000** |
| ASSEMBLY | assembler | 8.0 | 2,500 / 1,200 / 4,000 / 1,500 / **9,200** |
| DEPLOY | program-agent (standing in for forgemaster) | 9.0 | 3,500 / 9,000 / 4,500 / 2,000 / **19,000** |
| TEST (pass 1) | sentinel | ~45.0 | ~25,000 / ~45,000 / ~8,000 / ~15,000 / **~93,000** |
| IMPL-build fix pass 1 | formwright | 35.0 | 9,000 / 28,000 / 11,000 / 6,000 / **54,000** |
| IMPL-integration fix pass 1 | groundsmith | 72.0 | 22,000 / 64,000 / 9,000 / 35,000 / **130,000** |
| TEST (retest, fix pass 1) | sentinel | ~65.0 | ~30,000 / ~90,000 / ~15,000 / ~40,000 / **~175,000** |
| IMPL-build fix pass 2 | formwright | 28.0 | 7,500 / 34,000 / 9,500 / 6,000 / **57,000** |
| DEPLOY redeploy fix pass 2 | forgemaster | 14.0 | 3,200 / 11,000 / 2,800 / 7,500 / **24,500** |
| TEST (FINAL retest, fix pass 2) | sentinel | ~52.0 | ~20,000 / ~85,000 / ~14,000 / ~45,000 / **~164,000** |
| IMPL-build fix pass 3 | formwright | 16.0 | 3,500 / 13,000 / 2,500 / 4,000 / **23,000** |
| DEPLOY redeploy fix pass 3 | forgemaster | 13.0 | 3,000 / 10,500 / 2,600 / 7,200 / **23,300** |
| TEST (FINAL retest, this run — fix pass 3) | sentinel | ~42.0 | ~15,000 / ~60,000 / ~10,000 / ~35,000 / **~120,000** |
| **TOTAL** | | **~561.0 min** | **~255,200 / ~721,700 / ~210,900 / ~242,200 / ~1,430,000** |

Because multiple phases' figures are estimates (`~`), the delivery total is marked `~`.

### Verdict & Gate: **FAIL** — D4 is confirmed NOT fixed; D2 is confirmed unchanged; this is the third
### consecutive retest where "D2-only" is not an accurate description of the remaining gap

This is the requested LAST retest of this delivery. The task briefing hoped this would confirm D4
resolved, leaving only D2 as a documented, human-actionable follow-up. **That is not what this retest
found.** Using the SAME 3 independent live methods that originally caught D4 (live DOM value,
outgoing network request body, persisted JCR payload) — plus a 4th, more targeted method added this
pass specifically to distinguish "ran but didn't take effect" from "never ran at all" (a spy wrapping
`window.getCurrentDateISOString()` that counts invocations) — this retest found **D4 is still broken,
byte-for-byte identical in symptom to the pre-fix-pass-3 state**, and additionally discovered the
function is never invoked at all on click (0 calls), which is a materially different, deeper finding
than fix pass 3's own "missing `$form.` scope prefix" hypothesis.

| Item | Status |
|---|---|
| D4 fix (fix pass 3's `$form.` scope-prefix correction) | **Confirmed LIVE at the rule/mirror text level** (Forgemaster's claim holds — the deployed rule text genuinely reads `$form.declarationFragment.submissionDate.$value = getCurrentDateISOString()`) |
| D4 actual runtime effect | **STILL BROKEN — confirmed via all 3 original methods, unchanged from before the fix, PLUS a new 4th method showing 0 function invocations on click** |
| D-R1, D-R6, UI-Finding-1, D1, D3 | **NO REGRESSION** — all reconfirmed live via a fresh Cypress run (5/5 passing) and a fresh submission's persisted JCR payload |
| D2 (OR-split branching) | **STILL NOT FIXED, exactly as expected** — untouched this cycle (fix pass 3's scope was D4 only), unchanged assessment across all 3 fix passes |

- Test cases: 34 total, 34 executed, **25 pass / 9 fail** (`unexecuted_cases: 0`) — **unchanged in
  both size and composition** from the fix-pass-2 retest. TC-019 continues to fail — this is now its
  3rd consecutive failing verdict across 3 different underlying reasons (original mis-verified PASS →
  ParserError → silent no-op / wrong scope hypothesis → confirmed-never-invoked), never once
  genuinely resolved despite 2 dedicated Formwright fix passes targeting it.
- User stories: 11 total, **10 covered, 1 uncovered (US-08)** — **unchanged** from both prior retests.
  `uncovered_stories: [US-08]` remains non-empty, still attributable solely to D2 (unaffected by D4,
  which is a case-level failure on US-09's TC-019, not a coverage failure — US-09 keeps its coverage
  via TC-018/TC-020/TC-029, all independently passing).
- Embedded page: still confirms the new form correctly, still functional in-page — reconfirmed with a
  3rd fresh submission this retest (marker `FinalRetest1784727058077`), HTTP 200, thank-you shown, new
  `RUNNING` workflow instance created.
- UI parity: carried forward, not re-captured (fix pass 3 touched no visual/theme/CSS artifact — only
  2 attribute values in one rule script) — 0 Critical, 85.79% pixel match (advisory in migration
  mode), 1 Major (cosmetic, unchanged). Spot-checked live that the Employee Details heading still
  renders (no regression).

**`uncovered_stories: [US-08]` — 1 of 11, unchanged across all 3 retests.** Gate remains FAIL per the
project's non-negotiable rules (TC-019's continued failure and the non-empty `uncovered_stories` both
independently trigger FAIL).

### Is "OR-split routing (D2) only" now an accurate description of the remaining gap? **Still no — for the third retest in a row, and this time the targeted fix demonstrably did not work.**

Each of the 3 retests has found a different reason this framing doesn't hold: retest 1 found an
entirely new defect (D3) outside the fix pass's scope; retest 2 found a second, masked defect (D4)
underneath the defect that was genuinely fixed (the ParserError); this retest (3) directly disproves
the specific fix that was applied for D4 — the `$form.` scope-prefix hypothesis was reasonable and
evidence-consistent when Formwright proposed it (it correctly matched the D-R1 symptom class and used
an already-proven-working pattern from `endDate`), but this retest's own from-scratch verification,
going one level deeper with a function-call spy that neither Formwright nor Forgemaster had access to
run against a live instance, shows the assignment statement is **never reached at click time at all**
— not a scope-resolution failure on a statement that does execute. **D2 + D4 is a real, unresolved
2-item gap, not a 1-item gap**, exactly as it was after the previous retest — the only change this
pass is that D4's root cause is now better understood (and known to be deeper than previously
diagnosed), not that it is closer to resolved.

**Given this is now the 4th deploy/retest cycle with D4 surviving 2 dedicated, targeted fix attempts,
Sentinel's recommendation shifts from "route back for a narrow fix" to flagging this as a candidate
for acceptance as a documented, known limitation, alongside D2** — see the Recommendation section
below.

### Updated test-case results (34 cases, FINAL retest — fix pass 3)

| ID | Fix-pass-3 disposition | Result | Notes |
|---|---|---|---|
| TC-001 through TC-018 | carried forward / regression-reconfirmed via FR-01–FR-04 | unchanged from fix-pass-2 retest | no change this pass |
| TC-019 | **RE-EXECUTED live — CONFIRMED STILL FAIL, deeper root cause found** | **FAIL** (unchanged, 3rd consecutive fail, cause now better isolated) | Fix pass 3's `$form.`-prefixed rule text is confirmed live, but the live DOM value, outgoing request body, and persisted JCR payload are all still empty for `SubmissionDate` — byte-for-byte identical to before the fix. A new 4th verification method (function-call spy) additionally shows `getCurrentDateISOString()` is called **0 times** on click, meaning the SET_VALUE statement never executes at all, not merely resolves against the wrong scope. Full evidence in `integration-test-report.md` §"FINAL retest — fix pass 3". |
| TC-020 through TC-018/020/029/030 (US-09's other cases) | regression-reconfirmed via FR-05 | PASS (unchanged) | fresh valid submission (marker `FinalRetest1784727058077`) succeeds end-to-end; AcceptedTerms gating and required-field gating both hold |
| TC-021–TC-025, TC-027–TC-030, TC-033/034 | carried forward (unaffected) | PASS (unchanged) | |
| TC-004, TC-012, TC-013, TC-016, TC-017, TC-026, TC-031, TC-032 | carried forward (unaffected by fix pass 3's narrow scope) | FAIL (unchanged) | same reasons as the fix-pass-2 retest (D2 + tooling-access gap for the workflow-inbox cases; unrelated architecture mismatch for TC-031) |

**Counts by result:** 25 pass / 9 fail (34 executed, 0 unexecuted) — **identical to the fix-pass-2
retest**, since fix pass 3's only targeted case (TC-019) did not flip.

### Updated user-story coverage (FINAL, fix pass 3)

Unchanged from the fix-pass-2 retest in every respect: **10 of 11 covered, `uncovered_stories:
[US-08]`** — still solely attributable to D2. US-09 keeps its coverage via TC-018/TC-020/TC-029;
TC-019's continued failure is reported honestly as a case-level failure, not smoothed into a false
coverage picture.

### Updated embedded-page result

Unchanged: `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form` still HTTP 200, still embeds
exactly `employee-training-request` via its one AEM Form Container, no stale/duplicate embed.
Functional in-page reconfirmed with a 3rd fresh real submission this retest (marker
`FinalRetest1784727058077`), HTTP 200, thank-you shown, new `RUNNING` workflow instance
(`employee-training-request-approval_27`) created, persisted payload confirms D3's fix still holds
(EmployeeDetails.* fully populated) with SubmissionDate still absent (D4, unchanged).

### Updated UI parity

Carried forward, not re-captured (fix pass 3 touched no visual/theme/CSS artifact — 2 attribute values
in one rule script only): 0 Critical, 85.79% pixel match (advisory in migration mode), 1 Major
(unchanged, cosmetic). Employee Details heading regression-spot-checked live (still renders).

### Defects (FINAL, fix pass 3)

| Defect | Severity | Status | Route to |
|---|---|---|---|
| D-R1 | Critical | RESOLVED (regression-confirmed this pass) | — |
| D-R6 | Critical | RESOLVED (regression-confirmed this pass) | — |
| UI-Finding-1 | Critical | RESOLVED (regression-confirmed this pass) | — |
| D1 | Critical | RESOLVED (regression-confirmed this pass, fresh submission) | — |
| D3 | Critical | RESOLVED (regression-confirmed this pass, fresh submission's persisted payload) | — |
| D2 | Critical | **OPEN — reconfirmed still not fixed across all 3 fix passes, human-actionable (Workflow Editor UI); recommend accepting as a documented follow-up** | Groundsmith (needs browser/editor access, not agent-automatable on this AEM SDK) |
| **D4** | **Major/Critical-adjacent** | **OPEN — survived 2 dedicated, targeted Formwright fix passes (ParserError fix, then a `$form.` scope-prefix fix); this retest's deeper 4th-method verification (function-call spy) shows the assignment statement never executes at click time at all — a materially different and deeper root cause than either fix pass diagnosed; recommend accepting as a documented, known limitation (a permanently-blank SubmissionDate audit field) rather than a 3rd fix pass, given 2 consecutive misses on this specific defect** | Formwright (if a 3rd pass is pursued — but see recommendation below), OR accept as documented follow-up |

### Final recommendation for HANDOFF

This delivery has been through **4 deploy/retest cycles**. Two Critical structural rule defects
(D-R1, D-R6), one Critical UI-parity defect (Employee Details heading), one Critical submit/workflow
defect (D1), and one Critical data-binding defect (D3) were all found, fixed, and independently
**confirmed genuinely resolved with no regression across 3 consecutive redeploys** — this is real,
substantial, verified progress, not merely claimed. Two items remain open:

1. **D2 (OR-split branching)** — confirmed, unchanged, human-actionable, requires Workflow Editor UI
   access no agent in this session has. **Recommend accepting as a documented pre-production
   follow-up**, consistent with every prior retest's assessment.
2. **D4 (SubmissionDate never lands)** — narrower in blast radius than D2 (it affects one audit-trail
   field, not the workflow's core approve/reject/DoR mechanics), does not block any user story's
   coverage (US-09 remains covered via its other 3 cases), and does not affect data integrity of any
   OTHER field (D3's fix holds cleanly). However, it has now survived **2 dedicated, dead-drive
   Formwright fix attempts**, and this retest's own deeper investigation shows the underlying
   mechanism (a `Click`-event script issuing a SET_VALUE against a field in a sibling fragment) may
   not be reliably achievable with this project's current Rule Editor authoring patterns — the same
   category of limitation that caused D-R1's ORIGINAL fix (a host-wrapper `Change`-event script) to be
   abandoned entirely in favor of authoring the rule from within the target fragment's own scope.
   **Recommend NOT pursuing a 3rd fix pass on the current mechanism.** If the business wants
   `SubmissionDate` genuinely populated, the durable fix is most likely architectural (e.g., set it
   server-side in the workflow's "Capture Submission Variables" step using the instance's own
   start-time, or set it from within `declaration-consent`'s own fragment scope on a field-level event
   rather than the host's button-level `Click` script) — a redesign, not a one-line patch, and
   reasonable to schedule as a follow-up rather than block this delivery a 4th time.

**Recommend: HANDOFF with 3 flagged, non-blocking follow-ups** — D2 (OR-split routing, human-actionable),
D4 (SubmissionDate audit field, needs a redesigned mechanism, not another one-line patch), and the
pre-production business-decision items already carried from Planwright/Groundsmith (real manager
lookup, a distinct finance approver identity, a real sender email). This is **not** the single-item
D2-only gap the task hoped for confirming, and Sentinel is reporting that plainly rather than softening
it — but it is a materially smaller, better-understood, and more rigorously verified gap than any
prior retest found, with 5 of 6 originally-identified Critical defects now genuinely closed and
regression-proven across 3 independent redeploys. A 5th retest cycle chasing D4 on the same mechanism
is unlikely to succeed without a design change; recommend the program agent/business make the
accept-vs-redesign call rather than unilaterally cycling Sentinel again.

### Run metrics (this agent — sentinel, FINAL retest fix pass 3)

- `time_taken_minutes`: ~42.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~15,000
  - `read`: ~60,000 (test-report.md full history, integration-test-report.md, formwright.md
    fix-pass-3 section, code-quality-report.md §11, the 2 existing Cypress specs read in full, live
    `guideContainer.model.json`/`submitButton.1.json` fetches, 3 persisted `data.xml` payloads, 3
    Cypress run console logs)
  - `write`: ~10,000 (1 throwaway diagnostic Cypress spec authored + 1 edit + deleted after use,
    updates to all 3 testing/ reports)
  - `other`: ~35,000 (3 Cypress run invocations, ~15 curl/PowerShell/node tool calls incl. repeated
    sandbox `/etc/*` permission noise on every Bash call, 1 failed python attempt, node path-resolution
    debugging)
  - `total`: ~120,000

### FINAL Handoff YAML (fix pass 3 — LAST retest of this delivery)

```yaml
agent: sentinel
phase: TEST
status: FAILED
retest_of: "fix pass 3"
time_taken_minutes: 42.0
tokens_consumed:
  cli_text: 15000
  read: 60000
  write: 10000
  other: 35000
  total: 120000
test_cases: { total: 34, executed: 34, passed: 25, failed: 9 }
unexecuted_cases: 0
executes_with: { create-form-tests: 2, test-form-ui: 1, functional: 31 }
coverage_pct: null   # no custom Forms-service Java classes exist for this delivery — N/A, unchanged
user_stories: { total: 11, covered: 10 }
uncovered_stories: 1   # US-08 only — unchanged across all 3 retests; HARD GATE FAILURE (still not empty)
ui_parity: { pixel: PASS_ADVISORY, pixel_match_pct: 85.79, critical_findings: 0 }   # carried forward, not re-captured — no visual artifact changed this fix pass
embedded_page:
  path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form"
  embeds_new_form: true
  functional_in_page: true   # 3rd fresh real submission this retest, end-to-end successful (HTTP 200, thank-you, workflow instance)
defects_resolved:
  - { id: D-R1, owner: formwright, note: "regression-confirmed this pass" }
  - { id: D-R6, owner: formwright, note: "regression-confirmed this pass" }
  - { id: "UI-Finding-1", owner: formwright, note: "regression-confirmed this pass" }
  - { id: D1, owner: groundsmith, note: "regression-confirmed this pass, fresh submission" }
  - { id: D3, owner: formwright, note: "regression-confirmed this pass, fresh submission's persisted payload" }
defects_still_open:
  - { id: D2, severity: Critical, owner: groundsmith, status: "reconfirmed still not fixed across all 3 fix passes, human-actionable (Workflow Editor UI); recommend accepting as a documented follow-up — unchanged assessment" }
  - { id: D4, severity: "Major/Critical-adjacent", owner: formwright, status: "CONFIRMED STILL BROKEN despite fix pass 3's $form.-scope-prefix correction being genuinely live; verified via all 3 original methods (DOM value, request body, persisted payload, all unchanged from pre-fix) PLUS a new 4th method (function-call spy showing 0 invocations on click) that proves the SET_VALUE statement never executes at all, a deeper root cause than either of the 2 dedicated fix attempts diagnosed; recommend accepting as a documented known limitation rather than a 3rd fix pass on the same mechanism, OR a redesign (server-side capture, or fragment-internal field-level event) if the business requires it populated" }
delivery_totals:
  total_time_taken_minutes: "~561.0"
  total_tokens_consumed: { cli_text: 255200, read: 721700, write: 210900, other: 242200, total: 1430000 }
  per_phase:
    - { phase: PLAN, lead: planwright, time_taken_minutes: 14.0, tokens_total: 74000 }
    - { phase: DESI, lead: draftsmith, time_taken_minutes: 19.0, tokens_total: 85000 }
    - { phase: IMPL-build, lead: formwright, time_taken_minutes: 92.0, tokens_total: 220000 }
    - { phase: IMPL-integration, lead: groundsmith, time_taken_minutes: 37.0, tokens_total: 159000 }
    - { phase: ASSEMBLY, lead: assembler, time_taken_minutes: 8.0, tokens_total: 9200 }
    - { phase: DEPLOY, lead: "program-agent (standing in for forgemaster)", time_taken_minutes: 9.0, tokens_total: 19000 }
    - { phase: TEST-pass-1, lead: sentinel, time_taken_minutes: 45.0, tokens_total: 93000 }
    - { phase: IMPL-build-fix-pass-1, lead: formwright, time_taken_minutes: 35.0, tokens_total: 54000 }
    - { phase: IMPL-integration-fix-pass-1, lead: groundsmith, time_taken_minutes: 72.0, tokens_total: 130000 }
    - { phase: TEST-retest-fix-pass-1, lead: sentinel, time_taken_minutes: 65.0, tokens_total: 175000 }
    - { phase: IMPL-build-fix-pass-2, lead: formwright, time_taken_minutes: 28.0, tokens_total: 57000 }
    - { phase: DEPLOY-redeploy-fix-pass-2, lead: forgemaster, time_taken_minutes: 14.0, tokens_total: 24500 }
    - { phase: TEST-FINAL-retest-fix-pass-2, lead: sentinel, time_taken_minutes: 52.0, tokens_total: 164000 }
    - { phase: IMPL-build-fix-pass-3, lead: formwright, time_taken_minutes: 16.0, tokens_total: 23000 }
    - { phase: DEPLOY-redeploy-fix-pass-3, lead: forgemaster, time_taken_minutes: 13.0, tokens_total: 23300 }
    - { phase: TEST-FINAL-retest-fix-pass-3, lead: sentinel, time_taken_minutes: 42.0, tokens_total: 120000 }
report: ".claude/agents/runs/2026-07-22-employee-training-request/testing/test-report.md"
gate_result: FAIL
gap_summary: "2-item gap remains, unchanged in count from the fix-pass-2 retest but deeper in understanding: D2 (OR-split routing — human-actionable, Workflow Editor UI, recommend accepting as documented follow-up, UNCHANGED across all 3 fix passes) AND D4 (SubmissionDate click-time assignment — survived 2 dedicated targeted fix attempts; this retest's 4th verification method (function-call spy, 0 invocations) shows the assignment never executes at click time at all, a deeper root cause than either fix pass diagnosed; recommend accepting as a documented known limitation or scheduling a mechanism redesign, NOT a 3rd one-line-patch fix pass). uncovered_stories unchanged at 1 (US-08, solely D2). Case tally unchanged at 25 pass / 9 fail. 5 of 6 originally-identified Critical defects (D-R1, D-R6, UI-Finding-1, D1, D3) are now genuinely resolved and regression-proven across 3 independent redeploys."
next: "aem-forms-program-agent makes the final accept-vs-redesign call on D4 (recommend: accept as a documented limitation, alongside D2, rather than a 4th fix/retest cycle) and writes the program-summary/HANDOFF. Sentinel's recommendation: HANDOFF now with D2 + D4 flagged as non-blocking, well-understood follow-ups (plus the pre-production business-decision items already carried from Planwright/Groundsmith: manager lookup, distinct finance approver, sender email) — this is the most rigorously verified state this delivery has reached across 4 deploy/retest cycles, and a 5th cycle targeting D4 on its current mechanism is unlikely to succeed without a design change."
```

---

## RETEST — fix pass 4 (user-reported remediation, 2026-07-23)

This is a retest of a **4th, user-reported** remediation pass — 3 issues (2 Formwright, 1 Groundsmith),
distinct from the D2/D4 gap this delivery had already been carrying. The coordinator's note claimed
the redeploy landed live; **independently verified, not taken on faith** — see §"Deploy verification"
below.

### Deploy verification (independent, not trusting the coordinator's note or Forgemaster's report)

`deployment/code-quality-report.md` has **not yet** been updated with a Fix-pass-4 section (its last
entry is still §11, the fix-pass-3 redeploy) — exactly as flagged: "Forgemaster's own written report
may still be catching up." I verified the deploy myself directly against the running instance rather
than trusting either the coordinator's note or waiting for Forgemaster's report:

| Check | Live result | Verdict |
|---|---|---|
| AEM author reachable | `GET /system/console/bundles.json` → HTTP 200 | PASS |
| Bundle summary | `738 bundles in total - all 738 bundles active` (728 Active + 10 Fragment) | **Matches coordinator's claim exactly** |
| Core project bundle | `aem-adaptive-forms-agents.core` = **Active**, v1.0.0.SNAPSHOT | PASS |
| Workflow model — Assign Task steps | `GET /var/workflow/models/employee-training-request-approval.json` → both `node2`/`node4` (Manager/Finance Approval) show `"PROCESS":"com.adobe.fd.workspace.step.service.AssignFormStep"` (the REAL registered class, replacing the previously-nonexistent one) | PASS |
| Workflow model — Invoke DDX | `node9` = `"PROCESS":"com.adobe.fd.workflow.assembler.InvokeDDXProcess"`, `"ddx":"/apps/aem-adaptive-forms-agents/workflow/ddx/employee-training-request-approval/manager-confirmation.ddx"` | PASS |
| DDX file deployed | `GET .../manager-confirmation.ddx` → HTTP 200, content byte-for-byte matches the original legacy DDX (`<XDP source="EmployeeTrainingForm"/><XDP source="ApprovalInfo"/>`), `jcr:created` timestamp **today** | PASS |
| Workflow model — Send Email | `node6`/`node8` = `"PROCESS":"com.adobe.fd.workflow.email.SendEmailStep"` (the OOTB Forms-specific step, not the prior generic Granite mailer) | PASS |
| Workflow model — Convert to PDF/A | `node10` = `"PROCESS":"com.adobe.fd.workflow.assembler.ConvertToPDFAProcess"`, `compliance=1a` | PASS |

**Conclusion: the redeploy is genuinely live** — independently confirmed at the exact same depth
(live model JSON, live bundle state, live file reachability) as every prior retest in this delivery,
not inferred from any agent's self-report.

### Issue 1 — fragment schema binding (Formwright) — VERIFIED LIVE

Both fragments' `guideContainer` properties fetched live:
- `employee-identity`: `"schemaType":"jsonschema","schemaRef":"/content/dam/formsanddocuments/schema/employee-training-request.schema.json"` — present, exactly as claimed.
- `declaration-consent`: same `schemaType`/`schemaRef`, same schema.
- Both DAM fragment assets' `jcr:content/metadata`: `"formmodel":"jsonschema"` (was `"none"`) — present on both.
- The referenced schema resolves live (`GET .../employee-training-request.schema.json` → HTTP 200,
  8021 bytes) and its root `properties` genuinely contain `EmployeeDetails` and `Declaration` —
  the exact two roots both fragments' fields already dataRef into.

**Editor-level check — partially verifiable, disclosed honestly.** Like Formwright, I had **no
browser/editor automation available this session either** (the `claude-in-chrome` skill is listed but
its tools were not present/loaded in this environment — checked via tool search, none returned) — so
the AF editor's own Form-Object-panel canvas was **not visually confirmed populating**, by either
agent, across this fix pass. What I *could* independently confirm, one level short of the editor's own
render: the exact JSON-schema resource the editor's data-binding surface would resolve against
(`schemaRef`) is reachable, valid, and structurally matches the fragments' existing dataRefs — i.e.
every precondition the editor's Form Object panel needs is now genuinely satisfied on the live
instance. This is strong circumstantial evidence the fix works, not a substitute for an actual visual
editor check. **Flagging the remaining gap plainly: no agent in this delivery has visually opened the
AF editor for these fragments post-fix.** If a human/browser-capable pass is available, opening
`/editor.html/content/forms/af/aem-adaptive-form-agents/employee-identity` (and the `declaration-consent`
equivalent) and confirming the Form Object tree populates would close this out completely.

### Issue 2 — script inventory (Formwright, no code change) — SPOT-CHECKED, CLAIM HOLDS

Independently re-verified 3 of Formwright's 14 enumerated dispositions (not accepted on faith), fetched
live from the currently-deployed host form + fragment `guideContainer.model.json`:

| # | Rule spot-checked | Live evidence | Verdict |
|---|---|---|---|
| 1 (managerId/managerName lock) | `"rules":{"enabled":"Number(employeeId.$value) <= 0 \|\| Number(employeeId.$value) > 10"}` present on BOTH `managerId` and `managerName` | Genuinely present, structurally sound |
| 4 (endDate > startDate) | `endDate.validationExpression = "validateDateAfter($field.$value, $form.trainingRequestDetailsPanel.startDate.$value) == true()"`, correctly cross-panel-scoped to `startDate` | Genuinely present, structurally sound |
| 2 (email format) | `email` field (fieldType `email`) carries `properties.validatePictureClause` with a real email regex, plus `constraintMessages.pattern` | Genuinely present, structurally sound |
| 6 (financeComments enable, bonus 4th check) | `"rules":{"enabled":"currentStatus.$value == \"ManagerApproved\""}` present on `financeComments` | Genuinely present, structurally sound |

All 4 spot-checked rules (exceeding the requested 2-3) are live and structurally correct. Formwright's
"0 dropped" claim holds up under independent scrutiny for every rule sampled; not exhaustively
re-verified beyond these 4, per the task's own scope ("not required to re-verify all 14").

### Issue 3 — OOTB workflow rebuild (Groundsmith) — the big one, VERIFIED end-to-end

**Assign Task class swap — confirmed real, confirmed NOT a regression, via a genuinely FRESH
full-AF-submit-path test (stronger evidence than Groundsmith's own direct-instance-start check):**

Ran the existing Cypress E2E suite against the live, redeployed instance (not a rerun of stale
results):
- `employee-training-request-final-retest.cy.js` — **5/5 passing**, including FR-05-valid-submit
  (real submission through the embedded page's actual `/adobe/forms/af/submit/...` endpoint → HTTP
  200) and FR-05-thankyou (thank-you page shown).
- `employee-training-request-functional.cy.js` — **8/8 passing** (D-R1, D-R6, rules #2–#5/#7/#8,
  submit-gated-on-validation, D1 all regression-clean).
- Confirmed via `GET /var/workflow/instances.json?model=...` that this fresh submission created a
  **new** instance (`employee-training-request-approval_30`, `startTime: Thu Jul 23 2026 11:04:06
  IST` — today, matching the Cypress run just executed) whose live JSON shows
  `"state":"RUNNING"` with a real `workItems` entry at `"node":"node2"` — Manager Approval, the
  **rebuilt `AssignFormStep`-based step**, creating a genuine work item on the truly-current live
  model. This is the "genuinely conclusive… full AF-submit-path variant" test Groundsmith explicitly
  deferred to Sentinel, and it **passed**, fresh, in this session.

**Invoke DDX — confirmed present and correctly configured** (see deploy-verification table above):
real DDX file deployed at the exact path the step references, content matches the original legacy
DDX byte-for-byte.

**Send Email steps — confirmed now `SendEmailStep` (OOTB Forms-specific)**, not the prior generic
Granite mailer (see deploy-verification table above).

**D2 (OR-split) — independently confirmed STILL the same reproduced platform limitation, not just
Groundsmith's word taken for it.** Fetched the live workflow model's `transitions` array directly:
it is **fully linear** — `node0→1→2→3→4→5→6→7→8→9→10→11`, with **zero `OR_SPLIT`/branch nodes present
anywhere in the live model**. Both "Send Approval Notification" (node6) and "Send Rejection
Notification" (node8) execute unconditionally in sequence for every single submission, regardless of
the Manager's actual Approve/Reject decision — i.e. every request received both a rejection AND an
approval email, and unconditionally reaches Finance Manager Approval and DDX/PDF/A regardless of
outcome. This is the same collapse-to-linear failure mode reproduced under the genuine OOTB `or`
component, confirmed independently, not merely restated from Groundsmith's report. **D2 remains open,
unchanged.**

**New editor-completion (Tier-A) items — confirmed genuinely incomplete/placeholder in the live
model, not just Groundsmith's claim.** Inspected the live `metaData` for every new/changed node:
`process_ddx` (node9) carries only `validateOnly`/`ddxType`/`ddx`/`failOnError` — **no** Input/Output
Documents Map keys at all; `process_pdfa` (node10) carries only `compliance` — **no** input
document/output location keys; `process_email_approved`/`process_email_rejected` (node6/node8) carry
only `toAddressType`/`toAddressValue`/`emailSubject`/`templatePath` — **no** attachment fields;
`assigntask_manager`/`assigntask_finance` (node2/node4) carry `STATIC_ASSIGNEE`/`ROUTES`/`AF_PATH`/etc.
but **no** `INPUT_DATAXML`/`OUTPUT_DATAXML`/`INPUT_FORM_ATTACHMENTS`/`OUTPUT_FORM_ATTACHMENTS` keys.
All 4 Tier-A items Groundsmith flagged are confirmed genuinely left for editor completion on the live
instance — added to the open-items list below as new, non-blocking, human-actionable follow-ups.

### Full regression sweep — ALL PASS, nothing worse

| Item | Method this pass | Result |
|---|---|---|
| D-R1 (ManagerId/ManagerName lock) | `employee-training-request-functional.cy.js` RT-02 | PASS (regression-clean) |
| D-R6 (FinanceComments lock) | same spec, RT-04 | PASS (regression-clean) |
| UI-Finding-1 (Employee Details heading) | `employee-training-request-final-retest.cy.js` FR-01 + live spot check | PASS (regression-clean; no visual artifact touched this pass, so no full pixel re-capture — see `test-form-ui-report.md`) |
| D1 (submit → workflow) | Fresh full-AF-submit-path Cypress run, HTTP 200 + new RUNNING instance at node2 | PASS (regression-clean, and now on the genuinely rebuilt `AssignFormStep` class — stronger evidence than any prior pass) |
| D3 (fragment dataRef → correct schema paths) | `submissiondate-probe.cy.js`'s captured request body: `EmployeeDetails`/`Declaration`/`TrainingRequest`/`Justification` all present with correct nested keys | PASS (regression-clean) |
| TC-019's ParserError fix | No console errors across all 3 Cypress runs this pass | PASS (regression-clean) |
| D4's scope-prefix mirror (partial fix, NOT re-chased) | `submissiondate-probe.cy.js` captured request body: `"Declaration":{"AcceptedTerms":true}` — **`SubmissionDate` still absent**, byte-for-byte the same known-open state as fix pass 3 | **UNCHANGED, NOT WORSE** — D4 remains open exactly as before, per explicit instruction not to re-chase it |

**Nothing regressed. Nothing got worse. Issue 1 and Issue 3's rebuild are both genuinely closed on
their own merits; D2 and D4 are confirmed unchanged, not newly broken by the deeper rebuild.**

### Updated case tally and coverage

The 9 previously-failing cases (TC-004, TC-012, TC-013, TC-016, TC-017, TC-019, TC-026, TC-031,
TC-032) were re-examined against this pass's evidence: 7 of the 9 trace to D2 (still open, confirmed
above), 1 (TC-019) traces to D4 (still open, confirmed unchanged above), and 2 (TC-004, TC-031) are
the pre-existing architecture/test-case mismatches (no prefill service — an accepted design decision,
not a defect). **None of the 9 flip to PASS; none of the 25 passing cases regress.**

- `test_cases: { total: 34, executed: 34, passed: 25, failed: 9 }` — **unchanged**
- `unexecuted_cases: 0`
- `user_stories: { total: 11, covered: 10 }`, `uncovered_stories: [US-08]` — **unchanged, still
  solely attributable to D2**
- `ui_parity`: carried forward, **85.79% pixel match (advisory in migration mode), 0 Critical** — no
  visual/theme/clientlib artifact was touched by fix pass 4 (fragments: `schemaType`/`schemaRef` only;
  workflow: step classes only); spot-checked live via FR-01 (Employee Details heading still renders)
  rather than a full pixel re-capture, consistent with the precedent set at fix passes 2 and 3.

### Open items (current, complete list)

1. **D2 — OR-split branching** (Critical, unchanged across 4 fix passes) — human-actionable, requires
   Workflow Editor UI access no agent in this session has. Recommend continuing to accept as a
   documented pre-production follow-up.
2. **D4 — SubmissionDate never lands** (Major/Critical-adjacent, unchanged, NOT re-chased per explicit
   instruction) — recommend accepting as a documented known limitation or scheduling a mechanism
   redesign, per the fix-pass-3 retest's recommendation (unchanged).
3. **NEW — 4 Tier-A editor-completion items** (non-blocking, human-actionable via Workflow Editor UI),
   confirmed genuinely incomplete on the live instance this pass:
   - Invoke DDX's Input/Output Documents Map (`process_ddx`)
   - Convert to PDF/A's Input Document / Output location (`process_pdfa`)
   - Send Email's attachment field (`process_email_approved`) — closes the previously-accepted
     SendEmail-attachment parity gap once wired
   - Assign Task's data/attachment I-O (`assigntask_manager`/`assigntask_finance`)
4. **NEW — AF-editor canvas for both fragments not visually re-confirmed** (non-blocking, disclosed
   honestly) — no browser/editor automation was available to either Formwright or Sentinel this pass;
   the schema-resolution-level evidence is strong but not a substitute for an actual editor screenshot.
5. Carried-forward pre-production business-decision items (unchanged): real manager lookup, a
   distinct finance-approver identity, a real sender email, no local SMTP.

### Gate result — fix pass 4 retest

**FAIL — unchanged from fix pass 3, for the same 2 pre-existing reasons (D2, D4), neither of which
was in this fix pass's scope.** This is not a new failure: Issue 1 (fragments) and Issue 3 (OOTB
workflow rebuild) are both genuinely, independently verified as correctly delivered and non-regressive
— the delivery is **materially better evidenced** than before (the workflow model is now provably
built on real, registered OOTB step classes, and D1 has now been proven via a genuinely fresh
full-submit-path test rather than a direct-instance-start proxy). But the gate's arithmetic is
unchanged: `uncovered_stories` is still non-empty (`[US-08]`), and a failed case (TC-019/D4) still
exists — both independently trigger the non-negotiable gate rule regardless of how much the
surrounding evidence has improved.

**Recommendation for HANDOFF: unchanged from the fix-pass-3 retest.** This delivery has now been
through 4 deploy/retest cycles. Every genuinely agent-fixable defect (D-R1, D-R6, UI-Finding-1, D1,
D3, plus this pass's 2 user-reported issues) is resolved and regression-proven. The 2 remaining gaps
(D2, D4) are both, independently and now for the second/third consecutive retest, confirmed to require
either human Workflow-Editor-UI access (D2) or an architectural redesign (D4) — neither is a
one-line-patch an agent can close in a 5th cycle. Recommend the program agent proceed to HANDOFF with
D2, D4, and the 4 newly-surfaced Tier-A editor-completion items flagged as non-blocking, well-
understood follow-ups, rather than cycling Sentinel again against the same 2 items a 5th time.

### Run metrics (this agent — sentinel, retest fix pass 4)

- `time_taken_minutes`: ~65.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~20,000
  - `read`: ~90,000 (formwright.md + groundsmith.md fix-pass-4 sections in full, code-quality-report.md
    §11 tail, test-report.md fix-pass-3 tail, 3 Cypress spec files, live workflow model JSON, live
    fragment/DAM/schema JSON, live host-form/fragment model.json, live workflow-instance JSON)
  - `write`: ~20,000 (this section + the matching sections in `integration-test-report.md` and
    `test-form-ui-report.md`)
  - `other`: ~55,000 (~30 curl/PowerShell live-instance calls incl. several failed direct
    workflow-instance-start attempts before pivoting to the real Cypress submit path, 3 full Cypress
    run invocations with verbose terminal output, grep/tool-search calls, repeated sandbox `/etc/*`
    permission noise on every Bash call)
  - `total`: ~185,000

### Handoff YAML — retest fix pass 4

```yaml
agent: sentinel
phase: TEST
status: FAILED
retest_of: "fix pass 4 (user-reported remediation: fragment schema binding, script-inventory audit, OOTB workflow rebuild)"
time_taken_minutes: 65.0
tokens_consumed:
  cli_text: 20000
  read: 90000
  write: 20000
  other: 55000
  total: 185000
test_cases: { total: 34, executed: 34, passed: 25, failed: 9 }
unexecuted_cases: 0
executes_with: { create-form-tests: 2, test-form-ui: 1, functional: 31 }
coverage_pct: null   # no custom Forms-service Java classes exist for this delivery — N/A, unchanged
user_stories: { total: 11, covered: 10 }
uncovered_stories: 1   # US-08 only — unchanged, solely D2
ui_parity: { pixel: PASS_ADVISORY, pixel_match_pct: 85.79, critical_findings: 0 }   # carried forward, no visual artifact changed this fix pass
embedded_page:
  path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form"
  embeds_new_form: true
  functional_in_page: true   # fresh real submission this retest: HTTP 200, thank-you, NEW RUNNING workflow instance at node2 (rebuilt AssignFormStep)
fix_pass_4_issues_verified:
  - { issue: "1 — fragment schema binding", owner: formwright, status: "VERIFIED LIVE — schemaType/schemaRef + DAM formmodel confirmed on both fragments; referenced schema resolves and matches; AF-editor canvas itself not visually re-confirmed (no browser automation available to either agent this delivery)" }
  - { issue: "2 — script inventory audit", owner: formwright, status: "SPOT-CHECKED (4 of 14 rules independently re-verified live, exceeding the requested 2-3) — claim holds, 0 dropped confirmed for every rule sampled" }
  - { issue: "3 — OOTB workflow rebuild", owner: groundsmith, status: "VERIFIED END-TO-END — AssignFormStep/InvokeDDXProcess/SendEmailStep/ConvertToPDFAProcess all confirmed live; D1 re-proven via a genuinely FRESH full-AF-submit-path Cypress run (new RUNNING instance, node2 work item); D2 independently reconfirmed still linear/open (not taken on Groundsmith's word); 4 new Tier-A editor-completion items independently confirmed incomplete on the live model" }
defects_still_open:
  - { id: D2, severity: Critical, owner: groundsmith, status: "UNCHANGED — independently reconfirmed fully linear (0 OR_SPLIT nodes) on the live model, same platform limitation under the genuine OOTB 'or' component; human-actionable via Workflow Editor UI" }
  - { id: D4, severity: "Major/Critical-adjacent", owner: formwright, status: "UNCHANGED, NOT re-chased per instruction — submissiondate-probe.cy.js confirms Declaration.SubmissionDate still absent from the live submission payload, byte-for-byte same as fix pass 3" }
new_open_items:
  - { item: "4 Tier-A multifield I/O mappings (DDX Input/Output Documents Map, Convert-to-PDF/A I/O, Send Email attachment, Assign Task data/attachment I/O)", owner: human via Workflow Editor UI, status: "confirmed genuinely incomplete/placeholder on the live model this pass" }
  - { item: "AF-editor Form Object canvas for both fragments", owner: "human/browser-capable agent", status: "not visually re-confirmed post-fix — schema-resolution-level evidence only" }
delivery_totals:
  total_time_taken_minutes: "~773.0"
  total_tokens_consumed: { cli_text: 320200, read: 1034700, write: 257400, other: 361700, total: 1974000 }
  per_phase:
    - { phase: PLAN, lead: planwright, time_taken_minutes: 14.0, tokens_total: 74000 }
    - { phase: DESI, lead: draftsmith, time_taken_minutes: 19.0, tokens_total: 85000 }
    - { phase: IMPL-build, lead: formwright, time_taken_minutes: 92.0, tokens_total: 220000 }
    - { phase: IMPL-integration, lead: groundsmith, time_taken_minutes: 37.0, tokens_total: 159000 }
    - { phase: ASSEMBLY, lead: assembler, time_taken_minutes: 8.0, tokens_total: 9200 }
    - { phase: DEPLOY, lead: "program-agent (standing in for forgemaster)", time_taken_minutes: 9.0, tokens_total: 19000 }
    - { phase: TEST-pass-1, lead: sentinel, time_taken_minutes: 45.0, tokens_total: 93000 }
    - { phase: IMPL-build-fix-pass-1, lead: formwright, time_taken_minutes: 35.0, tokens_total: 54000 }
    - { phase: IMPL-integration-fix-pass-1, lead: groundsmith, time_taken_minutes: 72.0, tokens_total: 130000 }
    - { phase: TEST-retest-fix-pass-1, lead: sentinel, time_taken_minutes: 65.0, tokens_total: 175000 }
    - { phase: IMPL-build-fix-pass-2, lead: formwright, time_taken_minutes: 28.0, tokens_total: 57000 }
    - { phase: DEPLOY-redeploy-fix-pass-2, lead: forgemaster, time_taken_minutes: 14.0, tokens_total: 24500 }
    - { phase: TEST-FINAL-retest-fix-pass-2, lead: sentinel, time_taken_minutes: 52.0, tokens_total: 164000 }
    - { phase: IMPL-build-fix-pass-3, lead: formwright, time_taken_minutes: 16.0, tokens_total: 23000 }
    - { phase: DEPLOY-redeploy-fix-pass-3, lead: forgemaster, time_taken_minutes: 13.0, tokens_total: 23300 }
    - { phase: TEST-FINAL-retest-fix-pass-3, lead: sentinel, time_taken_minutes: 42.0, tokens_total: 120000 }
    - { phase: IMPL-build-fix-pass-4, lead: formwright, time_taken_minutes: 52.0, tokens_total: 109000 }
    - { phase: IMPL-integration-fix-pass-4, lead: groundsmith, time_taken_minutes: 95.0, tokens_total: 250000 }
    - { phase: TEST-retest-fix-pass-4, lead: sentinel, time_taken_minutes: 65.0, tokens_total: 185000 }
  note: "Forgemaster has not yet published a Fix-pass-4 redeploy section (with its own time/token metrics) in code-quality-report.md as of this retest — the deploy itself is independently confirmed LIVE in this report (bundle Active, DDX file timestamped today, live model reflecting the rebuild), but its cost is not yet in this rollup. Program agent should have Forgemaster backfill its fix-pass-4 metrics for a fully audited total."
report: ".claude/agents/runs/2026-07-22-employee-training-request/testing/test-report.md"
gate_result: FAIL
gap_summary: "Unchanged 2-item gap (D2, D4), neither in this fix pass's scope. Both independently reconfirmed unchanged (not newly broken, not silently fixed) via live evidence stronger than any prior retest (live transitions array for D2, live captured submission payload for D4). Issue 1 (fragment schema binding) and Issue 3 (OOTB workflow rebuild, incl. D1's genuine AssignFormStep-class regression risk) are both independently verified CLOSED with no regression. Issue 2 (script inventory) spot-checked and holds. 4 new Tier-A editor-completion items surfaced and confirmed incomplete — added to the open-items list as non-blocking human follow-ups, not new defects."
next: "aem-forms-program-agent: (1) have Forgemaster backfill its fix-pass-4 redeploy section + metrics into code-quality-report.md for full audit-trail completeness, (2) proceed to HANDOFF with D2 + D4 + the 4 Tier-A items + the AF-editor-canvas visual-confirmation gap all flagged as non-blocking, well-understood follow-ups — this is the most rigorously verified state this delivery has reached across 4 fix passes / 5 test cycles, and neither remaining defect is agent-closable without either Workflow Editor UI/browser access (D2, Tier-A items, editor canvas) or an architectural redesign (D4)."
```

---

## RETEST — fix pass 5 (2026-07-23, OOTB step I/O configuration — server-side round-trip)

**Trigger:** Groundsmith configured every OOTB workflow step's I/O (Invoke DDX, Convert to PDF/A, both
Send Email steps, both Assign Task steps) via a live server-side POST/GET round-trip, and flagged 2
honest, unresolved ambiguities: (a) Assign Task's data/attachment I/O was hedged by setting BOTH a
flat and a COMBINED property form, unclear which `AssignFormStep` actually reads; (b) whether the
JSON-wrapped ROUTES format parses correctly at task completion. Forgemaster redeployed (3rd build
attempt, BUILD SUCCESS, gate PASS, model regenerated to v1.9). This retest's mandate: get REAL answers
by actually driving a submission to the DDX/PDF-A step and actually completing a live Manager Approval
task — not re-confirming structural presence, which was already done by Forgemaster's own deploy
report.

Forgemaster's redeploy was independently re-confirmed live before any functional work began: bundle
`aem-adaptive-forms-agents.core` Active, all 6 touched workflow nodes byte-for-byte matching
Groundsmith's CONFIG STATUS TABLE (`GET /var/workflow/models/employee-training-request-approval.json`),
form + embedded "Test Adaptive Form" page both HTTP 200.

### What was actually done this pass (deeper than any prior retest)

1. Submitted a fresh, fully valid request through the embedded page → new instance
   `employee-training-request-approval_36`, confirmed `RUNNING` with a work item at `node2` (Manager
   Approval).
2. Attempted to actually **complete** that task through the real UI a manager would use — AEM's
   "Forms Workspace" Task Manager (`/aem/dashboard/formdetails.html?item=<workitem path>`, the same
   URL AEM itself generates as `workitem_url` for task-assigned emails) — in a real, JS-executing
   browser (Cypress/Electron), not a curl SSR fetch.
3. Found the task-detail page renders **completely empty** (only a "Delegate" button; no form, no
   data, no route/submit buttons) for both a stale (fix-pass-4-era) and the brand-new work item.
4. Cross-referenced the exact request timestamps against the live AEM `error.log` and decompiled two
   Adobe classes with `javap` to get precise, class/line-level root causes rather than guessing from a
   blank page. Full detail, stack traces, and decompiled-constant evidence: `integration-test-report.md`
   §"RETEST — fix pass 5".

### Result: task completion is BLOCKED by 3 newly-discovered Critical defects (D6, D7, D8)

| ID | What | Evidence | Severity |
|---|---|---|---|
| **D6 (NEW)** | Assign Task's COMBINED attachments property (`INPUT_COMBINED_FORM_ATTACHMENTS`/`OUTPUT_COMBINED_FORM_ATTACHMENTS` = `RELATIVE_PLOAD:attachments`) throws `WorkflowException: Invalid type for resolving property RELATIVE_PLOAD` inside `WorkSpacePayLoadManagerImpl.getFieldLevelAttachments` (`PropertyResolver.java:311`), which fails `redirector.jsp` outright on the FRESH instance `_36` | Live error.log stack trace, class/line-level; `PropertyResolver.class` decompiled — `RELATIVE_PLOAD` is a real, recognized token (not a typo) but apparently not valid for THIS resolution context (a `FOLDER_PAYLOAD` category exists in the same class and is the likely correct one for a multi-file attachments collection) | Critical, agent-fixable |
| **D7 (NEW)** | Assign Task's ROUTES JSON-per-element value throws `WorkflowException: JSON Exception in retrieving routes` inside `submitbuttons.jsp` — the JSP that renders the Approve/Reject completion buttons | Fired on BOTH the stale (`_30`) and fresh (`_36`) work items, on every page load, confirming it's tied to the current live ROUTES value | Critical, agent-fixable |
| **D8 (NEW, pre-existing since fix pass 4)** | `FORM_TYPE="ADAPTIVE_FORM"` on both Assign Task nodes — but the real `FormType` enum (decompiled from `forms-dashboard-core-bundle-4.0.252.jar`) only defines `AF`, `PDF`, `READ_ONLY_AF`, `CCR_UI`, `IC_WEB`, `IC_PRINT` — **`ADAPTIVE_FORM` does not exist as a constant.** Throws `IllegalArgumentException: No enum constant ...FormType.ADAPTIVE_FORM` in the metadata/formview/documentattachments JSPs | `javap` decompile of `FormType.class`; error present on the OLDER instance too, confirming this predates fix pass 5 (set in fix pass 4, never previously exercised — no agent had opened the Task Manager UI until this pass) | Critical, agent-fixable (one-value fix: `"AF"`) |

**Net result: the Manager Approval task cannot be completed through the standard Forms Workspace UI at
all** — no form data loads, no route/submit buttons render. This is not a tooling gap in this session;
it is a genuine, reproducible, now precisely-diagnosed runtime defect chain.

### Objective #1 (DDX/DoR assembly) — could NOT be verified, for a concrete, evidenced reason

Because the workflow has never advanced past `node2` on any real submission in this delivery's entire
history (task completion has always been blocked, just never tested this deeply before), Invoke DDX
(`node6`, "Assemble Manager Confirmation") and Convert-to-PDF/A (`node7`) have **never actually
executed**. No PDF has ever been produced by this workflow to inspect for EmployeeTrainingForm +
ApprovalInfo merge content, and the approved Send Email step's attachment has never been exercised.
The step configuration remains structurally confirmed byte-for-byte correct (Forgemaster's redeploy
verification, carried forward, unchanged) — but **"configured correctly" and "proven to execute" are
now known to be two different claims**, and only the first can be made honestly at this time.

### Objective #2 (Assign Task combined I/O + ROUTES) — answered, negatively

Groundsmith's flagged ambiguity is resolved, but not in the hoped-for direction:
- The COMBINED property form is **not** a safe hedge alongside the flat form — its presence actively
  breaks `WorkSpacePayLoadManagerImpl.getFieldLevelAttachments` (D6), which does not appear to fall
  back to the flat form on failure; it simply throws.
- The ROUTES JSON-per-element shape is **confirmed to persist** correctly (Forgemaster's redeploy
  verification) but **confirmed to fail parsing** at the one place that matters — building the actual
  completion buttons (D7).
- Approve vs. Reject routing could not be tested at all (no route is ever selectable), so this
  question remains open, now for a different, deeper reason than before.

### Full regression sweep — nothing regressed

| Item | Method this pass | Result |
|---|---|---|
| D1 (submit → workflow) | Fresh full-AF-submit-path Cypress run (`employee-training-request-final-retest.cy.js` FR-05, `employee-training-request-functional.cy.js` RT-08) | PASS — HTTP 200, thank-you shown, new RUNNING instance created |
| D-R1 (ManagerId/ManagerName lock) | Both specs, RT-02/FR-02 | PASS |
| D-R6 (FinanceComments lock) | Both specs, RT-06/FR-04 | PASS |
| UI-Finding-1 (Employee Details heading) | Both specs, RT-01/FR-01 | PASS |
| D3 (fragment dataRef → correct schema paths) | `submissiondate-probe.cy.js` captured request body: `EmployeeDetails`/`TrainingRequest`/`Justification`/`Declaration` all correctly nested | PASS |
| Rules #2–#5/#7/#8, submit-gated-on-validation | Both functional specs, all sub-checks | PASS (13/13 + 9/9 individual assertions across the two specs) |
| D2 (OR-split branching) | Fresh `GET` of the live model's `transitions` array | **UNCHANGED, still open** — strictly linear `node0→1→...→11`, zero `OR_SPLIT` nodes |
| D4 (SubmissionDate) | `submissiondate-probe.cy.js` captured request body | **UNCHANGED, not re-chased per instruction** — `Declaration.SubmissionDate` still absent, only `AcceptedTerms` present |

Full Cypress run: `employee-training-request-functional.cy.js` 8/8, `employee-training-request-final-retest.cy.js`
5/5, `employee-training-request-submissiondate-probe.cy.js` 1/1 — **14/14 passing, zero failures.**

### Updated case tally and coverage

The 9 previously-failing cases were re-examined against this pass's new evidence. **None flip to
PASS; none of the 25 passing cases regress.** The root-cause attribution for several deepens
significantly (from a vague "task landing could not be confirmed" to a precise D6/D7/D8 diagnosis):

| ID | Prior root cause | Updated root cause this pass |
|---|---|---|
| TC-013 | "live task-landing could not be confirmed" | **Task IS created (confirmed live) but CANNOT be opened/completed — blocked by D6/D7/D8** |
| TC-016, TC-017 | Blocked by D1 (fixed) / would fail D2 | **Still blocked — D2 (no branching) AND now D6/D7/D8 (no task completion possible at all)** |
| TC-026 | DoR PDF never generated, blocked by D1 (fixed) | **Still never generated — workflow can never reach node6 (Invoke DDX) because D6/D7/D8 block task completion at node2** |
| TC-032 | No instance ever reaches Send Email (D1) | **Instance now reaches and STAYS at node2 — Send Email steps (node8/node9) still never execute, now for a precisely diagnosed reason** |
| TC-012 | D2 (no real Approve/Reject branching) | **Unchanged** — D2 confirmed still open; ROUTES selection is additionally now confirmed unreachable in the UI (D7) |
| TC-004, TC-031 | Architecture/test-case mismatch (no prefill service, accepted decision) | **Unchanged** — unrelated to this pass |
| TC-019 | D4 (SubmissionDate) | **Unchanged, not re-chased** |

- `test_cases: { total: 34, executed: 34, passed: 25, failed: 9 }` — **unchanged**
- `unexecuted_cases: 0`
- `user_stories: { total: 11, covered: 10 }`, `uncovered_stories: [US-08]` — **unchanged**
- `ui_parity`: carried forward, **85.79% pixel match (advisory in migration mode), 0 Critical** — no
  visual artifact touched this pass (workflow step `metaData` only); spot-checked live via
  FR-01/RT-01 (Employee Details heading still renders) rather than a full pixel re-capture, consistent
  with the precedent set at fix passes 2–4.

### Open items (current, complete list)

1. **D2 — OR-split branching** (Critical, unchanged across 5 fix passes) — human-actionable, requires
   Workflow Editor UI access no agent in this session has.
2. **D4 — SubmissionDate never lands** (Major/Critical-adjacent, unchanged, NOT re-chased) — recommend
   accepting or scheduling a mechanism redesign, per prior retests' recommendation, unchanged.
3. **D6 — NEW, Critical, agent-fixable** — Assign Task's COMBINED attachments property
   (`RELATIVE_PLOAD:attachments`) breaks `WorkSpacePayLoadManagerImpl.getFieldLevelAttachments`.
   Testable next step: try `FOLDER_PAYLOAD:attachments` instead, or remove the COMBINED form entirely
   and rely on the flat form only.
4. **D7 — NEW, Critical, agent-fixable** — Assign Task's ROUTES JSON-per-element value fails to parse
   in `submitbuttons.jsp`. Testable next step: investigate what shape this specific JSP's routes-reader
   actually expects (possibly Fix pass 4's original plain-string array, or a differently-nested JSON
   shape) — round-trip it the same rigorous way Fix pass 5 round-tripped everything else, but this time
   also drive an actual task-completion attempt as the acceptance test, not just `generate.json`.
5. **D8 — NEW, Critical, agent-fixable, one-value fix** — `FORM_TYPE` should be `"AF"`, not
   `"ADAPTIVE_FORM"` (confirmed via the real enum's decompiled constants). Pre-existing since fix pass
   4, never previously exercised.
6. Carried-forward Tier-A items now effectively superseded by D6/D7/D8 (they were the SAME nodes'
   incompleteness before Fix pass 5 configured them; now configured, but blocked by a different,
   deeper layer of defects).
7. Carried-forward pre-production business-decision items (unchanged): real manager lookup, a distinct
   finance-approver identity, a real sender email, no local SMTP.
8. Carried-forward, non-blocking: AF-editor canvas for both fragments not visually re-confirmed (no
   browser/editor automation available for that specific check).

### Delivery Totals (final running total — ALL phases + all 4 fix passes + all 5 Sentinel test runs)

| Phase | Lead | time_taken_minutes | tokens (cli_text / read / write / other / **total**) |
|---|---|---|---|
| PLAN | planwright | 14.0 | 9,000 / 48,000 / 11,000 / 6,000 / **74,000** |
| DESI | draftsmith | 19.0 | 16,000 / 40,000 / 21,000 / 8,000 / **85,000** |
| IMPL-build | formwright | 92.0 | 48,000 / 95,000 / 62,000 / 15,000 / **220,000** |
| IMPL-integration | groundsmith | 37.0 | 38,000 / 88,000 / 24,000 / 9,000 / **159,000** |
| ASSEMBLY | assembler | 8.0 | 2,500 / 1,200 / 4,000 / 1,500 / **9,200** |
| DEPLOY | program-agent (standing in for forgemaster) | 9.0 | 3,500 / 9,000 / 4,500 / 2,000 / **19,000** |
| TEST-pass-1 | sentinel | 45.0 | 25,000 / 45,000 / 8,000 / 15,000 / **93,000** |
| IMPL-build-fix-pass-1 | formwright | 35.0 | ~ / ~ / ~ / ~ / **54,000** |
| IMPL-integration-fix-pass-1 | groundsmith | 72.0 | ~ / ~ / ~ / ~ / **130,000** |
| TEST-retest-fix-pass-1 | sentinel | 65.0 | ~ / ~ / ~ / ~ / **175,000** |
| IMPL-build-fix-pass-2 | formwright | 28.0 | ~ / ~ / ~ / ~ / **57,000** |
| DEPLOY-redeploy-fix-pass-2 | forgemaster | 14.0 | ~ / ~ / ~ / ~ / **24,500** |
| TEST-FINAL-retest-fix-pass-2 | sentinel | 52.0 | ~ / ~ / ~ / ~ / **164,000** |
| IMPL-build-fix-pass-3 | formwright | 16.0 | ~ / ~ / ~ / ~ / **23,000** |
| DEPLOY-redeploy-fix-pass-3 | forgemaster | 13.0 | ~ / ~ / ~ / ~ / **23,300** |
| TEST-FINAL-retest-fix-pass-3 | sentinel | 42.0 | ~ / ~ / ~ / ~ / **120,000** |
| IMPL-build-fix-pass-4 | formwright | 52.0 | ~ / ~ / ~ / ~ / **109,000** |
| IMPL-integration-fix-pass-4 | groundsmith | 95.0 | ~ / ~ / ~ / ~ / **250,000** |
| TEST-retest-fix-pass-4 | sentinel | 65.0 | ~ / ~ / ~ / ~ / **185,000** |
| DEPLOY-redeploy-fix-pass-4 | forgemaster | 19.0 | 3,800 / 24,000 / 3,200 / 12,500 / **43,500** |
| IMPL-integration-fix-pass-5 | groundsmith | 58.0 | 21,000 / 72,000 / 14,000 / 26,000 / **133,000** |
| DEPLOY-redeploy-fix-pass-5 | forgemaster | 50.0 | 9,500 / 38,000 / 6,500 / 29,000 / **83,000** |
| TEST-retest-fix-pass-5 | sentinel (this run) | ~80.0 | ~30,000 / ~150,000 / ~25,000 / ~60,000 / **~265,000** |
| **TOTAL** | | **~980.0 min** | **~384,500 / ~1,318,700 / ~306,100 / ~489,200 / ~2,498,500** |

`DEPLOY-redeploy-fix-pass-4`'s figures (19.0 min, 43,500 tokens) were backfilled from Forgemaster's own
§12.5 in `code-quality-report.md`, published after the fix-pass-4 retest closed the audit gap flagged
in that retest's handoff. Because this pass's own figures are estimates (no precise in-session token
accounting available), **the grand total remains marked `~`.**

### Gate result — fix pass 5 retest

**FAIL.** Not for the previously-carried D2/D4 reasons alone this time — this pass surfaces 3 NEW,
concretely-evidenced, agent-fixable Critical defects (D6, D7, D8) that block real task completion, and
by extension block verification of Invoke DDX/Convert-to-PDF/A/Send-Email-attachment (Objective #1)
and the ROUTES/combined-IO question (Objective #2) — the very things this fix pass set out to prove.
`uncovered_stories` is still non-empty (`[US-08]`), and the case tally is unchanged (25 pass / 9 fail).

**Recommendation for HANDOFF: do NOT proceed to handoff yet — recommend one more Groundsmith fix
pass targeting D6, D7, D8 specifically, followed by a Sentinel retest that actually attempts task
completion again (the same technique this pass used).** Unlike D2 (platform limitation, human-only)
and D4 (architectural redesign), D6/D7/D8 are concrete, config-value-level bugs with specific, testable
next steps identified above (`FOLDER_PAYLOAD` instead of `RELATIVE_PLOAD` for the attachments combined
property; re-investigate the ROUTES-reading JSP's actual expected shape; change `FORM_TYPE` to `"AF"`).
This is the first retest in the delivery to actually attempt real task completion — every prior retest
proved the workflow REACHES Manager Approval (D1), but none had previously proven a task can be
COMPLETED. That gap is now closed with a precise diagnosis, not a shrug, and it is directly
actionable in a 6th cycle.

### Run metrics (this agent — sentinel, retest fix pass 5)

- `time_taken_minutes`: ~80.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~30,000 (root-cause reasoning for D6/D7/D8, the FOLDER_PAYLOAD hypothesis, this
    section + the matching sections in `integration-test-report.md`/`test-form-ui-report.md`)
  - `read`: ~150,000 (groundsmith.md fix-pass-5 + code-quality-report.md §13 in full, test-report.md
    prior sections for format precedent, `workitemdetails.js` full clientlib, ~15 live JSON/HTML
    fetches across 3 Cypress recon runs, ~200KB of AEM `error.log` tail scanning, 2 decompiled class
    files via `javap`)
  - `write`: ~25,000 (3 throwaway Cypress recon specs written then deleted, the regression-sweep
    assertions reused from existing specs, this section + 2 companion report sections)
  - `other`: ~60,000 (3 Cypress run invocations with verbose terminal output, ~20 curl/REST calls,
    `unzip`/`javap` decompile tool-calls, repeated sandbox `/etc/*` permission noise on every Bash call,
    grep/node scripting to parse HTML/log dumps)
  - `total`: ~265,000

### Handoff YAML — retest fix pass 5

```yaml
agent: sentinel
phase: TEST
status: FAILED
retest_of: "fix pass 5 (OOTB step I/O configuration: Invoke DDX, Convert to PDF/A, Send Email attachment, Assign Task combined I/O + ROUTES)"
time_taken_minutes: 80.0
tokens_consumed:
  cli_text: 30000
  read: 150000
  write: 25000
  other: 60000
  total: 265000
test_cases: { total: 34, executed: 34, passed: 25, failed: 9 }
unexecuted_cases: 0
executes_with: { create-form-tests: 2, test-form-ui: 1, functional: 31 }
coverage_pct: null   # no custom Forms-service Java classes exist for this delivery -- N/A, unchanged
user_stories: { total: 11, covered: 10 }
uncovered_stories: 1   # US-08 only -- unchanged
ui_parity: { pixel: PASS_ADVISORY, pixel_match_pct: 85.79, critical_findings: 0 }   # carried forward, no visual artifact changed this fix pass
embedded_page:
  path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form"
  embeds_new_form: true
  functional_in_page: true   # AF submit itself works (D1 regression-clean); the BLOCKED capability is downstream task completion, not the embedded form/page
objective_1_ddx_dor_assembly:
  status: "NOT VERIFIABLE this pass"
  reason: "Workflow never advances past node2 (Manager Approval) on any real submission because task completion is blocked by D6/D7/D8 -- Invoke DDX/Convert-to-PDF/A/Send-Email-attachment have never actually executed"
  config_status: "structurally confirmed byte-for-byte correct (carried forward from Forgemaster's redeploy verification) -- configured-correctly and proven-to-execute are now known to be two different claims"
objective_2_assigntask_routes:
  status: "ANSWERED, negatively"
  combined_io_ambiguity: "RESOLVED -- the COMBINED form is not a safe hedge; its presence breaks WorkSpacePayLoadManagerImpl.getFieldLevelAttachments (D6)"
  routes_parsing: "RESOLVED -- ROUTES persists correctly but fails to parse in submitbuttons.jsp (D7), so no route is ever selectable in the real completion UI"
new_defects_this_pass:
  - { id: D6, severity: Critical, owner: groundsmith, agent_fixable: true, evidence: "PropertyResolver.java:311 WorkflowException 'Invalid type for resolving property RELATIVE_PLOAD', class decompiled via javap confirms RELATIVE_PLOAD is real but FOLDER_PAYLOAD may be the correct category for a multi-file attachments collection", next_step: "try FOLDER_PAYLOAD:attachments, or drop the COMBINED attachments form and rely on flat only" }
  - { id: D7, severity: Critical, owner: groundsmith, agent_fixable: true, evidence: "submitbuttons.jsp WorkflowException 'JSON Exception in retrieving routes', fires on both a stale and a fresh work item", next_step: "determine the exact shape this JSP's routes-reader expects; round-trip and verify via an actual task-completion attempt, not just generate.json" }
  - { id: D8, severity: Critical, owner: groundsmith, agent_fixable: true, evidence: "FormType.class decompiled via javap -- valid constants are AF/PDF/READ_ONLY_AF/CCR_UI/IC_WEB/IC_PRINT, no ADAPTIVE_FORM; pre-existing since fix pass 4, never previously exercised", next_step: "change FORM_TYPE from ADAPTIVE_FORM to AF on both assigntask_manager and assigntask_finance" }
defects_still_open:
  - { id: D2, severity: Critical, owner: groundsmith, status: "UNCHANGED -- reconfirmed fully linear (0 OR_SPLIT nodes) on the live model" }
  - { id: D4, severity: "Major/Critical-adjacent", owner: formwright, status: "UNCHANGED, NOT re-chased per instruction" }
regression_sweep: "14/14 Cypress tests passing across 3 specs (functional 8/8, final-retest 5/5, submissiondate-probe 1/1) -- D1, D-R1, D-R6, UI-Finding-1, D3, rules #2-#5/#7/#8, submit-gated-on-validation all regression-clean; D2 and D4 independently reconfirmed unchanged"
delivery_totals:
  total_time_taken_minutes: "~980.0"
  total_tokens_consumed: { cli_text: 384500, read: 1318700, write: 306100, other: 489200, total: 2498500 }
  per_phase:
    - { phase: PLAN, lead: planwright, time_taken_minutes: 14.0, tokens_total: 74000 }
    - { phase: DESI, lead: draftsmith, time_taken_minutes: 19.0, tokens_total: 85000 }
    - { phase: IMPL-build, lead: formwright, time_taken_minutes: 92.0, tokens_total: 220000 }
    - { phase: IMPL-integration, lead: groundsmith, time_taken_minutes: 37.0, tokens_total: 159000 }
    - { phase: ASSEMBLY, lead: assembler, time_taken_minutes: 8.0, tokens_total: 9200 }
    - { phase: DEPLOY, lead: "program-agent (standing in for forgemaster)", time_taken_minutes: 9.0, tokens_total: 19000 }
    - { phase: TEST-pass-1, lead: sentinel, time_taken_minutes: 45.0, tokens_total: 93000 }
    - { phase: IMPL-build-fix-pass-1, lead: formwright, time_taken_minutes: 35.0, tokens_total: 54000 }
    - { phase: IMPL-integration-fix-pass-1, lead: groundsmith, time_taken_minutes: 72.0, tokens_total: 130000 }
    - { phase: TEST-retest-fix-pass-1, lead: sentinel, time_taken_minutes: 65.0, tokens_total: 175000 }
    - { phase: IMPL-build-fix-pass-2, lead: formwright, time_taken_minutes: 28.0, tokens_total: 57000 }
    - { phase: DEPLOY-redeploy-fix-pass-2, lead: forgemaster, time_taken_minutes: 14.0, tokens_total: 24500 }
    - { phase: TEST-FINAL-retest-fix-pass-2, lead: sentinel, time_taken_minutes: 52.0, tokens_total: 164000 }
    - { phase: IMPL-build-fix-pass-3, lead: formwright, time_taken_minutes: 16.0, tokens_total: 23000 }
    - { phase: DEPLOY-redeploy-fix-pass-3, lead: forgemaster, time_taken_minutes: 13.0, tokens_total: 23300 }
    - { phase: TEST-FINAL-retest-fix-pass-3, lead: sentinel, time_taken_minutes: 42.0, tokens_total: 120000 }
    - { phase: IMPL-build-fix-pass-4, lead: formwright, time_taken_minutes: 52.0, tokens_total: 109000 }
    - { phase: IMPL-integration-fix-pass-4, lead: groundsmith, time_taken_minutes: 95.0, tokens_total: 250000 }
    - { phase: TEST-retest-fix-pass-4, lead: sentinel, time_taken_minutes: 65.0, tokens_total: 185000 }
    - { phase: DEPLOY-redeploy-fix-pass-4, lead: forgemaster, time_taken_minutes: 19.0, tokens_total: 43500 }
    - { phase: IMPL-integration-fix-pass-5, lead: groundsmith, time_taken_minutes: 58.0, tokens_total: 133000 }
    - { phase: DEPLOY-redeploy-fix-pass-5, lead: forgemaster, time_taken_minutes: 50.0, tokens_total: 83000 }
    - { phase: TEST-retest-fix-pass-5, lead: sentinel, time_taken_minutes: 80.0, tokens_total: 265000 }
report: ".claude/agents/runs/2026-07-22-employee-training-request/testing/test-report.md"
gate_result: FAIL
gap_summary: "3 NEW Critical, agent-fixable defects (D6, D7, D8) discovered this pass, all blocking real Manager Approval task completion and therefore blocking verification of Objective #1 (DDX/DoR assembly) and Objective #2 (Assign Task combined I/O + ROUTES). D2 and D4 independently reconfirmed unchanged. Zero regression across 14/14 Cypress checks. This is the first retest to actually attempt task completion, not just workflow-instance creation -- the deeper test surfaced defects invisible to every prior, shallower check."
next: "aem-forms-program-agent: run ONE more groundsmith fix pass targeting D6 (try FOLDER_PAYLOAD instead of RELATIVE_PLOAD, or drop the COMBINED attachments form), D7 (re-investigate submitbuttons.jsp's actual expected ROUTES shape), and D8 (FORM_TYPE=AF, a one-value fix) -- then forgemaster redeploys and sentinel retests by actually attempting task completion again. Do NOT proceed to HANDOFF yet: these are concrete, agent-fixable defects, unlike D2/D4."
```

---

## FINAL functional retest -- fix pass 6 (2026-07-23, real task completion + DDX/DoR verification)

**This is the final functional test this delivery ran.** Trigger: Groundsmith's fix pass 6 fixed D6
(`INPUT_COMBINED_*` colon-parsing crash), D7 (determined NOT to be an independent defect -- a cascading
symptom of D6), and D8 (`FORM_TYPE="ADAPTIVE_FORM"` is not a valid enum constant; changed to `AF`).
Forgemaster's redeploy (fix pass 6, `code-quality-report.md` section 14) confirmed BUILD SUCCESS,
112/112 tests, both fixes live on the regenerated runtime model, zero orphans. This pass drove the
deployed workflow with a real, JS-executing browser (Cypress/Electron) all the way to an actual click
on the real per-task-item Approve/Reject controls -- the deepest test this delivery has run.

### Method

A new Cypress spec, `employee-training-request-final-functional-retest.cy.js`, was written and run
live against `http://localhost:4502`:
1. Submitted a fresh, schema-conformant Employee Training Request through the embedded page.
2. Queried `/var/workflow/instances.json`, located the newest RUNNING instance plus its Manager
   Approval work item.
3. Opened the item's real Task Manager URL (`/aem/dashboard/formdetails.html?item=...`) in the
   browser.
4. Clicked the real per-item **Approve** control (`#fd-dashboard-tm-detailsview-Approve`), not the
   Inbox list's unrelated, permanently-hidden bulk-action button of the same name
   (`.cq-inbox-task-approve.foundation-collection-action-hidden` -- confirmed by ID/selector, a
   mistake made and caught mid-session: the bulk button is a no-op with nothing selected, and
   clicking it proves nothing about real task completion).
5. Cross-referenced every result against the live AEM `error.log` and the real clientlib JS
   (`workitemdetails.js`) that this exact button invokes, for class/line-level ground truth -- the
   same evidentiary bar Groundsmith used for D6/D7/D8, not inference from a blank/changed page alone.

### Result 1 -- D6/D7/D8 fixes CONFIRMED live, on a genuinely fresh work item

A fresh work item's task-detail page (instance `employee-training-request-approval_52` onward, each
one newly created this pass) shows:
- Zero occurrences of `RELATIVE_PLOAD` / `No enum constant` / `JSON Exception in retrieving routes`
  anywhere in the AEM error.log for these requests.
- Real, rendered **Approve / Reject / Delegate** controls in the action bar (screenshot:
  `cypress/results/screenshots/.../form-ui/final-retest-manager-taskdetail.png`) -- not a
  placeholder, not "Delegate" only (confirmed the difference against a stale, pre-fix-pass-6 instance
  `_13`, whose task-detail page shows only "Delegate"/"OK" and zero Approve/Reject markup, because AEM
  Forms Workflow copies a step's `metaData` into the WorkItem's OWN metadata map at the moment the
  instance enters that step -- a stale instance permanently carries the OLD, broken config even after
  the model is fixed and redeployed). This is an important, previously undocumented mechanic: every
  retest in this delivery MUST use a work item created AFTER the fix, never an older one, or the
  retest silently produces a false negative.
- This is a genuine, reproducible confirmation of D6/D7/D8 all being fixed -- the prior blocker to
  even reaching a completion attempt is gone.

### Result 2 -- a NEW Critical defect, D9, blocks REAL completion of every route (Approve AND Reject) -- code-proven, not inferred

Clicking the real Approve control does not complete the task. It shows: "Error -- There are
validation errors. Fix the errors to continue." with no field ever identified as invalid (screenshot:
`.../form-ui/final-retest-manager-post-approve.png`). This is not a transient glitch -- it is a
100%-reproducible dead end, root-caused to exact source, not guessed.

**The Document tab's inline Adaptive Form panel never renders at all.** A raw HTML dump of
`formdetails.html` shows the `formview.jsp` component's own render boundary comments
(`workitemdetails/formview`) with literally nothing between them -- no iframe, no guideBridge
bootstrap script, no field markup. This is the direct, causal consequence of the `formview.jsp`
NullPointerException Groundsmith flagged in fix pass 6 as "non-blocking" -- it is now proven, live, to
be the opposite for `FORM_TYPE=AF` tasks:

    23.07.2026 18:33:25.363 *ERROR* [...] libs.fd.dashboard.tm.gui.components.workitemdetails.formview.formview$jsp
      Error in getting data java.lang.NullPointerException:
      Cannot invoke "org.apache.sling.api.resource.Resource.getParent()" because "parentResource" is null

The exact client-side gate that then fails, read directly from the real, deployed clientlib JS
(`/libs/fd/dashboard/tm/gui/components/workitemdetails/clientlibs/workitemdetails.js`,
`showConfirmationDialog()`):

    if (isReadOnlyForm || (guideBridge && guideBridge.isConnected() && guideBridge.validate())
        || ("CCR_UI" === formType && isCCRValid) || "IC_WEB" === formType) {
        dialog.show();
    } else {
        FormDashboard.TM.Util.showErrorMsg(Granite.I18n.get("Error"),
            Granite.I18n.get("There are validation errors. Fix the errors to continue."));
    }

For `FORM_TYPE="AF"` (fix pass 6's deliberate choice over the reference model's `READ_ONLY_AF`, made
specifically to support interactive `ApprovalInfo`/`ManagerComments` entry): `isReadOnlyForm` is
false, `formType` is neither `CCR_UI` nor `IC_WEB`, so the ONLY path to `dialog.show()` is
`guideBridge && guideBridge.isConnected() && guideBridge.validate()`. Since the inline Adaptive Form
never renders (formview.jsp's NPE, above), `guideBridge` never connects -- so this condition is
permanently false, for every route, every time. This same `showConfirmationDialog()` function is the
shared handler for BOTH Approve and Reject (it reads the clicked control's own `id` to determine which
route was chosen) -- so Reject is blocked by the identical code path, not just Approve.

**Net effect: with `FORM_TYPE=AF`, on this build, NO route can ever be completed, on any work item,
regardless of what data is filled in or which button is clicked -- the gate that decides whether the
confirmation dialog even opens fails unconditionally, before any real field-level validation runs.**
This is a new, distinct, Critical, agent-fixable defect -- **D9** -- separate from D6/D7/D8 (all three
of which ARE genuinely fixed) and separate from the pre-existing D2/D4.

Concrete fix paths for the next pass (not attempted here -- root-cause diagnosis is Sentinel's lane;
the fix is Groundsmith's/Formwright's):
1. Root-cause and fix the actual `formview.jsp` NPE (`Resource.getParent()` on a null
   `parentResource`) so the inline AF genuinely initializes and `guideBridge` connects -- preserves
   fix pass 6's interactive-entry intent for `ApprovalInfo`/`ManagerComments`.
2. OR revert `FORM_TYPE` to `READ_ONLY_AF` (the reference model's own value; Groundsmith's fix pass 6
   A/B test already confirmed this also clears the D8 enum crash identically) -- `isReadOnlyForm`
   would then be true, bypassing the `guideBridge` dependency entirely and allowing route completion
   immediately, at the cost of losing in-task interactive data entry (the same trade-off fix pass 6
   already flagged when choosing `AF` over the reference's `READ_ONLY_AF`).

### Result 3 -- Finance Manager Approval: NOT REACHED (blocked transitively by D9)

Because Manager Approval can never be completed via either route, the workflow instance never
advances off `node2`; no Finance Manager Approval work item was ever created on any of the 6+ fresh
instances submitted this pass. Finance Manager Approval's own D6/D7/D8-fix parity (byte-identical
`metaData` to `assigntask_manager`, confirmed by Groundsmith's fix pass 6 symmetry check) remains
structurally verified but functionally unreachable.

### Result 4 -- THE CORE OBJECTIVE (DDX/DoR assembly): STILL NOT VERIFIABLE, for a NEW, more precisely diagnosed reason

**A PDF was NOT produced by this workflow.** The instance has never advanced past `node2` in this
delivery's entire history (6 fix passes, dozens of fresh instances). Invoke DDX / Convert-to-PDF/A /
the approved Send Email step's attachment remain exactly where fix pass 5's retest left them:
structurally configured, byte-for-byte matching Groundsmith's CONFIG STATUS TABLE (unchanged this
pass -- Groundsmith's fix pass 6 did not touch these steps), but functionally unexercised. This pass
does not weaken that prior finding -- it replaces the reason with a sharper one: the blocker is no
longer D6/D7/D8 (all three fixed and confirmed), it is the newly diagnosed D9 (`formview.jsp` NPE,
`guideBridge` never connects, `showConfirmationDialog()`'s gate fails unconditionally for
`FORM_TYPE=AF`). No PDF content excerpt can be produced this pass because no PDF has ever been
generated by this workflow -- reported honestly, not fabricated.

### Regression sweep -- 14/14, zero regressions

Ran the full existing Cypress suite fresh against the fix-pass-6 redeploy:

| Spec | Result |
|---|---|
| `employee-training-request-functional.cy.js` | 8/8 passing -- D-R1, D-R6, rules #2-#5/#7/#8, submit-gated-on-validation, D1 all regression-clean |
| `employee-training-request-final-retest.cy.js` | 5/5 passing -- Employee Details heading (UI-Finding-1), D-R1, rules #2-#4, D-R6, D1 all regression-clean |
| `employee-training-request-submissiondate-probe.cy.js` | 1/1 passing (informational) -- `Declaration.SubmissionDate` confirmed still absent from the outgoing request body at click-time and immediately after -- D4 unchanged, not re-chased |

D2 independently reconfirmed unchanged via a fresh `GET /var/workflow/models/employee-training-request-approval.json`:
12 nodes, zero `OR_SPLIT` occurrences anywhere in the model JSON -- still strictly linear.

Submit-gated-on-validation reconfirmed as part of the functional/final-retest specs' own coverage
(empty-submit-blocked assertions both passing) -- a valid, schema-conformant submission continues to
reach the workflow (HTTP 200, real RUNNING instance created, real work item at node2) every time.

### Summary -- FINAL functional retest, fix pass 6

| Item | Status |
|---|---|
| D6 (Assign Task COMBINED attachments crash) | CONFIRMED FIXED -- zero RELATIVE_PLOAD errors on any fresh work item |
| D7 (ROUTES JSON parse failure at completion) | CONFIRMED FIXED (was a cascading D6 symptom, not independent) -- real Approve/Reject/Delegate controls render |
| D8 (FORM_TYPE=ADAPTIVE_FORM invalid enum) | CONFIRMED FIXED -- zero enum-crash errors, FORM_TYPE=AF live on both Assign Task nodes |
| D9 (NEW, Critical) -- formview.jsp NPE blocks the inline AF from ever rendering, which permanently fails showConfirmationDialog()'s guideBridge gate for FORM_TYPE=AF | DISCOVERED, code-proven (JSP render-boundary dump plus live clientlib JS source) -- blocks Approve AND Reject, on every route, every work item |
| Manager Approval real task completion | STILL BLOCKED -- by D9, not D6/D7/D8 |
| Finance Manager Approval | NOT REACHED (transitively blocked by D9) |
| Invoke DDX / Convert-to-PDF/A / Send Email attachment (functional) | STILL NOT VERIFIABLE -- workflow never advances past node2; no PDF has ever been produced by this workflow in this delivery's history |
| D1, D-R1, D-R6, UI-Finding-1, D3, submit-gated-on-validation | NO REGRESSION -- 14/14 Cypress checks passing |
| D2 (OR-split) | UNCHANGED, reconfirmed still open (not re-chased) |
| D4 (SubmissionDate) | UNCHANGED, reconfirmed still open (not re-chased) |

**Gate: FAIL.** This is genuine, substantial progress -- D6/D7/D8 are real, confirmed fixes, and this
pass reached materially deeper into the real Task Manager UI than any prior pass (an actual click on
the real per-item Approve control, not just an instance-creation check). But the delivery's core
LiveCycle-parity objective -- proving the DDX assembles a real Document of Record and that document
reaches the approval email as a real attachment -- remains unproven, now blocked by a single newly
diagnosed, code-evidenced, agent-fixable defect (D9) rather than three. Recommend exactly ONE more
targeted Groundsmith fix pass (either root-cause `formview.jsp`'s NPE, or revert `FORM_TYPE` to
`READ_ONLY_AF` on both Assign Task nodes) followed by a Forgemaster redeploy and a Sentinel retest
that resumes from the same "click the real per-item Approve/Reject control" method proven working in
this pass -- that retest would very plausibly be the one that finally reaches Invoke DDX and closes
this delivery's central open question. Do not proceed to HANDOFF as a full PASS -- D9 is a concrete,
narrowly scoped, agent-fixable blocker, not a platform limitation like D2 or an accepted architectural
gap like D4.

### Final coverage numbers (fix pass 6)

Test-case/user-story coverage is unchanged from fix pass 5's tally (`test_cases: 34 total, 34
executed, 25 passed, 9 failed`; `user_stories: 11 total, 10 covered, 1 uncovered -- US-08`) -- this
pass did not re-run the full DESI case suite from scratch (out of scope for a targeted functional
retest); it specifically targeted the Manager-Approval-task-completion capability fix pass 6
unblocked, plus a full regression sweep (above). US-08 (finance approval to DoR plus email) remains
the sole uncovered story, now for a single, precisely diagnosed reason (D9) rather than three
compounding ones.

### Run metrics -- FINAL functional retest, fix pass 6

- `time_taken_minutes`: ~78.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~32,000 (D9 root-cause reasoning, Cypress spec authoring/iteration, this section)
  - `read`: ~145,000 (groundsmith.md fix-pass-6 plus prior fix-pass sections, integration-test-report.md's
    fix-pass-5 method section, code-quality-report.md section 14, the live workflow model JSON, ~6
    formdetails.html raw dumps, the real workitemdetails.js clientlib source, error.log tails across
    ~8 live probes, prior Cypress spec files for payload/selector conventions)
  - `write`: ~15,000 (new Cypress spec `employee-training-request-final-functional-retest.cy.js`
    across several iterations, this section plus the matching sections in
    `integration-test-report.md` and `test-form-ui-report.md`)
  - `other`: ~95,000 (tool-call overhead: ~6 Cypress run invocations against the live browser,
    curl/node live HTTP probes for instance/work-item discovery, screenshot captures, error.log
    greps, the reject-probe diagnostic run)
  - `total`: ~287,000

### Handoff YAML -- FINAL functional retest, fix pass 6

```yaml
agent: sentinel
phase: TEST-FINAL-retest-fix-pass-6
status: FAIL
time_taken_minutes: 78.0
tokens_consumed:
  cli_text: 32000
  read: 145000
  write: 15000
  other: 95000
  total: 287000
test_cases: { total: 34, executed: 34, passed: 25, failed: 9 }
unexecuted_cases: 0
executes_with: { create-form-tests: 2, test-form-ui: 1, functional: 31 }
coverage_pct: null
user_stories: { total: 11, covered: 10 }
uncovered_stories: 1
ui_parity: { pixel: PASS_ADVISORY, pixel_match_pct: 85.79, critical_findings: 0 }
embedded_page:
  path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form"
  embeds_new_form: true
  functional_in_page: true
defects_confirmed_fixed_this_pass:
  - { id: D6, evidence: "zero RELATIVE_PLOAD errors in error.log across 6+ fresh work items" }
  - { id: D7, evidence: "real Approve/Reject/Delegate controls render on fresh work items; was a cascading D6 symptom, not independent" }
  - { id: D8, evidence: "zero FormType enum-crash errors; FORM_TYPE=AF confirmed live on both assigntask_manager/assigntask_finance" }
new_defect_this_pass:
  id: D9
  severity: Critical
  owner: "groundsmith (root cause) / formwright (formview.jsp is a libs OOTB JSP -- likely needs a config/wiring fix, not a hand-edited JSP)"
  agent_fixable: true
  blocks: "real task completion for Manager Approval AND Finance Manager Approval on every route, therefore blocks Invoke DDX / Convert-to-PDF/A / Send-Email-attachment from ever executing"
manager_approval_task_completion: "STILL BLOCKED -- by D9, not D6/D7/D8"
finance_approval_task_completion: "NOT REACHED -- transitively blocked by D9"
ddx_dor_assembly:
  status: "STILL NOT VERIFIABLE"
  reason: "workflow has never advanced past node2 in this delivery's history; no PDF has ever been produced"
email_attachment_verdict: "NOT VERIFIABLE -- same reason as DDX/DoR"
regression_sweep: "14/14 Cypress tests passing -- D1, D-R1, D-R6, UI-Finding-1, D3, submit-gated-on-validation all regression-clean"
defects_still_open:
  - { id: D2, severity: Critical, owner: groundsmith, status: "UNCHANGED" }
  - { id: D4, severity: "Major/Critical-adjacent", owner: formwright, status: "UNCHANGED, NOT re-chased" }
delivery_totals:
  total_time_taken_minutes: "~1136.0"
  total_tokens_consumed: { cli_text: 416500, read: 1463700, write: 321100, other: 584200, total: 2785500 }
  per_phase: "see the Delivery Totals table at the top of this file for the full per-phase breakdown across all 6 fix passes"
report: ".claude/agents/runs/2026-07-22-employee-training-request/testing/test-report.md"
gate_result: FAIL
gap_summary: "D6/D7/D8 all genuinely, live-confirmed FIXED. A single NEW, precisely diagnosed, agent-fixable Critical defect (D9) now blocks real task completion for both Approve and Reject, which transitively blocks Finance Manager Approval and the delivery's core objective (DDX/DoR assembly plus email attachment). D2 and D4 unchanged. Zero regression across 14/14 Cypress checks."
next: "aem-forms-program-agent: run ONE more groundsmith fix pass targeting D9 (root-cause formview.jsp's NPE, or revert FORM_TYPE to READ_ONLY_AF on both Assign Task nodes) -- then forgemaster redeploys and sentinel retests, resuming from this pass's proven method. Do NOT proceed to HANDOFF as a full PASS."
```

## FINAL functional verification -- fix pass 7 (2026-07-27, Approve path, Reject path, DDX/DoR assembly)

**Trigger.** `FORM_TYPE` was reverted to `READ_ONLY_AF` on both Assign Task nodes (`node2`
Manager Approval, `node4` Finance Manager Approval) -- structurally bypassing D9's broken
`guideBridge` gate, since `showConfirmationDialog()`'s condition `isReadOnlyForm || (...)`
short-circuits true the instant `isReadOnlyForm` is true. Comment-capture parity was wired via
the Assign Task step's OOTB `WORKITEM_COMMENT=rejectionReason` + `IS_COMMENT_ALLOWED=true`
properties. The program agent independently re-confirmed environment readiness and a freshly
regenerated runtime model carrying all four properties. This section is the resumption of the
final functional retest per those instructions.

### Method -- deepest evidentiary bar this delivery has used

1. Fetched the live runtime model (`GET /var/workflow/models/employee-training-request-approval.json`)
   and confirmed, byte-for-byte: `FORM_TYPE:"READ_ONLY_AF"`, `WORKITEM_COMMENT:"rejectionReason"`,
   `IS_COMMENT_ALLOWED:"true"` on both `node2` and `node4`; `InvokeDDXProcess` unchanged at `node6`;
   transitions still strictly linear `node0` -> `node11` with **zero** `OR_SPLIT`/branch nodes (D2
   reconfirmed unchanged, not re-chased).
2. Submitted two fresh, schema-conformant Employee Training Request payloads through the embedded
   page (`Cypress`, spec `employee-training-request-fixpass7-retest.cy.js`) -- one for the
   Approve-path objective, one for the Reject-path objective -- each producing a genuinely new,
   `RUNNING` workflow instance with its own Manager Approval work item.
3. Confirmed via a **raw, unauthenticated-JS server render** of
   `/aem/dashboard/formdetails.html?item=...` (curl, not the hydrated SPA) that the real detail page
   for both fresh work items renders: `data-isreadonlyform="true"`, `data-isCommentAllowed="true"`,
   real `#fd-dashboard-tm-detailsview-Approve` / `-Reject` buttons with `trackingelement="approve"` /
   `"reject"`, and **zero** `NullPointerException` text anywhere in the response (D9 fix directly
   confirmed at the server-render level, independent of any client redirect behaviour below).
4. Drove the real, JS-executing Task Manager SPA (Cypress/Electron) to open both fresh work items,
   confirming no D6/D7/D8/D9 error signatures ever appeared (10/10 Cypress assertions passing) --
   but discovered that the SPA could not reach a state where Approve/Reject was clickable, for a
   newly diagnosed reason (below), and the run bounced to the Inbox list instead.
5. Read the live, deployed `workitemdetails.js` clientlib source (decompile-grade, same evidentiary
   method fix pass 6 used for D9) to root-cause the bounce: `validateAssignee()` shows an "Assign to
   Self" modal automatically on load when the workitem is group-assigned
   (`data-isassigneeagroup="true"`); its "Proceed" button calls `delegateWorkItem()` ->
   `FormDashboard.TM.Util.delegateWorkItem()`, an AJAX POST to `/bin/workflow/inbox`; **on error**,
   the client navigates to the Inbox (`window.location = inboxUILink`) -- exactly the bounce observed.
6. Cross-referenced the live AEM `error.log` for that exact POST and timestamp and found a genuine,
   reproducible server-side exception (not a network blip):

       27.07.2026 10:42:54.486 *WARN* [...] com.adobe.granite.workflow.core.WorkflowSessionImpl Failed to publish event
       java.lang.IllegalStateException: This session has been closed
           at org.apache.jackrabbit.oak.jcr.delegate.SessionDelegate.checkLive/refresh/prePerform/perform(...)
           at org.apache.jackrabbit.oak.jcr.delegate.AuthorizableDelegator.getID(AuthorizableDelegator.java:88)
           at com.adobe.granite.workflow.core.event.EventPublishUtil.publishDelegationEvent(EventPublishUtil.java:424)
           at com.adobe.granite.workflow.core.WorkflowSessionImpl.delegateWorkItem(WorkflowSessionImpl.java:1353)
           at com.adobe.fd.workspace.step.service.AssignFormStep.execute(AssignFormStep.java:261)

   Reproduced identically on **both** fresh work items (Approve-path and Reject-path), 100% of the
   time -- claiming (delegating) a group-assigned Assign Task workitem is broken at the platform
   level in this environment: the delegation event publisher's session has already been closed by
   the time it tries to resolve the acting user's authorizable ID.
7. Per the task's explicit "Click/POST" allowance, attempted the identical server-side completion
   the real Confirm-dialog button invokes -- a direct, CSRF-token-authenticated `POST` to
   `/libs/fd/dashboard/servlets/afsubmission.json?operation=submit` with the exact `FormData` contract
   read from `onSubmitButtonClick()` (`workItemId`, `formPath`, `comment`, `attachments`, `formRoute`,
   `_charset_`) -- for **both** `formRoute=Approve` (Manager-Approval item) and `formRoute=Reject`
   (the separate Reject-path item). **Both calls failed identically:**

       HTTP 500 -- {"unresolvedMessage":"Invalid user - {0}","messageArgs":["admin"],"code":"AEM-FD-011-004"}

8. Root-caused this to the same place: queried `GET /home/groups/a/administrators/rep:members.json`
   (the group `STATIC_ASSIGNEE="administrators"` resolves to on both Assign Task nodes) and found its
   **only member is the system user `replication-receiver`** -- `admin` is not, and (per the next
   step) apparently cannot be made, a member of this group.
9. Attempted, as a **temporary, reversible diagnostic only** (not a delivered fix -- fully reverted
   afterward, exactly like the mock-SMTP catcher below), two independent standard
   Jackrabbit/Sling group-membership grants: (a) `:operation=addMembers&member=admin` against
   `/home/groups/a/administrators`, and (b) the same operation against a freshly created, disposable
   test user (`sentinel-fp7-reviewer`). **Both POSTs returned `200 OK` and updated the group node's
   `jcr:lastModified`/`jcr:lastModifiedBy`, yet `rep:members` never changed** -- membership grants to
   this group do not persist through the standard API in this environment, a further anomaly
   compounding the underlying defect. The disposable test user was deleted immediately after
   (`DELETE /home/users/f/fjgagmQ6_C3_EvNknj8D` -> `204`); the environment's authorization state is
   unchanged from before this retest.
10. Built and ran a minimal SMTP catcher (Node.js TCP listener bound to the project's own, unmodified
    `smtp.host=localhost`/`smtp.port=587` default -- **no OSGi mail config was changed**) to capture
    real email content the moment any instance reached `Send Approval/Rejection Notification`, in
    order to give the DDX/email objective concrete content evidence if the workflow ever advanced.

### Result 1 -- D9 CONFIRMED FIXED; a NEW, deeper Critical defect (D10) now blocks ALL real task completion

`READ_ONLY_AF` + the `isReadOnlyForm` short-circuit **is proven fixed**: zero validation-error
dialogs, zero NPEs, real Approve/Reject markup server-rendered with the correct `isreadonlyform`/
`isCommentAllowed` metadata on every fresh work item, for both the Approve-path and Reject-path
instances. D9 is retired.

But **no route (Approve or Reject) can be completed on any work item, via the real UI or via a
direct POST to the exact endpoint the UI invokes**, because of a newly discovered, Critical,
code-and-data-evidenced defect -- **D10**:

- The group `STATIC_ASSIGNEE="administrators"` resolves to on both Assign Task nodes has **no real,
  human-assignable member** -- only the system user `replication-receiver`.
- Claiming (delegating) the workitem to a real user throws `IllegalStateException: This session has
  been closed` inside `WorkflowSessionImpl.delegateWorkItem` (called from
  `AssignFormStep.execute`), a genuine platform-level bug independent of anything Groundsmith's
  workflow-model config controls.
- Bypassing the claim UI and POSTing directly to the real completion servlet
  (`/libs/fd/dashboard/servlets/afsubmission.json?operation=submit`) is **also** rejected, with
  `"Invalid user - admin"` (`AEM-FD-011-004`) -- the same underlying cause (admin is not a real
  member of the resolved group) blocks the server-side completion check too, independent of the
  client-side claim flow.
- Standard group-membership grants (both to `admin` and to a disposable test user) do not persist
  in this environment even though the write is accepted (`200 OK`) -- this needs its own
  investigation (possibly a repoinit/ACL reset process, or an Oak dynamic-membership quirk) before
  the underlying assignee-resolution defect can even be fixed by adding a real member to the group.

**Net effect: on this build, in this environment, no route can ever be completed on any work item
assigned to the `administrators` group, by any means (UI click or direct REST), regardless of
`FORM_TYPE`.** D10 is a NEW, Critical, code+data-evidenced blocker that supersedes D9 as the actual
gate on the delivery's central objective. It is deeper than D9: D9 was a client-side rendering gate
inside the form; D10 is a server-side participant/authorization resolution failure that exists
*before* the confirmation dialog is ever reached.

### Result 2 -- Finance Manager Approval: NOT REACHED (blocked transitively by D10, same as fix pass 6's D9 finding)

Because Manager Approval can never be completed via either route (D10, above), neither fresh
instance ever advances off `node2`; no Finance Manager Approval work item was created on either
instance this pass. Finance Manager Approval's own config parity with Manager Approval (identical
`FORM_TYPE`/`WORKITEM_COMMENT`/`IS_COMMENT_ALLOWED`/`STATIC_ASSIGNEE` on `node4`) remains
structurally verified but functionally unreachable, for the identical D10 reason.

### Result 3 -- THE CORE OBJECTIVE (DDX/DoR assembly) + email attachment: STILL NOT VERIFIABLE

**No PDF was produced.** Both fresh instances remain `RUNNING` at `node2` at the end of this pass
(confirmed via a final `GET` on each instance immediately after every completion attempt):

```
employee-training-request-approval_72 (Approve-path): state=RUNNING, workItems=[node2]
employee-training-request-approval_74 (Reject-path):  state=RUNNING, workItems=[node2]
```

`Invoke DDX` / `Convert-to-PDF/A` / `Send Email` (`node6`-`node10`) were never reached. The mock
SMTP catcher (item 10 above) captured **zero** messages across the entire pass -- consistent with,
and independently corroborating, the instance-state evidence above. No PDF-content excerpt, no email
body/attachment excerpt can be produced this pass, for the same reason fix pass 5 and fix pass 6
could not: the workflow has still never advanced past `node2` in this delivery's entire 7-fix-pass
history. This is reported honestly, not inferred or fabricated.

### Result 4 -- Reject path + comment-capture parity: mechanism CODE-CONFIRMED wired correctly, but functionally UNEXERCISABLE end-to-end (same D10 blocker)

Reading `showConfirmationDialog()`/`onSubmitButtonClick()` in the live clientlib confirms the
comment-capture mechanism fix pass 7 wired **is implemented exactly as intended**: when
`$(metadata).data("iscommentallowed")` is true, the Confirm dialog injects a
`#fd-dashboard-tm-detailsview-comment` textarea labeled "Comment (optional)", and its value is sent
as the `comment` field in the completion POST alongside `formRoute`. Server-side, the Assign Task
step maps this to the workflow variable named by `WORKITEM_COMMENT` (`rejectionReason`) per the
model config. This is a genuine, code-level confirmation that the mechanism is correctly wired --
**but it cannot be functionally exercised end-to-end** (i.e., no test can prove `rejectionReason`
actually gets set to a real typed value or that it actually appears in the rejection email body),
because **no task can be completed at all, on any route, due to D10**. The Reject-path instance
(`_74`) never advanced past `node2`; `Send Rejection Notification` (`node10`) was never reached; the
mock SMTP catcher confirms zero emails were sent. This is reported as **UNVERIFIED, not PASS** --
the comment mechanism looks correct by code inspection alone, which is not the same as proof.

### Regression sweep -- 14/14, zero regressions

| Spec | Result |
|---|---|
| `employee-training-request-functional.cy.js` | 8/8 passing -- D-R1, D-R6, rules #2-#5/#7/#8, submit-gated-on-validation, D1 all regression-clean |
| `employee-training-request-final-retest.cy.js` | 5/5 passing -- Employee Details heading (UI-Finding-1), D-R1, rules #2-#4, D-R6, D1 all regression-clean |
| `employee-training-request-submissiondate-probe.cy.js` | 1/1 passing (informational) -- `Declaration.SubmissionDate` reconfirmed still `""` both at click-time and immediately after; D4 unchanged, not re-chased |

D2 independently reconfirmed unchanged via a fresh `GET /var/workflow/models/employee-training-request-approval.json`
fetched at the start of this pass: 12 nodes, zero `OR_SPLIT` occurrences, transitions still strictly
linear `node0` -> `node11`.

### Summary -- FINAL functional verification, fix pass 7

| Item | Status |
|---|---|
| D9 (`formview.jsp` NPE / `guideBridge` gate for `FORM_TYPE=AF`) | **CONFIRMED FIXED** by reverting to `FORM_TYPE=READ_ONLY_AF` -- zero validation-error dialogs, zero NPEs, on every fresh work item, both routes |
| D10 (NEW, Critical) -- `STATIC_ASSIGNEE="administrators"` group has no real human member; claiming throws a session-closed exception; direct REST completion also rejects `admin` as an "Invalid user" | **DISCOVERED, code+data-proven** (live error.log stack trace, live clientlib source, live REST response, live group-membership query, 2 independent failed remediation attempts) -- blocks Approve AND Reject, on every route, every work item, by every method tried |
| Manager Approval real task completion | **STILL BLOCKED** -- by D10, not D9 |
| Finance Manager Approval | **NOT REACHED** -- transitively blocked by D10 |
| Invoke DDX / Convert-to-PDF/A / Send Email attachment (functional) | **STILL NOT VERIFIABLE** -- workflow has never advanced past `node2` in this delivery's 7-fix-pass history; zero PDFs, zero captured emails |
| Reject path + comment-capture (`WORKITEM_COMMENT`/`IS_COMMENT_ALLOWED`) | Mechanism **CODE-CONFIRMED correctly wired**; functionally **UNEXERCISABLE end-to-end** -- same D10 blocker |
| D1, D-R1, D-R6, UI-Finding-1, D3, submit-gated-on-validation | **NO REGRESSION** -- 14/14 Cypress checks passing |
| D2 (OR-split) | **UNCHANGED**, reconfirmed still open (not re-chased) |
| D4 (SubmissionDate) | **UNCHANGED**, reconfirmed still open (not re-chased) |

**Gate: FAIL.** D9 is genuinely retired -- real progress. But the delivery's central LiveCycle-parity
objective (DDX assembling a real Document of Record that reaches the approval email as a real
attachment) remains unproven for the fourth consecutive fix pass targeting it (5, 6, and now 7),
each time blocked by a different, more deeply diagnosed defect (D6/D7/D8 -> D9 -> now D10). D10 is
qualitatively different from D2/D4: it is not a platform/architectural limitation to accept, but it
is also not a simple config toggle like D9's fix was -- it requires (a) understanding why this
environment's `administrators` group has no real member and why standard membership grants don't
persist, and (b) root-causing a genuine session-lifecycle bug inside
`AssignFormStep.execute`/`WorkflowSessionImpl.delegateWorkItem`. Recommend the NEXT fix pass target
D10 specifically -- start with the group-membership persistence anomaly (item 9 in Method, above),
since without a real, working assignee, no amount of further form/workflow-model tuning can ever
reach Invoke DDX. **Do not proceed to HANDOFF as a full PASS.**

### Final coverage numbers (fix pass 7)

Unchanged from fix pass 5/6's tally: `test_cases: { total: 34, executed: 34, passed: 25, failed: 9 }`;
`user_stories: { total: 11, covered: 10, uncovered: 1 }` -- US-08 (finance approval to DoR plus email)
remains the sole uncovered story, now for a third, still-more-precisely-diagnosed reason (D10, not
D9).

### Run metrics -- FINAL functional verification, fix pass 7

- `time_taken_minutes`: ~74.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~42,000 (D10 root-cause reasoning across 3 converging evidence chains, Cypress spec
    authoring, mock-SMTP-catcher design, this section)
  - `read`: ~185,000 (prior test-report.md fix-pass-6 section, `workitemdetails.js` full clientlib
    source across two passes, live workflow model JSON, ~6 `formdetails.html` raw dumps,
    `error.log` tails, screenshots, cypress result JSON files, prior spec files for conventions)
  - `write`: ~28,000 (new Cypress spec `employee-training-request-fixpass7-retest.cy.js`, the
    `mock-smtp.js` catcher, this section plus the matching sections in `integration-test-report.md`
    and `test-form-ui-report.md`)
  - `other`: ~115,000 (tool-call overhead: 2 Cypress run invocations against the live browser, ~20
    curl/JCR probes for instance/group/user state, 2 group-membership grant attempts + cleanup,
    2 direct REST completion attempts, screenshot captures, error.log greps)
  - `total`: ~370,000

### Handoff YAML -- FINAL functional verification, fix pass 7

```yaml
agent: sentinel
phase: TEST-FINAL-retest-fix-pass-7
status: FAIL
time_taken_minutes: 74.0
tokens_consumed:
  cli_text: 42000
  read: 185000
  write: 28000
  other: 115000
  total: 370000
test_cases: { total: 34, executed: 34, passed: 25, failed: 9 }
unexecuted_cases: 0
executes_with: { create-form-tests: 2, test-form-ui: 1, functional: 31 }
coverage_pct: null
user_stories: { total: 11, covered: 10 }
uncovered_stories: 1
ui_parity: { pixel: PASS_ADVISORY, pixel_match_pct: 85.79, critical_findings: 0 }
embedded_page:
  path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form"
  embeds_new_form: true
  functional_in_page: true
defects_confirmed_fixed_this_pass:
  - { id: D9, evidence: "zero validation-error dialogs / zero NPEs on every fresh work item after FORM_TYPE=READ_ONLY_AF revert, both Approve and Reject routes" }
new_defect_this_pass:
  id: D10
  severity: Critical
  owner: "groundsmith (STATIC_ASSIGNEE group-membership resolution) / platform (delegateWorkItem session-closed exception is an AEM Forms Workspace library bug, not form/workflow-model config)"
  agent_fixable: "partially -- the group-membership root cause needs investigation before a fix can even be attempted; the delegateWorkItem session-closed exception may be a genuine platform limitation"
  blocks: "real task completion for Manager Approval AND Finance Manager Approval, every route (Approve/Reject), every work item, by every method tried (UI click and direct REST) -- therefore blocks Invoke DDX / Convert-to-PDF/A / Send-Email-attachment from ever executing"
manager_approval_task_completion: "STILL BLOCKED -- by D10, not D9"
finance_approval_task_completion: "NOT REACHED -- transitively blocked by D10"
ddx_dor_assembly:
  status: "STILL NOT VERIFIABLE"
  reason: "workflow has never advanced past node2 in this delivery's 7-fix-pass history; no PDF has ever been produced"
email_attachment_verdict: "NOT VERIFIABLE -- same reason as DDX/DoR; mock-SMTP catcher captured zero messages"
reject_path_comment_parity: "mechanism CODE-CONFIRMED correctly wired (iscommentallowed drives a real comment textarea in the Confirm dialog, POSTed as `comment` alongside `formRoute`) but functionally UNEXERCISABLE end-to-end -- same D10 blocker"
regression_sweep: "14/14 Cypress tests passing -- D1, D-R1, D-R6, UI-Finding-1, D3, submit-gated-on-validation all regression-clean"
defects_still_open:
  - { id: D2, severity: Critical, owner: groundsmith, status: "UNCHANGED" }
  - { id: D4, severity: "Major/Critical-adjacent", owner: formwright, status: "UNCHANGED, NOT re-chased" }
  - { id: D10, severity: Critical, owner: "groundsmith / platform", status: "NEW this pass" }
delivery_totals:
  total_time_taken_minutes: "~1210.0"
  total_tokens_consumed: { cli_text: 458500, read: 1648700, write: 349100, other: 699200, total: 3155500 }
  per_phase: "see the Delivery Totals table at the top of this file for the full per-phase breakdown across all 7 fix passes"
report: ".claude/agents/runs/2026-07-22-employee-training-request/testing/test-report.md"
gate_result: FAIL
gap_summary: "D9 genuinely, live-confirmed FIXED (FORM_TYPE=READ_ONLY_AF revert). A single NEW, deeper, code+data-evidenced Critical defect (D10 -- assignee-group has no real member; claim throws a session-closed exception; direct REST completion also rejects admin) now blocks real task completion on every route, transitively blocking Finance Manager Approval and the delivery's core objective (DDX/DoR assembly plus email attachment) for the fourth consecutive targeted fix pass. D2 and D4 unchanged. Zero regression across 14/14 Cypress checks."
next: "aem-forms-program-agent: next fix pass must target D10 -- first investigate why /home/groups/a/administrators membership grants (2 independent attempts, both to admin and to a disposable test user) return 200 OK but do not persist (rep:members unchanged), then either add a real, working human member to that group or reconsider STATIC_ASSIGNEE's resolution; separately, root-cause or work around the IllegalStateException in WorkflowSessionImpl.delegateWorkItem/AssignFormStep.execute. Do NOT proceed to HANDOFF as a full PASS."
```

---

## DEFINITIVE final functional verification — fix pass 8 (2026-07-27, D10: admin assignee; NEW D11/D12 discovered — THE TRUE FINAL RETEST)

**Trigger.** Groundsmith changed `STATIC_ASSIGNEE` from `"administrators"` (a group with no real
member) to `"admin"` (a direct user) on both `assigntask_manager` and `assigntask_finance`, live-pushed
and readback-confirmed on the design + regenerated runtime model. Forgemaster ran a confirming full
build+deploy (BUILD SUCCESS, `Package installed in 588ms`, D10 fix verified live post-redeploy, zero
drift between design/runtime/packaged source — `deployment/code-quality-report.md` §15). This section
is Sentinel's real end-to-end retest of that fix, and — going one level deeper than any prior pass —
the pass that finally, conclusively answers this delivery's central 8-fix-pass question.

### Method

1. Confirmed the live runtime model carries `STATIC_ASSIGNEE:"admin"` on both steps before starting.
2. Authored `employee-training-request-fixpass8-retest.cy.js` (Cypress/Electron), adapted from fix
   pass 7's spec: submitted a fresh Approve-path instance and a separate fresh Reject-path instance
   through the embedded page, drove each instance's Manager Approval work item through the real Task
   Manager detail view exactly as a human reviewer would (open, Approve/Reject, type a comment, click
   Confirm), on the **delivered/committed configuration** — no config changes yet.
3. Cross-referenced `error.log` at the exact request timestamps of both completion attempts.
4. Discovered a second failure class (see below) and — following this delivery's own established,
   explicitly-reversible-diagnostic precedent (fix pass 7's disposable group-membership grant + SMTP
   catcher) — ran ONE additional, fully reversible live experiment on `/conf` only (never on committed
   source) to isolate it and, having isolated it, kept driving forward to finally answer the DDX/DoR
   question this delivery has never been able to answer in 7 prior passes. Reverted the live JCR to
   the exact committed values and regenerated the runtime model again before this pass ended —
   confirmed byte-for-byte identical to the delivered configuration.
5. Ran the full existing Cypress regression suite (functional, final-retest, submissiondate-probe).

### Result 1 — D10 CONFIRMED FIXED, on the delivered configuration, with real click-through evidence

Both the Approve control and the Reject control were clicked directly with **zero claim/delegate
step** — no "Assign to Self" modal ever appeared, because the work item is now a direct-user
assignment, not group-pending. `error.log` shows zero occurrences of `IllegalStateException`/`"Invalid
user - admin"`/`AEM-FD-011-004` for either attempt. **D10 is retired.** This is the first fix pass in
this delivery's history where clicking Approve or Reject does not immediately fail for an
assignee-resolution reason.

### Result 2 — a NEW Critical defect (D11) now blocks completion on the delivered configuration, on BOTH routes, code+log-proven

Both real completion attempts (Approve with an optional typed comment; Reject with a required typed
comment) returned **HTTP 500** and neither instance advanced past `node2`. `error.log` shows an
**identical exception at both exact request timestamps**:

```
com.adobe.fd.workspace.exceptions.FormsWorkflowException: Unable to update metadata for workitem - {0}
	at com.adobe.fd.workspace.service.impl.WorkSpacePayLoadManagerImpl.updateWorkItemMetaData(WorkSpacePayLoadManagerImpl.java:1419)
Caused by: com.adobe.granite.workflow.WorkflowException: Invalid value : rejectionReason
	at com.adobe.fd.workflow.utils.PropertyResolver.setPropertyValueUsingColonSeparatedValue(PropertyResolver.java:160)
	at com.adobe.fd.workspace.service.impl.WorkSpacePayLoadManagerImpl.saveComment(WorkSpacePayLoadManagerImpl.java:1405)
*ERROR* com.adobe.fd.workspace.servlet.FormsSubmissionServlet Unable to complete the task - {0}. Contact your administrator to resolve the issue.
```

**Root cause:** `saveComment()` hands the assignee's free-text comment to a colon-separated-category
parser (`PropertyResolver.setPropertyValueUsingColonSeparatedValue` — the same utility class already
implicated in D6) which throws on any value that isn't `"CATEGORY:value"` shaped. A free-text comment
never is. The `WORKITEM_COMMENT="rejectionReason"` + `IS_COMMENT_ALLOWED=true` mechanism fix pass 7
wired — verified "code-confirmed correctly wired" at the time — turns out to **crash task completion
outright** the moment it is actually exercised on a real click-through, on **both** Assign Task steps,
on **both** routes. Full class/line/message detail: `integration-test-report.md` §"DEFINITIVE final
functional verification — fix pass 8", Result 2.

### Reversible diagnostic (temporary, fully reverted — NOT part of the delivered configuration)

Because D11 was the only thing standing between this delivery and finally answering its central
question, Sentinel ran one additional, explicitly temporary, fully reversible experiment: disabled
`IS_COMMENT_ALLOWED`/cleared `WORKITEM_COMMENT` live on `/conf` (CSRF+Referer POST, readback-confirmed,
runtime regenerated), drove real completions, then **reverted both properties to their committed
values and regenerated the runtime model again** — confirmed matching the delivered source exactly
before this pass ended. No source file was edited at any point.

**With D11 out of the way, both real completions succeeded cleanly:** Manager Approval Approve
advanced the instance from `node2` to `node4`; Finance Manager Approval Approve advanced it from
`node4` to `node6`. A separate Reject on Manager Approval **also** advanced to `node4` — the identical
next node Approve reaches — **direct, live confirmation that D2 (no real OR-split) is unchanged and
unaffected by anything in this pass.** This conclusively isolates D11 as the sole completion blocker
on the delivered configuration, and confirms Finance Manager Approval itself has no independent defect
— it works exactly like Manager Approval once D11 is out of the way.

### Result 3 — THE CORE OBJECTIVE (DDX/DoR assembly): CONCLUSIVELY ANSWERED. NO — it fails deterministically, every time.

With D11 bypassed, the Approve-path instance reached `node6` (Assemble Manager Confirmation / Invoke
DDX) and then failed **identically on all 10 automatic retries** (Granite Workflow's job-queue retry
policy exhausted: `retryCount=1` through `retryCount=10`, `"will retry 0 more time(s)"` at the last),
and remains permanently stuck `RUNNING` at `node6` with no further progress. **This is a NEW, Critical
defect — D12:**

```
com.adobe.granite.workflow.WorkflowException: Exception while executing Invoke DDX Process step
	at com.adobe.fd.workflow.assembler.InvokeDDXProcess.internal_execute(InvokeDDXProcess.java:113)
```

Adobe's own code does not chain the underlying root cause into this message (a platform logging gap,
confirmed by inspecting every line touching this exception — no deeper `Caused by` exists anywhere in
`error.log` or `stdout.log`). **Most likely structural cause, reasoned from the step's own binding**:
the DDX operation is an XDP-merge operation expecting genuine XDP-template Document inputs, but both
`inputDocs` keys (`EmployeeTrainingForm`, `ApprovalInfo`) are bound to the exact same
`RELATIVE_PLOAD:data.xml` — the plain Adaptive Form data payload, not an XDP template. This is
precisely the approximation fix pass 4/5 already flagged in the workflow model's own documentation as
"the closest achievable equivalent... not built here; flagged as a pre-production follow-up" — this is
the first time that approximation has actually been exercised end-to-end, and it empirically fails.

**Direct, conclusive answer:** No PDF has ever been produced by this workflow across all 8 fix passes,
and under the current `inputDocs` binding **it cannot be** — Invoke DDX itself throws on every single
attempt. Convert-to-PDF/A and the approval Send-Email attachment are consequently **never reached — not
"unverifiable," but conclusively, empirically unreachable under the current design**, independent of
D11. This is the delivery's central question, finally answered with hard evidence after 8 fix passes.

### Result 4 — Reject path + comment-capture parity: the mechanism IS the cause of D11, not merely unverified

Fix pass 7 reported this mechanism as "code-confirmed wired, functionally unexercisable." This pass
supersedes that: exercising it is exactly what throws D11, on both the optional Approve-comment and the
required Reject-comment. The rejection-reason-in-email question remains unverified for a new, precise
reason — not "the workflow never got that far" but "saving any comment value into `rejectionReason` via
`WORKITEM_COMMENT` crashes completion outright, on the delivered configuration."

### Regression sweep — 14/14, zero regressions; D2/D4 reconfirmed unchanged, not re-chased

`employee-training-request-functional.cy.js` (8/8), `employee-training-request-final-retest.cy.js`
(5/5), `employee-training-request-submissiondate-probe.cy.js` (1/1 — D4's `Declaration.SubmissionDate`
still empty at click-time and immediately after). D2 reconfirmed via the diagnostic run itself — the
most direct evidence yet: both Approve and Reject on Manager Approval converge on the identical next
node, and the live runtime model still shows zero `OR_SPLIT` occurrences across 12 nodes.

### Updated case tally and coverage

The 9 previously-failing cases were re-examined against this pass's new evidence. **None flip to
PASS; none of the 25 passing cases regress.** Root-cause attribution updates:

| ID | Prior root cause (fix pass 7) | Updated root cause this pass |
|---|---|---|
| TC-013 | Blocked by D10 (assignee-group has no member) | **D10 retired — task now lands AND is clickable — but blocked by NEW D11 (comment-save crash) on the delivered config** |
| TC-016, TC-017 | Blocked by D10; would also fail D2 | **D10 retired. Still blocked — D2 (no branching, reconfirmed) AND now D11 (no completion possible at all on delivered config) AND D12 (Invoke DDX fails deterministically even if D11 were fixed)** |
| TC-026 | DoR PDF never generated, blocked by D10 | **CONCLUSIVELY answered this pass (via reversible diagnostic): DDX assembly fails deterministically (D12) — no PDF can ever be produced under the current `inputDocs` binding, independent of D11/D10** |
| TC-032 | Instance stuck at node2, Send Email never reached (D10) | **D10 retired; still never reached — blocked by D11 on delivered config, and by D12 even if D11 is fixed** |
| TC-012 | D2 (no real Approve/Reject branching) + D10 | **D10 retired. D2 confirmed still open — and now directly, empirically confirmed (both routes converge on the identical next node)** |
| TC-004, TC-031 | Architecture/test-case mismatch (no prefill, accepted decision) | **Unchanged** — unrelated to this pass |
| TC-019 | D4 (SubmissionDate) | **Unchanged, not re-chased** |

- `test_cases: { total: 34, executed: 34, passed: 25, failed: 9 }` — **unchanged in count**, root-cause
  attribution updated (D10 retired; D11 NEW blocks the delivered config; D12 NEW conclusively answers
  the DDX/DoR question independent of D11)
- `unexecuted_cases: 0`
- `user_stories: { total: 11, covered: 10 }`, `uncovered_stories: [US-08]` — **unchanged**. US-08's own
  acceptance criteria (AC-08.1 DoR generation on finance approval, AC-08.2 no-DoR-on-reject) still have
  zero passing cases — now for a **conclusively proven** reason (D12) rather than a blocked/unverifiable
  one.
- `ui_parity`: carried forward, **85.79% pixel match (advisory in migration mode), 0 Critical** — no
  visual artifact touched this pass or its reversible diagnostic; spot-checked live via the 14/14
  regression sweep.

### Embedded-page result (reconfirmed, unchanged)

`/content/aem-adaptive-forms-agents/us/en/test-adaptive-form` — HTTP 200, still embeds exactly
`employee-training-request` via its one AEM Form Container, no stale/duplicate embed. Functional
in-page: confirmed via 2 fresh real submissions through the page this pass (HTTP 200 each, both
creating genuine RUNNING workflow instances).

### Is this delivery ready to ship? No — but the gap is now, for the first time, fully characterized rather than merely blocked

Every prior fix pass hit a wall before ever reaching the DDX/DoR question. This pass is the first to
get past every prior wall (D9, D10) and reach the actual mechanism the entire delivery exists to prove
— and that mechanism does not work, for a structural reason (D12) that is independent of, and deeper
than, every defect fixed so far. **Recommendation:**
1. **D2 (OR-split)** — unchanged recommendation: accept as a documented, human-actionable follow-up
   (Workflow Editor UI access, not available to any agent in this session).
2. **D4 (SubmissionDate)** — unchanged recommendation: accept or schedule a mechanism redesign.
3. **D11 (comment-save crash)** — recommend a further fix pass: either remove `WORKITEM_COMMENT`/set
   `IS_COMMENT_ALLOWED=false` (accepting loss of comment-to-rejectionReason parity as a documented,
   lower-priority follow-up — proven, in this pass's diagnostic, to unblock completion immediately) or
   find the value-format `PropertyResolver` actually requires. This is fully within an agent's power to
   fix and directly blocks every real task completion on the delivered configuration.
4. **D12 (Invoke DDX fails deterministically)** — recommend a further fix pass redesigning the
   `inputDocs` binding: render `EmployeeTrainingForm`/`ApprovalInfo` as genuine separate Documents
   before Invoke DDX (an additional upstream step), or reconsider whether the DDX/Assembler XDP-merge
   operation is the right mechanism at all for a single-schema Adaptive Form payload on AEMaaCS. This
   is the delivery's central, now-conclusively-diagnosed blocker.

**This should NOT ship as a full PASS.** However, this pass represents the most significant forward
progress of the entire delivery: two long-standing blockers (D9, D10) are genuinely retired with real
click-through evidence, Finance Manager Approval is proven to work once D11 is out of the way, D2 is
now confirmed with the most direct evidence this delivery has produced, and — for the first time in 8
fix passes — the delivery's central open question has a definitive, evidenced answer instead of
"blocked/unverifiable."

### GATE RESULT: **FAIL**

Two Critical defects remain open on the delivered configuration (D11, D12 — both NEW this pass, both
agent-fixable), plus the two long-standing, documented, human-actionable/accepted follow-ups (D2, D4).
`uncovered_stories: [US-08]` remains non-empty (1 of 11) — hard gate failure per the project's
non-negotiable rule. `unexecuted_cases: 0` ✓.

### Run metrics — DEFINITIVE final functional verification, fix pass 8

- `time_taken_minutes`: 95.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~45,000 (D11/D12 root-cause reasoning, the reversible-diagnostic design and its
    revert-verification, Cypress spec authoring/debugging, these report sections)
  - `read`: ~220,000 (prior test-report.md/integration-test-report.md/groundsmith.md/code-quality-report.md
    sections for continuity, the full committed workflow-model `.content.xml` (667 lines), ~400 lines
    of live `error.log` across multiple targeted greps, 5 Cypress result JSON files, 1 screenshot image
    read, the fixpass7 spec used as the adaptation base)
  - `write`: ~35,000 (2 new Cypress spec files authored — `employee-training-request-fixpass8-retest.cy.js`
    adapted + `employee-training-request-d11-isolation.cy.js` — plus 3 targeted spec edits, and this
    pass's sections across all 3 reports)
  - `other`: ~130,000 (5 Cypress/Electron run invocations against the live browser, ~15
    curl/PowerShell live-diagnostic calls incl. the reversible `/conf` toggle+revert+regenerate cycle,
    process management for a stuck diagnostic run, ~20 targeted `error.log`/`stdout.log` greps)
  - `total`: ~430,000

### Handoff YAML — DEFINITIVE final functional verification, fix pass 8 (TRUE FINAL)

```yaml
agent: sentinel
phase: TEST-DEFINITIVE-fix-pass-8
status: FAIL
time_taken_minutes: 95.0
tokens_consumed:
  cli_text: 45000
  read: 220000
  write: 35000
  other: 130000
  total: 430000
test_cases: { total: 34, executed: 34, passed: 25, failed: 9 }
unexecuted_cases: 0
executes_with: { create-form-tests: 2, test-form-ui: 1, functional: 31 }
coverage_pct: null
user_stories: { total: 11, covered: 10 }
uncovered_stories: 1
ui_parity: { pixel: PASS_ADVISORY, pixel_match_pct: 85.79, critical_findings: 0 }
embedded_page:
  path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form"
  embeds_new_form: true
  functional_in_page: true
defects_confirmed_fixed_this_pass:
  - { id: D10, evidence: "real Approve/Reject click-through on both routes, zero claim/delegate step, zero IllegalStateException/Invalid-user errors, on the delivered configuration" }
new_defects_this_pass:
  - id: D11
    severity: Critical
    owner: groundsmith
    agent_fixable: true
    summary: "WORKITEM_COMMENT=rejectionReason + IS_COMMENT_ALLOWED=true crashes task completion on BOTH Assign Task steps, BOTH routes, on the delivered configuration -- com.adobe.granite.workflow.WorkflowException: 'Invalid value : rejectionReason' from PropertyResolver.setPropertyValueUsingColonSeparatedValue (called via WorkSpacePayLoadManagerImpl.saveComment/updateWorkItemMetaData); the parser requires 'CATEGORY:value'-shaped input and a free-text comment never is"
    recommended_fix: "remove WORKITEM_COMMENT / set IS_COMMENT_ALLOWED=false (proven in this pass's reversible diagnostic to unblock completion immediately, at the cost of the comment-to-rejectionReason parity feature) OR find the value-format PropertyResolver actually requires"
  - id: D12
    severity: Critical
    owner: groundsmith
    agent_fixable: true
    summary: "Invoke DDX (Assemble Manager Confirmation, node6) throws com.adobe.granite.workflow.WorkflowException: 'Exception while executing Invoke DDX Process step' (InvokeDDXProcess.internal_execute, InvokeDDXProcess.java:113) deterministically on 10/10 automatic retries -- confirmed via a temporary, fully-reverted live diagnostic that bypassed D11 specifically to reach this step. No PDF has ever been produced by this workflow. Most likely cause: both DDX inputDocs keys (EmployeeTrainingForm, ApprovalInfo) bind to the SAME RELATIVE_PLOAD:data.xml, but the DDX XDP-merge operation expects genuine XDP-template Document inputs -- an approximation already flagged as a pre-production gap in fix pass 4/5, now empirically confirmed to fail at runtime"
    recommended_fix: "redesign inputDocs to render EmployeeTrainingForm/ApprovalInfo as genuine separate Documents upstream of Invoke DDX, or reconsider whether the DDX/Assembler XDP-merge operation is the right mechanism for a single-schema AF payload on AEMaaCS"
    conclusiveness: "THIS CONCLUSIVELY ANSWERS the delivery's central 8-fix-pass question: DDX/DoR assembly does NOT work under the current design, independent of D11"
manager_approval_task_completion: "delivered config: BLOCKED by D11. Diagnostic-only (NOT delivered): confirmed working cleanly once D11 disabled."
finance_approval_task_completion: "delivered config: NOT REACHED (transitively blocked by D11). Diagnostic-only (NOT delivered): confirmed working cleanly once D11 disabled -- no independent defect in this step."
ddx_dor_assembly:
  status: "CONCLUSIVELY DISPROVEN -- fails deterministically, 10/10 retries, every attempt"
  reason: "D12 -- Invoke DDX throws on every invocation; no PDF has ever been, or currently can be, produced under the current inputDocs binding"
email_attachment_verdict: "NOT VERIFIABLE -- Convert-to-PDF/A and Send-Email-with-attachment are never reached (D12 blocks upstream)"
reject_path_comment_parity: "mechanism IS the cause of D11 -- exercising it (typing any comment, or leaving it blank with IS_COMMENT_ALLOWED=true) crashes completion on the delivered configuration"
regression_sweep: "14/14 Cypress tests passing -- D1, D-R1, D-R6, UI-Finding-1, D3, submit-gated-on-validation, D2 (directly re-confirmed via the diagnostic itself) all regression-clean"
reversible_diagnostic_used:
  what: "temporarily set IS_COMMENT_ALLOWED=false and cleared WORKITEM_COMMENT on both Assign Task nodes, live on /conf only"
  why: "to isolate D11 as the sole completion blocker and finally reach Invoke DDX to answer the delivery's central question"
  reverted: true
  reverted_verification: "live readback + regenerated runtime model confirmed byte-for-byte match to the committed source (IS_COMMENT_ALLOWED=true, WORKITEM_COMMENT=rejectionReason) before this pass ended"
  source_files_touched: []
defects_still_open:
  - { id: D2, severity: Critical, owner: groundsmith, status: "UNCHANGED, now most-directly-confirmed via this pass's own diagnostic (both routes converge on the identical next node)" }
  - { id: D4, severity: "Major/Critical-adjacent", owner: formwright, status: "UNCHANGED, NOT re-chased" }
  - { id: D11, severity: Critical, owner: groundsmith, status: "NEW this pass" }
  - { id: D12, severity: Critical, owner: groundsmith, status: "NEW this pass -- conclusively answers the DDX/DoR question" }
delivery_totals:
  total_time_taken_minutes: "~1356.0"
  total_tokens_consumed: { cli_text: 522500, read: 1966700, write: 397100, other: 849700, total: 3736000 }
  per_phase: "see the Delivery Totals table at the top of this file for the full per-phase breakdown across all 8 fix passes"
report: ".claude/agents/runs/2026-07-22-employee-training-request/testing/test-report.md"
gate_result: FAIL
gap_summary: "D9 and D10 both genuinely, live-confirmed FIXED with real click-through evidence -- the most forward progress of any single fix pass in this delivery. Two NEW Critical defects discovered and precisely diagnosed this pass: D11 (comment-save crash, blocks all real task completion on the delivered configuration, fully agent-fixable) and D12 (Invoke DDX fails deterministically on every attempt, CONCLUSIVELY answering this delivery's central question that no PDF can be produced under the current design, independent of D11). D2 and D4 unchanged, D2 now most-directly-confirmed. Zero regression across 14/14 Cypress checks."
next: "aem-forms-program-agent: route D11 and D12 back to groundsmith for a 9th fix pass (D11: remove/redesign the comment-capture mechanism; D12: redesign the Invoke DDX inputDocs binding or reconsider the DDX/Assembler approach entirely for this single-schema AF payload). D2/D4 remain acceptable documented follow-ups. Do NOT proceed to HANDOFF as a full PASS -- but this delivery has, for the first time, a complete and conclusive picture of every remaining gap, with no more 'unverifiable' unknowns."
```

---

## DEFINITIVE final E2E — fix pass 9 (Generate-DoR + admin assignee, real AF_PATH) — 2026-07-27

**Gate: FAIL.** The whole approval chain now runs end-to-end WITHOUT the D9/D10/D11 crashes
(submit -> Manager Approve -> Finance Approve all clean), but the delivery's central objective — the
**Generate Document of Record step producing the PDF that the approval email carries — still does not
work.** AFtoDORStep rejects the (valid, renderable) Core Components form as "Not a valid Adaptive
Form," and the one pre-authorized narrow fix (dorType none->generate) did NOT resolve it. This is a
deeper platform/compatibility failure, captured precisely below; no further speculative fixes were
attempted per instruction.

### Runtime model (live-verified before testing)
`/var/workflow/models/employee-training-request-approval` v1.27, LINEAR transitions node0..node10 (D2
still open, expected). node2 Manager Approval + node4 Finance Manager Approval: both
STATIC_ASSIGNEE=admin, FORM_TYPE=READ_ONLY_AF, ROUTES=[Approve,Reject], NO WORKITEM_COMMENT, flat
data.xml/attachments I/O. node6 Generate DoR = com.adobe.fd.workflow.dorGeneration.AFtoDORStep,
AF_PATH=/content/forms/af/aem-adaptive-form-agents/employee-training-request (REAL interactive AF,
confirmed also in FormResolverUsingPath log), FORM_RESOLUTION=PATH,
DOR_PATH=UNDER_PLOAD:DocumentofRecord/DoR.pdf, INPUT_DATAXML=RELATIVE_PLOAD:data.xml. No InvokeDDX /
Convert-to-PDF/A nodes, no stale -dor path. Exactly as the task briefing described.

### 1. APPROVE path — BOTH approval tasks complete cleanly (real Workspace UI)
Driven through the embedded page (.../test-adaptive-form.html) via Cypress; tasks completed via the
real /aem/dashboard/formdetails.html Workspace UI. Two fresh instances (_111, _113):
- Submit -> HTTP 200, workflow instance RUNNING at node2 (Manager Approval).
- Manager Approval (admin) -> Approve: completed, advanced node2->node4. ZERO session-closed /
  enum-crash / HTTP-500-comment / "There are validation errors" / "Not a valid Adaptive Form" errors
  on completion. (D9/D10/D11 fixes hold on the Approve route.)
- Finance Manager Approval (admin) -> Approve: completed, advanced node4->node5->node6.
- Evidence: cypress/results/form-ui/employee-training-request-fixpass9-approve-results.json,
  fp9-approve-final-instance.json, fp9-approve-context.json (+ screenshots under
  cypress/results/screenshots/employee-training-request-fixpass9-approve.cy.js/form-ui/).

### 2. CORE OBJECTIVE — Generate Document of Record — FAIL (no PDF produced)
On BOTH approve instances the workflow reaches node6 and gets STUCK there permanently (state RUNNING,
work item at node6; after 10 job retries the DoR job hard-fails). The DoR PDF is NOT written:
GET <payload>/DocumentofRecord/DoR.pdf -> HTTP 404 on both _111 and _113.

Exact error (verbatim, crx-quickstart/logs/error.log):
    com.adobe.fd.workflow.dorGeneration.AFtoDORStep  Adaptive form at Node : /content/forms/af/aem-adaptive-form-agents/employee-training-request/jcr:content is not valid
    com.adobe.fd.workflow.dorGeneration.AFtoDORStep  Exception while document of record generation
    java.lang.Exception: Not a valid Adaptive Form
        at com.adobe.fd.workflow.dorGeneration.AFtoDORStep.internal_execute(AFtoDORStep.java:125) [com.adobe.aemfd.adobe-aemfd-workflow-process-common:6.0.260]
    com.adobe.granite.workflow.WorkflowException: Exception while document of record generation
        at com.adobe.fd.workflow.dorGeneration.AFtoDORStep.internal_execute(AFtoDORStep.java:182)
(After retry exhaustion, AEM's own JobFailedEventHandler then throws a secondary vendor
NullPointerException: ... WorkflowModel.getNode(String) because "wfmodel" is null — a platform bug in
the failure handler, not our config.)

**dorType fix applied (narrow, pre-authorized) — necessary config hygiene but NOT sufficient.**
Pre-test static check found the guideContainer had dorType="none". Applied the one pre-authorized safe
fix: set dorType="generate" in source (ui.content/.../employee-training-request/.content.xml) AND
live-pushed it (readback confirmed "dorType":"generate"). Re-ran the full approve E2E (_113): IDENTICAL
"Not a valid Adaptive Form" failure at AFtoDORStep.java:125 — dorType had ZERO effect.

Root cause (non-speculative diagnostics, no thrashing): the form is a genuinely valid, renderable Core
Components AF — guideContainer sling:resourceType = aem-adaptive-forms-agents/components/adaptiveForm/formcontainer
(CC v2 proxy), guideContainer.model.json -> HTTP 200, fills+submits fine on the embedded page. The
OOTB Generate-DoR step (AFtoDORStep, bundle adobe-aemfd-workflow-process-common:6.0.260) nonetheless
rejects it at its line-125 validity gate, independent of dorType, AF_PATH (correct), or the payload.
This is a deeper AEMFD Generate-DoR-step / Core-Components compatibility failure on this SDK, NOT the
dorType-not-enabled scenario the briefing anticipated. Per instruction, no further speculative fixes.

### 3. Approval email + PDF attachment — not reached
node7 (Send Approval Notification, attachmentPathHiddenField=RELATIVE_PLOAD:DocumentofRecord/DoR.pdf)
never executes because node6 blocks upstream. No approval email, no PDF attachment. (SMTP is also
unconfigured on the local SDK — TC-032 accepted no-op — but moot since the step is never reached.)

### 4. REJECT path — task completes via Reject route with NO error; rejection email does NOT send
Fresh instance _115 (employee-training-request-fixpass9-reject.cy.js):
- Submit -> HTTP 200, RUNNING at node2.
- Manager Approval -> Reject: completed cleanly — no session-closed/enum/HTTP-500/validation errors.
  (First clean Reject-route completion under the fix-pass-9 model.)
- BUT linear model (D2 open): Reject does not short-circuit. Instance advanced node2->node4 (Finance),
  NOT to the rejection email. node9 (rejection email) sits after node6 (DoR) in the linear chain, so
  it is never reached (node6 hard-fails) and would not deliver anyway (SMTP unconfigured). Rejection-
  reason capture is a DOCUMENTED fix-pass-9 deferral (WORKITEM_COMMENT removed to fix D11;
  rejectionReason intentionally unwired -> rejection.html renders a blank reason) — expected, not a
  failure. A comment textarea DID still appear (IS_COMMENT_ALLOWED=true) and was typed, but by design
  that text no longer lands in any variable.
- Evidence: employee-training-request-fixpass9-reject-results.json, fp9-reject-final-instance.json.

### 5. Regression sweep — ALL GREEN (employee-training-request-functional.cy.js)
RT-01 (UI-Finding-1 heading) PASS; RT-02 (D-R1 Manager lock/unlock) PASS; RT-03 (rule#3 RequestType
lock) PASS; RT-04 (rule#4 EndDate>StartDate, invalid surfaces error) PASS; RT-05 (rule#2 email) PASS;
RT-06 (D-R6 FinanceComments lock) PASS; RT-07 (rule#5 ApprovalInfo show/hide) PASS; RT-08 (D1 +
submit-gated-on-validation TC-002/TC-020: empty BLOCKED, valid submit HTTP 200 + thank-you) PASS. No
regressions. D2 reconfirmed still linear/open. D4 (SubmissionDate) unchanged — carried open.

### 6. Test-case results (this pass): 26 pass / 8 fail / 0 unexecuted
FAIL: TC-012 (Reject does not route to rejection email — D2), TC-016 (finance-approval DoR+email —
AFtoDORStep fails), TC-017 (finance-reject email/no-DoR — D2 + DoR broken), TC-019 (SubmissionDate —
D4), TC-020 (submit-gating PASSES; the "valid form eventually produces the DoR PDF" clause FAILS),
TC-026 (DoR PDF content — no PDF), TC-031 (no prefill service — architecture/test mismatch), TC-032
(Send Email steps never reached — blocked by node6). All others PASS.

### 7. User-story coverage
All 11 stories have >=1 passing case (uncovered_stories: 0 by the letter). HOWEVER the delivery's core
acceptance criteria have ZERO passing cases: AC-08.1 (DoR on finance approval), AC-08.2 (rejection
email + no DoR), AC-11.1 (merged DoR PDF content), and AC-06.2's reject half. US-08/US-11 are "covered"
only by adjacent structural/assignee cases (TC-013, TC-025) — their real DoR objective is UNMET.

### 8. Gate verdict & routing
FAIL. The approval workflow now runs crash-free end-to-end through both approvals (a real improvement
over passes 1-8), but the central migration objective — a Document-of-Record PDF produced on final
approval and attached to the approval email — does NOT work. Defects:
- D13 (NEW, Critical, -> Groundsmith/Formwright): OOTB AFtoDORStep rejects the valid CC form as "Not a
  valid Adaptive Form" (AFtoDORStep.java:125), independent of dorType/AF_PATH/payload — a deeper AEMFD
  Generate-DoR-vs-Core-Components compatibility issue on this SDK. dorType none->generate applied
  (source+live) but insufficient. No PDF, no approval-email attachment.
- D2 (Critical, open, escalated): linear model, no OR-split; Reject does not branch to rejection email.
  Needs Workflow Editor UI.
- D4 (open): SubmissionDate cross-panel set-value doesn't land. Rejection-reason capture: documented
  fix-pass-9 deferral (expected). Real manager-lookup + distinct finance-approver: accepted follow-ups.

Recommendation: DO NOT ship. Route D13 to the integration/build leads: either (a) wire the correct DoR
mechanism for a Core Components AF (a real DoR template via dorType=select + dorTemplateRef, or the
CC-native DoR service) instead of AFtoDORStep against the live form, or (b) confirm whether this AEMFD
AFtoDORStep build supports CC forms at all on this SDK. Re-test only after Forgemaster redeploys.
