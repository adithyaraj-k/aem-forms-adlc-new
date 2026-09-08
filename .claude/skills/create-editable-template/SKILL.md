---
name: create-editable-template
description: >
  Creates an AEM Editable Template for Adaptive Forms on AEM as a Cloud Service
  Core Components. Builds the editable template under /conf based on the project's
  Adaptive Form (Core Components) page template-type (af-page-v2): generates the
  cq:Template node, initial content (header / form container / footer), structure,
  content-policy mappings, and the ui.content filter entries. Use when you need a
  reusable, author-governed form template.
version: 2.0.0
ide:
  cursor: .cursor/skills/create-editable-template/
  github-copilot: .github/skills/create-editable-template/
  claude-code: .claude/skills/create-editable-template/
  vscode: .vscode/skills/create-editable-template/
---

# Skill: create-editable-template

## Role

You are an expert AEM Adaptive Forms developer specialising in AEM as a Cloud Service.
You create **Editable Templates** (the modern, author-governed template model — never
static `cq:Template` page-component templates). Every Adaptive Form template you build
is based on the project's **Adaptive Form (Core Components) page template-type**
(`af-page-v2`) and uses **Core Components** form resource types — proxied by this
project under `{project}/components/...`.

Before generating any file, read `.aem-forms-config.yaml` at the project root to load
project-specific settings (`project`, `package`, `group`, `defaultTheme`, `formsContentRoot`).
If the file is missing, ask the developer to confirm the values first.

---

## Trigger

This skill activates when the developer asks to:
- Create / build / scaffold an **editable template** for Adaptive Forms
- Create a reusable form template authors can use in the "Create Form" wizard
- Set up a template based on the `af-page-v2` template-type
- Add a layout for forms (header, footer, form container)

---

## PRECONDITION — reuse-first gate (run this BEFORE generating anything)

> 🛑 **Do NOT create a new editable template per form. FIRST evaluate whether an existing template
> can be reused.** This project already has ~20 templates and forms have been needlessly forking a
> new one each time — that stops here. Before you author a single node, run this gate:
>
> 1. **Enumerate existing templates** under `/conf/{project}/settings/wcm/templates/` — glob the repo
>    AND check the running instance (`http://localhost:4502`). (Examples already present: `basic-af`,
>    `blank-af`, `blank-af-v2`, plus many form-specific ones.)
> 2. A template is **REUSABLE** when: it is built on the same **template-type** (`af-page-v2` / Core
>    Components); its **structure** (header / form container / footer, locked vs unlocked regions)
>    fits the form; and its **content policy** allows the components the form needs. A generic base
>    (`blank-af-v2` / `basic-af`) is broadly reusable. **Per-form theme/brand differences do NOT
>    disqualify reuse** — the theme is a per-form theme + clientlib + conf context, not the template.
> 3. **A suitable existing template is found → STOP. Do NOT create a template.** Instruct the caller
>    to REUSE it: set the form's `cq:template` reference to that template's `/conf` path (this is a
>    `create-adaptive-form` input, not a new template). Skip the rest of this skill.
> 4. **Only when NO existing template fits** (the form needs a distinct locked structure, a distinct
>    allowed-components policy, or a governed layout for a new form family) do you proceed below and
>    create a new editable template.
> 5. **Record the decision** (`reused: <name>` | `created: <name>` + reason) in the run
>    implementation notes.
>
> In a pipeline delivery the architect (`architect-form-solution`) has usually already made this
> call — honor it: if the plan says `template: reuse:"<path>" (skip create-editable-template)`, do
> NOT run this skill.

---

## Key concept — template-type vs editable template

| Thing | Path | Who owns it | Editable? |
|-------|------|-------------|-----------|
| **Template-type** | `/conf/{project}/settings/wcm/template-types/af-page-v2` | Your project (seeded from the AEM Forms Core Components template-type) | No — it is the blueprint |
| **Editable template** | `/conf/{project}/settings/wcm/templates/{templateName}` | Your project | Yes — this is what you create |

The `af-page-v2` template-type is the **blueprint** for Adaptive Form Core Components
pages. When an author creates a new editable template in the Templates console and picks
the Adaptive Form type, AEM copies the template-type's skeleton into
`/conf/.../templates/`. This skill reproduces that result as committable source code so
the template is delivered through the build, not hand-authored on each environment.

> The project references the template-type via `cq:templateType`. Do **not** author new
> components against Adobe's `/libs` — extend the Core Components via the project proxies
> under `{project}/components/...` (`sling:resourceSuperType` to
> `core/fd/components/...`).

