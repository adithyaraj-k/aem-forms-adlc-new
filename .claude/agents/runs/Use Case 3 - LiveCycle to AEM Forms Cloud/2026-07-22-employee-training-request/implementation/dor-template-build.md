# Document-of-Record template content — employee-training-request-dor

Path: `/content/forms/af/aem-adaptive-form-agents/employee-training-request-dor`
Role: **Document-of-Record template content — NOT a user-facing interactive form.** Never embedded
on a Sites page (assembler skips it), never wired with a submit action or prefill service.

## Why an Adaptive-Form-shaped page, and how it's actually used

This project already has a working, PROVEN pattern for AEM Forms cloud "Document of Record"
generation — `com.adobe.fd.workflow.aem.process.GenerateDocumentOfRecordStep`, used by the existing
`health-insurance-underwriting` and `life-insurance-underwriting` workflow models (see
`ui.content/.../conf/global/settings/workflow/models/aem-adaptive-forms-agents/health-insurance-underwriting/.content.xml`,
node `process_dor`). That step takes a `formPath=` PROCESS_ARG pointing at a DEPLOYED Adaptive Form
page and renders IT (populated with the in-flight workflow data) to PDF.

Those two existing workflows point `formPath` at the SAME interactive form the employee fills in —
which works for them because their DoR is a straight snapshot of the whole form. **Our case is
different**: per `solution-architecture.yaml`'s `dor_template_decision`, the DoR must be a *merge* of
the request-data section (from the legacy `EmployeeTrainingRequest.xdp`) and the approval-decision
section (from `ApprovalInfo.xdp`) — WITHOUT the Attachments panel (the legacy DDX merge never
included attachments). Pointing `formPath` at the live `employee-training-request` form directly
would render Attachments too, which is out of scope for the PDF. So a SEPARATE, dedicated
`employee-training-request-dor` page was built — same schema, same theme, same 2 fragments (single
source of truth), all fields display-only, Attachments omitted, Approval Information always visible
(no show/hide — the DoR is a static point-in-time snapshot generated only after final approval, so
no workflow-state scripting is needed or ported).

## Artifacts produced (this phase)

1. **Page** — `ui.content/.../jcr_root/content/forms/af/aem-adaptive-form-agents/employee-training-request-dor/.content.xml`
   - `guideContainer` (`formcontainer`), schema-bound to the SAME `employee-training-request.schema.json`,
     themed with the SAME `aem-adaptive-forms-agents-training-request` theme (no second theme build).
   - NO `actionType`, NO `thankYou*` — never meant to be submitted by an end user; the workflow engine
     renders it server-side against already-submitted data.
   - Explicit AF Title component (same convention as the interactive form).
   - 5 panels: Employee Details (`employee-identity` fragment), Training Request Details (inline,
     duplicated field set, all `readOnly="{Boolean}true"`), Justification (inline, all readOnly),
     Declaration (`declaration-consent` fragment), Approval Information (inline, all readOnly, always
     visible). **NO Attachments panel** (intentional, matches legacy DDX scope exactly).
2. **DAM guide asset** — `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request-dor`
   (`type="guide"`, `guide="1"`) — makes this a real, deployable, renderable page, matching the
   `formPath=` mechanism the two existing DoR workflows already use.
3. **Per-form conf context** — `/conf/forms/aem-adaptive-form-agents/employee-training-request-dor`.
4. **Filter entry** — `/conf/forms/aem-adaptive-form-agents/employee-training-request-dor` added.

## Exact path for Groundsmith (flag this explicitly)

When Groundsmith builds the `employee-training-request-approval` workflow model's "Generate Document
of Record" step, wire it EXACTLY like the existing `health-insurance-underwriting`/
`life-insurance-underwriting` precedent:

```xml
<process_dor jcr:primaryType="nt:unstructured" sling:resourceType="cq/workflow/components/model/process"
    jcr:title="Generate Document of Record">
    <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.GenerateDocumentOfRecordStep"
        PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="formPath=/content/forms/af/aem-adaptive-form-agents/employee-training-request-dor,locale=en,dorOutputPath=DocumentOfRecord/EmployeeTrainingRequest-DoR.pdf"/>
</process_dor>
```

This step should run on the FINANCE-MANAGER-APPROVED path only (per `workflow_design`), immediately
before the "Send Email (Approved)" step that attaches the generated PDF.

## Reused fragments (single source of truth)

Both `employee-identity` and `declaration-consent` are embedded here IDENTICALLY to the interactive
form — editing either fragment once updates both the live form and the DoR output.

## UI-parity reference (for Sentinel, structural not pixel-perfect)

- `EmployeeTrainingRequest.xdp`'s DAM preview JPEG (request-data section parity)
- `ApprovalInfo.xdp`'s DAM preview JPEG (approval-decision section parity)
