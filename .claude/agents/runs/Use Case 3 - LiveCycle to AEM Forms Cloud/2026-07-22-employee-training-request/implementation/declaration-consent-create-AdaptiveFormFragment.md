# create-AdaptiveFormFragment — declaration-consent

NEW fragment, reuses the `blank-af-v2-fragment` template created for `employee-identity` (not
recreated).

## Artifacts

1. **Fragment page** — `/content/forms/af/aem-adaptive-form-agents/declaration-consent`
2. **DAM fragment asset** — `/content/dam/formsanddocuments/aem-adaptive-form-agents/declaration-consent`
   (`type="affragment"` + `affragment="1"`)
3. **Per-fragment conf context** — `/conf/forms/aem-adaptive-form-agents/declaration-consent`
4. **Filter entry** — `/conf/forms/aem-adaptive-form-agents/declaration-consent` added to filter.xml
5. **Validation clientlib** — NOT built. `AcceptedTerms` uses the built-in `required`+`mandatoryMessage`
   (no custom function); `SubmissionDate` is a plain readOnly display field (system-set by the
   CONSUMING form's rule, not the fragment). No custom-function validations exist on this fragment.

## Canonical binding shape

`$.declaration.*` (acceptedTerms, submissionDate). Remapped at embed time to `$.Declaration.*` in
both `employee-training-request` and `employee-training-request-dor` via the Fragment reference
node's `dataRef="$.Declaration"` override.

## Layout (evidence-based deviation — no header band)

Single flat panel band — `hideTitle="{Boolean}true"` on the fragment's `fragmentcontainer`, per DESI's
explicit finding from the DAM preview JPEGs: the AcceptedTerms checkbox + SubmissionDate sit directly
on the panel body with NO blue header band/caption above them (unlike every other panel in this
form). This is why NO separate CSS override was needed to suppress the header band in the form
clientlib — `hideTitle=true` structurally omits the `.cmp-container__label` element entirely.

## Fields (2)

- `AcceptedTerms` — checkbox, `required="true"`, `enum="[true]"` (single-option checkbox pattern —
  NOT a schema `const`), `mandatoryMessage="You must accept the terms to submit"`.
- `SubmissionDate` — text-input, `readOnly="{Boolean}true"`, `required="false"` — display-only; the
  consuming form's rule (create-form-rules rule #8) sets its value at submit time via the submit
  button's extended `fd:click`/`fd:events` AST (`declarationFragment.submissionDate.$value = new
  Date().toISOString()`), not a rule authored inside this fragment.

## Reused by

`employee-training-request` (interactive form) AND `employee-training-request-dor` (DoR template
content, display-only in both places).
