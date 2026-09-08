# Variant B — CRX / DAM JSON prefill

Reads a JSON file stored in the AEM repository (DAM or JCR) and returns its
contents for field population. Uses a service ResourceResolver — resolver
lifecycle is critical.

---

## How path resolution works

There are two sub-modes. Decide which applies and generate accordingly:

**B1 — Static path**: the same JSON file is used for every form load.
Set the path in OSGi config. Useful for demo, shared reference data, or lookup tables.

**B2 — Dynamic path**: a different JSON file is loaded per user (or per request parameter).
The path is constructed at runtime — typically by appending the user ID to a base path.
E.g. `/content/dam/myproject/prefilldata/user123.json`

Ask the developer which sub-mode they need if they haven't specified.

> ✅ **For the Excel-file / manual-values "fill the form on every render" case (the SKILL.md
> Step 0 default), use B1 — Static path.** Build the service so it returns the JSON
> **unconditionally**, NOT gated on the current user — the B2 pattern returns `{}` for
> anonymous users, so an anonymously-rendered form (e.g. embedded on a public Sites page)
> would come up EMPTY. The static code below reads `staticJsonPath` on every call regardless
> of user, which is what "prefill on every render" requires.

---

## Complete Java class (B2 dynamic path — most common)

```java
package {package}.forms.prefill;

import com.adobe.forms.common.service.ContentType;
import com.adobe.forms.common.service.DataOptions;
import com.adobe.forms.common.service.DataProvider;
import com.adobe.forms.common.service.FormsException;
import com.adobe.forms.common.service.PrefillData;
import org.apache.sling.api.resource.LoginException;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceResolver;
import org.apache.sling.api.resource.ResourceResolverFactory;
import org.osgi.service.component.annotations.Activate;
import org.osgi.service.component.annotations.Component;
import org.osgi.service.component.annotations.Reference;
import org.osgi.service.metatype.annotations.AttributeDefinition;
import org.osgi.service.metatype.annotations.Designate;
import org.osgi.service.metatype.annotations.ObjectClassDefinition;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.ByteArrayInputStream;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.Collections;
import java.util.Map;

@Component(service = DataProvider.class, immediate = true)
@Designate(ocd = {ServiceName}PrefillService.Config.class)
public class {ServiceName}PrefillService implements DataProvider {

    private static final Logger LOG =
        LoggerFactory.getLogger({ServiceName}PrefillService.class);

    private static final String SERVICE_NAME = "{serviceName}PrefillService";
    private static final String SUBSERVICE   = "forms-prefill-service";
    private static final String EMPTY_JSON   = "{}";

    @ObjectClassDefinition(name = "{ServiceDisplayName} Prefill Service Config")
    public @interface Config {
        @AttributeDefinition(name = "DAM base path",
            description = "Base DAM path where per-user JSON files are stored. " +
                          "The resolved user ID is appended as '/{userId}.json/jcr:content'.")
        String damBasePath() default "/content/dam/myproject/prefilldata";
    }

    private String damBasePath;

    @Reference
    private ResourceResolverFactory resolverFactory;

    @Activate
    protected void activate(Config config) {
        this.damBasePath = config.damBasePath();
        LOG.info("{ServiceName}PrefillService activated, DAM base: {}", damBasePath);
    }

    @Override
    public String getServiceName() { return SERVICE_NAME; }

    @Override
    public String getServiceDescription() {
        return "{ServiceDisplayName} CRX/DAM JSON prefill provider";
    }

    @Override
    public PrefillData getPrefillData(DataOptions options) throws FormsException {
        String userId = resolveUserId(options);
        if (userId == null || userId.isBlank() || "anonymous".equals(userId)) {
            LOG.debug("Unauthenticated request — returning empty prefill");
            return jsonPrefill(EMPTY_JSON);
        }

        // Build the full JCR content node path for the user's JSON asset in DAM.
        // DAM assets have their binary at: {assetPath}/jcr:content/renditions/original/jcr:content
        // For a raw JSON file uploaded to DAM the binary is at: {assetPath}/jcr:content
        String jsonNodePath = damBasePath + "/" + userId + ".json/jcr:content";

        Map<String, Object> authInfo = Collections.singletonMap(
            ResourceResolverFactory.SUBSERVICE, SUBSERVICE);

        try (ResourceResolver resolver =
                resolverFactory.getServiceResourceResolver(authInfo)) {

            Resource jsonContent = resolver.getResource(jsonNodePath);
            if (jsonContent == null) {
                LOG.warn("Prefill JSON not found at: {}", jsonNodePath);
                return jsonPrefill(EMPTY_JSON);
            }

            // adaptTo(InputStream) reads the jcr:data binary of an nt:resource node
            InputStream stream = jsonContent.adaptTo(InputStream.class);
            if (stream == null) {
                LOG.warn("Cannot read binary from node: {}", jsonNodePath);
                return jsonPrefill(EMPTY_JSON);
            }

            String json = new String(stream.readAllBytes(), StandardCharsets.UTF_8);
            LOG.debug("Prefill data loaded for user '{}', {} bytes", userId, json.length());
            return jsonPrefill(json);

        } catch (LoginException e) {
            LOG.error("Service resolver login failed — check system user mapping", e);
            return jsonPrefill(EMPTY_JSON);
        } catch (Exception e) {
            LOG.error("Prefill failed for user '{}': {}", userId, e.getMessage(), e);
            return jsonPrefill(EMPTY_JSON);
        }
        // resolver is auto-closed by try-with-resources
    }

    private String resolveUserId(DataOptions options) {
        if (options == null) return null;
        var resource = options.getFormResource();
        if (resource == null) return null;
        return resource.getResourceResolver().getUserID();
    }

    private PrefillData jsonPrefill(String json) {
        return new PrefillData(
            new ByteArrayInputStream(json.getBytes(StandardCharsets.UTF_8)),
            ContentType.JSON);
    }
}
```

