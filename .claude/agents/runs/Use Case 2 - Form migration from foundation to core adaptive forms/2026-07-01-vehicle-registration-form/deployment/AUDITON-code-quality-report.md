# AUDITON — Code Quality & Deployment Report

- **Form:** `vehicle-registration-form`
- **runId:** `2026-07-01-vehicle-registration-form`
- **Lead:** auditon (DEPLOY) — third of `blockwright → bridgesmith → auditon → sentinel`
- **AEM target:** `http://localhost:4502` (local AEM SDK, AEMaaCS)
- **Date:** 2026-07-01
- **Scope:** single authoritative build + deploy of the full Maven reactor; code-quality report + deployment gate. No artifact authoring, no functional/UI test suite (Sentinel).

---

## 0. REDEPLOY — DEF-01 required-asterisk fix (2026-07-01, latest deploy of record)

Blockwright fixed gate-failing defect **DEF-01** (required-field red asterisks not rendering) in the
`{project}-vehicle-registration` theme — author-only (see
`implementation/phase08-create-form-theme-def01-asterisk-fix.md`). Auditon re-ran the single
authoritative full-reactor build + deploy to make the fix live.

### Redeploy verdict

| Item | Value |
|------|-------|
| **Verdict** | **BUILD SUCCESS** |
| Command (of record) | `mvn clean install -PautoInstallSinglePackage -Djavax.net.ssl.trustStoreType=Windows-ROOT -Dvault.user=admin -Dvault.password=admin` |
| Resume (Zscaler workaround) | `mvn install -PautoInstallSinglePackage -rf :{project}.all -Daem.analyser.skip=true -Djavax.net.ssl.trustStoreType=Windows-ROOT -Dvault.user=admin -Dvault.password=admin -DskipTests` |
| AEM target | `http://localhost:4502` |
| Full reactor | Yes — no `-pl` (all 11 modules) |
| Deploy confirmed | **Yes** — `Package imported. / Package installed in 1863ms.` |
| Java / Maven | Java 21.0.11 LTS (≥11), Maven 3.9.9 (≥3.3.9) — enforcer PASS |
| Total time | Pass 1 (clean install, all modules + tests): 1:11 min · Pass 2 (resume: package build + deploy + downstream): 37.0 s |
| Cleared stale state | `.m2/.../com/adobe/aem/*.lastUpdated` deleted before build |

Same documented Zscaler/TLS-interception behaviour as the prior deploy: pass 1 compiled every module and
**ran + passed all 90 unit tests** and **built every package**, but the `all` module's
`aemanalyser-maven-plugin:project-analyse` (cloud-readiness static analysis) could not download
`com.adobe.aem:aem-sdk-api` from Maven Central (`403 Forbidden` / PKIX) and aborted the `all` module
**before** the `PackageInstall` deploy goal. Per the "Deploy Zscaler workaround" the build was resumed
from `:{project}.all` with `-Daem.analyser.skip=true` + `Windows-ROOT` truststore; the
resume reached **BUILD SUCCESS** and installed the `all` package. Hard PASS — the analyser is offline
cloud-readiness only and has no effect on the deployed artifacts.

### Redeploy — per-module results

| Reactor module | Type | Result |
|----------------|------|--------|
| `{project}` (root) | pom | SUCCESS |
| `{project}.core` | bundle (jar) | SUCCESS (compiled + 90 tests) |
| `{project}.ui.apps.structure` | content-package | SUCCESS |
| `{project}.ui.apps` | content-package | SUCCESS (carries the fixed theme.zip) |
| `{project}.ui.content` | content-package | SUCCESS (carries the fixed DAM theme-json + original rendition) |
| `{project}.ui.config` | content-package | SUCCESS |
| `{project}.all` | aggregate content-package | SUCCESS + **installed on AEM** |
| `{project}.ui.frontend` | pom/clientlib | SUCCESS |
| `{project}.it.tests` | jar | SUCCESS |
| `{project}.dispatcher` | dispatcher config | SUCCESS |
| `com.adobe.cq.cloud.testing.ui.cypress` (ui.tests) | tar (Cypress) | SUCCESS |

