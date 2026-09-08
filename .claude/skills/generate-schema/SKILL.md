---
name: generate-schema
description: >
  Generates the data schema that backs an AEM Adaptive Form on AEM as a Cloud Service.
  Accepts five input types — a natural-language description, an existing Adaptive Form
  XML / JCR tree, a REST API / Swagger / OpenAPI spec, a Figma / wireframe / screenshot,
  or a PUBLIC WEBPAGE URL (fetch the page, isolate the embedded form, reverse-engineer the
  schema from its discovered fields — for a URL-replica delivery).
  Emits the schema in the representation the developer requests (JSON Schema or XSD),
  and always produces FDM-ready bindings (fd:formDataRef paths) so the schema drops
  straight into the create-adaptive-form skill.
version: 1.0.0
ide:
  cursor: .cursor/skills/generate-schema/
  github-copilot: .github/skills/generate-schema/
  claude-code: .claude/skills/generate-schema/
  vscode: .vscode/skills/generate-schema/
---

# Skill: generate-schema

## Role

You are an expert AEM Adaptive Forms data-modelling specialist for AEM as a Cloud Service.
You produce the **schema** that an Adaptive Form binds to — never the form UI itself
(that is the job of `create-adaptive-form`). You always generate FDM-ready binding paths,
keep field names `camelCase`, and produce schemas that are valid, well-typed, and
internationalisation-friendly.

Before generating any file, read `.aem-forms-config.yaml` at the project root to load
project-specific settings (`project`, `package`, `fdmEnabled`, `fdmRoot`,
`formsContentRoot`, `schemaContentRoot`). If the file is missing, ask the developer to
confirm the values. Conventional `schemaContentRoot` (standard AEM Forms path):
`/content/dam/formsanddocuments/schema`.

This skill is the **upstream** of `create-adaptive-form`: it produces the data model;
that skill builds the form bound to it.

---

## Hard rules — never violate (read first)

These are the constraints most often gotten wrong. A schema that breaks any of these
is broken even if the JSON itself is valid — it will not load in AEM's schema picker.

1. **Location** — the schema MUST be written under `schemaContentRoot`
   (`/content/dam/formsanddocuments/schema`). NEVER write it under
   `/content/forms/schemas`, `/content/forms/af`, or any other path. There is no
   "simple file" shortcut — that path does not exist for schemas.
2. **Filename** — the asset folder MUST be `{schemaName}.schema.json` for JSON Schema
   (or `{schemaName}.xsd` for XSD). NEVER name it `{schemaName}.json`,
   `{schemaName}/schema.json`, or `{schemaName}/{schemaName}.json`. The `.schema.json`
   suffix is mandatory.
3. **Node type** — the schema MUST be emitted as the FileVault **`dam:Asset`** structure
   (folder + `.content.xml` with `type="lcResource"` + `_jcr_content/renditions/original`).
   NEVER write the schema body as a bare `nt:file`. A bare file does NOT appear in the
   Adaptive Form data-model picker, so the schema is effectively invisible to authors.
4. **Bindings manifest** — always emit `{schemaName}.bindings.json` alongside the asset.
5. **Filter coverage** — ensure `/content/dam/formsanddocuments/schema` is a filter root
   in `ui.content/.../META-INF/vault/filter.xml`.
6. **Parent folder must be a Forms folder** — the `schema` folder itself MUST have its own
   `.content.xml` defining it as `jcr:primaryType="sling:Folder"`, `type="lcFolder"`,
   `lcFolder="{Long}0"`, with a `jcr:content` (nt:unstructured) carrying `jcr:title`.
   If it does not, FileVault auto-creates it as a bare `nt:folder` and the folder is
   INVISIBLE in the Forms & Documents console (`/aem/forms.html/content/dam/formsanddocuments`)
   — even though the assets inside still deploy and still appear in the schema picker.
   Always write/verify `{schemaContentRoot}/.content.xml` (see "Output files" below).
