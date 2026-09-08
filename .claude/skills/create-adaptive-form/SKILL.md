---
name: create-adaptive-form
description: >
  Creates a complete, working AEM Adaptive Form (Core Components, AEM as a Cloud Service)
  from scratch — including its field-level validation clientlib. Covers all 5 required
  artifacts: the form page, DAM guide asset, per-form conf context, filter entry, AND the
  per-form validation clientlib (custom functions + Rule-Editor-visible validation rules)
  derived from the requirement document — OR from a FIELD INVENTORY discovered from a form on
  a public webpage (URL replica), building the form as an EXACT VISUAL + FUNCTIONAL replica of
  that source form (the form only, not the page chrome). Follow every instruction here exactly —
  each rule exists because omitting it causes a specific, hard-to-diagnose failure.
version: 3.2.0
ide:
  cursor: .cursor/skills/create-adaptive-form/
  github-copilot: .github/skills/create-adaptive-form/
  claude-code: .claude/skills/create-adaptive-form/
---

# Skill: create-adaptive-form

## Role

You are an AEM Adaptive Forms (Core Components) author for AEM as a Cloud Service.
When asked to create an adaptive form, you produce **all 5 artifacts** defined in this
skill — in the correct structure, with the correct attributes — so the form appears in
Forms & Documents, opens in the editor, renders with its theme, AND validates its fields
at runtime. As the **mandatory final step you deploy the form to the local AEM instance
(`http://localhost:4502`) and verify it on the running server** (see "Deployment" below) —
authoring without deploying is not a complete result.

You never skip an artifact. You never skip the deploy. You never invent a path or attribute
value. You ask the user for the inputs listed below before generating any file.

> **Requirement-driven validation (always do this).** Whenever the request includes a
> requirement document in ANY format — PDF, image/screenshot, Word/text, a schema, or a
> plain-language description — extract the per-field validation rules from it (formats,
> lengths, digit counts, ranges, allowed values, cross-field constraints, file types/size)
> and deliver them as **Artifact 5: the form's validation clientlib** (custom functions +
> Rule-Editor-visible rules). Creating the form without its clientlib is incomplete. This
> is a standing project convention — see [[clientlib-per-form-convention]].

> **URL-replica build (from a field inventory discovered from a public webpage form).** When the
> input is a **field inventory** discovered from a form on a public webpage (the URL-replica flow —
> as run for the `complaint-form`), build the form as an **EXACT VISUAL + FUNCTIONAL replica of that
> source form — the form only, NOT the page chrome** (host nav/header/footer/marketing). The
> inventory gives each field's label, input type, `name`, required flag, options, placeholder, and
> client-side validation, already mapped to an AEM Core Components AF field type. Then:
> - **Keep EVERY discovered field exactly** — same label (→ `jcr:title`), mapped AF type, `required`
>   flag, options (`enum`/`enumNames`), placeholder (`placeholderText`), client-side validation, and
>   **the same ORDER** as the source. Do not add, drop, rename, or reorder fields.
> - **`aria-label` on every field** (as always) — set to the field's visible label.
> - **Match the source layout:** group fields the source shows **side-by-side** into a
>   `panelcontainer` with **2-column rows** — set each such field's `<default width="6">` (half of 12)
>   instead of the full-width `width="12"`, so the row mirrors the source. Full-width fields stay
>   `width="12"`.
> - **Reproduce client-side behaviour** (show/hide, validate, calculate, cascade) via
>   `create-form-rules`; the exact-replica **theme** comes from `create-form-theme` (bound via
>   `themeRef` + conf context) — both are handled by the build lead (formwright).
> - **Submit (default) = the shared `Custom-Submit-GeneratePDF` download-PDF-on-submit action**
>   (gated on validation — see the submit-action wiring table below). Do NOT scaffold a per-form PDF
>   action.

