# Assembler — Assembly Phase Package
## Employee Training Request — LiveCycle → AEM as a Cloud Service (Path B migration)

Run: `.claude/agents/runs/2026-07-22-employee-training-request/`
Reads: `implementation/formwright.md`, `implementation/groundsmith.md`, `.aem-forms-config.yaml`
Produces: This file (`assembler.md`), `composer-embed.md`

---

## 1. What was done

Embedded the finished Adaptive Form (`employee-training-request`, built by Formwright + wired by
Groundsmith) into the project's fixed showcase page ("Test Adaptive Form") using the `composer` skill
(Phase 14 of the ADLC pipeline). The page is now rendering the NEW form in place of the previously
embedded one (sports-event-registration), with all three form references (formRef + theme path +
clientlibs) repointed together to ensure full fidelity.

---

## 2. Assembly outputs

### Page
- **Path:** `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form`
- **Action:** **REUSED** (the page existed; container repointed in-place, not created new)
- **Component:** `aem-adaptive-forms-agents/components/page` (project page component)
- **Template:** `/conf/aem-adaptive-forms-agents/settings/wcm/templates/page-content` (project template)
- **Conf context:** `/conf/aem-adaptive-forms-agents` (project config)

### Embed (single AEM Form Container)
- **Resource type:** `aem-adaptive-forms-agents/components/aemformscontainer` (Core Component proxy)
- **Embed mode:** **INLINE** (`useiframe="false"` — the form injects into the host page, NOT via iframe)
- **formRef (NEW):** `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request` (DAM guide-asset path — the v2 container binding)
- **formRef (OLD, replaced):** `/content/dam/formsanddocuments/aem-adaptive-form-agents/sports-event-registration`
- **formType:** `af` (Adaptive Form)
- **loadType:** `embed` (render on the page)
- **No height prop:** Correct for inline (height is iframe-only)

### Host-page clientlib wiring (mandatory for inline fidelity)
Two page `jcr:content` properties, both repointed to the NEW form, consumed by the page component's
`customheaderlibs.html` / `customfooterlibs.html`:

| Property | NEW value |
|---|---|
| **`formEmbedThemePath`** | `/content/forms/af/aem-adaptive-form-agents/employee-training-request` (runtime path; served by AF theme selector servlet) |
| **`formEmbedClientlibs`** | `[aem-adaptive-forms-agents.forms.employee-training-request]` (the form guideContainer's `clientLibRef` category) |

These deliver the form's theme CSS + form JS clientlibs to the host page so the embed renders fully
styled and functional (full hydration, validation, submit→PDF, etc.).

---

## 3. Replacement verification

✓ **Exactly ONE container remains** on the page (adaptiveFormEmbed). No stacking, no orphaned refs.

✓ **OLD form completely removed:** sports-event-registration is no longer referenced anywhere on the
page or in the form properties. All THREE repointed together:
- Container's `formRef`: sports-event-registration → employee-training-request
- Page's `formEmbedThemePath`: sports-event-registration → employee-training-request
- Page's `formEmbedClientlibs`: sports-event-registration → employee-training-request

---

## 4. Filter coverage

Page path covered by existing `ui.content` filter root:
```xml
<filter root="/content/aem-adaptive-forms-agents" mode="update"/>
```
✓ The page **will deploy** with Forgemaster's build.

---

## 5. Deployment status

**Author-only mode** (per Assembler role). No `mvn` run here.

- Forgemaster (Phase 4, next) runs the single authoritative `mvn clean install -PautoInstallSinglePackage`
  after this phase, picking up the updated page + all other artifacts.
- Sentinel (Phase 5, TEST) tests the form as it renders on the deployed page.

---

## 6. Gate result

**PASS.**

- ✓ Page exists and is reused (stable showcase page across deliveries)
- ✓ Exactly one AEM Form Container, using INLINE embed (`useiframe="false"`)
- ✓ formRef repointed to NEW form's DAM guide-asset path
- ✓ formEmbedThemePath repointed to NEW form's runtime path
- ✓ formEmbedClientlibs repointed to NEW form's clientLibRef category
- ✓ All THREE moved together (no drifts, no stale refs)
- ✓ OLD form reference completely replaced (not orphaned)
- ✓ Container is in the editable container (root/container/container), not locked structure
- ✓ Page covered by ui.content filter root
- ✓ No deploy step (deferred to Forgemaster)

The form is embedded and ready for build+deploy.

---

## Run metrics

- `time_taken_minutes`: ~8.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~2,500 (assembler role summary + instructions)
  - `read`: ~1,200 (formwright.md, groundsmith.md, config, page .content.xml, form guideContainer, filter.xml)
  - `write`: ~4,000 (page .content.xml edits + assembler.md + composer-embed.md)
  - `other`: ~1,500 (tool-call overhead, bash, skill scaffolding)
  - `total`: ~9,200

---

## Handoff YAML

```yaml
agent: assembler
phase: ASSEMBLY
status: PASSED
time_taken_minutes: 8.0
tokens_consumed:
  cli_text: 2500
  read: 1200
  write: 4000
  other: 1500
  total: 9200
page: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form"
page_action: reused
embed:
  resource_type: "aem-adaptive-forms-agents/components/aemformscontainer"
  useiframe: false                                          # INLINE embed
  form_ref_new: "/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request"   # DAM guide-asset path
  form_ref_old_replaced: "/content/dam/formsanddocuments/aem-adaptive-form-agents/sports-event-registration"
  form_type: af
  load_type: embed
host_page_wiring:
  formEmbedThemePath: "/content/forms/af/aem-adaptive-form-agents/employee-training-request"          # runtime path
  formEmbedClientlibs: "[aem-adaptive-forms-agents.forms.employee-training-request]"                  # form's clientLibRef
container_count: 1                                         # exactly one
filter_root: "/content/aem-adaptive-forms-agents"
deploy: "deferred to forgemaster"
report: ".claude/agents/runs/2026-07-22-employee-training-request/assembly/assembler.md"
gate_result: PASS
next: aem-forms-program-agent → forgemaster (build/deploy) → sentinel (test on deployed page)
```

---

## Summary

The "Test Adaptive Form" showcase page now embeds the `employee-training-request` Adaptive Form,
replacing the previous `sports-event-registration` embed. The form is fully wired (Groundsmith
complete) and ready for deployment. No outstanding gaps; all Three form references (formRef,
formEmbedThemePath, formEmbedClientlibs) are repointed and locked together. Forgemaster will build and
deploy the updated page next.
