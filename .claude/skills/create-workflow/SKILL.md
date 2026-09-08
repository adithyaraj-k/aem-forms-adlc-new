---
name: create-workflow
description: >
  Creates a complete, working AEM Forms-centric workflow for an Adaptive Form on
  AEM as a Cloud Service (Core Components). From a requirement it either REUSES an
  existing workflow model or generates a new one, then produces every artifact that
  makes the workflow run: the runtime workflow model (cq:WorkflowModel) plus its
  editable /conf design copy, typed workflow variables, Assign Task / OR-split /
  AND-split / Generate Document of Record / Invoke FDM / Send Email / Adobe Sign
  steps, dynamic-participant ECMA scripts, the OSGi workflow launcher, the Day CQ
  Mail Service config, AND the "Invoke an AEM Workflow" submit-action wiring on the
  form's guideContainer. Use whenever a user asks to create, configure, reuse, or
  troubleshoot an AEM workflow for an adaptive form — approval flows, document
  review, leave/onboarding/mortgage/invoice processes, or any form-triggered process
  automation. Triggers on: workflow model, assign task, workflow launcher, process
  step, route variable, actionTaken, workflow stages, workflow inbox, document of
  record on approval, parallel review, conditional routing, or Adobe Sign signing.
version: 1.0.0
ide:
  cursor: .cursor/skills/create-workflow/
  github-copilot: .github/skills/create-workflow/
  claude-code: .claude/skills/create-workflow/
---

# Skill: create-workflow

## Role

You are an expert AEM Forms backend/workflow developer for **AEM as a Cloud Service**.
You build **forms-centric workflows** for **Adaptive Forms Core Components** and wire them
to a form so that submitting the form starts the workflow, human reviewers act on tasks in
the AEM Inbox, and the process advances (route → DoR → email → end) exactly as the
requirement describes.

You always do two things first, in order:
1. **Decide reuse vs. create** — never scaffold a new workflow model when an existing one
   already satisfies the requirement (see Step 2). This is a standing project rule.
2. **Use the AEMaaCS storage model** — workflow models on Cloud Service live under
   `/conf/global/settings/workflow/models` (design-time, editable) **and**
   `/var/workflow/models` (runtime, executable). The legacy 6.5-only path
   `/etc/workflow/models` is **forbidden** here, as is writing anything under `/libs`.

You never invent a PROCESS class, a property name, or a path. The step classes, the
`actionTaken` route variable, the OR-split rule syntax, and the storage locations are all
load-bearing and verified against the Adobe docs linked at the bottom.

---

## Project facts (this repository)

These come from `AGENTS.md`. Use them as the concrete values — they are **not** generic
placeholders to ask the user for.

| Token | Value | Used in |
|---|---|---|
| `{project}` | `aem-demo-site` | `/apps/{project}/…`, `/conf/{project}/…`, `/content/forms/af/{project}/…`, `/conf/forms/{project}/…`, Java package root |
| `{package}` | `com.aem.forms.agents` | Java for any programmatic trigger / process step |
| AEM target | **AEMaaCS** (cloud) | storage model, cfg.json, no `/etc` |
| Forms type | **Core Components** | submit-action wiring is on `guideContainer` |
| FDM root | `/conf/{project}/settings/cloudconfigs/fdm` | Invoke FDM Service step |

> ⚠️ `{project}` (`aem-demo-site`) is the single project namespace — use it in every path root:
> `/apps/{project}/…`, `/conf/{project}/…`, AND the folder segment under `/content/forms/af/…` and
> `/conf/forms/…`. There is no separate app folder; a form's DAM guide-asset path must match its
> `/content/forms/af/{project}/{formName}` path.

---

## Trigger

Activate when the developer asks to:
- Create or configure an AEM workflow for an adaptive form (approval, review, onboarding,
  leave, mortgage, invoice, KYC, etc.)
- Add reviewers / approval levels / sign-off to a form submission
- Route a submission conditionally (by amount, department, role) or in parallel (AND-split)
- Generate a Document of Record after approval, or send a workflow notification email
- Add an Adobe Sign e-signature step to a form process
- Wire a form's Submit to **Invoke an AEM Workflow**
- Reuse an existing workflow model for a new form
- Troubleshoot a workflow that does not trigger, stalls, or does not route

---

## AEMaaCS storage model (read first — corrected for Cloud Service)

A workflow model on Cloud Service has **two** representations, and they are **NOT** deployed the
same way. This is the most important correction vs. AEM 6.5 guides — get it wrong and the model
either won't deploy or won't be editable:

| Representation | Path | Format | Deploy? |
|---|---|---|---|
| **Design model (source of truth)** | `/conf/global/settings/workflow/models/{workflowName}` | an **editor page**: `cq:Page` + `cq:PageContent` containing a `flow` parsys of step components (`nt:unstructured` + `sling:resourceType`). **Not** a `cq:WorkflowModel`. | **YES — this is what you commit and package** |
| **Runtime model (executable)** | `/var/workflow/models/{workflowName}` | `cq:WorkflowModel` / `cq:WorkflowNode` / `cq:WorkflowTransition` | **Generated from the design model by Sync** — do **not** hand-author it. Package it only if you want the runtime present without a manual Sync. |

Rules:
- **The `/conf` design model is the deployable source of truth.** It is the only artifact you
  *must* package. **Never write a `cq:WorkflowModel` node into `/conf`**, and **never** treat a
  hand-authored `/var` model as the source.
- **The runtime is generated by Sync.** After the `/conf` design model is deployed, open it once
  in the Workflow editor and **Sync** (or it regenerates) so `/var/workflow/models/{name}` exists
  before the submit action resolves it. Deploying **only** `/var` makes the model show as
  **"not editable"**; deploying **only** `/conf` without a Sync means the runtime may not exist
  yet. Optionally also package the `/var` tree so the runtime ships immediately on every
  environment — but treat it as generated output, not the source.
- **Authoring is editor-based.** The `/conf` page format (parsys + step component resource types +
  the serialized model) is managed by the Workflow editor; build/refine the model there, then
  **capture the `/conf` `cq:Page` tree** into `ui.content`. The `cq:WorkflowModel` XML in the
  reference files is a **logical/flow reference** for planning the model and configuring its
  steps — it is **not** the deploy artifact and must not be written to `/conf`.
- **Scripts** (dynamic participant, timeout handlers) go under
  `/apps/{project}/workflow/scripts/` (immutable apps) — **never** `/etc/workflow/scripts`.
- **Launcher** configs are OSGi `.cfg.json` in `ui.config` — **never** `/etc/workflow/launcher`.
- **Never** write to `/libs`, `/etc/workflow/models`, or a `cq:WorkflowModel` under `/conf`.

---

## Authoring model — read this before generating any model XML (accuracy note)

Adobe documents AEM Forms workflow steps as things you configure in the **Workflow Model
editor UI** with named fields — **not** as raw `cq:WorkflowModel` XML. The official step
reference gives **no Java `PROCESS` class name and no `PROCESS_ARGS` string** for the
human-centric steps (Assign Task, Sign Document, Generate Document of Record). Their node
structure is complex, version-specific, and the editor writes load-bearing properties that are
impractical and error-prone to hand-author.

Treat the work in two tiers:

**Tier A — build/refine in the Workflow editor (source of truth).** The workflow **model**
itself, and especially **Assign Task / Sign / Generate DoR / routing** steps, should be created
in the editor at `/conf/global/settings/workflow/models/{workflowName}`, then **Sync**ed (which
generates the `/var/workflow/models/{workflowName}` runtime copy). After it works, capture the
**`/conf` design tree** (and optionally the generated `/var` runtime) into `ui.content` for
source control. The runtime `cq:WorkflowModel` XML in the
reference files is a **scaffold / visual reference** to recreate the same shape in the editor and
to review diffs — **not** guaranteed-deployable output. Any `PROCESS=`/`PROCESS_ARGS=` value in
those references is **unverified** and must be confirmed in the editor; do not present it to the
user as a definitive, copy-paste-and-run artifact. The accurate, documented configuration is the
**UI field list** in [workflow-model-spec.md](./references/workflow-model-spec.md).

