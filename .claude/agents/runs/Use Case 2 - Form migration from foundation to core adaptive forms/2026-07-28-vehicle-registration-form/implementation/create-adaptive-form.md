# create-adaptive-form — vehicle-registration-form

**Phase:** 3 · **Status:** COMPLETE (DEFECT FIX applied 2026-07-28 — see section below)

## DEFECT FIX (2026-07-28) — Address/Declaration sections converted from fragment embeds to INLINE fields

**User-reported defect:** the deployed form's Address Details (addressSection) and Declaration
(declarationSection) panels embedded `address-details-fragment` and `declaration-fragment` **by
reference** (`aem-adaptive-forms-agents/components/adaptiveForm/fragment`, `fragmentPath=...`).
Because those 6 fields (Address Line, City, State, Pincode, Submitted By, Date) lived only inside
the external fragment pages' own content, they did **not appear in this form's own hierarchy /
form-editor tree** — a real defect for an author who needs to see/edit every field of this form
in one place.

**Fix applied:** removed both `adaptiveForm/fragment` nodes (`addressDetailsFragment`,
`declarationFragment`) from `addressSection` / `declarationSection` and replaced them with the
**same fields authored INLINE** as direct panel children, preserving exact field parity with what
the fragments rendered (same `name`, label, type, `dataRef`, `required`, messages):
- `addressSection` now has 4 inline fields: `addressLine` (text, full width), `city`, `state`,
  `pincode` (3-col row).
- `declarationSection` keeps its inline static text (`declarationConsentText`) and now also has 2
  inline fields: `submittedBy`, `declarationDate` (2-col row).

The shared `address-details-fragment` and `declaration-fragment` pages themselves were **NOT
modified or deleted** — other forms in this project may still reference them; this form simply
stopped embedding them. `dataRef` bindings (`$.address.*`, `$.declaration.*`) were kept unchanged
so the existing schema (which already nests `address`/`declaration` objects matching those paths)
required no change.

**Grep verification (post-fix):**
```
grep -c "adaptiveForm/fragment" vehicle-registration-form/.content.xml   → 0
grep -c "fragmentPath="          vehicle-registration-form/.content.xml   → 0
```
All ~20 field nodes (addressLine, city, state, pincode, submittedBy, declarationDate included) are
now direct descendants of a panel inside `guideContainer` — confirmed by listing every opening tag
name in the file.

