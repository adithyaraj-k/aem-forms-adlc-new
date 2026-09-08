---
name: create-form-clientlib
description: >
  Generates a reusable AEM client library (cq:ClientLibraryFolder) for an Adaptive
  Form on AEM as a Cloud Service Core Components. Produces the clientlib node, css.txt
  / js.txt manifests, CSS assets, and JavaScript for Rule-Editor custom functions and
  form-level / field-level client-side validation. Wires the clientlib to a form
  declaratively via clientLibRef on the guideContainer (no edits to the page component).
  Use when a form needs custom client-side behaviour, validation, or shared styling
  beyond what a theme (create-form-theme) or a single custom field (create-form-component)
  provides.
version: 1.2.0
ide:
  cursor: .cursor/skills/create-form-clientlib/
  github-copilot: .github/skills/create-form-clientlib/
  claude-code: .claude/skills/create-form-clientlib/
  vscode: .vscode/skills/create-form-clientlib/
---

# Skill: create-form-clientlib

## Role

You are an expert AEM Adaptive Forms front-end engineer for AEM as a Cloud Service.
You build **client libraries** for Adaptive Form (Core Components) pages: Rule-Editor
custom functions, client-side validation, and form-scoped CSS. You follow Granite
clientlib conventions (categories, dependencies, embed, `allowProxy`, `js.txt`/`css.txt`
with `#base=`), and you attach clientlibs to forms the supported way — never by editing
`/libs` or the project page component.

Before generating, read `.aem-forms-config.yaml` for `project`, `package`, and
`formsContentRoot`. If missing, ask the developer to confirm. This project's values:
`project = aem-demo-site`, clientlibs root
`/apps/{project}/clientlibs/`.

This skill is **complementary**, not overlapping:
- `create-form-theme` → branding/visual tokens (theme clientlib under `/apps/fd/af/themes`).
- `create-form-component` → clientlib for a single custom field component.
- **`create-form-clientlib`** (this skill) → a **form-level** functional clientlib:
  custom functions + validation logic + shared CSS, attached to a whole form.

---

## Base clientlib vs form-specific clientlib (reuse-first — decide BEFORE writing any function)

Clientlibs follow a **reuse-first split, exactly like templates**: a shared base for generic
logic, a thin per-form clientlib for what is unique to one form. Do this before authoring any
function or CSS rule.

- **ONE shared BASE forms clientlib holds GENERIC scripts/CSS reusable across most forms** —
  common validators (email/phone/date-range), formatters, common Rule-Editor custom functions,
  and shared form CSS. It is created **once and reused by every form** — the single source of
  truth for generic logic. Home it under `/apps/{project}/clientlibs` with a **forms base
  category** (e.g. `{project}.forms.base`); **extend the existing
  `clientlib-base` area rather than creating parallel bases.** (This project already has
  `/apps/{project}/clientlibs/clientlib-base`, category
  `{project}.base`; the forms-scoped base category is the single home for
  generic FORM scripts.)
- **The BASE clientlib is also the theme BASE LAYER (Option A design tokens).** Because AF has NO
  native theme-extends-theme inheritance, theme inheritance is done with CSS custom properties +
  the cascade. So the base clientlib CSS **declares ALL design tokens with default values in
  `:root`** (the theme API — see the token contract below) **AND writes the STANDARD styling for
  common elements CONSUMING those tokens** (labels, text/number inputs, dropdowns, radios/checkboxes,
  textarea, `.cmp-title__text`, section headings `.cmp-container__label`, section separators, the
  required asterisk `[data-cmp-required] …__label::after`, buttons, the served grid, **the native date
  field (kept selectable)**, **and the validation-failure state (red message + red border)**). This
  standard styling is authored **ONCE here**, not per form.
  > **Date picker — keep it NATIVE and selectable.** The AF datepicker is a native `<input type="date">`.
  > **NEVER set `appearance: none` / `-webkit-appearance: none` on `.cmp-adaptiveform-datepicker__widget`**
  > — it strips the native calendar in Chromium so the field looks like plain text and won't open the
  > picker (a real regression). Just clamp the widget width (shared widget rule) and, at most, style the
  > native indicator lightly: `.cmp-adaptiveform-datepicker__widget::-webkit-calendar-picker-indicator { cursor: pointer; opacity: 1; }`.
  > Do NOT hide or absolutely-position the indicator.
  > **DATE UNCLICKABLE fix (Chrome "can't pick a date").** The native date input's intrinsic min-width
  > can overflow a narrow (width-4) grid column so an ADJACENT column overlaps it and intercepts the
  > click. Clamp the width AND lift the date column into its own stacking context:
  > `.aem-GridColumn:has(.cmp-adaptiveform-datepicker){position:relative;z-index:1}` +
  > `.cmp-adaptiveform-datepicker{position:relative}`. See [[datepicker-icon-pinned-in-field]].
  > **Validation-failure state (red).** On failure the AF runtime sets `data-cmp-valid="false"` on the
  > field wrapper and `aria-invalid="true"` on the widget. Style BOTH so the message and border go red:
  > ```css
  > .cmp-adaptiveform-<type>__errormessage { color: var(--af-error); font-size: 12px; margin-top: 4px; }
  > [data-cmp-valid="false"] .cmp-adaptiveform-<type>__widget,
  > .cmp-adaptiveform-<type>__widget[aria-invalid="true"] { border-color: var(--af-error) !important; }
  > ```
  > Enumerate every field type + textarea. See [[validation-failure-styling-red]]. A **form-specific theme then overrides only the
  `:root` tokens** (see `create-form-theme` → "Base theme via CSS-variable design tokens (Option A)")
  — it does NOT re-author the standard element styling. **The token contract** (the only knobs a
  theme touches): color `--af-primary`, `--af-primary-contrast`, `--af-bg`, `--af-section-heading`,
  `--af-field-border`, `--af-error`, `--af-required-asterisk`; type `--af-font`,
  `--af-font-size-base`, `--af-heading-weight`; shape/space `--af-radius`, `--af-field-height`,
  `--af-gap`, `--af-section-rule`. The base clientlib loads BEFORE the theme `<link>`, so a theme's
  equal-specificity `:root` override naturally wins — keep base tokens at plain `:root` specificity.
- **The base is a CANONICAL REFERENCE LIBRARY, but per-form clientlibs are SELF-CONTAINED —
  they do NOT reference the base.** The base holds the canonical copy of each generic validator +
  the standard styling so there is one place to read/update them. But a per-form clientlib must
  **NOT** `embed` or `depend on` `{project}.forms.base` (see the hard rule below).
  Instead, each form-specific clientlib **copies the subset it needs** (the referenced `@name`
  functions + the base styling) into its own files, so it stands alone.
- 🩺 **FIRST, if the Rule Editor shows custom-function rules "Broken" / the customfunctions
  endpoint returns `{"customFunction":[]}`, the cause is USUALLY NOT the clientlib.** The endpoint
  builds the form MODEL to extract functions; the model build needs the form's data model. For a
  **schema-backed** form the #1 cause is a JSON SCHEMA that fails AEM's importer (valid JSON can
  still fail). Check `crx-quickstart/logs/error.log` for
  `GuideModelImporterImpl Unable to parse JSON Schema` →
  `AdaptiveFormCustomFunctionProviderServlet Error getting form model for function extraction`.
  If you see that, FIX THE SCHEMA (see `generate-schema` → Hard rule 7), not the clientlib.
  (Learned the hard way on sports-event-registration-form, 2026-07-06: rewriting the clientlib
  to single-category/self-contained did NOT change the empty result — the schema was the cause.)
- **Prefer a SINGLE-category, self-contained form clientlib (clean, proven-working shape).** The
  working single-clientlib forms (business-registration, kyc-form, lab-test-form,
  life-insurance-new-policy) each use ONE category with their own self-contained `functions.js`
  + css and `dependencies=[core.forms.components.runtime.all]`. Recommended:
  `clientLibRef="{project}.forms.{formName}"` (single category) with the clientlib
  self-contained. NOTE: a comma-separated multi-value `clientLibRef` and embedding `forms.base`
  were previously suspected of breaking the endpoint — that was a **misdiagnosis** (the real cause
  was the schema, above). The single-category self-contained shape is still preferred for clarity
  and to avoid duplicate `@name` definitions in the bundle, but do not expect it to "fix" a
  Broken-rules symptom whose real cause is the model/schema.
- **Avoid duplicate `@name` definitions in one bundle anyway (hygiene).** If a form clientlib both
  copies a function AND embeds `forms.base` (which also defines it), the built JS defines it twice
  (`grep -c "function validateName"` = 2). Even if this did not cause the endpoint failure here,
  keep each `@name` defined once. You MAY depend on libs with **NO `@name` functions** —
  `core.forms.components.runtime.all`, `{project}.forms.generate-pdf`.
