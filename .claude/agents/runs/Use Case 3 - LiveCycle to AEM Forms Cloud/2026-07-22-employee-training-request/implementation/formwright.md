# Formwright — IMPL-BUILD Package
## Employee Training Request — LiveCycle → AEM as a Cloud Service (Path B migration)

Run: `.claude/agents/runs/2026-07-22-employee-training-request/`
Reads: `plan/planwright.md`, `plan/user-stories.yaml`, `plan/solution-architecture.yaml`,
`design/draftsmith.md`, `design/component-design-spec.yaml`

---

## 1. What was built

| Artifact | JCR path | Notes |
|---|---|---|
| **Schema** | `/content/dam/formsanddocuments/schema/employee-training-request.schema.json` (+ `.bindings.json`) | From `employeeTrainingRequest.xsd`; PascalCase legacy node names preserved (deviation, documented); `enum:[true]` not `const` for AcceptedTerms |
| **Fragment template (NEW, once, reused)** | `/conf/aem-adaptive-forms-agents/settings/wcm/templates/blank-af-v2-fragment` | Genuine fragment template (`afv2-fragment-page` type, `fragmentcontainer` root, `fd:type="fragment"`) — did not exist before this run |
| **Fragment: employee-identity** | `/content/forms/af/aem-adaptive-form-agents/employee-identity` (+ DAM asset + conf context) | 8 fields; originally built with a `$.employee.*` generic shape — **CORRECTED in fix pass 2 (defect D3)** to bind to the real schema roots `$.EmployeeDetails.*` (see §"Fix pass 2" below) |
| **Fragment: declaration-consent** | `/content/forms/af/aem-adaptive-form-agents/declaration-consent` (+ DAM asset + conf context) | 2 fields, `hideTitle=true` (no header band); originally built with a `$.declaration.*` generic shape — **CORRECTED in fix pass 2 (defect D3)** to bind to the real schema root `$.Declaration.*` (see §"Fix pass 2" below) |
| **Template** | `/conf/aem-adaptive-forms-agents/settings/wcm/templates/blank-af-v2` | REUSED — `create-editable-template` skipped |
| **Form: employee-training-request** | `/content/forms/af/aem-adaptive-form-agents/employee-training-request` (+ DAM asset + conf context) | 6 panels, 30 fields, embeds both fragments by reference |
| **8 business rules** | authored on the form's `.content.xml` (`fd:rules`/`fd:events`) | See create-form-rules.md for per-rule mechanism + confidence level |
| **Form clientlib (NEW)** | `/apps/clientlibs/employee-training-request-clientlib` | Self-contained; 1 copied function (`validateDateAfter`) + copied base styling + brand-token mirror |
| **Base clientlib (extended, additive)** | `/apps/aem-adaptive-forms-agents/clientlibs/clientlib-forms-base` | 2 new token contract entries (`--af-section-heading-bg`, `--af-panel-bg`) + `--af-heading-size` promoted to a token; neutral defaults, non-breaking for other forms |
| **Theme (NEW)** | `/apps/fd/af/themes/aem-adaptive-forms-agents-training-request` (+ DAM theme-json) | Exact-replica token override; reused (not rebuilt) for the DoR |
| **DoR template content** | `/content/forms/af/aem-adaptive-form-agents/employee-training-request-dor` (+ DAM asset + conf context) | Non-interactive; `formPath=` target for Groundsmith's future `GenerateDocumentOfRecordStep` |

Per-phase run notes: `generate-schema.md`, `employee-identity-create-AdaptiveFormFragment.md`,
`declaration-consent-create-AdaptiveFormFragment.md`, `create-adaptive-form.md`,
`create-form-rules.md`, `create-form-theme.md`, `dor-template-build.md` (all in this same folder).

## 2. Deviations from the DESI/Planwright plan (all documented, all justified)

