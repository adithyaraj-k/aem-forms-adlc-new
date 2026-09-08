# Service Workflow (Data Integration) + Email Setup — AEMaaCS

Use when the workflow is fully automated — no human tasks, just data operations (FDM service
calls, variable extraction, DoR, confirmation email).

Project tokens: `{project}` = `aem-demo-site` (the single project namespace; the folder segment under `/content/forms/af`, `/content/dam/formsanddocuments`, and `/conf/forms` is also `{project}` — there is no separate app folder).
FDM root: `/conf/{project}/settings/cloudconfigs/fdm`.

---

> ⚠️ The `cq:WorkflowModel` XML below is an **unverified scaffold** — `PROCESS`/`PROCESS_ARGS`
> values are not officially documented. Recreate this shape in the **Workflow editor** using the
> documented step fields, Sync, then capture to `ui.content`. The email **template HTML** and the
> `${...}` template variables (further down) are reliable and ship as committed files.

## Service step configurations

### Invoke Form Data Model Service Step
```xml
<node1 jcr:primaryType="cq:WorkflowNode" title="Write to Data Source" type="PROCESS" x="160" y="280">
  <metaData jcr:primaryType="nt:unstructured"
    PROCESS="com.adobe.fd.workflow.aem.process.InvokeFormDataModelServiceStep" PROCESS_AUTO_ADVANCE="true"
    PROCESS_ARGS="formDataModelPath=/conf/{project}/settings/cloudconfigs/fdm/[fdm-name],serviceName=[service-operation],inputType=Variable,inputValue=submittedData,outputVariable=caseId"/>
</node1>
```

| Arg | Description |
|---|---|
| `formDataModelPath` | Full CRX path to the FDM config |
| `serviceName` | FDM service operation name (as defined in the FDM editor) |
| `inputType` | `Literal`, `Variable`, or `JsonPath` |
| `inputValue` | Value / variable name / JSON path |
| `outputVariable` | Workflow variable that stores the response |

### Set Variable from payload
```xml
<node jcr:primaryType="cq:WorkflowNode" title="Extract Form Data" type="PROCESS" x="300" y="280">
  <metaData jcr:primaryType="nt:unstructured"
    PROCESS="com.adobe.granite.workflow.core.process.SetVariableProcess" PROCESS_AUTO_ADVANCE="true"
    PROCESS_ARGS="variableName=applicantEmail, variableType=String, variableValue=${payload.jcr:content/data/applicantEmail}; variableName=submittedData, variableType=JSON, variableValue=${payload.jcr:content/data}"/>
</node>
```

---

## Reading the submitted payload in a custom step — `data.xml` is often JSON, not XML

A custom Java step that reads `<payload>/data.xml` **must not assume the content is XML.** For a
**JSON-schema** Adaptive Form (`schemaType="jsonschema"`) submitted via the modern Forms Dashboard
action, the file named `data.xml` actually contains **JSON** (flat, keyed by field name):

```json
{ "firstName": "Jane", "lastName": "Doe", "age": 12, "grade": "7", "shirtSize": "M" }
```

Feeding that to a DOM/XML parser throws `org.xml.sax.SAXParseException: Content is not allowed in
prolog.` and the step fails (no output written). **Sniff the first non-whitespace character** and
branch:

```java
String text = stripBom(new String(rawBytes, StandardCharsets.UTF_8)).trim();
if (text.isEmpty()) { /* nothing submitted */ }
else if (text.charAt(0) == '{' || text.charAt(0) == '[') {
    // JSON-schema form (or Dashboard submit) — already JSON, use as-is
} else {
    // XSD/basic form — parse as XML (XXE-hardened) and convert to JSON
}
```

Also strip a leading UTF-8 BOM (another "Content not allowed in prolog" cause). Remember the
Dashboard payload lives under `/var/fd/dashboard/payload/...`, and the data path is whatever
`dataXMLPath` was set to on the form (e.g. `data.xml`) — see submit-action-and-launcher.md.

