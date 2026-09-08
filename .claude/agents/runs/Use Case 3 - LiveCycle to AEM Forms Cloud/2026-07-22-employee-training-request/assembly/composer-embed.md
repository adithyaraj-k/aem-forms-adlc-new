# Composer — Page Embed (Assembly Phase)
## Employee Training Request — LiveCycle → AEM as a Cloud Service (Path B migration)

Run: `.claude/agents/runs/2026-07-22-employee-training-request/`
Reads: `implementation/formwright.md` (form path), `.aem-forms-config.yaml` (project tokens)
Produces: This file (`composer-embed.md`) + updates to `test-adaptive-form/.content.xml`

---

## Execution summary

**Pipeline mode:** Author-only (deploy deferred to Forgemaster).

The "Test Adaptive Form" Sites page (`/content/aem-adaptive-forms-agents/us/en/test-adaptive-form`)
was updated to repoint its single embedded Adaptive Form from `sports-event-registration` to the newly
built `employee-training-request` form (Formwright + Groundsmith complete, status PASS).

---

## Artifact 1 — Test Adaptive Form Sites page (reused)

| Property | Value |
|---|---|
| Page path | `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form` |
| jcr:primaryType | `cq:Page` |
| Page component | `aem-adaptive-forms-agents/components/page` |
| Page template | `/conf/aem-adaptive-forms-agents/settings/wcm/templates/page-content` |
| cq:conf / sling:configRef | `/conf/aem-adaptive-forms-agents` |
| **Page action** | **REUSED** (page already existed; container repointed in-place) |
| JCR path | `ui.content/src/main/content/jcr_root/content/aem-adaptive-forms-agents/us/en/test-adaptive-form/.content.xml` |

**Pre-flight:** Page verified to use the project's editable template and component. The AEM Form
Container is authored inside the EDITABLE container (`root/container/container`), not the locked
structure node — ensuring it renders on the page.

---

## Artifact 2 — Single AEM Form Container (INLINE embed)

Artifact 2 is the sole form embed on the page, repointed to the new form:

| Property | Value |
|---|---|
| Node name | `adaptiveFormEmbed` |
| sling:resourceType | `aem-adaptive-forms-agents/components/aemformscontainer` (Core Component proxy) |
| **useiframe** | **`false`** (INLINE embed — the form is injected directly into the host Sites page) |
| formType | `af` (Adaptive Form) |
| loadType | `embed` (render on the page, not as a link) |
| **formRef (NEW)** | **`/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request`** (DAM guide-asset path — the v2 container binding) |
| **formRef (OLD, replaced)** | `/content/dam/formsanddocuments/aem-adaptive-form-agents/sports-event-registration` |
| **height** | NOT SET (inline form flows naturally; height is an iframe-only prop) |

**Exactly ONE container:** Verified — no second form container on the page; old reference completely replaced.

---

## Host-page clientlib wiring (mandatory for inline embed)

An INLINE embed (`useiframe="false"`) injects the form directly into the page, so the form's own
clientlibs (theme CSS + JS for validation/submit-to-PDF) do NOT emit automatically. The page
component (`aem-adaptive-forms-agents/components/page`) provides hooks (`customheaderlibs.html` /
`customfooterlibs.html`) that load two page `jcr:content` properties — **both repointed to the NEW
form**:

| Property | NEW value | OLD value (replaced) |
|---|---|---|
| **`formEmbedThemePath`** | `/content/forms/af/aem-adaptive-form-agents/employee-training-request` (runtime path; served by AF theme selector servlet) | `/content/forms/af/aem-adaptive-form-agents/sports-event-registration` |
| **`formEmbedClientlibs`** | `[aem-adaptive-forms-agents.forms.employee-training-request]` (the form guideContainer's single `clientLibRef` category, which itself embeds the shared base + runtime deps) | `[aem-adaptive-forms-agents.forms.sports-event-registration]` |

These two properties are consumed by:
- `customheaderlibs.html`: loads `formEmbedThemePath` CSS (theme) + `formEmbedClientlibs` CSS
- `customfooterlibs.html`: loads `formEmbedClientlibs` JS non-async, before the AF runtime block (so
  the form's custom validation + submit actions initialize correctly)

Verified: `{project}/components/page` includes these hooks — no custom page component required.

---

## Replacement logic — THREE-POINT repointing

Embedding a new form updated ALL THREE in tandem (form ref + theme path + clientlibs), ensuring the
page renders the new form with its own theme and JS clientlibs intact:

1. **Container `formRef`** → the NEW form's DAM guide-asset path
   (`/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request`)
2. **Page `formEmbedThemePath`** → the NEW form's runtime path
   (`/content/forms/af/aem-adaptive-form-agents/employee-training-request`)
3. **Page `formEmbedClientlibs`** → the NEW form's `clientLibRef` category
   (`aem-adaptive-forms-agents.forms.employee-training-request`)

**No stacking:** The page now embeds ONLY the employee-training-request form; the old
sports-event-registration reference is gone (not orphaned/stale).

---

## Filter coverage

Verified in `ui.content/src/main/content/META-INF/vault/filter.xml`:
```xml
<filter root="/content/aem-adaptive-forms-agents" mode="update"/>
```

This filter root covers the page path `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form`.
✓ The page **will deploy** with Forgemaster's `mvn clean install -PautoInstallSinglePackage`.

---

## Verification checklist

- [x] Page path: `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form`
- [x] Page reused (not created new); exact same page node, template, component
- [x] AEM Form Container authored inside the editable container (`root/container/container`), not locked structure
- [x] Exactly ONE container (no stacking)
- [x] Container uses INLINE embed (`useiframe="false"`)
- [x] `formRef` = DAM guide-asset path (NEW): `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request`
- [x] `formType` = `af`, `loadType` = `embed`
- [x] No fixed `height` prop on the container
- [x] Page `formEmbedThemePath` = runtime path (NEW): `/content/forms/af/aem-adaptive-form-agents/employee-training-request`
- [x] Page `formEmbedClientlibs` = NEW form's `clientLibRef`: `[aem-adaptive-forms-agents.forms.employee-training-request]`
- [x] OLD form reference completely removed (sports-event-registration repointed off)
- [x] Page covered by `ui.content` filter: `/content/aem-adaptive-forms-agents`
- [x] Author-only mode: no `mvn` run; deploy deferred to Forgemaster

---

## Deploy status

**Author-only.** Forgemaster's next step (`forgemaster` lead, Phase 4 of the pipeline) runs the single
authoritative `mvn clean install -PautoInstallSinglePackage`, which includes this updated page in the
deploy. Sentinel (Phase 5, TEST) then verifies the form renders correctly on the deployed page.

---

## Run metrics

- `time_taken_minutes`: ~8.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~2,500
  - `read`: ~1,200 (page .content.xml, form guideContainer clientLibRef, filter.xml)
  - `write`: ~1,800 (page .content.xml updates, this deliverable)
  - `other`: ~1,500 (tool-call overhead, bash output)
  - `total`: ~7,000

---

## Handoff

The page is ready for Forgemaster's build/deploy. All artifacts authored, no outstanding gaps. The
employee-training-request form is now the active embed on the "Test Adaptive Form" showcase page,
and its theme + clientlibs are wired to the host page for full inline fidelity.

Next: `aem-forms-program-agent` → `forgemaster` (build/deploy) → `sentinel` (test).
