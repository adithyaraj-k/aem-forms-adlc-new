# Phase 3 — create-adaptive-form · Vehicle Registration Form

**Run:** 2026-07-01-vehicle-registration-form
**Skill:** create-adaptive-form
**Lead:** Blockwright (IMPL build) · Phase 3
**Status:** PASSED (self quality gate)
**Deployed:** NO — author-only. Deployment is centralized in Auditon (pipeline mode). No `mvn` run.

## Inputs consumed
- `design/DESI-design-form-components.yaml` (field-by-field authority: resource types, names, enums, aria-labels, order, columns, validation)
- `implementation/phase01-generate-schema.md` (schema path + JSONPath bindings; 25 value-bound leaves, array-of-enum checklist)
- Reference screenshot `c:\Users\330073\Downloads\Vehicle Reg.jpg` (layout/labels/order)
- Existing repo conventions (health-insurance-form) for exact CC attribute shapes

## Project tokens (kept distinct)
- project (resource-type root): `{project}`
- appFolder (forms/dam/conf roots): `{appFolder}`
- Template reused (built-in af-page-v2 based): `/conf/{project}/settings/wcm/templates/blank-af-v2`

## 5 artifacts authored on disk
1. **Form page** (cq:Page, guideContainer extending core/fd/components/form/container/v2/container via proxy `formcontainer`)
   `ui.content/src/main/content/jcr_root/content/forms/af/{appFolder}/vehicle-registration-form/.content.xml`
2. **DAM guide asset** (type=guide, guide=1, formmodel=jsonschema)
   `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/{appFolder}/vehicle-registration-form/.content.xml`
3. **Per-form conf context** (SiteConfig + HtmlPageItemsConfig; theme injection)
   `ui.content/src/main/content/jcr_root/conf/forms/{appFolder}/vehicle-registration-form/.content.xml`
4. **Filter entry** (conf-context root added; form-page + dam roots already covered by existing broad roots)
   `ui.content/src/main/content/META-INF/vault/filter.xml`  → added `/conf/forms/{appFolder}/vehicle-registration-form`
5. **Validation clientlib SCAFFOLD** (node + js.txt/css.txt manifests + placeholder functions.js/form.css; bodies authored in Phase 9)
   `ui.apps/src/main/content/jcr_root/apps/clientlibs/vehicle-registration-form-clientlib/`
   - `.content.xml` (cq:ClientLibraryFolder, categories `[{project}.forms.vehicle-registration-form]`, allowProxy, dep `core.forms.components.runtime.all`)
   - `js.txt` → `functions.js`; `css.txt` → `form.css`; scaffold `js/functions.js`, `css/form.css`
   - `ui.apps` filter already covers `/apps/clientlibs` (verified) — no filter edit needed.

## Form JCR path
`/content/forms/af/{appFolder}/vehicle-registration-form`

## guideContainer wiring
- `title="VEHICLE REGISTRATION FORM"`, `showTitle="{Boolean}true"`, `aria-label="Vehicle Registration Form"`
- `schemaType="jsonschema"` + `schemaRef=/content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json` (bound to Phase-1 JSON Schema)
- `themeRef=/apps/fd/af/themes/{project}-vehicle-registration` (Phase 8) — identical string in Artifact 2 metadata + Artifact 3 themeArtifact/siteTemplatePath
- `clientLibRef={project}.forms.vehicle-registration-form` (Phase 9)
- Submit: `actionType=fd/af/components/guidesubmittype/submitservice` + `submitService="Custom-Submit-GeneratePDF"` (shared action; Bridgesmith ensures/wires it — no per-form submit action scaffolded)
- `sling:configRef` ends with trailing `/`
- Title (H1) + subtitle (text/plain, textIsRich=false) authored above the panels