## Custom Java process step pattern (e.g. save data to DAM) — Cloud-safe

When no built-in step does what you need (e.g. "save submission as a JSON asset"), write a
`com.adobe.granite.workflow.exec.WorkflowProcess` and select it by FQCN in the model's Process step
(`metaData/PROCESS={fqcn}`, `PROCESS_AUTO_ADVANCE=true`). Key rules learned the hard way:

- **Never write to the local filesystem** (`new File("C:\\...")`) — that only works on the local SDK
  JVM and cannot deploy to Cloud. Write through the repository (`AssetManager.createAsset(path,
  inputStream, "application/json", true)`).
- **Don't assume the workflow user can write `/content/dam`.** Use a dedicated **service user** +
  `ResourceResolverFactory.getServiceResourceResolver(SUBSERVICE)` in try-with-resources, with a
  repoinit `create path` for the target folder and `allow jcr:read,rep:write` ACL. Read the payload
  with the workflow session (it always sees its own payload); write the asset with the service user.
- **Throw `WorkflowException` on failure** so the instance shows the error instead of silently
  advancing — then check `crx-quickstart/logs/error.log` for the real root cause.

Reference implementations in this repo (all Cloud-safe — write to a DAM folder via `AssetManager`
+ a dedicated service user, sniff `data.xml` as JSON-or-XML, throw `WorkflowException` on failure):
- `core/.../forms/workflow/SaveFormDataAsJsonProcess.java` — JSON asset per submission.
- `core/.../forms/workflow/SaveFormDataAsPdfProcess.java` — PDF per submission (dependency-free
  PDF writer); pairs with the committed `/conf` model `save-form-data-pdf-workflow`.
- `core/.../forms/workflow/SaveFormDataAsExcelProcess.java` — appends one row per submission to a
  single `.xlsx` DAM asset (JDK-only OOXML, read-modify-write); pairs with the committed `/conf`
  model `save-form-data-excel-workflow`. Note: read-modify-write on one shared asset — fine for
  typical volume, but very high concurrent submit rates could interleave writes.

Each pairs with a `ServiceUserMapperImpl.amended~...` + `RepositoryInitializer~...` cfg.json in
`ui.config` (the repoinit `create path` + `allow jcr:read,rep:write` ACL on the DAM folder).

⚠️ **Do not reuse `core/.../forms/workflow/ExcelExportProcess.java`** for a DAM/asset-folder
requirement — it writes the workbook to a **local filesystem path** and, per its own Javadoc,
**cannot deploy to AEM as a Cloud Service**. Use `SaveFormDataAsExcelProcess` instead.

---

