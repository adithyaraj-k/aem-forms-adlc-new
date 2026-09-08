# create-adaptive-form — node examples

Every snippet below is taken **verbatim** (only reformatted/renamed for clarity) from the
verified `benefits-enroll` form in this repo. Copy these; adjust `name`, `jcr:title`, and
messages. Component `sling:resourceType` always uses the `{project}` namespace (`aem-demo-site`).

---

## Field nodes

### Text input (required, single line)

```xml
<FirstName jcr:primaryType="nt:unstructured" jcr:title="First name"
    sling:resourceType="{project}/components/adaptiveForm/textinput"
    autocomplete="off" enabled="{Boolean}true" fieldType="text-input"
    mandatoryMessage="Please enter your first name." name="FirstName"
    readOnly="{Boolean}false" required="true"
    textIsRich="[true,true,true]" validatePictureClauseMessage="[,]" visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
    </cq:responsive>
</FirstName>
```

### Textarea (multi-line) — add `multiLine="true"`

```xml
<Address jcr:primaryType="nt:unstructured" jcr:title="Mailing address"
    sling:resourceType="{project}/components/adaptiveForm/textinput"
    autocomplete="off" enabled="{Boolean}true" fieldType="text-input"
    mandatoryMessage="Please provide your communication address." multiLine="true"
    name="Address" readOnly="{Boolean}false" required="true"
    textIsRich="[true,true,true]" validatePictureClauseMessage="[,]" visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
    </cq:responsive>
</Address>
```

### Email

```xml
<Email jcr:primaryType="nt:unstructured" jcr:title="Email"
    sling:resourceType="{project}/components/adaptiveForm/emailinput"
    enabled="{Boolean}true" fieldType="email"
    mandatoryMessage="Please enter your email id." name="Email"
    readOnly="{Boolean}false" required="true" textIsRich="[true,true]" visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
    </cq:responsive>
</Email>
```

### Telephone (with 10-digit pattern)

```xml
<PhoneNumber jcr:primaryType="nt:unstructured" jcr:title="Phone number"
    sling:resourceType="{project}/components/adaptiveForm/telephoneinput"
    enabled="{Boolean}true" fieldType="text-input"
    mandatoryMessage="Please enter a valid US phone number, including the area code. Example: (123) 456-7890"
    name="PhoneNumber" pattern="^[0-9]{10}$" readOnly="{Boolean}false" required="true"
    textIsRich="[true,true]"
    validateExpMessage="Please enter a valid US phone number, including the area code. Example: (123) 456-7890"
    validatePatternMessage="Please enter a valid US phone number, including the area code. Example: (123) 456-7890"
    visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
    </cq:responsive>
</PhoneNumber>
```

### Date picker

```xml
<DOB jcr:primaryType="nt:unstructured" jcr:title="Date of birth"
    sling:resourceType="{project}/components/adaptiveForm/datepicker"
    enabled="{Boolean}true" fieldType="date-input"
    mandatoryMessage="Please enter your date of birth." name="DOB"
    readOnly="{Boolean}false" required="true" textIsRich="[true,true]" visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" behavior="newline" offset="0" width="12"/>
    </cq:responsive>
</DOB>
```

### Dropdown (single-select)

```xml
<HighestQualification jcr:primaryType="nt:unstructured" jcr:title="Select your highest qualification"
    sling:resourceType="{project}/components/adaptiveForm/dropdown"
    enabled="{Boolean}true"
    enum="[0,1,2,3,4,5]"
    enumNames="[High School Diploma,Associate's Degree,Bachelor's Degree,Master's Degree,Professional Certification,Other Qualification]"
    fieldType="drop-down" mandatoryMessage="Please select your highest qualification."
    name="HighestQualification" readOnly="{Boolean}false" required="true"
    textIsRich="[true,true]" tooltipVisible="true" type="string" typeIndex="0" visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" behavior="newline" offset="0" width="12"/>
    </cq:responsive>
</HighestQualification>
```

### Checkbox group (multi-select)

