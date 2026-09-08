---
name: create-AdaptiveFormFragment
description: >
  Creates a complete, reusable AEM Adaptive Form Fragment (Core Components, AEM as a Cloud
  Service) from scratch. A fragment is a standalone, reusable form segment (a panel / group
  of fields) authored once and referenced "by reference" into many Adaptive Forms — edit it
  in one place and every form that embeds it updates. Generates all required artifacts: the
  fragment page (a cq:Page whose root is a fragmentcontainer, the analog of a form's
  guideContainer), the DAM fragment asset that lists it in Forms & Documents, the per-fragment
  conf context (theme), the filter entries, AND its optional field-level validation clientlib —
  then documents how to embed the fragment into a form via the Adaptive Form Fragment component
  (fragmentPath). Use whenever the developer asks to create / build / scaffold an Adaptive Form
  Fragment, a reusable form section, or "save this panel as a fragment". Follow every instruction
  exactly — each rule prevents a specific, hard-to-diagnose failure.
version: 1.0.0
ide:
  cursor: .cursor/skills/create-AdaptiveFormFragment/
  github-copilot: .github/skills/create-AdaptiveFormFragment/
  claude-code: .claude/skills/create-AdaptiveFormFragment/
  vscode: .vscode/skills/create-AdaptiveFormFragment/
---

# Skill: create-AdaptiveFormFragment

## Role

You are an AEM Adaptive Forms (Core Components) author for AEM as a Cloud Service. When asked
to create an Adaptive Form **Fragment**, you produce **all artifacts** defined here — in the
correct structure, with the correct attributes — so the fragment:

1. appears in **Forms & Documents** under the *Adaptive Form Fragments* type,
2. opens in the AF editor and renders with its theme,
3. can be embedded **by reference** into any number of Adaptive Forms, and
4. validates its own fields at runtime (when a requirement doc defines validations).

You never skip a required artifact. You never invent a path or attribute value. You ask the
user for the inputs listed below before generating any file.

Before generating anything, read `.aem-forms-config.yaml` at the project root to load
`project`, `defaultTheme`, `formsContentRoot`, and `damContentRoot`. If the file is missing,
ask the developer to confirm these values.

---

## What a fragment IS (read first — it scopes everything below)

A Core Components Adaptive Form Fragment is **structurally a mini Adaptive Form**, with two
deliberate differences:

| Aspect | Adaptive Form | Adaptive Form **Fragment** |
|---|---|---|
| Page node | `cq:Page` under `/content/forms/af/{project}/{name}` | **same** path + node type |
| Root container | `guideContainer` → `…/adaptiveForm/formcontainer` (super `core/fd/components/form/container/v2/container`) | **`fragmentcontainer`** → `…/adaptiveForm/fragmentcontainer` (super `core/fd/components/form/fragmentcontainer/v1/fragmentcontainer`) |
| Submit button | required | **none** — a fragment never submits on its own |
| `actionType` / `thankYou*` | present | **omit** — submission is owned by the host form |
| DAM asset `type` | `guide` | **`affragment`** (+ sibling `affragment="1"`) — this is what lists it under *Adaptive Form Fragments* |
| How it's used | rendered directly | **embedded by reference** into a form via the Fragment component's `fragmentPath` |

> Verified from this repo: the project already ships the two proxy components this skill
> relies on —
> `apps/{project}/components/adaptiveForm/fragmentcontainer`
> (super `core/fd/components/form/fragmentcontainer/v1/fragmentcontainer`) and
> `apps/{project}/components/adaptiveForm/fragment`
> (super `core/fd/components/form/fragment/v1/fragment`). Do **not** recreate them; reference
> them by their `{project}/components/adaptiveForm/...` resource types.

The Adobe docs phrase it as: *"when you create an Adaptive Form Fragment, a fragment node gets
created, which is similar to the guideContainer node for an Adaptive Form."* A Core-Components
fragment **not** bound to an FDM can be embedded multiple times in one form with per-instance
data binding (e.g. one address fragment reused for permanent / billing / communication
address).

