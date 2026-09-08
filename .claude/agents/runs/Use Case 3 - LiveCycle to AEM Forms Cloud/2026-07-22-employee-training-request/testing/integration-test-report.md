# Integration / Functional / Workflow Test Report — Employee Training Request

Run: `.claude/agents/runs/2026-07-22-employee-training-request/`
Scope: live verification against `http://localhost:4502` of (1) the 8 migrated business rules,
(2) submit → workflow invocation, (3) the workflow model's approval routing, (4) the DoR generation
step. All checks performed via `guideContainer.model.json`, raw JCR node inspection
(`.infinity.json`), a real submission attempt against the AF REST submit endpoint
(`/adobe/forms/af/submit/...`), the AEM error log, and the live `/var/workflow/models/...` +
`/conf/global/settings/workflow/models/...` definitions. No browser-automation tool was available in
this session (no `claude-in-chrome` MCP tools were actually exposed despite the skill listing), so
click-through UI interaction was not performed; every finding below is evidence from the live
runtime JSON/JCR state and a real REST-level submission attempt, not inference from source code.

---

## 1. The 8 migrated business rules — live verification (formwright.md §4 flagged items)

| # | Rule | Mechanism | Live status |
|---|---|---|---|
| 1 | ManagerId/ManagerName lock on EmployeeId≤10 | `fd:change` script on `employeeDetailsFragment` (host wrapper around the `employee-identity` fragment) | **BROKEN** — see Defect D-R1 |
| 2 | Email format validate | `validatePictureClause` regex on the email field | **WORKS** — confirmed in `guideContainer.model.json`: `"validatePictureClause":"^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+.[A-Za-z]{2,}$"` |
| 3 | RequestType locked until RequestId non-empty | `fd:enabled` (ENABLE_EXPRESSION), `$form.trainingRequestDetailsPanel.requestId` | **WORKS** — live model shows `"rules":{"enabled":"requestId.$value != \"\""}` on `requestType` |
| 4 | EndDate > StartDate validate | `validationExpression` calling `validateDateAfter()` (form clientlib custom function) | **WORKS** — live model shows `"validationExpression":"validateDateAfter($field.$value, $form.trainingRequestDetailsPanel.startDate.$value) == true()"` on `endDate` |
| 5 | Approval Information panel show/hide on EmployeeId + CurrentStatus | `fd:visible` (SHOW_EXPRESSION) on `approvalInformationPanel`, referencing `employeeDetailsFragment.employeeId` (a field nested inside the embedded fragment) | **WORKS** — live model shows `"rules":{"visible":"employeeDetailsFragment.employeeId.$value != \"\" && currentStatus.$value != \"FinanceApproved\""}`. This is the **first proven case in this repo of a host-form rule correctly reading a field nested inside an embedded fragment.** |
| 6 | FinanceComments lock/unlock on CurrentStatus=="ManagerApproved" | `fd:enabled` (ENABLE_EXPRESSION) on `financeComments` | **BROKEN** — see Defect D-R6 |
| 7 | AcceptedTerms required | native `required`+`enum:[true]` on the checkbox | **WORKS** — confirmed |
| 8 | SubmissionDate set-value on submit | inline script on the Submit button's `click` event | **WORKS** — live model shows `"events":{"click":["declarationFragment.submissionDate.$value = new Date().toISOString()"," submitForm()"]}` on `submitButton`; `submissionDate` is `readOnly:true` |

**Result: 6/8 rules verified working live; 2/8 confirmed broken.** Formwright's own flagged
lowest-confidence items were rules #1 and #5 (fragment-nested-field references) plus #3/#6
("follow-the-pattern", not yet demonstrated). The actual outcome: #5 and #3 — both correctly
verified — work; #1 and #6 — the two rules whose "then" action needs a live reactive binding
(a lock/unlock, not a single boolean expression) — are broken.

### Defect D-R1 (Critical) — ManagerId/ManagerName lock rule does not fire live
`employeeDetailsFragment`'s authored rule (`.content.xml` `fd:rules/fd:change`) is a multi-statement
`Change`-event script:
```
if (Number(employeeId.$value) > 0 && Number(employeeId.$value) <= 10) {
  managerId.enabled = false; managerName.enabled = false;
} else {
  managerId.enabled = true; managerName.enabled = true;
}
```
It is attached to the **host wrapper panel** that embeds the `employee-identity` fragment (`fd:path
.../employeeDetailsFragment`), the exact "no prior in-repo precedent" pattern Formwright flagged in
`formwright.md` §4. Live evidence: neither the host `guideContainer.model.json` nor the fragment's
own `employee-identity/jcr:content/guideContainer.model.json` shows a `"change"` key under `events`
for `employeeDetailsFragment` / `employeeId` — compare with the Submit button's `click` script (rule
#8), which DOES compile into `"events":{"click":[...]}` live. The `Change`-event script targeting
fields nested inside an embedded fragment from the host wrapper's rule scope simply does not attach
at runtime. `managerId`/`managerName` render permanently `enabled:true` (never locked) for every
EmployeeId value.
**Impact:** TC-003, TC-004 fail; US-02 has zero passing cases (uncovered).
**Route to:** Formwright (`create-form-rules` / `create-AdaptiveFormFragment`) — per Formwright's own
recommendation, round-trip this rule through the Rule Editor UI on the `employee-identity` fragment
consumer to find a compiling equivalent (e.g. authoring the lock directly inside the fragment's own
rules, keyed on its own sibling `employeeId`, rather than on the host wrapper).

### Defect D-R6 (Critical) — FinanceComments lock/unlock rule does not fire live
`financeComments`'s authored top-level `enabled` property is the bare expression string
`"currentStatus.$value == \"ManagerApproved\""`. Comparing its JCR node
(`.../approvalInformationPanel/financeComments.infinity.json`) against the **working** sibling
pattern on `requestType` (`.../trainingRequestDetailsPanel/requestType.infinity.json`): the working
node's `fd:rules` block carries a companion simple-expression string as a **direct sibling of
`fd:enabled`** inside `fd:rules` (`"fd:rules":{"enabled":"requestId.$value != \"\"", "fd:enabled":[...]}`).
`financeComments`'s `fd:rules` block is **missing that companion string** — it has only
`fd:enabled` (the full AST) and `validationStatus`, no `enabled` sibling. Without it, the AF runtime
never produces the live `"rules":{"enabled":...}` binding seen on `requestType`/
`approvalInformationPanel`; `financeComments` instead renders a **static, one-time-evaluated**
`"enabled":false` in `guideContainer.model.json` that never re-evaluates as `CurrentStatus` changes.
**Impact:** TC-014, TC-015 fail; US-07 has zero passing cases (uncovered).
**Route to:** Formwright (`create-form-rules`) — re-author the `fd:enabled` rule on `financeComments`
through the Rule Editor so it produces the same `fd:rules`-nested simple-expression sibling that
`requestType`'s (working) rule has, then re-verify via `guideContainer.model.json`.

---

## 2. Submit → workflow invocation — live attempt

A real submission was attempted against the live REST endpoint with a **fully schema-valid**
payload (all required fields per `employee-training-request.schema.json`, zero attachments):

```
POST /adobe/forms/af/submit/L2NvbnRlbnQvZm9ybXMvYWYvYWVtLWFkYXB0aXZlLWZvcm0tYWdlbnRzL2VtcGxveWVlLXRyYWluaW5nLXJlcXVlc3Q=
(multipart, data=<valid JSON>, CSRF token attached)
→ HTTP 502
{"detail":"com.adobe.aem.forms.af.rest.exception.ExternalSubmitException:
 {\"errorCausedBy\":\"FORM_SUBMISSION\",\"errorMessage\":\"Unable to save file attachment for
 workItem - {0}\",\"originCode\":\"500\"}"}
```

AEM `error.log` root cause:
```
com.adobe.fd.workspace.exceptions.FormsWorkflowException: Unable to save file attachment for workItem
  at WorkSpacePayLoadManagerImpl.saveTaskAttachments
  at WorkSpacePayLoadManagerImpl.saveAttachmentsUnderPayloadOrVariable
  at WorkSpacePayLoadManagerImpl.createInitialFormData
  at FormsSubmissionServiceImpl.triggerFormsWorkflow
  at AFNativeSubmitServiceImpl.submit
  at FormSubmitActionManagerServiceImpl.submit (x2, overload)
  at AdaptiveFormSubmitServlet.doPost
```

Confirmed via `GET /var/workflow/instances.json?...modelId=employee-training-request-approval` →
**`[]` (zero instances)** both before and after the attempt. The submission never reaches the
workflow trigger; **it fails inside Adobe's own stock `aemworkflowsubmit`("Invoke an AEM Workflow")
action**, specifically in the OOTB attachment-folder-creation step of the initial workflow payload —
this fires **even with zero actual attachment files submitted**.

**Control check performed:** the identical multipart POST mechanism was verified working against
`business-registration` (a form using a different, non-workflow submit action) — it returned
`HTTP 200` with a thank-you redirect, ruling out a generic curl/multipart/CSRF issue. A repo-wide
search confirms `employee-training-request` is the **only** form in this project ever wired to
`aemworkflowsubmit` (`fd/dashboard/components/actions/aemworkflowsubmit`) — this is the first live
exercise of the "Invoke an AEM Workflow" submit action + `attachmentsType=FOLDER_PAYLOAD` combination
in this project, and it does not work as configured.

### Defect D1 (Critical) — Submit action fails server-side; workflow never starts
**Impact:** TC-002, TC-020 fail outright (0 workflow instances from any real submission attempt);
cascades to block live verification of TC-011, TC-012, TC-013, TC-016, TC-017, TC-026, TC-030, TC-032
— none of these can be exercised because no workflow instance can ever be created. US-01/AC-01.2 has
zero passing live-verification cases.
**Route to:** Groundsmith (`create-workflow` / the `aemworkflowsubmit` wiring) — the
`attachmentsFolderPath="attachments/"` + `attachmentsType="FOLDER_PAYLOAD"` combination on the
`guideContainer`, paired with `workflowModel`, needs either a different attachments configuration
(e.g. confirm the target DAM/content folder for `WorkSpacePayLoadManagerImpl` exists and is
writable, or switch `attachmentsType`) or an Adobe support/config fix for the OOTB action. Re-test
end-to-end only after a real submission returns 200 and produces ≥1 workflow instance.

---

## 3. Workflow model — approval routing (structural, since D1 blocks any live instance)

`/conf/global/settings/workflow/models/employee-training-request-approval` (design, via
`jcr:content.infinity.json`) DOES define `orsplit_manager` and `orsplit_finance` nodes
(`sling:resourceType: cq/workflow/components/model/orsplit`) between each Assign Task step and its
following Set-Status step — 11 nodes total, matching `groundsmith.md`'s description.

However, the **synced runtime copy** (`/var/workflow/models/employee-training-request-approval.json`,
`lastSynced: 2026-07-22T14:03:05+05:30`) contains **11 nodes but ZERO of type `OR_SPLIT`** — both
`orsplit_manager` and `orsplit_finance` are absent from the runtime node list entirely, and the
runtime `transitions` array is **fully linear with no conditional `rule` on any transition**:
`Start → Capture Submission Variables → Manager Approval (task) → Set Status—Manager Approved →
Finance Manager Approval (task) → Set Status—Finance Approved → Generate Document of Record → Send
Approval Notification → Set Status—Rejected → Send Rejection Notification → End`.

Root cause: the design-time OR-split nodes are **empty placeholders** — `.content.xml`/live JSON
shows each `orsplit_*` node with only `jcr:primaryType`, `jcr:title`, `sling:resourceType`; **no
child route/rule nodes** (the usual `route0`/`route1` children carrying a `WORKFLOW_ROUTE_RULE`
against the `actionTaken` variable) were ever authored. An OR-split with no routing children appears
to be dropped by the design→runtime sync, collapsing the model to one straight-through path.

**Practical effect if D1 were fixed today:** every workflow instance — regardless of whether the
manager or finance approver clicks Approve or Reject on their Assign Task — would unconditionally
execute ALL of: Generate Document of Record, Send Approval Notification, Set Status—Rejected, AND
Send Rejection Notification, in that fixed order, every time. A Reject outcome would still generate
and "approve-email" the DoR before also sending a rejection email. The `AssignTaskStep` nodes
correctly capture the chosen route into `routeVariable=actionTaken` (`routes=Approve,Reject`), but
nothing downstream consumes that variable to actually change path.

