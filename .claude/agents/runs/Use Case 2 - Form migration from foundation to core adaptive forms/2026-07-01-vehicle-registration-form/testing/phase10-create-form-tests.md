# Phase 10 — create-form-tests · Unit / Integration + Coverage

- **Form:** vehicle-registration-form
- **runId:** 2026-07-01-vehicle-registration-form
- **Executor:** `create-form-tests` (delegated by the Sentinel TEST lead) + Sentinel runtime validator execution
- **Scope:** JUnit 5 unit tests for the shared Forms service classes this delivery relies on (the DoR PDF submit action + the shared PDF servlet), JaCoCo line/branch coverage (DESI TC-035 ≥80%), and runtime execution of the five clientlib JS validators (TC-004/007/009/014) against the served clientlib.
- **Result:** **PASS**

## Command of record

```
# Full suite (run from repo root)
mvn test -pl core -o -Djavax.net.ssl.trustStoreType=Windows-ROOT
```

Notes on environment:
- The corporate Zscaler proxy returns HTTP 403 for `.jar` downloads from Maven Central, so the JaCoCo 0.8.12 plugin JARs (`org.jacoco.*`, `org.ow2.asm:asm:9.7`) cannot be freshly downloaded in this environment. They were already present in the local `~/.m2` cache from a prior run, so the coverage run executed **offline (`-o`)** with the Windows-ROOT truststore and produced a real `jacoco.csv`.
- `jacoco-maven-plugin` (0.8.12, prepare-agent + report bound to the `test` phase, no enforcing `check` rule) is present in `core/pom.xml`. This closes the "no coverage tool configured" gap Auditon flagged.

## Test results (authoritative, this session)

| Metric | Value |
|--------|-------|
| Total tests (core module) | **90** |
| Passed | **90** |
| Failed | **0** |
| Errors | 0 |
| Skipped | 0 |
| Build | **BUILD SUCCESS** |

Per Forms-service test class:
- `CustomSubmitGeneratePDFActionTest` — **4/4 passed** (getServiceName identity, valid submit → `FORM_SUBMISSION_COMPLETE=TRUE`, null submitInfo handled, forced-exception error branch).
- `GeneratePDFServletTest` — **13/13 passed** (happy path, empty/null content → placeholder line, filename sanitization, long-line wrap, escape of special/non-ASCII, and every `storeInDam` branch: success + X-PDF-Dam-Path header, null AssetManager, null resolverFactory, LoginException, generic exception).
- Remaining 73 pre-existing core tests unchanged and green.

### Defect found and fixed during this phase (real, not soft-passed)

`GeneratePDFServletTest.testNoContentStillGeneratesPdf` **initially FAILED**: an empty submission was expected to render the placeholder line "No form data was submitted." in the PDF, but the servlet's empty-guard `if (lines.isEmpty())` could never fire — `"".split("\\r?\\n")` returns `[""]` (one empty element, not an empty array), so `lines` was `[""]` and the placeholder was never added; the PDF was a silently blank page.

**Fix applied** to `core/src/main/java/com/aem/forms/agents/core/servlets/GeneratePDFServlet.java`: the empty-guard now treats a `lines` list that is empty **or contains only blank lines** as "no data" and renders the placeholder. This makes the code's own documented intent (empty submission → placeholder) actually true and satisfies TC-035's empty/null-path requirement. After the fix the full suite is 90/90 green.

> Impact on the deployed instance: none for real submissions — every real vehicle-registration submission carries field content, and the live servlet already returns a valid PDF (verified: POST → HTTP 200, `%PDF-1.4`…`%%EOF`, DAM copy written). The fix improves the empty-submission DoR only and is picked up on the next reactor build.

## Coverage — the two in-scope Forms service classes (measured from `core/target/site/jacoco/jacoco.csv`, post-fix)

| Class | Line % | Branch % | Instruction % | ≥80% line? |
|-------|--------|----------|---------------|-----------|
| `com.aem.forms.agents.core.servlets.GeneratePDFServlet` | **94.5%** (137/145) | 86.4% | 96.4% | **YES** |
| `com.aem.forms.agents.forms.submit.CustomSubmitGeneratePDFAction` | **100%** (12/12) | 100% | 100% | **YES** |

Both exceed the DESI TC-035 ≥80% line-coverage bar with a real measured number.

## Clientlib JS validators executed at runtime (TC-004/007/009/014)

The DESI unit cases TC-004/007/009/014 target the **clientlib JavaScript validators**, not Java. Sentinel executed the five validators **from the live served clientlib** (`/etc.clientlibs/clientlibs/vehicle-registration-form-clientlib.js`, HTTP 200) against the DESI test vectors:

| Validator | Vectors | Result |
|-----------|---------|--------|
| `validateMobile10Digits` | `9876543210`→true, `123`→false, `abcd`→false, ``→true, null→true | 5/5 PASS |
| `validatePin6Digits` | `560001`→true, `5600`→false, `ab0001`→false, ``→true | 4/4 PASS |
| `validateEmailWhenPresent` | ``→true (optional), `bad`→false, `a@b.com`→true | 3/3 PASS |
| `validateDateDDMMYYYY` | `15/06/2000`→true, `32/13/2000`→false, ``→true, ISO `2000-06-15`→true | 4/4 PASS |
| `validateDobNotFuture` | past→true, future→false, ``→true | 3/3 PASS |

**19/19 vectors pass.** Every validator is global-scope (Rule-Editor discoverable), empty-safe (returns true on null/undefined/""), and all five are served live and wired into the form's `fd:rules` Validate ASTs — so no `ReferenceError` at runtime (js.txt honored; the proxied clientlib serves them).

## DESI test cases executed by this phase

| TC | Traces to | Result |
|----|-----------|--------|
| TC-004 (validateDateDDMMYYYY + validateDobNotFuture) | US-01 / AC-01.2 | **PASS** (served validators executed with vectors) |
| TC-007 (validateMobile10Digits) | US-02 / AC-02.1 | **PASS** |
| TC-009 (validateEmailWhenPresent) | US-02 / AC-02.2 | **PASS** |
| TC-014 (validatePin6Digits) | US-03 / AC-03.4 | **PASS** |
| TC-035 (Custom-Submit-GeneratePDF + GeneratePDFServlet unit, ≥80% coverage) | US-10 / AC-10.2 | **PASS** — 90/90 tests; servlet 94.5% line, action 100% line |

## Files created / modified this delivery

- `core/src/test/java/com/aem/forms/agents/core/servlets/GeneratePDFServletTest.java` (13 tests) — created by create-form-tests.
- `core/src/test/java/com/aem/forms/agents/forms/submit/CustomSubmitGeneratePDFActionTest.java` (4 tests) — extended.
- `core/pom.xml` — `jacoco-maven-plugin` 0.8.12 added (measure/report only).
- `core/src/main/java/com/aem/forms/agents/core/servlets/GeneratePDFServlet.java` — **empty-submission placeholder guard fixed by Sentinel** (see defect above).
- Reports: `core/target/site/jacoco/jacoco.csv`, `core/target/site/jacoco/jacoco.xml`.

## Verdict: **PASS**