---

## Trigger

This skill activates when the developer asks to:
- Create / build / scaffold an **Adaptive Form Fragment** (Core Components)
- Build a **reusable form section / panel** to share across forms
- "Save this panel as a fragment" / "make this group of fields reusable"
- Make a repeatable, referenceable form segment

> Division of labour: for a full submittable **form** use `create-adaptive-form`; for a
> reusable author-governed **page template** use `create-editable-template`; for a custom
> **field type** use `create-form-component`. Use **create-AdaptiveFormFragment** for a
> reusable *content* segment embedded by reference.

---

## Step 1 — Collect inputs from the user

Ask for these before writing anything:

| Input | Description | Example |
|---|---|---|
| `{project}` | The single project namespace used in component `sling:resourceType` paths, under `/conf/{project}`, AND as the folder segment under `/content/forms/af/`, `/content/dam/formsanddocuments/`, `/conf/forms/` (derived from `.aem-forms-config.yaml`) | `aem-demo-site` |
| `{fragmentName}` | Kebab-case node name for the fragment | `address-fragment` |
| `{fragmentTitle}` | Human-readable title shown in the UI | `Address Fragment` |
| `{theme}` | Theme suffix; the theme must exist under `/apps/fd/af/themes/{project}-{theme}` | `wknd` |
| `{template}` | A **FRAGMENT** editable template under `/conf/{project}/settings/wcm/templates/` — its `cq:templateType` is `/libs/settings/wcm/template-types/afv2-fragment-page` and its `structure`/`initial` root `guideContainer` is a **`fragmentcontainer`** carrying **`fd:type="fragment"`**. **Do NOT use the FORM template** (e.g. `blank-af-v2`, whose structure root is a `formcontainer`) — a fragment opened on a form template misbehaves in the AF editor. If no fragment template exists, create one first (see the box below). | `blank-af-v2-fragment` |
| Form model | `none` (default), `schema` (JSON/XSD), or `fdm` — what the fields bind to | `none` |
| Fields & validations | The fields the fragment contains + any per-field rules (from the requirement doc) | see Step 4/5 |

> ⚠️ `{project}` is a **single** namespace. Use it everywhere: component `sling:resourceType`
> values, `/conf/{project}`, AND the folder segment of the content, DAM and `/conf/forms` paths.
> There is NO separate hardcoded app folder — the fragment's DAM asset path
> (`/content/dam/formsanddocuments/{project}/{fragmentName}`) MUST match its page path
> (`/content/forms/af/{project}/{fragmentName}`).

If the user wants to turn an **existing panel of an existing form** into a fragment, read that
panel's XML, lift the `panelcontainer` subtree verbatim into Artifact 1's fragmentcontainer,
then (optionally) replace the panel in the original form with a Fragment reference (Step 7).

> ⚠️ **When lifting a panel, do NOT carry over `validationExpression` / `validateExpMessage` /
> `fd:rules` (`fd:validate`) that call CUSTOM functions** (e.g. `validatePostalCode(...)`,
> `validateName(...)`) unless you ALSO create and wire the fragment's own validation clientlib
> (Artifact 5, `clientLibRef` on the fragmentcontainer). Those custom functions live in the source
> FORM's clientlib; a standalone fragment has no such clientlib, so the function is **undefined** and
> the AF editor / rule layer throws when it loads the fragment — the canvas comes up blank / "not
> working". Either wire the clientlib, or keep only built-in field validation (`required` +
> `mandatoryMessage` + `maxLength`, and built-in picture-clause / pattern validation).

