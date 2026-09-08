# Parallel Review Workflow (AND Split) — AEMaaCS

Use when multiple reviewers must ALL approve before the workflow proceeds (Legal + Finance + HR
review simultaneously). Saves time vs. sequential when reviewers are independent.

Project tokens: `{project}` = `aem-demo-site` (the single project namespace; the folder segment under `/content/forms/af`, `/content/dam/formsanddocuments`, and `/conf/forms` is also `{project}` — there is no separate app folder).

---

> ⚠️ The `cq:WorkflowModel` XML below is an **unverified scaffold** — `PROCESS`/`PROCESS_ARGS`
> values are not officially documented. Recreate this shape in the **Workflow editor** using the
> documented UI fields ([workflow-model-spec.md](./workflow-model-spec.md)), Sync, then capture to
> `ui.content`. Use the XML to review structure, not as final output.
>
> The join-condition scripts below (ANDing/ORing three reviewers' decisions) are a legitimate use
> of hand-written `script{N}` ECMA — the editor's graphical Rule Definition builder only handles a
> single variable-equals-literal comparison, not this kind of multi-variable boolean logic. For a
> plain single-variable Approve/Reject OR-split elsewhere in the same model, prefer the Rule
> Definition builder (`expression{N}`) instead — see workflow-model-spec.md → "OR-split condition:
> use the editor's Rule Definition builder" for why (live-verified: even a corrected
> `graniteWorkflowData` script failed real Inbox routing for that simpler case on this project).

## How AND Split / AND Join behave

- **AND Split** fans out: all outgoing transitions fire at once, creating parallel branches.
- **AND Join** blocks until **all** incoming branches complete, then lets the workflow continue.
- Each branch is an independent Inbox work item — different users work them concurrently.
- Per-reviewer decision variables (`reviewer1Decision`, …) must be captured by each Assign Task
  step's Route Variable (use a distinct route variable per branch) so the post-join OR-split can
  test "did everyone approve?".

---

