# Integration & Unit Test Report — Vehicle Registration Form

| | |
|---|---|
| Run | `2026-08-12-vehicle-registration-form` |
| Date | 2026-08-13 |
| Form | `/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form` |
| Merged commit | `e570de76f0436cda6e8fa9d1c7f387762c505a87` (on `origin/main` via merge `bfbd5ad`, PR #5) |

## Where each check actually ran — read this first

| Check | Environment | Why |
|---|---|---|
| JUnit 5 / AEM Mocks unit suite (`create-form-tests`) | **LOCAL** — `mvn test -pl core` on the merged source | AEM Mocks needs no running instance. This is the **one** check that legitimately does not run on cloud DEV, and it is labelled as local everywhere it is reported. |
| Author-tier HTTP surface, model JSON, rule ASTs, clientlib delivery, theme delivery | **cloud DEV author** (`p185256` / `e1945105`) | the deployed artifact of record |

**No local result is presented as a cloud DEV result anywhere in this report.**

---

## Part 1 — Unit / integration suite (executed LOCALLY, AEM Mocks)

Command: `mvn test -pl core` · **BUILD SUCCESS** · exit 0

| Test class | Tests run | Failures | Errors | Skipped | Time |
|---|---|---|---|---|---|
| `com.aem.forms.agents.core.filters.LoggingFilterTest` | 1 | 0 | 0 | 0 | 7.581 s |
| `com.aem.forms.agents.core.listeners.SimpleResourceListenerTest` | 1 | 0 | 0 | 0 | 0.002 s |
| `com.aem.forms.agents.core.models.HelloWorldModelTest` | 1 | 0 | 0 | 0 | 0.719 s |
| `com.aem.forms.agents.core.schedulers.SimpleScheduledTaskTest` | 1 | 0 | 0 | 0 | 0.093 s |
| **`com.aem.forms.agents.core.servlets.GeneratePDFServletTest`** | **13** | 0 | 0 | 0 | 1.951 s |
| `com.aem.forms.agents.core.servlets.SimpleServletTest` | 1 | 0 | 0 | 0 | 0.106 s |
| **`com.aem.forms.agents.forms.submit.CustomSubmitGeneratePDFActionTest`** | **4** | 0 | 0 | 0 | 0.638 s |
| **TOTAL** | **22** | **0** | **0** | **0** | — |

### Coverage (JaCoCo 0.8.12)

**Overall: 97.1% instruction · 95.8% line.** Gate is ≥ 80% on Forms service classes — **met**.

| Class | Instruction % | Line % | Forms service? |
|---|---|---|---|
| `forms.submit.CustomSubmitGeneratePDFAction` | **100.0** | **100.0** | **yes** — the form's submit action |
| `core.servlets.GeneratePDFServlet` | **96.4** | **94.5** | **yes** — PDF generation on submit |
| `core.filters.LoggingFilter` | 100.0 | 100.0 | no |
| `core.models.HelloWorldModel` | 100.0 | 100.0 | no |
| `core.schedulers.SimpleScheduledTask` | 100.0 | 100.0 | no |
| `core.servlets.SimpleServlet` | 100.0 | 100.0 | no |
| `core.listeners.SimpleResourceListener` | 100.0 | 100.0 | no |

Both Forms service classes clear 80% comfortably. These two classes are what back
**TC-08-003** (valid submit generates the PDF).

> Scope note: there is **no prefill service** and **no Sling Model specific to this form** in the
> codebase, so no unit tests exist for those — correctly, since the form declares `prefill: none`.
> There is likewise **no workflow class**, which is itself a finding (see Part 3, D9).

---

## Part 2 — Author-tier HTTP surface (executed on cloud DEV AUTHOR)

All requests carried the Adobe IMS `login-token` session cookie. **There is no `admin:admin` on
AEMaaCS**, and the user's IMS email/password is rejected as Basic auth (401, verified). The token was
never written into `runs/`.

| URL | Status | Bytes | Verdict |
|---|---|---|---|
| `…/us/en/test-adaptive-form.html?wcmmode=disabled` (the embedded page) | **200** | 64,343 | PASS |
| `…/vehicle-registration-form.html` (standalone, isolation cross-check) | **200** | 589,283 | PASS |
| `…/vehicle-registration-form/jcr:content/guideContainer.model.json` | **200** | 38,791 | PASS |
| `…/vehicle-registration-form/jcr:content/guideContainer.html` | **200** | 559,460 | PASS |
| `…/vehicle-registration-form.theme/_default/theme.css` | **200** | 15,074 | PASS |
| `…/vehicle-registration-form.theme/_default/theme.js` | **200** | 18 | PASS (intentionally near-empty — no custom theme JS) |
| `/etc.clientlibs/clientlibs/vehicle-registration-form-clientlib.…min.js` | **200** | 1,229 (decompressed) | PASS |
| `/etc.clientlibs/…/core-forms-components-runtime-all.…min.js` | **200** | — | PASS |

All 10 CSS/JS URLs the author page requests return 200. **Zero 404s on the author tier.**

> Measurement caveat worth recording: the clientlib is served **gzip-encoded**. A `curl` without
> `--compressed` returns 411 bytes of binary and a naive `grep` for function names finds nothing —
> which briefly looked like "the validators aren't deployed". Re-fetching with `--compressed` showed
> all five functions present. Always pass `--compressed` before concluding a clientlib is empty.

### `guideContainer.model.json` structural validity

- Parses as valid JSON. Top-level keys: `id`, `fieldType`, `title`, `action`, `properties`,
  `columnCount`, `columnClassNames`, `gridClassNames`, `events`, `lang`, `:itemsOrder`, `metadata`,
  `adaptiveform`, `:type`, `:items`, `allowedComponents`.
- **37 nodes**: 1 form + 6 panels + 3 plain-text + 25 fields + 2 buttons.
- Field types are all Core Components types: `text-input`, `date-input`, `radio-group`, `email`,
  `multiline-input`, `drop-down`, `number-input`, `checkbox-group`, `button`, `panel`, `plain-text`.
- Option sets are fully populated: State **37**, Vehicle Type **6**, Fuel Type **7**,
  Manufacturing Year **31**, Document Checklist **6**.
- `constraintMessages` present on every required field with authored copy.

**PASS** — structurally valid and complete.

### Declarative validation constraints (from the model)

| Field | Constraints |
|---|---|
| `fullName` | `maxLength: 100`, `required` |
| `mobileNumber` | `pattern: ^[0-9]{10}$`, `maxLength: 10`, `required`, `validateMobile10Digits` |
| `emailAddress` | `required: false`, `validateEmailWhenPresent` |
| `pinCode` | `number-input`, `required`, `validatePin6Digits` |
| `dateOfBirth` | `format: date`, `required`, `validateDateDDMMYYYY && validateDobNotFuture` |
| `policyValidTill` | `format: date`, `required`, `validateDateDDMMYYYY` |
| `declarationDate` | `format: date`, `required`, `validateDateDDMMYYYY` **only — no future-date guard** |
| all other required fields | `required` + authored `constraintMessages.required` |

### Rule ASTs — every `nodeName` is valid

Rule ASTs live in the JCR as `fd:validate` / `fd:click` properties on the field nodes. They are
**not** serialised into `guideContainer.model.json` (the runtime receives `validationExpression`
strings instead) — so an empty `"rules"` in the model JSON is expected and is **not** a defect.

Inspected on the deployed form node. **7 rule ASTs**:

| # | Field | Property | Root `nodeName` | `eventName` | `script` type | `isValid` / `enabled` |
|---|---|---|---|---|---|---|
| 1 | `dateOfBirth` | `fd:validate` | `VALIDATE_EXPRESSION` | `Validate` | **string** | true / true |
| 2 | `mobileNumber` | `fd:validate` | `VALIDATE_EXPRESSION` | `Validate` | **string** | true / true |
| 3 | `emailAddress` | `fd:validate` | `VALIDATE_EXPRESSION` | `Validate` | **string** | true / true |
| 4 | `pinCode` | `fd:validate` | `VALIDATE_EXPRESSION` | `Validate` | **string** | true / true |
| 5 | `policyValidTill` | `fd:validate` | `VALIDATE_EXPRESSION` | `Validate` | **string** | true / true |
| 6 | `declarationDate` | `fd:validate` | `VALIDATE_EXPRESSION` | `Validate` | **string** | true / true |
| 7 | `submitButton` | `fd:click` | `EVENT_SCRIPTS` | `Click` | **array** `["submitForm()"]` | true / true |

Node names encountered across all 7 ASTs — all legitimate members of the Rule Editor's
`ruleToEventMap` / grammar:

`ROOT`, `STATEMENT`, `VALIDATE_EXPRESSION`, `EVENT_SCRIPTS`, `EVENT_CONDITION`,
`EVENT_AND_COMPARISON`, `EVENT_AND_COMPARISON_OPERATOR`, `AFCOMPONENT`, `COMPONENT`, `Using`,
`Expression`, `Then`, `CONDITION`, `COMPARISON_EXPRESSION`, `FUNCTION_CALL`, `PRIMITIVE_EXPRESSION`,
`OPERATOR`, `EQUALS_TO`, `BOOLEAN_LITERAL`, `True`, `BLOCK_STATEMENTS`, `BLOCK_STATEMENT`,
`SUBMIT_FORM`, `is clicked`.

- **`VALUE_EXPRESSION` occurrences: 0** — the forbidden node name is absent. (`CALC_EXPRESSION` is
  also absent, correctly, because this form has no calculate rules.)
- `script` is a **string** for the 6 expression rules and an **array** for the click event rule —
  matching the required storage convention exactly.

**PASS** — no rule will show as `Unknown Field - null` in the Rule Editor; all 7 remain author-editable.

### Custom validation functions actually delivered and working

Fetched from the served clientlib on DEV author and additionally evaluated **in page context** via
`cy.window()`:

| Function | In served clientlib | Present in page scope | Live behaviour |
|---|---|---|---|
| `validateMobile10Digits` | yes | yes | — |
| `validatePin6Digits` | yes | yes | rejects `12` |
| `validateEmailWhenPresent` | yes | yes | rejects `not-an-email`, allows empty |
| `validateDateDDMMYYYY` | yes | yes | `('2030-01-15') → true`, `('garbage') → false` |
| `validateDobNotFuture` | yes | yes | `('15/01/2030') → false`, `('1990-05-14') → true` |

**5/5 PASS.**

### Submit-action and theme wiring on the deployed node

| Property | Value | Verdict |
|---|---|---|
| `actionType` | `aem-adaptive-forms-agents/fd/af/submitactions/Custom-Submit-GeneratePDF` | PASS — shared PDF submit action |
| `dorType` | `none` | **note** — Document of Record is OFF; the PDF comes from the custom action, not DoR |
| `themeRef` | `/apps/fd/af/themes/aem-adaptive-forms-agents-vehicle-registration` | PASS — a **dedicated** per-form theme, not a shared/generic one |
| `clientLibRef` | `…forms.vehicle-registration-form,…forms.generate-pdf` | PASS |
| `fd:version` | `2.1` | PASS |
| workflow wiring | **absent** — 1 `actionType` only, no `assign-task-to-admin` model anywhere in the repo | **FAIL** (D9) |

The dedicated theme + correct page background + correct 3-column grid together confirm this is **not**
the "wrong theme" failure mode (generic/shared theme overriding the base grid) — the form has its own
token-override theme and the `conf` context resolves it correctly.

---

## Part 3 — Defects raised by this report

| ID | Sev | Finding | Route |
|---|---|---|---|
| D6 | Major | `mobileNumber` and `emailAddress` show **generic** validation copy ("Please match the format requested." / "Specify the value in allowed format : email.") instead of the authored `constraintMessages.validationExpression` ("Enter a valid 10-digit mobile number." / "Enter a valid email address."). The built-in `pattern` / `email`-format constraint fires before the authored expression and wins the message slot. | formwright (`create-adaptive-form` / `create-form-rules`) |
| D8 | Major | `declarationDate` does not reject a future date. Only `validateDateDDMMYYYY` is wired; there is no `NotFuture` guard, confirmed both in the model and live (`15/01/2030` accepted silently while the same value on `dateOfBirth` was correctly rejected). | formwright (`create-form-rules`) |
| D9 | Major | The `assign-task-to-admin` approval workflow required by the PLAN (`submit: ["dor_pdf","workflow"]`, AC-08.4) **does not exist**: no workflow model in the repo, no "Invoke an AEM Workflow" wiring, single `actionType`. | groundsmith (`create-workflow`) |

Pre-existing publish-tier defects **D1** and **D2** are recorded in the main report; both URLs return
**200 on author with an authenticated session**, which isolates their cause to anonymous access /
publish delivery rather than missing artifacts.

## Verdict

| Track | Verdict |
|---|---|
| Unit / integration suite (local, AEM Mocks) | **PASS** — 22/22, 0 failures |
| Coverage on Forms service classes (local) | **PASS** — 100% and 96.4%, both ≥ 80% |
| Author-tier HTTP surface (cloud DEV) | **PASS** — all 200, zero 404 |
| `model.json` structural validity (cloud DEV) | **PASS** |
| Rule ASTs — valid `nodeName`s only (cloud DEV) | **PASS** — 7/7, zero `VALUE_EXPRESSION` |
| Custom validator delivery (cloud DEV) | **PASS** — 5/5 served and functioning |
| Theme / clientlib / submit-action wiring (cloud DEV) | **PASS** |
| Approval-workflow wiring (cloud DEV) | **FAIL** — D9 |
| Authored validation message fidelity (cloud DEV) | **FAIL** — D6 |
| `declarationDate` future-date guard (cloud DEV) | **FAIL** — D8 |

**Integration gate: FAIL** — on D6, D8 and D9. The build, deployment, delivery surface and rule
storage are all sound; the failures are three specific behaviour/wiring gaps, each routed above.
