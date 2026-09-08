# Code Quality / Deployment Report — Forgemaster (DEPLOY)

**Run:** 2026-07-28-employee-training-request
**Latest pass:** FIX pass 15 / SENTINEL-D15-03 (this section) — see history below for prior passes.
**Phase:** DEPLOY (build + deploy of record) · pipeline `formwright → groundsmith → assembler → forgemaster → sentinel`
**Date:** 2026-07-28

---

## ★ CURRENT DEPLOY — Fix pass 15 / SENTINEL-D15-03 (authoritative)

**Change deployed:** config-only, `ui.content` —
`conf/global/settings/workflow/models/employee-training-request-approval/.content.xml`.
On BOTH Assign Task nodes (`assigntask_manager`, `assigntask_finance`) the flat
`INPUT_DATAXML="data.xml"` + `OUTPUT_DATAXML="data.xml"` were RESTORED (fix pass 11 had removed them;
they are the only property `FormsDashboardPrefillServiceImpl` reads to locate a fresh task's payload).
`INPUT_DATAJSON`/`OUTPUT_DATAJSON`/`icDataSourceType=PROVIDEDATADOCUMENT` kept; colon-prefixed
`*_COMBINED_*` props remain absent in source; `CONTENT_TYPE` stays JSON. No core/Java change this pass.

### Build verdict

| Field | Value |
|---|---|
| Verdict | **BUILD SUCCESS** |
| Command | `mvn clean install -PautoInstallSinglePackage` (full reactor, foreground) |
| Reactor total time | 04:19 min |
| AEM target | http://localhost:4502 (author, admin/admin) — confirmed reachable (HTTP 200) before deploy |
| Toolchain | Java 11+ / Maven 3.3.9+ (enforcer satisfied; JAVA_TOOL_OPTIONS Windows-ROOT truststore) |
| Finished | 2026-07-28T17:59:37+05:30 |

### Per-module results (all 11 SUCCESS)

| # | Module | Result | Time |
|---|--------|--------|------|
| 1 | aem-adaptive-forms-agents (reactor root) | SUCCESS | 0.847 s |
| 2 | core | SUCCESS | 39.368 s |
| 3 | ui.apps.structure | SUCCESS | 1.700 s |
| 4 | ui.apps | SUCCESS | 19.866 s |
| 5 | **ui.content** (carries the workflow-model fix) | SUCCESS | 27.040 s |
| 6 | ui.config | SUCCESS | 1.758 s |
| 7 | all (aggregate deployable) | SUCCESS | 1:39 min |
| 8 | ui.frontend | SUCCESS | 2.355 s |
| 9 | it.tests | SUCCESS | 34.683 s |
| 10 | dispatcher | SUCCESS | 1.463 s |
| 11 | ui.tests (Cypress) | SUCCESS | 25.824 s |

### Unit tests & coverage

| Metric | Value |
|---|---|
| Tests run | **120** |
| Passed | 120 |
| Failed / Errors / Skipped | 0 / 0 / 0 |
| Coverage — Forms service classes | ~85.8% instruction (unchanged; no Java change this pass) — meets ≥80% target |

Full `core` unit-test suite green (120/120). No test touches the workflow design-model XML (a JCR
content artifact); the fix is validated by the live-instance readback below and Sentinel's functional pass.

### Deployment artifacts (name + version)

| Artifact | Type | Version | Installed |
|---|---|---|---|
| `aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip` | content-package (aggregate) | 1.0.0-SNAPSHOT | yes — `Package installed in 1071ms`, `lastUnpacked` advanced |
| `aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | yes (embedded in all) |
| `aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | yes (embedded in all) — carries the fix |
| `aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | yes (embedded in all) |
| `aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip` | content-package (structure) | 1.0.0-SNAPSHOT | yes (embedded in all) |
| `aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar` | OSGi bundle | 1.0.0.SNAPSHOT | yes — bundle **Active** on localhost:4502 |
| `aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip` | content-package (dispatcher) | 1.0.0-SNAPSHOT | built |
| `aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT(-jar-with-dependencies).jar` | test bundle | 1.0.0-SNAPSHOT | built |
| `com.adobe.cq.cloud.testing.ui.cypress.tests-0.0.1-SNAPSHOT-ui-test-docker-context.tar.gz` | UI-test artifact | 0.0.1-SNAPSHOT | built |

### Deploy confirmation

- `BUILD SUCCESS`; the `all` package installed via `crx/packmgr/service.jsp` — `Package installed in 1071ms`.
- `all` package `lastUnpacked` advanced to this run (`installed:true`, `lastUnpackedBy=admin`).
- Core bundle `aem-adaptive-forms-agents.core` (v1.0.0.SNAPSHOT) = **Active**.
- "Test Adaptive Form" page `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form` returns **HTTP 200** (form embed live).

### /var runtime regeneration + readback (mandatory this pass)

A plain deploy updates `/conf` but does NOT regenerate `/var`. After deploy the runtime
`/var/workflow/models/employee-training-request-approval` was regenerated from the deployed `/conf`
via `ModelGenerateServlet` (`POST {model}/jcr:content.generate.json`, CSRF token + Referer + Origin) →
`{"msg":"Model successfully generated."}`. **Resulting runtime version: 1.35.**

Live `/var` readback — BOTH Assign Task nodes now carry:

| Property | assigntask_manager | assigntask_finance |
|---|---|---|
| `INPUT_DATAXML` | `data.xml` ✓ | `data.xml` ✓ |
| `OUTPUT_DATAXML` | `data.xml` ✓ | `data.xml` ✓ |
| `INPUT_DATAJSON` / `OUTPUT_DATAJSON` | `data.xml` / `data.xml` | `data.xml` / `data.xml` |
| `icDataSourceType` | `PROVIDEDATADOCUMENT` | `PROVIDEDATADOCUMENT` |
| `FORM_RESOLUTION` | `PATH` ✓ | `PATH` ✓ |
| `AF_PATH` | `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request` ✓ | same ✓ |
| `INPUT_COMBINED_*` / `OUTPUT_COMBINED_*` | **NONE** ✓ | **NONE** ✓ |

### Deploy integrity — stale-property reconciliation (rule 7 / step 3b)

The `/conf` design model filter root is `mode="update"`
(`ui.content/.../META-INF/vault/filter.xml` line 6). On the **first** post-deploy readback the
`assigntask_manager` node did **not** match source: it was missing the flat
`INPUT_DATAXML`/`OUTPUT_DATAXML`/flat attachments and still carried stale
`INPUT_COMBINED_DATAXML`/`OUTPUT_COMBINED_DATAXML`/`INPUT_COMBINED_FORM_ATTACHMENTS`/`OUTPUT_COMBINED_FORM_ATTACHMENTS`
= `FOLDER_PAYLOAD:…` (colon-token props — the D6 `RELATIVE_PLOAD` crash trigger), left over from an
earlier fix pass. `mode="update"` adds/updates package properties but **never deletes** instance
properties absent from the package, so the source edit never removed them and the first `/var`
regeneration (v1.34) inherited the wrong manager config. The `assigntask_finance` node was clean.

**Purge/reconcile (make instance match the already-correct source — source NOT edited):** Sling POST
to `/conf/.../flow/assigntask_manager/metaData` (CSRF + Referer + Origin, HTTP 200): set flat
`INPUT_DATAXML`/`OUTPUT_DATAXML="data.xml"` + `INPUT_FORM_ATTACHMENTS`/`OUTPUT_FORM_ATTACHMENTS="attachments"`,
and `@Delete` the four stale `*_COMBINED_*` props. Re-verified `/conf` manager now matches source
(flat props, no COMBINED), regenerated `/var` (→ **v1.35**), and confirmed BOTH runtime nodes match
source (table above). Stale/orphan props removed: `INPUT_COMBINED_DATAXML`, `OUTPUT_COMBINED_DATAXML`,
`INPUT_COMBINED_FORM_ATTACHMENTS`, `OUTPUT_COMBINED_FORM_ATTACHMENTS` on `assigntask_manager`.

### Verdict & gate — **PASS**

BUILD SUCCESS + confirmed deploy (core bundle Active, `all` installed) + embed page HTTP 200 +
0 failed unit tests + `/var` regenerated to v1.35 with BOTH Assign Task nodes reconciled to source
(flat `INPUT_DATAXML`/`OUTPUT_DATAXML="data.xml"`, no COMBINED props) → **gate PASS**. Cleared for
Sentinel's end-to-end render proof.

---

## HISTORY — Fix pass 12 / D15 (superseded by the section above)

**Delivery type:** FIX / REMEDIATION (Fix pass 12 / D15) — workflow design-model correction
**Date:** 2026-07-28

---

## Build verdict

| Field | Value |
|---|---|
| Verdict | **BUILD SUCCESS** |
| Command | `mvn clean install -PautoInstallSinglePackage` |
| Reactor total time | 03:29 min |
| AEM target | http://localhost:4502 (author, admin/admin) — confirmed reachable (HTTP 200) before deploy |
| Toolchain | Java 21.0.9 LTS, Apache Maven 3.9.16 (enforcer: Java 11+ / Maven 3.3.9+ satisfied) |
| Finished | 2026-07-28T11:13:26+05:30 |

**Change deployed:** the corrected workflow **design** model
`ui.content/.../conf/global/settings/workflow/models/employee-training-request-approval/.content.xml`
(both Assign Task nodes `assigntask_manager` + `assigntask_finance` with the crashing
`INPUT_COMBINED_DATAJSON` / `OUTPUT_COMBINED_DATAJSON="RELATIVE_PLOAD:data.xml"` removed; flat
`INPUT_DATAJSON`/`OUTPUT_DATAJSON="data.xml"` + `icDataSourceType="PROVIDEDATADOCUMENT"` retained).
The `/var` runtime model was regenerated live on the SDK by Groundsmith and is not a build artifact.

---

## Per-module results

| # | Module | Result | Time |
|---|--------|--------|------|
| 1 | AEM Adaptive Forms Agents (reactor root) | SUCCESS | 0.785 s |
| 2 | core | SUCCESS | 32.589 s |
| 3 | ui.apps.structure (Repository Structure) | SUCCESS | 1.677 s |
| 4 | ui.apps | SUCCESS | 10.759 s |
| 5 | **ui.content** (carries the workflow model fix) | SUCCESS | 13.688 s |
| 6 | ui.config | SUCCESS | 1.275 s |
| 7 | all (aggregate deployable) | SUCCESS | 1:26 min |
| 8 | ui.frontend | SUCCESS | 1.920 s |
| 9 | it.tests | SUCCESS | 29.976 s |
| 10 | dispatcher | SUCCESS | 0.851 s |
| 11 | ui.tests (Cypress) | SUCCESS | 24.495 s |

All 11 reactor modules built successfully.

---

## Unit tests & coverage

| Metric | Value |
|---|---|
| Tests run | **112** |
| Passed | 112 |
| Failed | 0 |
| Errors | 0 |
| Skipped | 0 |
| Coverage — Forms service classes (submit + prefill) | **85.8%** instruction (JaCoCo 0.8.12) — **meets ≥80% target** |
| Coverage — overall core module | 75.7% instruction |

All unit tests executed in the `core` module reactor build passed. No test touches the workflow
design-model XML (a JCR content artifact); the fix is validated by the live-instance readback below
and by Sentinel's functional pass. Integration (`it.tests`) and UI (`ui.tests`) modules packaged with
"No tests to run" during the build (executed by Sentinel, not in this reactor pass).

---

## Static analysis

| Check | Result |
|---|---|
| Compiler ERRORS | 0 |
| Compiler warnings (deprecation / unchecked) | none surfaced |
| `[WARNING]` lines (build/plugin advisories) | 31 — informational (plugin/version advisories); no compilation or bytecode warnings |
| SpotBugs / Checkstyle / PMD | none configured in the reactor |

No new static-analysis findings introduced by this fix (a two-attribute deletion in a JCR
`.content.xml` — no Java changed).

---

## Deployment artifacts

| Artifact (name + version) | Type | Installed to AEM |
|---|---|---|
| `aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip` | content-package (aggregate) | ✅ installed in 785ms; `lastUnpacked` 2026-07-28 |
| `aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip` | content-package | ✅ embedded in `all`; `lastUnpacked` 2026-07-28 |
| `aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip` | content-package (structure) | ✅ embedded in `all` |
| `aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip` | content-package (**carries the workflow model fix**) | ✅ embedded in `all`; `lastUnpacked` 2026-07-28 |
| `aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip` | content-package | ✅ embedded in `all`; `lastUnpacked` 2026-07-28 |
| `aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip` | content-package (dispatcher) | built (not installed to author — dispatcher artifact) |
| `aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar` | OSGi bundle | ✅ embedded in `all` |

Test-only artifacts produced (not deployed): `aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar`
(+ `-jar-with-dependencies`), `com.adobe.cq.cloud.testing.ui.cypress.tests-0.0.1-SNAPSHOT-ui-test-docker-context.tar.gz`.

---

## Deploy confirmation

- Package-manager install log evidence:
  `Installing aem-adaptive-forms-agents.all (…/all/target/aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip) to http://localhost:4502/crx/packmgr/service.jsp` → **`Package installed in 785ms.`**
- Install signal — package `lastUnpacked` advanced to **2026-07-28** for `all`, `ui.apps`, `ui.content`, `ui.config` (verified via `/crx/packmgr/list.jsp`).
- "Test Adaptive Form" page live: `GET /content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html` → **HTTP 200** (unchanged by this fix; assembly not re-run this pass).
- `/var` runtime workflow model live: `GET /var/workflow/models/employee-training-request-approval.json` → **HTTP 200**.

---

## Deploy integrity — instance matches source

Filter root `/conf/global/settings/workflow/models/employee-training-request-approval` is `mode="update"`.
The fix removes **two attributes on two existing nodes** (no node was added, renamed, or moved), so no
orphan/stale-node duplication is possible; update mode reconciles the covered nodes' properties.

Live-instance readback of the deployed `/conf` design model (`jcr:content.infinity.json`) vs source:

| Property | Source | Live instance | Match |
|---|---|---|---|
| `INPUT_COMBINED_DATAJSON` / `OUTPUT_COMBINED_DATAJSON` | absent (removed) | **0 occurrences** | ✅ |
| `INPUT_DATAJSON` / `OUTPUT_DATAJSON="data.xml"` | present, both nodes | **2 occurrences** | ✅ |
| `icDataSourceType="PROVIDEDATADOCUMENT"` | present, both nodes | **2 occurrences** | ✅ |

**Instance matches source. No orphan / stale / duplicate nodes. Clean — no purge required.**
(The `COMBINED_DATAJSON` tokens remaining in the source `.content.xml` are inside `<!-- … -->`
documentation comments — the "do not re-introduce" fix notes — not live attributes; they do not
serialize into the JCR.)

---

## Verdict & gate

| Gate criterion | Result |
|---|---|
| BUILD SUCCESS | ✅ |
| Confirmed deploy to localhost:4502 (package installed + lastUnpacked advanced) | ✅ |
| Embedded "Test Adaptive Form" page live (HTTP 200) | ✅ |
| No failed unit tests (112/112 pass) | ✅ |
| Instance matches source (no unresolved orphan/duplicate nodes) | ✅ |

### GATE RESULT: **PASS**

Handing back to `aem-forms-program-agent` to run **Sentinel** (testing) — gate is PASS.

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
build: BUILD SUCCESS
command: "mvn clean install -PautoInstallSinglePackage"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps.structure: PASS, ui.apps: PASS, ui.content: PASS, ui.config: PASS, all: PASS, ui.frontend: PASS, it.tests: PASS, dispatcher: PASS, ui.tests: PASS }
unit_tests: { run: 112, passed: 112, failed: 0, skipped: 0, coverage_forms_services_pct: 85.8, coverage_core_pct: 75.7 }
static_analysis: { errors: 0, warnings: 31, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }   # carries workflow model fix
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip", type: content-package, installed: false }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
deploy_confirmed: true
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true }
deploy_integrity: { instance_matches_source: true, orphans_purged: [] }
report: ".claude/agents/runs/2026-07-28-employee-training-request/deployment/code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (testing)
```

---
---

# REDEPLOY — Fix pass 13 / SENTINEL-D15-01 (workflow AF_PATH → DAM guide-asset)

**Redeploy date:** 2026-07-28 (14:33 build finished · 14:38 /var regenerated)
**Trigger:** SENTINEL-D15-01 — filled review form not visible; vendor NPE
`Cannot invoke Resource.getParent() because parentResource is null` in
`CommonUtils.getFormRenderPathForActiveWorkItem` (formview.jsp) when `AF_PATH` was a
`/content/forms/af/...` authoring path.
**Change authored by Groundsmith (committed to source):** both Assign Task nodes in
`ui.content/.../conf/global/settings/workflow/models/employee-training-request-approval/.content.xml`
(`assigntask_manager` + `assigntask_finance`) — and the canonical
`assign-task-to-admin/.content.xml` — keep `sling:resourceType=fd/workflow/components/dashboard/afParticipantStep`
+ `PROCESS=com.adobe.fd.workspace.step.service.AssignFormStep`, with `AF_PATH` repointed to the DAM
guide-asset path `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request`
and `FORM_RESOLUTION` changed to `PATH`. No `com.adobe.fd.workflow.aem.process.AssignTaskStep` present
(the earlier raced edit that used that unregistered class was reverted) — confirmed in source and on
the deployed instance.

## Build verdict

| Field | Value |
|---|---|
| Verdict | **BUILD SUCCESS** |
| Command | `mvn clean install -PautoInstallSinglePackage` |
| Reactor total time | 02:28 min |
| AEM target | http://localhost:4502 (author, admin/admin) — reachable (HTTP 200) before deploy |
| Finished | 2026-07-28T14:33:12+05:30 |

## Per-module results (11/11 SUCCESS)

| Module | Result | Time |
|--------|--------|------|
| AEM Adaptive Forms Agents (reactor root) | SUCCESS | 0.913 s |
| core | SUCCESS | 33.823 s |
| ui.apps.structure (Repository Structure) | SUCCESS | 1.860 s |
| ui.apps | SUCCESS | 11.061 s |
| **ui.content** (carries the workflow model fix) | SUCCESS | 10.312 s |
| ui.config | SUCCESS | 1.187 s |
| all (aggregate deployable) | SUCCESS | 50.721 s |
| ui.frontend | SUCCESS | 0.615 s |
| it.tests | SUCCESS | 17.959 s |
| dispatcher | SUCCESS | 0.592 s |
| ui.tests (Cypress) | SUCCESS | 15.821 s |

## Unit tests & coverage

| Metric | Value |
|---|---|
| Tests run | **112** (Failures 0, Errors 0, Skipped 0) |
| Result | ALL PASS |
| Coverage — Forms service classes | 85.8% instruction (unchanged; no Java touched — JCR-only change) — meets ≥80% target |

No test exercises the workflow design-model XML (a JCR content artifact); the fix is validated by the
live `/conf` + `/var` readback below and by Sentinel's functional re-verification.

## Static analysis

Compiler ERRORS: 0. `[WARNING]` lines: 31 (informational plugin/version advisories; no compilation or
bytecode warnings). SpotBugs/Checkstyle/PMD: none configured. No new findings (config-only JCR change).

## Deployment artifacts

| Artifact (name + version) | Type | Installed to AEM |
|---|---|---|
| `aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip` | content-package (aggregate) | ✅ `Package installed in 1194ms.` |
| `aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip` | content-package | ✅ embedded in `all` |
| `aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip` | content-package (structure) | ✅ embedded in `all` |
| `aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip` | content-package (**carries the workflow model fix**) | ✅ embedded in `all` |
| `aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip` | content-package | ✅ embedded in `all` |
| `aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip` | content-package (dispatcher) | built (not installed to author) |
| `aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar` | OSGi bundle | ✅ embedded in `all` |

Test-only (not deployed): `aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar`,
`com.adobe.cq.cloud.testing.ui.cypress.tests-...-ui-test-docker-context.tar.gz`.

## Deploy confirmation

- Install evidence: `Installing aem-adaptive-forms-agents.all (…/all/target/…all-1.0.0-SNAPSHOT.zip) to
  http://localhost:4502/crx/packmgr/service.jsp` → **`Package installed in 1194ms.`**
