# create-form-rules — employee-training-request

Target: `/content/forms/af/aem-adaptive-form-agents/employee-training-request` (host form)
Also touches: `employee-training-request-clientlib` (NEW — see deviation note below)

## 8 rules migrated from DocRoute.xdp XFA JavaScript

| # | Rule | Mechanism | Storage | Confidence |
|---|---|---|---|---|
| 1 | ManagerId/ManagerName lock when EmployeeId<=10 (VALUE deferred to Groundsmith prefill — no hardcoded 'venuga5'/'Arjun Venugopal') | `fd:change` on the `employeeDetailsFragment` Fragment-reference node, reacting to nested `employeeId`, toggling nested `managerId.enabled`/`managerName.enabled` | AST on `employeeDetailsFragment` | **LOWEST** — first cross-fragment-boundary rule in this project, no prior in-repo precedent; verify live |
| 2 | Email format validation | Built-in `validatePictureClause` on the email field, already authored in the `employee-identity` fragment | none needed (no Rule-Editor rule, no clientlib) | High — built-in picture-clause is standard AEM Forms mechanism |
| 3 | RequestType locked until RequestId non-empty | `fd:enabled` on `requestType`, condition `requestId.$value != ""` | AST + mirror `enabled` attr | Medium — first `fd:enabled` use in this project (⚠️ follow-the-pattern per skill) |
| 4 | EndDate must be strictly after StartDate | `validationExpression="validateDateAfter(...)==true()"` + `fd:validate` AST on `endDate` | AST + `validationExpression` | High — copies the leave-request-form's proven `fd:validate` AST shape; function copied from base clientlib |
| 5 | ApprovalInformation panel visible only when EmployeeId non-empty AND CurrentStatus != 'FinanceApproved' | `fd:visible` AND-combined SHOW_EXPRESSION on `approvalInformationPanel`, referencing `employeeDetailsFragment.employeeId` (cross-fragment) and `currentStatus` | AST + mirror `visible` attr; panel default flipped to `visible="{Boolean}false"` | Medium — verified `fd:visible` shape, but the cross-fragment reference is unprecedented; verify live |
| 6 | FinanceComments unlocks only when CurrentStatus == 'ManagerApproved' | `fd:enabled` on `financeComments` | AST + mirror `enabled` attr | Medium (same caveat as #3) |
| 7 | AcceptedTerms must be checked before submit | Built-in `required="true"` + `mandatoryMessage`, already authored in the `declaration-consent` fragment | none needed | High |
| 8 | SubmissionDate system-set at submit time | Extended the submit button's `fd:click` AST: `SET_VALUE` on `declarationFragment.submissionDate` (`new Date().toISOString()`) immediately before `SUBMIT_FORM`; `fd:events click` updated to `[declarationFragment.submissionDate.$value = new Date().toISOString(), submitForm()]` | AST + `fd:events` | Medium — extends the verified submit-click pattern with an added SET_VALUE step |

## Deviation from DESI's default clientlib plan

DESI's `clientlib_plan` (component-design-spec.yaml) defaulted to **SKIP** `create-form-clientlib` unless
the Rule Editor couldn't reproduce the legacy alert-dialog UX for **email** validation. Email turned out
to need no clientlib (built-in picture-clause suffices). However, rule #4 (EndDate > StartDate) needs a
**cross-field date comparison function** — this project's existing `leave-request-form` establishes the
precedent that this class of rule is NOT natively expressible via the Rule Editor's comparison operators
alone and requires a custom function. The shared base clientlib (`aem-adaptive-forms-agents.forms.base`)
already has a generic `validateDateAfter(value, other)` function for exactly this pattern (used by its
`departureDate` usage note). Per project rule 3ab (self-contained form clientlib, never embed/depend on
`forms.base` to avoid duplicate `@name` registration), I created a **minimal** new clientlib —
`employee-training-request-clientlib` (`ui.apps/.../apps/clientlibs/employee-training-request-clientlib/`)
— containing a **verbatim self-contained copy** of `validateDateAfter` (+ its private date-parser helper).
No other custom functions were needed. This is a justified, minimal deviation, not a rebuild of DESI's
plan — recorded here per formwright's mandatory clientlib-placement rule.

`clientLibRef="aem-adaptive-forms-agents.forms.employee-training-request"` (single category, already set
on the form's `guideContainer` in Artifact 1) resolves to this new clientlib.

## Confirmed — no changes needed

- **Rule 2 (Email)**: `validatePictureClause`/`validatePictureClauseMessage` already on the
  `employee-identity` fragment's `email` field. No Rule-Editor rule or clientlib function needed.
- **Rule 7 (AcceptedTerms)**: `required="true"` + `mandatoryMessage` already on the
  `declaration-consent` fragment's `acceptedTerms` field. No additional rule needed.

## Files touched

- `ui.content/.../jcr_root/content/forms/af/aem-adaptive-form-agents/employee-training-request/.content.xml`
  (rules 1, 3, 4, 5, 6, 8 authored as `fd:rules`/`fd:events` node edits)
- `ui.apps/.../jcr_root/apps/clientlibs/employee-training-request-clientlib/` (NEW — `.content.xml`,
  `js.txt`, `css.txt`, `js/functions.js`; `css/form.css` completed in the theme/clientlib-styling step)

## Verification required post-deploy (flag to Sentinel/Forgemaster)

- Confirm all 8 rules appear correctly in `guideContainer.model.json` and the live Rule Editor —
  especially rules #1 and #5 (the two cross-fragment-boundary rules, novel to this project).
- Confirm 0 occurrences of `fd:visibility`/`SET_PROPERTY` (Foundation-era mistakes) in the deployed model.
- Confirm `validateDateAfter` resolves without `ReferenceError` (clientlib JS must serve at
  `/etc.clientlibs/clientlibs/employee-training-request-clientlib.js`).

---

## Fix pass 1 (post-Sentinel gate-failure #1)

Sentinel's live verification (`testing/integration-test-report.md`) confirmed 6/8 rules working
(#2,#3,#4,#5,#7,#8) and 2/8 broken live (#1, #6) — exactly the two rules this document's original
confidence table flagged LOWEST/"follow-the-pattern". Root causes diagnosed from Sentinel's evidence
(live `guideContainer.model.json` + raw JCR node inspection) and fixed as follows.

### Rule #1 (D-R1) — ManagerId/ManagerName lock — WRONG MECHANISM, moved and re-authored

**Root cause:** a `Change`-event script (`EVENT_SCRIPTS`/`fd:change`) attached to the
`employeeDetailsFragment` Fragment-reference wrapper node in the HOST form never compiled into a live
event binding. Sentinel confirmed neither the host's nor the fragment's own `guideContainer.model.json`
ever showed a `"change"` key for this field, unlike the Submit button's `click` script (Rule #8), which
DOES compile — proving the mechanism itself (event-script targeting fields nested inside an embedded
fragment, attached to the host wrapper) is not a valid live binding path in this project, independent
of the specific JS content.

**Fix:** replaced the `Change`-event script entirely with two `fd:enabled` ENABLE_EXPRESSION rules,
authored directly on `managerId` and `managerName` **inside the `employee-identity` fragment's own
`.content.xml`** (not the host form) — the same mechanism class as the WORKING Rule #3
(`requestType`'s `fd:enabled`). Condition: `Number(employeeId.$value) <= 0 || Number(employeeId.$value)
> 10` (enabled when EmployeeId is NOT in the 1-10 lock range), applied as BOTH the top-level `enabled`
mirror attribute on each field AND a nested `enabled` attribute inside their `<fd:rules>` element
(sibling of `fd:enabled`) — see the Rule #6 root-cause note below for why both attributes are required.
Because `managerId`/`managerName` are direct siblings of `employeeId` within the SAME fragment, the
condition needs no cross-fragment reference at all (unlike Rule #5, which genuinely does cross from the
host into this fragment for its own, unrelated, panel-visibility condition — left untouched).
**Files touched:** `employee-identity/.content.xml` (added), `employee-training-request/.content.xml`
(removed the broken `fd:rules`/`fd:events` from `employeeDetailsFragment`).
**Deviation flag:** the `employee-identity` fragment is no longer purely data-shape-generic — see
`employee-identity-create-AdaptiveFormFragment.md` Fix pass 1 §1 for the reuse trade-off this creates.

### Rule #6 (D-R6) — FinanceComments lock/unlock — MISSING MIRROR ATTRIBUTE

**Root cause:** `financeComments`'s `<fd:rules fd:enabled="[...]" jcr:primaryType="..."
validationStatus="valid"/>` element was missing the companion simple-expression `enabled` attribute
that its WORKING sibling `requestType` (Rule #3) carries as a **direct sibling attribute of
`fd:enabled` inside the SAME `<fd:rules>` element** (`fd:rules fd:enabled="[...]" ... enabled="requestId.$value
!= ''" .../>`). Without that nested `enabled` attribute, the AF runtime evaluates `financeComments`'s
top-level `enabled="currentStatus.$value == \"ManagerApproved\""` attribute as a static, one-time value
instead of a reactive binding — it never re-evaluates as `CurrentStatus` changes. This is a genuinely
new finding (not previously documented in this file): the LIVE reactive binding depends on the nested
`<fd:rules>` attribute being present, not just the top-level field attribute.

**Fix:** added `enabled="currentStatus.$value == &quot;ManagerApproved&quot;"` as a sibling attribute
of `fd:enabled` inside `financeComments`'s `<fd:rules>` element (same value as the pre-existing
top-level mirror attribute). No change to the `fd:enabled` AST JSON itself, no change to the top-level
attribute — purely the missing nested attribute, added.
**File touched:** `employee-training-request/.content.xml` only.

### JSON well-formedness note (verification performed, per this task's explicit ask)

Both new fragment rules' JSON was round-tripped through `ConvertFrom-Json`/`ConvertTo-Json` before
embedding, and re-extracted from the deployed XML attribute post-edit to confirm it parses cleanly.
Interestingly, the PRE-EXISTING `fd:enabled` blobs on `requestType` (Rule #3, confirmed WORKING live by
Sentinel) and the original `financeComments` (Rule #6) do NOT parse as standalone JSON even after
undoing FileVault's `\,` comma-escaping — both have a `],"isValid":true,...` trailing shape that isn't
valid JSON on its own. Since Rule #3 is proven working live despite this, the AEM Rule Editor evidently
does not require this AST blob to be strictly parseable JSON — what actually produces the live binding
is the mirror attribute(s), not this internal tree. The new managerId/managerName rules were
nonetheless constructed as genuinely valid, parseable JSON (a small improvement over the pre-existing
convention, at no cost), and both mirror attributes were prioritized as the primary fix target,
consistent with this diagnosis.

### Not touched (per task scope)

Rules #2, #3, #4, #5, #7, #8 — all confirmed working live by Sentinel — were left exactly as-is.
`approvalInformationPanel`'s Rule #5 (the cross-fragment `fd:visible` panel show/hide) was used only as
a reference pattern for understanding the fragment-boundary mechanics; its own AST/attributes are
unmodified.
