# Phase 06 — create-submit-action (Bridgesmith / IMPL integration)

- **Run:** 2026-07-01-vehicle-registration-form
- **Phase:** 6 — Submit Action (Document of Record)
- **Lead:** bridgesmith (IMPL integration)
- **Mode:** AUTHOR ONLY — deployment deferred to Auditon
- **Requirement:** US-10 / AC-10.2 — the form must generate a PDF Document of Record on submit.
- **Outcome:** The SHARED, generic `Custom-Submit-GeneratePDF` action was **already present** and
  correctly wired to the vehicle-registration-form. **No new shared artifacts created.** One missing
  quality-checklist item (a dedicated unit test for the shared action) was authored — no duplication
  of the shared service / servlet / clientlib and no per-form action.

## Verdict: PASS — shared action fully present and correctly reused

## Shared-action gate — item-by-item results

| # | Gate check | Result | Evidence |
|---|-----------|--------|----------|
| 1 | BOTH the OSGi service AND the JCR node exist | **PASS** | Service `CustomSubmitGeneratePDFAction.java` + node `.../Custom-Submit-GeneratePDF/.content.xml` both present |
| 2 | `getServiceName()` (Java) === JCR `submitService` === `"Custom-Submit-GeneratePDF"` | **PASS** | Java `SERVICE_NAME = "Custom-Submit-GeneratePDF"` (line 41) == node `submitService="Custom-Submit-GeneratePDF"` (line 9) |
| 3 | JCR node is `sling:Folder` with `guideComponentType="fd/af/components/guidesubmittype"` + `submitService` set | **PASS** | `jcr:primaryType="sling:Folder"`, `guideComponentType="fd/af/components/guidesubmittype"`, `guideDataModel="basic,xfa,xsd"`, `submitService` set; also carries `sling:resourceType="{project}/Custom-Submit-GeneratePDF"` (canonical shared-action node) |
| 4 | Single shared `GeneratePDFServlet`, POST-bound; ResourceResolver try-with-resources; no hardcoded secrets | **PASS** | `sling.servlet.methods=POST`, `sling.servlet.paths=/bin/{project}/generatePDF`; DAM write uses service-user resolver in try-with-resources (line 146); no secrets; zero new Maven dependency (hand-written PDF 1.4 + Helvetica) |
| 5 | Shared clientlib exists with category `{project}.forms.generate-pdf` | **PASS** | `clientlib-generate-pdf/.content.xml` → `categories="[{project}.forms.generate-pdf]"`, `allowProxy="{Boolean}true"`, `dependencies="[core.forms.components.runtime.all]"`; `js.txt` → `generate-pdf.js`; JS POSTs to `/bin/{project}/generatePDF` (**matches** the servlet path exactly) |
| 6 | vehicle-registration-form guideContainer wired to the shared action (same pattern as deployed health-insurance-form) | **PASS** | `submitService="Custom-Submit-GeneratePDF"`, `actionType="fd/af/components/guidesubmittype/submitservice"`, `clientLibRef` includes `{project}.forms.generate-pdf` |

## Existing shared artifacts (verified, reused — NOT recreated)

- **OSGi service (FQN):** `com.aem.forms.agents.forms.submit.CustomSubmitGeneratePDFAction`
  - `getServiceName()` → `"Custom-Submit-GeneratePDF"`; `submit()` returns
    `GuideConstants.FORM_SUBMISSION_COMPLETE = Boolean.TRUE`; never throws.
  - File: `core/src/main/java/com/aem/forms/agents/forms/submit/CustomSubmitGeneratePDFAction.java`
- **Shared servlet (FQN):** `com.aem.forms.agents.core.servlets.GeneratePDFServlet`
  - POST-bound, path `/bin/{project}/generatePDF`; renders PDF, uploads to DAM
    under `/content/dam/{project}` (service-user, try-with-resources), streams back
    with `Content-Disposition: attachment` for auto-download.
  - File: `core/src/main/java/com/aem/forms/agents/core/servlets/GeneratePDFServlet.java`