- **The form-specific clientlib is SELF-CONTAINED (functions + styling):**
  - `js/functions.js` physically declares — **exactly once** — the `@name` custom functions the
    form's rules reference (seeded as a one-time copy from the base; the base is the reference for
    what to copy, but is **not** wired in). GLOBAL-scope, `js.txt` manifest.
  - `css/` carries the standard styling the form needs. Copy the base styling
    (`clientlib-forms-base/css/base.css`) into the form clientlib's `css/` (e.g. `base.css`) and
    list it in `css.txt` **before** the form-specific CSS. Do **NOT** reference `forms.base` to
    obtain the styling.
  - `.content.xml` `dependencies="[core.forms.components.runtime.all]"` (+
    `,{project}.forms.generate-pdf` when the form generates a PDF on submit).
    **No `embed`. No `forms.base`.**
  This is the proven-working shape: every working single-clientlib form (business-registration,
  kyc-form, lab-test-form, life-insurance-new-policy) has its own single category + self-contained
  `functions.js` + css, dependencies on core runtime only, and does **not** reference `forms.base`.
- **A form that needs no custom functions of its own** still uses its own single self-contained
  form-specific category (with the copied base styling); do **NOT** point `clientLibRef` at the
  bare `...forms.base` category, and do **NOT** use a comma list.
- `functions.js` stays **GLOBAL-scope (no IIFE)** with a `js.txt` manifest, so the Rule Editor
  discovers the functions.

> **Why self-contained (hygiene, not a bug fix):** matching the proven working forms
> (business-registration et al.) — one category, own `functions.js` + css, dependencies on core
> runtime (+ generate-pdf) — keeps each `@name` defined once and avoids ambiguity. Reuse of the
> base is by **one-time copy of the needed subset**, not by wiring the base in. This is a clarity
> choice; it is NOT the fix for a "Broken rules" symptom — that is almost always the form
> model/schema failing to import (see the 🩺 diagnostic bullet above).

---

## Trigger

Activate when the developer asks to:
- Create / generate a client library (clientlib) for an Adaptive Form
- Add custom JavaScript functions usable in the Rule Editor
- Add client-side form-level or field-level validation in JavaScript
- Add functions wired to **any** rule event — Validate, Value Commit (set value on change),
  Initialize (set value on load), Calculate, or Click — not just Validate
- Add form-scoped CSS/JS that isn't a theme and isn't a single custom component

---

## How clientlibs attach to an Adaptive Form in this project (CRITICAL)

> A form references a functional clientlib by **category**, set on its
> `guideContainer`. The project's AF page component
> (`{project}/components/adaptiveForm/page`) already contains
> `customheaderlibs.html` and `customfooterlibs.html` that read
> `FormStructureParser.getClientLibRefFromFormContainer()` and render:
> - **CSS** in the head: `clientlib.css @ categories=<clientLibRef>`
> - **JS** in the footer (async): `clientlib.js @ categories=<clientLibRef>`
>
> So attaching a clientlib is **declarative**: add `clientLibRef="<category>"` to the
> form's `guideContainer` node. **Do NOT edit `customheaderlibs.html`/`customfooterlibs.html`
> or anything under `/libs`.** Verified against this repo's page component.

`clientLibRef` holds a **clientlib category string** (e.g.
`{project}.forms.employee-registration`), **not** a JCR path. (The
path-style `clientlibRef` seen on legacy theme definitions under
`/content/dam/formsanddocuments-themes` is a different, theme-only property — don't
confuse the two.)

> ⚠️ `clientLibRef` MUST hold a **SINGLE category**, not a comma-separated list. Although the
> page runtime *would* render every category in a comma list, the Rule-Editor custom-functions
> endpoint resolves `clientLibRef` as one category name — a comma-separated value returns an
> empty function list and marks every custom-function rule "Broken." Make the form-specific
> clientlib **self-contained** (its own copied functions + styling) rather than pulling in the
> base: `clientLibRef="{project}.forms.{formName}"` with the clientlib's
> `dependencies="[core.forms.components.runtime.all,{project}.forms.generate-pdf]"`.
> 🛑 Never `embed`/`depend on` `forms.base` (or any lib that redefines the same `@name` functions) —
> duplicate `@name` definitions make the endpoint return empty (all rules "Broken").

---

## Confirmation step (always do this first)

Echo back and wait for confirmation:

```
Clientlib name : clientlib-{name}        (folder under .../clientlibs/)
Category       : {project}.forms.{name}
Attach to form : {form path under /content/forms/af, or "none — shared library"}
CSS            : {yes — describe | no}
JS             : {custom functions? form-level validation? field-level validation?}
Dependencies   : [core.forms.components.runtime.all]   (+ others if needed)
Embed          : {other categories to bundle, or none}
```

---

## Files to generate

```
ui.apps/src/main/content/jcr_root/apps/{project}/clientlibs/clientlib-{name}/
├── .content.xml          ← cq:ClientLibraryFolder (category, deps, allowProxy)
├── js.txt                ← JS load order (#base=js)
├── css.txt               ← CSS load order (#base=css)   [omit if no CSS]
├── js/
│   ├── functions.js      ← Rule-Editor custom functions (JSDoc-annotated)
│   └── validations.js    ← form-level / field-level validation logic
└── css/
    └── styles.css        ← form-scoped styles            [omit if no CSS]
```

The `ui.apps` filter already covers `/apps/{project}/clientlibs` — no
filter.xml change is needed when generating under that root. If you place a clientlib
elsewhere, add a covering filter root.

---

## `.content.xml` — the clientlib node

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0"
          jcr:primaryType="cq:ClientLibraryFolder"
          allowProxy="{Boolean}true"
          categories="[{project}.forms.{name}]"
          dependencies="[core.forms.components.runtime.all]"/>
```

Rules:
- **`categories`** — array, unique. Convention: `{project}.forms.{name}`.
  This is the exact string you put in the form's `clientLibRef`.
- **`allowProxy="{Boolean}true"`** — always, so assets serve via
  `/etc.clientlibs/...` (AEMaaCS never exposes `/apps` to the browser).
- **`dependencies`** — `core.forms.components.runtime.all` makes the AF Core Components
  runtime (the field/form model API) load before your JS. Add more deps as needed.
- **`embed`** — bundles another clientlib's content INTO this one category. 🛑 **Never `embed`
  (or `depend on`) a clientlib that defines the same `@name` custom functions as this one** (e.g.
  `{project}.forms.base`): the merged bundle then defines each function twice and
  the customfunctions endpoint returns empty (all rules "Broken"). Only embed/depend libs with NO
  `@name` custom functions (e.g. `...forms.generate-pdf`). For generic validators, **copy** the
  needed subset into this clientlib's own `functions.js` (once) instead of embedding the base.
- Do **not** set `categories` to a name that collides with a theme or component library.

### `js.txt`

```
#base=js
functions.js
validations.js
```

### `css.txt`

```
#base=css
styles.css
```

`#base=` points at the subfolder holding the listed files; order in the file is the
load order. Keep JS and CSS in separate subfolders to stay readable as the library grows.

---

## Client-side validation — two supported layers

Choose based on whether authors should see/use the logic in the Rule Editor.

### Layer 1 (preferred, DEFAULT) — Rule-Editor custom functions + auto-wired rules

JSDoc-annotated functions in a referenced clientlib become **Custom Functions** in the
Rule Editor (listed under *Function Output*). This keeps logic discoverable, testable,
and surviving form edits. **This is the default deliverable for a "field validation"
clientlib:** emit one validator per field in `functions.js`, then — in THIS skill, not by
delegating — author the `Validate` / `Set Value` rules on the fields that call them (see
"Wiring functions into rules" below).

> ⚠️ **Functions MUST be declared at global scope — no IIFE, no namespace object.** The
> Rule Editor scanner only discovers **top-level `function` declarations**. If you wrap
> them in `(function(){ … })()` or hang them off `window.MyNs = {…}`, they still run but
> **will not appear in the Rule Editor** and rules cannot call them. (This is the #1
> mistake — a clientlib whose functions are invisible to authors.) Expose helpers only as
> plain globals; prefix internal-only helpers (e.g. `enrollmentParseDate`) and mark them
> `@private` so they are skipped by the scanner.

> **Make every validator empty-safe.** A `Validate` rule fires even when the field is
> empty; let the field's own `required` flag own emptiness. So each validator should
> `return true` for `null` / `undefined` / `""`. This also fails safe if a value token
> doesn't resolve — the field stays valid (submittable) instead of being stuck invalid.

End `functions.js` with a **Rule Editor usage notes** comment block mapping each form
field to the function and rule type (Validate / Set Value) an author should pick — it
documents the wiring you generate and guides anyone editing rules later.

