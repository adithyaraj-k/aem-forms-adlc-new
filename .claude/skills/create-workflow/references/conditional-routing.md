# Conditional Routing Workflow (OR Split) — AEMaaCS

Use when the path depends on form data (e.g. amount > 10000 → Director, else Manager).

Project tokens: `{project}` = `aem-demo-site` (the single project namespace; the folder segment under `/content/forms/af`, `/content/dam/formsanddocuments`, and `/conf/forms` is also `{project}` — there is no separate app folder).

---

> ⚠️ The `cq:WorkflowModel` XML below is an **unverified scaffold** — `PROCESS`/`PROCESS_ARGS`
> values are not officially documented. Recreate this shape in the **Workflow editor** using the
> documented UI fields ([workflow-model-spec.md](./workflow-model-spec.md)), Sync, then capture to
> `ui.content`. The routing **rule functions** (`meta.get('actionTaken', String)`, amount checks)
> are the reliable part — copy those verbatim.

## Common routing rule patterns

```javascript
// Route by amount (set into a variable by a Set Variable step first)
function check() {
  var meta = workItem.getWorkflowData().getMetaDataMap();
  return parseInt(meta.get('requestAmount', '0'), 10) > 10000;
}
```
```javascript
// Route by department from form data
function check() {
  var meta = workItem.getWorkflowData().getMetaDataMap();
  var dept = meta.get('department', '');
  return dept === 'Engineering' || dept === 'Product';
}
```
```javascript
// Route by group membership of the submitter
function check() {
  var wfData = workItem.getWorkflowData();
  var submittedBy = wfData.getMetaDataMap().get('startedBy', String);
  var um = workflowSession.getSession().getUserManager();
  var group = um.getAuthorizable('senior-employees');
  return group && group.isMember(um.getAuthorizable(submittedBy));
}
```

---