- Deployed `/conf` design-model readback (both Assign Task nodes):
  `sling:resourceType=fd/workflow/components/dashboard/afParticipantStep`,
  `PROCESS=com.adobe.fd.workspace.step.service.AssignFormStep`, `FORM_RESOLUTION=PATH`,
  `AF_PATH=/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request`. ✅

## /var runtime regeneration + readback (step 2 — mandatory)

- **generate.json response:**
  `{"msg":"Model successfully generated.","modelPath":"/var/workflow/models/employee-training-request-approval"}`
  (POST `…/jcr:content.generate.json` with CSRF token + Referer/Origin).
- **Runtime version: `1.32`** · `lastSynced` 2026-07-28T14:38:42+05:30 ·
  `cq:generatingPage=/conf/global/settings/workflow/models/employee-training-request-approval/jcr:content`.
- **Runtime node readback** (model `.json` API — reports each step's executable `PROCESS` FQCN + metaData):

| Runtime node | PROCESS | FORM_RESOLUTION | AF_PATH |
|---|---|---|---|
| node2 "Manager Approval" | `com.adobe.fd.workspace.step.service.AssignFormStep` | `PATH` | `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request` |
| node4 "Finance Manager Approval" | `com.adobe.fd.workspace.step.service.AssignFormStep` | `PATH` | `/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request` |

Both Assign Task steps confirmed as `AssignFormStep` (the registered runtime step — NOT the reverted,
unregistered `AssignTaskStep`), `FORM_RESOLUTION=PATH`, DAM guide-asset `AF_PATH`, `FORM_TYPE=READ_ONLY_AF`,
`icDataSourceType=PROVIDEDATADOCUMENT`. `/conf` and `/var` are now in sync.
(The separate DoR step node6 keeps its own `AF_PATH=/content/forms/af/...` by design — out of D15-01 scope.)

## Deploy integrity — instance matches source

Filter root `/conf/global/settings/workflow/models/…` is `mode="update"`. This fix changes attributes on
existing nodes only (no node added, renamed, or moved) → no orphan/duplicate possible. Live `/conf`
readback equals source on both nodes (table above). **Clean — no orphans, no purge required.**

## Verdict & gate — REDEPLOY

| Gate criterion | Result |
|---|---|
| BUILD SUCCESS | ✅ |
| Confirmed deploy to localhost:4502 (`Package installed in 1194ms`) | ✅ |
| No failed unit tests (112/112 pass) | ✅ |
| `/var` regenerated from `/conf` + readback proves fix on both nodes (runtime v1.32) | ✅ |
| Instance matches source (no orphan/duplicate nodes) | ✅ |

### GATE RESULT: **PASS** — handing back to `aem-forms-program-agent` to run **Sentinel** re-verification.

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
fix_pass: 13
sentinel_ref: SENTINEL-D15-01
build: BUILD SUCCESS
command: "mvn clean install -PautoInstallSinglePackage"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps.structure: PASS, ui.apps: PASS, ui.content: PASS, ui.config: PASS, all: PASS, ui.frontend: PASS, it.tests: PASS, dispatcher: PASS, ui.tests: PASS }
unit_tests: { run: 112, passed: 112, failed: 0, skipped: 0, coverage_forms_services_pct: 85.8 }
static_analysis: { errors: 0, warnings: 31, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }   # Package installed in 1194ms
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }   # carries workflow model fix
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip", type: content-package, installed: false }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
deploy_confirmed: true
var_runtime:
  generate_json: "Model successfully generated."
  runtime_version: "1.32"
  last_synced: "2026-07-28T14:38:42+05:30"
  assigntask_manager: { process: "com.adobe.fd.workspace.step.service.AssignFormStep", form_resolution: PATH, af_path: "/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request" }
  assigntask_finance: { process: "com.adobe.fd.workspace.step.service.AssignFormStep", form_resolution: PATH, af_path: "/content/dam/formsanddocuments/aem-adaptive-form-agents/employee-training-request" }
