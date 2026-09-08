# generate-schema — vehicle-registration-form (DEFECT FIX — 2 passes)

**Phase:** 1 (schema) · **Status:** COMPLETE (pass 2) · **Type:** FIX to an existing schema (no new schema authored)

> **Pass 1 (`minLength` removal, below) did NOT fully resolve the defect.** The user redeployed and
> retested: the main form's Rule Editor "Form Objects" tree was still COMPLETELY BLANK — not even
> the schema-independent Submit/Reset buttons appeared, and the Rule Editor title read "Rule Editor -"
> with an empty object name. See "PASS 2" section below for the deeper re-audit and second fix. Pass 1
> is retained for history/traceability — its diagnosis (schema-import failure) was directionally
> correct, just incomplete (it caught only one of the non-whitelisted keywords).

## Defect being fixed

AEM Adaptive Forms Rule Editor: fields placed directly under the main
`vehicle-registration-form` guideContainer (ownerFullName, dateOfBirth, gender, contactNumber,
emailAddress, emergencyContactNo, vehicleType, makeModel, registrationYear, chassisNumber,
fuelType, rcBookNumber, insurancePolicyNumber, pucCertificateNumber) were **not enumerated
under "Form Objects"** and showed no rules, while fields inside the two embedded Adaptive Form
Fragments (`address-details-fragment`, `declaration-fragment`) enumerated and showed rules
correctly.

## Root cause

`vehicle-registration-form.schema.json` used the JSON-Schema keyword **`minLength`** on 3
properties (`ownerDetails.contactNumber`, `ownerDetails.emergencyContactNo`,
`vehicleDetails.registrationYear`). Per this project's own verified Hard Rule 7 ("the schema
must parse cleanly in AEM's model importer — not just be valid JSON", verified on
sports-event-registration-form 2026-07-06), the importer-safe keyword whitelist is: `type`,
`title`, `properties`, `required`, `enum`, `maxLength`, `minimum`/`maximum`, `pattern`,
`default`, `format` (date|email), `aem:afProperties`. `minLength` is **not** on that list.

Audited all 8 schema-backed forms in the repo:

| Schema | uses `minLength`? |
|---|---|
| college-admission-registration | no |
| doctor-appointment-registration | no |
| employee-registration-form | no |
| patient-registration-form | no |
| school-admission-registration | no |
| sports-event-registration | no |
| employee-training-request | no |
| **vehicle-registration-form** | **yes — 3 occurrences (only form in the repo)** |

When the main form's schema (`schemaType="jsonschema"`, `schemaRef=.../vehicle-registration-form.schema.json`)
fails `GuideModelImporterImpl`, the Rule Editor's form-model build fails for every component
resolved against that schema — i.e. every direct field of the main form. The two embedded
Fragments are unaffected because each fragment's own `guideContainer` has `schemaType="none"`
— their field model does not depend on the main form's schema at all, so they always enumerate
correctly. This produces exactly the reported asymmetry.

## Fix applied

Removed the 3 `minLength` entries from
`ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json/_jcr_content/renditions/original`:

- `ownerDetails.contactNumber` — removed `"minLength": 10` (kept `"maxLength": 10` and
  `"pattern": "^[0-9]{10}$"`)
- `ownerDetails.emergencyContactNo` — removed `"minLength": 10` (kept `"maxLength": 10` and
  `"pattern": "^[0-9]{10}$"`)
- `vehicleDetails.registrationYear` — removed `"minLength": 4` (kept `"maxLength": 4` and
  `"pattern": "^[0-9]{4}$"`)

No validation strength is lost: each field's `pattern` already enforces the exact digit count
(`^[0-9]{10}$`, `^[0-9]{4}$`) on its own, so `minLength` was redundant even before removal.
Nothing else in the schema was touched. `bindings.json` required no change (it carries no
`minLength` references). The form's `.content.xml` required no change — the corresponding AF
field nodes (`contactNumber`, `emergencyContactNo`, `registrationYear`) already carried only
`maxLength`/`pattern` attributes, never a `minLength` attribute.

## Verification performed (author-side, no AEM instance available here)

- [x] File re-parsed with `node -e "JSON.parse(...)"` — still valid JSON.
- [x] `grep -c "minLength"` on the file now returns `0`.
- [x] Diffed against the pre-fix version — only the 3 `minLength` lines were removed; every
      other key/value, ordering, and formatting is untouched.
