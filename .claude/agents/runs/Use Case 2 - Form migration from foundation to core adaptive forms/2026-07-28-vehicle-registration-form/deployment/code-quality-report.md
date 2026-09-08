# Code Quality Report — vehicle-registration-form (RE-EMBED deploy)

**Run ID:** 2026-07-28-vehicle-registration-form
**Phase:** DEPLOY (Forgemaster)
**Delivery type:** Same-day re-embed — Formwright re-authored the form `.content.xml` to REVERSE the prior
inlining cycle: `addressSection` (4 inline fields → single `addressDetailsFragment` reference node) and
`declarationSection` (kept the static `declarationConsentText` text node, 2 inline fields →
`declarationFragment` reference node). Formwright also tightened `pincode` (native `numberinput` +
`pattern="^[0-9]{6}$"` + `validationExpression="validateExactSixDigits(...)"`) and `declarationDate`
(`validationExpression="validateNotFutureDate(...)"`), each with a full `fd:validate` Rule-Editor AST.
No clientlib changes were needed this cycle (`vehicle-registration-form-clientlib` already defines both
functions from the prior pass). `ownerDetailsPanel`, `vehicleDetailsPanel`, `documentInsurancePanel` and
their rules are unchanged. Groundsmith and Assembler artifacts were **not** touched in this cycle; their
prior outputs remain valid.
**Supersedes:** the earlier same-day "DEFECT FIX" deploy report for this run (which had inlined the
fields the opposite direction) — this report reflects the current, authoritative on-disk/on-instance state.
**Date executed:** 2026-07-28
**AEM target:** `http://localhost:4502` (local AEMaaCS SDK author instance)

---

## 1. Build verdict

**BUILD SUCCESS**

- **Command:** `mvn clean install` (full reactor, tests included)
- **Total time:** 48.288 s
- **Finished at:** 2026-07-28T15:50:51+05:30
- **Reactor:** `aem-adaptive-forms-agents` (`core`, `ui.apps`, `ui.config`, `ui.content`, `all`)
- **Deploy method (local-SDK workaround):** per project memory (`sdk-package-install-hang-workaround`),
  the `-PautoInstallSinglePackage` / `-PautoInstallPackage` Maven profiles reliably hang indefinitely on
  this local SDK (`Packager manager not ready: 1 packages left for installation ...` loop that never
  returns a verdict). To get a verifiable BUILD SUCCESS, the reactor was built with plain
  `mvn clean install` (no autoInstall profile — avoids the hang, still runs all unit tests), and the
  produced `all` aggregate package was then deployed directly via the CRX Package Manager HTTP API
  (upload + install), which is the equivalent deployment of record for this environment. This does not
  change what gets deployed — the same `all` aggregate zip — only how the install is triggered.

## 2. Per-module results

| Module | Status | Time |
|---|---|---|
| aem-adaptive-forms-agents (reactor root) | SUCCESS | 1.178 s |
| core (Core Bundle) | SUCCESS | 32.785 s |
| ui.apps (UI Apps) | SUCCESS | 5.028 s |
| ui.config (UI Config) | SUCCESS | 1.144 s |
| ui.content (UI Content) | SUCCESS | 2.527 s |
| all (single deployable) | SUCCESS | 1.143 s |

No `dispatcher` module in this reactor.

## 3. Unit tests & coverage

**Tests: 28 run / 28 passed / 0 failed / 0 skipped** (reactor `core` module — unchanged by this cycle's
`ui.content`-only change; no new unit tests were added or required):

| Test class | Run | Failures | Errors | Skipped |
|---|---|---|---|---|
| CheckboxFieldModelTest | 2 | 0 | 0 | 0 |
| FileInputFieldModelTest | 2 | 0 | 0 | 0 |
| CustomSubmitGeneratePDFActionTest | 4 | 0 | 0 | 0 |
| GeneratePDFServletTest | 7 | 0 | 0 | 0 |
| SimpleInterestServletTest | 3 | 0 | 0 | 0 |
| SimpleInterestSubmitActionTest | 7 | 0 | 0 | 0 |
| AssembleTrainingRequestPacketTest | 3 | 0 | 0 | 0 |
| **Total** | **28** | **0** | **0** | **0** |

