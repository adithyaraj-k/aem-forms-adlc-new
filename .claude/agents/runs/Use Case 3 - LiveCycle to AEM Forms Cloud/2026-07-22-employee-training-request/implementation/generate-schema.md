# generate-schema — employee-training-request

Source: `employeeTrainingRequest.xsd` (legacy LiveCycle schema, Data/employeeTrainingRequest.xsd)
Output representation: **JSON Schema** (per Planwright's data-backing decision — schema only, no FDM)

## Output location (all 5 hard rules honored)

- `ui.content/.../jcr_root/content/dam/formsanddocuments/schema/employee-training-request.schema.json/`
  - `.content.xml` (`dam:Asset`, `jcr:content type="lcResource"`)
  - `_jcr_content/renditions/original` (schema body)
  - `_jcr_content/renditions/original.dir/.content.xml` (`nt:file`, `jcr:mimeType="application/schema+json"`)
- `ui.content/.../jcr_root/content/dam/formsanddocuments/schema/employee-training-request.bindings.json`
  (field → `fd:formDataRef` manifest)
- Schema **folder** `.content.xml` (`sling:Folder`, `type=lcFolder`) already existed in this project
  (confirmed present, not recreated).
- Filter root `/content/dam/formsanddocuments/schema` (mode="merge") already covers this path — no
  filter.xml change needed.

## Deviation from the skill's default camelCase convention (intentional, documented)

Per this delivery's explicit build instructions, property names PRESERVE the legacy XSD's exact
PascalCase node names (`EmployeeDetails.EmployeeId`, `TrainingRequest.RequestType`, etc.) rather than
being renamed to camelCase — this is required so the downstream cloud Workflow / DoR generation
(which Groundsmith builds later) can reference the exact same node names the legacy
Main.process/AssembleManagerConformation.process used, preserving cross-system compatibility for this
migration. This is a deliberate, recorded deviation from generate-schema's normal camelCase rule.

## Structure

Top-level object with 6 nested objects mirroring the XSD: `EmployeeDetails`, `TrainingRequest`,
`Justification`, `Attachments`, `ApprovalInfo`, `Declaration`. `GeneratedDocuments` from the XSD was
DROPPED (not used in the AF/DoR field inventory per the DESI component-design-spec.yaml).

## Importer-safety (critical rule 3d / Hard rule 7)

- Only `type`, `title`, `properties`, `required`, `enum`, `maxLength`, `minimum`, `pattern`, `default`,
  `aem:afProperties` used.
- `Declaration.AcceptedTerms` (must-be-true consent boolean) uses `"enum": [true]`, **NOT** `const` —
  per this project's verified importer-safety rule.
- No `oneOf`/`allOf`/`if-then`/external `$ref` used anywhere.

## Required fields (per DESI component-design-spec.yaml)

- `EmployeeDetails`: EmployeeId, EmployeeName, Email, Department, ManagerId, ManagerName, Location
  required; BusinessUnit optional.
- `TrainingRequest`: RequestId, RequestType, ProviderName, CourseName, StartDate, EndDate,
  TrainingMode, Amount, Currency required; Location optional.
- `Justification`: all 3 fields required.
- `Attachments`: all 3 fields optional.
- `ApprovalInfo`: all 4 fields optional (workflow/system-driven).
- `Declaration`: AcceptedTerms required (`enum:[true]`); SubmissionDate optional (system-set).

## FDM-ready bindings

Every leaf carries `aem:afProperties.fd:formDataRef` = `$.{Object}.{Field}` (e.g.
`$.EmployeeDetails.EmployeeId`), even though FDM itself is NOT used for this form (Planwright's
explicit decision — schema-only). The companion `employee-training-request.bindings.json` manifest
lists every field → path for `create-adaptive-form` to consume without re-deriving.

## Verification required post-deploy

- Confirm the schema imports cleanly in AEM's model importer (customfunctions endpoint returns a
  NON-empty array once rules are wired) — grep `error.log` for `GuideModelImporterImpl` if not.