---

## Confirmation step (always do this first)

Before generating any files, echo back:

```
Template name      : {templateName}            (kebab-case node name, e.g. editable-template)
Template title     : {Template Display Title}  (shown in Create Form wizard)
Based on type      : /conf/{project}/settings/wcm/template-types/af-page-v2
Status             : {enabled | draft}          (enabled = usable immediately)
Default theme      : {defaultTheme from config}
Layout             : {header + form container + footer | form container only}
```

Wait for the developer to confirm or correct before proceeding.

---

## Files to generate

Editable templates live under `ui.content`. Generate ALL of the following:

```
ui.content/src/main/content/jcr_root/conf/{project}/settings/wcm/templates/{templateName}/
├── .content.xml                       ← cq:Template node + cq:PageContent (title, status, cq:templateType)
├── initial/
│   └── .content.xml                   ← initial page content (header, form container, footer)
├── structure/
│   └── .content.xml                   ← structural editable areas (header / form container / footer)
├── policies/
│   └── .content.xml                   ← content-policy mappings for structure components
└── thumbnail.png                      ← (optional) template thumbnail; note if not generated
```

Also update the ui.content filter so the template path is included in the package:

```
ui.content/src/main/content/META-INF/vault/filter.xml   ← add a <filter root=".../templates/{templateName}"/>
```

> The template-type (`/conf/{project}/settings/wcm/template-types/af-page-v2`) is **not**
> generated by this skill — it is seeded with the project. You only reference it via
> `cq:templateType`.

---

## 1. Template root — .content.xml

The `cq:Template` node and its `cq:PageContent`. `status="enabled"` makes the template
immediately selectable. `cq:templateType` binds it to the project's `af-page-v2`
template-type.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="cq:Template">
    <jcr:content
        author="{project}"
        cq:templateType="/conf/{project}/settings/wcm/template-types/af-page-v2"
        jcr:primaryType="cq:PageContent"
        jcr:title="{Template Display Title}"
        jcr:description="{One-line description of this form template}"
        status="enabled"/>
</jcr:root>
```

Notes:
- The template's `jcr:content` is **`cq:PageContent`** (not `nt:unstructured`).
- `status` — `enabled` (ready to use), `draft` (still editing), or `disabled`. Draft
  templates do not appear in the Create Form wizard.
- `cq:templateType` — **must** be `/conf/{project}/settings/wcm/template-types/af-page-v2`.
  This binds the editable template to the project's Adaptive Form Core Components type.

---

## 2. initial/.content.xml — initial content

The content created when an author starts a form from this template. `cq:template` points
back at the template, and `sling:resourceType` is the project's Adaptive Form page proxy.
This contains the authored defaults: a header, the form container, and a footer.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:fd="http://www.adobe.com/aemfd/fd/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="cq:Page">
    <jcr:content
        cq:deviceGroups="[/etc/mobile/groups/responsive]"
        cq:template="/conf/{project}/settings/wcm/templates/{templateName}"
        jcr:primaryType="cq:PageContent"
        sling:resourceType="{project}/components/adaptiveForm/page"
        sling:configRef="/conf/{project}/forms"
        guideComponentType="fd/af/templates">
        <container1
            jcr:primaryType="nt:unstructured"
            sling:resourceType="{project}/components/container"
            layout="responsiveGrid">
            <pageheader
                jcr:primaryType="nt:unstructured"
                jcr:title="Header"
                sling:resourceType="{project}/components/adaptiveForm/pageheader"
                fieldType="page-header">
                <text
                    jcr:primaryType="nt:unstructured"
                    sling:resourceType="{project}/components/text"
                    text="&lt;p>{Template Display Title}&lt;/p>&#xd;&#xa;"
                    textIsRich="true"
                    value="Company Name Here"/>
            </pageheader>
        </container1>
        <guideContainer
            fd:version="2.1"
            actionType="fd/af/components/guidesubmittype/restendpoint"
            fieldType="form"
            schemaType="none"
            textIsRich="true"
            thankYouMessage="Thank you for submitting the form."
            thankYouOption="page"
            jcr:primaryType="nt:unstructured"
            sling:resourceType="{project}/components/adaptiveForm/formcontainer"/>
        <container2
            jcr:primaryType="nt:unstructured"
            sling:resourceType="{project}/components/container"
            layout="responsiveGrid">
            <footer
                jcr:primaryType="nt:unstructured"
                jcr:title="Footer"
                sling:resourceType="{project}/components/adaptiveForm/footer">
                <text
                    jcr:primaryType="nt:unstructured"
                    jcr:title="Footer"
                    sling:resourceType="{project}/components/text"
                    css="footerText"
                    text="&lt;p>© YYYY Company Name | All rights reserved.&lt;/p>&#xd;&#xa;"
                    textIsRich="true"/>
            </footer>
        </container2>
    </jcr:content>
</jcr:root>
```

