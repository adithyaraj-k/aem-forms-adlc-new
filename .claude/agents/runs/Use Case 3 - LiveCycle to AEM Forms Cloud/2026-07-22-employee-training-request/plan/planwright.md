# Planwright — PLAN Package
## Employee Training Request — LiveCycle → AEM as a Cloud Service (Path B migration)

Run: `.claude/agents/runs/2026-07-22-employee-training-request/`
Source: `TestMigrationApp.lca` (AEM Forms on J2EE / LiveCycle 6.4.22)

---

## 1. Form role decisions (evidence-based — overrides naive name-guessing)

| XDP | Role | Key evidence |
|---|---|---|
| **Forms/DocRoute.xdp** (~106KB) | **PRIMARY interactive Adaptive Form** | The ONLY xdp wired as `tloPath` at ALL THREE LC Workspace touchpoints in Main.process (start endpoint, assignToManager, assignToFinanceManager). Carries workflow-state-driven scripts absent elsewhere: ApprovalInfo panel show/hide keyed on `CurrentStatus`, and FinanceComments lock/unlock keyed on `CurrentStatus=="ManagerApproved"`. |
| **Forms/EmployeeTrainingRequest.xdp** (~101KB) | **Document-of-Record template** (request-data section) | Referenced only as a static `document` variable fed into the DDX merge (`AssemblerService.invokeDDX`) in `AssembleManagerConformation.process`. Same field layout as DocRoute but NO state-driven scripts on CurrentStatus — never a `tloPath`. |
| **Forms/ApprovalInfo.xdp** (~15KB) | **Document-of-Record fragment** (approval-decision section only) | Contains ONLY the Approval Information panel; fed as the second DDX merge source. Never independently interactive. |

Visual confirmation: DAM preview JPEGs show DocRoute and EmployeeTrainingRequest rendering identically (ApprovalInfo panel hidden, matching the show/hide script since EmployeeId is empty in the static preview); ApprovalInfo.xdp's preview shows only its one small panel.

This **reverses** the naive assumption that EmployeeTrainingRequest.xdp is the primary form — the process-XML wiring (`tloPath`) and script placement are the deciding evidence.

DAM preview JPEGs (UI-parity references, no live 6.4.22 Workspace reachable):
- `C:\tmp-inner-pkg\jcr_root\content\dam\formsanddocuments\TestMigration\1.0\Forms\DocRoute.xdp\_jcr_content\renditions\cq5dam.web.1280.1280.jpeg` — primary AF parity reference
- `C:\tmp-inner-pkg\...\Forms\EmployeeTrainingRequest.xdp\_jcr_content\renditions\cq5dam.web.1280.1280.jpeg` — DoR (request-data) parity reference
- `C:\tmp-inner-pkg\...\Forms\ApprovalInfo.xdp\_jcr_content\renditions\cq5dam.web.1280.1280.jpeg` — DoR (approval) parity reference

---

## 2. Structured Requirements & User Stories

Full detail: `plan/user-stories.yaml`

- 2 forms in scope: `employee-training-request` (interactive AF) and `employee-training-request-dor` (DoR templates, not a separate AF).
- Full field inventory (24 fields across Employee Details / Training Request Details / Justification / Attachments / Declaration / Approval Information), schema-bound (no FDM — no external system of record in the source).
- 8 business rules migrated from XFA JavaScript (all JS, no FormCalc found): manager auto-assign+lock, email regex validate, RequestType lock/unlock, EndDate>StartDate validate, ApprovalInfo show/hide, FinanceComments lock/unlock, AcceptedTerms required, SubmissionDate set-value.
- Exact-replica style spec captured from XDP `<font>/<fill>/<color>/<border>`: Myriad Pro, header band RGB(196,219,251)/maroon 20pt bold caption RGB(128,64,64), panel body RGB(225,242,219), 3-column boxed rows.
- Workflow: Main.process (manager → finance-manager sequential approval, reject-to-email shortcuts) + AssembleManagerConformation.process (DDX merge → Document of Record PDF → approved email).
- **11 user stories**, every one with ≥1 acceptance criterion, covering every field/rule/submit behaviour (US-01 through US-11). 0 stories missing acceptance criteria.

---

## 3. Solution Architecture

Full detail: `plan/solution-architecture.yaml`

