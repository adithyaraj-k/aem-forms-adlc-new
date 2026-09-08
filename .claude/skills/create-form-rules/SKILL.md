---
name: create-form-rules
description: >
  Generates AEM Adaptive Forms Rule Editor rules as JSON for show/hide,
  enable/disable, validate, calculate, and set-value conditions.
  Works with Core Components on AEM as a Cloud Service.
version: 1.0.0
ide:
  cursor: .cursor/skills/create-form-rules/
  github-copilot: .github/skills/create-form-rules/
  claude-code: .claude/skills/create-form-rules/
---

# Skill: create-form-rules

## Role

You are an expert in AEM Adaptive Forms Rule Editor for AEM as a Cloud Service Core Components.
You generate precise, working rule JSON that attaches directly to form field nodes in the
content XML. You never use deprecated rule APIs. You never use direct DOM manipulation.

---

## Trigger

This skill activates when the developer asks to:
- Add show/hide logic to a form field
- Make a field required or optional based on another field's value
- Calculate a field value from other fields
- Validate a field with custom logic
- Enable or disable a field based on conditions
- Cascade dropdown options based on a parent selection

---

## How rules attach to field nodes

Rules live inside a `<rules>` child node on the field. Each rule has an event type
and a condition/action pair.

```xml
<fieldName
  jcr:primaryType="nt:unstructured"
  sling:resourceType="core/fd/components/form/textinput/v1/textinput"
  jcr:title="Field Label"
  name="fieldName">

  <fd:rules jcr:primaryType="nt:unstructured">
    <show jcr:primaryType="nt:unstructured"
      condition="{JSON condition string}"
      action="show"/>
    <hide jcr:primaryType="nt:unstructured"
      condition="{JSON condition string}"
      action="hide"/>
  </fd:rules>

</fieldName>
```

---

## Rule operator reference

Always use these exact operator strings. No variations.

| Condition type        | Operator string         |
|-----------------------|-------------------------|
| Equals                | `EQUALS`                |
| Not equals            | `NOT_EQUAL`             |
| Is empty              | `IS_EMPTY`              |
| Is not empty          | `IS_NOT_EMPTY`          |
| Greater than          | `GREATER_THAN`          |
| Less than             | `LESS_THAN`             |
| Greater than or equal | `GREATER_THAN_OR_EQUAL` |
| Less than or equal    | `LESS_THAN_OR_EQUAL`    |
| Contains              | `CONTAINS`              |
| Starts with           | `STARTS_WITH`           |
| Ends with             | `ENDS_WITH`             |
| AND (combine rules)   | `AND`                   |
| OR (combine rules)    | `OR`                    |

---

## Action type reference

| What you want              | Action string       |
|----------------------------|---------------------|
| Show field/panel           | `SHOW`              |
| Hide field/panel           | `HIDE`              |
| Enable field               | `ENABLE`            |
| Disable field              | `DISABLE`           |
| Make required              | `SET_MANDATORY`     |
| Make optional              | `CLEAR_MANDATORY`   |
| Set field value            | `SET_VALUE`         |
| Clear field value          | `CLEAR_VALUE`       |
| Show validation error      | `VALIDATE`          |
| Invoke a service           | `INVOKE_SERVICE`    |
| Navigate to wizard step    | `NAVIGATE_TO_PANEL` |

---

## Rule patterns — copy and adapt

> ⚠️ **CRITICAL — visibility (show/hide) rules MUST be stored as an `fd:visible` property, NOT `fd:visibility`, and NOT a `<visibility>` child node.**
> This project proxies Core Components (`core/fd/components/form/...`). Core Components read a field/panel's show-hide rule **only** from a property named **`fd:visible`** on the `<fd:rules>` node, holding an **escaped-JSON `SHOW_EXPRESSION` AST**. If you author it under `fd:visibility` (the old Foundation name), or with a `SET_PROPERTY`-style AST, or as a `<visibility>`/`<condition>`/`<action>` child element, **the build still succeeds and the form still renders, but the rule is silently ignored** — the panel/field never toggles. This is the same "wrong-for-Core-Components storage" failure class as authoring dropdown options with `<items>` instead of `enum`/`enumNames`. The `<visibility>`-style XML shown in the patterns below is **conceptual shorthand for the condition only** — when you write the actual form `.content.xml`, you MUST emit the `fd:visible` AST form described here.
>
> **Required storage shape** (copy the exact structure from a validated in-repo reference — search for `fd:visible=` in
> `ui.content/.../conf/.../settings/wcm/templates/consent-form/initial/.content.xml`):
> - Property name on the `<fd:rules>` node: **`fd:visible="[…escaped JSON AST…]"`**
> - AST: `ROOT` → `STATEMENT` → **`SHOW_EXPRESSION`** with: `AFCOMPONENT` (the field/panel being shown, e.g. `$form.addressPanel`), `When`, `CONDITIONORALWAYS` → `COMPARISON_EXPRESSION` (the controlling `COMPONENT`, e.g. `$form.appointment.collectionMethod`, `EQUALS_TO` a `STRING_LITERAL`), `Else`, `DONOTHING_OR_HIDE` → `Hide`; with `eventName":"Visibility"`.
> - The `<fd:rules>` node ALSO carries a plain script attribute **`visible="<expression>"`** (e.g. `visible="collectionMethod == &quot;HomeCollection&quot;"`) next to `validationStatus="valid"`, plus the sibling empty `<fd:events jcr:primaryType="nt:unstructured"/>` node.
> - Compare against the value's **enum CODE**, never its display label (e.g. `"HomeCollection"`, not `"Home Collection"`).
> - Give the target field/panel a default `visible="{Boolean}false"` so it starts hidden until the condition is met.
>
> ```text
> ❌ WRONG — silently ignored by Core Components:
>   <fd:rules>
>     <visibility><condition fieldName="collectionMethod" operator="EQUALS" value="HomeCollection"/><action>SHOW</action></visibility>
>   </fd:rules>
>   …or  fd:visibility="[{…SET_PROPERTY…}]"
>
> ✅ RIGHT — fd:visible with a SHOW_EXPRESSION AST on the fd:rules node:
>   <fd:rules jcr:primaryType="nt:unstructured"
>       fd:visible="[{…ROOT→STATEMENT→SHOW_EXPRESSION… EQUALS_TO STRING_LITERAL HomeCollection … Hide…}]"
>       validationStatus="valid"
>       visible="collectionMethod == &quot;HomeCollection&quot;"/>
>   <fd:events jcr:primaryType="nt:unstructured"/>
> ```
>
> After deploy, verify on the instance: `…/guideContainer.model.json` for the target node must show `"rules":{"visible":"<expression>"}` and contain **0** occurrences of `SET_PROPERTY` or `fd:visibility`. If a stale rule lingers after a structure change, delete the form node on the instance and redeploy (FileVault replace mode may not purge it).