deploy_integrity: { instance_matches_source: true, orphans_purged: [] }
report: ".claude/agents/runs/2026-07-28-employee-training-request/deployment/code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (SENTINEL-D15-01 re-verification)
```

---
---

# REBUILD — Fix pass 14 / SENTINEL-D15-02 (prefill defer-on-foreign-dataRef) — **BUILD FAILURE**

**Rebuild date:** 2026-07-28 (17:01 build finished)
**Trigger:** SENTINEL-D15-02 — all 5 prefill DataProviders
(BusinessRegistration, ContactUs, Complaint, Withdrawal, LifeInsurance) now defer (return `null`) on
foreign / work-item dataRefs via an `isForeignRequest(DataOptions)` guard, so the platform binds the
work-item `data.xml` on `fdtask://` requests instead of returning static prefill defaults. Core-only
change (`com.aem.forms.agents.core`); no ui.content / ui.config change.

## Build verdict

| Field | Value |
|---|---|
| Verdict | **BUILD FAILURE** |
| Command | `mvn clean install -PautoInstallSinglePackage` |
| Reactor total time | 34.271 s (aborted in `core`) |
| AEM target | http://localhost:4502 (reachable — HTTP 200 — but **nothing deployed**; reactor aborted before packaging/install) |
| Toolchain | Java 21.0.9, Apache Maven 3.9.16 |
| Finished | 2026-07-28T17:01:42+05:30 |
| Failing goal | `maven-surefire-plugin:2.22.1:test (default-test) @ aem-adaptive-forms-agents.core` — "There are test failures." |