```xml
<BenefitSelectionCheckBox jcr:primaryType="nt:unstructured"
    jcr:title="Select the service benefits you would like to enroll in"
    sling:resourceType="{project}/components/adaptiveForm/checkboxgroup"
    enabled="{Boolean}true"
    enum="[0,1,2,3]" enumNames="[Education,Healthcare,Child Care,Other]"
    fieldType="checkbox-group" mandatoryMessage="Please select one or more service benefits."
    name="BenefitSelectionCheckBox" orientation="horizontal" readOnly="{Boolean}false"
    required="true" textIsRich="[true,true]" type="string[]" visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" behavior="newline"/>
    </cq:responsive>
</BenefitSelectionCheckBox>
```

### Single consent checkbox (T&C) — scalar enum

```xml
<TermsAndConditions jcr:primaryType="nt:unstructured" jcr:title="Terms and Conditions"
    sling:resourceType="{project}/components/adaptiveForm/checkboxgroup"
    enabled="{Boolean}true" enum="0"
    enumNames="I hereby declare that the information provided here is true to the best of my knowledge."
    fieldType="checkbox-group"
    mandatoryMessage="Please agree to the terms and conditions before submitting the form."
    name="TermsAndConditions" orientation="vertical" readOnly="{Boolean}false" required="true"
    textIsRich="[true,true]" type="number[]" visible="{Boolean}true"/>
```

---

## Submit button (always needed — it's what submits the form)

```xml
<submit jcr:primaryType="nt:unstructured" jcr:title="Submit"
    sling:resourceType="{project}/components/adaptiveForm/actions/submit"
    buttonType="submit" dorExclusion="true" enabled="{Boolean}true" fieldType="button"
    name="submitButton" readOnly="{Boolean}false" textIsRich="[true,true]" visible="{Boolean}true">
    <cq:responsive jcr:primaryType="nt:unstructured">
        <default jcr:primaryType="nt:unstructured" offset="0" width="12"/>
    </cq:responsive>
    <fd:rules
        fd:click="[{&quot;nodeName&quot;:&quot;ROOT&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;STATEMENT&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;EVENT_SCRIPTS&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;EVENT_CONDITION&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;EVENT_AND_COMPARISON&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;COMPONENT&quot;\,&quot;value&quot;:{&quot;id&quot;:&quot;$form.SignaturePanel.submitButton&quot;\,&quot;type&quot;:&quot;BUTTON&quot;\,&quot;name&quot;:&quot;submitButton&quot;}}\,{&quot;nodeName&quot;:&quot;EVENT_AND_COMPARISON_OPERATOR&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;is clicked&quot;\,&quot;value&quot;:null}}\,{&quot;nodeName&quot;:&quot;PRIMITIVE_EXPRESSION&quot;\,&quot;choice&quot;:null}]}\,&quot;nested&quot;:false}\,{&quot;nodeName&quot;:&quot;Then&quot;\,&quot;value&quot;:null}\,{&quot;nodeName&quot;:&quot;BLOCK_STATEMENTS&quot;\,&quot;items&quot;:[{&quot;nodeName&quot;:&quot;BLOCK_STATEMENT&quot;\,&quot;choice&quot;:{&quot;nodeName&quot;:&quot;SUBMIT_FORM&quot;\,&quot;items&quot;:[]}}]}]}}]\,&quot;isValid&quot;:true\,&quot;enabled&quot;:true\,&quot;version&quot;:1\,&quot;script&quot;:[&quot;submitForm()&quot;]\,&quot;eventName&quot;:&quot;Click&quot;\,&quot;ruleType&quot;:&quot;&quot;\,&quot;description&quot;:&quot;&quot;}]"
        jcr:primaryType="nt:unstructured" validationStatus="valid"/>
    <fd:events jcr:primaryType="nt:unstructured" click="[submitForm()]"/>
</submit>
```

---

## Header and footer (siblings of `guideContainer`)