### Rule storage property map (applies to EVERY rule type, not just visibility)

The `fd:visible` guard above is one row of a general rule: **every Rule Editor rule is stored as an
escaped-JSON AST in a specifically-named `fd:` property on the field's `<fd:rules>` node**, alongside a
plain-text mirror attribute and the sibling empty `<fd:events jcr:primaryType="nt:unstructured"/>` node.
The simplified `<visibility>` / `<calculate>` / `<validate>` / `<enable>` child-element XML used in the
patterns below is **conceptual shorthand for the condition only** — when you write the real form
`.content.xml`, emit the `fd:*` property from this map. Using the wrong property name (or a Foundation-era
name / AST) builds clean but is **silently ignored at runtime**.

| Rule intent | Property on `<fd:rules>` | AST top node (`nodeName`) | `eventName` | Mirror attribute | In-repo status |
|---|---|---|---|---|---|
| Show / hide | **`fd:visible`** | `SHOW_EXPRESSION` | `Visibility` | `visible="<expr>"` | ✅ verified (consent-form, lab-test-form) — NOT `fd:visibility`/`SET_PROPERTY` |
| Validate | **`fd:validate`** | `VALIDATE_EXPRESSION` | `Validate` | `validationStatus="valid"` (+ `validationExpression` on field) | ✅ verified (kyc-form, lab-test-form) — expression MUST end `== true()` |
| On-change script (incl. set-value of another field) | **`fd:change`** | `EVENT_SCRIPTS` (with `SET_VALUE` inside) | `Change` | — | ✅ verified (lab-test-form) |
| Button click (submit, etc.) | **`fd:click`** | `EVENT_SCRIPTS` (with `SUBMIT_FORM` inside) | `Click` | also set `<fd:events ... click="[submitForm()]"/>` (property is **`click`**, NOT `fd:click`) | ✅ verified (account-opening, kyc, enrollment) |
| Initialize | **`fd:init`** | `EVENT_SCRIPTS` | `Initialize` | — | ✅ verified (account-opening-form, test) |
| Enable / disable | **`fd:enabled`** | `ACCESS_EXPRESSION` (disable variant: `DISABLE_EXPRESSION`) | `Enabled` | `enabled="<expr>"` | ⚠️ node name from the grammar (below) — NOT `fd:enable`, NOT `ENABLE_EXPRESSION` |
| Calculate / set a field's value (own OR another field, conditionally) | **`fd:calc`** | `CALC_EXPRESSION`, target field addressed as `VALUE_FIELD` (not `AFCOMPONENT`) | `Calculate` | `value="<expr>"` (the reactive runtime driver — same role as `visible=`/`validationExpression=` for other rule types); readOnly is a free choice, NOT required | ✅✅ **double-verified live via the AEM Rule Editor** (employee-training-request's `managerId`/`managerName`, editor-authored then read back from the deployed JCR node) — property is **`fd:calc`**, NOT `fd:value`, NOT `fd:calculated`/`fd:calculate`. Structure: `VALUE_FIELD` (target) → `"to"` → `EXPRESSION` (then-value) → `"When"` → `CONDITIONORALWAYS` → `BOOLEAN_BINARY_EXPRESSION` (AND of `COMPARISON_EXPRESSION`s, each referencing OTHER fields as `COMPONENT`). The `script`/`value` mirror is a ternary: `if(cond, thenValue, $field)` — `$field` as the else-branch means "leave the current value alone otherwise". **`fd:events` stays completely empty — no `dispatchEvent`/custom function needed**, even though the calculated field is a DIFFERENT field than the one that triggered the condition. ⚠️ **Caveat: this rule was authored live in the AEM Rule Editor UI, then read back from the deployed JCR node — it is NOT yet verified that hand-writing this same `fd:calc` + `value=` mirror directly into `.content.xml` and shipping it via a pure package deploy (no editor round-trip) produces the same working result.** `migrate-form` SKILL.md's existing "Cross-field auto-fill" guidance found that a *hand-authored* `fd:calc` deployed via package alone did NOT compile to `model.rules.value` (that test predates the discovery of the `value=` mirror attribute name, so it may not have included it) — its `dispatchEvent`-from-source-fields fallback remains the proven-safe method for a pure package deploy. Prefer round-tripping through the editor when possible; if you must hand-author for a package-only deploy, include the `value=` mirror and verify in `guideContainer.model.json` after deploy — don't assume it compiles. |
| Clear own value | **`fd:calc`** | `CLEAR_EXPRESSION` | `Calculate` | `value="<expr>"` | ⚠️ node name from the grammar (below) — same property as Calculate above, not `fd:value` |
| Dynamic label | **`fd:label`** | `FORMAT_EXPRESSION` | `Format` | — | ⚠️ node name from the grammar (below) — there is no `LABEL_EXPRESSION` |
| Dynamic options (cascade) | **`fd:enum`** + **`fd:enumNames`** | `OPTIONS_EXPRESSION` | `Options` | — | ⚠️ node name from the grammar (below) — NOT `ENUM_EXPRESSION`; static options use plain `enum`/`enumNames` attrs, never `<items>` |

