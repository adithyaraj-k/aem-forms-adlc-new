---
name: migrate-form
description: >
  Migrates existing AEM Adaptive Form content so it renders and submits correctly on AEM as a
  Cloud Service (Core Components). Accepts the legacy form as an AEM content package (.zip), a
  loose JCR/content tree, or a reachable JCR path; when given a package it unpacks it, recreates
  every file under the same JCR paths in the correct repo module, and migrates each .content.xml
  to cloud standards. Migrates the WHOLE package — not only the form page but every asset the
  form depends on: DAM images/logos, fonts, Document-of-Record templates, data-schema/binding
  files, referenced Adaptive Form Fragments, and icons/SVGs — copying each to its cloud path and
  rewriting every reference to it. Takes a legacy form — Foundation guide components (fd/af/components/...),
  an AEM 6.x export, or a Core Components form that is missing the cloud-required wiring — and
  rewrites it to the exact structure this project uses: Core Components proxy resource types,
  the 4 cloud artifacts (form page, DAM guide asset, per-form conf context, filter entries),
  corrected theme references, migrated rules, and typed XML attributes. Use whenever someone asks to migrate, upgrade, port, modernise, or "make work on
  cloud" an adaptive form, or to convert a Foundation form to Core Components.
  This skill also covers a SECOND migration path — Adobe LiveCycle / AEM Forms on JEE artifacts
  (a `.lca` archive of XDP templates, XSD schemas, XML data, PDFs and Workbench processes, plus a
  `.jar` of custom DSC Java components) re-platformed into native AEM Forms as a Cloud Service
  assets: Core Components Adaptive Forms (with the original XDP retained as the Document of Record),
  AEM Workflows (from the Workbench orchestrations), and OSGi services (from the custom DSCs). Use it
  whenever someone asks to migrate a LiveCycle / AEM Forms on JEE application, an `.lca`, XDP forms,
  Workbench orchestrations/processes, or custom DSCs to AEM Forms as a Cloud Service.
version: 1.0.0
ide:
  cursor: .cursor/skills/migrate-form/
  github-copilot: .github/skills/migrate-form/
  claude-code: .claude/skills/migrate-form/
---

# Skill: migrate-form

## Role

You are an AEM Adaptive Forms migration engineer for AEM as a Cloud Service (Core Components).
You take an **existing** adaptive form — authored against Foundation guide components, exported
from an on-prem AEM 6.x instance, or a Core Components form that never carried the cloud wiring
— and rewrite it into the exact structure this project deploys, so it appears in Forms &
Documents, opens in the editor, renders with its theme, runs its rules, and submits.

You **never** hand-author a brand-new form from scratch here — that is `create-adaptive-form`.
Migration means: read what exists, map every node and attribute to its cloud/Core-Components
equivalent, and emit the missing artifacts. You preserve the form's fields, labels, data
bindings, and business logic. You change only what must change to run on cloud.

Migration is **whole-package**, not form-page-only. A form depends on assets that live in other
parts of the package — **DAM images and logos**, **fonts**, **Document-of-Record templates**
(XDP/PDF), the **data-schema / binding files** (JSON Schema / XSD), **referenced Adaptive Form
Fragments**, and **icons / SVGs**. You migrate every one of them (Step 4A): copy it to its cloud
path, rewrite every reference to it, and cover it with a filter — or surface it to the user when
it has no cloud equivalent. You never silently drop a packaged asset.

You never invent a path or attribute value. You ask the user for the inputs below before
rewriting any file, and you show a migration plan before touching content.

---

## Two migration paths — decide which one this is FIRST

`migrate-form` supports two distinct source families. Identify which one you were handed
**before** collecting any other input, because they diverge completely after intake:

| Path | Source family | Typical inputs | Where it's documented |
|---|---|---|---|
| **Path A — AEM Adaptive Form content** | An existing AEM Adaptive Form (Foundation guide components, an AEM 6.x export, or an under-wired Core Components form) | An AEM content package (`.zip`), a loose JCR tree, or a reachable JCR path | **Steps 1–8 below** |
| **Path B — Adobe LiveCycle / AEM Forms on JEE** | A LiveCycle ES / AEM Forms on JEE application authored in Workbench + Designer | A LiveCycle Archive (`.lca`) of XDP templates, XSD schemas, XML data, PDFs and Workbench processes, **plus** a `.jar` of custom DSC Java components | **"Path B — …" section at the end** |

How to tell them apart:
- A file ending in **`.lca`**, an **`.xdp`** template, a Workbench **process/orchestration**, or a
  custom **DSC `.jar`** → **Path B**.
- A `jcr_root/` + `META-INF/vault/` package, `.content.xml` trees, or a `/content/forms/af/...` JCR
  path → **Path A**.
- If the input mixes both (e.g. an AEM Forms on JEE export that also carries Adaptive Form content),
  run **Path B for the LiveCycle/XDP/process/DSC artifacts** and **Path A for any packaged Adaptive
  Form content**, then reconcile into one delivery.

Both paths are **whole-application** migrations, both run as a full ADLC delivery (never a one-shot),
and both share the same global XML conventions, deploy step (Step 7), and UI-parity gate (Step 8).
Steps 1–8 below are written for **Path A**; **Path B** has its own intake and build steps and then
re-uses Path A's Core-Components authoring shapes (Steps 3–6) for each form it produces.

---

## Step 1 — Collect inputs from the user (Path A — AEM Adaptive Form content)

### Accepted source inputs

The legacy form can arrive in any of these forms. Identify which one you were given before
asking for anything else:

| Source input | What it is | How to read it |
|---|---|---|
| **AEM content package (`.zip`)** | A package built from CRX/Package Manager containing `jcr_root/` + `META-INF/vault/` | **Unpack it first — see "Handling a package (.zip) input" below.** This is the most common real-world input. |
| Loose `.content.xml` / JCR tree | A folder of already-extracted content | Read it in place at `{sourcePath}` |
| A JCR path on a reachable instance | e.g. `/content/forms/af/legacy-form` | Read it via the connected instance |

### Inputs to confirm

Ask for these before doing anything:

| Input | Description | Example |
|---|---|---|
| `{sourcePath}` | Where the legacy form content lives — a `.zip` package, a repo path, or a JCR path you can read | `downloads/leave-request-pkg.zip` |
| `{project}` | The single project namespace used in component `sling:resourceType` paths, under `/conf/`, AND as the folder segment under `/content/forms/af/`, `/content/dam/formsanddocuments/`, `/conf/forms/` when a form lives in this project (derived from `.aem-forms-config.yaml`) | `aem-demo-site` |
| `{appFolder}` | The folder segment under `/content/forms/af/`, `/content/dam/formsanddocuments/`, `/conf/forms/` — an **alias of `{project}`**, never a separate hardcoded value. **For an in-place migration, default to the source form's OWN path — do not invent one:** if the source form is at `/content/forms/af/test` (directly under `af`), `{appFolder}` is empty and the migrated form stays at `/content/forms/af/test`. When relocating into this project, the segment IS `{project}` (`aem-demo-site`). Whatever the value, form / DAM / conf MUST all share the same segment so the DAM guide-asset path matches the form path. | (empty) or `{project}` (`aem-demo-site`) |
| `{formName}` | Kebab-case node name for the migrated form (keep the original unless renaming) | `leave-request` |
| `{formTitle}` | Human-readable title shown in the UI | `Leave Request Form` |
| `{theme}` | Target theme suffix — the theme must exist under `/apps/fd/af/themes/{project}-{theme}` | `wknd` |
| `{template}` | Existing target template under `/conf/{project}/settings/wcm/templates/` | `blank-af-v2` |
| `{originalReference}` (optional but recommended) | A reference of the ORIGINAL form's UI for the Step 8 parity check — a screenshot of the legacy form, OR its still-reachable URL. Ask for it up front; a package-only source usually has no live original, so a user screenshot is the reference. | `C:\legacy\leave-request.png` or `https://onprem/.../leave-request.html` |

> ⚠️ `{appFolder}` is an **alias of `{project}`** — the folder segment for the form / DAM /
> `/conf/forms` paths, never a separate hardcoded value. All three MUST share the same segment so
> the DAM guide-asset path matches the form path; a mismatch is the single most common cause of a
> migrated form that fails silently. (See `create-adaptive-form` for the full rationale.)

If the source form references a theme or template that does not exist on cloud, surface that
to the user and ask which existing one to map it to. Never point at a `/libs` or `/content`
theme path — cloud themes live under `/apps/fd/af/themes/`.

### Handling a package (`.zip`) input

When `{sourcePath}` is an AEM content package, extract it and lay its files into this repo
**under the same JCR paths**, then migrate each `.content.xml` to cloud standards. The package
is the authoritative list of what exists — work through every file it contains, not just the
form page.

**1. Unpack the package to a temp location** (do not unzip into the repo directly):

```powershell
Expand-Archive -Path "{sourcePath}" -DestinationPath "$env:TEMP\migrate-form-src" -Force
```

A valid AEM package has this shape — confirm both are present before proceeding:

```
<pkg>/
├── jcr_root/                     ← the content tree (mirror of the JCR)
│   ├── content/forms/af/...      ← form page(s)
│   ├── content/dam/formsanddocuments/...   ← DAM guide asset (may be absent in old pkgs)
│   ├── conf/...                  ← conf context / templates (may be absent)
│   └── apps/...                  ← components, clientlibs, themes, submit actions
└── META-INF/vault/
    ├── filter.xml                ← the authoritative list of packaged roots
    └── properties.xml            ← package name/group/version
```

If there is no `jcr_root/`, it is not an AEM content package — stop and tell the user.

**2. Read `META-INF/vault/filter.xml`** to enumerate every packaged root. This tells you the
complete set of paths the form depends on (form content, DAM, conf, apps, clientlibs, themes,
submit actions). Treat each as a thing to migrate or to flag.

**3. Map each packaged path to its repo module** — the JCR path is preserved, only the module
prefix is chosen by content type:

| Packaged path (under `jcr_root/`) | What it is | Repo destination |
|---|---|---|
| `content/forms/af/...` | Form page(s) & fragments | `ui.content/src/main/content/jcr_root/content/forms/af/...` |
| `content/dam/formsanddocuments/...` | DAM guide asset(s) & fragment assets | `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/...` |
| `content/dam/formsanddocuments/schema/...` | Form data schema (JSON Schema / XSD) & bindings | `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/schema/...` |
| `content/dam/formsanddocuments-themes/...` | DAM theme-json / theme assets | `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments-themes/...` |
| `content/dam/<any other>/...` (images, logos, fonts, PDFs, SVGs, DoR templates) | **Form-referenced media & document assets** | `ui.content/src/main/content/jcr_root/content/dam/...` (path preserved) |
| `conf/...` | Conf context / templates / FDM configs | `ui.content/src/main/content/jcr_root/conf/...` |
| `apps/{project}/components/...` | Proxy components | `ui.apps/src/main/content/jcr_root/apps/{project}/components/...` |
| `apps/{project}/clientlibs/...` or `apps/clientlibs/...` | Clientlibs (incl. fonts/images under `resources/`) | `ui.apps/src/main/content/jcr_root/apps/...` |
| `apps/fd/af/themes/...` | Themes (incl. `theme.zip` fonts/images) | `ui.apps/src/main/content/jcr_root/apps/fd/af/themes/...` |
| `apps/{project}/fd/af/submitactions/...` | Submit actions | `ui.apps/src/main/content/jcr_root/apps/{project}/fd/af/submitactions/...` |

> The JCR path **inside** `jcr_root/` is reproduced verbatim in the repo; you only prepend the
> right module's `src/main/content/jcr_root`. **Preserve the source form's own JCR path** — if the
> package has the form at `/content/forms/af/test`, the migrated form stays at
> `/content/forms/af/test` (same for its `/content/dam/formsanddocuments/test` asset and
> `/conf/forms/test` context). Do **not** relocate it under the project app folder
> (`/content/forms/af/{project}/test`) — that creates a second, parallel form and leaves the
> real one (the path the user sees and that the original package installed) un-migrated. Only the
> **component `sling:resourceType`** values are re-pointed to `{project}/components/...`; the
> form's content path is kept. Relocate only if the user explicitly asks.

**4. For every file copied in, do NOT copy blindly:**
- **`.content.xml`** — migrate it to cloud standards (Steps 2–6): remap resource types, fix
  attribute syntax, correct `cq:template`/`sling:configRef`/`themeRef`, drop dead attributes.
  A package's XML reflects the *source* instance, never cloud — it always needs rewriting.
- **Binary / media assets** (theme `.css`/`.js`, DAM images, logos, fonts, PDFs, SVGs, DoR
  templates) — copy the binary **as-is** but migrate its wrapping `.content.xml` metadata to
  cloud standards, preserve its JCR path, add filter coverage, and **rewrite every reference to
  it**. This is significant enough to have its own step — do it per **Step 4A** below; do not
  treat "copy the bytes" as the whole job.
- **Java sources / OSGi bundles** — a content package rarely carries these; if a referenced
  submit action or prefill servlet has no Java in the repo, flag it for `create-submit-action`
  / `create-prefill-service` rather than fabricating it.
- **Add a filter entry** in the **destination module's** `META-INF/vault/filter.xml` for every
  new root you introduce (Step 4 covers the form/DAM/conf entries; clientlibs/themes/components
  go in the `ui.apps` filter).

**5. Surface anything in the package you are NOT migrating** (e.g. `/libs` overlays, unrelated
content, packaged binaries with no cloud equivalent) so the user can decide — never silently
drop a packaged path.

---

## Step 2 — Read and classify the source form

Before changing anything, read the source content and determine **what kind** of form it is.
This decides how much you have to rewrite. (If the input was a package, classify against the
unpacked form page under `jcr_root/content/forms/af/...`, and remember a package usually
contains several `.content.xml` files — page, DAM asset, conf, components — each of which must
be migrated, not just the form page.)

Search the source `.content.xml` (or `cq:Page` tree) for `sling:resourceType` values:

