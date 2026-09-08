# Example — "Editable Template"

A complete editable template based on the `af-page-v2` template-type for the
`{project}` project (values pulled from `.aem-forms-config.yaml`).

Developer prompt:
> *Create an editable template called `editable-template` titled "Editable Template",
> enabled, with a locked header/footer layout around an editable form container.*

Resulting paths (template node = `editable-template`):

```
ui.content/src/main/content/jcr_root/conf/{project}/settings/wcm/templates/editable-template/
├── .content.xml
├── initial/.content.xml
├── structure/.content.xml
├── policies/.content.xml
└── thumbnail.png
```

---

## .content.xml (template root)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="cq:Template">
    <jcr:content
        author="{project}"
        cq:templateType="/conf/{project}/settings/wcm/template-types/af-page-v2"
        jcr:primaryType="cq:PageContent"
        jcr:title="Editable Template"
        jcr:description="Editable template for Adaptive Forms based on the core Adaptive Form (Core Components) template-type. Provides a locked header/footer layout around an editable form container."
        status="enabled"/>
</jcr:root>
```

---

## initial/.content.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:fd="http://www.adobe.com/aemfd/fd/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="cq:Page">
    <jcr:content
        cq:deviceGroups="[/etc/mobile/groups/responsive]"
        cq:template="/conf/{project}/settings/wcm/templates/editable-template"
        jcr:primaryType="cq:PageContent"
        sling:resourceType="{project}/components/adaptiveForm/page"
        sling:configRef="/conf/{project}/forms"
        guideComponentType="fd/af/templates">
        <container1
            jcr:primaryType="nt:unstructured"
            sling:resourceType="{project}/components/container"
            layout="responsiveGrid">
            <pageheader
                jcr:primaryType="nt:unstructured"
                jcr:title="Header"
                sling:resourceType="{project}/components/adaptiveForm/pageheader"
                fieldType="page-header">
                <text
                    jcr:primaryType="nt:unstructured"
                    sling:resourceType="{project}/components/text"
                    text="&lt;p>Editable Template&lt;/p>&#xd;&#xa;"
                    textIsRich="true"
                    value="Company Name Here"/>
            </pageheader>
        </container1>
        <guideContainer
            fd:version="2.1"
            actionType="fd/af/components/guidesubmittype/restendpoint"
            fieldType="form"
            schemaType="none"
            textIsRich="true"
            thankYouMessage="Thank you for submitting the form."
            thankYouOption="page"
            jcr:primaryType="nt:unstructured"
            sling:resourceType="{project}/components/adaptiveForm/formcontainer"/>
        <container2
            jcr:primaryType="nt:unstructured"
            sling:resourceType="{project}/components/container"
            layout="responsiveGrid">
            <footer
                jcr:primaryType="nt:unstructured"
                jcr:title="Footer"
                sling:resourceType="{project}/components/adaptiveForm/footer">
                <text
                    jcr:primaryType="nt:unstructured"
                    jcr:title="Footer"
                    sling:resourceType="{project}/components/text"
                    css="footerText"
                    text="&lt;p>© YYYY Company Name | All rights reserved.&lt;/p>&#xd;&#xa;"
                    textIsRich="true"/>
            </footer>
        </container2>
    </jcr:content>
</jcr:root>
```

---