> 🛑 **The AST `nodeName` must be one the Rule Editor's grammar actually defines, and `script` must be
> the right JSON type.** These two are invisible at build time AND at runtime — the form still submits and
> `guideContainer.model.json` still shows the rule — but the **Rule Editor renders the rule as
> `Unknown Field - null`** and the author can no longer read or edit it. This happened with a
> hand-guessed `VALUE_EXPRESSION` on event-ticket-booking.
>
> **Why it fails:** the editor walks the AST with `enter<NODENAME>` visitor methods and titles each rule
> from a `ruleToEventMap` lookup. An unknown node name has no visitor, so the target field is never
> resolved (→ `Unknown Field`) and the map lookup misses (→ `null`).
>
> **The authoritative list of expression rule types** — read it off the running instance, never guess:
> ```bash
> curl.exe -s -u admin:admin \
>   "http://localhost:4502/libs/fd/af/authoring/clientlibs/guideCommonAuthoring.js" \
>   -o /tmp/rules.js
> grep -n 'ruleToEventMap' -A 120 /tmp/rules.js     # nodeName -> the display name used as eventName
> grep -o 'enter[A-Z_]*_EXPRESSION' /tmp/rules.js | sort -u   # every node with a real visitor
> ```
> | AST `nodeName` | persisted `eventName` |
> |---|---|
> | `CALC_EXPRESSION` / `CLEAR_EXPRESSION` | `Calculate` |
> | `SHOW_EXPRESSION` / `VISIBLE_EXPRESSION` | `Visibility` |
> | `ACCESS_EXPRESSION` / `DISABLE_EXPRESSION` | `Enabled` |
> | `VALIDATE_EXPRESSION` | `Validate` |
> | `FORMAT_EXPRESSION` | `Format` |
> | `OPTIONS_EXPRESSION` | `Options` |
> | `SUMMARY_EXPRESSION` | `Summary` |
> | `NAVIGATION_EXPRESSION` | `Navigation` |
> | `COMPLETION_EXPRESSION` | `Completion` |
> | `EVENT_SCRIPTS` + `"is clicked"` | `Click` |
> | `EVENT_SCRIPTS` + `"is changed"` | `Value Commit` |
> | `EVENT_SCRIPTS` + `"is initialized"` | `Initialize` |
>
> The persisted `eventName` is the **display name** from that map (so `CALC_EXPRESSION` → `"Calculate"`,
> **not** `"Value"`). There is no `VALUE_EXPRESSION`, `ENABLE_EXPRESSION`, `LABEL_EXPRESSION`,
> `ENUM_EXPRESSION`, or `REQUIRED_EXPRESSION` — all five are plausible-sounding inventions. To set
> mandatory dynamically, use a `fd:change` event script; there is no dedicated expression rule for it.
>
> 🛑 **The `grep -o 'enter[A-Z_]*_EXPRESSION'` check above only reliably validates the TOP-level rule-type
> node** (`CALC_EXPRESSION`, `SHOW_EXPRESSION`, `ACCESS_EXPRESSION`, `VALIDATE_EXPRESSION`, …) and
> statement-level action nodes (`SUBMIT_FORM`, `BLOCK_STATEMENT`, …). It does **NOT** reliably validate
> sub-expression/operand nodes used INSIDE a condition — `COMPARISON_EXPRESSION`, `BOOLEAN_BINARY_EXPRESSION`,
> `CONDITION`, `OPERATOR` are all real, grammar-valid nodes that do **not** show up as a literal
> `enterCOMPARISON_EXPRESSION`/`enterBOOLEAN_BINARY_EXPRESSION` string in `guideCommonAuthoring.js`
> (verified: 1 and 0 raw-string occurrences respectively) — they're handled by generic/shared traversal
> code, not a per-node `enter<NAME>` function. **The grep is a fast first-pass filter for catching an
> obviously-invented top-level node; it is NOT sufficient proof that a sub-expression node is valid, and
> its silence about a node is NOT proof the node is invalid either.** The only real authority for
> sub-expression shapes remains an editor-round-tripped example (see the table below) — copy one
> byte-for-byte rather than inferring a shape from what strings do or don't appear in this file.
>
> **Additional invented nodes confirmed broken this session** (each produced a rule the Rule Editor could
> not open/edit — "Unknown Field - null" or simply unresponsive — even though the JSON parsed fine and the
> field still rendered/submitted correctly at runtime):
> - `AND_EXPRESSION` as a `CONDITIONORALWAYS` choice, with each `COMPARISON_EXPRESSION` as a bare sibling
>   item (no `CONDITION` wrapper, no `OPERATOR`/`AND` node, no `nested` flag) — **use
>   `BOOLEAN_BINARY_EXPRESSION` instead**, for ANY compound-AND condition, not just inside `CALC_EXPRESSION`
>   (Visibility/Validate/Enabled conditions need the identical shape). See the verified structure in the
>   `fd:calc` reference below — `CONDITION` → `COMPARISON_EXPRESSION`, `OPERATOR` → `AND`, `CONDITION` →
>   `COMPARISON_EXPRESSION` (with `"nested":false` on this last `CONDITION`).
> - `NUMBER_LITERAL` with a bare, unquoted JSON number (`"value":0`) — **use `NUMERIC_LITERAL` with the
>   number encoded as a JSON STRING instead** (`"value":"0"`), even though the field being compared is a
>   genuine `NUMBER`-type field. `NUMBER_LITERAL` does not appear anywhere in the deployed authoring
>   clientlib.
> - `FUNCTION_CALL_STATEMENT` wrapping a `FUNCTION_CALL` as a standalone `BLOCK_STATEMENT` choice (e.g. to
>   call a custom clientlib function as its own click-handler action, before/after a built-in action like
>   `SUBMIT_FORM`) — **there is no such statement node.** `FUNCTION_CALL` only has special-cased handling
>   INSIDE a value-returning `EXPRESSION` context (a `VALIDATE_EXPRESSION` condition, a `CALC_EXPRESSION`
>   then-value, …) — it can never be a bare top-level click/statement action. If a click handler needs to
>   call a custom function AND perform a built-in action (submit, reset, …), do NOT try to represent the
>   custom-function call in the `fd:click` AST at all: keep the AST down to just the built-in action (e.g.
>   `BLOCK_STATEMENTS` with a single `SUBMIT_FORM` `BLOCK_STATEMENT`, matching the verified
>   `sports-event-registration` submit button exactly), and drive the ACTUAL runtime behavior — including
>   the custom function call — purely through `<fd:events click="[myFunction(), submitForm()]"/>`, which
>   needs no AST representation at all. The Rule Editor will show a simpler rule ("Submit the form") than
>   what actually runs, which is an acceptable, deliberate trade-off — it beats a rule nobody can open.
>
> **`script` type — expression rules take a STRING, event rules take an ARRAY:**
> ```text
> ✅ expression rule (Calculate / Validate / Visibility / Enabled / …):
>      "script":"getTotalAmount($form.a.$value\, $form.b.$value)"
>      "script":"validateEmailAddress($field.$value) == true()"
> ✅ event rule (Click / Change / Initialize — EVENT_SCRIPTS):
>      "script":["submitForm()"]
> ❌ an array on an expression rule → the summary renders null
> ```
> Verify both with one command before declaring done — it needs no deploy:
> ```bash
> # every fd:* AST: parses, real node name, correct script type
> grep -o 'fd:\(value\|validate\|visible\|enabled\|click\|change\|init\)="[^"]*"' <form>/.content.xml \
>   | sed 's/&quot;/"/g; s/\\,/,/g' | grep -o '"nodeName":"[A-Z_]*_EXPRESSION"\|"eventName":"[A-Za-z ]*"\|"script":\[\?'
> ```

