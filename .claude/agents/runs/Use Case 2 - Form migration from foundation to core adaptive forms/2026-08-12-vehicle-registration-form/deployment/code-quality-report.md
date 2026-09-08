# Code Quality Report — vehicle-registration-form (ASSEMBLY-only re-embed deploy)

**Run ID:** 2026-08-12-vehicle-registration-form
**Phase:** DEPLOY (Forgemaster)
**Delivery type:** ASSEMBLY-only change. Formwright and Groundsmith did **not** run in this run —
`vehicle-registration-form` and its artifacts are unchanged from run `2026-07-28-vehicle-registration-form`.
Assembler repointed the "Test Adaptive Form" page's single AEM Form Container from
`event-ticket-booking` to `vehicle-registration-form` (page reused, container repointed, no second
container added). Additionally, `.aem-forms-config.yaml`'s previously-blank `cloudDev:` block was
populated this run (`p185256` / `e1945105`, author + publish URLs, `siteRoot`), closing defect **D9**
from run `2026-08-06-scoped-two-form-deploy`. Both changes ship in the same PR.
**Date executed:** 2026-08-12
**AEM target:** `http://localhost:4502` (local AEMaaCS SDK author instance)
**IMPL scope note:** `implementation/formwright.md` / `implementation/groundsmith.md` do not exist in
*this* run directory by design — this is an assembly-only cycle. The artifact inventory for
`vehicle-registration-form` was verified against `.claude/agents/runs/2026-07-28-vehicle-registration-form/implementation/`.

---

## 1. Build verdict

**BUILD SUCCESS**

- **Command:** `mvn clean install` (full reactor, tests included — no autoInstall profile)
- **Reactor:** `aem-adaptive-forms-agents` (`core`, `ui.apps`, `ui.config`, `ui.content`, `all`)
- **Deploy method (local-SDK workaround, per project memory `sdk-package-install-hang-workaround` and
  the identical deviation recorded in the 2026-07-28 report):** the `-PautoInstallSinglePackage` /
  `-PautoInstallPackage` Maven profiles hang indefinitely on this local SDK
  (`Packager manager not ready: 1 packages left for installation ...` loop that never returns a
  verdict). To get a verifiable `BUILD SUCCESS`, the reactor was built with plain `mvn clean install`
  (avoids the hang, still runs all unit tests), and the produced `all` aggregate package was deployed
  directly via the CRX Package Manager HTTP API (upload + install) — the equivalent deployment of
  record for this environment. This changes only *how* the install is triggered, not *what* is
  deployed (still the single `all` aggregate zip).

## 2. Per-module results

| Module | Status |
|---|---|
| aem-adaptive-forms-agents (reactor root) | SUCCESS |
| core (Core Bundle) | SUCCESS |
| ui.apps (UI Apps) | SUCCESS |
| ui.config (UI Config) | SUCCESS |
| ui.content (UI Content) | SUCCESS |
| all (single deployable) | SUCCESS |

No `dispatcher` module in this reactor.

## 3. Unit tests & coverage