**Coverage (JaCoCo, `core/target/site/jacoco/jacoco.csv`)** on the Forms service classes relevant to this
delivery (vehicle-registration-form is wired to the shared `Custom-Submit-GeneratePDF` submit action):

| Class | Instruction coverage | Line coverage | vs. ≥80% target |
|---|---|---|---|
| `CustomSubmitGeneratePDFAction` | 57/68 = **83.8%** | 15/18 = 83.3% | PASS |
| `GeneratePDFServlet` | 522/577 = **90.5%** | 119/137 = 86.9% | PASS |
| `SimpleInterestServlet` | 203/245 = 82.9% | 45/53 = 84.9% | PASS (other form, informational) |
| `SimpleInterestSubmitAction` | 478/540 = 88.5% | 93/109 = 85.3% | PASS (other form, informational) |

Unchanged from the prior deploy (this cycle touched only `ui.content`, not `core`). `SimpleInterestExcelWriter`
(0% — untested writer used by another form) and `AssembleTrainingRequestPacket` (29.5% — workflow packet
assembler for another form) remain below target but belong to unrelated forms and are pre-existing,
out-of-scope for this delivery.

## 4. Static analysis

- **Compiler warnings:** 2 unique messages, none blocking:
  - `system modules path not set in conjunction with -source 11` (informational `javac` cross-compile notice, ×2)
  - `Ignoring incompatible plugin version 4.0.0-beta-{1,2,3}` (Maven ignoring newer plugin descriptors not
    compatible with the local Maven 3.9.16 — cosmetic)
- **FileVault filter validation:** several `ValidationViolation: Filter root's ancestor ... is not covered
  by any of the specified dependencies nor a valid root` messages on `ui.apps`, `ui.config`, `ui.content`,
  and `all` — these are the project's existing, pre-existing filter-root warnings (unrelated to this
  cycle's change) and did not fail the build (`filevault-package:validate-package` reports them as
  informational violations, not errors, for this package type).
- **Lint / SpotBugs / Checkstyle / PMD:** none configured in this reactor's `pom.xml` — no findings to report.

## 5. Deployment artifacts (mandatory)

| Artifact name | Type | Version | Installed |
|---|---|---|---|
| `aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip` | content-package (aggregate, deployment of record) | 1.0.0-SNAPSHOT | Yes — uploaded to `/etc/packages/aem-adaptive-forms-agents/...` via CRX Package Manager API, `{"success":true,"msg":"Package installed"}`, `lastUnpacked` advanced to 1785234239811 (~29s before verification) |
| `aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip` | content-package (embedded sub-package in `all`) — carries the re-embedded `vehicle-registration-form` `.content.xml` and both fragment `.content.xml` files | 1.0.0-SNAPSHOT | Yes (via `all`) |
| `aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip` | content-package (embedded sub-package in `all`, unchanged this cycle) | 1.0.0-SNAPSHOT | Yes (via `all`) |
| `aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip` | content-package (embedded sub-package in `all`, unchanged this cycle) | 1.0.0-SNAPSHOT | Yes (via `all`) |
| `aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar` | OSGi bundle (embedded in `all`, unchanged this cycle) | 1.0.0-SNAPSHOT | Yes — bundle `aem-adaptive-forms-agents.core`, state **Active** (confirmed via `/system/console/bundles.json`) |

All five artifacts were produced under their respective module `target/` directories and installed to the
AEM instance as one atomic package (`all`), per the single-authoritative-deploy rule.

## 6. Deploy confirmation

- **Package-install evidence (CRX Package Manager API, used in place of the hanging autoInstall profile):**
  - Upload: `curl -F package=@all/target/aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip -F force=true .../cmd=upload` → `{"success":true,"msg":"Package uploaded","path":"/etc/packages/aem-adaptive-forms-agents/aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip"}`
  - Install: `curl -X POST .../etc/packages/aem-adaptive-forms-agents/aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip?cmd=install` → `{"success":true,"msg":"Package installed"}`
  - `/crx/packmgr/list.jsp` package record: `"installed":true`, `"lastUnpacked":1785234239811` — confirmed
    advancing to ~29 seconds before the verification check (not a stale/no-op reinstall).
  - `mvn clean install` (build phase): `[INFO] BUILD SUCCESS`, Total time 48.288 s.