Notes:
- The `fd` namespace is `http://www.adobe.com/aemfd/fd/1.0`.
- The Adaptive Form container node is named **`guideContainer`** and carries
  `fd:version="2.1"`, `fieldType="form"`, and a `sling:resourceType` proxy
  (`{project}/components/adaptiveForm/formcontainer`).
- `guideComponentType="fd/af/templates"` marks this as an AF template page.

---

## 3. structure/.content.xml — structure

The structural definition of the template. Components here define the editable areas
authors work within. The header / form-container / footer areas are marked
`editable="{Boolean}true"` so authors can edit them; remove `editable` (or set it to
`false`) on any area you want locked.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:fd="http://www.adobe.com/aemfd/fd/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="cq:Page">
    <jcr:content
        cq:deviceGroups="[/etc/mobile/groups/responsive]"
        cq:template="/conf/{project}/settings/wcm/templates/{templateName}"
        jcr:primaryType="cq:PageContent"
        sling:resourceType="{project}/components/adaptiveForm/page"
        guideComponentType="fd/af/templates">
        <container1
                jcr:primaryType="nt:unstructured"
                sling:resourceType="{project}/components/container"
                layout="responsiveGrid"
                editable="{Boolean}true" />
        <guideContainer
            fd:version="2.1"
            fieldType="form"
            jcr:primaryType="nt:unstructured"
            editable="{Boolean}true"
            sling:resourceType="{project}/components/adaptiveForm/formcontainer"/>
        <container2
                jcr:primaryType="nt:unstructured"
                sling:resourceType="{project}/components/container"
                layout="responsiveGrid"
                editable="{Boolean}true" />
        <cq:responsive jcr:primaryType="nt:unstructured">
            <breakpoints jcr:primaryType="nt:unstructured">
                <phone
                        jcr:primaryType="nt:unstructured"
                        title="Smaller Screen"
                        width="{Long}768"/>
                <tablet
                        jcr:primaryType="nt:unstructured"
                        title="Tablet"
                        width="{Long}1200"/>
            </breakpoints>
        </cq:responsive>
    </jcr:content>
</jcr:root>
```

Locking rules:
- `editable="{Boolean}true"` on an area lets authors add/edit components there.
- To lock an area (authors cannot edit it), set `editable="{Boolean}false"`.
- The structure node names (`container1`, `guideContainer`, `container2`) must match the
  names used in `initial/.content.xml` and the `policies` mappings.

---

## 4. policies/.content.xml — content policy mappings

Maps each structural area to a content policy (allowed components, default props).
Without a mapping, the form container has no allowed components and authors cannot add
fields. Node names must match the structure.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="cq:Page">
    <jcr:content
        cq:policy="{project}/components/page/policy"
        jcr:primaryType="nt:unstructured"
        sling:resourceType="wcm/core/components/policies/mappings">
        <container1
                jcr:primaryType="nt:unstructured"
                sling:resourceType="wcm/core/components/policies/mapping"
                cq:policy="{project}/components/container/cc-af-header-policy"
                layout="responsiveGrid"/>
        <guideContainer
            cq:policy="{project}/components/adaptiveForm/formcontainer/default"
            jcr:primaryType="nt:unstructured"
            sling:resourceType="wcm/core/components/policies/mapping">
            <{project} jcr:primaryType="nt:unstructured">
                <components jcr:primaryType="nt:unstructured">
                    <adaptiveForm jcr:primaryType="nt:unstructured">
                        <numberinput
                          cq:policy="{project}/components/adaptiveForm/numberinput/default"
                          jcr:primaryType="nt:unstructured"
                          sling:resourceType="wcm/core/components/policies/mapping"/>
                        <datepicker
                          cq:policy="{project}/components/adaptiveForm/datepicker/default"
                          jcr:primaryType="nt:unstructured"
                          sling:resourceType="wcm/core/components/policies/mapping"/>
                        <telephoneinput
                          cq:policy="{project}/components/adaptiveForm/telephoneinput/default"
                          jcr:primaryType="nt:unstructured"
                          sling:resourceType="wcm/core/components/policies/mapping"/>
                        <panelcontainer
                          cq:policy="{project}/components/adaptiveForm/panelcontainer/default"
                          jcr:primaryType="nt:unstructured"
                          sling:resourceType="wcm/core/components/policies/mapping"/>
                        <switch
                          cq:policy="{project}/components/adaptiveForm/switch/default"
                          jcr:primaryType="nt:unstructured"
                          sling:resourceType="wcm/core/components/policies/mapping"/>
                    </adaptiveForm>
                </components>
            </{project}>
        </guideContainer>
        <container2
                jcr:primaryType="nt:unstructured"
                sling:resourceType="wcm/core/components/policies/mapping"
                cq:policy="{project}/components/container/cc-af-footer-policy"
                layout="responsiveGrid"/>
    </jcr:content>
</jcr:root>
```

