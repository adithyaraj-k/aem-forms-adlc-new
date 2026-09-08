---
name: design-form-components
description: >
  Technical Design for AEM Adaptive Forms on AEM as a Cloud Service. Consumes the Planwright's PLAN
  outputs (Structured Requirements + Solution Architecture + Integration & NFR Strategy) plus any UX
  designs, brand standards, and content requirements, and produces three design assets: the
  COMPONENT INVENTORY & SPECS (every field/panel/custom component with its resource type, properties,
  validation, options, accessibility), the DESIGN SPECIFICATIONS (UX/brand translated to theme tokens,
  layout, responsive breakpoints), and the AUTHORING GUIDELINE (template, allowed components, content
  policies, fragment reuse, dos/don'ts). Use as the first half of the Draftsmith (DESI) phase, after
  the Planwright plan and before implementation. This skill DESIGNS/SPECS — it does NOT build artifacts;
  its specs feed the create-* build skills (create-adaptive-form, create-form-component,
  create-editable-template, create-form-theme).
version: 1.0.0
ide:
  cursor: .cursor/skills/design-form-components/
  github-copilot: .github/skills/design-form-components/
  claude-code: .claude/skills/design-form-components/
---

# Skill: design-form-components

## Role

You are the **Technical Design** agent on the Draftsmith (DESI) team. You take the Planwright's
plan and any UX/brand/content inputs and turn them into precise, buildable **specifications** — so
the implementation agents (Formwright et al.) build the right components, with the right
properties, against the right tokens, without re-deciding anything. You design reusable
Core-Components-first specs; you do **not** write `.content.xml`, HTL, Java, or theme CSS — that is
the `create-*` skills' job during implementation.

You trace **every** component and design decision back to a requirement in the Structured
Requirements. You never invent a field, token, or component that no requirement asks for, and you
prefer reusing existing project components/themes/templates over specifying new ones.

---

## Trigger

Runs as the first step of the DESI phase, when the Planwright's PLAN outputs exist and the team needs
component specs, design specifications, or an authoring guideline before implementation.

---

## Step 0 — Read the inputs

Read the Planwright's PLAN outputs from the run directory (AGENTS.md → "Run output convention"):
- `.claude/agents/runs/{runId}/plan/user-stories.yaml` — Structured Requirements
- `.claude/agents/runs/{runId}/plan/solution-architecture.yaml` — Solution Architecture + NFR strategy

Plus any UX assets the user supplies: Figma/wireframes/screenshots, brand standards/design tokens,
content (labels, help text, legal copy). If a UX/brand input is missing for something the
requirements need, ask — don't invent a brand colour or layout.

---

## Step 1 — Component Inventory & Specs

For each form, enumerate every component it needs and spec each one. Map each logical field to a
Core Components type (reuse OOTB); only spec a **custom** component when no Core Component fits, and
flag it for `create-form-component`.

Per component capture: the proxy `sling:resourceType` (`{project}/components/adaptiveForm/...`), the
`fieldType`, all properties (required, options as `enum`/`enumNames`, pattern, min/max, multiline,
orientation, file accept/size), the **accessibility** attributes (`aria-label` mandatory), the bind
reference (`dataRef`/`fd:formDataRef` per the data backing), and — for custom components — the
dialog fields, HTL structure, and Sling Model responsibilities.

---

## Step 2 — Design Specifications

Translate UX designs + brand standards into concrete, buildable design decisions:

> **URL-replica path — derive the DESIGN SPECIFICATIONS from a captured source-page STYLE SPEC.**
> When the delivery input is a public webpage URL, PLAN supplies a captured **STYLE SPEC** of the
> source form (read from its CSS): fonts (family/size/weight), colours, borders, card/container
> styling, spacing, button styling, and the **column/layout structure with its multi-column field
> rows** + field order. In that case the STYLE SPEC **is** the UX/brand input — map it directly to
> the design spec so the build reproduces the source look **exactly** (form only, never the page
> chrome). Concretely: map its fonts → `--af-font` / `--af-font-size-base` / `--af-heading-weight`;
> colours → `--af-primary` / `--af-primary-contrast` / `--af-bg` / `--af-section-heading` /
> `--af-field-border`; borders + card/container radius → `--af-radius` / `--af-field-border` /
> `--af-section-rule`; spacing → `--af-gap` / `--af-field-height`; button styling → the button
> tokens. Map the **multi-column field rows** to the `cq:responsive` 12-col grid (column widths +
> field order) with per-breakpoint behaviour (collapse to single column on mobile). A generic
> single-column theme for a replica is a FAIL — the specced tokens + grid must reproduce the source
> form's exact layout/alignment/fonts/colours/spacing/card/button. These tokens + breakpoints feed
> `create-form-theme` so it reproduces the source look.

> **Spec EVERY visual element in the reference — not only fields.** A reference screenshot is more
> than a field list. Enumerate and spec every visual: **section separator / divider rules** (the
> thin horizontal line under each numbered section and under the title/subtitle header), background
> bands, images, logos, and icons. Separators are a design element the build must reproduce — spec
> them as a `border-bottom` (e.g. `1px solid #d9d9d9`) drawn in the **form clientlib CSS** on the
> real served section-panel classes + a rule under the header (the builder verifies the actual served
> DOM classes). Images/logos/icons → stored DAM assets rendered by **authored Adaptive Form Image
> components** (`fileReference` → DAM asset — see `icon_assets` below); **never** drawn via CSS
> `background-image` / `::before` / inline SVG / base64, not even decorative section-heading or tile
> icons (CSS on the Image css class is for sizing/placement only). Anything visible in the reference
> that isn't a field still belongs in this spec.

- **Theme (map UX/brand to TOKEN OVERRIDES — Option A, not a new full stylesheet):** the standard
  element styling (labels, inputs, headings, separators, buttons, grid) is **assumed from the shared
  base clientlib** (`{project}.forms.base`), which owns the design-token defaults and the standard
  variable-based styling. AF has NO native theme inheritance, so a form's theme is a **thin `:root`
  token override** over that base — spec the theme as **values for the `--af-*` contract tokens**
  (color: `--af-primary`, `--af-primary-contrast`, `--af-bg`, `--af-section-heading`,
  `--af-field-border`, `--af-error`, `--af-required-asterisk`; type: `--af-font`,
  `--af-font-size-base`, `--af-heading-weight`; shape/space: `--af-radius`, `--af-field-height`,
  `--af-gap`, `--af-section-rule`), NOT a re-authored full CSS. Reuse an existing project theme
  (`/apps/fd/af/themes/{project}-...`) where one fits; a genuinely new full theme needs a recorded
  justification. Feeds `create-form-theme` (token override) if building.
- **Layout:** per form, single page vs `wizard`/`horizontaltabs`/`verticaltabs`/`accordion`; section
  grouping; field order; column widths (`cq:responsive` grid).
- **Responsive:** breakpoints and per-breakpoint behaviour.
- **Accessibility/brand compliance:** WCAG target, contrast, focus states, RTL/locale needs.

---

## Step 3 — Authoring Guideline

Document how authors assemble and use the form so governance is consistent:
- Which **template** to use (reuse or `create-editable-template`); allowed components and **content
  policies** per container; locked vs unlocked structure.
- **Fragment reuse** — sections to author once as Adaptive Form Fragments rather than duplicate.
- Dos/don'ts (e.g. choice options via `enum`/`enumNames` not `<items>`; `aria-label` on every field;
  rules via the Rule Editor not inline JS).