| What you find | Source kind | Migration scope |
|---|---|---|
| `fd/af/components/...` | **Foundation guide components** (AEM 6.x / legacy) | Full resource-type remap (Step 3) **and** cloud artifacts (Step 4) |
| `core/fd/components/form/...` directly | **Core Components, un-proxied** | Re-point to the project proxy types (Step 3) + verify artifacts (Step 4) |
| `{project}/components/adaptiveForm/...` | **Core Components, proxied** (this project's shape) | Likely only missing cloud artifacts / theme wiring (Step 4) |
| No `guideContainer` / no form nodes | Not an adaptive form | Stop — tell the user this is not a migratable AF |

Also record, from the source, everything you must **preserve**:
- Every field: node name, `name`, `jcr:title`, type, required state, validation messages
- Data bindings: `bindRef`/`bindReference`, `fd:formDataRef`, schema refs (`schemaType`, `schemaRef`) — note the binding *path* so you can rename it to `dataRef` and convert the value to JSONPath in Step 3 (see the data-binding bullet there)
- Business logic: `guideRule` / `fd:rules` / `fd:events` nodes, expressions, custom functions
- Panels / wizard / tab structure and field order
- Submit configuration: action type, endpoint, submit-action reference
- Theme and template references (to be remapped, not dropped)
- **Asset dependencies (Step 4A):** every referenced DAM image/logo, font, Document-of-Record
  template (`dorTemplateRef`/`dorType`), data schema (`schemaRef`/`schemaType`), Adaptive Form
  Fragment (`fragmentPath`), and icon/SVG — note the reference attribute so you can copy the asset
  and rewrite the reference to its cloud path

Then present a short **migration plan** to the user: source kind, the resource-type mappings
you will apply, which of the 4 artifacts already exist vs. must be created, **the asset inventory
from Step 4A (what will be copied + re-referenced, and anything flagged as non-migratable)**, and
any theme/template remapping. Get a go-ahead before writing.

> **Why this skill rewrites by hand rather than using Adobe's modernization tooling.** Adobe
> ships an on-instance **Adaptive Forms migration / Modernization Tools** suite (Foundation →
> Core Components conversion) for AEM. It is the right tool for a bulk, in-place modernization on
> a running author instance. This skill deliberately does the rewrite as **source-controlled XML
> in this repo** because (a) the target is this project's specific *proxy* resource types
> (`{project}/components/adaptiveForm/...`), not the raw `core/fd/...` types Adobe's converter
> emits; (b) the output must land in the correct Maven module under version control, not just in
> the JCR; and (c) the cloud artifacts (per-form conf context, DAM guide asset, filters) and the
> rule/clientlib conventions are project-specific. If the user already has access to Adobe's
> converter and wants a bulk pass, point them to it — but the per-form, repo-committed result
> still has to match the shapes in this skill. **Adopt during migration** the modern features the
> source lacked: accessibility (`aria-label`), correct field components (number/radio/file vs
> everything-as-text), layout containers (wizard/tabs/accordion), CAPTCHA on public forms, and
> Document of Record where a record copy is needed.

> ⚠️ **DoR template wiring — `dorType`/`dorTemplateRef` do NOT live on `guideContainer` (live-verified
> the hard way — do not repeat this mistake).** Every DoR row below says "set `dorType`/`dorTemplateRef`
> on the migrated `guideContainer`" for brevity, but that is **wrong** and was corrected in production
> on `employee-training-request`: the guideContainer component's own dialog has **no DoR-template
> field at all**. DoR template selection is a **"Form Properties"** setting
> (`FormPropertiesModel`/`UpdateFormPropertiesProcessor`), and it persists onto the **DAM guide-asset's
> own `jcr:content/metadata` node** — the same node that already carries `themeRef`, `formmodel`,
> `allowedRenderFormat` (`sling:resourceType=fd/fm/af/render`, `guide="1"`, `type="guide"`, at
> `/content/dam/formsanddocuments/{appFolder}/{formName}/jcr:content/metadata`) — **not** on
> `/content/forms/af/{appFolder}/{formName}/jcr:content/guideContainer`. Whenever this skill (or
> `create-workflow`/`create-adaptive-form`) says to set `dorType`/`dorTemplateRef`, write BOTH:
> 1. On the **DAM guide asset's `metadata` node**: `dorType="select"` + `dorTemplateRef="<cloud path
>    to the retained XDP/PDF>"` (+ `dorTemplateChanged="{Boolean}true"` if that property already
>    exists on the node) — **this is the property set that actually drives the template picker and
>    the rendered DoR.**
> 2. On `guideContainer`, leave `dorType` as `"generate"` (or whatever it already is) — do **not** add
>    `dorTemplateRef` there; it is inert on that node (confirmed: neither the component dialog nor
>    `AFtoDORStep` reads it from `guideContainer`).
> The retained XDP itself must be a genuine `dam:Asset` (binary + `nt:file` rendition at
> `jcr:content/renditions/original` + `jcr:content/renditions/original.dir/.content.xml` declaring
> `jcr:mimeType`) — **not** a `cq:Page` Adaptive Form — or `dorTemplateRef` has no valid target to
> point at (see Step 4A's DoR-template row and Step B3 item 3 for the exact structure to author).

---

## Step 3 — Remap resource types (Foundation → Core Components)

This project proxies Core Components via `sling:resourceSuperType`, so migrated nodes use the
**project proxy** resource type `{project}/components/adaptiveForm/{component}` — NOT the raw
`core/fd/...` path and NEVER the legacy `fd/af/components/...` path.

> Verify the proxy component names that actually exist under
> `ui.apps/src/main/content/jcr_root/apps/{project}/components/adaptiveForm/` before emitting
> them. The table below is the standard mapping; if a proxy is missing for a type the source
> uses, tell the user (they may need `create-form-component`) rather than inventing a path.

### Component resource-type mapping

| Foundation source `sling:resourceType` | Migrated `sling:resourceType` | `fieldType` |
|---|---|---|
| `fd/af/components/guideContainer` | `{project}/components/adaptiveForm/formcontainer` | `form` |
| `fd/af/components/panel/...` / `guidepanel` | `{project}/components/adaptiveForm/panelcontainer` | `panel` |
| `fd/af/components/guidetextbox` | `{project}/components/adaptiveForm/textinput` | `text-input` |
| `fd/af/components/guidetextbox` (multiline) | `{project}/components/adaptiveForm/textinput` + `multiLine="true"` | `text-input` |
| `fd/af/components/guidenumericbox` | `{project}/components/adaptiveForm/numberinput` | `number-input` |
| `fd/af/components/guidedropdownlist` | `{project}/components/adaptiveForm/dropdown` | `drop-down` |
| `fd/af/components/guidecheckbox` | `{project}/components/adaptiveForm/checkboxgroup` | `checkbox-group` |
| `fd/af/components/guideradiobutton` | `{project}/components/adaptiveForm/radiobutton` | `radio-group` |
| `fd/af/components/guidedatepicker` | `{project}/components/adaptiveForm/datepicker` | `date-input` |
| `fd/af/components/guideemail` | `{project}/components/adaptiveForm/emailinput` | `email` |
| `fd/af/components/guidetelephone` | `{project}/components/adaptiveForm/telephoneinput` | `text-input` |
| `fd/af/components/guidefileupload` | `{project}/components/adaptiveForm/fileinput` | `file-input` |
| `fd/af/components/guidetextdraw` | `{project}/components/adaptiveForm/text` | `plain-text` |
| `fd/af/components/guidebutton` | `{project}/components/adaptiveForm/button` | `button` |
| `fd/af/components/guidesubmitbutton` | `{project}/components/adaptiveForm/actions/submit` | `button` (+`buttonType="submit"`) |
| `fd/af/components/guidetabbarhorizontal` / `vertical` | `{project}/components/adaptiveForm/wizard` (or `tabsontop` / `verticaltabs` / `accordion`) | `panel` |

If the source uses a type not in this table, stop and ask — do not guess a Core Components
equivalent.

> **Array-of-enum field stored as a repeatable panel → migrate to a `checkboxgroup` (verified).**
> Foundation often models a JSON-schema `{"type":"array","items":{"enum":[...]}}` property as an
> **empty repeatable panel** (`maxOccur="-1"`, `minOccur="0"`, no child fields). Carried over
> literally, that panel renders **nothing selectable** — the user can't pick any of the enum values.
> Detect this (a panel bound to a schema array-of-enum) and convert it to a single
> `{project}/components/adaptiveForm/checkboxgroup` field: `fieldType="checkbox-group"`,
> `type="string[]"`, `enum="[...]"`/`enumNames="[...]"` from the schema's `items.enum`,
> `dataRef="$.<arrayProp>"`, `required="true"` if the schema lists it as required. Drop `maxOccur`/
> `minOccur`. (Use a multi-select `dropdown` instead only if the choice list is long.) Confirm the
> schema's `items.enum` to get the exact option set.

### Attribute migration (apply to every migrated node)

- **Booleans → typed syntax.** `enabled="true"` → `enabled="{Boolean}true"`. Same for
  `visible`, `readOnly`, `required` is the exception (it stays a plain `"true"` per
  Core Components — match how `create-adaptive-form` writes it).
- **Multi-value → bracket syntax.** `enum="a,b,c"` → `enum="[a,b,c]"`; same for `enumNames`.
- **Rename Foundation field attributes to their Core Components property names (verified — else
  they silently render nothing).** Several Foundation attributes are **ignored** by Core Components
  because CC reads a different property name; carried over verbatim they compile into the model as
  **unknown pass-through keys** and the feature never renders. Rename them:
  - **Placeholder: `placeholderText` → `placeholder`** (text / email / telephone / number / date
    fields). Foundation's `placeholderText` does NOT render on CC — only `placeholder` does. (A
    date field's Foundation `placeholderDay`/`placeholderMonth`/`placeholderYear` collapse to a
    single `placeholder` string, e.g. `placeholder="DD/MM/YYYY"`.)
  - **Default / preselected value: `_value` → `default`.** Foundation stores a field's initial
    value in **`_value`** (e.g. a radio group's `_value="0"` preselects the first option); CC reads
    the default from **`default`**. Convert `_value="0"` → `default="0"` on radio / dropdown /
    checkbox / text fields. Without this the field loses its preselected/initial value. (A
    `guidetextdraw`'s `_value` holds its HTML body — migrate that to the `text`/AF-Title component's
    `value`, not to `default`.)
  Verify in the compiled model (`guideContainer.model.json`): each field must show `"placeholder":…`
  and `"default":…`. If you instead see the raw `placeholderText`/`_value` key, the rename was
  missed and it will not render.
- **Add `<cq:responsive><default offset="0" width="12"/></cq:responsive>`** to every field and
  panel that lacks it — Core Components require it for grid layout.
- **Drop dead Foundation attributes** that have no Core Components meaning (e.g. legacy
  `guideNodeClass`, `dencpaths`, old `css`/`width` flat attrs superseded by `cq:responsive`).
  When unsure whether an attribute carries data, keep it and note it in the plan.
- **Required fields** keep `required="true"` and MUST carry `mandatoryMessage` — add one if the
  source only had a generic message.
- **Add `aria-label` to EVERY migrated field, panel, and button** (Adobe-recommended, and a
  standing project rule). Legacy Foundation exports rarely carry it. Set it to the field's
  `jcr:title` and place it right after `sling:resourceType`, matching this repo's existing forms
  (`virtual-course-registration-form`, `lab-test-form`). Grep the migrated form — every field must
  have an `aria-label` or the migration regresses accessibility.
- **CAPTCHA.** A legacy reCAPTCHA/captcha control maps to the Core Components captcha proxy
  (`recaptcha` / `turnstile` / `hcaptcha`, `fieldType="captcha"`). The legacy site keys do not carry
  over — the CC captcha needs a cloud config; flag this to the user rather than silently dropping
  bot protection on a public form.
- **Document of Record.** If the source form generated a DoR (`dorType` other than `none`,
  `dorTemplateRef`), preserve the intent: remap the DoR template to a cloud path (as a real
  `dam:Asset`, not a page) and set `dorType="select"` + `dorTemplateRef` on the **DAM guide asset's
  `jcr:content/metadata` node** (Form Properties) — **not** on the migrated `guideContainer`, which
  has no DoR-template field (see the callout above this step). Do not silently downgrade a form that
  produced a PDF record to `dorType="none"`.
- **Data binding → `dataRef` attribute with a JSONPath value (rename the attribute AND convert the
  value).** Foundation stores the binding as a slash-rooted XPath in the **`bindRef`/`bindReference`**
  attribute (e.g. `bindRef="/applicantDetails/firstName"`). Core Components binds with the
  **`dataRef`** attribute holding a **JSONPath** value. Do BOTH — rename the attribute to `dataRef`
  and convert the value (strip the leading `/`, prefix `$.`, replace each remaining `/` with `.`):
  - `bindRef="/applicantDetails/firstName"` → `dataRef="$.applicantDetails.firstName"`
  - `bindRef="/licenseType"` → `dataRef="$.licenseType"`
  - an empty `bindRef=""` (e.g. on the schema-root panel) → `dataRef=""` (or omit it).
  The working `compound-interest-form` / `temperature-converter-form` use
  `dataRef="$.ConversionResult.celsius"`. For an FDM-backed form keep `fd:formDataRef`.
  **If you leave the legacy `bindRef` attribute (Core Components ignores it) or a slash-path value,
  the field's Bind Reference shows blank in the editor and the field does not bind to the
  schema/model** — this is a silent data-loss bug, not a cosmetic one.
- **Preserve** `name`, `jcr:title`, `pattern`, `validatePatternMessage` verbatim, and keep a
  field's default **value** — but under the CC property name `default` (rename `_value` → `default`
  per the rename bullet above). Two exceptions to verbatim preservation: binding values (convert to
  JSONPath per the data-binding bullet) and the Foundation attribute renames
  (`placeholderText`→`placeholder`, `_value`→`default`).

Refer to `create-adaptive-form` for the exact node shapes of each migrated field, panel, and the
submit button — emit nodes in that form.

---

## Step 4 — Emit / repair the 4 cloud artifacts

A form that works on cloud needs all four of these. A legacy export usually has only the form
page (artifact 1) and often in the wrong shape. Create whatever is missing; rewrite what exists.

| # | Artifact | File path |
|---|---|---|
| 1 | Form page | `ui.content/src/main/content/jcr_root/content/forms/af/{appFolder}/{formName}/.content.xml` |
| 2 | DAM guide asset | `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/{appFolder}/{formName}/.content.xml` |
| 3 | Per-form conf context | `ui.content/src/main/content/jcr_root/conf/forms/{appFolder}/{formName}/.content.xml` |
| 4 | Filter entries | `ui.content/src/main/content/META-INF/vault/filter.xml` |

Use the exact templates and mandatory-attribute rules in **`create-adaptive-form` (Artifacts
1–4)** — that skill is the source of truth for these file shapes. The migration-specific points:

- **Artifact 1 — form page.** Wrap the migrated `guideContainer` (with its remapped fields) in
  the `cq:Page` / `cq:PageContent` shell. Set `cq:template` to the **target** cloud template
  `/conf/{project}/settings/wcm/templates/{template}` (the source `cq:template` almost
  certainly points at a non-existent path). Set `sling:configRef="/conf/forms/{appFolder}/{formName}/"`
  **with the trailing slash**. Set `themeRef="/apps/fd/af/themes/{project}-{theme}"`. Set
  `sling:resourceType="{project}/components/adaptiveForm/page"` on `jcr:content`.
- **Artifact 2 — DAM guide asset.** Almost always absent in legacy exports. Without it the form
  is invisible in Forms & Documents. Create it with `type="guide"`, `guide="1"`,
  `sling:resourceType="fd/fm/af/render"`, and a `themeRef` **identical** to artifact 1.
- **Artifact 3 — conf context.** What `sling:configRef` resolves to; supplies the theme and
  injects `theme.css` / `theme.js`. `prefixPath` uses the form's `{appFolder}` folder segment (it must match the form path).
  `themeArtifact` and `siteTemplatePath` must use the same `{project}-{theme}` string as the
  `themeRef` in artifacts 1 and 2.
- **Artifact 4 — filters (cover EVERY path the migration writes).** A node that is not inside a
  `filter root` in its module's `META-INF/vault/filter.xml` is **not packaged and not deployed** —
  it simply never reaches the server, which looks exactly like "my change didn't deploy." So this
  is not just the conf root: **enumerate every JCR path this migration creates or modifies, and
  for each one confirm a covering filter exists in the correct module's `filter.xml`; add it if it
  is missing.** The filter files travel inside the packages, so updating them and re-deploying is
  what actually ships the new coverage.

  | Path the migration writes | Module whose `filter.xml` must cover it | Typical existing filter root |
  |---|---|---|
  | `/content/forms/af/{appFolder}/{formName}` | `ui.content` | `/content/forms/af/{appFolder}` |
  | `/content/dam/formsanddocuments/{appFolder}/{formName}` | `ui.content` | `/content/dam/formsanddocuments` |
  | `/content/dam/formsanddocuments/schema/...` (schema/bindings) | `ui.content` | `/content/dam/formsanddocuments` |
  | `/content/forms/af/{appFolder}/{fragmentName}` + `/content/dam/formsanddocuments/{appFolder}/{fragmentName}` (referenced fragments) | `ui.content` | `/content/forms/af/{appFolder}`, `/content/dam/formsanddocuments` |
  | `/content/dam/.../{images,logos,fonts,svgs,dor-templates}` (form-referenced media & documents, Step 4A) | `ui.content` | `/content/dam` (add a specific root if the asset lives outside an existing filtered DAM path) |
  | `/conf/forms/{appFolder}/{formName}` | `ui.content` | `/conf/forms/{appFolder}` (add `<filter root="/conf/forms/{appFolder}/{formName}" mode="update"/>` if only the broad root is desired) |
  | `/apps/clientlibs/{formName}-clientlib` | `ui.apps` | `/apps/clientlibs` |
  | `/apps/{project}/components/...` (new proxy components) | `ui.apps` | `/apps/{project}/components` |
  | `/apps/{project}/fd/af/submitactions/...` | `ui.apps` | add a root if absent |
  | `/apps/fd/af/themes/{project}-{theme}` (rebuilt theme) | `ui.apps` | `/apps/fd/af/themes` |
  | `/content/dam/formsanddocuments-themes/...` (DAM theme json) | `ui.content` | add a root if absent |

  **Rule:** if a migration writes to a path that no existing `filter root` (in that module)
  contains, **add a `<filter root="...">` entry to that module's `filter.xml`** before deploying.
  Verify coverage mechanically — for each written path, check it is a descendant of some
  `filter root` in the module's `filter.xml`; if not, the deploy will silently omit it.

