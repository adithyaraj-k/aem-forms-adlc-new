# Common patterns — all prefill variants

## Verified SPI (AEM Forms SDK for AEMaaCS)

The correct interface for AEM as a Cloud Service:

```java
package com.adobe.forms.common.service;

public interface DataProviderBase {
    String getServiceName();
    String getServiceDescription();
}

public interface DataProvider extends DataProviderBase {
    PrefillData getPrefillData(DataOptions options) throws FormsException;
}
```

`DataOptions` useful getters:
- `getFormResource()` — the `Resource` for the form; its `ResourceResolver` gives you the current user via `getUserID()`
- `getDataRef()` — the `dataRef` attribute on the form container (used to pass a CRX path or draft ID)
- `getExtras()` — a `Map<String, Object>` for additional parameters
- `getPagePath()`, `getAemFormContainer()`, `getServiceName()`, `getContentType()`

`PrefillData` construction:
```java
new PrefillData(InputStream inputStream, ContentType contentType)
// ContentType enum values: JSON, XML
```

For AEMaaCS with Core Components forms → always use `ContentType.JSON`.
The legacy `afData / afBoundData` XML envelope is only needed for XFA / Foundation Components.

---

## Base class skeleton (all variants share this shell)

```java
package {package}.forms.prefill;

import com.adobe.forms.common.service.ContentType;
import com.adobe.forms.common.service.DataOptions;
import com.adobe.forms.common.service.DataProvider;
import com.adobe.forms.common.service.FormsException;
import com.adobe.forms.common.service.PrefillData;
import org.osgi.service.component.annotations.Activate;
import org.osgi.service.component.annotations.Component;
import org.osgi.service.metatype.annotations.Designate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.ByteArrayInputStream;
import java.nio.charset.StandardCharsets;

@Component(service = DataProvider.class, immediate = true)
@Designate(ocd = {ServiceName}PrefillService.Config.class)
public class {ServiceName}PrefillService implements DataProvider {

    private static final Logger LOG =
        LoggerFactory.getLogger({ServiceName}PrefillService.class);

    // This value is what you set in the form container's "Prefill Service" field.
    // It must be unique across all prefill services deployed to the same AEM instance.
    private static final String SERVICE_NAME = "{serviceName}PrefillService";

    private static final String EMPTY_JSON = "{}";

    // --- variant-specific @Reference fields and @Activate config go here ---

    @Override
    public String getServiceName() {
        return SERVICE_NAME;
    }

    @Override
    public String getServiceDescription() {
        return "{Service Display Name} prefill provider";
    }

    @Override
    public PrefillData getPrefillData(DataOptions options) throws FormsException {
        // --- variant-specific implementation goes here ---
        return jsonPrefill(EMPTY_JSON);
    }

    // Shared helper: wraps any JSON string in a PrefillData object.
    protected PrefillData jsonPrefill(String json) {
        String safeJson = (json != null && !json.isBlank()) ? json : EMPTY_JSON;
        return new PrefillData(
            new ByteArrayInputStream(safeJson.getBytes(StandardCharsets.UTF_8)),
            ContentType.JSON);
    }
}
```

---

## OSGi configuration (.cfg.json format — AEMaaCS only)

File path pattern:
```
ui.config/src/main/content/jcr_root/apps/{appId}/config/
  com.{package}.forms.prefill.{ServiceName}PrefillService.cfg.json
```

General rules:
- All config files must be `.cfg.json`. XML sling:OsgiConfig is not supported on AEMaaCS.
- Secrets use `$[secret:key-name]` — Cloud Manager Secret Environment Variables supply the value.
- Non-secret strings use `$[env:VAR_NAME;default=fallback]` for environment-specific values.
- For run-mode-specific config (author vs publish), use sub-folders:
  `config.author/` or `config.publish/` instead of `config/`.

Example:
```json
{
  "timeoutMs": 3000
}
```

---

## System user — repoinit (required on AEMaaCS)

You cannot manually create JCR users on AEMaaCS. Declare everything in repoinit.

File path:
```
ui.config/src/main/content/jcr_root/apps/{appId}/config/
  org.apache.sling.jcr.repoinit.RepositoryInitializer~{appId}-forms-prefill.cfg.json
```

```json
{
  "scripts": [
    "create service user {appId}-forms-prefill-service with path system/forms\n\ncreate path (sling:Folder) {packageCreatedPath}\n\nset ACL for {appId}-forms-prefill-service\n  allow jcr:read on {alwaysExistingPath}\n  allow jcr:read on {packageCreatedPath}\nend"
  ]
}
```

Important: the `scripts` value is a JSON array of strings. Newlines inside the
repoinit script must be `\n` (JSON-escaped). Each `set ACL` block must end with `end`.

⚠️ **`set ACL` on a non-existent path aborts the whole script.** repoinit runs at
startup, before content packages create paths like `/content/dam/{appId}/prefilldata`
or `/var/fd/dashboard/data/drafts`. If `set ACL` targets a path that doesn't exist yet,
it throws and the earlier `create service user` never commits — the user is silently
missing and prefill fails with `LoginException: Cannot derive user name … sub service
forms-prefill-service`. **Emit `create path (sling:Folder) <path>` for every
package-created / runtime path BEFORE the `set ACL` block.** Paths that always exist at
startup (e.g. `/content/forms/af`) need no `create path`. Verify post-deploy with a
querybuilder lookup on `/home/users/system` for the principal (`total:1`).

