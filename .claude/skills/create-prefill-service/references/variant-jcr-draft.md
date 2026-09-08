# Variant C — JCR Draft prefill

Loads a previously saved form draft from the AEM JCR (typically under
`/var/fd/dashboard/data/drafts/`). The draft ID is passed to the service
via the form container's `dataRef` attribute.

---

## How draft ID resolution works

The form container has a `dataRef` property. When a user returns to a draft,
the URL typically contains the draft path or ID. The prefill service reads it via:

```java
String dataRef = options.getDataRef();
// dataRef might be: "service://FDR?draftid=abc123"
// or a direct JCR path: "/var/fd/dashboard/data/drafts/abc123"
```

Two common formats — handle both:
1. **Direct path** — `dataRef` is the full JCR path to the draft node.
2. **Service URL** — `dataRef` is `service://{serviceName}?draftid={id}`. Parse out the ID.

The skill defaults to direct-path mode (simpler and most common for custom implementations).
If the developer uses the built-in AEM Forms Draft Save service, they likely need the URL-param mode.

---

## Complete Java class

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
import org.apache.sling.api.resource.ValueMap;
import org.osgi.service.component.annotations.Activate;
import org.osgi.service.component.annotations.Component;
import org.osgi.service.component.annotations.Reference;
import org.osgi.service.metatype.annotations.AttributeDefinition;
import org.osgi.service.metatype.annotations.Designate;
import org.osgi.service.metatype.annotations.ObjectClassDefinition;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.ByteArrayInputStream;
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

    // Property name where the draft's form data is stored on the draft JCR node.
    // AEM Forms default is "fd:data"; adjust if using a custom draft storage strategy.
    private static final String DATA_PROPERTY = "fd:data";

    @ObjectClassDefinition(name = "{ServiceDisplayName} Draft Prefill Service Config")
    public @interface Config {
        @AttributeDefinition(name = "Draft base path",
            description = "JCR path under which draft nodes are stored.")
        String draftBasePath() default "/var/fd/dashboard/data/drafts";
    }

    private String draftBasePath;

    @Reference
    private ResourceResolverFactory resolverFactory;

    @Activate
    protected void activate(Config config) {
        this.draftBasePath = config.draftBasePath();
        LOG.info("{ServiceName}PrefillService activated, draft base: {}", draftBasePath);
    }

    @Override
    public String getServiceName() { return SERVICE_NAME; }

    @Override
    public String getServiceDescription() {
        return "{ServiceDisplayName} JCR draft prefill provider";
    }

    @Override
    public PrefillData getPrefillData(DataOptions options) throws FormsException {
        if (options == null) {
            LOG.debug("Null DataOptions — returning empty prefill");
            return jsonPrefill(EMPTY_JSON);
        }

        String draftPath = resolveDraftPath(options);
        if (draftPath == null || draftPath.isBlank()) {
            LOG.debug("No draft path in dataRef — returning empty prefill");
            return jsonPrefill(EMPTY_JSON);
        }

        Map<String, Object> authInfo = Collections.singletonMap(
            ResourceResolverFactory.SUBSERVICE, SUBSERVICE);

        try (ResourceResolver resolver =
                resolverFactory.getServiceResourceResolver(authInfo)) {

            Resource draftResource = resolver.getResource(draftPath);
            if (draftResource == null) {
                LOG.warn("Draft node not found: {}", draftPath);
                return jsonPrefill(EMPTY_JSON);
            }

            ValueMap properties = draftResource.getValueMap();
            String savedData = properties.get(DATA_PROPERTY, String.class);

            if (savedData == null || savedData.isBlank()) {
                LOG.warn("Draft found but '{}' property is empty: {}", DATA_PROPERTY, draftPath);
                return jsonPrefill(EMPTY_JSON);
            }

            LOG.debug("Draft loaded from: {}, {} bytes", draftPath, savedData.length());
            return jsonPrefill(savedData);

        } catch (LoginException e) {
            LOG.error("Service resolver login failed — check system user mapping", e);
            return jsonPrefill(EMPTY_JSON);
        } catch (Exception e) {
            LOG.error("Draft prefill failed for path '{}': {}", draftPath, e.getMessage(), e);
            return jsonPrefill(EMPTY_JSON);
        }
        // resolver is auto-closed by try-with-resources
    }

    /**
     * Resolves the JCR draft node path from the form's dataRef.
     *
     * Supports two formats:
     *   1. Direct path:    /var/fd/dashboard/data/drafts/abc123
     *   2. Service URL:    service://{serviceName}?draftid=abc123
     *      → resolves to:  {draftBasePath}/abc123
     */
    private String resolveDraftPath(DataOptions options) {
        String dataRef = options.getDataRef();
        if (dataRef == null || dataRef.isBlank()) return null;

        // Format 1: already a direct JCR path
        if (dataRef.startsWith("/")) {
            return dataRef;
        }

        // Format 2: service://... URL — extract draftid query param
        if (dataRef.contains("draftid=")) {
            String draftId = dataRef.substring(dataRef.indexOf("draftid=") + 8);
            // Strip any additional query params after the draftId
            int ampIdx = draftId.indexOf('&');
            if (ampIdx > 0) draftId = draftId.substring(0, ampIdx);
            return draftBasePath + "/" + draftId;
        }

        // Format 3: bare ID — append to base path
        return draftBasePath + "/" + dataRef;
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
  "draftBasePath": "/var/fd/dashboard/data/drafts"
}
```

---

## repoinit — system user

The system user needs read access to the draft storage path.

```json
{
  "scripts": [
    "create service user {appId}-forms-prefill-service with path system/forms\n\nset ACL for {appId}-forms-prefill-service\n  allow jcr:read on /var/fd/dashboard/data/drafts\n  allow jcr:read on /content/forms/af\nend"
  ]
}
```

---

## Draft node structure (for reference)

When AEM Forms saves a draft, the JCR node looks like:

```
/var/fd/dashboard/data/drafts/
  └── {draftId}/
        jcr:primaryType = nt:unstructured
        fd:data          = "{\"firstName\":\"Arun\",...}"   ← JSON string
        fd:formPath      = /content/forms/af/my-form
        fd:submitServiceName = ...
        jcr:created      = ...
        jcr:lastModified = ...