> `<{project}>` is a literal element named after the project (substitute the `{project}`
> value, e.g. `my-forms-app`).
> The nested `{project}/components/adaptiveForm/*` mappings declare which Core Component
> fields authors may drop inside the form container. The referenced policies live under
> `/conf/{project}/settings/wcm/policies/...`; create matching `default` policy nodes there
> (resourceType `wcm/core/components/policies/Policy`) if they do not yet exist.

---

## 5. filter.xml — include the template in the package

Add (do not replace existing entries) to
`ui.content/src/main/content/META-INF/vault/filter.xml`:

```xml
<filter root="/conf/{project}/settings/wcm/templates/{templateName}"/>
<filter root="/conf/{project}/settings/wcm/policies" mode="merge"/>
```

Use `mode="merge"` for the policies root so deployments don't clobber sibling policy
nodes written by other skills.

---

## Common mistakes to avoid

- **NEVER** point `cq:templateType` at `/libs/...` — bind it to the project's
  `/conf/{project}/settings/wcm/template-types/af-page-v2`.
- **NEVER** set the template root `jcr:content` to `nt:unstructured` — it must be
  `cq:PageContent`.
- **NEVER** invent `fd` properties (`fd:formType`) or namespaces — the AF namespace is
  `http://www.adobe.com/aemfd/fd/1.0`, the container is `guideContainer` with
  `fd:version="2.1"` and `fieldType="form"`.
- **NEVER** author new components against Adobe `/libs` — use the project proxies under
  `{project}/components/...` (which `sling:resourceSuperType` the Core Components).
- **NEVER** create a static (page-component) template — always editable templates under
  `/conf/{project}/settings/wcm/templates/`.
- **NEVER** leave `status="draft"` if the developer asked for a usable template — draft
  templates do not appear in the Create Form wizard.
- **NEVER** forget the policy mappings — without them the form container has no allowed
  components and authors cannot add fields.
- **ALWAYS** keep node names (`container1`, `guideContainer`, `container2`) consistent
  across `initial`, `structure`, and `policies`.
- **ALWAYS** keep `cq:template` (in initial + structure) pointing at the template's own
  `/conf` path.

---

## Quality checklist — verify before finishing

- [ ] **Reuse-first gate ran FIRST** — existing templates were enumerated and evaluated; no suitable
      one was reusable (a new template is justified). If one WAS reusable, this skill should not have
      run — the form reuses it instead. The reuse-vs-create decision is recorded.
- [ ] Template node is `jcr:primaryType="cq:Template"` under `/conf/{project}/settings/wcm/templates/`
- [ ] Template `jcr:content` is `jcr:primaryType="cq:PageContent"`
- [ ] `jcr:content/@cq:templateType` = `/conf/{project}/settings/wcm/template-types/af-page-v2`
- [ ] `status="enabled"` (unless developer explicitly wants a draft)
- [ ] `initial` and `structure` both set `cq:template` to this template's `/conf` path
- [ ] Page `sling:resourceType` is `{project}/components/adaptiveForm/page` and `guideComponentType="fd/af/templates"`
- [ ] `fd` namespace is `http://www.adobe.com/aemfd/fd/1.0`; container node is `guideContainer` with `fd:version="2.1"` + `fieldType="form"`
- [ ] Node names match across `initial`, `structure`, and `policies` (`container1`/`guideContainer`/`container2`)
- [ ] `policies/.content.xml` maps the `guideContainer` to a policy listing the allowed AF field components
- [ ] `filter.xml` includes the template path (and policies path, `mode="merge"`)
- [ ] No `/libs` paths authored/overwritten; components use project proxies

---

## Example

See `references/example-template.md` for a complete generated editable template
("Editable Template") including all `.content.xml` files and the filter entries, using the
a sample project's values (substitute your own `{project}`).