---

## Service user mapping (.cfg.json)

File path:
```
ui.config/src/main/content/jcr_root/apps/{appId}/config/
  org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended~{appId}-forms.cfg.json
```

```json
{
  "user.mapping": [
    "{bundleSymbolicName}:forms-prefill-service={appId}-forms-prefill-service"
  ]
}
```

The subservice name `forms-prefill-service` is what you pass to
`ResourceResolverFactory.SUBSERVICE` in your Java code. The right side of `=` is
the system user name declared in repoinit.

---

## ResourceResolver — always try-with-resources

Only variant B (CRX/DAM) and C (JCR Draft) open a ResourceResolver.
Variant A (REST API) does not need one and must not open one unnecessarily.

```java
// In the Java class — reference field:
@Reference
private ResourceResolverFactory resolverFactory;

private static final String SUBSERVICE = "forms-prefill-service";

// Usage — always try-with-resources:
Map<String, Object> authInfo = Collections.singletonMap(
    ResourceResolverFactory.SUBSERVICE, SUBSERVICE);

try (ResourceResolver resolver =
        resolverFactory.getServiceResourceResolver(authInfo)) {
    // use resolver here
} catch (LoginException e) {
    LOG.error("Cannot obtain service resolver", e);
    return jsonPrefill(EMPTY_JSON);
}
// resolver is auto-closed — no finally block needed or allowed
```

---

## Unit test — base structure

Uses wcm.io AEM Mocks + Mockito. Always mock `DataOptions` — do not instantiate it
directly (it is not a plain POJO).

```java
package {package}.forms.prefill;

import com.adobe.forms.common.service.DataOptions;
import com.adobe.forms.common.service.PrefillData;
import io.wcm.testing.mock.aem.junit5.AemContext;
import io.wcm.testing.mock.aem.junit5.AemContextExtension;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceResolver;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.io.InputStream;
import java.nio.charset.StandardCharsets;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith({AemContextExtension.class, MockitoExtension.class})
class {ServiceName}PrefillServiceTest {

    private final AemContext ctx = new AemContext();

    @Mock private DataOptions mockOptions;
    @Mock private Resource mockFormResource;
    @Mock private ResourceResolver mockResolver;

    private {ServiceName}PrefillService service;

    @BeforeEach
    void setUp() {
        // registerInjectActivateService handles @Activate and injects config values.
        // Add variant-specific config properties as additional String pairs here.
        service = ctx.registerInjectActivateService(
            new {ServiceName}PrefillService()
            // e.g. "apiEndpoint", "https://test.example.com"
        );
    }

    @Test
    void getServiceName_returnsExpectedId() {
        assertEquals("{serviceName}PrefillService", service.getServiceName());
    }

    @Test
    void getServiceDescription_isNotBlank() {
        assertNotNull(service.getServiceDescription());
        assertFalse(service.getServiceDescription().isBlank());
    }

    @Test
    void getPrefillData_nullOptions_returnsEmptyJsonGracefully() throws Exception {
        PrefillData result = service.getPrefillData(null);
        assertNotNull(result);
        assertNotNull(result.getInputStream());
        String body = new String(
            result.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
        // Must be valid JSON and not throw; exact content may vary
        assertFalse(body.isBlank());
    }

    @Test
    void getPrefillData_anonymousUser_returnsEmptyJson() throws Exception {
        when(mockOptions.getFormResource()).thenReturn(mockFormResource);
        when(mockFormResource.getResourceResolver()).thenReturn(mockResolver);
        when(mockResolver.getUserID()).thenReturn("anonymous");

        PrefillData result = service.getPrefillData(mockOptions);
        assertNotNull(result);
        String body = new String(
            result.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
        assertEquals("{}", body);
    }
}
```

Test dependencies to add to `core/pom.xml` (scope test):
```xml
<dependency>
  <groupId>io.wcm.testing.aem</groupId>
  <artifactId>io.wcm.testing.aem-mock.junit5</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-junit-jupiter</artifactId>
  <scope>test</scope>
</dependency>
```

---

## Form wiring — how to connect the service to a form

After deploying the bundle:

**Option A — via AEM Forms UI:**
1. Open the Adaptive Form in Edit mode.
2. Click the root "Guide Container" → wrench icon → Edit.
3. Go to the "Prefill Service" tab.
4. Select your service by its description from the dropdown.
5. Save.

**Option B — via CRX/DE directly:**
Navigate to: `/content/forms/af/{form-name}/jcr:content/guideContainer`
Set property:
- Name: `prefillService`
- Type: `String`
- Value: `{serviceName}PrefillService` ← must match `getServiceName()` exactly

---

## Verification checklist

After deployment, verify in this order:

1. **Bundle active** — `/system/console/bundles` → search your bundle name → status must be `Active`
2. **Service registered** — `/system/console/components` → search `{ServiceName}PrefillService` → state `active`
3. **System user exists** — `/crx/de` → navigate to `/home/users/system/forms/{appId}-forms-prefill-service`
4. **Service mapping** — `/system/console/serviceusers` → confirm your subservice maps to the system user
5. **Prefill fires** — open the form URL in browser → bound fields should be pre-filled on load
6. **Logs** — `/system/console/slinglog` → set DEBUG on `{package}.forms.prefill` → watch for errors