```

The `fd:data` property holds the serialized form field values as a JSON string.

---

## How to pass the draft ID to the prefill service

There are two patterns:

**Pattern 1 — URL parameter (recommended for resume-draft flows)**

When generating the "continue draft" link, append the draft path as a URL parameter
and configure the form container to read it:

```
/content/forms/af/my-form.html?wcmmode=disabled&dataRef=/var/fd/dashboard/data/drafts/abc123
```

The `dataRef` URL parameter is automatically picked up by AEM Forms and passed to
`options.getDataRef()`.

**Pattern 2 — Form container property (for always-load-latest-draft flows)**

Set the `dataRef` property directly on the guide container node in CRX:
```
/content/forms/af/{form}/jcr:content/guideContainer
  dataRef = service://{serviceName}?draftid={draftId}
```

---

## Additional unit tests

```java
@Mock private ResourceResolverFactory mockRRF;
@Mock private Resource mockDraftResource;

@Test
void getPrefillData_validDraftPath_returnsSavedData() throws Exception {
    String savedJson = "{\"firstName\":\"Arun\",\"mobile\":\"9876543210\"}";
    ValueMap vm = new org.apache.sling.api.wrappers.ValueMapDecorator(
        Map.of("fd:data", savedJson));

    when(mockOptions.getDataRef())
        .thenReturn("/var/fd/dashboard/data/drafts/abc123");
    when(mockRRF.getServiceResourceResolver(any())).thenReturn(mockResolver);
    when(mockResolver.getResource("/var/fd/dashboard/data/drafts/abc123"))
        .thenReturn(mockDraftResource);
    when(mockDraftResource.getValueMap()).thenReturn(vm);

    PrefillData result = service.getPrefillData(mockOptions);
    String body = new String(
        result.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
    assertTrue(body.contains("Arun"));
}

@Test
void getPrefillData_draftNotFound_returnsEmptyJson() throws Exception {
    when(mockOptions.getDataRef())
        .thenReturn("/var/fd/dashboard/data/drafts/nonexistent");
    when(mockRRF.getServiceResourceResolver(any())).thenReturn(mockResolver);
    when(mockResolver.getResource(anyString())).thenReturn(null);

    PrefillData result = service.getPrefillData(mockOptions);
    String body = new String(
        result.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
    assertEquals("{}", body);
}

@Test
void getPrefillData_noDataRef_returnsEmptyJson() throws Exception {
    when(mockOptions.getDataRef()).thenReturn(null);

    PrefillData result = service.getPrefillData(mockOptions);
    String body = new String(
        result.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
    assertEquals("{}", body);
}
```