- **Core bundle:** `aem-adaptive-forms-agents.core` — **Active** (`/system/console/bundles.json`).
- **Form content live:**
  - `GET /content/forms/af/aem-adaptive-form-agents/vehicle-registration-form.html` → **HTTP 200**
  - `GET /editor.html/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form.html` (AF editor) → **HTTP 200**, opens without error
- **"Test Adaptive Form" page live and embeds the form:**
  - `GET /content/aem-adaptive-forms-agents/us/en/test-adaptive-form.html` → **HTTP 200**
- **Re-embed content verified live (JCR):**
  - `addressSection` live children: exactly one — `addressDetailsFragment`
    (`sling:resourceType=.../fragment`, `fragmentPath=/content/forms/af/aem-adaptive-form-agents/address-details-fragment`).
    No inline `addressLine`/`city`/`state`/`pincode` fields remain, and no duplicate/orphan fragment node.
  - `declarationSection` live children: `declarationConsentText` (static text, kept) +
    `declarationFragment` (`fragmentPath=/content/forms/af/aem-adaptive-form-agents/declaration-fragment`).
    No inline `submittedBy`/`declarationDate` fields remain.
  - `address-details-fragment` field `pincode`: live as `sling:resourceType=.../numberinput`,
    `fieldType=number-input`, `pattern="^[0-9]{6}..."`, `validationExpression="validateExactSixDigits($field.$value) == true()"`,
    `fd:validate` node present.
  - `declaration-fragment` field `declarationDate`: live with
    `validationExpression="validateNotFutureDate($field.$value) == true()"`, `fd:validate` node present.
  - `vehicle-registration-form-clientlib/js/functions.js` (deployed, `/apps/clientlibs/...`) confirmed to
    still define both `validateExactSixDigits` and `validateNotFutureDate` — in scope at runtime for
    vehicle-registration-form's own page.
