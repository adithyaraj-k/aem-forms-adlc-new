# Sentinel Test Report — employee-training-request-approval (Fix pass 12 / D15)

**Run:** 2026-07-28-employee-training-request
**Delivery type:** FIX / REMEDIATION verification (workflow task-open defect AEM-FD-008-013)
**Phase:** TEST · pipeline `formwright → groundsmith → assembler → forgemaster → sentinel`
**Instance under test:** localhost:4502 author (admin/admin) — confirmed up, HTTP 200
**Date:** 2026-07-28

---

## VERDICT & GATE: **FAIL**

| Success criterion | Result |
|---|---|
| (a) Manager Approval task OPENS clean — no AEM-FD-008-013 red box, ZERO RELATIVE_PLOAD / "Not able to get input data JSON" for the request | **PASS** |
| (b) **THE FILLED FORM IS VISIBLE** — READ_ONLY_AF review form renders the submitted values (not blank) | **FAIL (blocking)** |

The original RELATIVE_PLOAD crash **is fixed** (a). But the end goal — a reviewer opening the task
and seeing the **prefilled form** — **is NOT met** (b). A **new, distinct** blocking error now stops
the review form from rendering. Because (b) is the critical criterion, the delivery gate is **FAIL**.
This independently confirms the user's report: *"when I view payload in Manager Approval notification
from workflow, the prefilled document is not present."*

---

## What was proven

### Model state (deployed runtime, v1.29) — the fix landed
Both Assign Task nodes of `/var/workflow/models/employee-training-request-approval` (version **1.29**)
are clean:

| Node | icDataSourceType | INPUT_DATAJSON | OUTPUT_DATAJSON | *_COMBINED_DATAJSON | RELATIVE_PLOAD |
|---|---|---|---|---|---|
| node2 Manager Approval | PROVIDEDATADOCUMENT | data.xml | data.xml | **absent** | **none** |
| node4 Finance Manager Approval | PROVIDEDATADOCUMENT | data.xml | data.xml | **absent** | **none** |

(node7 "Send Approval Notification" carries `attachmentPathHiddenField=RELATIVE_PLOAD:DocumentofRecord/DoR.pdf`
— a DoR-PDF attachment reference on an email step, unrelated to the task-open data-read path.)

### Fresh instance started on the FIXED model
- **Workflow instance:** `/var/workflow/instances/server0/2026-07-20/employee-training-request-approval_126`
  — state RUNNING, initiator admin, started 2026-07-28 12:07:18 IST, **modelVersion 1.29**,
  payloadType JCR_PATH.
- **Payload:** `/var/fd/dashboard/payload/server0/2026-07-20/GUNJERI456UYALKSHH2SS3P3QU_67`
  (`CONTENT_TYPE=JSON`; `data.xml` present, HTTP 200).
- **Submitted data (data.xml JSON):** `Email=adithya@gmail.com`, `RequestId=1`, `RequestType=Training`,
  `TrainingMode=Classroom`, `Currency=GBP`, `StartDate=2026-07-06`, `TrainingCategory=Optional`,
  `Declaration.AcceptedTerms=true`, `ApprovalInfo.CurrentStatus=Submitted`.
- **Work item / task:** `.../employee-training-request-approval_126/workItems/node2_var_workflow_instances_server0_2026-07-20_employee-training-request-approval_126`
  — nodeId **node2 (Manager Approval)**, status **ACTIVE**, assignee **admin**.

> Note: the fresh v1.29 instance was already present on the instance (initiated by the user's own
> submit at 12:07:18); it is a genuine fresh instance on the fixed model and is the one tested. (A
> parallel synthetic-payload approach was started but abandoned when the real v1.29 instance was found.)

### Criterion (a) — task opens clean — PASS
Opened the exact GET that used to crash:
`GET /aem/dashboard/formdetails.html?item=%2Fvar%2Fworkflow%2Finstances%2Fserver0%2F2026-07-20%2Femployee-training-request-approval_126%2FworkItems%2Fnode2_...`
→ **HTTP 200**, 82 KB. Server request id **`[1785221667107]`**.