---

## Static path variant (B1) — simpler version

For static path, replace the `getPrefillData` body with:

```java
@Override
public PrefillData getPrefillData(DataOptions options) throws FormsException {
    // staticJsonPath comes from OSGi config, e.g. /content/dam/myproject/prefilldata/defaults.json/jcr:content
    Map<String, Object> authInfo = Collections.singletonMap(
        ResourceResolverFactory.SUBSERVICE, SUBSERVICE);

    try (ResourceResolver resolver =
            resolverFactory.getServiceResourceResolver(authInfo)) {

        Resource jsonContent = resolver.getResource(staticJsonPath);
        if (jsonContent == null) {
            LOG.warn("Static prefill JSON not found: {}", staticJsonPath);
            return jsonPrefill(EMPTY_JSON);
        }
        InputStream stream = jsonContent.adaptTo(InputStream.class);
        if (stream == null) return jsonPrefill(EMPTY_JSON);
        return jsonPrefill(new String(stream.readAllBytes(), StandardCharsets.UTF_8));

    } catch (LoginException e) {
        LOG.error("Service resolver login failed", e);
        return jsonPrefill(EMPTY_JSON);
    } catch (Exception e) {
        LOG.error("Static prefill read failed: {}", e.getMessage(), e);
        return jsonPrefill(EMPTY_JSON);
    }
}
```

---

## OSGi config — .cfg.json

```json
{
  "damBasePath": "$[env:PREFILL_DAM_BASE_PATH;default=/content/dam/myproject/prefilldata]"
}
```

---

## repoinit — system user (requires DAM read access)

```json
{
  "scripts": [
    "create service user {appId}-forms-prefill-service with path system/forms\n\ncreate path (sling:Folder) /content/dam/{appId}/prefilldata\n\nset ACL for {appId}-forms-prefill-service\n  allow jcr:read on /content/forms/af\n  allow jcr:read on /content/dam/{appId}/prefilldata\nend"
  ]
}
```