```js
// js/functions.js

/**
 * Validate that a value is a well-formed work email (no free webmail domains).
 * @name validateWorkEmail Validate work email
 * @param {string} email the email address to check
 * @return {boolean} true when the email is a valid corporate address
 */
function validateWorkEmail(email) {
    if (!email) return false;
    var re = /^[^@\s]+@[^@\s]+\.[^@\s]+$/;
    if (!re.test(email)) return false;
    var banned = ["gmail.com", "yahoo.com", "hotmail.com", "outlook.com"];
    var domain = email.split("@")[1].toLowerCase();
    return banned.indexOf(domain) === -1;
}

/**
 * Cross-field check: end date must be on/after start date.
 * @name dateRangeValid Validate date range
 * @param {string} startDate ISO date string
 * @param {string} endDate ISO date string
 * @return {boolean} true when endDate >= startDate
 */
function dateRangeValid(startDate, endDate) {
    if (!startDate || !endDate) return true; // let "required" handle emptiness
    return new Date(endDate).getTime() >= new Date(startDate).getTime();
}
```

JSDoc rules that make a function appear in the Rule Editor:
- A description line, then `@name <functionId> <Display Label>`.
- One `@param {type} name desc` per argument; one `@return {type} desc`.
- Supported `{type}`: `string`, `number`, `boolean`, `date`, `array`, `object`.
- To access the form model inside a function, add a final param documented as
  `@param {scope} globals` — the runtime injects the globals object (read
  `globals.form`, call `globals.functions.setProperty(...)`, etc.). Only declare it
  when needed.

#### Wiring functions into validation (generate these in THIS skill)

Set `clientLibRef` to this library's category on the `guideContainer`, then wire each
target field with BOTH parts below. **Part A makes validation run; Part B makes the rule
VISIBLE in the Rule Editor.** Generating only A (the trap I hit) runs the validation but
shows **no rule** in the editor — authors can't see or edit it.

**Part A — runtime (two plain field attributes):**
- **`validationExpression`** — a json-formula expression the runtime evaluates; returns
  boolean (`true` = valid). Compiles to a top-level `validationExpression` on the field in
  `guideContainer.model.json`. Plain attribute (NOT escaped JSON, NOT a `[...]` multi-value),
  so commas need NO escaping: `validationExpression="isValidName($field.$value, 50) == true()"`.
- **`validateExpMessage`** — message shown when it returns false → `constraintMessages.validationExpression`.

**Part B — Rule-Editor visibility (an `fd:rules` child + empty `fd:events`):**
The editor renders rules from the authored model stored in `fd:rules/@fd:validate` (escaped
JSON). An empty `items:[]` AST stores but **does NOT show** — you need the real node tree.
This is the VERIFIED shape (round-tripped against an editor-authored reference form):

```xml
<AadhaarNumber jcr:primaryType="nt:unstructured" jcr:title="Aadhaar Number (or National ID)"
    sling:resourceType="{project}/components/adaptiveForm/textinput"
    fieldType="text-input" name="AadhaarNumber" required="true" maxLength="12"
    mandatoryMessage="Please enter the 12-digit Aadhaar / National ID number."
    validationExpression="isValidAadhaar($field.$value) == true()"
    validateExpMessage="Enter exactly 12 digits (numbers only)."
    enabled="{Boolean}true" readOnly="{Boolean}false" visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
    </cq:responsive>
    <fd:rules
        fd:validate="[{&quot;nodeName&quot;:&quot;ROOT&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;STATEMENT&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;VALIDATE_EXPRESSION&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;AFCOMPONENT&quot;\,&quot;value&quot;:{&quot;id&quot;:&quot;$form.IdentificationPanel.AadhaarNumber&quot;\,&quot;type&quot;:&quot;AFCOMPONENT&quot;\,&quot;name&quot;:&quot;AadhaarNumber&quot;}}\,{&quot;nodeName&quot;:&quot;Using&quot;\,&quot;value&quot;:null}\,{&quot;nodeName&quot;:&quot;Expression&quot;\,&quot;value&quot;:null}\,{&quot;nodeName&quot;:&quot;CONDITION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;COMPARISON_EXPRESSION&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;EXPRESSION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;FUNCTION_CALL&quot;\,&quot;parentNodeName&quot;:&quot;EXPRESSION&quot;\,&quot;functionName&quot;:{&quot;id&quot;:&quot;isValidAadhaar&quot;\,&quot;displayName&quot;:&quot;Validate Aadhaar / National ID&quot;\,&quot;type&quot;:&quot;BOOLEAN&quot;\,&quot;isDuplicate&quot;:false\,&quot;displayPath&quot;:&quot;&quot;\,&quot;args&quot;:[{&quot;type&quot;:&quot;STRING&quot;\,&quot;name&quot;:&quot;value&quot;\,&quot;description&quot;:&quot;value to validate&quot;\,&quot;isMandatory&quot;:true}]\,&quot;impl&quot;:&quot;$0($1)&quot;}\,&quot;params&quot;:[{&quot;nodeName&quot;:&quot;EXPRESSION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;COMPONENT&quot;\,&quot;value&quot;:{&quot;id&quot;:&quot;$form.IdentificationPanel.AadhaarNumber&quot;\,&quot;displayName&quot;:&quot;Aadhaar Number (or National ID)&quot;\,&quot;type&quot;:&quot;STRING&quot;\,&quot;displayPath&quot;:&quot;FORM/Identification &amp; Demographics/Aadhaar Number (or National ID)/&quot;\,&quot;name&quot;:&quot;AadhaarNumber&quot;\,&quot;parent&quot;:&quot;$form.IdentificationPanel&quot;\,&quot;metadata&quot;:{&quot;isAncestorRepeatable&quot;:false}}}}]}}\,{&quot;nodeName&quot;:&quot;OPERATOR&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;EQUALS_TO&quot;\,&quot;value&quot;:null}}\,{&quot;nodeName&quot;:&quot;EXPRESSION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;BOOLEAN_LITERAL&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;True&quot;\,&quot;value&quot;:null}}}]}\,&quot;nested&quot;:false}]}}]\,&quot;isValid&quot;:true\,&quot;enabled&quot;:true\,&quot;version&quot;:1\,&quot;script&quot;:&quot;isValidAadhaar($field.$value) == true()&quot;\,&quot;eventName&quot;:&quot;Validate&quot;\,&quot;ruleType&quot;:&quot;&quot;\,&quot;description&quot;:&quot;&quot;\,&quot;communicationComposerRuleType&quot;:&quot;server&quot;\,&quot;communicationComposerRuleLanguageType&quot;:&quot;javascript&quot;}]"
        jcr:primaryType="nt:unstructured" validationStatus="valid"/>
    <fd:events jcr:primaryType="nt:unstructured"/>
</AadhaarNumber>
```

The authored AST anatomy (one `STATEMENT` → `VALIDATE_EXPRESSION`):
- `AFCOMPONENT` = the field being validated (`id` = `$form.<Panel>.<Field>`).
- `CONDITION.choice` = a `COMPARISON_EXPRESSION` of **3 items**: the `FUNCTION_CALL`
  expression, an `OPERATOR` (`EQUALS_TO`), and a `BOOLEAN_LITERAL` `True`. (Conditions are
  always comparisons — wrap a boolean function as `<call> == true()`.)
- ⚠️ **json-formula booleans are FUNCTIONS: `true()` / `false()`, not bare `true`/`false`.**
  A bare `true` resolves to an undefined path, so `<call> == true` is `boolean == undefined`
  → **false for every input** (valid input wrongly fails). The `BOOLEAN_LITERAL True` node
  compiles to `true()`, so the `script` / `validationExpression` MUST read `== true()`.
  (Same for numbers — bare `50` — and strings — `'null'`.)
- `FUNCTION_CALL.functionName` mirrors the JSDoc: `id` = the `@name` id, `displayName` =
  its label, `type` `BOOLEAN`, one `args` entry per parameter, `impl` = `"$0($1, $2, …)"`.
- `FUNCTION_CALL.params` = one `EXPRESSION` per argument: a `COMPONENT` (field value →
  `$field.$value` in `script`), a `NUMERIC_LITERAL` (e.g. `50`), or another `COMPONENT`
  for a cross-field arg (`$form.<Panel>.<Field>.$value`).
- Top-level `script` = the exact runtime expression string (matches `validationExpression`),
  plus `eventName:"Validate"`, `version:1`, `isValid:true`, `enabled:true`,
  `communicationComposerRuleType:"server"`, `communicationComposerRuleLanguageType:"javascript"`.
- `fd:events` is **empty** for a validate rule (runtime comes from `validationExpression`,
  not a compiled event).

> **Escaping (a JCR multi-value `[...]` attribute):** inside `fd:validate`, every JSON `"`
> → `&quot;`, every `,` → `\,`, every `&` → `&amp;`, `<`/`>` → `&lt;`/`&gt;`. Because the
> escaping is mechanical and one slip breaks the whole rule, **generate these with a throwaway
> script rather than hand-typing them** — build the AST as a JS object, `JSON.stringify` it,
> then apply the escapes (`.replace(/&/g,'&amp;').replace(/"/g,'&quot;').replace(/,/g,'\\,')`,
> plus `<`/`>`), and emit the field XML. The script is a build-time helper, not a repo
> artifact — delete it once the `.content.xml` is generated.

#### Beyond Validate — Initialize, Value Commit, Calculate, Click (generate these too)

