# Submit Action, Launcher & Programmatic Trigger — AEMaaCS

Reference: https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/forms/integrate/set-submit-action/configure-submit-action-workflow

Project tokens: `{project}` = `aem-demo-site` (the single project namespace; the folder segment under `/content/forms/af`, `/content/dam/formsanddocuments`, and `/conf/forms` is also `{project}` — there is no separate app folder). `{package}` = `com.aem.forms.agents`.

---

## 1. Submit Action — "Invoke an AEM Workflow" (PRIMARY trigger for form submissions)

### Core Components — UI steps

1. Open the form in edit mode → **Content** browser → select the **Guide Container**.
2. Open its properties (wrench) → **Submission** tab.
3. **Submit Action** → **Invoke an AEM Workflow**.
4. **Workflow Model** → select the model (`/var/workflow/models/[workflow-name]`).
5. **Store Data file using** → path (`data.xml`) or variable.
6. **Store attachments using** → path (`attachments/`) or variable.
7. **Documents of record using** → path (`DocumentofRecord/DoR.pdf`) or variable.
8. **Done**.

### guideContainer JCR properties — VERIFIED on this SDK (use the MODERN action)

Path: `/content/forms/af/{project}/[form-name]/jcr:content/guideContainer`

The current Core Components action is **`fd/dashboard/components/actions/aemworkflowsubmit`** (the
"Forms Dashboard" submit). It uses **different property names** than the older
`fd/af/components/guidesubmittype/workflow` action — getting these wrong causes a hard submit
failure (see the gotcha below). Verified working set:

```xml
<guideContainer
    ...
    actionType="fd/dashboard/components/actions/aemworkflowsubmit"
    workflowModel="/var/workflow/models/[workflow-name]"
    dataXMLType="FOLDER_PAYLOAD"
    dataXMLPath="data.xml"
    attachmentsType="FOLDER_PAYLOAD"
    attachmentsFolderPath="attachments/"
    dorType="none"
    storeAfSubmittedData="{Boolean}true"/>
```

Property reference for the modern action (dialog field → property):

| Dialog field | Property | Value |
|---|---|---|
| Workflow Model | `workflowModel` | `/var/workflow/models/[workflow-name]` |
| Data File Path | `dataXMLPath` | path **relative to payload**, e.g. `data.xml` |
| (data storage mode) | `dataXMLType` | `FOLDER_PAYLOAD` (relative to payload) or `VARIABLE` |
| (data variable) | `dataXMLVariable` | variable name — only when `dataXMLType=VARIABLE` |
| Attachment Path | `attachmentsFolderPath` | e.g. `attachments/` |
| (attachment mode) | `attachmentsType` | `FOLDER_PAYLOAD` or `VARIABLE` |
| Document of Record Path | `dorPath` | e.g. `DocumentofRecord/DoR.pdf` |
| (DoR mode) | `workflowDorType` / `dorType` | `FOLDER_PAYLOAD`, `VARIABLE`, or `none` |

> ⚠️ **The #1 submit failure — read this.** If `dataXMLPath` is empty, absolute, or a browser
> URL (e.g. someone pastes `/assets.html/content/dam/...`), submit dies server-side with:
> `FormsWorkflowException: Unable to submit the form. No output path relative to payload,
> configured to store the form data.` — **no workflow instance is created.** `dataXMLPath` is
> where the AF submit stashes the raw payload data so the workflow can read it
> (`<payload>/data.xml`); it is **NOT** where any downstream step writes its output. Set it to a
> simple payload-relative path like `data.xml`.

Rules:
- **Do not use the legacy `afDataFile` / `afAttachmentsPath` / `afDoRPath` / `workflowDataType`
  properties with the `aemworkflowsubmit` action** — that action ignores them, leaving the data
  path effectively unset and triggering the failure above. (Those belong to the older
  `fd/af/components/guidesubmittype/workflow` action.)
- **External data storage:** if the model is marked for it, use `*Type="VARIABLE"` + the
  `*Variable` properties only — no payload paths — and declare all variables in the model first.
- The reliable way to set these is the Submission-tab dropdown in the editor (it writes the
  correct property names); confirm the result matches the table above before committing. To set
  them programmatically (Sling POST), post `dataXMLPath`, `dataXMLType`, etc. directly to the
  `guideContainer`.
