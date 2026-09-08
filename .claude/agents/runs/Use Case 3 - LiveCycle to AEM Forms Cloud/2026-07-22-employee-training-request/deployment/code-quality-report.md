# Code Quality Report — Employee Training Request
## Path-B Migration (AEM Forms on J2EE / LiveCycle 6.4.22 → AEM as a Cloud Service, Core Components)

Run: `.claude/agents/runs/2026-07-22-employee-training-request/`
Agent: forgemaster (DEPLOY lead)
Reads: `implementation/formwright.md`, `implementation/groundsmith.md`, `assembly/assembler.md`, `.aem-forms-config.yaml`
Produces: this file (end-deliverable of the DEPLOY phase)

**Note on authorship:** the coordinator (`aem-forms-program-agent`) briefly wrote a stand-in version
of this report after concluding this agent had stalled, since no report existed while a background
build was still running. This agent had not stalled — it was actively diagnosing and fixing a
build-environment root cause (§1) and had a background build genuinely in flight, per the correct
"don't poll a background task" pattern. This is the completed, authoritative version, written by
forgemaster itself once all verification (including the mandatory deploy-integrity orphan sweep,
§7, which the stand-in version did not perform) was done. It supersedes the interim version.

---

## 1. Build verdict

**BUILD SUCCESS** (final, authoritative run)

- **Command:** `mvn clean install -PautoInstallSinglePackage`
- **AEM target:** `http://localhost:4502` (local author, confirmed reachable pre-flight via `status-productinfo.txt`, HTTP 200)
- **Total time:** 3 min 30 s
- **Finished at:** 2026-07-22T13:53:16+05:30

### Build history (this run required 3 attempts — root causes documented below)

| Attempt | Result | Cause |
|---|---|---|
| 1 | BUILD FAILURE | `core` module unit tests: 2 errors in `GeneratePDFServletTest` — `UnsupportedClassVersionError` (Mockito tried to mock a class compiled at Java 17 bytecode while the build JVM is Java 11) |
| 2 | BUILD SUCCESS (Maven), but **not deploy-clean** | Fixed the version pin, but the replacement version raised the `com.day.cq.wcm.api` import requirement to `[1.33,2)`, which the actually-running local AEM instance (exports `1.32.0`) could not satisfy — `aem-adaptive-forms-agents.core` bundle installed but stayed **Installed**, not **Active**. Caught by checking bundle state directly, not just trusting Maven's exit code. |
| 3 | **BUILD SUCCESS**, fully deploy-clean | Corrected to a version satisfying both constraints — see below |

### Root cause & fix (not caused by any authored form artifact)

Root `pom.xml`'s `<aem.sdk.api>` property was pinned to `2026.5.26353.20260528T211800Z-260500`. That specific SDK API snapshot has an anomaly: `org/apache/jackrabbit/api/security/user/User.class` inside it is compiled to bytecode major version 61 (Java 17), while the project's `maven-compiler-plugin` targets `<release>11</release>` and the build JVM is Java 11.0.31 — so Mockito's mock creation for the pre-existing, shared `GeneratePDFServletTest` (unrelated to this delivery — Employee Training Request uses the stock `aemworkflowsubmit` action, not the custom GeneratePDF submit action) threw `UnsupportedClassVersionError`.

Fix required finding an `aem.sdk.api` version already cached locally that satisfies **two independent constraints**:
1. Java-11-compatible bytecode for the affected class (ruled out `2026.5.26353` and `2026.5.25892`, both bytecode major 61).
2. A `com.day.cq.wcm.api` Import-Package version compatible with what the local AEM author instance actually exports (`1.32.0`, from bundle `com.day.cq.wcm.cq-wcm-api` v5.15.50) — this ruled out the newest cached version, `2026.7.27083` (declares `[1.33,2)`).

**Final pin: `2026.6.26908.20260625T154911Z-260600`** — verified via a scoped `mvn -pl core package` + manifest inspection (`com.day.cq.wcm.api;version="[1.32,2)"`, bytecode major 55) *before* committing to the full reactor rebuild. This is a build-configuration correction to the SDK API version pin only — no Formwright/Groundsmith/Assembler-authored artifact was touched, and it is a repo-wide fix that future deliveries also inherit.

---

## 2. Per-module results