7. **The schema MUST parse cleanly in AEM's model importer — not just be valid JSON.**
   ⚠️ VERIFIED (sports-event-registration-form, 2026-07-06): a schema that is perfectly valid
   JSON can still FAIL AEM's `GuideModelImporterImpl`. When a schema-backed form's schema fails
   to import, **the Rule Editor marks EVERY custom-function rule "Broken"** — because the
   customfunctions endpoint (`GET /adobe/forms/af/customfunctions/<base64(formPath)>`) builds the
   form MODEL to extract functions, the model build needs the schema, the schema parse throws, and
   the servlet returns `{"customFunction":[]}`. (Runtime form-fill can still work — this breaks
   AUTHORING, not the rendered form — so it is easy to miss.) Diagnosis signature in
   `crx-quickstart/logs/error.log`:
   `GuideModelImporterImpl  Unable to parse JSON Schema` →
   `AdaptiveFormCustomFunctionProviderServlet  Error getting form model for function extraction`.
   To stay importer-safe, emit only the JSON-Schema constructs AEM's importer supports:
   - Prefer `enum` for fixed value sets. **Avoid `const`** (draft-2019 keyword) — use
     `"enum": [true]` instead of `"const": true` for a required-true consent boolean.
     Bare `type: boolean` for a checkbox is fine; the "must be checked" rule belongs in a
     Validate rule / `required`, not `const`.
   - Keep `type`, `title`, `properties`, `required`, `enum`, `maxLength`, `minimum`/`maximum`,
     `pattern`, `default`, and `aem:afProperties`. Treat any other keyword (`const`, `oneOf`,
     `allOf`, `if`/`then`, `$ref` to external files, `format` beyond `date`/`email`) as
     suspect and confirm it imports.
   - **VERIFY after deploy (mandatory for schema-backed forms):** hit the customfunctions
     endpoint for the form and confirm a NON-empty `customFunction` array; if empty, grep the
     error.log for the signature above and fix the schema — the clientlib is NOT the cause.

> Anti-pattern that has happened before: writing
> `/content/forms/schemas/studentDetails/studentDetails.json` as a plain file. This is
> wrong on rules 1, 2, and 3 simultaneously. If you find yourself about to write a
> `.json` file outside `/content/dam/formsanddocuments/schema`, STOP — you are not using
> this skill's output contract.

Echo the resolved location and filename back to the developer (Step 2 plan) and
self-check against these five rules before writing any file.

---

## Trigger

This skill activates when the developer asks to:
- Generate / create a schema or data model for a form
- Turn an API spec (Swagger / OpenAPI) into a form schema
- Reverse-engineer a schema from an existing Adaptive Form
- Produce a JSON Schema or XSD for a form use case
- Convert a wireframe / Figma / screenshot into a form data model
- Reverse-engineer a schema from a form on a PUBLIC WEBPAGE URL (URL-replica delivery)

---

## Step 1 — Detect the input type

The developer will provide ONE of the following. Auto-detect which, and state your
detection back to them.

| Input provided | How to detect | How to process |
|---|---|---|
| **Natural language** | Free-text description of a form / use case | Extract entities → fields → types (see entity extraction rules) |
| **Existing AF XML / JCR** | `.content.xml`, `sling:resourceType="core/fd/components/form/..."` nodes | Walk the node tree, map each field component back to a schema property (reverse map of the field-type table) |
| **REST / Swagger / OpenAPI** | `openapi:`, `swagger:`, `paths:`, `components.schemas`, or a `.yaml`/`.json` spec | Pull the request/response schema for the relevant operation; flatten `$ref`s |
| **Figma / wireframe / screenshot** | Image file or Figma export / link | Infer field labels and types from visual layout; ask the developer to confirm ambiguous types |
| **Public webpage URL** | An `http(s)` link to a page that has a form embedded on it | **WebFetch the page**, isolate the `<form>` (ignore the page chrome), and reverse-map each control (label / input type / name / required / options / placeholder / validation) through the field-type table to schema properties — for a URL-replica delivery. If discovery already ran, prefer its captured field inventory. For a JS-rendered form WebFetch can't see, state the limitation and fall back to user-supplied fields |

If the input is ambiguous (e.g. a screenshot with unlabelled fields), list your
assumptions and ask the developer to confirm before generating.

---

## Step 2 — Confirm output representation (always ask if unstated)

The developer must specify the target representation. If they did not, ask:

```
Output format : JSON Schema | XSD
```

Then echo the full plan back before generating:

```
Schema name   : {schemaName}
Source input  : {natural language | AF XML | OpenAPI spec | Figma}
Representation: {JSON Schema | XSD}
Root object   : {rootObjectName}
Fields        : {list every field with type, required/optional, and constraints}
FDM bindings  : always generated (fd:formDataRef paths)
Output path   : {fdmRoot}/{schemaName}/  and  {schemaContentRoot}/{schemaName}.schema.json (JSON Schema) | {schemaContentRoot}/{schemaName}.xsd (XSD)
```

Wait for confirmation or correction before writing files.

---

## Field type mapping (the canonical table)

This is the single source of truth. Map every field through it. The right two columns
are the reverse map used when reading an existing Adaptive Form.

| Logical field    | JSON Schema type                               | XSD type                          | AF component (reverse map)                          |
|------------------|------------------------------------------------|-----------------------------------|-----------------------------------------------------|
| Single-line text | `"type": "string"`                             | `xs:string`                       | `textinput`                                         |
| Multi-line text  | `"type": "string"`                             | `xs:string`                       | `textarea`                                          |
| Number (decimal) | `"type": "number"`                             | `xs:decimal`                      | `numberinput`                                       |
| Integer          | `"type": "integer"`                            | `xs:integer`                      | `numberinput` (step=1)                              |
| Email            | `"type": "string", "format": "email"`          | `xs:string` + email pattern       | `emailinput`                                        |
| Phone / tel      | `"type": "string"`                             | `xs:string` + phone pattern       | `telephoneinput`                                    |
| Date             | `"type": "string", "format": "date"`           | `xs:date`                         | `datepicker`                                        |
| Date-time        | `"type": "string", "format": "date-time"`      | `xs:dateTime`                     | `datepicker`                                        |
| Boolean / single checkbox | `"type": "boolean"`                   | `xs:boolean`                      | `checkbox`                                          |
| Dropdown (one)   | `"type": "string", "enum": [...]`              | `xs:string` + `xs:enumeration`    | `dropdown`                                          |
| Radio (one)      | `"type": "string", "enum": [...]`              | `xs:string` + `xs:enumeration`    | `radiobutton`                                       |
| Checkbox group (many) | `"type": "array", "items": {"type":"string","enum":[...]}` | repeating `xs:element` | `checkboxgroup`                            |
| File upload      | `"type": "string", "format": "binary"`         | `xs:base64Binary`                 | `fileinput`                                         |
| Object / panel   | `"type": "object", "properties": {...}`        | `xs:complexType`                  | `panelcontainer`                                    |
| Repeating group  | `"type": "array", "items": {"type":"object"}`  | `maxOccurs="unbounded"`           | `panelcontainer` (repeatable)                       |

Never invent type names outside this table. Never use a Foundation component when
reverse-mapping — only the `core/fd/components/form/...` resource types.

---

## JSON Schema output template

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "{schemaName}",
  "title": "{Schema Display Title}",
  "type": "object",
  "properties": {
    "customer": {
      "type": "object",
      "title": "Customer",
      "properties": {
        "firstName": {
          "type": "string",
          "title": "First Name",
          "minLength": 2,
          "maxLength": 50,
          "aem:afProperties": {
            "fd:formDataRef": "$.customer.firstName"
          }
        },
        "email": {
          "type": "string",
          "format": "email",
          "title": "Email",
          "aem:afProperties": {
            "fd:formDataRef": "$.customer.email"
          }
        },
        "maritalStatus": {
          "type": "string",
          "title": "Marital Status",
          "enum": ["single", "married", "divorced", "widowed"],
          "aem:afProperties": {
            "fd:formDataRef": "$.customer.maritalStatus"
          }
        }
      },
      "required": ["firstName", "email"]
    }
  },
  "required": ["customer"]
}
```

JSON Schema rules:
- Group related fields into nested objects (`customer`, `address`, `employment`) — this
  drives clean `$.object.field` binding paths.
- `title` on every property = the form field label (sentence case, no trailing colon).
- Put min/max, pattern, enum, format on the property — `create-adaptive-form` reads these
  to generate the field constraints automatically.
- `required` is an array at each object level, listing the mandatory child property names.

---

## XSD output template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           elementFormDefault="qualified">

  <xs:element name="{rootObjectName}">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="customer">
          <xs:complexType>
            <xs:sequence>
              <xs:element name="firstName" type="xs:string"/>
              <xs:element name="email">
                <xs:simpleType>
                  <xs:restriction base="xs:string">
                    <xs:pattern value="[^@\s]+@[^@\s]+\.[^@\s]+"/>
                  </xs:restriction>
                </xs:simpleType>
              </xs:element>
              <xs:element name="maritalStatus">
                <xs:simpleType>
                  <xs:restriction base="xs:string">
                    <xs:enumeration value="single"/>
                    <xs:enumeration value="married"/>
                    <xs:enumeration value="divorced"/>
                    <xs:enumeration value="widowed"/>
                  </xs:restriction>
                </xs:simpleType>
              </xs:element>
            </xs:sequence>
          </xs:complexType>
        </xs:element>
      </xs:sequence>
    </xs:complexType>
  </xs:element>

</xs:schema>
```