**Tests: 22 run / 22 passed / 0 failed / 0 skipped** (reactor `core` module — unaffected by this
cycle's assembly-only, `ui.content`-page + config change; no new unit tests required):

| Test class | Run | Failures | Errors | Skipped |
|---|---|---|---|---|
| LoggingFilterTest | 1 | 0 | 0 | 0 |
| SimpleResourceListenerTest | 1 | 0 | 0 | 0 |
| HelloWorldModelTest | 1 | 0 | 0 | 0 |
| SimpleScheduledTaskTest | 1 | 0 | 0 | 0 |
| GeneratePDFServletTest | 13 | 0 | 0 | 0 |
| SimpleServletTest | 1 | 0 | 0 | 0 |
| CustomSubmitGeneratePDFActionTest | 4 | 0 | 0 | 0 |
| **Total** | **22** | **0** | **0** | **0** |

**Coverage (JaCoCo, `core/target/site/jacoco/jacoco.csv`)** on the Forms service classes relevant to
this delivery (`vehicle-registration-form` is wired to the shared `Custom-Submit-GeneratePDF` submit
action):

| Class | Instruction coverage | Line coverage | vs. ≥80% target |
|---|---|---|---|
| `CustomSubmitGeneratePDFAction` | 42/42 = **100%** | 12/12 = **100%** | PASS |
| `GeneratePDFServlet` | 662/687 = **96.4%** | 137/145 = **94.5%** | PASS |

## 4. Static analysis

- **Compiler warnings:** none observed in this build run.
- **`mvn validate` filter/violation check:** no warnings surfaced.
- **Lint / SpotBugs / Checkstyle / PMD:** none configured in this reactor's `pom.xml` — no findings.

## 5. Deployment artifacts (mandatory)

| Artifact name | Type | Version | Installed |
|---|---|---|---|
| `aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip` | content-package (aggregate, deployment of record) | 1.0.0-SNAPSHOT | Yes — CRX Package Manager `cmd=upload` then `cmd=install`, both `{"success":true}`; `lastUnpacked` 1786540835694 confirmed advanced (~18s before verification) |
| `aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip` | content-package (embedded sub-package in `all`) — carries the repointed "Test Adaptive Form" page embed | 1.0.0-SNAPSHOT | Yes (via `all`) |
| `aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip` | content-package (embedded sub-package in `all`, unchanged this cycle) | 1.0.0-SNAPSHOT | Yes (via `all`) |
| `aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip` | content-package (embedded sub-package in `all`, unchanged this cycle) | 1.0.0-SNAPSHOT | Yes (via `all`) |
| `aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar` | OSGi bundle (embedded in `all`, unchanged this cycle) | 1.0.0-SNAPSHOT | Yes — bundle `aem-adaptive-forms-agents.core`, state **Active** (738/738 bundles active, confirmed via `/system/console/bundles/aem-adaptive-forms-agents.core.json`) |

All five artifacts were produced under their respective module `target/` directories and installed to
the AEM instance as one atomic package (`all`), per the single-authoritative-deploy rule.

## 6. Deploy confirmation

- **Package-install evidence (CRX Package Manager API, in place of the hanging autoInstall profile):**
  - Upload: `{"success":true,"msg":"Package uploaded","path":"/etc/packages/com.aem.forms.agents/aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip"}`
  - Install: `{"success":true,"msg":"Package installed"}`
  - `/crx/packmgr/list.jsp`: `"lastUnpacked":1786540835694` vs. verification time
    `1786540853416` — advanced ~18s prior, confirming a real (not stale/no-op) reinstall.
- **Core bundle:** `aem-adaptive-forms-agents.core` — **Active** (738/738 bundles total active).
- **Form content live:** `GET /content/forms/af/aem-adaptive-form-agents/vehicle-registration-form.html` → **HTTP 200**.
- **"Test Adaptive Form" page live and embeds the correct form:**
  - `GET /content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html` → **HTTP 200**.
  - Served HTML grep: **46 occurrences of `vehicle-registration-form`, 0 occurrences of
    `event-ticket-booking`** — proven from the rendered HTML, not the source XML.

## 7. Deploy integrity (step 3b — orphan/stale-node sweep)

Diffed the live container tree against source at
`/content/aem-adaptive-forms-agents/us/en/test-adaptive-form/jcr:content/root/container/container`:

```json
{
  "adaptiveFormEmbed": {
    "formRef": "/content/dam/formsanddocuments/aem-adaptive-form-agents/vehicle-registration-form",
    "sling:resourceType": "aem-adaptive-forms-agents/components/aemformscontainer",
    "loadType": "embed"
  }
}
```

**Result: clean — no orphans.** Exactly ONE child node, `adaptiveFormEmbed`, pointing at
`vehicle-registration-form`. No stale/duplicate embed node and no leftover `event-ticket-booking`
reference. No purge was necessary. The `mode="update"` filter-root hazard flagged by Assembler did not
materialize in this cycle because Assembler repointed the *same* container node's `formRef` property
rather than renaming/removing/adding a node — there was nothing for `update` mode to orphan.

## 8. Fragment reference check (item 6 of the DEPLOY gate) — factual correction

The DEPLOY brief asked to confirm `address-details-fragment` and `declaration-fragment` still resolve
in the embedded form. Verified against the **current, live** source and instance, not assumption:

- `GET /content/forms/af/aem-adaptive-form-agents/address-details-fragment.html` → **HTTP 404**
- `GET /content/forms/af/aem-adaptive-form-agents/declaration-fragment.html` → **HTTP 404**
- Neither fragment exists under `/content/forms/af/aem-adaptive-form-agents/` on the instance — its
  only children are `vehicle-registration-form` and `event-ticket-booking`.
- The current `ui.content` source for `vehicle-registration-form/.content.xml` contains **no
  `fragmentPath` / fragment-reference nodes at all** — `pinCode` and `declarationDate` are **inline**
  fields (`ownerDetailsPanel.pinCode`, `declarationPanel.declarationDate`), validated by
  `validatePin6Digits(...)` and `validateDateDDMMYYYY(...)` respectively, both defined in
  `ui.apps/.../vehicle-registration-form-clientlib/js/functions.js` (confirmed present, alongside
  `validateMobile10Digits`, `validateEmailWhenPresent`, `validateDobNotFuture`).

**Conclusion:** the fragment-based architecture described in the 2026-07-28 report and in this run's
brief has been **superseded** — most likely by the `5c490f5` "reduce demo forms to the two-form cloud
DEV deployment scope" commit, which is ahead of that report in the commit history. The form is now
fully self-contained (no `AdaptiveFormFragment` dependency), and both validation functions it actually
uses (`validatePin6Digits`, `validateDateDDMMYYYY`) are defined in the form's own clientlib and directly
in scope on the form's own `guideContainer` (`clientLibRef` includes `aem-adaptive-forms-agents.forms.vehicle-registration-form`) —
not routed through a fragment's separate `guideContainer`. This is reported as a factual finding, not
a defect: the architecture changed since the 07-28 baseline, and no fragment purge or fix is applicable
because there is nothing left to purge — the fragments are simply absent from current source. This
finding should be routed to Planwright/Formwright only if the fragment reuse pattern was intended to be
retained; forgemaster does not judge that call, only reports what is live.

## 9. Carried-forward defect — re-stated as moot given item 8

The 2026-07-28 report left an OPEN defect: both fragments' `guideContainer` carried only
`clientLibRef="core.forms.components.runtime.all"`, so `validateExactSixDigits` / `validateNotFutureDate`
were unresolved and the Rule Editor showed "Broken & Enabled" when a fragment was opened standalone.
Given item 8 above — the fragments no longer exist in current source/instance and the form's current
validation functions are `validatePin6Digits` / `validateDateDDMMYYYY`, not
`validateExactSixDigits` / `validateNotFutureDate` — **this specific defect no longer has an artifact to
attach to**. It is not fixed (out of forgemaster's lane either way); it is reported as superseded/moot.
If fragment reuse is reintroduced in a future cycle, the same clientLibRef-scoping risk applies and
should be re-verified then, routed to **Formwright**.

## 10. Verdict & gate

| Gate criterion | Result |
|---|---|
| BUILD SUCCESS | PASS |
| Unit tests (0 failed) | PASS (22/22) |
| Coverage ≥80% on Forms service classes | PASS (100%, 96.4%) |
| Deployment artifact manifest present | PASS (5 artifacts, §5) |
| Deploy confirmed to local SDK | PASS (§6) |
| "Test Adaptive Form" page live + correct form embedded | PASS (§6) |
| Instance matches source / no unresolved orphans | PASS (§7 — clean) |
| Fragment references (as briefed) | N/A — fragments absent from current source (§8, factual, not a defect) |

**GATE RESULT: PASS**

BUILD SUCCESS, confirmed deploy (form + embedded page, correct form proven from served HTML), no
failed unit tests, and the instance tree matches source with no unresolved orphan/duplicate nodes.
Control passes to Pilot for commit + push + PR to `main`.

---

## Handoff YAML

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
build: BUILD SUCCESS
command: "mvn clean install (autoInstall profile skipped — known local-SDK hang; deployed via CRX Package Manager HTTP API instead)"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps: PASS, ui.content: PASS, ui.config: PASS, all: PASS }
unit_tests: { run: 22, passed: 22, failed: 0, coverage_pct: 96 }
static_analysis: { warnings: 0, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
deploy_confirmed: true
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true, renders: "vehicle-registration-form (46 refs, 0 event-ticket-booking refs)" }
deploy_integrity: { instance_matches_source: true, orphans_purged: [] }
fragment_check: { address_details_fragment: "absent from current source/instance (404)", declaration_fragment: "absent from current source/instance (404)", note: "superseded architecture — form is now fully inline; not a defect" }
report: ".claude/agents/runs/2026-08-12-vehicle-registration-form/deployment/code-quality-report.md"
report_html: ".claude/agents/runs/2026-08-12-vehicle-registration-form/deployment/code-quality-report.html"
gate_result: PASS
next: aem-forms-program-agent runs pilot (commit + push + PR to main) only if gate_result == PASS; sentinel runs later, on cloud DEV, after the human merge + Cloud Manager DEV pipeline + an explicit prompt
```