| Module | Status |
|---|---|
| core | PASS (compiled, 112/112 unit tests passed, bundle deployed **Active**) |
| ui.apps.structure | PASS |
| ui.apps | PASS |
| ui.content | PASS (includes the repointed "Test Adaptive Form" page from Assembler, the new workflow design model from Groundsmith, the form/fragments/DoR page from Formwright) |
| ui.config | PASS |
| all | PASS (aggregate package built and installed to AEM) |
| ui.frontend | PASS |
| it.tests | PASS (no integration tests defined in this repo — build only) |
| dispatcher | PASS |
| ui.tests (Cypress) | PASS (npm install/lint skipped per this environment's frontend-plugin config; docker-context tar built) |

11/11 modules green.

---

## 3. Unit tests & coverage

- **Tests run:** 112
- **Passed:** 112
- **Failed:** 0
- **Errors:** 0
- **Skipped:** 0

No new Java unit tests were added by this delivery — Employee Training Request's integration uses the stock `aemworkflowsubmit` submit action (no custom `FormSubmitActionService`) and Groundsmith's reasoned "no prefill" decision (no custom `DataProvider`), so there is no new core service code to test (per groundsmith.md §2/§6).

### Coverage on Forms service classes (`com.aem.forms.agents.forms.*`)

Aggregated from `core/target/site/jacoco/jacoco.csv`:

| Package | Class | Line coverage |
|---|---|---|
| forms.prefill | ComplaintPrefillService | 29/36 (80.6%) |
| forms.prefill | ContactUsPrefillService | 40/48 (83.3%) |
| forms.prefill | WithdrawalPrefillService | 29/36 (80.6%) |
| forms.prefill | LifeInsurancePrefillService | 169/189 (89.4%) |
| forms.prefill | BusinessRegistrationPrefillService | 111/118 (94.1%) |
| forms.submit | HealthInsuranceFdmSubmitAction | 86/104 (82.7%) |
| forms.submit | LifeInsuranceNewPolicySubmitAction | 136/162 (84.0%) |
| forms.submit | CustomSubmitGeneratePDFAction | 12/12 (100%) |
| forms.submit | SubmitGeneratePDFAction | 0/12 (0%) — legacy/unused duplicate class, no test authored (pre-existing, unrelated to this delivery) |
| forms.submit | BusinessRegistrationSubmitAction | 103/127 (81.1%) |
| forms.workflow | UnderwritingAuditProcess | 0/63 (0%) — pre-existing OSGi workflow process class from a prior delivery (health/life insurance underwriting), no unit test authored |
| forms.workflow | DeriveRiskRoutingProcess | 0/27 (0%) — same as above |

**Aggregate: 715/934 lines = 76.6%** — below the ≥80% target, driven entirely by 3 pre-existing zero-coverage classes (`SubmitGeneratePDFAction`, `UnderwritingAuditProcess`, `DeriveRiskRoutingProcess`) from **prior deliveries**, none of which belong to Employee Training Request. This delivery added **no new core Java classes** to test. **Flagged as a pre-existing gap, not a regression introduced by this delivery; does not block the DEPLOY gate** (build succeeded, no test failures).

One benign, pre-existing observation: 11× `[Fatal Error] :1:1: Content is not allowed in prolog.` logged during `LifeInsuranceNewPolicySubmitActionTest` (an unrelated form's test) — a SAX-parser warning from a mock/empty stream that does not affect that suite's own result (`Tests run: 15, Failures: 0, Errors: 0`). Not introduced by, or blocking, this delivery.

---

## 4. Static analysis

- **Compiler warnings:** 1 — `GeneratePDFServlet.java` uses/overrides a deprecated API (pre-existing, unrelated to this delivery).
- **Lint/SpotBugs/Checkstyle:** none configured beyond the Maven enforcer (Java 11+, Maven 3.3.9+) and the Cloud Manager content-package validator, both of which passed cleanly (0 errors; OSGi feature-model analysis for all 4 aggregates completed with 0 errors, only pre-existing warnings unrelated to this delivery).

---

## 5. Deployment artifacts

| Artifact | Type | Version | Size | Installed |
|---|---|---|---|---|
| `aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip` | content-package (aggregate, deployment of record) | 1.0.0-SNAPSHOT | 25,405,698 bytes | ✅ (`Package installed in 1601ms`) |
| `aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar` | OSGi bundle | 1.0.0-SNAPSHOT | 102,452 bytes | ✅ (embedded in `all`; bundle state **Active**) |
| `aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | 22,705,804 bytes | ✅ (embedded in `all`) |
| `aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | 6,194 bytes | ✅ (embedded in `all`) |
| `aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | 3,264,805 bytes | ✅ (embedded in `all`) — carries the form, fragments, theme, DoR page, workflow design model, mail templates, repointed "Test Adaptive Form" page |
| `aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | 22,124 bytes | ✅ (embedded in `all`) |
| `aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip` | dispatcher config bundle | 1.0.0-SNAPSHOT | 20,609 bytes | N/A (dispatcher artifact, not installed to author) |
| `aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar` | test jar | 1.0.0-SNAPSHOT | 13,130 bytes | N/A (test tooling artifact) |

The `all` aggregate package is the single deployment of record, installed via `-PautoInstallSinglePackage` to `http://localhost:4502/crx/packmgr/service.jsp`.

---

## 6. Deploy confirmation

- **Package install log:** `Installing aem-adaptive-forms-agents.all (...) to http://localhost:4502/crx/packmgr/service.jsp` → `Package imported.` → `Package installed in 1601ms.`
- **Bundle state:** `aem-adaptive-forms-agents.core` = **Active** (verified via `/system/console/bundles.json`); full instance status: **"738 bundles in total - all 738 bundles active."**
- **Form live:** `GET /content/forms/af/aem-adaptive-form-agents/employee-training-request.html` → **HTTP 200**
- **"Test Adaptive Form" page live:** `GET /content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html` → **HTTP 200**, confirmed embedding `employee-training-request` — verified `formRef` on the live `adaptiveFormEmbed` node = `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request`; no `sports-event-registration` references remain anywhere on the page (see §7 for the full structural diff).

### Workflow model sync (design → runtime)

Per this project's established convention (same as `business-registration-approval` and `health-insurance-underwriting`), only the `/conf` design model is committed; the `/var` runtime model is generated, never hand-authored.

- POSTed to `/conf/global/settings/workflow/models/employee-training-request-approval/jcr:content.generate.json` (CSRF token + same-origin Referer, per `ModelGenerateServlet` requirements) — re-run after the final (attempt 3) deploy to bind it to the authoritative build.
- Response: `{"msg":"Model successfully generated.","modelPath":"/var/workflow/models/employee-training-request-approval"}`
- Verified: `GET /var/workflow/models/employee-training-request-approval.json` → HTTP 200, title "Employee Training Request — Approval".
- Form's `guideContainer/@workflowModel` = `/var/workflow/models/employee-training-request-approval` (confirmed live, matches Groundsmith's wiring).

---

## 7. Deploy integrity (orphan/stale-node sweep, step 3b)

Both re-authored artifacts diffed live-tree-vs-source (`mode="update"` filter roots do not delete removed/renamed nodes, so this check is mandatory — a page rendering twice or a stale duplicate node would otherwise pass silently):

| Path | Source children | Live children | Result |
|---|---|---|---|
| `/content/forms/af/aem-adaptive-form-agents/employee-training-request/jcr:content/guideContainer` | `formTitle, employeeDetailsFragment, trainingRequestDetailsPanel, justificationPanel, attachmentsPanel, declarationFragment, approvalInformationPanel, actionsPanel` (8) | Same 8, same names | **Clean — no orphans** |
| `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form/jcr:content/root/container` | Exactly one `adaptiveFormEmbed` node (Assembler repointed in-place, did not add a second container) | Exactly one `adaptiveFormEmbed`, `formRef` = new form's DAM guide-asset path; no `sports-event-registration` reference anywhere | **Clean — no orphans, no stacking** |

**No purge was necessary.** Instance tree matches source exactly for both re-authored artifacts.

---

## 8. Verdict & gate

**BUILD SUCCESS** + **confirmed deploy** (form + embedded page both HTTP 200, bundle Active) + **0 failed unit tests** (112/112 passed) + **instance matches source** (no orphan/duplicate nodes) + **workflow runtime model generated and live**.

### GATE RESULT: **PASS**

Flagged (non-blocking) items carried forward for Sentinel/downstream awareness:
1. Forms-service coverage is 76.6% (below 80% target), entirely attributable to 3 pre-existing zero-coverage classes from prior deliveries — not a regression from this delivery.
2. Two pre-production follow-ups already documented by Groundsmith: real manager-lookup, real distinct finance-approver (both confirmed business decisions, not build gaps).
3. Known parity gap: approval email does not attach the DoR PDF directly (documented deviation, see groundsmith.md).
4. Local SDK has no SMTP configured — Send Email workflow steps may no-op on localhost (expected, not a defect; Sentinel's TC-032 already accounts for this).
5. `pom.xml`'s `<aem.sdk.api>` property was corrected from `2026.5.26353.20260528T211800Z-260500` to `2026.6.26908.20260625T154911Z-260600` as part of this deploy — a project-wide build-configuration fix (not scoped to this delivery's authored artifacts) that future deliveries will inherit.

---

## Run metrics

- `time_taken_minutes`: 60.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~10,000
  - `read`: ~36,000 (implementation/assembly/groundsmith summaries, workflow model .content.xml, pom.xml, jacoco.csv, manifest inspections across 5 aem-sdk-api versions, 3 build logs)
  - `write`: ~7,000 (pom.xml edits x2, this report)
  - `other`: ~30,000 (3 full/partial mvn build runs' tool-call overhead, curl/bundle/JSON diagnostics, background task monitoring, peer coordination)
  - `total`: ~83,000

---

## Handoff YAML

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
time_taken_minutes: 60.0
tokens_consumed:
  cli_text: 10000
  read: 36000
  write: 7000
  other: 30000
  total: 83000
build: BUILD SUCCESS
command: "mvn clean install -PautoInstallSinglePackage"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps: PASS, ui.apps.structure: PASS, ui.content: PASS, ui.config: PASS, all: PASS, ui.frontend: PASS, it.tests: PASS, dispatcher: PASS, ui.tests: PASS }
unit_tests: { run: 112, passed: 112, failed: 0, coverage_pct: 76.6 }
static_analysis: { warnings: 1, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip", type: dispatcher-config, installed: false }
  - { name: "aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar", type: test-jar, installed: false }
deploy_confirmed: true
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true }
deploy_integrity: { instance_matches_source: true, orphans_purged: [] }
workflow_model_sync: { design_path: "/conf/global/settings/workflow/models/employee-training-request-approval", runtime_path: "/var/workflow/models/employee-training-request-approval", generated: true, verified_live: true }
build_config_fix: { property: "aem.sdk.api", old_value: "2026.5.26353.20260528T211800Z-260500", new_value: "2026.6.26908.20260625T154911Z-260600", reason: "Java-11 bytecode incompatibility in old pin's User.class caused Mockito test failures; the next-newest cached version broke bundle resolution against the locally running AEM instance's com.day.cq.wcm.api 1.32.0; this version satisfies both constraints." }
report: ".claude/agents/runs/2026-07-22-employee-training-request/deployment/code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (testing)
```

---

## 9. Redeploy — fix pass 1 (gate-failure #1 remediation)

**Trigger:** Sentinel's first TEST-gate run failed (gate-failure #1). Formwright and Groundsmith each
authored fixes for their flagged defects (no re-authoring performed by this agent — see the 4 changed
files listed below, all pre-verified in source before build). This section documents the rebuild +
redeploy of record for the fix-pass cycle.

### 9.1 Pre-flight

- AEM author reachable: `GET /system/console/bundles.json` → HTTP 200.
- Toolchain: Maven 3.9.16, build JVM **Java 11.0.31** (matches enforcer requirement; a stray `java -version`
  on PATH resolves to 21, but Maven's own `mvn -version` output confirms it runs on 11.0.31 — no action needed).
- Confirmed all 4 authored fixes present in source **before** building (no re-authoring by this agent):
  1. `employee-training-request/.content.xml` — `financeComments` carries
     `enabled="currentStatus.$value == \"ManagerApproved\""` + matching nested `fd:rules` `enabled` attribute;
     `attachmentsFolderPath="attachments"` (no trailing slash).
  2. `employee-identity/.content.xml` — `managerId`/`managerName` both carry
     `enabled="Number(employeeId.$value) <= 0 || Number(employeeId.$value) > 10"` authored directly (sibling
     scope, no cross-fragment reference) with matching nested `fd:rules` attributes; new `sectionTitle` AF
     Title (h2) component added; fragment container `hideTitle="{Boolean}true"`.
  3. `employee-training-request-clientlib/css/form.css` — `.etr-section-title` rule present.
  4. Workflow model `.content.xml` — both `AssignTaskStep` nodes' `inputAttachmentsPath`/`outputAttachmentsPath`
     changed from `attachments/` to `attachments` (no trailing slash).
- Did **not** touch the OR-split branching gap (`orsplit_manager`/`orsplit_finance`) per explicit instruction —
  that remains a recorded, human-actionable follow-up requiring Workflow Editor UI access.

### 9.2 Build & deploy (redeploy of record)

- **Command:** `mvn clean install -PautoInstallSinglePackage -q -Daem.host=localhost -Daem.port=4502`
- **Result:** **BUILD SUCCESS** on the **first attempt** (exit code 0) — no root-cause fixing needed this
  cycle; the `aem.sdk.api` pin from the prior deploy (`2026.6.26908.20260625T154911Z-260600`) remains
  correct and required no further change.
- Only console output under `-q`: 11× benign `[Fatal Error] :1:1: Content is not allowed in prolog.` —
  the same pre-existing, non-blocking SAX-parser warning from `LifeInsuranceNewPolicySubmitActionTest`
  (unrelated form, documented in §3 above). No new warnings, no failures.
- **Deploy landed:** package-manager listing confirms `lastUnpacked` advanced to `2026-07-22 15:18:02`
  IST for `all`, `ui.apps`, `ui.config`, and `ui.content` (all timestamped to this build run — not a
  stale prior install). `aem-adaptive-forms-agents.core` bundle `Last Modification: Wed Jul 22 15:18:12
  IST 2026`, state **Active**; instance-wide `"738 bundles in total - all 738 bundles active."`

### 9.3 Live verification of the 4 fixes

| # | Fix | Live verification | Result |
|---|---|---|---|
| 1 | `managerId`/`managerName` reactive lock | `GET .../employee-identity/jcr:content/guideContainer.model.json` → both fields show live `"rules":{"enabled":"Number(employeeId.$value) <= 0 \|\| Number(employeeId.$value) > 10"}` | **PASS** |
| 2 | `financeComments` reactive lock | `GET .../employee-training-request/jcr:content/guideContainer.model.json` → `"rules":{"enabled":"currentStatus.$value == \"ManagerApproved\""}` live on `financeComments` | **PASS** |
| 3 | `employee-identity` section heading rendered | `GET .../employee-identity/jcr:content/guideContainer/sectionTitle.json` → node live with `fd:htmlelementType="h2"`, `css="etr-section-title"`, `value="Employee Details"`, `sling:resourceType=".../title"` (component JSON confirms the new AF Title node deployed; title/plain-text components — same as the host form's own `formTitle` — don't surface as data items in `model.json`, which is expected, not a defect) | **PASS** |
| 4 | No-trailing-slash `attachmentsFolderPath` (502 avoidance) | `GET .../employee-training-request/jcr:content/guideContainer.json` → `attachmentsFolderPath="attachments"` (dry structural check, no real submission attempted, per instruction); workflow model `.content.xml` regenerated (`/conf/.../jcr:content.generate.json` → `"Model successfully generated."`) and `GET /var/workflow/models/employee-training-request-approval.json` confirms both `AssignTaskStep`s now show `inputAttachmentsPath=attachments` / `outputAttachmentsPath=attachments`, no trailing slash | **PASS** |

### 9.4 Deploy integrity — orphan found and purged (step 3b)

Diffed live tree vs. source for both re-authored form artifacts:

| Path | Source children | Live children (before sweep) | Result |
|---|---|---|---|
| `.../employee-training-request/.../guideContainer` (top-level, 8 nodes) | `formTitle, employeeDetailsFragment, trainingRequestDetailsPanel, justificationPanel, attachmentsPanel, declarationFragment, approvalInformationPanel, actionsPanel` | Same 8, same names | Clean |
| `.../employee-training-request/.../guideContainer/employeeDetailsFragment` (wrapper — fix pass 1 removed its `fd:rules`/`fd:events` from source since the lock logic moved into the fragment itself) | No `fd:rules`/`fd:events` children in source `.content.xml` | **`fd:rules` (still containing the OLD broken cross-fragment `fd:change` script) and `fd:events` were still live** — a stale `mode="update"` leftover from before the fix pass | **ORPHAN FOUND** |
| `.../employee-identity/.../guideContainer` (9 nodes incl. new `sectionTitle`) | `sectionTitle, employeeId, employeeName, email, department, managerId, managerName, location, businessUnit` | Same 9, same names | Clean |
| `.../test-adaptive-form/.../root/container/container` | Exactly one `adaptiveFormEmbed` | Exactly one `adaptiveFormEmbed`, `formRef` = employee-training-request's DAM guide-asset path | Clean — no stacking |

**Purge performed:** the two orphaned nodes under `employeeDetailsFragment` were deleted via Sling POST
`:operation=delete` (CSRF token + matching Referer/Origin, both returned HTTP 200 with `ChangeLog`
confirming `deleted(...)`):
- `.../employee-training-request/jcr:content/guideContainer/employeeDetailsFragment/fd:rules`
- `.../employee-training-request/jcr:content/guideContainer/employeeDetailsFragment/fd:events`

**Re-verified:** `employeeDetailsFragment` live child/property set now has no `fd:rules`/`fd:events` keys
(matches source exactly); the form page (`.../employee-training-request.html`) and the "Test Adaptive
Form" page both still return **HTTP 200** after the purge. This was a deploy-integrity reconciliation
only — no source or form design was edited; the source was already correct, the stale instance node was
removed.

### 9.5 Verdict & gate — redeploy

**BUILD SUCCESS** (first attempt) + **confirmed deploy** (packages `lastUnpacked` advanced this run,
core bundle Active, form + embedded "Test Adaptive Form" page both HTTP 200) + **0 failed unit tests**
(same 112/112 suite, unaffected by this content-only fix pass) + **all 4 fixes verified live** + **instance
matches source after purge of 1 orphan pair** (`employeeDetailsFragment/fd:rules` + `fd:events`).

### GATE RESULT: **PASS**

Carried-forward flags (unchanged from §8, still non-blocking): forms-service coverage 76.6% (pre-existing
gap, no new core classes this cycle), the two documented pre-production follow-ups, the DoR-attachment
email deviation, no local SMTP. **New, recorded follow-up:** the OR-split branching gap
(`orsplit_manager`/`orsplit_finance`) remains unresolved — root-caused by Groundsmith as requiring human
Workflow Editor UI access; explicitly out of scope for this agent and not attempted.

### Run metrics — redeploy (fix pass 1)

- `time_taken_minutes`: 22.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~4,500
  - `read`: ~14,000 (prior report, 2 form `.content.xml` files, workflow model grep, model.json/node.json fetches)
  - `write`: ~2,500 (this report section)
  - `other`: ~9,000 (mvn build tool-call overhead, curl/bundle/package-manager checks, CSRF/orphan-purge calls, workflow model regenerate)
  - `total`: ~30,000

### Handoff YAML — redeploy (fix pass 1)

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
time_taken_minutes: 22.0
tokens_consumed:
  cli_text: 4500
  read: 14000
  write: 2500
  other: 9000
  total: 30000
build: BUILD SUCCESS
command: "mvn clean install -PautoInstallSinglePackage -q -Daem.host=localhost -Daem.port=4502"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps: PASS, ui.apps.structure: PASS, ui.content: PASS, ui.config: PASS, all: PASS, ui.frontend: PASS, it.tests: PASS, dispatcher: PASS, ui.tests: PASS }
unit_tests: { run: 112, passed: 112, failed: 0, coverage_pct: 76.6 }
static_analysis: { warnings: 1, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip", type: dispatcher-config, installed: false }
  - { name: "aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar", type: test-jar, installed: false }
deploy_confirmed: true
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true }
fixes_verified_live:
  - { fix: "managerId/managerName reactive lock", verified: true }
  - { fix: "financeComments reactive lock", verified: true }
  - { fix: "employee-identity section heading (sectionTitle AF Title component)", verified: true }
  - { fix: "attachmentsFolderPath no trailing slash (form + workflow AssignTaskStep)", verified: true }
deploy_integrity: { instance_matches_source: true, orphans_purged: ["employee-training-request/jcr:content/guideContainer/employeeDetailsFragment/fd:rules", "employee-training-request/jcr:content/guideContainer/employeeDetailsFragment/fd:events"] }
workflow_model_sync: { design_path: "/conf/global/settings/workflow/models/employee-training-request-approval", runtime_path: "/var/workflow/models/employee-training-request-approval", generated: true, verified_live: true }
known_unfixed: { item: "OR-split branching (orsplit_manager/orsplit_finance) real Approve/Reject routing conditions", owner: "human via Workflow Editor UI", attempted_by_forgemaster: false }
report: ".claude/agents/runs/2026-07-22-employee-training-request/deployment/code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (retest)
```

---

## Redeploy — fix pass 1 (after TEST gate failure #1)

**Note on authorship:** the coordinator (`aem-forms-program-agent`) is appending this section directly.
The delegated `forgemaster` redeploy agent kicked off the rebuild, pre-flight-verified all 4 source
fixes were present, and reported it would confirm live once the build finished — but its transcript
went quiet before writing this update, mirroring the first deploy's stall pattern. The coordinator
independently re-verified the redeploy directly against the live author rather than wait further.

Sentinel's first TEST pass (pre-fix) found 3 form-side defects (formwright) and 2 integration-side
defects (groundsmith). All 5 were fixed in source; this redeploy carries those fixes to the running
instance. Full defect detail: `testing/integration-test-report.md`, `testing/test-form-ui-report.md`.

### Live re-verification performed by the coordinator (admin:admin against http://localhost:4502)

| Check | Endpoint | Result |
|---|---|---|
| FinanceComments reactive lock now live | `GET .../employee-training-request/jcr:content/guideContainer.model.json` | `enabled` rule referencing `ManagerApproved` **present** |
| ManagerId/ManagerName reactive lock now live | same endpoint | `managerId` + an `enabled` rule **present** |
| Submit-attachment trailing-slash fix deployed | `GET .../guideContainer.json` → `attachmentsFolderPath` | `"attachments"` — **no trailing slash** (was `"attachments/"`, the confirmed root cause of the prior HTTP 502) |
| Workflow model re-synced after fix | `GET /var/workflow/models/employee-training-request-approval.json` | `lastSynced` advanced to `2026-07-22T15:08:42+05:30` (post-fix), model still resolves 200 |
| Employee-identity fragment heading fix | attempted via `.../employee-identity/jcr:content/fragmentcontainer.model.json` | **404 on this guessed path** — inconclusive by this method; not confirmed working or broken this way. Defer to Sentinel's structural/live-render check, which is the authoritative method for this specific fix. |

### Gate status for this redeploy

**PASS on build/deploy mechanics** — the fixed source is confirmably live for 3 of 4 targeted areas
via direct JCR/model inspection; the 4th (fragment heading) needs Sentinel's rendering-level check
rather than a raw JSON endpoint guess. Not re-declaring full DEPLOY-phase PASS/FAIL here since the
substantive functional verdict (does the fix actually resolve each of Sentinel's 5 findings) belongs
to Sentinel's retest, not to a second deploy-mechanics check alone.

```yaml
agent: aem-forms-program-agent (standing in for forgemaster, redeploy verification)
phase: DEPLOY (redeploy — fix pass 1)
status: PASSED
fixes_confirmed_live: [financeComments_enabled_rule, managerId_enabled_rule, attachmentsFolderPath_no_trailing_slash, workflow_model_resynced]
fixes_unconfirmed_by_this_method: [employee_identity_heading_render]
not_attempted: [or_split_branching_fix — known not fixed, documented human follow-up, not part of this redeploy's scope]
next: aem-forms-program-agent runs sentinel (TEST retest — fix-pass-1 verification)
```

---

## 10. Redeploy — fix pass 2 (post-Sentinel gate-failure #2 remediation)

**Trigger:** Sentinel's retest of the fix-pass-1 scope found one NEW Critical defect (D3 — both
reusable fragments' field-level `dataRef` bindings resolved to the wrong schema paths, discovered
from a real browser-driven submission's `data.xml`) and corrected one mis-verified test (TC-019 —
the submit button's `SubmissionDate` set-value rule threw a live `ParserError` on every click because
the original AST used the unsupported `new Date().toISOString()` expression; TC-019 had previously
only been checked for the rule's *presence* in `guideContainer.model.json`, never actually exercised).
Formwright authored both fixes in source (no re-authoring performed by this agent). Full defect
detail: `implementation/formwright.md` §"Fix pass 2".

### 10.1 Pre-flight

- AEM author reachable: `GET /system/console/bundles.json` → HTTP 200.
- Toolchain: Maven 3.9.16, build JVM confirmed **Java 11.0.31** via `mvn -version` (a stray `java`
  on PATH resolves to 21 — expected, does not affect the Maven build JVM, same as both prior deploys).
- Confirmed both authored fixes present in source **before** building (no re-authoring by this agent):
  1. `employee-identity/.content.xml` — all 8 fields' `dataRef` corrected from `$.employee.*` to
     `$.EmployeeDetails.*`.
  2. `declaration-consent/.content.xml` — both fields' `dataRef` corrected from `$.declaration.*` to
     `$.Declaration.*`.
  3. `employee-training-request/.content.xml` — submit button's `fd:rules/fd:click` AST and
     `fd:events/click` both rewired to call `getCurrentDateISOString()` instead of the broken
     `new Date()` expression.
  4. `employee-training-request-clientlib/js/functions.js` — `getCurrentDateISOString()` function
     present.
- Did **not** touch the OR-split branching gap (`orsplit_manager`/`orsplit_finance`) — remains the
  documented, human-actionable follow-up requiring Workflow Editor UI access.

### 10.2 Build & deploy (redeploy of record)

- **Command:** `mvn clean install -PautoInstallSinglePackage -q -Daem.host=localhost -Daem.port=4502`
- **Result:** **BUILD SUCCESS** on the **first attempt** — no root-cause fixing needed this cycle;
  the `aem.sdk.api` pin from the original deploy (`2026.6.26908.20260625T154911Z-260600`) remains
  correct and required no further change.
- Only console output under `-q`: the same 11× benign `[Fatal Error] :1:1: Content is not allowed
  in prolog.` SAX-parser warning from `LifeInsuranceNewPolicySubmitActionTest` (unrelated form,
  documented in §3). No new warnings, no failures.
- **Deploy landed (package `lastUnpacked` advanced — the reliable "installed" signal, not just exit
  code):** `/etc/packages/com.aem.forms.agents/aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip`
  `lastUnpacked` = `2026-07-22T17:50:53+05:30`, timestamped to this build run (checked against
  wall-clock time immediately after, 17:52:51 — well within the build window, not a stale prior
  install).
- **Bundle state:** `aem-adaptive-forms-agents.core` v1.0.0.SNAPSHOT = **Active**. Instance-wide:
  `"738 bundles in total - all 738 bundles active."`
- **Unit tests:** aggregated from `core/target/surefire-reports/*.txt` — **112 run, 0 failures,
  0 errors, 0 skipped** (same suite as prior deploys; content-only fix pass, no new/changed Java).

### 10.3 Live verification of the 2 fixes

| # | Fix | Live verification | Result |
|---|---|---|---|
| 1 | D3 — fragment `dataRef` bindings | `GET .../employee-identity/jcr:content/guideContainer.model.json` → all 8 fields show `"dataRef":"$.EmployeeDetails.*"` (EmployeeId, EmployeeName, Email, Department, ManagerId, ManagerName, Location, BusinessUnit). `GET .../declaration-consent/jcr:content/guideContainer.model.json` → both fields show `"dataRef":"$.Declaration.AcceptedTerms"` / `"$.Declaration.SubmissionDate"`. No `$.employee.*`/`$.declaration.*` traces remain. | **PASS** |
| 2 | TC-019 — SubmissionDate ParserError fix | `GET .../employee-training-request/jcr:content/guideContainer/actionsPanel/submitButton.1.json` → `fd:events.click` = `["declarationFragment.submissionDate.$value = getCurrentDateISOString()", " submitForm()"]`; `fd:rules.fd:click` AST's `FUNCTION_CALL` node references `functionName.id":"getCurrentDateISOString"` with empty `params:[]`. No `new Date()` reference remains anywhere in the live rule. Also confirmed `getCurrentDateISOString()` is defined and live at `GET /apps/clientlibs/employee-training-request-clientlib/js/functions.js` → HTTP 200. | **PASS** |

Both fixes verified at the actual rule/binding level (not just presence-in-JSON, learning directly
from the TC-019 mis-verification this fix pass itself was correcting).

### 10.4 Deploy integrity — orphan/stale-node sweep (step 3b)

Diffed live tree vs. source for every path touched by this fix pass, plus the previously-known-risky
`employeeDetailsFragment` wrapper (site of the fix-pass-1 orphan):

| Path | Source children/properties | Live | Result |
|---|---|---|---|
| `.../employee-training-request/.../guideContainer` (top-level, 8 nodes) | `formTitle, employeeDetailsFragment, trainingRequestDetailsPanel, justificationPanel, attachmentsPanel, declarationFragment, approvalInformationPanel, actionsPanel` | Same 8, same names, same order | **Clean** |
| `.../guideContainer/employeeDetailsFragment` wrapper | `dataRef="$.EmployeeDetails"`, no `fd:rules`/`fd:events` children (purged in fix pass 1) | Same — no stray `fd:rules`/`fd:events` reappeared | **Clean — prior purge held** |
| `.../employee-identity/.../guideContainer` (9 nodes) | `sectionTitle, employeeId, employeeName, email, department, managerId, managerName, location, businessUnit` | Same 9, same names | **Clean** |
| `.../declaration-consent/.../guideContainer` (2 nodes) | `acceptedTerms, submissionDate` | Same 2, same names | **Clean** |
| `.../submitButton` node | `fd:rules`, `fd:events` (both fixed content) | Both present, content matches source's fixed AST exactly (no duplicate/old `fd:rules` sibling) | **Clean** |
| `.../test-adaptive-form/.../root/container/container` | Exactly one `adaptiveFormEmbed` | Exactly one `adaptiveFormEmbed`, `formRef` = `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request` | **Clean — no stacking** |

**No purge was necessary.** Instance tree matches source exactly for all re-authored artifacts in
this fix pass; the fix-pass-1 orphan purge (`employeeDetailsFragment/fd:rules`+`fd:events`) held and
did not reappear (update-mode redeploys don't resurrect nodes absent from source).

### 10.5 Verdict & gate — redeploy (fix pass 2)

**BUILD SUCCESS** (first attempt) + **confirmed deploy** (package `lastUnpacked` advanced this run,
core bundle Active, form + embedded "Test Adaptive Form" page both HTTP 200) + **0 failed unit
tests** (112/112, unaffected by this content-only fix pass) + **both fixes verified live at the
rule/binding level** + **instance matches source, zero orphans** (including re-confirming the
fix-pass-1 purge held).

### GATE RESULT: **PASS**

Carried-forward flags (unchanged, still non-blocking): forms-service coverage 76.6% (pre-existing
gap, no new core Java classes this cycle either), the two documented pre-production follow-ups
(manager lookup, finance-approver identity), the DoR-attachment email deviation, no local SMTP.
OR-split branching gap remains the recorded, human-actionable follow-up — untouched by this agent,
per explicit instruction.

### Run metrics — redeploy (fix pass 2)

- `time_taken_minutes`: 14.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~3,200
  - `read`: ~11,000 (prior report sections, formwright.md fix-pass-2 section, 3 `.content.xml` files, functions.js, prior deploy-integrity tables for comparison)
  - `write`: ~2,800 (this report section)
  - `other`: ~7,500 (1 mvn build's tool-call overhead, curl/bundle/package-manager/JCR node checks, surefire aggregation)
  - `total`: ~24,500

### Handoff YAML — redeploy (fix pass 2)

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
time_taken_minutes: 14.0
tokens_consumed:
  cli_text: 3200
  read: 11000
  write: 2800
  other: 7500
  total: 24500
build: BUILD SUCCESS
command: "mvn clean install -PautoInstallSinglePackage -q -Daem.host=localhost -Daem.port=4502"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps: PASS, ui.apps.structure: PASS, ui.content: PASS, ui.config: PASS, all: PASS, ui.frontend: PASS, it.tests: PASS, dispatcher: PASS, ui.tests: PASS }
unit_tests: { run: 112, passed: 112, failed: 0, coverage_pct: 76.6 }
static_analysis: { warnings: 1, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip", type: dispatcher-config, installed: false }
  - { name: "aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar", type: test-jar, installed: false }
deploy_confirmed: true
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true }
fixes_verified_live:
  - { fix: "D3 - employee-identity fragment dataRef -> $.EmployeeDetails.*", verified: true }
  - { fix: "D3 - declaration-consent fragment dataRef -> $.Declaration.*", verified: true }
  - { fix: "TC-019 - submitButton SubmissionDate rule now calls getCurrentDateISOString() (no ParserError)", verified: true }
deploy_integrity: { instance_matches_source: true, orphans_purged: [] }
workflow_model_sync: { design_path: "/conf/global/settings/workflow/models/employee-training-request-approval", runtime_path: "/var/workflow/models/employee-training-request-approval", generated: true, verified_live: true, note: "unchanged this cycle - fix pass 2 touched no workflow artifacts" }
known_unfixed: { item: "OR-split branching (orsplit_manager/orsplit_finance) real Approve/Reject routing conditions", owner: "human via Workflow Editor UI", attempted_by_forgemaster: false }
report: ".claude/agents/runs/2026-07-22-employee-training-request/deployment/code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (final retest)
```

## 11. Redeploy — fix pass 3 (post-Sentinel gate-failure #3, defect D4)

**Trigger:** Sentinel's retest of the fix-pass-2 scope confirmed D3 and TC-019 both fully resolved
(across 2 full spec runs), but found the SubmissionDate *assignment itself* still never took effect —
confirmed via 3 independent live methods (live DOM value stays empty, the outgoing network request
body carries no `SubmissionDate` key, the persisted JCR payload has no `Declaration.SubmissionDate`
value), with a control check (`CurrentStatus` DOES appear in the payload) ruling out "readOnly fields
get dropped." Formwright root-caused this as D4 — a narrower defect than TC-019: the function call
now runs cleanly with no error, but the assignment has no effect — and fixed it in source (no
re-authoring performed by this agent). Full defect detail: `implementation/formwright.md` §"Fix pass 3".

### 11.1 Pre-flight

- AEM author reachable: `GET /system/console/bundles.json` → HTTP 200.
- Toolchain: Maven 3.9.16, build JVM confirmed **Java 11.0.31** via `mvn -version` (same stray `java`
  21 on PATH as prior deploys — does not affect the Maven build JVM).
- Confirmed the authored fix present in source **before** building (no re-authoring by this agent):
  `employee-training-request/.content.xml` — submit button's `fd:click` JSON metadata `script` array
  and `fd:events`/`click` attribute both now read
  `$form.declarationFragment.submissionDate.$value = getCurrentDateISOString()` (grep-confirmed the
  bare/unscoped `declarationFragment.submissionDate.$value` form is gone — 0 occurrences as a click
  target). The AFCOMPONENT AST node's `id` was already `$form.declarationFragment.submissionDate`
  and required no change.
- Did **not** touch the OR-split branching gap (`orsplit_manager`/`orsplit_finance`) — remains the
  documented, human-actionable follow-up requiring Workflow Editor UI access.

### 11.2 Build & deploy (redeploy of record)

- **Command:** `mvn clean install -PautoInstallSinglePackage -q -Daem.host=localhost -Daem.port=4502`
- **Result:** **BUILD SUCCESS** on the **first attempt** (exit code 0) — no root-cause fixing needed
  this cycle.
- Only console output under `-q`: the same 11× benign `[Fatal Error] :1:1: Content is not allowed in
  prolog.` SAX-parser warning from `LifeInsuranceNewPolicySubmitActionTest` (unrelated form,
  documented in §3). No new warnings, no failures.
- **Deploy landed (package `lastUnpacked` advanced — the reliable "installed" signal, not just exit
  code):** `all` = `2026-07-22T13:04:50.717Z`, `ui.apps` = `2026-07-22T13:04:56.926Z`, `ui.config` =
  `2026-07-22T13:04:56.956Z`, `ui.content` = `2026-07-22T13:05:07.071Z` — all timestamped to this
  build run (checked against wall-clock `2026-07-22T13:11:02Z` immediately after, well within the
  build window, not a stale prior install).
- **Bundle state:** `aem-adaptive-forms-agents.core` v1.0.0.SNAPSHOT = **Active**. Instance-wide bundle
  summary `[738 total, 728 active, 10 fragments, 0 resolved, 0 installed]` — no non-fragment bundle
  outside Active.
- **Unit tests:** aggregated from `core/target/surefire-reports/*.txt` — **112 run, 0 failures,
  0 errors, 0 skipped** (same suite as all 3 prior deploys; content-only fix pass, no new/changed
  Java).

### 11.3 Live verification of the D4 fix

| # | Fix | Live verification | Result |
|---|---|---|---|
| 1 | D4 — SubmissionDate set-value target missing `$form.` scope prefix | `GET .../employee-training-request/jcr:content/guideContainer/actionsPanel/submitButton.1.json` → `fd:click` JSON metadata `script` array now reads `"$form.declarationFragment.submissionDate.$value = getCurrentDateISOString()"` then `"submitForm()"`; `fd:events.click` = `["$form.declarationFragment.submissionDate.$value = getCurrentDateISOString()", " submitForm()"]`; the AFCOMPONENT AST's `SET_VALUE` target node confirmed unchanged at `"id":"$form.declarationFragment.submissionDate"`. Zero bare/unscoped `declarationFragment.submissionDate.$value` occurrences remain as a click target. | **PASS** |

This is a rule-content fix, not a Java/service change, so live verification was done at the actual
rule/mirror level (the same JSON endpoint the Rule Editor runtime parses), matching the depth Sentinel
used to find D4 in the first place. The genuinely conclusive check — does the field actually populate
in the live DOM / outgoing request / persisted JCR payload on a real click-through submission — is
Sentinel's to run at retest; this agent confirmed the fixed rule text is live on the instance.

### 11.4 Deploy integrity — orphan/stale-node sweep (step 3b)

Diffed live tree vs. source for every path touched by this fix pass, plus the previously-known-risky
`employeeDetailsFragment` wrapper (site of the fix-pass-1 orphan) and the "Test Adaptive Form" embed:

| Path | Source children/properties | Live | Result |
|---|---|---|---|
| `.../employee-training-request/.../guideContainer` (top-level, 8 nodes) | `formTitle, employeeDetailsFragment, trainingRequestDetailsPanel, justificationPanel, attachmentsPanel, declarationFragment, approvalInformationPanel, actionsPanel` | Same 8, same names, same order | **Clean** |
| `.../guideContainer/employeeDetailsFragment` wrapper | No `fd:rules`/`fd:events` children (purged fix pass 1) | Same — no stray `fd:rules`/`fd:events` reappeared | **Clean — prior purge held** |
| `.../guideContainer/declarationFragment` | `cq:responsive` only (fragment reference container) | Same — `cq:responsive` only | **Clean** |
| `.../guideContainer/actionsPanel` | `cq:responsive, submitButton, resetButton` | Same 3, no extra/duplicate button nodes | **Clean** |
| `.../actionsPanel/submitButton` node | `fd:rules`, `fd:events` (both carrying the D4-fixed mirrors) | Both present, content matches source's fixed mirrors exactly (no duplicate/old `fd:rules` sibling, no residual bare-reference copy) | **Clean** |
| `.../test-adaptive-form/.../root/container/container` | Exactly one `adaptiveFormEmbed` | Exactly one `adaptiveFormEmbed`, `formRef` = `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request` | **Clean — no stacking** |

**No purge was necessary.** Instance tree matches source exactly for every path touched by this fix
pass; the fix-pass-1 orphan purge (`employeeDetailsFragment/fd:rules`+`fd:events`) continues to hold
and did not reappear (update-mode redeploys don't resurrect nodes absent from source). Both the form
page and the embedded "Test Adaptive Form" page return **HTTP 200**.

### 11.5 Verdict & gate — redeploy (fix pass 3)

**BUILD SUCCESS** (first attempt) + **confirmed deploy** (all 4 content packages' `lastUnpacked`
advanced this run, core bundle Active, form + embedded "Test Adaptive Form" page both HTTP 200) +
**0 failed unit tests** (112/112, unaffected by this content-only fix pass) + **D4 fix verified live
at the rule/mirror level** + **instance matches source, zero new orphans** (including re-confirming
the fix-pass-1 purge held).

### GATE RESULT: **PASS**

Carried-forward flags (unchanged, still non-blocking): forms-service coverage 76.6% (pre-existing gap,
no new core Java classes this cycle either), the two documented pre-production follow-ups (manager
lookup, finance-approver identity), the DoR-attachment email deviation, no local SMTP. OR-split
branching gap remains the recorded, human-actionable follow-up — untouched by this agent, per explicit
instruction. **New note:** the genuinely conclusive test for D4 (does `SubmissionDate` actually land
in the live DOM / network payload / persisted JCR on a real submission) is Sentinel's to run at
retest — this deploy confirms the fixed rule text is live, not the end-to-end submission outcome.

### Run metrics — redeploy (fix pass 3)

- `time_taken_minutes`: 13.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~3,000
  - `read`: ~10,500 (prior report §9/§10, formwright.md fix-pass-3 section, submitButton `.content.xml`
    fragment, bundle/package-manager JSON, node JSON dumps for the orphan sweep)
  - `write`: ~2,600 (this report section)
  - `other`: ~7,200 (1 mvn build's tool-call overhead, curl/bundle/package-manager/JCR node checks,
    surefire aggregation, timestamp conversions)
  - `total`: ~23,300

### Handoff YAML — redeploy (fix pass 3)

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
time_taken_minutes: 13.0
tokens_consumed:
  cli_text: 3000
  read: 10500
  write: 2600
  other: 7200
  total: 23300
build: BUILD SUCCESS
command: "mvn clean install -PautoInstallSinglePackage -q -Daem.host=localhost -Daem.port=4502"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps: PASS, ui.apps.structure: PASS, ui.content: PASS, ui.config: PASS, all: PASS, ui.frontend: PASS, it.tests: PASS, dispatcher: PASS, ui.tests: PASS }
unit_tests: { run: 112, passed: 112, failed: 0, coverage_pct: 76.6 }
static_analysis: { warnings: 1, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip", type: dispatcher-config, installed: false }
  - { name: "aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar", type: test-jar, installed: false }
deploy_confirmed: true
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true }
fixes_verified_live:
  - { fix: "D4 - submitButton SubmissionDate set-value target now $form.-scoped ($form.declarationFragment.submissionDate.$value) in both fd:click script mirror and fd:events click attribute", verified: true }
deploy_integrity: { instance_matches_source: true, orphans_purged: [] }
workflow_model_sync: { design_path: "/conf/global/settings/workflow/models/employee-training-request-approval", runtime_path: "/var/workflow/models/employee-training-request-approval", generated: true, verified_live: true, note: "unchanged this cycle - fix pass 3 touched no workflow artifacts" }
known_unfixed: { item: "OR-split branching (orsplit_manager/orsplit_finance) real Approve/Reject routing conditions", owner: "human via Workflow Editor UI", attempted_by_forgemaster: false }
report: ".claude/agents/runs/2026-07-22-employee-training-request/deployment/code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (final retest of D4 via live DOM value, outgoing network request body, and persisted JCR payload, plus regression of D3/TC-019 and rules #1-#7)
```

## 12. Redeploy — fix pass 4 (user-reported remediation, post-HANDOFF)

**Trigger:** The delivery had reached HANDOFF after 3 Sentinel-driven fix cycles. The user then
reviewed the deployed migration directly and found 2 further defects. Formwright fixed Issue 1
(fragments had no schema binding of their own — `schemaType="none"` while every field carried an
absolute `dataRef` anchored to the host form's schema, breaking the AF-editor's Form-Object
surface); Issue 2 required no code change (exhaustive 14-script inventory audit found 0 dropped
scripts). Groundsmith fixed Issue 3 — rebuilt the workflow model's Assign Task steps off a
**non-existent** OSGi class (`com.adobe.fd.workflow.aem.process.AssignTaskStep`, confirmed absent
from `/system/console/components`) onto the real registered `com.adobe.fd.workspace.step.service.AssignFormStep`,
and added `InvokeDDXProcess` + `ConvertToPDFAProcess` (replacing a generic DoR step with the
original legacy DDX) and upgraded Send Email to `SendEmailStep`. No re-authoring performed by this
agent. Full defect detail: `implementation/formwright.md` §"Fix pass 4", `implementation/groundsmith.md`
§"Fix pass 4".

### 12.1 Pre-flight

- AEM author reachable: `GET /system/console/bundles.json` → HTTP 200.
- Toolchain: Maven 3.9.16, build JVM confirmed **Java 11.0.31** via `mvn -version` (stray `java` on
  PATH resolves to 21, as in every prior deploy — does not affect the Maven build JVM).
- Confirmed all authored fixes present in source **before** building (no re-authoring by this agent):
  1. `employee-identity/.content.xml` and `declaration-consent/.content.xml` — `fragmentcontainer`
     `schemaType="none"` → `schemaType="jsonschema"` + `schemaRef="/content/dam/formsanddocuments/schema/employee-training-request.schema.json"`.
  2. Both fragments' DAM asset `.content.xml` — `metadata/formmodel="none"` → `formmodel="jsonschema"`.
  3. Workflow model `.content.xml` — both `AssignTaskStep` nodes rebuilt onto
     `PROCESS="com.adobe.fd.workspace.step.service.AssignFormStep"`; new `InvokeDDXProcess` and
     `ConvertToPDFAProcess` nodes added; Send Email upgraded to `PROCESS="com.adobe.fd.workflow.email.SendEmailStep"`.
  4. New file `ui.apps/.../apps/aem-adaptive-forms-agents/workflow/ddx/employee-training-request-approval/manager-confirmation.ddx`
     present on disk, containing the original legacy DDX extracted from `AssembleManagerConformation.process`.
- Did **not** touch D2 (OR-split routing — re-confirmed by groundsmith as a genuine platform
  limitation this pass, unchanged) or D4 (SubmissionDate redesign) — out of scope per instruction.

### 12.2 Build & deploy (redeploy of record)

- **Command:** `mvn clean install -PautoInstallSinglePackage -q -Daem.host=localhost -Daem.port=4502`
- **Result:** **BUILD SUCCESS** on the **first attempt** (exit code 0) — the `aem.sdk.api` pin from
  the original deploy remains correct, no further build-configuration change needed.
- Only console output under `-q`: the same 11× benign `[Fatal Error] :1:1: Content is not allowed
  in prolog.` SAX-parser warning from `LifeInsuranceNewPolicySubmitActionTest` (unrelated form,
  documented in §3). No new warnings, no failures.
- **Deploy landed (package `lastUnpacked` advanced — the reliable "installed" signal, not just exit
  code):**

  | Package | Prior `lastUnpacked` | New `lastUnpacked` |
  |---|---|---|
  | `all` | 1784725490717 | 1784783991535 |
  | `ui.apps` | (prior run) | 1784783997507 |
  | `ui.config` | (prior run) | 1784783997534 |
  | `ui.content` | (prior run) | 1784784006802 |

  All four advanced to this build run (checked against wall-clock immediately after — well within
  the build window, not a stale prior install).
- **Bundle state:** `aem-adaptive-forms-agents.core` v1.0.0.SNAPSHOT = **Active**. Instance-wide:
  `"738 bundles in total - all 738 bundles active."`
- **Unit tests:** aggregated from `core/target/surefire-reports/*.txt` — **112 run, 0 failures,
  0 errors, 0 skipped** (same suite as all 4 prior deploys; this fix pass touched no core Java).
- **Coverage:** unchanged at 76.6% on Forms service classes (`forms.*`) — carried forward, no new/
  changed core Java classes this pass (confirmed against both formwright's and groundsmith's
  "files changed" lists, which name only `.content.xml`/`.ddx` files).

### 12.3 Live verification of the 3 issues

| # | Issue | Live verification | Result |
|---|---|---|---|
| 1a | Fragment schema binding — `employee-identity` | `GET .../employee-identity/jcr:content/guideContainer.json` → `"schemaType":"jsonschema"`, `"schemaRef":"/content/dam/formsanddocuments/schema/employee-training-request.schema.json"` live | **PASS** |
| 1b | Fragment schema binding — `declaration-consent` | Same endpoint for `declaration-consent` → identical `schemaType`/`schemaRef` live | **PASS** |
| 1c | DAM fragment asset `formmodel` — both fragments | `GET .../content/dam/formsanddocuments/aem-adaptive-form-agents/{employee-identity,declaration-consent}/jcr:content/metadata.json` → `"formmodel":"jsonschema"` live on both | **PASS** |
| 2 | Script inventory audit | N/A — no code change; nothing to deploy or verify live for this issue (formwright's audit is a documentation artifact, not a source change) | **N/A (no fix required)** |
| 3a | Workflow Assign Task steps rebuilt onto real class | `GET /var/workflow/models/employee-training-request-approval.json` → both "Manager Approval" and "Finance Manager Approval" nodes show `"PROCESS":"com.adobe.fd.workspace.step.service.AssignFormStep"`. Zero occurrences of the old `com.adobe.fd.workflow.aem.process.AssignTaskStep` anywhere in the live runtime model. | **PASS** |
| 3b | New Invoke DDX / Convert-to-PDF/A / upgraded Send Email steps | Same endpoint → `"Assemble Manager Confirmation"` node = `"PROCESS":"com.adobe.fd.workflow.assembler.InvokeDDXProcess"`; `"Convert Manager Confirmation to PDF/A"` node = `"PROCESS":"com.adobe.fd.workflow.assembler.ConvertToPDFAProcess"`; both `"Send Approval Notification"`/`"Send Rejection Notification"` nodes = `"PROCESS":"com.adobe.fd.workflow.email.SendEmailStep"` | **PASS** |
| 3c | DDX file deployed and reachable | `GET /apps/aem-adaptive-forms-agents/workflow/ddx/employee-training-request-approval/manager-confirmation.ddx` → **HTTP 200**, body byte-for-byte matches the committed source (`<XDP source="EmployeeTrainingForm"/>` + `<XDP source="ApprovalInfo"/>` merge, `result="pdfComplete"`) | **PASS** |

**Critical check — are the new step classes actually registered on this instance, not just
referenced in source?** Cross-checked every PROCESS FQCN against `/system/console/components.json`
(OSGi component registry, not just JCR reference):

| Class | Registered? | State |
|---|---|---|
| `com.adobe.fd.workspace.step.service.AssignFormStep` | ✅ yes | `active` |
| `com.adobe.fd.workflow.assembler.InvokeDDXProcess` | ✅ yes | `active` |
| `com.adobe.fd.workflow.assembler.ConvertToPDFAProcess` | ✅ yes | `active` |
| `com.adobe.fd.workflow.email.SendEmailStep` | ✅ yes | `active` |
| `com.adobe.fd.workflow.aem.process.AssignTaskStep` (the ORIGINAL, now-removed class) | ❌ **no** | n/a — confirms groundsmith's fix-pass-4 discovery that the original build's class never existed on this instance |

This directly confirms the task's critical instruction: the new classes are genuinely registered
and Active OSGi components (not merely referenced in source), and the previously-referenced class
is confirmed absent — the old build was silently broken at the class-resolution level, now fixed.

- **Workflow model re-synced:** POSTed to `/conf/global/settings/workflow/models/employee-training-request-approval/jcr:content.generate.json`
  (CSRF token + same-origin Referer) → `{"msg":"Model successfully generated.","modelPath":"/var/workflow/models/employee-training-request-approval"}`.

### 12.4 Deploy integrity — orphan/stale-node sweep (step 3b)

Diffed live tree vs. source for every path touched by this fix pass, plus all previously-known-risky
paths (fix-pass-1 orphan site, the "Test Adaptive Form" embed):

| Path | Source | Live | Result |
|---|---|---|---|
| `.../employee-identity/.../guideContainer` (9 nodes) | `sectionTitle, employeeId, employeeName, email, department, managerId, managerName, location, businessUnit` | Same 9, same names | **Clean** |
| `.../declaration-consent/.../guideContainer` (2 nodes) | `acceptedTerms, submissionDate` | Same 2, same names | **Clean** |
| `.../guideContainer/employeeDetailsFragment` wrapper (fix-pass-1 orphan site) | No `fd:rules`/`fd:events` children | Same — prior purge still holds | **Clean — prior purge held** |
| Workflow model design tree (`/conf/.../employee-training-request-approval/jcr:content/flow`) — 12 titled step nodes in source | `Capture Submission Variables, Manager Approval, Manager Decision, Set Status — Manager Approved, Finance Manager Approval, Finance Manager Decision, Set Status — Finance Approved, Assemble Manager Confirmation, Convert Manager Confirmation to PDF/A, Send Approval Notification, Set Status — Rejected, Send Rejection Notification` | Same 12 titles, same structure, live | **Clean — exact match** |
| Workflow runtime model (`/var/workflow/models/employee-training-request-approval`) — PROCESS-type step count | 4× `SetVariableProcess`, 2× `AssignFormStep`, 1× `InvokeDDXProcess`, 1× `ConvertToPDFAProcess`, 2× `SendEmailStep` (10 total) expected from the 12 titled steps (2 are OR-split decision nodes, not PROCESS type) | Exact same counts confirmed live | **Clean** |
| `.../test-adaptive-form/.../root/container/container` | Exactly one `adaptiveFormEmbed` | Exactly one, `formRef` unchanged, no stacking | **Clean** |

**Orphan found and purged (outside the standard filter-root sweep, but a genuine instance/source
mismatch):** groundsmith's fix-pass-4 investigation created a disposable probe workflow model
(`zz-steps-probe`) to round-trip-verify the new step classes before committing to the real model,
and reported it "torn down." Verification found the **design node** (`/conf/.../zz-steps-probe`)
was indeed deleted (404), but its **generated runtime model** (`/var/workflow/models/zz-steps-probe`)
was left behind and still resolved HTTP 200 — a stray artifact present on the instance with zero
trace in source anywhere (`ui.content`/`ui.apps` grep for "zz-steps-probe": 0 hits).

- A Sling POST `:operation=delete` on the `/var/workflow/models/zz-steps-probe` path returned
  **HTTP 400 "Invalid parameters"** — `/var/workflow/models` is served by the Workflow Model
  resource provider, not the plain JCR resource provider, so the standard Sling delete verb does
  not apply here.
- **Purged instead via HTTP `DELETE`** (same CSRF token + Referer/Origin headers) →
  **HTTP 204**. Re-verified: `GET /var/workflow/models/zz-steps-probe.json` → **"Model does not
  exist."** Purge confirmed.
- This was a deploy-integrity reconciliation only (an instance artifact with zero source presence,
  left over from the fix's own verification process) — no source or form/workflow design was
  edited.

**Re-verified after purge:** the real `employee-training-request-approval` model (design + runtime)
is unaffected; both the form page and the "Test Adaptive Form" page still return **HTTP 200**.

### 12.5 Verdict & gate — redeploy (fix pass 4)

**BUILD SUCCESS** (first attempt) + **confirmed deploy** (all 4 content packages' `lastUnpacked`
advanced this run, core bundle Active, form + embedded "Test Adaptive Form" page both HTTP 200) +
**0 failed unit tests** (112/112, unaffected by this content-only fix pass) + **all 3 issues verified
live** (fragments' schema binding, DAM `formmodel`, workflow OOTB steps + DDX file) + **new step
classes confirmed registered and Active OSGi components on this instance** (the critical check —
the previously-referenced class confirmed absent, exactly matching groundsmith's discovery) +
**instance matches source, one orphan found and purged** (the disposable `zz-steps-probe` runtime
model, unrelated to the delivery's own artifacts but a genuine instance/source mismatch this sweep
is designed to catch).

### GATE RESULT: **PASS**

Carried-forward flags (unchanged, still non-blocking): forms-service coverage 76.6% (pre-existing
gap, no new core Java classes this cycle either), the two documented pre-production follow-ups
(manager lookup, finance-approver identity), no local SMTP (Send Email steps may no-op on
localhost — expected). D2 (OR-split branching) remains the recorded, human-actionable follow-up,
re-confirmed unchanged by groundsmith this pass — untouched by this agent, per explicit instruction.
D4 (SubmissionDate redesign) also untouched this pass, per explicit instruction. **New note:** the
AF-editor's own canvas/Form-Object rendering for Issue 1 (as opposed to the structural JCR/model.json
level this agent verified) is a browser-rendering-level check outside this agent's tooling — Sentinel
or a human editor session is the authoritative verification for that specific surface.

### Run metrics — redeploy (fix pass 4)

- `time_taken_minutes`: 19.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~3,800
  - `read`: ~24,000 (prior report §9–11, formwright.md + groundsmith.md fix-pass-4 sections in full,
    4 fixed `.content.xml`/DAM files, workflow model `.content.xml`, DDX file, live model.json /
    metadata.json / components.json responses, package-listing JSON)
  - `write`: ~3,200 (this report section)
  - `other`: ~12,500 (1 mvn build's tool-call overhead, curl/bundle/package-manager/JCR node checks
    across ~20 live HTTP calls, CSRF-token fetches, the zz-steps-probe delete-verb troubleshooting
    and purge, surefire aggregation)
  - `total`: ~43,500

### Handoff YAML — redeploy (fix pass 4)

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
time_taken_minutes: 19.0
tokens_consumed:
  cli_text: 3800
  read: 24000
  write: 3200
  other: 12500
  total: 43500
build: BUILD SUCCESS
command: "mvn clean install -PautoInstallSinglePackage -q -Daem.host=localhost -Daem.port=4502"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps: PASS, ui.apps.structure: PASS, ui.content: PASS, ui.config: PASS, all: PASS, ui.frontend: PASS, it.tests: PASS, dispatcher: PASS, ui.tests: PASS }
unit_tests: { run: 112, passed: 112, failed: 0, coverage_pct: 76.6 }
static_analysis: { warnings: 1, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip", type: dispatcher-config, installed: false }
  - { name: "aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar", type: test-jar, installed: false }
deploy_confirmed: true
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true }
fixes_verified_live:
  - { fix: "Issue 1 - employee-identity fragmentcontainer schemaType=jsonschema + schemaRef", verified: true }
  - { fix: "Issue 1 - declaration-consent fragmentcontainer schemaType=jsonschema + schemaRef", verified: true }
  - { fix: "Issue 1 - both DAM fragment assets formmodel=jsonschema", verified: true }
  - { fix: "Issue 2 - script inventory audit (no code change needed)", verified: "N/A, no fix required" }
  - { fix: "Issue 3 - Assign Task steps rebuilt onto AssignFormStep (registered+active, old class confirmed absent)", verified: true }
  - { fix: "Issue 3 - InvokeDDXProcess/ConvertToPDFAProcess added, SendEmailStep upgraded (all registered+active)", verified: true }
  - { fix: "Issue 3 - manager-confirmation.ddx deployed and reachable, byte-for-byte matches source", verified: true }
new_step_classes_registered_active:
  - { class: "com.adobe.fd.workspace.step.service.AssignFormStep", registered: true, state: active }
  - { class: "com.adobe.fd.workflow.assembler.InvokeDDXProcess", registered: true, state: active }
  - { class: "com.adobe.fd.workflow.assembler.ConvertToPDFAProcess", registered: true, state: active }
  - { class: "com.adobe.fd.workflow.email.SendEmailStep", registered: true, state: active }
old_class_confirmed_absent: { class: "com.adobe.fd.workflow.aem.process.AssignTaskStep", registered: false }
deploy_integrity: { instance_matches_source: true, orphans_purged: ["/var/workflow/models/zz-steps-probe (disposable probe model runtime node, design node already deleted by groundsmith, runtime node left behind - purged via HTTP DELETE after Sling :operation=delete returned 400)"] }
workflow_model_sync: { design_path: "/conf/global/settings/workflow/models/employee-training-request-approval", runtime_path: "/var/workflow/models/employee-training-request-approval", generated: true, verified_live: true }
known_unfixed:
  - { item: "D2 - OR-split branching (orsplit_manager/orsplit_finance) real Approve/Reject routing conditions", owner: "human via Workflow Editor UI", attempted_by_forgemaster: false }
  - { item: "D4 - SubmissionDate redesign", owner: "not in this pass's scope", attempted_by_forgemaster: false }
report: ".claude/agents/runs/2026-07-22-employee-training-request/deployment/code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (retest Issue 1 at AF-editor canvas/browser-render level, confirm Issue 3's steps execute end-to-end on a real workflow instance, regression of D1/D3/TC-019 and rules #1-#7)
```

## 13. Redeploy — fix pass 5 (OOTB step I/O configuration, server-side round-trip)

**Trigger:** Groundsmith configured every OOTB workflow step's I/O (Invoke DDX, Convert to PDF/A,
both Send Email steps, both Assign Task steps' data-I/O fields) via a live server-side POST/GET
round-trip against the design model's real `cq:dialog` + clientlib JS, then committed the round-tripped
values back into `.content.xml` source and regenerated the runtime model. This section rebuilds and
redeploys that source so the deployable artifact matches what groundsmith left live. Full defect/config
detail: `implementation/groundsmith.md` §"Fix pass 5" (CONFIG STATUS TABLE).

### 13.1 Pre-flight

- AEM author reachable: `GET /system/console/bundles.json` → HTTP 200.
- Toolchain: Maven 3.9.16, build JVM confirmed **Java 11.0.31** via `mvn -version` (stray `java` on
  PATH resolves to 21, as in every prior deploy — does not affect the Maven build JVM).
- Confirmed the source `.content.xml` already carried groundsmith's fix-pass-5 values for all 6 touched
  nodes (`process_ddx`, `process_pdfa`, `process_email_approved`, `process_email_rejected`,
  `assigntask_manager`, `assigntask_finance`) before building — matched byte-for-byte against the
  CONFIG STATUS TABLE in `groundsmith.md` §"Fix pass 5".

### 13.2 Build & deploy — attempt 1: BUILD FAILURE (new root cause, found and fixed this pass)

**Result:** BUILD FAILURE on `aem-adaptive-forms-agents.ui.content`'s `validate-package` goal:

```
[ERROR] ValidationViolation: Could not parse FileVault Document View XML: unknown type:
"outputkey":"pdfComplete","outputPath":"VARIABLE:assembledPacket"
@ .../employee-training-request-approval/.content.xml, line 406, column 50,
validator: jackrabbit-docviewparser,
JCR node path: .../employee-training-request-approval/jcr:content/flow/process_ddx/metaData
```

**Root cause (packaging-only, not a config-value defect):** groundsmith's live round-trip correctly
persisted `process_ddx`'s `outputDocs` as the scalar JSON string
`{"outputkey":"pdfComplete","outputPath":"VARIABLE:assembledPacket"}` via a Sling POST — a code path
that never touches DocView XML. But FileVault's DocView **attribute serialization** interprets any
value that starts with `{` as a `{type}value` type-hint (e.g. `{Boolean}true`); since the JSON value's
outer braces don't close until the very end, the parser read the entire JSON body as an unrecognized
"type" and failed. This is a packaging/serialization technicality that only surfaces at
`filevault-package-maven-plugin:validate-package` time — groundsmith's live JCR round-trip (Sling
POST/GET, not package install) never exercises this code path, so it could not have caught it.

**Fix (scoped to escaping only, semantic value unchanged):** added an explicit `{String}` type-hint
prefix so the literal `{` is treated as string content, not a type boundary — applied to `outputDocs`
(scalar) and, initially, to `inputDocs`/`ROUTES` (multi-valued, wrapped in `[...]`).

### 13.3 Build & deploy — attempt 2: BUILD SUCCESS, but a self-caught regression in the fix itself

Attempt 2 (full reactor) returned **BUILD SUCCESS**. Per this task's mandate to live-verify the design
model against groundsmith's config table (not just trust the build), fetched
`.../flow/{process_ddx,assigntask_manager,assigntask_finance}/metaData.json` and found the escaping fix
was **only half-correct**: the scalar `outputDocs` came back clean, but the multi-valued `inputDocs`
and both nodes' `ROUTES` had the literal text `{String}` leaked into their **first array element's
persisted value** (e.g. `ROUTES[0]` read `"{String}{\"Route_Label\":\"Approve\",...}"` instead of the
clean `"{\"Route_Label\":\"Approve\",...}"`). Root cause: for a multi-valued DocView property the
`{type}` hint must precede the opening `[` (applies once to the whole array), not sit inside it
immediately after `[` (where it is read as literal text of the first element instead). This was **my
own escaping mistake**, introduced while fixing 13.2 — caught only because this task's mandated
live-verification-against-source-table step was actually run, not skipped.

**Fix:** moved the `{String}` hint to precede `[` — `inputDocs="{String}[{&quot;inputkey&quot;:...]"`,
`ROUTES="{String}[{&quot;Route_Label&quot;:...]"` (both Assign Task nodes). Verified the hypothesis
cheaply via a scoped, explicitly-iterating hot-deploy (`mvn -pl ui.content clean install
-PautoInstallPackage -q` — not the deployment of record) before committing to a third full-reactor
build: live readback confirmed `inputDocs`/`ROUTES` on both nodes now persist with **zero** literal
`{String}` residue, exactly matching groundsmith's CONFIG STATUS TABLE.

### 13.4 Build & deploy — attempt 3 (the deployment of record)

- **Command:** `mvn clean install -PautoInstallSinglePackage -q -Daem.host=localhost -Daem.port=4502`
- **Result:** **BUILD SUCCESS.** Total time ~2 min 10 s (14:33:03–14:35:13 IST).
- Only console output: the same 11× benign `[Fatal Error] :1:1: Content is not allowed in prolog.`
  SAX-parser warning from `LifeInsuranceNewPolicySubmitActionTest` (unrelated form, documented in §3).
  No new warnings, no failures.
- **Deploy landed (package `lastUnpacked` advanced, the reliable "installed" signal):**

  | Package | `lastUnpacked` (UTC) | = IST | Within build window? |
  |---|---|---|---|
  | `all` | 2026-07-23T09:04:50Z | 14:34:50 | yes |
  | `ui.apps` | 2026-07-23T09:04:54Z | 14:34:54 | yes |
  | `ui.config` | 2026-07-23T09:04:54Z | 14:34:54 | yes |
  | `ui.content` | 2026-07-23T09:05:00Z | 14:35:00 | yes |

- **Bundle state:** `aem-adaptive-forms-agents.core` = **Active**, `Last Modification: Thu Jul 23
  14:34:56 IST 2026` (within this build's window). Instance-wide: `"738 bundles in total - all 738
  bundles active."`
- **Unit tests:** aggregated from `core/target/surefire-reports/*.txt` — **112 run, 0 failures, 0
  errors, 0 skipped** (same suite as every prior deploy; this fix pass touched no core Java).
- **Coverage:** unchanged at 76.6% (no new/changed core Java classes this pass).
- **Workflow runtime model regenerated:** `POST jcr:content.generate.json` →
  `{"msg":"Model successfully generated.","modelPath":"/var/workflow/models/employee-training-request-approval"}`;
  live `GET /var/workflow/models/employee-training-request-approval.json` → `"version":"1.9"`,
  `"lastSynced":"2026-07-23T14:56:35.459+05:30"`.

### 13.5 Live verification — design model now matches groundsmith's CONFIG STATUS TABLE, byte-for-byte

Fetched `.../jcr:content/flow/{node}/metaData.json` for all 6 touched nodes post-deploy:

| Node | Field(s) | Live value | Matches table? |
|---|---|---|---|
| `process_ddx` | `ddx` | `ABSOLUTE_PATH:/apps/.../manager-confirmation.ddx` | ✅ |
| `process_ddx` | `inputDocs` | `["{\"inputkey\":\"EmployeeTrainingForm\",\"sourcePath\":\"RELATIVE_PLOAD:data.xml\"}","{\"inputkey\":\"ApprovalInfo\",\"sourcePath\":\"RELATIVE_PLOAD:data.xml\"}"]` — clean, no `ddxType`, no `{String}` residue | ✅ |
| `process_ddx` | `outputDocs` | `{"outputkey":"pdfComplete","outputPath":"VARIABLE:assembledPacket"}` (scalar) | ✅ |
| `process_pdfa` | `inDoc`/`pdfaDoc`/`compliance` | `VARIABLE:assembledPacket` / `RELATIVE_PLOAD:DocumentofRecord/DoR.pdf` / `1a` (unchanged, as directed) | ✅ |
| `process_email_approved` | recipient/subject/template/attachment | `Variable`/`applicantEmail`, `Literal`/`Employee Request Approved`, `.../approval.html`, `Variable`/`assembledPacket` | ✅ |
| `process_email_rejected` | recipient/subject/template | `Variable`/`applicantEmail`, `Literal`/`Request Rejected`, `.../rejection.html` | ✅ |
| `assigntask_manager` | flat `INPUT_DATAXML`/`OUTPUT_DATAXML`/`INPUT_FORM_ATTACHMENTS`/`OUTPUT_FORM_ATTACHMENTS` | `data.xml`/`data.xml`/`attachments`/`attachments` — **no trailing slash on any of the 4** | ✅ |
| `assigntask_manager` | combined `INPUT_COMBINED_DATAXML`/`OUTPUT_COMBINED_DATAXML`/`INPUT_COMBINED_FORM_ATTACHMENTS`/`OUTPUT_COMBINED_FORM_ATTACHMENTS` | `RELATIVE_PLOAD:data.xml`/`RELATIVE_PLOAD:data.xml`/`RELATIVE_PLOAD:attachments`/`RELATIVE_PLOAD:attachments` — **no trailing slash on any of the 4** | ✅ |
| `assigntask_finance` | same 8 fields (flat + combined) | identical shape and values to `assigntask_manager` | ✅ |
| `assigntask_manager` / `assigntask_finance` | `ROUTES` | `["{\"Route_Label\":\"Approve\",\"Route_Icon_Name\":\"\"}","{\"Route_Label\":\"Reject\",\"Route_Icon_Name\":\"\"}"]` on both nodes — clean, no `{String}` residue | ✅ |

**Trailing-slash clean-check (explicit ask, D1's exact original bug class): all 16 checks clean**
(4 flat + 4 combined fields × 2 Assign Task nodes) — no trailing slash anywhere, including the
COMBINED variants added this fix pass.

**Runtime model cross-check:** `grep`ped the full `/var/workflow/models/employee-training-request-approval.json`
body for the literal string `String}` (zero matches — confirms the attempt-2 regression did not survive
into the final deployed runtime) and for any `.../data.xml/"` or `.../attachments/"` trailing-slash
pattern (zero matches). Design and runtime now agree with each other and with groundsmith's source table.

**Form guideContainer (D1 regression, unrelated node but same bug class):** `attachmentsFolderPath` =
`"attachments"` — no trailing slash, unchanged, D1 still holds. `workflowModel` =
`/var/workflow/models/employee-training-request-approval` — correctly wired.

### 13.6 Deploy integrity — orphan/stale-node sweep (step 3b)

| Path | Source | Live | Result |
|---|---|---|---|
| `.../employee-training-request/.../guideContainer` (8 nodes) | `formTitle, employeeDetailsFragment, trainingRequestDetailsPanel, justificationPanel, attachmentsPanel, declarationFragment, approvalInformationPanel, actionsPanel` | Same 8, same names, same order | **Clean** |
| `.../guideContainer/employeeDetailsFragment` wrapper (fix-pass-1 orphan site) | No `fd:rules`/`fd:events` children | Same — prior purge still holds | **Clean — prior purge held** |
| Workflow design flow (`/conf/.../flow`) — 12 nodes | `process_setvars, assigntask_manager, orsplit_manager, process_setstatus_managerapproved, assigntask_finance, orsplit_finance, process_setstatus_financeapproved, process_ddx, process_pdfa, process_email_approved, process_setrejection, process_email_rejected` | Same 12, **same names AND same order** (the Fix-pass-4 "live node order vs committed source order" caveat is now resolved — this fresh deploy re-installed source in the correct order) | **Clean** |
| `.../test-adaptive-form/.../root/container` | Exactly one form-embed container | Exactly one `container` node, `formRef` = `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request` | **Clean — no stacking** |

**Orphan found and purged (outside the standard filter-root sweep, a genuine instance/source
mismatch):** while re-sweeping every known-risky path plus the disposable-probe class of leftover
from prior fix passes, found `/var/workflow/models/zz-or-probe2` (design node already 404 — deleted —
but the generated **runtime** node was left behind and still resolved HTTP 200) — a second instance of
the exact same leftover-probe-runtime pattern documented for `zz-steps-probe` in Fix pass 4 §12.4. This
is groundsmith's Fix pass 5 D2 re-investigation probe ("re-tested LIVE on a fresh probe model
(zz-or-probe2)" — see `groundsmith.md` §"Fix pass 5"), with zero real source presence (the one grep hit
for the string "zz-or-probe2" is a documentation comment inside the real model's `.content.xml`, not an
artifact belonging to a `zz-or-probe2` model).

- Sling POST `:operation=delete` on `/var/workflow/models/zz-or-probe2` → **HTTP 400** (same
  `/var/workflow/models` resource-provider limitation as Fix pass 4).
- Purged via **HTTP `DELETE`** (CSRF token + Referer/Origin) → **HTTP 204**.
- Re-verified: `GET /var/workflow/models/zz-or-probe2.json` → **"Model does not exist."** (404).
- Re-verified after purge: the real model, the form page, and the "Test Adaptive Form" page are all
  unaffected (all HTTP 200). This was a deploy-integrity reconciliation only — no source or form/
  workflow design was edited to perform it.

### 13.7 Verdict & gate — redeploy (fix pass 5)

**BUILD SUCCESS** (attempt 3, deployment of record — attempts 1–2 diagnosed/fixed a real packaging
defect and a self-caught escaping regression before committing) + **confirmed deploy** (all 4 content
packages' `lastUnpacked` advanced this run, core bundle Active, form + embedded "Test Adaptive Form"
page both HTTP 200) + **0 failed unit tests** (112/112) + **source now matches the live round-tripped
state for design AND runtime** (all 6 touched nodes byte-for-byte against groundsmith's CONFIG STATUS
TABLE, zero `{String}` escaping residue, runtime regenerated to v1.9) + **trailing-slash clean-check:
16/16 clean** (flat + combined, both Assign Task nodes) + **instance matches source, one orphan found
and purged** (`zz-or-probe2` runtime leftover from groundsmith's own D2 probe investigation).

### GATE RESULT: **PASS**

Carried-forward flags (unchanged, still non-blocking): forms-service coverage 76.6% (pre-existing gap,
no new core Java classes this cycle), the two documented pre-production follow-ups (manager lookup,
finance-approver identity), no local SMTP (Send Email steps may no-op on localhost — expected). D2
(OR-split branching) remains the recorded, human-actionable follow-up — unchanged, untouched by this
agent. **New note carried forward for Sentinel:** groundsmith flagged that `assigntask_manager`/
`assigntask_finance` set BOTH the flat and COMBINED data/attachment property forms because it's
genuinely unclear which one `AssignFormStep`'s Java reads at runtime — this deploy confirms both forms
are live and clean (no trailing slash, no escaping artifacts) on both nodes, but **not** which pair is
actually load-bearing; that functional confirmation needs a completed task, which is Sentinel's to run.
Likewise, the ROUTES JSON-per-element shape's runtime parsing by `AssignFormStep` at task-completion is
confirmed **persisted** correctly but not confirmed **consumed** correctly — also Sentinel's to verify.

### Run metrics — redeploy (fix pass 5)

- `time_taken_minutes`: 50.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~9,500 (root-cause diagnosis of the DocView escaping failure, the self-caught
    regression analysis, the array-vs-scalar type-hint hypothesis and its write-up)
  - `read`: ~38,000 (groundsmith.md fix-pass-5 section re-read in full, this report's history sections
    for format/precedent, the workflow model `.content.xml` twice, 3 build-log tails, ~20 live
    metaData.json/runtime-model JSON fetches across 3 build attempts, package vlt:definition JSON
    ×4, bundle JSON)
  - `write`: ~6,500 (6 targeted `.content.xml` edits across 2 fix iterations, this report section)
  - `other`: ~29,000 (3 full/scoped mvn build attempts' tool-call overhead, CSRF-token fetches, the
    zz-or-probe2 delete-verb troubleshooting + purge, surefire aggregation, ~15 additional live
    HTTP verification calls)
  - `total`: ~83,000

### Handoff YAML — redeploy (fix pass 5)

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
time_taken_minutes: 50.0
tokens_consumed:
  cli_text: 9500
  read: 38000
  write: 6500
  other: 29000
  total: 83000
build: BUILD SUCCESS
build_attempts: 3
command: "mvn clean install -PautoInstallSinglePackage -q -Daem.host=localhost -Daem.port=4502"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps: PASS, ui.apps.structure: PASS, ui.content: PASS, ui.config: PASS, all: PASS, ui.frontend: PASS, it.tests: PASS, dispatcher: PASS, ui.tests: PASS }
unit_tests: { run: 112, passed: 112, failed: 0, coverage_pct: 76.6 }
static_analysis: { warnings: 1, findings: 0 }
build_issues_found_and_fixed_by_forgemaster:
  - { issue: "FileVault DocView rejects a scalar attribute value starting with '{' as an unrecognized type-hint (process_ddx outputDocs)", fix: "prefixed with explicit {String} type-hint", scope: "packaging/escaping only, semantic value unchanged" }
  - { issue: "self-caught regression: {String} hint placed INSIDE the multi-value [...] wrapper leaked literally into inputDocs[0]/ROUTES[0] on both Assign Task nodes", fix: "moved {String} hint to precede the opening [ (applies once to the whole array), verified via scoped ui.content-only hot-deploy before the full-reactor rebuild", caught_by: "mandatory live-verification-against-source-table step (13.5), not by the build itself" }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip", type: dispatcher-config, installed: false }
  - { name: "aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar", type: test-jar, installed: false }
deploy_confirmed: true
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true }
design_matches_live_roundtrip:
  process_ddx: { ddxType_removed: true, ddx: true, inputDocs: true, outputDocs: true, verified: true }
  process_pdfa: { inDoc: true, pdfaDoc: true, compliance_unchanged: true, verified: true }
  process_email_approved: { recipient: true, subject: true, template: true, attachment: true, verified: true }
  process_email_rejected: { recipient: true, subject: true, template: true, verified: true }
  assigntask_manager: { flat_io: true, combined_io: true, routes: true, verified: true }
  assigntask_finance: { flat_io: true, combined_io: true, routes: true, verified: true }
runtime_model_sync: { design_path: "/conf/global/settings/workflow/models/employee-training-request-approval", runtime_path: "/var/workflow/models/employee-training-request-approval", generated: true, version: "1.9", verified_live: true }
trailing_slash_check: { fields_checked: 16, clean: 16, dirty: 0, scope: "flat + combined INPUT/OUTPUT_DATAXML and FORM_ATTACHMENTS on both Assign Task nodes" }
deploy_integrity: { instance_matches_source: true, orphans_purged: ["/var/workflow/models/zz-or-probe2 (groundsmith's Fix pass 5 D2 re-investigation probe; design node already deleted, runtime node left behind; purged via HTTP DELETE after Sling :operation=delete returned 400)"] }
known_unfixed:
  - { item: "D2 - OR-split branching (orsplit_manager/orsplit_finance) real Approve/Reject routing conditions", owner: "human via Workflow Editor UI", attempted_by_forgemaster: false }
  - { item: "which of flat vs COMBINED Assign Task I/O properties is actually load-bearing at runtime", owner: "sentinel via a completed task", attempted_by_forgemaster: false }
  - { item: "ROUTES JSON-per-element shape's runtime parsing by AssignFormStep at task-completion", owner: "sentinel via a completed task", attempted_by_forgemaster: false }
report: ".claude/agents/runs/2026-07-22-employee-training-request/deployment/code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (retest: confirm D1 via full AF-submit path, functionally verify Invoke DDX/Convert-to-PDF/A/SendEmail-attachment/ROUTES parsing by actually completing a manager-approval task end-to-end, confirm which Assign Task I/O property pair is load-bearing, D2 remains the documented open item)
```

---

## 14. Redeploy — fix pass 6 (D6/D7/D8 remediation + new canonical `assign-task-to-admin` model)

**Trigger:** Groundsmith's Fix pass 6 fixed 3 Critical defects (D6/D7/D8) that were blocking Manager
Approval task completion entirely — `FORM_TYPE` on both Assign Task nodes changed from the invalid
`ADAPTIVE_FORM` to the real enum member `AF`, and Fix pass 5's `INPUT_COMBINED_*`/`OUTPUT_COMBINED_*`
hedge properties were removed entirely (kept only the flat `INPUT_DATAXML`/`OUTPUT_DATAXML`/
`INPUT_FORM_ATTACHMENTS`/`OUTPUT_FORM_ATTACHMENTS`, still no trailing slash). Along the way Groundsmith
discovered the project's documented canonical shared reference model, `assign-task-to-admin`, did not
actually exist on this instance and created it per rule 0a. Full defect/config detail:
`implementation/groundsmith.md` §"Fix pass 6 — D6/D7/D8 fix via reference-model mirror".

### 14.1 Pre-flight

- AEM author reachable: `GET /system/console/bundles.json` → HTTP 200.
- Toolchain: Maven 3.9.16, build JVM confirmed Java 11.0.31 via `mvn -version` (stray `java` on PATH
  still resolves to 21, as in every prior deploy — does not affect the Maven build JVM).
- Confirmed both authored source changes present before building (no re-authoring by this agent):
  1. `employee-training-request-approval/.content.xml` — `assigntask_manager`/`assigntask_finance`
     both `FORM_TYPE="AF"`, zero `*_COMBINED_*` properties, `ROUTES` unchanged from Fix pass 5.
  2. **NEW FILE** `assign-task-to-admin/.content.xml` — single `assigntask_admin` Assign Task step,
     `STATIC_ASSIGNEE="admin"`, `TASK_PRIORITY="MEDIUM"`, `ROUTES="Complete"`, `FORM_TYPE="READ_ONLY_AF"`.
  3. `ui.content/.../META-INF/vault/filter.xml` — new filter root
     `/conf/global/settings/workflow/models/assign-task-to-admin` (`mode="update"`) present.

### 14.2 Build & deploy (redeploy of record)

- **Command:** `mvn clean install -PautoInstallSinglePackage -q`
- **Result:** **BUILD SUCCESS** on the **first attempt** — exit code confirmed `0` explicitly (echoed
  post-pipeline, since `-q` suppresses Maven's own `BUILD SUCCESS` banner text). Only console output:
  the same 11× benign `[Fatal Error] :1:1: Content is not allowed in prolog.` SAX-parser warning from
  `LifeInsuranceNewPolicySubmitActionTest` (unrelated form, documented in §3) — no new warnings, no
  failures, no root-cause diagnosis needed this cycle.
- **Unit tests:** aggregated from `core/target/surefire-reports/*.txt` — **112 run, 0 failures, 0
  errors, 0 skipped** (same suite as every prior deploy; this fix pass touched no core Java).
  **Coverage:** unchanged at 76.6% (no new/changed core Java classes this pass).
- **Deploy landed** — package `lastUnpacked` (the reliable "installed" signal) advanced for all 4
  content packages within the build window (epoch ms, all within ~15s of each other and ~3 min of the
  post-build verification):
  `all`=1784808876907, `ui.apps`=1784808881004, `ui.config`=1784808881032, `ui.content`=1784808891391.
- **Bundle state:** `aem-adaptive-forms-agents.core` (id 738) = **Active**, version `1.0.0.SNAPSHOT`.
  Instance-wide: 738 bundles total, all Active.

### 14.3 Live verification of D6/D7/D8 on the design model

Fetched `.../employee-training-request-approval/jcr:content/flow/{assigntask_manager,assigntask_finance}/metaData.json`:

| Check | `assigntask_manager` | `assigntask_finance` | Result |
|---|---|---|---|
| `FORM_TYPE` | `"AF"` | `"AF"` | ✅ not `ADAPTIVE_FORM` |
| `*_COMBINED_*` properties present | none | none | ✅ zero (D6 hedge fully removed) |
| Flat `INPUT_DATAXML`/`OUTPUT_DATAXML` | `data.xml`/`data.xml` | `data.xml`/`data.xml` | ✅ no trailing slash |
| Flat `INPUT_FORM_ATTACHMENTS`/`OUTPUT_FORM_ATTACHMENTS` | `attachments`/`attachments` | `attachments`/`attachments` | ✅ no trailing slash |
| `ROUTES` | `["{\"Route_Label\":\"Approve\"...}","{\"Route_Label\":\"Reject\"...}"]` | identical shape | ✅ unchanged from Fix pass 5 (D7 not touched, as instructed) |

### 14.4 Runtime regeneration — design → `/var`

Per this project's established convention, only the `/conf` design model is committed; `/var` is
generated. Regenerated both models (CSRF token + same-origin Referer, per `ModelGenerateServlet`):

- `POST /conf/global/settings/workflow/models/employee-training-request-approval/jcr:content.generate.json`
  → `{"msg":"Model successfully generated.","modelPath":"/var/workflow/models/employee-training-request-approval"}`.
  Runtime advanced **v1.14 → v1.15**. Live `GET /var/workflow/models/employee-training-request-approval.json`
  confirms: `FORM_TYPE` appears exactly twice, both `"AF"`; zero occurrences of the string `COMBINED`;
  zero occurrences of `ADAPTIVE_FORM`; all 8 flat I/O values (`INPUT_DATAXML`/`OUTPUT_DATAXML`/
  `INPUT_FORM_ATTACHMENTS`/`OUTPUT_FORM_ATTACHMENTS` × 2 nodes) clean, no trailing slash on any.
  Node order in the generated flow (`Capture Submission Variables → Manager Approval → Set Status —
  Manager Approved → Finance Manager Approval → Set Status — Finance Approved → Assemble Manager
  Confirmation → Convert Manager Confirmation to PDF/A → Send Approval Notification → Set Status —
  Rejected → Send Rejection Notification`) matches source order exactly — same 12-node shape as Fix
  pass 5's confirmed order. The `orsplit_manager`/`orsplit_finance` design nodes still do not appear in
  the generated runtime flow — this is the pre-existing, still-open D2 platform limitation (silent
  linear collapse), unchanged by this pass and explicitly out of scope for it, not a new regression.

- **`assign-task-to-admin` — deployed and reachable for the first time on this instance:**
  Design page `GET /conf/global/settings/workflow/models/assign-task-to-admin.json` → **HTTP 200**.
  A `/var` runtime copy (v1.2) was already present pre-build from Groundsmith's own creation-time
  `generate.json` call; regenerated it again as part of this pass's confirmation step to bind it to the
  authoritative build → `{"msg":"Model successfully generated.","modelPath":"/var/workflow/models/assign-task-to-admin"}`,
  advancing **v1.2 → v1.3**. Live runtime confirms `STATIC_ASSIGNEE="admin"`, `TASK_PRIORITY="MEDIUM"`,
  `FORM_TYPE="READ_ONLY_AF"`, `ROUTES="Complete"` (bare scalar — Groundsmith's own flagged, non-blocking,
  out-of-scope note that this bare-string form would fail the same `ArgumentParser.getRoutes` JSON
  parse D7 diagnosed for the other model; not this model's problem to fix, and not reported broken by
  any consumer yet).

### 14.5 Deploy integrity — orphan/stale-node sweep (step 3b)

| Path | Source children | Live children | Result |
|---|---|---|---|
| `/conf/.../employee-training-request-approval/jcr:content/flow` (12 nodes) | `process_setvars, assigntask_manager, orsplit_manager, process_setstatus_managerapproved, assigntask_finance, orsplit_finance, process_setstatus_financeapproved, process_ddx, process_pdfa, process_email_approved, process_setrejection, process_email_rejected` | Same 12, same names, same order | **Clean** |
| `/conf/.../assign-task-to-admin/jcr:content/flow` (**new filter root, first deploy**) | `assigntask_admin` (1 node) | `assigntask_admin` (1 node) | **Clean — new model deployed with zero drift** |
| `.../employee-training-request/jcr:content/guideContainer` (8 nodes) | `formTitle, employeeDetailsFragment, trainingRequestDetailsPanel, justificationPanel, attachmentsPanel, declarationFragment, approvalInformationPanel, actionsPanel` | Same 8, same names, same order | **Clean — prior fix-pass-1 purge still holds** |
| `.../test-adaptive-form/.../root/container/container` | Exactly one `adaptiveFormEmbed` | Exactly one `adaptiveFormEmbed`, `formRef="/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request"` | **Clean — no stacking** |

**No purge was necessary.** Instance tree matches source exactly for both re-authored workflow model
nodes, the brand-new `assign-task-to-admin` filter root, the form's `guideContainer`, and the embedded
"Test Adaptive Form" page.

### 14.6 Live confirmation — form + page

- `GET /content/forms/af/aem-adaptive-form-agents/employee-training-request.html` → **HTTP 200**
- `GET /content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html` → **HTTP 200**

### 14.7 New, unfixed finding carried forward (not this agent's to fix)

Groundsmith flagged that `formview.jsp` throws a `NullPointerException` on every task-detail load,
independent of the FORM_TYPE fix, surfaced only once D6/D8 stopped masking it. Per groundsmith's own
testing this does not block button rendering (real Approve/Reject buttons render correctly regardless).
Not attempted by this agent — flagged for Sentinel to characterize functionally.

### 14.8 Verdict & gate — redeploy (fix pass 6)

**BUILD SUCCESS** (first attempt, no root-cause fixing needed) + **confirmed deploy** (all 4 content
packages' `lastUnpacked` advanced this run, core bundle Active, form + embedded "Test Adaptive Form"
page both HTTP 200) + **0 failed unit tests** (112/112) + **D6/D7/D8 fixes verified live on the design
model AND the regenerated runtime model** (`FORM_TYPE=AF` ×2, zero `COMBINED`/`ADAPTIVE_FORM` residue,
zero trailing slashes, `ROUTES` unchanged) + **new `assign-task-to-admin` model deployed cleanly and
reachable** (design HTTP 200, runtime regenerated to v1.3, single-node flow matches source exactly) +
**instance matches source everywhere checked, zero orphans found, no purge needed**.

### GATE RESULT: **PASS**

Carried-forward flags (unchanged, still non-blocking): forms-service coverage 76.6% (pre-existing gap,
no new core Java classes this cycle), the two documented pre-production follow-ups (manager lookup,
finance-approver identity), the DoR-attachment email deviation, no local SMTP. D2 (OR-split branching)
and D4 (SubmissionDate) remain the recorded, human-actionable follow-ups — unchanged, untouched by this
agent. **New non-blocking flag for Sentinel:** the `formview.jsp` NullPointerException on task-detail
load (§14.7) — does not block button rendering per groundsmith's testing, needs Sentinel to
characterize whether it affects any functional path.

### Run metrics — redeploy (fix pass 6)

- `time_taken_minutes`: 16.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~3,200
  - `read`: ~15,500 (groundsmith.md fix-pass-6 section, both workflow model `.content.xml` files in
    full, filter.xml, prior report sections 9/13 for format precedent, this file's tail)
  - `write`: ~2,800 (this report section)
  - `other`: ~8,500 (1 full mvn build's tool-call overhead, package-list/bundle JSON fetches, CSRF
    token + 2 generate.json calls, ~10 live metaData/runtime-model/guideContainer verification calls)
  - `total`: ~30,000

### Handoff YAML — redeploy (fix pass 6)

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
time_taken_minutes: 16.0
tokens_consumed:
  cli_text: 3200
  read: 15500
  write: 2800
  other: 8500
  total: 30000
build: BUILD SUCCESS
build_attempts: 1
command: "mvn clean install -PautoInstallSinglePackage -q"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps: PASS, ui.apps.structure: PASS, ui.content: PASS, ui.config: PASS, all: PASS, ui.frontend: PASS, it.tests: PASS, dispatcher: PASS, ui.tests: PASS }
unit_tests: { run: 112, passed: 112, failed: 0, coverage_pct: 76.6 }
static_analysis: { warnings: 1, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip", type: dispatcher-config, installed: false }
  - { name: "aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar", type: test-jar, installed: false }
deploy_confirmed: true
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true }
design_model_fixes_verified:
  employee_training_request_approval:
    assigntask_manager: { form_type: "AF", combined_props_removed: true, routes_unchanged: true, trailing_slash: "none" }
    assigntask_finance: { form_type: "AF", combined_props_removed: true, routes_unchanged: true, trailing_slash: "none" }
  assign_task_to_admin:
    design_live: true
    runtime_generated: true
    runtime_version: "1.3"
    flow_matches_source: true
runtime_model_sync:
  employee-training-request-approval: { design_path: "/conf/global/settings/workflow/models/employee-training-request-approval", runtime_path: "/var/workflow/models/employee-training-request-approval", generated: true, version: "1.15", verified_live: true }
  assign-task-to-admin: { design_path: "/conf/global/settings/workflow/models/assign-task-to-admin", runtime_path: "/var/workflow/models/assign-task-to-admin", generated: true, version: "1.3", verified_live: true, first_deploy: true }
deploy_integrity: { instance_matches_source: true, orphans_purged: [] }
known_unfixed:
  - { item: "D2 - OR-split branching (orsplit_manager/orsplit_finance) real Approve/Reject routing conditions", owner: "human via Workflow Editor UI", attempted_by_forgemaster: false }
  - { item: "D4 - SubmissionDate", owner: "documented follow-up", attempted_by_forgemaster: false }
  - { item: "formview.jsp NullPointerException on task-detail load (new, surfaced once D6/D8 stopped masking it)", owner: "sentinel to characterize", attempted_by_forgemaster: false }
report: ".claude/agents/runs/2026-07-22-employee-training-request/deployment/code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (retest Manager/Finance Approval task completion end-to-end now that D6/D7/D8 are cleared; verify the formview.jsp NPE's functional impact; D2/D4 remain documented open items)
```

---

## 15. Redeploy — fix pass 8 (D10 remediation: assignee `administrators` group → `admin` user)

**Trigger:** Sentinel's real task-completion attempt surfaced a NEW, deeper Critical defect (D10)
superseding D9: with `STATIC_ASSIGNEE="administrators"` (a group), completing either Assign Task work
item requires a claim/delegate step first, and `delegateWorkItem` throws
`java.lang.IllegalStateException: This session has been closed` on this instance (direct REST
completion independently fails with HTTP 500 "Invalid user - admin", since `admin` is not a real
member of the `administrators` group and group-membership grants were found not to persist via the
standard API here). Per the user's explicit direction, Groundsmith changed `STATIC_ASSIGNEE` from
`"administrators"` to the `admin` USER directly on both `assigntask_manager` and `assigntask_finance`
in `ui.content/.../conf/global/settings/workflow/models/employee-training-request-approval/.content.xml`
— sidesteps claim/delegate entirely since a task already assigned to a specific user needs no
claiming. `PROCESS_PARTICIPANT_TYPE="static"` and every other metaData property (FORM_TYPE, WORKITEM_COMMENT,
IS_COMMENT_ALLOWED, flat I/O paths, ROUTES) are unchanged. This fix was already live-pushed and
readback-confirmed by Groundsmith directly (design model + regenerated runtime model both showed
`STATIC_ASSIGNEE=admin`, zero drift) — this section is the confirming full-reactor build+deploy that
makes the packaged artifact match that live state exactly, so a future redeploy from a stale `all`
package cannot silently revert the fix. Full defect detail: `implementation/groundsmith.md`
§"Fix pass 8".

### 15.1 Pre-flight

- AEM author reachable: `GET /system/console/bundles.json` → HTTP 200.
- Toolchain: Maven 3.9.16, build JVM **Java 11.0.31** (confirmed via `mvn -version`; a stray `java`
  on PATH resolves to 21 — same benign discrepancy noted on every prior deploy, does not affect the
  Maven build JVM).
- Confirmed the authored fix present in source **before** building (no re-authoring by this agent):
  both `assigntask_manager` and `assigntask_finance` nodes in
  `employee-training-request-approval/.content.xml` carry `STATIC_ASSIGNEE="admin"`,
  `PROCESS_PARTICIPANT_TYPE="static"` unchanged, all other metaData properties (FORM_TYPE=READ_ONLY_AF,
  WORKITEM_COMMENT=rejectionReason, IS_COMMENT_ALLOWED=true, flat I/O paths, ROUTES) unchanged from
  fix passes 6/7.
- Did **not** touch D2 (OR-split branching) or D4 (SubmissionDate) — both remain the recorded,
  documented follow-ups, unchanged, untouched by this agent.

### 15.2 Build & deploy (redeploy of record)

- **Command:** `mvn clean install -PautoInstallSinglePackage` (also confirmed with a `-q` dry pass —
  identical result, only the same 11× benign `[Fatal Error] :1:1: Content is not allowed in prolog.`
  SAX-parser warning appears, pre-existing from `LifeInsuranceNewPolicySubmitActionTest`, documented in
  §3, unrelated to this delivery and non-blocking).
- **Result:** **BUILD SUCCESS** on the **first attempt** — no root-cause fixing needed this cycle; the
  `aem.sdk.api` pin remains correct and required no further change.
- **Total time:** 3 min 18 s. Finished at `2026-07-27T11:27:53+05:30`.
- **Reactor Summary:** all 11 modules SUCCESS (core, ui.apps.structure, ui.apps, ui.content, ui.config,
  all, ui.frontend, it.tests, dispatcher, ui.tests, root pom) — 11/11 green.
- **Unit tests:** aggregated from the build's surefire output — **112 run, 0 failures, 0 errors,
  0 skipped** (same suite as every prior deploy; this is a content-only workflow-metaData fix, no
  new/changed Java).
- **Deploy landed:** `Installing aem-adaptive-forms-agents.all (...) to
  http://localhost:4502/crx/packmgr/service.jsp` → `Installing content...` → `Package installed in
  588ms.` — the authoritative "installed" signal for this run.
- **Bundle state:** `aem-adaptive-forms-agents.core` confirmed **Active** via
  `/system/console/bundles.json`.

### 15.3 Live verification — D10 fix (design + runtime, post-redeploy)

| # | Check | Endpoint | Result |
|---|---|---|---|
| 1 | Design model, `assigntask_manager` | `GET .../employee-training-request-approval/jcr:content/flow/assigntask_manager/metaData.json` | `"STATIC_ASSIGNEE":"admin"`, `"PROCESS_PARTICIPANT_TYPE":"static"`, `"FORM_TYPE":"READ_ONLY_AF"` — no reversion to `administrators` |
| 2 | Design model, `assigntask_finance` | `GET .../employee-training-request-approval/jcr:content/flow/assigntask_finance/metaData.json` | `"STATIC_ASSIGNEE":"admin"`, `"PROCESS_PARTICIPANT_TYPE":"static"`, `"FORM_TYPE":"READ_ONLY_AF"` — no reversion to `administrators` |
| 3 | Runtime model regeneration | `POST /conf/global/settings/workflow/models/employee-training-request-approval/jcr:content.generate.json` (CSRF + Referer) | `{"msg":"Model successfully generated.","modelPath":"/var/workflow/models/employee-training-request-approval"}` |
| 4 | Runtime model, both steps | `GET /var/workflow/models/employee-training-request-approval.json` | `STATIC_ASSIGNEE:"admin"` present under BOTH "Manager Approval" and "Finance Manager Approval" step titles |

This confirms the redeploy did **not** reintroduce the old `administrators` value from any stale
source — the packaged `all` artifact now carries the same `STATIC_ASSIGNEE=admin` value Groundsmith
had already live-pushed, closing the drift-risk window this section exists to close.

### 15.4 Deploy integrity — orphan/stale-node sweep (step 3b)

This fix pass changed only an attribute value (`STATIC_ASSIGNEE`) on two existing `metaData` nodes —
no panel re-authoring, no node renaming/removal — so no orphan risk was introduced, but the sweep was
still run per protocol:

| Path | Source children | Live children | Result |
|---|---|---|---|
| `.../assigntask_manager` (single child) | `metaData` only | `metaData` only, same properties as source (`STATIC_ASSIGNEE=admin` etc.) | **Clean** |
| `.../assigntask_finance` (single child) | `metaData` only | `metaData` only, same properties as source (`STATIC_ASSIGNEE=admin` etc.) | **Clean** |
| `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form` | Exactly one `adaptiveFormEmbed` | HTTP 200, unaffected by this fix pass | **Clean — no stacking** |

**No purge was necessary.** Instance tree matches source exactly for both nodes touched by this fix
pass; the fix-pass-1 orphan purge (`employeeDetailsFragment/fd:rules`+`fd:events`) and every
subsequent clean sweep continue to hold.

### 15.5 Verdict & gate — redeploy (fix pass 8 / D10)

**BUILD SUCCESS** (first attempt) + **confirmed deploy** (`Package installed in 588ms`, core bundle
Active, "Test Adaptive Form" page HTTP 200) + **0 failed unit tests** (112/112, unaffected by this
content-only fix pass) + **D10 fix verified live on both design model and regenerated runtime model,
no drift** + **instance matches source, zero orphans**.

### GATE RESULT: **PASS**

Carried-forward flags (unchanged, still non-blocking): forms-service coverage 76.6% (pre-existing gap,
no new core Java classes this cycle), the two documented pre-production follow-ups (manager lookup,
finance-approver identity), the DoR-attachment email deviation, no local SMTP, D2 (OR-split branching)
and D4 (SubmissionDate) remain the recorded, human-actionable/documented follow-ups — unchanged,
untouched by this agent. **New note carried to Sentinel:** `STATIC_ASSIGNEE` is now the `admin` USER
(not the `administrators` group) on both Assign Task steps — a local-testing sidestep for the
claim/delegate `IllegalStateException`, not a production-ready assignee resolution; Sentinel's next
retest should attempt a real Approve/Reject click-through completion on both tasks now that the
claim/delegate blocker is removed.

### Run metrics — redeploy (fix pass 8 / D10)

- `time_taken_minutes`: 14.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~3,000 (reasoning about the pre-flight check, build result interpretation, this
    section's write-up)
  - `read`: ~13,000 (groundsmith.md fix-pass-8 section, the full committed workflow model
    `.content.xml`, `.aem-forms-config.yaml`, prior report tail for section-numbering/format
    precedent, full build log grep passes)
  - `write`: ~3,000 (this report section)
  - `other`: ~9,500 (1 full mvn build's tool-call overhead, bundle/CSRF/metaData/runtime-model
    live-verification calls, HTTP 200 page check)
  - `total`: ~28,500

### Handoff YAML — redeploy (fix pass 8 / D10)

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
time_taken_minutes: 14.0
tokens_consumed:
  cli_text: 3000
  read: 13000
  write: 3000
  other: 9500
  total: 28500
build: BUILD SUCCESS
build_attempts: 1
command: "mvn clean install -PautoInstallSinglePackage"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps: PASS, ui.apps.structure: PASS, ui.content: PASS, ui.config: PASS, all: PASS, ui.frontend: PASS, it.tests: PASS, dispatcher: PASS, ui.tests: PASS }
unit_tests: { run: 112, passed: 112, failed: 0, coverage_pct: 76.6 }
static_analysis: { warnings: 1, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip", type: dispatcher-config, installed: false }
  - { name: "aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar", type: test-jar, installed: false }
deploy_confirmed: true
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true }
design_model_fixes_verified:
  employee_training_request_approval:
    assigntask_manager: { static_assignee: "admin", process_participant_type: "static", form_type: "READ_ONLY_AF", drift_from_administrators: false }
    assigntask_finance: { static_assignee: "admin", process_participant_type: "static", form_type: "READ_ONLY_AF", drift_from_administrators: false }
runtime_model_sync:
  employee-training-request-approval: { design_path: "/conf/global/settings/workflow/models/employee-training-request-approval", runtime_path: "/var/workflow/models/employee-training-request-approval", generated: true, verified_live: true, static_assignee_both_steps: "admin" }
deploy_integrity: { instance_matches_source: true, orphans_purged: [] }
known_unfixed:
  - { item: "D2 - OR-split branching (orsplit_manager/orsplit_finance) real Approve/Reject routing conditions", owner: "human via Workflow Editor UI", attempted_by_forgemaster: false }
  - { item: "D4 - SubmissionDate", owner: "documented follow-up", attempted_by_forgemaster: false }
  - { item: "STATIC_ASSIGNEE=admin is a local-testing sidestep, not a production-ready assignee resolution", owner: "pre-production follow-up (real manager/finance-approver lookup)", attempted_by_forgemaster: false }
report: ".claude/agents/runs/2026-07-22-employee-training-request/deployment/code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (retest real Approve/Reject click-through completion on both Assign Task steps now that the claim/delegate blocker is removed; D2/D4 remain documented open items)
```

---

## 16. Redeploy — fix pass 10 (D13 guide marker: Document-of-Record "Not a valid Adaptive Form")

**Trigger:** Fix pass 10 (formwright) remediated **D13** — the OOTB `AFtoDORStep` threw *"Not a valid
Adaptive Form"* and the Document-of-Record PDF never generated. Root cause was decompiled precisely:
`AFtoDORStep.java:125` gates validity on the form page's `jcr:content` carrying a String property
`guide="1"` (the legacy Foundation Adaptive-Form marker), which Core Components AF pages do not emit —
so fix-pass-9's `dorType none→generate` tweak had zero effect (the gate is independent of `dorType`).
Formwright added the `guide="1"` marker to both form pages' `jcr:content` (and set the DoR page's
guideContainer `dorType` `none→generate`). This section is the rebuild + redeploy of record that makes
that source change permanent on the instance. Full defect detail: `implementation/formwright.md`
§"Fix pass 10".

### 16.1 Pre-flight

- AEM author reachable: `GET /libs/granite/core/content/login.html` → HTTP 200.
- Toolchain: Maven 3.9.16, build JVM **Java 11.0.31** (confirmed via `mvn -version`; a stray `java` on
  PATH resolves to 21 — expected, does not affect the Maven build JVM, same as every prior deploy).
- Confirmed the authored fixes present in source **before** building (no re-authoring by this agent):
  1. `employee-training-request/.content.xml` — `jcr:content` carries `guide="1"`; guideContainer
     `dorType="generate"`.
  2. `employee-training-request-dor/.content.xml` — `jcr:content` carries `guide="1"`; guideContainer
     `dorType="generate"` (was `none`).
- Did **not** touch D2 (OR-split branching), D4 (SubmissionDate), or the rejection-reason capture —
  all remain the documented, human-actionable/deferred follow-ups.

### 16.2 Build & deploy (redeploy of record)

- **Command:** `mvn clean install -PautoInstallSinglePackage`
- **Result:** **BUILD SUCCESS** on the **first attempt** — no root-cause fixing needed this cycle; the
  `aem.sdk.api` pin (`2026.6.26908...`) remains correct.
- **Total time:** 4 min 00 s. Finished `2026-07-27T18:24:39+05:30`.
- **Reactor Summary:** all 11 modules SUCCESS (core, ui.apps.structure, ui.apps, ui.content, ui.config,
  all, ui.frontend, it.tests, dispatcher, ui.tests, root pom) — 11/11 green. 1 pre-existing compiler
  warning (`GeneratePDFServlet.java` deprecated API, unrelated, non-blocking).
- **Unit tests:** aggregated from surefire output — **112 run, 0 failures, 0 errors, 0 skipped** (same
  suite as every prior deploy; this is a content-only marker fix, no new/changed Java).
- **Deploy landed:** `Installing aem-adaptive-forms-agents.all (...) to
  http://localhost:4502/crx/packmgr/service.jsp` → `Package imported.` → `Package installed in 700ms.`
  Package-manager `lastUnpacked` advanced this run: `all` = `1785156825824`, `ui.content` =
  `1785156842385` (≈`2026-07-27T18:24 IST`, timestamped to this build run, not a stale prior install).
- **Bundle state:** instance-wide `"738 bundles in total - all 738 bundles active."` — `core` Active.

### 16.3 Live verification — D13 fix + regression (post-redeploy)

The critical concern was that the package deploy might strip the `guide` marker or that the marker
addition might break normal rendering. Both were verified live:

| # | Check | Endpoint | Result |
|---|---|---|---|
| 1 | `guide="1"` retained on the interactive form after deploy | `GET .../employee-training-request/jcr:content.json` | `"guide":"1"` **present** |
| 2 | `guide="1"` retained on the DoR page after deploy | `GET .../employee-training-request-dor/jcr:content.json` | `"guide":"1"` **present** |
| 3 | Form still renders (marker did not break rendering) | `GET .../employee-training-request/jcr:content/guideContainer.model.json` | **HTTP 200** |
| 4 | Form page loads | `GET .../employee-training-request.html` | **HTTP 200** |
| 5 | DoR page loads | `GET .../employee-training-request-dor.html` | **HTTP 200** |
| 6 | guideContainer `dorType` (CC-idiomatic auto-DoR) live | `GET .../employee-training-request/jcr:content/guideContainer.json` | `"dorType":"generate"` |
| 7 | Embedded "Test Adaptive Form" page still live | `GET /content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html` | **HTTP 200** |
| 8 | Workflow runtime unchanged (no drift — this pass touched only form pages) | `GET /var/workflow/models/employee-training-request-approval.json` | **HTTP 200** (unchanged; no workflow artifact was in this fix pass) |

The `guide="1"` marker survived the package install on **both** pages, and its addition did **not**
break rendering — the form's `guideContainer.model.json` and both page `.html`s all return HTTP 200,
exactly as formwright's analysis predicted. The final proof (the DoR PDF actually generating through
`AFtoDORStep` on a real workflow run) is Sentinel's to exercise at retest; this agent confirms the
marker and DoR config are live on the instance.

### 16.4 Deploy integrity — orphan/stale-node sweep (step 3b)

This fix pass changed only `jcr:content` **properties** (`guide`) and a guideContainer attribute
(`dorType`) — no panel re-authoring, no node rename/removal — so no orphan risk was introduced, but the
sweep was run per protocol:

| Path | Source children | Live children | Result |
|---|---|---|---|
| `.../employee-training-request/.../guideContainer` (8 nodes) | `formTitle, employeeDetailsFragment, trainingRequestDetailsPanel, justificationPanel, attachmentsPanel, declarationFragment, approvalInformationPanel, actionsPanel` | Same 8, same names | **Clean** |
| `.../employee-training-request/jcr:content` | `guide="1"`, guideContainer child | Same — `guide="1"` present, no stray duplicate props | **Clean** |
| `.../test-adaptive-form/.../root/container` | Exactly one `adaptiveFormEmbed` | HTTP 200, one embed, unaffected by this fix pass | **Clean — no stacking** |

**No purge was necessary.** Instance tree matches source exactly; every prior clean sweep (incl. the
fix-pass-1 `employeeDetailsFragment/fd:rules`+`fd:events` purge) continues to hold.

### 16.5 Verdict & gate — redeploy (fix pass 10 / D13)

**BUILD SUCCESS** (first attempt) + **confirmed deploy** (`Package installed in 700ms`, `lastUnpacked`
advanced this run, all 738 bundles Active) + **0 failed unit tests** (112/112) + **`guide="1"` verified
live-retained on BOTH form pages post-deploy** + **form still renders (guideContainer.model.json +
both page `.html` all HTTP 200)** + **workflow runtime unchanged (no drift)** + **instance matches
source, zero orphans**.

### GATE RESULT: **PASS**

Carried-forward flags (unchanged, still non-blocking): forms-service coverage 76.6% (pre-existing gap,
no new core Java this cycle); the two pre-production follow-ups (manager lookup, finance-approver
identity); DoR-attachment email deviation; no local SMTP. **Documented open items left untouched by
this agent, per instruction:** D2 (OR-split branching), D4 (SubmissionDate), rejection-reason capture.
The genuine end-to-end DoR PDF generation through `AFtoDORStep` is Sentinel's to verify at retest now
that the `guide="1"` validity marker is live.

### Run metrics — redeploy (fix pass 10 / D13)

- `time_taken_minutes`: 11.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~3,000
  - `read`: ~14,000 (formwright.md fix-pass-10 context, prior report tail for section/format precedent,
    both form `.content.xml` grep, `.aem-forms-config.yaml`)
  - `write`: ~3,000 (this report section)
  - `other`: ~8,500 (1 full mvn build's tool-call overhead, curl guide-marker/model.json/page/bundle/
    package-manager/orphan-sweep live checks)
  - `total`: ~28,500

### Handoff YAML — redeploy (fix pass 10 / D13)

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
time_taken_minutes: 11.0
tokens_consumed:
  cli_text: 3000
  read: 14000
  write: 3000
  other: 8500
  total: 28500
build: BUILD SUCCESS
build_attempts: 1
command: "mvn clean install -PautoInstallSinglePackage"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps: PASS, ui.apps.structure: PASS, ui.content: PASS, ui.config: PASS, all: PASS, ui.frontend: PASS, it.tests: PASS, dispatcher: PASS, ui.tests: PASS }
unit_tests: { run: 112, passed: 112, failed: 0, coverage_pct: 76.6 }
static_analysis: { warnings: 1, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip", type: dispatcher-config, installed: false }
  - { name: "aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar", type: test-jar, installed: false }
deploy_confirmed: true
d13_guide_marker_verified_live:
  employee-training-request: { jcr_content_guide: "1", guideContainer_dorType: "generate", model_json_http: 200, page_html_http: 200 }
  employee-training-request-dor: { jcr_content_guide: "1", guideContainer_dorType: "generate", page_html_http: 200 }
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true }
workflow_model_sync: { runtime_path: "/var/workflow/models/employee-training-request-approval", http: 200, changed_this_pass: false, note: "fix pass 10 touched only form pages; no workflow drift" }
deploy_integrity: { instance_matches_source: true, orphans_purged: [] }
known_unfixed:
  - { item: "D2 - OR-split branching (orsplit_manager/orsplit_finance) real Approve/Reject routing", owner: "human via Workflow Editor UI", attempted_by_forgemaster: false }
  - { item: "D4 - SubmissionDate", owner: "documented follow-up", attempted_by_forgemaster: false }
  - { item: "rejection-reason capture", owner: "documented follow-up", attempted_by_forgemaster: false }
report: ".claude/agents/runs/2026-07-22-employee-training-request/deployment/code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (verify DoR PDF now generates end-to-end through AFtoDORStep with the guide="1" validity marker live; D2/D4/rejection-reason remain documented open items)
```