## Runtime Model — `/var/workflow/models/[workflow-name]/.content.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          jcr:primaryType="cq:WorkflowModel"
          jcr:title="[WORKFLOW_TITLE] — Parallel Review"
          description="Parallel multi-reviewer approval"
          version="1">
  <metaData jcr:primaryType="nt:unstructured" tags="[]">
    <variables jcr:primaryType="nt:unstructured">
      <reviewer1Decision jcr:primaryType="nt:unstructured" type="String"/>
      <reviewer2Decision jcr:primaryType="nt:unstructured" type="String"/>
      <reviewer3Decision jcr:primaryType="nt:unstructured" type="String"/>
    </variables>
  </metaData>

  <nodes jcr:primaryType="nt:unstructured">
    <node0 jcr:primaryType="cq:WorkflowNode" title="Start"     type="START"     x="20"  y="280"><metaData jcr:primaryType="nt:unstructured"/></node0>
    <node1 jcr:primaryType="cq:WorkflowNode" title="AND Split" type="AND_SPLIT" x="160" y="280"><metaData jcr:primaryType="nt:unstructured"/></node1>

    <node2 jcr:primaryType="cq:WorkflowNode" title="Legal Review" type="PROCESS" x="300" y="120">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.AssignTaskStep" PROCESS_AUTO_ADVANCE="false"
        PROCESS_ARGS="taskTitle=Legal Review,assignee=[LEGAL_GROUP],assigneeType=STATIC,notifyParticipant=true,routeVariable=reviewer1Decision,routes=Approve,Reject,formType=READ_ONLY_ADAPTIVE_FORM,formPath=/content/forms/af/{project}/[FORM_NAME]"/>
    </node2>
    <node3 jcr:primaryType="cq:WorkflowNode" title="Finance Review" type="PROCESS" x="300" y="280">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.AssignTaskStep" PROCESS_AUTO_ADVANCE="false"
        PROCESS_ARGS="taskTitle=Finance Review,assignee=[FINANCE_GROUP],assigneeType=STATIC,notifyParticipant=true,routeVariable=reviewer2Decision,routes=Approve,Reject,formType=READ_ONLY_ADAPTIVE_FORM,formPath=/content/forms/af/{project}/[FORM_NAME]"/>
    </node3>
    <node4 jcr:primaryType="cq:WorkflowNode" title="HR Review" type="PROCESS" x="300" y="440">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.AssignTaskStep" PROCESS_AUTO_ADVANCE="false"
        PROCESS_ARGS="taskTitle=HR Review,assignee=[HR_GROUP],assigneeType=STATIC,notifyParticipant=true,routeVariable=reviewer3Decision,routes=Approve,Reject,formType=READ_ONLY_ADAPTIVE_FORM,formPath=/content/forms/af/{project}/[FORM_NAME]"/>
    </node4>

    <node5 jcr:primaryType="cq:WorkflowNode" title="AND Join" type="AND_JOIN" x="460" y="280"><metaData jcr:primaryType="nt:unstructured"/></node5>
    <node6 jcr:primaryType="cq:WorkflowNode" title="Check All Approved" type="OR_SPLIT" x="600" y="280"><metaData jcr:primaryType="nt:unstructured"/></node6>

    <node7 jcr:primaryType="cq:WorkflowNode" title="Generate DoR" type="PROCESS" x="740" y="180">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.GenerateDocumentOfRecordStep" PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="formPath=/content/forms/af/{project}/[FORM_NAME],locale=en,dorOutputPath=[Payload_Directory]/DocumentofRecord/DoR.pdf"/>
    </node7>
    <node8 jcr:primaryType="cq:WorkflowNode" title="Notify Rejection" type="PROCESS" x="740" y="380">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.day.cq.workflow.process.SendEmail" PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="template=/apps/{project}/workflow/notification/email/[workflow-name]/rejection.html,sendTo=${payload.jcr:content/applicantEmail}"/>
    </node8>

    <node9 jcr:primaryType="cq:WorkflowNode" title="End" type="END" x="880" y="280"><metaData jcr:primaryType="nt:unstructured"/></node9>
  </nodes>

  <transitions jcr:primaryType="nt:unstructured">
    <transition0 jcr:primaryType="cq:WorkflowTransition" from="node0" rule="" to="node1" x="90"  y="280"><metaData jcr:primaryType="nt:unstructured"/></transition0>
    <transition1 jcr:primaryType="cq:WorkflowTransition" from="node1" rule="" to="node2" x="230" y="200"><metaData jcr:primaryType="nt:unstructured"/></transition1>
    <transition2 jcr:primaryType="cq:WorkflowTransition" from="node1" rule="" to="node3" x="230" y="280"><metaData jcr:primaryType="nt:unstructured"/></transition2>
    <transition3 jcr:primaryType="cq:WorkflowTransition" from="node1" rule="" to="node4" x="230" y="360"><metaData jcr:primaryType="nt:unstructured"/></transition3>
    <transition4 jcr:primaryType="cq:WorkflowTransition" from="node2" rule="" to="node5" x="380" y="200"><metaData jcr:primaryType="nt:unstructured"/></transition4>
    <transition5 jcr:primaryType="cq:WorkflowTransition" from="node3" rule="" to="node5" x="380" y="280"><metaData jcr:primaryType="nt:unstructured"/></transition5>
    <transition6 jcr:primaryType="cq:WorkflowTransition" from="node4" rule="" to="node5" x="380" y="360"><metaData jcr:primaryType="nt:unstructured"/></transition6>
    <transition7 jcr:primaryType="cq:WorkflowTransition" from="node5" rule="" to="node6" x="530" y="280"><metaData jcr:primaryType="nt:unstructured"/></transition7>
    <transition8 jcr:primaryType="cq:WorkflowTransition" from="node6"
      rule="function check(){var meta=graniteWorkflowData.getMetaDataMap();return meta.get('reviewer1Decision','')=='Approve' &amp;&amp; meta.get('reviewer2Decision','')=='Approve' &amp;&amp; meta.get('reviewer3Decision','')=='Approve';}"
      to="node7" x="670" y="230"><metaData jcr:primaryType="nt:unstructured"/></transition8>
    <transition9 jcr:primaryType="cq:WorkflowTransition" from="node6"
      rule="function check(){var meta=graniteWorkflowData.getMetaDataMap();return meta.get('reviewer1Decision','')=='Reject' || meta.get('reviewer2Decision','')=='Reject' || meta.get('reviewer3Decision','')=='Reject';}"
      to="node8" x="670" y="330"><metaData jcr:primaryType="nt:unstructured"/></transition9>
    <transition10 jcr:primaryType="cq:WorkflowTransition" from="node7" rule="" to="node9" x="810" y="230"><metaData jcr:primaryType="nt:unstructured"/></transition10>
    <transition11 jcr:primaryType="cq:WorkflowTransition" from="node8" rule="" to="node9" x="810" y="330"><metaData jcr:primaryType="nt:unstructured"/></transition11>
  </transitions>
</jcr:root>
```

> `&amp;&amp;` and `||` must be XML-escaped inside `rule` (`&amp;&amp;` for `&&`). For 2 branches,
> drop Branch 3's node and `reviewer3Decision`.