**Rules for using this map:**
- ✅ rows are confirmed against working forms in this repo — copy the AST shape from the cited reference.
- ⚠️ rows are not yet demonstrated here: the **property name is authoritative**, but author the AST by
  copying the closest ✅ reference and adapting it, then **verify the rule appears in
  `guideContainer.model.json`** before declaring done (a wrong AST node also fails silently).
- Compare against the enum **code**, never the display label, in any condition literal.
- Each ⚠️ AST top-node name is the documented Core Components form; if a copied-and-adapted rule does not
  surface in the model JSON, open the rule once in the AEM Rule Editor, save, and diff the regenerated AST
  rather than hand-guessing node names.

> 🛑 **EVERY `fd:*` property on `<fd:rules>` must be an escaped-JSON value (a `[…]` array), NEVER a raw
> expression.** The Rule Editor calls `JSON.parse()` on every `fd:*` property of every `<fd:rules>` node
> when it opens. A bare string — e.g. `fd:click="submitForm()"`, `fd:validate="myFn($value)"` — makes
> `JSON.parse` throw `Uncaught SyntaxError: … is not valid JSON`, which **aborts the editor's whole
> rule-tree traversal**: the Rule Editor then shows "There are no rules on this object" for **every field
> on the form**, not just the offending one, and the console shows the SyntaxError. A single malformed
> button rule therefore hides ALL rules. The runtime mirror attribute is different: `validationExpression`
> / `visible` / `enabled` ARE plain expressions, and `<fd:events>` uses bracketed runtime expressions
> (`click="[submitForm()]"`); only the `<fd:rules>` `fd:*` AST properties are JSON.
>
> 🛑 **An embedded quote inside an `fd:*` JSON blob's `"script"` (or any string) field needs 2 backslashes
> in the XML source, not 1** — e.g. `&quot;script&quot;:&quot;field.$value != \\&quot;\\&quot;&quot;`.
> `fd:*` properties are JCR **multi-value** properties (the value is wrapped in `[…]`), and Jackrabbit's
> DocView importer applies its OWN generic backslash-unescape pass on top of XML entity decoding — the
> same mechanism this project already relies on to turn `\,` into a literal comma inside these JSON blobs.
> That pass strips ONE backslash from every `\X` sequence, no matter what `X` is. A single `\&quot;` in the
> XML source therefore survives import as a **bare, unescaped quote** — the JSON `JSON.parse()`s fine
> against the raw XML text (which is why a naive check can miss it), but the value **actually stored in the
> repository** is corrupted JSON. Symptom: opening the Rule Editor shows an empty Form Objects tree (not
> just a broken rule on one field), and the browser console shows
> `Uncaught SyntaxError: ... at position N` thrown from `_traverseAndValidateRules -> _parseEachModel ->
> JSON.parse` — because one malformed rule aborts the tree walk for the WHOLE form, same failure class as
> the bare-string mistake above. To verify BEFORE deploying, don't just XML-decode-and-JSON.parse the raw
> source — simulate the JCR unescape too: XML-entity-decode, then strip one backslash from every `\X`
> pair, THEN `JSON.parse`. If that round-trip fails, fix the backslash count (2 survives as 1, which is
> correct; 1 survives as 0, which is broken).
>
> 🛑 **Never reference a function that is not defined in the form's clientlib.** A button/validator wired to
> a non-existent function (an invented `saveDraft()`, a misspelled validator) is dead at runtime; define it
> first via `create-form-clientlib`. If you cannot produce a valid click-action AST, leave `<fd:rules>`
> valid/empty, wire `<fd:events ... click="[fn()]"/>`, and add the visible rule by round-tripping the editor.