- [x] `bindings.json` and the form/fragment `.content.xml` files confirmed unaffected (grepped
      for `minLength` — 0 matches in each).

## Verification still required on the deployed instance (flag to Forgemaster/Sentinel)

After Forgemaster's build+deploy:
1. Hit `GET /adobe/forms/af/customfunctions/<base64(/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form)>`
   and confirm the response is `{"customFunction":[...]}` with a **non-empty** array (was
   previously empty/broken due to the schema-import failure).
2. Grep `crx-quickstart/logs/error.log` for `GuideModelImporterImpl` / `Unable to parse JSON Schema`
   around the form's schema path — should show **no new occurrences** after redeploy.
3. Open the Rule Editor on `vehicle-registration-form` and confirm **all direct fields**
   (ownerFullName, dateOfBirth, gender, contactNumber, emailAddress, emergencyContactNo,
   vehicleType, makeModel, registrationYear, chassisNumber, fuelType, rcBookNumber,
   insurancePolicyNumber, pucCertificateNumber) now appear under "Form Objects", with their
   existing rules (dateOfBirth, contactNumber, emailAddress, emergencyContactNo,
   registrationYear) visible and not "Broken".
4. Confirm the two fragment fields (pincode, declarationDate) still enumerate correctly (no
   regression) and their `fd:validate` rules still show.

## Author-only

No `mvn` build/deploy was run. Deployment is deferred to the user / to Forgemaster's single
authoritative build+deploy.

## Files touched (pass 1)

- `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json/_jcr_content/renditions/original` (MODIFIED — 3 `minLength` keys removed)

---

## PASS 2 — deeper re-audit (2026-08-03, same day, second pass)

### New evidence

After redeploying pass 1, the user reported the main form's Rule Editor still shows a **completely
blank "Form Objects" tree** — no fields at all, not even the Submit/Reset buttons (which have no
`dataRef` and no dependency on the schema whatsoever) — and the Rule Editor title reads
`"Rule Editor -"` with an empty object name; the body says "There are no rules on this object right
now." Fragments continued to work fine.

**This is a stronger signal than pass 1's diagnosis accounted for.** If only *some* schema-bound
fields failed to resolve, the Submit/Reset buttons (schema-independent) should still have appeared.
The fact that literally *nothing* enumerates — including schema-independent components — means the
model-build call for the whole `guideContainer` throws and aborts before it finishes assembling the
JSON payload, taking every component down with it (schema-bound or not, since they're all part of
the one JSON tree the failed build never finished). This is consistent with `GuideModelImporterImpl`
still throwing on `vehicle-registration-form.schema.json` — i.e. at least one more non-whitelisted
keyword remained after pass 1 removed `minLength`.

### Re-audit method

1. Re-listed **every** JSON key used in `vehicle-registration-form.schema.json` and diffed it against
   Hard Rule 7's explicit whitelist (`type`, `title`, `properties`, `required`, `enum`, `maxLength`,
   `minimum`/`maximum`, `pattern`, `default`, `format` limited to `date`/`email`, `aem:afProperties`).
2. Cross-checked against `college-admission-registration.schema.json` — the reference schema-bound
   form in this repo whose Rule Editor pattern the fragments were themselves modeled on — which uses
   **zero** non-whitelisted keys.
3. Separately (not part of this skill, done by the invoking agent) verified: every `dataRef` in the
   form/fragment `.content.xml` matches a schema `fd:formDataRef` path 1:1 in both directions (zero
   mismatches); the form's XML is well-formed (`[xml]` parse); every `fd:validate`/`fd:click` AST in
   the form decodes (FileVault-unescaped) to valid JSON; there is no `integer`/`number` type anywhere
   in the schema (every field is `"type": "string"`, matching the reference pattern); no
   `additionalProperties`, `examples`, `const`, `oneOf`/`allOf`/`if`-`then`, external `$ref`, or
   `minItems` anywhere in the schema.

### Two more non-whitelisted keywords found

| Keyword | Occurrences | Where |
|---|---|---|
| `"enumNames"` | 3 | `ownerDetails.gender`, `vehicleDetails.vehicleType`, `vehicleDetails.fuelType` |
| `"description"` | 2 | `address` (object-level annotation), `declaration` (object-level annotation) |