- **Rule Editor — CORRECTION (user-verified, screenshot evidence):** my original check only confirmed that
  `vehicle-registration-form`'s own editor URL returns HTTP 200 and that both fragments' `fd:validate` AST
  nodes are well-formed JSON. That is **not the same thing** as the Rule Editor showing the rule as valid.
  The user manually opened **`address-details-fragment`'s own standalone editor**
  (`/editor.html/content/forms/af/aem-adaptive-form-agents/address-details-fragment.html`) and the
  `pincode - Validate` rule is flagged **"Broken & Enabled"** in the Form Objects panel (screenshot:
  `error7.png`). My earlier "Rule Editor: not broken" line in this report was wrong as stated — it is
  **broken in this specific editing context** — and I am correcting it here rather than letting the
  overclaim stand.

  **Root cause (confirmed by inspecting the deployed JCR, not guessed):**
  - `address-details-fragment`'s and `declaration-fragment`'s own `guideContainer` each carry
    `clientLibRef="core.forms.components.runtime.all"` only — neither references any clientlib that
    defines `validateExactSixDigits`/`validateNotFutureDate`.
  - Both functions live in the shared `clientlib-forms-base` (category
    `aem-adaptive-forms-agents.forms.base`, confirmed present in
    `ui.apps/.../clientlibs/clientlib-forms-base/js/functions.js`).
  - `vehicle-registration-form`'s own clientlib (`clientLibRef="aem-adaptive-forms-agents.vehicle-registration-form"`)
    already depends on that base clientlib (`dependencies="[core.forms.components.runtime.all,aem-adaptive-forms-agents.forms.base]"`)
    — so when the fragment's field is edited/rendered **in the host form's own context**, the function
    should resolve. But when a fragment page is opened **standalone** (its own editor, as in the
    screenshot), only `core.forms.components.runtime.all` loads — the function-defining clientlib never
    loads in that context — so the Rule Editor cannot resolve `validateExactSixDigits` and correctly
    reports the rule as Broken there. The same applies to `declaration-fragment`'s `declarationDate`
    rule (`validateNotFutureDate`) for the identical reason — I checked its `guideContainer` and it has
    the identical `clientLibRef="core.forms.components.runtime.all"` with no base-clientlib reference.
  - This is the same underlying pattern already flagged in section 8 (known follow-up risk) —
    but it is now a **confirmed, screenshot-verified defect on the fragments this deploy touched**, not
    just a hypothetical risk on sibling forms.
  - **I have not verified** whether the rule actually fires correctly at runtime when the fragment is
    rendered inside `vehicle-registration-form` (the host clientlib does define the function, so it
    plausibly works there) — that is a functional-testing question for Sentinel, not something curl/JCR
    inspection can confirm. Do not treat "works when embedded in the host" as proven by this report.
  - **Fix (not performed here — out of Forgemaster's lane):** add
    `aem-adaptive-forms-agents.forms.base` to the `clientLibRef`/dependency chain the FRAGMENT pages
    themselves load (e.g. give each fragment's `guideContainer` a `clientLibRef` that includes the base
    clientlib), so the Rule Editor resolves the function even when a fragment is opened standalone. This
    is Formwright/create-form-clientlib authoring work.

## 7. Deploy integrity (step 3b — orphan/stale-node sweep)

**Why this matters here:** re-authoring `addressSection`/`declarationSection` from inline fields back to
fragment references is exactly the "renamed/restructured panel children" pattern that can leave orphan
duplicates under a `mode="update"` filter root.

**Pre-deploy condition (checked by Formwright before hand-off, reconfirmed by Forgemaster intent):** the
live `addressSection`/`declarationSection` returned **zero children** prior to this deploy (no stale
inline fields, no fragment node) — so there was no pre-existing orphan risk requiring a pre-delete step.

**Post-deploy verification (live JCR `.1.json` vs. source `.content.xml`):**
- `addressSection` live children: `addressDetailsFragment` only — matches source exactly (source panel
  contains only the one fragment node). No orphaned inline fields, no duplicate fragment node.
- `declarationSection` live children: `declarationConsentText`, `declarationFragment` — matches source
  exactly (source panel contains the static text node plus the one fragment node). No orphaned inline
  fields.
- `address-details-fragment` and `declaration-fragment` guideContainer trees deployed with their updated
  field definitions (`pincode`, `declarationDate`) — no duplicate field nodes found.

**Verdict: clean — no orphans found, no purge required.**

## 8. Known follow-up risks (not gate blockers under the 4 hard criteria; both require Formwright action)

**8a. CONFIRMED — fragment's own standalone editor shows the rule Broken (user-verified, section 6).**
`address-details-fragment` (`pincode` / `validateExactSixDigits`) and `declaration-fragment`
(`declarationDate` / `validateNotFutureDate`) both show their new validate rule as **"Broken & Enabled"**
when the fragment is opened in its own standalone AF editor, because neither fragment's `guideContainer`
loads a clientlib that defines the function (`clientLibRef="core.forms.components.runtime.all"` only).
Fix: add the `aem-adaptive-forms-agents.forms.base` clientlib reference to each fragment's own
`guideContainer` — Formwright/create-form-clientlib work, not performed here.

**8b. Sibling forms (informational, not re-verified this cycle).** `address-details-fragment` and
`declaration-fragment` are also embedded by 4 sibling forms (`college-admission-registration`,
`doctor-appointment-registration`, `school-admission-registration`, `sports-event-registration`). None of
those 4 forms' own clientlibs define `validateExactSixDigits`; 2 of them
(`college-admission-registration`, `sports-event-registration`) also lack `validateNotFutureDate`. Same
root cause as 8a, applied to those forms' own editor/render context. Known, already-flagged risk (project
memory `fragment-rules-need-shared-base-functions`). Durable fix is the same: reference
`aem-adaptive-forms-agents.forms.base` from wherever each affected context loads its clientlib. Out of
scope for this deploy gate; flagging for Sentinel and the program agent before broader regression testing.

## 9. Verdict & gate