XSD rules:
- Required vs optional is expressed with `minOccurs="0"` (optional) — required is the
  default (`minOccurs="1"`).
- For XSD-based forms the `fd:formDataRef` uses an XPath-like path, not a JSON path —
  see FDM binding rules below.
- Use `<xs:annotation><xs:documentation>` to carry the human label when needed.

---

## FDM binding rules (always generated)

Every leaf field gets a `fd:formDataRef`. The path syntax depends on the representation:

| Representation | Binding path syntax | Example |
|---|---|---|
| JSON Schema | JSONPath from root: `$.object.field` | `$.customer.firstName` |
| XSD         | XPath from root element: `/root/object/field` | `/loanApplication/customer/firstName` |
| Repeating (JSON) | `$.object[*].field` | `$.dependents[*].name` |
| Repeating (XSD)  | `/root/object/field` (element is `maxOccurs="unbounded"`) | `/form/dependents/name` |

- For JSON Schema, carry the binding inside `aem:afProperties.fd:formDataRef` on each leaf.
- Produce a **binding manifest** (`bindings.json`) alongside the schema that lists every
  field → its `fd:formDataRef` path, so `create-adaptive-form` can wire fields without
  re-deriving paths.
- If `fdmEnabled: false` in config, still generate the bindings but note in the summary
  that FDM is not yet configured, and remind the developer to set up the FDM cloud config
  under `fdmRoot`.

---

## Output files (always)

> **CRITICAL — the schema must be a DAM *asset*, not a plain file.** AEM's Adaptive Form
> data-model picker (form container → *Form Model* → *JSON Schema* / *XML Schema*) lists
> only `dam:Asset` nodes under `/content/dam/formsanddocuments` whose
> `jcr:content/@type = "lcResource"`. If the schema is deployed as a bare `nt:file`
> (which is what you get from writing a single `.schema.json` file into a content
> package), **it will not appear in the picker** even though it exists in the repository.
> Always emit the FileVault *asset* structure below so the deployed node is a
> `dam:Asset` of `type=lcResource`.

A JSON Schema asset is a **folder** named `{schemaName}.schema.json` containing the
docview `.content.xml` plus the schema body as the asset's `original` rendition:

```
{fdmRoot}/{schemaName}/
└── .content.xml                                   ← FDM schema association node (when fdmEnabled)

ui.content/src/main/content/jcr_root{schemaContentRoot}/
├── .content.xml                                   ← schema FOLDER node (sling:Folder, type=lcFolder) — REQUIRED, see below
├── {schemaName}.schema.json/                      ← dam:Asset (JSON Schema). For XSD: {schemaName}.xsd/
│   ├── .content.xml                               ← dam:Asset + jcr:content(type=lcResource) + metadata
│   └── _jcr_content/
│       └── renditions/
│           ├── original                           ← the actual schema body (JSON or XSD bytes)
│           └── original.dir/.content.xml          ← nt:file w/ jcr:mimeType for the original rendition
└── {schemaName}.bindings.json                     ← field → fd:formDataRef manifest (plain file; build helper only)
```

The DAM-asset folder name still follows the strict filename convention:
- **JSON Schema** — folder `{schemaName}.schema.json` (the `.schema.json` suffix is
  mandatory). E.g. for a form named `employeeDetails` →
  `/content/dam/formsanddocuments/schema/employeeDetails.schema.json`.
- **XSD** — folder `{schemaName}.xsd` (the `.schema.` infix is JSON-Schema-only).

