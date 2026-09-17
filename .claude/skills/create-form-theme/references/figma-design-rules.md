# Figma Design Rules for Adaptive Forms (Use Case 1)

Rules for turning a Figma-sourced `style_spec` (captured by `discover-form-requirements` Step 1b
via the Figma MCP tools) into the project's Adaptive Forms theme artifacts — the `--af-*` token
override (`create-form-theme` Option A) and the field grid spans (`create-adaptive-form`). This
is additive to the main `create-form-theme/SKILL.md` rules (served-class table, Option A token
contract, "Match the reference exactly") — it does not replace them. Everything here exists
because a real delivery (`college-event-registration-form`, 2026-09-11 run) mis-rendered a
Figma-sourced form until it was hand-remediated; these rules close that gap up front.

---

## Rule 1: Figma variable/style names → the `--af-*` token contract

`mcp__figma__get_variable_defs` returns colour, typography, spacing, and radius variables by
**name**, not by AF role. Map them onto the existing token contract (never invent new tokens
without adding a base default first — see `create-form-theme` → "Token contract"):

| Figma variable/style name pattern | AF token |
|---|---|
| Primary / Brand / Accent / Button colour | `--af-primary` |
| Primary text-on-colour / Button label colour | `--af-primary-contrast` |
| Background / Surface / Page colour | `--af-bg` |
| Heading / Section / Title colour (when distinct from primary) | `--af-section-heading` |
| Border / Stroke / Outline (field-level) | `--af-field-border` |
| Error / Danger / Destructive colour | `--af-error` |
| Required / Asterisk colour (if a separate token exists; else reuse `--af-error`) | `--af-required-asterisk` |
| Body / Input font family | `--af-font` |
| Body / Input font size | `--af-font-size-base` |
| Heading font weight | `--af-heading-weight` |
| Corner radius (field/button/card) | `--af-radius` |
| Field height / input sizing | `--af-field-height` |
| Gap / spacing (between fields, rows, sections) | `--af-gap` |
| Divider / section-rule stroke | `--af-section-rule` |

Record the exact Figma variable name next to each token value in the theme's header comment
(e.g. `--af-primary: #0057b8; /* Figma: Color/Brand/Primary */`) so the mapping is traceable back
to the design system, the same way `figma-design-rules.md` in a Sites-component project traces
CSS values back to Tailwind classes.

**Never substitute or "improve" a colour/font** — use the exact value from `get_variable_defs` or,
if variables aren't used in the file, the exact value sampled from the `get_design_context` /
`get_screenshot` output.

---

## Rule 2: Figma frame dimensions are border-box — apply `box-sizing: border-box`

Figma frame width/height always **include** padding (outer dimensions). CSS `width`/`height`
default to `content-box`, which adds padding on top and makes the rendered element larger than
the Figma frame. Any AF element whose size is derived from a Figma frame — the form
`.cmp-adaptiveform-container`, a panel acting as a "card" (`.panelcontainer.responsivegrid`), or
a field's `…__widget` — needs `box-sizing: border-box` wherever a Figma-derived width, min-height,
or padding is set:

```css
:root { --af-field-height: 46px; }
.cmp-adaptiveform-textinput__widget,
.cmp-adaptiveform-emailinput__widget,
.cmp-adaptiveform-numberinput__widget {
    box-sizing: border-box;   /* Figma's 46px already includes the field's own padding */
    min-height: var(--af-field-height);
}
```

Skipping this reproduces the exact defect this rule exists to prevent: fields/cards rendering
visibly larger/taller than the Figma frame even though every colour and font value is correct.

---

## Rule 3: Figma auto-layout rows → `aem-Grid` column spans (multi-column fields)

Figma communicates a multi-column field row as a **horizontal auto-layout frame** containing two
or more field frames. `create-adaptive-form` authors that row as a `panelcontainer` whose child
fields carry `<default width="N">` (of 12) — see `create-adaptive-form` → "Match the source
layout". When the source is Figma, derive `N` from the auto-layout, not by eyeballing the
screenshot:

| Figma auto-layout row | Field `width` (of 12) |
|---|---|
| `layoutMode: HORIZONTAL` with 2 equal-width children | `6` each |
| `layoutMode: HORIZONTAL` with 3 equal-width children | `4` each |
| `layoutMode: HORIZONTAL` with 4 equal-width children | `3` each |
| `layoutMode: HORIZONTAL` with unequal children | nearest supported span (`12/6/4/3`) proportional to each child's `width` relative to the frame's total content width; round, don't invent a fractional span |
| `layoutMode: VERTICAL` (stacked) or a single child | `12` (full width) |

The theme/clientlib CSS never re-declares the grid (per `create-form-theme` → "Layout columns are
the platform FLOAT grid") — column widths are authored on the fields themselves, sourced from the
Figma auto-layout structure captured in `get_design_context`.

---

## Rule 4: Figma image/icon assets are temporary — never reference them directly

Image and SVG URLs returned by the Figma MCP tools point at Figma's CDN and **expire in days**.
For a Figma-sourced replica:

1. **Never** hardcode a Figma asset URL in `theme.css`, the DAM theme-json, or any field's default
   value.
2. Every icon/logo/photo visible in the Figma frame becomes a **DAM asset + an authored Adaptive
   Form Image component** (`fileReference`), exactly as `create-adaptive-form` → "Icons & imagery
   from a reference" and `create-form-theme` → "Reproduce icons — as AUTHORED IMAGE COMPONENTS,
   never via theme CSS" already require for any reference-driven build. Figma is not a special
   case here — it is simply another reference source subject to the same rule.
3. If a Figma icon is a simple vector (single path, one or two colours), extract and inline the
   SVG markup instead of downloading the temporary raster; if it's complex, save it once as a DAM
   asset and reference it — do not re-fetch the Figma URL on every render.

---

## Figma-sourced pixel-perfect checklist (in addition to the main Quality checklist)

- [ ] Every `--af-*` token value is traced to a named Figma variable/style (or an exact sampled
      value when the file uses no variables) — recorded in the theme header comment
- [ ] `box-sizing: border-box` is applied to every AF element (container, card panel, field
      widget) whose width/height/padding came from a Figma frame dimension
- [ ] Every Figma horizontal auto-layout field row is reproduced as `panelcontainer` children with
      `<default width="N">` derived from the auto-layout child count/proportions (Rule 3) — not a
      guessed span
- [ ] Zero Figma CDN URLs appear in `theme.css`, the DAM theme-json, or any field default — every
      image/icon is a DAM asset rendered by an authored Image component
- [ ] Rendered form verified VISUALLY against the `mcp__figma__get_screenshot` capture (the
      `reference_for_ui_check`) — not just "CSS present" or "build succeeded"