Scope the DAM ACL to only the prefill data folder — not all of `/content/dam`.

> ⚠️ **`create path` before `set ACL` is mandatory.** repoinit runs at startup, before the
> `ui.content` package installs `/content/dam/{appId}/prefilldata`. `set ACL` on a
> not-yet-existing path throws and aborts the whole script, so the `create service user`
> never commits — the user is silently missing and every prefill fails with
> `LoginException: Cannot derive user name … sub service forms-prefill-service`, returning
> `{}`. Emit a `create path (sling:Folder) …` line for every package-created / runtime path
> (the DAM prefill folder, and `/var/fd/dashboard/data/drafts` for a draft provider) BEFORE
> the ACL block. `/content/forms/af` already exists at startup. After deploy, confirm the
> user exists: `GET /bin/querybuilder.json?path=/home/users/system&property=rep:principalName&property.value={appId}-forms-prefill-service`
> → `total:1`. If the project already ships a prefill user + mapping, REUSE them (just add
> the `create path` + ACL for the new folder if needed) rather than creating a duplicate mapping.

---

## JSON file format

The JSON file stored in DAM must have keys matching the form field names exactly.
There is no `afData` wrapper needed — return the data object directly.

```json
{
  "firstName": "Arun",
  "lastName":  "Kumar",
  "email":     "arun.kumar@example.com",
  "mobile":    "9876543210",
  "address": {
    "street": "12 Anna Salai",
    "city":   "Chennai",
    "state":  "Tamil Nadu",
    "pin":    "600002"
  }
}
```

---

## DAM upload instructions for developers

1. Go to AEM Assets → Files → navigate to `{damBasePath}`.
2. Upload the JSON file named `{userId}.json` (for dynamic) or any name (for static).
3. Ensure the asset has a `jcr:content` child node with `jcr:mimeType = application/json`
   and `jcr:data` containing the binary. DAM upload via UI handles this automatically.
4. Verify via CRX/DE: `{damBasePath}/{userId}.json/jcr:content` should exist with `jcr:data`.

---

## Additional unit test — resource found path

```java
@Mock private org.apache.sling.api.resource.ResourceResolverFactory mockRRF;
@Mock private Resource mockJsonContent;

@Test
void getPrefillData_jsonFoundInDam_returnsPopulatedData() throws Exception {
    String sampleJson = "{\"firstName\":\"Arun\",\"lastName\":\"Kumar\"}";
    InputStream jsonStream = new ByteArrayInputStream(
        sampleJson.getBytes(StandardCharsets.UTF_8));

    when(mockOptions.getFormResource()).thenReturn(mockFormResource);
    when(mockFormResource.getResourceResolver()).thenReturn(mockResolver);
    when(mockResolver.getUserID()).thenReturn("user123");
    when(mockRRF.getServiceResourceResolver(any())).thenReturn(mockResolver);
    when(mockResolver.getResource(anyString())).thenReturn(mockJsonContent);
    when(mockJsonContent.adaptTo(InputStream.class)).thenReturn(jsonStream);

    PrefillData result = service.getPrefillData(mockOptions);
    assertNotNull(result);
    String body = new String(
        result.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
    assertTrue(body.contains("Arun"));
}

@Test
void getPrefillData_jsonNotFoundInDam_returnsEmptyJson() throws Exception {
    when(mockOptions.getFormResource()).thenReturn(mockFormResource);
    when(mockFormResource.getResourceResolver()).thenReturn(mockResolver);
    when(mockResolver.getUserID()).thenReturn("user999");
    when(mockRRF.getServiceResourceResolver(any())).thenReturn(mockResolver);
    when(mockResolver.getResource(anyString())).thenReturn(null);

    PrefillData result = service.getPrefillData(mockOptions);
    String body = new String(
        result.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
    assertEquals("{}", body);
}
```