All 11 modules SUCCESS.

### Redeploy — unit tests

- **Total: 90 tests run · 90 passed · 0 failed · 0 errors · 0 skipped** (pass 1, `core` module, surefire).
- Up from 76 in the prior deploy — the new `GeneratePDFServletTest` (4) is now included alongside
  `CustomSubmitGeneratePDFActionTest` (4). No test regressions.
- Coverage: jacoco still not configured (pre-existing project gap) — no measured %.

### Redeploy — static analysis

- No SpotBugs/Checkstyle/PMD/jacoco configured — "none configured".
- Compiler: one benign deprecation notice on `GeneratePDFServlet.java` (and its test) — non-fatal, compiles clean.
- FileVault validators: one benign orphaned-filter WARNING (`/apps/{project}-vendor-packages`) — pre-existing, unrelated to this form.
- aem-analyser cloud-readiness: skipped offline (TLS-intercepted); re-run in CI with proxy trust.

### Redeploy — deployment artifacts (mandatory manifest)

| Artifact (name) | Type | Version | Installed on AEM |
|-----------------|------|---------|------------------|
| `{project}.all-1.0.0-SNAPSHOT.zip` | content-package (aggregate) | 1.0.0-SNAPSHOT | **Yes** — `Package installed in 1863ms` |
| `{project}.core-1.0.0-SNAPSHOT.jar` | OSGi bundle | 1.0.0-SNAPSHOT | **Yes** — embedded in `all` |
| `{project}.ui.apps-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | **Yes** — embedded in `all` (carries fixed theme.zip) |
| `{project}.ui.content-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | **Yes** — embedded in `all` (carries fixed DAM theme-json + original rendition) |
| `{project}.ui.config-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | **Yes** — embedded in `all` |
| `{project}.ui.apps.structure-1.0.0-SNAPSHOT.zip` | content-package (repo structure) | 1.0.0-SNAPSHOT | **Yes** — deploy dependency |
| `{project}.dispatcher.cloud-1.0.0-SNAPSHOT.zip` | dispatcher config | 1.0.0-SNAPSHOT | n/a (dispatcher; not installed to author) |
| `com.adobe.cq.cloud.testing.ui.cypress.tests-0.0.1-SNAPSHOT-ui-test-docker-context.tar.gz` | UI-test bundle (Cypress) | 0.0.1-SNAPSHOT | n/a (test harness) |

### Redeploy — asterisk-fix live confirmation

Verified against `http://localhost:4502` **after** the redeploy:

| Check | Evidence | Result |
|-------|----------|--------|
| Package install | `Package imported. / Package installed in 1863ms.` | PASS |
| Theme served (/apps) | `GET /apps/fd/af/themes/{project}-vehicle-registration/theme.zip` → HTTP **200** | PASS |
| **Corrected asterisk selector LIVE** | served theme.css (from deployed theme.zip) has the scoped `.cmp-adaptiveform-{type}[data-cmp-required="true"] .cmp-adaptiveform-{type}__label::after` block (lines 165–178) across all 9 field types (**10** `data-cmp-required` selector lines) | PASS |
| Asterisk glyph + colour | `content: " *"; color: var(--vrf-asterisk-red)` where `--vrf-asterisk-red: #e02020` (RED) | PASS |
| Broken selector removed | **0** `__label__qualifier` selectors in served theme.css (only string left is inside an explanatory comment) | PASS |
| Form renders | `GET .../vehicle-registration-form.html` → HTTP **200** | PASS |

`theme.css` fetched directly at `/apps/.../theme.css` returns 404 — that is expected: theme.css is
served from *inside* theme.zip by `ServeThemeArtifactsServlet`; the deployable/served artifact is the
theme.zip (200), whose contents are verified above.