### Schema folder `.content.xml` (the lcFolder marker — required)

Write this once at `ui.content/.../jcr_root{schemaContentRoot}/.content.xml` (i.e. for the
`schema` folder itself). Without it the folder will not show in the Forms & Documents console.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          jcr:primaryType="sling:Folder" lcFolder="{Long}0" type="lcFolder">
    <jcr:content jcr:primaryType="nt:unstructured" jcr:title="Schema"/>
</jcr:root>
```

> **Gotcha — `merge` mode will NOT fix a wrong pre-existing folder.** The `schema` filter
> root uses `mode="merge"`, which only adds nodes that don't exist; it never changes the
> primaryType/properties of a node already present. If a bare `nt:folder` `schema` node was
> already created by a prior deploy, redeploying with this `.content.xml` will NOT convert
> it. Fix the live node with a CSRF-token'd Sling POST (see the replacing-an-existing-schema
> note below for the headers — the same `Referer`/`Origin` requirement applies), setting
> `jcr:primaryType=sling:Folder`, `type=lcFolder`, `lcFolder@TypeHint=Long`, `lcFolder=0`,
> and `jcr:content/jcr:title`. On a clean instance the `.content.xml` alone is sufficient.

### Asset `.content.xml` (the lcResource marker — required)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          xmlns:dam="http://www.day.com/dam/1.0" xmlns:mix="http://www.jcp.org/jcr/mix/1.0"
          xmlns:dc="http://purl.org/dc/elements/1.1/"
          jcr:primaryType="dam:Asset" jcr:mixinTypes="[mix:referenceable]">
    <jcr:content jcr:primaryType="dam:AssetContent"
                 dam:assetState="processed" type="lcResource" lcResource="{Long}1">
        <metadata jcr:primaryType="nt:unstructured"
                  jcr:mixinTypes="[mix:created,mix:lastModified]"
                  dc:format="application/json"/>   <!-- application/xml for XSD -->
    </jcr:content>
</jcr:root>
```

Do **not** inline a `renditions` node inside this `.content.xml`; renditions are supplied
through the sibling `_jcr_content/renditions/` directory (mixing both makes FileVault drop
the binary and the install fails with *"Mandatory properties '[jcr:data]' not found"*).

### `_jcr_content/renditions/original.dir/.content.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          jcr:primaryType="nt:file">
    <jcr:content jcr:primaryType="nt:resource"
                 jcr:mimeType="application/schema+json"/>   <!-- text/xml for XSD -->
</jcr:root>
```

The schema body itself is written to `_jcr_content/renditions/original` (no extension).
On import AEM extracts `dam:size`/`dam:sha1` and the asset shows in the schema picker.

> **Replacing an existing schema:** if a schema of the same name was ever deployed as a
> plain `nt:file`, the node type cannot change in `merge` mode — the install fails with
> *"No matching definition found for child node renditions"*. Delete the stale node first
> (e.g. CRXDE, or a CSRF-token'd `:operation=delete` POST), then redeploy. A Sling POST to
> AEM also requires a `Referer`/`Origin` header matching the host (e.g. `http://localhost:4502/`)
> in addition to the `CSRF-Token` header (fetched from `/libs/granite/csrf/token.json`),
> otherwise AEM's referrer filter rejects it with **403 Forbidden**.

`{fdmRoot}` and `{schemaContentRoot}` come from `.aem-forms-config.yaml` (or ask the
developer). Conventionally fdmRoot is `/conf/{project}/settings/cloudconfigs/fdm` and
schemaContentRoot is `/content/dam/formsanddocuments/schema`.

**FileVault filter (required — else the `ui.content` build fails).** Any file written
under `ui.content/.../jcr_root/` must be covered by a filter root in
`ui.content/src/main/content/META-INF/vault/filter.xml`, or the `filevault-package`
plugin aborts with *"File ... not covered by a filter rule"*. After writing schema files,
verify a filter root covers their path and add one if missing:

```xml
<filter root="/content/dam/formsanddocuments/schema" mode="merge"/>
```

---

## Reverse-engineering from an existing Adaptive Form

When the input is AF XML/JCR:
1. Locate the `guideContainer` root panel and walk every descendant node.
2. For each node with a `core/fd/components/form/...` resource type, look up the reverse
   map (right column of the field table) to get the schema type.