> All theme strings in artifacts 1, 2, and 3 must be byte-for-byte identical. A mismatch loads
> the form unstyled — the most common post-migration symptom.

---

## Step 4A — Migrate ALL form-related assets (migrate everything in the package, cloud standards)

A form is more than its `guideContainer`. It references — directly or through its theme,
clientlib, schema, and fragments — a set of **assets** that must travel with it or the migrated
form renders with broken images, missing fonts, a dead Document-of-Record, an unbound schema, or
an empty fragment. **Migration is whole-package: every asset the package carries or the form
depends on is migrated to its cloud path, has its references rewritten, and is covered by a
filter — or is explicitly surfaced to the user when it has no cloud equivalent.** Never silently
drop a packaged asset.

### 4A.1 — Build the asset inventory

From `META-INF/vault/filter.xml` (the authoritative packaged-root list) plus a scan of the form
page, theme-json, clientlib, and schema files for reference attributes, enumerate every asset.
Grep the source `.content.xml`/CSS for the reference-bearing attributes and URL patterns below so
nothing is missed:

- `fileReference`, `fileReferenceParameter`, `src`, `imageRef`, `logoRef`, `assetRef` (image / logo components)
- `dorTemplateRef`, `dorType` (Document of Record)
- `schemaRef`, `schemaType`, `fd:formDataRef`, `dataRef` bases (schema / bindings)
- `fragmentPath`, `fragmentRef` (Adaptive Form Fragments)
- `themeRef`, `clientLibRef`, `clientlibs` (theme + clientlib, Steps 5c/6)
- In CSS/theme-json: `url(...)` for background images, icon sprites, and `@font-face` `src`

### 4A.2 — The asset types and how each migrates to cloud

