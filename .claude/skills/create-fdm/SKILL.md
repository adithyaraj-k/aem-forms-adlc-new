---
name: create-fdm
description: >
  Creates an AEM Forms Data Integration data source (Swagger/REST cloud configuration) and a
  Form Data Model (FDM) that consumes it, on AEM as a Cloud Service. Produces the data-source
  cloud config under /conf/{project}/settings/cloudconfigs/fdm, its uploaded Swagger file, and the
  FDM model under /content/dam/formsanddocuments-fdm/{project} — including the critical
  jcr:content/sources/source1 binding that links the model to the data source. Covers no-auth,
  Basic-auth, and OAuth2 (Adobe-recommended for production) REST sources, deployment, and how to
  test the model. Use when asked to create / wire /
  commit a Form Data Model, a data source, an FDM, or to back a form's data with an external/REST
  (Swagger / OpenAPI) service.
version: 1.2.0
ide:
  cursor: .cursor/skills/create-fdm/
  github-copilot: .github/skills/create-fdm/
  claude-code: .claude/skills/create-fdm/
---

# Skill: create-fdm

## Role

You are an AEM Forms Data Integration engineer for AEM as a Cloud Service. You create a **data
source** (a Swagger/REST cloud configuration) and a **Form Data Model (FDM)** that consumes it, as
committed repository content that deploys and works. You produce the exact JCR structures this
project uses (verified by round-tripping editor-created artifacts), including the binding wiring
that hand-authoring most often misses.

> **Schema vs FDM (read first).** A JSON Schema/XSD and an FDM are **mutually exclusive** form data
> bindings — use FDM **or** a schema, never both for the same form. This skill is the FDM path. See
> [[schema-when-creating-forms]].

---

## Trigger

Activate when the developer asks to:
- Create / wire / commit a **Form Data Model (FDM)** or a **data source**
- Back a form's data with an external **REST / Swagger / OpenAPI** service via FDM
- Add `fd:formDataRef` bindings driven by a data model
- Import a Swagger/OpenAPI spec into AEM Forms as a data source + model

---

## The two artifacts (both required for a working FDM)

| # | Artifact | JCR location | Repo path |
|---|---|---|---|
| 1 | **Data source** (cloud config) | `/conf/{project}/settings/cloudconfigs/fdm/{name}-cloud-config` | `ui.content/.../jcr_root/conf/{project}/settings/cloudconfigs/fdm/{name}-cloud-config/` |
| 2 | **FDM model** | `/content/dam/formsanddocuments-fdm/{project}/{name}-data-model` | `ui.content/.../jcr_root/content/dam/formsanddocuments-fdm/{project}/{name}-data-model/` |

For this project `{project} = aem-demo-site`. The FDM paths use `{project}`
(`/content/dam/formsanddocuments-fdm/{project}`) — the SAME `{project}` folder segment that form
content, DAM guide assets, and `/conf/forms` use. There is no separate app folder.

> ⚠️ **The reliable way to build an FDM is the editor, then export to the repo.** The data-source
> dialog is **JSP-rendered** and the FDM model's data-source **binding** + (for Basic auth) the
> **encrypted password** are written by the editor — these are tedious and error-prone to
> hand-author. Strongly prefer **Approach A**. Use **Approach B** (hand-author) only when the editor
> isn't available, and verify against a known-good editor-created model.

---

## Approach A — editor-first, then export (RECOMMENDED)

1. **Create the data source** in the AEM author UI: **Tools → Cloud Services → Data Sources** →
   container `/conf/{project}` → **Create** → **RESTful Service** → set the **Service Endpoint**
   (base URL), upload the **Swagger** file, choose **Authentication** (None / Basic / OAuth), save.
2. **Create the FDM**: **Tools → Forms → Data Models → Create → Form Data Model** → **select the
   data source** → Next → add the entities/services from the imported Swagger → **Create**.
3. **Test** in the model editor's **Test Model** tab (pick the service, enter inputs, run).
4. **Export both into the repo**: copy the JCR content (data source + model) into `ui.content` under
   the paths above. Confirm it matches the structures in Approach B (especially the
   `sources/source1` node) and commit. This captures the editor-written binding so it is
   reproducible. Apply the FileVault escaping rule for the encrypted password (below).

