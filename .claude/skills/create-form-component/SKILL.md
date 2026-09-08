---
name: create-form-component
description: >
  Generates a custom AEM Adaptive Forms field component for AEM as a Cloud Service
  Core Components. Produces the Sling Model, HTL, component dialog XML, clientlib,
  and unit test. Use when out-of-the-box field types do not meet the requirement.
version: 1.0.0
ide:
  cursor: .cursor/skills/create-form-component/
  github-copilot: .github/skills/create-form-component/
  claude-code: .claude/skills/create-form-component/
---

# Skill: create-form-component

## Role

You are an AEM Forms Core Components expert for AEM as a Cloud Service.
You build custom field components that correctly integrate with the AEM Forms
runtime event system, dispatch value changes properly, and meet WCAG 2.1 AA
accessibility requirements. You never extend Foundation components.

---

## Trigger

This skill activates when the developer asks to:
- Create a custom form field not available out of the box
- Build a rich field (e.g. star rating, signature pad, OTP input, slider)
- Extend an existing Core Components field with extra behaviour
- Create a composite field (multiple inputs acting as one logical field)

---

## What to generate (always all 5 files)

```
ui.apps/src/main/content/jcr_root/apps/{project}/components/form/{componentName}/
├── .content.xml                    ← Component definition
├── {componentName}.html            ← HTL template
└── _cq_dialog/
    └── .content.xml                ← Author dialog

ui.frontend/src/main/webpack/components/form/{componentName}/
├── {componentName}.js              ← Client-side behaviour
└── {componentName}.scss            ← Component styles

core/src/main/java/{package}/forms/components/
└── {ComponentName}Model.java       ← Sling Model

core/src/test/java/{package}/forms/components/
└── {ComponentName}ModelTest.java   ← Unit test
```

---

## Component definition — .content.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:cq="http://www.day.com/jcr/cq/1.0"
          xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
          jcr:primaryType="cq:Component"
          jcr:title="{Component Display Name}"
          jcr:description="{What this field does in one sentence}"
          componentGroup="{project} Forms"
          sling:resourceSuperType="core/fd/components/form/base/v1/base"/>
```

`sling:resourceSuperType` must always point to the Forms base component.
Never inherit from `wcm/foundation/components/parbase` or similar non-Forms types.

---

## HTL template — {componentName}.html

```html
<!--/* {componentName}.html — {Component Display Name} */-->
<sly data-sly-use.model="com.{package}.forms.components.{ComponentName}Model"
     data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html"/>

<sly data-sly-call="${clientlib.css @ categories='forms.custom.{componentName}'}"/>

<div class="cmp-adaptiveform-{componentName}"
     data-cmp-is="{componentName}"
     data-cmp-adaptiveformcontainer-path="${model.formContainerPath}"
     id="${model.id}"
     data-cmp-visible="${model.visible}"
     data-cmp-enabled="${model.enabled}">

  <label class="cmp-adaptiveform-{componentName}__label"
         for="${model.id}-input"
         data-cmp-required="${model.required}">
    ${model.label.value @ context='text'}
    <span aria-hidden="true" data-sly-test="${model.required}"> *</span>
  </label>

  <!--/* Your custom input markup here */-->
  <input type="text"
         id="${model.id}-input"
         name="${model.name}"
         class="cmp-adaptiveform-{componentName}__widget"
         value="${model.value @ context='attribute'}"
         placeholder="${model.placeholder @ context='attribute'}"
         aria-label="${model.label.value @ context='attribute'}"
         aria-required="${model.required}"
         aria-invalid="false"
         data-sly-test="${!model.readOnly}"/>

  <div class="cmp-adaptiveform-{componentName}__errormessage"
       aria-live="assertive"
       role="alert"
       id="${model.id}-error">
  </div>

  <div class="cmp-adaptiveform-{componentName}__description"
       data-sly-test="${model.tooltip}">
    ${model.tooltip @ context='text'}
  </div>

</div>

<sly data-sly-call="${clientlib.js @ categories='forms.custom.{componentName}'}"/>
```

---

## Sling Model — {ComponentName}Model.java

```java
package com.{package}.forms.components;

import com.adobe.cq.export.json.ComponentExporter;
import com.adobe.cq.forms.core.components.internal.form.AbstractFormFieldImpl;
import com.adobe.cq.forms.core.components.models.form.FieldType;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.models.annotations.Default;
import org.apache.sling.models.annotations.Model;
import org.apache.sling.models.annotations.injectorspecific.ValueMapValue;
import javax.annotation.PostConstruct;

@Model(
    adaptables = { SlingHttpServletRequest.class },
    adapters   = { {ComponentName}Model.class, ComponentExporter.class },
    resourceType = {ComponentName}Model.RESOURCE_TYPE
)
public class {ComponentName}Model extends AbstractFormFieldImpl {

    public static final String RESOURCE_TYPE =
        "{project}/components/form/{componentName}";

    // Custom properties from dialog
    @ValueMapValue
    @Default(values = "")
    private String customProperty;

    @PostConstruct
    protected void init() {
        // Custom initialisation after all injections complete
        super.initBaseModel();
    }

    /**
     * Return the field type string — used by AEM Forms runtime
     * to identify this field in rules and submissions.
     */
    @Override
    public FieldType getFieldType() {
        return FieldType.TEXT_INPUT; // Change to the closest base type
    }