## Conditional Model Template — `/var/workflow/models/[workflow-name]/.content.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          jcr:primaryType="cq:WorkflowModel"
          jcr:title="[WORKFLOW_TITLE] — Conditional"
          description="Route based on form data conditions" version="1">
  <metaData jcr:primaryType="nt:unstructured" tags="[]">
    <variables jcr:primaryType="nt:unstructured">
      <requestAmount jcr:primaryType="nt:unstructured" type="String" description="Routing value"/>
      <department    jcr:primaryType="nt:unstructured" type="String"/>
      <actionTaken   jcr:primaryType="nt:unstructured" type="String"/>
    </variables>
  </metaData>
  <nodes jcr:primaryType="nt:unstructured">
    <node0 jcr:primaryType="cq:WorkflowNode" title="Start" type="START" x="20" y="280"><metaData jcr:primaryType="nt:unstructured"/></node0>

    <!-- Extract form data into variables -->
    <node1 jcr:primaryType="cq:WorkflowNode" title="Set Variables from Form" type="PROCESS" x="160" y="280">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.granite.workflow.core.process.SetVariableProcess" PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="variableName=requestAmount, variableType=String, variableValue=${payload.jcr:content/data/requestAmount}; variableName=department, variableType=String, variableValue=${payload.jcr:content/data/department}"/>
    </node1>

    <node2 jcr:primaryType="cq:WorkflowNode" title="Amount Check" type="OR_SPLIT" x="300" y="280"><metaData jcr:primaryType="nt:unstructured"/></node2>

    <node3 jcr:primaryType="cq:WorkflowNode" title="Director Approval" type="PROCESS" x="440" y="160">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.AssignTaskStep" PROCESS_AUTO_ADVANCE="false"
        PROCESS_ARGS="taskTitle=High-Value Request Review,assignee=[DIRECTOR_GROUP],assigneeType=STATIC,notifyParticipant=true,routeVariable=actionTaken,routes=Approve,Reject,formType=ADAPTIVE_FORM,formPath=/content/forms/af/{project}/[FORM_NAME]"/>
    </node3>
    <node4 jcr:primaryType="cq:WorkflowNode" title="Manager Approval" type="PROCESS" x="440" y="400">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.AssignTaskStep" PROCESS_AUTO_ADVANCE="false"
        PROCESS_ARGS="taskTitle=Standard Request Review,assignee=[MANAGER_GROUP],assigneeType=STATIC,notifyParticipant=true,routeVariable=actionTaken,routes=Approve,Reject,formType=ADAPTIVE_FORM,formPath=/content/forms/af/{project}/[FORM_NAME]"/>
    </node4>

    <node5 jcr:primaryType="cq:WorkflowNode" title="Post-Approval Split" type="OR_SPLIT" x="580" y="280"><metaData jcr:primaryType="nt:unstructured"/></node5>

    <node6 jcr:primaryType="cq:WorkflowNode" title="Generate DoR" type="PROCESS" x="720" y="180">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.GenerateDocumentOfRecordStep" PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="formPath=/content/forms/af/{project}/[FORM_NAME],locale=en"/>
    </node6>
    <node7 jcr:primaryType="cq:WorkflowNode" title="Notify Rejection" type="PROCESS" x="720" y="380">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.day.cq.workflow.process.SendEmail" PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="template=/apps/{project}/workflow/notification/email/[workflow-name]/rejection.html,sendTo=${payload.jcr:content/applicantEmail}"/>
    </node7>
    <node8 jcr:primaryType="cq:WorkflowNode" title="End" type="END" x="860" y="280"><metaData jcr:primaryType="nt:unstructured"/></node8>
  </nodes>

  <transitions jcr:primaryType="nt:unstructured">
    <transition0 jcr:primaryType="cq:WorkflowTransition" from="node0" rule="" to="node1" x="90"  y="280"><metaData jcr:primaryType="nt:unstructured"/></transition0>
    <transition1 jcr:primaryType="cq:WorkflowTransition" from="node1" rule="" to="node2" x="230" y="280"><metaData jcr:primaryType="nt:unstructured"/></transition1>
    <transition2 jcr:primaryType="cq:WorkflowTransition" from="node2"
      rule="function check(){var meta=workItem.getWorkflowData().getMetaDataMap();return parseInt(meta.get('requestAmount','0'),10)>10000;}"
      to="node3" x="370" y="220"><metaData jcr:primaryType="nt:unstructured"/></transition2>
    <transition3 jcr:primaryType="cq:WorkflowTransition" from="node2"
      rule="function check(){var meta=workItem.getWorkflowData().getMetaDataMap();return parseInt(meta.get('requestAmount','0'),10)&lt;=10000;}"
      to="node4" x="370" y="340"><metaData jcr:primaryType="nt:unstructured"/></transition3>
    <transition4 jcr:primaryType="cq:WorkflowTransition" from="node3" rule="" to="node5" x="510" y="220"><metaData jcr:primaryType="nt:unstructured"/></transition4>
    <transition5 jcr:primaryType="cq:WorkflowTransition" from="node4" rule="" to="node5" x="510" y="340"><metaData jcr:primaryType="nt:unstructured"/></transition5>
    <transition6 jcr:primaryType="cq:WorkflowTransition" from="node5"
      rule="function check(){var meta=workItem.getWorkflowData().getMetaDataMap();return meta.get('actionTaken',String)=='Approve';}"
      to="node6" x="650" y="230"><metaData jcr:primaryType="nt:unstructured"/></transition6>
    <transition7 jcr:primaryType="cq:WorkflowTransition" from="node5"
      rule="function check(){var meta=workItem.getWorkflowData().getMetaDataMap();return meta.get('actionTaken',String)=='Reject';}"
      to="node7" x="650" y="330"><metaData jcr:primaryType="nt:unstructured"/></transition7>
    <transition8 jcr:primaryType="cq:WorkflowTransition" from="node6" rule="" to="node8" x="790" y="230"><metaData jcr:primaryType="nt:unstructured"/></transition8>
    <transition9 jcr:primaryType="cq:WorkflowTransition" from="node7" rule="" to="node8" x="790" y="330"><metaData jcr:primaryType="nt:unstructured"/></transition9>
  </transitions>
</jcr:root>
```

> `<` and `<=` must be XML-escaped inside `rule` (`&lt;`, `&lt;=`).

---

## Dynamic Participant ECMA Script (AEMaaCS path)

Save at: `ui.apps/src/main/content/jcr_root/apps/{project}/workflow/scripts/[workflow-name]/dynamic-participant.ecma`
(**never** `/etc/workflow/scripts` on AEMaaCS). Reference this absolute path in the Assign Task
step's Participant Chooser, with `assigneeType=DYNAMIC`.

```javascript
// Resolves assignee dynamically from form data
var meta = workItem.getWorkflowData().getMetaDataMap();
var department = meta.get("department", String);
var assignee;
switch (department) {
  case "Engineering": assignee = "engineering-managers"; break;
  case "Finance":     assignee = "finance-approvers";    break;
  case "HR":          assignee = "hr-directors";         break;
  default:            assignee = "default-approvers";
}
assignee; // last expression is the resolved group/user id
```
