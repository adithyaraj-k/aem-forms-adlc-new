---
name: composer
description: >
  Embeds a created AEM Adaptive Form (Core Components, AEM as a Cloud Service) into an AEM Sites
  page — the fixed demo/showcase page named "Test Adaptive Form" — using the AEM Form Container
  component. The newly generated form REPLACES whatever form was previously embedded on that page:
  the page is reused, its single AEM Form Container's form reference is repointed to the new form,
  and no second container is ever added. Produces the site page (created once, reused thereafter),
  the embedded AEM Form Container node, and ensures the page path is in the ui.content filter so it
  deploys. Use whenever a form must be surfaced on a Sites page, or when the request is to embed /
  place / show / host / mount an adaptive form on the "test adaptive form" page. Runs after the form
  is built+wired and BEFORE the build/deploy (forgemaster), so forgemaster deploys the updated page too and
  sentinel tests the form as it renders inside the page.
version: 1.0.0
ide:
  cursor: .cursor/skills/composer/
  github-copilot: .github/skills/composer/
  claude-code: .claude/skills/composer/
---

# Skill: composer

## Role

You are an AEM Sites + Adaptive Forms author for AEM as a Cloud Service. When asked to embed a
form, you take the **already-created** Adaptive Form and surface it on a **single, persistent AEM
Sites page named "Test Adaptive Form"** by authoring an **AEM Form Container** component that points
at the form. The page is a stable showcase: every new form generated in this project is embedded
here, **replacing** the previously embedded form. The output of this skill is always the same page
node (`test-adaptive-form`) now rendering the **new** form.

You never author the form itself (that is `create-adaptive-form`). You never create a second page or
a second container. You never invent a form path — you use the exact path of the form built earlier
in the delivery.

> **Replace, don't accumulate.** The page has **exactly one** AEM Form Container. Embedding a new
> form means **repointing that one container's form reference** to the new form path — not adding
> another container and not leaving the old form referenced. The result is the "Test Adaptive Form"
> page showing the newly generated form and nothing else.

---

## Step 1 — Collect inputs

Read these from the delivery (the Program Agent / Formwright pass them in the handoff) or ask the
user if invoked standalone:

| Input | Source | Default |
|-------|--------|---------|
| `{formPath}` — full JCR path of the form to embed | `create-adaptive-form` / `formwright.md` | `{formsContentRoot}/{project}/{formName}` |
| `{formName}` — kebab-case form node name | the built form | — |
| `{sitePagePath}` — the showcase page path | project convention (below) | `{siteRoot}/test-adaptive-form` |
| `{pageTitle}` — page title | fixed | `Test Adaptive Form` |

Project tokens (from `.aem-forms-config.yaml`):
- `{project}` = `aem-demo-site` — the single project namespace, used in `sling:resourceType`, `/conf/`, AND as the folder segment under `/content/forms/af/…`, `/content/dam/formsanddocuments/…`, `/conf/forms/…`
- `{formsContentRoot}` = `/content/forms/af`
- `{siteRoot}` = `/content/{project}/us/en` (the existing project Sites tree)
- `{pageTemplate}` = `/conf/{project}/settings/wcm/templates/page-content`
- `{pageComponent}` = `{project}/components/page`

> ⚠️ The folder segment under `/content/forms/af/`, `/content/dam/formsanddocuments/`, and
> `/conf/forms/` is `{project}` — the SAME token used for the page, template and components. The form
> lives under `{formsContentRoot}/{project}/{formName}`. There is no separate hardcoded app folder;
> a form's DAM guide-asset path (`/content/dam/formsanddocuments/{project}/{formName}`) MUST match its
> `/content/forms/af/{project}/{formName}` path. A mismatched folder is the #1 cause of a blank /
> "Form not found" embed.

The page node name is **always** `test-adaptive-form`; the page always lives at
`{siteRoot}/test-adaptive-form`. This is fixed so the embed target is stable across deliveries.

---

## Step 2 — Understand what you must produce

Two artifacts (plus one filter check):

1. **The showcase Sites page** — `{siteRoot}/test-adaptive-form/.content.xml`, a `cq:Page` using the
   project page template + page component. **Create it only if it does not already exist**; if it
   exists, reuse it untouched except for the container repoint in Artifact 2.
2. **The AEM Form Container** — one node inside the page content whose `sling:resourceType` is the
   project's AEM Form Container proxy and whose form reference points at `{formPath}`.
3. **Filter coverage** — confirm the page path is inside a `ui.content` filter root (it already is,
   under `/content/{project}`); add an explicit entry only if it is not.

