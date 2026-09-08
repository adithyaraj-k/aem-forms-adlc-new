# Sequential Approval Workflow Template (AEMaaCS)

Use for: 1–3 level human approval chains (e.g. Manager → HR, or a single approver).

Reference: https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/create-form-centric-workflows/aem-forms-workflow-step-reference

Project tokens: `{project}` = `aem-demo-site` (the single project namespace; the folder segment under `/content/forms/af`, `/content/dam/formsanddocuments`, and `/conf/forms` is also `{project}` — there is no separate app folder).

---

> ⚠️ The `cq:WorkflowModel` XML below is an **unverified scaffold** — `PROCESS`/`PROCESS_ARGS`
> values are not officially documented. Recreate this shape in the **Workflow editor** using the
> documented UI fields ([workflow-model-spec.md](./workflow-model-spec.md)), Sync, then capture to
> `ui.content`. Use the XML to review structure, not as final output.

## Key official patterns

- The Assign Task step's **Route Variable** is `actionTaken` (String, declared in the model).
- Routes (buttons) are added on the step's **Actions** tab: `Approve`, `Reject`.
- The OR-split transition checks `meta.get('actionTaken', String) == 'Approve'`.
- Each step references a **Workflow Stage** defined in `metaData/stages`.
- Review-only steps use `formType=READ_ONLY_ADAPTIVE_FORM` + `formReadOnly=true`.

---

## Template: 2-Level Sequential Approval (Manager → HR)

