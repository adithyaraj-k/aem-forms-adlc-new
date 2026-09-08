# Phase 8 — create-form-theme · Vehicle Registration Form

**Run:** 2026-07-01-vehicle-registration-form
**Skill:** create-form-theme
**Lead:** Blockwright (IMPL build) · Phase 8
**Status:** PASSED (self quality gate)
**Deployed:** NO — author-only. Deployment is centralized in Auditon (pipeline mode). No `mvn` run.

## Inputs consumed
- `design/DESI-design-form-components.yaml` → `design_specifications.theme.tokens` (navy/light-blue palette, radii, fonts), `layout`, `responsive`, `accessibility`, `icon_assets: []`
- Reference screenshot `c:\Users\330073\Downloads\Vehicle Reg.jpg` (visual parity authority — centered navy title, light-blue page, section headings, red asterisks, navy Submit / outlined Reset, centered button row)
- `implementation/phase03-create-adaptive-form.md` (theme wiring already in place; CSS hooks on panels/buttons)
- Verified working template: `{project}-wecare` theme (both /apps theme.zip theme.css and DAM theme-json) — reused its verified float-grid / title-visibility / date-picker / focus-ring / responsive structure

## Theme name
`{project}-vehicle-registration`

## Artifact 1 — /apps theme.zip (LOCAL rendering, served CSS)
`ui.apps/src/main/content/jcr_root/apps/fd/af/themes/{project}-vehicle-registration/`
- `.content.xml` — `sling:Folder` + `jcr:content/metadata` (title/description/author). NOT `fd:AEMFormTheme`.
- `theme.zip` — built with PowerShell `Compress-Archive` (NOT Git Bash zip). Verified entries at zip ROOT: `theme.css`, `theme.js` (no subfolders). theme.css = 14,178 bytes / 428 lines (full runtime BEM ruleset); theme.js = 18 bytes no-op.
  - `theme.js` = no-op `(function(){})();`
  - `theme.css` = navy/light-blue palette targeting the LIVE `.cmp-adaptiveform-*` runtime BEM classes and the platform float grid.

This is the ONLY thing that renders locally (ServeThemeArtifactsServlet reads theme.zip → serves `{form}.theme/_default/theme.css`).

## Artifact 2 — DAM theme-json (picker listing + CLOUD rendering)
`ui.content/src/main/content/jcr_root/content/dam/formsanddocuments-themes/{project}/vehicle-registration/`
- `.content.xml` — `dam:Asset`; `jcr:content` `sling:resourceType="fd/fm/theme/render"`, `theme="1"`, `type="theme"`; metadata `title="Vehicle Registration Theme"` (picker label), `clientlibCategory=fdtheme.{project}.vehicle-registration`, breakpoints (default/smallScreen/phone/tablet).
- `_jcr_content/renditions/theme-json/.content.xml` — partial theme (`formRef` inherits default). `af_page`, `af_formtitle`, `af_fieldlabel`, `af_widgetAndText`, `af_button` nodes with `_x0023_` state attrs and `\,`-escaped commas. Tokens IDENTICAL to theme.css.
- `_jcr_content/renditions/original` + `original.dir/.content.xml` — real CSS `original` rendition (nt:file, text/css) so the DAM `renditions/original` path is non-404 locally (optional convenience; matches wecare). Content = the same theme.css.

## PATH CONVENTION NOTE (resolved discrepancy)
The DESI spec and phase-3 report referenced the DAM theme under
`/content/dam/formsanddocuments/{appFolder}/themes/...`.
The ACTUAL working repo convention (per the create-form-theme SKILL and every existing
deployed theme: wecare, gov, enrollment, healthcare, fsi, …) is
`/content/dam/formsanddocuments-themes/{project}/{theme}`.
I used the working convention (the one that lists in the picker and is a filter root).
Note the DAM appFolder here is `{project}` (WITH the trailing 's'), which is
how every existing theme in this repo is stored under `formsanddocuments-themes`.

## Design tokens applied (theme.css ↔ theme-json consistent)
| Token | Value | theme.css | theme-json |
|---|---|---|---|
| Headings / section / labels / Submit bg / Reset border+text | navy `#1a2b5e` | ✅ | ✅ (af_formtitle color, af_fieldlabel color, af_button background) |
| Submit hover | `#12204a` | ✅ | ✅ (af_button hover) |
| Page background | light-blue `#e8eef7` | ✅ | ✅ (af_page background) |
| Field background | `#ffffff` | ✅ | ✅ (af_widgetAndText background) |
| Field border | `#c9d3e6` | ✅ | ✅ (af_widgetAndText border-color) |
| Required asterisk / error | red `#e02020` | ✅ | ✅ (af_widgetAndText error) |
| Field/button radius | 4px (0.25rem) | ✅ | ✅ |
| Font | Arial/Helvetica/Segoe UI system stack | ✅ | ✅ |
| Title | ~28px bold centered | ✅ (28px center) | ✅ (1.75rem center) |
| Section heading | ~18px bold uppercase | ✅ | (title styled via theme.css; JSON styles field-level) |
| Focus ring | rgba(26,43,94,.35) | ✅ | ✅ (af_widgetAndText focus) |