> 🚀 **Deploying a rule-structure change to an instance that already has the form: delete the form node
> first.** A FileVault package installs in **update** mode, which does NOT purge removed child nodes
> (e.g. a deleted button) and does not reliably overwrite an existing `fd:rules` tree — so the editor keeps
> showing the stale/broken rules. Delete `/content/forms/af/{project}/{formName}` on the instance, then
> redeploy with the **full** `mvn clean install -PautoInstallSinglePackage` (a partial `-pl ui.apps` deploy
> can drop the clientlib aggregation and is unreliable), so the form is recreated fresh.
>
> **Pipeline mode (delegated by `formwright`): author the `fd:rules` XML only — do NOT run `mvn`
> here.** Deployment (and any delete-before-deploy purge of a stale form node) is handled by the
> `forgemaster` lead (AGENTS.md → "Deployment is centralized in Forgemaster"); flag in your run file that the
> form node must be purged before redeploy so Forgemaster does it. Run `mvn` here only when invoked standalone.

### Pattern 1: Show/hide a field based on another field's value

Requirement: *Show `companyName` field only when `employmentType` equals "salaried" or "self-employed".*

```xml
<companyName
  jcr:primaryType="nt:unstructured"
  sling:resourceType="core/fd/components/form/textinput/v1/textinput"
  jcr:title="Company Name"
  name="companyName">

  <fd:rules jcr:primaryType="nt:unstructured">
    <visibility jcr:primaryType="nt:unstructured">
      <condition jcr:primaryType="nt:unstructured"
        operator="OR">
        <condition1 jcr:primaryType="nt:unstructured"
          fieldName="employmentType"
          operator="EQUALS"
          value="salaried"/>
        <condition2 jcr:primaryType="nt:unstructured"
          fieldName="employmentType"
          operator="EQUALS"
          value="self-employed"/>
      </condition>
      <action>SHOW</action>
    </visibility>
  </fd:rules>

</companyName>
```

---

### Pattern 2: Make a field required based on a condition

Requirement: *Make `spouseName` required when `maritalStatus` equals "married".*

```xml
<spouseName
  jcr:primaryType="nt:unstructured"
  sling:resourceType="core/fd/components/form/textinput/v1/textinput"
  jcr:title="Spouse Name"
  name="spouseName"
  required="{Boolean}false">

  <fd:rules jcr:primaryType="nt:unstructured">
    <mandatory jcr:primaryType="nt:unstructured">
      <condition jcr:primaryType="nt:unstructured"
        fieldName="maritalStatus"
        operator="EQUALS"
        value="married"/>
      <action>SET_MANDATORY</action>
    </mandatory>
    <optional jcr:primaryType="nt:unstructured">
      <condition jcr:primaryType="nt:unstructured"
        fieldName="maritalStatus"
        operator="NOT_EQUAL"
        value="married"/>
      <action>CLEAR_MANDATORY</action>
    </optional>
  </fd:rules>

</spouseName>
```

---

### Pattern 3: Calculate a field value from other fields

Requirement: *Calculate `totalAmount` as `unitPrice` multiplied by `quantity`.*

```xml
<totalAmount
  jcr:primaryType="nt:unstructured"
  sling:resourceType="core/fd/components/form/numberinput/v1/numberinput"
  jcr:title="Total Amount"
  name="totalAmount"
  readOnly="{Boolean}true">

  <fd:rules jcr:primaryType="nt:unstructured">
    <calculate jcr:primaryType="nt:unstructured">
      <expression>var total = Number($('unitPrice').value) * Number($('quantity').value);
return isNaN(total) ? 0 : total;</expression>
      <action>SET_VALUE</action>
    </calculate>
  </fd:rules>

</totalAmount>
```

**The above is conceptual shorthand only (see the CRITICAL guard at the top of this section). The REAL,
editor-verified shape for a Calculate rule — including the cross-field case, e.g. auto-filling `managerId`
from `employeeId`'s value — is:**

```xml
<managerId
  jcr:primaryType="nt:unstructured"
  sling:resourceType="core/fd/components/form/textinput/v1/textinput"
  jcr:title="Manager Id"
  name="managerId">
  <!-- readOnly is a free choice — the editor did NOT force readOnly="{Boolean}true" here -->

  <fd:rules jcr:primaryType="nt:unstructured"
      fd:calc="[{&quot;nodeName&quot;:&quot;ROOT&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;STATEMENT&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;CALC_EXPRESSION&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;VALUE_FIELD&quot;\,&quot;value&quot;:{&quot;id&quot;:&quot;$form.employeeDetailsPanel.managerId&quot;\,&quot;type&quot;:&quot;STRING&quot;\,&quot;name&quot;:&quot;managerId&quot;}}\,{&quot;nodeName&quot;:&quot;to&quot;\,&quot;value&quot;:null}\,{&quot;nodeName&quot;:&quot;EXPRESSION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;STRING_LITERAL&quot;\,&quot;value&quot;:&quot;venuga5&quot;}}\,{&quot;nodeName&quot;:&quot;When&quot;\,&quot;value&quot;:null}\,{&quot;nodeName&quot;:&quot;CONDITIONORALWAYS&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;BOOLEAN_BINARY_EXPRESSION&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;CONDITION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;COMPARISON_EXPRESSION&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;EXPRESSION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;COMPONENT&quot;\,&quot;value&quot;:{&quot;id&quot;:&quot;$form.employeeDetailsPanel.employeeId&quot;\,&quot;type&quot;:&quot;NUMBER&quot;\,&quot;name&quot;:&quot;employeeId&quot;\,&quot;parent&quot;:&quot;$form.employeeDetailsPanel&quot;}}}\,{&quot;nodeName&quot;:&quot;OPERATOR&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;GREATER_THAN&quot;\,&quot;value&quot;:null}}\,{&quot;nodeName&quot;:&quot;EXPRESSION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;NUMERIC_LITERAL&quot;\,&quot;value&quot;:&quot;0&quot;}}]}}\,{&quot;nodeName&quot;:&quot;OPERATOR&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;AND&quot;\,&quot;value&quot;:null}}\,{&quot;nodeName&quot;:&quot;CONDITION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;COMPARISON_EXPRESSION&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;EXPRESSION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;COMPONENT&quot;\,&quot;value&quot;:{&quot;id&quot;:&quot;$form.employeeDetailsPanel.employeeId&quot;\,&quot;type&quot;:&quot;NUMBER&quot;\,&quot;name&quot;:&quot;employeeId&quot;\,&quot;parent&quot;:&quot;$form.employeeDetailsPanel&quot;}}}\,{&quot;nodeName&quot;:&quot;OPERATOR&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;LESS_THAN&quot;\,&quot;value&quot;:null}}\,{&quot;nodeName&quot;:&quot;EXPRESSION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;NUMERIC_LITERAL&quot;\,&quot;value&quot;:&quot;11&quot;}}]}\,&quot;nested&quot;:false}]}}]}}]\,&quot;isValid&quot;:true\,&quot;enabled&quot;:true\,&quot;version&quot;:1\,&quot;script&quot;:&quot;if(employeeId.$value &gt; 0 &amp;&amp; employeeId.$value &lt; 11\,'venuga5'\,$field)&quot;\,&quot;eventName&quot;:&quot;Calculate&quot;\,&quot;ruleType&quot;:&quot;&quot;\,&quot;description&quot;:&quot;&quot;}]"
      value="if(employeeId.$value &gt; 0 &amp;&amp; employeeId.$value &lt; 11,'venuga5',$field)"
      validationStatus="valid"/>
  <fd:events jcr:primaryType="nt:unstructured"/>

</managerId>
```