Runtime model — `ui.content/src/main/content/jcr_root/var/workflow/models/[workflow-name]/.content.xml`.
Replace every `[PLACEHOLDER]`. (Also deploy the editable `/conf/global/settings/workflow/models/[workflow-name]`
design copy — see workflow-model-spec.md.)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          jcr:primaryType="cq:WorkflowModel"
          jcr:title="[WORKFLOW_TITLE]"
          description="[DESCRIPTION]"
          version="1">
  <metaData jcr:primaryType="nt:unstructured" tags="[]">
    <stages jcr:primaryType="nt:unstructured">
      <stage0 jcr:primaryType="nt:unstructured" title="Application Submitted"/>
      <stage1 jcr:primaryType="nt:unstructured" title="Manager Review"/>
      <stage2 jcr:primaryType="nt:unstructured" title="HR Review"/>
      <stage3 jcr:primaryType="nt:unstructured" title="Completed"/>
    </stages>
    <variables jcr:primaryType="nt:unstructured">
      <actionTaken      jcr:primaryType="nt:unstructured" type="String" description="Route variable — Approve or Reject"/>
      <approverComments jcr:primaryType="nt:unstructured" type="String"/>
      <applicantEmail   jcr:primaryType="nt:unstructured" type="String"/>
    </variables>
  </metaData>

  <nodes jcr:primaryType="nt:unstructured">

    <node0 jcr:primaryType="cq:WorkflowNode" title="Start" type="START" x="20" y="280"><metaData jcr:primaryType="nt:unstructured"/></node0>

    <!-- LEVEL 1: Manager review (editable form) -->
    <node1 jcr:primaryType="cq:WorkflowNode" title="Manager Review" type="PROCESS" x="160" y="280">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.AssignTaskStep"
        PROCESS_AUTO_ADVANCE="false"
        PROCESS_ARGS="taskTitle=Review Request — [FORM_TITLE]
          taskDescription=Please review the submitted request and take action.
          taskPriority=MEDIUM
          taskDueDateInterval=2
          taskDueDateIntervalUnit=DAYS
          notifyParticipant=true
          assignee=[MANAGER_USER_OR_GROUP]
          assigneeType=STATIC
          workflowStage=Manager Review
          formType=ADAPTIVE_FORM
          formPath=/content/forms/af/{project}/[FORM_NAME]
          formReadOnly=false
          routeVariable=actionTaken
          routes=Approve,Reject
          inputDataFile=[Payload_Directory]/workflow/data.xml
          inputAttachmentsPath=[Payload_Directory]/attachments/
          outputDataFile=[Payload_Directory]/workflow/data.xml
          outputAttachmentsPath=[Payload_Directory]/attachments/"/>
    </node1>

    <node2 jcr:primaryType="cq:WorkflowNode" title="Manager Decision" type="OR_SPLIT" x="300" y="280"><metaData jcr:primaryType="nt:unstructured"/></node2>

    <!-- APPROVED → Level 2: HR review (read-only) -->
    <node3 jcr:primaryType="cq:WorkflowNode" title="HR Review" type="PROCESS" x="440" y="160">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.AssignTaskStep"
        PROCESS_AUTO_ADVANCE="false"
        PROCESS_ARGS="taskTitle=Final HR Review — [FORM_TITLE]
          taskDescription=Manager approved. Please complete final HR review.
          taskPriority=MEDIUM
          notifyParticipant=true
          assignee=[HR_USER_OR_GROUP]
          assigneeType=STATIC
          workflowStage=HR Review
          formType=READ_ONLY_ADAPTIVE_FORM
          formPath=/content/forms/af/{project}/[FORM_NAME]
          formReadOnly=true
          routeVariable=actionTaken
          routes=Approve,Reject
          inputDataFile=[Payload_Directory]/workflow/data.xml
          inputAttachmentsPath=[Payload_Directory]/attachments/
          outputDoRPath=[Payload_Directory]/DocumentofRecord/DoR.pdf"/>
    </node3>

    <!-- REJECTED → notify applicant -->
    <node4 jcr:primaryType="cq:WorkflowNode" title="Send Rejection Notification" type="PROCESS" x="440" y="400">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.day.cq.workflow.process.SendEmail"
        PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="template=/apps/{project}/workflow/notification/email/[workflow-name]/rejection.html,sendTo=${workflowData.metaDataMap.applicantEmail}"/>
    </node4>

    <node5 jcr:primaryType="cq:WorkflowNode" title="HR Decision" type="OR_SPLIT" x="580" y="160"><metaData jcr:primaryType="nt:unstructured"/></node5>

    <!-- HR Approved → Generate DoR -->
    <node6 jcr:primaryType="cq:WorkflowNode" title="Generate Document of Record" type="PROCESS" x="720" y="80">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.adobe.fd.workflow.aem.process.GenerateDocumentOfRecordStep"
        PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="formPath=/content/forms/af/{project}/[FORM_NAME],locale=en,dorOutputPath=[Payload_Directory]/DocumentofRecord/DoR.pdf"/>
    </node6>

    <node7 jcr:primaryType="cq:WorkflowNode" title="Send Approval Notification" type="PROCESS" x="860" y="80">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.day.cq.workflow.process.SendEmail"
        PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="template=/apps/{project}/workflow/notification/email/[workflow-name]/approval.html,sendTo=${workflowData.metaDataMap.applicantEmail}"/>
    </node7>

    <node8 jcr:primaryType="cq:WorkflowNode" title="Send HR Rejection Notification" type="PROCESS" x="720" y="240">
      <metaData jcr:primaryType="nt:unstructured"
        PROCESS="com.day.cq.workflow.process.SendEmail"
        PROCESS_AUTO_ADVANCE="true"
        PROCESS_ARGS="template=/apps/{project}/workflow/notification/email/[workflow-name]/rejection.html,sendTo=${workflowData.metaDataMap.applicantEmail}"/>
    </node8>

    <node9 jcr:primaryType="cq:WorkflowNode" title="End" type="END" x="1000" y="280"><metaData jcr:primaryType="nt:unstructured"/></node9>

  </nodes>

  <transitions jcr:primaryType="nt:unstructured">
    <t0 jcr:primaryType="cq:WorkflowTransition" from="node0" rule="" to="node1" x="90"  y="280"><metaData jcr:primaryType="nt:unstructured"/></t0>
    <t1 jcr:primaryType="cq:WorkflowTransition" from="node1" rule="" to="node2" x="230" y="280"><metaData jcr:primaryType="nt:unstructured"/></t1>
    <t2 jcr:primaryType="cq:WorkflowTransition" from="node2"
      rule="function check(){var meta=workItem.getWorkflowData().getMetaDataMap();return meta.get('actionTaken',String)=='Approve';}"
      to="node3" x="370" y="220"><metaData jcr:primaryType="nt:unstructured"/></t2>
    <t3 jcr:primaryType="cq:WorkflowTransition" from="node2"
      rule="function check(){var meta=workItem.getWorkflowData().getMetaDataMap();return meta.get('actionTaken',String)=='Reject';}"
      to="node4" x="370" y="340"><metaData jcr:primaryType="nt:unstructured"/></t3>
    <t4 jcr:primaryType="cq:WorkflowTransition" from="node3" rule="" to="node5" x="510" y="160"><metaData jcr:primaryType="nt:unstructured"/></t4>
    <t5 jcr:primaryType="cq:WorkflowTransition" from="node5"
      rule="function check(){var meta=workItem.getWorkflowData().getMetaDataMap();return meta.get('actionTaken',String)=='Approve';}"
      to="node6" x="650" y="120"><metaData jcr:primaryType="nt:unstructured"/></t5>
    <t6 jcr:primaryType="cq:WorkflowTransition" from="node5"
      rule="function check(){var meta=workItem.getWorkflowData().getMetaDataMap();return meta.get('actionTaken',String)=='Reject';}"
      to="node8" x="650" y="200"><metaData jcr:primaryType="nt:unstructured"/></t6>
    <t7  jcr:primaryType="cq:WorkflowTransition" from="node6" rule="" to="node7" x="790" y="80"><metaData jcr:primaryType="nt:unstructured"/></t7>
    <t8  jcr:primaryType="cq:WorkflowTransition" from="node7" rule="" to="node9" x="930" y="180"><metaData jcr:primaryType="nt:unstructured"/></t8>
    <t9  jcr:primaryType="cq:WorkflowTransition" from="node4" rule="" to="node9" x="720" y="340"><metaData jcr:primaryType="nt:unstructured"/></t9>
    <t10 jcr:primaryType="cq:WorkflowTransition" from="node8" rule="" to="node9" x="860" y="260"><metaData jcr:primaryType="nt:unstructured"/></t10>
  </transitions>
</jcr:root>
```

---

## UI steps (if authoring in the Workflow editor instead)

1. Create model → title `[WORKFLOW_TITLE]` (stored under `/conf/global/settings/workflow/models`).
2. Model properties → **Stages** → add "Application Submitted", "Manager Review", "HR Review", "Completed".
3. Delete the default Step 1.
4. Drag **Assign Task** → Form tab: select the Adaptive Form; Assignee tab: `[MANAGER_GROUP]`, enable email; Actions tab: routes `Approve`,`Reject`, Route Variable `actionTaken`; Stage: "Manager Review".
5. Drag **OR Split** after Manager Review.
6. Second **Assign Task** on the Approve branch → HR, `READ_ONLY_ADAPTIVE_FORM`.
7. **Send Email** on the Reject branch.
8. Second **OR Split** after HR → **Generate Document of Record** on Approve, **Send Email** on Reject.
9. Connect all branches to **End** → **Sync**.

---

## Variations

- **Single-level:** remove node3/node5/node8 and HR transitions; wire node2 Approve → node6, Reject → node4.
- **3-level:** insert a third Assign Task + OR Split after node5's Approve branch, before DoR.
- `backRouteable=true` on a step lets an assignee return the task to the previous step.