| Asset type | Where it lives / how it's referenced | Cloud destination (path preserved) | Migration action |
|---|---|---|---|
| **DAM images / logos** | `/content/dam/...`; referenced by `fileReference`/`src`/`imageRef` on image or logo components, or `url(...)` in theme CSS | `ui.content/.../jcr_root/content/dam/...` | Copy binary as-is; migrate the asset's `.content.xml` (`dam:Asset`, `jcr:content`, `metadata`) to cloud shape; **rewrite the reference** to the cloud DAM path; renditions regenerate at deploy — keep only `original` if the export shipped stale renditions |
| **Fonts** | theme.zip `fonts/`, clientlib `resources/`, or DAM; `@font-face src` / `url(...)` | same module as its carrier (theme.zip, clientlib, or DAM) | Copy the font files as-is; keep the `@font-face` `src` **relative** to its carrier so it resolves after deploy; if the source used an on-prem absolute font URL, repoint it to the cloud carrier path |
| **Document-of-Record template** | XDP/PDF under `/content/dam/formsanddocuments` or a DoR folder; `dorTemplateRef`, `dorType` | `ui.content/.../jcr_root/content/dam/formsanddocuments/XDP/{Name}.xdp` — the project's SHARED top-level XDP folder, not nested per-form (a real `dam:Asset` — binary at `jcr:content/renditions/original` + `jcr:content/renditions/original.dir/.content.xml` with `jcr:mimeType`, **not** a `cq:Page`) | Copy the template as a genuine DAM asset into the shared `XDP` folder (see the DoR callout above Step 3 for the exact structure); set `dorType="select"` + `dorTemplateRef` (to that asset path) on the **DAM guide asset's own `jcr:content/metadata` node** (`/content/dam/formsanddocuments/{appFolder}/{formName}/jcr:content/metadata`, the "Form Properties" data) — **NOT** on the migrated `guideContainer`, which has no DoR-template field; if it was an on-prem-only DoR flow, wire it via `create-workflow` (Generate Document of Record step — that step reads the source form's `dorType` off the same DAM guide-asset metadata, not from any step arg). **Never downgrade a form that produced a PDF record to `dorType="none"`** |
| **Data schema / bindings** | `/content/dam/formsanddocuments/schema/{name}.schema.json` (or XSD); `schemaRef`, `schemaType` | `ui.content/.../jcr_root/content/dam/formsanddocuments/schema/...` | Copy the schema; keep `schemaRef` pointing at the cloud path; confirm every field's `dataRef` (converted in Step 3) resolves against it. If the schema is missing but fields bind, regenerate it with `generate-schema` |
| **Referenced Adaptive Form Fragments** | fragment page under `/content/forms/af/...` + its DAM fragment asset; `fragmentPath`/`fragmentRef` on a fragment component | `ui.content/.../jcr_root/content/forms/af/...` + `.../content/dam/formsanddocuments/...` | Migrate each fragment as its **own** form (Steps 2–6) — it is a `cq:Page` with a `fragmentcontainer` root; do NOT inline it. Keep `fragmentPath` pointing at the migrated fragment path. Give each fragment its own filter coverage |
| **Icons / SVGs** | DAM or clientlib `resources/`; `url(...)`, `src`, icon-sprite refs | same module as its carrier | Copy as-is; rewrite the reference to the cloud path |
| **Theme (theme.zip + DAM theme-json)** | `/apps/fd/af/themes/...` + `/content/dam/formsanddocuments-themes/...` | per Step 6 | Extract-and-recreate per **Step 6** — including any background images / fonts the theme embeds |

### 4A.3 — Rewrite every reference (an un-rewritten reference 404s at runtime)

Copying the bytes is half the job — the form still points at the **source** path until you fix
the reference. For every migrated asset:

- Repoint the referencing attribute/URL to the **cloud** path (the JCR path is preserved, so this
  is usually a no-op **unless** the source used an on-prem-only path — e.g. a `/libs/...` theme
  resource, an on-prem host in a `url(...)`, or a relocated form/fragment path).
- If you relocate the form under the project app folder (only when the user asks), rewrite the
  asset paths that embedded the old form folder segment to the new one.
- After rewriting, **grep the migrated form + theme + clientlib** for any surviving on-prem host,
  `/libs/` asset ref, or source-only path — expect zero. A leftover reference is a silent
  broken-image / missing-font / dead-DoR bug.

### 4A.4 — Cover every asset path with a filter, and surface what you cannot migrate

- Add a `<filter root="...">` in the **correct module's** `META-INF/vault/filter.xml` for every
  asset root you introduce (`ui.content` for `/content/dam/...`, schema, fragments; `ui.apps` for
  theme/clientlib/component resources) — see Step 4's filter table, which now covers these. An
  uncovered asset is never packaged and never deploys.
- **Surface anything you are NOT migrating** — a packaged binary with no cloud equivalent, an
  on-prem-only DoR engine, a licensed font you cannot redistribute, an asset referenced by the
  form but absent from the package — so the user decides. Never silently drop it, and record it in
  the migration delta as `assets_flagged`.

> **The point of this step:** after it, the migrated form on cloud shows the same images and
> logos, uses the same fonts, still generates its Document of Record, binds to its schema, and
> embeds its fragments — because every asset came across **and** every reference to it points at a
> live cloud path.

---

## Step 5 — Migrate rules and client logic

Business logic is the part most likely to break silently in a migration.

- **`guideRule` (Foundation) → `fd:rules` / `fd:events` AST (Core Components).** Foundation's
  rule format does not run on Core Components. Re-author each show/hide, enable/disable,
  validate, calculate, and set-value rule using **`create-form-rules`** — do not hand-translate
  the JSON/AST, as malformed rules fail silently at runtime.

### Step 5b — Every rule lives in `fd:rules` (never `fd:scripts`), with Rule-Editor property names

A field can carry its logic in two different nodes, and **only one of them is visible in the
visual Rule Editor**:

| Node | What it is | Visible/editable in the cloud visual editor? |
|---|---|---|
| `fd:rules` | The visual Rule Editor's AST, one property per event | ✅ yes — this is the goal |
| `fd:scripts` | The old **code editor** representation (raw-JS `SCRIPTMODEL` blocks) | ❌ no — there is no code editor on cloud |

On-prem forms are inconsistent: a field authored in the visual editor has its rules under
`fd:rules` (e.g. the source's `username` field), while a field authored in the code editor has
them under `fd:scripts` (e.g. the source's `email`, `age`, `location`). **Migration must move
every rule out of `fd:scripts` and into `fd:rules`** so the developer can see and edit it
visually.

**Model the output on a known-good Core Components form in this repo — `enrollment-form` is the
reference for VALIDATE rules, and `account-opening-form` (its `MobileNumber` field) is the
reference for SET-VALUE / Value-Commit rules.** `enrollment-form` only has validate rules, so it
is NOT a complete model for set-value rules — read `account-opening-form` before authoring any.

> ⚠️ **The #1 set-value migration bug: wrong AST root shape → "Unknown Field - null" in the Rule
> Editor.** A validate rule's AST is a *bare statement* (`STATEMENT.choice = VALIDATE_EXPRESSION`,
> whose target field is an `AFCOMPONENT` node with a minimal `{id,type:"AFCOMPONENT",name}` value).
> A **Value Commit / set-value rule is NOT a bare statement — it is an EVENT script** ("when this
> field *is changed*, then set its value"). Do **not** reuse the validate shape for it. The set-value
> AST MUST be:
> `STATEMENT.choice = EVENT_SCRIPTS` →
>   `items[0] = EVENT_CONDITION` whose `choice = EVENT_AND_COMPARISON` with
>     `[ {COMPONENT, value:{id,type:"TEXT FIELD|FIELD|AFCOMPONENT|STRING",name}}, {EVENT_AND_COMPARISON_OPERATOR, choice:{nodeName:"is changed"}}, {PRIMITIVE_EXPRESSION, choice:{nodeName:"STRING_LITERAL",value:null}} ]` →
>   `items[1] = {nodeName:"Then"}` →
>   `items[2] = BLOCK_STATEMENTS` → `BLOCK_STATEMENT.choice = SET_VALUE_STATEMENT` with
>     `items[0] = {nodeName:"VALUE_FIELD", value:{ id, displayName, type:"STRING", isDuplicate:false, displayPath:"FORM/Panel Title/Field Title/", name, parent:"$form.<panel-path>" }}` (a **full** component descriptor — NOT `AFCOMPONENT`, and NOT a minimal value),
>     `items[1] = {nodeName:"to"}` (lowercase `to`),
>     `items[2] = EXPRESSION.choice = FUNCTION_CALL` (your single-arg clientlib function).
> Top-level `script` is an **array** (`["myFn($field.$value)"]`) for set-value, and `eventName` is
> `"Value Commit"`. Getting `nodeName:"AFCOMPONENT"` / `type:"AFCOMPONENT"` / capital `"To"` / a bare
> `SET_VALUE_STATEMENT` under `STATEMENT.choice` is exactly what makes the editor render
> "Unknown Field - null" while validate rules on the same form still work. Diff your output against
> `account-opening-form`'s `fd:valueCommit` before finishing.

The rule of thumb per rule type:

- **No `fd:scripts` node at all.** Do not carry it over, not even empty. (`enrollment-form` has
  zero `fd:scripts` nodes.)
- **Validate rules → EMPTY `fd:events`.** `<fd:events jcr:primaryType="nt:unstructured"/>`. The
  runtime comes from the field's `validationExpression="fn(...) == true()"` attribute; the
  `fd:rules`/`fd:validate` AST is only for editor visibility. (This is the `enrollment-form` shape.)
- **Set-value rules (Value Commit / Initialize / Calculate) → `fd:events` MUST be POPULATED.**
  ⚠️ **This is the part `enrollment-form` does not show, and getting it wrong silently breaks the
  rule at runtime** (the field updates in the editor but nothing happens on the rendered form).
  The set-value RUNTIME comes from `fd:events`, NOT from the `fd:rules` AST. Author it as the
  guarded json-formula the Rule Editor itself produces (verified by reading
  `guideContainer.model.json`):
  - **Value Commit / change:** `change="[if(contains($event.payload.changes[].propertyName\, 'value')\, {value : myFn($field.$value)}\, {})]"`
  - **Initialize:** `initialize="[{value : myFn()}]"`
  - The `{value : ...}` object is what actually sets the field's value; the `if(contains(... 'value' ...))`
    guard prevents a change handler from re-triggering itself. Commas inside the `[...]` are
    escaped `\,` (FileVault multi-value). Keep the matching `fd:rules`/`fd:valueCommit` (or
    `fd:init`) AST too, for editor visibility.
- **Submit button's `fd:events`:** `click="[submitForm()]"` (matches `enrollment-form`'s submit).

> Net: empty `fd:events` is correct ONLY for validate. For any set-value/event rule, an empty
> `fd:events` means the rule never executes — populate it with the `{value:...}` runtime form and
> confirm via `guideContainer.model.json` that the field shows the `change`/`initialize` event.

> **A `change` handler that sets its OWN field loops unless the operation is idempotent.** The
> Foundation "Value Commit" fired once on user blur and did NOT re-fire on programmatic sets; the
> Core Components `change` event fires on EVERY value change, including the rule's own set. So:
> - **Idempotent self-set is safe** (e.g. `changeFirstLetterToUpperCase` — capitalising an
>   already-capitalised value yields the same value, so the loop converges in one cycle and the
>   value stops changing). Use the plain guard:
>   `change="[if(contains($event.payload.changes[].propertyName\, 'value')\, {value : myFn($field.$value)}\, {})]"`.
> - **Non-idempotent self-set loops** (e.g. `addTwoToValue` — each cycle adds 2 again, so AEM
>   runs it up to its recalc cap, ~9 cycles → the value jumps by ~18 instead of 2). Break the
>   loop by skipping the rule's own echo — the echo is the change whose delta equals exactly what
>   the rule adds:
>   `change="[if(($field.$value - $event.payload.changes[0].prevValue) != 2\, {value : addTwoToValue($field.$value)}\, {})]"`.
>   (Caveat: a user edit that happens to change the value by exactly the added constant is skipped
>   — acceptable for most fields; for stricter needs, apply the transform at submit time via
>   `create-submit-action` instead of a self-referential change handler.)

**Use the EXACT Rule-Editor property names + AST node types on `fd:rules`.** These are **not
guessable** and several differ from their human label — the only authority is what the AEM Forms
Cloud Service Rule Editor itself emits. Do **not** invent names (`fd:change`, `fd:initialize`,
`fd:calculate`) — those are wrong and the rule silently won't render/run.

| Rule (editor label) | `fd:rules` property | AST top node (`STATEMENT.choice.nodeName`) | `eventName` | Confidence | Runtime driver |
|---|---|---|---|---|---|
| Validate | `fd:validate` | `VALIDATE_EXPRESSION` | `Validate` | ✅ high | `validationExpression="fn(...) == true()"` **attribute** (AST editor-only); `fd:events` empty |
| Value Commit (set this field on change) | `fd:valueCommit` | `EVENT_SCRIPTS` → `VALUE_FIELD` | `Value Commit` | ✅ high | `fd:events` **`change="[...]"` (POPULATED)** — AST editor-only |
| Click (buttons) | `fd:click` | `EVENT_SCRIPTS` → `SUBMIT_FORM` | `Click` | ✅ high | `fd:events` `click="[submitForm()]"` |
| **Calculate (derived value)** | **`fd:calc`** (NOT `fd:calculate`) | **`CALC_EXPRESSION`** → `VALUE_FIELD` | `Calculate` | ✅ editor round-trip | model `rules.value` (editor-save/cloud only). ⚠️ **package deploy does NOT compile `fd:calc`→`rules.value`** (SDK: `rules: undefined`) — also wire the source-field `dispatchEvent` runtime (see cross-field block) |
| Initialize (set value on load) | `fd:init` | `EVENT_SCRIPTS` | `Initialize` | 🟡 observed (account-opening) | `fd:events` `initialize="[...]"` |
| Show / Hide | `fd:visible` | `SHOW_EXPRESSION` | `Visibility` | 🟡 observed (lab-test-form) | model `rules.visible` |
| (conditional) Change | `fd:change` | `EVENT_SCRIPTS` (with `IF`) | `Change` | ❓ **suspect** — appears once (lab-test-form, likely skill-generated); may be a distinct valid event OR an artifact. Round-trip before using |
| Enable / Disable | *not in repo* | — | — | ⬜ **round-trip needed** | model `rules.enabled` |
| Make Mandatory (required) | *not in repo* | — | — | ⬜ **round-trip needed** | model `rules.required` |
| Set value of **another** field | *not in repo* | — | — | ⬜ **round-trip needed** | likely `dispatchEvent(...'custom:setProperty'...)` |

> Confidence legend: ✅ verified (editor round-trip, or present across many forms and matching Adobe's
> shape) · 🟡 observed in 1–2 repo forms but not editor-round-trip-confirmed · ❓ suspect/conflicting
> · ⬜ not present in this repo — property name/AST unknown, **must** be obtained by round-trip.
>
> **The only authority is the editor.** To resolve any 🟡/❓/⬜ row: create that rule once in the Rule
> Editor on a scratch field, save, read the node back
> (`curl -u admin:admin .../<field>/fd:rules.json`), then copy the exact property + AST into source.
> This is how `fd:calc` was nailed after a hand-authored `fd:calculate`/`SET_VALUE_STATEMENT` proved
> wrong. **Never guess a property name or AST node type.**
>
> 🛑 **Three more invented nodes confirmed broken migrating `employee-training-request` from XDP — every
> one produced a rule the Rule Editor could not open/edit, even though the JSON parsed fine and the field
> still worked at runtime:**
> - **Compound AND conditions** (very common when migrating an XFA script with `if (a && b)`): don't
>   invent `AND_EXPRESSION` with bare `COMPARISON_EXPRESSION` siblings. The verified node — used inside
>   `SHOW_EXPRESSION`, `ACCESS_EXPRESSION`, `VALIDATE_EXPRESSION`, AND `CALC_EXPRESSION` alike — is
>   `BOOLEAN_BINARY_EXPRESSION`: each side wrapped in its own `CONDITION` node, an explicit
>   `OPERATOR`→`AND` node between them, `"nested":false` on the last `CONDITION`. See `create-form-rules`
>   SKILL.md's Pattern 10 for the full shape.
> - **Numeric literals in a condition**: don't use `NUMBER_LITERAL` with a bare JSON number (`"value":0`).
>   Use `NUMERIC_LITERAL` with the number encoded as a JSON STRING (`"value":"0"`), even when the compared
>   field is a genuine `NUMBER` type. `NUMBER_LITERAL` does not appear anywhere in the authoring clientlib.
> - **A click handler that must call a custom function AND a built-in action** (e.g. an XFA submit button
>   whose click script does `stampDate(); submitForm();`): don't invent `FUNCTION_CALL_STATEMENT` wrapping
>   a `FUNCTION_CALL` as a standalone `BLOCK_STATEMENT`. There is no such statement node — `FUNCTION_CALL`
>   only has special-cased handling INSIDE a value-returning `EXPRESSION` (a validate condition, a calc
>   then-value), never as a bare click-statement. Keep the `fd:click` AST down to just the built-in action
>   (`BLOCK_STATEMENTS` with a single `SUBMIT_FORM` `BLOCK_STATEMENT` — matches
>   `sports-event-registration`'s submit button exactly) and drive the REAL runtime behavior, including the
>   custom function call, purely through `<fd:events click="[myFunction(), submitForm()]"/>` — that
>   attribute needs no AST representation at all. The Rule Editor will show a simpler rule than what
>   actually runs; that's an acceptable trade-off for a rule that opens at all.

> **Set-value rules must call a SINGLE-ARGUMENT function (verified).** AEM compiles the
> `fd:rules` AST into the runtime model's `events` (`change`/`initialize`) — so the empty
> `fd:events` is fine and the AST drives runtime. BUT a **multi-argument** function call in a
> `SET_VALUE_STATEMENT` (e.g. `addToValue($field.$value, 2)` with a `NUMERIC_LITERAL` second
> param) **silently fails to compile** — the field ends up with no `change` event in the model
> and the rule never fires at runtime (validate rules tolerate multi-arg; set-value does not).
> **Bake any constants into the clientlib function so the rule is a one-argument call:**
> `addToValue($field.$value, 2)` → a function `addTwoToValue(value)` that returns `value + 2`,
> called as `addTwoToValue($field.$value)`. Verify by fetching the compiled model
> (`<form>/jcr:content/guideContainer.model.json`) and confirming each set-value field has its
> `change`/`initialize` event present.

> **Cross-field auto-fill (target ≠ source, or genuinely needs ≥2 inputs) → put `fd:calc` on the
> target for editor visibility, but drive the RUNTIME with `dispatchEvent` from the SOURCE fields'
> change handlers (verified).** A hand-authored/deployed `fd:calc` does NOT auto-compile to the
> runtime `model.rules.value` on a package deploy (only an editor save / cloud compile does), so a
> calculate like `middleName = buildMiddleName(firstName, lastName)` deployed via package produces
> **`rules: undefined`** in the model — the field never updates on its own.
> ⬜ **Untested refinement to try next time:** a genuinely EDITOR-SAVED cross-field `fd:calc` (confirmed
> live on `employee-training-request`'s `managerId`/`managerName`, auto-filled from `employeeId`) carries
> a `VALUE_FIELD`/`"to"`/`"When"`/`CONDITIONORALWAYS`/`BOOLEAN_BINARY_EXPRESSION` AST shape (script mirror
> a ternary: `if(cond, thenValue, $field)`) PLUS a plain `value="<expr>"` mirror attribute on `<fd:rules>`
> — with `fd:events` empty, no `dispatchEvent` at all. It is not yet verified whether hand-authoring that
> exact `value=` mirror (which this skill didn't know the name of until now) into a package-only deploy
> would compile where a bare AST-without-mirror did not — see `create-form-rules` SKILL.md's Pattern 3 for
> the full verified shape to copy if you want to test this. Until confirmed, keep using the proven
> `dispatchEvent` fallback below for package-only migrations. Keep the `fd:calc` (so the
> rule shows under the target field in the editor), AND additionally,
> in EACH source field's `fd:events` `change`, append a `dispatchEvent` to the target:
> `dispatchEvent($form.<panel>.<target>\, 'custom:setProperty'\, {value : myFn($form.<panel>.<a>.$value\, $form.<panel>.<b>.$value)})`.
> Multi-arg functions ARE fine here — the restriction is only on the `SET_VALUE_STATEMENT` *compile*,
> not on json-formula evaluated in a raw `fd:events` handler. This is the proven pattern (see
> `temperature-converter-form`, which uses `dispatchEvent(... 'custom:setProperty' ...)`). No loop
> results as long as the target has no handler that writes back to the sources. Verify in
> `guideContainer.model.json` that each source field's `events.change` array contains the
> `dispatchEvent`. (Trade-off: this runs via the source fields' events and is not an editor-visible
> rule on the target — acceptable; an editor-visible target rule would require a single-argument
> calculate, e.g. concatenation baked into a 1-arg function fed one combined value.)

> **Re-deploy hygiene.** FileVault `update`/merge installs do **not** delete properties absent
> from the new content, so iterating on rule names/events can leave orphaned properties
> (`fd:change`, stale `fd:events` keys) on the server. When re-deploying a form you have been
> changing, delete the server form node first
> (`curl -u admin:admin -F ":operation=delete" <form-path>`) so it is recreated cleanly, then
> verify the live `fd:rules`/`fd:events` keys match the file.
- **Custom expressions / functions.** Core Components Rule Editor uses json-formula and
  **global-scope** clientlib functions. Move any reusable validation/calculation logic into a
  per-form clientlib via **`create-form-clientlib`** and reference it. Remember json-formula
  booleans are **`true()` / `false()`**, not `true`. See the mandatory inline-JS rule below.

### Step 5c — Migrating an existing clientlib: keep its category name unchanged

If the source form already ships a clientlib (e.g. `/apps/clientlibs/clientlib-test` with
`categories="[cq.test]"`, referenced by `clientLibRef="cq.test"` on the `guideContainer`),
**preserve its category name verbatim** through the migration. Do **not** rename it to the
`{project}.forms.{formName}` convention (that convention is for *new* clientlibs created by
`create-adaptive-form`/`create-form-clientlib`, not for migrating an existing one).

- Keep the clientlib's `categories` exactly as the source (`cq.test` stays `cq.test`), and keep
  the source folder name (`clientlib-test`).
- Set the migrated form's `clientLibRef` to that **same** source category (`cq.test`) — the
  `clientLibRef` and the clientlib `categories` must match, and both equal the source value.
- You may still add `dependencies="[core.forms.components.runtime.all]"` for Core Components
  runtime/Rule-Editor availability, and `allowProxy="{Boolean}true"` — neither changes the
  category. Migrate the JS itself per Step 5a (extract inline rule JS into these functions).

> Renaming the category silently breaks the link: the form's `clientLibRef` then points at a
> category no clientlib provides, so the custom functions never load and every rule that calls
> them fails at runtime. Same name on both sides, equal to the source.

> **A clientlib MUST ship a `js.txt` manifest — without it NO JavaScript loads (verified).** A
> `cq:ClientLibraryFolder` only serves the JS named in `js.txt`; files sitting in `js/` with no
> manifest are **ignored entirely**, so every migrated rule function (`capitalizeFirstLetter`,
> `buildMiddleName`, validators, …) silently fails to load and every rule that calls them throws
> `ReferenceError` at runtime / the functions never appear in the Rule Editor. The compiled
> `guideContainer.model.json` still shows the rule expressions (it stores strings), so a model
> check does NOT catch this — you must verify the clientlib JS is actually served. Every migrated
> clientlib needs **all three** parts:
> - `/apps/clientlibs/{name}/.content.xml` — `jcr:primaryType="cq:ClientLibraryFolder"`, `allowProxy="{Boolean}true"`, `categories="[...]"`, `dependencies="[core.forms.components.runtime.all]"`
> - `/apps/clientlibs/{name}/js.txt` — the manifest:
>   ```
>   #base=js
>
>   functions.js
>   ```
>   (`#base=js` then every JS file under `js/`, in load order — add `css.txt` + `#base=css` if there is CSS)
> - `/apps/clientlibs/{name}/js/functions.js` — the actual functions
> After writing the files, **grep the clientlib folder for `js.txt` and confirm it lists every file
> in `js/`**; after deploy, fetch `/etc.clientlibs/{...}/{category}.js` (or the proxied clientlib
> URL) and confirm `functions.js`'s content is actually returned — not just HTTP 200 on the folder.

### Step 5a — Replace inline JavaScript with named function calls (MANDATORY)

On-prem Foundation forms store **complete JavaScript definitions** inside rule scripts
(`fd:scripts` / `guideScript` / `fd:rules` script bodies) — e.g. a field's "Value Commit" or
"Initialize" handler is raw ECMAScript like
`if (this.value !== "") { this.value = guidelib.util.GuideUtil._addValues(this.value, 2); }`.

**AEM as a Cloud Service has no code editor** — the Adaptive Form Rule Editor is *visual only*.
A rule whose body is raw JS cannot be opened, displayed, or edited there; at best it is opaque,
at worst it silently does not run (many on-prem APIs like `guidelib.*` / `GuideUtil` do not
exist on cloud). Therefore, for **every** field rule that contains inline JS, you **MUST**:

1. **Extract the logic into a named, global-scope function** in the form's clientlib
   (`create-form-clientlib`), JSDoc-annotated with an `@name` so it appears under
   "Function Output" in the Rule Editor. Fold any inline guards (null/empty checks, `if`
   wrappers) **into** the function so the rule body is a single call.
2. **Re-implement any on-prem-only API** the inline JS used. `guidelib.*`, `GuideUtil`,
   `guideBridge`, `window.guideBridge`, jQuery, and direct DOM access do **not** exist on the
   cloud runtime — replace them with plain JS inside your new function.
3. **Replace the rule body with a single function call** expressed through the visual Rule
   Editor representation:
   - Validate → `validationExpression="myFn($field.$value) == true()"` + the `fd:validate` AST.
   - Set Value / Value Commit / Calculate → a `SET_VALUE_STATEMENT` whose expression is a
     `FUNCTION_CALL` to your function (event runtime mirrored in `fd:events`, e.g.
     `change="[myFn($field.$value)]"`, `initialize="[myFn()]"`).
   - ⚠️ **Escape commas inside `fd:events` expression arrays.** `fd:events` values use the
     FileVault multi-value form `[...]`, so a literal comma in a multi-arg call is treated as a
     value separator and silently splits the expression. Write
     `change="[addToValue($field.$value\, 2)]"` (escaped `\,`), **not**
     `change="[addToValue($field.$value, 2)]"` — the latter deploys as the broken two-value
     array `["addToValue($field.$value"," 2)"]`. (The `fd:rules` AST already escapes every comma
     as `\,`; apply the same to `fd:events`.)
   - ⚠️ **The same FileVault multi-value escaping applies to embedded QUOTES inside an `fd:rules`
     JSON blob's `"script"` (or any string) field — and here it needs 2 backslashes, not 1.**
     Jackrabbit's DocView importer strips ONE backslash from every `\X` sequence in a multi-value
     (`[...]`) property, no matter what `X` is — the exact mechanism that makes `\,` survive as a
     literal comma. A migrated script that needs an embedded quote (e.g.
     `field.$value != ""`) must therefore be written in the XML source as
     `&quot;script&quot;:&quot;field.$value != \\&quot;\\&quot;&quot;` (2 backslashes per quote) —
     1 backslash survives import as a bare, unescaped quote, corrupting that rule's JSON. This is
     NOT caught by decoding XML entities and running `JSON.parse` on the raw source text (that
     check passes with 1 backslash and fails with 2, which is backwards) — you must simulate the
     DocView unescape too: decode XML entities, THEN strip one backslash per `\X` pair, THEN
     `JSON.parse`. The failure mode is severe: one malformed rule aborts the Rule Editor's whole
     tree walk, so the **entire form's Form Objects list goes empty** (not just the broken rule),
     with `Uncaught SyntaxError ... at position N` from `_traverseAndValidateRules` in the browser
     console — easy to misdiagnose as a schema/authoring problem instead of an escaping bug.
   - A rule that only assigns a **constant** (e.g. `this.value = "Hassan"`) becomes a plain
     "Set Value Of … to <literal>" (a `STRING_LITERAL` / `NUMERIC_LITERAL`) — no function needed.
4. **Leave no raw JS in the form XML.** After migration, grep the form's `.content.xml`: it must
   contain **no** `this.value`, `guidelib`, `GuideUtil`, `function `, `var `, `if (`, `=>` inside
   rule scripts. Every rule `script` must be a function call, a comparison of a function call to
   `true()`, or a literal assignment.

**Worked example (the on-prem "age + 2 on commit" rule):**

| | On-prem (inline JS) | Cloud (function call) |
|---|---|---|
| Rule body | `if (this.value !== null && this.value !== "") { this.value = guidelib.util.GuideUtil._addValues(this.value, 2); }` | `addToValue($field.$value, 2)` |
| Clientlib | — | `function addToValue(value, increment) { if (value === null || value === undefined || value === "") return value; ... return Number(value) + Number(increment); }` |
| Visible in visual editor? | ❌ no (raw JS, no code editor) | ✅ yes (Set Value Of → addToValue) |

The empty-guard `if` from the on-prem script is absorbed into `addToValue`, so the rule reduces
to one clean, editor-visible call.
- **Field-level validation.** If the source enforced formats/lengths/ranges, recreate them as
  the `validationExpression="<fn>(...) == true()"` + `validateExpMessage` pattern plus the
  `fd:validate` AST, per `create-form-clientlib`. This is also the project's standing
  per-form-clientlib convention — see [[clientlib-per-form-convention]].
- **Submit action.** Map the legacy submit config to a Core Components action. Built-in REST
  endpoint submit uses `actionType="fd/af/components/guidesubmittype/restendpoint"` on the
  `guideContainer`. If the source used a custom Foundation submit servlet, re-create it with
  **`create-submit-action`** (OSGi service + JCR node) — a legacy servlet will not be invoked
  by a Core Components form.
- **Prefill.** If the legacy form prefilled from a servlet/datasource, re-create it with
  **`create-prefill-service`**.

---

## Step 6 — Migrate the theme (EXACT reproduction from source truth — never invent a palette)

If the source carried its own theme and the user wants to keep that look, the migrated form must look
**the same as the on-prem original**. A `/libs/...` or on-prem theme path does not exist on cloud, and an
on-prem theme is **not** raw CSS — it is a Foundation **theme-editor JCR style tree**. You therefore
**re-create** the look on the Core Components DOM. But re-create ≠ approximate: **derive every value from
the source tree.** Never hand-pick a "close enough" palette, and never leave a `themeRef` pointing at `/libs`
or `/content`.

> ⚠️ **The #1 theme-migration failure: approximating the palette / dropping the form's border & background.**
> The on-prem theme's colours, the form/card **border**, and the page/form **background** are all stored as
> exact values in the theme tree. Guessing them (e.g. defaulting to a generic Spectrum-blue "Canvas" look)
> produces a form that is visibly NOT the original — wrong colours, and a missing card border/background are
> the two most-reported regressions. Extract, don't guess.

### 6.1 — Find and EXHAUSTIVELY parse the source theme tree (the ground truth)

1. **Locate the theme the form references.** The form's `guideContainer` (or the DAM guide asset) carries a
   `themeRef` such as `/content/dam/formsanddocuments-themes/themeLibrary/<theme>`. In the package, that theme's
   values live in its **theme-json rendition**:
   `jcr_root/content/dam/formsanddocuments-themes/.../<theme>/_jcr_content/renditions/theme-json/.content.xml`
   (may also appear as a `rawCss` property for some themes — read whichever exists).
2. **Walk EVERY component style node and dump the raw properties.** The tree is a set of `af_*` nodes under
   `jcr:content/components/af_guideContainer/...`, each holding style arrays like
   `default_x0023_default="[background:#ffffff,border-color:#cccccc,...]"` (`_x0023_` is `#`; states are
   `#default`, `#hover`, `#focus`, `#active`, `#error`, `#success`, `#disabled`; breakpoints are `phone`/`tablet`).
   List every `af_*` node, then read the specific ones below and **copy the hex/rgb/rgba/px/rem values verbatim**.
   Dump what you find so the ground truth is visible before you write CSS.

### 6.2 — The style nodes that carry the values that matter (map to Core Components DOM)

| On-prem theme node | Carries | Core Components target (`cmp-adaptiveform-*`) |
|---|---|---|
| `af_form` | **page/form background** (often a colour, e.g. `rgb(...)`) | `.cmp-adaptiveform-container` background |
| `af_responsivepanel` / `af_panel` | **THE FORM BORDER** (border-width/style/color), **border-radius**, panel text colour, background, box-shadow | `.cmp-adaptiveform-container__form` (the card) |
| `af_page` | font-family, line-height, page margins | `.cmp-adaptiveform-container` |
| `af_formtitle` | title colour + font-size | `.cmp-title__text` / the heading text |
| `af_fieldlabel` | **label colour**, font-size, weight | `.cmp-adaptiveform-*__label` |
| `af_widgetAndText` | input **border/radius/text colour/background/height** | `.cmp-adaptiveform-*__widget` |
| `af_field` | per-field **validity accent** (default/`#success`/`#error` left border + error bg) | field wrapper `[data-cmp-valid]` |
| `af_fielderrormessage` | error text colour | `.cmp-adaptiveform-*__errormessage` |
| `af_radiobuttonitemlabel` / `af_radiobuttonwidget` | radio label colour, radio radius | `.cmp-adaptiveform-radiobutton__option*` |
| `af_dropdownlistwidgetandtext` | select border/radius/background | `.cmp-adaptiveform-dropdown__widget` |
| `af_submitbutton` | submit button bg/text/hover (usually SOLID) | `.cmp-adaptiveform-button__widget` (submit) |
| `af_button` / `af_resetbutton` | generic/reset button bg/text/border/radius/hover (often OUTLINE) | `.cmp-adaptiveform-button__widget[type="reset"]` |
| `af_checkboxlabel` | checkbox label colour | `.cmp-adaptiveform-checkbox*__label` |
| `af_datewidgetlabel` | date sub-label colour | datepicker sub-labels |
| `af_footer` / `af_footertext` / `af_header` / `af_headertext` | footer/header bg + text | footer/header (if present) |

> **Do not stop at colours.** The **border** (width, style, colour, radius) and **background** on
> `af_form` and `af_responsivepanel`/`af_panel` are the form's card look — reproduce them exactly, including
> `border-radius` and any `box-shadow` (empty `shadowContainer` = no shadow). Also carry the field validity
> **left-accent bar** (`af_field` `#success`/`#error`) — it is a signature of these themes.

### 6.3 — Regenerate ALL THREE theme carriers with the extracted values

Every value goes, byte-for-byte, into all three:
1. **`/apps/fd/af/themes/{project}-{theme}/theme.zip`** — `theme.css` (unscoped) + `theme.js`; rebuild the zip.
2. **The per-form clientlib CSS** (`create-form-clientlib`) — the SAME rules with **`!important`** on the
   layout-critical properties. This is the reliable render driver (see 6.4).
3. **The DAM theme-json** (`/content/dam/formsanddocuments-themes/{project}/{theme}/.../theme-json/.content.xml`)
   — the `af_*` component-style nodes carrying the extracted values (so the theme is editable and compiles on cloud).

Keep the theme string identical across form page / DAM guide asset / conf context (see Step 4). You may map to
an existing `/apps/fd/af/themes/{project}-{theme}` theme ONLY if the user accepts that look; to preserve the
original, extract-and-recreate as above.

### 6.4 — The theme-servlet aggregate cache (local SDK) — verify the RENDER, not just the file

On the local SDK, the AEM Forms theme-servlet **aggregate** (`…/{form}.theme/_default/theme.css`) is
**JVM-cached** and frequently does **not** refresh on a content redeploy — deleting the form/theme nodes does
not clear it short of a restart. This looks like "my theme change didn't deploy."

- This is exactly why the styling is **also** served through the per-form clientlib CSS with `!important`: a
  non-`!important` stale aggregate is deterministically overridden by the fresh `!important` clientlib, so the
  render is correct now, and on cloud the aggregate compiles fresh from the new `theme.zip` (both agree there).
- **Verify by value, not by HTTP 200:** after deploy, fetch the served **clientlib CSS**
  (`/etc.clientlibs/clientlibs/{name}.css`) and confirm the extracted values (the exact background `rgb(...)`,
  the `Npx solid #RRGGBB` border, the `border-radius`, the label colour) are literally present; fetch the
  **server `theme.zip`** and confirm its `theme.css` contains them too. If only the aggregate is stale, the
  `!important` clientlib still renders correctly — state this in the report rather than chasing the cache.
- Re-extract from source and re-apply if the user reports the theme is wrong — do not tweak toward a guess.

---

## Global XML conventions

Apply in every file you write (same as the other skills):

- **Booleans** use typed syntax: `enabled="{Boolean}true"` ✓ — never `enabled="true"` ✗.
- **Multi-value attributes** use bracket syntax: `enum="[a,b,c]"` ✓.
- **Namespaces** — every prefix used in a file (`jcr`, `nt`, `cq`, `sling`, `fd`, `dam`, `xmp`)
  must be declared on the root `<jcr:root>` node, or package install fails with a cryptic XML
  parse error. Legacy exports often omit `fd`.
- **Node names** — PascalCase for panel/field nodes; the `name` attribute matches the node name.
  Keep the source's existing names to preserve data bindings unless the user asks to rename.

---

## Failure symptoms and their causes (post-migration)

| Symptom | Cause | Fix |
|---|---|---|
| Component renders blank / 404 on resourceType | A `fd/af/components/...` type was left un-remapped, or a proxy doesn't exist | Remap to `{project}/components/adaptiveForm/...`; create the proxy if missing |
| Form not visible in Forms & Documents | Artifact 2 missing, or `type="guide"`/`guide="1"` absent | Create artifact 2 with both markers |
| Form opens with an error | `cq:template` still points at the legacy/on-prem path | Set it to the target cloud template |
| Form opens completely unstyled | Artifact 3 missing, `sling:configRef` missing trailing `/`, or theme strings mismatch | Create artifact 3; fix the slash; make all three theme strings identical |
| Migrated theme colours wrong / form border or background missing | Palette approximated instead of extracted; `af_form` background and `af_responsivepanel`/`af_panel` border+radius not carried over | Re-parse the source theme-json (Step 6.1–6.2), copy every colour + the border (width/style/colour/radius) + background VERBATIM into all three carriers (Step 6.3) |
| Theme edited & redeployed but render unchanged on local SDK | AEM Forms theme-servlet aggregate (`…/{form}.theme/_default/theme.css`) is JVM-cached and did not refresh | Serve the styling via the per-form clientlib CSS with `!important` (overrides the stale aggregate); verify the served clientlib CSS/`theme.zip` contain the values; cloud compiles fresh (Step 6.4) |
| CSS/JS 404 at runtime | `prefixPath` in artifact 3 does not match the migrated form's `/content/forms/af/{appFolder}/{formName}` path | Correct `prefixPath` to the form's folder segment |
| Placeholders missing / a radio or field loses its preselected value | Foundation attribute names kept — CC ignores `placeholderText` and `_value`, passing them through the model as unknown keys | Rename `placeholderText` → `placeholder` and `_value` → `default` on the migrated fields (Step 3 attribute-migration); confirm `"placeholder"`/`"default"` in `guideContainer.model.json` |
| Broken image / missing logo on the rendered form | The DAM image was not migrated, or its reference (`fileReference`/`src`/`imageRef`/`url(...)`) still points at the source/on-prem path | Copy the DAM asset to its cloud path with filter coverage and rewrite every reference (Step 4A.2–4A.4) |
| Text renders in a fallback font | Font files not migrated, or `@font-face`/`url(...)` points at an on-prem absolute URL | Copy the fonts into their carrier (theme.zip/clientlib/DAM) and keep the `src` relative/repointed to the cloud path (Step 4A.2) |
| Submit produces no PDF / Document of Record | DoR template not migrated, `dorType` downgraded to `none`, or `dorType`/`dorTemplateRef` set on `guideContainer` instead of the DAM guide-asset's `metadata` node | Copy the DoR template as a real `dam:Asset`, set `dorType="select"`+`dorTemplateRef` on the **DAM guide asset's `jcr:content/metadata`** (Form Properties), or wire via `create-workflow` (Step 4A.2) |
| Fields don't bind / Bind Reference blank even after `dataRef` fix | The schema/bindings file was not migrated, so `dataRef`/`schemaRef` resolves against nothing | Copy the schema to `/content/dam/formsanddocuments/schema/...` with filter coverage, or regenerate with `generate-schema` (Step 4A.2) |
| Embedded fragment renders empty / 404 | The referenced fragment (page + DAM fragment asset) was not migrated, or `fragmentPath` still points at the source path | Migrate the fragment as its own form (Steps 2–6), keep `fragmentPath` on the cloud path, add filter coverage (Step 4A.2) |
| Rules don't fire | `guideRule` left in Foundation format, or `== true` instead of `== true()` | Re-author via `create-form-rules`; use `true()` |
| Rule is blank / not editable in the visual Rule Editor; logic doesn't run | Inline JS left in the rule body (no code editor on cloud), or an on-prem API (`guidelib.*`, `GuideUtil`) was kept | Extract the JS into a named global clientlib function and call it from the rule (Step 5a) |
| Submit does nothing | Legacy submit servlet not invoked, or submit button missing `fd:rules`/`fd:events` | Re-create via `create-submit-action`; add the click rule/event pair |
| Conf deployed but no effect | Filter entry for `/conf/forms/{appFolder}/{formName}` missing | Add the filter line, redeploy |
| Re-deployed form keeps old/removed rules, fields, or properties (changes not fully replacing) | `ui.content` filter is `mode="update"` (merge) and `ui.content` installs as an embedded sub-package of `all`, so the install never deletes nodes/properties absent from the package | Delete the form's server roots before the build (Step 7.2), then install — the package recreates them clean |
| Package install fails with XML error | A used namespace prefix isn't declared on the root | Add the missing `xmlns:...` |

---

## Pre-flight checklist

- [ ] All 7 inputs confirmed; `{appFolder}` is the form/DAM/conf folder segment (alias of `{project}` when relocating), shared identically across all three artifacts
- [ ] If a `.zip` package: unpacked; `jcr_root/` + `filter.xml` present; every packaged root enumerated and mapped to its repo module (path preserved, module prefix added); binaries copied as-is and their metadata + references migrated per Step 4A; nothing packaged silently dropped
- [ ] Source form read and classified (Foundation / un-proxied Core / proxied Core); migration plan shown and approved
- [ ] Every `sling:resourceType` remapped to `{project}/components/adaptiveForm/...`; no `fd/af/components/...` remains
- [ ] Each used proxy component verified to exist under `apps/{project}/components/adaptiveForm/` (or flagged for `create-form-component`)
- [ ] All fields, names, titles, and field order preserved; every `bindRef`/`bindReference` attribute renamed to `dataRef` with its value converted from slash-path to JSONPath (`/a/b` → `$.a.b`), `fd:formDataRef` kept for FDM forms — grep the form for `bindRef` and expect zero matches, and confirm every `dataRef` value starts with `$.`
- [ ] Every set-value / Value-Commit rule uses the `EVENT_SCRIPTS` → `VALUE_FIELD` AST shape (not a bare `SET_VALUE_STATEMENT`/`AFCOMPONENT`); diffed against `account-opening-form`
- [ ] Foundation field attributes renamed to CC property names: **`placeholderText` → `placeholder`** and a field's default **`_value` → `default`** (radio/dropdown/checkbox/text); a `guidetextdraw`'s `_value` HTML went to the `text`/Title `value`, not `default`. Verify in `guideContainer.model.json` that each field shows `"placeholder"`/`"default"` — a leftover `placeholderText`/`_value` is an unknown key that renders nothing
- [ ] Booleans typed (`{Boolean}true`); enums bracketed; `<cq:responsive>/<default>` on every field & panel
- [ ] `aria-label` added to every migrated field/panel/button (legacy exports lack it) — grep the migrated form and expect every field to carry one; CAPTCHA remapped to a CC captcha proxy (cloud config flagged) and any source Document of Record preserved (not downgraded to `dorType="none"`)
- [ ] Artifact 1 form page: target `cq:template`, `sling:configRef` with trailing `/`, correct `themeRef`, page `sling:resourceType`
- [ ] Artifact 2 DAM guide asset created with `type="guide"` + `guide="1"` + matching `themeRef`
- [ ] Artifact 3 conf context created; `prefixPath` uses the form's `{appFolder}` folder segment (matches the form path); `themeArtifact`/`siteTemplatePath` match the theme string
- [ ] **Assets (Step 4A): the WHOLE package migrated, not just the form page.** Asset inventory built from `filter.xml` + a grep of the form/theme/clientlib/schema for `fileReference`/`src`/`imageRef`/`dorTemplateRef`/`schemaRef`/`fragmentPath`/`url(...)`/`@font-face`; every DAM image/logo, font, Document-of-Record template, schema/bindings file, referenced fragment, and icon/SVG copied to its cloud path (binary as-is, wrapping `.content.xml` migrated); every reference to each rewritten to the cloud path; grep of the migrated form/theme/clientlib shows NO surviving on-prem host, `/libs/` asset ref, or source-only path; each asset root covered by a filter; referenced fragments migrated as their own forms (not inlined); any non-migratable asset surfaced to the user (recorded as `assets_flagged`), never silently dropped
- [ ] Artifact 4: EVERY path the migration writes is covered by a `filter root` in the correct module's `filter.xml` (form, DAM asset, schema, referenced fragments, form-referenced DAM media/DoR, conf in `ui.content`; clientlib, new components, submit actions, themes in `ui.apps`); any uncovered path has a new `<filter root>` added — an uncovered node is silently never deployed
- [ ] All theme strings identical across artifacts 1, 2, 3
- [ ] Rules re-authored via `create-form-rules`; custom functions moved to a clientlib (`create-form-clientlib`); json-formula uses `true()`
- [ ] If migrating an existing clientlib (Step 5c): its `categories` kept **identical to the source** (not renamed to `{project}.forms.{formName}`); the form's `clientLibRef` equals that same source category; clientlib `categories` and form `clientLibRef` match
- [ ] Clientlib ships a `js.txt` manifest (`#base=js` + every file in `js/`): grep the clientlib folder for `js.txt` and expect it to exist and list `functions.js`; without it NO JS loads and every rule function throws `ReferenceError`. After deploy, fetch the proxied clientlib JS and confirm `functions.js`'s content is returned (not just a 200 on the folder)
- [ ] No inline JS remains in any rule (Step 5a): every rule body is a named function call or a literal; on-prem APIs (`guidelib.*`, `GuideUtil`, jQuery, DOM) re-implemented; grep of the form `.content.xml` shows no `this.value` / `guidelib` / `function ` / `var ` / `if (` inside rule scripts
- [ ] Rule structure uses the EXACT `fd:rules` property names + AST node types from the verified table above: `fd:validate` (VALIDATE_EXPRESSION) / `fd:valueCommit` (EVENT_SCRIPTS→VALUE_FIELD) / `fd:calc` (CALC_EXPRESSION, **NOT** `fd:calculate`) / `fd:click` (EVENT_SCRIPTS→SUBMIT_FORM); **no `fd:scripts` node anywhere**; no invented names (`fd:change`/`fd:calculate`/`fd:initialize`); any unverified rule type (e.g. `fd:init`) round-tripped through the editor first
- [ ] Every embedded quote inside an `fd:rules` JSON blob's string fields (e.g. a `script` field doing `field.$value != ""`) uses 2 backslashes in the XML source (`\\&quot;`), not 1 — verify by simulating the JCR DocView unescape (XML-decode, strip 1 backslash per `\X`, THEN `JSON.parse`) rather than parsing the raw source text directly; 1 backslash survives import as a bare quote and blanks the Rule Editor's entire Form Objects tree, not just that rule
- [ ] Every migrated compound `if (a && b)` XFA condition uses `BOOLEAN_BINARY_EXPRESSION` (not `AND_EXPRESSION`), every migrated numeric literal uses `NUMERIC_LITERAL` with a JSON-string value (not `NUMBER_LITERAL` with a bare number), and no click/event handler invents a `FUNCTION_CALL_STATEMENT` to call a custom function as a standalone action — all three are confirmed-invented nodes with no Rule Editor visitor (see the warning above)
- [ ] Runtime present, not just the editor AST: each `fd:valueCommit` has a populated `fd:events` `change`; each `fd:calc` target ALSO has a `dispatchEvent` runtime from its source fields (package deploy does not compile `fd:calc`→`rules.value`); submit has `fd:events` `click="[submitForm()]"`
- [ ] Deployment deletes the form's server nodes first (Step 7.2) so changed nodes are truly replaced (no orphaned rules/properties/fields linger); a `update`-filter package install alone does NOT replace, even via the `all` package — verified by re-installing and confirming an injected orphan is gone
- [ ] Submit action mapped (built-in) or re-created (`create-submit-action`); prefill re-created if present (`create-prefill-service`)
- [ ] Theme: source theme tree (`.../themeLibrary/<theme>/.../theme-json`) EXHAUSTIVELY parsed and its raw values dumped; every colour + the form **border** (width/style/colour/radius) + page/form **background** copied VERBATIM (no invented/approximated palette), mapped to the `cmp-adaptiveform-*` DOM per Step 6.2, and written into ALL THREE carriers (theme.zip `theme.css`, per-form clientlib CSS with `!important`, DAM theme-json `af_*` nodes); no `/libs` or `/content` theme refs. After deploy, the served clientlib CSS and server `theme.zip` were fetched and confirmed to literally contain the extracted background/border/label values (SDK theme aggregate may stay cached — the `!important` clientlib overrides it; note it rather than guessing)
- [ ] All namespace prefixes declared on each file's root node
- [ ] Deployment run automatically (Step 7): AEM reachability pre-checked, `mvn clean install -PautoInstallSinglePackage` executed, BUILD SUCCESS confirmed and reported (or failure diagnosed and re-run)
- [ ] UI parity check run (Step 8) via `test-form-ui`: migrated form URL compared against the original (screenshot or legacy URL); judged on STRUCTURAL parity (zero Critical findings — same fields/labels/order/types/sections + submit), NOT pixel identity (theme remap intentionally changes colour/spacing — pixel gate advisory); `.claude/agents/runs/{YYYY-MM-DD}-{formName}/test/sentinel/test-form-ui-report.md` produced and folded into the outcome; any Critical regression fixed and re-tested. If no original reference exists, captured the migrated form as a record and noted no baseline

---

## Step 7 — Deploy to the AEM server automatically (standalone only)

> **Pipeline mode (delegated by `formwright`): SKIP this step — author the migrated artifacts only.**
> Deployment is centralized in the `forgemaster` lead, which runs the single authoritative build+deploy
> after Groundsmith (AGENTS.md → "Deployment is centralized in Forgemaster"). The pre-flight checks below
> (filter coverage, AEM reachability) still matter — run them — but do **not** run the `mvn`
> build/deploy here when part of a delivery pipeline. Only deploy here when invoked **standalone**.

When run standalone, a migration is not finished until the result is installed on the running AEM
instance. After all artifacts are written and the pre-flight checklist passes, **automatically run the
deployment** — do not just print the command and stop.

**1. Pre-checks (fast, before the long build):**
- **Filter coverage (do this first).** For every JCR path this migration created or modified
  (form page, DAM guide asset, conf, schema/bindings, clientlib, any new proxy components, submit
  actions, theme), confirm it is a descendant of a `filter root` in the **correct module's**
  `META-INF/vault/filter.xml` (`ui.content` for `/content` & `/conf`, `ui.apps` for `/apps`). Any
  path not covered is silently dropped from the package — add the missing `<filter root="...">`
  entry now (Artifact 4) so the change actually ships. Re-run this check whenever the migration
  starts writing to a new path.
- Confirm Maven is available: `mvn -v`.
- Confirm an AEM author instance is reachable (default `http://localhost:4502`). A quick probe:
  ```powershell
  try { (Invoke-WebRequest -Uri "http://localhost:4502/libs/granite/core/content/login.html" -UseBasicCredentials -TimeoutSec 5).StatusCode } catch { "AEM-NOT-REACHABLE" }
  ```
  If it is not reachable, **stop before building** and tell the user to start the SDK/author
  instance (or pass the correct host/port) — a build that cannot install wastes minutes.

**2. Delete the migrated form's existing server nodes FIRST (MANDATORY — guarantees clean replace).**
A standard package install does **not** reliably replace an already-deployed form: the project's
`ui.content` filter uses `mode="update"` (merge), which adds/updates properties but never deletes
ones absent from the package, and AEMaaCS installs `ui.content` as an embedded sub-package of the
`all` package — so even a `replace`-mode filter on the form path does not take authoritative
effect. The result is that **changed nodes are not fully replaced**: orphaned rules, stale
`fd:events`/`fd:rules` properties, and removed fields linger on the server.

The reliable fix is to delete the form's own JCR roots immediately before the build, so the
package recreates them from scratch (a true replace). Delete the three per-form roots (the
clientlib in `ui.apps` already replaces cleanly via its `/apps/clientlibs` filter):

```powershell
$base = "http://localhost:4502"; $cred = "admin:admin"
foreach ($p in @(
    "/content/forms/af/{appFolder}/{formName}",
    "/content/dam/formsanddocuments/{appFolder}/{formName}",
    "/conf/forms/{appFolder}/{formName}")) {
  curl.exe -s -u $cred -F ":operation=delete" "$base$p" -o NUL -w "delete $p : HTTP %{http_code}`n"
}
```

This is safe — every deleted node is recreated by the package in the next step. Do **not** delete
the broad parent roots (`/content/forms/af/{appFolder}`); only the specific form's nodes. Skip a
path that returns 404 (first-time migration — nothing to delete yet).

**3. Run the deployment.** A migration almost always touches **both** `ui.apps` (the per-form
clientlib with the migrated custom functions) **and** `ui.content` (the form page, DAM asset,
conf, schema). Therefore run the **full** single-package build so the aggregated `all` package
installs everything together:

```bash
mvn clean install -PautoInstallSinglePackage
```

- This installs to `http://localhost:4502` as `admin` by default. For a different target, pass the
  archetype properties, e.g.
  `mvn clean install -PautoInstallSinglePackage -Daem.host=<host> -Daem.port=<port> -Dvault.user=<user> -Dvault.password=<pwd>`.
- Use `-pl ui.content` **only** when the migration produced no `ui.apps` changes at all (rare —
  the clientlib lives in `ui.apps`). When in doubt, run the full build.
- Run it as a background/long-running command (builds take minutes) and surface BUILD SUCCESS /
  BUILD FAILURE. On failure, read the Maven output, fix the cause (often a malformed `.content.xml`
  or filter gap), and re-run — do not leave the form half-installed.

**4. Report the outcome** to the user: the nodes deleted, the exact command run, the target
instance, and BUILD SUCCESS or the failure reason.

### Post-deploy verification

After BUILD SUCCESS, verify on the running instance (default `http://localhost:4502`):

1. Forms & Documents lists the migrated form.
2. It opens in the AF editor without errors; every field renders (no 404 placeholders); each
   migrated rule is visible/editable in the **visual Rule Editor** (no opaque inline JS — Step 5a).
3. The published URL applies the theme; migrated rules behave as they did on the source.
4. Submitting empty triggers required-field validation; valid input submits to the mapped action.
5. Spot-check data bindings (`fd:formDataRef`) still resolve against the schema/FDM.
5a. **Assets render (Step 4A).** Every migrated image/logo loads (no broken-image icon / DAM 404),
   text uses the migrated fonts, an embedded fragment renders its fields, and — if the form has a
   Document of Record — a submit produces the PDF. Open the browser network tab and confirm no
   asset request 404s or points at an on-prem host.
6. **Confirm event rules compiled into the runtime model.** Fetch
   `<form>/jcr:content/guideContainer.model.json` and verify each set-value field carries its
   `change` / `initialize` event (e.g. age shows `"change":[...addTwoToValue($field.$value)...]`).
   A field whose set-value rule is missing from the model has a multi-arg call that failed to
   compile — convert it to a single-argument function (see Step 5b) and redeploy.

---

## Step 8 — UI parity check against the original (MANDATORY when a reference is available)

A migration must not silently drop or reorder fields, change labels, or lose a section. The most
reliable way to catch that is to **compare the migrated form's rendered UI against the original
form** — so after a successful deploy, invoke the **`test-form-ui`** skill with the original form as
the reference.

- **Form under test (`{formUrl}`):** the migrated cloud render,
  `http://localhost:4502{formsContentRoot}/{appFolder}/{formName}.html`.
- **Reference:** the **original** form — whichever you have:
  - `{referenceUrl}` — the legacy/on-prem form's URL if it is still reachable; or
  - `{referenceImage}` — a screenshot of the original (ask the user for one during Step 1 if the
    legacy instance is gone). A package-only migration usually has no live original, so the user's
    screenshot is the reference.
- If **no** reference of the original exists at all, you cannot do parity comparison — instead run
  `test-form-ui` with the migrated URL to capture a screenshot of the result as a record, and state
  in the report that no original baseline was available.

> ⚠️ **Structural parity is the hard gate; colour/border parity is expected too when Step 6 was done right.**
> Because Step 6 **extracts the theme's exact values from the source** (not an approximated palette), the
> migrated colours, form **border**, background, and button styling **should match** the original — a wrong
> colour or a missing/incorrect form border is a **theme-extraction defect to fix (re-do Step 6)**, NOT an
> "intended theme change." Only genuine DOM/rendering-engine differences (Foundation vs Core Components box
> model — minor spacing, font hinting, radio/checkbox chrome) are acceptable residuals. For the parity check:
> - The **pixel-diff** gate may still read high due to those box-model residuals and font rendering — treat
>   the pixel % as **advisory**, but investigate any large colour-block diff (it usually means a background/
>   border value was missed in Step 6, not a box-model residual).
> - Define migration **PASS = zero Critical findings** from the vision pass: the migrated form has the
>   **same fields, labels, order, field types, panels/sections, the submit button, AND the same key colours +
>   form border/background** as the original.
> - **Critical findings to fix:** a field/label/section that vanished, changed, or reordered; a field type
>   that changed (radio→dropdown); **OR a colour / form-border / background that does not match the source**
>   → for the first three, fix the form XML (Steps 3–5); for theme mismatches, re-extract and re-apply
>   (Step 6). Re-deploy + re-test until both structural and colour/border parity are clean.

Fold the resulting `.claude/agents/runs/{YYYY-MM-DD}-{formName}/test/sentinel/test-form-ui-report.md` into the migration
outcome you report to the user, listing any structural (Critical) differences AND any colour/border/background
mismatches still to fix (the latter mean Step 6 must be re-done from source), noting only genuine
box-model/font-rendering residuals as accepted.

---

# Path B — Migrate Adobe LiveCycle / AEM Forms on JEE to AEM Forms as a Cloud Service

Path A (Steps 1–8) migrates AEM Adaptive Form *content*. Path B migrates a whole **Adobe LiveCycle
ES / AEM Forms on JEE application** — the document-centric platform authored in **LiveCycle
Workbench** and **Designer** — into native AEM Forms as a Cloud Service artifacts. AEMaaCS does
**not** run LiveCycle JEE orchestrations, DSCs, or XFA-based server rendering, so Path B is a
**re-platforming**, not a lift-and-shift: every LiveCycle construct is re-implemented on its
cloud-native equivalent while its behaviour and record output are preserved. Nothing is silently
dropped — anything with no cloud equivalent is surfaced to the user and recorded as flagged.

## The two Path B inputs

| Input | What it is | What it carries |
|---|---|---|
| **`.lca` (LiveCycle Archive)** | The application export from Workbench (a ZIP) | **XDP** form templates (XFA), **XSD** data schemas, sample **XML** data, **PDF**s (interactive/print), Workbench **processes / orchestrations**, XDP **fragments**, images/resources, and the process references to DSC operations |
| **`.jar` (custom DSC)** | The compiled **Document Services Component(s)** installed into Workbench and invoked from the orchestrations | Java classes implementing custom operations (the component id + `operation` methods) called as process activities |

Ask for **both** up front — a `.lca` without its DSC `.jar` leaves every custom process activity
unresolved; a `.jar` without the `.lca` has no orchestration to wire into. If one is genuinely
absent, surface exactly which operations/processes will be left as flagged stubs.

## Target mapping (LiveCycle → AEMaaCS) — the decisions Path B makes

| LiveCycle / JEE artifact | AEMaaCS target | Skill / module that builds it |
|---|---|---|
| **XDP form template** (XFA) | **Core Components Adaptive Form** (exact-replica of fields/layout) **+ the original XDP retained as the Document-of-Record template** | `generate-schema` → `create-adaptive-form` (+ DoR wiring); Path A Steps 3–6 shapes |
| **XSD data schema** | Form data schema (JSON Schema) / FDM schema | `generate-schema` |
| **Sample XML data** | Prefill data + test fixtures | `create-prefill-service` |
| **PDF (interactive/print)** | Document of Record output (rendered from the retained XDP) | DoR wiring on the form container / `create-workflow` Generate-DoR step |
| **XDP fragment** | Adaptive Form Fragment | `create-AdaptiveFormFragment` |
| **XFA script** (FormCalc / XFA-JS in the XDP) | Adaptive Form rule (json-formula) + clientlib function | `create-form-rules` + `create-form-clientlib` (Path A Step 5a extraction rules apply verbatim) |
| **Workbench process / orchestration** | **AEM Workflow model** (Assign Task / route / AND-OR split / Generate DoR / Invoke FDM / email / Adobe Sign) | `create-workflow` |
| **Custom DSC operation** (`.jar`) | **OSGi service** + a **workflow process step** and/or **custom submit action** | `create-submit-action` / `create-workflow` (process step) + hand-authored `core`-module Java |
| **Built-in LiveCycle service call** (Forms, Output, Assembler, PDF Generator, Signature/DocAssurance, Reader Extensions, Correspondence Mgmt) | The **OOTB** AEMaaCS workflow step / Document Service that already ships the capability (never re-built as custom code) | Use the OOTB step and author it to match the source activity — see **Step B5.1**; flag only anything with no cloud equivalent |
| **Render service** (renders XDP → HTML/PDF) | Adaptive Form HTML render (Core Components) | native — no artifact; the AF renders itself |
| **Submit / invoke-process** | Submit action wired to **"Invoke an AEM Workflow"** | `create-submit-action` / `create-workflow` submit wiring |

> ⚠️ **Path B is a re-platforming — preserve behaviour, not implementation.** LiveCycle
> orchestrations, DSCs, and XFA server scripts have **no runtime on AEMaaCS**. Do not attempt to run
> them; re-implement each on its cloud equivalent above and prove parity of *behaviour and output*
> (same fields, same validations, same routing, same record PDF), exactly as Path A proves UI parity.

## Path B runs as a full ADLC delivery (mandatory)

Like every migration, a LiveCycle migration runs the whole pipeline
(PLAN → DESI → IMPL → ASSEMBLY → DEPLOY → TEST → HANDOFF), never as a one-shot skill run. When
invoked standalone, hand it to the `aem-forms-program-agent` to orchestrate; this skill then runs as
the IMPL-build step, authoring artifacts only and deferring the single build+deploy to the deploy
lead. The **original XDP/PDF rendered output is the parity reference** for the TEST phase.

---

## Step B1 — Collect inputs and unpack the `.lca`

Confirm the same project tokens as Path A (`{project}`, `{appFolder}`, `{theme}`, `{template}`), plus:

| Input | Description | Example |
|---|---|---|
| `{lcaPath}` | The LiveCycle Archive to migrate | `downloads/CorrespondenceApp.lca` |
| `{dscJarPath}` | The custom DSC bundle whose operations the processes call | `downloads/custom-dsc-1.2.jar` |
| `{dscSource}` (optional) | Java source for the DSC if available (avoids decompiling) | `src/custom-dsc/` |
| `{originalReference}` | The original form's rendered output for the Step 8 parity check — a screenshot/PDF of the XDP-rendered form | `C:\legacy\correspondence.pdf` |

**`{formName}` / `{formTitle}` are NOT collected up front here.** An `.lca` commonly contains **more
than one XDP** — do not assume a single form. Each top-level XDP found in Step B2's inventory gets its
**own** derived `{formName}`/`{formTitle}`, decided in Step B2, not asked as a single pair now.

An `.lca` is a ZIP. Unpack it to a temp location (never into the repo):

```powershell
Copy-Item "{lcaPath}" "$env:TEMP\lca-src.zip" -Force
Expand-Archive -Path "$env:TEMP\lca-src.zip" -DestinationPath "$env:TEMP\migrate-lca-src" -Force
```

A LiveCycle application archive typically contains, per application/version:
- **`*.xdp`** — XFA form templates (and XDP fragments)
- **`*.xsd`** — data schemas
- **`*.xml`** — sample/config data, and the **process/orchestration** definitions
- **`*.pdf`** — interactive or print PDFs
- a **manifest** listing the application assets and their DSC/service references

If there is no XDP/process content, it is not a migratable LiveCycle application — stop and tell the
user.

## Step B2 — Inventory and classify every LiveCycle asset

Before building anything, enumerate the whole archive and the DSC, and record what each becomes. Do
not migrate the forms and ignore the processes — a LiveCycle app is forms **plus** the orchestrations
and services that drive them.

Build the **LiveCycle inventory**:
- **Forms:** every XDP — its fields, subforms (panels), layout (positioned vs flowed), bindings to
  the XSD, and any XFA FormCalc/JavaScript (calculations, validations, event scripts).
- **Fragments:** shared XDP fragments referenced by more than one form.
- **Schemas:** every XSD and which forms bind to it.
- **Data:** sample XML files (become prefill fixtures + test data).
- **PDFs:** which are the print/record output of which XDP.
- **Processes:** every orchestration — its variables, its activity sequence, each activity's
  operation (built-in LiveCycle service **or** a custom DSC operation), routes/gateways, user
  (Assign Task) steps, and start-point (event / watched-folder / REST).
- **DSC operations:** from the `.jar`, every custom operation an orchestration calls — its component
  id, operation name, input/output signature, and logic.
- **Built-in service calls:** LiveCycle service invocations (Forms, Output, Assembler, PDF
  Generator, Signature/DocAssurance, Reader Extensions, Correspondence Mgmt) — each maps to an
  **OOTB** AEMaaCS step / Document Service (Step B5.1), not custom code.

### Multiple XDPs — derive one `{formName}` per top-level XDP (mandatory)

An `.lca` frequently contains **several XDPs**. Classify each one found during the inventory as either:
- **Top-level (a real, independently-submittable form)** — gets its own `{formName}` (kebab-case its
  own filename, or its internal `<template>` name if that is more descriptive — e.g.
  `EmployeeTrainingRequest.xdp` → `employee-training-request`) and its own `{formTitle}`. It is migrated
  by **repeating Steps B3, B3.5 and B4 once per top-level XDP** (see Step B3's loop note below).
- **Fragment (referenced by one or more other XDPs, never submitted on its own)** — migrated **once**
  as an `AdaptiveFormFragment` in Step B7, never duplicated as a standalone form per referencing parent.

**Collision rule:** if two top-level XDPs would derive the **same** `{formName}`, stop and ask the user
to disambiguate explicitly — never silently auto-suffix (`-2`, `-copy`, etc.).

Then present **ONE consolidated Path B migration plan covering every top-level XDP found** — one row
per XDP (derived `{formName}`, field count, rule count, whether any process/workflow targets it, DoR
need) — plus the process/DSC mapping table below, and get a **single** go-ahead for the whole set
before writing anything. Do not confirm XDPs one at a time.

## Step B3 — Reverse-engineer each XDP into a Core Components Adaptive Form (+ retain the XDP as DoR)

For every **top-level** XDP identified in Step B2, produce a **Core Components Adaptive Form that is
an exact replica of the original**, and keep the XDP itself as the Document-of-Record template so the
record PDF is preserved. **Repeat this entire step, and B3.5 and B4, once per top-level XDP** from the
consolidated plan — do not stop after the first one. Skip any XDP Step B2 classified as a **fragment**;
those are handled once, collectively, in Step B7 instead.

1. **Derive the schema.** Feed the XDP (and its bound XSD) to **`generate-schema`** to produce the
   form data schema + FDM-ready `dataRef` bindings. Prefer the XSD when present (it is the
   authoritative data contract); use the XDP's binding refs to map each field to its schema path.
2. **Build the form.** Hand the field inventory to **`create-adaptive-form`**, building each XDP
   field as its Core Components equivalent and each **subform** as a `panelcontainer` (a flowed
   subform with `maxOccur=-1` becomes a repeatable panel; an XFA choice/exclusion group becomes a
   `radiobutton`/`checkboxgroup`). Map field types exactly:

   | XFA field (XDP) | Core Components field |
   |---|---|
   | `textField` | `textinput` (`multiLine="true"` if multi-line) |
   | `numericField` / `decimalField` | `numberinput` |
   | `dropDownList` | `dropdown` |
   | `checkButton` **exclusion group** (`exclGroup`) | `radiobutton` |
   | `checkButton` (independent) | `checkboxgroup` |
   | `dateField` / `dateTimeField` | `datepicker` |
   | `imageField` / attachment | `fileinput` |
   | `draw` / static text | `text` |
   | `button` (submit) | `actions/submit` |

   Preserve field order, titles, captions (→ label), and mandatory state.
3. **Set the XDP as the Document of Record.** Upload the original XDP as a genuine `dam:Asset` under
   the project's SHARED top-level XDP folder — `/content/dam/formsanddocuments/XDP/{Name}.xdp` (every
   retained-XDP DoR template lives here, not nested per-form — a template is not tied one-to-one to a
   single form's own DAM folder) — binary at `jcr:content/renditions/original` + a sibling
   `jcr:content/renditions/original.dir/.content.xml` (`nt:file`/`nt:resource`,
   `jcr:mimeType=application/vnd.adobe.xdp+xml`); a `cq:Page` is **not** a valid target. Then set
   `dorType="select"` + `dorTemplateRef` (pointing at that asset) on the **DAM guide asset's own
   `jcr:content/metadata` node** at
   `/content/dam/formsanddocuments/{appFolder}/{formName}/jcr:content/metadata` (the "Form Properties"
   data, `sling:resourceType=fd/fm/af/render`) — **do NOT** set them on the migrated `guideContainer`;
   that component's dialog has no DoR-template field and a direct edit there is inert (live-verified:
   neither the editor UI nor `AFtoDORStep` reads `dorTemplateRef` from `guideContainer`). Cover the new
   asset path with the existing `/content/dam/formsanddocuments` filter root (usually already present).
   **Never downgrade a form that produced a PDF record to `dorType="none"`.**
4. **Author the resulting AF using Path A's shapes.** Everything downstream — resource types (Step
   3), the 4 cloud artifacts (Step 4), asset migration (Step 4A), the global XML conventions — is
   **identical to Path A**. Follow those steps for the generated form; Path B only changes where the
   form *came from*. **The theme is the exception: it does NOT come from a Foundation theme tree — it
   is extracted from the XDP's own XML. See Step B3.5 (mandatory).**

### B3.5 — Extract the XDP's embedded visual styling into the theme (MANDATORY — never default to a stock theme)

An XDP is **self-styled**: unlike a Path-A Foundation form (whose look lives in a separate theme-editor
JCR tree, parsed by Step 6), an XDP carries its fonts, colours, borders, backgrounds and layout
**inline in its own XML** — on every `<subform>`, `<field>`, `<draw>`, `<caption>` and `<value>`. So
for Path B the theme ground-truth **is the XDP XML**, and extracting it is **mandatory**: the migrated
Adaptive Form's **HTML render** (not only the DoR PDF) must reproduce the XDP's look to **≥ 90% visual
parity**. Do **not** fall back to a stock project theme (e.g. `-wknd`) on the assumption that the
retained XDP DoR carries the look — the DoR is only the print record; the on-screen form must match too.

> Users generally **cannot** supply a rendered screenshot for every XDP in an `.lca`. The skill must
> therefore recover the look **from the XDP XML by default**. A screenshot, when the user does provide
> one, is only an extra parity baseline for Step 8 — never a precondition for extracting the theme.

**B3.5.1 — Parse every style-bearing XFA node.** XFA colours are decimal `"r,g,b"` (e.g. `128,64,64`
→ `#804040` / `rgb(128,64,64)`; `196,219,251` → `#C4DBFB`; `225,242,219` → `#E1F2DB`) — convert each to
hex/rgb. Walk the whole XDP tree and record:

| XFA node (in the XDP) | Carries | Core Components theme target |
|---|---|---|
| `<font typeface size weight posture>` + its `<fill><color>` | field/caption/heading **text** font family, size (pt), bold/italic, **text colour** | `.cmp-adaptiveform-*__label` / `__widget` / text-component styles; map `typeface` to the closest web font + web-safe fallback stack (declare `@font-face` only if the font file ships in the LCA) |
| root/section `<subform>` → `<fill><color>` | **section / panel body background** | `.cmp-adaptiveform-panel` (and `af_panel`) `background-color` |
| header `<draw>` / title-caption `<fill><color>` | **section-header band** background | the panel's header/title element `background-color` |
| `<field>` → `<ui>` `<border><edge stroke thickness presence>` | **input border** width/style/colour (`presence="hidden"` = no border; `stroke="lowered"` ≈ 1px solid) | `.cmp-adaptiveform-*__widget` `border` |
| `<field>` → `<ui>` `<fill><color>` | **input background** | `.cmp-adaptiveform-*__widget` `background-color` |
| `<pageArea>/<contentArea>` + root subform `<fill>` | **page / form background** | `af_form` / page background |
| `<subform layout=…>` + column widths, `<margin>` | **layout** (multi-column caption+field rows, spacing) | grid/column layout + field spacing (theme/clientlib) |
| `<caption><para hAlign vAlign>` + caption reserve/placement | caption placement (left-of-field vs above) & alignment | label placement / text-align |

**B3.5.2 — Reproduce the values VERBATIM in all three theme carriers**, exactly as Path A Step 6.3
requires: theme.zip `theme.css` (unscoped), per-form clientlib CSS with `!important`, and the DAM
theme-json `af_*` nodes. Copy every colour / font / border / background as the exact extracted value —
never a "close enough" substitute. Keep the theme string identical across form page / DAM guide asset /
conf context (Step 4). **Create a NEW `/apps/fd/af/themes/{project}-{theme}` theme for the extracted
look** (name it after the app, e.g. `{project}-<appName>`); reuse an existing stock theme ONLY if the
user explicitly accepts a look that differs from the XDP.

**B3.5.3 — Fonts.** XDP typefaces (e.g. `Myriad Pro`, `Effra`) are usually not web-available. If the
LCA ships the font file, migrate it (Step 4A.2) and declare `@font-face`; otherwise map to the closest
web font with a documented fallback stack and **flag the substitution** to the user — it is the one
place exact parity may be impossible. Never silently render in an unrelated default font.

**B3.5.4 — Verify parity.** Run Step 8 against the XDP render (open the XDP / its PDF preview) or the
user-supplied screenshot; a ≥ 90% match on section bands, text colour, field borders, and layout is the
gate. Apply the **same** extracted styles to sections the screenshot does not show (the XDP XML covers
them all).

## Step B4 — Migrate XFA scripts to Adaptive Form rules

XFA FormCalc / XFA-JavaScript embedded in the XDP has **no runtime on a Core Components Adaptive
Form**. Re-author each as an Adaptive Form rule, following **Path A Step 5 / Step 5a verbatim**:
- Every calculation/validation/set-value becomes a `create-form-rules` rule whose body is a **single
  named clientlib function call** (no inline JS in the form XML).
- Re-implement FormCalc built-ins and any `xfa.*` API used (`xfa.host`, `xfa.resolveNode`, `Sum`,
  `Concat`, date math) as plain JS inside the clientlib function.
- json-formula booleans are `true()`/`false()`; escape commas in `fd:events`; verify each rule
  compiled into `guideContainer.model.json`. All of Path A Step 5's rule-AST rules apply.

### B4.1 — Find EVERY script: XFA holds them in TWO places — harvest both (MANDATORY)

Do **not** look only at field calculations/validations. XFA logic lives in two distinct locations and
**both** must be walked for **every** field, subform, exclusion group, and the root/host:

1. **Inline event handlers** — `<event activity="…">` blocks containing a `<script>` (or a nested
   `<validate>`/`<calculate>` with a `<script>`). Enumerate **all** activities, not just a few:
   `initialize`, `ready` / `form:ready` / `docReady`, `enter`, `exit`, `change`, `full`, `click`,
   `mouseUp`/`mouseDown`, `preSubmit`/`preSave`/`preOpen`, `calculate`, and `validate`. Map each XFA
   event to its Adaptive Form rule trigger:

   | XFA event activity | Adaptive Form rule trigger |
   |---|---|
   | `initialize`, `ready`/`form:ready`/`docReady` | run on **initialize** / "when form loads" |
   | `exit` | run on **blur** (commit) |
   | `change` | run on **change** |
   | `enter` | run on **focus** |
   | `click` / `mouseUp` (button) | button/action rule |
   | `<calculate>` | **calculate** rule |
   | `<validate>` (`nullTest`/`scriptTest`/`formatTest`) | **validation** rule (with the same required/warning/error severity) |

2. **Reusable `<script>` script objects** — form- or subform-level named `<script>` objects (e.g.
   `<script name="Utils" contentType="application/x-javascript">`) that other events call as shared
   functions. Port each into the form's clientlib as a **named global function**, and have the migrated
   rules call it — preserving the "define once, call from many events" structure.

**Recognise and DROP LiveCycle boilerplate harness scripts — do not port them.** Generated plumbing
such as `ContainerFoundation_JS`, `event__form_ready` / `FormReady`, and any block guarded by
`// DO NOT MODIFY THE CODE BEYOND THIS POINT …` is LiveCycle-runtime scaffolding with **no** Core
Components analog. Skip these (they would break or do nothing on cloud) and **record them in the delta
as intentionally-not-ported harness** — never fabricate an equivalent, and never mistake them for
business logic. Migrate only the author-written logic; every author-written event/script becomes a
rule, counted in `xfa_scripts_migrated`.

## Step B5 — Re-platform Workbench processes as AEM Workflows (use OOTB services, authored to match)

Each orchestration becomes an **AEM Workflow model** via **`create-workflow`**, mapped
piece-by-piece — do not drop a routing branch, an approval step, or a service call.

**The governing rule: reuse OOTB, don't rebuild.** A LiveCycle orchestration activity is one of two
things, and they migrate differently:

1. A call to a **built-in LiveCycle service** (Output, Assembler, DocAssurance/Reader-Extensions/
   Signature, PDF Generator, Forms, Form Data Integration, Distribute/Email, Task Manager/Assign
   Task, Adobe Sign, Correspondence Management, Set Value, decision/route). **AEMaaCS ships an OOTB
   equivalent for each** — an AEM Forms workflow **process step** and/or a Document Services API.
   **Use the OOTB step — never re-implement a built-in service as custom Java.** Then **author the
   OOTB step to mirror the original activity's configuration** so the behaviour is identical.
2. A call to a **custom DSC operation** (from the `.jar`) that has **no** OOTB counterpart → and only
   then → a custom workflow process step backed by the OSGi service you build in **Step B6**.

### B5.1 — LiveCycle built-in service → AEMaaCS OOTB service/step, and how to author it to match

For each built-in service the orchestration used, drop in the OOTB step below and carry over **every**
configuration value from the LiveCycle activity (the right-hand column is the authoring checklist —
matching it is what makes the migrated model actually behave like the original):

| LiveCycle built-in service / activity | AEMaaCS OOTB step / Document Service | Author it to match — carry over VERBATIM |
|---|---|---|
| **Output** — `generatePDFOutput` / `generatePrintedOutput` (XDP + data → PDF/print) | **Generate Document of Record** step, or the **Output Service** (`com.adobe.fd.output.api.OutputService`) via a Document-Services process step | The migrated **XDP template**, the **data**/XML source (form data or FDM output), locale, render options (tagged PDF, embedded fonts), and output type (PDF vs print-stream/PCL/PS/ZPL) |
| **Assembler** — `invokeDDX` (assemble / split / merge / stitch / watermark PDFs) | **Assembler** step (`com.adobe.fd.assembler.client.AssemblerService.invokeDDX`) | The original **DDX** document; map each DDX-referenced source to a workflow variable/document; preserve PDF/A conformance and assembly order |
| **DocAssurance / Reader Extensions / Encryption / Digital Signature** | **Secure Document** step (`com.adobe.fd.docassurance.client.api.DocAssuranceService.secureDocument`) | The **credential alias**, the **usage rights** (Reader Extensions), the encryption type (password / certificate) + recipients, and the signature field, appearance, and hash algorithm |
| **PDF Generator (PDFG) / Distiller** — `createPDF` / convert-to-PDF | **Convert to PDF** (PDF Generator) step | Source format, conversion/file-type settings, and PDF settings/security carried from the LiveCycle job options |
| **Forms** — `renderPDFForm` / `processFormSubmission` | Native **Adaptive Form render** (Core Components) + **Import/Export Data** step for round-tripping data | No render artifact (the AF renders itself); for data merge/extract use the Import/Export Data step against the migrated schema |
| **Form Data Integration** — data service call | **Invoke Form Data Model** step | The **FDM operation**, its input bindings (mapped from workflow variables/form data) and output binding back into a variable |
| **Email / Distribute** — `sendWithAttachments` | **Send Email** step (+ **Day CQ Mail Service** config) | The to/cc/bcc/from, the **subject & body templates** (with the same placeholders), and the **attachment** (the generated DoR/PDF from the Output/Assembler step) |
| **Task Manager / Workspace** — Assign Task / user step | **Assign Task** step (AEM Forms workflow inbox) | The **assignee/group/queue** (as a **dynamic-participant ECMA script** if it was dynamic), the **form or task summary** shown to the user, the **routes/actions** (Approve/Reject/Return → `actionTaken` route values), and the **SLA / reminder / escalation** |
| **Adobe Sign / Signature** — e-sign activity | **Adobe Sign** OOTB steps | The **signers** and signing **order**, the signature fields, and the agreement/template settings |
| **Correspondence Management** — letter / interactive document | **Interactive Communications** / Generate Correspondence step | The letter/IC **template**, its **data-dictionary** bindings, and the delivery channel (print vs web) |
| **Set Value** activity | **Set Variable** step | The **target variable** and the **expression/mapping** that populates it |
| **Decision point / route / gateway** (exclusive/parallel) | **OR Split** / **AND Split** | Each branch's **route condition** expression (rewritten against workflow variables / `actionTaken`) |
| **Execute Script (QPAC / foundation script)** | **Process Step** (ECMAScript) — or an OSGi `WorkflowProcess` if it uses on-prem APIs | Re-implement the script logic; replace any on-prem-only API; keep the same inputs/outputs |
| Custom **DSC operation** (no OOTB equivalent) | **Custom workflow process step** backed by the OSGi service from **Step B6** | The operation's input/output contract and business logic (Step B6) |

> ⚠️ **Author, don't just place.** Dropping the right OOTB step onto the model is half the job — a
> Generate-DoR step with no template, an Assembler step with no DDX, an Assign-Task step with no
> participant/route, or a Send-Email step with an empty body **compiles but does nothing useful**. For
> every OOTB step, fill in the same values the LiveCycle activity carried (the table's right column)
> and verify against the source orchestration that inputs, outputs, participants, routes, and options
> all match. This is what makes the migrated model behave **identically** to the original.

### B5.2 — Map the process skeleton (variables, sequence, start-point)

| Workbench process construct | AEM Workflow equivalent |
|---|---|
| Process variables (with types) | **Typed workflow variables** (same names/types so expressions port cleanly) |
| Activity sequence | The step order in the workflow model |
| Route / gateway | **OR Split** / **AND Split** (B5.1) |
| Built-in service activity | The OOTB step from **B5.1** |
| Custom DSC activity | Custom process step from **Step B6** |
| Start point (event / watched folder / REST endpoint) | **Workflow launcher**, a scheduled trigger, or the **"Invoke an AEM Workflow"** submit action on the form |

Wire the migrated form's Submit to **"Invoke an AEM Workflow"** pointing at the new model
(`create-workflow` produces this wiring), so submitting the Adaptive Form starts the re-platformed
process — the cloud analog of the LiveCycle render→submit→invoke-process flow.

> **Verify the model runs end-to-end (not just deploys).** After deploy, start the workflow (submit
> the form / trigger the launcher) and confirm each OOTB step actually produces its output — the DoR
> PDF is generated from the XDP, the Assembler output assembles, the task lands in the correct
> inbox/queue with the right routes, the email arrives with its attachment, the FDM call returns
> data, and every route branch is taken under the right condition — matching what the LiveCycle
> orchestration did.

## Step B6 — Re-implement the custom DSC `.jar` as OSGi services

> **OOTB-first — confirm there is no OOTB equivalent before building custom.** This step is **only**
> for operations that are genuinely custom business logic. If an operation actually wraps a built-in
> LiveCycle service (Output, Assembler, DocAssurance, PDF Generator, email, FDM, …), do **not**
> re-implement it here — use the OOTB step from **Step B5.1** instead. Custom Java is a last resort,
> not the default.

Each **custom** DSC operation an orchestration calls (one with no OOTB counterpart) must be
re-implemented as cloud-native Java in the **`core`** module, then referenced from the workflow
(Step B5) or a submit action.

1. **Recover the operation contracts.** If `{dscSource}` was provided, read it. Otherwise inspect the
   `.jar`: list classes and method signatures, and — only when necessary and permitted — decompile to
   recover logic:
   ```powershell
   jar tf "{dscJarPath}"                                       # list classes / resources
   javap -p -classpath "{dscJarPath}" <fully.qualified.ClassName>   # signatures without source
   ```
   Read the DSC's `component.xml` / service descriptor inside the jar to map **component id +
   operation name** (how the orchestration references it) to the implementing class/method.
2. **Re-implement each operation as an OSGi service** under `com.aem.forms.agents...` in `core`,
   preserving the input/output contract. Replace every LiveCycle API the DSC used
   (`com.adobe.livecycle.*`, `com.adobe.idp.*`, `ServiceClientFactory`, `DocumentManager`) with its
   AEM/AEMaaCS equivalent (Sling, JCR, the cloud document-services API) — none of the LiveCycle
   runtime classes exist on cloud.
3. **Expose it** as a custom workflow process step (`WorkflowProcess`) and/or to the form as a
   **`create-submit-action`** handler, matching how the orchestration invoked the operation.
4. **Unit-test each re-implemented operation** (`create-form-tests`).

> ⚠️ **Decompiled bytecode is a reference, not a paste.** Recover the *intent* (inputs, outputs,
> business rules) and re-author idiomatic AEM Java — never ship decompiled sources. If an operation's
> logic can't be recovered or depends on an on-prem-only subsystem with no cloud equivalent, **flag
> it to the user as a stub** with its signature, and record it in the delta — do not fabricate logic.

## Step B7 — Migrate schema, data, fragments, and assets

- **Schema (XSD).** Land the generated schema (Step B3.1) under
  `/content/dam/formsanddocuments/schema/{formName}.schema.json` with filter coverage; confirm every
  field's `dataRef` resolves against it (Path A Step 4A).
- **Sample XML → prefill.** Convert representative sample XML into the prefill data and wire it via
  **`create-prefill-service`**; keep the rest as test fixtures for the TEST phase.
- **XDP fragments → Adaptive Form Fragments.** Migrate each shared XDP fragment as its own fragment
  via **`create-AdaptiveFormFragment`** and reference it by `fragmentPath` (do not inline).
- **Images / PDFs / fonts.** Migrate every referenced asset to its cloud DAM path and rewrite the
  reference, exactly as Path A Step 4A. The retained XDP DoR template is one of these assets.

## Step B8 — Deploy, then verify parity (shared with Path A)

Path B uses the **same** deploy step (**Step 7**) and **UI/behaviour-parity gate (Step 8)** as Path A
— with these Path-B specifics:
- **UI/record parity:** compare the migrated Adaptive Form's render **and** its generated Document of
  Record against the **original XDP-rendered form / PDF** (`{originalReference}`). PASS = zero
  Critical findings: same fields, labels, order, types, sections, submit — and the record PDF carries
  the same data/layout.
- **Behaviour parity:** exercise each migrated rule (former XFA script), each workflow route (former
  orchestration branch), and each re-implemented DSC operation, proving the cloud behaviour matches
  the LiveCycle behaviour. This is the Path-B analog of "rules fire / submit works".
- Confirm no `com.adobe.livecycle.*` / `com.adobe.idp.*` import survives in the deployed bundle, and
  no XFA script remains inline in the form.

---

## Path B — failure symptoms and their causes

| Symptom | Cause | Fix |
|---|---|---|
| `.lca` won't unpack | Not a ZIP / corrupt export | Re-export from Workbench; confirm it opens as a ZIP before proceeding |
| Migrated form missing fields present in the XDP | Positioned-subform / nested-subform fields skipped during reverse-engineering | Re-walk the XDP subform tree; map every field & subform (Step B3.2) |
| Record PDF lost / blank | XDP not retained as DoR, `dorType` downgraded to `none`, or `dorType`/`dorTemplateRef` were set on `guideContainer` instead of the DAM guide-asset's `metadata` node (inert there — no field, not read by `AFtoDORStep`) | Copy the XDP to DAM as a real `dam:Asset`, set `dorType="select"`+`dorTemplateRef` on the **DAM guide asset's `jcr:content/metadata`** node (Form Properties), add filter coverage (Step B3.3) |
| Migrated HTML form doesn't look like the XDP (stock/blue theme, wrong colours, no section bands) | The XDP's inline `<font>/<fill>/<color>/<border>` styling was not extracted; a stock project theme (`-wknd`) was used instead | Parse the XDP style nodes and reproduce every value in all three theme carriers (Step B3.5); build a new app-named theme, don't reuse a stock one |
| XFA calculation/validation doesn't run | XFA script left in the XDP; it has no Core Components runtime | Re-author as a clientlib-function rule (Step B4 / Path A Step 5a) |
| Workflow step does nothing / process route missing | Orchestration activity/gateway not mapped to a workflow step | Map every activity & route via `create-workflow` (Step B5) |
| OOTB step deploys but produces no output (empty DoR, no email/attachment, task in wrong queue) | The right OOTB step was placed but **not authored** to match the source activity (no template/DDX/data mapping/participant/route) | Fill in every config value the LiveCycle activity carried (Step B5.1 right-hand column) and verify end-to-end |
| Built-in service needlessly re-implemented as custom Java (extra bundle, LiveCycle-API compile errors) | An activity that wraps a built-in service was sent to Step B6 instead of mapped to its OOTB step | Replace the custom code with the OOTB step/Document Service from Step B5.1 (custom is only for no-OOTB-equivalent DSC ops) |
| Build fails on `com.adobe.livecycle.*` / `com.adobe.idp.*` import | DSC re-implemented against LiveCycle APIs that don't exist on cloud | Replace with Sling/JCR/cloud document-services APIs (Step B6.2) |
| Custom process activity unresolved at runtime | DSC operation not re-implemented, or workflow step not pointed at the new OSGi service | Re-implement the operation and wire the process step (Step B6.3) |
| Submit doesn't start the process | Submit not wired to "Invoke an AEM Workflow" | Wire the submit action to the new workflow model (Step B5) |

---

## Path B — pre-flight checklist

- [ ] Both inputs collected: `.lca` unpacked (XDPs, XSDs, XML, PDFs, processes enumerated) and the DSC `.jar` (operations/signatures recovered); `{originalReference}` captured for parity
- [ ] LiveCycle inventory built (forms, fragments, schemas, data, PDFs, processes, DSC operations, built-in service calls) and the Path B migration plan shown and approved
- [ ] Every XDP reverse-engineered to a Core Components Adaptive Form (exact field/subform/type/order replica), authored via `generate-schema` + `create-adaptive-form` using Path A Steps 3–4 shapes (theme comes from the XDP per Step B3.5, not Path A Step 6)
- [ ] The original XDP retained as the Document-of-Record template — uploaded as a genuine `dam:Asset` (not a page), `dorType="select"`+`dorTemplateRef` set on the **DAM guide asset's `jcr:content/metadata`** node (Form Properties — NOT `guideContainer`, which has no DoR-template field), filter-covered — no form that produced a PDF downgraded to `dorType="none"`
- [ ] **Theme extracted from the XDP's own XML (Step B3.5), NOT a stock theme** — every `<font>` (typeface/size/weight/colour), section/panel `<fill><color>`, header-band colour, `<field>` `<border>/<edge>` and input background parsed and reproduced VERBATIM in all three carriers under a new app-named `{project}-{theme}`; HTML render matches the XDP to ≥ 90%; font substitutions (non-web XDP typefaces) flagged to the user
- [ ] Every XFA FormCalc/JS script re-authored as an Adaptive Form rule (single clientlib-function call, no inline JS, `true()`/`false()`), verified in `guideContainer.model.json` (Path A Step 5a) — **BOTH sources harvested (Step B4.1): inline `<event activity=…>` handlers (initialize/ready/docReady/enter/exit/change/click/calculate/validate) AND reusable `<script>` script objects, for every field/subform/host**; LiveCycle harness scripts (`ContainerFoundation_JS`, `FormReady`, `DO NOT MODIFY` blocks) recognised and recorded as intentionally-not-ported, not fabricated
- [ ] Every Workbench process re-implemented as an AEM Workflow (every variable, sequence, route/gateway, and start-point mapped); submit wired to "Invoke an AEM Workflow"
- [ ] Every **built-in** LiveCycle service call mapped to its **OOTB** AEMaaCS step/Document-Service (Step B5.1) — NOT re-built as custom code — AND each OOTB step **authored to match** the source activity (template/DDX, data/document mappings, participants/queues, route conditions, SLA/reminders, credentials/usage-rights, output options); verified end-to-end that each step produces its output and every route branch fires under the right condition
- [ ] Only genuinely-custom DSC operations (no OOTB equivalent) re-implemented as an OSGi service in `core` (contract preserved, LiveCycle APIs replaced, unit-tested); no `com.adobe.livecycle.*`/`com.adobe.idp.*` import survives; unrecoverable operations flagged as stubs, never fabricated
- [ ] Schema (from XSD) landed + filter-covered; sample XML wired as prefill (`create-prefill-service`) + kept as test fixtures; XDP fragments migrated as Adaptive Form Fragments; all images/PDFs/fonts migrated & re-referenced (Path A Step 4A)
- [ ] Global XML conventions, deploy (Step 7), and parity gate (Step 8) applied; UI+record parity vs the original XDP render/PDF is zero-Critical; behaviour parity (rules, routes, DSC ops) proven
- [ ] Anything with no cloud equivalent (on-prem-only service, unrecoverable DSC logic, licensed font) surfaced to the user and recorded in the migration delta — nothing silently dropped