| Gate criterion | Result |
|---|---|
| BUILD SUCCESS | PASS |
| Deploy confirmed (package installed, `lastUnpacked` advanced, bundle Active) | PASS |
| Form live (HTTP 200) | PASS |
| AF editor opens without error (host form editor URL) | PASS |
| Rule Editor shows rule as valid/intact | **FAIL for the fragment's own standalone editor** (confirmed broken, section 6/8a) — not independently confirmed valid in the host form's editing context either (not testable via curl/JCR; needs Sentinel/manual UI check) |
| "Test Adaptive Form" page live (HTTP 200) | PASS |
| Unit tests | PASS (28/28, 0 failures) |
| Coverage on delivery-relevant Forms service classes ≥80% | PASS (83.8% / 90.5%) |
| Instance matches source (no orphan/duplicate nodes) | PASS (clean) |
| Fragment re-embed + tightened validations deployed live in both fragments | PASS (deployed correctly; Rule Editor resolution of the function is the separate, confirmed-broken issue above) |

**GATE RESULT: PASS on the 4 hard deploy criteria (BUILD SUCCESS, confirmed deploy of form+page, no failed
unit tests, instance matches source) — build/deploy is sound and is the deployment of record.**
**However, a real authoring defect is confirmed and unresolved: both fragments' custom-function validate
rules show "Broken" in their own standalone Rule Editor (screenshot-verified), because the fragments don't
load the clientlib that defines the function. This is flagged as a MUST-FIX for Formwright before this
delivery is considered functionally complete, and Sentinel should explicitly verify actual rule behavior
(not just page-load HTTP status) on both the fragment's own editor and the host form's editor/render/submit
path.**

---

## Handoff YAML

```yaml
agent: forgemaster
phase: DEPLOY
status: PASSED
build: BUILD SUCCESS
command: "mvn clean install (full reactor, tests included); deploy via CRX Package Manager API upload+install of the 'all' aggregate zip (local-SDK autoInstall-profile hang workaround per project memory)"
aem_target: "http://localhost:4502"
modules: { core: PASS, ui.apps: PASS, ui.content: PASS, ui.config: PASS, all: PASS }
unit_tests: { run: 28, passed: 28, failed: 0, coverage_pct: 83.8 }
static_analysis: { warnings: 2, findings: 0 }
deployment_artifacts:
  - { name: "aem-adaptive-forms-agents.all-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.content-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.apps-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.ui.config-1.0.0-SNAPSHOT.zip", type: content-package, installed: true }
  - { name: "aem-adaptive-forms-agents.core-1.0.0-SNAPSHOT.jar", type: bundle, installed: true }
deploy_confirmed: true
test_adaptive_form_page: { path: "/content/aem-adaptive-forms-agents/us/en/test-adaptive-form", live: true }
deploy_integrity: { instance_matches_source: true, orphans_purged: [] }
rule_editor_status: "CONFIRMED BROKEN (user-verified screenshot) — pincode-Validate rule shows 'Broken & Enabled' when address-details-fragment is opened in its own standalone editor; declarationDate rule has the identical root cause in declaration-fragment. Root cause: neither fragment's guideContainer clientLibRef references aem-adaptive-forms-agents.forms.base, the clientlib that actually defines validateExactSixDigits/validateNotFutureDate. Not fixed in this deploy — Formwright must add that clientlib reference to each fragment's own guideContainer."
known_risk: "Same root cause additionally affects 4 sibling forms (college-admission-registration, doctor-appointment-registration, school-admission-registration, sports-event-registration) embedding these fragments without the base clientlib — flagged for formwright/groundsmith, not remediated in this deploy"
report: ".claude/agents/runs/2026-07-28-vehicle-registration-form/deployment/code-quality-report.md"
gate_result: PASS
gate_note: "PASS on the 4 hard deploy criteria (build/deploy/tests/instance-integrity) only. A real, confirmed authoring defect (Rule Editor Broken on both re-embedded fragments) remains open and must be fixed by formwright before this delivery is functionally complete; sentinel must explicitly verify rule behavior, not just page-load status."
next: aem-forms-program-agent runs sentinel (testing) only if gate_result == PASS — but must relay the open Rule Editor defect to formwright in parallel
```