## Minimal service-only model — `/var/workflow/models/[workflow-name]/.content.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          jcr:primaryType="cq:WorkflowModel" jcr:title="[WORKFLOW_TITLE] — Data Integration"
          description="Automated data processing — no human steps" version="1">
  <metaData jcr:primaryType="nt:unstructured" tags="[]">
    <variables jcr:primaryType="nt:unstructured">
      <submittedData jcr:primaryType="nt:unstructured" type="JSON"/>
      <caseId        jcr:primaryType="nt:unstructured" type="String" description="Downstream response id"/>
    </variables>
  </metaData>
  <nodes jcr:primaryType="nt:unstructured">
    <node0 jcr:primaryType="cq:WorkflowNode" title="Start" type="START" x="20" y="280"><metaData jcr:primaryType="nt:unstructured"/></node0>
    <node1 jcr:primaryType="cq:WorkflowNode" title="Set Variables" type="PROCESS" x="160" y="280">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.granite.workflow.core.process.SetVariableProcess" PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="variableName=submittedData, variableType=JSON, variableValue=${payload.jcr:content/data}"/>
    </node1>
    <node2 jcr:primaryType="cq:WorkflowNode" title="Invoke FDM Service" type="PROCESS" x="300" y="280">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.InvokeFormDataModelServiceStep" PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="formDataModelPath=/conf/{project}/settings/cloudconfigs/fdm/[fdm-name],serviceName=[service-name],inputType=Variable,inputValue=submittedData,outputVariable=caseId"/>
    </node2>
    <node3 jcr:primaryType="cq:WorkflowNode" title="Generate DoR" type="PROCESS" x="440" y="280">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.GenerateDocumentOfRecordStep" PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="formPath=/content/forms/af/{project}/[FORM_NAME],locale=en"/>
    </node3>
    <node4 jcr:primaryType="cq:WorkflowNode" title="Send Confirmation Email" type="PROCESS" x="580" y="280">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.day.cq.workflow.process.SendEmail" PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="template=/apps/{project}/workflow/notification/email/[workflow-name]/confirmation.html,sendTo=${payload.jcr:content/applicantEmail}"/>
    </node4>
    <node5 jcr:primaryType="cq:WorkflowNode" title="End" type="END" x="720" y="280"><metaData jcr:primaryType="nt:unstructured"/></node5>
  </nodes>
  <transitions jcr:primaryType="nt:unstructured">
    <transition0 jcr:primaryType="cq:WorkflowTransition" from="node0" rule="" to="node1" x="90"  y="280"><metaData jcr:primaryType="nt:unstructured"/></transition0>
    <transition1 jcr:primaryType="cq:WorkflowTransition" from="node1" rule="" to="node2" x="230" y="280"><metaData jcr:primaryType="nt:unstructured"/></transition1>
    <transition2 jcr:primaryType="cq:WorkflowTransition" from="node2" rule="" to="node3" x="370" y="280"><metaData jcr:primaryType="nt:unstructured"/></transition2>
    <transition3 jcr:primaryType="cq:WorkflowTransition" from="node3" rule="" to="node4" x="510" y="280"><metaData jcr:primaryType="nt:unstructured"/></transition3>
    <transition4 jcr:primaryType="cq:WorkflowTransition" from="node4" rule="" to="node5" x="650" y="280"><metaData jcr:primaryType="nt:unstructured"/></transition4>
  </transitions>
</jcr:root>
```

---

## Email template (store under /apps on AEMaaCS, not /etc)

Save at:
`ui.apps/src/main/content/jcr_root/apps/{project}/workflow/notification/email/[workflow-name]/approval.html`
and reference that absolute path from the Send Email step's `template` arg.

```html
<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"/><title>Request Approved</title></head>
<body style="font-family:Arial,sans-serif;color:#333;max-width:600px;margin:0 auto;padding:20px">
  <h2 style="color:#2c7c3f">Your request has been approved ✓</h2>
  <p>Dear <strong>${payload.jcr:content/data/applicantName}</strong>,</p>
  <p>Your <strong>[FORM_TITLE]</strong> submitted on <strong>${workflowData.metaDataMap.startTime}</strong> has been approved.</p>
  <table style="border-collapse:collapse;width:100%;margin:20px 0">
    <tr style="background:#f5f5f5"><td style="padding:8px 12px;border:1px solid #ddd;font-weight:bold">Reference ID</td><td style="padding:8px 12px;border:1px solid #ddd">${workitem.id}</td></tr>
  </table>
  <p style="color:#666;font-size:13px">Automated notification — please do not reply.</p>
</body>
</html>
```

Save sibling `rejection.html` (heading "rejected", color `#c0392b`) and `confirmation.html`
(heading "submitted", pending status).

**Template variables:** `${workitem.id}`, `${workitem.metaData.title}`,
`${payload.jcr:content/data/[fieldName]}`, `${workflowData.metaDataMap.startTime}`,
`${workflowData.metaDataMap.userId}`, `${workflowData.metaDataMap.[customVar]}`.

> Mail delivery requires **Day CQ Mail Service** + **Day CQ Link Externalizer** — see
> submit-action-and-launcher.md for the cfg.json snippets.