### Redeploy gate

| Gate criterion | Status |
|----------------|--------|
| BUILD SUCCESS | PASS |
| Confirmed deploy to AEM (localhost:4502) | PASS (`Package installed in 1863ms`) |
| No failed unit tests | PASS (90/90) |
| Deployment artifacts named (name + version) | PASS |
| DEF-01 asterisk-fix CSS live in served theme | PASS |

### REDEPLOY GATE RESULT: **PASS** — DEF-01 fix is live; Sentinel may re-verify.

---

## (Prior deploy record — retained below)

---

## 1. Build verdict

| Item | Value |
|------|-------|
| **Verdict** | **BUILD SUCCESS** |
| Command (of record) | `mvn clean install -PautoInstallSinglePackage` |
| Deploy profile | `autoInstallSinglePackage` (aggregate `all` package → localhost:4502) |
| Full reactor | Yes — no `-pl` (all 11 modules) |
| Deploy confirmed | **Yes** — `Package imported.` / `Package installed in 582ms.` |
| Java / Maven | Java 21.0.11 LTS (≥11), Maven 3.9.9 (≥3.3.9) — enforcer PASS |
| Total time | Pass 1: 50.3 s (clean install, all modules compiled + unit tests) · Pass 2 (resume): 34.9 s (package build + deploy + downstream) |

### Two-pass note (documented Zscaler/TLS workaround applied)

The first `mvn clean install -PautoInstallSinglePackage` compiled every module, ran and **passed all 76 unit tests**, and **built every package**, but the `all` module's `aemanalyser-maven-plugin:project-analyse` goal failed trying to download the AEM SDK API from Maven Central:

```
PKIX path building failed: ... unable to find valid certification path to requested target (certificate_unknown)
```

This is the documented corporate-proxy/Zscaler TLS-interception failure — it is the cloud-readiness **static-analysis** goal, **not** the deploy, and it aborted the `all` module *before* the `PackageInstall` (deploy) goal could run. Per the project's "Deploy Zscaler workaround" the build was resumed from the `all` module with:

```
mvn install -PautoInstallSinglePackage -rf :{project}.all \
    -Daem.analyser.skip=true -Djavax.net.ssl.trustStoreType=Windows-ROOT -DskipTests
```

- `Windows-ROOT` truststore — trusts the intercepting proxy cert (TLS).
- Cleared stale `.m2/.../com/adobe/aem/*.lastUpdated` before the resume.
- `aem.analyser.skip` — skips the offline-blocked cloud-readiness analysis only (does not affect what deploys).
- `-DskipTests` on the **resume only** — the 76 unit tests already ran and passed in pass 1; no source changed between passes.

The resumed run reached **BUILD SUCCESS** and installed the `all` package on the instance. This is a **hard PASS** — no soft-pass: every module built, all tests passed, and the deploy is confirmed live on AEM (see §5–6).

---

## 2. Per-module results

| # | Reactor module | Type | Result |
|---|----------------|------|--------|
| 1 | `{project}` (reactor root) | pom | SUCCESS |
| 2 | `{project}.core` | bundle (jar) | SUCCESS (compiled + 76 tests) |
| 3 | `{project}.ui.apps.structure` | content-package | SUCCESS |
| 4 | `{project}.ui.apps` | content-package | SUCCESS |
| 5 | `{project}.ui.content` | content-package | SUCCESS |
| 6 | `{project}.ui.config` | content-package | SUCCESS |
| 7 | `{project}.all` | aggregate content-package | SUCCESS + **installed on AEM** |
| 8 | `{project}.ui.frontend` | pom/clientlib | SUCCESS |
| 9 | `{project}.it.tests` | jar | SUCCESS |
| 10 | `{project}.dispatcher` | dispatcher config | SUCCESS |
| 11 | `com.adobe.cq.cloud.testing.ui.cypress` (ui.tests) | tar (Cypress) | SUCCESS |

