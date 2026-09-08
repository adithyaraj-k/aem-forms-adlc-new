# Composer — ASSEMBLY summary · patient-registration-form

**Run:** 2026-07-17-patient-registration-form · **Lead:** composer (ASSEMBLY)
**Pipeline position:** formwright → bridgesmith → **composer** → auditron → sentinel.
**Mode:** AUTHOR ONLY — no `mvn` build/deploy (centralized in Auditron; the `composer` skill was
invoked with "author only — defer the build+deploy to Auditron").

## Inputs verified
- `implementation/formwright.md` — form built: `/content/forms/af/aem-adaptive-form-agents/patient-registration-form`; guideContainer `clientLibRef` = `aem-adaptive-forms-agents.forms.patient-registration-form` (single category).
- `implementation/bridgesmith.md` — integration wired (shared PDF submit + shared assign-task-to-admin). Form complete → safe to embed.
- `.aem-forms-config.yaml` — project `aem-adaptive-forms-agents`; form/DAM/conf segment `aem-adaptive-form-agents`.

## Page
- **`/content/aem-adaptive-forms-agents/us/en/test-adaptive-form`** — **reused** (already existed).
- Reuses project page component `aem-adaptive-forms-agents/components/page` and template
  `/conf/aem-adaptive-forms-agents/settings/wcm/templates/page-content`. No new page/template forked.

## Embed (single INLINE AEM Form Container)
- `sling:resourceType` = `aem-adaptive-forms-agents/components/aemformscontainer`
- `useiframe="false"` (INLINE embed — never iframe), no `height`.
- `formRef` (NEW) = `/content/dam/formsanddocuments/aem-adaptive-form-agents/patient-registration-form` (DAM guide-asset path).
- `formType="af"`, `loadType="embed"`.
- Authored in the editable container `root/container/container` (not the locked `root/container`).

## Host-page wiring (page jcr:content — repointed with formRef)
- `formEmbedThemePath` (NEW) = `/content/forms/af/aem-adaptive-form-agents/patient-registration-form` (runtime path).
- `formEmbedClientlibs` (NEW) = `[aem-adaptive-forms-agents.forms.patient-registration-form]` (guideContainer clientLibRef, verbatim).
- Consumed by the page component's `customheaderlibs.html` / `customfooterlibs.html` — mandatory for an inline form to be styled + functional.

## Replacement — Replace, don't accumulate
Page pre-existed embedding `college-admission-registration`; all THREE references repointed together:

| Property | OLD (replaced) | NEW |
|---|---|---|
| `formRef` | `.../aem-adaptive-form-agents/college-admission-registration` (DAM) | `.../aem-adaptive-form-agents/patient-registration-form` (DAM) |
| `formEmbedThemePath` | `/content/forms/af/aem-adaptive-form-agents/college-admission-registration` | `/content/forms/af/aem-adaptive-form-agents/patient-registration-form` |
| `formEmbedClientlibs` | `[aem-adaptive-forms-agents.college-admission-registration]` | `[aem-adaptive-forms-agents.forms.patient-registration-form]` |

**Container count = 1.** Old form no longer referenced anywhere; no second container added.

## Filter
`ui.content` covers the page path — explicit `<filter root="/content/aem-adaptive-forms-agents/us/en/test-adaptive-form" mode="update"/>` present (plus the broader `/content/aem-adaptive-forms-agents` root). Page will deploy.

## Deploy
**Deferred to Auditron** (pipeline) — no `mvn` executed in Composer. Auditron's single
`mvn clean install -PautoInstallSinglePackage` after this phase deploys the updated page too; Sentinel
then tests the form as it renders inside the page.

## Gate: PASS
Page embeds the NEW form via exactly one INLINE container; `formRef` + `formEmbedThemePath` +
`formEmbedClientlibs` all repointed to patient-registration-form; old form gone; page/template reused;
page path in a `ui.content` filter.