---

## Artifact 1 — The "Test Adaptive Form" Sites page

Path: `ui.content/src/main/content/jcr_root/content/{project}/us/en/test-adaptive-form/.content.xml`

Author it modelled on the existing site page (`{siteRoot}/.content.xml`) — same template, page
component, conf context. **CRITICAL — node depth:** the `page-content` template's editable content
drop-zone is the **NESTED** node `root/container/container` (its structure `.content.xml` marks that
inner container `editable="{Boolean}true"`). The outer `root/container` is a **locked structure**
node (it holds the header XF, the `title`, the inner editable container, and the footer XF in a fixed
render order). A component authored as a direct child of `root/container` **deploys to the JCR but is
NEVER rendered** — the page shows an empty content area. **The AEM Form Container MUST be authored
inside the editable container `root/container/container`**, not at `root/container`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="cq:Page">
    <jcr:content
        jcr:primaryType="cq:PageContent"
        jcr:title="Test Adaptive Form"
        pageTitle="Test Adaptive Form"
        cq:conf="/conf/{project}"
        sling:configRef="/conf/{project}"
        cq:template="/conf/{project}/settings/wcm/templates/page-content"
        sling:resourceType="{project}/components/page"
        formEmbedThemePath="/content/forms/af/{project}/{formName}"
        formEmbedClientlibs="[{project}.forms.{formName}]">
        <root
            jcr:primaryType="nt:unstructured"
            sling:resourceType="{project}/components/container"
            layout="responsiveGrid">
            <container
                jcr:primaryType="nt:unstructured"
                sling:resourceType="{project}/components/container"
                layout="responsiveGrid">
                <container
                    jcr:primaryType="nt:unstructured"
                    sling:resourceType="{project}/components/container"
                    layout="responsiveGrid">
                    <!-- Artifact 2 — the single AEM Form Container. MUST be here (root/container/container),
                         the template's EDITABLE container, NOT one level up at root/container.
                         useiframe="false" (inline) + the two formEmbed* page props above = full fidelity. -->
                    <adaptiveFormEmbed
                        jcr:primaryType="nt:unstructured"
                        sling:resourceType="{project}/components/aemformscontainer"
                        formType="af"
                        loadType="embed"
                        useiframe="false"
                        formRef="/content/dam/formsanddocuments/{project}/{formName}"/>
                </container>
            </container>
        </root>
    </jcr:content>