Neither is in the Hard Rule 7 whitelist. `enumNames` inside the **schema** is distinct from (and
redundant with) the AF **field's own** `enum`/`enumNames` XML attributes already present on the
`gender`/`vehicleType`/`fuelType` dropdown components in the form's `.content.xml` — those render the
same human-readable labels at runtime and were **not** touched, so no user-facing labelling is lost
by removing the schema-level copy. `description` was pure documentation (never read by AF rendering
or binding) explaining the address/declaration objects are the canonical shapes the shared fragments
bind to.

### Fix applied (pass 2)

Removed all 5 occurrences from
`ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json/_jcr_content/renditions/original`:
- `ownerDetails.gender` — removed `"enumNames": ["Male", "Female", "Other"]`
- `vehicleDetails.vehicleType` — removed `"enumNames": ["Two Wheeler", "Four Wheeler", "Commercial Vehicle", "Electric Vehicle"]`
- `vehicleDetails.fuelType` — removed `"enumNames": ["Petrol", "Diesel", "CNG", "Electric", "Hybrid"]`
- `address` — removed the object-level `"description"` annotation
- `declaration` — removed the object-level `"description"` annotation

Everything else (`enum` values themselves, `title`, `pattern`, `maxLength`, `default`, `format`,
`aem:afProperties`/`fd:formDataRef`, `required`, `$schema`, `$id`) is untouched.

### Verification performed after pass 2 (author-side)

- [x] `node -e "JSON.parse(...)"` — still valid JSON.
- [x] Re-listed the full key set of the file — now contains **only**: `$schema`, `$id`, `title`,
      `type`, `properties`, `required`, `enum`, `maxLength`, `pattern`, `default`, `format`,
      `aem:afProperties`, `fd:formDataRef` (all whitelisted or property/object names) — zero
      remaining occurrences of `minLength`, `enumNames`, `description`, `const`, `oneOf`, `allOf`,
      `additionalProperties`, `examples`, `minItems`, `$ref`.
- [x] `bindings.json` and every form/fragment `.content.xml` confirmed unaffected — the schema edit
      touched only the schema's `original` rendition.

### Still-open operational checks (cannot be verified without a running/deployable instance)

These are flagged for the user / Forgemaster / Sentinel because they are outside what a content-only
audit can rule out:
1. **Stale cached model.** If AEM cached a broken model for this `$id` before either fix was
   deployed, a plain redeploy of the schema file may not invalidate that cache — a full
   package reinstall (delete + reinstall the DAM asset node, or an author instance restart) may be
   needed to force a fresh `GuideModelImporterImpl` parse.
2. **Link Externalizer 'author' domain.** This repo has a previously-diagnosed, VERIFIED, unrelated
   Rule-Editor failure mode on this local SDK (see project memory
   `rule-editor-broken-needs-externalizer-author`) where an unresolved Externalizer `author` domain
   causes the customfunctions/info.json endpoint to 500 and marks rules "Broken" — a different
   symptom (rules show Broken, fields still enumerate) from what's reported here (fields don't
   enumerate at all), but worth a quick independent check since `ui.config/.../ExternalizerImpl.cfg.json`
   already carries an env-default `localhost:4502` fallback — confirm it is actually being picked up
   on this instance.
3. **error.log signature.** After redeploying pass 2, grep `crx-quickstart/logs/error.log` for
   `GuideModelImporterImpl` around the time the Rule Editor is opened on `vehicle-registration-form`.
   If the exact same `Unable to parse JSON Schema` signature still appears, that conclusively proves
   the importer is still choking on something in this schema and a further, more targeted schema
   audit is needed (at that point, the fastest path is bisection: temporarily strip the schema to the
   flat set of leaf fields with only `type`/`title`/`aem:afProperties` and reintroduce
   `pattern`/`maxLength`/`enum`/`format`/`default` one group at a time, redeploying between each, to
   catch anything the strings-in-source-comparison approach used here could not surface without a
   live importer to test against).

## Files touched

- `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json/_jcr_content/renditions/original`
  — pass 1: 3 `minLength` keys removed. Pass 2: 3 `enumNames` keys + 2 `description` keys removed.
  No other file touched in either pass.