## Structure (as-built)
- **Panels:** 5 numbered (OWNER, VEHICLE, INSURANCE, DOCUMENT CHECKLIST, DECLARATION) in order, + 1 hidden `actionsPanel` footer for Submit/Reset.
- **Value-bound fields (fd:formDataRef):** 25 — owner 9, vehicle 9, insurance 3, documents 1, declaration 3. (declarationText is static text/plain, correctly UNBOUND.)
- **Required:** 23 (email + documentChecklist optional per DESI/screenshot).
- **Layout:** 3-column responsive (`width="4"` per field, `behavior="newline"` at row starts); Address, Document Checklist, and declaration text span full width (`width="12"`).
- **Dropdowns (enum/enumNames):** state(37), vehicleType(6), fuelType(7), manufacturingYear(2026..1996 = 31). Radio: gender(3). Checkbox-group: documentChecklist(6, `type="string[]"` = array-of-enum).
- **manufacturingYear:** current year (2026) first, descending to 1996.
- Date pickers use `displayFormat/editFormat=date|DD/MM/YYYY`, placeholder `DD / MM / YYYY`.
- mobileNumber: telephoneinput, `pattern=^[0-9]{10}$`, maxLength 10, inputMode numeric.
- pinCode: numberinput, `pattern=^[0-9]{6}$`, maxLength 6, inputMode numeric, `leadingZerosAllowed={Boolean}true`, type=string (preserve leading zeros).

## Self quality gate (verified)
- [x] `<items` child nodes in form XML: **0** (grep) — all choices via enum/enumNames attributes
- [x] CAPTCHA references: **0** — NO CAPTCHA
- [x] Every field/panel/button/title/text has `aria-label` (37 aria-label; no field missing)
- [x] enum length == enumNames length for all 6 choice fields (3/37/6/7/31/6 — all equal)
- [x] checkboxgroup `type="string[]"` bound to `$.documents.documentChecklist` (array-of-enum, not scalar)
- [x] Bound to Phase-1 JSON Schema (`schemaType=jsonschema`, correct schemaRef)
- [x] Submit button `fd:rules/@fd:click` = full EVENT_SCRIPTS→SUBMIT_FORM escaped-JSON AST; `fd:events @click="[submitForm()]"` (NOT raw fd:click)
- [x] Reset button `actions/reset`, buttonType=reset
- [x] Booleans typed `{Boolean}true/false`; `sling:configRef` trailing `/`; resource types are CC proxies (never Foundation fd/af/components/...)
- [x] project (`{project}`) vs appFolder (`{appFolder}`) kept distinct across all paths/attrs
- [x] No fd:rules validation AST authored (Phase 4 owns it); validation field attributes (required/pattern/maxLength/mandatoryMessage/validatePatternMessage) set
- [x] All 4 form XML files + filter.xml well-formed (PowerShell [xml] parse)
- [x] No hardcoded `/content/forms/af` path in code — token-based per convention
- [x] Icon assets: DESI `icon_assets: []` (pure text/field form, no logo/icon) — none to store/reference

## Notes / handoff
- No fields invented beyond the DESI/wireframe list. Field order matches the screenshot per-panel.
- Phase 4 (create-form-rules): author the fd:rules validation ASTs (mobile 10-digit, PIN 6-digit, email-when-present, DOB not-future, 3 dates DD/MM/YYYY) referencing the clientlib functions.
- Phase 8 (create-form-theme): build theme `{project}-vehicle-registration` (apps theme.zip + DAM theme-json).
- Phase 9 (create-form-clientlib): fill functions.js (5 validators, global-scope/empty-safe) + 3-col grid & date-width CSS; wire validationExpression on validated fields.
- Bridgesmith: ensure shared `Custom-Submit-GeneratePDF` submit action exists (submitService label must match exactly).
```

## Handoff YAML
```yaml
agent: create-adaptive-form
phase: 3
status: PASSED
artifacts:
  - form_page: "ui.content/src/main/content/jcr_root/content/forms/af/{appFolder}/vehicle-registration-form/.content.xml"
  - dam_asset: "ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/{appFolder}/vehicle-registration-form/.content.xml"
  - conf_context: "ui.content/src/main/content/jcr_root/conf/forms/{appFolder}/vehicle-registration-form/.content.xml"
  - filter_entries: updated
  - clientlib: "ui.apps/src/main/content/jcr_root/apps/clientlibs/vehicle-registration-form-clientlib/"
form_jcr_path: "/content/forms/af/{appFolder}/vehicle-registration-form"
field_count: 25
panel_count: 5
gate_result: PASS
deployed: false
```