Log window scoped strictly to that request id (`crx-quickstart/logs/error.log`):
```
12:24:27.130 INFO  FormsSubmissionServiceImpl  getFormDataForWorkItem execution started
12:24:27.134 INFO  WorkSpacePayLoadManagerImpl getDataXMLForWorkItem   execution started   <- the class that used to throw
12:24:27.135 INFO  CoercionUtils               convertToObject         execution started
12:24:27.136 INFO  FormsSubmissionServiceImpl  getFormDataForWorkItem execution completed  <- data-read OK, no RELATIVE_PLOAD
```
- **RELATIVE_PLOAD in request `[1785221667107]`: 0**
- **"Not able to get input data JSON file" in request: 0**
- **AEM-FD-008-013 / "An error occurred while opening the task" in response: 0**

The 22 `RELATIVE_PLOAD` hits in the surrounding wall-clock window all belong to the **old v1.28**
instance `_125` (payload `3MABBNA3KEY26UNY5BYRJ3NNHQ_66`), which is stuck in a `JobHandler` retry
loop at node6 — pre-existing buggy-model instances, NOT this v1.29 request, and NOT tied to any
`formdetails.html` GET (0). The fix removed the crash for new (v1.29) instances.

### Criterion (b) — filled form visible — FAIL (blocking)
During the **same** request `[1785221667107]`, after the data-read completed and the form path
resolved, the review-form render failed:
```
12:24:27.357 *ERROR* [ ... [1785221667107] GET /aem/dashboard/formdetails.html ... ]
  libs.fd.dashboard.tm.gui.components.workitemdetails.formview.formview$jsp
  Error in getting data java.lang.NullPointerException:
  Cannot invoke "org.apache.sling.api.resource.Resource.getParent()" because "parentResource" is null
```
Evidence the prefilled form does NOT render:
- Rendered `formdetails.html` response contains **ZERO** submitted values:
  `adithya@gmail.com` = 0 occurrences, `RequestId` = 0, `EmployeeName` = 0.
- **No** adaptive-form markup: `guideContainer` = 0, `cmp-adaptiveform*` = 0, `adaptiveform` = 0,
  `READ_ONLY` = 0. No form iframe/render src.
- The error fires in `formview.jsp` at `getFormRenderPathForActiveWorkItem` — the READ_ONLY_AF review
  form for the work item cannot be rendered.
- User-confirmed independently in a real browser: prefilled document not present.

The form itself is healthy (`/content/forms/af/aem-adaptive-form-agents/employee-training-request`
= 200, its guideContainer = 200, DAM guide asset = 200) and the resolver returns the correct form
path — so this is **not** a missing-form or data-read problem; it is a **render-resource resolution**
failure inside the FD dashboard task formview.

---

## Defect (new — distinct from the fixed RELATIVE_PLOAD crash)

