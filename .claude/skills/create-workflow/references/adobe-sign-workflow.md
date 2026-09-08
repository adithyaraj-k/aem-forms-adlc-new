# Adobe Sign E-Signature Workflow Integration — AEMaaCS

Reference: https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/create-form-centric-workflows/aem-forms-workflow-step-reference

Project tokens: `{project}` = `aem-demo-site` (the single project namespace; the folder segment under `/content/forms/af`, `/content/dam/formsanddocuments`, and `/conf/forms` is also `{project}` — there is no separate app folder).

---

> ⚠️ The Sign-step `cq:WorkflowModel` XML below is an **unverified scaffold** — `PROCESS`/`PROCESS_ARGS`
> values are not officially documented. Configure the **Sign Document** step in the **Workflow
> editor** (agreement name, recipients, sequential/parallel, deadline, reminders, signed-doc
> storage), Sync, then capture to `ui.content`. The **prerequisites** below are the reliable part.

## Prerequisites

1. Configure the Adobe Sign cloud service: Tools → Cloud Services → Adobe Sign, under
   `/conf/{project}/settings/cloudconfigs/adobe-sign` (or `/conf/global/...`).
2. Enable Adobe Sign on the Adaptive Form (form properties → select the cloud config).
3. Add a signature field to the form (Adobe Sign Signature Block component).

---

## Adobe Sign step (Send for Signature)

Place after the human-approval step, before End. **`PROCESS_AUTO_ADVANCE="false"`** — the step
is asynchronous and the workflow pauses until all signers complete or the deadline passes.

```xml
<node_sign jcr:primaryType="cq:WorkflowNode" title="Send for Electronic Signature" type="PROCESS" x="600" y="160">
  <metaData jcr:primaryType="nt:unstructured"
    PROCESS="com.adobe.fd.workflow.aem.process.AdobeSignStep"
    PROCESS_AUTO_ADVANCE="false"
    PROCESS_ARGS="adobeSignConfiguration=/conf/{project}/settings/cloudconfigs/adobe-sign,
      signers=${workflowData.metaDataMap.applicantEmail},
      agreementTitle=[FORM_TITLE] — Signature Required,
      agreementMessage=Please review and sign the attached document.,
      reminderFrequency=WEEKLY,
      completionDeadline=7,
      sendToAll=true"/>
</node_sign>
```

| Arg | Description |
|---|---|
| `adobeSignConfiguration` | Path to the Adobe Sign cloud config |
| `signers` | Comma-separated emails OR a String workflow variable |
| `agreementTitle` / `agreementMessage` | Agreement name / signer message |
| `reminderFrequency` | `DAILY`, `WEEKLY`, or none |
| `completionDeadline` | Days before the agreement expires |
| `sendToAll` | `true` = all sign at once; `false` = sequential |
| `signerVariable` | String variable holding the signer email (instead of a literal) |

---

## Typical flow

```
Start → Assign Task (Manager approval) → OR Split (actionTaken)
   ├─ Approve → Adobe Sign (send to applicant) → Generate Signed DoR → Send Email → End
   └─ Reject  → Send Email (rejection) → End
```

```xml
<node_dor jcr:primaryType="cq:WorkflowNode" title="Generate Signed DoR" type="PROCESS" x="740" y="160">
  <metaData jcr:primaryType="nt:unstructured"
    PROCESS="com.adobe.fd.workflow.aem.process.GenerateDocumentOfRecordStep" PROCESS_AUTO_ADVANCE="true"
    PROCESS_ARGS="formPath=/content/forms/af/{project}/[FORM_NAME],locale=en,dorOutputPath=[Payload_Directory]/DocumentofRecord/signed-DoR.pdf"/>
</node_dor>
```

---

## Notes

- The signed PDF is stored back in the workflow payload automatically after signing completes.
- Use `${workflowData.metaDataMap.applicantEmail}` to pull the signer email from a variable.
- On AEMaaCS, keep the Adobe Sign cloud config under `/conf/global/` or the project conf path.