## Per-module results

| Module | Result |
|--------|--------|
| AEM Adaptive Forms Agents (reactor root) | SUCCESS |
| **core** | **FAILURE** (1 unit-test failure) |
| ui.apps.structure | SKIPPED |
| ui.apps | SKIPPED |
| ui.content | SKIPPED |
| ui.config | SKIPPED |
| all (aggregate deployable) | SKIPPED |
| ui.frontend | SKIPPED |
| it.tests | SKIPPED |
| dispatcher | SKIPPED |
| ui.tests (Cypress) | SKIPPED |

Compilation succeeded (25 main + 17 test sources compiled; only a pre-existing `GeneratePDFServlet`
deprecation note). The failure is a **unit-test assertion**, not a compile error.

## Unit tests & coverage

| Metric | Value |
|---|---|
| Tests run | **120** (up from 112 — 8 new defer cases across the 5 prefill services) |
| Passed | 119 |
| **Failed** | **1** |
| Errors | 0 |
| Skipped | 0 |

### Prefill test outcomes (the 5 updated services)

| Prefill service test | Tests run | Result |
|---|---|---|
| BusinessRegistrationPrefillServiceTest | 15 | ✅ ALL PASS (incl. new `fdtask://` defer case) |
| ContactUsPrefillServiceTest | 7 | ✅ ALL PASS (incl. new `fdtask://` defer case) |
| ComplaintPrefillServiceTest | 6 | ✅ ALL PASS (incl. new `fdtask://` defer case) |
| WithdrawalPrefillServiceTest | 6 | ✅ ALL PASS (incl. new `fdtask://` defer case) |
| **LifeInsurancePrefillServiceTest** | 15 | ❌ **1 FAILURE** (the new `fdtask://` defer case passes; a **pre-existing** test was not reconciled) |