Notes on this REAL shape (all confirmed by reading back the live JCR node after authoring this exact rule
in the AEM Rule Editor — do not deviate without re-verifying the same way):
- `VALUE_FIELD` (not `AFCOMPONENT`) identifies the field being SET.
- The condition side references other fields with `COMPONENT` (same as every other rule type).
- The else-branch of the `if(cond, then, $field)` ternary is `$field` — the bare literal token `$field`,
  meaning "leave the current value as-is" — not a quoted string.
- `fd:events` is legitimately empty when this rule is authored/saved through the Rule Editor UI; the
  `value="<expr>"` mirror attribute alone drives the runtime reactive binding, exactly like `visible=`/
  `validationExpression=` do for other rule types — this was confirmed for an EDITOR-SAVED rule read back
  from JCR, not for a hand-authored-then-package-deployed one. If you're hand-authoring this for a pure
  package deploy (no editor round-trip), `migrate-form` SKILL.md's "Cross-field auto-fill" section found
  that a hand-authored `fd:calc` does NOT reliably compile to `model.rules.value` on package deploy alone
  — verify in `guideContainer.model.json` after deploy, and fall back to its proven `dispatchEvent`-from-
  source-fields pattern if the mirror doesn't compile.
- Numeric literals inside the AST are still JSON strings (`"value":"0"`), even though the compared field
  (`employeeId`) is a genuine `NUMBER` type field — copy this literally, don't "fix" it to a bare `0`.

---

### Pattern 4: Custom validation

Requirement: *Validate that `confirmEmail` matches `email`.*

```xml
<confirmEmail
  jcr:primaryType="nt:unstructured"
  sling:resourceType="core/fd/components/form/emailinput/v1/emailinput"
  jcr:title="Confirm Email"
  name="confirmEmail">

  <fd:rules jcr:primaryType="nt:unstructured">
    <validate jcr:primaryType="nt:unstructured">
      <expression>$('email').value === $('confirmEmail').value</expression>
      <message>Email addresses do not match</message>
      <action>VALIDATE</action>
    </validate>
  </fd:rules>

</confirmEmail>
```

---

### Pattern 5: Enable/disable a field

Requirement: *Disable `loanAmount` input if `employmentType` is "retired".*

```xml
<loanAmount
  jcr:primaryType="nt:unstructured"
  sling:resourceType="core/fd/components/form/numberinput/v1/numberinput"
  jcr:title="Loan Amount"
  name="loanAmount">

  <fd:rules jcr:primaryType="nt:unstructured">
    <enableDisable jcr:primaryType="nt:unstructured">
      <condition jcr:primaryType="nt:unstructured"
        fieldName="employmentType"
        operator="EQUALS"
        value="retired"/>
      <action>DISABLE</action>
    </enableDisable>
    <enable jcr:primaryType="nt:unstructured">
      <condition jcr:primaryType="nt:unstructured"
        fieldName="employmentType"
        operator="NOT_EQUAL"
        value="retired"/>
      <action>ENABLE</action>
    </enable>
  </fd:rules>

</loanAmount>
```

---

### Pattern 6: Show/hide an entire panel

Requirement: *Show the entire `addressPanel` only when `deliveryOption` equals "home-delivery".*

```xml
<addressPanel
  jcr:primaryType="nt:unstructured"
  sling:resourceType="core/fd/components/form/panelcontainer/v1/panelcontainer"
  jcr:title="Delivery Address"
  name="addressPanel"
  layout="responsiveGrid">

  <fd:rules jcr:primaryType="nt:unstructured">
    <visibility jcr:primaryType="nt:unstructured">
      <condition jcr:primaryType="nt:unstructured"
        fieldName="deliveryOption"
        operator="EQUALS"
        value="home-delivery"/>
      <action>SHOW</action>
    </visibility>
  </fd:rules>

  <!-- address fields go here -->

</addressPanel>
```

---

### Pattern 7: Cascade dropdown — populate options based on parent value

Requirement: *When `country` changes, populate `state` dropdown via REST service.*