The Validate recipe above is one event. The Rule Editor calls functions on **other events**
too, and this skill generates them with the same two-part structure — but the runtime half
**inverts**. Read this before emitting any non-validate rule.

> ⚠️ **The runtime source flips for non-validate events.** For **Validate**, the runtime comes
> from the `validationExpression` attribute and `fd:events` stays **empty**. For **every other
> event (Initialize / Value Commit / Calculate / Click), the runtime comes from a POPULATED
> `fd:events`** — the `fd:rules` AST alone is editor-only and does **not** execute. An empty
> `fd:events` on a set-value rule = the field updates in the editor but **nothing happens on the
> rendered form** (verified). So: populate `fd:events`, then confirm the compiled event is
> present in `guideContainer.model.json`.

Each event uses the same `fd:rules` node with a different `fd:<event>` property (the authored
AST, for editor visibility) plus a matching key on `fd:events` (the runtime). Use these exact
property names — do **not** invent json-model names like `fd:change` / `fd:initialize`:

| Rule event | `fd:rules` property (AST) | AST `eventName` label | `fd:events` runtime key | Function returns |
|---|---|---|---|---|
| Validate | `fd:validate` | `"Validate"` | *(empty — runtime via `validationExpression`)* | `boolean` |
| Value Commit (set value on change) | `fd:valueCommit` | `"Value Commit"` | `change` | the new value (`string`/`number`) |
| Initialize (set value on load) | `fd:init` | `"Initialize"` | `initialize` | the value (`string`/`number`) |
| Calculate | `fd:calculate` | `"Calculate"` | `calculate` | the computed value |
| Click (button) | `fd:click` | `"Click"` | `click` | *(action, e.g. `submitForm()`)* |

**The verified `fd:events` runtime forms** (these are what actually fire; commas inside the
`[...]` multi-value MUST be escaped `\,` or FileVault splits the expression into a broken array):

- **Value Commit / change** — guarded so the handler doesn't re-trigger on its own set:
  ```
  change="[if(contains($event.payload.changes[].propertyName\, 'value')\, {value : myFn($field.$value)}\, {})]"
  ```
- **Initialize:**
  ```
  initialize="[{value : myFn()}]"
  ```
- **Click (submit button):**
  ```
  click="[submitForm()]"
  ```

> ⚠️ **A value-event (Initialize / Value Commit / Calculate) can only SET A VALUE — it cannot
> "run" a side-effect function (verified by a crash).** The runtime assigns the event
> expression's result to the component's `value`. So the expression MUST be the set-value form
> `[{value : myFn(...)}]`. A bare call `initialize="[myFn()]"` makes the runtime do
> `component.value = myFn()` — and on a panel/container (whose `value` is getter-only) that throws
> `Uncaught TypeError: Cannot set property value ... which has only a getter` at
> `createFormInstance`, which **fails the whole form load** (blank form). Only the **Click action
> event** runs a bare function call (`click="[submitForm()]"`). For DOM-level work that has no
> return value to assign — autofocus, input masking, strength meters, prefill, attaching
> listeners — there is **no rule**: hook the form's `AF_FormContainerInitialised` event in the
> clientlib (Layer 2) and call your global function from there. (And because FileVault update-mode
> won't delete a rule you removed from source, a bad value-event rule lingers on the server and
> keeps crashing until you delete the form node and redeploy — see "Testing & deploy gotchas".)

> ⚠️ **Set-value rules MUST call a SINGLE-ARGUMENT function (verified).** A multi-argument call
> in a set-value/calculate AST (e.g. `addToValue($field.$value, 2)` with a `NUMERIC_LITERAL`
> second param) **silently fails to compile** — the field gets no `change`/`initialize` event in
> the model and the rule never runs. (Validate tolerates multi-arg; set-value does not.) **Bake
> constants into the clientlib function** so the rule is one argument: write `addTwoToValue(value)`
> returning `value + 2`, called as `addTwoToValue($field.$value)`.

> ⚠️ **A `change` handler that sets its OWN field loops unless idempotent.** The Core Components
> `change` event fires on EVERY value change, including the rule's own set.
> - **Idempotent self-set is safe** (e.g. upper-casing an already-upper value converges in one
>   cycle) — use the plain `contains(... 'value' ...)` guard above.
> - **Non-idempotent self-set loops** (e.g. "+2" runs up to AEM's recalc cap, ~9× → value jumps
>   ~18). Break the loop by skipping the rule's own echo (the change whose delta equals what the
>   rule adds):
>   `change="[if(($field.$value - $event.payload.changes[0].prevValue) != 2\, {value : addTwoToValue($field.$value)}\, {})]"`.
>   For stricter needs, apply the transform at submit time via `create-submit-action` instead.

**Set-value function shape** — unlike a validator (returns `boolean`), a set-value/calculate
function returns the **new value** and is empty-safe by passing the value through:

```js
/**
 * Add two to a numeric value (constant baked in so the rule is a single-arg call).
 * @name addTwoToValue Add two to value
 * @param {number} value the current field value
 * @return {number} the value plus two (unchanged when empty)
 */
function addTwoToValue(value) {
    if (value === null || value === undefined || value === "") { return value; }
    return Number(value) + 2;
}
```

**The Click AST** (verified, byte-for-byte, against an editor-authored submit button — this is
the `EVENT_SCRIPTS → SUBMIT_FORM` tree, NOT a plain `fd:click="submitForm()"` string; the Rule
Editor parses every `fd:rules` attribute as JSON, so a bare string breaks the whole rule parse):

```xml
<submitButton jcr:primaryType="nt:unstructured" jcr:title="Submit"
    sling:resourceType="{project}/components/adaptiveForm/actions/submit"
    buttonType="submit" fieldType="button" name="submitButton"
    enabled="{Boolean}true" visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" behavior="newline" offset="0" width="12"/>
    </cq:responsive>
    <fd:rules
        fd:click="[{&quot;nodeName&quot;:&quot;ROOT&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;STATEMENT&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;EVENT_SCRIPTS&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;EVENT_CONDITION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;EVENT_AND_COMPARISON&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;COMPONENT&quot;\,&quot;value&quot;:{&quot;id&quot;:&quot;$form.submitButton&quot;\,&quot;type&quot;:&quot;BUTTON&quot;\,&quot;name&quot;:&quot;submitButton&quot;}}\,{&quot;nodeName&quot;:&quot;EVENT_AND_COMPARISON_OPERATOR&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;is clicked&quot;\,&quot;value&quot;:null}}\,{&quot;nodeName&quot;:&quot;PRIMITIVE_EXPRESSION&quot;\,&quot;choice&quot;:null}]}\,&quot;nested&quot;:false}\,{&quot;nodeName&quot;:&quot;Then&quot;\,&quot;value&quot;:null}\,{&quot;nodeName&quot;:&quot;BLOCK_STATEMENTS&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;BLOCK_STATEMENT&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;SUBMIT_FORM&quot;\,&quot;items&quot;:[]}}]}]}}]\,&quot;isValid&quot;:true\,&quot;enabled&quot;:true\,&quot;version&quot;:1\,&quot;script&quot;:[&quot;submitForm()&quot;]\,&quot;eventName&quot;:&quot;Click&quot;\,&quot;ruleType&quot;:&quot;&quot;\,&quot;description&quot;:&quot;&quot;}]"
        jcr:primaryType="nt:unstructured" validationStatus="valid"/>
    <fd:events jcr:primaryType="nt:unstructured" click="[submitForm()]"/>
</submitButton>
```

**The set-value AST** (`fd:valueCommit` / `fd:init`) is **NOT** the Validate shape — it is an
`EVENT_SCRIPTS` tree (the same family as the Click rule), and getting it wrong makes the editor
render the rule as **"Unknown Field - null"** (runtime still works from `fd:events`, but the rule
is uneditable). This is the VERIFIED shape, round-tripped from an editor-authored reference:

```
ROOT → STATEMENT → choice:
  EVENT_SCRIPTS, items:[
    { EVENT_CONDITION → choice: EVENT_AND_COMPARISON, items:[
        { COMPONENT, value:{ id:"$form.<Panel>.<Field>",
            type:"TEXT FIELD|FIELD|AFCOMPONENT|STRING",   // pipe-delimited! NUMBER → "NUMBER|FIELD|AFCOMPONENT|NUMBER"
            name:"<Field>" } },
        { EVENT_AND_COMPARISON_OPERATOR, choice:{ nodeName:"is changed", value:null } },  // Initialize → "is initialized"
        { PRIMITIVE_EXPRESSION, choice:{ nodeName:"STRING_LITERAL", value:null } }
      ], nested:false },
    { Then, value:null },
    { BLOCK_STATEMENTS, items:[ { BLOCK_STATEMENT, choice:{
        SET_VALUE_STATEMENT, items:[
          { VALUE_FIELD, value:{ id, displayName, type:"STRING"|"NUMBER", isDuplicate,
              displayPath:"FORM/<PanelTitle>/<FieldTitle>/", name, parent:"$form.<Panel>" } },  // NOT "AFCOMPONENT"
          { to, value:null },                                                                    // lowercase "to"
          { EXPRESSION, choice:{ FUNCTION_CALL, parentNodeName:"EXPRESSION",
              functionName:{ id, displayName, type, isDuplicate:false, displayPath:"", args:[…], impl:"$0($1)" },
              params:[ { EXPRESSION, choice:{ COMPONENT, value:{ …same as VALUE_FIELD… } } } ] } }  // [] for a no-arg init fn
        ] } } ] }
  ]
```
Top-level (siblings of `nodeName:"ROOT"`): `isValid:true`, `enabled:true`, `version:1`,
**`script` is an ARRAY** (`["myFn($field.$value)"]`, not a string), `eventName:"Value Commit"` /
`"Initialize"`, `ruleType:""`, `description:""`. ⚠️ **Do NOT add `communicationComposerRuleType` /
`communicationComposerRuleLanguageType`** here (those belong only to the Validate AST).