1. **Legacy PascalCase schema node names preserved** (not camelCase) — required for workflow/DoR
   cross-compatibility with the legacy XSD node names (per the build brief's explicit instruction).
2. **`clientLibRef` is a SINGLE category, not a comma-separated base+form-specific list** — the
   more detailed, defect-verified `create-form-clientlib`/`create-adaptive-form` skill guidance
   overrides the simplified top-level convention: a multi-value `clientLibRef` breaks the
   Rule-Editor customfunctions endpoint. The form clientlib is SELF-CONTAINED instead (copies base
   styling + the one function it needs).
3. **A new form-specific clientlib WAS built** (`employee-training-request-clientlib`), despite
   DESI's default-skip plan — needed solely to host a self-contained copy of the base's
   `validateDateAfter` function for the EndDate>StartDate rule (email + AcceptedTerms both turned
   out to need no clientlib, confirming the rest of DESI's plan).
4. **Base clientlib extended additively** with 2 new token-contract entries (`--af-section-heading-bg`,
   `--af-panel-bg`) to support this form's banded-header/coloured-panel-body exact-replica design —
   neutral (`transparent`) defaults mean zero visual change for the ~20 other forms sharing the base.
5. **Fragment template `blank-af-v2-fragment` created** — did not exist in this project before this
   run (DESI/Planwright correctly flagged "no existing fragment" but a fragment TEMPLATE also had to
   be created once, reusing the project's pre-existing `afv2-fragment-page` template-type).
6. **DoR content built as a SEPARATE page**, not by enabling `dorType` on the interactive form itself
   — because the DoR must EXCLUDE the Attachments panel the interactive form has, and this project's
   proven DoR mechanism (`GenerateDocumentOfRecordStep` `formPath=`, already used by
   health-insurance-underwriting/life-insurance-underwriting) renders whatever page it points at
   wholesale. See dor-template-build.md.

## 3. Gaps carried forward (NOT resolved here — business decisions, flagged for Groundsmith/business)

1. **Manager auto-assign lookup** — rule #1's LOCK structure is built (employeeId ≤ 10 →
   managerId/managerName disabled), but the VALUE population is deliberately left for Groundsmith's
   prefill-service (ask-then-build flow). Do NOT hardcode 'venuga5'/'Arjun Venugopal'.
2. **Finance-manager approval participant** — unresolved in the source (same ManagerId as the
   manager step); Groundsmith's `create-workflow` must get a real answer before wiring the Assign
   Task step for finance-manager approval.
3. **Email sender address** — legacy `rs.gbseforms@medtronic.com` needs a real cloud sender
   identity; Groundsmith's Day CQ Mail Service config.

## 4. Lowest-confidence build items (flag for Sentinel/Forgemaster verification)

- **Rule #1** (ManagerId/ManagerName lock) and **Rule #5** (Approval Information panel show/hide) are
  this project's FIRST rules that reference fields NESTED INSIDE an embedded Fragment from the HOST
  FORM's rule scope (`$form.employeeDetailsFragment.employeeId`). No prior in-repo precedent exists
  for this pattern — the AST constructions are best-effort, adapted from the fully-verified
  `fd:visible`/`fd:validate` shapes. **Must be verified live** in the Rule Editor +
  `guideContainer.model.json` after Forgemaster's deploy; round-trip through the editor if either
  rule does not surface correctly.
- **Rule #3** and **Rule #6** use `fd:enabled` (ENABLE_EXPRESSION) — marked "follow-the-pattern" (not
  yet demonstrated elsewhere in this repo) by the `create-form-rules` skill itself. Same
  live-verification caveat applies.

## 5. User story coverage

| Story | Satisfied by |
|---|---|
| US-01 (submit all required fields → workflow) | Form + both fragments + default workflow submit wiring (Groundsmith repoints the model) |
| US-02 (manager auto-assign+lock) | Rule #1 LOCK structure built; VALUE deferred to Groundsmith (flagged gap, not a build failure) |
| US-03 (email format validation) | Built-in `validatePictureClause` on the fragment's email field |
| US-04 (RequestType locked until RequestId) | Rule #3 |
| US-05 (EndDate after StartDate) | Rule #4 + `employee-training-request-clientlib` |
| US-06 (manager approve/reject routing) | Rule #5 (panel show/hide); routing itself is Groundsmith's workflow |
| US-07 (FinanceComments unlock post-manager-approval) | Rule #6 |
| US-08 (finance approval → DoR + email) | DoR template content built; workflow step wiring is Groundsmith's |
| US-09 (AcceptedTerms required + SubmissionDate set) | Built-in required (fragment) + Rule #8 |
| US-10 (exact visual parity) | Theme + form clientlib; pixel verification is Sentinel's (test-form-ui) |
| US-11 (DoR merges request-data + approval-decision) | `employee-training-request-dor` page built, reusing both fragments |

**0 build-side stories unsatisfied.** US-01, US-06, US-08 have an integration-only remainder
(workflow model, real submit wiring) that is explicitly Groundsmith's phase, not a build gap.

## 6. Zero-defect pre-handoff checklist

- [x] All screenshot visuals reproduced — no separators/dividers (DESI confirmed none exist in the
      source — intentional, evidence-based), no icons/logos (icon_assets:[] confirmed), section-heading
      band + panel-body colour block built via `--af-section-heading-bg`/`--af-panel-bg`.
- [x] Title via explicit AF Title component (both the interactive form and the DoR page).
- [x] Required red asterisks — inherited via the base/form clientlib's `data-cmp-required` styling.
- [x] Submit gated on validation — native `submitForm()` (Core Components standard); no PDF-before-
      validation risk (DoR is generated mid-workflow post-approval, not on initial submit).
- [x] Clientlib split — SELF-CONTAINED single-category form clientlib (deviation documented in §2.2).
- [x] Theme is a thin token override (Option A) — confirmed in create-form-theme.md.
- [x] Multi-column layout — panels use the project's existing shared grid mechanism
      (`clientlib-forms-base` + `clientlib-grid`), same as every other form in this project; not
      modified here.
- [x] Brand colours in the embedded page — mirrored into the form-specific clientlib's `:root`.
- [x] `dataRef` JSONPath bindings used throughout (never `fd:formDataRef` on authored fields).
- [ ] Submit-action node under `fd/af/submitactions` — N/A yet; default shared-workflow action is in
      place per rule 0, Groundsmith wires the real submit/workflow model.
- [ ] PDF empty-content guard — N/A; no direct PDF submit action on this form (DoR via workflow).
- [x] `fd:rules` AST correctness — no bare-string `fd:click`; all `fd:*` properties are escaped-JSON
      arrays. 2 rules flagged lowest-confidence (see §4) pending live verification.
- [x] Schema importer-safety — `enum:[true]` not `const`; only supported keywords used.
- [x] Footer buttons use `actions/submit`/`actions/reset` (not the generic `button`).
- [x] Date pickers — `displayFormat`==`editFormat` (`date|DD/MM/YYYY`) on every datepicker (form +
      DoR page).
- [x] Date picker native+selectable — inherited from base/form clientlib (no `appearance:none`).
- [x] Validation-failure state red — inherited from base/form clientlib.

## 7. Flags for Groundsmith (read before starting integration)

1. Repoint `employee-training-request`'s `guideContainer/@workflowModel` from the default
   `/var/workflow/models/assign-task-to-admin` to the real
   `/conf/global/settings/workflow/models/employee-training-request-approval` (or wherever you build
   it) once the workflow model exists.
2. Build the manager-lookup prefill service (ask-then-build) targeting
   `$form.employeeDetailsFragment.managerId` / `.managerName` — the lock rule (build-time, rule #1)
   is already in place; you supply the VALUE.
3. Wire the Document-of-Record step with `formPath=/content/forms/af/aem-adaptive-form-agents/employee-training-request-dor`
   (see dor-template-build.md) — NOT the interactive form's own path.
4. Resolve the finance-manager approval participant (open question — do not reuse ManagerId).
5. Confirm the real Day CQ Mail Service sender identity (legacy address is a placeholder).

## Run metrics

- `time_taken_minutes`: ~92.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~48,000
  - `read`: ~95,000 (PLAN/DESI yaml/md, XSD, existing form/theme/clientlib/template reference reads)
  - `write`: ~62,000 (schema, 2 fragments, template, form, DoR page, theme, clientlibs, run notes)
  - `other`: ~15,000 (tool-call overhead, PowerShell zip build, bash scans)
  - `total`: ~220,000

---

## Fix pass 1 (post-Sentinel gate-failure #1)

Sentinel ran live verification against the deployed form and found 3 concrete defects — 2 of the
build's own 8 rules ("lowest-confidence" per §4 above, exactly as flagged) plus 1 fragment-rendering
gap that only surfaced via the visual UI-parity check. Groundsmith is separately fixing 2 unrelated
integration defects (submit-action attachment failure, workflow OR-split branching) in parallel — not
in scope here.

| Defect | What Sentinel found | Root cause | Fix |
|---|---|---|---|
| D-R1 — ManagerId/ManagerName lock (Rule #1) | `Change`-event script on the host's Fragment-reference wrapper never compiled into a live event binding; fields stayed permanently `enabled:true` | Event-script rules targeting fields nested inside an embedded fragment, attached to the host wrapper node, do not produce a live binding in this project (mechanism-level issue, not a JS-content bug) | Replaced with two `fd:enabled` ENABLE_EXPRESSION rules authored directly on `managerId`/`managerName` **inside the `employee-identity` fragment itself**, referencing sibling `employeeId` (no cross-fragment reference needed) — same mechanism class as the working Rule #3 |
| D-R6 — FinanceComments lock (Rule #6) | Renders as a static, non-reactive value; never re-evaluates as `CurrentStatus` changes | `<fd:rules>` element was missing the companion `enabled` mirror attribute (sibling of `fd:enabled`) that the working `requestType` (Rule #3) carries — this nested attribute, not just the top-level field attribute, is what the AF runtime needs to produce a live `"rules":{"enabled":...}` binding | Added the missing nested `enabled="currentStatus.$value == \"ManagerApproved\""` attribute inside `financeComments`'s `<fd:rules>` element |
| "Employee Details" heading missing (test-form-ui finding #1) | JCR shows `visible:true`/`hideTitle:false` but no heading ever rendered in the DOM | `employee-identity`'s root is a `fragmentcontainer` — the SAME class of root-level container as a form's own `guideContainer`, both subject to the known Core Components gap (critical rule 3c) where the native title band never emits a visible element. Sibling PANELS render fine because they are `panelcontainer` instances (a different, working, Core Component) | Flipped `hideTitle` to `true` (suppress the dead band) and added an explicit AF Title (v2) `sectionTitle` component as the fragment's first child, mirroring the host form's own `formTitle` workaround; styled via a new `.etr-section-title` CSS rule in the form clientlib reusing the existing section-heading tokens |

**Files touched:**
- `ui.content/.../content/forms/af/aem-adaptive-form-agents/employee-training-request/.content.xml` —
  removed the broken `fd:rules`/`fd:events` from `employeeDetailsFragment`; added the missing nested
  `enabled` attribute to `financeComments`'s `<fd:rules>`.
- `ui.content/.../content/forms/af/aem-adaptive-form-agents/employee-identity/.content.xml` — added
  `sectionTitle` (AF Title v2), flipped `hideTitle` to true, added `fd:enabled` rules to `managerId`/
  `managerName`.
- `ui.apps/.../apps/clientlibs/employee-training-request-clientlib/css/form.css` — added
  `.etr-section-title`/`.etr-section-title .cmp-title__text` rules.

**Deviation flag (carried forward):** the `employee-identity` fragment is no longer purely
data-shape-generic — it now carries the ManagerId/ManagerName lock business rule, which is specific to
`employee-training-request`'s US-02. This was the only mechanism found that compiles into a live
binding (event-scripts crossing the fragment boundary from the host do not). Flagged for review if this
fragment is ever reused by a form where that lock should not apply.

**Verification performed before handoff:**
- Both edited `.content.xml` files parsed successfully as well-formed XML (.NET `XmlDocument`).
- The two new `fd:enabled` JSON blobs (managerId, managerName) were round-tripped through
  `ConvertFrom-Json`/`ConvertTo-Json` and confirmed to parse as valid JSON both before and after
  embedding in the file.
- Both the top-level `enabled` mirror attribute and the nested `<fd:rules enabled="...">` attribute
  were confirmed present and identical (extracted via XML DOM) for `managerId`, `managerName`, and
  `financeComments`.
- Did not have live access to the AEM author instance in this pass to re-check
  `guideContainer.model.json` directly (no running instance reachable this session) — this is flagged
  for Forgemaster's redeploy + Sentinel's retest to do the final live confirmation, per this task's
  instruction that "full confirmation will come from Sentinel's retest."
- Not touched: Rules #2, #3, #4, #5, #7, #8; the theme; the DoR page; Groundsmith's 2 separate defects.

**Fix pass 1 run metrics:**
- `time_taken_minutes`: ~35.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~9,000
  - `read`: ~28,000 (prior run docs, both .content.xml files re-read multiple times, CSS file)
  - `write`: ~11,000 (3 source-file edits, JSON scratch files, 3 run-doc updates)
  - `other`: ~6,000 (PowerShell JSON/XML validation calls)
  - `total`: ~54,000

---

## Handoff YAML

```yaml
agent: formwright
phase: IMPL-build
status: PASSED
time_taken_minutes: 92.0
tokens_consumed:
  cli_text: 48000
  read: 95000
  write: 62000
  other: 15000
  total: 220000
delivery: brownfield
data_backing: schema
phases_executed: [1, 3, 4, 8]
phases_skipped: [2, 5, 9]
phases_skipped_reasons:
  2: "create-editable-template — reuse blank-af-v2, per solution-architecture.yaml"
  5: "create-form-component — no custom field type needed, no DSC/custom-code to migrate"
  9: "create-form-clientlib — DESI default-skip mostly held; a MINIMAL self-contained clientlib was
      still built (deviation, see formwright.md §2.3), not the full clientlib DESI would have skipped"
core_components_reused: 30
custom_components_built: 0
fragments_built: 2
user_stories_satisfied: 8
user_stories_deferred_to_groundsmith: 3   # US-01, US-06, US-08 — integration-only remainder (workflow model, real submit wiring)
user_stories_unsatisfied: 0
artifacts:
  schema_or_fdm: "/content/dam/formsanddocuments/schema/employee-training-request.schema.json"
  template: reused
  components: []
  fragments:
    - "/content/forms/af/aem-adaptive-form-agents/employee-identity"
    - "/content/forms/af/aem-adaptive-form-agents/declaration-consent"
  form: "/content/forms/af/aem-adaptive-form-agents/employee-training-request"
  dor_content: "/content/forms/af/aem-adaptive-form-agents/employee-training-request-dor"
  clientlib: "/apps/clientlibs/employee-training-request-clientlib"
  theme: "/apps/fd/af/themes/aem-adaptive-forms-agents-training-request"
build_summary: ".claude/agents/runs/2026-07-22-employee-training-request/implementation/formwright.md"
gate_result: PASS
next: aem-forms-program-agent runs groundsmith (integration: prefill lookup, workflow model + DoR step wiring, finance-approver resolution) -> forgemaster (build/deploy) -> sentinel (test, incl. live verification of the 2 lowest-confidence rules)
```

## Handoff YAML — Fix pass 1 (post-Sentinel gate-failure #1)

```yaml
agent: formwright
phase: IMPL-build (fix pass 1)
status: FIXES_APPLIED
time_taken_minutes: 35.0
tokens_consumed:
  cli_text: 9000
  read: 28000
  write: 11000
  other: 6000
  total: 54000
defects_fixed:
  - id: D-R1
    description: "ManagerId/ManagerName lock rule (Rule #1) never compiled into a live event binding"
    fix: "Replaced host-wrapper fd:change event-script with two fd:enabled ENABLE_EXPRESSION rules authored directly on managerId/managerName inside the employee-identity fragment (sibling reference to employeeId, same mechanism class as working Rule #3)"
    files: ["employee-identity/.content.xml", "employee-training-request/.content.xml"]
  - id: D-R6
    description: "FinanceComments lock rule (Rule #6) rendered as a static, non-reactive value"
    fix: "Added the missing companion `enabled` attribute as a sibling of fd:enabled inside financeComments's <fd:rules> element, matching the working requestType (Rule #3) structure"
    files: ["employee-training-request/.content.xml"]
  - id: "UI-parity finding #1"
    description: "Employee Details fragment section heading missing from render despite visible:true in JCR"
    fix: "employee-identity's fragmentcontainer root suffers the same known title-band gap as a form's own guideContainer (critical rule 3c); hideTitle flipped to true and an explicit AF Title (v2) sectionTitle component added, styled via new .etr-section-title CSS in the form clientlib"
    files: ["employee-identity/.content.xml", "employee-training-request-clientlib/css/form.css"]
verification_performed:
  - "Both edited .content.xml files parsed as well-formed XML (.NET XmlDocument)"
  - "New fd:enabled JSON blobs (managerId, managerName) round-tripped through ConvertFrom-Json/ConvertTo-Json — valid"
  - "Top-level `enabled` mirror attribute + nested <fd:rules enabled=...> attribute confirmed present and identical for managerId, managerName, financeComments (extracted via XML DOM)"
verification_not_performed:
  - "No live AEM author instance reachable this session — guideContainer.model.json was NOT re-checked live; deferred to Forgemaster's redeploy + Sentinel's retest for final confirmation"
scope_respected: "Only the 3 flagged defects were touched. Rules #2/#3/#4/#5/#7/#8, the theme, the DoR page, and Groundsmith's 2 separate integration defects (D1 submit-action failure, D2 workflow OR-split branching) were left untouched, per task constraint."
deviation_flags:
  - "employee-identity fragment is no longer purely data-shape-generic — it now carries the ManagerId/ManagerName lock business rule (US-02-specific). This was the only mechanism that compiles into a live binding. Flag for review if this fragment gains another consumer that should not have this lock."
confidence: "High for D-R6 (direct root-cause match against a proven working sibling pattern, single-attribute fix). Medium-high for D-R1 and the heading fix (correct mechanism per project's own critical rules and proven sibling patterns, JSON/XML verified well-formed, but not re-confirmed against a live guideContainer.model.json in this pass)."
build_summary: ".claude/agents/runs/2026-07-22-employee-training-request/implementation/formwright.md"
gate_result: FIXES_APPLIED_PENDING_RETEST
next: aem-forms-program-agent -> forgemaster (rebuild/redeploy, after groundsmith's parallel fixes land) -> sentinel (retest all 3 defects + full regression of rules #2-#8)

---

## Fix pass 2 (post-Sentinel gate-failure #2)

Sentinel's retest of the 5-item fix-pass-1 scope found one NEW Critical defect (D3, surfaced only by
inspecting a real, browser-driven submission's `data.xml` — the first genuine fragment-rendering
submission this delivery had run) and corrected a previously-mis-verified test (TC-019, which had only
been checked for the rule's *presence* in `guideContainer.model.json`, never actually exercised in a
browser). Both are contained, scoped code fixes; nothing else was touched.

### D3 — fragment fields serialized to the wrong schema paths

**Root cause:** both reusable fragments (`employee-identity`, `declaration-consent`) were originally
authored with an internal generic data shape (`$.employee.*`, `$.declaration.*`) rather than the actual
schema's root property names (`EmployeeDetails.*`, `Declaration.*` — confirmed against
`/content/dam/formsanddocuments/schema/employee-training-request.bindings.json`). Both fragments are
embedded into their host (the interactive form, and the DoR template) via a `fragmentcontainer` wrapper
node that itself carries a context-setting `dataRef="$.EmployeeDetails"` / `dataRef="$.Declaration"` —
but every field INSIDE the fragment used an **absolute** `dataRef="$.…"`, which resolves from the
document root and bypasses the wrapper's relative context entirely. The result: the wrapper's intended
root (`EmployeeDetails`/`Declaration`) stayed empty on every real submission, while the actual data
landed under sibling, unschema'd `employee.*`/`declaration.*` keys — confirmed directly in Sentinel's
captured `data.xml`.

**Fix:** corrected all 10 field-level `dataRef` attributes to the schema's real absolute paths:

| Fragment | Field | Old `dataRef` | New `dataRef` |
|---|---|---|---|
| employee-identity | employeeId | `$.employee.id` | `$.EmployeeDetails.EmployeeId` |
| employee-identity | employeeName | `$.employee.name` | `$.EmployeeDetails.EmployeeName` |
| employee-identity | email | `$.employee.email` | `$.EmployeeDetails.Email` |
| employee-identity | department | `$.employee.department` | `$.EmployeeDetails.Department` |
| employee-identity | managerId | `$.employee.managerId` | `$.EmployeeDetails.ManagerId` |
| employee-identity | managerName | `$.employee.managerName` | `$.EmployeeDetails.ManagerName` |
| employee-identity | location | `$.employee.location` | `$.EmployeeDetails.Location` |
| employee-identity | businessUnit | `$.employee.businessUnit` | `$.EmployeeDetails.BusinessUnit` |
| declaration-consent | acceptedTerms | `$.declaration.acceptedTerms` | `$.Declaration.AcceptedTerms` |
| declaration-consent | submissionDate | `$.declaration.submissionDate` | `$.Declaration.SubmissionDate` |

Since the fix fully resolves each field to its correct absolute schema path, no wrapper-level change was
needed on either host (`employee-training-request` or `employee-training-request-dor`) — both hosts'
`fragmentcontainer` `dataRef` attributes were already `$.EmployeeDetails` / `$.Declaration` and now
agree with what the fragments actually write. **Fixing the fragments once fixes both consumers** (the
interactive form's real submissions AND the DoR template's rendering), since both embed the identical
fragment by reference.

**Deviation flag (superseding the fix pass 1 note and the original design intent):** critical rule 1b
calls for a fragment to bind to a generic, schema-agnostic shape so it is portable across forms with
different schemas. This fix intentionally binds both fragments to THIS schema's exact root property
names (`EmployeeDetails.*`, `Declaration.*`) instead, per this task's explicit instruction and Sentinel's
finding that the generic shape was the actual defect (an absolute `dataRef` inside a fragment does not
inherit the host wrapper's relative context, so a truly generic shape cannot be reconciled with the
wrapper-based reuse pattern used here without a host-side data transform Groundsmith/formwright have not
built). **Confirmed via grep that no other form in this project currently embeds either fragment**
(`employee-training-request` and `employee-training-request-dor` are the only two references to
`employee-identity`/`declaration-consent` in `ui.content/.../content/forms/af`) — so this binds the
fragments to this delivery's schema with zero blast radius today. If either fragment is EVER reused by
a form with a different schema, its field `dataRef`s will need per-consumer overrides or a host-side
canonical-to-schema mapping; flagged for future review, consistent with the fix pass 1 deviation note on
this same fragment.

### TC-019 — SubmissionDate set-value threw a ParserError on every submit

**Root cause:** the submit button's `fd:click` rule (Rule 8, US-09) set
`declarationFragment.submissionDate.$value = new Date().toISOString()` before calling `submitForm()`.
Sentinel reproduced live in-browser: this literal expression throws
`ParserError: Unexpected token type: UnquotedIdentifier, value: Date` on every click, because the Rule
Editor's runtime simple-expression compiler (which parses the plain-text `click`/`script` mirror
attributes, as distinct from the `fd:click` property's own JSON AST) does not support the `new` operator
— it understands field references, literals, and function calls, but not object construction. The
`submitForm()` call in the same click handler is a separate statement and still fired independently,
which is why submit itself was unaffected and why the rule's mere *presence* in
`guideContainer.model.json` was mistakenly read as PASS in the original (pre-retest) verification — the
assignment silently never executed. **Confirmed no interaction with the D3 fix**: the SET_VALUE AST
targets the field by its runtime form-tree path (`$form.declarationFragment.submissionDate`, i.e. the
host wrapper's node name + the fragment field's own node name), which is entirely independent of the
field's `dataRef` binding — correcting D3 did not require or cause any change to this reference.

**Fix:** added a new registered custom function, `getCurrentDateISOString()` (zero-arg, returns
`new Date().toISOString()` as ordinary, unrestricted JavaScript — this file executes directly in the
browser, it is not parsed by the mini-expression grammar), to
`employee-training-request-clientlib/js/functions.js` (documented with the required `@name` JSDoc so it
registers with the Rule Editor's customfunctions endpoint, alongside the already-working
`validateDateAfter`). Replaced the button's SET_VALUE expression with a call to this function — the same
FUNCTION_CALL AST mechanism already proven working for `validateDateAfter($1, $2)` on `endDate` (Rule 4),
just with an empty `params` array (the documented zero-arg-init pattern from `create-form-clientlib`'s
SKILL.md). Both the `fd:click` JSON AST (Rule Editor display) and the `fd:events`/`click` plain-text
mirror (actual runtime execution) were updated to
`declarationFragment.submissionDate.$value = getCurrentDateISOString()`.

**Files touched:**
- `ui.apps/.../apps/clientlibs/employee-training-request-clientlib/js/functions.js` — added
  `getCurrentDateISOString()` with `@name` registration; updated the file-header comment and the
  Rule-Editor usage-note footer to describe both functions now defined here.
- `ui.content/.../content/forms/af/aem-adaptive-form-agents/employee-training-request/.content.xml` —
  replaced the submit button's `fd:click` AST `FUNCTION_CALL` (from the built-in-looking `now`/
  `new Date().toISOString()` literal to `getCurrentDateISOString`/`$0()`) and the `fd:events click`
  mirror attribute; updated the adjacent explanatory comment.
- `ui.content/.../content/forms/af/aem-adaptive-form-agents/employee-identity/.content.xml` — 8
  `dataRef` corrections (D3).
- `ui.content/.../content/forms/af/aem-adaptive-form-agents/declaration-consent/.content.xml` — 2
  `dataRef` corrections (D3).

**Verification performed before handoff:**
- Confirmed via `grep` that `employee-training-request` and `employee-training-request-dor` are the
  ONLY two references to `employee-identity`/`declaration-consent` anywhere under
  `ui.content/.../content/forms/af` — no other form is affected by the binding-path correction.
- Confirmed via `grep` that zero residual `dataRef="$.employee.…"` / `dataRef="$.declaration.…"`
  occurrences remain in either fragment; all 10 now read `$.EmployeeDetails.*` / `$.Declaration.*`.
- All 3 edited `.content.xml` files (`employee-identity`, `declaration-consent`,
  `employee-training-request`) parsed successfully as well-formed XML (.NET `XmlDocument`).
- Extracted the submit button's `fd:click` attribute directly from the edited, well-formed XML (via
  XPath, namespace-aware) and round-tripped it through `ConvertFrom-Json`: parses as a valid 2-element
  array (`ROOT` AST object + metadata object), with `script` reading
  `"declarationFragment.submissionDate.$value = getCurrentDateISOString()"` and
  `eventName:"Click"` — confirming the JSON is well-formed and the intended text change landed exactly
  where expected.
- `functions.js` passed a Node.js syntax check (`node --check`) after the edit — no JS syntax errors.
- Did not have live access to the AEM author instance in this session — `guideContainer.model.json`
  and the actual submitted `data.xml` were NOT re-checked live; this is deferred to Forgemaster's
  redeploy + Sentinel's retest for final live confirmation, exactly as flagged in fix pass 1.
- **Not touched:** Rules #1/#2/#3/#5/#6/#7 (all previously confirmed passing), the theme, the DoR page
  structure (only the two shared fragments' field-level `dataRef`s changed, not the DoR page's own
  layout/panels), the workflow model, the submit action, and Groundsmith's integration artifacts.

**Fix pass 2 run metrics:**
- `time_taken_minutes`: 28.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~7,500
  - `read`: ~34,000 (integration-test-report.md excerpts, formwright.md fix-pass-1 section, both
    fragment `.content.xml` files, host form `.content.xml` excerpts, schema bindings JSON,
    create-form-rules/create-form-clientlib SKILL.md excerpts, functions.js, sibling forms' rule
    patterns for reference)
  - `write`: ~9,500 (10 dataRef edits, 1 AST + 1 mirror-attribute edit, 2 functions.js edits, this
    run-doc update)
  - `other`: ~6,000 (PowerShell JSON/XML validation + XPath extraction calls, grep verification calls,
    `node --check`)
  - `total`: ~57,000

---

## Handoff YAML — Fix pass 2 (post-Sentinel gate-failure #2)

```yaml
agent: formwright
phase: IMPL-build (fix pass 2)
status: FIXES_APPLIED
time_taken_minutes: 28.0
tokens_consumed:
  cli_text: 7500
  read: 34000
  write: 9500
  other: 6000
  total: 57000
defects_fixed:
  - id: D3
    description: "employee-identity and declaration-consent fragment fields serialized to a fragment-internal generic data shape ($.employee.*, $.declaration.*) instead of the schema's real root paths (EmployeeDetails.*, Declaration.*); every real submission's Employee Details / Declaration data landed unschema'd, leaving the DoR blank and applicantEmail/managerId workflow-variable capture empty"
    fix: "Corrected all 10 field-level dataRef attributes (8 in employee-identity, 2 in declaration-consent) to the schema's real absolute paths ($.EmployeeDetails.*, $.Declaration.*), confirmed against employee-training-request.bindings.json. No host-side (form or DoR) wrapper change needed — both hosts' fragmentcontainer dataRef was already $.EmployeeDetails/$.Declaration and now agrees with the fragments."
    files: ["employee-identity/.content.xml", "declaration-consent/.content.xml"]
  - id: TC-019
    description: "SubmissionDate set-value rule (Rule 8) threw a live ParserError on every submit (Unexpected token type: UnquotedIdentifier, value: Date) because the Rule Editor runtime expression compiler does not support the `new` operator; the assignment never executed despite being present in guideContainer.model.json (previous mis-verification)"
    fix: "Added a registered zero-arg custom function getCurrentDateISOString() to employee-training-request-clientlib/js/functions.js (real unrestricted JS, executes in-browser); replaced the submit button's fd:click AST FUNCTION_CALL and the fd:events click mirror attribute to call it instead of the literal new Date().toISOString(), using the same FUNCTION_CALL/zero-params mechanism already proven working for validateDateAfter."
    files: ["employee-training-request-clientlib/js/functions.js", "employee-training-request/.content.xml"]
verification_performed:
  - "grep confirmed employee-training-request + employee-training-request-dor are the ONLY consumers of both fragments — no other form affected by the dataRef correction"
  - "grep confirmed zero residual $.employee.*/$.declaration.* dataRef occurrences remain; all 10 corrected"
  - "All 3 edited .content.xml files parsed as well-formed XML (.NET XmlDocument)"
  - "Submit button's fd:click attribute extracted via namespace-aware XPath from the edited XML and round-tripped through ConvertFrom-Json: valid 2-element array, script text and eventName confirmed correct"
  - "functions.js passed `node --check` syntax validation after the edit"
verification_not_performed:
  - "No live AEM author instance reachable this session — guideContainer.model.json and a real submitted data.xml were NOT re-checked live; deferred to Forgemaster's redeploy + Sentinel's retest for final confirmation"
scope_respected: "Only D3 (both fragments' dataRef) and TC-019 (submit button click rule + its supporting custom function) were touched. Rules #1/#2/#3/#5/#6/#7, the theme, the DoR page structure, the workflow model, and the submit action were left untouched, per task constraint."
deviation_flags:
  - "employee-identity and declaration-consent are no longer schema-agnostic/canonical (critical rule 1b's general preference) — they now bind directly to this delivery's schema root paths (EmployeeDetails.*, Declaration.*), per this task's explicit instruction and because an absolute dataRef inside a fragment does not inherit the host wrapper's relative context, so genuine schema-agnosticism is not reconcilable with this project's wrapper-based fragment reuse pattern without a host-side data transform that does not exist yet. Confirmed zero other current consumers, so zero blast radius today; flagged again for review if either fragment gains a second consumer with a different schema."
confidence: "High for both. D3: root cause directly confirmed against Sentinel's captured data.xml and the schema's own bindings.json; fix is a mechanical, verified path correction with no other consumers affected. TC-019: root cause matches Sentinel's exact reproduced ParserError text; fix reuses an already-proven-working mechanism (validateDateAfter's FUNCTION_CALL pattern) rather than inventing a new one. Both fixes verified well-formed/parseable from the edited source; live guideContainer.model.json / actual submission re-verification deferred to Forgemaster's redeploy + Sentinel's retest, consistent with fix pass 1's own disclosed limitation (no live author access in this session)."
build_summary: ".claude/agents/runs/2026-07-22-employee-training-request/implementation/formwright.md"
gate_result: FIXES_APPLIED_PENDING_RETEST
next: aem-forms-program-agent -> forgemaster (rebuild/redeploy) -> sentinel (retest D3 + TC-019, plus full regression of rules #1-#7, the theme, and the DoR to confirm no regressions from this pass)
```

---

## Fix pass 3 (post-Sentinel gate-failure #3, defect D4)

Sentinel's retest confirmed fix pass 2 fully resolved D3 and the TC-019 ParserError (both gone across
2 full spec runs), but found the SubmissionDate *assignment itself* still never took effect — confirmed
via 3 independent live methods (live DOM value stays empty, the outgoing network request body carries
no `SubmissionDate` key, the persisted JCR payload has no `Declaration.SubmissionDate` value). A control
check (the sibling `readOnly` field `CurrentStatus` DOES appear correctly in the payload) ruled out
"readOnly fields get dropped" as the cause. This is a narrower, NEW defect (D4) — the function call now
runs cleanly with no error, but the assignment has no effect.

### D4 — SubmissionDate set-value assignment silently no-ops (missing `$form.` scope prefix)

**Root cause confirmed:** the submit button's Rule 8 (`fd:click`) SET_VALUE assignment addresses its
target via the plain-text mirror attributes — the `script` array inside the `fd:click` JSON metadata,
and the `fd:events`/`click` attribute — which fix pass 2's own diagnosis already established are what
the Rule Editor runtime actually parses and executes (as distinct from the JSON AST's own structural
node references). Both mirrors read the bare, unscoped `declarationFragment.submissionDate.$value`.
`submitButton` lives inside `actionsPanel`; `declarationFragment` is a SIBLING panel under
`guideContainer` — not a descendant of `actionsPanel` — so from the button's own rule-evaluation scope
that bare reference resolves relative to `actionsPanel` and finds nothing. It does not throw (unlike the
pass-2 `new Date()` ParserError); it just silently fails to write, which is exactly why it survived
fix pass 2's static/JSON verification and only surfaced under Sentinel's live DOM/network/JCR checks.

Confirmed the AFCOMPONENT AST node for this same SET_VALUE statement already carried the CORRECT,
fully-scoped reference — `"id":"$form.declarationFragment.submissionDate"` — proving only the two
plain-text mirrors were wrong, not the AST/target-field wiring itself. Cross-checked against the
WORKING `endDate` rule (Rule 4): its plain-text `script` fully qualifies its cross-field reference as
`$form.trainingRequestDetailsPanel.startDate.$value`, even though `startDate` is a same-panel sibling of
`endDate` — confirming this project's required convention is to ALWAYS `$form.`-prefix any cross-field
reference in the plain-text mirror, never a bare/relative name, regardless of how close the two fields
are in the tree.

**Fix:** prefixed the assignment TARGET (left-hand side only) with `$form.` in both plain-text mirrors:

- `fd:click` JSON metadata `script` array: `declarationFragment.submissionDate.$value = getCurrentDateISOString()` → `$form.declarationFragment.submissionDate.$value = getCurrentDateISOString()`
- `fd:events` `click` attribute: same change, same array position (`submitForm()` as the second
  statement is untouched).

The RHS (`getCurrentDateISOString()`, the pass-2 fix) was not touched — confirmed unchanged in both
mirrors. The AFCOMPONENT AST node's `id` (`$form.declarationFragment.submissionDate`) was already
correct and required no edit.

**Files touched:**
- `ui.content/.../content/forms/af/aem-adaptive-form-agents/employee-training-request/.content.xml` —
  2 edits (the `fd:click` JSON metadata `script` array entry, and the `fd:events` `click` attribute),
  plus an added explanatory comment documenting this fix pass. Nothing else in the file changed.

**Verification performed before handoff:**
- The full file re-parsed as well-formed XML (.NET `XmlDocument`) after both edits.
- The `fd:click` attribute value was extracted via namespace-aware XPath and, after normalizing this
  project's pre-existing DocView comma-escaping convention (`\,` → `,` — the same escaping already
  present throughout this file, including in the known-working `endDate` `fd:validate` blob, confirmed
  by parsing both side-by-side with the identical normalization), parsed successfully as valid JSON.
  The metadata `script` array was extracted from the parsed JSON and confirmed to read exactly
  `$form.declarationFragment.submissionDate.$value = getCurrentDateISOString()` then `submitForm()`.
- Grep-confirmed: the AFCOMPONENT SET_VALUE target `"id":"$form.declarationFragment.submissionDate"`
  is present and unchanged; zero residual bare/unprefixed `click="[declarationFragment.submissionDate`
  mirrors remain; the new `$form.`-prefixed assignment string is present exactly once.
- Grep-confirmed the only remaining occurrence of the literal text `new Date()` anywhere in the file is
  inside the pass-2 explanatory XML comment (historical documentation of the OLD defect) — not in any
  live `fd:click`/`fd:events`/`fd:rules` attribute — so this fix does not reintroduce the pass-2
  ParserError pattern.
- Did not have live access to the AEM author instance in this session — the actual DOM/network
  request/JCR payload were NOT re-checked live; this is deferred to Forgemaster's redeploy +
  Sentinel's re-verification via the same 3 methods it used to confirm the defect (live DOM value,
  outgoing network request body, persisted JCR payload).
- **Not touched:** the `getCurrentDateISOString()` function itself, the D3 fragment `dataRef` fixes,
  Rules #1/#2/#3/#4/#5/#6/#7, the theme, the DoR page, the workflow model, the submit action, and
  Groundsmith's integration artifacts.

**Fix pass 3 run metrics:**
- `time_taken_minutes`: 16.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~3,500
  - `read`: ~13,000 (integration-test-report.md D4 section + summary table, formwright.md fix-pass-2
    section, the form `.content.xml` full read, targeted re-reads of the submitButton/endDate blocks)
  - `write`: ~2,500 (2 targeted attribute edits + 1 explanatory comment + this run-doc update)
  - `other`: ~4,000 (PowerShell XML well-formedness + JSON normalization/round-trip + grep verification
    calls)
  - `total`: ~23,000

---

## Handoff YAML — Fix pass 3 (post-Sentinel gate-failure #3, defect D4)

```yaml
agent: formwright
phase: IMPL-build (fix pass 3)
status: FIXES_APPLIED
time_taken_minutes: 16.0
tokens_consumed:
  cli_text: 3500
  read: 13000
  write: 2500
  other: 4000
  total: 23000
defects_fixed:
  - id: D4
    description: "SubmissionDate set-value assignment (submit button's Rule 8) silently no-op'd: getCurrentDateISOString() ran with no error, but the field never updated in the live DOM, the outgoing network request body, or the persisted JCR payload"
    fix: "The plain-text mirrors (fd:click JSON metadata 'script' array + fd:events 'click' attribute) addressed the assignment target as the bare, unscoped 'declarationFragment.submissionDate.$value'; since submitButton (in actionsPanel) and declarationFragment are sibling panels under guideContainer, the bare reference resolved relative to the button's own scope and found nothing. Prefixed the target with '$form.' in both mirrors, matching the working endDate rule's own convention of always fully-$form-qualifying cross-field references. The AFCOMPONENT AST node already had the correct $form.-scoped id and needed no change; getCurrentDateISOString() (the RHS) was not touched."
    files: ["employee-training-request/.content.xml"]
verification_performed:
  - "Full file re-parsed as well-formed XML (.NET XmlDocument) after both edits"
  - "fd:click JSON blob extracted via namespace-aware XPath, normalized for this file's pre-existing DocView comma-escaping (\\, -> ,), and parsed as valid JSON -- script array confirmed to read the new $form.-prefixed assignment then submitForm()"
  - "Cross-checked the same normalization against the known-working endDate fd:validate blob -- parses identically, confirming the JSON itself (not just the normalization trick) is sound"
  - "Grep-confirmed the AFCOMPONENT SET_VALUE id ($form.declarationFragment.submissionDate) is unchanged; zero residual bare/unprefixed click mirrors remain; the new prefixed assignment string is present exactly once"
  - "Grep-confirmed the only remaining 'new Date()' text in the file is inside the pass-2 historical explanatory comment, not in any live rule attribute -- pass-2's ParserError pattern is not reintroduced"
verification_not_performed:
  - "No live AEM author instance reachable this session -- the live DOM value, outgoing network request body, and persisted JCR payload were NOT re-checked; deferred to Forgemaster's redeploy + Sentinel's re-verification via the same 3 methods used to confirm D4"
scope_respected: "Only the submit button's SET_VALUE assignment target (2 plain-text mirror edits, 1 explanatory comment) was touched. getCurrentDateISOString() itself, the D3 dataRef fixes, Rules #1-#7, the theme, the DoR page, the workflow model, and the submit action were left untouched, per task constraint."
confidence: "High. Root cause directly matches Sentinel's own stated hypothesis and evidence (3 independent live confirmation methods all agreeing the value never lands, with a control check ruling out the readOnly-field theory); the fix mirrors a proven, already-working in-repo pattern (endDate's fully-$form.-qualified cross-field reference) rather than inventing a new mechanism; JSON/XML verified well-formed and parseable pre- and post-edit; explicitly re-confirmed the pass-2 fix (getCurrentDateISOString(), no 'new Date()' literal) is untouched and not regressed. Residual uncertainty is solely the lack of a live author instance to re-run Sentinel's 3 confirmation methods in this session."
build_summary: ".claude/agents/runs/2026-07-22-employee-training-request/implementation/formwright.md"
gate_result: FIXES_APPLIED_PENDING_RETEST
next: aem-forms-program-agent -> forgemaster (rebuild/redeploy) -> sentinel (retest D4 via all 3 confirmation methods -- live DOM value, outgoing network request body, persisted JCR payload -- plus regression of D3/TC-019 and rules #1-#7)
```

---

## Fix pass 4 — user-reported remediation (2 direct user findings, post-handoff)

The delivery had reached HANDOFF after 3 Sentinel-driven fix cycles. The user then reviewed the
deployed migration directly and found 2 further defects. Before starting, `migrate-form` was
re-invoked to pick up its updated §B5 (OOTB-first workflow-step guidance, Groundsmith's concern)
and Path-B checklist — the checklist's fragment bullet and the B4 script-inventory bullet directly
scoped the two issues below. Both issues were investigated against a **reachable local AEM author
instance** (`http://localhost:4502`, confirmed `HTTP 200`), so live evidence (not just source-file
reasoning) was available for both — a first for this delivery's fix passes.

### Issue 1 — Fragments had no schema binding of their own (AF-editor "no form object" report)

**Root cause.** Both fragments' `fragmentcontainer` root declared `schemaType="none"` (set at
original build time, and left untouched through fix passes 1–3). Fix pass 2 (defect D3) had since
rebound every FIELD inside both fragments to ABSOLUTE `dataRef`s anchored to the host form's real
schema (`$.EmployeeDetails.*`, `$.Declaration.*`, confirmed against
`employee-training-request.schema.json`) — but nobody went back and told the fragment's own
`fragmentcontainer` about that schema. The result: each fragment's fields carry dataRefs that
presuppose a schema the fragment itself never declares. This is exactly the failure mode the
freshly re-read `migrate-form` Path-B checklist names explicitly: *"a `cq:Page` whose root is a
`fragmentcontainer` bound to the schema... A fragment authored without the fragmentcontainer/model
binding opens with no form object and cannot be edited."*

**Live evidence gathered before concluding.** I fetched the CURRENTLY DEPLOYED (pre-fix)
`guideContainer.model.json` for both fragments directly from the running instance:
- Both showed `"schemaType":"none"` in `properties`, exactly as the source `.content.xml` had it.
- Surprisingly, the RUNTIME rendering model (`:items`) was fully populated for both fragments —
  every field appeared with a real `id`/`name`/`dataRef`, and `employee-identity`'s `managerId`/
  `managerName` even showed their live `"rules":{"enabled":...}` bindings (Rule #1, fix pass 1).
  `customfunctions` for `employee-identity` returned `{"customFunction":[{"id":"Number"}]}`
  (non-empty — model import succeeded at the servlet level too).
- This means the **runtime rendering path** (what Sentinel's functional tests exercise) was not
  broken — it walks the JCR tree structurally and does not require `schemaType`/`schemaRef` to
  resolve fields. The **AF-editor's own canvas/Form-Object/data-binding surface** is a different
  code path (gated on the DAM asset's `formmodel` / the fragmentcontainer's `schemaType`) that I
  could not directly probe with `curl` (no browser-automation tool was invoked this pass — I did
  not fetch `editor.html`'s client-rendered DOM). I disclose this rather than overstate the
  verification: the fix is applied on the explicit, authoritative textual guidance from the
  freshly-reread skill (which describes this exact symptom and this exact remedy) plus the
  structural dataRef/schema mismatch I confirmed directly in the source — but I could not visually
  reconfirm the editor canvas populating post-fix in this pass (no redeploy has happened yet either
  — author-only, per my mandate).

**Fix applied (both fragments' Artifact 1 + Artifact 2):**

| File | Change |
|---|---|
| `employee-identity/.content.xml` (`guideContainer`/`fragmentcontainer`) | `schemaType="none"` → `schemaType="jsonschema"` + added `schemaRef="/content/dam/formsanddocuments/schema/employee-training-request.schema.json"` (same casing/value as the host form's own `guideContainer`, and the project's dominant convention — 11 of 15 non-FDM forms use lowercase `jsonschema`) |
| `declaration-consent/.content.xml` (`guideContainer`/`fragmentcontainer`) | same change, same schema |
| `content/dam/.../employee-identity/.content.xml` (DAM fragment asset, Artifact 2 `metadata`) | `formmodel="none"` → `formmodel="jsonschema"` (Artifact 2's own mandatory rule: "Set `formmodel` to match the chosen model") |
| `content/dam/.../declaration-consent/.content.xml` (DAM fragment asset) | same change |

This does **not** change the embedded/host-rendered behaviour (the host form's own schema already
satisfied every field's dataRef there) — it only gives the fragment a model to resolve against when
opened **standalone**, which is what the AF editor and the Rule Editor's field-picker need.

**Template choice re-verified, NOT changed.** The `create-AdaptiveFormFragment` SKILL.md contains an
internal contradiction (one "Mandatory rules" bullet says use `blank-af-v2`/never
`blank-af-v2-fragment`, while its own Prerequisite box, Form-model table, Failure-symptoms table, AND
Pre-flight checklist all say the opposite — use a genuine FRAGMENT template, never the FORM
template). Weight of evidence (3 sections vs. 1) says the fragment template is correct; both fragments
already use `blank-af-v2-fragment` (`afv2-fragment-page` type, `fragmentcontainer` root,
`fd:type="fragment"`), built once in the original pass and confirmed unchanged here. **Not the cause
of Issue 1** — left as-is (touching it would also have risked breaking the OTHER fragment, since both
share the one template, per this task's own caution).

**Verification performed:**
- Both fragments' `.content.xml` and both DAM assets' `.content.xml` re-parsed as well-formed XML
  (PowerShell `[xml]` cast) after every edit.
- Confirmed via the schema's own rendition JSON (`employee-training-request.schema.json`) that
  `EmployeeDetails` and `Declaration` are real root properties, matching every dataRef the fragments'
  fields already use (fix pass 2) byte-for-byte.
- Confirmed via `querybuilder.json` that these are the ONLY 2 Adaptive Form Fragments on the instance
  (no other precedent fragment exists in this project to diff against — both repo-wide and on the
  live instance).
- Live pre-fix `guideContainer.model.json` fetched and inspected for both fragments (see "Live
  evidence" above) to ground the diagnosis in the actually-deployed state, not just source reasoning.
- **Not verified:** the post-fix AF-editor canvas (no redeploy has run yet — author-only; no browser
  automation invoked this pass). Flagged for Forgemaster's redeploy + Sentinel's re-check: open both
  fragments in the AF editor and confirm the canvas/Form-Object tree populates, and that the HOST
  form's Rule Editor field-picker can resolve `employeeDetailsFragment.employeeId` /
  `declarationFragment.submissionDate` (the two cross-fragment references Rules #1/#5/#8 already use).

### Issue 2 — Exhaustive XFA script inventory across all 3 XDPs

Extracted **every** `<script>` element (regardless of parent — `<event>`, `<validate>`, or any
other) from all three XDPs via an XPath-based parse (`//t:script` against the
`xfa-template/3.6` namespace), confirmed the count matched a raw `grep -c "<script"` on each file
(no XPath under-match), and traced each script's owning field + event activity.

**Inventory: 14 `<script>` elements found across 3 XDPs.**

| # | XDP | Owner | Event | Purpose | Status in source | Disposition |
|---|---|---|---|---|---|---|
| 1 | DocRoute | `EmployeeId` | exit | Manager auto-assign + lock when EmployeeId≤10 | ACTIVE | ✅ Already covered — Rule #1 (fix pass 1: `fd:enabled` on `managerId`/`managerName` in `employee-identity`) |
| 2 | DocRoute | `EmployeeId` | docReady | Show/hide ApprovalInfo (EmployeeId non-empty AND CurrentStatus≠FinanceApproved) | ACTIVE | ✅ Already covered — Rule #5 (`fd:visible` on `approvalInformationPanel`) |
| 3 | DocRoute | `Email` | exit | Regex validate + `messageBox` alert + `setFocus` | ACTIVE | ✅ Already covered — Rule #2 (built-in `validatePictureClause`/Message on the fragment's `email` field); the modal `messageBox`+`setFocus` UI chrome has no web equivalent and is superseded by native inline field-error display (the correct web-native replacement, not a drop) |
| 4 | DocRoute | `RequestId` | exit | Lock/unlock RequestType until RequestId non-empty | ACTIVE | ✅ Already covered — Rule #3 (`fd:enabled` on `requestType`) |
| 5 | DocRoute | `EndDate` | validate | EndDate must be after StartDate (FormCalc-adjacent `Date2Num` + `xfa.event.valid=false`) | ACTIVE | ✅ Already covered — Rule #4 (`validationExpression` calling `validateDateAfter()`) |
| 6 | DocRoute | `CurrentStatus` | docReady | Lock/unlock FinanceComments unless CurrentStatus==ManagerApproved | ACTIVE | ✅ Already covered — Rule #6 (fix pass 1: `fd:enabled` on `financeComments`) |
| 7 | DocRoute | root subform | (none) | Adobe Designer's standard `ContainerFoundation_JS` FormBridge library (PDF/Reader↔host messaging: submit/print/save/getData/setData/etc.) | ACTIVE (boilerplate) | ⬜ N/A — PDF/Acrobat-Reader embedded-messaging plumbing; no meaning in a server-rendered Core Components web form; no migration target exists or is needed |
| 8 | EmployeeTrainingRequest | `Page1` subform | ready | Calls `ContainerFoundation_JS.RegisterMessageHandler()`/`notifyInitialized()` | ACTIVE (boilerplate) | ⬜ N/A — same reason as #7 |
| 9 | EmployeeTrainingRequest | `Page1` subform | (none) | Second copy of the `ContainerFoundation_JS` library | ACTIVE (boilerplate) | ⬜ N/A — same reason as #7 |
| 10 | EmployeeTrainingRequest | `EmployeeId` | exit | Same manager-lock script as #1 | **COMMENTED OUT** in source | ⬜ N/A — disabled by the original XDP author; also moot in a display-only DoR (no lock semantics apply to a static record) |
| 11 | EmployeeTrainingRequest | `Email` | exit | Same email-validate script as #3 | **COMMENTED OUT** in source | ⬜ N/A — disabled by the original XDP author; DoR has no editable email field to validate |
| 12 | EmployeeTrainingRequest | `RequestId` | exit | Same RequestType lock as #4 | ACTIVE in source | ⬜ N/A — live-verified: the DoR's `requestType` is unconditionally `readOnly="{Boolean}true"` (confirmed both in source `.content.xml` and the LIVE deployed `guideContainer.model.json`), so the access-toggle has zero observable effect there |
| 13 | EmployeeTrainingRequest | `EndDate` | validate | Same EndDate>StartDate check as #5 | **COMMENTED OUT** in source | ⬜ N/A — disabled by the original XDP author; no validation applies to a static display-only record |
| 14 | EmployeeTrainingRequest | `ApprovalInfo` subform | docReady | Show/hide ApprovalInfo (EmployeeId non-empty only — simpler than DocRoute's #2, no CurrentStatus check) | ACTIVE in source | ⬜ N/A — DESI's original design (component-design-spec.yaml) explicitly decided this panel is ALWAYS visible in the DoR (generated only post-final-approval, so the guard can never trigger hidden); live-verified: the DoR's `approvalInformationPanel` is unconditionally `"visible":true` with NO rules object in the deployed model.json |

`ApprovalInfo.xdp`: **0** `<script>` elements (confirmed via both XPath and raw grep) — nothing to
migrate from this file.

**Summary: 14 scripts found across 3 XDPs → 0 newly migrated (none were needed) → 6 distinct
business-logic scripts (11 of the 14 occurrences: #1–6 in DocRoute, #10–14's business-purpose
duplicates in EmployeeTrainingRequest) already fully covered by the original 8 rules → 3 determined
not applicable (Adobe FormBridge/`ContainerFoundation_JS` platform boilerplate, #7–9, no
AEMaaCS/Core-Components equivalent needed) → 5 of EmployeeTrainingRequest's occurrences determined
not applicable to the DoR context specifically (#10, #11, #13 already disabled at the XFA-source
level; #12 and #14 technically active in source but rendered moot by the DoR's own unconditional
`readOnly`/`visible` design, independently re-verified against the LIVE deployed DoR model.json,
not just claimed) → **0 silently dropped.****

**Live rule/mirror-depth verification (matching Sentinel's own methodology — not just "a node
exists"):** fetched the currently-deployed host form's `guideContainer.model.json` and grepped every
`"rules":{...}` object present:
```
"rules":{"enabled":"Number(employeeId.$value) <= 0 || Number(employeeId.$value) > 10"}   (managerId)
"rules":{"enabled":"Number(employeeId.$value) <= 0 || Number(employeeId.$value) > 10"}   (managerName)
"rules":{"enabled":"requestId.$value != \"\""}                                            (requestType)
"rules":{"visible":"employeeDetailsFragment.employeeId.$value != \"\" && currentStatus.$value != \"FinanceApproved\""}   (approvalInformationPanel)
"rules":{"enabled":"currentStatus.$value == \"ManagerApproved\""}                         (financeComments)
```
Plus `endDate`'s `validationExpression` (Rule #4) and `email`'s `constraintMessages.pattern` (Rule
#2, confirmed on the fragment's own model.json) — all 6 business rules are live on the currently
deployed system, each with its actual reactive binding present (mirror-depth), not merely a node.
Also live-confirmed on the deployed DoR page: `requestType.readOnly:true` and
`approvalInformationPanel.visible:true` with no attached rules object — both N/A determinations hold
on the running instance, not only in source.

**Traceability note (audit finding, not a defect — no action taken, outside this task's 2 issues).**
Two of the original 8 rules do not trace back to any `<script>` at all: Rule #7 (AcceptedTerms
required) has no `<validate nullTest>` on the source `AcceptedTerms` field either — it is a genuine
XFA field with neither a required-validation nor a script; and Rule #8 (SubmissionDate system-set)
has NO source script anywhere across all 3 XDPs — the source `SubmissionDate` field instead carries
`<validate nullTest="error"/>` (i.e. the source treats it as a REQUIRED user-entered field, not a
readOnly/system-set one). Both rules appear to derive from `user-stories.yaml`/DESI's design intent
(US-09) rather than a literal reverse-engineered XFA script. This is a traceability observation for
the record, not a script-migration gap — there is no dropped script here, only a documented business
decision layered on top of the source. Per this task's explicit scope ("do not touch D4… not yours to
chase again"), Rule #8's mechanics are NOT touched in this pass.

**Files touched (Issue 2): none.** The exhaustive audit found no gap to fix — every business script
was already covered by the original 8 rules (verified live), and every DoR-context script omission is
independently justified and live-verified. No source file was edited for Issue 2.

### Fix pass 4 files touched (Issue 1 only)

- `ui.content/.../content/forms/af/aem-adaptive-form-agents/employee-identity/.content.xml` —
  `schemaType`/`schemaRef` added to the `fragmentcontainer`, plus an explanatory comment.
- `ui.content/.../content/forms/af/aem-adaptive-form-agents/declaration-consent/.content.xml` — same.
- `ui.content/.../content/dam/formsanddocuments/aem-adaptive-form-agents/employee-identity/.content.xml` —
  `formmodel` metadata updated to match.
- `ui.content/.../content/dam/formsanddocuments/aem-adaptive-form-agents/declaration-consent/.content.xml` — same.

### Not touched (per task scope)

D4 (SubmissionDate assignment mechanics), D2 (OR-split workflow — Groundsmith's), the theme, the DoR
page's own layout, the workflow model, the submit action, Rules #1–#8's own logic (only their
LIVE-VERIFIED presence was checked, nothing was edited), and the fragment template
(`blank-af-v2-fragment`, re-verified correct, left untouched to avoid risking the shared template's
other consumer).

**Fix pass 4 run metrics:**
- `time_taken_minutes`: 52.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~11,000
  - `read`: ~78,000 (formwright.md all 3 prior passes, component-design-spec.yaml, planwright.md,
    migrate-form SKILL.md re-read in full, create-AdaptiveFormFragment SKILL.md in full, both
    fragment + both DAM asset `.content.xml` files, host form `.content.xml` in full, DoR page
    `.content.xml` in full, schema.json rendition, 3 XDPs' extracted script bodies, live
    model.json/customfunctions responses)
  - `write`: ~10,500 (4 source-file edits + this run-doc update)
  - `other`: ~9,500 (PowerShell XML parsing/validation, XPath script extraction across 3 XDPs, live
    `curl`/AEM-reachability probes, `querybuilder.json` fragment enumeration, grep verification calls)
  - `total`: ~109,000

---

## Handoff YAML — Fix pass 4 (2 user-reported defects, post-HANDOFF)

```yaml
agent: formwright
phase: IMPL-build (fix pass 4)
status: FIXES_APPLIED
time_taken_minutes: 52.0
tokens_consumed:
  cli_text: 11000
  read: 78000
  write: 10500
  other: 9500
  total: 109000
defects_fixed:
  - id: "Issue 1 — fragments have no form object / field model in the AF editor"
    description: "employee-identity and declaration-consent fragmentcontainers declared schemaType=\"none\" while every field inside carried an ABSOLUTE dataRef anchored to the host form's real schema (set in fix pass 2/D3) — a fragment with no schema of its own cannot resolve those dataRefs when opened STANDALONE in the AF editor, per migrate-form's freshly re-read Path-B checklist (\"a fragment authored without the fragmentcontainer/model binding opens with no form object\")"
    fix: "Set schemaType=\"jsonschema\" + schemaRef=\".../employee-training-request.schema.json\" on both fragments' fragmentcontainer nodes (Artifact 1), matching the host form's own schema declaration; updated both DAM fragment assets' formmodel metadata (Artifact 2) from \"none\" to \"jsonschema\" to match, per Artifact 2's own mandatory rule"
    files: ["employee-identity/.content.xml", "declaration-consent/.content.xml", "dam/.../employee-identity/.content.xml", "dam/.../declaration-consent/.content.xml"]
    caveat: "Live pre-fix model.json showed the RUNTIME rendering path already resolving fields fine (structural JCR walk, schema-independent) — the reported break is specifically in the AF-editor's own canvas/Form-Object surface, which I could not visually re-verify post-fix (no redeploy yet, no browser automation invoked this pass). Fix is grounded in the skill's explicit, authoritative guidance plus a real, confirmed dataRef/schema mismatch — not merely a guess."
  - id: "Issue 2 — exhaustive XFA script inventory (B4 gap re-audit)"
    description: "Re-audited ALL 14 <script> elements across all 3 XDPs (not just DocRoute) against the updated migrate-form B4 guidance"
    fix: "NONE NEEDED — exhaustive, live-verified audit found 0 scripts silently dropped: 6 distinct business scripts (11 occurrences) already covered by the original 8 rules (confirmed live at rule/mirror depth), 3 occurrences are Adobe FormBridge/ContainerFoundation_JS platform boilerplate (no AEMaaCS equivalent needed), 5 EmployeeTrainingRequest occurrences are genuinely not applicable to the DoR context (3 disabled at source, 2 rendered moot by the DoR's own unconditional readOnly/visible design, independently live-verified against the deployed DoR model.json)"
    files: []
script_inventory:
  total_scripts_found: 14
  by_xdp: { DocRoute: 7, EmployeeTrainingRequest: 7, ApprovalInfo: 0 }
  newly_migrated: 0
  already_covered_by_original_8_rules: 6   # distinct business scripts, 11 of 14 occurrences
  not_applicable_boilerplate: 3            # ContainerFoundation_JS FormBridge library + its ready-event wrapper
  not_applicable_to_dor_context: 5         # EmployeeTrainingRequest occurrences (3 source-disabled + 2 moot-by-design, all live-verified)
  silently_dropped: 0
verification_performed:
  - "Live AEM author instance reachable this session (HTTP 200) — first fix pass with live access"
  - "Fetched pre-fix guideContainer.model.json for both fragments — fields/rules fully resolved at runtime despite schemaType=none, confirming the break is editor-surface-specific, not runtime-rendering"
  - "Fetched customfunctions endpoint for employee-identity — non-empty, model import succeeds at servlet level"
  - "querybuilder.json confirmed these are the ONLY 2 Adaptive Form Fragments on the instance (no precedent to diff against)"
  - "XPath-extracted every <script> element across all 3 XDPs (14 total), cross-checked count against raw grep -c (exact match, no under-extraction)"
  - "Fetched live host form model.json and grepped every \"rules\":{...} object — confirmed all 5 non-validate business rules present at mirror depth (managerId, managerName, requestType, approvalInformationPanel, financeComments), plus endDate's validationExpression and email's constraintMessages.pattern"
  - "Fetched live DoR page model.json — confirmed requestType.readOnly:true and approvalInformationPanel.visible:true with no rules object, verifying both N/A determinations hold on the deployed system"
  - "All 4 edited .content.xml files re-parsed as well-formed XML after editing"
verification_not_performed:
  - "Did not redeploy (author-only) — the schemaType/schemaRef fix has not yet been reflected in a fresh live model.json; Forgemaster's redeploy is required before this can be re-checked live"
  - "Did not browser-drive editor.html — the AF-editor's own canvas/Form-Object panel was not visually confirmed populating, pre- or post-fix, in this session"
scope_respected: "Only Issue 1 (schemaType/schemaRef on both fragments + their DAM assets) was code-changed. Issue 2 required no code change (audit-only, evidence-backed). D4, D2, the theme, the DoR page layout, the workflow model, the submit action, Rules #1-#8's own logic, and the fragment template were all left untouched, per task constraint."
confidence: "Issue 1: Medium-high — the fix is squarely grounded in the freshly re-read skill's explicit, named failure mode and a real, confirmed structural mismatch (schema-less fragment, schema-bound field dataRefs); residual uncertainty is the AF-editor's own canvas, which needs a redeploy + a visual/browser check to fully close out. Issue 2: High — the audit is exhaustive (XPath-verified complete script extraction, cross-checked against raw grep), and every disposition (covered / boilerplate / DoR-N/A) is backed by LIVE evidence from the actually-deployed system, not source-only reasoning."
build_summary: ".claude/agents/runs/2026-07-22-employee-training-request/implementation/formwright.md"
gate_result: FIXES_APPLIED_PENDING_RETEST
next: aem-forms-program-agent -> forgemaster (rebuild/redeploy) -> sentinel (open both fragments in the AF editor post-redeploy and confirm the canvas/Form-Object tree populates; re-run the live rule/mirror checks above to confirm they survive the redeploy; no re-test needed for Issue 2 since no code changed, but Sentinel may spot-check the same 6 rules if desired)
```

---

## Fix pass 10 — D13 (form's own DoR template wiring / AFtoDORStep validity gate)

D13 (Sentinel fix-pass-9): the OOTB Generate-DoR step
(`com.adobe.fd.workflow.dorGeneration.AFtoDORStep`) hard-fails with
`java.lang.Exception: Not a valid Adaptive Form` at `AFtoDORStep.java:125` when pointed at the CC
interactive form via `AF_PATH` (FORM_RESOLUTION=PATH), and Sentinel's `dorType none->generate`
tweak had ZERO effect. The user approved the CC-idiomatic fix. **I decompiled the actual bundle to
find the real gate rather than guess at `dorType` values** — and the gate turned out NOT to be a
`dorType`/`dorTemplateRef` thing at all.

### Root cause — decompiled, definitive (not inferred)

Extracted `AFtoDORStep.class` from the live bundle
`com.adobe.aemfd.adobe-aemfd-workflow-process-common:6.0.260` (OSGi id 651,
`crx-quickstart/launchpad/felix/bundle651/version0.0/bundle.jar`) and ran `javap -p -c -l`.
`internal_execute` (JDK 11 javap) does this, in order:

1. `String formPath = formResolverHelper.getFormPath(workItem);`  (register 6 — the resolved `AF_PATH`)
2. builds `String node = formPath + "/" + "jcr:content";`
3. `if (session.nodeExists(node)) { Node n = session.getNode(node);`
4. **`if (!(n.hasProperty("guide") && n.getProperty("guide").getString().equals("1"))) {`**
   **`  log.info("Adaptive form at Node : {} is not valid", n.getPath());`**
   **`  throw new Exception("Not a valid Adaptive Form"); }`**  ← **this is line 125**
5. only AFTER passing does it build `DoROptions`, `setFormResource(<AF_PATH resource>)`, `setData(...)`,
   `setLocale(...)`, and call **`com.adobe.aemds.guide.addon.dor.DoRService.render(dorOptions)`** — the
   **Core Components** DoR service — then writes the bytes to `DOR_PATH`.

**So the line-125 validity gate is EXACTLY: `<AF_PATH>/jcr:content` must carry a String property named
`guide` whose value is `"1"`. It is completely INDEPENDENT of `dorType` and `dorTemplateRef`** — which
is precisely why Sentinel's `dorType none->generate` had zero effect, and why it would have had zero
effect no matter what `dorType`/`dorTemplateRef` value was set. `guide="1"` is the LEGACY (Foundation)
Adaptive-Form marker; a Core Components AF page does not carry it by default — confirmed live: neither
form's `jcr:content` had a `guide` property.

Cross-corroborated with the editor's own DoR advisory servlet in the SAME bundle,
`FormSelectServlet` (decompiled): its `isAdaptiveForm(session, formPath)` runs the identical
`jcr:content/@guide == "1"` check — i.e. the runtime step and the editor advisory reject the CC form
for the one same reason.

### What I set (source + live, readback-confirmed)

| Node | Property | Value | Why |
|---|---|---|---|
| `employee-training-request/jcr:content` | `guide` | `"1"` | clears AFtoDORStep line 125 (this is the current `AF_PATH` per fix-pass-9) |
| `employee-training-request/jcr:content/guideContainer` | `dorType` | `generate` (KEPT — not changed) | the form is its own auto-generated DoR (CC-idiomatic "form's own DoR template"); `DoRService.render` auto-generates from the form layout in this mode |
| `employee-training-request-dor/jcr:content` | `guide` | `"1"` | Groundsmith's passes have alternated `AF_PATH` between the interactive form and this dedicated DoR page; marking BOTH means the gate passes whichever one `AF_PATH` resolves to |
| `employee-training-request-dor/jcr:content/guideContainer` | `dorType` | `none` -> `generate` | valid stage-2 render config if this page is the `AF_PATH` target |

Live-pushed via Sling POST (CSRF token + `Referer`), all HTTP 200, all readback-confirmed:
`employee-training-request/jcr:content @guide="1"`, `guideContainer dorType="generate"`;
`employee-training-request-dor/jcr:content @guide="1"`, `guideContainer dorType="generate"`.
Source `.content.xml` for both pages updated to match (with full inline decompile evidence in the
interactive form's `jcr:content` comment).

### Which DoR-template target is valid, and why I did NOT set `dorType="select"`

The task offered three `dorTemplateRef` candidates; my investigation determined **none is a valid
`dorTemplateRef`, and `select` is the wrong mode here regardless**:
- **(a) `employee-training-request-dor` page** — NOT valid. A `dorTemplateRef` for `dorType="select"`
  must be an XDP/print DoR template asset; the `-dor` page is another `cq:Page` Adaptive Form, not an
  XDP. Also, `AFtoDORStep` never reads `dorTemplateRef` at all.
- **(b) legacy `EmployeeTrainingRequest.xdp` / `ApprovalInfo.xdp`** — do NOT exist as assets. Verified:
  zero `*.xdp` files in the repo, and a live query returned **0** `*.xdp` assets under
  `/content/dam`. The Path-B migration converted them to AF pages/fragments; it did not retain the XDPs
  as print templates.
- **(c) create/register a proper XDP DoR template** — unnecessary and contrary to the approved
  CC-idiomatic direction. In Core Components, "the form's own Document of Record template" IS
  `dorType="generate"` (automatic DoR from the form's own layout, rendered by `DoRService`); `select`
  is only for an external XDP. There is no XDP to point at, and switching to `select` with an invalid
  template would BREAK the stage-2 `DoRService.render` (regressing from the working `generate` mode).
  Per the task's explicit "do NOT thrash trying random configs" constraint, I kept `generate`.

Note on the editor advisory: I decompiled `AdaptiveFormUtil.isDORConfigured` (bundle 672,
`com.adobe.aemds.guide.aemds-guide-core`) — it is
`StringUtils.isNotEmpty(GuideUtils.getDoRTemplateRef(...))`, i.e. it returns `true` ONLY when a
SELECTED XDP template ref exists, so it is (correctly) `false` for `generate` mode. **`AFtoDORStep`
does NOT call `isDORConfigured`** — so that advisory being `false` is irrelevant to whether the
workflow step succeeds; it only affects the editor's "DoR configured?" badge.

### AFtoDORStep-compatibility finding (the core question)

**AFtoDORStep CAN render a Core Components form's DoR on this SDK** — it delegates to the CC
`com.adobe.aemds.guide.addon.dor.DoRService.render(DoROptions)`. This is NOT a case of "AFtoDORStep
genuinely cannot do CC DoR." Its ONLY blocker was the legacy `guide=="1"` marker gate at line 125,
which a CC page lacks by default. With `guide="1"` now present:
- **Line 125 will pass.** Proven LIVE, not just by decompile: the DoR advisory servlet's
  `isAdaptiveForm` operation — which runs the byte-for-byte identical `jcr:content/@guide == "1"`
  check — now returns `{"isAdaptiveForm":true}` for BOTH pages (it returned false before the marker).
  This is the strongest live proof obtainable without completing a workflow work item.
- **Stage-2 render** then proceeds through `DoRService.render` with `dorType="generate"` (auto-DoR
  from the form layout — a standard, supported CC capability; no template ref required in this mode).

### Verification performed
- `javap` decompile of `AFtoDORStep.internal_execute` (line 125 gate) and `FormSelectServlet`
  (`isAdaptiveForm`) — both gate solely on `jcr:content/@guide == "1"`.
- `javap` decompile of `AdaptiveFormUtil.isDORConfigured` — requires a selected XDP templateRef;
  confirms why `isDorConfigured` is false for `generate` mode and that AFtoDORStep does not use it.
- Live Sling POST + readback of all 4 property writes (HTTP 200, values confirmed).
- Live `FormSelectServlet` `isAdaptiveForm` — flipped from (implicitly) false to **`true`** for both
  pages after the marker was set (the decisive live confirmation the line-125 gate is cleared).
- Live query: 0 `*.xdp` assets under `/content/dam` (no valid `dorTemplateRef` target exists).

### Verification NOT performed (honest limitation)
- Did not render an actual DoR PDF end-to-end: that requires completing both Assign Task work items
  in the Workspace UI to reach `node6`, and there is no headless "complete work item" path on this SDK
  (same tooling gap Sentinel/Groundsmith recorded for D2). So stage-2 (`DoRService.render` actually
  emitting PDF bytes for `generate` mode) is proven by decompile + the fact that it IS the standard CC
  auto-DoR service, not by a captured PDF this session. Sentinel should complete an Approve->Approve
  run post-redeploy and confirm `GET <payload>/DocumentofRecord/DoR.pdf` returns a real PDF (was 404).
- Also flagged: `guide="1"` on a CC page's `jcr:content` is a legacy marker; it is a benign extra
  String property (CC runtime keys off `sling:resourceType`/`guideContainer`, not this), but Sentinel
  should confirm the form still renders/edits/submits normally post-redeploy.

### Files touched
- `ui.content/.../content/forms/af/aem-adaptive-form-agents/employee-training-request/.content.xml`
  — added `guide="1"` to `jcr:content` (+ decompile-evidence comment); `dorType="generate"` unchanged.
- `ui.content/.../content/forms/af/aem-adaptive-form-agents/employee-training-request-dor/.content.xml`
  — added `guide="1"` to `jcr:content`; `dorType` `none`->`generate` on the guideContainer.

Not touched: the workflow model (AF_PATH / DOR_PATH / Assign Task steps / D2 OR-split / D4 / the
SubmissionDate rule), fragments, theme, clientlib, rules, submit action.

### Fix pass 10 — run metrics
- `time_taken_minutes`: ~46.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~9,000
  - `read`: ~118,000 (integration-test-report D13 + prior retests, groundsmith fix-pass-4/5/9 DoR
    sections, both form `.content.xml` full reads, decompiled `javap` output for 3 classes)
  - `write`: ~7,000 (2 `.content.xml` edits with evidence comments, this run-doc section)
  - `other`: ~17,000 (bundle-jar extraction + `javap` runs, ~10 live curl HTTP calls: CSRF, 3 POSTs,
    readbacks, advisory-servlet checks, XDP query)
  - `total`: ~151,000

---

## Handoff YAML — Fix pass 10 (D13 — AFtoDORStep validity gate)

```yaml
agent: formwright
phase: IMPL-build (fix pass 10)
status: FIXES_APPLIED
time_taken_minutes: 46.0
tokens_consumed:
  cli_text: 9000
  read: 118000
  write: 7000
  other: 17000
  total: 151000
defects_fixed:
  - id: D13
    description: "OOTB AFtoDORStep threw 'Not a valid Adaptive Form' at AFtoDORStep.java:125 for the CC form; dorType none->generate had zero effect"
    root_cause: "Decompiled AFtoDORStep.internal_execute (bundle com.adobe.aemfd.adobe-aemfd-workflow-process-common:6.0.260): line 125 gate is `<AF_PATH>/jcr:content` must have String property guide == '1' (legacy Foundation AF marker) — INDEPENDENT of dorType/dorTemplateRef. A CC AF page lacks this marker. Corroborated by FormSelectServlet.isAdaptiveForm using the identical check."
    fix: "Added guide='1' to jcr:content of BOTH the interactive form (current AF_PATH) and the -dor page; kept dorType='generate' on the interactive form and changed the -dor page's dorType none->generate. Did NOT set dorType='select' — no valid XDP dorTemplateRef exists (0 *.xdp assets; the -dor page is an AF cq:Page not an XDP; AFtoDORStep does not even read dorTemplateRef) and select would break the working DoRService.render generate path."
    files:
      - "ui.content/.../content/forms/af/aem-adaptive-form-agents/employee-training-request/.content.xml"
      - "ui.content/.../content/forms/af/aem-adaptive-form-agents/employee-training-request-dor/.content.xml"
dor_config_applied:
  interactive_form: { "jcr:content@guide": "1", "guideContainer@dorType": "generate (unchanged)" }
  dor_page:        { "jcr:content@guide": "1", "guideContainer@dorType": "none->generate" }
  dorTemplateRef:  "NOT set — no valid XDP DoR-template target exists; generate mode needs none"
aftodorstep_compatibility: "CAN do CC DoR — delegates to the CC com.adobe.aemds.guide.addon.dor.DoRService.render; only the legacy guide=='1' line-125 gate blocked it. Gate now cleared."
live_verification: "Sling POSTs HTTP 200 + readback-confirmed all 4 writes; FormSelectServlet isAdaptiveForm now returns true for BOTH pages (identical gate to line 125) — decisive live proof line 125 will pass. isDorConfigured stays false (checks for a SELECTED XDP templateRef; AFtoDORStep does not call it)."
verification_not_performed: "End-to-end DoR PDF render not exercised — needs completing both Assign Task work items in the Workspace UI (no headless complete-work-item path), same tooling gap as D2. Stage-2 generate-mode render proven by decompile + it being the standard CC auto-DoR service, not a captured PDF this session."
scope_respected: "Only the two AF pages' guide marker + dorType touched. Workflow model (AF_PATH/DOR_PATH/Assign Task/D2 OR-split/D4/SubmissionDate), fragments, theme, clientlib, rules, submit action untouched."
confidence: "HIGH that AFtoDORStep line 125 now passes (deterministic bytecode gate + live isAdaptiveForm:true, the identical check). MEDIUM-HIGH that the full DoR PDF renders end-to-end (generate mode is the standard supported CC auto-DoR path via DoRService.render, but not exercised through a completed work item this session)."
recommendation_if_gate_still_fails: "If a post-redeploy Approve->Approve run still yields no DoR PDF despite guide='1' + isAdaptiveForm:true, the residual would be stage-2 DoRService.render for generate mode — at which point the CC-native alternative is authoring a real XDP DoR template asset and switching dorType->select + dorTemplateRef (a heavier, separate task). Do NOT re-toggle dorType blindly."
build_summary: ".claude/agents/runs/2026-07-22-employee-training-request/implementation/formwright.md"
gate_result: FIXES_APPLIED_PENDING_RETEST
next: aem-forms-program-agent -> forgemaster (rebuild/redeploy — picks up guide='1' + dorType on both pages) -> sentinel (Approve->Approve E2E, confirm node6 AFtoDORStep no longer throws at line 125 and GET <payload>/DocumentofRecord/DoR.pdf returns a real PDF; confirm the form still renders/edits/submits with the guide marker present)
```