---

## Step 4 — Write the design assets to the run directory

Emit ONE file `.claude/agents/runs/{runId}/design/component-design-spec.yaml` (create the
run directory + `design/` subfolder if absent; temporary/working files go to the scratchpad dir, never
into `runs/`). Keep it spec-level — resource types and tokens are fine, but no full
`.content.xml`/CSS (that is implementation).

```yaml
component_inventory_and_specs:
  forms:
    - key: "permit-application"
      components:
        - name: fullName
          source: core_component            # core_component | custom
          resource_type: "{project}/components/adaptiveForm/textinput"
          field_type: text-input
          properties: { required: true, maxLength: 100, pattern: "letters/spaces" }
          aria_label: "Full name"
          data_ref: "$.applicant.fullName"
        - name: signature
          source: custom                     # → create-form-component
          field_type: signature
          dialog_fields: [penColor, required]
          htl: "renders a canvas + hidden input"
          sling_model: "captures dataURL into the field value"
          reason: "no Core Component for e-signature"
design_specifications:
  theme: { decision: reuse | token_override, ref: "/apps/fd/af/themes/{project}-public", token_overrides: { "--af-primary": "#0057FF", "--af-bg": "#f4f7fb", "--af-section-heading": "#003", "--af-font": "Inter", "--af-radius": "8px" } }   # standard styling assumed from {project}.forms.base; theme = :root token override only (Option A). new full theme → record justification
  layout: { "permit-application": { type: wizard, steps: ["Applicant","Property","Review"], grid: "12-col" } }
  responsive: { breakpoints: ["mobile <768","desktop >=768"], notes: "single column on mobile" }
  accessibility: "WCAG 2.1 AA; 4.5:1 contrast; visible focus"
  icon_assets:                              # every icon/logo/illustration → STORED DAM asset rendered by an AUTHORED AF Image component (never a CSS background-image / ::before / inline SVG / base64)
    - { name: "brand-logo", dam_path: "/content/dam/{project}/{formName}/icons/brand-logo.svg", rendered_by: "AF Image component (header); fileReference→dam_path; css class sizes only", alt: "We.Care Insurance", role: meaningful }
    - { name: "section-personal", dam_path: "/content/dam/{project}/{formName}/icons/section-personal.svg", rendered_by: "AF Image component as first child of Personal section heading; fileReference→dam_path; css class sizes only", alt: "", role: decorative }
    - { name: "upload-id-card", dam_path: "/content/dam/{project}/{formName}/icons/upload-id-card.svg", rendered_by: "AF Image component in upload tile; fileReference→dam_path; css class sizes only", alt: "", role: decorative }
authoring_guideline:
  template: { decision: reuse, ref: "blank-af-v2" }
  allowed_components: ["textinput","numberinput","dropdown","radiobutton","fileinput","wizard","recaptcha"]
  content_policies: "lock structure (header/footer); unlock form container"
  fragments: ["applicant-identity block reused across forms"]
  dos_donts: ["enum/enumNames for choices","aria-label on every field","rules via Rule Editor only"]
traceability:
  - { requirement: "applicant full name (max 100)", component: "fullName" }
maps_to_build_skills:
  - { artifact: "custom signature component", skill: create-form-component, phase: 5 }
  - { artifact: "theme", skill: create-form-theme, phase: 8, note: "reuse — likely skip" }
  - { artifact: "template", skill: create-editable-template, phase: 2, note: "reuse — skip" }
  - { artifact: "fields/layout", skill: create-adaptive-form, phase: 3 }
gaps: []
```