## structure/.content.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:fd="http://www.adobe.com/aemfd/fd/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="cq:Page">
    <jcr:content
        cq:deviceGroups="[/etc/mobile/groups/responsive]"
        cq:template="/conf/{project}/settings/wcm/templates/editable-template"
        jcr:primaryType="cq:PageContent"
        sling:resourceType="{project}/components/adaptiveForm/page"
        guideComponentType="fd/af/templates">
        <container1
                jcr:primaryType="nt:unstructured"
                sling:resourceType="{project}/components/container"
                layout="responsiveGrid"
                editable="{Boolean}true" />
        <guideContainer
            fd:version="2.1"
            fieldType="form"
            jcr:primaryType="nt:unstructured"
            editable="{Boolean}true"
            sling:resourceType="{project}/components/adaptiveForm/formcontainer"/>
        <container2
                jcr:primaryType="nt:unstructured"
                sling:resourceType="{project}/components/container"
                layout="responsiveGrid"
                editable="{Boolean}true" />
        <cq:responsive jcr:primaryType="nt:unstructured">
            <breakpoints jcr:primaryType="nt:unstructured">
                <phone
                        jcr:primaryType="nt:unstructured"
                        title="Smaller Screen"
                        width="{Long}768"/>
                <tablet
                        jcr:primaryType="nt:unstructured"
                        title="Tablet"
                        width="{Long}1200"/>
            </breakpoints>
        </cq:responsive>
    </jcr:content>
</jcr:root>
```

---

## policies/.content.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="cq:Page">
    <jcr:content
        cq:policy="{project}/components/page/policy"
        jcr:primaryType="nt:unstructured"
        sling:resourceType="wcm/core/components/policies/mappings">
        <container1
                jcr:primaryType="nt:unstructured"
                sling:resourceType="wcm/core/components/policies/mapping"
                cq:policy="{project}/components/container/cc-af-header-policy"
                layout="responsiveGrid"/>
        <guideContainer
            cq:policy="{project}/components/adaptiveForm/formcontainer/default"
            jcr:primaryType="nt:unstructured"
            sling:resourceType="wcm/core/components/policies/mapping">
            <{project} jcr:primaryType="nt:unstructured">
                <components jcr:primaryType="nt:unstructured">
                    <adaptiveForm jcr:primaryType="nt:unstructured">
                        <numberinput
                          cq:policy="{project}/components/adaptiveForm/numberinput/default"
                          jcr:primaryType="nt:unstructured"
                          sling:resourceType="wcm/core/components/policies/mapping"/>
                        <datepicker
                          cq:policy="{project}/components/adaptiveForm/datepicker/default"
                          jcr:primaryType="nt:unstructured"
                          sling:resourceType="wcm/core/components/policies/mapping"/>
                        <telephoneinput
                          cq:policy="{project}/components/adaptiveForm/telephoneinput/default"
                          jcr:primaryType="nt:unstructured"
                          sling:resourceType="wcm/core/components/policies/mapping"/>
                        <panelcontainer
                          cq:policy="{project}/components/adaptiveForm/panelcontainer/default"
                          jcr:primaryType="nt:unstructured"
                          sling:resourceType="wcm/core/components/policies/mapping">
                        </panelcontainer>
                        <switch
                          cq:policy="{project}/components/adaptiveForm/switch/default"
                          jcr:primaryType="nt:unstructured"
                          sling:resourceType="wcm/core/components/policies/mapping">
                        </switch>
                    </adaptiveForm>
                </components>
            </{project}>
        </guideContainer>
        <container2
                jcr:primaryType="nt:unstructured"
                sling:resourceType="wcm/core/components/policies/mapping"
                cq:policy="{project}/components/container/cc-af-footer-policy"
                layout="responsiveGrid"/>
    </jcr:content>
</jcr:root>
```

---

## filter.xml entries

```xml
<filter root="/conf/{project}/settings/wcm/templates/editable-template"/>
<filter root="/conf/{project}/settings/wcm/policies" mode="merge"/>
```

---

## Verify after deploy

```bash
mvn clean install -PautoInstallSinglePackage
```

Then in AEM:
1. Open **Tools → Templates → {project}** — the "Editable Template"
   should appear with status **Enabled**.
2. Open **Forms → Forms & Documents → Create → Adaptive Form** — the template should be
   offered.
3. Create a form from it: the header, form container, and footer areas are editable; the
   form container accepts the Core Components fields declared in the policy mapping
   (number, date, telephone, panel, switch, …).