    public String getCustomProperty() {
        return customProperty;
    }

    @Override
    public String getExportedType() {
        return RESOURCE_TYPE;
    }
}
```

---

## Author dialog — _cq_dialog/.content.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          xmlns:granite="http://www.adobe.com/jcr/granite/1.0"
          xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
          jcr:primaryType="nt:unstructured"
          jcr:title="{Component Display Name}"
          sling:resourceType="cq/gui/components/authoring/dialog">

  <content jcr:primaryType="nt:unstructured"
           sling:resourceType="granite/ui/components/coral/foundation/tabs">
    <items jcr:primaryType="nt:unstructured">

      <!--/* Basic tab — shared by all form fields */-->
      <basic jcr:primaryType="nt:unstructured"
             jcr:title="Basic"
             sling:resourceType="granite/ui/components/coral/foundation/include"
             path="/libs/fd/af/authoring/dialog/common/tabs/basic"/>

      <!--/* Custom tab for component-specific properties */-->
      <custom jcr:primaryType="nt:unstructured"
              jcr:title="{Component Name} Settings"
              sling:resourceType="granite/ui/components/coral/foundation/container">
        <items jcr:primaryType="nt:unstructured">
          <customProperty jcr:primaryType="nt:unstructured"
                          sling:resourceType="granite/ui/components/coral/foundation/form/textfield"
                          fieldLabel="Custom Property"
                          fieldDescription="Describe what this property controls"
                          name="./customProperty"/>
        </items>
      </custom>

      <!--/* Validation tab — shared */-->
      <validation jcr:primaryType="nt:unstructured"
                  jcr:title="Validation"
                  sling:resourceType="granite/ui/components/coral/foundation/include"
                  path="/libs/fd/af/authoring/dialog/common/tabs/validation"/>

      <!--/* Accessibility tab — shared */-->
      <accessibility jcr:primaryType="nt:unstructured"
                     jcr:title="Accessibility"
                     sling:resourceType="granite/ui/components/coral/foundation/include"
                     path="/libs/fd/af/authoring/dialog/common/tabs/accessibility"/>

    </items>
  </content>
</jcr:root>
```

---

## Client-side JS — {componentName}.js

```javascript
(function(document, $) {
    "use strict";

    const SELECTOR = "[data-cmp-is='{componentName}']";

    /**
     * {ComponentName} — AEM Adaptive Forms custom field.
     * Dispatches 'change' events to the Forms bridge so rules
     * and validation work correctly with this custom field.
     */
    function {ComponentName}(element) {
        this.element = element;
        this.widget  = element.querySelector(".cmp-adaptiveform-{componentName}__widget");
        this.error   = element.querySelector(".cmp-adaptiveform-{componentName}__errormessage");

        this._init();
    }

    {ComponentName}.prototype._init = function() {
        if (!this.widget) return;

        // Listen for native value changes
        this.widget.addEventListener("change", this._handleChange.bind(this));
        this.widget.addEventListener("blur",   this._handleBlur.bind(this));
    };

    {ComponentName}.prototype._handleChange = function(event) {
        var value = this.widget.value;

        // REQUIRED: dispatch to AEM Forms bridge so rules fire correctly
        this.element.dispatchEvent(new CustomEvent("AF_CustomFieldValueChangeEvent", {
            bubbles: true,
            detail: {
                field:    this.element.id,
                value:    value,
                fieldRef: this.element.dataset.cmpAdaptiveformcontainerPath
            }
        }));
    };

    {ComponentName}.prototype._handleBlur = function() {
        // Trigger validation on blur
        this.element.dispatchEvent(new CustomEvent("AF_CustomFieldBlurEvent", {
            bubbles: true,
            detail: { field: this.element.id }
        }));
    };

    {ComponentName}.prototype.setValue = function(value) {
        if (this.widget) {
            this.widget.value = value;
        }
    };

    {ComponentName}.prototype.showError = function(message) {
        if (this.error) {
            this.error.textContent = message;
            this.widget && this.widget.setAttribute("aria-invalid", "true");
            this.widget && this.widget.setAttribute("aria-describedby",
                this.error.id);
        }
    };

    {ComponentName}.prototype.clearError = function() {
        if (this.error) {
            this.error.textContent = "";
            this.widget && this.widget.setAttribute("aria-invalid", "false");
        }
    };

    // Initialise all instances on DOM ready
    document.addEventListener("DOMContentLoaded", function() {
        document.querySelectorAll(SELECTOR).forEach(function(el) {
            new {ComponentName}(el);
        });
    });

})(document, $);
```

---

## Quality checklist

Before delivering:

- [ ] `sling:resourceSuperType` points to `core/fd/components/form/base/v1/base`
- [ ] Sling Model uses `adaptables = SlingHttpServletRequest.class`
- [ ] HTL uses `data-cmp-is` attribute for JS initialisation
- [ ] Every input has `aria-label`, `aria-required`, `aria-invalid`, `aria-describedby`
- [ ] Error message container uses `role="alert"` and `aria-live="assertive"`
- [ ] JS dispatches `AF_CustomFieldValueChangeEvent` on value change
- [ ] JS dispatches `AF_CustomFieldBlurEvent` on blur (triggers validation)
- [ ] `getFieldType()` returns the closest matching base type
- [ ] No Foundation component inheritance
- [ ] Unit test covers model property accessors and field type
