# generate-schema — worked examples

One example per input type. Each shows the prompt, the detected input, and the full
generated output (schema + `bindings.json`).

---

## Example 1 — Natural language → JSON Schema

**Prompt:** *"Generate a JSON Schema for a loan application — applicant name, email, PAN,
marital status dropdown, monthly income, and a list of dependents (name + age). FDM-ready."*

**Detected input:** Natural language → representation: JSON Schema.

> Output file: `/content/dam/formsanddocuments/schema/loanApplication.schema.json`

### `loanApplication.schema.json`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "loanApplication",
  "title": "Loan Application",
  "type": "object",
  "properties": {
    "applicant": {
      "type": "object",
      "title": "Applicant",
      "properties": {
        "fullName": {
          "type": "string",
          "title": "Full Name",
          "minLength": 2,
          "maxLength": 80,
          "aem:afProperties": { "fd:formDataRef": "$.applicant.fullName" }
        },
        "email": {
          "type": "string",
          "format": "email",
          "title": "Email",
          "aem:afProperties": { "fd:formDataRef": "$.applicant.email" }
        },
        "pan": {
          "type": "string",
          "title": "PAN",
          "pattern": "[A-Z]{5}[0-9]{4}[A-Z]{1}",
          "aem:afProperties": { "fd:formDataRef": "$.applicant.pan" }
        },
        "maritalStatus": {
          "type": "string",
          "title": "Marital Status",
          "enum": ["single", "married", "divorced", "widowed"],
          "aem:afProperties": { "fd:formDataRef": "$.applicant.maritalStatus" }
        },
        "monthlyIncome": {
          "type": "number",
          "title": "Monthly Income",
          "minimum": 0,
          "aem:afProperties": { "fd:formDataRef": "$.applicant.monthlyIncome" }
        }
      },
      "required": ["fullName", "email", "pan", "monthlyIncome"]
    },
    "dependents": {
      "type": "array",
      "title": "Dependents",
      "items": {
        "type": "object",
        "properties": {
          "name": {
            "type": "string",
            "title": "Dependent Name",
            "aem:afProperties": { "fd:formDataRef": "$.dependents[*].name" }
          },
          "age": {
            "type": "integer",
            "title": "Age",
            "minimum": 0,
            "maximum": 120,
            "aem:afProperties": { "fd:formDataRef": "$.dependents[*].age" }
          }
        },
        "required": ["name", "age"]
      }
    }
  },
  "required": ["applicant"]
}
```

### `loanApplication.bindings.json`

```json
{
  "schema": "loanApplication",
  "representation": "json-schema",
  "bindings": [
    { "field": "fullName",       "fd:formDataRef": "$.applicant.fullName" },
    { "field": "email",          "fd:formDataRef": "$.applicant.email" },
    { "field": "pan",            "fd:formDataRef": "$.applicant.pan" },
    { "field": "maritalStatus",  "fd:formDataRef": "$.applicant.maritalStatus" },
    { "field": "monthlyIncome",  "fd:formDataRef": "$.applicant.monthlyIncome" },
    { "field": "name",           "fd:formDataRef": "$.dependents[*].name" },
    { "field": "age",            "fd:formDataRef": "$.dependents[*].age" }
  ]
}
```

---

## Example 2 — OpenAPI spec → JSON Schema

**Prompt:** *"Generate a form schema from the `createCustomer` request body in this OpenAPI
spec."*

**Input (excerpt):**

```yaml
components:
  schemas:
    CreateCustomerRequest:
      type: object
      required: [firstName, lastName, email]
      properties:
        id:        { type: string, readOnly: true }
        firstName: { type: string, maxLength: 50 }
        lastName:  { type: string, maxLength: 50 }
        email:     { type: string, format: email }
        country:   { type: string, enum: [IN, US, UK, AU] }
        createdAt: { type: string, format: date-time, readOnly: true }
```

**Processing notes:** `id` and `createdAt` are `readOnly` (server-only) → dropped after
confirming with the developer. `country` enum → dropdown.

> Output file: `/content/dam/formsanddocuments/schema/createCustomer.schema.json`

### `createCustomer.schema.json`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "createCustomer",
  "title": "Create Customer",
  "type": "object",
  "properties": {
    "customer": {
      "type": "object",
      "title": "Customer",
      "properties": {
        "firstName": {
          "type": "string", "title": "First Name", "maxLength": 50,
          "aem:afProperties": { "fd:formDataRef": "$.customer.firstName" }
        },
        "lastName": {
          "type": "string", "title": "Last Name", "maxLength": 50,
          "aem:afProperties": { "fd:formDataRef": "$.customer.lastName" }
        },
        "email": {
          "type": "string", "format": "email", "title": "Email",
          "aem:afProperties": { "fd:formDataRef": "$.customer.email" }
        },
        "country": {
          "type": "string", "title": "Country", "enum": ["IN", "US", "UK", "AU"],
          "aem:afProperties": { "fd:formDataRef": "$.customer.country" }
        }
      },
      "required": ["firstName", "lastName", "email"]
    }
  },
  "required": ["customer"]
}
```

---