**ID:** SENTINEL-D15-01
**Severity:** Critical (blocks the delivery's purpose — reviewer cannot see the submitted form)
**Symptom:** Manager Approval task opens (HTTP 200, no red box) but the prefilled review form is blank.
**Root-cause pointer:** `libs.fd.dashboard.tm.gui.components.workitemdetails.formview.formview.jsp`
throws `NullPointerException: Resource.getParent() because "parentResource" is null` during
`FormsSubmissionServiceImpl.getFormRenderPathForActiveWorkItem` — the READ_ONLY_AF render resource
for the work item resolves to null. Data-read (`getDataXMLForWorkItem`/`getFormDataForWorkItem`) and
form-path resolution both succeed; the failure is in building the form-**render** resource for the
Assign Task, not in the JSON data source that Fix pass 12 corrected.
**Route to:** **Groundsmith** (workflow / Assign-Task AF-render integration). The Assign Task step /
payload render-node wiring must produce a resolvable READ_ONLY_AF render resource so `formview.jsp`
can attach the JSON data and render the filled form. Re-test only after Forgemaster redeploys.

---

## Criterion 5 — Approve/Reject routing: NOT VERIFIED
Not exercised. (1) The form does not render, so there is no usable task to route from; (2) per the
documented **headless OR-split limitation** for this model, Approve/Reject branch routing requires
the Workflow Editor UI and cannot be completed headlessly. Stated as a known limitation — **not**
claimed as passed.

---

## Housekeeping note (informational, out of scope)
Many stale RUNNING instances on old model versions (v1.23–v1.28, e.g. `_125`) are looping in
`JobHandler` errors and generating continuous `RELATIVE_PLOAD` / `WorkflowModel.getNode ... wfmodel
is null` log noise. They predate the fix and should be terminated/cleaned so the log is quiet for
future verification. Not a fix-pass defect.

---

## Handoff
```yaml
agent: sentinel
phase: TEST
status: FAILED
defect_verified: true
criteria:
  a_task_opens_clean: PASS      # v1.29 request [1785221667107]: 0 RELATIVE_PLOAD, no AEM-FD-008-013, data-read completed
  b_filled_form_visible: FAIL   # formview.jsp NPE 'parentResource is null'; 0 submitted values rendered; user-confirmed blank
workflow_instance: /var/workflow/instances/server0/2026-07-20/employee-training-request-approval_126   # v1.29, RUNNING
work_item: node2 Manager Approval (ACTIVE, assignee admin)
payload: /var/fd/dashboard/payload/server0/2026-07-20/GUNJERI456UYALKSHH2SS3P3QU_67   # data.xml JSON present
routing_approve_reject: NOT_VERIFIED   # form not rendering + documented headless OR-split UI-only limitation
defects:
  - id: SENTINEL-D15-01
    severity: critical
    summary: "READ_ONLY_AF review form blank in Manager Approval task; formview.jsp NPE parentResource null in getFormRenderPathForActiveWorkItem"
    route_to: groundsmith
gate_result: FAIL
next: aem-forms-program-agent routes SENTINEL-D15-01 to Groundsmith; re-test after Forgemaster redeploys
```

---

# POST-DEPLOY RE-TEST (Fix pass 13 / D15 — model v1.32) — 2026-07-28 ~14:55 IST

**Scope:** value-level verification of criterion (b) on the DEPLOYED v1.32 fix (FORM_RESOLUTION=PATH +
AF_PATH=DAM guide-asset path on both Assign Task nodes). Instance localhost:4502 confirmed up (HTTP 200).

## VERDICT: **(a) PASS · (b) FAIL (new distinct root cause)** — gate remains **FAIL**

| Criterion | Result |
|---|---|
| (a) Manager Approval task OPENS clean — form-render RESOLVES, ZERO getParent NPE / RELATIVE_PLOAD / AEM-FD-008-013 | **PASS — the D15 NPE (SENTINEL-D15-01) is FIXED** |
| (b) Filled review form shows the ACTUAL SUBMITTED values | **FAIL — form renders but binds the WRONG data (foreign prefill), not the submitted values** |

**Filled form visible with real submitted values: NO.**

## Deployed model confirmed (v1.32)
`/var/workflow/models/employee-training-request-approval` **version 1.32**. Both Assign Task nodes
(node2 Manager Approval, node4 Finance Manager Approval): `PROCESS=…AssignFormStep`,
`FORM_RESOLUTION=PATH`, `AF_PATH=/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request`,
`FORM_TYPE=READ_ONLY_AF`, `icDataSourceType=PROVIDEDATADOCUMENT`, `INPUT_DATAJSON=data.xml`. Transitions
are strictly LINEAR (node2→node3→node4→…→node9→node10) — **no OR-split wired** (see routing note).

## Fresh submission with distinctive values (recorded exactly)
Submitted headlessly via the CC runtime `POST /adobe/forms/af/submit/<b64>` (urlencoded `data=`) → **HTTP 200**,
thank-you returned. Values entered:
`EmployeeId=EMP-90210`, `EmployeeName=Zephyrine Q Blackwood`, `Email=zephyrine.blackwood@sentineltest.example`,
`Department=Quantum Robotics`, `ManagerId=MGR-77042`, `ManagerName=Dorian Vexley`, `Location=Reykjavik HQ`,
`BusinessUnit=Deep Space RnD`, `RequestId=REQ-SENTINEL-20260728-XY9`, `RequestType=Conference`,
`ProviderName=Helios Institute`, `CourseName=Advanced Cryogenic Systems 7000`, `StartDate=2026-09-14`,
`EndDate=2026-09-18`, `TrainingMode=Hybrid`, `Location=Geneva`, `Amount=4287.65`, `Currency=EUR`,
`TrainingCategory=Mandatory`, `AcceptedTerms=true`.
- **Workflow instance:** `/var/workflow/instances/server0/2026-07-20/employee-training-request-approval_133` — RUNNING, **modelVersion 1.32**, initiator admin.
- **Work item:** `node2_var_workflow_instances_server0_2026-07-20_employee-training-request-approval_133` — nodeId node2 (Manager Approval), status ACTIVE, assignee admin.
- **Payload:** `/var/fd/dashboard/payload/server0/2026-07-20/3MYAQPYUSHEB5Z4C7URSQPGXYY_69` — `data.xml` present.
- **Payload `data.xml` contains ALL distinctive values** (verified): EMP-90210, Zephyrine Q Blackwood, REQ-SENTINEL-20260728-XY9, Advanced Cryogenic Systems 7000, Helios Institute, 4287.65, EUR, etc. → submission + persistence are correct.

## Criterion (a) — render resolution NPE is FIXED (PASS)
`GET /aem/dashboard/formdetails.html?item=<work item>` → **HTTP 200 (~82 KB)**. Log request `[1785230542091]`:
```
14:52:22.097 FormsSubmissionServiceImpl  getFormDataForWorkItem            execution started
14:52:22.098 WorkSpacePayLoadManagerImpl getDataXMLForWorkItem             execution started
14:52:22.098 FormsSubmissionServiceImpl  getFormDataForWorkItem            execution completed
14:52:22.135 FormResolverUsingPath       getResolvedFormPath … resolved form path is /content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request
14:52:22.164 FormsSubmissionServiceImpl  getFormRenderPathForActiveWorkItem execution started   <- threw NPE in v1.29
14:52:22.172 FormsSubmissionServiceImpl  getFormRenderPathForActiveWorkItem execution COMPLETED  <- now clean, no NPE
```
- **`getParent` NPE = 0 · NullPointerException = 0 · RELATIVE_PLOAD = 0 · AEM-FD-008-013 = 0** in the request window.
- Task-detail iframe present: `src=/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request/jcr:content?wcmmode=disabled&dataRef=fdtask://…` (DAM guide-asset path = the AF_PATH the fix set), `data-isEditableAF='true'`.
- Inner AF render (that iframe src) → **HTTP 200**, **44× `guideContainer` + 248× `cmp-adaptiveform`** markup nodes, 0 NPE. Only a benign cosmetic content-policy WARN (title policy under a fragment template), no ERROR.
> SENTINEL-D15-01 (the `getFormRenderPathForActiveWorkItem` `parentResource is null` NPE) is RESOLVED — the READ_ONLY_AF review form now renders.

## Criterion (b) — submitted values NOT shown (FAIL — new defect SENTINEL-D15-02)
Verified in a REAL JS-executing browser (Cypress/electron, admin session), opening the same task detail,
intercepting the data call and scraping the inner AF DOM:
- guideBridge issued `GET /adobe/forms/af/data/<b64>?wcmmode=disabled&dataRef=fdtask://<fresh token>` → **HTTP 200**.
- **Response body was the WRONG data:** `{"data":{"businessRegistration":{"applicant":{},"address":{"state":"Tamil Nadu","country":"India"}}},"metadata":{…}}` — this is the foreign **`BusinessRegistrationPrefillService`** (defaultState=Tamil Nadu, defaultCountry=India from its `.cfg.json`), NOT the work item's `data.xml`.
- **Zero** distinctive submitted values in the af/data body or the rendered field DOM. Rendered inputs held only defaults: `acceptedTerms=on`, `currentStatus=Submitted` (schema default), plus hidden `:formstart` / `:redirect`. Employee name, RequestId, course, amount, etc. = EMPTY.
- **Scoping proof this is a dataRef-triggered prefill hijack:** the plain standalone form load `GET /adobe/forms/af/data/<b64>?wcmmode=disabled` (NO dataRef) returns clean `{"data":{}}` — the hijack fires ONLY when the `fdtask://` dataRef is present, i.e. specifically on the workflow task-render path.
- Evidence files (Cypress working dir, not copied into runs/):
  - `ui.tests/test-module/cypress/results/d15-afdata-body.json`
  - `ui.tests/test-module/cypress/results/d15-dom-values.json`
  - screenshot `ui.tests/test-module/cypress/results/screenshots/employee-training-request-d15-postdeploy-values.cy.js/form-ui/d15-postdeploy-manager-taskdetail.png`
  - spec `ui.tests/test-module/cypress/e2e/employee-training-request-d15-postdeploy-values.cy.js`

**Net:** the v1.32 fix genuinely advanced the state — the review form now RENDERS (NPE gone) — but the end
goal is still unmet: a reviewer opening the Manager Approval task sees an EMPTY training-request form because
the af/data resolution for the `fdtask://` work-item dataRef returns a foreign prefill DataProvider's static
defaults instead of the submitted `data.xml`. This defect was previously MASKED by the D15 render NPE (the
form never rendered far enough to reveal it).

### Defect SENTINEL-D15-02 (new)
- **Severity:** Critical (blocks the delivery purpose — reviewer cannot see submitted values).
- **Symptom:** Manager Approval review form renders but is empty; `/adobe/forms/af/data/…?dataRef=fdtask://…`
  returns `BusinessRegistrationPrefillService` static data (Tamil Nadu/India), not the work item `data.xml`.
- **Root-cause pointer:** a custom `DataProvider` (`com.aem.forms.agents.forms.prefill.BusinessRegistrationPrefillService`,
  `@Component(service=DataProvider.class, immediate=true)`, and 4 sibling prefill providers from prior deliveries) is
  resolving the training-request form's dataRef-bearing af/data request although the form declares NO prefill service.
  Its `getPrefillData` ignores `options.getDataRef()` and returns address defaults, pre-empting the `fdtask://`
  work-item-data resolution. The `fdtask` dataRef must resolve to the work item `data.xml` (which is correct on disk),
  not be overridden by an unrelated prefill provider.
- **Route to:** **Groundsmith** (prefill / DataProvider integration). The task-render data binding must resolve the
  `fdtask://` dataRef to the work item data; gate/scope the custom prefill DataProviders so they do not intercept
  other forms' (or the task-render) af/data. Re-test after Forgemaster redeploys.

## OR-split / Approve-Reject routing — NOT exercised (unchanged)
`/var` model v1.32 transitions are LINEAR — the Approve/Reject reject-shortcut OR-split is NOT wired in the runtime
model. Branch routing was NOT exercised and is NOT claimed to pass; it requires a manual Workflow Editor wire + Sync
(documented headless OR-split limitation for this model).

## Manual browser steps for the user to confirm criterion (b) independently
1. Log in to http://localhost:4502 (admin/admin).
2. Open http://localhost:4502/aem/inbox (or the FD dashboard) → find the "Manager Approval" task for the
   "Employee Training Request — Approval" instance `_133` (assignee admin).
3. Open it and observe the review form. Expected-if-fixed: fields show `Zephyrine Q Blackwood`,
   `REQ-SENTINEL-20260728-XY9`, `Helios Institute`, `Advanced Cryogenic Systems 7000`, `4287.65 EUR`, etc.
   Observed now: the form renders but those fields are EMPTY (only the accept-terms + status defaults set).

## Handoff (post-deploy)
```yaml
agent: sentinel
phase: TEST
status: FAILED
model_version_tested: "1.32"
criteria:
  a_task_opens_and_form_renders_no_npe: PASS   # request [1785230542091]: getFormRenderPathForActiveWorkItem completed; 0 getParent NPE / RELATIVE_PLOAD / FD-008-013; 44 guideContainer + 248 cmp-adaptiveform rendered
  b_filled_form_shows_submitted_values: FAIL   # af/data(fdtask dataRef)=200 but returns BusinessRegistrationPrefillService data (Tamil Nadu/India), not data.xml; 0 submitted values in DOM
filled_form_visible_with_real_values: NO
workflow_instance: /var/workflow/instances/server0/2026-07-20/employee-training-request-approval_133   # v1.32 RUNNING
work_item: node2 Manager Approval (ACTIVE, assignee admin)
payload: /var/fd/dashboard/payload/server0/2026-07-20/3MYAQPYUSHEB5Z4C7URSQPGXYY_69   # data.xml holds all distinctive submitted values (verified)
d15_01_render_npe: FIXED
routing_approve_reject: NOT_VERIFIED   # /var model linear, OR-split not wired; headless UI-only limitation
defects:
  - id: SENTINEL-D15-02
    severity: critical
    summary: "Manager Approval review form renders but shows no submitted values; af/data for the fdtask work-item dataRef returns BusinessRegistrationPrefillService static defaults (Tamil Nadu/India) instead of the work item data.xml"
    route_to: groundsmith
gate_result: FAIL
next: aem-forms-program-agent routes SENTINEL-D15-02 to Groundsmith; re-test after Forgemaster redeploys
```

---

# POST-DEPLOY RE-TEST (Fix pass 14 / D15-02 prefill-defer fix — model v1.32) — 2026-07-28 ~17:20 IST

**Scope:** value-level verification that the prefill-defer fix (all 5 custom DataProviders now DEFER on
foreign/`fdtask://` dataRefs) makes the Manager Approval review form bind the SUBMITTED work-item values.
Instance localhost:4502 confirmed up (HTTP 200); core bundle `aem-adaptive-forms-agents.core` **Active**.

## VERDICT: **(b) FAIL** — gate remains **FAIL**. Filled form visible with real submitted values: **NO.**

The businessRegistration hijack (D15-02) IS resolved — af/data no longer returns Tamil Nadu/India — BUT
the review form still renders EMPTY. With all custom providers deferring, a NEW, distinct root cause is now
exposed: the FDM Integrated-Content prefill provider intercepts the `fdtask://` work-item dataRef, errors,
and the work-item `data.xml` never binds → af/data returns `{"data":{}}`.

## Fresh submission (distinctive values, recorded)
Submitted via CC runtime `POST /adobe/forms/af/submit/<b64>` (CSRF token + urlencoded `data=`) → **HTTP 200**,
thank-you returned. Values entered:
`EmployeeId=EMP-73519`, `EmployeeName=Thaddeus O Ravenscroft`, `Email=thaddeus.ravenscroft@sentinel-d16.example`,
`Department=Photonics Division`, `ManagerId=MGR-31088`, `ManagerName=Isolde Marchetti`, `Location=Helsinki Lab/Zurich`,
`BusinessUnit=Advanced Optics`, `RequestId=REQ-SENTINEL-20260728-Z14`, `RequestType=Conference`,
`ProviderName=Nimbus Aerospace Academy`, `CourseName=Photonic Lattice Engineering 9200`, `StartDate=2026-10-05`,
`EndDate=2026-10-09`, `TrainingMode=Hybrid`, `Amount=7731.40`, `Currency=EUR`, `TrainingCategory=Mandatory`, `AcceptedTerms=true`.
- **Workflow instance:** `/var/workflow/instances/server0/2026-07-20/employee-training-request-approval_134` — RUNNING, model v1.32, initiator admin, started 17:19:38 IST.
- **Work item:** `node2_...employee-training-request-approval_134` — nodeId node2 (Manager Approval), status ACTIVE, assignee admin.
- **Payload:** `/var/fd/dashboard/payload/server0/2026-07-20/HOKZZAD4N6NXSH5ZS2DOO6P3VM_70` — `data.xml` contains ALL distinctive values (verified on disk: EMP-73519, Thaddeus O Ravenscroft, REQ-SENTINEL-20260728-Z14, Photonic Lattice Engineering 9200, Nimbus Aerospace Academy, 7731.40, EUR, Isolde Marchetti). Submission + persistence CORRECT.

## Criterion (a) — task opens / form-render resolves, D15-01 stays fixed (PASS)
`GET /aem/dashboard/formdetails.html?item=<_134 work item>` → **HTTP 200 (82,869 bytes)**. In the task-open
window (error.log since baseline): **getParent NPE = 0 · RELATIVE_PLOAD = 0 · AEM-FD-008-013 = 0 · "Not able
to get input data JSON" = 0**. `getFormRenderPathForActiveWorkItem` started→**completed** with no NPE (requests
`[1785239528363]`, `[1785239763245]`). SENTINEL-D15-01 remains RESOLVED.