> ### Prerequisite — a FRAGMENT editable template (create once, reuse)
> A fragment MUST be authored on a fragment template, NOT the form template. Check for one under
> `/conf/{project}/settings/wcm/templates/` whose `structure`/`initial` root `guideContainer` is a
> `fragmentcontainer` with `fd:type="fragment"`. If none exists, create `blank-af-v2-fragment`:
> - `.content.xml` → `cq:Template`; `jcr:content` with
>   `cq:templateType="/libs/settings/wcm/template-types/afv2-fragment-page"`, `status="enabled"`.
> - `structure/.content.xml` → `jcr:content` (`sling:resourceType="{project}/components/adaptiveForm/page"`,
>   `guideComponentType="fd/af/templates"`) with a child `guideContainer`:
>   `fd:type="fragment"`, `fd:version="2.1"`, `fieldType="form"`, `editable="{Boolean}true"`,
>   `sling:resourceType="{project}/components/adaptiveForm/fragmentcontainer"`.
> - `initial/.content.xml` → same `guideContainer` (add `schemaType="none"`, `textIsRich="true"`,
>   `sling:configRef="/conf/{project}/forms"`) — **no `actionType`/`thankYou*`** (a fragment never submits).
> - `policies/.content.xml` → map `guideContainer` to the `formcontainer/default` policy and list the
>   allowed field components (panelcontainer, textinput, datepicker, dropdown, emailinput,
>   telephoneinput, text, title).
> - Add a filter entry for the template root and deploy. Then set the fragment page's
>   `cq:template` to it. (This is exactly the pattern the AEM demo-site `blank-af-v2-fragment` uses.)

---

## Step 2 — Artifacts to create

| # | Artifact | File path |
|---|---|---|
| 1 | Fragment page | `ui.content/src/main/content/jcr_root/content/forms/af/{project}/{fragmentName}/.content.xml` |
| 2 | DAM fragment asset | `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/{project}/{fragmentName}/.content.xml` |
| 3 | Per-fragment conf context | `ui.content/src/main/content/jcr_root/conf/forms/{project}/{fragmentName}/.content.xml` |
| 4 | Filter entries | add lines to `ui.content/src/main/content/META-INF/vault/filter.xml` |
| 5 | Validation clientlib *(only when the requirement doc defines field validations)* | `ui.apps/src/main/content/jcr_root/apps/clientlibs/{fragmentName}-clientlib/` |

Generate 1–4 always. Generate 5 when (and only when) the fragment's fields have validation
rules to enforce.

---

## Artifact 1 — Fragment page

A `cq:Page` whose `jcr:content` root container is a **`fragmentcontainer`** (NOT a
`guideContainer`/`formcontainer`). It holds the fragment's panels and fields. **No submit
button. No `actionType`/`thankYou*`.**

**File:** `.../jcr_root/content/forms/af/{project}/{fragmentName}/.content.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:fd="http://www.adobe.com/aemfd/fd/1.0"
    jcr:primaryType="cq:Page">

    <jcr:content
        cq:deviceGroups="[/etc/mobile/groups/responsive]"
        cq:template="/conf/{project}/settings/wcm/templates/{template}"
        jcr:language="en"
        jcr:primaryType="cq:PageContent"
        jcr:title="{fragmentTitle}"
        sling:configRef="/conf/forms/{project}/{fragmentName}/"
        sling:resourceType="{project}/components/adaptiveForm/page">

        <guideContainer
            fd:version="2.1"
            jcr:primaryType="nt:unstructured"
            sling:resourceType="{project}/components/adaptiveForm/fragmentcontainer"
            dorType="none"
            fieldType="panel"
            name="{fragmentNameCamel}"
            schemaType="none"
            textIsRich="true"
            themeRef="/apps/fd/af/themes/{project}-{theme}"
            title="{fragmentTitle}">

            <!-- ADD PANELS AND FIELDS HERE (Step 4). NO submit button. -->

        </guideContainer>

    </jcr:content>
</jcr:root>
```

### Mandatory rules for Artifact 1