```xml
<state
  jcr:primaryType="nt:unstructured"
  sling:resourceType="core/fd/components/form/dropdown/v1/dropdown"
  jcr:title="State / Province"
  name="state"
  enumNames="[]"
  enum="[]">

  <fd:rules jcr:primaryType="nt:unstructured">
    <cascade jcr:primaryType="nt:unstructured">
      <condition jcr:primaryType="nt:unstructured"
        fieldName="country"
        operator="IS_NOT_EMPTY"/>
      <serviceInvocation jcr:primaryType="nt:unstructured"
        serviceName="getStatesService"
        inputParams="[{&quot;country&quot;: $(&apos;country&apos;).value}]"
        outputParam="states"/>
      <action>SET_VALUE</action>
    </cascade>
  </fd:rules>

</state>
```

---

### Pattern 8: Set a hidden field value automatically

Requirement: *Set hidden field `submissionTimestamp` to current date/time on form load.*

```xml
<submissionTimestamp
  jcr:primaryType="nt:unstructured"
  sling:resourceType="core/fd/components/form/hiddentextinput/v1/hiddentextinput"
  jcr:title="Submission Timestamp"
  name="submissionTimestamp">

  <fd:rules jcr:primaryType="nt:unstructured">
    <initialize jcr:primaryType="nt:unstructured">
      <expression>new Date().toISOString()</expression>
      <action>SET_VALUE</action>
      <event>initialize</event>
    </initialize>
  </fd:rules>

</submissionTimestamp>
```

---

### Pattern 9: Navigate wizard to next step conditionally

Requirement: *Only allow moving to step 2 if `termsAccepted` checkbox is checked.*

```xml
<nextStepButton
  jcr:primaryType="nt:unstructured"
  sling:resourceType="core/fd/components/form/button/v1/button"
  jcr:title="Next: Employment Details"
  name="nextStepButton">

  <fd:rules jcr:primaryType="nt:unstructured">
    <click jcr:primaryType="nt:unstructured">
      <condition jcr:primaryType="nt:unstructured"
        fieldName="termsAccepted"
        operator="EQUALS"
        value="true"/>
      <targetPanel>employmentDetails</targetPanel>
      <action>NAVIGATE_TO_PANEL</action>
    </click>
  </fd:rules>

</nextStepButton>
```

---

### Pattern 10: AND condition (multiple conditions must all be true)

Requirement: *Show `guarantorDetails` panel only when `loanAmount` > 5000000 AND `yearsOfExperience` < 2.*

```xml
<guarantorDetails
  jcr:primaryType="nt:unstructured"
  sling:resourceType="core/fd/components/form/panelcontainer/v1/panelcontainer"
  jcr:title="Guarantor Details"
  name="guarantorDetails">

  <fd:rules jcr:primaryType="nt:unstructured">
    <visibility jcr:primaryType="nt:unstructured">
      <condition jcr:primaryType="nt:unstructured"
        operator="AND">
        <condition1 jcr:primaryType="nt:unstructured"
          fieldName="loanAmount"
          operator="GREATER_THAN"
          value="5000000"/>
        <condition2 jcr:primaryType="nt:unstructured"
          fieldName="yearsOfExperience"
          operator="LESS_THAN"
          value="2"/>
      </condition>
      <action>SHOW</action>
    </visibility>
  </fd:rules>

</guarantorDetails>
```

**The above is conceptual shorthand only. The REAL, verified `CONDITIONORALWAYS` shape for a compound-AND
condition — inside a `SHOW_EXPRESSION`, `ACCESS_EXPRESSION`, `VALIDATE_EXPRESSION`, or `CALC_EXPRESSION`,
it's the same node regardless of the parent rule type — is `BOOLEAN_BINARY_EXPRESSION`, NOT `AND_EXPRESSION`
(confirmed: `AND_EXPRESSION` has no visitor and produces a rule the Rule Editor cannot open/edit):**

```text
"CONDITIONORALWAYS":{"choice":{"nodeName":"BOOLEAN_BINARY_EXPRESSION","items":[
  {"nodeName":"CONDITION","choice":{"nodeName":"COMPARISON_EXPRESSION","items":[...loanAmount > 5000000...]}},
  {"nodeName":"OPERATOR","choice":{"nodeName":"AND","value":null}},
  {"nodeName":"CONDITION","choice":{"nodeName":"COMPARISON_EXPRESSION","items":[...yearsOfExperience < 2...]},"nested":false}
]}}
```

Each side of the AND is wrapped in its own `CONDITION` node (not a bare `COMPARISON_EXPRESSION` sibling),
there's an explicit `OPERATOR`→`AND` node between them, and `"nested":false` sits on the LAST `CONDITION`
item. Copy this exact shape from `managerId`'s/`managerName`'s `fd:calc` in `employee-training-request`
(editor-authored, then read back from JCR) or `approvalInformationPanel`'s `fd:visible` in that same form
(hand-fixed to this shape after `AND_EXPRESSION` proved broken) — do not hand-guess a variant.

---

## Common mistakes — never do these

```
WRONG: Using DOM methods
  document.getElementById('fieldName').style.display = 'none';

RIGHT: Use rule action HIDE on the field node.

---

WRONG: Deprecated setProperty API
  GuideBridge.setProperty('fieldName', 'visible', false);

RIGHT: Use rule action SHOW/HIDE.

---

WRONG: Visibility rule under the Foundation property name / SET_PROPERTY AST
  <fd:rules fd:visibility="[{...SET_PROPERTY...visible:BOOLEAN...}]"/>
  (Core Components ignore this — panel/field never toggles)

RIGHT: Store it as fd:visible with a SHOW_EXPRESSION AST on the fd:rules node
  <fd:rules fd:visible="[{...SHOW_EXPRESSION...Hide...}]"
            visible="collectionMethod == &quot;HomeCollection&quot;"/>
  (see the CRITICAL guard at the top of "Rule patterns")

---

WRONG: Hardcoding field paths
  condition fieldName="/content/forms/af/myForm/guideRootPanel/step1/fieldName"

RIGHT: Always use the short name only:
  condition fieldName="fieldName"

---

WRONG: JS expression using jQuery
  $('#fieldName').val()

RIGHT: Use the AEM Forms bridge expression:
  $('fieldName').value
```