## Example 3 — Existing Adaptive Form XML → JSON Schema (reverse-engineer)

**Input (excerpt of a `guideContainer` panel):**

```xml
<personalDetails sling:resourceType="core/fd/components/form/panelcontainer/v1/panelcontainer"
                 jcr:title="Personal Details" name="personalDetails">
  <firstName sling:resourceType="core/fd/components/form/textinput/v1/textinput"
             jcr:title="First Name" name="firstName"
             required="{Boolean}true" minLength="{Long}2" maxLength="{Long}50"
             fd:formDataRef="$.customer.firstName"/>
  <dob sling:resourceType="core/fd/components/form/datepicker/v1/datepicker"
       jcr:title="Date of Birth" name="dob"/>
</personalDetails>
```

**Processing:** `panelcontainer` → object `personalDetails`; `textinput` → string;
`datepicker` → string/date. Existing `fd:formDataRef` on `firstName` preserved; one is
synthesised for `dob`.

> Output file: `/content/dam/formsanddocuments/schema/customerProfile.schema.json`

### `customerProfile.schema.json`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "customerProfile",
  "title": "Customer Profile",
  "type": "object",
  "properties": {
    "personalDetails": {
      "type": "object",
      "title": "Personal Details",
      "properties": {
        "firstName": {
          "type": "string", "title": "First Name", "minLength": 2, "maxLength": 50,
          "aem:afProperties": { "fd:formDataRef": "$.customer.firstName" }
        },
        "dob": {
          "type": "string", "format": "date", "title": "Date of Birth",
          "aem:afProperties": { "fd:formDataRef": "$.personalDetails.dob" }
        }
      },
      "required": ["firstName"]
    }
  }
}
```

---

## Example 4 — Same loan application as XSD

**Prompt:** *"Give me the loan application as an XSD instead."*

> Output file: `/content/dam/formsanddocuments/schema/loanApplication.xsd`

### `loanApplication.xsd`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema" elementFormDefault="qualified">
  <xs:element name="loanApplication">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="applicant">
          <xs:complexType>
            <xs:sequence>
              <xs:element name="fullName" type="xs:string"/>
              <xs:element name="email">
                <xs:simpleType>
                  <xs:restriction base="xs:string">
                    <xs:pattern value="[^@\s]+@[^@\s]+\.[^@\s]+"/>
                  </xs:restriction>
                </xs:simpleType>
              </xs:element>
              <xs:element name="pan">
                <xs:simpleType>
                  <xs:restriction base="xs:string">
                    <xs:pattern value="[A-Z]{5}[0-9]{4}[A-Z]{1}"/>
                  </xs:restriction>
                </xs:simpleType>
              </xs:element>
              <xs:element name="maritalStatus" minOccurs="0">
                <xs:simpleType>
                  <xs:restriction base="xs:string">
                    <xs:enumeration value="single"/>
                    <xs:enumeration value="married"/>
                    <xs:enumeration value="divorced"/>
                    <xs:enumeration value="widowed"/>
                  </xs:restriction>
                </xs:simpleType>
              </xs:element>
              <xs:element name="monthlyIncome" type="xs:decimal"/>
            </xs:sequence>
          </xs:complexType>
        </xs:element>
        <xs:element name="dependents" minOccurs="0" maxOccurs="unbounded">
          <xs:complexType>
            <xs:sequence>
              <xs:element name="name" type="xs:string"/>
              <xs:element name="age" type="xs:integer"/>
            </xs:sequence>
          </xs:complexType>
        </xs:element>
      </xs:sequence>
    </xs:complexType>
  </xs:element>
</xs:schema>
```

### `loanApplication.bindings.json` (XSD representation — XPath paths)

```json
{
  "schema": "loanApplication",
  "representation": "xsd",
  "bindings": [
    { "field": "fullName",      "fd:formDataRef": "/loanApplication/applicant/fullName" },
    { "field": "email",         "fd:formDataRef": "/loanApplication/applicant/email" },
    { "field": "pan",           "fd:formDataRef": "/loanApplication/applicant/pan" },
    { "field": "maritalStatus", "fd:formDataRef": "/loanApplication/applicant/maritalStatus" },
    { "field": "monthlyIncome", "fd:formDataRef": "/loanApplication/applicant/monthlyIncome" },
    { "field": "name",          "fd:formDataRef": "/loanApplication/dependents/name" },
    { "field": "age",           "fd:formDataRef": "/loanApplication/dependents/age" }
  ]
}
```

---

## Example 5 — Figma / screenshot → schema

**Prompt:** *"Here's a screenshot of a contact form. Generate the JSON Schema."*

**Processing:** Infer fields from visible labels. State assumptions for confirmation:

```
Detected fields (please confirm types):
- "Your name"      → string (text)
- "Email address"  → string, format: email
- "Reason"         → string, enum [General, Billing, Support]  (looked like a dropdown)
- "Message"        → string (multi-line — textarea)
- "Attach file"    → string, format: binary (file upload)

Assumptions: all fields required except "Attach file". Confirm before I generate.
```

After confirmation, emit the JSON Schema following the same pattern as Example 1.