</jcr:root>
```

> **Verify the drop-zone against the template before authoring.** Read the template structure at
> `ui.content/.../conf/{project}/settings/wcm/templates/page-content/structure/.content.xml` and author
> the embed inside the node marked `editable="{Boolean}true"` (here `root/container/container`). If the
> project's template nests the editable container at a different depth, match THAT depth — never drop the
> embed on a locked structure node. Symptom of getting this wrong: page returns HTTP 200 but the content
> area is empty and no AF runtime clientlib loads.

### Mandatory attribute rules for Artifact 1
- `sling:resourceType` on the page **must** be `{project}/components/page` (the
  project page component), never a Foundation/`wcm/foundation` type.
- Reuse the project page template `{pageTemplate}`; do not fork a new template — the site's
  `cq:allowedTemplates` already permits `page-content`.
- Set `cq:conf` and `sling:configRef` to `/conf/{project}` so the page resolves the
  project context (theme/clientlibs) exactly like the existing site page.
- If the page already exists, **do not** rewrite it — only update the container's `formRef` (Step 3).

---

## Artifact 2 — The AEM Form Container (the embed)

The embed is a single component node inside the page's `container`. Use the project's AEM Form
Container proxy so the embed inherits the Core Component behaviour:

- `sling:resourceType` = **`{project}/components/aemformscontainer`**
  (this proxies `core/fd/components/aemform/v2/aemform` — verified in
  `ui.apps/.../components/aemformscontainer/.content.xml`; title "Adaptive Form - Embed").
- **`formRef`** = the form's **DAM guide-asset path**
  **`{damGuideRoot}/{project}/{formName}`** — i.e.
  **`/content/dam/formsanddocuments/{project}/{formName}`**
  (e.g. `/content/dam/formsanddocuments/{project}/employee-information-form`).
  **NOT the `/content/forms/af/...` runtime path.** The v2 AEM Form Container dialog binds the form by
  its DAM guide asset; pointing `formRef` at `/content/forms/af/...` is a common cause of the container
  failing to render the form on the page. This is the property that changes when a new form replaces
  the old one. (VERIFIED: with `formRef` = the DAM guide-asset path and the embed in the editable
  container, the page renders the form — the runtime HTML shows `guideContainer` + `cmp-adaptiveform`
  markup and loads the forms clientlib.)
- `formType` = **`af`** (Adaptive Form; the container also supports interactive documents — not used
  here).
- `loadType` = **`embed`** (render the form on the page, not as a link).
- **`useiframe` = `false`** — **INLINE embed. This is the mode that delivers full fidelity here.**
  The AEM Form Container (`core/fd/components/aemform/v2/aemform`) has two modes; **neither one carries
  the form's own clientLibRef JS clientlibs automatically** — this was verified against the deployed
  component HTL + dialog on `localhost:4502` (see the box below). Read this before choosing a mode:
  - `useiframe="true"` (iframe) → **DO NOT USE for this showcase.** `aemform.html` hardwires the iframe
    `src` to `${resource.path}.iframe.<locale>.html` — the container's OWN limited iframe rendition;
    there is **no property** to point the iframe at the form's real runtime page. That rendition
    (`iframe.html` → `formheaderlibs.html`) emits the AF runtime **only** under `!wcmmode.edit` (so it is
    dead in author edit mode — the `.iframe.` URL carries no wcmmode and defaults to `edit` on author),
    and its `formcontainer.html` pulls the form via `data-sly-resource`, so the form's **own clientLibRef
    JS clientlibs never load in ANY mode** (custom validation + submit→PDF broken even on publish). The
    dialog exposes only `./cssClientlib` (CSS) and `./themeRef` — **no JS-clientlib hook** — so the
    iframe can never be made fully functional. (An iframe pointing at the form's real page would need a
    component overlay or a hand-authored raw iframe — both out of composer's lane / against the invariant.)
  - `useiframe="false"` (inline / SSR) → `aemform.html` injects `formcontainer.html` **directly into the
    host Sites page**, so the **host page** supplies the runtime + theme + the form's clientlibs. **Use
    this**, together with the host-page clientlib wiring below. Inline has no `!wcmmode.edit` gate at the
    page level, so it works in view AND author.

> **Inline embed REQUIRES host-page clientlib wiring — this is mandatory, not optional.** An inline
> AEM Form Container renders the form via `data-sly-resource`, so the form's page-head clientlibs
> (its `themeRef` theme + its `clientLibRef` JS/CSS) do **not** emit. Wire them onto the host page.
> This project's page component (`{project}/components/page`) already provides the hook:
> - `customfooterlibs.html` already loads `core.forms.components.runtime.all` when the page has a form,
>   and each project form clientlib declares `dependencies=[core.forms.components.runtime.all]`, so the
>   AF runtime also loads **transitively** — hydration does not depend on `containsFormContainer`.
> - `customheaderlibs.html` / `customfooterlibs.html` load **two page-content properties** that the
>   composer skill sets on the page's `jcr:content` (generic + repointable — pages without them are
>   unaffected):
>   - **`formEmbedThemePath`** = the form's **`/content/forms/af/{project}/{formName}`** runtime path.
>     The theme is served by the AF theme selector servlet at `{formEmbedThemePath}.theme/_default/theme.css`
>     (and `…/theme.js`) — it is **not** a clientlib category, so it must be linked by this path.
>   - **`formEmbedClientlibs`** = the form guideContainer's **`clientLibRef`** categories verbatim.
>     Set it to the **single category matching the form's single-category `clientLibRef`** — i.e. the
>     one per-form clientlib `[{project}.forms.{formName}]` — **which itself `embed`s the
>     shared libs** (`…forms.base` + `…forms.generate-pdf`) and `dependencies`-declares
>     `core.forms.components.runtime.all`. AEM's clientlib resolver walks that `embed`/`dependencies`
>     closure, so the single category delivers exactly the CSS/JS the standalone form loads — no more,
>     no less. Do **not** re-list the embedded shared libs alongside it (that duplicates their bytes and
>     drifts from the standalone form). Only list multiple categories if the form's `clientLibRef` itself
>     lists multiple flat (non-embedding) categories.
> Ground-truth for the exact list: fetch the standalone form
> (`{formsContentRoot}/{project}/{formName}.html`) on the running instance and copy the `<link>`/`<script>`
> clientlib categories it emits (minus the site base + runtime, already on the page) — or simply mirror
> the form guideContainer's `clientLibRef` value verbatim.

> **Verify against the live dialog if it still won't render.** The proxy extends
> `core/fd/components/aemform/v2/aemform`. If the form does not appear, open the container's
> **Configure** dialog on the page in the AEM editor, pick the form, save, and export the node — then
> match the exact property name + value the dialog wrote (`formRef` = the DAM guide-asset path is the
> Core Component v2 binding; `fileReference` is the legacy Foundation equivalent). Keep `useiframe=false`.

Do not set a fixed `height` — that is an iframe-only prop; an inline form flows naturally in the page.

---

## Step 3 — Replacement logic (the core behaviour)

Repointing a new form updates **three** things (the container binding + the two host-page clientlib
properties). All three must move together, or the page renders the new form's markup with the old
form's theme/clientlibs.

When this skill runs for a **new** form:

1. **If `{siteRoot}/test-adaptive-form` does not exist** → create the page (Artifact 1) with one AEM
   Form Container (Artifact 2, `useiframe="false"`) whose `formRef` is the new form, and set the two
   host-page properties on the page `jcr:content` (below). Done.
2. **If the page already exists** → do NOT create a new page and do NOT add a second container. Repoint,
   in place:
   - the existing AEM Form Container node's **`formRef`** → the new form's DAM guide-asset path
     (`/content/dam/formsanddocuments/{project}/{formName}`);
   - the page `jcr:content` **`formEmbedThemePath`** → the new form's runtime path
     (`{formsContentRoot}/{project}/{formName}`);
   - the page `jcr:content` **`formEmbedClientlibs`** → the new form guideContainer's `clientLibRef`
     categories verbatim.
   Leave every other page property and node untouched. The old form reference and old clientlib list
   must be gone — the page now embeds only the new form.
3. If, for any reason, the existing page has more than one AEM Form Container (a prior mistake),
   collapse to **one**: keep a single container, repoint it to the new form, remove the extras.

> The invariant after this skill runs: **`{siteRoot}/test-adaptive-form` exists, has exactly one AEM
> Form Container (`useiframe="false"`), that container's `formRef` equals the newly generated form's
> DAM guide-asset path, and the page's `formEmbedThemePath` + `formEmbedClientlibs` point at the same
> new form.**

---

## Step 4 — Filter coverage

The `ui.content` filter already contains `<filter root="/content/{project}" mode="update"/>`,
which covers `{siteRoot}/test-adaptive-form`. **Confirm** this entry is present in
`ui.content/src/main/content/META-INF/vault/filter.xml`. Only if it is somehow missing, add:

```xml
<filter root="/content/{project}/us/en/test-adaptive-form" mode="update"/>
```

Do not add a redundant, narrower filter when the broader site root already covers the page.

---

## Pre-flight checklist (verify before handing off)

- [ ] `{siteRoot}/test-adaptive-form/.content.xml` exists, is a `cq:Page`, uses `{pageComponent}` +
      `{pageTemplate}`, and carries `cq:conf`/`sling:configRef` = `/conf/{project}`.
- [ ] The AEM Form Container node is authored inside the template's **editable** container
      (`root/container/container`), NOT on the locked structure node `root/container`.
- [ ] The page has **exactly one** AEM Form Container node (`…/aemformscontainer`), with
      **`useiframe="false"`** (inline) and **no `height`** prop.
- [ ] That container's **`formRef`** equals the **DAM guide-asset path**
      `/content/dam/formsanddocuments/{project}/{formName}` for the NEW form (NOT the
      `/content/forms/af/...` path) — the previously embedded form is no longer referenced anywhere.
- [ ] `formType="af"` and `loadType="embed"` are set.
- [ ] The page `jcr:content` carries **`formEmbedThemePath`** = the NEW form's
      `{formsContentRoot}/{project}/{formName}` runtime path and **`formEmbedClientlibs`** = the NEW
      form guideContainer's `clientLibRef` categories — both repointed off the old form.
- [ ] The page component's `customheaderlibs.html` (theme CSS + `formEmbedClientlibs` CSS) and
      `customfooterlibs.html` (`formEmbedClientlibs` JS + theme JS, non-async, before runtime.all) load
      those properties.
- [ ] `{project}` is used for the folder segment of the form/DAM path AND for the page
      component/template/conf — a single namespace, not a separate app folder.
- [ ] The page path is covered by a `ui.content` filter root.
- [ ] On the running instance the page **actually renders the form** (HTML shows `guideContainer` /
      `cmp-adaptiveform` + the forms clientlib), not just HTTP 200 — and the embed loads the same theme +
      clientlib categories as the standalone `{formsContentRoot}/{project}/{formName}.html`.

---

## Failure symptoms and their causes

| Symptom | Cause |
|---------|-------|
| Page returns HTTP 200 but the **content area is empty** and no form renders (no AF clientlib loads) | Embed authored on the **locked structure node** `root/container` instead of the editable `root/container/container`. The node deploys but the template never emits it. Move it one level deeper (Artifact 1). |
| Page shows the component placeholder / empty box, no form | `formRef` points at the `/content/forms/af/...` runtime path instead of the **DAM guide-asset path** `/content/dam/formsanddocuments/{project}/{formName}` — the v2 container binds the DAM asset (Artifact 2). |
| Page renders the **old** form after a new delivery | `formRef` not repointed — Step 3 replacement not applied |
| Two forms stacked on the page | A second container was added instead of repointing the existing one (Step 3 violation) |
| "Form not found" | the form's folder segment does not match `{project}`, or the form isn't deployed yet |
| Page 404s after deploy | Page path not in a `ui.content` filter root (Step 4) |
| Form renders but is **unstyled + non-functional** on the page (Times New Roman, **Submit disabled**, accordion won't toggle, native date inputs, no asterisks) — yet the standalone form is fine | Inline embed (`useiframe="false"`) without the theme + form clientlibs wired onto the host page — the form never hydrates. Set the page `jcr:content` **`formEmbedThemePath`** + **`formEmbedClientlibs`** and confirm `customheaderlibs/customfooterlibs` load them (Artifact 2, host-page wiring). Do **not** "fix" this by switching to `useiframe="true"` — the iframe rendition can NEVER load the form's clientLibRef JS clientlibs (no dialog hook) and is dead in author edit mode. |
| Form is styled but **custom validation / submit→PDF don't run** in the embed | `formEmbedClientlibs` is missing the form's JS categories (e.g. `…forms.{formName}`, `…forms.generate-pdf`), or they loaded after the runtime initialised — ensure they are in `formEmbedClientlibs` and loaded NON-async **before** the runtime.all block (see `customfooterlibs.html`). |
| Form shows default brand colours, not the form's theme | `formEmbedThemePath` unset/stale, or `{formPath}.theme/_default/theme.css` 404s — verify the path equals the NEW form's `/content/forms/af/{project}/{formName}`. |
| Embed worked when viewed on publish but was blank in the author **editor** | Old symptom of the iframe mode (runtime gated on `!wcmmode.edit`). Fixed by inline embed — do not reintroduce `useiframe="true"`. |
| Form renders unstyled inside the page | Page missing `cq:conf`/`sling:configRef` so the form's theme/clientlib context doesn't resolve |

---

## Run output convention (mandatory)

Write this skill's end-deliverable to the delivery's run directory under the **`assembly/`**
SDLC-cycle subfolder (AGENTS.md → "Run output convention"), using the delivery's `{runId}`:

```
.claude/agents/runs/{runId}/assembly/composer-embed.md
```

Record: the page path, whether it was created or reused, the OLD `formRef` (if replaced) and the NEW
`formRef`, the container resource type, and the filter status. Temporary/working files go to the
scratchpad directory — never into `runs/`.

---

## Deployment (final step)

> **Pipeline mode (delegated by the `assembler` lead inside a delivery): SKIP the deploy — author the
> page + container only.** Deployment is centralized in the `forgemaster` lead, which runs the single
> authoritative `mvn clean install -PautoInstallSinglePackage` **after** assembler, so it picks up the
> updated page too (AGENTS.md → "Deployment is centralized in Forgemaster"). Assembler runs
> `formwright → groundsmith → assembler → forgemaster → pilot → ⏸ MANUAL GATE → sentinel`.

When invoked **standalone / directly** (not via the pipeline), deploy and verify on the running
instance:

```bash
# Build and deploy the FULL reactor — this is the ONLY command that reaches AEM
mvn clean install -PautoInstallSinglePackage
```

Then verify on `http://localhost:4502`:

1. Open the page in the AEM editor — `.../test-adaptive-form.html` — the AEM Form Container renders
   the **new** adaptive form inline (fields, theme, validation), not a placeholder.
2. Fetch the published page and confirm HTTP 200 and that the embedded form model loads:
   ```bash
   curl -u admin:admin -s -o /dev/null -w "%{http_code}\n" \
     http://localhost:4502/content/{project}/us/en/test-adaptive-form.html
   ```
3. Confirm the page embeds the intended form and only that form (no stacked/old form).

Only after the page renders the new form on the running instance is the embed complete.