Differences from Validate that bite (each one alone yields "Unknown Field - null"): root statement is
`EVENT_SCRIPTS` (not `VALIDATE_EXPRESSION`); the target is a `VALUE_FIELD` with full metadata (not a
bare `AFCOMPONENT`); the `EVENT_CONDITION.COMPONENT.type` is the pipe-delimited
`"TEXT FIELD|FIELD|AFCOMPONENT|STRING"` form; `script` is an array. Build it as a JS object,
`JSON.stringify`, apply the `&`→`&amp;`, `"`→`&quot;`, `<`/`>`→`&lt;`/`&gt;`, `,`→`\,` escapes with a
throwaway script, and pair it with the populated `fd:events` runtime key above. The fastest way to
get a correct one is to author a Set Value rule in the editor on the running instance and read it
back from `…/jcr:content/guideContainer/<…>/fd:rules/@fd:valueCommit`.

> **Verify every non-validate rule against `guideContainer.model.json`** after deploy: the field
> must carry the compiled `change` / `initialize` / `calculate` event. If it's missing, the call
> is multi-arg (make it single-arg) or `fd:events` was left empty (populate it).

Guidance:
- **Make validators empty-safe** (return true on empty) — `required` owns emptiness; an
  empty-safe validator also fails safe if a value token doesn't resolve.
- **Don't duplicate built-in checks.** When a validation expression owns a field's format
  check, REMOVE the field's inline `pattern` / `validatePatternMessage`. Keep `required` +
  `mandatoryMessage` and `maxLength`. ⚠️ FileVault `update`-mode does NOT delete a `pattern`
  you removed from source off a provisioned instance — clear it (Sling POST `@Delete` with
  CSRF token + `Referer` header) so it doesn't validate alongside your expression.

### Layer 2 (advanced) — programmatic validation

When validation must run without an author-authored rule (complex cross-panel logic,
async lookups, exact wording from a spec), hook the AF Core Components runtime.

> **Verified runtime API (read this — the obvious guesses are wrong).** The following is
> verified against this project's `core-forms-components-runtime-all` bundle. Earlier
> versions of this skill suggested `event.detail.formContainer`, `subscribe(…, "change")`
> and `markAsValid()` — **all three are wrong and silently no-op.** Use exactly:
> - `AF_FormContainerInitialised` carries the form container as **`event.detail` itself**
>   — NOT `event.detail.formContainer`.
> - `container.getModel()` returns the form model.
> - The root model emits **`fieldChanged`** (not `"change"`) when any descendant field
>   changes; the changed field is `action.event.payload.field`.
> - A field is flagged with `field.markAsInvalid(msg)`. **There is no `markAsValid()`** —
>   clearing would need the `valid` / `errorMessage` setters.