- **JCR submit-action node:** `/apps/{project}/Custom-Submit-GeneratePDF`
  - File: `ui.apps/src/main/content/jcr_root/apps/{project}/Custom-Submit-GeneratePDF/.content.xml`
- **Shared clientlib:** `/apps/{project}/clientlibs/clientlib-generate-pdf`
  (category `{project}.forms.generate-pdf`)
  - Files: `.content.xml`, `js.txt`, `js/generate-pdf.js`
- **Service user + mapping (shared, one set):**
  - repoinit: `ui.config/.../osgiconfig/config/org.apache.sling.jcr.repoinit.RepositoryInitializer~{project}.cfg.json`
    creates service user `pdf-writer-service` (path `system/{project}`) with
    `jcr:read,rep:write` on `/content/dam/{project}`.
  - mapping: `...ServiceUserMapperImpl.amended~{project}-pdf.cfg.json` →
    `{project}.core:pdf-writer=pdf-writer-service`. Bundle symbolic name matches
    the core module artifactId `{project}.core`; subservice `pdf-writer` matches the
    servlet's `SUBSERVICE`.
- **filter.xml (ui.apps):** `<filter root="/apps/{project}/Custom-Submit-GeneratePDF"/>`
  and `<filter root="/apps/{project}/clientlibs"/>` (covers the shared clientlib) both present.

## Artifact authored this phase (fix — the one missing quality-checklist item)

- **Unit test:**
  `core/src/test/java/com/aem/forms/agents/forms/submit/CustomSubmitGeneratePDFActionTest.java`
  - Asserts `getServiceName() == "Custom-Submit-GeneratePDF"` (the load-bearing identity check),
    that a submit signals `FORM_SUBMISSION_COMPLETE = TRUE`, and that `submit(null)` never throws
    and always returns a completion flag. Uses AEM Mocks + Mockito (project convention).
  - **No OSGi `.cfg.json` was created** — the shared action has no `@Designate`/`@ObjectClassDefinition`,
    so it needs no configuration (correct for a generic, config-free action).

## Exact guideContainer wiring proving US-10 is satisfied

`ui.content/src/main/content/jcr_root/content/forms/af/{appFolder}/vehicle-registration-form/.content.xml`
(`jcr:content/guideContainer`):

- `submitService="Custom-Submit-GeneratePDF"`
- `actionType="fd/af/components/guidesubmittype/submitservice"`
- `clientLibRef="{project}.forms.vehicle-registration-form,{project}.forms.generate-pdf"`
- `thankYouOption="message"` / `thankYouMessage="Thank you. Your vehicle registration details have been submitted."`

Same proven wiring pattern as the deployed health-insurance-form
(`actionType="fd/af/components/guidesubmittype/submitservice"`), with the PDF clientlib category
appended to `clientLibRef` so `generate-pdf.js` loads on this form and drives the DoR download.

## For Auditon (build/deploy) to verify

- Run the single authoritative `mvn clean install -PautoInstallSinglePackage` (no per-module build).
- New test compiles/passes: `CustomSubmitGeneratePDFActionTest` (core module).
- Confirm the shared bundle registers `CustomSubmitGeneratePDFAction` (FormSubmitActionService) and
  `GeneratePDFServlet` after deploy; repoinit creates `pdf-writer-service` and the ACL on
  `/content/dam/{project}`.

## For Sentinel (test) to verify on the running form

- On submit of vehicle-registration-form: the browser auto-downloads a PDF (`Content-Disposition:
  attachment`) containing the filled fields as `aria-label: value` lines, a copy is written to
  `/content/dam/{project}/`, and the form shows the configured thank-you message.
- Confirm `generate-pdf.js` actually loads on the form (clientLibRef category present) and the POST to
  `/bin/{project}/generatePDF` returns 200 with a PDF body.

## Result

existed-or-created: **EXISTED** (shared action fully present and correctly reused) — plus one authored
unit test for the shared action (quality-checklist completion, no duplication, no per-form action).