## Criterion (b) — submitted values NOT shown (FAIL — new defect SENTINEL-D15-03)
Verified BOTH by direct curl AND in a REAL JS browser (Cypress/electron, admin session, spec
`employee-training-request-d16-postfix-values.cy.js`):
- guideBridge issued `GET /adobe/forms/af/data/<b64=/content/forms/af/.../employee-training-request>?wcmmode=disabled&dataRef=fdtask://<fresh token gzQ22fRct...>` → **HTTP 200**, body **`{"data":{},"metadata":{...}}`** — EMPTY.
- **Zero** submitted values AND **zero** businessRegistration values in the af/data body (hijack gone; nothing bound).
- Rendered inner-iframe DOM held only defaults: `acceptedTerms=on`, `currentStatus=Submitted` (+ hidden `:formstart`/`:redirect`). Employee name, RequestId, course, amount = EMPTY.
- **Root cause (error.log, request `8ff9f66a-...`, token matches the browser af/data URL exactly):**
  ```
  *ERROR* com.adobe.aem.forms.ic.data.impl.DefaultIcFdmService [IC] [Prefill] Incorrect form data model path set fdtask://gzQ22fRct.../{02b09b9f...}
  com.adobe.aem.dermis.exception.DermisException: javax.jcr.RepositoryException: Invalid name or path:
    fdtask://gzQ22fRct.../{02b09b9f...}/jcr:content/jcr:lastModified
    at com.adobe.aem.dermis.core.service.slingmodel.FormDataModelManager.getFormDataModelDAMAssetInCache(...)
  ```
  The platform's own `FormsDashboardPrefillServiceImpl.getDataXMLForDataRef` runs and completes cleanly (it CAN
  resolve the token), but the af/data prefill provider chain lets `DefaultIcFdmService` (FDM Integrated-Content
  prefill — active because FDM is enabled project-wide) and `DraftPrefillService` run against the `fdtask://`
  dataRef; the FDM provider treats the dataRef as an FDM DAM asset path, throws, and the correct work-item data
  never wins/binds → empty. This was previously MASKED by the businessRegistration provider returning first.