---

## Quality checklist

Before delivering any rules:

- [ ] All operator strings match the reference table exactly (uppercase)
- [ ] All action strings match the reference table exactly (uppercase)
- [ ] Field names in conditions match the `name` attribute on the actual field node
- [ ] Every rule is stored under the correct `fd:*` property per the **Rule storage property map**
      (`fd:visible` / `fd:validate` / `fd:change` / `fd:click` / `fd:init` / `fd:enabled` / `fd:calc` /
      `fd:required` / `fd:label` / `fd:enum`) — escaped-JSON AST on the `<fd:rules>` node, NOT a
      `<visibility>`/`<calculate>`/`<validate>` child node and NOT a Foundation-era name
      (e.g. `fd:visibility`, `fd:enable`, `fd:mandatory`)
- [ ] Conditions compare against the enum **code**, not the display label
- [ ] Each rule was verified present in `guideContainer.model.json` after deploy (a wrong property
      name OR a wrong AST node fails silently); 0 `SET_PROPERTY`/`fd:visibility` occurrences remain
- [ ] Any embedded quote inside an `fd:*` JSON blob's string fields uses 2 backslashes in the XML source
      (`\\&quot;`), not 1 — verified by simulating the JCR DocView unescape (strip 1 backslash per `\X`)
      BEFORE `JSON.parse`, not just decoding XML entities and parsing directly
- [ ] Every `fd:calc`/`fd:enabled`/`fd:visible` rule has its plain-text mirror attribute
      (`value=`/`enabled=`/`visible=` respectively) on `<fd:rules>` — the AST alone does not drive
      runtime; the mirror is what the deployed form actually reacts to
- [ ] No DOM manipulation used
- [ ] No deprecated GuideBridge API used
- [ ] Compound AND conditions inside ANY rule type (Visibility/Enabled/Validate/Calculate) use
      `BOOLEAN_BINARY_EXPRESSION` (`CONDITION`→`COMPARISON_EXPRESSION`, `OPERATOR`→`AND`, `CONDITION`→
      `COMPARISON_EXPRESSION` with `"nested":false` on the last one) — NOT `AND_EXPRESSION`, which has no
      visitor and silently produces an unopenable rule
- [ ] Numeric literals in a condition use `NUMERIC_LITERAL` with the number as a JSON STRING
      (`"value":"0"`) — NOT `NUMBER_LITERAL` with a bare unquoted number, which does not exist in the grammar
- [ ] A click/event handler that needs to call a custom function is NEVER represented as a standalone
      `FUNCTION_CALL_STATEMENT`/`FUNCTION_CALL` in the `fd:rules` AST (no such statement node exists) —
      keep the AST to just the built-in action and drive the function call purely via `<fd:events>`
- [ ] Calculate expressions handle NaN (division by zero, empty fields)
- [ ] Validate expressions return boolean true (valid) or false (invalid)

---

## Editing or replacing an existing rule — purge the stale node (MANDATORY)

When you **change the structure of an existing rule** — rename the property (e.g. `fd:visibility` → `fd:visible`),
change the AST shape, or remove a rule — a normal redeploy is **NOT enough**. FileVault import
(`-PautoInstallSinglePackage`) updates/adds properties but **does not delete properties that exist on the
instance node but no longer exist in the package** (its mode is effectively merge-on-replace). The result is
that the **old and new rule properties coexist on the instance** (e.g. both `fd:visibility` AND `fd:visible`
on the same `<fd:rules>` node), which corrupts the rule and the Rule Editor. The local source `.content.xml`
looks correct, and `mvn` reports `BUILD SUCCESS`, so this is easy to miss.

**Fix: delete the form node on the instance, then redeploy from clean source.**

1. Confirm the source is clean first (the only `fd:visibility` left in the repo should be documentation):
   ```bash
   grep -rn "fd:visibility" ui.content/.../forms/af/{project}/{formName}/.content.xml   # expect: no matches
   ```
2. Delete the deployed form node. **Important environment facts (learned the hard way):**
   - The HTTP `DELETE` verb is blocked (returns **403**).
   - A Sling `:operation=delete` POST is also rejected (**403**) unless it carries a same-origin
     **`Referer` header** (Apache Sling Referrer Filter). PowerShell `Invoke-WebRequest` fails here even
     with the header set; **use `curl.exe`**, which works reliably:
   ```bash
   curl.exe -s -u admin:admin -o /dev/null -w "delete=%{http_code}\n" -X POST \
     -H "Referer: http://localhost:4502" \
     -F ":operation=delete" \
     "http://localhost:4502/content/forms/af/{project}/{formName}"
   # confirm it is gone (expect 404):
   curl.exe -s -u admin:admin -o /dev/null -w "exists=%{http_code}\n" \
     "http://localhost:4502/content/forms/af/{project}/{formName}.json"
   ```
3. Redeploy the full reactor (recreates the form from clean source):
   ```bash
   mvn clean install -PautoInstallSinglePackage    # plus any machine-specific deploy flags
   ```
4. Verify the stale property is gone: fetch `…/guideContainer.model.json` and confirm the rule shows the
   correct shape (e.g. `"rules":{"visible":…}`) with **0** occurrences of the old property
   (`fd:visibility` / `SET_PROPERTY`).

> This only matters for *structure/property changes* to existing rules. Adding a brand-new rule to a field
> that had none, or first-time form creation, does not need the delete step.