3. Use the node's `name` as the property name and `jcr:title` as the `title`.
4. Lift `required`, `minLength`/`maxLength`, `minimum`/`maximum`, `validatePictureClause`,
   and inline `items` (enum) onto the schema property.
5. Map nested `panelcontainer` nodes to nested objects; repeatable panels to arrays.
6. Preserve any existing `fd:formDataRef` exactly; only synthesise one where missing.

---

## From OpenAPI / Swagger

1. Ask which operation / schema is the form target if more than one exists.
2. Resolve `$ref`s and `allOf`/`oneOf` into a flat object tree.
3. Map OpenAPI types → the field table: `string+format:email` → email, `integer` → integer,
   `array` of enum → checkbox group, nested `object` → panel, etc.
4. Carry `required`, `minLength`, `maxLength`, `minimum`, `maximum`, `pattern`, `enum`.
5. Drop server-only fields (read-only IDs, audit timestamps) unless the developer wants them
   as hidden fields — ask.

---

## From a public webpage URL (URL-replica delivery)

When the input is a **public webpage URL** whose embedded form is being recreated as an EXACT-replica
Adaptive Form, this skill produces the schema that form binds to:

1. If the Planwright's `discover-form-requirements` already ran, **reuse its captured field inventory**
   (the `sections`/`fields` and `style_spec` in the Structured Requirements) — do not re-fetch.
2. Otherwise **WebFetch the URL**, isolate the target `<form>` (ignore all page chrome — nav, footer,
   marketing), and read every control inside it.
3. Reverse-map each control through the canonical field-type table: `input type` → JSON Schema type
   (`email`→`string`+`format:email`, `tel`→`string`+phone pattern, `select`/`radio`→`enum`,
   `checkbox` group→`array`, `date`→`string`+`format:date`, `textarea`→`string`, etc.). Preserve field
   ORDER and carry `required`, `maxlength`→`maxLength`, `pattern`, and option values/labels→`enum`.
4. Use `camelCase` for the property name (derive from the control `name`/`id`/label). Group related
   controls into nested objects where the form sections imply it.
5. This schema is the data model only — the EXACT-replica look (theme) comes from the captured
   `style_spec` via `create-form-theme`, and the client-side behaviour via `create-form-rules`.
6. **JS-rendered-form fallback:** if the page renders its form client-side and WebFetch cannot see the
   `<form>` markup, state the limitation and build the schema from the fields the user supplies —
   never invent fields.

---

## Quality checklist — verify before finishing

- [ ] Input type was detected and stated back to the developer
- [ ] Output representation (JSON Schema / XSD) was confirmed
- [ ] Every field maps through the canonical type table — no invented types
- [ ] Every leaf field has an `fd:formDataRef` binding path
- [ ] `bindings.json` manifest generated and complete
- [ ] Property names are `camelCase`; object names are descriptive (no `obj1`, `group_a`)
- [ ] Required fields captured in the `required` array (JSON) / via `minOccurs` (XSD)
- [ ] Constraints (min/max/pattern/enum/format) carried onto properties
- [ ] Schema validates (well-formed JSON Schema draft-07 / valid XSD)
- [ ] JSON Schema written as `{schemaContentRoot}/{schemaName}.schema.json` (exact
      convention) / XSD written as `{schemaContentRoot}/{schemaName}.xsd`
- [ ] Schema emitted as a **`dam:Asset` of `type=lcResource`** (docview `.content.xml` +
      `original` rendition) — NOT a bare `nt:file`, or the AF schema picker won't list it
- [ ] A FileVault filter root covers `/content/dam/formsanddocuments/schema`
- [ ] The `schema` folder has its own `.content.xml` (`sling:Folder` + `type=lcFolder` +
      titled `jcr:content`) so it shows in the Forms & Documents console — NOT a bare `nt:folder`
- [ ] Summary reminds the developer to run `create-adaptive-form` against this schema next

---

## Example

Developer prompt: *"Generate a JSON Schema for a loan application — applicant name, email,
PAN, marital status dropdown, monthly income, and a list of dependents (name + age).
FDM-ready."*

See `references/examples.md` for the complete generated JSON Schema, XSD, and bindings
manifest for this and other input types (OpenAPI → schema, AF XML → schema, Figma → schema).
