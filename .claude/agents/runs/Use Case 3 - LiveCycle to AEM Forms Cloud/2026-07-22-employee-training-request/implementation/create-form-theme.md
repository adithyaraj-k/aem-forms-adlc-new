# create-form-theme — aem-adaptive-forms-agents-training-request

## Decision: BUILD NEW (per solution-architecture.yaml / DESI — no existing theme fits)

## Artifacts produced

1. **`/apps` theme.zip** (local rendering):
   `ui.apps/.../jcr_root/apps/fd/af/themes/aem-adaptive-forms-agents-training-request/`
   - `.content.xml` (sling:Folder + metadata)
   - `theme.zip` (built via PowerShell `Compress-Archive`, contains `theme.css` + `theme.js`)
2. **DAM theme-json** (picker + cloud rendering):
   `ui.content/.../jcr_root/content/dam/formsanddocuments-themes/aem-adaptive-forms-agents/training-request/`
   - `.content.xml` (`dam:Asset`, `type="theme"`, `sling:resourceType="fd/fm/theme/render"`)
   - `_jcr_content/renditions/theme-json/.content.xml` (`af_page`, `af_formtitle`, `af_fieldlabel`,
     `af_widgetAndText`, `af_button` nodes carrying the same token values as theme.css)

## Option A — thin token override (confirmed, not a full stylesheet)

`theme.css` is a single `:root { --af-*: … }` block — no re-authored standard element styling.
Token values traced to DocRoute.xdp evidence (see design/component-design-spec.yaml):

| Token | Value | Source |
|---|---|---|
| `--af-font` | `'Myriad Pro', Arial, sans-serif` | `<font typeface="Myriad Pro"/>` |
| `--af-heading-size` | `20pt` | `<font size="20pt">` |
| `--af-heading-weight` | `bold` | `weight="bold"` |
| `--af-section-heading` | `rgb(128,64,64)` | `<color value="128,64,64"/>` |
| `--af-section-heading-bg` | `rgb(196,219,251)` | `<color value="196,219,251"/>` |
| `--af-panel-bg` | `rgb(225,242,219)` | `<color value="225,242,219"/>` |
| `--af-field-border` | `#8c8c8c` | extrapolated (`<edge stroke="lowered"/>`, no RGB) |
| `--af-radius` | `0px` | square corners in all 3 DAM previews |
| `--af-primary` | `rgb(128,64,64)` | extrapolated (no button in static XDP) |
| `--af-required-asterisk` | `rgb(128,64,64)` | matches section heading |
| `--af-error` | base default retained | not specified in source |

**No `--af-section-rule` override** — the 3 DAM preview JPEGs show no hairline divider under
section headers; the header-band-to-panel-body colour transition is the only boundary. This is
recorded as an intentional deviation from the generic divider default, per DESI's explicit call.

## Base clientlib extended (2 new tokens promoted, per Option A "new tokens go in the base first")

DESI's design introduced two tokens not yet in the shared base contract:
`--af-section-heading-bg` (section-heading BAND background, default `transparent`) and
`--af-panel-bg` (panel BODY background, distinct from the page `--af-bg`, default `transparent`),
plus `--af-heading-size` (was previously hardcoded `18px` in the base, now a token). Added their
**neutral defaults** to `clientlib-forms-base/css/base.css` `:root` FIRST (so every other existing
form is visually unaffected — transparent = no band), then `.cmp-container__label` and
`.panelcontainer .cmp-container` were updated to CONSUME them. This form's theme + form clientlib
then override the values to the maroon/blue/green palette. This is the correct Option A order
(base declares the contract; the form theme only supplies values) and keeps the change additive/
non-breaking for the ~20 other forms sharing the base clientlib.

## Embedded-page mirroring (mandatory per project rule)

The theme selector does not apply when this form is later embedded into the "Test Adaptive Form"
Sites page (assembler phase) — only `formEmbedClientlibs` categories load. The SAME token override
values were mirrored into `employee-training-request-clientlib/css/form.css`'s own `:root` block
(see create-form-rules.md / clientlib section) so the form renders on-brand both standalone and
embedded.

## Reuse for the Document-of-Record template

Per solution-architecture.yaml, this theme is REUSED (not rebuilt) for the
`employee-training-request-dor` template content Groundsmith's `create-workflow` will author — no
second `create-form-theme` invocation.

## Wiring verification (rule 3ace — both places, byte-identical)

- `guideContainer/@themeRef` on the form + both fragments: `/apps/fd/af/themes/aem-adaptive-forms-agents-training-request` ✓ (set in create-adaptive-form.md / fragment run notes)
- Per-form/per-fragment conf-context `SiteConfig/@themeArtifact` = `aem-adaptive-forms-agents-training-request`, `@siteTemplatePath` = `/apps/fd/af/themes/aem-adaptive-forms-agents-training-request` ✓ (3 conf-context files: employee-training-request, employee-identity, declaration-consent)
- No residual references to any other theme name in the form/fragment `.content.xml` or conf `.content.xml` files (this is a brand-new form — no prior theme to purge).

## Deviation / additions beyond a pure token file

- Extended the shared base clientlib with 2 new token contract entries (documented above) —
  this is an ADDITIVE change to `aem-adaptive-forms-agents.forms.base`, not a rebuild; verified to
  default to `transparent` so no other form's visual output changes.

## Verification required post-deploy (flag to Forgemaster/Sentinel)

- Confirm theme appears in the DAM Forms & Documents theme picker.
- Confirm `{form}.theme/_default/theme.css` serves 200 locally with the maroon/blue/green palette.
- Pixel-verify: section header bands (blue bg + maroon bold 20pt caption), panel body green fill,
  NO hairline divider under headers, Declaration panel has no header band at all, square corners,
  Myriad Pro (falls back to Arial) font throughout — against the 3 DAM preview JPEGs (test-form-ui,
  Sentinel phase).
