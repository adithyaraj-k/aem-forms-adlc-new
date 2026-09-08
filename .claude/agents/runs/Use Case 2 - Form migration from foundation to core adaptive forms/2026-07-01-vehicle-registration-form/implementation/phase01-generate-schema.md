# Phase 1 — generate-schema · Vehicle Registration Form

**Run:** 2026-07-01-vehicle-registration-form
**Skill:** generate-schema
**Lead:** Blockwright (IMPL build) · Phase 1
**Status:** PASSED
**Deployed:** NO (author-only; deployment centralized in Auditon)

## Inputs consumed
- `design/DESI-design-form-components.yaml` (authoritative field/enum/binding spec)
- `plan/PLAN-architect-form-solution.yaml` (data_backing = schema, NO FDM)
- `.aem-forms-config.yaml` (project tokens, schemaContentRoot)

## Decision
- **Representation:** JSON Schema (NOT XSD)
- **Data backing:** JSON Schema only — **NO FDM**, no data source, no workflow (screenshot-only input)
- **Node type:** `dam:Asset` (`type=lcResource`) FileVault structure — NOT a bare nt:file

## Artifacts authored on disk
1. Schema asset (dam:Asset folder)
   `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json/`
   - `.content.xml` — dam:Asset + jcr:content(type=lcResource) + metadata(dc:format=application/json)
   - `_jcr_content/renditions/original` — the JSON Schema body (draft-07)
   - `_jcr_content/renditions/original.dir/.content.xml` — nt:file, jcr:mimeType=application/schema+json
2. Bindings manifest (build helper)
   `ui.content/.../schema/vehicle-registration-form.bindings.json`
3. Filter coverage (no new root needed)
   The schema path is already covered by the existing broad root in
   `ui.content/src/main/content/META-INF/vault/filter.xml`:
   `<filter root="/content/dam/formsanddocuments" mode="update"/>` (line 23).
   No separate `/content/dam/formsanddocuments/schema` root was added — the parent root covers it.
4. Folder lcFolder marker
   `ui.content/.../schema/.content.xml` — already present and correct (sling:Folder, type=lcFolder, titled jcr:content). Not modified.

## JCR path
`/content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json`

## Structure verified (node validation)
- Top-level section objects: **5** — owner, vehicle, insurance, documents, declaration
- Value-bound leaf properties: **25**
- Every leaf carries `aem:afProperties.fd:formDataRef` (0 missing)
- `documents.documentChecklist` = **array-of-enum** (`type: array`, `items.enum` = 6 values) → checkboxgroup binding, NOT a scalar
- `owner.state` enum = 37 codes
- `vehicle.manufacturingYear` enum = 31 entries (2026..1996)
- Bindings manifest: 25 entries, `fdmEnabled: false`
- Schema JSON and bindings JSON both parse cleanly

## Field inventory (25 value-bound)
- **owner (9):** fullName(req, maxLength 100), dateOfBirth(req, date), gender(req, enum male/female/other), mobileNumber(req, ^[0-9]{10}$), emailAddress(**optional**, email), address(req), city(req), state(req, 37-enum), pinCode(req, ^[0-9]{6}$, string to keep leading zeros)
- **vehicle (9):** vehicleType(req, 6-enum), manufacturer(req), model(req), registrationNumber(req), chassisNumber(req), engineNumber(req), fuelType(req, 7-enum), color(req), manufacturingYear(req, 31-enum)
- **insurance (3):** insuranceCompany(req), policyNumber(req), policyValidTill(req, date)
- **documents (1):** documentChecklist(**optional**, array-of-enum, 6 options)
- **declaration (3):** place(req), declarationDate(req, date), signatureFullName(req)

`declarationText` is static display content — intentionally NOT a schema property.

## Quality gate (self-check vs SKILL hard rules)
- [x] Location under schemaContentRoot `/content/dam/formsanddocuments/schema`
- [x] Filename `vehicle-registration-form.schema.json` (mandatory suffix)
- [x] Emitted as dam:Asset (type=lcResource) with original rendition — not nt:file
- [x] bindings.json emitted alongside
- [x] Filter root covers the schema path
- [x] schema folder has its own lcFolder .content.xml
- [x] Every leaf has fd:formDataRef; camelCase names; descriptive object names
- [x] Required arrays set per object; constraints (pattern/enum/format/maxLength) carried
- [x] documentChecklist is array-of-enum (checkbox-group)
- [x] NO FDM; NO deploy performed

## Next
Phase 3 (`create-adaptive-form`) binds the form's guideContainer to this schema
(`schemaRef = /content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json`)
and wires each field via the JSONPaths in the bindings manifest.
