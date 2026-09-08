# Phase 9 — create-form-clientlib · Vehicle Registration Form

**Run:** 2026-07-01-vehicle-registration-form
**Skill:** create-form-clientlib
**Lead:** Blockwright (IMPL build) · Phase 9
**Status:** PASSED (self quality gate)
**Deployed:** NO — author-only. Deployment is centralized in Auditon (pipeline mode). No `mvn` run.

## Inputs consumed
- `design/DESI-design-form-components.yaml` — validation_fn references per field (authoritative for function names/mapping)
- `plan/PLAN-architect-form-solution.yaml` — clientlib function list
- `implementation/phase03-create-adaptive-form.md` — scaffold clientlib + guideContainer clientLibRef
- Form XML `content/forms/af/{appFolder}/vehicle-registration-form/.content.xml` — panel/field structure + actual clientLibRef value

## Clientlib built out (existing Phase 3 scaffold — not duplicated)
`ui.apps/src/main/content/jcr_root/apps/clientlibs/vehicle-registration-form-clientlib/`
- `.content.xml` — cq:ClientLibraryFolder, `allowProxy="{Boolean}true"`, `dependencies="[core.forms.components.runtime.all]"`
- `js.txt` — `#base=js` → `functions.js`
- `css.txt` — `#base=css` → `form.css`
- `js/functions.js` — 5 global-scope validators
- `css/form.css` — layout + initial-load fixes owned by this skill

## CRITICAL FIX — category mismatch corrected
The Phase 3 scaffold `.content.xml` declared `categories="[vehicle-registration-form-clientlib]"`, but the
form's `guideContainer` sets `clientLibRef="{project}.forms.vehicle-registration-form"`.
These did **not** match, so the clientlib would NOT have resolved. Corrected the clientlib category to
`{project}.forms.vehicle-registration-form` to match the guideContainer clientLibRef exactly.

- guideContainer `clientLibRef` = `{project}.forms.vehicle-registration-form` (form XML line 19)
- clientlib `categories` = `[{project}.forms.vehicle-registration-form]`  → **RESOLVES**

## functions.js — 5 global-scope, empty-safe validators (exact names)
| Function | Field(s) | Rule (Phase 4) |
|---|---|---|
| `validateMobile10Digits` | ownerDetailsPanel.mobileNumber | `validateMobile10Digits($field.$value) == true()` (^[0-9]{10}$) |
| `validatePin6Digits` | ownerDetailsPanel.pinCode | `validatePin6Digits($field.$value) == true()` (^[0-9]{6}$) |
| `validateEmailWhenPresent` | ownerDetailsPanel.emailAddress | `validateEmailWhenPresent($field.$value) == true()` (blank passes — does NOT block submit) |
| `validateDateDDMMYYYY` | dateOfBirth, insuranceDetailsPanel.policyValidTill, declarationPanel.declarationDate | real DD/MM/YYYY (or ISO) calendar date |
| `validateDobNotFuture` | ownerDetailsPanel.dateOfBirth | date not in the future |

- **Global scope confirmed:** 5 top-level `function` declarations; 0 IIFE / `window.` / namespace / arrow patterns (grep = 0). Rule-Editor scanner will discover all 5.
- **Empty-safe:** every validator returns `true` for null/undefined/"" — `required` owns emptiness; blank email/date never wrongly blocks submit.
- **Full JSDoc:** each has a description line + `@name <id> <label>` + `@param` + `@return {boolean}`.
- **Dates** accept both DD/MM/YYYY display and canonical ISO yyyy-mm-dd model value; real-calendar-date check (rejects 31/02, etc.).
- Ends with a **Rule-Editor usage notes** block mapping field → function → rule type.
- `node -c` syntax check: OK.

## css.txt / form.css (lightweight — theme owns branding)
`css.txt` present and lists `form.css`. `form.css` carries only the layout root causes THIS skill owns:
1. **Panel multi-column row layout** via float→flex on `.panelcontainer .cmp-container > .aem-Grid`
   (theme processor strips grid selectors, so this must live in the clientlib), with mobile collapse to 1 column.
2. **Native date-input width clamp** — `.cmp-adaptiveform-datepicker__widget { width/max-width:100%; min-width:0; box-sizing:border-box; }`.
3. **Suppress empty-errormessage glyph at initial load** — `…__errormessage:empty::before { content:none !important; }`
   across textinput/email/telephone/number/datepicker/dropdown/checkboxgroup/radiobutton.

No JS validates on load; validation runs only via the Phase 4 Validate rules (blur/change/submit).

## Self quality gate (verified)
- [x] `cq:ClientLibraryFolder` with `allowProxy="{Boolean}true"` (XML parses OK)
- [x] `categories` matches guideContainer `clientLibRef` exactly → **clientLibRef resolves**
- [x] `dependencies` includes `core.forms.components.runtime.all`
- [x] `js.txt` lists `functions.js` (with `#base=js`); `css.txt` lists `form.css` (with `#base=css`)
- [x] All 5 functions global-scope (no IIFE/namespace), exact names, empty-safe, full JSDoc
- [x] functions.js ends with Rule-Editor usage-notes block; expressions documented as `== true()`
- [x] css.txt present; CSS scoped to AF classes (grid fix, date clamp, error-glyph suppression)
- [x] No deploy — author-only (no `mvn`)

## Handoff / notes
- Phase 4 (create-form-rules) authors the `fd:rules/@fd:validate` ASTs + `validationExpression`/`validateExpMessage`
  attributes that call these functions; function names above match the DESI validation_fn references exactly.
- Scaffold reused in place — no duplicate clientlib created elsewhere.