- Evidence (Cypress working dir, NOT copied into runs/):
  - `ui.tests/test-module/cypress/results/d16-afdata-body.json` (af/data URL + body + needle check)
  - `ui.tests/test-module/cypress/results/d16-dom-values.json`
  - screenshot `ui.tests/test-module/cypress/results/screenshots/employee-training-request-d16-postfix-values.cy.js/form-ui/d16-postfix-manager-taskdetail.png`
  - spec `ui.tests/test-module/cypress/e2e/employee-training-request-d16-postfix-values.cy.js`

## Criterion 4 — NO-REGRESSION (PASS)
business-registration own on-load prefill still works: `GET /adobe/forms/af/data/<br-b64>?wcmmode=disabled`
(NO dataRef) → **HTTP 200**, `{"data":{"businessRegistration":{"applicant":{},"address":{"state":"Tamil Nadu","country":"India"}}},...}`.
The defer guard (dataRef null + own serviceName → provider answers) did NOT break legitimate own-form prefill.

## OR-split / Approve-Reject routing — NOT exercised (unchanged)
`/var` model v1.32 transitions are LINEAR — the Approve/Reject OR-split is NOT wired in the runtime model. Not
exercised, NOT claimed to pass; requires a manual Workflow Editor wire + Sync (documented headless OR-split limitation).