- Dashboard-action payloads are created under **`/var/fd/dashboard/payload/...`** (not
  `/content/forms/fp`); a custom step reads `<that payload>/data.xml`.

> The workflow starts on **author**. Assign Task steps that render the form require the form to
> be **published**.

---

## 2. OSGi Workflow Launcher (only when a JCR-event trigger is needed)

Use when the workflow should fire on a JCR node create/modify event, independent of the submit
action. For form submissions, prefer the submit action above.

Path: `ui.config/src/main/content/jcr_root/apps/{project}/osgiconfig/config.author/com.adobe.granite.workflow.core.launcher.WorkflowLauncherImpl~[launcher-name].cfg.json`

```json
{
  "enabled": true,
  "eventTypes": ["NODE_ADDED"],
  "glob": "/content/forms/fp/[form-name]/[*]/jcr:content",
  "nodeType": "nt:unstructured",
  "excludes": [],
  "conditions": "",
  "workflow": "/var/workflow/models/[workflow-name]",
  "runModes": ["author"]
}
```

`eventTypes`: `NODE_ADDED`, `NODE_MODIFIED`, `NODE_REMOVED`.
The `ui.config` filter already covers `/apps/{project}/osgiconfig`, so no new
filter entry is needed.

---

## 3. Programmatic trigger (Java/OSGi — only for bespoke logic)

Prefer the built-in submit action; write Java only when you need extra logic around the start.

```java
@Reference private WorkflowService workflowService;
@Reference private ResourceResolverFactory resolverFactory;

public void triggerWorkflow(String payloadPath) throws WorkflowException {
    Map<String, Object> authInfo = Collections.singletonMap(
        ResourceResolverFactory.SUBSERVICE, "{project}-workflow");
    // Always try-with-resources — never close() manually
    try (ResourceResolver resolver = resolverFactory.getServiceResourceResolver(authInfo)) {
        WorkflowSession wfSession = workflowService.getWorkflowSession(resolver.adaptTo(Session.class));
        WorkflowModel model = wfSession.getModel("/var/workflow/models/[workflow-name]");
        WorkflowData wfData = wfSession.newWorkflowData("JCR_PATH", payloadPath);
        wfData.getMetaDataMap().put("applicantEmail", "user@example.com");
        wfSession.startWorkflow(model, wfData);
    } catch (LoginException e) {
        throw new WorkflowException("Cannot resolve workflow service user", e);
    }
}
```

Service-user mapping
(`ui.config/.../config.author/org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended~{project}-workflow.cfg.json`):
```json
{ "user.mapping": [ "[bundle-symbolic-name]:{project}-workflow=[system-user]" ] }
```
`[bundle-symbolic-name]` must match the built core bundle. Create the system user via repoinit.

---

## 4. Day CQ Mail Service + Link Externalizer (required for any email)

Both are required for Assign Task notifications and Send Email steps.

`com.day.cq.mailer.DefaultMailService.cfg.json`:
```json
{
  "smtp.host": "[SMTP_HOST]",
  "smtp.port": 587,
  "smtp.user": "[SMTP_USER]",
  "smtp.password": "$[secret:forms.mail.smtpPassword]",
  "from.address": "noreply@[domain].com",
  "smtp.ssl": false,
  "smtp.starttls": true,
  "debug.email": false
}
```

`com.day.cq.commons.servlets.RootMappingServlet` is unrelated; the externalizer is
`com.day.cq.commons.impl.ExternalizerImpl`:
```json
{ "externalizer.domains": [
    "local http://localhost:4502",
    "author https://author-[program]-[env].adobeaemcloud.com",
    "publish https://publish-[program]-[env].adobeaemcloud.com"
] }
```

Place both in `ui.config/.../osgiconfig/config.author/`. Never hardcode the SMTP password —
use `$[secret:key]` (resolved from Cloud Manager secret env vars).

---

## 5. Payload path conventions

| Item | Recommended path (relative to payload) |
|---|---|
| Submitted data | `data.xml` (or `addresschange/data.xml` to nest) |
| Attachments | `attachments/` |
| Document of Record | `DocumentofRecord/[form-name].pdf` |

Full payload path in CRX is auto-generated under `/content/forms/fp/[form-name]/[auto-id]/`.