**Exact failure:**
```
LifeInsurancePrefillServiceTest.getPrefillData_nonDraftDataRef_isIgnored_authenticatedCrmPathDegrades:320
  org.opentest4j.AssertionFailedError: expected: not <null>
```

**Root cause (regression introduced by the Fix pass 14 change, NOT a deploy/infra issue):**
The new `isForeignRequest(DataOptions)` guard in `LifeInsurancePrefillService` (returns `null` at the
top of `getPrefillData`, lines 182-184 / 455-471) classifies **any** dataRef that is not
`service://lifeInsurancePrefillService*` and not under `draftBasePath` as foreign. The pre-existing
test at `LifeInsurancePrefillServiceTest:313-321` feeds
`dataRef = "/content/forms/af/life-insurance-new-policy"` for an authenticated user and asserts
`assertNotNull(result)` (its old contract: "a non-draft dataRef is ignored → falls through to CRM →
degrades to valid JSON"). Under the new contract that dataRef is now foreign → the service returns
`null` → `assertNotNull` fails. The 4 other prefill services reconciled cleanly; only
`LifeInsurancePrefillServiceTest` has a stale case that contradicts the new defer-on-foreign-dataRef
behavior.

## Static analysis

Compiler ERRORS: 0. Warnings: 1 pre-existing deprecation note in `GeneratePDFServlet` (unchanged by
this fix). SpotBugs/Checkstyle/PMD: none configured.

## Deployment artifacts

**NONE produced / NONE deployed.** The reactor aborted in the `core` module before any package was
built or installed; the `all` aggregate and all content packages/bundles were SKIPPED. The instance
still holds the artifacts from the last successful build (Fix pass 13). The `com.aem.forms.agents.core`
bundle ACTIVE-state check was **not performed** (no new bundle was installed).

## Deploy confirmation

**Not deployed.** No `Package installed in …ms` log line; no `lastUnpacked` advance. Deploy did not occur.

## Verdict & gate — REBUILD (Fix pass 14)

| Gate criterion | Result |
|---|---|
| BUILD SUCCESS | ❌ BUILD FAILURE |
| All unit tests pass | ❌ 1 failure (LifeInsurancePrefillServiceTest) |
| Confirmed deploy to localhost:4502 | ❌ not deployed (reactor aborted) |
| core bundle ACTIVE | ❌ not verified (nothing installed) |

### GATE RESULT: **FAIL**

**Bounce to `groundsmith`** (owns the Fix pass 14 prefill change). Reconcile the stale pre-existing
test `LifeInsurancePrefillServiceTest.getPrefillData_nonDraftDataRef_isIgnored_authenticatedCrmPathDegrades`
with the new defer-on-foreign-dataRef contract (either update the test to expect `null` for a foreign
`/content/forms/af/...` dataRef, consistent with the other 4 services, or — if a non-draft authoring
path should still yield a degraded CRM prefill — narrow `isForeignRequest` accordingly and keep the 5
services consistent). Forgemaster does NOT edit source/tests (out of lane). Re-run the deploy gate after
Groundsmith's fix.

```yaml
agent: forgemaster
phase: DEPLOY
status: FAILED
fix_pass: 14
sentinel_ref: SENTINEL-D15-02
build: BUILD FAILURE
command: "mvn clean install -PautoInstallSinglePackage"
aem_target: "http://localhost:4502"
modules: { core: FAILURE, ui.apps.structure: SKIPPED, ui.apps: SKIPPED, ui.content: SKIPPED, ui.config: SKIPPED, all: SKIPPED, ui.frontend: SKIPPED, it.tests: SKIPPED, dispatcher: SKIPPED, ui.tests: SKIPPED }
unit_tests: { run: 120, passed: 119, failed: 1, skipped: 0 }
failing_test: "com.aem.forms.agents.forms.prefill.LifeInsurancePrefillServiceTest#getPrefillData_nonDraftDataRef_isIgnored_authenticatedCrmPathDegrades"
failure: "org.opentest4j.AssertionFailedError: expected: not <null> (LifeInsurancePrefillServiceTest.java:320)"
prefill_tests:
  BusinessRegistrationPrefillServiceTest: { run: 15, result: PASS }
  ContactUsPrefillServiceTest: { run: 7, result: PASS }
  ComplaintPrefillServiceTest: { run: 6, result: PASS }
  WithdrawalPrefillServiceTest: { run: 6, result: PASS }
  LifeInsurancePrefillServiceTest: { run: 15, result: FAIL, failures: 1 }
deployment_artifacts: []   # reactor aborted in core — no package built/installed
deploy_confirmed: false
core_bundle_active: not_verified   # nothing installed
gate_result: FAIL
bounce_to: groundsmith
next: groundsmith reconciles the stale LifeInsurance pre-existing test with the Fix pass 14 defer-on-foreign-dataRef contract; forgemaster re-runs the deploy gate after
```

---
---

# REBUILD (RE-RUN) — Fix pass 14 / SENTINEL-D15-02 (prefill defer-on-foreign-dataRef) — **BUILD SUCCESS**

**Rebuild date:** 2026-07-28 (build finished 2026-07-28T17:10:59+05:30)
**Trigger:** Re-run of the deploy gate after Groundsmith reconciled the single failing unit test that
blocked the previous (FAILED) build of this fix pass.
**Change since the FAILED build (test-only):** Groundsmith renamed the stale
`LifeInsurancePrefillServiceTest` case
`getPrefillData_nonDraftDataRef_isIgnored_authenticatedCrmPathDegrades` →
`getPrefillData_nonDraftForeignDataRef_defers`, which now asserts `null` for a foreign dataRef —
consistent with the 4 sibling prefill services — and removed the now-unused CRM stubs. **Production /
guard code is UNCHANGED** (the `isForeignRequest(DataOptions)` defer logic in all 5 prefill
DataProviders is identical to the previous build). Legitimate no-dataRef / own-`service://` on-load
prefill still asserts non-null → no regression.

## Build verdict

| Field | Value |
|---|---|
| Verdict | **BUILD SUCCESS** |
| Command | `mvn clean install -PautoInstallSinglePackage` |
| Reactor total time | 02:24 min |
| AEM target | http://localhost:4502 (author, admin/admin) — confirmed reachable (HTTP 200) before deploy |
| Toolchain | Java 21.0.9 LTS, Apache Maven 3.9.16 (enforcer: Java 11+ / Maven 3.3.9+ satisfied) |
| Finished | 2026-07-28T17:10:59+05:30 |

## Per-module results (11/11 SUCCESS)

| Module | Result | Time |
|--------|--------|------|
| AEM Adaptive Forms Agents (reactor root) | SUCCESS | 0.599 s |
| core | SUCCESS | 29.363 s |
| ui.apps.structure (Repository Structure) | SUCCESS | 1.371 s |
| ui.apps | SUCCESS | 11.118 s |
| ui.content | SUCCESS | 10.370 s |
| ui.config | SUCCESS | 1.269 s |
| all (aggregate deployable) | SUCCESS | 52.654 s |
| ui.frontend | SUCCESS | 0.626 s |
| it.tests | SUCCESS | 17.226 s |
| dispatcher | SUCCESS | 0.598 s |
| ui.tests (Cypress) | SUCCESS | 15.646 s |

## Unit tests & coverage

| Metric | Value |
|---|---|
| Tests run | **120** |
| Passed | **120** |
| Failed | **0** |
| Errors | **0** |
| Skipped | **0** |
| Coverage — Forms service classes (submit + prefill) | ~85.8% instruction (JaCoCo) — meets ≥80% target (no production code changed vs last SUCCESS build) |

**The previously-failing test is now resolved** — the whole reactor unit suite is green.

### Prefill test outcomes (all 5 services — the Fix pass 14 focus)

| Prefill service test | Tests run | Failures | Result |
|---|---|---|---|
| BusinessRegistrationPrefillServiceTest | 15 | 0 | ✅ ALL PASS (incl. `fdtask://` defer case) |
| ContactUsPrefillServiceTest | 7 | 0 | ✅ ALL PASS (incl. `fdtask://` defer case) |
| ComplaintPrefillServiceTest | 6 | 0 | ✅ ALL PASS (incl. `fdtask://` defer case) |
| WithdrawalPrefillServiceTest | 6 | 0 | ✅ ALL PASS (incl. `fdtask://` defer case) |
| **LifeInsurancePrefillServiceTest** | 15 | 0 | ✅ **ALL PASS** (renamed `getPrefillData_nonDraftForeignDataRef_defers` now asserts null; consistent with the 4 siblings) |

All 5 prefill classes are now consistent with the defer-on-foreign-dataRef contract. (Surefire prints
class-level summaries only; every prefill class reports `Failures: 0, Errors: 0`. The
`[Fatal Error] :1:1: Content is not allowed in prolog` lines in the log are benign — tests deliberately
feed non-XML into a parser to exercise degrade paths; the owning classes still report 0 failures.)

## Static analysis

Compiler ERRORS: 0. Warnings: 1 pre-existing deprecation note in `GeneratePDFServlet` (unchanged by
this fix). SlingFeature analyser: aggregated author/publish features 0 errors (info-level warnings on
user-aggregated features only). SpotBugs/Checkstyle/PMD: none configured. No new findings (test-only change).

## Deployment artifacts

| Artifact (name + version) | Type | Installed to AEM |
|---|---|---|
| `aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip` | content-package (aggregate) | ✅ `Package installed in 1064ms.`; `lastUnpacked` advanced (epoch 1785238825775) |
| `aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip` | content-package | ✅ embedded in `all`; `lastUnpacked` advanced (1785238830382) |
| `aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip` | content-package (structure) | ✅ embedded in `all` |
| `aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip` | content-package | ✅ embedded in `all`; `lastUnpacked` advanced (1785238838612) |
| `aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip` | content-package | ✅ embedded in `all`; `lastUnpacked` advanced (1785238830414) |
| `aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip` | content-package (dispatcher) | built (not installed to author — dispatcher artifact) |
| `aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar` | OSGi bundle (Bundle-SymbolicName `aem-adaptive-forms-agents.core`, v1.0.0.SNAPSHOT) | ✅ embedded in `all`; bundle **ACTIVE** (id 738, stateRaw 32) |

Test-only (not deployed): `aem-adaptive-forms-agents.it.tests-1.0.0-SNAPSHOT.jar`
(+ `-jar-with-dependencies`), `com.adobe.cq.cloud.testing.ui.cypress.tests-0.0.1-SNAPSHOT-ui-test-docker-context.tar.gz`.

## Deploy confirmation

- Install evidence: `Installing aem-adaptive-forms-agents.all (…/all/target/…all-1.0.0-SNAPSHOT.zip) to
  http://localhost:4502/crx/packmgr/service.jsp` → **`Package installed in 1064ms.`**
- Install signal — `lastUnpacked` advanced to 2026-07-28 for `all`, `ui.apps`, `ui.content`, `ui.config`
  (verified via `/crx/packmgr/list.jsp`, `lastUnpackedBy=admin`).
- **Core OSGi bundle ACTIVE:** `/system/console/bundles.json` → `symbolicName=aem-adaptive-forms-agents.core`,
  **`state=Active`** (stateRaw 32), version `1.0.0.SNAPSHOT`, id 738. (The orchestrator referenced this as
  `com.aem.forms.agents.core`, which is the Java **package**; the OSGi Bundle-SymbolicName is
  `aem-adaptive-forms-agents.core`.) No prior-known SDK inactive-bundle issue observed this pass.
- "Test Adaptive Form" page live: `GET /content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html`
  → **HTTP 200** (unchanged by this test-only fix; assembly not re-run).

## Deploy integrity — instance matches source

This is a **test-only** change (a renamed test method + removed stubs in the `core` test sources); it
touches **no** `ui.content` / `ui.config` JCR node and no `mode="update"` filter root. No node was added,
renamed, or moved on the instance → **no orphan / stale / duplicate node possible. Clean — no purge required.**
The `all`, `ui.apps`, `ui.content`, `ui.config` packages reinstalled unchanged (FileVault reconciled).

## Verdict & gate — REBUILD RE-RUN (Fix pass 14)

| Gate criterion | Result |
|---|---|
| BUILD SUCCESS | ✅ |
| All unit tests pass (120/120, incl. all 5 prefill classes) | ✅ |
| Confirmed deploy to localhost:4502 (`Package installed in 1064ms` + `lastUnpacked` advanced) | ✅ |
| Core OSGi bundle `aem-adaptive-forms-agents.core` ACTIVE | ✅ |
| Embedded "Test Adaptive Form" page live (HTTP 200) | ✅ |
| Instance matches source (no orphan/duplicate nodes) | ✅ |

### GATE RESULT: **PASS** — handing back to `aem-forms-program-agent` to run **Sentinel** value-level re-verification (SENTINEL-D15-02).

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
fix_pass: 14
sentinel_ref: SENTINEL-D15-02
build: BUILD SUCCESS
command: "mvn clean install -PautoInstallSinglePackage"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps.structure: PASS, ui.apps: PASS, ui.content: PASS, ui.config: PASS, all: PASS, ui.frontend: PASS, it.tests: PASS, dispatcher: PASS, ui.tests: PASS }
unit_tests: { run: 120, passed: 120, failed: 0, errors: 0, skipped: 0, coverage_forms_services_pct: 85.8 }
prefill_tests:
  BusinessRegistrationPrefillServiceTest: { run: 15, failures: 0, result: PASS }
  ContactUsPrefillServiceTest: { run: 7, failures: 0, result: PASS }
  ComplaintPrefillServiceTest: { run: 6, failures: 0, result: PASS }
  WithdrawalPrefillServiceTest: { run: 6, failures: 0, result: PASS }
  LifeInsurancePrefillServiceTest: { run: 15, failures: 0, result: PASS }   # renamed getPrefillData_nonDraftForeignDataRef_defers now asserts null
static_analysis: { errors: 0, warnings: 1, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }   # Package installed in 1064ms
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.dispatcher.cloud-1.0.0-SNAPSHOT.zip", type: content-package, installed: false }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }   # Bundle-SymbolicName aem-adaptive-forms-agents.core — ACTIVE (id 738)
deploy_confirmed: true
core_bundle: { symbolic_name: "aem-adaptive-forms-agents.core", state: Active, state_raw: 32, version: "1.0.0.SNAPSHOT", id: 738 }
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true }
deploy_integrity: { instance_matches_source: true, orphans_purged: [] }
report: ".claude/agents/runs/2026-07-28-employee-training-request/deployment/code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (SENTINEL-D15-02 value-level re-verification)
```