Also added at this step: native `pattern="^[0-9]{6}$"` + `validatePatternMessage` on `pincode`
(tightened from the fragment's generic `^[0-9]+$`), and `validationExpression`/`validateExpMessage`
+ a full `fd:validate` AST (not-a-future-date) on `declarationDate` — see `create-form-rules.md`
for the complete Rule-Editor rule inventory added in the companion defect-fix pass.

## Artifacts produced
1. Form page: `ui.content/src/main/content/jcr_root/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form/.content.xml`
2. DAM guide asset: `ui.content/src/main/content/jcr_root/content/dam/formsanddocuments/aem-adaptive-form-agents/vehicle-registration-form/.content.xml`
3. Per-form conf context: `ui.content/src/main/content/jcr_root/conf/forms/aem-adaptive-form-agents/vehicle-registration-form/.content.xml`
4. Filter entries added to `ui.content/src/main/content/META-INF/vault/filter.xml`:
   - `/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form`
   - `/content/dam/formsanddocuments/aem-adaptive-form-agents/vehicle-registration-form`
   - `/conf/forms/aem-adaptive-form-agents/vehicle-registration-form`

(Artifact 5 — the validation clientlib — is a separate deliverable; see `create-form-clientlib.md`.)

## Template / theme / schema wiring
- `cq:template` = reused `/conf/aem-adaptive-forms-agents/settings/wcm/templates/blank-af-v2` (no new template — Phase 2 skipped per PLAN).
- `schemaType="jsonschema"`, `schemaRef="/content/dam/formsanddocuments/schema/vehicle-registration-form.schema.json"`.
- `themeRef="/apps/fd/af/themes/aem-adaptive-forms-agents-vehicle-registration"` (a **new dedicated theme** — see note below), identical across form page + DAM asset + conf context (rule 3ace).
- `clientLibRef="aem-adaptive-forms-agents.vehicle-registration-form"` — a **single category** (self-contained form clientlib), not a comma list. This deliberately deviates from the PLAN's `solution-architecture.yaml` comma-separated "base,form-specific" clientlib wiring — see the "Deviation from PLAN" note in `formwright.md`.
- Submit: default shared workflow action (`fd/dashboard/components/actions/aemworkflowsubmit`, `workflowModel=/var/workflow/models/assign-task-to-admin`), `dataXMLType=FOLDER_PAYLOAD`, `dataXMLPath=data.xml`, `storeAfSubmittedData=true`. **Groundsmith** owns any further submit/PDF/workflow changes.
- Explicit AF Title component (`formTitle`, `fd:htmlelementType="h1"`) + `showTitle="{Boolean}false"` on guideContainer (rule 3c — avoids the empty showTitle-band defect).

## Structure built (in order)
- `formTitle` (AF Title) + `formSubtitle` (plain-text)
- `ownerDetailsPanel` "1. OWNER DETAILS" — 6 INLINE fields, 3-col (width=4): ownerFullName, dateOfBirth, gender, contactNumber, emailAddress, emergencyContactNo
- `vehicleDetailsPanel` "2. VEHICLE DETAILS" — 5 INLINE fields, 3-col (width=4): vehicleType (default `electric_vehicle`), makeModel, registrationYear, chassisNumber, fuelType
- `addressSection` "3. ADDRESS DETAILS" — **4 INLINE fields** (defect fix, was a fragment embed): `addressLine` (full width, width=12), then a 3-col row (width=4): `city`, `state`, `pincode`
- `documentInsurancePanel` "4. DOCUMENT & INSURANCE DETAILS" — 3 INLINE fields, 3-col (width=4): rcBookNumber, insurancePolicyNumber, pucCertificateNumber
- `declarationSection` "5. DECLARATION" — inline static text ("I confirm that the above information is accurate.") + **2 INLINE fields** (defect fix, was a fragment embed), 2-col (width=6): `submittedBy`, `declarationDate`
- `buttonRow` — `submit` (`actions/submit`, full Click AST + `fd:events click="[submitForm()]"`) + `reset` (`actions/reset`, empty `fd:events`, native reset — no `fd:click`)

All 20 fields carry `aria-label`, `required="{Boolean}true"`, `mandatoryMessage`, and a `<cq:responsive>` child. Dropdowns use `enum`/`enumNames` attributes (never `<items>`).

## New dedicated theme — deviation from PLAN (documented)
PLAN (`solution-architecture.yaml`) specified reusing `/apps/fd/af/themes/aem-adaptive-forms-agents-wknd`.
**That theme does not exist anywhere in this repository** (verified: only per-form themes exist —
college-admission, doctor-appointment, employee-reg, patient-registration, school-admission,
sports-event-registration, testmigration). Per critical rule 3ace ("a branded form needs its OWN
dedicated theme, wired in BOTH places") and the established convention of every other form in this
project, `vehicle-registration-form` gets its own dedicated theme
(`aem-adaptive-forms-agents-vehicle-registration`) instead — see `create-form-theme.md`.

## Fragment reuse — SUPERSEDED by the 2026-07-28 defect fix
This form previously embedded `address-details-fragment` and `declaration-fragment` **by
reference** (`fragments_built: 0`, both reused, no edits to either fragment page). Per the defect
fix above, this form **no longer embeds either fragment** — both sections are now built as inline
panel children so every field is visible in this form's own hierarchy. The shared fragment pages
themselves remain untouched in source control (`address-details-fragment/.content.xml`,
`declaration-fragment/.content.xml`) in case another form still references them.
