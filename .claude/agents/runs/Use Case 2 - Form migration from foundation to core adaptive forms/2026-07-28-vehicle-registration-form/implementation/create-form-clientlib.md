# create-form-clientlib — vehicle-registration-form

**Phase:** 9 · **Status:** COMPLETE

## Artifacts produced
`ui.apps/src/main/content/jcr_root/apps/clientlibs/vehicle-registration-form-clientlib/`
- `.content.xml` — category `aem-adaptive-forms-agents.vehicle-registration-form` (SINGLE
  category), `dependencies="[core.forms.components.runtime.all]"`, `allowProxy=true`
- `css.txt` (`base.css`, `form.css`) / `css/base.css` (self-contained copy) / `css/form.css` (form-unique)
- `js.txt` (`functions.js`, `accessibility.js`, `fragment-rules.js`) / the three JS files

## Clientlib architecture decision — single category, self-contained (deviation from PLAN)
PLAN's `solution-architecture.yaml` specified a comma-separated `clientLibRef`
(`"aem-adaptive-forms-agents.forms.base,vehicle-registration-form-clientlib"`), following
formwright's generic rule 3ab (base + form-specific, comma list). **This build instead follows
the `create-adaptive-form`/`create-form-clientlib` skills' own VERIFIED, more specific project
learning**: `clientLibRef` MUST be a single category, or the Rule-Editor customfunctions endpoint
resolves it as one category and returns an empty function list (every custom-fn rule shows
"Broken"). Every existing sibling form in this repo (employee-registration-form,
sports-event-registration, etc.) uses exactly this single-category, self-contained pattern —
confirmed by inspecting their `.content.xml` and clientlib folders. This build follows that
established, verified precedent over the more generic PLAN instruction.

Concretely: `css/base.css` is a **verbatim copy** (not embedded/dependency) of
`clientlib-forms-base/css/base.css`; `js/functions.js` **copies** (does not embed) the subset of
`@name` validators this form's rules reference. `dependencies` carries only
`core.forms.components.runtime.all` (no `forms.base`, no `generate-pdf` — this form's submit is
the shared workflow, not a PDF action).

## `css/form.css` — form-unique
- Brand `:root` token override, identical to the dedicated theme's tokens (so the embedded-page
  form on the "Test Adaptive Form" Sites page also renders on-brand — the theme selector does not
  apply in-page; rule critical-checklist item "brand colours render in the embedded page").
- Section dividers under every numbered section (3 inline panels + 2 fragment-hosting panels).
- Multi-column grid: clientlib OWNS the per-span `flex-basis` widths (12/6/4) for both native
  panel grids AND the two embedded fragments' inner grids (`.fragment .cmp-container > .aem-Grid`),
  with the `:not(.button)` guard so Submit/Reset aren't forced to 50%. Mobile collapse to 100% under
  767px.
- Centered green Submit (`af-submit`) + outlined green Reset (`af-reset`) button row.
- `.clientlib-error` styling for the Layer-2 fragment-scoped validation (see `create-form-rules.md`).

## `js/functions.js` — self-contained Rule-Editor custom functions
4 global-scope, empty-safe, JSDoc-annotated functions (each defined exactly once):
- `validateName` (copied from base) — referenced informally by ownerFullName's native `pattern` (no Rule-Editor rule wired; see `create-form-rules.md` R01 decision)
- `validateNotFutureDate` (copied from base) — wired to `dateOfBirth` (R02) via `create-form-rules`
- `validateExactSixDigits` (NEW, form-specific) — for the pincode supplemental rule (R15/GAP-02)
- `validateYearNotFuture` (NEW, form-specific) — wired to `registrationYear` (R09) via `create-form-rules`

## `js/accessibility.js` — copied verbatim
Generic MutationObserver-based aria-label injector (from `employee-registration-form-clientlib`),
applies to any rendered `.cmp-adaptiveform-container`, including the two embedded fragments'
render surfaces.

## `js/fragment-rules.js` — see `create-form-rules.md`
Implements the two fragment-scoped supplemental rules (R15 pincode exact-6-digit, R20
declarationDate default-to-today + not-future) as non-invasive Layer-2 JS, since there is no
verified safe mechanism to attach a Rule-Editor AST to a fragment-nested field from the host
form's source without either editing the shared fragment or hand-guessing an unverified AST.

## Filter coverage
`/apps/clientlibs` is already a filter root in `ui.apps/src/main/content/META-INF/vault/filter.xml` — no change needed.