All 11 modules SUCCESS.

---

## 3. Unit tests & coverage

- **Total: 76 tests run · 76 passed · 0 failed · 0 errors · 0 skipped.**
- Run in the `core` module during pass 1 (`mvn clean install`). Test runner: surefire 2.22.1 with wcm.io AEM Mocks + Mockito.
- The `[Fatal Error] :1:1: Content is not allowed in prolog.` lines in the log are **benign** AEM-Mocks context-loader XML notices, not test failures (0 failures/errors confirmed).

### Per Forms service class

| Test class | Class under test | Tests | Result |
|------------|------------------|-------|--------|
| **CustomSubmitGeneratePDFActionTest** *(new this delivery)* | `com.aem.forms.agents.forms.submit.CustomSubmitGeneratePDFAction` | **3** | PASS |
| BusinessRegistrationSubmitActionTest | `...forms.submit.BusinessRegistrationSubmitAction` | 13 | PASS |
| HealthInsuranceFdmSubmitActionTest | `...forms.submit.HealthInsuranceFdmSubmitAction` | 9 | PASS |
| LifeInsuranceNewPolicySubmitActionTest | `...forms.submit.LifeInsuranceNewPolicySubmitAction` | 15 | PASS |
| BusinessRegistrationPrefillServiceTest | `...forms.prefill.BusinessRegistrationPrefillService` | 12 | PASS |
| LifeInsurancePrefillServiceTest | `...forms.prefill.LifeInsurancePrefillService` | 14 | PASS |
| ContactUsPrefillServiceTest | `...forms.prefill.ContactUsPrefillService` | 5 | PASS |
| LoggingFilterTest / SimpleResourceListenerTest / HelloWorldModelTest / SimpleScheduledTaskTest / SimpleServletTest | core scaffolding | 1 each (5) | PASS |

### Coverage note (target ≥80% on Forms service classes)

- **No coverage tool (jacoco) is configured** in the root or `core` POM, so an exact coverage percentage **cannot be measured from this build**. This is a pre-existing project gap, not a regression from this delivery.
- **Behavioural coverage of this delivery's Forms service class** (`CustomSubmitGeneratePDFAction`) is complete against its contract: the new `CustomSubmitGeneratePDFActionTest` (3 tests) exercises `getServiceName()` identity (`"Custom-Submit-GeneratePDF"`) and `submit()` returning `FORM_SUBMISSION_COMPLETE`/never-throw — the full public surface of the class.
- `GeneratePDFServlet` (the shared PDF servlet) is validated at runtime on the instance (POST-bound, registered — see §5) rather than by unit test.
- **Recommendation for the program:** add `jacoco-maven-plugin` to `core` to produce a measurable ≥80% number on the Forms `submit`/`prefill` packages. Flagged, does not block this gate.

---

## 4. Static analysis / warnings

- **No SpotBugs / Checkstyle / PMD / jacoco plugins configured** in the reactor — "none configured" for dedicated static analysis.
- **Compiler:** one benign `javac` deprecation notice — `GeneratePDFServlet.java` "uses or overrides a deprecated API" (recompile with `-Xlint:deprecation` for detail). Non-fatal; compiles clean; no errors.
- **FileVault package validators (12 validators run):** package build clean except two benign orphaned-filter WARNINGs for **pre-existing, unrelated** entries (`/apps/{project}/MakeMyTrip-Submit`, `/apps/{project}-vendor-packages`) — not part of `vehicle-registration-form` and not introduced by this delivery.
- **bnd baseline:** "No previous version to baseline against" — expected for a SNAPSHOT with no prior release; informational.
- **aem-analyser (cloud-readiness):** not executed — blocked offline by the TLS-interception proxy and skipped per the documented workaround. It has **no effect on the deployed artifacts**; re-run in the CI pipeline (with proxy trust) for the cloud-readiness gate.

