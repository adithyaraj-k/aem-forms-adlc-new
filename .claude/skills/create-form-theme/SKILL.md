---
name: create-form-theme
description: >
  Generates a custom AEM Adaptive Forms theme for AEM as a Cloud Service Core Components.
  Produces BOTH artifacts that make a theme work: the /apps theme.zip (renders in the local
  SDK) and the DAM theme-json entry (lists in the theme picker and renders on AEM Cloud).
version: 2.0.0
ide:
  cursor: .cursor/skills/create-form-theme/
  github-copilot: .github/skills/create-form-theme/
  claude-code: .claude/skills/create-form-theme/
---

# Skill: create-form-theme

## Role

You are an AEM Adaptive Forms UI specialist for AEM as a Cloud Service. You produce themes
that **actually apply** — verified end-to-end on a running AEM. The earlier clientlib-CSS
theme model (token `.css` under `/apps/.../clientlib`) does NOT serve and is abandoned.

> **Path vocabulary (collect from the user or `.aem-forms-config.yaml` — never hardcode a
> specific project):**
> `{project}` — the single project namespace, used both in `/apps/fd/af/themes/{project}-{theme}`
> AND as the folder segment in `/content/dam/formsanddocuments-themes/{project}/{theme}` (there is
> no separate app folder — the DAM themes root shares the same `{project}` segment as the form,
> DAM guide-asset, and `/conf/forms` roots);
> `{theme}` — theme name suffix; `{Theme Title}` — the human-readable picker label.

---

## How AF themes actually render (read first — decides what to generate)

A form's theme CSS is served at `{form}.theme/_default/theme.css` by
`com.adobe.aem.wcm.site.manager.internal.servlets.ServeThemeArtifactsServlet`, which reads a
**`theme.zip`** artifact. Two delivery locations exist, and a complete theme needs BOTH:

| Artifact | Path | Lists in picker? | Renders LOCAL SDK? | Renders CLOUD? |
|---|---|---|---|---|
| **`/apps` theme.zip** | `/apps/fd/af/themes/{project}-{theme}/theme.zip` | ❌ no | ✅ yes (the only thing that renders locally) | ✅ yes |
| **DAM theme-json** | `/content/dam/formsanddocuments-themes/{project}/{theme}/` | ✅ yes | ❌ no (cloud pipeline compiles it; local SDK can't) | ✅ yes |

So: generate the **`/apps` theme.zip** for local rendering + the **DAM theme-json** so the
theme appears in the AF Properties → Theme picker and renders on cloud. Use the SAME visual
design in both.

> **The DAM `…/{theme}/jcr:content/renditions/original` path returns HTTP 404 on the local
> SDK — that is NORMAL, not a bug.** The cloud pipeline compiles the theme-json into that
> rendition; the local SDK cannot. Do **not** chase the 404 as the cause of a broken local
> form. The local stylesheet source of truth is **`theme.zip` → `theme.css`**, served at
> `{form}.theme/_default/theme.css`. If the form looks unstyled locally, the fix is to rebuild
> `theme.zip` with the correct `theme.css` — never to "make the 404 go away".

---

## 🛑 NEVER style a form via a `css="…"` class hook — the property is NOT rendered as a DOM class

A field/panel's **`css="my-hook"` property does NOT become a class on the served element.** This
project renders Adaptive Forms with the Core Components **HTL** (server side); the `css` value
surfaces only as `"css":"my-hook"` inside `guideContainer.model.json` and is never written into the
markup. So every CSS rule keyed to `.my-hook` matches **nothing**.

This is silent in every check short of looking at pixels: the build succeeds, the clientlib and
`theme.css` both serve HTTP 200, the rule is visibly present in the CSS, and the class is visible in
the model JSON. Only the rendered form is wrong — it falls back to base styling. On
event-ticket-booking this killed the form title, subtitle rule, section bands, read-only amount
fills, declaration checkbox, terms accent, and both buttons at once.

**Confirm before you rely on ANY selector** (works on the local SDK, no browser needed):

```bash
curl.exe -s -u admin:admin \
  "http://localhost:4502/content/forms/af/{appFolder}/{formName}.html" -o /tmp/f.html
grep -o 'my-hook' /tmp/f.html | wc -l     # 0 => the selector is dead, rewrite it
```

**Style the classes/attributes the HTL really emits.** Verified served structure:

| Target | Served selector |
|---|---|
| Form title (h1 AF Title) | `h1.cmp-title__text` (wrapper `.cmp-title`, grid cell `.title`) |
| Section heading (h2 AF Title) | `h2.cmp-title__text` |
| Section heading (panel label) | `.cmp-container__label` |
| Static text / subtitle | `.cmp-adaptiveform-text__label` |
| Panel | `.panelcontainer.responsivegrid` |
| Field wrapper | `.cmp-adaptiveform-{textinput,emailinput,telephoneinput,numberinput,datepicker,dropdown,checkbox,radiobutton}` |
| Label / widget / error | `…__label`, `…__widget`, `…__errormessage` |
| Multi-line text input | `.cmp-adaptiveform-textinput textarea` |
| Radio/checkbox GROUP container | `.cmp-adaptiveform-radiobutton__widget` (+ `HORIZONTAL`/`VERTICAL`) |
| Radio INPUT (the actual control) | `.cmp-adaptiveform-radiobutton__option__widget` |
| Radio option label | `.cmp-adaptiveform-radiobutton__option-label` |
| Required marker | `[data-cmp-required="true"] …__label::after` |
| Read-only field | `[data-cmp-readonly="true"] …__widget` |
| Invalid field | `[data-cmp-valid="false"] …__widget`, `…__widget[aria-invalid="true"]` |
| Submit vs Reset button | `button[type="submit"]`, `button[type="reset"]` (both `.cmp-adaptiveform-button__widget`) |
| Grid column span | `.aem-GridColumn--default--{12,6,4,3}` |

Because `.cmp-title__text` is shared by the form title and every section heading, discriminate them
by **element** (`h1` vs `h2`, set by `fd:htmlelementType` on each AF Title). `h1.cmp-title__text` is
specificity (0,1,1) so it also beats a bare `.cmp-title__text` base rule. Where no class distinguishes
a region, use structure (`:has()` is supported and already used in this repo's base clientlib), e.g.
the actions row: `.panelcontainer:has(.cmp-adaptiveform-button) > .cmp-container > .aem-Grid`.

The same applies to **JS**: `document.querySelectorAll(".my-hook")` finds nothing. Target served
classes there too. A class your own JS *injects* is fine to style (that class does exist at runtime).

---

## Base theme via CSS-variable design tokens (Option A) — READ BEFORE AUTHORING

**AF has NO native theme-extends-theme inheritance.** A form references exactly ONE theme via
`themeRef`, and a theme is a self-contained compiled `theme.css` bundle (served from
`/apps/fd/af/themes/{project}-{theme}/theme.zip`, injected as a `<link>` by the conf
HtmlPageItemsConfig) plus a DAM theme-json entry for the picker. There is no "parent theme".
Inheritance is therefore implemented with **CSS custom properties (design tokens) + the cascade**,
NOT a theme parent. This is **Option A**, the mandatory architecture for this project:

- **BASE LAYER = the shared base forms clientlib** (`{project}.forms.base`, under
  `/apps/{project}/clientlibs`, loaded on EVERY form via the comma-separated `clientLibRef` — see
  `create-form-clientlib`). The base declares **ALL design tokens with default values in `:root`**
  and writes the **STANDARD styling for common elements CONSUMING those tokens** — labels, text /
  number inputs, dropdowns, radios / checkboxes, textarea, `.cmp-title__text`, section headings
  `.cmp-container__label`, section separators, the required asterisk
  (`[data-cmp-required] …__label::after`), buttons, and the served grid. **Standard styling is
  authored ONCE in the base — never re-authored per form.**
- **A FORM-SPECIFIC THEME = token override ONLY.** Each form's `theme.zip` `theme.css` (and the DAM
  theme-json) shrinks to a `:root { --token: <brand value>; }` block that overrides just the brand
  tokens (plus rare form-unique rules). **Do NOT re-author the standard element styling in a form
  theme** — the base already provides it; the theme only supplies brand values.
- **The TOKEN CONTRACT (the theme API — the only knobs a form theme touches):**
  - color: `--af-primary`, `--af-primary-contrast`, `--af-bg`, `--af-section-heading`,
    `--af-field-border`, `--af-error`, `--af-required-asterisk`
  - type: `--af-font`, `--af-font-size-base`, `--af-heading-weight`
  - shape / space: `--af-radius`, `--af-field-height`, `--af-gap`, `--af-section-rule`

  The contract is extensible, but keep it the **documented** set — new tokens go in the base `:root`
  defaults first, then this list.
- **CASCADE / ORDER GOTCHA.** The form theme's `:root` override must WIN over the base defaults. The
  theme `<link>` is injected AFTER the base clientlib loads, so equal-specificity `:root` rules from
  the theme naturally override the base — **preserve that load order and specificity** (don't raise
  base token specificity above a plain `:root`, and don't move the theme link before the base).
- 🛑 **EMBEDDED PAGE: the theme alone is NOT enough.** When the form is embedded in a Sites page
  (the `composer`/`assembler` phase — every delivery), the **form theme selector does not apply
  in-page** — only the page's `formEmbedClientlibs` categories load, so the `runtime.all` default
  **blue** wins and the in-page form is off-brand. The brand `:root` override MUST therefore ALSO be
  mirrored into the **form-specific clientlib** `form.css` (the category wired into
  `formEmbedClientlibs`), with the SAME token values you set here. Author the token override in BOTH
  places. See `create-form-clientlib` → "The form clientlib MUST mirror the brand `:root` token
  override" and [[form-clientlib-mirrors-brand-tokens-for-embed]].
- **REUSE-FIRST for themes** (the mirror of the template/clientlib reuse rule): a new form does NOT
  get a hand-authored full stylesheet — it **overrides tokens over the base**. A genuinely new full
  theme (re-authoring standard styling) needs a **recorded justification**; default to a thin token
  override.

Example thin form theme `theme.css` (this is the WHOLE stylesheet a branded form needs):
```css
/* {project}-{theme} — token override ONLY; standard styling comes from {project}.forms.base */
:root {
    --af-primary: #0b5cab;
    --af-primary-contrast: #ffffff;
    --af-bg: #f4f7fb;
    --af-section-heading: #0b3a6b;
    --af-field-border: #c7d2e0;
    --af-font: "Inter", system-ui, sans-serif;
    --af-radius: 8px;
    --af-section-rule: 1px solid #d9e2ec;
}
/* rare form-unique rules only — never re-declare the standard element styling */
```
Update the **DAM theme-json the same way** — its `af_*` nodes carry only the brand token overrides
(page background, primary/button colour, heading colour), not a full re-styling of every element.

---

## Trigger

Create a custom theme, brand a form to a design system, change form colours/typography, or
add a theme to the picker.

---

## Artifact 1 — `/apps` theme.zip (LOCAL rendering)

```
ui.apps/src/main/content/jcr_root/apps/fd/af/themes/{project}-{theme}/
├── .content.xml          ← sling:Folder + jcr:content/metadata (title/description)
└── theme.zip             ← contains theme.css + theme.js at the zip root
```

`.content.xml` (plain `sling:Folder` — NOT `fd:AEMFormTheme`, which is unregistered):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0" xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    jcr:primaryType="sling:Folder">
    <jcr:content jcr:primaryType="nt:unstructured">
        <metadata jcr:primaryType="nt:unstructured" author="{project}"
            description="{one-line}" title="{project}-{theme}"/>
    </jcr:content>
</jcr:root>
```

Build `theme.zip` (Git Bash has no `zip` — use PowerShell):
```powershell
Compress-Archive -Path theme.css,theme.js -DestinationPath theme.zip -CompressionLevel Optimal
```
`theme.js` can be a no-op: `(function(){})();`. `theme.css` uses the runtime BEM classes
below (UNscoped — the theme only applies to forms that select it):

```css
.cmp-adaptiveform-container { background: linear-gradient(135deg,#A,#B) fixed; min-height:100vh; padding:2.5rem 1.5rem; }
/* Form TITLE — the DOM is `<div class="cmp-title"><h1 class="cmp-title__text">`, NOT
   `cmp-adaptiveform-container__title` (that class does not exist in the served DOM). Target the
   real class or the title renders unstyled/left-aligned: */
.cmp-title { text-align:center; }
.cmp-title__text { text-align:center; color:#333; font-size:2rem; font-weight:700; }
/* SECTION HEADINGS render as `<label class="cmp-container__label">` inside a role="heading" div and
   are NOT bold by default — you MUST style them (references usually show bold section headers): */
.cmp-container__label { font-weight:700; color:#333; }
.cmp-adaptiveform-{textinput,emailinput,telephoneinput,numberinput,datepicker,dropdown}__widget { background:#fff; border:1px solid #e2e2ea; border-radius:8px; box-shadow:0 1px 3px rgba(0,0,0,.08); min-height:46px; padding:10px 14px; width:100%; }
.cmp-adaptiveform-*__label { color:#333; font-weight:500; }       /* labels */
/* Required red asterisk — DO NOT rely on __label__qualifier: Core Components does NOT emit that
   span, so a rule targeting it renders NO asterisk. Style ::after off the wrapper's
   data-cmp-required="true" attribute (optional fields are data-cmp-required="false" → auto-excluded): */
[data-cmp-required="true"] .cmp-adaptiveform-textinput__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-emailinput__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-telephoneinput__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-numberinput__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-datepicker__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-dropdown__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-radiobutton__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-checkbox__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-checkboxgroup__label::after {
    content:" *"; color:#e3342f; font-weight:700;                  /* required asterisk */
}
.cmp-adaptiveform-button__widget { background:#PRIMARY; color:#fff; border-radius:8px; font-weight:600; padding:12px 36px; }
.cmp-adaptiveform-button { text-align:center; }
```

> **Required-asterisk gotcha (verify by pixels).** The page loads a cached, aggregated
> `.../{form}.theme/_default/theme.css` that **lags the redeployed theme.zip** — a package install
> does NOT bust this AF theme-servlet cache (JCR touch, clientlib invalidate, even a bundle restart
> do not refresh it). So the asterisk rule can be correct in `theme.zip`/DAM `original` yet the served
> aggregate is stale and the `*` never appears. For a guaranteed render, ALSO put the identical
> `[data-cmp-required="true"] …__label::after` rule in the **per-form clientlib CSS** (create-form-clientlib),
> which is served fresh on every deploy. Confirm the red `*` actually renders on required labels (and
> NOT on optional fields) in a rendered capture — CSS presence in a file is not proof.

---

## Artifact 2 — DAM theme-json (PICKER listing + CLOUD rendering)

```
ui.content/src/main/content/jcr_root/content/dam/formsanddocuments-themes/{project}/{theme}/
├── .content.xml                                  ← dam:Asset, type=theme
└── _jcr_content/renditions/theme-json/.content.xml  ← the theme structure (af_* nodes)
```

Root `.content.xml` (the `type="theme"` + `sling:resourceType="fd/fm/theme/render"` markers
are what make it appear in the picker; `title` is the picker label):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:fd="http://www.adobe.com/aemfd/fd/1.0" xmlns:dam="http://www.day.com/dam/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="dam:Asset">
    <jcr:content jcr:primaryType="dam:AssetContent" sling:resourceType="fd/fm/theme/render" theme="1" type="theme">
        <metadata author="{project}" fd:targetVersion="2.0" fd:version="2.0" jcr:primaryType="nt:unstructured"
            clientlibCategory="fdtheme.{project}.{theme}" clientlibRef="/etc/clientlibs/fd/themes/{project}"
            formRef="/libs/fd/af/themes/default" title="{Theme Title}">
            <breakpoints jcr:primaryType="nt:unstructured">
                <default jcr:primaryType="nt:unstructured" title="Default" width="-1"/>
                <smallScreen jcr:primaryType="nt:unstructured" title="Smaller Screen" width="479"/>
                <phone jcr:primaryType="nt:unstructured" title="Phone" width="767"/>
                <tablet jcr:primaryType="nt:unstructured" title="Tablet" width="991"/>
            </breakpoints>
        </metadata>
    </jcr:content>
</jcr:root>
```

`_jcr_content/renditions/theme-json/.content.xml` (a **partial** theme — `formRef` inherits
the default theme; override only the brand nodes). State attrs use `_x0023_` = `#`
(`default#default`, `default#hover`); values are bracketed `[prop:value,prop:value]` arrays
with commas INSIDE a value escaped `\,`. Pair each style attr with its `..._x0023_..._x0023_ui`
companion. Key nodes: `af_page` (page bg/gradient), `af_formtitle`, `af_fieldlabel`,
`af_widgetAndText` (inputs), `af_button` (submit):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="sling:Folder">
    <jcr:content jcr:primaryType="nt:unstructured" rendition.handler.id="theme.structure">
        <components jcr:primaryType="nt:unstructured">
            <af_guideContainer jcr:primaryType="nt:unstructured" component="fd/af/components/guideContainer">
                <af_page jcr:primaryType="nt:unstructured"
                    default_x0023_default="[min-height:100vh,padding-top:2.5rem,padding-bottom:2.5rem,padding-left:1.5rem,padding-right:1.5rem,background:linear-gradient(135deg\, #A 0%\, #B 100%) fixed]"
                    default_x0023_default_x0023_ui="[backgroundColor:#B]"/>
                <af_button jcr:primaryType="nt:unstructured"
                    default_x0023_default="[background:#PRIMARY,color:#ffffff,border-color:#PRIMARY,border-style:solid,border-top-width:1px,border-left-width:1px,border-bottom-width:1px,border-right-width:1px,border-bottom-left-radius:0.5rem,border-bottom-right-radius:0.5rem,border-top-right-radius:0.5rem,border-top-left-radius:0.5rem,padding-top:0.7rem,padding-bottom:0.7rem,padding-left:2.25rem,padding-right:2.25rem,font-weight:600,cssOverride:cursor: pointer;]"
                    default_x0023_default_x0023_ui="[backgroundColor:#PRIMARY,borderWidthPopover:1px,borderRadiusPopover:0.5rem]"
                    default_x0023_hover="[background:#PRIMARY_DARK,color:#ffffff,border-color:#PRIMARY_DARK]"
                    default_x0023_hover_x0023_ui="[backgroundColor:#PRIMARY_DARK]"/>
                <!-- add af_formtitle, af_fieldlabel, af_widgetAndText similarly -->
            </af_guideContainer>
        </components>
    </jcr:content>
</jcr:root>
```

Filter: `/content/dam/formsanddocuments-themes/{project}` is already a filter root in
`ui.content` — DAM theme entries deploy without changes.

---

## Wiring the theme to a form

A form's theme is wired in **TWO places, and BOTH must point at the same theme** — repointing
only one silently leaves the old theme serving CSS. This bit a real delivery (customer-feedback
rendered with the wrong theme's background + grid because only `themeRef` was changed):

1. **The form's `guideContainer/@themeRef`** — in
   `ui.content/.../content/forms/af/{project}/{formName}/.content.xml`. Points to the **`/apps`**
   path (`/apps/fd/af/themes/{project}-{theme}`) so it renders locally.
2. **The per-form conf-context `SiteConfig`** — in
   `ui.content/.../conf/forms/{project}/{formName}/.content.xml`, attributes **`themeArtifact`**
   and **`siteTemplatePath`**. This is what serves the aggregated `{form}.theme/_default/theme.css`.

> **⚠️ Repoint BOTH when changing a form's theme.** If you update `themeRef` but leave the conf
> `SiteConfig` `themeArtifact`/`siteTemplatePath` on the old theme, the servlet keeps serving the
> OLD theme's `theme.css` at `{form}.theme/_default/theme.css` — so the form still renders with the
> old background, grid, and colours no matter what `themeRef` says. When migrating a form off one
> theme onto another, grep the form's `.content.xml` AND its conf `.content.xml` and confirm **zero
> residual references** to the old theme name.

On cloud / via the picker, AEM writes the DAM-theme selection. Both resolve the same theme.

### Never leave a form on a generic / shared theme and expect reference parity

A form branded to a reference needs its **OWN dedicated theme** — do **NOT** bind it to a generic,
shared, or sample theme (e.g. a `wknd`/starter theme) and hope it matches. A shared theme's compiled
`theme.css` typically ships a **decorative page background (gradient / diagonal bands / image) and its
own grid + container styling** that **override the base clientlib's clean full-width flex grid**,
producing exactly the failure the user reports: off-brand background, fields crammed into a narrow
single column, boxed sections, misaligned layout. The fix that works (and the pattern to mirror) is a
**thin per-form token-override theme** (Option A, above) whose `theme.css` sets ONLY `:root --af-*`
brand tokens and a clean background — and does **NOT** re-declare grid/container rules — so the base
clientlib's grid and standard styling still apply. One dedicated theme per branded form.

---

## Match the reference exactly — title, icons, colours (mandatory when a screenshot/design is the source)

When the form is being branded to a **reference screenshot, mockup, or design**, the theme is
the artifact that makes the rendered form *look like the reference*. Every one of these has
bitten a real delivery — apply all of them:

- **The form title MUST be visibly rendered — and it comes from an explicit AF Title component.**
  The guideContainer v2 `showTitle` band does **NOT** emit a `.cmp-adaptiveform-container__title`
  element in the served DOM, so a rule targeting that class styles nothing and the title renders as an
  **empty band** (defect TC-031). `create-adaptive-form` therefore adds an explicit **AF Title (v2)**
  component (first child of the form, `fd:htmlelementType="h1"`), with the guideContainer
  `showTitle="{Boolean}false"`. This emits a real `<div class="cmp-title"><h1 class="cmp-title__text">…`.
  So style the **actual served classes** `.cmp-title` / `h1.cmp-title__text` — NOT
  `.cmp-adaptiveform-container__title` (which does not exist in the DOM) and NOT the node's `css=`
  value (never emitted — see the guard at the top of this skill). For a guaranteed render put
  the title styling in the **form clientlib CSS** (served fresh; the aggregated theme.css lags). A
  missing/invisible title is "wrong class / empty showTitle band", **not** missing content — confirm the
  title text actually shows on the rendered form (by pixels).
- **Reproduce every word.** Title, subtitle/description, section headers, legends, help text,
  button labels, declaration text — if it is in the reference it must be present AND visible in
  the rendered form. (Requirements capture lists them; the theme must not hide any of them.)
- **Match colours to the exact reference value.** Sample the real hex from the screenshot for
  section-header colour, primary/button colour, accents — never guess or reuse a previous
  form's brand colour. A "close enough" colour is a parity failure.
- **Reproduce EVERY visual element in the reference — not only fields.** This explicitly includes
  **section separator / divider rules**: the thin horizontal line under each numbered section and
  under the title/subtitle header, plus background bands. Draw separators in the **form clientlib
  CSS** (served fresh on every deploy) as a `border-bottom` (e.g. `1px solid #d9d9d9`) on the real
  served section-panel classes + a rule under the header — verify the actual served DOM classes, do
  not invent them. Confirm by pixels after redeploy, not by CSS presence.
- **Reproduce icons — as AUTHORED IMAGE COMPONENTS, never via theme CSS.** Branding/header icons,
  per-section-header icons, and in-tile icons (upload/choice cards) from the reference must appear.
  Every one is a stored DAM asset rendered by an authored Adaptive Form **Image** component
  (`fileReference` → DAM asset — see `create-adaptive-form` → "Icons & imagery from a reference").
  The theme/clientlib CSS must **NOT** supply any image — do **NOT** use `background-image:url(...)`,
  `::before` glyphs, inline SVG, or base64 to draw icons, **not even** decorative section-heading
  micro-icons or tile icons. This supersedes any earlier guidance that allowed the "CSS
  background-image URL" approach. Theme CSS only **sizes and positions** those authored Image
  components (via the served `.cmp-adaptiveform-image*` classes / grid cell — **not** a `css=` hook,
  which is never emitted) and applies brand tokens — the image itself always comes from
  the component's `fileReference`. If the reference has an icon, the stored asset must exist and an
  Image component must render it.
- **Layout columns are the platform FLOAT grid.** Column widths, multi-up rows, and field
  overlap are governed by `.aem-Grid` / `.aem-GridColumn` — see `create-form-clientlib` for the
  exact verified selector chain. Don't invent `.cmp-adaptiveform-panel__fields`-style wrappers;
  they don't exist in Core Components and the CSS silently does nothing → overlap/stagger.
- **Verify VISUALLY, not structurally.** "CSS serves HTTP 200" and "the rule is present" do
  **not** prove the title/icons/colours actually render. The only proof is the rendered form
  matching the reference (the `test-form-ui` parity pass). Do not report a UI fix as done on a
  serve-200 check alone.

---

## Build the theme from a captured source-form STYLE SPEC (exact-visual-replica path)

When the delivery is a **URL replica** — turning a form on a public webpage into a native AEM
Adaptive Form — the design phase hands you a captured **STYLE SPEC** of the source form (read from
its CSS): fonts (family/size/weight), colours, borders, card/container styling, spacing,
**multi-column field rows** (column structure + field order), and button styling. This theme is the
artifact that makes the AF render look like the source form. Build it as the usual thin
token-override (Option A), with the token VALUES taken straight from the STYLE SPEC and applied to
the real AF `cmp-adaptiveform-*` runtime markup:

- **Fonts** → `--af-font` / `--af-font-size-base` / `--af-heading-weight` (label, input, and title
  type match the source family/size/weight).
- **Colours** → `--af-primary` (buttons/accents), `--af-primary-contrast`, `--af-bg` (page/card
  background), `--af-section-heading`, `--af-field-border`, `--af-error` — sample the exact source
  values, never guess.
- **Card / container + borders + spacing** → `--af-radius`, `--af-field-border`, `--af-section-rule`,
  `--af-gap`, `--af-field-height` so the container/card and field spacing match the source.
- **Button** → the source's fill/border/radius/padding on `.cmp-adaptiveform-button__widget`
  (via the button tokens).
- **Multi-column field rows** → these are the platform FLOAT grid (`.aem-Grid` / `.aem-GridColumn`),
  authored on the form and driven by `cq:responsive`; the theme reproduces the source's column widths
  and row alignment by supplying the brand tokens/spacing — do NOT re-declare the grid in the theme
  (see "Never leave a form on a generic / shared theme" above; a generic single-column theme is a
  replica FAIL).

Then wire this theme to the replica form in BOTH places ("Wiring the theme to a form", above) so it
serves. Only the **form** is replicated — never the page chrome (nav/header/footer/ads).

> **Honest scope: faithful match on the AF DOM, not guaranteed pixel-identical.** The AF
> `cmp-adaptiveform-*` markup differs structurally from the source page's HTML, so the goal is a
> **faithful visual match on the AF DOM** — the same fonts, colours, card, spacing, multi-column
> layout, and button — not a byte-for-byte pixel-identical clone. Verify VISUALLY against the source
> (the `test-form-ui` parity pass), and close the gap by tuning tokens — not by re-authoring a full
> stylesheet.

---

## Quality checklist

- [ ] **Option A honored — the theme is a THIN token override, not a full stylesheet.** `theme.css`
      redeclares only `:root { --af-* }` brand tokens (+ rare form-unique rules) over the base
      clientlib's standard styling; the standard element styling is NOT re-authored in the theme, and
      the DAM theme-json `af_*` nodes carry only the token overrides. A full new theme has a recorded
      justification. The token contract (`--af-primary`, `--af-bg`, `--af-section-heading`,
      `--af-field-border`, `--af-error`, `--af-font`, `--af-radius`, …) is the only API touched, and
      the theme `<link>` still loads after the base so the `:root` override wins (no native
      theme-parent — inheritance is tokens + cascade)
- [ ] `/apps/fd/af/themes/{project}-{theme}/` has `.content.xml` (`sling:Folder`) + `theme.zip` (theme.css+theme.js at root)
- [ ] `theme.zip` built with PowerShell `Compress-Archive` (not Git Bash `zip`)
- [ ] DAM entry `/content/dam/formsanddocuments-themes/{project}/{theme}/` with `.content.xml` (`type=theme`, `sling:resourceType=fd/fm/theme/render`, `title`) + `theme-json` rendition
- [ ] theme.css uses real `.cmp-adaptiveform-*` runtime classes; theme-json uses `af_*` nodes with `_x0023_` state attrs and `\,`-escaped commas
- [ ] **ZERO selectors keyed to a node's `css="…"` hook** (never emitted as a DOM class). Every class in `theme.css` and in the form clientlib CSS/JS was confirmed present in the served markup: `curl.exe -s -u admin:admin ".../{formName}.html" -o /tmp/f.html` then `grep -c '<class>' /tmp/f.html` > 0 for each. A count of 0 = dead rule → rewrite against the served-selector table
- [ ] Radio/checkbox: the 17px/accent-color sizing is on `…radiobutton__option__widget` (the input), NOT on `…radiobutton__widget` (the group container) — and there is no `…__widgets` class
- [ ] Where one class serves two roles, the discriminator is structural (`h1` vs `h2.cmp-title__text`, `button[type=submit]` vs `[type=reset]`, `:has()`), not an invented hook
- [ ] `theme.zip`'s `theme.css` is byte-identical to the form clientlib's copy if the theme is mirrored there (`diff` them) — a stale copy silently re-introduces the old look
- [ ] No `fd:AEMFormTheme` node type; no `/libs` overrides
- [ ] Deploy + VERIFY on running AEM: theme appears in the DAM themes list (picker), and a form wired to the `/apps` theme serves `{form}.theme/_default/theme.css` = 200 with the theme CSS
- [ ] Remember: DAM theme-json does NOT render in the local SDK (only cloud) — local rendering is via the `/apps` theme.zip; the DAM `renditions/original` 404 locally is EXPECTED, not the bug
- [ ] **When branding to a reference:** the form **title is visibly rendered**, EVERY word from the reference is present & visible, section-header **colours match the exact reference hex**, and all reference **icons render** — confirmed **VISUALLY** against the reference (serve-200 is NOT proof)
- [ ] **URL replica (exact-visual-replica path):** a captured source-form STYLE SPEC is mapped to `--af-*` token values (fonts, colours, card/container, borders, spacing, button) on the AF `cmp-adaptiveform-*` markup, the source's multi-column field rows are honoured via the grid (not collapsed to generic single column), the theme is wired in BOTH places, only the form (not page chrome) is replicated, and parity is verified VISUALLY as a faithful AF-DOM match (not guaranteed pixel-identical)