**Tier B — reliably code-generate (deterministic, documented).** These you can author directly
into the repo with confidence: the **"Invoke an AEM Workflow" submit-action wiring** (set via the
editor's Submission tab; XML for review), the **OSGi workflow launcher** `.cfg.json`, the **Day CQ
Mail Service + Link Externalizer** `.cfg.json`, the **service-user mapping + repoinit**, **ECMA
participant/timeout scripts** under `/apps/{project}/workflow/scripts/`, **email templates** under
`/apps/{project}/workflow/notification/email/`, and the **filter.xml** entries.

So: gather requirements, decide reuse-vs-create, give the user the **precise editor UI steps** to
build/verify the model (Tier A), and generate the Tier-B artifacts as committed files. Offer the
scaffold XML as a starting point only, clearly labelled as needing editor verification.

---

## Step 1 — Gather requirements

Use `AskUserQuestion` for the choices below when the request is ambiguous; extract answers
from a use-case description when the user already gave one ("2-level leave approval, HR then
manager, email on reject"), and **confirm your reading before generating**.

1. **Workflow name & title** — kebab-case node name (e.g. `leave-approval`) + display title.
2. **Which form** — the Adaptive Form path under `/content/forms/af/{project}/{formName}`.
3. **Shape** — sequential approval (1–3 levels), parallel review (AND-split), conditional
   routing (OR-split by data), or service-only (no human task). → picks the reference below.
4. **Assignees** — specific users/groups, or dynamic (script-based by form data)?
5. **Routes** — the action buttons reviewers see (e.g. `Approve`, `Reject`, `Request More Info`).
6. **Due dates** — should tasks expire? (interval + `DAYS`/`HOURS`).
7. **Notifications** — email on submit / approve / reject? To whom (form field or fixed)? Every
   recipient — on an Assign Task's "Send Notification Email" AND on every dedicated Send Email
   step — MUST resolve via a declared workflow **variable** (`RECIPIENT_EMAIL_RESOLUTION="VARIABLE"`
   + `EMAIL_VARIABLE=...`, or `toAddressType="Variable"` + `toAddressValue=...`); never author a
   literal recipient address. If no real source for that address exists yet (no form field, no
   payload path, no directory/lookup service), use `AskUserQuestion` to ask the user for a default
   recipient email address before finalizing the step — do not leave the variable blank or invent
   a plausible-looking address. Set that default in **both** places (they are independent — see
   workflow-model-spec.md → "Workflow Variables" for the full explanation): the variable's own
   `defaultValue` property on its `jcr:content/variables/<name>` declaration (editor-UI-facing;
   **not** `value` — that guess was tried and confirmed wrong), AND the model's initial "Capture
   Submission Variables" Set Variable step (`variableName=<var>, variableType=String,
   variableValue=<default-address>`, runtime-facing) — the same way
   `employee-training-request-approval`'s `managerEmail`/`financeEmail`/`applicantEmail` variables
   are seeded. The Set Variable step is a plain literal assignment, not a conditional fallback
   (`SetVariableProcess` has no "only if blank" logic).
8. **Document of Record** — generate a PDF after approval? (requires DoR enabled on the form).
9. **Data storage** — payload path (`data.xml`) or workflow **variables** (variables are
   mandatory if the model is marked for external data storage).
10. **Adobe Sign** — any e-signature step? (requires the Adobe Sign cloud config).
11. **Stages** — the progress labels shown in the AEM Inbox bar.

---

## Step 2 — Decide: REUSE an existing model, or CREATE a new one

> 🔁 **CANONICAL SHARED MODEL — `assign-task-to-admin` (create once, reuse for EVERY form).**
> This project has ONE standing submission workflow: on submit, **assign a Medium-priority task to
> the `admin` user**. It is the DEFAULT submit action for every new Adaptive Form (see
> `create-adaptive-form` → "STANDING PROJECT RULE"). Details:
> - **Model name:** `assign-task-to-admin`
> - **Design model (committed source of truth):**
>   `ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/assign-task-to-admin/.content.xml`
>   — a `cq:Page` whose `flow` parsys holds ONE Assign Task step
>   (`sling:resourceType=fd/workflow/components/dashboard/afParticipantStep`) with metaData:
>   `PROCESS=com.adobe.fd.workspace.step.service.AssignFormStep`, `PROCESS_AUTO_ADVANCE=false`,
>   `PROCESS_PARTICIPANT_TYPE=static`, `STATIC_ASSIGNEE=admin`, **`TASK_PRIORITY=MEDIUM`**,
>   `ROUTES=Complete`, `ROUTE_PROPERTYNAME=actionTaken`, `DUEDATE_UNIT=OFF`,
>   `FORM_RESOLUTION=AF_PATH`, `FORM_TYPE=READ_ONLY_AF`, `AF_PATH=<a form in this project>`,
>   `workflowStage=Task Assigned`. (This exact shape is verified working on the SDK — a submitted
>   instance produces a work item with `workitem_priority=MEDIUM`, `workitem_assignee=admin`,
>   `WORK_ITEM_TYPE=AF_ASSIGN_STEP`.)
> - **Runtime:** `/var/workflow/models/assign-task-to-admin` — **generated by Sync/`generate.json`**
>   from the design model; never hand-authored, never committed.
> - **Rule:** if this model is **absent** (no `/conf` design tree AND no `/var` runtime), create it
>   ONCE with the shape above, then generate the runtime. If it is **present** (which it is on this
>   project), **REUSE it** — do only the submit-action wiring on the form. **NEVER create a per-form
>   or duplicate workflow** for the "assign a task on submit" requirement.

**Always check for a reusable model before generating one.** This is an explicit project
requirement — do not create a near-duplicate workflow.

1. **Scan the repo** for existing models **and reusable building blocks**:
   - `ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/*` and
     `ui.content/src/main/content/jcr_root/var/workflow/models/*` — committed model trees.
   - ⚠️ **Two kinds of model live here — check both.** (a) **Editor-authored** models leave no
     committed tree; their only on-disk trace is **commented `<filter root=".../models/{name}"/>`
     entries** in `ui.content/.../META-INF/vault/filter.xml` (e.g. `save-form-data-json`,
     `form-submission-notification`). (b) **Service-only** models are **hand-authored as a `/conf`
     design `cq:Page` and committed** (with an *active*, uncommented filter root) — e.g.
     `ui.content/.../jcr_root/conf/global/settings/workflow/models/save-form-data-pdf-workflow`
     and `.../save-form-data-excel-workflow`. **These committed `/conf` trees are the best
     copy-paste template for a new service-only workflow** — copy one, swap the `PROCESS` FQCN and
     titles, add the filter root, then `generate.json` (see workflow-model-spec.md → "Deploy a
     model WITHOUT the editor"). Do NOT route a no-human-task workflow through the editor.
   - **Existing custom step implementations:** `core/src/main/java/{package}/forms/workflow/*` and
     `.../forms/submit/*`. For a service/data-transform task these are the thing to **model your new
     step on or reuse** — match their service-user, payload-reading, and DAM-write conventions:
     - `SaveFormDataAsPdfProcess` / `SaveFormDataAsJsonProcess` / `SaveFormDataAsExcelProcess` —
       **Cloud-safe**: write to a DAM folder via `AssetManager` + a dedicated service user, sniff
       `data.xml` as JSON-or-XML. These are the right model for any "save submission to
       `/content/dam/...`" requirement.
     - ⚠️ `ExcelExportProcess` writes the `.xlsx` to a **local filesystem path** (`C:\Users\...`) —
       its own Javadoc says it **cannot deploy to Cloud**. Do **not** reuse it for a DAM/asset-folder
       requirement; use (or model on) `SaveFormDataAsExcelProcess` instead.
   - **Existing wiring:** grep committed form `.content.xml` under
     `ui.content/.../content/forms/af/**` for `workflowModel=` / `actionType=…aemworkflowsubmit`
     to see which models are already in use and how the submit action is configured.
2. **Compare to the requirement.** A model is reusable when its shape (levels, routes,
   assignees-by-group, DoR/email steps) matches what the user asked for. The form it runs on
   does **not** matter — a generic "2-level approval" model is reused across many forms by
   pointing each form's submit action at it and (where the step needs it) parameterising the
   form path.
3. **Then either:**
   - **REUSE** → generate **no** model. Do only the **submit-action wiring** (Step 4.3) to
     point the form's `guideContainer` at the existing `/var/workflow/models/{name}`, plus any
     missing launcher/mail config. Tell the user which existing model you reused and why.
   - **EXTEND** → the existing model is close but missing a step (e.g. add an Adobe Sign step).
     Edit that model in place; do not fork a copy.
   - **CREATE** → no suitable model exists. Generate the full set of artifacts in Step 4.

> If the user explicitly names a model to reuse, skip the scan and wire to it. If a scan match
> is only partial, ask (`AskUserQuestion`) whether to reuse-and-extend or create new — don't
> silently fork.

---

## Step 3 — Select the variant and read its reference

| Requirement shape | Reference to read before generating |
|---|---|
| Sequential human approval (1–3 levels) | [sequential-approval.md](./references/sequential-approval.md) |
| Parallel review — all must approve (AND-split) | [parallel-review.md](./references/parallel-review.md) |
| Conditional routing by form data (OR-split) | [conditional-routing.md](./references/conditional-routing.md) |
| Service-only — FDM / data integration, no human task | [service-workflow.md](./references/service-workflow.md) |
| Adobe Sign e-signature | [adobe-sign-workflow.md](./references/adobe-sign-workflow.md) |
| Any model — node types, all step classes, Assign Task property reference | [workflow-model-spec.md](./references/workflow-model-spec.md) |
| Submit-action wiring, launcher, programmatic trigger, mail/externalizer config | [submit-action-and-launcher.md](./references/submit-action-and-launcher.md) |
| Workflow does not trigger / stalls / won't route / not editable | [troubleshooting.md](./references/troubleshooting.md) |

---

## Step 4 — Generate artifacts

Produce only what the requirement needs. For a **new** model, that is typically 4.1–4.3 plus
4.4/4.5 when relevant. For a **reuse**, it is only 4.3 (+ any missing 4.6/4.7).

### 4.1 Create the workflow model — `/conf` design model is the deliverable (Tier A)

The model **is** the workflow; this step "creates the workflow model." On AEMaaCS the deployable
artifact is the **`/conf` design model** (a `cq:Page` editor structure), and the `/var` runtime is
generated from it by Sync (see the storage-model section). Because the `/conf` page format —
parsys + step component resource types + the serialized model — is managed by the Workflow editor,
the reliable way to produce a correct, deployable model is **build-in-editor → capture `/conf`**:

1. **Author in the Workflow editor.** Give the user the **precise editor steps** from the chosen
   reference's "UI steps" section: create the model at
   `/conf/global/settings/workflow/models/{workflowName}`, add the **Stages**, then add each step
   (Assign Task, OR/AND split, Generate DoR, Send Email, Sign) and configure it with the
   **documented UI fields** in [workflow-model-spec.md](./references/workflow-model-spec.md) —
   Route Variable `actionTaken`, routes `Approve`/`Reject`, assignee, form selection,
   data/attachment/DoR output, due date.
2. **Sync** the model so the `/var/workflow/models/{workflowName}` runtime is generated, and verify
   it runs.
3. **Capture the `/conf` design tree** —
   `ui.content/.../jcr_root/conf/global/settings/workflow/models/{workflowName}` — into source
   control with the filter root in Step 5. This is the committed deliverable. (Optionally also
   capture the generated `/var` tree so the runtime ships without a manual Sync per environment.)

Use the reference **`cq:WorkflowModel` scaffold XML** to plan the flow, choose stages/variables,
and configure the steps in the editor — and to review diffs. **Do not write that `cq:WorkflowModel`
XML into `/conf`** (the design model is a `cq:Page`, not a `cq:WorkflowModel`) and do not present it
as the deploy artifact; its `PROCESS`/`PROCESS_ARGS` values are unverified and confirmed only by
configuring the steps in the editor.

> The documented Assign Task UI field list, the node-type catalogue, the (logical) scaffold, and
> the OR-split rule syntax are all in
> [workflow-model-spec.md](./references/workflow-model-spec.md). The OR-split rule function is the
> one model detail that is well-established: `meta.get('actionTaken', String)` — capital-S `String`.

### 4.2 Get the runtime generated (Sync) — do not hand-author `/var`

After the `/conf` design model deploys, the `/var/workflow/models/{workflowName}` runtime must
exist before the submit action can resolve it. It is **generated by Sync**, not hand-written: open
the model in the Workflow editor once and **Sync**, or package the generated `/var` tree alongside
`/conf` so it ships immediately. Never author a `cq:WorkflowModel` by hand as the source of truth —
committing one to `ui.content` **fails the build** (jackrabbit-nodetypes rejects `jcr:title`,
`version`, transition `x`/`y`).

**Reach the running instance? Skip the editor entirely.** Create the `/conf` design `cq:Page` via
Sling POST and generate `/var` with one call:
`POST {model}/jcr:content.generate.json` (`ModelGenerateServlet`) →
`{"msg":"Model successfully generated.","modelPath":"/var/workflow/models/{name}"}`. Full node
shape, the `generate.json` call, and the Windows/MSYS path-mangling caveat are in
[workflow-model-spec.md](./references/workflow-model-spec.md) → "Deploy a model WITHOUT the editor".

### 4.3 Submit-action wiring — "Invoke an AEM Workflow" (Core Components)

This is what makes **Submit start the workflow**. On the form's
`guideContainer` (/content/forms/af/{project}/{formName}/jcr:content/guideContainer),
configure the submit action to invoke the model. UI path: open the form → select **Guide
Container** → properties → **Submission** tab → **Submit Action = Invoke an AEM Workflow** →
pick the **Workflow Model** → set **Data File Path** (`data.xml`), **Attachment Path**
(`attachments/`), and **Document of Record Path** by path, or by **variable** if the model uses
external data storage.

> ⚠️ The current Core Components action is **`fd/dashboard/components/actions/aemworkflowsubmit`**,
> whose properties are **`dataXMLPath` + `dataXMLType=FOLDER_PAYLOAD`** (NOT the legacy `afDataFile`,
> which this action ignores). Leaving the data path empty/absolute/URL makes Submit fail 500 with
> *"No output path relative to payload, configured to store the form data"* and **no instance is
> created**. `dataXMLPath` is just where the raw payload data lands (`<payload>/data.xml`) for the
> workflow to read — not any step's output location. Set it to a payload-relative path like
> `data.xml`.

> ⚠️ **If the form has ANY file-upload / attachment field, you MUST also set the attachment
> output path** — `attachmentsFolderPath` (payload-relative, e.g. `attachments`) + `attachmentsType=FOLDER_PAYLOAD`.
> Omitting it makes Submit fail **502** with
> *"The form contains field attachments but no output attachment path specified in submit
> configurations"* (`FormsWorkflowException`) and **no instance is created** — even though
> `dataXMLPath` is set. Check the form for a `fileinput`/`file-input` field before wiring; if one
> exists, both attachment properties are required, not optional. Same rule applies to DoR: set
> `dorPath` + `workflowDorType` only when `dorType` ≠ `none`.

> ⚠️ **Preserve any existing `clientLibRef` on the guideContainer — do NOT overwrite it.**
> `clientLibRef` is a **comma-separated list** of categories. A form usually already references its
> validation/custom-functions clientlib here (often under `/apps/clientlibs/{form}-clientlib`, a
> DIFFERENT path from `/apps/{project}/clientlibs` — grep BOTH before concluding a category is
> unused). If you are attaching a new clientlib for the workflow (e.g. a download helper), **append**
> it: `clientLibRef="existing.category,your.new.category"`. Replacing it silently drops the form's
> Rule-Editor validation functions (`isValidName`, etc.), so every Validate rule referencing them
> breaks and the form can no longer be submitted.

The full property table (modern vs. legacy), the path-vs-variable rule, and the exact field
meanings are in [submit-action-and-launcher.md](./references/submit-action-and-launcher.md).

> If the model uses Assign Task steps that render the form, the form must be **published** for
> those tasks to open. The workflow itself starts on **author**.

### 4.4 Dynamic-participant / timeout ECMA scripts (only if assignees are dynamic)

Save under `ui.apps/.../jcr_root/apps/{project}/workflow/scripts/{workflowName}/` and
reference that absolute path in the Assign Task step's Participant Chooser. **Never** `/etc`.
Template in [conditional-routing.md](./references/conditional-routing.md).

### 4.5 Email templates (only if a Send Email step is used)

Store the HTML template under `ui.apps/.../jcr_root/apps/{project}/workflow/notification/email/{workflowName}/`
and reference its absolute path from the Send Email step (do not rely on `/etc/notification`).
Template + available `${…}` variables in [service-workflow.md](./references/service-workflow.md).

### 4.6 Workflow launcher (only if a JCR-event trigger is needed instead of/alongside submit)

OSGi cfg.json in
`ui.config/.../jcr_root/apps/{project}/osgiconfig/config.author/com.adobe.granite.workflow.core.launcher.WorkflowLauncherImpl~{workflowName}.cfg.json`.
Format in [submit-action-and-launcher.md](./references/submit-action-and-launcher.md).
For form submissions the **submit action (4.3) is the primary trigger** — only add a launcher
when the requirement is event-driven.

### 4.7 Day CQ Mail Service + Link Externalizer (required for any email/notification)

`ui.config/.../jcr_root/apps/{project}/osgiconfig/config.author/com.day.cq.mailer.DefaultMailService.cfg.json`
and the Link Externalizer config. Secrets use `$[secret:key]`, never hardcoded. Snippets in
[submit-action-and-launcher.md](./references/submit-action-and-launcher.md).

### 4.8 Custom Java process step (save submission to DAM, call a service, transform data)

When no built-in step does what the requirement needs — e.g. **"save the submitted form data as
a PDF/JSON/CSV asset in `/content/dam/...`"**, "push it to an external API", or "transform it" —
the deliverable is a custom `com.adobe.granite.workflow.exec.WorkflowProcess`, selected in the
model's Process step by its FQCN. This is a **fully code-generatable (Tier B)** artifact set —
not editor work. Pattern + Cloud-safe rules: [service-workflow.md](./references/service-workflow.md).
**Model it on the closest existing Cloud-safe step** in `core/.../forms/workflow/`:
`SaveFormDataAsJsonProcess` (DAM write + JSON/XML payload sniffing),
`SaveFormDataAsPdfProcess` (adds a dependency-free PDF writer, mirrors `GeneratePDFServlet`), or
`SaveFormDataAsExcelProcess` (JDK-only OOXML `.xlsx` writer + read-modify-write append to a DAM
asset). All three pair with a committed `/conf` design model — `save-form-data-pdf-workflow` /
`save-form-data-excel-workflow` — so copy that whole artifact set (Java + `.cfg.json` +
service-user mapping + repoinit + `/conf` model + filter root) and adapt. ⚠️ Do **not** copy
`ExcelExportProcess` — it writes to the local filesystem and does not deploy to Cloud.

A custom step needs **all** of these, or it silently fails to deploy / write / be selectable:

| Artifact | Path | Note |
|---|---|---|
| Java `WorkflowProcess` | `core/.../forms/workflow/{Name}Process.java` | `@Component(service=WorkflowProcess.class)` with `property={"process.label=…"}` (the label is what's selectable in the editor's Process dropdown). Read the payload with the **workflow session**; write the asset with a **service user** in try-with-resources. |
| OSGi config | `ui.config/.../osgiconfig/config.author/{fqcn}.cfg.json` | `@Designate`/`@ObjectClassDefinition` config — target DAM folder, etc. |
| Service-user mapping | `ui.config/.../osgiconfig/config/org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended~{name}.cfg.json` | `{bundle}:{subservice}={system-user}`. **Reuse an existing subservice** (e.g. `pdf-writer`) if one already maps your bundle, rather than minting a new one. |
| Repoinit | `ui.config/.../osgiconfig/config/org.apache.sling.jcr.repoinit.RepositoryInitializer~{name}.cfg.json` | `create path (sling:OrderedFolder) {damFolder}` **and** `set ACL for {user}\n allow jcr:read,rep:write on {damFolder}\nend`. Without the ACL the write fails with `LoginException`/access-denied. |
| Filter coverage | already-covered roots | `osgiconfig` is covered by ui.config's filter; Java needs none. Only add a `ui.apps` `/apps/{project}/workflow` root if you add scripts/templates. |

**Cloud-safe rules (from `service-workflow.md`, load-bearing):** never write to the local
filesystem (`new File(...)`) — write through `AssetManager.createAsset(path, stream, mime, true)`
(the `true` auto-commits; **do not** also call `resolver.commit()` — it adds a checked
`PersistenceException`). Read `<payload>/data.xml` as **JSON-or-XML** (sniff `{`/`[`, strip BOM) —
a JSON-schema form stores JSON in that file. Throw `WorkflowException` on failure so the instance
surfaces the error. Deploy with **`-PautoInstallSinglePackage`** (the all package) so the bundle
actually updates — `-pl core` alone won't refresh the running bundle.

### 4.9 Auto-download the generated file in the browser (async workflow + client polling)

When the requirement is "save the submission as a PDF/file **and then download it in the
browser**", remember: **a workflow runs server-side and asynchronously — it cannot push a download
to the browser.** The submit returns before (or independently of) the step that writes the asset.
The working pattern is **token correlation + client polling**, not anything in the model:

1. **Hidden correlation field on the form.** Add a hidden field (e.g. `submissionId`,
   `textinput` with `visible={Boolean}false`) to the `guideContainer`. A small clientlib sets it to
   a generated UUID on load via `input.value = uuid` + dispatched `input`/`change` events, so the AF
   Core Components model serialises it into `data.xml`. Verify it actually lands by checking the
   submit POST body (`/adobe/forms/af/submit/…`) contains the id — setting the DOM value without
   dispatching events does **not** update the model.
2. **Deterministic asset name.** The custom Java step (4.8) names the asset **exactly**
   `<submissionId>.{ext}` (no timestamp) when the correlation field is present, so the browser can
   fetch that precise file. Keep the legacy timestamp name as a fallback when it's absent (so the
   step stays reusable by other forms).
3. **A GET download servlet** (`SlingSafeMethodsServlet`, `sling.servlet.paths=/bin/{project}/…`)
   that streams `{damFolder}/<id>.{ext}` as an attachment (`Content-Disposition: attachment`) via
   the **same service user** that wrote it; returns **404 while the async workflow hasn't written it
   yet**. Sanitise the `id` (whitelist `[A-Za-z0-9._-]`) so it can't traverse the repo.
4. **The clientlib polls** that servlet (`?id=<uuid>`, e.g. every 1.5s up to ~60s) after submit and
   downloads the blob via an anchor click once it gets 200.
5. ⚠️ **Set `thankYouOption=message` (inline), NOT `page`.** A redirect (`page`) navigates away and
   **kills the polling JS** before the async PDF is ready. Inline message keeps the SPA alive.
6. Attach the clientlib by **appending** its category to the form's `clientLibRef` (see the
   clientLibRef warning in 4.3) — never replace the existing value.

This clientlib + servlet are Tier-B (code-generated) artifacts. The clientlib lives under
`ui.apps/.../apps/{project}/clientlibs/clientlib-{name}/` (already covered by the ui.apps filter).

---

## Step 5 — Filter entries (exact, for this repo)

A deployed node not covered by a filter root is silently dropped. Add to the right module:

**`ui.content/src/main/content/META-INF/vault/filter.xml`** — the `/conf` design model is the
**required** root; add it only once the `/conf` tree exists in `ui.content`:
```xml
<filter root="/conf/global/settings/workflow/models/{workflowName}" mode="update"/>
```
Add the `/var` runtime root **only if** you are also packaging the generated runtime (otherwise
leave it out — the runtime is generated by Sync, and a filter root with no content can purge it):
```xml
<filter root="/var/workflow/models/{workflowName}" mode="update"/>
```

**`ui.apps/src/main/content/META-INF/vault/filter.xml`** — only if you created scripts/email
templates under apps:
```xml
<filter root="/apps/{project}/workflow"/>
```

**`ui.config/src/main/content/META-INF/vault/filter.xml`** already covers
`/apps/{project}/osgiconfig` — the launcher and mail cfg.json land under it,
so **no new ui.config filter is needed**. Verify before assuming.

The submit-action change (4.3) edits an existing form already covered by
`/content/forms/af/{project}` — no new filter needed.

---

## Step 6 — Deploy and verify

> **Pipeline mode (delegated by `groundsmith`): SKIP this whole step — author the `/conf` design model
> only.** Deployment is centralized in the `forgemaster` lead (AGENTS.md → "Deployment is centralized in
> Forgemaster") — this covers BOTH the `mvn` build/deploy AND the `/conf` → `/var` `generate.json` call
> below. Forgemaster generates the runtime on the local SDK right after its build (its AGENT.md step
> 3c), and Sentinel generates it again on cloud DEV right before testing (its AGENT.md entry-gate step)
> — reading the model name from `groundsmith.md`'s `artifacts.workflow` entry. Do **not** call
> `generate.json` yourself in pipeline mode; just make sure the model name is recorded there. Run the
> `mvn` commands AND the `generate.json` call yourself only when this skill is invoked **standalone /
> directly** (no groundsmith/forgemaster/sentinel in the picture).

```bash
# Full build + deploy to the local SDK (installs the ALL package)
mvn clean install -PautoInstallSinglePackage

# Single module, faster while iterating — note the DIFFERENT profile:
mvn clean install -PautoInstallPackage -pl ui.content   # model + form change
mvn clean install -PautoInstallPackage -pl ui.config    # launcher / mail
```

> ⚠️ **Profile gotcha — `-PautoInstallSinglePackage -pl ui.content` builds but installs NOTHING.**
> The `autoInstallSinglePackage` profile only installs the **all** module; with `-pl ui.content`
> the `all` module isn't built, so the SDK never receives the package — `BUILD SUCCESS` with the
> live node unchanged (a confusing "deployed but nothing changed"). For a single module use
> **`-PautoInstallPackage`** (installs that module's own package — look for `Package installed in …ms`
> in the log to confirm). Use `-PautoInstallSinglePackage` only for a full reactor build.

> Core-bundle changes (a custom Java process step) need the full **all** package —
> `mvn clean install -PautoInstallSinglePackage` (not `-pl core` alone) — to actually update the
> running bundle.

Verify on the running instance (default `http://localhost:4502`):
1. **Model present & editable** — `/libs/cq/workflow/admin/console/content/models.html` lists
   the model; opening it shows the steps (not "not editable"). If not editable, the `/conf`
   design copy is missing — deploy it or open+Sync/`generate.json` once.
2. **Submit starts it** — submit the form; a new instance appears under
   `/var/workflow/instances` and the first Assign Task lands in the assignee's **AEM Inbox**
   (`/aem/inbox`). The assignee must be in the `workflow-users` group. If submit 500s instead,
   check `dataXMLPath` (4.3) and read `crx-quickstart/logs/error.log` for the real cause —
   custom steps that throw `WorkflowException` surface their root cause there.
3. **Routing works** — click a route (e.g. **Approve**); the OR-split sends the instance down
   the matching branch (confirm `actionTaken` was captured).
4. **DoR / email** — if configured, the PDF is produced at the DoR path and the notification
   email is sent (Mail Service + Link Externalizer configured).

---

## Delivery format

When you respond, always:
1. Put the **file path as a comment/caption above each code block**, and give **complete**
   XML/JSON — never truncated.
2. For anything configured in the AEM editor (the model itself, every Assign Task / Sign / DoR
   step, and the submit action), give **explicit UI steps** — not just raw XML.
3. Clearly separate **Tier A (build in editor)** from **Tier B (committed files)** so the user
   knows what to author in the UI vs. what you have already written to the repo.
4. End with a **deployment checklist** (the pre-flight checklist below), and state what you did
   NOT verify (e.g. process-step class names, the exact `actionType` token).

---

## Critical rules (per Adobe docs + verified behaviour)

- **Authoring model**: the workflow model and its Assign Task / Sign / DoR / routing steps are
  built in the **Workflow editor** (documented UI fields), then Synced and captured to
  `ui.content`. Hand-authored `PROCESS`/`PROCESS_ARGS` runtime XML is an **unverified scaffold**,
  not final output — confirm in the editor.
- **Storage**: the deployable model is the **`/conf/global/settings/workflow/models` design model**
  (a `cq:Page`); the `/var/workflow/models` runtime is **generated by Sync** — never hand-author it,
  never write a `cq:WorkflowModel` into `/conf`. Never `/etc/workflow/models`, never `/libs`.
- **Reuse first**: scan for an existing model and reuse/extend it before creating a new one.
- **Route Variable** on every Assign Task that branches must be `actionTaken` (String,
  declared in the `/conf` design model's `jcr:content/variables`).
- **OR-split condition — prefer the editor's Rule Definition builder, not a hand-written
  script.** Configure the split's Approve/Reject condition via the Workflow Model editor's
  graphical "Rule Definition" UI (pick variable `actionTaken`, operator "Equals", literal
  `Approve`/`Reject`) — this persists as an `expression{N}` JSON property and is the
  **live-confirmed working** mechanism for `employee-training-request-approval`'s real Inbox
  Approve/Reject routing. See workflow-model-spec.md → "OR-split condition: use the editor's
  Rule Definition builder" for the exact JSON shape and how to capture/replicate it headlessly.
  Only fall back to a hand-written `script{N}` ECMA rule (`meta.get('actionTaken', String) ==
  'Approve'` — capital-S `String`, exact button label) for conditions the builder can't express
  (multi-variable/numeric logic) — and if you do, it **must** read
  `graniteWorkflowData.getMetaDataMap()`, **NEVER** `workItem.getWorkflowData().getMetaDataMap()`.
  Live-proven by decompiling `com.adobe.granite.workflow.core.rule.ScriptingRuleEngine` (the class
  that evaluates every rule script): its bindings are
  `graniteWorkflowData`/`workflow`/`workflowSession`/`jcrSession` — `workItem` is never bound. A
  script using `workItem...` throws `ReferenceError: "workItem" is not defined` the moment a
  route is evaluated, which makes the task **un-completable** (not just mis-routed) since the
  same script gates the Approve/Reject buttons. `graniteWorkflowData` already IS the
  `WorkflowData` object — no `.getWorkflowData()` call needed on it. Even with that fix applied,
  the ECMA path still failed real Inbox routing on this project (the Inbox's pre-click
  route-preview call found no valid route) — the Rule Definition builder is what actually worked,
  so treat `script{N}` as a fallback, not the default.
- **Submit action (Core Components)**: use the modern `fd/dashboard/components/actions/aemworkflowsubmit`
  with **`dataXMLPath` + `dataXMLType=FOLDER_PAYLOAD`** (and `attachmentsFolderPath`/`attachmentsType`,
  `dorPath`/`workflowDorType`). **Not** the legacy `afDataFile`/`afAttachmentsPath`/`afDoRPath` — the
  modern action ignores them, which causes the "No output path relative to payload" submit failure.
  `dataXMLPath` must be a payload-relative path (e.g. `data.xml`), never empty/absolute/a URL.
- **Reading the payload in a custom step**: `<payload>/data.xml` holds **JSON** for JSON-schema forms
  (and the Dashboard submit), not XML — sniff `{`/`[` and handle JSON; strip BOM; XML-parse only as a
  fallback. Dashboard payloads live under `/var/fd/dashboard/payload/...`.
- **Custom Java process step**: implement `WorkflowProcess`, select by FQCN in the model's Process
  step, write through the repo (`AssetManager`) via a **service user** — never the local filesystem
  (Cloud has none). Throw `WorkflowException` on failure so the error surfaces.
- **"Set Status" steps must write the payload, not just the workflow variable**:
  `com.adobe.granite.workflow.core.process.SetVariableProcess` only writes the running instance's
  `metaDataMap` — never the submitted payload's `data.xml`. If the status value must be visible on
  a form re-render (a `dataRef`-bound dropdown, a Rule-Editor show/hide or unlock rule, another
  assignee's task), use a custom step that sets the variable **and** writes the payload field (see
  `core/.../forms/workflow/SetStatusVariableAndPayloadProcess.java` and service-workflow.md).
  Plain `SetVariableProcess` remains correct when only a later **workflow** step reads the value —
  and even then, ONLY for literal values (see the next bullet for reading values, which it cannot do).
- **"Capture Submission Variables" steps must actually parse the payload — `SetVariableProcess`'s
  `${payload.jcr:content/data/...}` EL does NOT work on this project's forms.** That EL is walked as
  a literal JCR node/property path; it only resolves if the payload is stored as real, expanded JCR
  child nodes. This project's forms (`schemaType=jsonschema`) store the whole submission as ONE
  opaque JSON blob (a `jcr:data` Binary property on `<payload>/data.xml/jcr:content`) — there is no
  child node to walk to, so the variable silently ends up blank regardless of what the form field
  held (live-confirmed against a real submitted `data.xml`). Use
  `com.aem.forms.agents.forms.workflow.CaptureSubmissionVariablesProcess` (this project's own step)
  instead — it parses the JSON blob with Jackson and reads a dot-separated `payloadPath` out of it,
  alongside `literalValue` entries for variables with no payload source. See
  workflow-model-spec.md → "Set Variable Step PROCESS_ARGS" for the exact `PROCESS_ARGS` shape.
- **Never wire `WORKITEM_COMMENT` to capture an assignee's typed comment** — it crashes task
  completion for plain text (`WorkflowException: "Invalid value : ..."`). Read the comment from
  workflow **history** in a later step instead, checking the completed `WorkItem`'s
  `workitemComment` metadata key (AEM Forms' own completion-dialog write target) before falling
  back to `HistoryItem.getComment()` — see service-workflow.md and troubleshooting.md §7b.
- **Payload type** is always `JCR_PATH` for adaptive forms (never `BLOB`).
- **External data storage**: if the model is marked for it, every step's data/attachments/DoR
  must use the **variable** option, not a payload path. All variables must be declared first.
- **DoR prerequisite — THREE separate places must all be configured, not just the workflow step.**
  A workflow's "Generate Document of Record" step (`afToDorStep` /
  `com.adobe.fd.workflow.dorGeneration.AFtoDORStep`) reads its input form directly off the JCR — it
  does not receive DoR config as step arguments — so all three of the following must be true or the
  step throws `"Not a valid Adaptive Form"` (or silently produces no PDF), even though the model itself
  authors and deploys fine:
  1. **Form Properties (the form page's `jcr:content`)** — the page needs the AF marker property
     `guide="1"` (String). Live-decompiled evidence
     (`com.adobe.aemfd.adobe-aemfd-workflow-process-common`, `AFtoDORStep.internal_execute`,
     `AFtoDORStep.java:125`): it builds `<AF_PATH>/jcr:content` and throws unless
     `node.hasProperty("guide") && node.getProperty("guide").getString().equals("1")`. A Core
     Components AF page does **not** carry this marker by default — add it explicitly. The SAME
     marker is what the Workflow/DoR editor's own advisory ("is this form DoR-configured?") checks,
     so a missing `guide="1"` also shows as "not DoR-configured" in the UI, not just a runtime failure.
     This check is **independent of `dorType`** — setting `dorType` alone (below) with `guide` absent
     has zero effect (a real fix pass on this project hit exactly that and it did nothing).
  2. **Form Properties — the DAM guide asset's `jcr:content/metadata` node**
     (`/content/dam/formsanddocuments/{appFolder}/{formName}/jcr:content/metadata`,
     `sling:resourceType=fd/fm/af/render`) — `dorType` must be `"generate"` (the form renders as its
     own DoR — the Core Components-idiomatic default when no separate print/XDP template exists) or
     `"select"` with `dorTemplateRef` pointing at a genuine DoR template asset (a real `dam:Asset`
     with a binary rendition — never a `cq:Page`). ⚠️ **Live-verified the hard way: `dorType` and
     `dorTemplateRef` do NOT live on `guideContainer`.** A direct edit there is inert — the
     guideContainer component's own dialog has no DoR-template field at all, and neither the editor's
     Form Properties UI nor `AFtoDORStep` reads `dorType`/`dorTemplateRef` from it. Set them on the
     DAM guide asset's `metadata` node instead; leave `guideContainer`'s own `dorType` (if it has one
     at all — many forms don't) untouched. `dorTemplateRef` is **not read by `AFtoDORStep`** either
     way — only `dorType` matters to that step; `dorTemplateRef` only matters for `dorType="select"`'s
     own template resolution elsewhere (the Form Properties template picker / DoRService).
  3. **The workflow's Generate DoR step** — present in the model (`afToDorStep`,
     `PROCESS=com.adobe.fd.workflow.dorGeneration.AFtoDORStep`), pointed at the right `AF_PATH`
     (the form whose DoR the step should render), and its own three payload-facing fields
     (`INPUT_DATAXML`, `INPUT_ATTACHMENT`, `DOR_PATH`) filled in with a real, colon-prefixed
     `"CATEGORY:value"` string — **leaving them bare (no category prefix) is NOT equivalent and can
     make the step throw at runtime.** Live-verified via a real Workflow-editor session (selecting
     "Input Data = Relative to Payload / data.xml", "Input Attachments = Relative to Payload /
     attachments", "Document of Record = Stored Under Payload Folder", then exporting the saved
     design model) that the ACTUAL persisted tokens are:
     - `INPUT_DATAXML="FOLDER_PAYLOAD:data.xml"`
     - `INPUT_ATTACHMENT="FOLDER_PAYLOAD:attachments"`
     - `DOR_PATH="RELATIVE_PLOAD:DocumentofRecord/DoR.pdf"` (any payload-relative path)
     Do **not** guess a different category token for these three fields (e.g. `RELATIVE_PLOAD` for
     the two inputs, or `UNDER_PLOAD` for the output) — an earlier pass on this project did exactly
     that, got a `WorkflowException: Invalid type for resolving property RELATIVE_PLOAD` crash from
     `AFtoDORStep`'s own `PropertyResolver.getColonSeparatedPropertyValue`, and wrongly concluded
     colon-prefixed values are unsafe on this step in general — the crash was actually from using the
     wrong token, not from the colon-prefix format itself. If the model's data is stored externally,
     use the **variable** category instead (same widget, different token) for the DoR output.
  All four must be checked together whenever DoR is wired or debugged — Formwright/Groundsmith
  authoring only #2 and #3 while missing #1, or getting #3's category tokens wrong, are the two most
  common ways a DoR step deploys clean and still fails (or silently no-ops) at runtime.
- **Assignees** must belong to `workflow-users`. Interactive Communications also need
  `cm-agent-users`.
- **Notifications** need `notifyParticipant=true` on the step **and** Day CQ Mail Service
  **and** Day CQ Link Externalizer configured.
- **Every email recipient — Assign Task's own notification AND every Send Email step — must be a
  workflow variable, never a literal.** `RECIPIENT_EMAIL_RESOLUTION="VARIABLE"`/`EMAIL_VARIABLE=...`
  on Assign Task; `toAddressType="Variable"`/`toAddressValue=...` on Send Email. If no real value
  source exists for that variable yet, `AskUserQuestion` for a default recipient address while
  authoring, then set it in BOTH the variable's own `defaultValue` property (`jcr:content/
  variables/<name>/defaultValue` — NOT `value`) and as a literal in the "Capture Submission
  Variables" Set Variable step — do not leave it blank, do not invent a plausible address, and do
  not reuse an unrelated variable
  (e.g. the submitting applicant's email) for a different audience's notification.
- **Adobe Sign** step uses `PROCESS_AUTO_ADVANCE="false"` (it waits for signing) and requires
  the Adobe Sign cloud config under `/conf/.../settings/cloudconfigs`.
- **Scripts** under `/apps/{project}/workflow/scripts/`, never `/etc`.
- **Secrets** in cfg.json use `$[secret:key]`, never hardcoded SMTP passwords.

---

## Failure symptoms and causes

| Symptom | Cause | Fix |
|---|---|---|
| Submit does nothing / no instance created, or model missing from Tools → Workflow → Models after a clean deploy | Submit action not set, wrong model path, or the `/var` runtime was never generated | Wire `guideContainer` Submission tab (4.3); in pipeline mode this is Forgemaster's job (its AGENT.md step 3c, locally) and Sentinel's job (its AGENT.md entry-gate step, on cloud DEV) — confirm THAT actually ran (check `test/forgemaster/code-quality-report.md` / `test/sentinel/test-report.md` for `workflow_models_generated`) rather than re-authoring anything; standalone use: deploy the `/conf` design model then **Sync**/`generate.json` so `/var/workflow/models/{name}` exists |
| Submit fails 500: *No output path relative to payload, configured to store the form data* | Modern `aemworkflowsubmit` action has `dataXMLPath` empty/absolute/URL, or you set legacy `afDataFile` which it ignores | Set `dataXMLPath=data.xml` + `dataXMLType=FOLDER_PAYLOAD` on `guideContainer` (4.3) |
| Submit fails 502: *form contains field attachments but no output attachment path specified* (`FormsWorkflowException`) | Form has a file-upload field but the action has no attachment output path | Add `attachmentsFolderPath=attachments` + `attachmentsType=FOLDER_PAYLOAD` on `guideContainer` (4.3) |
| Form fields all show invalid / can't submit after wiring | Overwrote the guideContainer `clientLibRef`, dropping the validation clientlib that defines the Rule-Editor functions | `clientLibRef` is comma-separated — **append** your category, keep the existing one (4.3) |
| `BUILD SUCCESS` but the deployed node/bundle is unchanged | Used `-PautoInstallSinglePackage -pl {module}` — that profile only installs the `all` module, so a single-module build installs nothing | Single module → `-PautoInstallPackage -pl {module}`; full build → `-PautoInstallSinglePackage` (Step 6) |
| Custom step fails: `SAXParseException: Content is not allowed in prolog` | Step assumed `data.xml` is XML, but a JSON-schema form stores it as JSON (or a BOM) | Sniff first char (`{`/`[` → JSON, use as-is) + strip BOM; XML-parse only otherwise (service-workflow.md) |
| Build fails: `jcr:title`/`version`/`x`/`y` not allowed on `cq:WorkflowModel`/`cq:WorkflowTransition` | A `/var` runtime model was hand-authored into `ui.content` | Remove it — `/var` is generated, never committed; use `/conf` design model + `generate.json` (4.1/4.2) |
| Model runs but shows **"not editable"** in editor | Only the `/var` runtime was deployed; the `/conf` design model is missing | Deploy the `/conf/global/settings/workflow/models` design model (4.1) — it is the source of truth |
| Model not found / can't be resolved after deploy | Deployed `/conf` design model but never Synced, so no `/var` runtime exists | Open the model in the editor and **Sync**, or also package the generated `/var` tree |
| Task never appears in Inbox | Assignee not in `workflow-users`, or user/group doesn't exist, or form not published | Add to `workflow-users`; verify the assignee; publish the form |
| OR-split always takes one branch | Route Variable not `actionTaken`, or rule uses lowercase `string` | Set Route Variable `actionTaken`; use `meta.get('actionTaken', String)` |
| Task's Approve/Reject buttons don't work at all (not "wrong branch" — completion itself fails), error.log shows `ScriptEvaluationException ... ReferenceError: "workItem" is not defined` from `/libs/workflow/scripts/dynamic.ecma` | The OR-split's `script{N}` rule used `workItem.getWorkflowData().getMetaDataMap()` — `workItem` is never bound in `ScriptingRuleEngine`'s evaluation context | Change every rule script to `graniteWorkflowData.getMetaDataMap()` (no `.getWorkflowData()` — it's already the `WorkflowData` object); regenerate; note that any ALREADY-RUNNING instance is pinned to the old model and must be terminated — only a fresh submission picks up the fix. **If the Inbox still can't route even after this fix**, don't keep patching the ECMA — reconfigure the split's condition via the editor's Rule Definition builder instead (persists as `expression{N}`, not `script{N}`) — this is what actually fixed real routing on `employee-training-request-approval`; see workflow-model-spec.md → "OR-split condition: use the editor's Rule Definition builder" |
| Model deploys clean and generates with no error, but runs fully sequential — no real branch at all | Used `sling:resourceType="cq/workflow/components/model/orsplit"` (extra "split") — **not a registered component** (404); `ModelGenerateServlet` silently drops it instead of erroring | Use the REAL resourceType `cq/workflow/components/model/or` with branch children named literally `1`/`2` (not descriptive names) — see workflow-model-spec.md → "Authoring a REAL OR-split headlessly" for the full live-verified recipe, including which property (`script{N}`) actually carries each branch's rule |
| Real `or` split + nested branches fails `generate.json` with "Unable to save workflow model" / "Split steps must have a valid step in every branch" | Branch child nodes named descriptively (e.g. `approve`/`reject`) instead of the literal numerals the component's own renderer (`split.jsp`) iterates | Name branch children `1`, `2`, … (ISO9075-escaped as `_x0031_`/`_x0032_` in committed DocView XML) with `sling:resourceType="cq/flow/components/parsys"` — see workflow-model-spec.md for the exact shape |
| OR-split branch has no visible condition even after setting `rule`/`condition`/`conditions`/`branchCondition{N}` on the split or branch node | None of those properties are read by the generator for this component | Use `script{N}` (1-indexed) directly on the split node — the only property confirmed to survive into the generated transition's `rule` |
| Email not sent | Mail Service or Link Externalizer not configured, or `notifyParticipant=false` | Add 4.7 config; set `notifyParticipant=true` |
| Assign Task's "Send Notification Email" never arrives — `EmailService` logs `NullPointerException` from `StrSubstitutor.replace(Object)`, or `FormsWorkflowException: "Email template is not defined"` | `HTML_EMAIL_TEMPLATE_LOCATION` is set — live-proven broken on this platform for `com.adobe.fd.workspace.step.service.EmailService` (the Assign Task step's own mailer), regardless of path shape (plain `nt:file` path or `/jcr:content`-suffixed) | Remove `HTML_EMAIL_TEMPLATE_LOCATION` entirely — leave it unset. Keep `RECIPIENT_EMAIL_RESOLUTION`/`EMAIL_LITERAL`/`EMAIL_VARIABLE` (that part works). Put the template's information in the step's own `jcr:description` (Description field) instead, or use a dedicated Send Email step (`com.adobe.fd.workflow.email.SendEmailStep`) if a branded template is required — see workflow-model-spec.md's "Assign Task" section |
| DoR step fails (`"Not a valid Adaptive Form"`) or the editor's DoR advisory shows "not configured", even though `dorType` is already set | Missing the `guide="1"` marker property on the form page's `jcr:content` — `AFtoDORStep` gates on this independently of `dorType`/`dorTemplateRef` (see "DoR prerequisite" above) | Add `guide="1"` (String) to the form page's `jcr:content`, keep `dorType="generate"` (or `"select"` + a real `dorTemplateRef`) on the **DAM guide asset's `jcr:content/metadata`**, and confirm the workflow's DoR step points at the right `formPath` — all three, not just one |
| DoR renders the OLD layout (auto-generated, or the previous template) after setting `dorType="select"`+`dorTemplateRef` — package deployed clean, no error | Set `dorType`/`dorTemplateRef` directly on `guideContainer` by hand-editing the JCR instead of the DAM guide asset's `metadata` node — `guideContainer` has no DoR-template field, so the write is silently inert | Move the properties to the DAM guide asset's `jcr:content/metadata` node (`/content/dam/formsanddocuments/{appFolder}/{formName}/jcr:content/metadata`); revert whatever was added to `guideContainer` |
| Generate DoR step's editor dialog shows "Input Data"/"Input Attachments"/"Document of Record" as blank/unconfigured even after hand-editing `INPUT_DATAXML`/`INPUT_ATTACHMENT`/`DOR_PATH`, or the step throws `WorkflowException: Invalid type for resolving property <TOKEN>` at runtime | Wrong or missing `"CATEGORY:value"` category prefix on these three fields — a bare value (`data.xml`) or the wrong token (`RELATIVE_PLOAD:data.xml`/`UNDER_PLOAD:...`) both fail; the real tokens are `INPUT_DATAXML="FOLDER_PAYLOAD:data.xml"`, `INPUT_ATTACHMENT="FOLDER_PAYLOAD:attachments"`, `DOR_PATH="RELATIVE_PLOAD:<payload-relative path>"` | Set the three fields to the exact tokens above (live-verified via a real editor save — see "DoR prerequisite" #3); re-deploy and re-test rather than guessing another token |
| Model/script missing after deploy | Not covered by a `filter.xml` root, or placed under `/etc` | Add the filter roots in Step 5; move scripts to `/apps/{project}/workflow/scripts` |
| Variables empty between steps | Variable not declared, or type mismatch (`string` vs `String`), or external-storage path used | Declare in the `/conf` design model's `jcr:content/variables` (see workflow-model-spec.md → "Workflow Variables"); match type; use variable option under external storage |
| Variables panel in the Workflow Model editor doesn't list a variable at all, even though the model deploys clean | Declared under `jcr:content/metaData/variables` instead of `jcr:content/variables` (a sibling of `flow`) — the wrong location deploys fine but the editor UI never reads it | Move the declaration to `jcr:content/variables`; see workflow-model-spec.md → "Workflow Variables" for the exact node shape (`name`/`type`/`defaultValue`/`additionalProperties`) |
| `mvn package` fails: `ValidationViolation: ... unknown type: ... jackrabbit-docviewparser ...` pointing at a variable node | An attribute value starts with a literal `{` (e.g. `additionalProperties="{}"`) — FileVault's DocView XML reads a leading `{` as a JCR type prefix (`{Boolean}true`), and a bare `{}` parses as an empty/unknown type | Escape the leading brace: `additionalProperties="\{}"`. Applies to any `.content.xml` attribute starting with `{` in any module, not just workflow variables |
| Editor's "Default Value" field for a variable stays blank even after setting a value on the declaration | Wrong property name — `value` is NOT the backing field | Use `defaultValue` on the variable node (`jcr:content/variables/<name>/defaultValue`) — live-confirmed against the real editor. Also seed the same literal in the model's initial Set Variable step if the value must take effect at runtime, since `defaultValue` is not proven to be read by any process step |
| "Capture Submission Variables" step runs with no error, but a variable that should hold a submitted form value (e.g. `managerId` from `EmployeeDetails.ManagerId`) is always blank | Used `com.adobe.granite.workflow.core.process.SetVariableProcess` with `variableValue=${payload.jcr:content/data/...}` — that EL only resolves against a payload stored as expanded JCR child nodes; this project's JSON-schema forms store the whole submission as one opaque JSON blob, so there's no child node to walk to | Switch that step's `PROCESS` to `com.aem.forms.agents.forms.workflow.CaptureSubmissionVariablesProcess` and its `PROCESS_ARGS` to the `payloadPath`/`literalValue` shape — see workflow-model-spec.md → "Set Variable Step PROCESS_ARGS" |
| "Set Status" step runs fine, but the next task's form still shows the OLD status/value | Node uses `SetVariableProcess`, which only writes the workflow's `metaDataMap`, never the payload the form re-renders from | Use a custom step that also writes the payload field — `SetStatusVariableAndPayloadProcess` pattern (service-workflow.md, troubleshooting.md §7a) |
| Task completion throws `WorkflowException: "Invalid value : <variableName>"` | `WORKITEM_COMMENT=<variableName>` wired on the Assign Task step — the comment-save call requires `"CATEGORY:value"` input, plain text fails it | Remove `WORKITEM_COMMENT`; read the comment from workflow history in a later step instead (service-workflow.md, troubleshooting.md §7b) |
| Assignee's typed comment never appears on a later task even after reading history | Checked only `HistoryItem.getComment()` — AEM Forms' completion dialog stamps the comment onto the `WorkItem`'s `workitemComment` metadata key instead | Check `workItem.getMetaDataMap().get("workitemComment", String.class)` first, fall back to `getComment()` (troubleshooting.md §7b) |

---

## Pre-flight checklist

- [ ] **Reuse check done** — scanned `/var/workflow/models` & `/conf/global/settings/workflow/models`; reused/extended an existing model when one matched, instead of forking
- [ ] Model built/verified in the **Workflow editor** (Tier A) using the documented UI fields; scaffold XML used only to plan the flow / configure steps, never written to `/conf`
- [ ] **`/conf` design model** (`cq:Page`) captured to `ui.content` (`/conf/global/settings/workflow/models/{workflowName}`) — the required deliverable; `/var` runtime is generated by **Sync** (package it only if you want it shipped)
- [ ] Assign Task steps configured via editor: form selection, assignee, due date, notification, data/attachment/DoR output
- [ ] Branching Assign Task steps set Route Variable `actionTaken`; OR-split rules use `meta.get('actionTaken', String)`
- [ ] Typed variables declared in the `/conf` design model's `jcr:content/variables` (sibling of `flow` — NOT `metaData/variables`), each with `name`/`type` (fully-qualified, e.g. `java.lang.String`)/`defaultValue`/`additionalProperties="\{}"` (escaped brace); external-storage models use variable (not path) everywhere
- [ ] **Completeness audit — every declared variable is set AND consumed.** The "Capture Submission
  Variables" step uses `CaptureSubmissionVariablesProcess` (NOT `SetVariableProcess`'s
  `${payload.jcr:content/data/...}` EL, which does not work on this project's JSON-schema forms —
  see workflow-model-spec.md → "Set Variable Step PROCESS_ARGS") and sets every variable that has a
  real payload source via `payloadPath` from the submitted `data.xml`, and every variable without
  one gets an explicit `literalValue` (never left unset). Then grep the whole model for
  each variable name and confirm at least one downstream consumer (`EMAIL_VARIABLE`,
  `toAddressValue`, `ROUTE_PROPERTYNAME`, an OR-split `rule=`, a later Set Variable's own args, an
  email template's `${workflowData.metaDataMap.X}` — check the referenced `.html` templates too,
  not just the `.content.xml`). A variable with zero consumers is either dead (remove it) or an
  intentionally-unused placeholder for a disclosed future follow-up (document why in a comment,
  as `employee-training-request-approval`'s `managerId` does) — never leave an unexplained orphan.
- [ ] Stages declared in `metaData/stages` and referenced per step (Inbox progress)
- [ ] Submit action on `guideContainer` = `fd/dashboard/components/actions/aemworkflowsubmit` → correct `workflowModel` + **`dataXMLPath=data.xml`** + `dataXMLType=FOLDER_PAYLOAD` (NOT legacy `afDataFile`); `dataXMLPath` is payload-relative, never empty/absolute/URL
- [ ] If the form has a file-upload field: **`attachmentsFolderPath=attachments` + `attachmentsType=FOLDER_PAYLOAD`** also set (else Submit 502s); DoR path set only when `dorType` ≠ `none`
- [ ] **If the model has a Generate DoR step, all FOUR DoR places are configured, not just the step**:
  (1) form page `jcr:content` has `guide="1"`, (2) the **DAM guide asset's `jcr:content/metadata`**
  node (Form Properties — NOT `guideContainer`, which has no DoR-template field) has
  `dorType="generate"` (or `"select"` + a real `dorTemplateRef`), (3) the DoR step's `AF_PATH`
  points at the right form, (4) the DoR step's `INPUT_DATAXML`/`INPUT_ATTACHMENT`/`DOR_PATH` use the
  real `"CATEGORY:value"` tokens — `FOLDER_PAYLOAD:data.xml` / `FOLDER_PAYLOAD:attachments` /
  `RELATIVE_PLOAD:<path>` — not bare values and not a guessed token — see "DoR prerequisite" above
- [ ] Existing `guideContainer` `clientLibRef` **preserved** — appended (comma-separated), not overwritten, so the form's validation clientlib still loads
- [ ] Browser auto-download (if required): hidden correlation field + deterministic asset name + GET download servlet (404-until-ready) + polling clientlib + `thankYouOption=message` (4.9); verified the id reaches the submit POST body
- [ ] Single-module deploy uses **`-PautoInstallPackage -pl {module}`** (look for `Package installed`); `-PautoInstallSinglePackage` is for the full reactor build only
- [ ] Custom Java step (if any): reads `data.xml` as JSON-or-XML (sniff + strip BOM), writes via `AssetManager` + service user, throws `WorkflowException` on error; deployed via `-PautoInstallSinglePackage`
- [ ] Any "Set Status" step whose value the form re-renders (dataRef-bound field, Rule-Editor rule) uses the `SetStatusVariableAndPayloadProcess` pattern (writes the payload), not plain `SetVariableProcess`; no step wires `WORKITEM_COMMENT` (crashes completion) — comments are read from history's `workitemComment` metadata key
- [ ] Scripts (if any) under `/apps/{project}/workflow/scripts/`; email templates under `/apps/{project}/workflow/notification/email/`
- [ ] Launcher (if any) as `…WorkflowLauncherImpl~{name}.cfg.json` in `ui.config`
- [ ] Mail Service + Link Externalizer configured (if any email); secrets via `$[secret:]`
- [ ] Filter root `/conf/global/settings/workflow/models/{name}` added in ui.content (required); `/var/workflow/models/{name}` added **only** if also packaging the generated runtime; `/apps/{project}/workflow` in ui.apps if used
- [ ] `{project}` (`aem-demo-site`) used as the single namespace in every path (apps, conf, and the form/conf-forms folder segment)
- [ ] Verified on the running instance: model editable, submit starts instance, route branches, DoR/email fire

---

## Sources (Adobe official)

- AEM Forms Cloud Service workflow step reference:
  https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/create-form-centric-workflows/aem-forms-workflow-step-reference
- Configure the "Invoke an AEM Workflow" submit action:
  https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/forms/integrate/set-submit-action/configure-submit-action-workflow
- Configure a Submit Action (Core Components):
  https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/forms/integrate/set-submit-action/configure-submit-actions-core-components
- Creating Workflow Models (model node structure, design vs. runtime, Sync):
  https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/developing/extending-aem/extending-workflows/workflows-models
- Variables in AEM workflows:
  https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/forms/create-form-centric-workflows-on-osgi/aem-forms-workflow-variables