---

## 5. Deployment artifacts (mandatory manifest)

Every artifact the reactor produced, with install confirmation:

| Artifact (name) | Type | Version | Size | Installed on AEM |
|-----------------|------|---------|------|------------------|
| `{project}.all-1.0.0-SNAPSHOT.zip` | content-package (aggregate) | 1.0.0-SNAPSHOT | 24.9 MB | **Yes** — `installed:true` in package registry; `Package installed in 582ms` |
| `{project}.core-1.0.0-SNAPSHOT.jar` | OSGi bundle | 1.0.0-SNAPSHOT | 93 KB | **Yes** — embedded in `all`; bundle **Active** (state=Active) |
| `{project}.ui.apps-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | 22.5 MB | **Yes** — embedded in `all` (application) |
| `{project}.ui.content-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | 2.8 MB | **Yes** — embedded in `all` (content) |
| `{project}.ui.config-1.0.0-SNAPSHOT.zip` | content-package | 1.0.0-SNAPSHOT | 21 KB | **Yes** — embedded in `all` (application) |
| `{project}.ui.apps.structure-1.0.0-SNAPSHOT.zip` | content-package (repo structure) | 1.0.0-SNAPSHOT | 6 KB | **Yes** — dependency of the deploy |
| `com.adobe.cq.cloud.testing.ui.cypress.tests-0.0.1-SNAPSHOT-ui-test-docker-context.tar.gz` | UI-test bundle (Cypress) | 0.0.1-SNAPSHOT | — | n/a (test harness; not installed to AEM) |

The `all` aggregate embeds core (bundle) + ui.apps + ui.content + ui.config and installs them as one package — this is the deployment of record.

---

## 6. Deploy confirmation (live on the instance)

Verified against `http://localhost:4502` after the deploy:

| Check | Evidence | Result |
|-------|----------|--------|
| Package install log | `Installing content... / Package imported. / Package installed in 582ms.` | PASS |
| `all` package in registry | `pid=com.aem.forms.agents:{project}.all:1.0.0-SNAPSHOT`, `installed:true`, `lastUnpackedBy:admin` | PASS |
| Core bundle | `{project}.core` v1.0.0.SNAPSHOT → **state=Active** | PASS |
| Submit action OSGi component | `com.aem.forms.agents.forms.submit.CustomSubmitGeneratePDFAction` → **active** | PASS |
| Shared PDF servlet OSGi component | `com.aem.forms.agents.core.servlets.GeneratePDFServlet` → **active** | PASS |
| generatePDF endpoint bound | `GET /bin/{project}/generatePDF` → HTTP **405** (POST-only servlet registered) | PASS |
| Form renders | `GET .../vehicle-registration-form.html` → HTTP **200** | PASS |
| fd:rules served (validation) | `guideContainer.model.json` → 19 validation/rule/expression hits present | PASS |
| Theme served (/apps) | `GET /apps/fd/af/themes/{project}-vehicle-registration/theme.zip` → HTTP **200** | PASS |

---

## 7. Post-deploy notes for Sentinel (TEST)

These are **live and ready** for functional/UI verification — Sentinel should confirm behaviour, not just presence:

1. **fd:rules / Rule Editor** — `guideContainer.model.json` returns 19 validation/rule hits; confirm the Rule Editor opens without a `JSON.parse` SyntaxError (blockwright flagged first-time `fd:validate` authoring) and that each field rule (dateOfBirth not-future, mobile 10-digit, email, PIN 6-digit, policyValidTill, declarationDate) fires.
2. **Proxied clientlib JS (js.txt honored)** — the per-form validation clientlib (`{project}.forms.vehicle-registration-form`, `functions.js`) and the shared `{project}.forms.generate-pdf` clientlib are both wired on the guideContainer `clientLibRef`; verify `functions.js` is actually served (not ignored) so rule functions don't throw `ReferenceError`.
3. **Theme render** — `/apps` theme.zip serves 200; verify the form renders with the branded 3-column grid, red asterisks, title/subtitle visible (pixel parity vs screenshot).
4. **Submit → PDF DoR (US-10)** — `CustomSubmitGeneratePDFAction` + `GeneratePDFServlet` are active. Verify on submit of a valid form: POST to `/bin/{project}/generatePDF` returns 200, a PDF lands under `/content/dam/{project}/`, the browser downloads it, the thank-you message shows, and submit is blocked while required fields are invalid (AC-10.1).
5. **pdf-writer-service** — repoinit/service-user mapping ships in the deployed `all` package; verify the `pdf-writer-service` system user exists and the DAM write in `GeneratePDFServlet` succeeds under it.

---

## 8. Verdict & gate

| Gate criterion | Status |
|----------------|--------|
| BUILD SUCCESS | PASS |
| Confirmed deploy to AEM (localhost:4502) | PASS (`installed:true`, bundle Active, form 200) |
| No failed unit tests | PASS (76/76) |
| Deployment artifacts named (name + version) | PASS (§5) |
| Report written to `deployment/` | PASS |

### GATE RESULT: **PASS** — Sentinel may proceed.

Residual (non-blocking): jacoco not configured (no measured coverage %); aem-analyser cloud-readiness skipped offline (TLS-intercepted) — re-run in CI with proxy trust; two pre-existing orphaned-filter warnings unrelated to this form; one benign deprecation notice in `GeneratePDFServlet`.

---

## Handoff YAML

```yaml
agent: auditon
phase: DEPLOY
status: PASSED
build: BUILD SUCCESS
command: "mvn clean install -PautoInstallSinglePackage"
resume_command: "mvn install -PautoInstallSinglePackage -rf :{project}.all -Daem.analyser.skip=true -Djavax.net.ssl.trustStoreType=Windows-ROOT -DskipTests"
aem_target: "http://localhost:4502"
modules:
  core: PASS
  ui.apps.structure: PASS
  ui.apps: PASS
  ui.content: PASS
  ui.config: PASS
  all: PASS
  ui.frontend: PASS
  it.tests: PASS
  dispatcher: PASS
  ui.tests: PASS
unit_tests: { run: 76, passed: 76, failed: 0, skipped: 0, coverage_pct: "not measured (jacoco not configured)" }
forms_service_under_test: { class: "CustomSubmitGeneratePDFAction", tests: 3, result: PASS }
static_analysis: { tools: "none configured (no jacoco/spotbugs/checkstyle)", compiler_warnings: 1, package_validator_warnings: 2 }
deployment_artifacts:
  - { name: "{project}.all-1.0.0-SNAPSHOT.zip", type: content-package, version: "1.0.0-SNAPSHOT", installed: true }
  - { name: "{project}.core-1.0.0-SNAPSHOT.jar", type: bundle, version: "1.0.0-SNAPSHOT", installed: true }
  - { name: "{project}.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, version: "1.0.0-SNAPSHOT", installed: true }
  - { name: "{project}.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, version: "1.0.0-SNAPSHOT", installed: true }
  - { name: "{project}.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, version: "1.0.0-SNAPSHOT", installed: true }
  - { name: "{project}.ui.apps.structure-1.0.0-SNAPSHOT.zip", type: content-package, version: "1.0.0-SNAPSHOT", installed: true }
deploy_confirmed: true
osgi_components_active: [CustomSubmitGeneratePDFAction, GeneratePDFServlet]
form_live: true   # .html 200, model.json rules present, theme.zip 200
report: ".claude/agents/runs/2026-07-01-vehicle-registration-form/deployment/AUDITON-code-quality-report.md"
gate_result: PASS
next: aem-forms-program-agent runs sentinel (testing)
```
