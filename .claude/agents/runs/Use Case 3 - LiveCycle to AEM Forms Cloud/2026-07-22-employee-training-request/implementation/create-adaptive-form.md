# create-adaptive-form — employee-training-request

Path: `/content/forms/af/aem-adaptive-form-agents/employee-training-request`
Role: PRIMARY interactive Adaptive Form (migrated from `DocRoute.xdp`, per Planwright's
evidence-based role decision).

## 5 artifacts

1. **Form page** — 6 panels, 30 fields (8 in `employee-identity` fragment + 10 in Training Request
   Details + 3 in Justification + 3 in Attachments + 2 in `declaration-consent` fragment + 4 in
   Approval Information), template REUSED (`blank-af-v2`, no new template created), schema-bound
   (`employee-training-request.schema.json`), themed (`aem-adaptive-forms-agents-training-request`,
   built in the theme phase).
2. **DAM guide asset** — `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request`
   (`type="guide"`, `guide="1"`).
3. **Per-form conf context** — `/conf/forms/aem-adaptive-form-agents/employee-training-request`.
4. **Filter entries** — added for the conf context (form/DAM roots already broadly covered).
5. **Validation clientlib** — see create-form-rules.md (`employee-training-request-clientlib`, built
   in the rules phase per this project's "clientlib last" ordering rule).

## Panel structure (in order)

1. **Employee Details** — embeds the `employee-identity` fragment BY REFERENCE
   (`fragmentPath="/content/forms/af/aem-adaptive-form-agents/employee-identity"`), with
   `dataRef="$.EmployeeDetails"` on the Fragment reference node remapping the fragment's canonical
   `$.employee.*` shape to this form's schema paths.
2. **Training Request Details** (inline, `source: form`) — RequestId, RequestType (dropdown,
   Certification/Training/Conference), ProviderName, CourseName, StartDate/EndDate (datepickers,
   matching `displayFormat`/`editFormat` = `date|DD/MM/YYYY`), TrainingMode (dropdown), Location
   (`aria-label="Training Location"` to disambiguate from the fragment's own "Location"), Amount
   (numberinput, `fracDigits="2"`), Currency (dropdown).
3. **Justification** (inline) — BusinessNeed, BenefitsToProject (both multiLine), TrainingCategory
   (dropdown).
4. **Attachments** (inline, single-column layout per DESI) — VendorQuotation, CourseBrochure,
   ManagerRecommendation — all optional `fileinput` (`accept="[application/pdf,image/jpeg,image/png]"`,
   `maxFileSize="10"`).
5. **Declaration** — embeds the `declaration-consent` fragment BY REFERENCE, `dataRef="$.Declaration"`
   remap.
6. **Approval Information** (inline) — CurrentStatus (readOnly dropdown, 6-value enum), ManagerComments,
   DepartmentHeadComments, FinanceComments (readOnly by default, unlocked by rule #6). Panel default
   `visible="{Boolean}false"` (rule #5 toggles it — see create-form-rules.md).

## Title

Explicit AF Title (v2) component `formTitle` (`fd:htmlelementType="h1"`, `css="etr-form-title"`) as
the first child of `guideContainer`; `showTitle="{Boolean}false"` on the container itself (project
critical rule 3c).

## Submit / workflow wiring (rule 0 — default, NOT overridden here)

`actionType="fd/dashboard/components/actions/aemworkflowsubmit"`,
`workflowModel="/var/workflow/models/assign-task-to-admin"` — the project's DEFAULT shared workflow
wiring, left in place per rule 0. **Groundsmith must repoint `workflowModel` to the real
`employee-training-request-approval` model it builds in its `create-workflow` phase** — this is
NOT a per-form workflow scaffold, just the standard placeholder every new form ships with until
Groundsmith wires the real one. Also set: `dataXMLType="FOLDER_PAYLOAD"`, `dataXMLPath="data.xml"`,
`storeAfSubmittedData="{Boolean}true"`, `attachmentsFolderPath="attachments/"`,
`attachmentsType="FOLDER_PAYLOAD"` (required because the form has file-upload fields).

## Footer buttons

`submitButton` uses `.../actions/submit` (NOT the generic `button`); `resetButton` uses
`.../actions/reset`, empty `<fd:events/>`, no invented `reset()` call. Submit button's `fd:click`
was EXTENDED in the rules phase to also set `SubmissionDate` (rule #8) — see create-form-rules.md.

## Accessibility

Every field carries `aria-label`; the two same-visible-label "Location" fields are disambiguated
("Employee Location" in the fragment vs "Training Location" here).

## No CAPTCHA

Internal authenticated form (employee/manager/finance-manager) — no bot-protection field added, per
NFR.

## Deviation flags

- `clientLibRef` is a **SINGLE category** (`aem-adaptive-forms-agents.forms.employee-training-request`)
  — NOT the comma-separated `base,form-specific,generate-pdf` list the high-level project convention
  describes elsewhere, because the more detailed and VERIFIED `create-form-clientlib`/
  `create-adaptive-form` skill guidance is explicit that a multi-value `clientLibRef` breaks the
  Rule-Editor customfunctions endpoint (returns `{"customFunction":[]}`, marking every custom-function
  rule "Broken"). The form-specific clientlib is SELF-CONTAINED (copies the base's standard styling +
  the one date-comparison function it needs) instead. This reconciles the two levels of guidance in
  favor of the more specific, defect-verified one.
- Business rules (show/hide, enable/disable, validate, set-value) are authored in the SEPARATE
  create-form-rules phase, not inline here, per the ADLC's explicit phase split (3 vs 4).