This is how the existing `salesforce-cloud-config` / `salesforce-data-model` and the
`temperature-cloud-config` / `temperature-data-model` artifacts were produced.

---

## Approach B — hand-authored structures (verified)

### Artifact 1 — Data source (RESTful, Swagger)

`cq:Page` at `/conf/{project}/settings/cloudconfigs/fdm/{name}-cloud-config`, resource type
`fd/fdm/gui/components/admin/fdmcloudservice/rest`. The Swagger is uploaded as a child `nt:file`
(`swaggerSource="swaggerFile"`).

**`.content.xml` (no authentication):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="cq:Page">
    <jcr:content
        jcr:primaryType="cq:PageContent"
        jcr:title="{Title}"
        sling:resourceType="fd/fdm/gui/components/admin/fdmcloudservice/rest"
        authenticationType="None"
        dataSourceType="REST"
        filename="{swagger-file-name}.json"
        location="header"
        name="{name}-cloud-config"
        restfulService="swagger"
        selectAuthentication="None"
        selectHeader="Authorization Bearer"
        serviceEndPoint="https://host[:port]/"
        swaggerSource="swaggerFile">
        <swaggerFile/>
    </jcr:content>
</jcr:root>
```

> ⚠️ **`serviceEndPoint` and `restfulService="swagger"` are REQUIRED** and the easiest to miss — the
> connector has no base URL without `serviceEndPoint`, and the source won't be recognized as a
> Swagger REST source without `restfulService="swagger"`. `selectAuthentication` is **`None`**, not
> `"No Authentication"`.

**Basic authentication** — change to:
```
authenticationType="Basic Authentication"
selectAuthentication="Basic Authentication"
userName="{user}"
password="\{enc-blob}"
```
- The **password is stored AEM-crypto-encrypted** (e.g. `{de0e6c2f...}`). You **cannot** hand-author
  a plaintext password — set it in the editor (which encrypts it) and export the encrypted blob.
- ⚠️ **Escape the leading `{` as `\{`** in the `.content.xml` (`password="\{de0e...}"`). A value
  starting with `{` is read by FileVault as a `{Type}value` prefix → **BUILD FAILURE**. The `\` is
  stripped on import, leaving the exact blob. (Same escape as `uiModel="\{..."`.)
- ⚠️ The encrypted blob is keyed to the instance's crypto keys: it round-trips on the **same**
  instance, but on another environment (different keys) it won't decrypt — re-enter the password
  there, or use an AEMaaCS **secret/env var**. Committing even an encrypted credential is dev-only;
  never commit real credentials.

**OAuth 2.0 (Adobe-recommended for PRODUCTION REST sources).** Basic auth is fine for a quick dev
spike, but for any real third-party API Adobe recommends **OAuth2** — the data-source dialog offers
**OAuth 2.0 Client Credentials** (server-to-server, the common machine-to-machine case),
**OAuth 2.0 Authorization Code** (on behalf of a user), and **Bearer/API-key** header auth. Prefer
client-credentials for FDM-to-API integration.

- **Author OAuth via Approach A (the editor), not by hand.** The OAuth fields — token endpoint,
  client id, client **secret** (crypto-encrypted), scope, grant type — are written and encrypted by
  the JSP dialog exactly like the Basic password. Hand-authoring the encrypted secret blob and the
  exact `selectAuthentication`/`authenticationType` enum values is error-prone; set them in
  **Tools → Cloud Services → Data Sources → Authentication = OAuth 2.0**, save, then export the
  resulting `.content.xml`. Round-trip the exact attribute names from that export — do NOT guess
  them (the same `\{` FileVault escape applies to the encrypted secret).
- Store the client secret as an AEMaaCS **secret/env var** for non-dev environments, never as a
  committed blob — the encrypted value is instance-keyed and won't decrypt elsewhere.
- A 401 on Test Model with OAuth configured usually means the token endpoint/scope is wrong or the
  secret didn't decrypt on this instance — re-enter it in the editor.

**The Swagger file** is two FileVault parts next to the config:
- `_jcr_content/swaggerFile` — the raw Swagger (JSON or YAML) text. Must include `host`, `basePath`,
  `schemes`, the operation(s) with `operationId`, parameters, and response `definitions`.
- `_jcr_content/swaggerFile.dir/.content.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="nt:file">
    <jcr:content jcr:lastModifiedBy="admin" jcr:mimeType="application/json" jcr:primaryType="nt:resource"/>
</jcr:root>
```

### Artifact 2 — FDM model

A `dam:Asset` at `/content/dam/formsanddocuments-fdm/{project}/{name}-data-model`.

**`.content.xml`** — the asset, its metadata, **and the `sources/source1` binding**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:dam="http://www.day.com/dam/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="dam:Asset">
    <jcr:content
        cq:conf="/conf/{project}"
        jcr:primaryType="dam:AssetContent"
        sling:resourceType="fd/fm/fdm/render"
        type="formdatamodel">
        <metadata
            author="admin"
            jcr:primaryType="nt:unstructured"
            title="{Title}"
            uiModel="\{&quot;models&quot;:{&quot;{Entity}&quot;:{&quot;x&quot;:60\,&quot;y&quot;:80}}}"/>
        <sources jcr:primaryType="sling:Folder">
            <source1
                jcr:primaryType="nt:unstructured"
                dataSourceType="REST"
                configurationPath="{name}-cloud-config"/>
        </sources>
    </jcr:content>
</jcr:root>
```

> ⚠️ **The `sources/source1` node is the binding** and is the #1 thing hand-authoring misses. Without
> it the Data Model editor's **Data Sources panel is blank** and every entity shows **"Unbound"**.
> `configurationPath` is the data-source config **name** (relative to
> `/conf/{project}/settings/cloudconfigs/fdm/`), e.g. `temperature-cloud-config`. Entities reference
> this only as the abstract token `fdm:source="source1"`.

**`_jcr_content/renditions/fdm-json/.content.xml`** — the model structure (a `sling:Folder`, NOT a
binary). Namespace `fdm="http://www.adobe.com/aemfd/fdm/1.0"`. Skeleton for one entity backed by one
GET service:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:fdm="http://www.adobe.com/aemfd/fdm/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="sling:Folder">
    <jcr:content jcr:primaryType="nt:unstructured" name="{name}-data-model" rendition.handler.id="fdm.structure" title="{Title}">
        <definitions jcr:primaryType="nt:unstructured">
            <{Entity}
                fdm:binaryEntity="{Boolean}false" fdm:bindRef="{Entity}" fdm:defaultBinaryEntity="{Boolean}false"
                fdm:genericOperationAllowed="{Boolean}false" fdm:rootEntity="{Boolean}true"
                fdm:schemaName="DEFAULT_SCHEMA" fdm:source="source1"
                jcr:primaryType="nt:unstructured" id="{Entity}" name="{Entity}" required="[{req1},{req2}]">
                <links jcr:primaryType="nt:unstructured">
                    <{linkNodeName}
                        fdm:source="source1" jcr:primaryType="nt:unstructured"
                        description="{desc}" genericOperation="{Boolean}false"
                        href="host[:port]/" id="GET /{path}_1" method="GET" name="GET /{path}" rel="self">
                        <fdm:data jcr:primaryType="nt:unstructured" {param}="${request.attribute.{param}}"/>
                        <schema fdm:bindRef="schema" fdm:computed="{Boolean}false" fdm:metadataKey="{Boolean}false"
                            fdm:nullable="{Boolean}true" fdm:primaryKey="{Boolean}false" fdm:protected="{Boolean}false"
                            fdm:required="{Boolean}false" jcr:primaryType="nt:unstructured"
                            description="operation input description" name="schema" type="object">
                            <properties jcr:primaryType="nt:unstructured">
                                <{param} fdm:bindRef="{param}" fdm:computed="{Boolean}false" fdm:in="query"
                                    fdm:metadataKey="{Boolean}false" fdm:nullable="{Boolean}true" fdm:primaryKey="{Boolean}false"
                                    fdm:protected="{Boolean}false" fdm:required="{Boolean}true" jcr:primaryType="nt:unstructured"
                                    description="{desc}" name="{param}" type="{number|string}"/>
                            </properties>
                        </schema>
                        <targetSchema fdm:bindRef="{Entity}" fdm:computed="{Boolean}false" fdm:metadataKey="{Boolean}false"
                            fdm:nullable="{Boolean}true" fdm:primaryKey="{Boolean}false" fdm:protected="{Boolean}false"
                            fdm:required="{Boolean}false" jcr:primaryType="nt:unstructured"
                            _x0024_ref="{Entity}" name="{Entity}" title="{Entity}" type="object"/>
                    </{linkNodeName}>
                </links>
                <properties jcr:primaryType="nt:unstructured">
                    <{prop} fdm:bindRef="{prop}" fdm:computed="{Boolean}false" fdm:metadataKey="{Boolean}false"
                        fdm:nullable="{Boolean}true" fdm:primaryKey="{Boolean}false" fdm:protected="{Boolean}false"
                        fdm:required="{Boolean}true" jcr:primaryType="nt:unstructured"
                        name="{prop}" title="{prop}" type="{number|string}"/>
                    <!-- one node per entity property -->
                </properties>
            </{Entity}>
        </definitions>
    </jcr:content>
</jcr:root>
```
Notes:
- `STATEMENT`/AST shapes here are FDM-specific (`fdm:*`), unrelated to the Rule-Editor `fd:rules` AST.
- The link `href` is the base URL (e.g. `localhost:4502/`), **not empty**.
- For a **no-argument** service the `schema.properties` is empty and `fdm:data` is empty.
- For a **POST/body** operation, the input property uses `fdm:in="body"` and `_x0024_ref` to the body entity.
- `_x0024_ref` is the FileVault encoding of the `$ref` property; `&quot;` for `"` and `\,` for `,`
  inside the escaped `uiModel` value.

#### Model-level services (the "Services" tab) — REQUIRED for the Services tab to populate

> ⚠️ **An operation placed only inside an entity's `links` does NOT appear in the model's "Services"
> tab** (it's just the entity's read/write op). The **Services tab is populated by a TOP-LEVEL
> `links` node** under `fdm-json/jcr:content` — a **sibling of `definitions`**. If you want the
> service callable from the Services tab (and invokable for prefill/submit), you MUST add it there.
> (Verified: a model with the op only under the entity showed an **empty Services tab**; adding it as
> a model service created the top-level `links` node.)

A fully editor-built `fdm-json/jcr:content` therefore has **three** top-level children:
```
jcr:content (name, title, rendition.handler.id="fdm.structure")
  ├─ definitions   — entities; each with its own <links> (entity ops) + <properties>
  ├─ links         — MODEL-LEVEL services (this is the "Services" tab)
  └─ properties    — model property list; one child per root entity with _x0024_ref="{Entity}"
```
The model-level service node (under the top-level `links`) mirrors the entity-link shape, with these
differences (verified from an editor-created model):
- `id` is the plain operation id (e.g. `GET /bin/temperature/celsius-to-fahrenheit`, no `_N` suffix);
  node name like `get_bin_temperaturecelsius-to-fahrenheit`.
- `fdm:data` is **empty** (`<fdm:data jcr:primaryType="nt:unstructured"/>`) — inputs are supplied at
  invocation, not pre-bound to request attributes.
- `schema` and its `properties` carry the full `fdm:*` set incl. `fdm:readOnly` and `fdm:freeForm`;
  query inputs use `fdm:in="query"`.
- `targetSchema` uses `_x0024_ref="{Entity}"` to the response entity, plus `fdm:readOnly`/`fdm:freeForm`.

The top-level `properties` lists each root entity:
```xml
<properties jcr:primaryType="nt:unstructured">
    <{Entity} jcr:primaryType="nt:unstructured" _x0024_ref="{Entity}" name="{Entity}"
        fdm:readOnly="{Boolean}false" fdm:protected="{Boolean}false" fdm:computed="{Boolean}false"
        fdm:required="{Boolean}false" fdm:nullable="{Boolean}true">
        <properties jcr:primaryType="nt:unstructured"><!-- one node per entity property --></properties>
    </{Entity}>
</properties>
```

> Because this top-level `links`/`properties` structure plus the per-property `fdm:readOnly`/
> `fdm:freeForm` flags are fiddly, the **fastest reliable way is to add the service to the model in
> the editor, then export the whole `fdm-json` node and convert it to docview XML** (fetch
> `…/renditions/fdm-json.infinity.json`, map each JSON object → element, booleans → `{Boolean}…`,
> `$ref` → `_x0024_ref`, `{`-prefixed values → `\{`). That captures the exact structure with no guessing.

---

## Filters

Both roots are usually already covered in `ui.content/src/main/content/META-INF/vault/filter.xml`:
- `/conf/{project}` (covers the data source) — `<filter root="/conf/{project}" mode="update"/>`
- `/content/dam/formsanddocuments-fdm/{project}` (covers the FDM model) — confirm present; add if missing.

Verify each written path is a descendant of a `filter root`; an uncovered node is silently not deployed.

---

## Deploy

> **Pipeline mode (delegated by `formwright`): SKIP this deploy — author the FDM/data-source
> artifacts only.** Deployment is centralized in the `forgemaster` lead (AGENTS.md → "Deployment is
> centralized in Forgemaster"). Run the `mvn` command only when invoked **standalone / directly**.

```bash
# Full reactor — the all package is what uploads to AEM (see [[deploy-needs-all-package]]).
mvn clean install -PautoInstallSinglePackage
```
- Do **NOT** use `-pl ui.content` alone — that skips the `all` module and nothing reaches AEM.
- **Delete-before-deploy for a clean replace** when iterating (FileVault `update` mode won't delete
  properties/nodes you removed from source, so stale bindings/props linger):
  ```
  curl -u admin:admin -F ":operation=delete" "http://localhost:4502/conf/{project}/settings/cloudconfigs/fdm/{name}-cloud-config"
  curl -u admin:admin -F ":operation=delete" "http://localhost:4502/content/dam/formsanddocuments-fdm/{project}/{name}-data-model"
  ```

---

## Testing

- **Test Model (UI):** the model editor's *Test Model* tab invokes the service — this is **browser
  only**; it cannot be driven headlessly. Pick the service, enter inputs, run, expect the mapped
  output. (Under the hood it POSTs to `{model}.executeDermisQuery.json`.)
- **Test the backing service directly** (headless, proves the endpoint the FDM wraps works):
  `curl -u admin:admin "https://host[:port]/{path}?{param}=..."`.
- **Verify deployment** resolves: data source `…/{name}-cloud-config/jcr:content.json` (200, has
  `fdmcloudservice`); model `…/{name}-data-model.6.json` → `jcr:content.sources.source1` present;
  model structure `…/renditions/fdm-json.infinity.json` shows the entity/service.

---

## Invoking an FDM service from a form (Invoke Service rule)

To **compute/populate a field from an FDM service** (a read — e.g. enter Celsius, show Fahrenheit),
use an **Invoke Service** rule, NOT the FDM submit action (which is a write-back; see the error table).
The service must be a **model-level service** (top-level `links` — see above), so it appears in the
Rule Editor's Invoke Service list.

---

## Submitting a form via FDM ("Submit using Form Data Model")

When the PLAN `submit` intent is `fdm` / "Submit using Form Data Model", the form writes its data
back through the FDM's write operations on submit. This is the **ONLY** case where
`actionType="fd/afaddon/components/actions/fdm"` should appear on the `guideContainer`.

**Required `guideContainer` attributes (FDM submit):**
```xml
<guideContainer
    ...
    schemaType="formdatamodel"
    schemaRef="/content/dam/formsanddocuments-fdm/{project}/{fdm}-data-model"
    actionType="fd/afaddon/components/actions/fdm"
    fdmEntityPath="$.{RootEntity}"
    storeAfSubmittedData="{Boolean}true"/>
```

- `schemaType` must be `"formdatamodel"` (all lowercase — NOT `formDataModel`); wrong case leaves the
  editor's Data Sources panel empty.
- `schemaRef` must be the full DAM path of the FDM model (produced by `create-fdm`).
- `fdmEntityPath` is the root entity bind reference — `$.{Entity}` (e.g. `$.SimpleInterestResult`).
- `actionType="fd/afaddon/components/actions/fdm"` is the **CC FDM submit action** path under `/libs`.
  Do **NOT** use `fd/af/components/guidesubmittype/restendpoint` here.

> ⚠️ **The FDM submit action is terminal — it does NOT trigger the `assign-task-to-admin` workflow.**
> The standing project rule (every form assigns a Medium-priority admin task on submit) still applies.
> Add a **Workflow Launcher** (see `create-workflow` → "OSGi Workflow Launcher") that fires on the
> submitted-data path so the admin task is created independently of the submit action.

### Canonical project example — `calculate-simple-interest`

The **`simple-interset-fdm`** / **`simple-interset-ds`** pair (committed at
`ui.content/.../conf/global/settings/cloudconfigs/fdm/simple-interset-ds` +
`ui.content/.../content/dam/formsanddocuments-fdm/simple-interset-fdm`) is the verified reference
FDM for this project:

| Artifact | Path | Notes |
|---|---|---|
| Swagger spec | `api-specs/simple-interest-servlet-swagger.yaml` | Source-of-truth; also copied into `_jcr_content/swaggerFile` |
| Data source cloud config | `ui.content/.../cloudconfigs/fdm/simple-interset-ds/` | No-auth REST; `serviceEndPoint=http://localhost:4502/`, `restfulService="swagger"` |
| FDM model | `ui.content/.../dam/formsanddocuments-fdm/simple-interset-fdm/` | Root entity `SimpleInterestResult`; model-level service `GET /calculate-simple-interest` |
| Backing servlet | `core/.../forms/submit/SimpleInterestServlet.java` | Path-bound at `/bin/aem-adaptive-forms-agents/calculate-simple-interest`; GET with `principal`, `rate`, `time` query params; returns `simpleInterest` + `totalAmount` JSON |

When creating a new FDM for a form that uses "Submit using Form Data Model", follow this same
pattern: Swagger → data source → FDM model → `guideContainer` with `actionType="fd/afaddon/components/actions/fdm"` + `fdmEntityPath` + Workflow Launcher.

Authored on a trigger (e.g. a button **Click**), the editor compiles it onto that component as
(VERIFIED, exported from an editor-created rule):
- **`fd:rules/@fd:click`** — an `EVENT_SCRIPTS` AST whose `BLOCK_STATEMENT` is a **`WSDL_STATEMENT`**:
  `onSuccess` holds a `WSDL_CALLBACK_STATEMENT` → `SET_VALUE_STATEMENT` (target `VALUE_FIELD` = the
  output field, value = `EVENT_PAYLOAD` `"{outputProp}"`); `inputModel` maps each service input to a
  form `COMPONENT`; `wsdlInfo` carries `formDataModelId` (the FDM path) + `operationName`.
- **`fd:events`** (the runtime — three keys, with **timestamped, correlated handler ids**):
  - `click="[awaitFn(retryHandler(requestWithRetry(externalize('<form>/jcr:content/guideContainer.af.dermis')\, 'POST'\, {…operationName…\,input:toString({…})\,functionToExecute:'invokeFDMOperation'\,apiVersion:'2'\,formDataModelId:'…'\,runValidation:'false'\,guideNodePath:'…/guideContainer/submit'}\, {…}\, 'custom:wsdlSuccess_<id>'\,'custom:wsdlError_<id>')))]"`
  - `custom_wsdlSuccess_<id>="[dispatchEvent($form.<Panel>.<Field>\, 'custom:setProperty'\, {value : toObject($event.payload.body).{outputProp}})]"`
  - `custom_wsdlError_<id>="[]"`

> The `custom:wsdlSuccess_<id>` / `custom:wsdlError_<id>` ids in the `fd:click` script MUST match the
> `custom_wsdlSuccess_<id>` / `custom_wsdlError_<id>` keys on `fd:events`. Because of these correlated
> ids and the verbose AST, **author the Invoke Service rule in the editor, then export the component
> node** (`…/guideContainer/{node}.infinity.json`) and convert it to docview XML — don't hand-write it.
> The Rule Editor has **no separate output column**: the output is mapped via **"Add Success Handler"**
> (a `SET_VALUE_STATEMENT` from the service `EVENT_PAYLOAD`).

## Common errors and their causes

| Symptom | Cause | Fix |
|---|---|---|
| Data Sources panel **blank**, entity **"Unbound"** | `jcr:content/sources/source1` binding node missing on the model | Add `sources/source1` with `dataSourceType` + `configurationPath` (Artifact 2) |
| **"Services" tab empty** (but data source tree shows the service) | The operation is only under an entity's `links`; no **top-level `links`** model service | Add the service as a top-level `links` node (add it in the editor + export) — see "Model-level services" |
| `AEM-FDM-001-059 … status code received - 401` on Test Model | No-auth data source calling an **auth-protected** endpoint (it sends no credentials). On AEM SDK/local the default Sling Authenticator requires auth on `/bin/` paths — the FDM service call is rejected 401. | (1) Add a `org.apache.sling.engine.impl.auth.SlingAuthenticator.cfg.json` under `ui.config/.../osgiconfig/config/` with `"sling.auth.requirements": ["-/bin/{your-servlet-path}"]` and redeploy; OR (2) Configure **Basic auth** on the data source (set in editor, encrypted). For this project the shared config at `osgiconfig/config/org.apache.sling.engine.impl.auth.SlingAuthenticator.cfg.json` adds `-/bin/aem-adaptive-forms-agents` to allow the `calculate-simple-interest` servlet to be called without auth. |
| BUILD FAILURE on the data source `.content.xml` | Encrypted `password="{…}"` read as a `{Type}value` prefix by FileVault | Escape as `password="\{…}"` |
| Data source not listed / not recognized as Swagger REST | Missing `serviceEndPoint` and/or `restfulService="swagger"`, or `selectAuthentication="No Authentication"` instead of `"None"` | Add `serviceEndPoint`, `restfulService="swagger"`; use `selectAuthentication="None"` |
| Change deployed but server still shows old binding/props | FileVault `update`-mode doesn't delete removed nodes/props | Delete-before-deploy (above) |
| Password works locally, fails on another env | Encrypted blob is keyed to the instance crypto keys | Re-enter the password in that env, or use an AEMaaCS secret/env var |
| Build WARNING: `Invalid node name 'fdm:data' … not a registered namespace prefix` | Cosmetic validator warning (pre-existing for the Salesforce/Dynamics models) | Ignore — it's a WARNING, not an error |

---

## Quality checklist

- [ ] Data source created at `/conf/{project}/settings/cloudconfigs/fdm/{name}-cloud-config`,
      resourceType `fd/fdm/gui/components/admin/fdmcloudservice/rest`
- [ ] Data source has `serviceEndPoint`, `restfulService="swagger"`, `dataSourceType="REST"`,
      `swaggerSource="swaggerFile"`, and `selectAuthentication`/`authenticationType` set correctly
      (`None`; or `Basic Authentication` + `userName` + escaped `password="\{…}"`; or **OAuth 2.0**
      for production — authored via the editor and exported, with the encrypted secret `\{…}`-escaped)
- [ ] Swagger file present (`_jcr_content/swaggerFile` + `swaggerFile.dir/.content.xml`) with host,
      basePath, schemes, operation(s) + response definitions
- [ ] FDM model at `/content/dam/formsanddocuments-fdm/{project}/{name}-data-model` (dam:Asset,
      `sling:resourceType="fd/fm/fdm/render"`, `type="formdatamodel"`, `cq:conf="/conf/{project}"`)
- [ ] **`jcr:content/sources/source1`** present with `dataSourceType` + `configurationPath` = the
      data-source name (the binding — without it the model is Unbound)
- [ ] `fdm-json` rendition has `definitions` with the entity, its `links` (service with `href` = base
      URL, input `schema`, `targetSchema`), and `properties`
- [ ] For a service that must appear in the **"Services" tab**: a **top-level `links`** node (sibling
      of `definitions`) holds the model-level service, and a top-level `properties` lists the entity —
      an entity-only `links` does NOT populate the Services tab
- [ ] FileVault `{`-prefixed values escaped as `\{` (encrypted password, `uiModel`)
- [ ] Both roots covered by `ui.content` `filter.xml`
- [ ] Deployed with the full `mvn ... -PautoInstallSinglePackage` (delete-before-deploy when iterating)
- [ ] Backing service tested directly (curl); Test Model confirmed in the editor (UI)
- [ ] No real credentials committed; encrypted dev password noted as instance-specific

---

## Related

- [[fdm-source-binding-structure]] — the verified binding/structure facts this skill encodes
- [[deploy-needs-all-package]] — why the full build is required to reach AEM
- [[schema-when-creating-forms]] — schema vs FDM are mutually exclusive
- `create-adaptive-form` — binds a form to the FDM via `schemaType="formdatamodel"` + `schemaRef`
  (FDM path) on the `guideContainer`, and `dataRef="$.{property}"` on each field (NOT `fd:formDataRef`;
  wrong-case `schemaType` leaves the editor Data Sources panel empty)
