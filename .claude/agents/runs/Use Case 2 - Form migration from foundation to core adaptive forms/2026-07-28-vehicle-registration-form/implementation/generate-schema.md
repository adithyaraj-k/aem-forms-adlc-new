# generate-schema — vehicle-registration-form

**Phase:** 1 · **Status:** COMPLETE

## Output
- Schema (dam:Asset, type=lcResource): `content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json`
  - `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json/.content.xml`
  - `.../vehicle-registration-form.schema.json/_jcr_content/renditions/original` (schema body)
  - `.../vehicle-registration-form.schema.json/_jcr_content/renditions/original.dir/.content.xml`
- Bindings manifest: `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/schema/vehicle-registration-form.bindings.json`

## Design
Root object groups 16 form-specific fields under 3 nested objects (`ownerDetails`,
`vehicleDetails`, `documentInsuranceDetails`) plus 2 **canonical shared objects** that match
the two REUSED Adaptive Form Fragments' existing dataRef namespaces exactly:
- `address` → `$.address.{addressLine,city,state,pincode}` (consumed by `address-details-fragment`)
- `declaration` → `$.declaration.{submittedBy,date}` (consumed by `declaration-fragment`)

This directly implements DESI GAP-01 (schema must nest address/declaration as objects, not
flat, to match the reused fragments' canonical shape).

Every leaf property carries `aem:afProperties.fd:formDataRef` matching its `dataRef`. No
`const` keyword used anywhere (importer-safe per generate-schema Hard rule 7 / rule 3d).
`vehicleType` carries `"default": "electric_vehicle"`.

## Filter coverage
`/content/dam/formsanddocuments/schema` is already a `mode="merge"` filter root in
`ui.content/src/main/content/META-INF/vault/filter.xml` — no new filter entry needed.

## Verification note for Forgemaster/Sentinel
After deploy, hit `/adobe/forms/af/customfunctions/<base64(formPath)>` and confirm a
NON-empty `customFunction` array (rule 3d) — this schema uses only importer-safe keywords
(`type`, `title`, `properties`, `required`, `enum`, `maxLength`, `minLength`, `pattern`,
`default`, `aem:afProperties`).