- **Keep the root child node named `guideContainer`** (AEM's well-known node name) but give it
  the **fragmentcontainer** `sling:resourceType` and **`fieldType="panel"`**. This is the one
  structural difference from a form — it is what makes the page a fragment, not a form.
- **No submit button, no `actionType`, no `thankYouMessage`/`thankYouOption`.** A fragment does
  not submit; the host form owns submission. Including them produces editor warnings and a
  fragment that behaves like a malformed form.
- **`sling:configRef` MUST end with a trailing `/`** → `/conf/forms/{project}/{fragmentName}/`.
  Without it Sling never resolves the conf context and the theme is silently dropped.
- **`themeRef`** is always `/apps/fd/af/themes/{project}-{theme}` (an `/apps` path, never
  `/content`) and MUST be byte-identical in Artifacts 1, 2 and 3.
- **Use `blank-af-v2` for `cq:template`** (the standard Adaptive Form Core Components blank
  template) — **NEVER `blank-af-v2-fragment`.** Authoring a fragment against
  `blank-af-v2-fragment` breaks the editor canvas and silently drops the theme; every working
  fragment in this project uses `blank-af-v2`, exactly like a form. A page becomes a fragment
  from the **`fragmentcontainer`** resource type on its root child — NOT from a fragment-specific
  template.
- **Every `sling:resourceType` uses `{project}`.**
- **`name`** is a camelCase identifier (e.g. `addressFragment`); it is how host forms address
  the fragment's data.

### Form-model variants (set on the fragmentcontainer)

| Model | Attributes to set on the `guideContainer` (fragmentcontainer) node |
|---|---|
| `none` (default) | `schemaType="none"` (as above) |
| `schema` (JSON/XSD) | `schemaType="jsonSchema"` (or `xsd`), `schemaRef="/content/dam/formsanddocuments/schema/{schema}.json"`. Bind fields with `bindRef="…"`. (Use `generate-schema` to produce the schema first.) |
| `fdm` | `schemaType="formDataModel"`, `dataModelRef="/conf/{project}/settings/cloudconfigs/fdm/{name}"`. Note: an FDM-bound fragment is **single-instance** per form (no multi-bind). Prefer `none`/`schema` when you need to reuse the fragment several times in one form. (Use `create-FDM` first.) |

> **Reusable ACROSS forms → bind to a CANONICAL, schema-agnostic shape.** A fragment is only truly
> reusable if its fields bind to a shared data shape (`$.address.*`, `$.declaration.*`, …), **not** one
> host form's schema paths. Bind it to a single form's schema and embedding it into another form leaves
> those fields **unbound** (the form captures no data); "making it generic" afterwards forces a schema
> migration across every consumer (and can break the original form). Decide the canonical shape up front
> and standardize it across all consuming forms' schemas so the same fragment binds everywhere.

### Panels & fields

Author panels and fields exactly as in a normal form (`panelcontainer` for groups; each field
needs `jcr:title`, `sling:resourceType` = `{project}/components/adaptiveForm/{fieldComponent}`,
`fieldType`, unique `name`, `enabled`/`readOnly`/`visible` booleans, and a `<cq:responsive>`
child with a `<default>` node). Required fields add `required="true"` + `mandatoryMessage`. The
field catalogue, enum syntax, and `cq:responsive` rules are identical to `create-adaptive-form`
— follow that skill's "Fields" section for the per-field XML. Just omit the submit button.

---

## Artifact 2 — DAM fragment asset

Registers the fragment in Forms & Documents and carries the theme reference. Without it the
fragment page exists in the JCR but is invisible in the UI and cannot be picked in the Fragment
component's reference browser.

**File:** `.../jcr_root/content/dam/formsanddocuments/{project}/{fragmentName}/.content.xml`

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
        affragment="1"
        cq:conf="\0"
        jcr:primaryType="dam:AssetContent"
        sling:resourceType="fd/fm/af/render"
        type="affragment">

        <metadata
            fd:version="2.1"
            jcr:language="en"
            jcr:primaryType="nt:unstructured"
            xmp:CreatorTool="AEM Forms AF Wizard"
            allowedRenderFormat="HTML"
            author="admin"
            availableInMobileApp="{Boolean}false"
            dorType="none"
            formmodel="none"
            themeRef="/apps/fd/af/themes/{project}-{theme}"
            title="{fragmentTitle}"/>

    </jcr:content>
</jcr:root>
```

### Mandatory rules for Artifact 2

- **`type="affragment"`** (with the sibling flag **`affragment="1"`**) is the marker that lists
  the asset under *Adaptive Form Fragments* (a form uses `type="guide"`). Using `guide` here makes
  it show up as a form; using `formfragment` or omitting `type` hides it from the fragment picker.
  Both the `type="affragment"` value and the `affragment="1"` flag match what the AEM Forms
  fragment wizard writes — verified in CRXDE against a wizard-created fragment on this instance.
- **`sling:resourceType="fd/fm/af/render"`** is a fixed platform value — do not replace it with
  a project resource type.
- **`themeRef` must byte-match Artifact 1's `guideContainer`.** Copy, don't retype.
- **`title` must match `jcr:title` on the fragment page (Artifact 1).** A mismatch shows two
  different names in different parts of the UI.
- Set `formmodel` to match the chosen model (`none` / `jsonSchema` / `formDataModel`).

> ✅ The fragment asset marker is **`type="affragment"`** plus the sibling **`affragment="1"`**
> flag — confirmed against a wizard-created fragment in CRXDE on this instance
> (`/content/dam/formsanddocuments/…/jcr:content` shows exactly `type="affragment"`,
> `affragment="1"`, `sling:resourceType="fd/fm/af/render"`). Do **not** use `formfragment` — that
> value hides the asset from the *Adaptive Form Fragments* list. If a future AEM build differs,
> re-verify against a fresh wizard-created fragment and note it back to the team.

---

## Artifact 3 — Per-fragment conf context

What `sling:configRef` (Artifact 1) points to. It tells Sling which theme to apply and injects
`theme.css` / `theme.js` into the page head. Without it the fragment opens unstyled (or fails
to open).

**File:** `.../jcr_root/conf/forms/{project}/{fragmentName}/.content.xml`

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
                prefixPath="/content/forms/af/{project}/{fragmentName}.theme/_default">

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

### Mandatory rules for Artifact 3

- Root node has **both** `jcr:primaryType="sling:Folder"` **and**
  `sling:resourceType="sling:Folder"`. Missing either → Sling skips the node, theme not applied.
- `themeArtifact` == `{project}-{theme}`; `siteTemplatePath` == full
  `/apps/fd/af/themes/{project}-{theme}`. Same suffix as `themeRef` in Artifacts 1 & 2.
- **`prefixPath` uses the `{project}` folder segment** → `/content/forms/af/{project}/{fragmentName}.theme/_default`.
  It must match the fragment's page path, or `theme.css`/`theme.js` 404 at runtime.

---

## Artifact 4 — Filter entries

The per-fragment conf context needs its own filter entry or it is excluded from the content
package and never deployed.

In `ui.content/src/main/content/META-INF/vault/filter.xml` add:

```xml
<filter root="/conf/forms/{project}/{fragmentName}" mode="update"/>
```

Verify these roots are already covered (add if absent):
- `/content/forms/af/{project}` — covers the fragment page
- `/content/dam/formsanddocuments/{project}` — covers the DAM asset

If you create the clientlib (Artifact 5), ensure the **`ui.apps`** filter
(`ui.apps/src/main/content/META-INF/vault/filter.xml`) covers `/apps/clientlibs`:
```xml
<filter root="/apps/clientlibs"/>
```
(The default `ui.apps` filter covers `/apps/{project}/clientlibs`, NOT `/apps/clientlibs`.)

---

## Artifact 5 — Validation clientlib (only when fields have validations)

If the fragment's fields carry validation rules (formats, lengths, ranges, cross-field
constraints from the requirement doc), the fragment ships its own validation clientlib — exactly
as a form does.

> ⚠️ **ORDER MATTERS — do the clientlib LAST, by INVOKING `create-form-clientlib`.** First
> finish Artifacts 1–4 so the fragment fully exists with its theme and template, then invoke
> `create-form-clientlib`. Do NOT hand-write `functions.js` interleaved with the fragment —
> the Rule Editor's scanner only discovers globally-scoped functions, and a bundled pass drops
> the function-authoring discipline, leaving the Rule Editor's Form Objects tree empty.

When you invoke `create-form-clientlib` for a fragment:
- Wire the clientlib to the fragment by setting `clientLibRef="{project}.forms.{fragmentName}"`
  on the **fragmentcontainer** node (Artifact 1) — the same hook a formcontainer uses.
- The clientlib is created at `/apps/clientlibs/{fragmentName}-clientlib/`.
- All of `create-form-clientlib`'s scanner-critical rules apply unchanged: global-scope
  functions only (no IIFE / namespace), escape EVERY `/` inside regex literals (including in
  `[...]` char classes — write `\/`), empty-safe functions (`return true` on empty), json-formula
  booleans are `true()`/`false()` (never bare `true`), and each validated field gets
  `validationExpression="<fn>(...) == true()"` + `validateExpMessage` plus the `fd:validate` AST
  for editor visibility.

> Note: validations authored on the fragment travel with it into every host form — that is the
> point of a fragment. Don't duplicate them in the host form.

---

## Step 7 — Embedding the fragment into a form (document this to the user)

A fragment delivers value only once a form references it. To embed it, add a **Fragment**
component inside a form's `guideContainer` (or a panel) and point its `fragmentPath` at the
fragment page:

```xml
<addressFragment
    jcr:primaryType="nt:unstructured"
    jcr:title="{fragmentTitle}"
    sling:resourceType="{project}/components/adaptiveForm/fragment"
    fieldType="panel"
    fragmentPath="/content/forms/af/{project}/{fragmentName}"
    name="addressFragment"
    enabled="{Boolean}true"
    visible="{Boolean}true"
    readOnly="{Boolean}false"
    repeatable="{Boolean}false"
    wrapData="{Boolean}false"
    hideTitle="{Boolean}false">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
    </cq:responsive>
</addressFragment>
```

Key points for the reference component:
- **`sling:resourceType="{project}/components/adaptiveForm/fragment"`** (the placeholder proxy),
  `fieldType="panel"`.
- **`fragmentPath`** is the path to the fragment PAGE (`/content/forms/af/...`), NOT the DAM
  asset. This is the actual JCR property behind the dialog's "Fragment reference" field.
- To embed the **same** fragment more than once in one form, add multiple Fragment nodes with
  **distinct `name`** values, and (for schema/none models) give each a different `bindReference`
  so instances bind to different data (e.g. permanent vs billing address). FDM-bound fragments
  cannot be multi-instanced.
- For a repeating fragment set `repeatable="{Boolean}true"` and add `minOccur`/`maxOccur`.
- The placeholder is **not editable inside the host form** — to change the fragment, edit the
  standalone fragment; changes propagate to every host form.
- Embedding via authoring is also fine (drag the *Adaptive Form Fragment* component, pick the
  reference in the Basic tab). Hand-author the node only when scripting the host form.

**When replacing an existing inline section with the fragment:**
- **Preserve field parity** — the embedded fragment must render the same fields, labels, order, and
  required flags the inline section had. If the shared fragment cannot represent the section without
  dropping or renaming a field, **report the mismatch** rather than silently diverging (and never
  break another form that already consumes the fragment).
- **Purge orphan inline nodes** — on an ALREADY-DEPLOYED form, removing the inline section leaves its
  old field nodes in the repository; a plain package update will NOT delete them and the form
  **double-renders**. List the exact orphan JCR paths so the deploy removes them (vault filter
  replace-mode on the form root, or explicit node deletion), and verify the old paths return **404**
  after deploy.

---

## Constraints (from Adobe docs — surface these to the user)

- A fragment **cannot be edited from within a host form** — edit the standalone fragment.
- Fragment **names must be unique**.
- Publishing a form does **not** auto-publish its referenced fragments — **publish the standalone
  fragment separately**, or the published form renders an empty placeholder.
- **Republishing** an updated standalone fragment propagates the changes to every published
  form that references it ("change once, reflect everywhere").
- The **Verify** component is **not recommended** inside a fragment (and a form containing
  Verify does not support anonymous users).
- **none-based and schema-based** fragments can be reused **multiple times** in one form, each
  instance bound independently (different `bindReference`). An **FDM-based** fragment is built
  on a **single data model object** — prefer none/schema when you need the same fragment
  embedded several times.
- **Auto-mapping:** a fragment built on a JSON Schema definition is auto-reused in forms
  created from the same schema — dragging the matching schema object inserts the fragment.
- Fragments support **nesting** (a fragment may reference another fragment).

---

## Failure symptoms and causes

| Symptom | Cause | Fix |
|---|---|---|
| Fragment not visible in Forms & Documents / fragment picker | Artifact 2 missing, or `type` != `affragment` (e.g. `formfragment`), or `affragment="1"` flag absent | Create Artifact 2 with `type="affragment"` and `affragment="1"` |
| Shows up as a *form*, not a *fragment* | `type="guide"` used on the DAM asset | Set `type="affragment"` (+ `affragment="1"`) |
| Opens unstyled | Artifact 3 missing, or `sling:configRef` missing trailing `/` | Create Artifact 3; add trailing `/` |
| Theme loads but wrong style | `themeRef` (Art. 1/2) != `themeArtifact` (Art. 3) | Make all three the same `{project}-{theme}` |
| `theme.css`/`theme.js` 404 | `prefixPath` does not match the fragment's `/content/forms/af/{project}/{fragmentName}` path | Correct `prefixPath` |
| Host form shows empty placeholder at runtime | Fragment not published, or `fragmentPath` wrong / points at the DAM asset | Publish the fragment; point `fragmentPath` at `/content/forms/af/...` |
| Fragment behaves like a broken form / submit errors | Submit button or `actionType` left on the fragment | Remove them — fragments don't submit |
| Fragment opens BLANK / "UI not working" in the AF editor (`editor.html`) | Fragment authored on the FORM template (`blank-af-v2`, structure root = `formcontainer`) instead of a fragment template | Point `cq:template` at a fragment template (`afv2-fragment-page` type, `fragmentcontainer` root + `fd:type="fragment"`); create `blank-af-v2-fragment` if absent |
| Fragment blank in editor / rule errors, though it renders standalone | Lifted-in `validationExpression`/`fd:validate` calls a CUSTOM function with no clientlib wired on the fragment → function undefined at load | Remove the custom-function validation, or add + wire the fragment's own validation clientlib (Artifact 5, `clientLibRef`) |
| Components 404 on resourceType | a `sling:resourceType` uses the wrong namespace | Audit every `sling:resourceType` — all use `{project}` |
| Rule Editor "Form Objects" empty | Validation clientlib hand-authored instead of via `create-form-clientlib`; regex/global-scope rules broken | Re-run `create-form-clientlib`; check regex `/`-escaping and global scope |
| Package install fails with XML error | Undeclared namespace prefix on a root node | Add the missing `xmlns:...` |

---

## Pre-flight checklist

- [ ] All inputs confirmed: `{project}`, `{fragmentName}`, `{fragmentTitle}`, `{theme}`, `{template}`, form model, fields/validations
- [ ] `{project}` is used as the single namespace in every path root (resourceType, `/conf/`, and the content/DAM/conf-forms folder segment)
- [ ] Artifact 1: fragment page at `/content/forms/af/{project}/{fragmentName}/.content.xml`
- [ ] Root container is `fragmentcontainer` (`{project}/components/adaptiveForm/fragmentcontainer`) with `fieldType="panel"` — NOT `formcontainer`
- [ ] `cq:template` points at a **FRAGMENT** template (`afv2-fragment-page` type, `fragmentcontainer` root + `fd:type="fragment"`) — NOT the form template; the template exists (create `blank-af-v2-fragment` if absent)
- [ ] No lifted-in custom-function `validationExpression`/`fd:validate` without a wired validation clientlib (else the editor loads blank)
- [ ] **No submit button, no `actionType`, no `thankYou*`** on the fragment
- [ ] `sling:configRef` ends with trailing `/`
- [ ] Artifact 2: DAM asset at `/content/dam/formsanddocuments/{project}/{fragmentName}/.content.xml` with `type="affragment"`, the `affragment="1"` flag, and `sling:resourceType="fd/fm/af/render"`
- [ ] Artifact 3: conf context with both `jcr:primaryType="sling:Folder"` and `sling:resourceType="sling:Folder"`; `prefixPath` uses the `{project}` folder segment
- [ ] `themeRef` (Art. 1 & 2) and `themeArtifact`/`siteTemplatePath` (Art. 3) all reference the same `{project}-{theme}`
- [ ] Artifact 4: filter entry for `/conf/forms/{project}/{fragmentName}` added; `/content/forms/af/{project}` and `/content/dam/formsanddocuments/{project}` covered
- [ ] Every field has `name`, `jcr:title`, `fieldType`, `<cq:responsive>/<default>`; required fields have `required="true"` + `mandatoryMessage`
- [ ] Artifact 5 (only if validations): `create-form-clientlib` invoked LAST; `clientLibRef="{project}.forms.{fragmentName}"` on the fragmentcontainer; `/apps/clientlibs` in the `ui.apps` filter
- [ ] All namespace prefixes declared on each file's `<jcr:root>`; all booleans use `{Boolean}true`/`{Boolean}false`; multi-value attrs use `[...]`
- [ ] Documented to the user how to embed via the Fragment component (`fragmentPath`) and that the fragment must be published separately

---

## Deployment & verification

> **Pipeline mode (delegated by `formwright`): SKIP the build/deploy below — author artifacts only.**
> Deployment is centralized in the `forgemaster` lead (AGENTS.md → "Deployment is centralized in Forgemaster").
> Run the `mvn` commands only when this skill is invoked **standalone / directly**.

```bash
# Content package only (fastest for fragment content)
mvn clean install -PautoInstallSinglePackage -pl ui.content

# Full build (needed when Artifact 5 clientlib was added under ui.apps)
mvn clean install -PautoInstallSinglePackage
```

After deploy, verify on the running instance (default `http://localhost:4502`):

1. **Forms & Documents** lists the new asset under **Adaptive Form Fragments** (not under Forms).
   If it's missing or mis-typed → re-check Artifact 2's `type="affragment"` (+ `affragment="1"`).
2. Open the fragment in the AF editor — it loads without errors and is themed.
3. In any Adaptive Form, add the **Adaptive Form Fragment** component → its reference browser
   lists this fragment → select it → the panel/fields appear as a non-editable placeholder.
4. (If validations) open the fragment's Rule Editor — the Form Objects tree populates with the
   rules; the clientlib serves at `/etc.clientlibs/clientlibs/{fragmentName}-clientlib.js`.
5. Publish the fragment, then publish a host form — the published form renders the fragment's
   fields (not an empty placeholder).

> Treat this skill's output as a draft until verified on the running instance — especially the
> Artifact 2 `type` marker (confirm it against a wizard-created fragment in CRXDE).