### Defect D2 (Critical) — Workflow has no real Approve/Reject branching
**Impact:** TC-012 fails (no routing exists); TC-017 fails (DoR would fire even on rejection,
violating "NO Document of Record" on reject). Independent of, and compounding, D1 — fixing D1 alone
would not make TC-012/TC-017 pass.
**Route to:** Groundsmith (`create-workflow`) — author real route-rule children under
`orsplit_manager` / `orsplit_finance` keyed on the `actionTaken` variable (`Approve` → continue to
the next Set-Status step; `Reject` → jump to `Set Status—Rejected` / `Send Rejection Notification` /
`End`, skipping the remaining approval + DoR + approval-email nodes), then re-sync `/conf` → `/var`
and re-verify the runtime model actually carries `OR_SPLIT` nodes with conditional transitions.

---

## 4. What DID verify correctly (functional/structural, independent of D1/D2)

- **Workflow model participants** — both `AssignTaskStep` nodes (`Manager Approval`,
  `Finance Manager Approval`) are statically configured `assignee=administrators,
  assigneeType=STATIC` — matches the confirmed business decision (TC-013's config-level bar), though
  the live task-landing check itself could not run (blocked by D1).
- **Workflow variables** — `managerId`, `applicantEmail`, `requestId`, `actionTaken`, `CurrentStatus`,
  `rejectionReason` all correctly sourced from the submitted payload paths
  (`${payload.jcr:content/data/EmployeeDetails/ManagerId}` etc.) in `Capture Submission Variables`.
- **Generate Document of Record step** targets the dedicated
  `/content/forms/af/aem-adaptive-form-agents/employee-training-request-dor` page (not the
  interactive form) — correct per Formwright's flag.
- **DoR template composition** — `/content/forms/af/aem-adaptive-form-agents/employee-training-request-dor`
  renders (HTTP 200) and its `guideContainer.model.json` confirms it reuses **both** fragments
  (`employee-identity`, `declaration-consent` — same `fragmentPath` as the interactive form) plus its
  own `trainingRequestDetailsPanel`/`justificationPanel`/`approvalInformationPanel`, and correctly
  **excludes** the Attachments panel. Fragment reuse is by live reference (`fragmentPath`), so
  "edit-once-reflects-both" (TC-028/TC-029) holds by construction — verified structurally on both
  fragments in both consumers.
- **Prefill — confirmed "none" as designed.** ManagerId/ManagerName render as ordinary empty,
  always-enabled text inputs (no fabricated auto-population for any EmployeeId), consistent with
  Groundsmith's documented decision. Note: TC-031's literal wording (expects a returned
  ManagerId/ManagerName pair for EmployeeId≤10) does not hold, since there genuinely is no lookup —
  this is a **test-case/architecture mismatch against an already-accepted decision**, not a new code
  defect; flagged for a future DESI test-case revision rather than bounced to Groundsmith for a fix.
- **Mail Service / Send Email steps** — configuration reused correctly (existing project
  `DefaultMailService.cfg.json`), but could not be exercised at all (TC-032) since no instance ever
  reaches a `SendEmail` node (blocked by D1).

---

## 5. Summary

| Defect | Severity | Owner | Blocks |
|---|---|---|---|
| D-R1 — ManagerId/ManagerName lock rule doesn't compile live | Critical | Formwright | TC-003, TC-004; US-02 uncovered |
| D-R6 — FinanceComments enable rule doesn't compile live | Critical | Formwright | TC-014, TC-015; US-07 uncovered |
| D1 — Submit action fails server-side (0 workflow instances) | Critical | Groundsmith | TC-002, TC-011, TC-012, TC-013, TC-016, TC-017, TC-020, TC-026, TC-030, TC-032; US-01/AC-01.2, US-06/AC-06.2, US-08 severely impacted |
| D2 — Workflow has no Approve/Reject branching (OR-splits empty/dropped on sync) | Critical | Groundsmith | TC-012, TC-017 (independent of D1) |

**4 Critical, independently-confirmed defects.** None of these were caught by the build/deploy
gates (BUILD SUCCESS, 112/112 unit tests, all bundles Active) because none of them are unit-testable
without live rule-editor compilation and a real end-to-end workflow submission — exactly what
Sentinel exists to catch, and exactly the live verification Formwright's own §4 and this task's brief
asked for.

---

## Retest — fix pass 1 (2026-07-22, independent re-verification against the redeployed instance)

Scope: retest D-R1, D-R6, D1, D2 from scratch against `http://localhost:4502` — live
`guideContainer.model.json` (host form AND the DoR template), a fresh independent Cypress-driven
end-to-end submission through the **embedded page** (not a hand-crafted REST payload), the
`/var/workflow/models/employee-training-request-approval.json` runtime model, and the actual
submitted `data.xml` payload content (not just "did a workflow instance get created").

### D-R1 — ManagerId/ManagerName lock rule: **FIXED, confirmed live**

`guideContainer.model.json` now shows a real reactive binding on both fields, matching the exact
mechanism class of the previously-proven-working `requestType` rule:
```
managerId:   "rules":{"enabled":"Number(employeeId.$value) <= 0 || Number(employeeId.$value) > 10"}
managerName: "rules":{"enabled":"Number(employeeId.$value) <= 0 || Number(employeeId.$value) > 10"}
```
Browser-level confirmation (`employee-training-request-functional.cy.js`, RT-02): typing
EmployeeId=7 (1-10) → both fields become `disabled`; changing to EmployeeId=55 (>10) → both become
enabled again. **Genuinely fixed**, not just JSON-present-but-inert as D-R1 was originally.

### D-R6 — FinanceComments lock rule: **FIXED, confirmed live**

`guideContainer.model.json` now shows:
```
financeComments: "rules":{"enabled":"currentStatus.$value == \"ManagerApproved\""}
```
— the exact missing binding class from the original defect, now present and structurally identical
to the working `requestType`/`managerId` pattern. Browser-level confirmation (RT-06): on the
employee's own render, CurrentStatus is (by design) `readOnly:true` — it is a
system/workflow-driven field only editable from the manager/finance-manager's Assign Task work-item
view, not this page — so the unlock transition itself cannot be legitimately exercised from the
employee-facing render. The initial locked state (CurrentStatus=Submitted → financeComments
disabled) was confirmed live, and the reactive **binding's presence** (the actual proof point
originally used to diagnose this defect as broken) is now confirmed compiled and correct.

### D1 — Submit → workflow HTTP 502: **FIXED, confirmed live via an independent end-to-end submission**