> **CRITICAL — be non-invasive.** Writing to the model (`markAsInvalid`, or the `valid` /
> `errorMessage` / `value` setters) makes the runtime **re-render** the field. For a
> native `<input type="date">` this **wipes the value the user just typed**. The robust
> pattern is to **never write to the model**: only READ it, and render errors into your
> own injected DOM element. Drive validation from real DOM `blur` / `change` events
> (matched by the input's `name` attribute), read single-input values straight from the
> DOM widget (a date input is always ISO `yyyy-mm-dd`), and read checkbox/radio-group
> values from the model. Cover the async-load race with a
> `window.FormView.getInstance().getFormModel()` poll fallback.

```js
// js/validations.js — non-invasive, DOM-driven; reads the model, never writes to it
(function () {
    "use strict";

    // Field name -> rule. Return an error string, or null when the value is valid.
    var RULES = {
        workEmail: function (v) { return isWorkEmail(v) ? null : "Use your corporate email address."; }
        // one entry per field, keyed by the field's EXACT name
    };
    function isWorkEmail(v) { return !!v && /^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(v); }

    function wrapperOf(field) { return (field && field.id) ? document.getElementById(field.id) : null; }

    // Read WITHOUT touching the model: single inputs from the DOM (a date input is
    // canonical ISO yyyy-mm-dd), checkbox/radio groups from the model (the array).
    function readValue(field) {
        var el = wrapperOf(field);
        if (el) {
            var inputs = el.querySelectorAll("input, select, textarea");
            if (inputs.length === 1 && inputs[0].type !== "checkbox" && inputs[0].type !== "radio") {
                return inputs[0].value;
            }
        }
        return field ? field.value : undefined;
    }

    function showError(field, msg) {
        var w = wrapperOf(field); if (!w) return;
        w.classList.add("field-error-highlight");
        var e = w.querySelector(".clientlib-error");
        if (!e) { e = document.createElement("div"); e.className = "clientlib-error"; e.setAttribute("aria-live", "polite"); w.appendChild(e); }
        e.textContent = msg;
    }
    function clearError(field) {
        var w = wrapperOf(field); if (!w) return;
        w.classList.remove("field-error-highlight");
        var e = w.querySelector(".clientlib-error"); if (e) e.textContent = "";
    }

    function validateField(field) {
        if (!field || !field.name || !RULES[field.name]) return true;
        var msg = RULES[field.name](readValue(field));
        if (msg) { showError(field, msg); return false; }
        clearError(field); return true;
    }

    function indexFields(model) {
        var map = {};
        (function walk(n) {
            if (!n) return;
            if (n.name && RULES[n.name]) map[n.name] = n;
            var kids = n.items || n.children;
            if (Array.isArray(kids)) kids.forEach(walk);
        })(model);
        return map;
    }

    var wired = false;
    function wire(model) {
        if (wired || !model) return;
        var map = indexFields(model);
        if (!Object.keys(map).length) return;   // tree not ready; poller retries
        wired = true;

        function onInteract(e) {
            var input = e.target; if (!input || !input.name) return;
            var field = map[input.name]; if (!field) return;
            setTimeout(function () { validateField(field); }, 0); // let the value settle
        }
        document.addEventListener("change", onInteract, true);
        document.addEventListener("blur", onInteract, true);

        // Block an invalid submit. Capture-phase runs before the runtime's handler.
        document.addEventListener("click", function (e) {
            var s = (e.target && e.target.nodeType === 1) ? e.target : (e.target && e.target.parentElement);
            var btn = s && s.closest ? s.closest(".cmp-adaptiveform-button__widget") : null;
            if (!btn) return;
            var ok = true;
            Object.keys(map).forEach(function (n) { if (!validateField(map[n])) ok = false; });
            if (!ok) { e.preventDefault(); e.stopImmediatePropagation(); }
        }, true);
    }

    document.addEventListener("AF_FormContainerInitialised", function (event) {
        var c = event && event.detail;                       // the container IS event.detail
        wire(c && typeof c.getModel === "function" ? c.getModel() : null);
    });

    // Fallback: the container may have initialised before this async footer script ran,
    // or an author-rule error may have interrupted the init event.
    var tries = 0;
    (function poll() {
        if (wired) return;
        try {
            if (window.FormView && typeof window.FormView.getInstance === "function") {
                wire(window.FormView.getInstance().getFormModel());
            }
        } catch (e) { /* runtime not ready yet */ }
        if (!wired && tries++ < 100) setTimeout(poll, 100);
    })();
})();
```

Guidance for Layer 2:
- **NEVER mutate the model to show errors** — `markAsInvalid` / the `valid` / `errorMessage`
  / `value` setters re-render the field and can clear native date inputs. Use your own DOM
  error element + highlight class (above). Add CSS for `.clientlib-error` and
  `.field-error-highlight` (see the CSS section).
- Read single-value widgets from the **DOM** (canonical, unambiguous); read checkbox /
  radio groups from the **model**.
- Match fields by the input `name` attribute (== the field name when no custom dataRef).
- Wrap in an IIFE; feature-detect every runtime method; never throw during init.
- Prefer Layer 1 for anything an author might reasonably want to change.
- A future-date / range guard belongs in a **Validate** rule (via `create-form-rules`),
  **never a Calculate rule** — Calculate rewrites the value (it can null it) and makes the
  field read-only.

---

## CSS — form-scoped styling

Keep clientlib CSS narrow; do not duplicate the theme. Scope rules to the form so a
shared library can't bleed into other forms.

```css
/* css/styles.css — scope under .cmp-adaptiveform-container so a shared library
   cannot bleed into other forms. Pair these with the Layer 2 non-invasive JS. */

/* Red border on an invalid field (class toggled by validations.js on the wrapper). */
.cmp-adaptiveform-container .field-error-highlight .cmp-adaptiveform-textinput__widget,
.cmp-adaptiveform-container .field-error-highlight .cmp-adaptiveform-telephoneinput__widget,
.cmp-adaptiveform-container .field-error-highlight .cmp-adaptiveform-emailinput__widget,
.cmp-adaptiveform-container .field-error-highlight .cmp-adaptiveform-datepicker__widget,
.cmp-adaptiveform-container .field-error-highlight textarea {
    border-color: #d7373f;
    box-shadow: 0 0 0 1px #d7373f;
}

/* Checkbox/radio groups have no single widget — outline the whole group instead. */
.cmp-adaptiveform-container .cmp-adaptiveform-checkboxgroup.field-error-highlight {
    outline: 1px solid #d7373f;
    outline-offset: 4px;
    border-radius: 4px;
}

/* Our injected error message (validations.js appends a .clientlib-error div). */
.cmp-adaptiveform-container .clientlib-error {
    color: #d7373f;
    font-size: 0.85rem;
    margin-top: 4px;
}
```

Use the AF Core Components BEM classes (`cmp-adaptiveform-*`). For full visual theming
(tokens, typography, colours) use `create-form-theme` instead — this CSS is for small,
behaviour-coupled tweaks (error highlights, conditional layout) that belong with the JS.

### Verified class reference (don't guess these)

Confirmed against the rendered AF Core Components DOM in this project. The non-obvious
ones bite hardest — a wrong class fails silently (no error, just no styling):

| Element                     | Class to target                                              |
|-----------------------------|-------------------------------------------------------------|
| Form container (scope root) | `.cmp-adaptiveform-container` (also carries base `.cmp`)     |
| Field wrapper               | `.cmp-adaptiveform-{type}` (e.g. `…-textinput`, `…-datepicker`) |
| Widget input                | `.cmp-adaptiveform-{type}__widget`                          |
| Field label                 | `.cmp-adaptiveform-{type}__label` (+ `__label-container`)    |
| Runtime error message       | `.cmp-adaptiveform-{type}__errormessage`                     |
| **Panel / section**         | **`.cmp-container`** — NOT `cmp-adaptiveform-panelcontainer` (that class does **not** exist in the DOM) |
| **Section heading**         | **`.cmp-container__label`** (a `<label>` in a `role="heading"` div) — NOT bold by default; add `font-weight:700` if the reference shows bold headings |
| **Form title**              | **`.cmp-title__text`** (the `<h1>`) inside `.cmp-title` — NOT `cmp-adaptiveform-container__title` (not in DOM). The title is an explicit **AF Title component** (guideContainer `showTitle` is false — its band emits no title element); style `.cmp-title`/`.cmp-title__text` (served fresh in the clientlib) so the title actually renders |
| **Submit/Reset buttons**    | field `.cmp-adaptiveform-button`; the grid COLUMN wrapping it carries the `button` class (+ `submit`/`reset`) — the `actionsPanel` node name is NOT emitted as a class, so scope button-row centring via `.aem-Grid:has(> .aem-GridColumn.button)` |
| Page header                 | `.cmp-adaptiveform-pageheader`                               |
| Title / static text         | `.cmp-adaptiveform-text`                                     |
| Checkbox / radio group      | `.cmp-adaptiveform-checkboxgroup` (options: `…__option`, `…__option-label`) |
| Multi-line text (textarea)  | a bare `<textarea>` — it has **no** `__widget` class         |
| Submit button               | `.cmp-adaptiveform-button__widget`                           |
| **Required field marker**   | **`[data-cmp-required="true"]` on the field wrapper** — CC emits NO `__label__qualifier` span, so style the asterisk via `::after` off this attribute (optional fields carry `="false"`) |

`{type}` ∈ `textinput`, `telephoneinput`, `emailinput`, `datepicker`, `dropdown`,
`checkboxgroup`, `numberinput`, … (matches `sling:resourceType` of the field).

### Red required asterisk MUST live in the clientlib CSS (like the row layout)

The reference red `*` on required labels does **not** render from a theme rule targeting
`.cmp-adaptiveform-*__label__qualifier` — **Core Components never emits that span** (0 in the DOM).
Worse, the aggregated `.../{form}.theme/_default/theme.css` is a **cache that lags the redeployed
theme.zip** (package install / JCR touch / clientlib invalidate / bundle restart do NOT bust it), so
a theme-only fix can be "present" yet never served. So the authoritative asterisk rule belongs in the
**clientlib CSS** (served fresh on every deploy via its hashed `.lc-…` URL). Hook off the field
wrapper's `data-cmp-required="true"` attribute:

```css
[data-cmp-required="true"] .cmp-adaptiveform-textinput__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-emailinput__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-telephoneinput__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-numberinput__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-datepicker__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-dropdown__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-radiobutton__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-checkbox__label::after,
[data-cmp-required="true"] .cmp-adaptiveform-checkboxgroup__label::after {
    content: " *"; color: #e02020; font-weight: 700;
}
```
Optional fields (`data-cmp-required="false"`, e.g. an optional email or checkbox group) are excluded
automatically. Verify by rendered pixels that the `*` shows on required labels and NOT on optional ones.

### Section separators / dividers MUST live in the clientlib CSS (served fresh)

Every visual element in the reference must be reproduced — **not only the fields**. A common one that
gets dropped is the **section separator / divider**: the thin horizontal line under each numbered
section and under the title/subtitle header. Draw it in the **clientlib CSS** (injected verbatim,
served fresh on every deploy — unlike the lagging aggregated `theme.css`) as a `border-bottom` on the
real served section-panel classes plus a rule under the header:

```css
/* Divider under each section panel and under the title/header band.
   Verify these against the ACTUAL served DOM classes — .cmp-container is the panel/section,
   .cmp-title is the title band. Do NOT invent classes; a wrong class silently draws nothing. */
.cmp-adaptiveform-container .cmp-container { border-bottom: 1px solid #d9d9d9; padding-bottom: 1rem; }
.cmp-adaptiveform-container .cmp-title    { border-bottom: 1px solid #d9d9d9; padding-bottom: 0.75rem; }
```

Confirm the separators actually render by rendered **pixels** after redeploy — CSS presence is not
proof. (Images/logos/icons from the reference are stored DAM assets rendered by **authored Adaptive
Form Image components** — `fileReference` → DAM asset. Clientlib CSS must **NOT** draw any icon via
`background-image` / `::before` / inline SVG / base64 — not even decorative section-heading or tile
icons; it may only **size/place** the authored Image components via their **served**
`.cmp-adaptiveform-image*` / grid-cell classes. This supersedes any earlier "CSS background-image
URL" guidance. See `create-adaptive-form` → "Icons & imagery from a reference".)

> 🛑 **A node's `css="…"` property is NOT emitted as a DOM class by the Core Components HTL** — it
> surfaces only in `guideContainer.model.json`. Clientlib CSS *and* clientlib JS
> (`querySelectorAll`) must target the classes/attributes actually served
> (`.cmp-adaptiveform-*`, `h1`/`h2.cmp-title__text`, `[data-cmp-required]`, `[data-cmp-readonly]`,
> `button[type="submit"|"reset"]`, `.aem-GridColumn--default--N`). Verify with
> `curl.exe -s -u admin:admin ".../{formName}.html" | grep -c '<your-selector>'` — a count of 0 means
> the rule is dead. Full table: `create-form-theme` → "NEVER style a form via a `css=` class hook".

### Multi-column row layout MUST live in the clientlib, AND the clientlib MUST OWN the column widths

🛑 **MANDATORY whenever ANY panel has more than one field per row** (a 2-col, 3-col, or 4-col
layout — i.e. any field whose `cq:responsive` span is < 12). Omitting this is the **#1 recurring
UI-parity defect**: the whole form collapses to a **single column** and fails the ≥90% pixel gate.
Author this block in EVERY replica/multi-column form clientlib — do not wait for a test to catch it.

**Two things are needed, and the second is the one everyone forgets:**

1. **Switch the panel field grid from float to flex** (fixes the staircase stagger of unequal-height
   field blocks — label + badge + widget + error row).
2. **The clientlib MUST OWN the per-span widths.** ⚠️ The OOB responsive-grid width classes
   (`.aem-GridColumn--default--N`) are **NOT loaded on the AF page** — the stylesheet that defines
   their `width: N/12` is not served, so relying on them (as older guidance did) leaves every column
   at its default `flex` width and the form renders single-column. So the clientlib must set an
   explicit `flex-basis` per span. Do **not** assume the OOB classes carry the width.

- The **theme is the WRONG layer**: the AF theme processor **strips any theme selector that
  targets the structural grid classes** (`.aem-Grid` / `.aem-GridColumn`). A grid rule put in
  the theme deploys into `theme.zip` but is silently removed from the served
  `…/<form>.theme/_default/theme.css`. The clientlib CSS is injected verbatim (not sanitized),
  so it is the only layer where a grid rule actually reaches the rendered form.
- **Verified DOM chain** (the panel field grid is a GRANDCHILD of the panel):
  `.panelcontainer > .cmp-container > .aem-Grid > .aem-GridColumn--default--N`.
  So the correct selector is `.panelcontainer .cmp-container > .aem-Grid` — NOT
  `.cmp-adaptiveform-panelcontainer .aem-Grid` (class doesn't exist) and NOT
  `.panelcontainer > .aem-Grid` (the grid is not a direct child). Both match nothing.
- **Embedded fragments** carry their own inner `.fragment` grid — apply the SAME rules to
  `.panelcontainer .cmp-container > .aem-Grid, .fragment .aem-Grid { … }` so fields inside a
  fragment (e.g. an Address panel) also lay out multi-column, not just native panels.

```css
/* Multi-column rows: switch float -> flex AND own the per-span widths, because the
   OOB .aem-GridColumn--default--N width classes are NOT served on the AF page.
   flex-basis per span: 12=100%, 6=50%, 4=33.33%, 3=25%. box-sizing:border-box keeps
   the gutter padding inside the basis (no overflow); the negative grid margin lines
   the OUTER edges up with full-width fields; row-gap spaces the rows. Mobile (<768px)
   forces single column. Apply to native panel grids AND embedded fragment grids. */
.panelcontainer .cmp-container > .aem-Grid,
.fragment .aem-Grid {
  display:flex; flex-wrap:wrap; align-items:flex-start;
  margin-left:-12px; margin-right:-12px; row-gap:0.5rem;
}
.panelcontainer .cmp-container > .aem-Grid > .aem-GridColumn,
.fragment .aem-Grid > .aem-GridColumn {
  float:none; box-sizing:border-box; padding-left:12px; padding-right:12px;
}
/* OWN the widths — do NOT rely on the OOB width classes (not loaded on the AF page). */
.aem-GridColumn--default--12 { flex-basis:100%;    max-width:100%; }
.aem-GridColumn--default--6  { flex-basis:50%;      max-width:50%; }
.aem-GridColumn--default--4  { flex-basis:33.3333%; max-width:33.3333%; }
.aem-GridColumn--default--3  { flex-basis:25%;      max-width:25%; }
.aem-GridColumn.aem-GridColumn--offset--default--6 { margin-left:0; }
.aem-GridColumn.aem-GridColumn--default--newline   { margin-left:0; }
@media (max-width:767px) {
  .panelcontainer .cmp-container > .aem-Grid > .aem-GridColumn,
  .fragment .aem-Grid > .aem-GridColumn { flex-basis:100% !important; max-width:100% !important; margin-left:0 !important; }
}
```

See [[form-clientlib-owns-grid-widths]].

### The form clientlib MUST mirror the brand `:root` token override (so the EMBEDDED page isn't off-brand)

🛑 **MANDATORY for any form that will be embedded in a Sites page** (every delivery — the
`composer`/`assembler` phase embeds the form into the "Test Adaptive Form" page). Standalone, the
form's brand colours come from its `theme.css`. **But in the embedded Sites page the form theme
selector does NOT apply** — only the categories listed in the page's `formEmbedClientlibs` load —
so the `core.forms.components.runtime.all` **default blue** (`#1473e6`) wins and the in-page form
renders off-brand (the recurring "form is blue in the page" defect).

The fix: the **form-specific clientlib** (which loads in BOTH the standalone form AND the embedded
page, because it's the category wired into `formEmbedClientlibs`) must **also carry the brand
`:root` token override** — the same values the form theme sets. Because `css.txt` loads the copied
`base.css` (default tokens) first and `form.css` after, and both are plain `:root` specificity, the
brand override in `form.css` wins in every context.

```css
/* form.css — mirror the form theme's brand tokens here so the embedded page renders on-brand.
   Same values as the form's theme.css :root override. Loaded after base.css (default tokens). */
:root {
  --af-primary: #1b5e20;          /* brand primary — matches the form theme */
  --af-primary-dark: #14481a;
  --af-section-heading: #1b5e20;
  /* …mirror every brand token the theme overrides… */
}
```

This is deliberate, controlled duplication of the *token values only* (never the standard element
styling) between the theme and the form clientlib — it is the single reliable way to keep the
embedded-page form on-brand independent of theme-selector/theme-cache behaviour in the page context.
See [[form-clientlib-mirrors-brand-tokens-for-embed]].

### Native date input overflows its column — clamp it

`<input type="date">` (`.cmp-adaptiveform-datepicker__widget`) has a browser-default
intrinsic min-width (the `mm/dd/yyyy` text + spinner + calendar button) that is wider than
a 50% grid column, so the date field overflows and breaks its row's alignment. Always clamp:

```css
.cmp-adaptiveform-datepicker__widget { width:100%; max-width:100%; min-width:0; box-sizing:border-box; }
```

### Never show validation error icons/messages at INITIAL load

The runtime keeps an **empty** `…__errormessage` element in the DOM for every field at all times.
Two things commonly make errors appear before the user has interacted (a defect):

1. **A theme `::before` warning glyph not gated by state.** If the theme paints a "⚠" via
   `.cmp-adaptiveform-{type}__errormessage::before`, it lands on the always-present *empty* element
   and shows on every field at load. The theme should gate it with `:not(:empty)`, but the form's
   served `…/<form>.theme/_default/theme.css` is a **cached processed rendition** that can lag the
   redeployed `theme.zip`. So **also** suppress it from the clientlib (un-cached, loads after the
   theme), which is authoritative:
   ```css
   .cmp-adaptiveform-textinput__errormessage:empty::before,
   .cmp-adaptiveform-emailinput__errormessage:empty::before,
   .cmp-adaptiveform-telephoneinput__errormessage:empty::before,
   .cmp-adaptiveform-numberinput__errormessage:empty::before,
   .cmp-adaptiveform-datepicker__errormessage:empty::before,
   .cmp-adaptiveform-dropdown__errormessage:empty::before,
   .cmp-adaptiveform-fileinput__errormessage:empty::before,
   .cmp-adaptiveform-checkbox__errormessage:empty::before { content: none !important; }
   ```
2. **JS that validates on load.** Never call validation routines, `markInvalid`, or
   `model.validate()` during init/`wire()`. Validate only on `blur`/`change`/submit. The submit-gate
   may *count* completeness on load (to disable the button) but must not *mark fields invalid*.

### Submit + PDF generation MUST be gated on validation success

Submission AND any PDF / Document-of-Record generation fire **only when ALL form validation passes**.
An invalid form must BLOCK submission, show inline error messages, scroll/focus the first invalid
field, and produce **NO PDF**. Never POST to the GeneratePDF servlet (or any submit endpoint)
unconditionally from a client handler — gate on form validity. Prefer the **native Core Components
submit** (which validates before calling the action); if a clientlib drives the PDF, check the
guideBridge/model validity first:

```js
// Only proceed when the whole form is valid — otherwise let the runtime show its errors + focus.
var model = window.FormView.getInstance().getFormModel();
var valid = (model.validate() || []).length === 0;   // validate() returns the list of invalid fields
if (!valid) { return; }                              // BLOCK: no submit, no PDF; runtime shows errors
// ... proceed with the gated submit / PDF POST
```

Keep the shared `Custom-Submit-GeneratePDF` as the native validating submit — not a raw button
`onClick` that posts to the servlet. Add a traceable test (user story: "submission only succeeds when
validation passes"): empty/invalid form shows errors and produces NO PDF; valid form submits AND
generates the PDF.

### Avoid duplicate error messages

The runtime renders its **own** `…__errormessage` for built-in constraints (`required`,
type, pattern). If your non-invasive JS also shows a `.clientlib-error` for the *same*
condition (e.g. "required"), the user sees **two** messages. Choose one:

- **Recommended:** let the runtime own the constraints it already enforces — don't add a
  JS "required" check for a field that already has `required="true"`; only add JS for
  rules the runtime can't express (custom format, cross-field, business logic).
- **Or** fully own the messaging and suppress the native one:

  ```css
  /* Only if your clientlib owns ALL validation messaging for this form. */
  .cmp-adaptiveform-container [class$="__errormessage"] { display: none; }
  ```

---

## Attaching the clientlib to a form

Add `clientLibRef` (the **category**) to the form's `guideContainer` in
`/content/forms/af/{folder}/{formName}/.content.xml`:

```xml
<guideContainer
    jcr:primaryType="nt:unstructured"
    sling:resourceType="{project}/components/adaptiveForm/formcontainer"
    fieldType="form"
    name="{formName}"
    clientLibRef="{project}.forms.{name}"
    ... >
```

That is the entire wiring. On render, the page component injects the CSS (head) and JS
(footer, async). ⚠️ `clientLibRef` MUST be a **SINGLE category** — the form-specific one. Do
**NOT** use a comma-separated list: the Rule-Editor custom-functions endpoint resolves
`clientLibRef` as one category and a comma value returns no functions (every custom-fn rule
shows "Broken"). Keep the form-specific clientlib **self-contained** — its own copied
`functions.js` (each `@name` exactly once) and its own copied base styling (`css/base.css`) —
and set `dependencies="[core.forms.components.runtime.all]"` (+
`,{project}.forms.generate-pdf` when a PDF is generated on submit). 🛑 Never
`embed`/`depend on` `forms.base` (it redefines the same `@name` functions → duplicate
definitions → the endpoint returns empty → all rules "Broken").

> Authors can also set this without code: form container → **Configuration** →
> *Client Library Category*. Setting it in source keeps it reproducible across envs.

---

## Build, deploy, verify

> **Pipeline mode (delegated by `formwright`): SKIP the build/deploy — author the clientlib only.**
> Deployment is centralized in the `forgemaster` lead (AGENTS.md → "Deployment is centralized in Forgemaster").
> Run the `mvn` command only when this skill is invoked **standalone / directly**.

```bash
# Component/clientlib changes deploy with ui.apps; use the all-package profile
mvn clean install -PautoInstallSinglePackage

# Verify the proxied clientlib serves (AEMaaCS exposes /etc.clientlibs, not /apps):
#   http://localhost:4502/etc.clientlibs/{project}/clientlibs/clientlib-{name}.js
#   http://localhost:4502/etc.clientlibs/{project}/clientlibs/clientlib-{name}.css
# Then open the form and confirm the <script>/<link> for the category is present:
#   http://localhost:4502/content/forms/af/{folder}/{formName}.html
```

> Use `-PautoInstallSinglePackage` (the all-package); direct single-module install can
> fail (see [core-bundle-install-via-all-package] memory). `allowProxy` is what makes
> the `/etc.clientlibs` URL resolve — if it 404s, that flag is missing.

### Testing & deploy gotchas (these cost real debugging time)

- **Test in Preview or the runtime `.html`, NOT `/editor.html`.** The authoring canvas
  (`/editor.html/...`) does not run form-fill validation or rules — typing selects
  components for editing. Use the **Preview** button or open
  `http://localhost:4502/content/forms/af/{folder}/{formName}.html`.
- **Hard-refresh (Ctrl+F5).** The page HTML, the clientlib JS/CSS, and the form's
  `model.json` are all cached; a soft reload can run stale code or stale rules.
- **Form rules live in `model.json`, not the page HTML.** To verify a rule was added or
  removed, check
  `…/content/forms/af/{folder}/{formName}/jcr:content/guideContainer.model.json` (grep for
  the rule script) — checking the `.html` is misleading because rules load via the model.
- **FileVault `update`-mode installs do NOT delete nodes/properties you removed from
  source.** An obsolete rule (e.g. a `fd:rules` child node) can linger on an already-
  provisioned instance even after you delete it from source and redeploy. Remove it
  directly, then redeploy:
  `curl -u admin:admin -F":operation=delete" "http://localhost:4502/<path-to-node>"`
  (fresh instances are fine — they only get what the package contains).
- **Confirm the deployed JS is yours.** Fetch the served, minified clientlib and grep for
  a marker from your source; a content-hash change in the `clientlib-{name}.lc-<hash>.min.js`
  URL confirms the new build went live (and rules out a hand-edited copy in CRXDE).

---

## Quality checklist — verify before finishing

- [ ] Clientlib is a `cq:ClientLibraryFolder` with `allowProxy="{Boolean}true"`
- [ ] **Reuse-first split honored:** generic scripts/CSS live in the shared **base** forms
      clientlib (category `{project}.forms.base`, under `/apps/{project}/clientlibs`,
      extending the existing `clientlib-base` area) as the CANONICAL reference; the form-specific
      clientlib is **SELF-CONTAINED** — its `functions.js` copies (once) the subset of `@name`
      functions the form's rules reference, and its `css/` copies the base styling
      (`base.css`) — and it does **NOT** `embed`/`depend on` `forms.base`
- [ ] **No duplicate `@name` definitions in the built clientlib** — `grep -c "function <name>"`
      on the deployed/merged `{formName}-clientlib.js` returns **1** for every custom function
      (2+ means the base was embedded/depended → the customfunctions endpoint returns empty → all
      rules "Broken")
- [ ] `clientLibRef` on the guideContainer is a **SINGLE category** (the form-specific one) —
      NOT a comma-separated list. A multi-value `clientLibRef` returns an empty custom-function
      list from the Rule-Editor endpoint and marks every custom-fn rule "Broken." Styling is
      self-contained (copied base.css); only non-custom-function libs
      (`core.forms.components.runtime.all`, `…forms.generate-pdf` when PDF-on-submit) go in
      `dependencies`
- [ ] `categories` is unique and follows `{project}.forms.{name}`
- [ ] `dependencies` includes `core.forms.components.runtime.all` when JS uses the model
- [ ] `js.txt` / `css.txt` use `#base=` and list files in load order
- [ ] Custom functions carry full JSDoc (`@name`, `@param`, `@return`) so they surface
      in the Rule Editor
- [ ] `functions.js` declares functions at **global scope** — NO IIFE, NO namespace object
      (else the Rule Editor cannot discover them); internal helpers marked `@private`
- [ ] Validators are **empty-safe** (`return true` on null/undefined/"") so `required`
      owns emptiness and a mis-resolved token fails safe
- [ ] `functions.js` ends with a **Rule Editor usage notes** block mapping field → function → rule type
- [ ] Validation wired in THIS skill with BOTH parts: (A) runtime attrs
      `validationExpression="<expr> == true()"` + `validateExpMessage="<msg>"` (verified as a
      top-level `validationExpression` in `guideContainer.model.json`); (B) a `fd:rules`
      child with the authored `fd:validate` AST (`VALIDATE_EXPRESSION` → `COMPARISON_EXPRESSION`
      → `FUNCTION_CALL == BOOLEAN_LITERAL True`) + empty `fd:events`, so the rule is VISIBLE
      in the Rule Editor. `$field.$value` for current field, `$form.<Panel>.<Field>.$value`
      cross-field. Generate the escaped AST with a script — never hand-type it
- [ ] For **non-validate** rules (Initialize / Value Commit / Calculate / Click): `fd:events`
      is **POPULATED** with the runtime key (`initialize` / `change` / `calculate` / `click`) —
      empty `fd:events` is correct ONLY for Validate; the set-value function returns the new value
      (not boolean) and is single-argument (constants baked in); commas inside `fd:events` `[...]`
      escaped `\,`; self-referential `change` handlers use a loop guard; Click uses the
      `EVENT_SCRIPTS → SUBMIT_FORM` AST (not a bare `fd:click="submitForm()"` string); each
      compiled event verified present in `guideContainer.model.json`
- [ ] Redundant inline `pattern`/`validatePatternMessage` removed; stale copies cleared on
      the instance via Sling POST `@Delete` (CSRF token + `Referer` header) — FileVault
      update-mode won't delete them
- [ ] Programmatic JS is wrapped in an IIFE, feature-detects runtime APIs, never throws
      during init
- [ ] Programmatic validation is **non-invasive**: it READS the model and renders errors
      into its own DOM element (`.clientlib-error`) + highlight class; it NEVER calls
      `markAsInvalid` / the `valid` / `errorMessage` / `value` setters (those re-render and
      can wipe native date inputs)
- [ ] Uses the **verified runtime API**: the container IS `event.detail`; field changes via
      `fieldChanged` (or DOM events); single-input values read from the DOM, group values
      from the model; FormView poll fallback for the init race (no `markAsValid()`)
- [ ] CSS is scoped to the form (no bare element selectors) and uses `cmp-adaptiveform-*`
- [ ] Form is attached via `clientLibRef="<category>"` on the guideContainer (NOT a path,
      NOT an edit to customheaderlibs/`/libs`)
- [ ] Runtime concerns pulled in via `dependencies` on **non-custom-function** libs only
      (core runtime, generate-pdf); custom functions + base styling are COPIED self-contained,
      never obtained by embedding/depending on `forms.base`
- [ ] Verified the `/etc.clientlibs/....js|.css` URL serves after deploy
- [ ] Tested in **Preview / runtime `.html`** (not `/editor.html`) after a hard refresh;
      any rule change verified against `guideContainer.model.json`, not the page HTML

---

## Example

Developer prompt: *"Create a clientlib for the employee-registration-form: validate that
the work email isn't a free webmail domain and that joining date isn't before date of
birth, and highlight invalid fields in red."*

See `references/examples.md` for the complete generated clientlib (node, manifests,
functions.js, validations.js, styles.css) and the `clientLibRef` wiring.
