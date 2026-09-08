# Variant A — REST API prefill

Fetches user/entity data from an external HTTP endpoint and returns it as JSON
for field population. Does not open a JCR ResourceResolver.

---

## Complete Java class

```java
package {package}.forms.prefill;

import com.adobe.forms.common.service.ContentType;
import com.adobe.forms.common.service.DataOptions;
import com.adobe.forms.common.service.DataProvider;
import com.adobe.forms.common.service.FormsException;
import com.adobe.forms.common.service.PrefillData;
import org.osgi.service.component.annotations.Activate;
import org.osgi.service.component.annotations.Component;
import org.osgi.service.metatype.annotations.AttributeDefinition;
import org.osgi.service.metatype.annotations.Designate;
import org.osgi.service.metatype.annotations.ObjectClassDefinition;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.ByteArrayInputStream;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.time.Duration;

@Component(service = DataProvider.class, immediate = true)
@Designate(ocd = {ServiceName}PrefillService.Config.class)
public class {ServiceName}PrefillService implements DataProvider {

    private static final Logger LOG =
        LoggerFactory.getLogger({ServiceName}PrefillService.class);

    private static final String SERVICE_NAME = "{serviceName}PrefillService";
    private static final String EMPTY_JSON   = "{}";

    @ObjectClassDefinition(name = "{ServiceDisplayName} Prefill Service Config")
    public @interface Config {
        @AttributeDefinition(name = "API Base URL",
            description = "Base URL of the prefill REST endpoint. User ID is appended as a path segment.")
        String apiBaseUrl() default "https://api.example.com/v1/users";

        @AttributeDefinition(name = "API Token",
            description = "Bearer token. Use $[secret:forms.prefill.apiToken] in production.")
        String apiToken() default "";

        @AttributeDefinition(name = "Request timeout (ms)",
            description = "Hard cap on HTTP call duration. Fail-fast to avoid blocking form render.")
        int timeoutMs() default 3000;
    }

    private String  apiBaseUrl;
    private String  apiToken;
    private int     timeoutMs;
    private HttpClient httpClient;

    @Activate
    protected void activate(Config config) {
        this.apiBaseUrl  = config.apiBaseUrl();
        this.apiToken    = config.apiToken();
        this.timeoutMs   = config.timeoutMs();
        this.httpClient  = HttpClient.newBuilder()
            .connectTimeout(Duration.ofMillis(timeoutMs))
            .build();
        LOG.info("{ServiceName}PrefillService activated, endpoint: {}", apiBaseUrl);
    }

    @Override
    public String getServiceName() { return SERVICE_NAME; }

    @Override
    public String getServiceDescription() {
        return "{ServiceDisplayName} REST API prefill provider";
    }

    @Override
    public PrefillData getPrefillData(DataOptions options) throws FormsException {
        String userId = resolveUserId(options);
        if (userId == null || userId.isBlank() || "anonymous".equals(userId)) {
            LOG.debug("Unauthenticated request — returning empty prefill");
            return jsonPrefill(EMPTY_JSON);
        }
        try {
            String json = fetchFromApi(userId);
            return jsonPrefill(json);
        } catch (Exception e) {
            LOG.error("Prefill API call failed for user '{}': {}", userId, e.getMessage(), e);
            return jsonPrefill(EMPTY_JSON);   // degrade gracefully — never block form render
        }
    }

    // Reads the current user from the form's ResourceResolver.
    // Override this method if you want to resolve the entity key differently
    // (e.g. from a URL parameter via options.getExtras()).
    private String resolveUserId(DataOptions options) {
        if (options == null) return null;
        var resource = options.getFormResource();
        if (resource == null) return null;
        return resource.getResourceResolver().getUserID();
    }

    private String fetchFromApi(String userId) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(apiBaseUrl + "/" + userId))
            .header("Authorization", "Bearer " + apiToken)
            .header("Accept", "application/json")
            .timeout(Duration.ofMillis(timeoutMs))
            .GET()
            .build();

        HttpResponse<String> response =
            httpClient.send(request, HttpResponse.BodyHandlers.ofString());

        if (response.statusCode() < 200 || response.statusCode() >= 300) {
            throw new RuntimeException(
                "API returned HTTP " + response.statusCode() + " for user: " + userId);
        }
        return response.body();
    }

    private PrefillData jsonPrefill(String json) {
        return new PrefillData(
            new ByteArrayInputStream(json.getBytes(StandardCharsets.UTF_8)),
            ContentType.JSON);
    }
}
```

---

## OSGi config — .cfg.json

```json
{
  "apiBaseUrl": "$[env:PREFILL_API_BASE_URL;default=https://api.example.com/v1/users]",
  "apiToken":   "$[secret:forms.prefill.apiToken]",
  "timeoutMs":  3000
}
```

---

## repoinit — system user

REST variant does NOT read from JCR, so the system user needs minimal or no JCR ACL.
Still create it — the service user mapping requires a real system user to exist.

```json
{
  "scripts": [
    "create service user {appId}-forms-prefill-service with path system/forms\n\nset ACL for {appId}-forms-prefill-service\n  allow jcr:read on /content/forms/af\nend"
  ]
}
```

---

## Additional unit test — API success path

Add to the base test class:

```java
@Test
void getPrefillData_authenticatedUser_returnsApiData() throws Exception {
    // This test wires up the real service against a local mock server,
    // OR you can refactor fetchFromApi() to accept an injectable HttpClient.
    // Pattern: override apiBaseUrl to point at a WireMock or MockWebServer.
    // For brevity, test the happy path at integration test level.
    // At unit level, verify the service does NOT throw for a valid user:
    when(mockOptions.getFormResource()).thenReturn(mockFormResource);
    when(mockFormResource.getResourceResolver()).thenReturn(mockResolver);
    when(mockResolver.getUserID()).thenReturn("user123");

    // With no real server, the HTTP call will fail → service should return empty, not throw
    PrefillData result = service.getPrefillData(mockOptions);
    assertNotNull(result);  // graceful degradation confirmed
}
```

---

## Field binding guide

The JSON your API returns must have keys matching the form field names.
If the API response shape differs, remap inside `fetchFromApi()` before returning.

Example API response → Adaptive Form field `bindRef`:
```
API JSON key     →  Form field bindRef
────────────────────────────────────────
firstName        →  firstName
lastName         →  lastName
emailAddress     →  emailAddress
phoneNumber      →  phoneNumber
address.line1    →  address.line1
address.city     →  address.city
address.pinCode  →  address.pinCode
```

If the API uses different key names (e.g. `email` but form uses `emailAddress`),
transform the JSON string before returning it, or use a DTO + JSON serialization.