## CRITICAL CSS rules — confirmed applied
- **Served CSS from /apps theme.zip.** theme.zip → theme.css is the local source of truth. Conf `HtmlPageItemsConfig` injects `<link href="/theme.css">`; ServeThemeArtifactsServlet serves it. DAM `renditions/original` 404 locally is EXPECTED (not chased).
- **Targets the LIVE platform float grid** — `.aem-Grid`, `.aem-Grid--12 > .aem-GridColumn--default--4` (3-up), `.aem-GridColumn--default--12` (full-width). Clearfix on `.aem-Grid` (::before/::after + clear:both) + `clear:both` on the full-width (12) columns so Address (multiline), Document Checklist, and Declaration text reset the row and blocks never stagger/overlap. Widths are NOT redefined (platform classes own width). NO synthetic `.cmp-adaptiveform-panel__fields` grid invented.
- **Form title forced visible** — `.cmp-adaptiveform-container__title { display:block; visibility:visible; color:#1a2b5e; font-size:28px; font-weight:700; text-align:center }` (+ inheritance for inner h1). Matches the centered navy title in the screenshot.
- **Red required asterisks** — every `.cmp-adaptiveform-*__label__qualifier` = `#e02020` bold.
- **Navy Submit / outlined Reset** — keyed on the form's `css` hooks: `.vrf-submit` = navy filled + white text; `.vrf-reset` = white bg + outlined navy border + navy text. Also `[type=submit]`/`[type=reset]` fallbacks. Actions row centered (`.vrf-actions .aem-Grid--12 { display:flex; justify-content:center; gap:1rem }`).
- **Date-picker overlay** — `.cmp-adaptiveform-datepicker` z-index + `:focus-within { z-index:1000 }` so the calendar overlay is not clipped by adjacent float columns; clickable calendar indicator; reachable on mobile.
- **Visible focus ring** on inputs/dropdowns/checkboxes/radios/buttons (navy border + box-shadow).
- **3-column responsive** — desktop 3-up via platform `--default--4`; `@media (max-width:767px)` forces every column to 100% width / float:none / clear:both (single-column stack), full-width buttons, larger touch targets.
- **Contrast ≥ 4.5:1** — navy `#1a2b5e` text on light-blue `#e8eef7` and white on navy Submit both well above AA. Required state also conveyed by attribute/aria (not colour alone).
- **Icons** — DESI `icon_assets: []` (pure text/field form; no logo/icon in the screenshot). No icon rules authored; nothing inlined; nothing to reference from DAM. Correct per parity.

## Wiring (already in place from Phase 3 — verified consistent, no edit needed)
- Form guideContainer `themeRef="/apps/fd/af/themes/{project}-vehicle-registration"`
- Conf context `SiteConfig`: `themeArtifact="{project}-vehicle-registration"` + `siteTemplatePath="/apps/fd/af/themes/{project}-vehicle-registration"`
- Conf context `HtmlPageItemsConfig`: injects `/theme.css` + `/theme.js` at `{form}.theme/_default`
- All three reference strings identical → resolve the same /apps theme.

## Filters (already cover both artifacts — no edit needed)
- ui.apps filter: `/apps/fd/af/themes` (covers theme.zip folder)
- ui.content filter: `/content/dam/formsanddocuments-themes/{project}` (covers DAM theme-json)

## Self quality gate (verified)
- [x] /apps theme folder has `.content.xml` (`sling:Folder`, not `fd:AEMFormTheme`) + `theme.zip`
- [x] theme.zip built with PowerShell `Compress-Archive`; entries `theme.css` + `theme.js` at ROOT (verified via ZipFile.OpenRead)
- [x] DAM entry has `.content.xml` (`type=theme`, `sling:resourceType=fd/fm/theme/render`, `title`) + `theme-json` rendition + `original` text/css rendition (+ original.dir marker)
- [x] theme.css uses real `.cmp-adaptiveform-*` runtime classes; theme-json uses `af_*` nodes with `_x0023_` state attrs and `\,`-escaped commas
- [x] theme.css tokens == theme-json tokens (navy/light-blue/white/border/red/4px/font — table above)
- [x] No `/libs` overrides; no `fd:AEMFormTheme` node type
- [x] Title forced visible (navy bold centered); red asterisks; navy Submit + outlined Reset; centered actions row; 3-col responsive → single column <768px
- [x] Float-grid clearfix + full-width row reset (Address / Document Checklist / Declaration); date-picker overlay z-index; visible focus rings
- [x] project (`{project}`) vs appFolder kept correct; DAM path uses the repo's working `formsanddocuments-themes/{project}` convention
- [x] All 4 authored XML files well-formed ([xml] parse)
- [x] Filters already cover both artifact roots — no filter edit needed
- [x] NOT deployed — no `mvn` run (Auditon owns the single build/deploy)

## Handoff notes
- Visual parity (title/colours/asterisks/buttons rendering as pixels) is confirmed by the `test-form-ui` parity pass in the TEST phase (Sentinel) after Auditon deploys — serve-200 is not treated as proof.
- Theme lists in the picker as **"Vehicle Registration Theme"**.
