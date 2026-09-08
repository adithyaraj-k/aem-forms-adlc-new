# create-form-rules — vehicle-registration-form (DEFECT FIX — de-fragment pass, PASS 3)

**Phase:** 4 (rules) · **Status:** COMPLETE

## Task

Re-author, directly on the now-inline `pincode` and `declarationDate` fields (see
`create-adaptive-form.md` for the field-inlining), the two `fd:validate` Rule-Editor rules
that most recently lived on the shared fragments themselves (added there during the
2026-07-28 "RE-EMBED" pass: `address-details-fragment`'s `pincode` and
`declaration-fragment`'s `declarationDate`), retargeting every AST `id` / `displayPath` /
`parent` reference from the fragment's internal panel names to this form's own panel
names.

## Rule 1 — `pincode` (exact 6-digit) validation

**Function:** `validateExactSixDigits($field.$value) == true()` (already defined in
`vehicle-registration-form-clientlib/js/functions.js` — no clientlib change needed).

**Retargeting applied** (fragment AST → this form's AST):

| AST field | Fragment value (source) | This form's new value |
|---|---|---|
| `AFCOMPONENT.value.id` | `$form.addressDetailsPanel.pincode` | `$form.addressSection.pincode` |
| `COMPONENT.value.id` | `$form.addressDetailsPanel.pincode` | `$form.addressSection.pincode` |
| `COMPONENT.value.displayPath` | `FORM/Address Details/Pincode / ZIP Code/` | `FORM/3. ADDRESS DETAILS/Pincode / ZIP Code/` |
| `COMPONENT.value.parent` | `$form.addressDetailsPanel` | `$form.addressSection` |

Everything else in the AST (`nodeName` tree shape — `ROOT`→`STATEMENT`→
`VALIDATE_EXPRESSION`→`AFCOMPONENT`+`CONDITION`→`COMPARISON_EXPRESSION`→`FUNCTION_CALL`
== `BOOLEAN_LITERAL True`, `eventName:"Validate"`, `script`,
`communicationComposerRuleType:"server"`) is byte-identical to the fragment's own rule —
only the 3 identifying strings above changed, because `pincode` now lives one level
shallower (`addressSection` is the field's direct parent, not `addressSection >
addressDetailsFragment(fragment) > addressDetailsPanel`).

Native `pattern="^[0-9]{6}$"` + `validatePatternMessage` + `maxLength="{Long}6"` stay on
the field as the first line of defence (unchanged from the fragment's own field
attributes); the `fd:validate` AST is what makes the rule visible/editable in the Rule
Editor.

## Rule 2 — `declarationDate` (not-future) validation

**Function:** `validateNotFutureDate($field.$value) == true()` (already defined in
`vehicle-registration-form-clientlib/js/functions.js` — no clientlib change needed).

**Retargeting applied:**

| AST field | Fragment value (source) | This form's new value |
|---|---|---|
| `AFCOMPONENT.value.id` | `$form.declarationPanel.declarationDate` | `$form.declarationSection.declarationDate` |
| `COMPONENT.value.id` | `$form.declarationPanel.declarationDate` | `$form.declarationSection.declarationDate` |
| `COMPONENT.value.displayPath` | `FORM/Declaration/Date/` | `FORM/5. DECLARATION/Date/` |
| `COMPONENT.value.parent` | `$form.declarationPanel` | `$form.declarationSection` |

Same rule as above applies: AST tree shape is otherwise byte-identical to the fragment's
own `declarationDate` rule.

## Storage shape verification (per this skill's Rule storage property map)

Both rules are stored correctly per the project's verified rule-storage rules:
- Property name on `<fd:rules>`: **`fd:validate`** (not a `<validate>` child node).
- AST top node: `ROOT`→`STATEMENT`→**`VALIDATE_EXPRESSION`** (not `SET_PROPERTY`).
- `eventName:"Validate"`.
- Mirror runtime attribute on the field itself: `validationExpression="…== true()"` +
  `validateExpMessage` (plain expression, NOT JSON — correctly NOT escaped).
- Sibling empty `<fd:events jcr:primaryType="nt:unstructured"/>` node present on both
  fields (validate rules do not need `fd:events` populated — runtime validation comes
  from `validationExpression`; the AST is editor-visibility only).
- Every `fd:*` property value is the full escaped-JSON array (`[{…}]`), never a bare
  string — confirmed by construction (built by editing the existing verified-working
  `dateOfBirth`/`contactNumber`/`registrationYear` rule ASTs already in this form, only
  retargeting the 3 identifying strings per rule, never freehand-authoring a new AST
  shape).

## Not touched

- `address-details-fragment/.content.xml`, `declaration-fragment/.content.xml` — their
  OWN `fd:validate` rules on `pincode`/`declarationDate` (added during the 2026-07-28
  RE-EMBED pass) are untouched; they still serve the 5 sibling forms that embed these
  fragments by reference.
- `dateOfBirth`, `contactNumber`, `emailAddress`, `emergencyContactNo`, `registrationYear`
  rules — pre-existing, already inline, unaffected by this pass.
- Submit/Reset button rules — unaffected.

## Verification performed (author-side, no running AEM instance here)

- [x] `[xml]` parse of the edited form — well-formed.
- [x] Manually diffed the new pincode/declarationDate ASTs against the pre-existing
      dateOfBirth/contactNumber ASTs in the same file — identical tree shape, only the 2
      leaf strings + `functionName`/`displayName` differ per rule (as expected — copied
      the verified working pattern rather than freehand-authoring).
- [x] Confirmed no bare-string `fd:*` property was introduced (every value starts with
      `[{` and is properly `&quot;`/`\,`-escaped, matching the file's existing convention).

## Redeploy note (Rule Editor structural-change caveat)

Per this skill's own "Editing or replacing an existing rule — purge the stale node"
guidance: this pass changed the STRUCTURE of the form (removed 2 fragment nodes, added 6
field nodes with new rules) at a path that is under the form's own filter root
(`mode`-unspecified → default `replace`), so a full `mvn clean install
-PautoInstallSinglePackage` redeploy should cleanly resync the `guideContainer` subtree.
If the Rule Editor still shows stale/duplicate rule state after redeploy (e.g. because a
previous partial deploy left `update`-mode residue), delete
`/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form` on the instance via
the documented `curl.exe` `:operation=delete` + `Referer` header recipe, then redeploy
fresh — flagged to Forgemaster/Sentinel, not performed here (author-only, no instance
available).

## Author-only

No `mvn` build/deploy was run. Deployment is deferred to the user / to Forgemaster's
single authoritative build+deploy step.