- **Data backing:** schema (generate-schema from employeeTrainingRequest.xsd) — no FDM, no external system of record evident.
- **Template:** REUSE `blank-af-v2` (af-page-v2 template-type, generic unlocked "Blank Form") — **create-editable-template is SKIPPED**. Enumerated the project's ~20 templates; none of the form-specific ones fit better than the generic base, and per-form branding does not require a new template.
- **Theme:** BUILD new (create-form-theme) — none of the 6 existing project themes match the exact-replica RGB palette required; a stock theme here is a parity FAIL.
- **Custom components:** none needed — every field maps to a stock Core Components type (including plain File Attachment for the 3 attachment fields).
- **Submit:** "Invoke an AEM Workflow" (create-submit-action) — NOT the generic Custom-Submit-GeneratePDF, since completion is workflow-routed, not a direct PDF download.
- **Prefill:** ask-then-build in Groundsmith (never pre-skipped) — candidate: Excel/CSV lookup replacing the hardcoded EmployeeId≤10 manager constant.
- **Workflow:** ONE cloud Workflow model (`employee-training-request-approval`) — 2× Assign Task (manager, finance-manager), 2× reject→Send Email routes, 1× Generate Document of Record (single merged template, replacing the legacy DDX/AssemblerService/OutputService chain), 1× approved Send Email with the DoR PDF attached.
- **DSC scope:** confirmed NONE — the .LCA has no DSC/.jar, so no DSC→OSGi migration step exists; create-form-component is out of scope for custom-code migration on this run.
- **Tests:** create-form-tests for the submit action + any prefill Java service; test-form-ui against the DAM preview JPEGs (DocRoute.xdp for the AF, EmployeeTrainingRequest.xdp + ApprovalInfo.xdp for the DoR).

### ADLC Execution Plan

| Phase | Skill | Status | Note |
|---|---|---|---|
| 0 | ensure-forms-agents-md | verify only | config already present |
| 1 | generate-schema | run | from employeeTrainingRequest.xsd |
| 2 | create-editable-template | **SKIP** | reuse blank-af-v2 |
| 3 | create-adaptive-form | run | employee-training-request, 5 panels |
| 4 | create-form-rules | run | 8 rules |
| 5 | create-form-component | **SKIP** | no gap; no DSC code to migrate |
| 6 | create-submit-action | run | "Invoke an AEM Workflow" |
| 7 | create-prefill-service | ask-then-build (Groundsmith) | EmployeeId→Manager lookup candidate |
| 8 | create-form-theme | run (build new) | exact-replica palette |
| 9 | create-form-clientlib | **SKIP (default)** | only if Rule Editor can't reproduce alert UX |
| 12 | create-workflow | run | manager→finance approval + DoR + emails |
| 10 | create-form-tests | run | submit action + prefill service |
| 13 | test-form-ui | run | vs. DAM preview JPEGs (3 references) |
| 11 | migrate-form | spine (Path B) | executed across the phases above via formwright/groundsmith, not a separate run |

Parallelizable: 8 with 4; 7 with 6; 10 with 13.

Pipeline division of labor: **formwright** (build) covers phases 1, 3, 4, 8, and the DoR template inside phase 12's build; **groundsmith** (integration) covers phases 6, 7, and the workflow-model wiring inside phase 12. Then assembler → forgemaster → sentinel proceed as standard.

---

## 4. Gaps & open questions (must be resolved with the business before/at build)

1. **Manager auto-assign is hardcoded** in the legacy XFA (`EmployeeId<=10` → `venuga5`/"Arjun Venugopal"). Do not carry this literal into the cloud rule — resolve via a real lookup (prefill service) before build.
2. **Finance-manager approval participant is unresolved** — the source reuses the SAME `ManagerId` xpath as the manager step, meaning no distinct finance-approver rule exists today. Must be confirmed with the business; the architecture explicitly does not invent a resolution.
3. **Legacy email sender address** (`rs.gbseforms@medtronic.com`) needs a real cloud-project sender identity before go-live.
4. No stated submission volume/SLA — assumed low/internal; revisit if given.

`gaps: []` in the skill sense (every requirement maps to a real catalog skill) — the above are business decisions, not missing tooling.

---

## 5. Gate status

**PASS** — pending user go-ahead below. All user stories have acceptance criteria (0 missing). All requirements map to catalog skills (0 unmapped gaps). Blocking business questions (manager lookup, finance-approver participant, sender address) are recorded as open questions for Groundsmith/business sign-off, not treated as blockers to planning — they do not require re-running discovery, only confirmation before the corresponding build step executes.

---

## Run metrics

- `time_taken_minutes`: ~14.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~9,000
  - `read`: ~48,000 (XDP/process/xsd reads, grep scans, 3 DAM preview JPEGs)
  - `write`: ~11,000 (user-stories.yaml, solution-architecture.yaml, planwright.md)
  - `other`: ~6,000 (tool-call overhead, zip extraction, bash/grep output)
  - `total`: ~74,000

---

## Handoff YAML

```yaml
agent: planwright
phase: PLAN
status: PASSED
time_taken_minutes: 14.0
tokens_consumed:
  cli_text: 9000
  read: 48000
  write: 11000
  other: 6000
  total: 74000
delivery_type: migration
produces:
  structured_requirements: present
  user_stories: present
  solution_architecture: present
  integration_nfr_strategy: present
  adlc_execution_plan: present
user_stories_count: 11
stories_without_acceptance_criteria: 0
phases_planned: [0, 1, 3, 4, 6, 7, 8, 12, 10, 13]
phases_skipped: [2, 5, 9]
gaps: 0
user_confirmed_plan: false   # pending — see below
gate_result: PASS
next: aem-forms-program-agent executes the adlc_execution_plan (formwright -> groundsmith -> assembler -> forgemaster -> sentinel), after user confirms this plan and the 3 open questions are resolved or explicitly accepted as build-time follow-ups
```
