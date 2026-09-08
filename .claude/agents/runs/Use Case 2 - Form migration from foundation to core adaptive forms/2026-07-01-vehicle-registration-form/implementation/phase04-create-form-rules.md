# Phase 4 — create-form-rules — vehicle-registration-form

Agent: create-form-rules (IMPL / blockwright, phase 4)
Date: 2026-07-01
Mode: AUTHOR ONLY — no `mvn` build/deploy run (deployment centralized in Auditon).

## Target artifact
`ui.content/src/main/content/jcr_root/content/forms/af/{appFolder}/vehicle-registration-form/.content.xml`

## Scope
Field-level **VALIDATION** rules only. No show/hide, no calculate, no set-value, no cascade,
no enable/disable. All rules stored as escaped-JSON `fd:validate` AST attributes on the field's
`<fd:rules>` node (kyc-form / account-opening gold-model style) — never `<validate>` child nodes.

## State found / result
The validation rules for this form were authored during Phase 3 (create-adaptive-form scaffolds a
form together with its per-form validation clientlib + the matching `fd:validate` rule ASTs). Phase 4
verified they are present on the correct real field names, are well-formed escaped-JSON ASTs, back the
Phase-9 clientlib functions by their exact names, and honor the required-state. All checks passed; no
edit to the form XML was required (editing a working, validated AST would only risk corrupting it).

## Fields that received `fd:validate` rules (6) — verified

| Field name        | Panel                | Component    | Validation enforced (AST `script` + field `validationExpression` mirror) | Clientlib fn(s) |
|-------------------|----------------------|--------------|--------------------------------------------------------------------------|-----------------|
| `dateOfBirth`     | ownerDetailsPanel    | datepicker   | valid date DD/MM/YYYY **AND** not in the future                          | `validateDateDDMMYYYY`, `validateDobNotFuture` |
| `mobileNumber`    | ownerDetailsPanel    | telephoneinput | exactly 10 digits (digits only)                                        | `validateMobile10Digits` |
| `emailAddress`    | ownerDetailsPanel    | emailinput   | valid email **when provided** (blank passes — field is optional)          | `validateEmailWhenPresent` |
| `pinCode`         | ownerDetailsPanel    | numberinput  | exactly 6 digits (digits only)                                           | `validatePin6Digits` |
| `policyValidTill` | insuranceDetailsPanel| datepicker   | valid date DD/MM/YYYY                                                     | `validateDateDDMMYYYY` |
| `declarationDate` | declarationPanel     | datepicker   | valid date DD/MM/YYYY                                                     | `validateDateDDMMYYYY` |

- Each rule: `<fd:rules fd:validate="[…escaped JSON VALIDATE_EXPRESSION AST…]" validationStatus="valid"/>`
  with the sibling empty `<fd:events jcr:primaryType="nt:unstructured"/>` node.
- `eventName":"Validate"`, `VALIDATE_EXPRESSION` top node, expression ends `== true()` — per the SKILL
  Rule storage property map (`fd:validate` row).
- The `dateOfBirth` AST `script` and the field-level `validationExpression` both chain the two functions
  with `&&`: `validateDateDDMMYYYY($field.$value) == true() && validateDobNotFuture($field.$value) == true()`,
  so the "not in the future" check is enforced at runtime.

## Required-field enforcement (requirement #7) — verified, unchanged
- `required="true"` on **23** fields: fullName, dateOfBirth, gender, mobileNumber, address, city, state,
  pinCode, vehicleType, manufacturer, model, registrationNumber, chassisNumber, engineNumber, fuelType,
  color, manufacturingYear, insuranceCompany, policyNumber, policyValidTill, place, declarationDate,
  signatureFullName.
- `required="false"` on exactly **2** fields: `emailAddress`, `documentChecklist` (correctly left optional).
- `declarationText` is display-only (plain-text) and carries no `required` attribute.
- No required flags were added or removed.

## AST style confirmation (no `<validate>` children, JSON well-formed)
Every `fd:*` property value on every `<fd:rules>` node was extracted and passed through `JSON.parse`
(after XML-entity + AEM backslash-comma unescaping). Result: **7 / 7 parse OK, 0 FAIL** —
6 × `fd:validate` (Validate) + 1 × `fd:click` (submitButton, Click, `["submitForm()"]`, from Phase 3).
No `fd:*` property holds a bare/raw expression, so the Rule Editor's `JSON.parse` of the rule tree will
not throw — all rules remain visible in the editor.

## Grep / structural verification
- `fd:validate=` occurrences: **6** (the 6 fields above).
- `fd:visibility`, `SET_PROPERTY`, `<visibility>` child, `<validate>` child: **0** (no forbidden Foundation
  or ignored-storage shapes; no show/hide/calculate/set-value/cascade authored).
- Total `fd:*` rule properties on `<fd:rules>` nodes: **7** (6 validate + 1 click).
- All five referenced clientlib functions are defined in the Phase-9 clientlib:
  `ui.apps/.../apps/clientlibs/vehicle-registration-form-clientlib/js/functions.js`
  (validateMobile10Digits, validatePin6Digits, validateEmailWhenPresent, validateDateDDMMYYYY,
  validateDobNotFuture).

## Notes for Auditon (deployment)
- This is a **first-time authoring** of these rules (no prior deployed rule structure being changed),
  so the delete-before-deploy stale-node purge is **not required** for this phase.
- After the centralized `mvn clean install -PautoInstallSinglePackage`, verify on the instance that
  `…/guideContainer.model.json` for each of the 6 fields shows its validation (0 `SET_PROPERTY` /
  `fd:visibility` occurrences) and that the Rule Editor opens without a `JSON.parse` SyntaxError.

## Handoff
```yaml
agent: create-form-rules
phase: 4
status: PASSED
artifacts:
  - updated_form_xml: "ui.content/src/main/content/jcr_root/content/forms/af/{appFolder}/vehicle-registration-form/.content.xml"
rules_applied:
  - field: "dateOfBirth"
    type: "validate (DD/MM/YYYY AND not-future)"
  - field: "mobileNumber"
    type: "validate (10 digits)"
  - field: "emailAddress"
    type: "validate (email when present; optional)"
  - field: "pinCode"
    type: "validate (6 digits)"
  - field: "policyValidTill"
    type: "validate (DD/MM/YYYY)"
  - field: "declarationDate"
    type: "validate (DD/MM/YYYY)"
required_fields: 23
optional_fields: ["emailAddress", "documentChecklist"]
gate_result: PASS
```
