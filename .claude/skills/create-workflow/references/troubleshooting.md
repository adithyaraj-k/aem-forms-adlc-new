# AEM Workflow Troubleshooting (AEMaaCS, Adaptive Forms)

Project tokens: `{project}` = `aem-demo-site` (the single project namespace; the folder segment under `/content/forms/af`, `/content/dam/formsanddocuments`, and `/conf/forms` is also `{project}` — there is no separate app folder).

---

## 1. Workflow not triggering on form submit

| Cause | Fix |
|---|---|
| Submit action not "Invoke an AEM Workflow" | Guide Container → Submission tab → select it |
| Wrong model path | Verify `/var/workflow/models/[name]` exists in CRX/DE |
| Model not synced | Open the model in the editor → **Sync** |
| `afDataFile` not set | Set the data file path (e.g. `data.xml`) on the submit action |
| Launcher disabled / glob mismatch | `enabled:true`; glob must match the real payload path |
| External-storage model missing variables | Declare all variables; use variable (not path) options |

---

## 2. Model runs but is "not editable" in the editor (AEMaaCS-specific)

- **Cause:** only the `/var/workflow/models` runtime model was deployed; the editable design
  copy at `/conf/global/settings/workflow/models/[name]` is missing.
- **Fix:** deploy the `/conf/global/settings/workflow/models/[name]` design copy, or open the
  runtime model once in the editor and **Sync** to regenerate it. Add both filter roots.

---

## 3. Assign Task not appearing in AEM Inbox

| Cause | Fix |
|---|---|
| Assignee user/group doesn't exist | Verify in `/home/users` or `/home/groups` |
| Assignee not in `workflow-users` | Add the user/group to `workflow-users` |
| `notifyParticipant` not set | Set `notifyParticipant=true` |
| Mail not configured | Configure Day CQ Mail Service + Link Externalizer |
| Form not published | Publish the form referenced by `formPath` |
| Interactive Communications | Users also need `cm-agent-users` |

---

## 4. OR Split not routing (actionTaken not set)

- **Cause:** Route Variable not set on the Assign Task step, or rule uses lowercase `string`.
- **Fix:** Actions tab → Route Variable `actionTaken` (declared String in the model); add routes
  whose labels exactly match the rule (`Approve`, `Reject`); use
  `meta.get('actionTaken', String) == 'Approve'` — capital-S `String`.

---

## 5. Email notifications not sending

- [ ] Day CQ Mail Service configured with valid SMTP host/port/credentials
- [ ] Day CQ Link Externalizer configured with the real author hostname (not `localhost`)
- [ ] `notifyParticipant=true` on the Assign Task step
- [ ] Email template path exists (absolute `/apps/{project}/workflow/notification/email/...`)
- [ ] Check `error.log` for `com.day.cq.mailer` entries

---

## 6. Generate Document of Record fails

- [ ] "Generate Document of Record" enabled on the form (Form Model tab), or XDP/PDF model used
- [ ] `formPath` matches the adaptive form JCR path exactly
- [ ] Form is published
- [ ] DoR output path: `[Payload_Directory]/DocumentofRecord/DoR.pdf`
- [ ] Workflow service user has `jcr:read` on the form path
- [ ] `com.adobe.fd.docmanager`-related bundles Active

---

## 7. Variables not carrying values between steps

| Cause | Fix |
|---|---|
| Variable not declared | Add under `metaData/variables` with the correct type |
| Type mismatch | `variableType` must match (`String` not `string`) |
| Invalid JSON/XML value | Validate before storing — invalid values fail silently |
| Wrong expression | `${payload.jcr:content/data/fieldName}` — field name must match exactly |
| External storage + path option | Must use the variable option, not a payload path |

---

## 8. Workflow instance stuck (RUNNING forever)

Causes: Assign Task waiting on a deleted user/group; AND Join waiting on a branch that errored;
uncaught exception in a process step.

Fix: open the workflow console (`/libs/cq/workflow/content/console`), find the instance →
**Terminate** or **Resume**; for AND Join, verify all parallel branches completed.

---

## 9. AEMaaCS deployment gotchas

| Issue | Fix |
|---|---|
| Model not found after deploy | Models are mutable content — must be in `ui.content` (`/var/workflow/models` + `/conf/global/settings/workflow/models`) with filter roots |
| Script path `/etc/workflow/scripts` not working | Use `/apps/{project}/workflow/scripts/` (immutable apps) |
| Launcher cfg not picked up | Place at `ui.config/.../osgiconfig/config.author/com.adobe.granite.workflow.core.launcher.WorkflowLauncherImpl~[name].cfg.json` |

---

## Useful debug paths

| What | Path |
|---|---|
| Running instances | `/var/workflow/instances` |
| Workflow payload data | `[payload-path]/jcr:content/data.xml` |
| Instance metadata & variables | `[instance-path]/data/metaData` |
| Runtime models | `/var/workflow/models` |
| Editable models | `/conf/global/settings/workflow/models` |
| AEM Inbox | `/aem/inbox` |

## Debug logging

In OSGi → Sling Log Support, add these loggers at DEBUG:
```
com.adobe.fd.workflow
com.adobe.granite.workflow
com.day.cq.workflow
com.day.cq.mailer
```