Built and ran `ui.tests/test-module/cypress/e2e/employee-training-request-functional.cy.js` — a
real browser-driven fill-and-submit through the **embedded "Test Adaptive Form" page**, not a
hand-crafted REST payload (to avoid relying on Groundsmith's own claimed verification). Result:
```
POST /adobe/forms/af/submit/L2NvbnRlbnQvZm9ybXMvYWYvYWVtLWFkYXB0aXZlLWZvcm0tYWdlbnRzL2VtcGxveWVlLXRyYWluaW5nLXJlcXVlc3Q=
→ HTTP 200 (was 502)
```
followed by a genuine thank-you page render (`guideContainer.guideThankYouPage.html`). Confirmed via
`GET /var/workflow/instances.json?modelId=...` that a **new** instance
(`employee-training-request-approval_9`) was created, `state: RUNNING`, sitting at `node2`
(Manager Approval) — exactly the expected first work item. `attachmentsFolderPath` confirmed
`"attachments"` (no trailing slash) on the live `guideContainer.json`. **D1 is genuinely fixed** —
independently reproduced by Sentinel, not merely carried forward from Groundsmith's own claim.

### D2 — OR-split branching: **CONFIRMED STILL NOT FIXED (as expected — Groundsmith's fix pass explicitly left this open)**

`GET /var/workflow/models/employee-training-request-approval.json` → still exactly **11 nodes, 0 of
type `OR_SPLIT`**, fully linear transitions, `lastSynced` advanced to `2026-07-22T15:25:58` (a fresh
resync happened) but the shape is unchanged from the original defect. Exactly matches Groundsmith's
own fix-pass report ("root-caused, escalated, needs Workflow Editor UI completion — do not report as
resolved"). **This is the one item retested here that behaves exactly as expected: still broken,
by design of the fix pass, pending human Workflow-Editor-UI access.**

### NEW — D3 (Critical, not one of the 5 items in scope): fragment fields serialize to the WRONG schema paths on every real submission

While inspecting the actual submitted `data.xml` from my own end-to-end submission (not just
"did an instance get created"), a previously-undiscovered defect surfaced:

```
{"EmployeeDetails":{},"employee":{"id":"77","name":"Retest Employee","email":"retest.employee@example.com",
 "department":"Engineering","managerId":"MGR-900","managerName":"Retest Manager","location":"Bengaluru"},
 ...
 "Declaration":{},"declaration":{"acceptedTerms":true} }
```

`EmployeeDetails` and `Declaration` — the schema's actual property names — are **always empty**.
The real data lands under unschema'd, lowercase `employee.*` / `declaration.*` keys instead. Root
cause confirmed via `guideContainer.model.json` (both the host form AND the DoR template
`employee-training-request-dor`, which reuses the same fragment by `fragmentPath`): every field
inside the two Adaptive Form Fragments carries a **fragment-internal generic `dataRef`**, not the
host schema's path:
```
employeeId      -> dataRef: $.employee.id            (schema expects $.EmployeeDetails.EmployeeId)
managerId        -> dataRef: $.employee.managerId      (schema expects $.EmployeeDetails.ManagerId)
managerName      -> dataRef: $.employee.managerName    (schema expects $.EmployeeDetails.ManagerName)
email/department/location/businessUnit -> all $.employee.*, same mismatch
acceptedTerms    -> dataRef: $.declaration.acceptedTerms (schema expects $.Declaration.AcceptedTerms)
submissionDate   -> dataRef: $.declaration.submissionDate (schema expects $.Declaration.SubmissionDate)
```

**Why this wasn't caught before:** the original Sentinel pass never got past the D1 502 to inspect
a real payload. Groundsmith's own "verified live x2" submissions used **hand-crafted, schema-shaped
JSON fed directly to the REST endpoint** (`23D56CJEW6EJRLVSBFD25KZLQE_3`,
`PMBWEMRXJNIPMVRYHTLPQF4X2E_4` — both show clean `EmployeeDetails`/`Declaration` data) — that
bypasses the actual fragments' live `dataRef` entirely, since a hand-built payload can put data
anywhere. This retest is the **first genuine browser-driven, fragment-rendering-and-serializing**
submission this delivery has ever completed, and it exposes the mismatch that a synthetic payload
cannot.

**Downstream impact (confirmed, not speculative):**
1. **Document of Record (AC-11.1) would be generated incomplete** — the DoR template
   (`employee-training-request-dor`) embeds the identical `employee-identity` fragment with the
   identical broken `dataRef`s, so the merged PDF's Employee Details + Declaration sections would
   render blank for every real submission, even once D2 is fixed and the workflow can reach that
   step.
2. **Workflow variable capture is broken for real submissions.** `Capture Submission Variables`
   sources `managerId` from `${payload.jcr:content/data/EmployeeDetails/ManagerId}` and
   `applicantEmail` from `${payload.jcr:content/data/EmployeeDetails/Email}` — both **always
   resolve empty** for a real submission, since the actual data lives under `employee.managerId` /
   `employee.email`. `managerId` being empty is currently masked (assignee is the static
   `administrators` group, not dynamically resolved), but `applicantEmail` empty means **every
   outcome email (approved/rejected) would have no valid recipient** once the workflow reaches a
   Send Email step — a second, independent reason (beyond D2 and SMTP-not-configured-locally) those
   notifications can never be confirmed end-to-end on this environment, and a real defect once SMTP
   is configured in a real environment.
3. **SubmissionDate never gets set, for two compounding reasons.** The submit button's `click`
   event script (`declarationFragment.submissionDate.$value = new Date().toISOString()`) throws a
   live browser console error on every submit attempt: `Unable to compile expression ... Exception:
   ParserError: Unexpected token type: UnquotedIdentifier, value: Date` — the rule engine's simple-
   expression parser does not support the `new Date()` constructor syntax, so this assignment never
   executes (the second action in the same click handler, `submitForm()`, still fires independently
   — this is why submit itself is unaffected). Even if it did compile, it would write to
   `declaration.submissionDate`, not the schema's `Declaration.SubmissionDate`. **This corrects the
   original pass's TC-019 "WORKS" verdict**, which was based only on the rule's *presence* in
   `guideContainer.model.json`, not on actually exercising it in a browser — the first time this
   delivery has done so.

**Route to:** Formwright (`create-AdaptiveFormFragment`) — the `employee-identity` and
`declaration-consent` fragments' field `dataRef`s must be corrected to bind to the host schema's
actual property paths (`EmployeeDetails.*`, `Declaration.*`), not a fragment-internal generic
namespace; and the submit button's `SubmissionDate` script needs to use a rule-engine-supported date
expression (not a raw `new Date()` constructor) once the dataRef is corrected.

**This is a newly-discovered Critical defect, not part of the 5-item retest scope**, and it changes
the shape of the remaining gap: it is **not** true that OR-split routing (D2) is the only thing
standing between this delivery and a clean gate. D3 independently blocks AC-11.1 (US-11) and taints
the workflow's own captured variables, regardless of D2's status.

---

## FINAL retest — fix pass 2 (2026-07-22, independent re-verification against the second redeploy)

Scope: verify Formwright's fix-pass-2 claims (D3 fragment `dataRef` correction, TC-019 `SubmissionDate`
ParserError fix) for real, end-to-end, against the redeployed instance — plus a full regression of
everything already confirmed (D-R1, D-R6, D1, rules #2–#5/#7/#8, submit-gated-on-validation) and a
re-confirmation of D2's status. Method: a fresh Cypress spec
(`employee-training-request-final-retest.cy.js`, 5 tests, all passing) driving the **embedded page**
end-to-end with its own brand-new submission (not reusing any prior instance's data), plus direct live
inspection of the resulting workflow instance's persisted payload, the live `guideContainer.model.json`
(host form + DoR template), the live aggregated clientlib JS as actually served to the browser, and the
live workflow model. A second, narrowly-focused Cypress probe
(`employee-training-request-submissiondate-probe.cy.js`) was written specifically to chase down an
anomaly found in the first payload inspection (see below).

### D3 — fragment `dataRef` fix: **CONFIRMED GENUINELY FIXED, via a real submission's persisted payload**

A fresh browser-driven submission (marker `FinalRetest1784723667132`, EmployeeId=91) created workflow
instance `employee-training-request-approval_11`
(`/var/workflow/instances/server0/2026-07-20/employee-training-request-approval_11`, state `RUNNING`
at `node2` = Manager Approval — the correct first work item). Its persisted payload
(`GET /var/fd/dashboard/payload/server0/2026-07-20/6TSX6ZXBGFHBPTHIZAY3UV3354_6/data.xml`) is:

```json
{"EmployeeDetails":{"EmployeeId":"91","EmployeeName":"FinalRetest1784723667132Employee",
 "Email":"finalretest.employee@example.com","Department":"Quality Engineering",
 "ManagerId":"MGR-FinalRetest17847","ManagerName":"Manager FinalRetest1784723667132","Location":"Chennai"},
 "TrainingRequest":{...all populated...},
 "Justification":{...all populated...},
 "Declaration":{"AcceptedTerms":true},
 "ApprovalInfo":{"CurrentStatus":"Submitted"}, ...}
```

`EmployeeDetails.*` and `Declaration.AcceptedTerms` are populated at their correct schema paths — the
**exact opposite** of the original D3 finding (`employee.*`/`declaration.*`). Zero `employee.*` /
`declaration.*` residue anywhere in the payload. **D3's core claim — the fragments now write to the
right schema roots — is genuinely fixed, independently reproduced, not carried forward from
Formwright's or Forgemaster's own claim.**

**DoR completeness (task item #2), verified structurally:** the DoR template's own live
`guideContainer.model.json` (`/content/forms/af/aem-adaptive-form-agents/employee-training-request-dor`)
shows the identical corrected bindings (`employeeDetailsFragment/employeeId` → `$.EmployeeDetails.EmployeeId`,
… `declarationFragment/acceptedTerms` → `$.Declaration.AcceptedTerms`, etc. — all 10 fields). Since (a)
the DoR template binds to the same schema paths and (b) a real submission's payload now actually
populates those paths, the DoR's Employee Details section and the Declaration/AcceptedTerms field would
render populated for the first time — **not blank**, as they would have been pre-fix. Actually driving a
submission through both Assign Task steps to render a real DoR PDF was not attempted this session — AEM
does not expose a simple REST "complete work item" operation (a POST with `:operation=complete` to the
work item path returns `500 Invalid operation specified for POST request`); completing a work item
requires the Workflow Inbox UI, the same tooling gap already recorded against D2. This structural
verification (schema-path match on both the write side and the read side, using a real payload) is the
strongest evidence obtainable without that UI.

**`applicantEmail` / `managerId` workflow variables (task item #3), verified by config+data cross-check:**
the live workflow model's `Capture Submission Variables` step (`node1`, `PROCESS_AUTO_ADVANCE=true`, so it
already ran for instance `_11`) sources:
```
managerId       = ${payload.jcr:content/data/EmployeeDetails/ManagerId}
applicantEmail  = ${payload.jcr:content/data/EmployeeDetails/Email}
```
— unchanged from the original design (this was already correctly authored; only the *data* was
previously in the wrong place). The fresh submission's persisted payload has real values at exactly
those paths (`EmployeeDetails.ManagerId = "MGR-FinalRetest17847"`, `EmployeeDetails.Email =
"finalretest.employee@example.com"`). Both halves of the equation — the variable-capture step's source
path, and the actual data's location — now agree, so `managerId`/`applicantEmail` would capture real
values for this instance. **Caveat, reported honestly:** no simple REST endpoint exposes the workflow
instance's live variable *values* directly (the `.json`/`.infinity.json` instance views return only
id/state/payload/workItems, not `workflowData`; the workflow console's instance-details HTML view did
not surface them either via a direct fetch). This conclusion is inferred with high confidence from two
independently-verified facts (source-path config + actual payload structure), not read directly off a
live variable — flagged as a verification-method limitation, not a defect.

### TC-019 — SubmissionDate: **PARSER ERROR CONFIRMED FIXED, but a SECOND, NEW defect found underneath it — the value never actually reaches the submitted data**

The literal ParserError (`Unexpected token type: UnquotedIdentifier, value: Date`) is gone — confirmed
no such error appears anywhere in this session's Cypress console capture across two full spec runs.
`getCurrentDateISOString()` is present in the live aggregated clientlib JS
(`/etc.clientlibs/clientlibs/employee-training-request-clientlib.lc-....min.js`, confirmed by fetching
it directly) and is callable: a direct browser-console call via `cy.window()` returned a real ISO
string (`2026-07-22T12:44:06.862Z`). **So far, this matches Formwright's and Forgemaster's fix-pass-2
claim.**

However, going one level deeper than either of them did — actually exercising the assignment and
inspecting its effect, rather than stopping at "does it compile / is the function callable" — surfaced
that the assignment **never lands**:

1. **Live DOM value, both before and immediately after clicking Submit:** a focused probe spec
   (`employee-training-request-submissiondate-probe.cy.js`) read `input[name="submissionDate"]`'s value
   immediately before and immediately after the click (racing the navigation). Both reads: `""`
   (empty). If the assignment had executed, the second read should show a populated ISO string, since
   `submissionDate`'s `readOnly:true` does not prevent programmatic `$value` writes from rendering (its
   sibling `CurrentStatus`, also `readOnly:true`, DOES reliably show its system-set value in the
   payload — see below).
2. **The outgoing network request body**, captured via `cy.intercept` before the request ever reaches
   the server: `"Declaration": {"AcceptedTerms": true}` — no `SubmissionDate` key at all. This proves
   the gap is client-side (the browser never set/sent the value), not a server-side drop.
2. **The persisted JCR payload** (both my `FinalRetest` submission and this probe's submission,
   independently): `"Declaration":{"AcceptedTerms":true}` — `SubmissionDate` absent, not even as an
   empty string.
3. **Control check ruling out "readOnly fields get dropped":** `CurrentStatus` is also `readOnly:true`
   on the employee's own render, and it DOES appear in the persisted payload
   (`"ApprovalInfo":{"CurrentStatus":"Submitted"}`) — so `readOnly` is not the reason `SubmissionDate`
   is missing; specifically this one field's assignment silently fails to have any effect.

**All three independent evidence sources agree.** The ParserError fix is real (the literal compile-time
bug named in the original defect is gone), but a second, previously-masked defect in the same rule is
now exposed: the click-time assignment `declarationFragment.submissionDate.$value =
getCurrentDateISOString()` compiles and the function itself runs correctly, but assigning into a field
that lives in a **different top-level panel than the button issuing the assignment**
(`declarationFragment` vs. the button's own `actionsPanel`) without an explicit `$form.` scope prefix
does not actually update the runtime field/model — the same *class* of cross-panel/cross-fragment
addressing defect as D-R1 (host rule referencing a field nested in a sibling fragment), just manifesting
as a silent no-op here instead of a compile error. (Compare: the working `endDate` validation rule
explicitly uses `$form.trainingRequestDetailsPanel.startDate.$value` — a full `$form.`-prefixed path —
for its own cross-field reference; the submit button's rule uses the shorter, unprefixed
`declarationFragment.submissionDate.$value`, which appears to be exactly the difference.) This is a
plausible, evidence-consistent root-cause hypothesis, not a confirmed fix — Formwright should verify
against the Rule Editor directly.

**Correcting the record:** TC-019 is **NOT** genuinely fixed end-to-end. Formwright's and Forgemaster's
fix-pass-2 verification (confirming the AST/JSON/JS syntax and the absence of a ParserError) was real
but incomplete — exactly the same class of gap as the original mis-verification this fix pass itself
was correcting (checking presence/compilability, not actual runtime effect). TC-019 must be marked
**FAIL** again, for a new, narrower reason.

**Impact:** `Declaration.SubmissionDate` is permanently blank for every real submission (client-side
issue, unrelated to and not reintroducing D3 — the `dataRef` is confirmed correct in
`guideContainer.model.json`: `$.Declaration.SubmissionDate`; the field simply never receives a value to
serialize). This means the DoR's Declaration section would render `AcceptedTerms` correctly but
`SubmissionDate` blank — a narrower completeness gap than the original D3 (which blanked the ENTIRE
Employee Details + Declaration sections), but still a real, unresolved data-completeness defect against
AC-11.1 and against US-09's documented rule "SubmissionDate is system-set at submit time." It does
**not** reopen US-09's coverage (TC-018, TC-020, TC-029 — the story's other cases — all independently
pass), so `uncovered_stories` is unaffected by this finding, but the case itself must fail per the
project's non-negotiable "never soft-pass" rule.

**New defect — call it D4** (fragment/cross-panel field addressing without `$form.` scoping silently
no-ops instead of erroring): **Route to Formwright** — either add the `$form.` prefix
(`$form.declarationFragment.submissionDate.$value = getCurrentDateISOString()`) to match the pattern
already proven working for `endDate`'s cross-panel reference, or author the assignment from within the
`declaration-consent` fragment's own scope (mirroring the D-R1 lesson: cross-boundary reactive logic is
more reliable authored from within the target field's own owning scope than from a distant host/sibling
panel). Re-verify via all three of the checks used here (live DOM value, outgoing request body, and the
persisted JCR payload) — not just AST/JSON presence — before claiming this fixed again.

---

## FINAL retest — fix pass 3 (2026-07-22, independent re-verification against the third redeploy)

Scope: verify Formwright's fix-pass-3 claim (the `$form.` scope-prefix correction on the submit
button's `SubmissionDate` click-time assignment) using the SAME 3 independent live methods that
originally caught D4, **plus a 4th, more targeted method added this pass specifically because the
first 3 are outcome-only signals** — a runtime spy on `window.getCurrentDateISOString` that proves
whether the function is *invoked at all* during the click event, not just whether its effect landed.
Also re-ran a full regression sweep (D-R1, D-R6, D1, D3, rules #2–#5/#7/#8, submit-gated-on-validation)
and re-checked D2's workflow-model status.

### D4 — SubmissionDate still never lands. Fix pass 3's `$form.` prefix did NOT resolve it.

**Confirmed the fix is genuinely live** at the rule/mirror level, exactly as Forgemaster reported:
```
GET .../employee-training-request/jcr:content/guideContainer/actionsPanel/submitButton.1.json
fd:events.click = ["$form.declarationFragment.submissionDate.$value = getCurrentDateISOString()", " submitForm()"]
```
the bare/unscoped form of the assignment is gone; the `$form.`-prefixed version is live in both the
`fd:click` JSON metadata `script` array and the `fd:events`/`click` mirror, matching Formwright's
fix-pass-3 description exactly.

**All 3 of the original evidence methods, re-run against a fresh submission (marker
`Probe`/EmployeeId=92, via `employee-training-request-submissiondate-probe.cy.js`), show the identical
symptom as before the fix:**

1. **Live DOM value, before and immediately after clicking Submit:** both reads `""` (empty) —
   unchanged from the pre-fix-pass-3 finding.
2. **The outgoing network request body** (captured via `cy.intercept` before the request left the
   browser): `"Declaration": {"AcceptedTerms": true}` — no `SubmissionDate` key. Full request body
   saved to `cypress/results/form-ui/submissiondate-probe-request-body.txt`.
3. **The persisted JCR payload**, fetched directly from the resulting workflow instance's payload
   node (`GET /var/fd/dashboard/payload/server0/2026-07-20/3PPMBVIF5F7CMOVBTYOZFMTCBM_10/data.xml`,
   instance `employee-training-request-approval_19`):
   ```json
   "Declaration":{"AcceptedTerms":true}
   ```
   `SubmissionDate` absent, not even as an empty string — identical to every prior check.

**4th method, new this pass — a function-call spy, added specifically because the first 3 methods are
outcome signals and don't distinguish "the function ran but its result didn't get written" from "the
function was never called at all":** a throwaway diagnostic Cypress spec wrapped
`window.getCurrentDateISOString` with a counting spy immediately after page load, then filled and
submitted the form and read the counter **synchronously right after the Submit click** (before waiting
on the network call, to avoid the post-submit page navigation invalidating the spied window).
Confirmed the spy was correctly installed (the wrapped function is callable and returns a real ISO
string when invoked directly — same positive result fix pass 2/3 already established). **Result:
`getCurrentDateISOString` call count = 0 after the click.** The function that fix pass 2 registered
and fix pass 3 re-targeted is **never invoked at all** when Submit is clicked, on this redeployed
instance — `submitForm()` (the click handler's second statement) still fires correctly (HTTP 200,
thank-you shown), so the click handler itself executes, but the SET_VALUE statement ahead of it does
not.

**This means fix pass 3's root-cause hypothesis (a missing `$form.` scope prefix causing the write to
resolve against the wrong scope and silently no-op) was not the actual defect, or was not the only
one.** The evidence now points one level deeper: the assignment statement itself never executes at
click time, regardless of how its target is scoped. Formwright's diagnosis correctly matched the
*symptom class* (the D-R1 pattern of a cross-panel/cross-fragment reference silently failing rather
than erroring) but the specific fix applied did not change the observed runtime behaviour at all —
the DOM value, the request body, and the persisted payload are byte-for-byte identical to the
pre-fix-pass-3 state. **D4 is NOT fixed.**

**Recommend for the next fix pass (if pursued):** don't re-guess the scope prefix again — instrument
the actual Rule Editor interpreter (or add a temporary `console.log` inside the generated click
handler) to confirm whether the `EVENT_SCRIPTS`/`SET_VALUE` block for this specific `fd:click` AST is
even being reached/parsed at runtime for THIS button, since the direct evidence here is "0 calls,"
not "wrong target." It is also worth testing whether a `Change`-event-style single-field-scoped rule
(authored inside `declaration-consent` itself, reacting to some available click/submit signal) succeeds
where a host-panel-scoped multi-statement `Click` script does not — mirroring the mechanism class that
already had to be abandoned for D-R1's original fix.

### Regression sweep — everything else confirmed still working, zero regressions

Re-ran `employee-training-request-final-retest.cy.js` (5 tests, all passing) against the fix-pass-3
redeploy, and independently fetched the resulting fresh submission's persisted JCR payload:

- **FR-01 (UI-Finding-1, Employee Details heading):** still renders. PASS.
- **FR-02 (D-R1, ManagerId/ManagerName lock):** EmployeeId=7 → both disabled; EmployeeId=55 → both
  re-enabled. PASS, no regression.
- **FR-03 (rules #2/#3/#4 — email format, RequestType lock, EndDate>StartDate):** all three still fire
  correctly. PASS, no regression.
- **FR-04 (D-R6, FinanceComments initial lock + rule #5 panel show/hide):** panel hidden while
  EmployeeId empty, shown + financeComments disabled once EmployeeId populated. PASS, no regression.
- **FR-05 (D1 + submit-gated-on-validation + D3):** empty/invalid submit blocked (no thank-you), a
  fresh valid submission (marker `FinalRetest1784727058077`) returns HTTP 200, shows the thank-you
  page, and creates a new `RUNNING` workflow instance (`employee-training-request-approval_27`). Its
  persisted payload (`/var/fd/dashboard/payload/server0/2026-07-20/ZU7R3PX43D4J64E324JISFYY3Q_14/data.xml`)
  confirms **D3 still resolved, no regression**: `EmployeeDetails.*` fully populated
  (`EmployeeId, EmployeeName, Email, Department, ManagerId, ManagerName, Location` all correct) and
  `Declaration.AcceptedTerms: true` — the exact same correct schema-path binding confirmed in the
  fix-pass-2 retest. `Declaration.SubmissionDate` is absent from this fresh payload too, consistent
  with the D4 finding above (not a new/different defect, the same one).
- **D2 (OR-split branching):** `GET /var/workflow/models/employee-training-request-approval.json` on
  this redeploy: still exactly **11 nodes** (1 `START`, 9 `PROCESS`, 1 `END`), **0 `OR_SPLIT` nodes**
  — byte-for-byte the same shape as every prior check across all 3 fix passes. Untouched, exactly as
  expected (fix pass 3's scope was D4 only).

**No regression anywhere.** D-R1, D-R6, UI-Finding-1, D1, D3, and submit-gated-on-validation all
continue to hold after the third redeploy.

### Summary — FINAL retest, fix pass 3

| Item | Status |
|---|---|
| D4 (SubmissionDate click-time assignment) | **STILL BROKEN — fix pass 3's `$form.` prefix did NOT resolve it.** Confirmed via all 3 original methods (DOM value, request body, persisted payload) PLUS a new 4th method (function-call spy: 0 invocations) that shows the assignment statement is never executed at all, not merely mis-scoped. |
| D-R1, D-R6, UI-Finding-1, D1, D3 | **NO REGRESSION** — all reconfirmed live via a fresh Cypress run and a fresh submission's persisted payload |
| D2 (OR-split branching) | **STILL NOT FIXED, exactly as expected** — untouched this cycle, unchanged assessment |

**One item remains open and is now confirmed NOT resolved by 3 consecutive fix passes' worth of
targeted attempts on the symptom (ParserError → silent no-op → wrong-scope hypothesis):** D4. D2
remains the second, separately-tracked, human-actionable item. `uncovered_stories` is unaffected by
D4 (US-09 keeps its coverage via TC-018/020/029; TC-019 itself fails, as it has for 2 consecutive
retests now, for evolving but never-resolved reasons).

### D1, D-R1, D-R6, UI-Finding-1 — regression-checked, all still hold (no regression from fix pass 2)

Re-verified live via the same fresh Cypress spec (`employee-training-request-final-retest.cy.js`, tests
FR-01 through FR-05), independently of anything Formwright/Forgemaster reported this cycle:
- **D-R1 (ManagerId/ManagerName lock):** FR-02 — EmployeeId=7 → both fields disabled; EmployeeId=55 →
  both re-enabled. Still works.
- **D-R6 (FinanceComments initial lock) + rule #5 (Approval Information panel show/hide):** FR-04 —
  panel hidden while EmployeeId empty, shown + `financeComments` disabled once EmployeeId populated
  and `CurrentStatus` still `Submitted`. Still works.
- **Rules #2/#3/#4 (email format, RequestType lock, EndDate>StartDate):** FR-03 — all three still fire
  correctly.
- **UI-Finding-1 (Employee Details heading):** FR-01 — still renders.
- **D1 (submit → workflow) + submit-gated-on-validation (critical rule 4b):** FR-05 — empty/invalid
  submit blocked (`[Custom-Submit-GeneratePDF] Submission blocked — form has validation errors. PDF not
  generated.` console warning, thank-you NOT shown), a fully valid submission returns HTTP 200, shows
  the thank-you page, creates a new `RUNNING` workflow instance, AND triggers a real
  `POST /bin/aem-adaptive-forms-agents/generatePDF` → 200 with a downloaded `Form-Submission.pdf`. Both
  halves of critical rule 4b (block+no-PDF on invalid, submit+PDF on valid) hold.

**No regression anywhere in the 4 previously-resolved items.**

### D2 — OR-split branching: **RECONFIRMED STILL NOT FIXED (as expected, untouched by fix pass 2)**

`GET /var/workflow/models/employee-training-request-approval.json` on the fix-pass-2 redeploy: still
exactly **11 nodes** (`START, PROCESS×9, END`), **0 of type `OR_SPLIT`**, and **0 of 10 transitions
carry a `rule`** — fully linear, byte-for-byte the same structural shape as every prior check. Exactly
as expected: fix pass 2's scope was D3 + TC-019 only, and explicitly did not touch workflow artifacts
(confirmed in Forgemaster's own redeploy log, §10.5: "OR-split branching gap remains the recorded,
human-actionable follow-up — untouched by this agent, per explicit instruction").

**Which stories D2 actually blocks (checked against `plan/user-stories.yaml`):** AC-06.2 ("Choosing
Approve routes... Reject sends...") and AC-08.1/AC-08.2 (DoR generation gated on the finance-manager's
actual Approve/Reject choice) are the two acceptance criteria that literally require real branching.
US-06 remains **covered** overall (TC-021/022/023/TC-011 pass on other ACs of the same story), but its
own AC-06.2 has zero passing cases. US-08's ENTIRE case set (TC-013, TC-016, TC-017) depends on reaching
a genuine finance-approval outcome and is therefore **still fully blocked** — this is the same,
unchanged, single uncovered story as the fix-pass-1 retest, now confirmed to be attributable **squarely
and only** to D2 (D3 no longer compounds it, per the D3 confirmation above).

### Summary — FINAL retest

| Item | Status |
|---|---|
| D3 (fragment dataRef → EmployeeDetails.*/Declaration.*) | **FIXED, confirmed via a real submission's persisted payload** |
| D3's downstream DoR-completeness impact | **RESOLVED for Employee Details + AcceptedTerms** (verified structurally, both write-side and read-side schema paths now agree) |
| D3's downstream applicantEmail/managerId impact | **RESOLVED** (verified via config+data cross-check; live variable value itself not directly readable with available tooling) |
| TC-019 ParserError (the literal named defect) | **FIXED** — confirmed gone from console across 2 full spec runs |
| TC-019 actual SubmissionDate assignment (found this pass, NOT part of the named defect) | **STILL BROKEN — new, narrower defect (D4)** — confirmed via 3 independent live methods |
| D-R1, D-R6, rules #2–#5/#7/#8, D1, submit-gated-on-validation | **NO REGRESSION** — all reconfirmed live this pass |
| D2 (OR-split branching) | **STILL NOT FIXED, exactly as expected** — untouched by this fix pass, human-actionable follow-up |

**Two items remain, not one:** D2 (OR-split — human-actionable, Workflow Editor UI, reasonable to
accept as a documented follow-up, unchanged assessment from the prior retest) AND **D4 (NEW — the
SubmissionDate assignment's cross-panel addressing, agent-fixable, should get one more narrow
Formwright pass)**. D4 does not reopen any coverage (`uncovered_stories` stays at `[US-08]`, unchanged)
but does mean TC-019 itself cannot be marked passing, so the case-level gate ("never soft-pass") remains
FAIL even though the coverage-level picture is unchanged from the previous retest.

---

## RETEST — fix pass 4 (2026-07-23, user-reported remediation — OOTB workflow rebuild + fragment schema binding)

Distinct from D2/D4: 3 user-reported issues (2 Formwright — fragment schema binding, exhaustive
script-inventory audit; 1 Groundsmith — rebuilding the workflow's Assign Task/Send Email/DoR steps
onto genuine, live-registered OOTB "Forms Workflow" classes instead of an unregistered scaffold
class). Deploy independently confirmed live first (see `test-report.md`'s "Deploy verification"
table) — not taken on the coordinator's or Forgemaster's word.

### Workflow model — direct live inspection (the authoritative check for this pass)

`GET /var/workflow/models/employee-training-request-approval.json`:

```
node2 / node4  (Manager / Finance Approval)  PROCESS = com.adobe.fd.workspace.step.service.AssignFormStep
node9          (Assemble Manager Confirmation) PROCESS = com.adobe.fd.workflow.assembler.InvokeDDXProcess
                                                ddx = /apps/aem-adaptive-forms-agents/workflow/ddx/employee-training-request-approval/manager-confirmation.ddx
node10         (Convert Manager Confirmation to PDF/A) PROCESS = com.adobe.fd.workflow.assembler.ConvertToPDFAProcess, compliance=1a
node6 / node8  (Send Approval / Rejection Notification) PROCESS = com.adobe.fd.workflow.email.SendEmailStep
transitions:   node0→1→2→3→4→5→6→7→8→9→10→11 — fully linear, 0 OR_SPLIT nodes, 0 transitions carry a rule
```

- **Assign Task class swap: CONFIRMED live**, and confirmed to be the REAL registered class (not a
  restatement of Groundsmith's audit — this is my own fresh `GET` against the current model).
- **Invoke DDX: CONFIRMED live**, and the DDX file it references is genuinely deployed:
  `GET .../manager-confirmation.ddx` → HTTP 200, body byte-for-byte
  `<XDP result="pdfComplete"><XDP source="EmployeeTrainingForm"/><XDP source="ApprovalInfo"/></XDP>`
  — the exact original legacy DDX, `jcr:created` timestamped **today**.
- **Send Email steps: CONFIRMED now `SendEmailStep`** (Forms-specific OOTB), not the prior generic
  `com.day.cq.workflow.process.SendEmail`.
- **D2 — RECONFIRMED STILL OPEN, independently, not on Groundsmith's word alone.** The live
  `transitions` array is fully linear with **zero** `OR_SPLIT` nodes anywhere — identical structural
  shape to every prior check in this delivery, now reproduced under the genuinely-OOTB step
  neighbors too. Both Send Email steps (approval AND rejection) fire unconditionally for every
  submission regardless of the Manager's actual decision.

### D1 — re-verified via a genuinely FRESH full-AF-submit-path test (stronger than a direct-instance-start proxy)

Ran the existing Cypress suite against the live, just-redeployed instance:

| Spec | Result |
|---|---|
| `employee-training-request-final-retest.cy.js` | **5/5 passing** — FR-05-valid-submit: real POST to `/adobe/forms/af/submit/...` → HTTP 200; FR-05-thankyou: thank-you shown; FR-05-empty-blocked: empty submit still correctly blocked |
| `employee-training-request-functional.cy.js` | **8/8 passing** — D-R1, D-R6, rules #2–#5/#7/#8, submit-gated-on-validation all regression-clean |
| `employee-training-request-submissiondate-probe.cy.js` | **1/1 passing** (informational probe) — captured request body confirms `EmployeeDetails`/`Declaration`/`TrainingRequest`/`Justification` all correctly nested (D3 regression-clean); `Declaration.SubmissionDate` still absent (D4 unchanged, not re-chased) |

Then confirmed, independent of the Cypress run's own HTTP-status assertion, that a **real new**
workflow instance was created by this submission:

```
GET /var/workflow/instances.json?model=/var/workflow/models/employee-training-request-approval
  -> new entry: employee-training-request-approval_30 (was absent before this session's Cypress run)
GET .../employee-training-request-approval_30.json
  "state": "RUNNING", "startTime": "Thu Jul 23 2026 11:04:06 IST"  (today, matches the Cypress run)
  "workItems": [{"node":"node2", ...}]   — Manager Approval, the REBUILT AssignFormStep-based step
```

**D1 does not regress — and this is now proven via the actual end-user submit path, not a
direct-instance-start proxy.** The Assign Task class swap (the single riskiest change in this fix
pass, since it changes the step's underlying implementation) creates a real work item exactly as
before.

### Tier-A editor-completion items — CONFIRMED genuinely incomplete on the live model (not just claimed)

Inspected every new/changed node's live `metaData` directly:

| Node | Keys present | Keys absent (Tier-A, confirmed empty) |
|---|---|---|
| `process_ddx` (Invoke DDX) | `validateOnly`, `ddxType`, `ddx`, `failOnError` | Input/Output Documents Map |
| `process_pdfa` (Convert to PDF/A) | `compliance` | Input Document / Output location |
| `process_email_approved`/`process_email_rejected` | `toAddressType`, `toAddressValue`, `emailSubject`, `templatePath` | attachment fields (`attachmentName`/`attachmentFileNameType`) |
| `assigntask_manager`/`assigntask_finance` | `STATIC_ASSIGNEE`, `ROUTES`, `AF_PATH`, `TASK_PRIORITY`, etc. | `INPUT_DATAXML`/`OUTPUT_DATAXML`/`INPUT_FORM_ATTACHMENTS`/`OUTPUT_FORM_ATTACHMENTS` |

All 4 Tier-A items Groundsmith flagged as deliberately left for editor completion are confirmed
genuinely incomplete on the live, deployed model — not guessed placeholders that were secretly wired,
and not items that got silently dropped either. Added to the delivery's open-items list as new,
non-blocking, human-actionable follow-ups (Workflow Editor UI required).

### Summary — retest fix pass 4

| Item | Status |
|---|---|
| Issue 1 (fragment schema binding) | **FIXED, confirmed live** (schemaType/schemaRef + DAM formmodel on both fragments; referenced schema resolves and matches) — editor canvas itself not visually re-confirmed (no browser automation available) |
| Issue 2 (script inventory audit) | **CLAIM HOLDS** — 4 of 14 rules independently spot-checked live, all genuinely present and structurally sound |
| Issue 3 (OOTB workflow rebuild) | **VERIFIED END-TO-END** — AssignFormStep/InvokeDDXProcess/SendEmailStep/ConvertToPDFAProcess all live; D1 re-proven via a fresh full-submit-path test |
| D2 (OR-split) | **STILL OPEN, independently reconfirmed** — fully linear, 0 OR_SPLIT nodes, unchanged |
| D4 (SubmissionDate) | **STILL OPEN, unchanged, NOT re-chased per instruction** — confirmed absent from the fresh submission's captured request body |
| D-R1, D-R6, rules #2–#5/#7/#8, D3, TC-019's ParserError | **NO REGRESSION** — all reconfirmed live this pass |
| 4 Tier-A editor-completion items | **NEW open items** — confirmed genuinely incomplete on the live model |

**Gate: FAIL, unchanged from fix pass 3, for the same D2/D4 reasons — neither was in this fix pass's
scope, and neither regressed.** This fix pass's own 3 issues are genuinely, independently verified
closed with no regression. Full gate rationale, updated case tally, and the HANDOFF recommendation
are in `test-report.md`'s "RETEST — fix pass 4" section.

---

## RETEST — fix pass 5 (2026-07-23, OOTB step I/O configuration — server-side round-trip)

Scope: drive the deployed workflow far enough to genuinely exercise Invoke DDX / Convert-to-PDF/A /
Send Email attachment (Objective #1), and actually attempt to COMPLETE a real Manager Approval task
in the AEM Forms Workspace UI to resolve Groundsmith's flagged flat-vs-COMBINED ambiguity and the
ROUTES JSON-per-element runtime-parsing question (Objective #2). This required going beyond every
prior retest's depth: not just starting a workflow instance (already proven since fix pass 4), but
opening and attempting to complete the resulting task through the real UI a real approver would use.

### Method

1. Submitted a fresh, fully valid Employee Training Request through the embedded page (new instance
   `employee-training-request-approval_36`, `state: RUNNING`, work item at `node2` — Manager Approval,
   confirmed via `GET /var/workflow/instances/.../employee-training-request-approval_36.json`).
2. Located the work item's canonical completion URL — the same `workitem_url` AEM itself generates for
   task-assigned emails: `/aem/dashboard/formdetails.html?item=<workitem path>` (Adobe's "Forms
   Workspace" Task Manager, the `fd/dashboard/tm` component suite).
3. Opened that URL in a real, JS-executing browser (Cypress/Electron — not a curl SSR fetch) for both
   a stale (fix-pass-4-era, instance `_30`) and the brand-new (`_36`) work item, waited 15s for the
   SPA to fully settle, then dumped the live DOM.
4. Cross-referenced every rendering gap against the AEM `error.log` for the exact request timestamps,
   to get class/line-level root causes rather than inferring from a blank page.

### Result: the task-detail page renders EMPTY for both instances — no form, no data, no route buttons

For both the stale and the brand-new work item, the only action-bar control that ever rendered was
"Delegate" — no form/data panel, no metadata panel, no Approve/Reject (or any) submit button. This is
not a timing artifact (15s wait, confirmed via a second 6s-wait pass too) and not an instance-staleness
artifact (identical empty result on a work item created fresh under this fix pass's own v1.9 model,
seconds after the AF submission it belongs to).

**Three independent, class/line-level-confirmed root causes were found in the live `error.log`,
all firing on the exact request timestamps of these page loads:**

#### D6 (NEW, Critical) — Assign Task's COMBINED attachments property is rejected by the runtime resolver it's actually read by

```
com.adobe.granite.workflow.WorkflowException: Invalid type for resolving property RELATIVE_PLOAD
	at com.adobe.fd.workflow.utils.PropertyResolver.getInputObjectInternal(PropertyResolver.java:311)
	at com.adobe.fd.workflow.utils.PropertyResolver.getColonSeparatedPropertyValue(PropertyResolver.java:103)
	at com.adobe.fd.workspace.service.impl.WorkSpacePayLoadManagerImpl.getFieldLevelAttachments(WorkSpacePayLoadManagerImpl.java:2055)
	at com.adobe.fd.workspace.service.impl.FormsSubmissionServiceImpl.getFormDataForWorkItem(FormsSubmissionServiceImpl.java:462)
	at org.apache.jsp...workitemdetails.redirector.redirector__002e__jsp._jspService(redirector__002e__jsp.java:211)
```

Thrown the very first time instance `_36`'s task-detail page is requested, and `redirector.jsp` — the
component that decides where to send the browser to render the actual form — fails outright,
propagating this exact stack. **`RELATIVE_PLOAD` is a genuinely recognized token in this class**
(confirmed by decompiling `PropertyResolver.class` from `adobe-aemfd-workflow-process-common-6.0.260.jar`
with `javap`: the constant pool contains `RELATIVE_PLOAD` alongside its own dedicated messages, e.g.
"File name is null or blank for RELATIVE_PLOAD" / "Object is not of type document for RELATIVE_PLOAD")
— so this is not a misspelled category. The same class also defines `FOLDER_PAYLOAD` and
`FILE_PAYLOAD` as separate, distinct categories. **Testable hypothesis for the next fix pass (not
applied here — out of Sentinel's lane): `RELATIVE_PLOAD` appears intended for resolving a SINGLE
Document (exactly how it's used, apparently successfully, on Invoke DDX's `inputDocs`/`outputDocs`
and Convert-to-PDF/A's `inDoc`/`pdfaDoc`), while `getFieldLevelAttachments` — invoked specifically for
`INPUT_COMBINED_FORM_ATTACHMENTS`/`OUTPUT_COMBINED_FORM_ATTACHMENTS` (a multi-file ATTACHMENTS
collection, not a single file) — may require `FOLDER_PAYLOAD` instead.** This directly and concretely
answers Groundsmith's flagged ambiguity #4: setting BOTH the flat and COMBINED forms did **not**
achieve safe hedging — the COMBINED form's presence is what breaks this runtime code path (it is not
skipped in favor of the flat form on failure; it throws).

#### D7 (NEW, Critical) — Assign Task's ROUTES JSON-per-element value fails to parse where the completion buttons are built

```
libs.fd.dashboard.tm.gui.components.workitemdetails.submitbuttons.submitbuttons$jsp
  Error in getting data com.adobe.granite.workflow.WorkflowException: JSON Exception in retrieving routes
```

Fired for **both** the stale (`_30`) and fresh (`_36`) work items on every page load, immediately
confirming this is tied to the CURRENT live `ROUTES` value (the JSON-per-element shape Fix pass 5
persisted), not to instance staleness. Net effect: the JSP responsible for rendering the actual
Approve/Reject (or any) completion buttons throws before it can render anything — **there is no
selectable route in the real UI at all.** This is the definitive, concrete answer to the second half
of the task brief's open question: ROUTES' JSON-per-element shape is proven to **persist** correctly
(Fix pass 5 / the fix-pass-5 redeploy both confirmed this via `generate.json` + live readback) but is
now proven to **fail to parse** at the one place that matters for actual task completion.

#### D8 (NEW, Critical, but NOT introduced by fix pass 5 — a pre-existing config value never previously exercised)

```
libs.fd.dashboard.tm.gui.components.workitemdetails.{metadata,formview,documentattachments}.*$jsp
  Error in getting data java.lang.IllegalArgumentException:
    No enum constant com.adobe.fd.workspace.step.service.client.type.FormType.ADAPTIVE_FORM
```

Decompiled `com.adobe.fd.workspace.step.service.client.type.FormType` (from
`forms-dashboard-core-bundle-4.0.252.jar`) with `javap`: **the enum's only valid constants are `AF`,
`PDF`, `READ_ONLY_AF`, `CCR_UI`, `IC_WEB`, `IC_PRINT` — there is no `ADAPTIVE_FORM` constant at all.**
The workflow model's `assigntask_manager`/`assigntask_finance` nodes carry `"FORM_TYPE":"ADAPTIVE_FORM"`
— a value that was set when Groundsmith rebuilt Assign Task onto `AssignFormStep` in **fix pass 4**
(confirmed present in the live model both before and after this fix pass; fix pass 5 did not touch
`FORM_TYPE`). This is a separate, independently-fatal bug from D6/D7 that ALSO prevents
metadata/form/attachment panels from rendering, on the exact same request. **The concrete, one-value
fix for the next pass: change `FORM_TYPE` from `"ADAPTIVE_FORM"` to `"AF"` on both Assign Task nodes.**

### Net conclusion — Objective #2 (Assign Task combined I/O + ROUTES at task-completion)

**Task completion could NOT be achieved this pass — not because of a missing capability in this
session's tooling, but because of three independently-confirmed, class/line-level-evidenced runtime
defects (D6, D7, D8) that together prevent the Manager Approval task from ever rendering in the
standard Forms Workspace completion UI.** Groundsmith's flagged ambiguity is answered, but not in the
hoped-for direction: the COMBINED form is not a safe hedge (D6), and the ROUTES JSON-per-element shape
does not survive to task-completion time (D7). All three are agent-fixable, config-value-level bugs
with a specific, testable next step each — unlike D2 (platform limitation) or D4 (architectural
redesign), these are NOT recommended for permanent acceptance; recommend one more Groundsmith pass
targeting these three specific values.

### Net conclusion — Objective #1 (Invoke DDX / Convert-to-PDF/A / Send Email attachment)

**Could not be functionally exercised this pass, for a concrete, evidenced reason, not a shrug.** The
workflow has never advanced past `node2` (Manager Approval) on any real submission in this delivery's
history, because task completion is blocked (see above). Invoke DDX / Convert-to-PDF/A / the approved
Send Email step's attachment therefore remain **structurally configured and byte-for-byte matching
Groundsmith's CONFIG STATUS TABLE** (re-confirmed live, see `test-report.md`'s deploy-verification
carry-forward) but **functionally unexercised** — no PDF has ever been produced by this workflow, and
none can be inspected for EmployeeTrainingForm+ApprovalInfo merge content until D6/D7/D8 are resolved
and a task can actually be completed to let the instance progress to `node6` (Assemble Manager
Confirmation) onward.

### Full regression sweep (D1, D-R1, D-R6, D3, TC-019/D4, D2, submit-gated-on-validation)

Ran the full existing Cypress suite fresh against the fix-pass-5 redeploy:

| Spec | Result |
|---|---|
| `employee-training-request-functional.cy.js` | **8/8 passing** — D-R1 lock/unlock, D-R6 lock, rules #2–#5/#7/#8, submit-gated-on-validation, D1 (valid submit → HTTP 200 + thank-you) all regression-clean |
| `employee-training-request-final-retest.cy.js` | **5/5 passing** — Employee Details heading (UI-Finding-1), D-R1, rules #2–#4, D-R6, D1 (fresh valid submit → HTTP 200 + thank-you) all regression-clean |
| `employee-training-request-submissiondate-probe.cy.js` | **1/1 passing** (informational) — captured request body: `EmployeeDetails`/`TrainingRequest`/`Justification`/`Declaration` all correctly nested (D3 regression-clean); `Declaration.SubmissionDate` still absent — **D4 unchanged, not re-chased, exactly as instructed** |

**D2 independently reconfirmed unchanged** via a fresh `GET /var/workflow/models/employee-training-request-approval.json`:
`transitions` is still strictly linear `node0→1→...→11`, **zero** `OR_SPLIT`-typed nodes, zero
occurrences of the string `OR_SPLIT` anywhere in the live model JSON.

**Nothing regressed.** D6/D7/D8 are newly *discovered* this pass (this is the first time in the
delivery's 5 fix passes that any agent actually attempted to open/complete a real task), not newly
*introduced* by anything this pass touched outside the ROUTES/combined-IO values themselves — D8
in particular predates fix pass 5 entirely (set in fix pass 4, never exercised until now).

### Summary — retest fix pass 5

| Item | Status |
|---|---|
| Invoke DDX / Convert-to-PDF/A / Send Email attachment (config) | **Structurally confirmed live, byte-for-byte matching Groundsmith's table** (carried forward from the redeploy verification) |
| Invoke DDX / Convert-to-PDF/A / Send Email attachment (functional, real PDF content) | **NOT VERIFIABLE this pass — workflow blocked at Manager Approval by D6/D7/D8, never reaches these steps** |
| Assign Task flat-vs-COMBINED ambiguity | **RESOLVED, negatively** — the COMBINED form actively breaks `WorkSpacePayLoadManagerImpl.getFieldLevelAttachments` (D6); does not achieve the hoped-for safe hedge |
| ROUTES JSON-per-element runtime parsing at task-completion | **RESOLVED, negatively** — `submitbuttons.jsp` throws a JSON exception building the completion buttons (D7); route selection is never reachable |
| D8 (FORM_TYPE=ADAPTIVE_FORM, no such enum constant) | **NEW finding, pre-existing since fix pass 4, never previously exercised** — concrete 1-value fix identified (`AF`) |
| D1, D-R1, D-R6, D3, D2, submit-gated-on-validation | **NO REGRESSION** — all reconfirmed live this pass |
| D4 (SubmissionDate) | **UNCHANGED, not re-chased per instruction** |

**Gate: FAIL.** Not for the previously-carried D2/D4 reasons alone this time — this pass surfaces 3
NEW, concretely-evidenced, agent-fixable Critical defects (D6, D7, D8) that block real task completion
and therefore block the very capability (Invoke DDX/PDF-A/SendEmail-attachment/ROUTES) this fix pass
set out to prove. Full case tally, coverage recount, and the HANDOFF recommendation are in
`test-report.md`'s "RETEST — fix pass 5" section.

---

## FINAL functional retest -- fix pass 6 (2026-07-23, real task completion + DDX/DoR verification)

Trigger: Groundsmith's fix pass 6 fixed D6 (`INPUT_COMBINED_*` colon-parsing crash), D7 (determined
NOT to be an independent defect -- a cascading D6 symptom), and D8 (`FORM_TYPE="ADAPTIVE_FORM"` ->
`AF`). This is the deepest live test this delivery has run: a real, JS-executing browser
(Cypress/Electron) driven all the way to an actual click on the real per-task-item Approve/Reject
controls in the AEM Forms Workspace Task Manager -- not just an instance-creation check.

### Method

New spec `employee-training-request-final-functional-retest.cy.js`:
1. Submit a fresh, schema-conformant Employee Training Request through the embedded page.
2. Query `/var/workflow/instances.json`, locate the newest RUNNING instance + its Manager Approval
   work item.
3. Open `/aem/dashboard/formdetails.html?item=<workitem path>` in the browser.
4. Click the real per-item Approve control (`#fd-dashboard-tm-detailsview-Approve`) -- explicitly
   NOT the Inbox list's unrelated, permanently-hidden bulk-action button of the same visible text
   (`.cq-inbox-task-approve.foundation-collection-action-hidden`). This distinction was discovered
   mid-session: an early iteration accidentally force-clicked the hidden bulk button (a no-op with no
   row selected) and produced a false "nothing happened" result that proved nothing about real task
   completion -- corrected by matching on the real per-item control's own DOM `id`.
5. Cross-reference every result against the live `error.log` and the actual deployed clientlib JS
   (`workitemdetails.js`) for class/line-level ground truth, the same evidentiary bar used for
   D6/D7/D8.

### D6/D7/D8 -- CONFIRMED FIXED, on genuinely fresh work items (not stale ones)

Every fresh work item created this pass (instances `_52` onward) shows in `error.log`: zero
`RELATIVE_PLOAD` errors, zero `No enum constant` errors, zero `JSON Exception in retrieving routes`
errors. The task-detail action bar renders real **Approve / Reject / Delegate** controls (screenshot
evidence in `cypress/results/screenshots/.../final-retest-manager-taskdetail.png`).

**Important mechanic discovered and worth recording:** a STALE work item created BEFORE fix pass 6
(instance `_13`) still shows only "Delegate"/"OK" -- zero Approve/Reject markup -- even after the
model fix was redeployed. Root cause: AEM Forms Workflow copies a step's `metaData` into the
WorkItem's OWN metadata map at the moment the instance enters that step; a work item created under
the OLD broken config permanently carries it, regardless of later fixes to the model. **Any retest
MUST use a work item created after the fix, or it silently produces a false negative** -- this
explains why an early, less careful check in this pass initially looked like D6/D7/D8 were not fixed,
before switching to a guaranteed-fresh instance.

### D9 (NEW, Critical) -- `formview.jsp`'s NPE unconditionally fails the Approve/Reject completion gate for `FORM_TYPE=AF`

Clicking the real Approve control produces: "Error -- There are validation errors. Fix the errors to
continue." -- with no specific field ever named. Root-caused to exact source, not inferred:

1. A raw HTML dump of `formdetails.html` shows the `formview.jsp` component's render boundary with
   **zero markup inside it** -- no iframe, no guideBridge bootstrap, no field HTML. The "Document"
   tab's inline Adaptive Form panel never renders, full stop. This is the direct effect of the
   NullPointerException Groundsmith flagged in fix pass 6:
   ```
   libs.fd.dashboard.tm.gui.components.workitemdetails.formview.formview$jsp
     Error in getting data java.lang.NullPointerException:
     Cannot invoke "org.apache.sling.api.resource.Resource.getParent()" because "parentResource" is null
   ```
2. The real, deployed clientlib JS that Approve/Reject actually invokes
   (`/libs/fd/dashboard/tm/gui/components/workitemdetails/clientlibs/workitemdetails.js`,
   function `showConfirmationDialog()`):
   ```js
   if (isReadOnlyForm || (guideBridge && guideBridge.isConnected() && guideBridge.validate())
       || ("CCR_UI" === formType && isCCRValid) || "IC_WEB" === formType) {
       dialog.show();
   } else {
       FormDashboard.TM.Util.showErrorMsg(Granite.I18n.get("Error"),
           Granite.I18n.get("There are validation errors. Fix the errors to continue."));
   }
   ```
   For `FORM_TYPE="AF"` (fix pass 6's deliberate choice, made to support interactive
   `ApprovalInfo`/`ManagerComments` entry): `isReadOnlyForm` is false and `formType` is neither
   `CCR_UI` nor `IC_WEB`, so the ONLY path to `dialog.show()` requires a connected, valid
   `guideBridge`. Since the inline form never renders (item 1), `guideBridge` never connects, so this
   branch is permanently false. **This is the same function for BOTH Approve and Reject** (it reads
   the clicked control's own `id` to pick the route) -- so this blocks every route identically, not
   just Approve.

**Net conclusion:** with `FORM_TYPE=AF` on this build, no route can ever be completed on any work
item -- the confirmation-dialog gate fails unconditionally, before any real field-level validation
ever runs. This is a new, distinct, Critical, code-proven, agent-fixable defect, **D9** -- separate
from D6/D7/D8 (genuinely fixed) and from D2/D4 (unchanged, pre-existing).

**Two concrete fix paths identified for the next pass (diagnosis only -- not attempted here, out of
Sentinel's lane):**
1. Root-cause and fix `formview.jsp`'s actual NPE so the inline AF initializes and `guideBridge`
   connects -- preserves the interactive-entry intent of choosing `FORM_TYPE=AF`.
2. OR revert `FORM_TYPE` to `READ_ONLY_AF` on both Assign Task nodes (the reference model's own
   value; already A/B-tested working for the D8 enum crash in fix pass 6) -- `isReadOnlyForm` becomes
   true, bypassing `guideBridge` entirely.

### Finance Manager Approval + DDX/DoR + email attachment -- still not reachable, for the D9 reason now

Because Manager Approval can never complete, no instance in this pass ever produced a Finance Manager
Approval work item, and the workflow never reached Invoke DDX / Convert-to-PDF/A / the approval Send
Email step. All three remain **structurally configured correctly** (unchanged from Groundsmith's
fix-pass-5 CONFIG STATUS TABLE, not touched this pass) but **functionally unexercised** -- no PDF has
ever been produced by this workflow in this delivery's history. This does not weaken fix pass 5's
finding; it replaces the blocking reason with a single, more precisely diagnosed one (D9, not
D6/D7/D8).

### D1/D2/D4 regression check

D1 (submit -> workflow): every fresh submission this pass (6+) returned HTTP 200 and created a
RUNNING instance at node2 -- reconfirmed not regressed. D2 (OR-split): reconfirmed unchanged via a
fresh live model fetch -- 12 nodes, zero `OR_SPLIT` occurrences, still strictly linear. D4
(SubmissionDate): reconfirmed unchanged via the existing `submissiondate-probe` spec -- still absent
from the outgoing payload at click-time and immediately after; not re-chased, per instruction.

### Summary -- FINAL functional retest, fix pass 6

D6/D7/D8: all CONFIRMED FIXED, live, on genuinely fresh work items. D9: NEW Critical defect,
code-proven, blocks Approve AND Reject unconditionally for `FORM_TYPE=AF`. Manager Approval task
completion: still blocked (by D9, not D6/D7/D8). Finance Manager Approval: not reached. Invoke
DDX/Convert-to-PDF/A/Send-Email-attachment: still not functionally verifiable -- no PDF has ever been
produced. D1/D2/D4: unchanged, reconfirmed. Full case tally and gate verdict in `test-report.md`'s
"FINAL functional retest -- fix pass 6" section.

## FINAL functional verification -- fix pass 7 (2026-07-27, Approve path, Reject path, DDX/DoR assembly)

Trigger: `FORM_TYPE` reverted to `READ_ONLY_AF` on both Assign Task nodes (bypasses D9's broken
`guideBridge` gate structurally, since `isReadOnlyForm` short-circuits `showConfirmationDialog()`'s
condition true). Comment-capture wired via `WORKITEM_COMMENT=rejectionReason` +
`IS_COMMENT_ALLOWED=true`.

**D9 CONFIRMED FIXED** -- zero validation-error dialogs, zero NPEs, real Approve/Reject markup
server-rendered with correct `isreadonlyform`/`isCommentAllowed` metadata, on genuinely fresh work
items for both an Approve-path and a separate Reject-path instance.

**A NEW, deeper Critical defect (D10) now blocks ALL real task completion**, superseding D9 as the
actual gate. Root-caused via three independently converging pieces of live evidence:

1. **Claim/delegate throws a genuine server exception.** The Task Manager SPA's automatic
   "Assign to Self" flow (shown because the workitem is group-assigned) calls
   `FormDashboard.TM.Util.delegateWorkItem()` -> `POST /bin/workflow/inbox` -> HTTP 500. The live
   `error.log` shows:
   `java.lang.IllegalStateException: This session has been closed` inside
   `WorkflowSessionImpl.delegateWorkItem` (`EventPublishUtil.publishDelegationEvent` ->
   `AuthorizableDelegator.getID` -> `SessionDelegate.checkLive` fails), called from
   `AssignFormStep.execute(AssignFormStep.java:261)`. On this error the client bounces to the Inbox
   (`onDelegateError` sets `window.location = inboxUILink`) -- exactly the observed behaviour.
2. **Direct REST completion (the same endpoint the real Confirm button invokes) also fails.**
   Per the task's explicit "Click/POST" allowance, `POST
   /libs/fd/dashboard/servlets/afsubmission.json?operation=submit` was called directly with the
   exact `FormData` contract read from the live `onSubmitButtonClick()` source
   (`workItemId`/`formPath`/`comment`/`attachments`/`formRoute`/`_charset_`), CSRF-token
   authenticated, for BOTH `formRoute=Approve` (Manager-Approval item) and `formRoute=Reject` (the
   separate Reject-path item). **Both return identically:**
   `HTTP 500 -- {"unresolvedMessage":"Invalid user - {0}","messageArgs":["admin"],"code":"AEM-FD-011-004"}`
3. **Root cause: the assignee group has no real member.** `GET
   /home/groups/a/administrators/rep:members.json` -- the group `STATIC_ASSIGNEE="administrators"`
   resolves to on both `node2` and `node4` -- shows its **only member is the system user
   `replication-receiver`**. `admin` is not a member, and two independent, standard
   Jackrabbit/Sling group-membership grants (one targeting `admin` directly, one targeting a
   disposable freshly created test user `sentinel-fp7-reviewer`) both returned `200 OK` and updated
   the group node's `jcr:lastModified`, yet **`rep:members` never changed** -- membership grants to
   this group do not persist via the standard API in this environment. This is a further anomaly on
   top of the underlying defect; both diagnostic actions were reverted/cleaned up immediately
   (disposable test user deleted, `204`) -- the environment's authorization state is unchanged from
   before this retest.

**Net conclusion: no route (Approve or Reject) can be completed on any work item assigned to the
`administrators` group, by any method (real UI click or direct REST POST), regardless of
`FORM_TYPE`.** Manager Approval and Finance Manager Approval are both structurally correct and
parity-confirmed (identical config on both nodes) but functionally unreachable for completion.

### Objective #1 (DDX/DoR assembly + email attachment) -- STILL NOT VERIFIABLE, for the D10 reason

Both fresh instances submitted this pass remain `RUNNING` at `node2` at pass end (confirmed via a
final `GET` on each instance). `Invoke DDX`/`Convert-to-PDF/A`/`Send Email` (`node6`-`node10`) were
never reached. A minimal SMTP catcher was stood up on the project's own, unmodified
`smtp.host=localhost`/`smtp.port=587` default (no OSGi config changed) specifically to capture real
email content the instant any instance reached a Send-Email step -- it captured **zero messages**
across the entire pass, independently corroborating the instance-state evidence. No PDF has ever
been produced by this workflow across all 7 fix passes in this delivery's history.

### Objective #2 (Reject path + comment-capture parity) -- mechanism CODE-CONFIRMED wired, functionally UNEXERCISABLE

Live clientlib inspection of `showConfirmationDialog()`/`onSubmitButtonClick()` confirms
`iscommentallowed` correctly injects a real `#fd-dashboard-tm-detailsview-comment` textarea into the
Confirm dialog, and its value is POSTed as `comment` alongside `formRoute` -- the mechanism fix pass
7 wired is implemented exactly as intended. But since no task can be completed at all (D10), this
cannot be proven end-to-end: `rejectionReason` cannot be confirmed to receive a real typed value, and
`Send Rejection Notification` (`node10`) was never reached to check the email body. Reported as
**UNVERIFIED**, not PASS -- correct by code inspection is not the same as functionally proven.

### Full regression sweep -- 14/14, zero regressions; D2/D4 reconfirmed unchanged, not re-chased

`employee-training-request-functional.cy.js` (8/8), `employee-training-request-final-retest.cy.js`
(5/5), `employee-training-request-submissiondate-probe.cy.js` (1/1, informational -- D4's
`Declaration.SubmissionDate` still empty at click-time and immediately after). D2 reconfirmed via a
fresh model fetch at the start of this pass: 12 nodes, zero `OR_SPLIT`, still strictly linear.

### Summary -- fix pass 7

D9: CONFIRMED FIXED. D10 (NEW, Critical): blocks Manager Approval and Finance Manager Approval task
completion on every route, every work item, every method tried -- code-proven via a live
`error.log` stack trace, live clientlib source, live REST rejection, and a live group-membership
query plus 2 failed remediation attempts. Manager Approval task completion: still blocked (by D10,
not D9). Finance Manager Approval: not reached. Invoke
DDX/Convert-to-PDF/A/Send-Email-attachment: still not functionally verifiable -- no PDF has ever been
produced across 7 fix passes. Reject-path comment-capture: code-confirmed wired, functionally
unexercisable. D1/D2/D4: unchanged, reconfirmed. Full case tally and gate verdict in
`test-report.md`'s "FINAL functional verification -- fix pass 7" section.

## DEFINITIVE final functional verification -- fix pass 8 (2026-07-27, D10: admin assignee; NEW D11/D12 discovered)

**Trigger.** `STATIC_ASSIGNEE` changed from `"administrators"` (group) to `"admin"` (direct user) on
both `assigntask_manager` and `assigntask_finance` -- Groundsmith's fix pass 8 -- to sidestep D10's
claim/delegate `IllegalStateException` entirely (a work item already assigned to a specific user needs
no claim). `PROCESS_PARTICIPANT_TYPE=static`, `FORM_TYPE=READ_ONLY_AF`, `WORKITEM_COMMENT=rejectionReason`,
`IS_COMMENT_ALLOWED=true` all carried forward unchanged from fix pass 7. Live model readback confirmed
`STATIC_ASSIGNEE="admin"` on both nodes before this pass began.

### Method -- real end-to-end completion attempts, on the DELIVERED/committed configuration, driven via Cypress/Electron against `http://localhost:4502`

1. Fresh Cypress spec `employee-training-request-fixpass8-retest.cy.js` (adapted from fix pass 7's
   spec, same conventions): submitted a fresh Approve-path instance and a separate fresh Reject-path
   instance through the embedded page, located each instance's Manager Approval work item, opened the
   real Task Manager detail view, and drove the actual Confirm dialog (Approve / Reject, with a typed
   comment) exactly as a human reviewer would.
2. Cross-referenced the live `error.log` for the exact request timestamps of both completion attempts.

### Result 1 -- D10 CONFIRMED FIXED: both routes are now clickable with zero claim/delegate step, on the delivered configuration

- `AP4-approve-clicked: PASS` -- `#fd-dashboard-tm-detailsview-Approve` was clicked directly; no
  "Assign to Self"/claim modal ever appeared (the work item is now a direct-user assignment, not
  group-pending).
- `RJ3-reject-button-present`/`RJ3-reject-clicked`/`RJ3-comment-box-present`/`RJ3-comment-typed: PASS`
  -- the Reject control, its Confirm dialog, and the `IS_COMMENT_ALLOWED=true` comment textarea all
  rendered and were driven exactly as fix pass 7's mechanism intended.
- Zero occurrences of `IllegalStateException`/`"Invalid user - admin"`/`AEM-FD-011-004` anywhere in
  `error.log` for either attempt. **D10 is retired -- the admin-user assignee fix genuinely works.**

### Result 2 -- a NEW Critical defect (D11) now blocks task completion on the delivered configuration -- both routes, code+log-proven

Both completion attempts (Approve on the Approve-path instance, Reject with a typed comment on the
Reject-path instance) returned **HTTP 500** from `POST
/libs/fd/dashboard/servlets/afsubmission.json?operation=submit`, and neither instance advanced off
`node2`. The live `error.log` shows an **identical exception at both exact request timestamps**
(11:57:11.631 for the Approve attempt on instance `_82`, 11:58:02.716 for the Reject attempt on
instance `_84`):

```
com.adobe.fd.workspace.exceptions.FormsWorkflowException: Unable to update metadata for workitem - {0}
	at com.adobe.fd.workspace.service.impl.WorkSpacePayLoadManagerImpl.updateWorkItemMetaData(WorkSpacePayLoadManagerImpl.java:1419)
	at com.adobe.fd.workspace.service.impl.FormsSubmissionServiceImpl.submitFormInternal(FormsSubmissionServiceImpl.java:261)
	...
Caused by: com.adobe.granite.workflow.WorkflowException: Invalid value : rejectionReason
	at com.adobe.fd.workflow.utils.PropertyResolver.setPropertyValueUsingColonSeparatedValue(PropertyResolver.java:160)
	at com.adobe.fd.workflow.utils.PropertyResolver.setPropertyValueUsingColonSeparatedValue(PropertyResolver.java:176)
	at com.adobe.fd.workspace.service.impl.WorkSpacePayLoadManagerImpl.saveComment(WorkSpacePayLoadManagerImpl.java:1405)
	at com.adobe.fd.workspace.service.impl.WorkSpacePayLoadManagerImpl.updateWorkItemMetaData(WorkSpacePayLoadManagerImpl.java:1416)
...
*ERROR* com.adobe.fd.workspace.servlet.FormsSubmissionServlet Unable to complete the task - {0}. Contact your administrator to resolve the issue.
```

**Root cause.** `WorkSpacePayLoadManagerImpl.saveComment()` -- invoked unconditionally during
`updateWorkItemMetaData()` whenever `WORKITEM_COMMENT` names a variable -- hands the assignee's raw
comment text to `PropertyResolver.setPropertyValueUsingColonSeparatedValue()`. This is the **same**
colon-separated-category parser class already implicated in D6 (the `RELATIVE_PLOAD:`-prefixed
attachment-path parsing that broke the combined-form properties). It expects its input in
`"CATEGORY:value"` form and throws `WorkflowException: Invalid value : <variableName>` for anything
that doesn't parse that way -- and a free-text comment (`"Approved by Sentinel fix-pass-7 retest."`,
or the typed rejection reason) is never colon-prefixed. Fix pass 7's decompiled evidence correctly
identified *which* method `saveComment` calls, but not that this method unconditionally requires
colon-separated input and throws on plain text -- so the `WORKITEM_COMMENT`/`IS_COMMENT_ALLOWED`
mechanism, as currently wired, cannot save any free-text comment into a plain String workflow variable
on this AEM Forms version. **This blocks completion of BOTH Manager Approval and Finance Manager
Approval, on both routes, on every real attempt, on the delivered/committed configuration** -- D10
being fixed only removed the claim/delegate obstacle; it exposed this deeper, independent one
underneath.

### Reversible diagnostic (NOT a delivered fix -- executed live on `/conf`, then fully reverted) to answer the delivery's central question despite D11

Per this delivery's established precedent (fix pass 7's disposable group-membership grant + SMTP
catcher, both explicitly temporary and reverted), and because D11 was blocking the one question this
entire 8-fix-pass delivery still had not answered, Sentinel ran one additional, fully reversible live
experiment: temporarily set `IS_COMMENT_ALLOWED=false` and cleared `WORKITEM_COMMENT` on both
`assigntask_manager`/`assigntask_finance` directly on the live `/conf` design node (CSRF+Referer POST,
readback-confirmed, runtime regenerated), submitted fresh instances, drove real completions, then
**reverted both properties to their exact committed values (`IS_COMMENT_ALLOWED=true`,
`WORKITEM_COMMENT=rejectionReason`) and regenerated the runtime model again** -- readback-confirmed
the live instance matches the committed source exactly, byte-for-byte, before this pass ended. No
source file was edited; this was a live-JCR-only, fully reversible probe, exactly as `AGENTS.md`'s
"never soft-pass" and this delivery's own established diagnostic pattern permit.

**With the comment mechanism disabled, both real completions succeeded cleanly:**
- Manager Approval Approve -> instance genuinely advanced from `node2` to `node4` (Finance Manager
  Approval), zero exceptions.
- Finance Manager Approval Approve -> instance genuinely advanced from `node4` to `node6` (Assemble
  Manager Confirmation / Invoke DDX), zero exceptions.
- Manager Approval Reject (separate instance) -> also advanced from `node2` to `node4` -- **direct,
  live confirmation that D2 (no real OR-split) is unchanged**: Reject routes to the identical next
  node Approve does, not to the rejection branch. Not re-chased; reported as expected, unchanged.

This conclusively isolates D11 as the sole blocker on the delivered config's completion path (nothing
else needed to change for both routes, on both Assign Task steps, to complete cleanly) -- and, as a
direct consequence, finally let this delivery's workflow reach Invoke DDX for the first time in 8 fix
passes.

### Result 3 -- THE CORE OBJECTIVE (DDX/DoR assembly): CONCLUSIVELY ANSWERED -- NO, it does not work, and here is exactly why

With the diagnostic bypass in place, the Approve-path instance reached `node6` (Assemble Manager
Confirmation / Invoke DDX) and then **failed deterministically, identically, on every one of 10
automatic retries** (Granite Workflow's job-queue retry policy), exhausting all retries
(`retryCount=1` through `retryCount=10`, `"will retry 0 more time(s)"` at the last), and the instance
remains permanently `RUNNING` at `node6` with no further progress -- confirmed live, well after the
retry window closed. **This is a NEW, Critical, code+log-proven defect -- D12:**

```
com.adobe.granite.workflow.WorkflowException: Exception while executing Invoke DDX Process step
	at com.adobe.fd.workflow.assembler.InvokeDDXProcess.internal_execute(InvokeDDXProcess.java:113)
	at com.adobe.fd.workflow.common.AEMFDWorkflowProcess$1.call(AEMFDWorkflowProcess.java:71)
	...
Caused by: com.adobe.granite.workflow.WorkflowException: Exception while executing Invoke DDX Process step
	at com.adobe.fd.workflow.common.AEMFDWorkflowProcess.execute(AEMFDWorkflowProcess.java:82)
	at com.adobe.granite.workflow.core.job.HandlerBase.executeProcess(HandlerBase.java:198)
	... 8 common frames omitted
```

Adobe's own `InvokeDDXProcess`/`AEMFDWorkflowProcess` classes do **not** chain the underlying root
exception into this message -- a logging deficiency in the platform code itself, confirmed by
inspecting every line this exception touches in `error.log` (no deeper `Caused by` exists; `stdout.log`
has no additional detail either). This is reported precisely, not guessed further, per this pass's
explicit instruction: exact class (`com.adobe.fd.workflow.assembler.InvokeDDXProcess`), exact method/
line (`internal_execute`, `InvokeDDXProcess.java:113`), exact message
(`"Exception while executing Invoke DDX Process step"`), confirmed reproducible on 10/10 retries.

**Most likely structural root cause** (reasoned from the DDX content and this step's own documented
binding, not guessed blind): the DDX operation
(`<XDP result="pdfComplete"><XDP source="EmployeeTrainingForm"/><XDP source="ApprovalInfo"/></XDP>`)
is an Assembler **XDP-merge** operation, which expects each `<XDP source="...">` input key to resolve
to a genuine XDP-format Document (an XFA form template, optionally combined with data) -- but this
step's `inputDocs` binds **both** keys to the exact same `RELATIVE_PLOAD:data.xml` (the Adaptive
Form's plain JSON/XML data payload, not an XDP template). This is precisely the approximation fix pass
4/5 already flagged in the workflow model's own inline documentation as "the closest achievable
equivalent given AEMaaCS's single-schema AF payload, not a guess... a perfect legacy-parity 2-document
split would require an earlier step that renders the two DoR sections as separate Documents first --
not built here; flagged as a pre-production follow-up, not silently dropped." **This retest is the
first time that documented approximation has actually been exercised end-to-end, and it empirically
fails** -- confirming the concern was not merely cosmetic/architectural but a genuine runtime blocker.

**Direct, conclusive answer to the delivery's central question:** No PDF has ever been produced by
this workflow, and with the current `inputDocs` binding **it cannot be**, because Invoke DDX itself
throws on every attempt. Convert-to-PDF/A (`node7`) and the approval Send-Email attachment (`node8`)
are consequently **never reached** -- not "unverifiable," but **conclusively, empirically unreachable
under the current design**, independent of D11.

### Result 4 -- Finance Manager Approval: reachable and completes cleanly once D11 is bypassed (diagnostic only)

Confirmed in the diagnostic run above: Finance Manager Approval's work item (`node4`) rendered
correctly and its real Approve completion succeeded with zero exceptions, advancing the instance to
`node6`. This closes the "is Finance Manager Approval itself broken" question independently of D11/D12
-- it is not; it works exactly like Manager Approval once the comment-save crash is out of the way.

### Result 5 -- Reject path + comment-capture parity: mechanism is the CAUSE of D11, not merely unverified

Fix pass 7 reported the comment mechanism as "code-confirmed wired, functionally unexercisable." This
pass upgrades that finding: it is not merely unexercised -- **exercising it is exactly what throws
D11's exception**, on both Approve (optional comment) and Reject (comment typed) attempts, on the
delivered configuration. The rejection-reason-in-email question therefore remains unverified for a
new, precise reason: not "the workflow never got that far" but "the very act of saving ANY comment
value, empty or not, into the `rejectionReason` variable via `WORKITEM_COMMENT` crashes completion."

### Full regression sweep -- 14/14, zero regressions; D2/D4 reconfirmed unchanged, not re-chased

`employee-training-request-functional.cy.js` (8/8), `employee-training-request-final-retest.cy.js`
(5/5), `employee-training-request-submissiondate-probe.cy.js` (1/1 -- D4's
`Declaration.SubmissionDate` still empty at click-time and immediately after, unchanged). D2
reconfirmed via the diagnostic run itself (the most direct evidence yet): both Approve and Reject on
Manager Approval route to the identical next node (`node4`), and the live runtime model still shows
zero `OR_SPLIT` occurrences.

### Summary -- DEFINITIVE fix pass 8

D10: **CONFIRMED FIXED** (admin-user assignee genuinely sidesteps claim/delegate; zero
`IllegalStateException`/"Invalid user" errors on either route). D11 (NEW, Critical): blocks task
completion on the delivered/committed configuration for both Assign Task steps, both routes --
code+log-proven (`PropertyResolver.setPropertyValueUsingColonSeparatedValue`, `"Invalid value :
rejectionReason"`). D12 (NEW, Critical): even bypassing D11 (temporary, fully-reverted diagnostic
only), Invoke DDX throws deterministically on 10/10 retries
(`InvokeDDXProcess.internal_execute`, `InvokeDDXProcess.java:113`) -- **conclusively answering this
delivery's central 8-fix-pass question: the DDX/DoR/email-attachment objective does not work, and will
not work, under the current `inputDocs` binding, independent of D11.** Manager Approval and Finance
Manager Approval task completion: both genuinely work once D11 is out of the way (diagnostic-confirmed,
NOT the delivered state). D1/D2/D4: unchanged, reconfirmed -- D2 now with the most direct evidence this
delivery has produced (both routes observed converging on the identical next node). Full case tally and
gate verdict in `test-report.md`'s "DEFINITIVE final functional verification -- fix pass 8" section.

---

## DEFINITIVE final E2E — fix pass 9 (Generate-DoR + admin assignee, real AF_PATH) — 2026-07-27

Full functional/workflow verification of the deployed form + regenerated runtime model, driven end to
end through the embedded page. Verdict: workflow crashes GONE, DoR generation STILL BROKEN.

### Approve path (instances _111, _113)
- Submit through embedded page -> HTTP 200; workflow RUNNING at node2.
- Manager Approval (STATIC_ASSIGNEE=admin, READ_ONLY_AF) completed via Approve through the real
  Workspace UI (/aem/dashboard/formdetails.html) -> advanced node2->node4. No D9 formview.jsp NPE, no
  D10 session-closed, no D11 HTTP-500 comment crash, no "validation errors" block. CONFIRMED.
- Finance Manager Approval (admin) completed via Approve -> advanced node4->node5->node6. CONFIRMED.

### node6 Generate Document of Record — FAILS
com.adobe.fd.workflow.dorGeneration.AFtoDORStep, AF_PATH = the REAL interactive form
(/content/forms/af/aem-adaptive-form-agents/employee-training-request), FORM_RESOLUTION=PATH. On both
instances node6 hard-fails after 10 retries with:
    Adaptive form at Node : .../employee-training-request/jcr:content is not valid
    java.lang.Exception: Not a valid Adaptive Form  (AFtoDORStep.java:125)
DoR PDF never written: GET <payload>/DocumentofRecord/DoR.pdf -> HTTP 404 (both instances).

The AF_PATH is correct (confirmed in the FormResolverUsingPath log line: "resolved form path is
/content/forms/af/aem-adaptive-form-agents/employee-training-request"), so the earlier stale "-dor
page" cause is genuinely fixed. The blocker now is AFtoDORStep's own form-validity gate rejecting a
valid Core Components AF. dorType none->generate was applied (source + live, readback confirmed) as the
one pre-authorized narrow fix and had ZERO effect — same failure on the post-fix run (_113). Diagnostic
evidence the form is genuinely valid: guideContainer sling:resourceType = the CC v2 formcontainer
proxy; guideContainer.model.json -> HTTP 200; the form fills+submits fine on the page. Conclusion:
deeper AEMFD Generate-DoR-step vs Core-Components incompatibility on this SDK (bundle
adobe-aemfd-workflow-process-common:6.0.260), not a config toggle. New defect D13. No thrashing.

### node7 approval email + attachment — never reached (node6 blocks). SMTP also unconfigured (TC-032).

### Reject path (instance _115)
Manager Approval -> Reject completed cleanly (HTTP-level task completion, no crash of any class).
Linear model (D2) means Reject advanced to node4 (Finance) rather than short-circuiting to the
rejection email; node9 is never reached (behind the failing node6). Rejection-reason capture is the
documented fix-pass-9 deferral (WORKITEM_COMMENT removed; rejectionReason unwired -> blank reason in
rejection.html) — expected. A comment box still rendered (IS_COMMENT_ALLOWED=true) but its text no
longer persists to a variable by design.

### Rules / submit regression (employee-training-request-functional.cy.js) — ALL GREEN
D-R1 (RT-02), D-R6 (RT-06), rules #2/#3/#4/#5 (RT-05/RT-03/RT-04/RT-07), UI-Finding-1 heading (RT-01),
and D1 + submit-gated-on-validation (RT-08: empty BLOCKED, valid submit HTTP 200 + thank-you) all pass.
D2 reconfirmed linear/open; D4 unchanged/open.

### Open integration items
- D13 (NEW, Critical): AFtoDORStep rejects the CC form -> no DoR PDF, no approval-email attachment.
- D2 (Critical, open): OR-split routing missing (linear); Reject/Approve do not branch.
- D4 (open): SubmissionDate cross-panel assignment doesn't land.
- Rejection-reason capture: documented deferral. Real manager lookup + distinct finance approver:
  accepted pre-production follow-ups.
