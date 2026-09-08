# Assembler — vehicle-registration-form (ASSEMBLY phase)

## Summary
Embedded `vehicle-registration-form` into the fixed "Test Adaptive Form" Sites page, replacing the
previously embedded `event-ticket-booking` form, via the `composer` skill (author-only mode).

- **Page**: `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form` — reused.
- **Embed**: AEM Form Container `aem-adaptive-forms-agents/components/aemformscontainer`,
  `useiframe="false"` (INLINE embed, unchanged), `formRef` (NEW, DAM guide-asset path) =
  `/content/dam/formsanddocuments/aem-adaptive-form-agents/vehicle-registration-form`,
  `formType="af"`, `loadType="embed"`.
- **Host-page wiring**: `formEmbedThemePath` =
  `/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form`; `formEmbedClientlibs` =
  `[aem-adaptive-forms-agents.forms.vehicle-registration-form]` — both repointed off the old form
  together with `formRef`.
- **Replacement**: old `formRef` was
  `/content/dam/formsanddocuments/aem-adaptive-form-agents/event-ticket-booking`; confirmed exactly
  one container node remains (`adaptiveFormEmbed`) and zero references to `event-ticket-booking`
  remain anywhere in the page source.
- **Filter**: `/content/aem-adaptive-forms-agents` filter root (`mode="update"`) in
  `ui.content/src/main/content/META-INF/vault/filter.xml` covers the page path.
- **Fragments**: form uses `address-details-fragment` and `declaration-fragment` by reference;
  nothing in the embed config restricts fragment rendering.
- **Deploy**: deferred to Forgemaster — no `mvn` run in this phase.

## Hazard for Forgemaster
`ui.content` filter roots use `mode="update"`, which never deletes instance nodes removed/renamed in
source. Forgemaster should diff live vs. source for
`/content/aem-adaptive-forms-agents/us/en/test-adaptive-form` post-deploy and sweep any stale
`event-ticket-booking` remnants left over from prior deployments.

## Files changed
- `ui.content/src/main/content/jcr_root/content/aem-adaptive-forms-agents/us/en/test-adaptive-form/.content.xml`
  (3 attribute repoints: `formEmbedThemePath`, `formEmbedClientlibs`, `adaptiveFormEmbed/@formRef`)

## Handoff YAML
```yaml
agent: assembler
phase: ASSEMBLY
status: PASSED
page: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form"
page_action: reused
embed:
  resource_type: "aem-adaptive-forms-agents/components/aemformscontainer"
  useiframe: false
  form_ref_new: "/content/dam/formsanddocuments/aem-adaptive-form-agents/vehicle-registration-form"
  form_ref_old_replaced: "/content/dam/formsanddocuments/aem-adaptive-form-agents/event-ticket-booking"
  form_type: af
  load_type: embed
host_page_wiring:
  formEmbedThemePath: "/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form"
  formEmbedClientlibs: "[aem-adaptive-forms-agents.forms.vehicle-registration-form]"
container_count: 1
filter_root: "/content/aem-adaptive-forms-agents"
deploy: "deferred to forgemaster"
report: ".claude/agents/runs/2026-08-12-vehicle-registration-form/assembly/assembler.md"
gate_result: PASS
next: aem-forms-program-agent runs forgemaster (build/deploy), which deploys the updated page too
hazard_for_forgemaster: "ui.content filter mode=update never deletes renamed/removed instance nodes — diff live vs source for the test-adaptive-form page and sweep any stale event-ticket-booking remnants"
```
