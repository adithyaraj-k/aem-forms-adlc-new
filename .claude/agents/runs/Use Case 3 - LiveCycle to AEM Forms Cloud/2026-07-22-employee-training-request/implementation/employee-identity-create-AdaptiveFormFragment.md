# create-AdaptiveFormFragment — employee-identity

NEW fragment (confirmed: no existing Adaptive Form Fragment content in this project before this run).

## Artifacts

1. **Fragment page** — `/content/forms/af/aem-adaptive-form-agents/employee-identity`
   (`ui.content/.../jcr_root/content/forms/af/aem-adaptive-form-agents/employee-identity/.content.xml`)
   Root = `fragmentcontainer` (`aem-adaptive-forms-agents/components/adaptiveForm/fragmentcontainer`),
   `fieldType="panel"`, `fd:type="fragment"`. No submit button, no `actionType`/`thankYou*`.
2. **DAM fragment asset** — `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-identity`
   `type="affragment"` + `affragment="1"` + `sling:resourceType="fd/fm/af/render"` (NOT `formfragment`,
   NOT `type="guide"`).
3. **Per-fragment conf context** — `/conf/forms/aem-adaptive-form-agents/employee-identity`
   (`sling:Folder` + `sling:resourceType="sling:Folder"`; `SiteConfig` themeArtifact/siteTemplatePath =
   `aem-adaptive-forms-agents-training-request`; `prefixPath` matches the fragment's own page path).
4. **Filter entries** — `/conf/forms/aem-adaptive-form-agents/employee-identity` added to
   `ui.content/.../META-INF/vault/filter.xml`; `/content/forms/af/aem-adaptive-form-agents` and
   `/content/dam/formsanddocuments` already broadly covered (mode="update"), no change needed there.
5. **Validation clientlib** — NOT built. Only the `email` field has a validation (email format), and it
   uses the BUILT-IN `validatePictureClause`/`validatePictureClauseMessage` attributes — no custom
   function, so no clientlib is warranted per the skill's "only when fields have validations
   requiring a custom function" rule.

## Critical fragment-template decision (project rule 1b / gotcha (b))

`cq:template` = **`/conf/aem-adaptive-forms-agents/settings/wcm/templates/blank-af-v2-fragment`** — a
genuine FRAGMENT template (`cq:templateType` = the project's existing
`.../settings/wcm/template-types/afv2-fragment-page`, structure/initial root = `fragmentcontainer` +
`fd:type="fragment"`) — **NOT** the FORM template `blank-af-v2`. This template did not exist before
this run; it was CREATED ONCE (4 files: `.content.xml`, `structure/.content.xml`,
`initial/.content.xml`, `policies/.content.xml`) reusing the project's pre-existing
`afv2-fragment-page` template-type, and is now reused by the second fragment (`declaration-consent`)
below — no need to recreate it again.

> **Note on a contradiction inside the loaded `create-AdaptiveFormFragment` skill text itself:** the
> skill's Step-1 "Prerequisite" box says to use a genuine fragment template (create
> `blank-af-v2-fragment` if absent) — matching this project's own critical rule 1b (gotcha b) verbatim
> — but a later "Mandatory rules for Artifact 1" section in the SAME skill file says the opposite
> ("use `blank-af-v2`, NEVER `blank-af-v2-fragment`"). I followed the AUTHORITATIVE, doubly-confirmed
> guidance (this project's own critical-rule text + the skill's own Step-1 prerequisite box + the
> skill's own "Failure symptoms" table, which says a FORM-template fragment "opens BLANK" in the
> editor) over the single contradicting paragraph, which reads like a stale/erroneous line in the
> skill file. Flagging this inconsistency for the skill maintainer.

## Canonical binding shape

`$.employee.*` (id, name, email, department, managerId, managerName, location, businessUnit) — the
fragment's own default `dataRef`s. Remapped at embed time in BOTH consuming forms via the Fragment
reference node's `dataRef="$.EmployeeDetails"` override (see create-adaptive-form.md and the DoR
build note).

## Fields (8, in order, 3-column grid, all with `aria-label`)

