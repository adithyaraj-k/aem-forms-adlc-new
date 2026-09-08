---
name: create-prefill-service
description: >
  Generates a complete, production-ready AEM Adaptive Forms custom prefill service
  for AEM as a Cloud Service. Creates all required files: OSGi Java service, OSGi
  configuration, system user repoinit script, service user mapping, and unit tests.
  ALWAYS starts by showing the form's field list and asking the user how to prefill it —
  two choices: give an Excel/CSV file, or no prefill. The Excel/CSV path builds a CRX/DAM
  JSON static-path prefill that fills the form on every render (there is no manual
  value-entry option). Also supports three low-level data-source variants — REST API,
  CRX/DAM JSON, and JCR draft data. Use this skill whenever the
  developer asks to pre-populate form fields, auto-fill fields on form load, fetch
  initial values from an API or user profile, load saved draft data, or create any
  kind of prefill or data provider for AEM Adaptive Forms. Also triggers for phrases
  like "prefill service", "DataProvider", "form pre-population", "fill fields on load",
  or "custom data provider for forms".
---

# Skill: create-prefill-service

## Role

You are an expert AEM Forms backend developer for AEM as a Cloud Service. Before
writing a single line of code, you ask the developer the minimum questions needed
to produce code that compiles and deploys without modification. You then generate
all required files at once so the developer can drop them directly into their project.

---

## Step 0 — Present the fields and choose the prefill approach (ALWAYS do this first)

Whenever this skill runs for a form — invoked directly OR by the program pipeline right
after a form is built — begin here, BEFORE Step 1. Do not assume an approach.

**1. List the form's fields.** Read the target form (its `*.schema.json` and/or the
`guideContainer` in `/content/forms/af/{appFolder}/{formName}/.content.xml`) and show
every bindable field with its **bind name**, type, and label, e.g.:

| Field (bind name) | Type | Label |
|---|---|---|
| category | drop-down | Category |
| complaint | text (multiline) | Please describe the problem |
| firstName | text | First name |
| email | email | Email address |
| … | … | … |

**2. Ask how to prefill — exactly two choices** (use the AskUserQuestion tool so the
choice is explicit; fall back to a plain question if it is unavailable):

- **Give an Excel/CSV file** — the user supplies a path to an `.xlsx`/`.csv` whose
  header row names the fields and whose first data row holds the values.
- **No prefill** — create nothing; confirm and STOP. The form renders empty as normal.

There is **no manual value-entry option** — prefill values come from a file, not from
typing them in per field.

**3. Route the choice:**

- **Excel/CSV** → build a **CRX/DAM JSON static-path** prefill service (Variant B, static
  sub-mode): write the file's values as a JSON file into the project's prefill DAM folder
  and generate a `DataProvider` that returns it on **every** render (including anonymous
  users). This is the default for "fill the form on every render from a fixed data set".
  Proceed to Step 1 with **variant = B (static path)** — do NOT re-ask the REST/CRX/JCR
  variant question; it is already decided.
- **No prefill** → stop; produce no artifacts and write no run file.

### Excel / CSV → prefill JSON (Excel choice)

- Header row = field names. Match each header to a form field **bind name**
  (case-insensitively; fall back to matching the field **label**). Report any column
  that maps to no field, and any field left without a value.
- Use the **first data row** for the values. If the sheet has more than one data row,
  say so and use only the first (one form render = one data set).
- **Normalize values to the field's expected form:** a dropdown/enum value must equal an
  actual option value exactly (e.g. Excel `Product Shortage` → the enum's `Product shortage`);
  numbers Excel stored in scientific notation (e.g. a phone `9.0032456789E10`) must be
  emitted as the plain string `90032456789`.
- **Never prefill a consent/agreement checkbox** unless the source explicitly sets it —
  consent must be given actively by the user.
- Omit fields the file leaves blank from the JSON (don't write empty strings) unless an
  empty string is meaningful.

### The prefill JSON

- Keys MUST equal the form field bind names exactly (case-sensitive) — see the "JSON keys
  match form field names" rule.
- Store it at `{damPrefillRoot}/{formName}.json` (the project's prefill folder, e.g.
  `/content/dam/{project}/prefilldata/{formName}.json`) with its `.dir/.content.xml`
  (`nt:file` → `jcr:content` `nt:resource` `jcr:mimeType="application/json"`), and point
  the service's data-file config at it. Give the service a form-specific `getServiceName()`
  (e.g. `{formName}PrefillService`) so it does not collide with other prefill services,
  and set `guideContainer/@prefillService` to that exact value.

---

## Step 1 — Gather project context (ask these, all at once)

> If Step 0 already selected Excel or Manual, the **variant is fixed to B (CRX/DAM JSON,
> static path)** — skip question 1 below and read the values from the project config
> (`.aem-forms-config.yaml` → package / project / bundle). Only ask what you cannot derive.
> The A/B/C variant question below applies to a **direct, low-level** invocation where the
> caller explicitly wants a REST or JCR-draft provider instead.

Ask the following in a single message before generating anything:

1. **Prefill variant** — which data source does this service read from?
   - **A) REST API** — call an external HTTP endpoint (user profile, CRM, etc.)
   - **B) CRX / DAM JSON** — read a JSON file stored in the AEM repository or DAM
   - **C) JCR Draft** — load a previously saved form draft from `/var/fd/dashboard/data/drafts/`

