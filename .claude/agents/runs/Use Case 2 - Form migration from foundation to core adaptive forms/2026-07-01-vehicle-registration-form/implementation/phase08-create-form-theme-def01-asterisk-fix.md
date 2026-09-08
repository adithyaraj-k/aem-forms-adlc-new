# Phase 08 — create-form-theme — DEF-01 Required-Asterisk Fix (AUTHOR-ONLY)

Theme: `{project}-vehicle-registration`
Mode: pipeline / AUTHOR-ONLY. **No mvn build or deploy was run** (Auditon redeploys).
Existing theme MODIFIED in place — no new theme created.

## Defect
DEF-01: Red required-field asterisks were NOT rendering despite 23 required fields.

## Root cause (verified against the live served DOM)
The theme colored a `.cmp-adaptiveform-*__label__qualifier` element that **does not exist**
in the served markup. A grep of the rendered form at
`http://localhost:4502/content/forms/af/{appFolder}/vehicle-registration-form.html`
returns **0 occurrences of "qualifier"**, so the qualifier rule had nothing to target and no
asterisk ever rendered.

The served required-field markup is:
```
<div class="cmp-adaptiveform-textinput" ... data-cmp-required="true" ...>
  <div class="cmp-adaptiveform-textinput__label-container">
    <label ... class="cmp-adaptiveform-textinput__label">Full Name</label>
  </div>
  <input class="cmp-adaptiveform-textinput__widget" ... required .../>
```
- `data-cmp-required="true"` sits on the component wrapper div and appears **exactly 23 times**
  (= the 23 required fields) — the reliable required-ONLY hook.
- There is **no qualifier span**, so the asterisk must be injected with a `::after`
  pseudo-element on the existing `__label`.
- `aria-required` is **not** present, so it cannot be relied on.

## Files changed (absolute paths)
1. `C:\Software\AEMaaCS\git\AEM-Adaptive-Forms-Agent\ui.apps\src\main\content\jcr_root\apps\fd\af\themes\{project}-vehicle-registration\theme.zip`
   (SERVED locally — theme.css inside rebuilt; theme.js unchanged no-op)
2. `C:\Software\AEMaaCS\git\AEM-Adaptive-Forms-Agent\ui.content\src\main\content\jcr_root\content\dam\formsanddocuments-themes\{project}\vehicle-registration\_jcr_content\renditions\theme-json\.content.xml`
   (DAM theme-json — added a `mandatory`-state `#e02020` asterisk marker on `af_fieldlabel`,
   consistent with theme.css; the existing `af_widgetAndText` error state already carries `#e02020`)
3. `C:\Software\AEMaaCS\git\AEM-Adaptive-Forms-Agent\ui.content\src\main\content\jcr_root\content\dam\formsanddocuments-themes\{project}\vehicle-registration\_jcr_content\renditions\original`
   (text/css rendition — was a stale copy carrying the broken `__qualifier` block; overwritten
   to match the corrected theme.css so the DAM path stays non-404 and consistent)

## OLD broken selector (removed)
```css
.cmp-adaptiveform-textinput__label__qualifier,
.cmp-adaptiveform-emailinput__label__qualifier,
.cmp-adaptiveform-telephoneinput__label__qualifier,
.cmp-adaptiveform-numberinput__label__qualifier,
.cmp-adaptiveform-datepicker__label__qualifier,
.cmp-adaptiveform-dropdown__label__qualifier,
.cmp-adaptiveform-radiobutton__label__qualifier,
.cmp-adaptiveform-checkbox__label__qualifier,
.cmp-adaptiveform-checkboxgroup__label__qualifier {
  color: var(--vrf-asterisk-red);
  font-weight: 700;
}
```
(The `original` rendition additionally carried an *unscoped* `[data-cmp-required="true"] .cmp-…__label::after`
variant AND still kept the broken qualifier block — both replaced by the scoped form below.)

## NEW selector (in all three artifacts)
```css
.cmp-adaptiveform-textinput[data-cmp-required="true"] .cmp-adaptiveform-textinput__label::after,
.cmp-adaptiveform-emailinput[data-cmp-required="true"] .cmp-adaptiveform-emailinput__label::after,
.cmp-adaptiveform-telephoneinput[data-cmp-required="true"] .cmp-adaptiveform-telephoneinput__label::after,
.cmp-adaptiveform-numberinput[data-cmp-required="true"] .cmp-adaptiveform-numberinput__label::after,
.cmp-adaptiveform-datepicker[data-cmp-required="true"] .cmp-adaptiveform-datepicker__label::after,
.cmp-adaptiveform-dropdown[data-cmp-required="true"] .cmp-adaptiveform-dropdown__label::after,
.cmp-adaptiveform-radiobutton[data-cmp-required="true"] .cmp-adaptiveform-radiobutton__label::after,
.cmp-adaptiveform-checkbox[data-cmp-required="true"] .cmp-adaptiveform-checkbox__label::after,
.cmp-adaptiveform-checkboxgroup[data-cmp-required="true"] .cmp-adaptiveform-checkboxgroup__label::after {
  content: " *";
  color: var(--vrf-asterisk-red);   /* #e02020 */
  font-weight: 700;
  margin-left: 2px;
}
```

## Why the new selector matches the served DOM
- Each required field's component wrapper carries `data-cmp-required="true"` — present **exactly
  23 times**, one per required field. The attribute selector on the wrapper scopes the rule to
  required fields ONLY (optional fields get no asterisk).
- The wrapper has a real child `.cmp-adaptiveform-{type}__label` (the base label rule already
  styles it navy/600). `::after` on that existing label injects the asterisk glyph — no
  non-existent `qualifier` span required.
- Covers every field component type used on the form (textinput, emailinput, telephoneinput,
  numberinput, datepicker, dropdown, radiobutton, checkbox, checkboxgroup).
- Does not rely on `aria-required` (absent in this render).

## theme.zip re-zip confirmation
- Unzipped the existing theme.zip, edited only theme.css, re-zipped with PowerShell
  `Compress-Archive ... -CompressionLevel Optimal -Force` (NOT git bash zip).
- Verified ZIP entries are at ROOT, no subfolders:
  - `theme.css` (15074 bytes)
  - `theme.js` (18 bytes) — unchanged no-op `(function(){})();`

## No token regression (verified in the rebuilt theme.css)
- `--vrf-asterisk-red: #e02020` :root token kept.
- Navy `#1a2b5e` present (headings/section headings/labels/Submit bg/Reset border+text).
- Light-blue `#e8eef7` page background; white `#ffffff` field bg; `#c9d3e6` field border — present.
- Navy filled `.vrf-submit` / white outlined-navy `.vrf-reset`; centered `.vrf-actions` row — unchanged.
- 3-col platform float grid (`.aem-Grid--12 > .aem-GridColumn--default--4`), clearfix, full-width
  (12) row reset — unchanged.
- Title forced visible (`.cmp-adaptiveform-container__title` navy bold centered) — unchanged.
- Date-picker overlay z-index; focus rings; `@media(max-width:767px)` single-column stack — unchanged.
- Only the asterisk/qualifier block changed. Grep confirms **0** `__qualifier` selectors remain
  (the single remaining string is inside the explanatory comment) and **10** `data-cmp-required`
  selector lines (9 component selectors + 1 in the added-comment description).

## Deploy confirmation
**No mvn build or deploy was executed.** Artifacts authored only; Auditon performs the single
authoritative build + deploy of record.
