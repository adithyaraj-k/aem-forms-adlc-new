# create-adaptive-form — vehicle-registration-form (DEFECT FIX — de-fragment pass, PASS 3)

**Phase:** 3 (form) · **Status:** COMPLETE · **Type:** structural edit to an existing form (no new form authored)

## Task

The user decided to permanently stop using embedded Adaptive Form Fragments on
`vehicle-registration-form`. Remove the two `adaptiveForm/fragment` embed components
(`addressDetailsFragment` → `address-details-fragment`, `declarationFragment` →
`declaration-fragment`) from the form's `guideContainer`, and author the same 6 fields
**natively inline**, preserving exact name / label / type / `dataRef` / `required` /
layout parity with the fragments' own field definitions.

This is a **reversal of the same-day 2026-07-28 "RE-EMBED" pass** (see
`.claude/agents/runs/2026-07-28-vehicle-registration-form/implementation/formwright.md`),
which itself had reversed an even earlier 2026-07-28 "DEFECT FIX" inlining pass. The form has
now flip-flopped between inline and fragment-embed three times; this pass is authoritative going
forward per the user's explicit, final instruction in this run.

## What changed — `vehicle-registration-form/.content.xml`

**`addressSection` panel** — removed the single `addressDetailsFragment` node
(`sling:resourceType=.../adaptiveForm/fragment`, `fragmentPath=/content/forms/af/aem-adaptive-form-agents/address-details-fragment`)
and replaced it with 4 native field nodes, copied field-for-field from
`address-details-fragment`'s `addressDetailsPanel` (the wrapper panel itself was not
re-created — `addressSection` already provides the same `layout="responsiveGrid"` width-12
panel, so the fields are direct children of `addressSection`, one nesting level shallower
than in the fragment, with **identical rendered layout**):

| Field | `sling:resourceType` | `dataRef` | Width | Notes |
|---|---|---|---|---|
| `addressLine` | `textinput` | `$.address.addressLine` | 12 | required, maxLength 250 |
| `city` | `textinput` | `$.address.city` | 4, `behavior="newline"` | required, maxLength 80 |
| `state` | `textinput` | `$.address.state` | 4 | required, maxLength 80 |
| `pincode` | `numberinput` | `$.address.pincode` | 4 | required, `pattern="^[0-9]{6}$"`, `maxLength=6`, `fd:validate` rule (see `create-form-rules.md`) |

**`declarationSection` panel** — kept the existing static `declarationConsentText` node
untouched, removed the single `declarationFragment` node
(`fragmentPath=/content/forms/af/aem-adaptive-form-agents/declaration-fragment`), and
replaced it with 2 native field nodes, copied field-for-field from
`declaration-fragment`'s `declarationPanel`:

| Field | `sling:resourceType` | `dataRef` | Width | Notes |
|---|---|---|---|---|
| `submittedBy` | `textinput` | `$.declaration.submittedBy` | 6, `behavior="newline"` | required, maxLength 100 |
| `declarationDate` | `datepicker` | `$.declaration.date` | 6 | required, `displayFormat`==`editFormat`==`date\|DD/MM/YYYY`, `fd:validate` rule (see `create-form-rules.md`) |

Every field carries `aria-label`, `mandatoryMessage`, `enabled="{Boolean}true"`,
`readOnly="{Boolean}false"`, `visible="{Boolean}true"`, and a `<cq:responsive>` child — no
attribute was dropped versus the fragment's own field definition.

**Not changed in this pass:** `ownerDetailsPanel`, `vehicleDetailsPanel`,
`documentInsurancePanel`, `buttonRow` (Submit/Reset), `formTitle`, `formSubtitle`, the
`guideContainer`'s own attributes (`schemaType`, `schemaRef`, `themeRef`, `actionType`,
`clientLibRef`) — all untouched.

## Schema / binding consistency (per this pass's requirement)

**No schema edit was needed.** `vehicle-registration-form.schema.json` already declares
top-level `address` (`addressLine`, `city`, `state`, `pincode`) and `declaration`
(`submittedBy`, `date`) objects with `aem:afProperties.fd:formDataRef` bindings that
exactly match the `dataRef` values used on the now-inline fields — this schema was
originally authored (2026-07-28) to share the SAME canonical JSONPath shape the two
fragments themselves bind to ($.address.*, $.declaration.*), specifically so it would stay
consistent whichever way (fragment-embed or inline) the address/declaration sections were
authored. Verified 1:1, both directions:

- `addressLine` → `$.address.addressLine` — in schema ✓, in form ✓
- `city` → `$.address.city` — in schema ✓, in form ✓
- `state` → `$.address.state` — in schema ✓, in form ✓
- `pincode` → `$.address.pincode` — in schema ✓, in form ✓
- `submittedBy` → `$.declaration.submittedBy` — in schema ✓, in form ✓
- `declarationDate` → `$.declaration.date` — in schema ✓, in form ✓

The schema's key set remains exactly the Hard-Rule-7-whitelisted set established by the
2026-08-03 PASS 1 + PASS 2 schema fixes earlier in this same run (`$schema`, `$id`,
`title`, `type`, `properties`, `required`, `enum`, `maxLength`, `pattern`, `default`,
`format`, `aem:afProperties`, `fd:formDataRef`) — this pass did not touch the schema file
at all. `generate-schema` was NOT re-invoked because no schema change was required.

`vehicle-registration-form.bindings.json` — updated only its `canonical_shapes` doc
comment (not a functional field) to record that `address`/`declaration` are now bound
inline on this form while remaining the canonical shape the shared fragments still use on
5 other forms.

## Fragment references removed (this form ONLY) — fragments themselves NOT deleted

**⚠️ Flagged deviation from the literal task instruction "DELETE the fragment artifacts
entirely" — see `formwright.md` PASS 3 for the full rationale.** `address-details-fragment`
and `declaration-fragment` are actively embedded by 5 OTHER forms in this repository
(`college-admission-registration`, `doctor-appointment-registration`,
`patient-registration-form`, `school-admission-registration`, `sports-event-registration`).
Deleting the fragment `cq:Page`s, their DAM assets, conf contexts, or filter entries would
break all 5 of those already-built forms (their embedded `adaptiveForm/fragment` nodes
would point at a 404'd path). This pass therefore:

- Removed ONLY `vehicle-registration-form`'s two `fragmentPath` references (above).
- Left `address-details-fragment` / `declaration-fragment` — their `.content.xml` (form,
  DAM, conf), and the filter.xml entries for all three of their paths — **completely
  untouched**. Grep-verified: `git diff`/file content shows zero changes to any
  fragment-owned file in this pass.

If the user's real intent is to retire these fragments repo-wide, that requires a
SEPARATE, coordinated de-fragmentation of all 6 consuming forms — out of scope for "fix
vehicle-registration-form."

## Grep verification performed

```
grep -c "adaptiveForm/fragment" vehicle-registration-form/.content.xml   → 0
grep -c "fragmentPath="          vehicle-registration-form/.content.xml   → 0
grep -c "<addressLine|<city|<state|<pincode|<submittedBy|<declarationDate" → 6 (one open-tag each)
[xml] parse of vehicle-registration-form/.content.xml                    → well-formed
```

## Orphan-node risk on redeploy (flag to Forgemaster)

The form's filter root `/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form`
in `ui.content/.../META-INF/vault/filter.xml` has **no explicit `mode` attribute**, so
FileVault import uses the **default `replace` mode** for that subtree — which DOES delete
instance nodes not present in the package. This means the old
`guideContainer/addressSection/addressDetailsFragment` and
`guideContainer/declarationSection/declarationFragment` component nodes should be purged
automatically on the next full package install/redeploy, without a manual
`:operation=delete` step. **Still verify this on the deployed instance** (see
`formwright.md` → "What the user must do to verify") — if either old node somehow survives
(e.g. a prior partial/`-pl` deploy left the filter root registered differently, or a cached
package uses `update` semantics), manually purge:
- `/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form/jcr:content/guideContainer/addressSection/addressDetailsFragment`
- `/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form/jcr:content/guideContainer/declarationSection/declarationFragment`

## Clientlib — no change needed

`vehicle-registration-form-clientlib`'s `functions.js` already defines
`validateExactSixDigits` and `validateNotFutureDate` (added in the earlier 2026-07-28
inline pass and never removed even when the form was re-embedded) — both are called by
the re-authored `fd:validate` rules on `pincode` / `declarationDate` (see
`create-form-rules.md`). `declaration-date-default.js` (the "default to today" UX
convenience) and `css/form.css` (which already targets plain `.panelcontainer` grids with
no `.fragment`-scoped selectors) both already assumed the inline structure — neither
needed any edit. Verified by reading both files before editing the form: zero clientlib
changes were required.

## Author-only

No `mvn` build/deploy was run. Deployment is deferred to the user / to Forgemaster's
single authoritative build+deploy step.