EmployeeId (required), EmployeeName (required), Email (required, built-in email pattern), Department
(required), ManagerId (required — locked by the CONSUMING form's rule, not here), ManagerName
(required — same), Location (required, `aria-label="Employee Location"` to disambiguate from Training
Request Details' own "Location" field), BusinessUnit (optional).

## Reused by

`employee-training-request` (interactive form) AND `employee-training-request-dor` (DoR template
content) — single source of truth for this block across both artifacts.

## Fix pass 1 (post-Sentinel gate-failure #1 — 2 of 3 defects fixed in this file)

Sentinel's live verification (`testing/integration-test-report.md`, `testing/test-form-ui-report.md`)
found 3 defects; 2 required changes to THIS fragment (the third, financeComments's missing companion
`enabled` attribute, is host-form-only — see `create-form-rules.md`).

1. **Defect D-R1 (ManagerId/ManagerName lock, Rule #1) — moved INTO this fragment.** The original
   design note above ("locked by the CONSUMING form's rule, not here") turned out to be
   non-implementable: a `Change`-event script attached to the HOST's Fragment-reference wrapper,
   targeting fields nested inside this fragment, never compiled into a live event binding (confirmed
   live: no `"change"` key ever appeared under `employeeDetailsFragment/employeeId` in
   `guideContainer.model.json`). The lock is now authored directly on `managerId`/`managerName` in
   THIS file as `fd:enabled` ENABLE_EXPRESSION rules, each referencing the sibling `employeeId` field
   (no cross-fragment reference needed, since all three fields live in this same fragment) — the same
   proven top-level-`enabled` + nested-`enabled`-inside-`<fd:rules>` two-attribute mechanism that Rule
   #3 (requestType) uses successfully.
   **Deviation flag:** this fragment is no longer purely data-shape-generic — it now carries a
   form-specific business rule (lock managerId/managerName when 1<=EmployeeId<=10). Today this
   fragment has one interactive consumer (`employee-training-request`); the DoR page also embeds it
   but non-interactively (enabled/disabled state does not affect a read-only render). If this fragment
   is ever reused by a form where that lock rule should NOT apply, this rule must be revisited (e.g.
   moved to a per-consumer override once the platform supports one, or accepted as this fragment's
   fixed behaviour). Flagging for future reuse review — not a new gap introduced by this fix, but a
   trade-off made to unblock D-R1 with the only mechanism that actually compiles live.
2. **Defect (test-form-ui finding #1, "Employee Details" heading missing) — root cause found and
   fixed.** Both `hideTitle` values (host wrapper AND this fragment's own `guideContainer`) were
   already `{Boolean}false` — ruling out the "leftover hideTitle=true" hypothesis. The real cause: this
   fragment's root is a `fragmentcontainer` (Core Component
   `core/fd/components/form/fragmentcontainer/v1/fragmentcontainer`), which is the SAME class of
   root-level container as the host form's own `guideContainer` — both subject to the known Core
   Components gap (critical rule 3c) where the native title/showTitle band never emits a visible
   element. Sibling PANELS in the host form (Training Request Details, Justification, Attachments)
   render their headings fine because they are `panelcontainer` instances (a different, working,
   Core Component), not `fragmentcontainer`. Fix: `hideTitle` flipped to `{Boolean}true` (suppress the
   dead band) and an explicit AF Title (v2) `sectionTitle` component added as the first child,
   mirroring the host form's own `formTitle` workaround exactly. Styled via a new `.etr-section-title`
   CSS rule in `employee-training-request-clientlib/css/form.css` reusing the same section-heading
   tokens the sibling panels use, so it matches their maroon/banded look.

Both fixes verified: XML well-formed (parsed via .NET `XmlDocument`), and the new `fd:enabled` JSON
blobs verified as valid parseable JSON (unlike the pre-existing rule3/rule6 blobs in the host form,
which contain literal FileVault `\,` escape sequences that only resolve during JCR package import —
confirmed as a pre-existing, evidently-harmless characteristic of this project's rule authoring, not
something these fixes needed to replicate exactly).