2. **Java package** — e.g. `com.mycompany.myproject` (you will place the service under
   `{package}.forms.prefill`)

3. **Service name (camelCase)** — a short identifier for the service, e.g. `UserProfile`,
   `LoanApplication`. This becomes `{ServiceName}PrefillService` in Java and the OSGi
   component name.

4. **Project/app name** — the `appId` used in your project's `ui.apps` and `ui.config`
   modules (e.g. `myproject`). Used for OSGi config paths and repoinit.

5. **Bundle symbolic name** — the OSGi bundle symbolic name for the `core` module
   (e.g. `com.mycompany.myproject.core`). Used in the service user mapping.

6. **Variant-specific extras:**
   - If **A (REST API)**: the API endpoint base URL, any auth header name needed
     (e.g. Bearer token), and the JSON field(s) the API returns that map to form fields.
   - If **B (CRX/DAM JSON)**: the DAM/JCR path where the JSON file lives
     (e.g. `/content/dam/myproject/prefilldata/`), and whether the path is
     static or resolved dynamically from the request (e.g. by user ID).
   - If **C (JCR Draft)**: confirm the default draft path
     `/var/fd/dashboard/data/drafts/` or provide a custom one.

Do NOT proceed to code generation until you have answers to all of the above.

---

## Step 2 — Files to generate (always all 5)

Once you have answers, generate all five files together:

```
core/src/main/java/{package}/forms/prefill/
  └── {ServiceName}PrefillService.java

core/src/test/java/{package}/forms/prefill/
  └── {ServiceName}PrefillServiceTest.java

ui.config/src/main/content/jcr_root/apps/{appId}/config/
  └── com.{package}.forms.prefill.{ServiceName}PrefillService.cfg.json

ui.config/src/main/content/jcr_root/apps/{appId}/config/
  └── org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended~{appId}-forms.cfg.json

ui.config/src/main/content/jcr_root/apps/{appId}/config/
  └── org.apache.sling.jcr.repoinit.RepositoryInitializer~{appId}-forms-prefill.cfg.json
```

Read the relevant variant reference file before generating:
- Variant A → `references/variant-rest-api.md`
- Variant B → `references/variant-crx-json.md`
- Variant C → `references/variant-jcr-draft.md`

Also read `references/common-patterns.md` — it defines the SPI, repoinit format,
service user mapping format, OSGi config rules, and unit test patterns that apply
to all variants.

---

## Step 3 — After generating files, always provide

1. **Form wiring instructions** — exactly how to set the prefill service on the form
   container (UI steps + CRX/DE property name and value).
2. **Field bindRef guide** — a table mapping the JSON keys your service returns to the
   `bindRef` values the developer must set on each Adaptive Form field.
3. **Verification checklist** — OSGi console URL, service registration check, how to
   confirm the repoinit user was created, and a curl/browser test to confirm prefill works.

---

## Critical correctness rules (apply to all variants)

These are non-negotiable. Violating any of them causes production failures.

- **SPI**: implement `com.adobe.forms.common.service.DataProvider`. Return
  `new PrefillData(InputStream, ContentType.JSON)`. Never use the AEM 6.x
  `DataXMLProvider` / `getDataXML()` — that interface does not exist on AEMaaCS.
- **ResourceResolver lifecycle**: every `ResourceResolver` opened from
  `ResourceResolverFactory` MUST be inside a `try-with-resources` block. A leaked
  resolver causes memory exhaustion within hours in production. Never use a manual
  `finally` block — try-with-resources is the only acceptable pattern.
- **OSGi config format**: all configs in `ui.config` must be `.cfg.json` files.
  XML-based `sling:OsgiConfig` nodes are NOT supported on AEMaaCS and will be ignored
  by Cloud Manager.
- **System user via repoinit**: you cannot manually create JCR system users on AEMaaCS.
  The system user and its ACLs must be declared in a repoinit `RepositoryInitializer`
  config. This is a required file — omitting it means `LoginException` at runtime.
- **repoinit must not `set ACL` on a path that doesn't exist at startup** — repoinit runs
  at bundle startup, BEFORE content packages install `/content/dam/.../prefilldata`,
  `/var/fd/dashboard/data/drafts`, etc. `set ACL` on a missing path throws and **aborts
  the whole script**, so the `create service user` above it never commits — the user is
  silently never created and every prefill returns `{}` with
  `LoginException: Cannot derive user name … sub service forms-prefill-service`. **Always
  emit `create path (sling:Folder) <path>` for each package-created / runtime path BEFORE
  the `set ACL` block.** `/content/forms/af` exists at startup and needs no `create path`.
  Verify after deploy: `GET /bin/querybuilder.json?path=/home/users/system&property=rep:principalName&property.value={appId}-forms-prefill-service`
  must return `total:1`. Reuse an existing project prefill user/mapping if one already exists
  rather than adding a duplicate mapping.
- **Secrets**: API tokens and passwords must use `$[secret:key-name]` substitution in
  `.cfg.json`. Never hardcode secrets.
- **JSON keys match form field names**: the JSON object your service returns must have
  keys that exactly match the Adaptive Form field names (or the FDM binding names).
  Case-sensitive. Document this mapping explicitly for the developer.
- **Graceful degradation**: null/anonymous user, missing resource, and API/network
  failures must all return `jsonPrefill("{}")` — never throw to the caller.