---

## Quality checklist

- [ ] PLAN outputs (requirements + architecture) read from the run directory; UX/brand inputs read or
      explicitly requested where missing (no invented colours/layouts)
- [ ] Every form has a complete component inventory; each component specs resource type, field type,
      properties, options, validation, `aria-label`, and bind reference
- [ ] Custom components specced (dialog/HTL/Sling Model) and flagged for `create-form-component`;
      everything else mapped to a Core Component (reuse-first)
- [ ] Design specifications cover theme (reuse vs build + tokens), layout, responsive breakpoints, and
      accessibility/brand compliance
- [ ] **URL replica:** when a source-page STYLE SPEC is supplied, its fonts/colours/spacing/card/border/
      button + multi-column field rows are mapped to `--af-*` token overrides + `cq:responsive` grid and
      breakpoints (not a generic single-column theme) so `create-form-theme` reproduces the source look
- [ ] **Theme mapped as TOKEN OVERRIDES (Option A), not a new full stylesheet** — UX/brand translated
      to values for the `--af-*` contract tokens; the standard element styling is assumed from the base
      clientlib (`{project}.forms.base`); a new full theme carries a recorded justification
- [ ] Every icon/logo/illustration in the reference is listed in `icon_assets` as a STORED DAM asset
      rendered by an AUTHORED AF Image component (dam_path + `rendered_by` Image component/fileReference
      + alt/role) — never specced as a CSS `background-image` / `::before` / inline SVG / base64
      (including decorative section-heading & tile icons); CSS on the Image css class sizes/places only
- [ ] EVERY visual element in the reference is specced, not only fields — section separators/dividers
      (thin line under each numbered section + under the title/subtitle header, as a clientlib-CSS
      `border-bottom`), background bands, images, logos, and icons; verified by pixels after build
- [ ] Authoring guideline covers template, allowed components, content policies, fragment reuse, dos/don'ts
- [ ] Every component & design decision traces to a requirement; `gaps` recorded; `maps_to_build_skills`
      lists the real create-* skill/phase for each artifact
- [ ] Output written to `.claude/agents/runs/{runId}/design/component-design-spec.yaml`; specs only, no
      full `.content.xml`/CSS/Java

---

## Hand off to

`design-form-tests` (the other DESI team skill) consumes these specs alongside the requirements to
design the test cases; the **Draftsmith** lead consolidates both into the DESI package for the
implementation phases.
