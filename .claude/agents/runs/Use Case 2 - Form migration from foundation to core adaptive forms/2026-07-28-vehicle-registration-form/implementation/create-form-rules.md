# create-form-rules — vehicle-registration-form

**Phase:** 4 · **Status:** COMPLETE (RE-EMBED pass applied 2026-07-28 — see section below; supersedes the DEFECT FIX section for R15/R20)

## RE-EMBED (2026-07-28, second pass) — pincode / declarationDate rules moved onto the shared fragments

The DEFECT FIX below (first pass) inlined `pincode` and `declarationDate` and authored their
`fd:validate` rules directly on the host form. That has now been **reversed**: fragment reuse for
Address Details / Declaration is restored, so R15 (pincode) and R20 (declarationDate) are now
authored **on the shared fragment fields themselves**, not on the host form:

| Rule | Field | Fragment | Mechanism |
|---|---|---|---|
| R15 | `pincode` | `address-details-fragment` (panel `addressDetailsPanel`) | Changed `sling:resourceType` to `numberinput`/`number-input`, native `pattern="^[0-9]{6}$"` (was the fragment's original generic `^[0-9]+$`) + `validationExpression="validateExactSixDigits($field.$value) == true()"` + full `fd:validate` AST targeting `$form.addressDetailsPanel.pincode`, displayPath `FORM/Address Details/Pincode / ZIP Code/`. |
| R20 | `declarationDate` | `declaration-fragment` (panel `declarationPanel`) | `validationExpression="validateNotFutureDate($field.$value) == true()"` + full `fd:validate` AST targeting `$form.declarationPanel.declarationDate`, displayPath `FORM/Declaration/Date/`. |

Both ASTs are modeled byte-for-byte on vehicle-registration-form's own previously-working
`dateOfBirth`/`contactNumber` rule shape (`VALIDATE_EXPRESSION`→`FUNCTION_CALL`==`BOOLEAN_LITERAL
True`, `eventName:"Validate"`, `communicationComposerRuleType:"server"`), only the
`AFCOMPONENT`/`COMPONENT` id/name/displayPath/parent were retargeted to the fragment's own panel
name (`addressDetailsPanel`, `declarationPanel` — NOT `addressSection`/`declarationSection`, which
are the host form's panel names and no longer contain these fields directly).

Because these two fragments are embedded by 5 forms total (vehicle-registration-form plus
college-admission-registration, doctor-appointment-registration, school-admission-registration,
sports-event-registration), these rules now apply to all 5 — intentional, per instruction ("the
added validations are universally valid and should apply to them too").

**Function availability caveat (flagged, not fixed here):** a `fd:validate`/`validationExpression`
AST only *renders* in the Rule Editor from the fragment's own source; the JS function it calls
(`validateExactSixDigits`, `validateNotFutureDate`) must be defined in whatever clientlib is
actually loaded on the page that embeds the fragment (the fragment's own `guideContainer` only
loads `core.forms.components.runtime.all`, no custom functions). Checked all 5 consuming forms'
`functions.js`:

| Form | `validateExactSixDigits` | `validateNotFutureDate` |
|---|---|---|
| vehicle-registration-form | yes (pre-existing) | yes (pre-existing) |
| college-admission-registration | **missing** | **missing** |
| doctor-appointment-registration | **missing** | yes |
| school-admission-registration | **missing** | yes |
| sports-event-registration | **missing** | **missing** |

Only `vehicle-registration-form` is fully covered today. The other 4 forms were explicitly
excluded from this pass ("do not modify those 4 forms"), so on those forms the new fragment rule
will attempt to call an undefined function at validation time. See `formwright.md`'s "RE-EMBED
addendum" for the recommended follow-up (promote both functions into the shared
`{project}.forms.base` clientlib).

**Grep evidence:**
- `address-details-fragment/.content.xml`: `fd:validate`/`validationExpression` present on `pincode`.
- `declaration-fragment/.content.xml`: `fd:validate`/`validationExpression` present on `declarationDate`.
- `vehicle-registration-form/.content.xml`: 0 occurrences of `pincode`/`declarationDate` as inline
  node names (both now live only in the fragments); 2 `adaptiveForm/fragment` embed nodes present.

**Author-only — no `mvn` run.** Deployment is deferred to the user's package-manager install /
Forgemaster's centralized build+deploy.

---

## DEFECT FIX (2026-07-28) — validations authored as VISIBLE Rule-Editor rules, not clientlib-only

> **NOTE: R15 (pincode) and R20 (declarationDate) below are superseded by the "RE-EMBED" section
> above — those two rules now live on the shared fragments, not on this host form.** The rest of
> this section (R01–R05, R09, and the functions list) is still accurate and unchanged.

**User-reported defect:** "the general/default validation rules are not configured in the form and
are not visible in the Rule Editor (they were implemented only as clientlib JS)". This referred
specifically to the pincode (exact-6-digit) and declarationDate (not-future) supplemental rules,
which the original build (see the superseded section below) implemented as Layer-2 client-side JS
(`fragment-rules.js`) because those two fields lived inside embedded Adaptive Form Fragments and
could not be safely targeted by a hand-authored `fd:validate` AST from the host form's source.

Now that `create-adaptive-form` has inlined both fields as direct panel children (no more fragment
embed — see `create-adaptive-form.md`), every validation below is authored as a REAL, Rule-Editor-
visible rule directly on the field: `validationExpression` (runtime) + a full escaped-JSON
`fd:validate` AST (editor visibility) + `validateExpMessage`, modeled byte-for-byte on this form's
own already-working `dateOfBirth` / `registrationYear` rule shape (`VALIDATE_EXPRESSION` →
`COMPARISON_EXPRESSION` → `FUNCTION_CALL` == `BOOLEAN_LITERAL True`, `impl:"Scaffold(the)"`,
`communicationComposerRuleType:"server"`), only retargeting the `AFCOMPONENT` id / name /
`displayPath` / parent to each new field. `fragment-rules.js` is now DELETED (superseded) — its
logic is fully replaced by native rules; nothing was left dead/duplicated.

## Rules authored on INLINE fields (Rule-Editor visible)

| Rule | Field | Panel | Mechanism |
|---|---|---|---|
| R01 | ownerFullName | ownerDetailsPanel | Native `pattern="^[A-Za-z][A-Za-z .'-]*$"` + `validatePatternMessage`. No additional `fd:validate` AST — the native pattern fully covers the requirement and `validateName` (still defined in functions.js, unused/available) would be redundant. |
| R02 | dateOfBirth | ownerDetailsPanel | `validationExpression="validateNotFutureDate($field.$value) == true()"` + full `fd:validate` AST (unchanged from original build). |
| R03 | contactNumber | ownerDetailsPanel | Native `pattern="^[0-9]{10}$"` **+ NEW** `validationExpression="validatePhoneNumber($field.$value) == true()"` + full `fd:validate` AST targeting `$form.ownerDetailsPanel.contactNumber`, displayPath `FORM/1. OWNER DETAILS/Contact Number/`. |
| R04 | emergencyContactNo | ownerDetailsPanel | Same as R03, function `validatePhoneNumber`, targeting `$form.ownerDetailsPanel.emergencyContactNo`, displayPath `FORM/1. OWNER DETAILS/Emergency Contact No./`. |
| R05 | emailAddress | ownerDetailsPanel | Native `pattern="^[^@\s]+@[^@\s]+\.[^@\s]+$"` **+ NEW** `validationExpression="validateEmailFormat($field.$value) == true()"` + full `fd:validate` AST targeting `$form.ownerDetailsPanel.emailAddress`, displayPath `FORM/1. OWNER DETAILS/Email Address/`. |
| R09 | registrationYear | vehicleDetailsPanel | Native `pattern="^[0-9]{4}$"` + `validationExpression="validateYearNotFuture($field.$value) == true()"` + full `fd:validate` AST (unchanged from original build). |
| R15 | pincode | addressSection | **CHANGED from clientlib-only.** Native `pattern="^[0-9]{6}$"` (tightened from the fragment's generic `^[0-9]+$` now that the field is inline and form-owned) **+ NEW** `validationExpression="validateExactSixDigits($field.$value) == true()"` + full `fd:validate` AST targeting `$form.addressSection.pincode`, displayPath `FORM/3. ADDRESS DETAILS/Pincode / ZIP Code/`. |
| R20 | declarationDate (not-future) | declarationSection | **CHANGED from clientlib-only.** `validationExpression="validateNotFutureDate($field.$value) == true()"` + full `fd:validate` AST targeting `$form.declarationSection.declarationDate`, displayPath `FORM/5. DECLARATION/Date/`. |
| — | Required (all ~20 fields) | all panels | `required="{Boolean}true"` + `mandatoryMessage` — already present on every field (confirmed, no gaps). |

All ASTs use the VERIFIED shape already proven working in this exact form (dateOfBirth /
registrationYear), not hand-guessed from scratch — only the `AFCOMPONENT`/`COMPONENT` id, name,
`displayPath`, and parent were retargeted per field.

## "Default to today" for declarationDate — NOT authored as a Rule-Editor AST (documented reason)

The defect list also asked for "Declaration Date: default to today." This is a **default-value
convenience**, not a validation rule — the actual validation (required + not-future) above IS a
real Rule-Editor rule. For the default itself, I deliberately did **not** hand-author an `fd:init`
(Initialize) or `fd:value` (Calculate) AST: this repository has **zero verified in-repo examples**
of either AST shape (`grep -rl "fd:init=" ui.content` and `grep -rl "SET_VALUE" ui.content` both
return no matches), and the skill's own guidance is explicit that a single malformed `fd:*`
property on any `<fd:rules>` node **breaks Rule Editor rule-tree traversal for the WHOLE form** —
not just the field it's on — and that an unverified AST should never be hand-guessed; it should be
round-tripped through the live Rule Editor and diffed. Given no running instance to verify against
here, hand-guessing that specific AST was judged too risky given the blast radius (it would have
put every other newly-added, now-working `fd:validate` rule at risk too).

**Mechanism used instead:** a small, safe, non-invasive clientlib script,
`ui.apps/.../vehicle-registration-form-clientlib/js/declaration-date-default.js` (replacing the
deleted `fragment-rules.js`), which — on form render, only if `declarationDate` is still empty —
sets the native date input's value to today's date and dispatches real `input`/`change` DOM events
(never a direct model/value setter), so the framework's own bound listeners (including the new
Rule-Editor "not a future date" validation above) pick it up exactly as if the user had typed it.
This script touches **no** `<fd:rules>` node and carries none of the whole-form-breaking risk.

**Flagged for Sentinel/Forgemaster:** if a verified in-repo `fd:init`/`fd:value` AST reference
becomes available on the running instance, migrating this default into a native Rule-Editor rule
is the more discoverable long-term home — optional, not required for this defect fix, which only
required the VALIDATION (not the default) to be Rule-Editor visible, and that is done.

## Functions (all in `vehicle-registration-form-clientlib/js/functions.js`, global-scope, empty-safe, unique `@name` each)
- `validateName` — available, not currently wired to an additional `fd:validate` rule (native pattern covers ownerFullName).
- `validateNotFutureDate` — dateOfBirth, declarationDate.
- `validatePhoneNumber` — **NEW** — contactNumber, emergencyContactNo.
- `validateEmailFormat` — **NEW** — emailAddress.
- `validateExactSixDigits` — pincode (now called from a real `fd:validate` AST instead of `fragment-rules.js`).
- `validateYearNotFuture` — registrationYear.

## Superseded (kept for history) — original fragment-scoped mechanism, no longer in effect

*The section below describes the PRE-defect-fix state and is retained only for traceability; it
does NOT reflect the current form.*

~~R15 and R20 were originally implemented as fragment-scoped supplemental rules because
`address-details-fragment` and `declaration-fragment` were embedded by reference and their fields
were not JCR nodes in this host form's `.content.xml`. The mechanism used then was Layer-2
client-side JS (`fragment-rules.js`) matching the rendered `pincode`/`declarationDate` inputs by
`name` attribute, showing a non-invasive DOM-only error element, and gating Submit. That file has
been DELETED — its two validations are now real Rule-Editor rules (see above) and its "default
today" behavior lives on in `declaration-date-default.js`.~~