```xml
<container1 jcr:primaryType="nt:unstructured"
    sling:resourceType="{project}/components/container" layout="responsiveGrid">
    <pageheader jcr:primaryType="nt:unstructured" jcr:title="Header"
        sling:resourceType="{project}/components/adaptiveForm/pageheader"
        fieldType="page-header">
        <image jcr:primaryType="nt:unstructured" jcr:title="Logo"
            sling:resourceType="{project}/components/image"
            alt="Logo" altValueFromDAM="false" disableLazyLoading="false" displayPopupTitle="true"
            fileName="wknd-logo.png" fileReference="/content/dam/{project}/wknd_logo.png"
            height="0" imageFromPageImage="false" isDecorative="false" linkTarget="_self"
            name="logoImage" titleValueFromDAM="false" width="0"/>
        <text jcr:primaryType="nt:unstructured"
            sling:resourceType="{project}/components/text"
            text="&lt;p>{Form Title}&lt;/p>" textIsRich="true" value="Company Name Here"/>
    </pageheader>
</container1>
<container2 jcr:primaryType="nt:unstructured"
    sling:resourceType="{project}/components/container" layout="responsiveGrid">
    <footer jcr:primaryType="nt:unstructured" jcr:title="Footer"
        sling:resourceType="{project}/components/adaptiveForm/footer">
        <text jcr:primaryType="nt:unstructured" jcr:title="Footer"
            sling:resourceType="{project}/components/text"
            css="footerText" text="&lt;p>© YYYY {Company} | All rights reserved.&lt;/p>" textIsRich="true"/>
    </footer>
</container2>
```

---

## Rule (`fd:rules` / `fd:events`) anatomy

A panel/field with conditional visibility carries an `fd:rules` child whose `fd:visible`
attribute holds the **authored model** as escaped JSON (note `\,` for commas inside the JSON
and `&quot;` for quotes), plus a plain-expression mirror (`visible="..."`) and
`validationStatus="valid"`. Example — the Education panel shows when checkbox option `0` is
selected (from benefits-enroll):

```xml
<fd:rules
    fd:visible="[{&quot;nodeName&quot;:&quot;ROOT&quot;\,&quot;items&quot;:[ ... &quot;script&quot;:&quot;contains($form.BasicDetailsPanel.BenefitSelectionCheckBox\, '0')&quot;\,&quot;eventName&quot;:&quot;Visibility&quot; ... }]"
    jcr:primaryType="nt:unstructured" validationStatus="valid"
    visible="contains($form.BasicDetailsPanel.BenefitSelectionCheckBox, '0')"/>
```

A value-commit rule (checkbox change → show a panel) lives on the source field's `fd:rules`
`fd:valueCommit` attr, with the runtime form on `fd:events` `change`:
```xml
<fd:events jcr:primaryType="nt:unstructured"
    change="[if(contains($field\, '0')\, dispatchEvent($form.EducationPanel\, 'custom:setProperty'\, {visible : true()})\, {})]"/>
```

**Don't hand-author the escaped JSON from scratch.** Either copy a benefits-enroll rule and
swap the component ids/names, or run the **`create-form-rules`** skill which emits this exact
structure. The full, unabridged JSON for every rule lives in benefits-enroll's `.content.xml`
(lines 178–183 for the date-calc, 230–236 for the checkbox value-commits, 278–282 / 337–341 /
402–406 / 467–471 / 497–501 / 609–613 for visibility rules).

---

## Boolean & multi-value attribute encoding (AEM JCR conventions)

| Want | Write |
|---|---|
| Boolean true/false | `enabled="{Boolean}true"` / `visible="{Boolean}false"` |
| Multi-value (array) | `enum="[0,1,2,3]"` — bracketed, comma-separated |
| Comma **inside** an array value | escape it: `validatePictureClauseMessage="[,]"` becomes `[a\,b]` |
| Rich-text HTML in an attr | escape `<` as `&lt;` : `text="&lt;p>Hi&lt;/p>"` |
| Date value | `cq:lastModified="{Date}2026-06-17T15:15:34.651+05:30"` |