> **Schema assessment (decide up front).** A **data schema** (JSON Schema / XSD) and a **Form
> Data Model (FDM)** are **mutually exclusive** ways to bind a form's data — a form uses **at most
> one**.
>
> **Input-type default:** when the form is being created from a **screenshot/image or a link to an
> HTML page** (and no requirement document supplies the integration logic), **default to a JSON
> Schema (`generate-schema`), NOT an FDM** — a picture/HTML carries no system-of-record, endpoint,
> auth, or process, so an FDM source (and any workflow) would be fabricated. Reserve FDM for when a
> **requirement document with proper integration logic** defines it. (In a pipeline delivery the
> architect has already made this call; honor it.)
>
> So the schema concept applies **only when FDM is NOT used**:
> - **If the form is built on an FDM** → the data structure comes from the FDM. Do **NOT** create
>   a schema. Bind via the FDM with these **VERIFIED** attributes (read from the form-container
>   dialog — getting the value wrong leaves the editor's **Data Sources panel empty**):
>   - on the `guideContainer`: **`schemaType="formdatamodel"`** (all lowercase — NOT `formDataModel`;
>     the allowed values are `none` / `jsonschema` / `formdatamodel`) and
>     **`schemaRef="/content/dam/formsanddocuments-fdm/{project}/{fdmName}-data-model"`** (the FDM path).
>   - on each bound field: **`dataRef`** (the CC field bind reference, NOT `fd:formDataRef`), using
>     the **full dotted path from the root entity** — `dataRef="$.{Entity}.{property}"` (e.g.
>     `dataRef="$.ConversionResult.celsius"`), not the bare `$.{property}`.
>   - the DAM guide asset (Artifact 2) metadata: `formmodel="formdatamodel"`.
>   - **Submit action:** for an FDM-bound form with **no custom submit logic**, default the
>     `guideContainer` to the **"Submit using Form Data Model"** action — set
>     **`actionType="fd/afaddon/components/actions/fdm"`** (NOT the REST endpoint
>     `fd/af/components/guidesubmittype/restendpoint`) plus **`fdmEntityPath="$.{Entity}"`** (the root
>     entity bind reference to write back, e.g. `$.ConversionResult`). On submit this writes the form
>     data through the FDM's write operations. (The CC submit actions live under
>     `/libs/fd/afaddon/components/actions/` — `fdm`, `restendpoint`-equivalents, storage, etc.)
>   The FDM↔form binding is otherwise editor-managed; if the panel is still empty after a hard
>   refresh, bind it in the editor (Form Container → Form Model → select the FDM) and export. Use
>   `create-fdm` to build the FDM first.
> - **Else, if the form should be data-bound** (structured/typed submit payload, API integration,
>   data model reused across forms, or the requirement already supplies a schema / Swagger /
>   OpenAPI spec) → create a schema with the **`generate-schema`** skill as the FIRST step (before
>   Artifact 1), then set `schemaType="jsonschema"`/`schemaRef` on the `guideContainer` and, on each
>   bound field, the **`dataRef`** attribute (the CC field bind reference) with the full JSONPath from
>   the schema root — `dataRef="$.{parent}.{property}"` (e.g. `dataRef="$.owner.fullName"`).
>   **Use `dataRef`, NOT `fd:formDataRef`** — Core Components reads the bind reference from `dataRef`
>   and IGNORES `fd:formDataRef`, so writing `fd:formDataRef` leaves every field UNBOUND with a **blank
>   Bind Reference in the editor** (the exact symptom of "JSON schema → bind reference empty, but FDM
>   works"; FDM works only because it also uses `dataRef`). Same attribute for both schema and FDM
>   binding. **When the prompt does NOT specify a format, default to JSON Schema** (not XSD).
>   See [[binding-must-be-jsonpath]].
> - **Else (simple standalone collect-and-submit form, no binding)** → leave `schemaType="none"`
>   (the default below); a schema adds no value.
>
> If it is a judgement call, briefly tell the user what you chose and why. See [[schema-when-creating-forms]].

---

## Step 1 — Collect inputs from the user

Ask for these before writing anything:

| Input | Description | Example |
|---|---|---|
| `{project}` | The single project namespace used in component `sling:resourceType` paths, under `/conf/`, AND as the folder segment under `/content/forms/af/`, `/content/dam/formsanddocuments/`, and `/conf/forms/` (derived from `.aem-forms-config.yaml`) | `aem-demo-site` |
| `{formName}` | Kebab-case node name for the new form | `leave-request` |
| `{formTitle}` | Human-readable title shown in the UI | `Leave Request Form` |
| `{theme}` | Theme name suffix — the theme must exist under `/apps/fd/af/themes/{project}-{theme}` | `canvas-3-0` |
| `{template}` | Name of an **existing, reusable** template under `/conf/{project}/settings/wcm/templates/` — reuse one by default (see note below) | `blank-af-v2` |

> ⚠️ `{project}` is a **single** namespace — use it in every path root: `sling:resourceType`,
> `/conf/`, AND the folder segment under `/content/forms/af/`, `/content/dam/formsanddocuments/`, and
> `/conf/forms/`. There is NO separate hardcoded app folder. A form's DAM guide-asset path
> (`/content/dam/formsanddocuments/{project}/{formName}`) MUST match its form path
> (`/content/forms/af/{project}/{formName}`); a mismatch is a common cause of a form that fails silently.

> ⚠️ **`{template}` should be an EXISTING reusable template by default — do NOT fork a new one per
> form.** This project already has ~20 templates under `/conf/{project}/settings/wcm/templates/`;
> reference one that fits (same `af-page-v2` type, header/form-container/footer structure that fits,
> and a content policy allowing the components this form needs — a generic base like `blank-af-v2` or
> `basic-af` is broadly reusable). **Per-form theme/brand differences do NOT need a new template** —
> the theme is set separately via `themeRef` + the conf context (Artifacts 1–3), not the template.
> Create a NEW template (via `create-editable-template`) only when NONE fits — a distinct locked
> structure, a distinct allowed-components policy, or a governed layout for a new form family. In a
> pipeline delivery the architect already made this call — honor the plan's `template: reuse:"<path>"`.

> ⚠️ **Author ONLY the fields the form spec / wireframe actually lists — do not invent extras.**
> The authoritative field list is the wireframe / mockup / explicit field table for THIS form.
> Do NOT add a field just because a narrative requirements document mentions it as a *general
> example* (e.g. a UBR saying "a business-registration-number field could appear for business
> entities"). Examples in prose describe possible behaviour, not this form's field inventory.
> If a wireframe and the prose disagree on which fields exist, the **wireframe wins** — and call
> out the discrepancy to the user rather than silently adding the extra field. Adding fields the
> user didn't ask for is a defect, not thoroughness.

---

## Step 2 — Understand what you must create

A form that works requires exactly **5 artifacts**. Creating only the form page is not
enough — the others are what make it appear in the UI, apply the theme, load correctly,
and validate its fields.

| # | Artifact | File path to create |
|---|---|---|
| 1 | Form page | `ui.content/src/main/content/jcr_root/content/forms/af/{project}/{formName}/.content.xml` |
| 2 | DAM guide asset | `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/{project}/{formName}/.content.xml` |
| 3 | Per-form conf context | `ui.content/src/main/content/jcr_root/conf/forms/{project}/{formName}/.content.xml` |
| 4 | Filter entry | Add lines to `ui.content/src/main/content/META-INF/vault/filter.xml` |
| 5 | Validation clientlib | `ui.apps/src/main/content/jcr_root/apps/clientlibs/{formName}-clientlib/` (functions + rules) |

Each artifact is fully specified below. Generate all five.

---

## Artifact 1 — Form page

This is the main form node. It holds the form container, all panels, all fields, and the
page layout containers.

**File:** `.../jcr_root/content/forms/af/{project}/{formName}/.content.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:fd="http://www.adobe.com/aemfd/fd/1.0"
    jcr:primaryType="cq:Page">

    <!-- If a workflow's "Generate Document of Record" step will render THIS form (dorType below
         is not "none"), add guide="1" (String) to jcr:content here as well — AFtoDORStep gates on
         that marker independently of dorType/dorTemplateRef; see the DoR callout below. -->
    <jcr:content
        cq:deviceGroups="[/etc/mobile/groups/responsive]"
        cq:template="/conf/{project}/settings/wcm/templates/{template}"
        jcr:language="en"
        jcr:primaryType="cq:PageContent"
        jcr:title="{formTitle}"
        sling:configRef="/conf/forms/{project}/{formName}/"
        sling:resourceType="{project}/components/adaptiveForm/page">

        <guideContainer
            fd:version="2.1"
            jcr:primaryType="nt:unstructured"
            sling:resourceType="{project}/components/adaptiveForm/formcontainer"
            actionType="fd/dashboard/components/actions/aemworkflowsubmit"
            workflowModel="/var/workflow/models/assign-task-to-admin"
            dataXMLType="FOLDER_PAYLOAD"
            dataXMLPath="data.xml"
            storeAfSubmittedData="{Boolean}true"
            clientLibRef="{project}.forms.{formName}"
            dorType="none"
            fieldType="form"
            schemaType="none"
            textIsRich="true"
            thankYouMessage="Thank you for submitting the form."
            thankYouOption="page"
            themeRef="/apps/fd/af/themes/{project}-{theme}"
            title="{formTitle}">

            <!-- ADD PANEL AND FIELD NODES HERE — see sections below -->

        </guideContainer>

        <container1
            jcr:primaryType="nt:unstructured"
            sling:resourceType="{project}/components/container"
            layout="responsiveGrid">
            <!-- page header -->
        </container1>

        <container2
            jcr:primaryType="nt:unstructured"
            sling:resourceType="{project}/components/container"
            layout="responsiveGrid">
            <!-- page footer -->
        </container2>

        <cq:featuredimage
            jcr:primaryType="nt:unstructured"
            sling:resourceType="core/wcm/components/image/v3/image"
            altValueFromDAM="false"/>

    </jcr:content>
</jcr:root>
```

### Mandatory attribute rules for Artifact 1

**`sling:configRef` must end with a trailing `/`**
Write `/conf/forms/{project}/{formName}/` — not without the slash. Without it, Sling
does not resolve the conf context and the theme is never applied.

**`themeRef` must use the `/apps/fd/af/themes/` path**
Always `/apps/fd/af/themes/{project}-{theme}`. Never a `/content` path.

> **The theme is a THIN token override over the base (Option A).** AF has no native
> theme-extends-theme inheritance, so the referenced theme is just a `:root { --af-*: … }` brand
> token override — the **standard element styling comes from the shared base clientlib**
> (`{project}.forms.base` — the canonical styling, COPIED self-contained into the form clientlib's
> `css/base.css`, NOT wired in via embed/dependency), which owns the design-token defaults and the
> standard variable-based styling. Do NOT expect (or author) a full per-form
> stylesheet; `create-form-theme` supplies only the token overrides. Reuse an existing theme where
> one fits; a new full theme needs a recorded justification.

**`sling:resourceType` on every node uses `{project}`**
All component resource types are under `{project}/components/...`.

**`themeRef` here must exactly match artifact 2 and artifact 3**
All three files must reference the same theme string. Any mismatch causes silent
theme failure — the form loads but renders unstyled.

**Show the form title — add an explicit AF Title component (NOT the guideContainer showTitle band)**
The guideContainer v2 `showTitle` band does **NOT** emit a `.cmp-adaptiveform-container__title`
element in the served DOM, so a rule targeting that class styles nothing and the title renders as an
**empty band** (defect TC-031). Instead, add an explicit **AF Title (v2)** component as the **first
child** of the form (inside `guideContainer`, before the first panel):
```xml
<formTitle
    jcr:primaryType="nt:unstructured"
    jcr:title="{formTitle}"
    sling:resourceType="{project}/components/adaptiveForm/title"
    aria-label="{formTitle}"
    fd:htmlelementType="h1"
    value="{formTitle}"
    css="form-title"
    visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
    </cq:responsive>
</formTitle>
```
This emits a real `<div class="cmp-title"><h1 class="cmp-title__text">…</h1></div>`. Set the
guideContainer's `showTitle="{Boolean}false"` (keep `title="{formTitle}"` for model / DoR metadata).
Style the **actual served classes** — `.cmp-title` / `h1.cmp-title__text` — in the **form clientlib
CSS**, NOT `.cmp-adaptiveform-container__title`.

> 🛑 **The `css="form-title"` value above is authoring metadata only — it is NOT emitted as a DOM
> class**, so a `.form-title { … }` rule matches nothing (it appears only as `"css"` in
> `guideContainer.model.json`). Use `h1.cmp-title__text` for the form title and `h2.cmp-title__text`
> for section headings (the `fd:htmlelementType` element is the discriminator). Full served-selector
> table + the one-line `curl | grep` check: `create-form-theme` → "NEVER style a form via a `css=`
> class hook".

**Verify the title renders by pixels.**

**Auto-wire the prefill service and submit action on the guideContainer (don't make the author pick them)**
Just like the FDM is auto-selected via `schemaType`/`schemaRef`, the prefill and submit must be
pre-bound so the form works on deploy without manual editor selection:

> 🔁 **STANDING PROJECT RULE — every new form is wired to the shared submission workflow.**
> On submit, every Adaptive Form in this project must **assign a Medium-priority task to the `admin`
> user** via the single, shared, reusable workflow model **`assign-task-to-admin`**
> (`/var/workflow/models/assign-task-to-admin`). This is the **DEFAULT submit action** and is wired
> automatically — the author never has to pick it. Wire it on the `guideContainer` with the modern
> "Invoke an AEM Workflow" action (exactly as shipped in the Artifact 1 template above):
> ```
> actionType="fd/dashboard/components/actions/aemworkflowsubmit"
> workflowModel="/var/workflow/models/assign-task-to-admin"
> dataXMLType="FOLDER_PAYLOAD"
> dataXMLPath="data.xml"
> storeAfSubmittedData="{Boolean}true"
> ```
> - **The workflow model is created ONCE and REUSED by every form** — point each form's
>   `workflowModel` at the SAME `/var/workflow/models/assign-task-to-admin`. **Never** scaffold a
>   per-form workflow model, and never create a second copy. If the model is missing, it is created
>   by the `create-workflow` skill (see its "canonical shared model" section) — but on this project
>   it already exists in source control at
>   `ui.content/.../jcr_root/conf/global/settings/workflow/models/assign-task-to-admin`.
> - If the form has a file-upload/attachment field, also set
>   `attachmentsFolderPath="attachments/"` + `attachmentsType="FOLDER_PAYLOAD"` (else Submit 502s).
> - `dataXMLPath` must stay a payload-relative path (`data.xml`) — never empty/absolute/URL, or Submit
>   fails 500 with "No output path relative to payload".

- **Submit (override case only):** the workflow above is the default. Wire a DIFFERENT terminal
  submit action **only when the ORIGINAL prompt / PLAN `submit` intent explicitly requires one**
  (a PDF/DoR, a REST write-back, or an email). In that case set
  `actionType="{project}/fd/af/submitactions/{ActionNode}"` (the submit-action NODE PATH, relative to
  /apps — NOT the generic `fd/af/components/guidesubmittype/submitservice` marker) +
  `submitService="{the custom action's getServiceName() / submitService label}"` (must match the JCR
  submit-action node exactly — see `create-submit-action`). **Why the node path:** the editor's
  Submission-tab dropdown maps its selected value by matching `actionType` to a listed node path; the
  generic marker binds at runtime but leaves the editor dropdown showing blank "Select" (confirmed
  against the exact value the editor writes on manual selection). When a terminal submit action is
  used, still assign the admin task by adding a Workflow Launcher on the submitted-data path (see
  `create-workflow` → OSGi Workflow Launcher) so the standing rule above still holds.
  - Map the explicit intent to the action. **The admin-task workflow stays the submit action in
    every row where the extra behaviour is client-side** (PDF is generated by the `forms.generate-pdf`
    clientlib on submit, independent of the server-side action), so those forms get **BOTH** the admin
    task AND the extra behaviour. A DIFFERENT server-side action is used only for a genuine server-side
    terminal write (REST/email/FDM), and there a Workflow Launcher preserves the admin task:
    | PLAN `submit` intent (from the prompt) | Server-side submit action (`actionType`) | Also wire |
    |---|---|---|
    | (none specified — **DEFAULT**) | `fd/dashboard/components/actions/aemworkflowsubmit` + `workflowModel=/var/workflow/models/assign-task-to-admin` | — |
    | `dor_pdf` / "generate a PDF" / "PDF of the submission" / Document of Record | **KEEP the workflow action** (assign-task-to-admin) — PDF is client-side | add `{project}.forms.generate-pdf` to the form-specific clientlib's `dependencies` (it has no `@name` functions, safe; NOT to `clientLibRef`, NOT via `embed` of `forms.base`) + `thankYouOption="message"` so the SPA stays alive for the client-side PDF. Result: **admin task + PDF**. |
    | `rest` / REST endpoint write-back | the custom REST submit action's name — `{project}/fd/af/submitactions/{restActionNode}` (+ `submitService=…`) | a **Workflow Launcher** (see `create-workflow`) on the submitted-data path so the admin task is still assigned |
    | `email` | the email submit action's name — `{project}/fd/af/submitactions/{emailActionNode}` (+ `submitService=…`) | Workflow Launcher, as above |
    | `workflow` / approval (a DIFFERENT process) | "Invoke an AEM Workflow" wiring (see `create-workflow`) — the workflow submit node path | if that process must ALSO assign the admin task, add the Assign Task step to it, or a launcher |
- **Prefill:** `prefillService="{the DataProvider SERVICE_NAME}"` on the `guideContainer` (the
  identifier the `create-prefill-service` DataProvider registers, e.g. `myFormPrefillService`).
  Omitting this is why a prefill service deploys but the form never calls it / the author has to
  select it manually.

---

### Panels

Group related fields inside `panelcontainer` nodes placed inside `guideContainer`.
The XML node name and the `name` attribute should be the same string.

```xml
<PersonalDetails
    jcr:primaryType="nt:unstructured"
    jcr:title="Personal Details"
    sling:resourceType="{project}/components/adaptiveForm/panelcontainer"
    enabled="{Boolean}true"
    fieldType="panel"
    hideTitle="true"
    layout="responsiveGrid"
    name="PersonalDetails"
    readOnly="{Boolean}false"
    visible="{Boolean}true"
    wrapData="{Boolean}false">

    <!-- field nodes go here -->

</PersonalDetails>
```

---

### Fields

Place field nodes inside a panel. Every field must have all of the following attributes —
missing any one causes the field to not render or not validate correctly:

- `jcr:primaryType="nt:unstructured"`
- `jcr:title` — the visible label
- `sling:resourceType` — `{project}/components/adaptiveForm/{fieldComponent}`
- `aria-label` — **mandatory on EVERY field** (screen-reader label; this is a standing project
  rule in AGENTS.md). Set it to the same human string as `jcr:title`. Place it immediately
  after `sling:resourceType`, matching the existing forms in this repo.
- `fieldType` — the logical type string (see table below)
- `name` — unique identifier within the form, used in rules as `$form.{name}`
- `enabled="{Boolean}true"`
- `readOnly="{Boolean}false"`
- `visible="{Boolean}true"`
- `<cq:responsive>` child with a `<default>` node (controls grid layout)

> ♿ **Accessibility (Adobe-recommended — do not skip).** A form that fails accessibility is
> not "done". On every field:
> - **`aria-label`** is mandatory (above). Without it a screen reader announces an unlabelled
>   control. Grep the finished form `.content.xml` — every field node must carry an `aria-label`.
> - Prefer a real visible `jcr:title` over `hideTitle="true"`; if a label must be hidden
>   visually, `aria-label` is what keeps it accessible.
> - For fields needing extra context, add `tooltip="..."` (short help) rather than relying on
>   placeholder text, which assistive tech does not treat as a label.
> - Required fields must pair `required="true"` with `mandatoryMessage` so the error is announced.
> - The error summary / `aria-live` announcement is handled by the validation clientlib
>   (Artifact 5) — keep it.

Required fields must additionally have:
- `required="true"`
- `mandatoryMessage="..."` — the validation message shown when left empty

**Field type reference:**

| Field | `fieldComponent` (append to `{project}/components/adaptiveForm/`) | `fieldType` value | Notes |
|---|---|---|---|
| Single-line text | `textinput` | `text-input` | |
| Multi-line text | `textinput` | `text-input` | add `multiLine="true"` |
| Number | `numberinput` | `number-input` | add `minimum` / `maximum` for ranges; use this for any numeric input — do NOT collect numbers as `text-input` |
| Email | `emailinput` | `email` | |
| Phone number | `telephoneinput` | `text-input` | add `pattern` + `validatePatternMessage` |
| Date picker | `datepicker` | `date-input` | **`displayFormat` and `editFormat` MUST use the SAME format token** (e.g. both `date\|DD/MM/YYYY`) + a matching `placeholderText`. A mismatch (e.g. `editFormat="date\|yyyy-MM-dd"` with `displayFormat="date\|DD/MM/YYYY"`) breaks the picker — the user cannot pick/commit a date. See defect note below. |
| Dropdown | `dropdown` | `drop-down` | add `type="string"`, `enum`, `enumNames` |
| Radio group | `radiobutton` | `radio-group` | single-select; add `type="string"`, `enum`, `enumNames`, `orientation` |
| Checkbox group | `checkboxgroup` | `checkbox-group` | multi-select; add `type="string[]"`, `enum`, `enumNames`, `orientation` |
| File attachment | `fileinput` | `file-input` | add `accept="[application/pdf,image/*]"` + `maxFileSize="2MB"`; for document uploads |
| Display / static text | `text` | `plain-text` | non-input note/instruction; add `value="..."`, `hideTitle="true"` |
| CAPTCHA | `recaptcha` / `turnstile` / `hcaptcha` | `captcha` | bot protection on public forms — see "CAPTCHA" below |
| Terms & conditions | (panel `fd:tnc="{Boolean}true"`) | `panel` | consent block — see "Terms & Conditions" below |
| Submit button | `actions/submit` | `button` | add `buttonType="submit"` — use the `actions/submit` component, NOT the generic `button` |
| Reset button | `actions/reset` | `button` | add `buttonType="reset"` — use the `actions/reset` component, NOT the generic `button`; needs NO `fd:click` (native reset), keep an empty `<fd:events/>` |

> Verify the proxy exists under `ui.apps/.../apps/{project}/components/adaptiveForm/` before
> using a type. This repo proxies `numberinput`, `radiobutton`, `fileinput`, `text`,
> `recaptcha`/`turnstile`/`hcaptcha`, `termsandconditions`, and the layout containers below;
> if a needed proxy is missing, run `create-form-component` rather than inventing a path.

**Radio group example** (single-select — uses `enum`/`enumNames`, NEVER `<items>`):
```xml
<Gender
    jcr:primaryType="nt:unstructured"
    jcr:title="Gender"
    sling:resourceType="{project}/components/adaptiveForm/radiobutton"
    aria-label="Gender"
    enabled="{Boolean}true"
    enum="[male,female,other]"
    enumNames="[Male,Female,Other]"
    fieldType="radio-group"
    name="Gender"
    orientation="horizontal"
    readOnly="{Boolean}false"
    type="string"
    visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" behavior="newline" offset="0" width="12"/>
    </cq:responsive>
</Gender>
```

### Layout containers (panels with a layout)

Adobe recommends grouping a long form into a **wizard / tabs / accordion** rather than one
long scroll. These are real proxy components in this repo — set the container's
`sling:resourceType` to the layout component (each still uses `fieldType="panel"` and holds
`panelcontainer` children as its steps/tabs):

| Layout | `fieldComponent` | When to use |
|---|---|---|
| Vertical scroll (default) | `panelcontainer` (`layout="responsiveGrid"`) | short, single-screen forms |
| Wizard (step-by-step, Next/Back) | `wizard` | long multi-step forms; one panel per step |
| Tabs on top | `horizontaltabs` | a few parallel sections |
| Vertical tabs | `verticaltabs` | many parallel sections |
| Accordion | `accordion` | collapsible sections, mobile-friendly |

Each child step/tab is a normal `panelcontainer`. Only the **outer** container's
`sling:resourceType` changes to the layout type.

**Enum syntax** (multi-value attribute, bracket format):
```
enum="[val1,val2,val3]"
enumNames="[Label One,Label Two,Label Three]"
```

> ⚠️ **CRITICAL — dropdown / checkbox / radio options MUST be `enum` + `enumNames` attributes on the field. NEVER use `<items>` / `<item0 text="" value=""/>` child nodes.**
> The project's choice components proxy Core Components (`core/fd/components/form/dropdown/...`, `.../checkboxgroup/...`, `.../radiobutton/...`). Core Components read options **only** from the `enum` (values/codes) and `enumNames` (display labels) attributes. The legacy Foundation `<items>/item0` child-node structure is **silently ignored** — the build succeeds, the form renders, but the control shows **no options** (empty dropdown). This is the single most common Core-Components authoring mistake; do not reproduce it from older forms.
>
> - `enum` holds the stored value/code; `enumNames` holds the user-visible label (same length, same order).
> - Keep `enum` codes **stable** when a rule references them (e.g. a rule `field == "BloodTest"` requires the enum to contain `BloodTest`, not the label `Blood Test`).
>
> ```xml
> ❌ WRONG — renders an empty dropdown under Core Components:
> <items jcr:primaryType="nt:unstructured">
>     <item0 jcr:primaryType="nt:unstructured" text="Blood Test" value="BloodTest"/>
>     <item1 jcr:primaryType="nt:unstructured" text="Urine Test" value="UrineTest"/>
> </items>
>
> ✅ RIGHT — attributes on the field element:
> enum="[BloodTest,UrineTest]"
> enumNames="[Blood Test,Urine Test]"
> ```

**Single-option checkbox** (e.g. a declaration or consent):
```
enum="true"
enumNames="I agree to the terms and conditions"
```

**cq:responsive child** (required on every field):
```xml
<cq:responsive jcr:primaryType="nt:unstructured">
    <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
</cq:responsive>
```
Add `behavior="newline"` to `<default>` to force the field onto its own row.

**Complete text field example:**

```xml
<FullName
    jcr:primaryType="nt:unstructured"
    jcr:title="Full name"
    sling:resourceType="{project}/components/adaptiveForm/textinput"
    aria-label="Full name"
    autocomplete="off"
    enabled="{Boolean}true"
    fieldType="text-input"
    mandatoryMessage="Please enter your full name."
    name="FullName"
    readOnly="{Boolean}false"
    required="true"
    visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
    </cq:responsive>
</FullName>
```

**Complete dropdown example:**

```xml
<Department
    jcr:primaryType="nt:unstructured"
    jcr:title="Department"
    sling:resourceType="{project}/components/adaptiveForm/dropdown"
    aria-label="Department"
    enabled="{Boolean}true"
    enum="[HR,Engineering,Finance,Operations]"
    enumNames="[HR,Engineering,Finance,Operations]"
    fieldType="drop-down"
    mandatoryMessage="Please select a department."
    name="Department"
    readOnly="{Boolean}false"
    required="true"
    type="string"
    visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
    </cq:responsive>
</Department>
```

---

### Submit button (mandatory on every form)

Every form must have exactly one submit button. Place it as the last node inside the
panel or directly inside `guideContainer`. The `fd:rules` and `fd:events` nodes are
required — without them the button does not trigger submission.

> 🛑 **Submit (and any PDF / Document-of-Record generation) MUST be gated on validation success.**
> Use the native Core Components submit (`submitForm()` → the action selected on the container), which
> validates the whole form BEFORE calling the submit action. An invalid form must BLOCK submission,
> show inline error messages, scroll/focus to the first invalid field, and produce **NO PDF**. Never
> wire the button to a raw handler that POSTs to a GeneratePDF servlet / submit endpoint
> unconditionally — that generates a PDF for an invalid form. If a PDF is produced client-side (the
> shared `Custom-Submit-GeneratePDF` clientlib), it must only run after the native submit's validation
> passes (see `create-submit-action`). This is a user story to test: "submission only succeeds when
> validation passes" — empty/invalid form shows errors and makes no PDF; valid form submits AND
> generates the PDF.

> 🛑 **CRITICAL — a button's `fd:click` is NOT a raw expression. It is an escaped-JSON AST.**
> The Rule Editor runs `JSON.parse()` on **every** property of **every** `<fd:rules>` node
> on the form when it loads. If `fd:click` (or any `fd:*` rule property) holds a bare string
> like `fd:click="submitForm()"` instead of a JSON array, `JSON.parse` throws
> `Uncaught SyntaxError: Unexpected token 's', "submitForm()" is not valid JSON`, the editor
> **aborts the entire rule-tree traversal**, and **NO rules render for ANY field** — the editor
> shows "There are no rules on this object" everywhere even though the field rules are fine.
> This is a whole-form-breaking defect, not button-local. (This is the same "wrong-for-Core-Components
> storage" failure class as `<items>` vs `enum`, and `fd:visibility` vs `fd:visible`.)
>
> Therefore:
> - On `<fd:rules>`: `fd:click` MUST be the full escaped-JSON `EVENT_SCRIPTS`→`SUBMIT_FORM` AST
>   (shown below — copy it verbatim; for a non-submit button use `create-form-rules`).
> - On `<fd:events>`: the property is **`click`** (NOT `fd:click`), value **`[submitForm()]`** (bracketed).
> - **Never** wire a button to a function that does not exist in the form's clientlib — a missing
>   function (e.g. an invented `saveDraft()`) renders the button dead and, if placed raw on `fd:click`,
>   also crashes the editor.
> - If you cannot produce a valid AST for a custom button action, leave `<fd:rules>` valid/empty
>   (`<fd:rules jcr:primaryType="nt:unstructured" validationStatus="valid"/>`), wire the runtime via
>   `<fd:events ... click="[yourFn()]"/>`, and author the editor-visible rule by round-tripping through
>   the Rule Editor — never hand-write a guessed AST.

```xml
<submit
    jcr:primaryType="nt:unstructured"
    jcr:title="Submit"
    sling:resourceType="{project}/components/adaptiveForm/actions/submit"
    aria-label="Submit"
    buttonType="submit"
    enabled="{Boolean}true"
    fieldType="button"
    name="submit"
    visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
    </cq:responsive>
    <fd:rules jcr:primaryType="nt:unstructured"
        fd:click="[{&quot;nodeName&quot;:&quot;ROOT&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;STATEMENT&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;EVENT_SCRIPTS&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;EVENT_CONDITION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;EVENT_AND_COMPARISON&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;COMPONENT&quot;\,&quot;value&quot;:{&quot;id&quot;:&quot;$form.submit&quot;\,&quot;type&quot;:&quot;BUTTON&quot;\,&quot;name&quot;:&quot;submit&quot;}}\,{&quot;nodeName&quot;:&quot;EVENT_AND_COMPARISON_OPERATOR&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;is clicked&quot;\,&quot;value&quot;:null}}\,{&quot;nodeName&quot;:&quot;PRIMITIVE_EXPRESSION&quot;\,&quot;choice&quot;:null}]}\,&quot;nested&quot;:false}\,{&quot;nodeName&quot;:&quot;Then&quot;\,&quot;value&quot;:null}\,{&quot;nodeName&quot;:&quot;BLOCK_STATEMENTS&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;BLOCK_STATEMENT&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;SUBMIT_FORM&quot;\,&quot;items&quot;:[]}}]}]}}]\,&quot;isValid&quot;:true\,&quot;enabled&quot;:true\,&quot;version&quot;:1\,&quot;script&quot;:[&quot;submitForm()&quot;]\,&quot;eventName&quot;:&quot;Click&quot;\,&quot;ruleType&quot;:&quot;&quot;\,&quot;description&quot;:&quot;&quot;}]"
        validationStatus="valid"/>
    <fd:events jcr:primaryType="nt:unstructured" click="[submitForm()]"/>
</submit>
```

> 🛑 **Footer buttons MUST use the action components — NOT the generic `button` (defect fix).**
> The **Submit** button's `sling:resourceType` must be `{project}/components/adaptiveForm/actions/submit`
> and the **Reset** button's must be `{project}/components/adaptiveForm/actions/reset`. The generic
> `{project}/components/adaptiveForm/button` component **renders a button but does NOT submit or reset
> the form** — clicking it does nothing. This exact mistake shipped a Course form whose Submit/Reset
> were dead while an IT-support form using `actions/submit`/`actions/reset` worked. The **Reset** button
> needs **no `fd:click`** (native reset clears fields + restores defaults) — keep an empty
> `<fd:events jcr:primaryType="nt:unstructured"/>`; never author a `click="[reset()]"` (there is no
> `reset()` function — it throws "Unknown function"). See [[button-click-rules-need-fd-click-ast]].

> 🛑 **Date picker `displayFormat` and `editFormat` MUST match (defect fix).** Both must use the SAME
> format token — e.g. `displayFormat="date|DD/MM/YYYY"` **and** `editFormat="date|DD/MM/YYYY"` (plus
> `placeholderText="DD / MM / YYYY"`). A mismatch such as `editFormat="date|yyyy-MM-dd"` against a
> `DD/MM/YYYY` display **breaks the picker** — the calendar won't commit and the user cannot pick a
> date. This shipped in a Course form (ISO `editFormat`) while the working IT-support form used
> `DD/MM/YYYY` for both. Keep the two formats identical for every `datepicker`.
> The date field must stay a **NATIVE, selectable `<input type="date">`** — the base clientlib must NOT
> set `appearance:none` on the date widget (that strips the picker so it looks like plain text and won't
> open). See [[datepicker-icon-pinned-in-field]]. **Validation feedback:** on failure the error message
> renders in red and the field border turns red (base clientlib styles `data-cmp-valid="false"` +
> `aria-invalid` with `--af-error`). See [[validation-failure-styling-red]].

---

### Icons & imagery from a reference — author as AF IMAGE COMPONENTS backed by DAM assets (never CSS/inline)

When the reference screenshot/design contains icons, logos, or illustrations (a header/brand
logo, per-section-header icons, in-tile icons on upload/choice cards), each one must be stored
as a **real DAM asset** and rendered by an **authored Adaptive Form Image component**. Do **NOT**
draw images via CSS `background-image` / `::before` pseudo-elements / inline SVG / base64 — **not
even** decorative section-heading micro-icons or tile icons. Authoring an Image component emits a
real `<img>` that renders reliably AND lets authors swap the image in the editor; a CSS-painted
glyph is unauthorable, unversioned, and easy to drop. This supersedes any earlier guidance that
allowed the "CSS background-image URL" approach for reference/decorative icons.

- **Store every reference icon as a DAM asset** under the form's icon folder, authored in
  `ui.content` with a `filter.xml` entry so it deploys:
  `ui.content/.../jcr_root/content/dam/{project}/{formName}/icons/{icon}.svg` (or `.png`).
  Reuse an existing shared asset if one already covers the icon.
- **Render EVERY image via an authored Image component** — the Adaptive Form **Image** component
  `{project}/components/adaptiveForm/image` (superType `core/fd/components/form/image/v1/image`),
  with `fileReference` pointing at the DAM asset path and `altText` / `aria-label` set (decorative
  → empty alt / `aria-hidden`; meaningful → real alt text). This is the mechanism for **all**
  imagery — brand/header logos, section illustrations, AND decorative section-header glyphs and
  in-tile card icons alike. Where a section/heading needs a tile icon, author the Image component
  as the **first child** of that section/heading.
- **CSS is for sizing/placement ONLY, never for supplying the image.** Style the Image component's
  css class (width/height, position, tile framing) in the theme/clientlib CSS — but the image
  itself always comes from the authored component's `fileReference`, never a `background-image`
  URL, `::before` glyph, inline SVG, or base64.
- **Every icon the reference shows must be a stored DAM asset AND an authored Image component** —
  cross-check against the icon inventory captured by `discover-form-requirements` /
  `design-form-components`. A reference icon that isn't a stored asset rendered by an Image
  component is an incomplete form.
- **Reproduce ALL visuals, not only fields and icons.** Every visual element in the reference must
  appear in the built form — including **section separator / divider rules** (the thin horizontal
  line under each numbered section and under the title/subtitle header) and background bands. Draw
  separators in the **form clientlib CSS** (Artifact 5, served fresh) as a `border-bottom`
  (e.g. `1px solid #d9d9d9`) on the real served section-panel classes + a rule under the header —
  verify the actual served DOM classes, never invent them. Confirm every visual by rendered pixels
  after redeploy, not by CSS presence.

### Conditional rules (show/hide, calculate, value-commit)

Only add rules when the user explicitly asks for conditional logic. Use the
`create-form-rules` skill to generate the `fd:rules` / `fd:events` node pair — do not
write rule JSON by hand as malformed JSON fails silently at runtime.

---

### Adobe-recommended optional features (add when the requirement calls for them)

These are first-class Core Components capabilities. Don't add them to every form, but
**evaluate each against the requirement** and tell the user when one applies.

**CAPTCHA / bot protection (any PUBLIC, unauthenticated form).** Add a captcha field as the
last field before submit. The proxies in this repo are `recaptcha` (Google reCAPTCHA
Enterprise), `turnstile` (Cloudflare), and `hcaptcha`. The field needs site keys configured
in a cloud service (`/conf/{project}/settings/cloudconfigs/...`) — without the cloud config the
field renders but never validates. Field shape:
```xml
<captcha
    jcr:primaryType="nt:unstructured"
    jcr:title="CAPTCHA"
    sling:resourceType="{project}/components/adaptiveForm/recaptcha"
    aria-label="CAPTCHA"
    fieldType="captcha"
    name="captcha"
    required="{Boolean}true"
    visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
    </cq:responsive>
</captcha>
```
Tell the user which provider's cloud config must exist; do not invent site keys.

**Terms & Conditions / consent (forms with a legal agreement).** Use a panel with
`fd:tnc="{Boolean}true"` containing a `text` (`fieldType="plain-text"`) for the legal copy and
a required `checkbox`/`checkbox-group` for the agreement — model it on this repo's
`termsandconditions` proxy. The agreement checkbox must be `required="true"` so submit is blocked
until consent is given.

**Document of Record (DoR) — PDF record of the submission.** The form-page template defaults to
`dorType="none"` on `guideContainer`. For regulated / government / financial forms where the user
needs a PDF copy of what they submitted, set `dorType="generate"` on `guideContainer` (the form
renders as its own DoR — the Core Components default when there's no separate print/XDP template).
If instead a genuine print/XDP template must be used, set `dorType="select"` + `dorTemplateRef`
(pointing at a real `dam:Asset`, never a `cq:Page`) on the **DAM guide asset's own
`jcr:content/metadata` node** (`/content/dam/formsanddocuments/{appFolder}/{formName}/jcr:content/metadata`,
the "Form Properties" data) — **NOT** on `guideContainer`, whose own dialog has no DoR-template
field at all (live-verified: a direct `dorTemplateRef` write there is inert). `guideContainer`'s own
`dorType` may still be left as `"generate"`/`"none"` regardless of what Form Properties says; only
`dorTemplateRef` is exclusively a Form-Properties/DAM-metadata concern. If the requirement mentions
"PDF of the submission", "record copy", "downloadable receipt", or compliance retention, raise DoR
with the user and wire it via `create-workflow` (its "Generate Document of Record" step) or
`create-submit-action` (the shared Generate-PDF action). Leave `dorType="none"` only when no record
copy is needed.

> ⚠️ **`dorType` alone is NOT enough if a workflow's "Generate Document of Record" step will render
> this form.** `AFtoDORStep` (the OOTB workflow DoR step) also requires the form page's own
> `jcr:content` to carry the String marker property **`guide="1"`** — decompiled evidence
> (`AFtoDORStep.java:125`) shows it throws `"Not a valid Adaptive Form"` when that property is
> absent, **independent of `dorType`/`dorTemplateRef`**; a Core Components AF page does not carry
> this marker by default, so add it explicitly whenever the form's DoR will be generated from a
> workflow. This is the SAME marker the editor's own DoR-configured advisory checks. Always set
> **all three** DoR places together — this form page's `guide="1"`, `guideContainer`'s `dorType`
> (plus, if a real template is used, `dorType="select"`+`dorTemplateRef` on the **DAM guide asset's
> `metadata` node**, not `guideContainer`), and (if applicable) the workflow's Generate DoR step
> pointed at this form — see `create-workflow`'s "DoR prerequisite" for the full three-place
> requirement and troubleshooting.

**Adaptive Form Fragments (reuse repeated content).** When the same block of fields recurs across
forms (address, applicant identity, bank details), Adobe recommends authoring it once as an
**Adaptive Form Fragment** and referencing it, instead of copy-pasting panels. Fragments live
under `/content/forms/af` like a form and are inserted via the Fragment component. If the
requirement has obviously reusable sections, suggest extracting them as a fragment rather than
duplicating the XML.

---

## Artifact 2 — DAM guide asset

This file registers the form in Forms & Documents and carries the theme reference.
Without it the form page exists in the JCR but is invisible in the Forms & Documents UI.

**File:** `.../jcr_root/content/dam/formsanddocuments/{project}/{formName}/.content.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:dam="http://www.day.com/dam/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    xmlns:xmp="http://ns.adobe.com/xap/1.0/"
    xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:fd="http://www.adobe.com/aemfd/fd/1.0"
    jcr:primaryType="dam:Asset">

    <jcr:content
        cq:conf="\0"
        jcr:primaryType="dam:AssetContent"
        sling:resourceType="fd/fm/af/render"
        guide="1"
        type="guide">

        <metadata
            fd:version="2.1"
            jcr:language="en"
            jcr:primaryType="nt:unstructured"
            xmp:CreatorTool="AEM Forms AF Wizard"
            allowedRenderFormat="HTML"
            author="admin"
            availableInMobileApp="{Boolean}false"
            dorTemplateChanged="Boolean"
            dorType="none"
            formmodel="none"
            themeRef="/apps/fd/af/themes/{project}-{theme}"
            title="{formTitle}"/>

    </jcr:content>
</jcr:root>
```

### Mandatory attribute rules for Artifact 2

**`type="guide"` and `guide="1"` are both required on `jcr:content`**
These two attributes together are the markers AEM uses to list the asset in Forms &
Documents. Removing either one makes the form invisible in the listing.

**`sling:resourceType="fd/fm/af/render"` is a fixed platform value**
Do not replace it with a project-specific resource type.

**`themeRef` must exactly match the value in Artifact 1's `guideContainer`**
Copy the value — do not retype it.

**`title` must match `jcr:title` on the form page (Artifact 1)**
A mismatch causes the form to show two different names in different parts of the UI.

---

## Artifact 3 — Per-form conf context

This file is what `sling:configRef` in Artifact 1 points to. It instructs Sling which
theme to apply and injects `theme.css` and `theme.js` into the page head. Without it the
form either fails to open or opens completely unstyled.

**File:** `.../jcr_root/conf/forms/{project}/{formName}/.content.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root
    xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="sling:Folder"
    sling:resourceType="sling:Folder">

    <sling:configs jcr:primaryType="nt:unstructured">

        <com.adobe.aem.wcm.site.manager.config.SiteConfig
            jcr:primaryType="nt:unstructured"
            themePackageName="themePackageName"
            themeArtifact="{project}-{theme}"
            siteTemplatePath="/apps/fd/af/themes/{project}-{theme}"/>

        <com.adobe.cq.wcm.core.components.config.HtmlPageItemsConfig
            jcr:primaryType="cq:Page">

            <jcr:content
                jcr:primaryType="cq:PageContent"
                prefixPath="/content/forms/af/{project}/{formName}.theme/_default">

                <items jcr:primaryType="nt:unstructured">

                    <css
                        jcr:primaryType="nt:unstructured"
                        element="link"
                        location="header">
                        <attributes jcr:primaryType="nt:unstructured">
                            <as   jcr:primaryType="nt:unstructured" name="as"   value="style"/>
                            <href jcr:primaryType="nt:unstructured" name="href" value="/theme.css"/>
                            <rel  jcr:primaryType="nt:unstructured" name="rel"  value="preload stylesheet"/>
                            <type jcr:primaryType="nt:unstructured" name="type" value="text/css"/>
                        </attributes>
                    </css>

                    <javascript
                        jcr:primaryType="nt:unstructured"
                        element="script"
                        location="header">
                        <attributes jcr:primaryType="nt:unstructured">
                            <src   jcr:primaryType="nt:unstructured" name="src"   value="/theme.js"/>
                            <async jcr:primaryType="nt:unstructured" name="async" value="true"/>
                            <type  jcr:primaryType="nt:unstructured" name="type"  value="text/javascript"/>
                        </attributes>
                    </javascript>

                </items>
            </jcr:content>
        </com.adobe.cq.wcm.core.components.config.HtmlPageItemsConfig>

    </sling:configs>
</jcr:root>
```

### Mandatory attribute rules for Artifact 3

**Root node must have both `jcr:primaryType="sling:Folder"` and `sling:resourceType="sling:Folder"`**
Both attributes are required on the root. If either is missing, Sling config resolution
skips this node entirely and the conf context has no effect.

**`themeArtifact` must equal `{project}-{theme}`**
This is the string `SiteConfig` uses to resolve the theme package. It must be the same
suffix as `themeRef` in artifacts 1 and 2.

**`siteTemplatePath` must be the full `/apps/fd/af/themes/{project}-{theme}` path**

**`prefixPath` uses the `{project}` folder segment**
The value is `/content/forms/af/{project}/{formName}.theme/_default`. This is the
virtual path AEM resolves `theme.css` and `theme.js` relative to. It must match the form's
`/content/forms/af/{project}/{formName}` path, or the CSS and JS return 404 at runtime.

---

## Artifact 4 — Filter entry

The per-form conf context must have its own filter entry or it is excluded from the
content package and never deployed to the server.

Open `ui.content/src/main/content/META-INF/vault/filter.xml` and add:

```xml
<filter root="/conf/forms/{project}/{formName}" mode="update"/>
```

Also verify that filter.xml already covers these roots (they are usually present but
confirm before assuming):
- `/content/forms/af/{project}` — covers the form page
- `/content/dam/formsanddocuments/{project}` — covers the DAM asset

If either is missing, add it.

The clientlib (Artifact 5) lives under `ui.apps`. Ensure **its** filter
(`ui.apps/src/main/content/META-INF/vault/filter.xml`) covers `/apps/clientlibs`:
```xml
<filter root="/apps/clientlibs"/>
```
Add it if absent (the default `ui.apps` filter covers `/apps/{project}/clientlibs`, NOT
`/apps/clientlibs`).

> **⚠ RE-AUTHORING WARNING — `mode="update"` leaves orphan nodes (duplicate sections).**
> The form filter roots (`/content/forms/af/{project}`, `/content/{project}`) use
> `mode="update"`, which imports/updates nodes in the package but **never deletes** instance nodes that
> were **removed, renamed, or moved** in source. So when you **re-author an existing, already-deployed
> form** — e.g. wrap flat panels inside `accordion` nodes (`guideContainer/panelX` →
> `guideContainer/accordionX/panelX`), rename a node, or restructure the hierarchy — the OLD nodes
> survive on the instance and the form renders the section **twice** (a flat copy above + the new copy
> in the accordion). A clean `mvn` redeploy does **not** fix this (update mode won't purge them).
> When you make such structural moves, flag it in your IMPL summary so **Forgemaster purges the orphan
> instance nodes** as part of its deploy-integrity sweep (Forgemaster step 3b — Sling POST
> `:operation=delete` with a CSRF token). Do **not** solve it by re-adding the old flat nodes to
> source; the accordion-nested structure is the correct source. Prefer moving a node's *contents*
> rather than leaving a same-named ghost, and keep node **names unique** across a restructure so the
> orphan is easy to detect (live child set vs source child set under `guideContainer`).

---

## Artifact 5 — Validation clientlib (derived from the requirement doc)

Every form ships with a per-form client library that holds its **field-level validation
functions** and the **Rule-Editor-visible validation rules** that call them.

> 🩺 **If Rule-Editor rules show "Broken" (customfunctions endpoint returns `{"customFunction":[]}`),
> suspect the SCHEMA/model FIRST, not the clientlib.** For a schema-backed form the endpoint builds
> the form model, which needs the JSON schema to import; a schema that fails AEM's importer (valid
> JSON can still fail — e.g. an unsupported `const`) makes the endpoint return empty and marks every
> custom-function rule "Broken" (runtime still works). Check `error.log` for
> `GuideModelImporterImpl Unable to parse JSON Schema` and fix the schema (see `generate-schema`
> Hard rule 7). Rewriting the clientlib will NOT fix this. VERIFIED on sports-event (2026-07-06).
>
> **Reuse-first split (canonical base + SELF-CONTAINED form clientlib).** The shared **base**
> forms clientlib (category `{project}.forms.base`, under `/apps/{project}/clientlibs`) is the
> CANONICAL reference for generic validators + standard styling — the one place to read/update
> them. BUT a per-form clientlib must **NOT** `embed`/`depend on` `{project}.forms.base`: the base
> re-defines the same `@name` functions, so wiring it in makes the built clientlib define
> `validateName`/etc. **twice**, and the customfunctions endpoint returns
> `{"customFunction":[]}` → every rule "Broken" (VERIFIED). Instead the per-form clientlib is
> **SELF-CONTAINED**: it COPIES (once) the subset of `@name` functions the form's rules reference
> into its own `functions.js`, and COPIES the base styling (`clientlib-forms-base/css/base.css`)
> into its own `css/`. ⚠️ The `guideContainer` `clientLibRef` MUST be a **SINGLE category** — the
> form-specific one (`{project}.forms.{formName}`); NEVER a comma-separated list (the endpoint
> resolves it as one category → empty list → all rules "Broken"). generate-pdf (which has NO
> `@name` functions, so it is safe) loads at runtime as a plain **dependency**:
> `dependencies="[core.forms.components.runtime.all,{project}.forms.generate-pdf]"` — **no
> `embed`, no `forms.base`.** Artifact 1 already sets `clientLibRef="{project}.forms.{formName}"`.

> ⚠️ **ORDER MATTERS — do the clientlib LAST, and produce it by INVOKING `create-form-clientlib`.**
> This artifact must be the **final step**: first finish artifacts 1–4 so the form fully
> exists with its **theme and template** (the `guideContainer`, panels, fields, conf context,
> and `cq:template` all in place), THEN invoke the **`create-form-clientlib`** skill to add the
> functions and wire the rules. Do **NOT** build the clientlib first, and do **NOT** inline a
> hand-written `functions.js` interleaved with the form. **VERIFIED:** when the clientlib/rules
> are improvised in the same pass as the form (instead of `create-form-clientlib` run last),
> the Rule Editor comes up with **no functions and no rules** ("Form Objects" empty) even though
> the form renders and the JS runs — because the bundled pass drops the function-authoring
> discipline the dedicated skill enforces. Building the form first (theme + template), then
> running `create-form-clientlib` last, is the sequence that works.

**1. Extract validations from the requirement doc.** From whatever was supplied (PDF,
image, text, schema, description), list each field's constraints: text-only / max length,
digit counts, number ranges, date constraints (e.g. past-only), email/format, allowed
enum values, cross-field rules (e.g. "must differ from X"), file types & size. One
validator function per distinct rule. (Just note them now — you author them in step 3.)

**2. Finish the form first.** Confirm artifacts 1–4 are complete and the form is wired to its
theme (`themeRef` / conf context) and template (`cq:template`). The `guideContainer` already
carries the SINGLE-category `clientLibRef="{project}.forms.{formName}"` (Artifact 1); the form
clientlib is self-contained (copied base styling) and loads generate-pdf via `dependencies`, so
the hook is in place. Deploy/verify the form renders before adding the clientlib if you can.

**3. Invoke `create-form-clientlib` as the LAST step.** Hand the noted validations to that
skill — it is the source of truth and it owns the **scanner-critical rules** that make the
Rule Editor discover your functions (these are exactly what a bundled pass drops):
- **Global scope only** — top-level `function name(...)`. NO IIFE, NO `window.X={}` namespace
  (the scanner only finds top-level declarations).
- **Escape EVERY `/` inside a regex literal — including inside `[...]` char classes.** A raw
  `/` in a char class (e.g. `/^[A-Za-z0-9 ,.\-/#]+$/`) is read as the regex terminator, so the
  scanner mis-parses the file and discovers **zero** functions. Write `\/`. (`node --check`
  does NOT catch this; grep for a `/` inside `[...]` before shipping.)
- **Empty-safe** (return `true` on empty so `required` owns emptiness); each function
  JSDoc-annotated (`@name`/`@param`/`@return`); helpers `@private`.
- It creates the clientlib at `/apps/clientlibs/{formName}-clientlib/` and authors each
  validated field's `validationExpression="<fn>(...) == true()"` + `validateExpMessage` plus the
  `fd:validate` AST (`FUNCTION_CALL.impl` = `"$0($1)"` / `"$0($1, $2)"`; cross-field `COMPONENT`
  `type` matches the real field type, e.g. `DATE`).

**4. Verify discovery before declaring done.** After deploy, open the form's Rule Editor and
confirm the Form Objects tree populates **with the rules visible**. Empty tree ⇒ the
`functions.js` scan failed — re-check regex-escaping / global-scope. The clientlib JS must serve
at `/etc.clientlibs/clientlibs/{formName}-clientlib.js`.

> ⚠️ Other verified traps: (a) json-formula booleans are **`true()` / `false()`** — a bare
> `== true` makes EVERY input fail. (b) A *validate* rule's `fd:events` stays empty — runtime
> validation comes from `validationExpression`; the `fd:validate` AST is only for editor
> visibility. Generate the escaped AST with a throwaway script, never by hand.

---

## Global XML conventions

Apply these rules in every file you generate:

**Booleans** — always use the typed syntax, never bare strings:
```
enabled="{Boolean}true"     ✓
enabled="true"               ✗
```

**Multi-value attributes** — always bracket syntax:
```
enum="[HR,Engineering,Finance]"     ✓
enum="HR,Engineering,Finance"        ✗
```

**Namespace declarations** — every namespace prefix used in a file must be declared on
the root `<jcr:root>` node. Missing declarations cause package installation to fail with
a cryptic XML parse error.

**Node names** — use PascalCase for panel and field nodes (e.g. `PersonalDetails`,
`FullName`). The `name` attribute value should match the node name exactly.

---

## Failure symptoms and their causes

If the form does not behave as expected after deployment, use this table to diagnose:

| Symptom | Cause | Where to fix |
|---|---|---|
| Form not visible in Forms & Documents | Artifact 2 missing, or `type="guide"` / `guide="1"` absent | Create artifact 2 with both attributes |
| Form page exists but opens with error | `cq:template` path does not exist | Confirm the template name with the user and correct it |
| Form opens but renders completely unstyled | Artifact 3 missing, or `sling:configRef` missing trailing `/` | Create artifact 3; add the trailing slash to `sling:configRef` |
| Theme loaded but wrong visual style | `themeRef` in artifact 1 or 2 differs from `themeArtifact` in artifact 3 | Make all three reference exactly the same `{project}-{theme}` value |
| CSS/JS returns 404 at runtime | `prefixPath` in artifact 3 does not match the form's `/content/forms/af/{project}/{formName}` path | Correct `prefixPath` to use the `{project}` folder segment |
| Field visible but required validation never triggers | `required="true"` present but `mandatoryMessage` missing | Add `mandatoryMessage` to the field |
| Dropdown / checkbox / radio renders with **no options** (empty) | Options authored as `<items>`/`item0` child nodes — ignored by Core Components | Replace with `enum` (codes) + `enumNames` (labels) attributes on the field; delete the `<items>` node |
| Conf context deployed but has no effect on server | Filter entry for `/conf/forms/{project}/{formName}` missing from filter.xml | Add the filter line and redeploy |
| Components not resolving (404 on resourceType) | a `sling:resourceType` uses the wrong namespace | Audit every `sling:resourceType` — all must use `{project}` |
| Package install fails with XML error | A namespace prefix used in a file is not declared on the root node | Add the missing `xmlns:...` declaration to the root |
| A **section/panel appears twice** after deploy (a flat copy above + the intended copy inside its accordion) | Orphan instance nodes from re-authoring: a panel was moved/renamed (e.g. `guideContainer/panelX` → `guideContainer/accordionX/panelX`) but `mode="update"` did not delete the old node | Source is correct — do NOT edit it. Purge the stale instance nodes (Forgemaster step 3b: Sling POST `:operation=delete` with CSRF token). Diff the live `guideContainer` child set against source; delete every live child not in source |

---

## Pre-flight checklist

Before deploying, verify every item. Each item maps to one of the failures above.

- [ ] User has confirmed all 5 inputs: `{project}`, `{formName}`, `{formTitle}`, `{theme}`, `{template}`
- [ ] `{project}` is used as the single namespace in every path root (resourceType, `/conf/`, and the form/DAM/conf-forms folder segment)
- [ ] Artifact 1 created: form page at `/content/forms/af/{project}/{formName}/.content.xml`
- [ ] Artifact 2 created: DAM asset at `/content/dam/formsanddocuments/{project}/{formName}/.content.xml`
- [ ] Artifact 3 created: conf context at `/conf/forms/{project}/{formName}/.content.xml`
- [ ] `sling:configRef` ends with a trailing `/`
- [ ] `themeRef` is identical in artifact 1 `guideContainer` and artifact 2 `metadata`
- [ ] `themeArtifact` and `siteTemplatePath` in artifact 3 match the same `{project}-{theme}` string
- [ ] `prefixPath` in artifact 3 uses the `{project}` folder segment (matches the form path)
- [ ] Artifact 3 root node has both `jcr:primaryType="sling:Folder"` and `sling:resourceType="sling:Folder"`
- [ ] Form title is an **explicit AF Title component** (first child of guideContainer:
      `sling:resourceType={project}/components/adaptiveForm/title`, `fd:htmlelementType="h1"`, `value`
      + `aria-label` set, a `css` class), with guideContainer `showTitle="{Boolean}false"` — NOT the
      empty showTitle band; the title renders visibly (verify by pixels)
- [ ] Every field node has `name`, `jcr:title`, `fieldType`, `aria-label`, and `<cq:responsive>/<default>`
      (grep the form `.content.xml` — every field/panel/button must carry an `aria-label`)
- [ ] Numeric inputs use `numberinput` (`number-input`), not `text-input`; choice fields use the
      right component (`radiobutton` single-select vs `checkboxgroup` multi-select)
- [ ] If the form is PUBLIC/unauthenticated, a CAPTCHA field was considered (and its cloud config
      flagged to the user); if it needs a PDF record, Document of Record was raised with the user
- [ ] Every dropdown / checkbox-group / radio field declares its options as `enum` +
      `enumNames` **attributes** — and there is **NO** `<items>`/`item0` child node anywhere
      in the form XML (grep the form `.content.xml` for `<items` — it must return nothing).
      `enum`/`enumNames` are equal length; enum codes match any rule that references them
- [ ] Every required field has `required="true"` AND `mandatoryMessage`
- [ ] Submit button present with `fd:rules fd:click="submitForm()"` and `fd:events fd:click="submitForm()"`
- [ ] Filter entry for `/conf/forms/{project}/{formName}` added to filter.xml
- [ ] All namespace prefixes used in each file are declared on its root `<jcr:root>` node
- [ ] All boolean values use `{Boolean}true` / `{Boolean}false` syntax
- [ ] Artifact 5 created: `/apps/clientlibs/{formName}-clientlib/` — SELF-CONTAINED: global-scope,
      empty-safe `functions.js` (one validator per requirement-doc rule, each `@name` defined
      ONCE) + `js.txt`, copied base styling (`css/base.css` + form CSS in `css.txt`) + node;
      `dependencies="[core.forms.components.runtime.all,{project}.forms.generate-pdf]"` — **no
      `embed`, no `forms.base`**
- [ ] **No duplicate `@name` definitions** — `grep -c "function <name>"` on the built
      `{formName}-clientlib.js` = 1 for each custom function (embedding/depending on `forms.base`
      makes it 2 → customfunctions endpoint returns empty → all rules "Broken")
- [ ] `clientLibRef` on the `guideContainer` is a **SINGLE category** — the form-specific one
      (`{project}.forms.{formName}`) — NOT a comma-separated list (a multi-value `clientLibRef`
      returns no custom functions from the Rule-Editor endpoint and marks every custom-fn rule
      "Broken"). Styling is self-contained (copied base.css); generate-pdf loads via `dependencies`;
      the base stays the canonical source but is NOT wired in; `ui.apps` filter covers
      `/apps/clientlibs`
- [ ] Each validated field has `validationExpression="…== true()"` + `validateExpMessage`
      AND a `fd:rules`/`fd:validate` authored AST (+ empty `fd:events`) so the rule is
      visible in the Rule Editor; no bare `== true`

---

## Zero-defect pre-handoff checklist (apply ALL learnings up-front)

Treat the accumulated learnings as a **mandatory zero-defect checklist** and self-verify each
(by pixels/DOM where visual) **before** the form is deployed/tested — aim for **0 issues on first
delivery**, not reactive fixes after the user flags them:

- [ ] **Template reused where possible, not needlessly forked** — `{template}` references an existing
      reusable template that fits; a newly-created template is the exception (justified only when none
      fit). Per-form theme/brand did not trigger a new template
- [ ] All screenshot visuals reproduced — separators/dividers (under each numbered section + the
      title/subtitle header), images, logos, icons, background bands (by pixels)
- [ ] Title via an **explicit AF Title component** renders visibly (not the empty showTitle band)
- [ ] Required red asterisks render on required labels (and NOT on optional fields)
- [ ] Submit gated on validation — an invalid form makes NO PDF and blocks submission
- [ ] `clientLibRef` is a **SINGLE category** (the form-specific clientlib) — NOT a comma-separated
      list (a multi-value clientLibRef breaks Rule-Editor custom-function discovery). The clientlib
      is SELF-CONTAINED (copied base.css + `functions.js` with each `@name` once) and does NOT
      `embed`/`depend on` `forms.base` (that would duplicate `@name` functions → endpoint empty →
      all rules "Broken"); generate-pdf loads via `dependencies`
- [ ] `dataRef` JSONPath bindings with a non-blank Bind Reference (not `fd:formDataRef`)
- [ ] Submit-action node under `fd/af/submitactions` with a node-path `actionType`
- [ ] PDF empty-content guard present (no silently blank PDF)
- [ ] `fd:rules` AST correctness — validate/set-value/click ASTs correct; no bare-string `fd:click`

## Deployment (final step)

> **Pipeline mode (delegated by `formwright`): SKIP this deploy step — author artifacts only.**
> Deployment is centralized in the `forgemaster` lead, which runs the single authoritative build+deploy
> after Groundsmith (AGENTS.md → "Deployment is centralized in Forgemaster"). Only run the `mvn` build
> below when this skill is invoked **standalone / directly**, not as part of a delivery pipeline.

When run standalone, deploying to the local AEM instance is **not optional** and **not "leave it to
the user"** — it is the final step of this skill. After all 5 artifacts pass the pre-flight checklist,
build and deploy to `http://localhost:4502`, then run the runtime verification below. A form
that is authored but never deployed is **not done** — many of the failure modes above (theme
not applied, clientlib functions not discovered, 404s) only surface on the running instance.

```bash
# Build and deploy the FULL reactor — this is the ONLY command that reaches AEM
mvn clean install -PautoInstallSinglePackage
```

> ⚠️ **Always build the full reactor — never restrict with `-pl`.** The
> `autoInstallSinglePackage` upload is bound to the **`all` aggregate module's** install
> phase. Running `mvn ... -PautoInstallSinglePackage -pl ui.content` (or `-pl ui.apps`) to
> "go faster" **excludes `all`**, so the build ends in `BUILD SUCCESS` but only installs the
> package zips into the local `~/.m2` repository — **nothing reaches AEM** and every form /
> clientlib URL 404s afterward. Confirm the line **`Package installed in NNNms.`** appears in
> the output — that line is the actual AEM upload. If you only see
> `Installing ... to C:\Users\...\.m2\repository`, it did **not** deploy.

> 💡 **Environment traps (corporate proxy / TLS interception).** If the build fails on TLS
> (`PKIX path building failed`), HTTP 403 on `.jar` downloads, or the `aem-analyser` plugin
> trying to fetch the latest SDK, the deploy command may need machine-specific flags (e.g. a
> Windows truststore and skipping the analyser). These are environment-specific, not part of
> the form — check project/agent memory for the working command on this machine before
> re-diagnosing.

After deploying, verify on the running instance (default: `http://localhost:4502`):

1. Open Forms & Documents — the new form must appear in the listing
2. Open the form in the AF editor — it must load without errors; each validated field
   shows its validation rule in the Rule Editor
3. Open the published form URL — theme CSS must be applied visually; the proxied clientlib
   serves at `/etc.clientlibs/clientlibs/{formName}-clientlib.js`
4. Submit the empty form — required field validation messages must appear; enter valid
   values — field validations must PASS (not fail), and invalid values must show the message

**Fast programmatic verification (no browser needed)** — fetch these on the running instance
and confirm each returns HTTP 200; the model JSON must contain your validator function names:

```bash
# clientlib JS serves (proxied)
curl -u admin:admin -s -o /dev/null -w "%{http_code}\n" \
  http://localhost:4502/etc.clientlibs/clientlibs/{formName}-clientlib.js

# runtime model resolves AND carries the validationExpression functions
curl -u admin:admin -s \
  "http://localhost:4502/content/forms/af/{project}/{formName}/jcr:content/guideContainer.model.json" \
  | grep -o "validate[A-Za-z]*"

# dropdown / checkbox / radio options reach the runtime model — every enumName label
# you authored must appear here. If a choice control renders empty, this is why:
# you used <items> child nodes (ignored by Core Components) instead of enum/enumNames.
curl -u admin:admin -s \
  "http://localhost:4502/content/forms/af/{project}/{formName}/jcr:content/guideContainer.model.json" \
  | grep -o "enumNames"
```

Only after the deploy succeeds and these checks pass is the form complete.
