---
name: create-form-tests
description: >
  Generates JUnit 5 unit tests for AEM Adaptive Forms services (prefill, submit action,
  Sling Models) and Selenium/WebDriver integration tests for form rule editor behaviour.
  Uses AEM Mocks (io.wcm.testing.mock.aem) and Mockito.
version: 1.0.0
ide:
  cursor: .cursor/skills/create-form-tests/
  github-copilot: .github/skills/create-form-tests/
  claude-code: .claude/skills/create-form-tests/
---

# Skill: create-form-tests

## Role

You are an AEM Forms test engineer. You write tests that give real confidence —
not tests that just verify methods are called. Every test must assert a meaningful
outcome. You cover the happy path, the empty/null path, and at least one error path.

---

## Trigger

This skill activates when the developer asks to:
- Write unit tests for a prefill service
- Write unit tests for a submit action
- Write unit tests for a Sling Model
- Write integration tests for form rules (show/hide, validate)
- Add test coverage to existing Forms services

---

## Inputs / tokens

This skill uses placeholder tokens — resolve them from the user or `.aem-forms-config.yaml`,
never hardcode a specific project. Substitute the project's own values everywhere a token
appears in the templates below (including the `AppAemContext` import and the JSON fixture
classpath, which must mirror the project's real package).

| Token | Meaning | Example |
|---|---|---|
| `{package}` | Java base package after `com.` (dotted) | `mycompany.forms` |
| `{packagePath}` | `{package}` with dots → slashes — used for classpath fixture paths | `mycompany/forms` |
| `{project}` | App folder used in `sling:resourceType` paths | `my-forms-app` |
| `{ComponentName}` / `{componentName}` | Component class / node name | `StarRating` / `starrating` |
| `{ServiceName}` / `{serviceName}` | Prefill / submit service name | `UserProfile` / `userProfile` |
| `{ActionName}` / `{actionName}` | Submit action name | `RestSubmit` / `restSubmit` |
| `{fieldName}` | Field node / `name` attribute under test | `firstName` |

---

## JUnit 5 test for Sling Model

```java
package com.{package}.forms.components;

import com.adobe.cq.forms.core.components.models.form.FieldType;
import com.{package}.core.testcontext.AppAemContext;
import io.wcm.testing.mock.aem.junit5.AemContext;
import io.wcm.testing.mock.aem.junit5.AemContextExtension;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(AemContextExtension.class)
class {ComponentName}ModelTest {

    // Use the project's AppAemContext — it registers the CACONFIG + CORE_COMPONENTS
    // aem-mock plugins that Forms Core Component models depend on. A bare
    // `new AemContext()` will not resolve those models.
    private final AemContext ctx = AppAemContext.newAemContext();

    @BeforeEach
    void setUp() {
        // Load test content from JSON fixture (classpath path mirrors the test package).
        ctx.load().json("/com/{packagePath}/forms/components/{componentName}/test-content.json",
            "/content/forms/af/test-form");

        // Register the model
        ctx.addModelsForPackage("com.{package}.forms.components");

        // Set current resource
        ctx.currentResource("/content/forms/af/test-form/guideRootPanel/{fieldName}");
    }

    @Test
    @DisplayName("Model adapts from current resource without error")
    void testModelAdaptsSuccessfully() {
        {ComponentName}Model model = ctx.request().adaptTo({ComponentName}Model.class);
        assertNotNull(model, "Model should adapt successfully");
    }

    @Test
    @DisplayName("getFieldType returns expected type")
    void testGetFieldTypeReturnsCorrectValue() {
        {ComponentName}Model model = ctx.request().adaptTo({ComponentName}Model.class);
        assertNotNull(model);
        assertEquals(FieldType.TEXT_INPUT, model.getFieldType());
    }

    @Test
    @DisplayName("Custom property read from resource correctly")
    void testCustomPropertyIsReadFromResource() {
        {ComponentName}Model model = ctx.request().adaptTo({ComponentName}Model.class);
        assertNotNull(model);
        assertEquals("expectedValue", model.getCustomProperty());
    }

    @Test
    @DisplayName("Null resource returns empty optional values gracefully")
    void testNullResourceHandledGracefully() {
        ctx.currentResource(null);
        // Model should not throw — should return safe defaults
        {ComponentName}Model model = ctx.request().adaptTo({ComponentName}Model.class);
        // Null is acceptable here — the model simply cannot adapt
        // This confirms no NPE is thrown
    }
}
```

---

## JSON test fixture — /test/resources/…/test-content.json

```json
{
  "jcr:primaryType": "nt:unstructured",
  "guideRootPanel": {
    "jcr:primaryType": "nt:unstructured",
    "sling:resourceType": "core/fd/components/form/panelcontainer/v1/panelcontainer",
    "{fieldName}": {
      "jcr:primaryType": "nt:unstructured",
      "sling:resourceType": "{project}/components/adaptiveForm/{componentName}",
      "jcr:title": "Test Field",
      "name": "{fieldName}",
      "customProperty": "expectedValue",
      "required": true
    }
  }
}
```

---

## JUnit 5 test for prefill service

A prefill service implements `com.adobe.forms.common.service.DataProvider` and its
method is `PrefillData getPrefillData(DataOptions) throws FormsException` (see the
`create-prefill-service` skill). Test against `DataOptions`, not a request:

```java
package com.{package}.forms.prefill;

import com.adobe.forms.common.service.ContentType;
import com.adobe.forms.common.service.DataOptions;
import com.adobe.forms.common.service.PrefillData;
import io.wcm.testing.mock.aem.junit5.AemContext;
import io.wcm.testing.mock.aem.junit5.AemContextExtension;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(AemContextExtension.class)
class {ServiceName}PrefillServiceTest {

    private final AemContext ctx = new AemContext();

    private {ServiceName}PrefillService service;

    @BeforeEach
    void setUp() {
        service = ctx.registerInjectActivateService(
            new {ServiceName}PrefillService(),
            "apiEndpoint", "https://test-api.example.com/user",
            "apiToken",    "test-bearer-token",
            "timeoutMs",   1000);
    }

    @Test
    @DisplayName("Service name and description are correctly defined")
    void testServiceMetadata() {
        assertEquals("{serviceName}", service.getServiceName());
        assertNotNull(service.getServiceDescription());
    }

    @Test
    @DisplayName("Null options returns non-null JSON prefill data")
    void testNullOptionsReturnsEmptyData() throws Exception {
        PrefillData result = service.getPrefillData(null);
        assertNotNull(result, "Prefill data must never be null");
        assertEquals(ContentType.JSON, result.getContentType());
    }

    @Test
    @DisplayName("Options without a form resource (unauthenticated) returns empty prefill, no throw")
    void testNoFormResourceReturnsEmptyData() {
        assertDoesNotThrow(() -> {
            PrefillData result = service.getPrefillData(new DataOptions());
            assertNotNull(result);
            assertEquals(ContentType.JSON, result.getContentType());
        });
    }
}
```

---

## JUnit 5 test for submit action

```java
package com.{package}.forms.submit;

A submit action implements `com.adobe.aemds.guide.service.FormSubmitActionService`
with `Map<String,Object> submit(FormSubmitInfo)`. `FormSubmitInfo.getData()` returns
a `String`; there is no `getRequest()`/`getThankYouPage()` (see `create-submit-action`):

```java
package com.{package}.forms.submit;

import com.adobe.aemds.guide.model.FormSubmitInfo;
import io.wcm.testing.mock.aem.junit5.AemContext;
import io.wcm.testing.mock.aem.junit5.AemContextExtension;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith({AemContextExtension.class, MockitoExtension.class})
class {ActionName}SubmitActionTest {

    private final AemContext ctx = new AemContext();

    @Mock private FormSubmitInfo mockSubmitInfo;

    private {ActionName}SubmitAction action;

    @BeforeEach
    void setUp() {
        action = ctx.registerInjectActivateService(
            new {ActionName}SubmitAction(),
            "apiEndpoint", "https://test-api.example.com/submit",
            "apiToken",    "test-token",
            "timeoutMs",   1000);
    }

    @Test
    @DisplayName("Service name matches the registered action name")
    void testServiceName() {
        assertEquals("{actionName}", action.getServiceName());
    }

    @Test
    @DisplayName("submit() with valid JSON data returns status in result map")
    void testSubmitWithValidData() {
        // getData() returns the payload as a String (JSON or XML).
        when(mockSubmitInfo.getData())
            .thenReturn("{\"firstName\":\"Priya\",\"email\":\"priya@example.com\"}");

        Map<String, Object> result = action.submit(mockSubmitInfo);

        assertNotNull(result);
        assertTrue(result.containsKey("status"), "Result must contain 'status' key");
    }

    @Test
    @DisplayName("submit() with null data returns an error status, never throws")
    void testSubmitWithNullData() {
        when(mockSubmitInfo.getData()).thenReturn(null);
        // submit() declares no checked exception — it must return a result map.
        Map<String, Object> result = action.submit(mockSubmitInfo);
        assertNotNull(result.get("status"));
    }
}
```

> **Strict stubs:** Mockito (4.x) defaults to `STRICT_STUBS` under `MockitoExtension`.
> Stub only what a test actually exercises, or wrap setup-only stubs in `lenient(...)`,
> to avoid `UnnecessaryStubbingException`.

---

## Selenium rule-editor integration test

Use this pattern to verify show/hide and validation rules fire correctly in the browser.

> ⚠️ **Prerequisites — this does NOT belong in `core` and needs setup the project does
> not currently have:**
> - Place IT classes in the **`it.tests`** (or `ui.tests`) module, not `core` — `core`
>   is unit-test only.
> - Add `org.seleniumhq.selenium:selenium-java` (test scope) to that module — it is not
>   in any project pom today.
> - The `-Pit` profile and a `maven-failsafe-plugin` execution must be wired in that
>   module (the parent pom defines failsafe only in `pluginManagement`, bound to nothing,
>   and has no `-Pit` profile). Without this, `mvn verify -Pit` runs nothing.
> - The form renders at the published Core Components URL; confirm `FORM_PATH` against
>   your actual form (CC forms are authored under the DAM forms root and render via their
>   container path).

```java
package com.{package}.forms.it;

import org.junit.jupiter.api.*;
import org.openqa.selenium.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.support.ui.*;

import java.time.Duration;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Integration test — requires a running AEM SDK instance + Selenium on the classpath.
 * Run with: mvn verify -Pit -Daem.host=localhost -Daem.port=4502
 */
@Tag("integration")
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class FormRuleEditorIT {

    private static WebDriver driver;
    private static WebDriverWait wait;

    private static final String AEM_HOST = System.getProperty("aem.host", "localhost");
    private static final String AEM_PORT = System.getProperty("aem.port", "4502");
    private static final String FORM_PATH = "/content/forms/af/{formName}";

    @BeforeAll
    static void setUpDriver() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--headless", "--no-sandbox", "--disable-dev-shm-usage");
        driver = new ChromeDriver(options);
        wait   = new WebDriverWait(driver, Duration.ofSeconds(10));
        driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(5));
    }

    @AfterAll
    static void tearDown() {
        if (driver != null) driver.quit();
    }

    @BeforeEach
    void loadForm() {
        driver.get("http://" + AEM_HOST + ":" + AEM_PORT + FORM_PATH + ".html");
        // Wait for form container to be present
        wait.until(ExpectedConditions.presenceOfElementLocated(
            By.cssSelector(".cmp-adaptiveform-container")));
    }

    @Test
    @Order(1)
    @DisplayName("Company name panel hidden by default when employment type is not selected")
    void testCompanyNameHiddenByDefault() {
        WebElement companyPanel = driver.findElement(
            By.cssSelector("[name='companyName']"));
        assertFalse(companyPanel.isDisplayed(),
            "Company name should be hidden when employment type is not selected");
    }

    @Test
    @Order(2)
    @DisplayName("Company name panel shows when employment type is set to salaried")
    void testCompanyNameShowsOnEmploymentTypeChange() {
        // Select employment type
        Select dropdown = new Select(driver.findElement(
            By.cssSelector("[name='employmentType']")));
        dropdown.selectByValue("salaried");

        // Wait for show animation
        WebElement companyPanel = wait.until(
            ExpectedConditions.visibilityOfElementLocated(
                By.cssSelector("[name='companyName']")));

        assertTrue(companyPanel.isDisplayed(),
            "Company name must be visible after selecting 'salaried'");
    }

    @Test
    @Order(3)
    @DisplayName("Required field shows validation error on empty submit")
    void testRequiredFieldShowsValidationError() {
        // Click submit without filling required fields
        driver.findElement(By.cssSelector("button[type='submit']")).click();

        WebElement errorMsg = wait.until(
            ExpectedConditions.visibilityOfElementLocated(
                By.cssSelector("[name='firstName'] .cmp-adaptiveform-textinput__errormessage")));

        assertFalse(errorMsg.getText().isEmpty(),
            "Required field error message must be shown on empty submit");
    }

    @Test
    @Order(4)
    @DisplayName("Submit is gated on validation — invalid form produces NO PDF, blocks submission")
    void testSubmitGatedOnValidation() {
        // Clicking Submit on an invalid form must BLOCK submission (no thank-you, no PDF/download)
        // and show inline errors. Only a valid form submits AND generates the PDF.
        driver.findElement(By.cssSelector("button[type='submit']")).click();
        // Still on the form (validation errors present) — not on the thank-you page.
        assertFalse(driver.findElements(
                By.cssSelector(".cmp-adaptiveform-textinput__errormessage")).isEmpty(),
            "Invalid submit must show validation errors and not proceed (no submit, no PDF)");
        // Assert no PDF was generated/downloaded (e.g. check the download dir / servlet was not hit).
    }
}
```

---

## Quality checklist

Before delivering:

- [ ] Submit-action test mocks `FormSubmitInfo` (not `AdaptiveFormSubmitInfo`), `getData()` returns a `String`, calls `submit(...)` — no `javax.json`, no `getRequest()`/`getThankYouPage()`
- [ ] Prefill test targets `getPrefillData(DataOptions)` returning `PrefillData` — not a request
- [ ] Sling Model test uses `AppAemContext.newAemContext()` (CACONFIG + CORE_COMPONENTS plugins), not a bare `new AemContext()`
- [ ] Fixture `sling:resourceType` matches the deployed proxy (`{project}/components/adaptiveForm/...`)
- [ ] Every test has a `@DisplayName` describing expected behaviour
- [ ] JSON fixture file created alongside unit tests
- [ ] Every service test covers: happy path, null/empty input, error/failure path
- [ ] Selenium IT lives in `it.tests`/`ui.tests` with the Selenium dep + `-Pit` profile wired (not `core`)
- [ ] No `Thread.sleep()` in Selenium tests — use `WebDriverWait`
- [ ] Integration tests annotated with `@Tag("integration")`
- [ ] Assertions check meaningful values, not just non-null
- [ ] A **submit-gated-on-validation** test exists (user story "submission only succeeds when
      validation passes"): invalid/empty form blocks submission, shows inline errors, produces NO
      PDF/DoR; valid form submits AND generates the PDF
