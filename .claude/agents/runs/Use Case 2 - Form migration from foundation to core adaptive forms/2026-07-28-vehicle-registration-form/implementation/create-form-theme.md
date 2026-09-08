# create-form-theme — vehicle-registration-form

**Phase:** 8 (executed, deviating from PLAN's "SKIP") · **Status:** COMPLETE

## Reuse-first gate result: NEW theme, justified
PLAN/DESI specced reuse of `/apps/fd/af/themes/aem-adaptive-forms-agents-wknd` + a clientlib-only
token override (Phase 8 marked SKIP). On build, this theme was found **not to exist anywhere in
the repository** — confirmed by listing `ui.apps/src/main/content/jcr_root/apps/fd/af/themes/`,
which contains only per-form dedicated themes (college-admission, doctor-appointment,
employee-reg, patient-registration, school-admission, sports-event-registration, testmigration).
Every existing form in this project has its own dedicated thin-token-override theme — there is no
shared generic theme precedent to reuse.

Per critical rule 3ace ("a branded form needs its OWN dedicated theme, wired in BOTH places") this
build creates `aem-adaptive-forms-agents-vehicle-registration`, following the EXACT pattern of the
sibling `aem-adaptive-forms-agents-employee-reg` theme (same `.content.xml` shape, same
`theme.zip` internal layout — `theme.css` + `theme.js`, same DAM theme-json `af_*` node
structure). This is a THIN Option-A token override only — no full stylesheet re-authored.

## Artifacts produced
- `ui.apps/src/main/content/jcr_root/apps/fd/af/themes/aem-adaptive-forms-agents-vehicle-registration/.content.xml` (sling:Folder + metadata)
- `ui.apps/src/main/content/jcr_root/apps/fd/af/themes/aem-adaptive-forms-agents-vehicle-registration/theme.zip` (theme.css + theme.js, built via PowerShell `Compress-Archive`)
- `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments-themes/aem-adaptive-forms-agents/vehicle-registration/.content.xml` (dam:Asset, type=theme)
- `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments-themes/aem-adaptive-forms-agents/vehicle-registration/_jcr_content/renditions/theme-json/.content.xml` (af_page/af_formtitle/af_fieldlabel/af_widgetAndText/af_button nodes)

No new filter entries needed for the DAM theme (already covered by the existing
`mode="merge"` filter root `/content/dam/formsanddocuments-themes/aem-adaptive-forms-agents`).
Added one new `ui.apps` filter entry: `/apps/fd/af/themes/aem-adaptive-forms-agents-vehicle-registration`.

## Token overrides (brand: forest green #228B22, exact-replica of the reference screenshot)
`--af-primary:#228B22`, `--af-primary-contrast:#ffffff`, `--af-primary-dark:#1c7a1c`,
`--af-bg:#ffffff`, `--af-title:#1a1a1a`, `--af-heading:#1a1a1a`, `--af-subtitle:#666666`,
`--af-section-heading:#228B22`, `--af-section-rule:1px solid #e0e0e0`,
`--af-field-border:1px solid #cccccc`, `--af-placeholder:#999999`,
`--af-required-asterisk:#d32f2f`, `--af-error:#d32f2f`,
`--af-font:Arial, Helvetica, system-ui, sans-serif`, `--af-font-size-base:14px`,
`--af-title-size:32px`, `--af-section-heading-size:16px`, `--af-heading-weight:700`,
`--af-radius:4px`, `--af-field-height:40px`, `--af-gap:16px`, `--af-label-gap:8px`.

## Wired in BOTH places (rule 3ace)
1. Form `guideContainer/@themeRef` = `/apps/fd/af/themes/aem-adaptive-forms-agents-vehicle-registration`
2. Conf-context `SiteConfig` `themeArtifact`/`siteTemplatePath` = same theme, in
   `ui.content/.../conf/forms/aem-adaptive-form-agents/vehicle-registration-form/.content.xml`

Grep-verified: no residual reference to `aem-adaptive-forms-agents-wknd` anywhere in the form's
or conf's `.content.xml`.

## Standard element styling
Owned entirely by the shared base clientlib (`aem-adaptive-forms-agents.forms.base`) — not
re-authored here. The form clientlib (`create-form-clientlib.md`) also mirrors these exact brand
tokens so the form renders on-brand when embedded in the "Test Adaptive Form" Sites page (where
the theme selector does not apply).