### Defect SENTINEL-D15-03 (new — distinct from D15-02)
- **Severity:** Critical (blocks the delivery purpose — reviewer sees an EMPTY review form).
- **Symptom:** Manager Approval review form renders but is empty; `/adobe/forms/af/data/…?dataRef=fdtask://…`
  returns `{"data":{}}` (no longer the businessRegistration defaults, but not the work-item data either).
- **Root-cause pointer:** with the 5 custom prefill DataProviders now deferring, the af/data provider chain routes
  the `fdtask://` work-item dataRef to `com.adobe.aem.forms.ic.data.impl.DefaultIcFdmService` (FDM Integrated-Content
  prefill, active because FDM is enabled project-wide), which throws `DermisException: Invalid name or path` on the
  `fdtask://` ref; `FormsDashboardPrefillServiceImpl.getDataXMLForDataRef` resolves the token cleanly but its result
  never binds. The `fdtask://` work-item dataRef must resolve to the payload `data.xml` and be kept out of the FDM
  IC / Draft prefill providers for the task-render af/data call.
- **Route to:** **Groundsmith** (prefill / FDM data-binding integration). Re-test after Forgemaster redeploys.

## Handoff (post-deploy, fix pass 14)
```yaml
agent: sentinel
phase: TEST
status: FAILED
model_version_tested: "1.32"
prefill_defer_fix: EFFECTIVE_BUT_INSUFFICIENT   # businessRegistration hijack removed; own-form prefill intact
criteria:
  a_task_opens_and_form_renders_no_npe: PASS    # 0 getParent NPE / RELATIVE_PLOAD / FD-008-013; getFormRenderPathForActiveWorkItem completed
  b_filled_form_shows_submitted_values: FAIL    # af/data(fdtask)=200 body {"data":{}}; DefaultIcFdmService DermisException on fdtask ref; 0 submitted values in DOM
filled_form_visible_with_real_values: NO
no_regression_business_registration_prefill: PASS   # standalone af/data returns Tamil Nadu/India own prefill
workflow_instance: /var/workflow/instances/server0/2026-07-20/employee-training-request-approval_134   # v1.32 RUNNING
work_item: node2 Manager Approval (ACTIVE, assignee admin)
payload: /var/fd/dashboard/payload/server0/2026-07-20/HOKZZAD4N6NXSH5ZS2DOO6P3VM_70   # data.xml holds all distinctive values
d15_01_render_npe: FIXED
d15_02_businessregistration_hijack: FIXED
routing_approve_reject: NOT_VERIFIED   # /var model linear, OR-split not wired; headless UI-only limitation
defects:
  - id: SENTINEL-D15-03
    severity: critical
    summary: "Manager Approval review form empty; af/data for fdtask work-item dataRef returns {\"data\":{}} because DefaultIcFdmService (FDM IC prefill) intercepts the fdtask ref and throws DermisException 'Invalid name or path'; work-item data.xml never binds"
    route_to: groundsmith
gate_result: FAIL
next: aem-forms-program-agent routes SENTINEL-D15-03 to Groundsmith; re-test after Forgemaster redeploys
```
