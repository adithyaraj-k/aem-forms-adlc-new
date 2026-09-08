# create-form-clientlib — worked example

A functional clientlib for the **employee-registration-form**
(`/content/forms/af/ai-forms/employee-registration-form`). It adds two Rule-Editor
custom functions (work-email check, joining-date-after-DOB check), a programmatic
field highlighter, and scoped error CSS.

- Clientlib name: `clientlib-employee-registration`
- Category: `{project}.forms.employee-registration`

---

## 1. `.content.xml` — `/apps/{project}/clientlibs/clientlib-employee-registration/.content.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:cq="http://www.day.com/jcr/cq/1.0" xmlns:jcr="http://www.jcp.org/jcr/1.0"
          jcr:primaryType="cq:ClientLibraryFolder"
          allowProxy="{Boolean}true"
          categories="[{project}.forms.employee-registration]"
          dependencies="[core.forms.components.runtime.all]"/>
```

## 2. `js.txt`

```
#base=js
functions.js
validations.js
```

## 3. `css.txt`

```
#base=css
styles.css
```

## 4. `js/functions.js` — Rule-Editor custom functions

```js
/**
 * Validate that an email is a corporate address (rejects common free webmail domains).
 * @name validateWorkEmail Validate work email
 * @param {string} email the email address to check
 * @return {boolean} true when the email is a valid corporate address
 */
function validateWorkEmail(email) {
    if (!email) return false;
    var re = /^[^@\s]+@[^@\s]+\.[^@\s]+$/;
    if (!re.test(email)) return false;
    var banned = ["gmail.com", "yahoo.com", "hotmail.com", "outlook.com", "icloud.com"];
    var domain = email.split("@")[1].toLowerCase();
    return banned.indexOf(domain) === -1;
}

/**
 * Validate that the joining date is on or after the date of birth.
 * @name joiningAfterDob Validate joining date vs DOB
 * @param {string} dateOfBirth ISO date string
 * @param {string} dateOfJoining ISO date string
 * @return {boolean} true when joining date is on/after DOB
 */
function joiningAfterDob(dateOfBirth, dateOfJoining) {
    if (!dateOfBirth || !dateOfJoining) return true; // "required" handles emptiness
    return new Date(dateOfJoining).getTime() >= new Date(dateOfBirth).getTime();
}
```

These appear in the Rule Editor as *Validate work email* and *Validate joining date vs
DOB*. Wire them with `create-form-rules`, e.g. a Validate rule on `email`:
`validateWorkEmail($field)`, and on `dateOfJoining`:
`joiningAfterDob($form.personalDetails.dateOfBirth, $form.employmentDetails.dateOfJoining)`.

## 5. `js/validations.js` — non-invasive, DOM-driven validation

> Uses the **verified** AF Core Components runtime API and the **non-invasive** pattern:
> it never writes to the model (no `markAsInvalid` / `valid` / `value` setters — those
> re-render a field and can wipe a native `<input type="date">`). It reads single-input
> values from the DOM (a date input is ISO `yyyy-mm-dd`) and renders errors into its own
> `.clientlib-error` element. See SKILL.md "Layer 2" for the full rationale.

```js
(function () {
    "use strict";

    // Field name -> rule. Return an error string, or null when valid.
    var RULES = {
        email: function (v) { return validateWorkEmail(v) ? null : "Use your corporate email address."; }
        // add one entry per field you validate programmatically, keyed by exact field name
    };

    function wrapperOf(field) { return (field && field.id) ? document.getElementById(field.id) : null; }

    function readValue(field) {                 // DOM for single inputs, model for groups
        var el = wrapperOf(field);
        if (el) {
            var inputs = el.querySelectorAll("input, select, textarea");
            if (inputs.length === 1 && inputs[0].type !== "checkbox" && inputs[0].type !== "radio") {
                return inputs[0].value;
            }
        }
        return field ? field.value : undefined;
    }

    function showError(field, msg) {
        var w = wrapperOf(field); if (!w) return;
        w.classList.add("field-error-highlight");
        var e = w.querySelector(".clientlib-error");
        if (!e) { e = document.createElement("div"); e.className = "clientlib-error"; e.setAttribute("aria-live", "polite"); w.appendChild(e); }
        e.textContent = msg;
    }
    function clearError(field) {
        var w = wrapperOf(field); if (!w) return;
        w.classList.remove("field-error-highlight");
        var e = w.querySelector(".clientlib-error"); if (e) e.textContent = "";
    }
    function validateField(field) {
        if (!field || !field.name || !RULES[field.name]) return true;
        var msg = RULES[field.name](readValue(field));
        if (msg) { showError(field, msg); return false; }
        clearError(field); return true;
    }
    function indexFields(model) {
        var map = {};
        (function walk(n) {
            if (!n) return;
            if (n.name && RULES[n.name]) map[n.name] = n;
            var kids = n.items || n.children;
            if (Array.isArray(kids)) kids.forEach(walk);
        })(model);
        return map;
    }

    var wired = false;
    function wire(model) {
        if (wired || !model) return;
        var map = indexFields(model);
        if (!Object.keys(map).length) return;
        wired = true;
        function onInteract(e) {
            var input = e.target; if (!input || !input.name) return;
            var field = map[input.name]; if (!field) return;
            setTimeout(function () { validateField(field); }, 0);
        }
        document.addEventListener("change", onInteract, true);
        document.addEventListener("blur", onInteract, true);
        document.addEventListener("click", function (e) {
            var s = (e.target && e.target.nodeType === 1) ? e.target : (e.target && e.target.parentElement);
            var btn = s && s.closest ? s.closest(".cmp-adaptiveform-button__widget") : null;
            if (!btn) return;
            var ok = true;
            Object.keys(map).forEach(function (n) { if (!validateField(map[n])) ok = false; });
            if (!ok) { e.preventDefault(); e.stopImmediatePropagation(); }
        }, true);
    }

    document.addEventListener("AF_FormContainerInitialised", function (event) {
        var c = event && event.detail;            // the container IS event.detail
        wire(c && typeof c.getModel === "function" ? c.getModel() : null);
    });
    var tries = 0;
    (function poll() {                            // race fallback if init fired first
        if (wired) return;
        try {
            if (window.FormView && typeof window.FormView.getInstance === "function") {
                wire(window.FormView.getInstance().getFormModel());
            }
        } catch (e) { /* not ready */ }
        if (!wired && tries++ < 100) setTimeout(poll, 100);
    })();
})();
```

> Every runtime call is feature-detected and the listener never throws during init —
> if the AF runtime API differs on your AEM version, the form still renders normally.

## 6. `css/styles.css` — scoped error styling

```css
.cmp-adaptiveform-container .field-error-highlight .cmp-adaptiveform-textinput__widget,
.cmp-adaptiveform-container .field-error-highlight .cmp-adaptiveform-telephoneinput__widget,
.cmp-adaptiveform-container .field-error-highlight .cmp-adaptiveform-emailinput__widget,
.cmp-adaptiveform-container .field-error-highlight .cmp-adaptiveform-datepicker__widget,
.cmp-adaptiveform-container .field-error-highlight textarea {
    border: 1px solid #d7373f;
    box-shadow: 0 0 0 1px #d7373f;
}

.cmp-adaptiveform-container .cmp-adaptiveform-checkboxgroup.field-error-highlight {
    outline: 1px solid #d7373f;
    outline-offset: 4px;
    border-radius: 4px;
}

/* Error text injected by validations.js (non-invasive — we own this element). */
.cmp-adaptiveform-container .clientlib-error {
    color: #d7373f;
    font-size: 0.85rem;
    margin-top: 4px;
}
```

## 7. Wiring — attach to the form

Add `clientLibRef` to the form's `guideContainer` in
`/content/forms/af/ai-forms/employee-registration-form/.content.xml`:

```xml
<guideContainer
    jcr:primaryType="nt:unstructured"
    sling:resourceType="{project}/components/adaptiveForm/formcontainer"
    fieldType="form"
    name="employeeRegistrationForm"
    clientLibRef="{project}.forms.employee-registration"
    schemaType="jsonschema"
    schemaRef="/content/dam/formsanddocuments/schema/employeeRegistration.schema.json"
    ... >
```

## 8. Verify after deploy

```
# Proxied assets resolve (allowProxy=true):
http://localhost:4502/etc.clientlibs/{project}/clientlibs/clientlib-employee-registration.js
http://localhost:4502/etc.clientlibs/{project}/clientlibs/clientlib-employee-registration.css

# The form page references the category's <link>/<script>:
http://localhost:4502/content/forms/af/ai-forms/employee-registration-form.html
```

---

## Reuse / automation notes

- **Shared validators across forms:** put generic functions in
  `clientlib-form-validators` (category `{project}.forms.shared-validators`)
  and have each form's library `embed` it — one `clientLibRef` per form still applies.
- **Per-form vs shared:** name per-form libraries `clientlib-{formName}`; keep cross-form
  logic in a single shared library to avoid divergence.
- **Source build:** if heavy tooling (TypeScript, bundling) is wanted, author in
  `ui.frontend/` and let `clientlib.config.js` emit the `cq:ClientLibraryFolder` into
  `ui.apps` at build time, instead of hand-writing JS. For small validation logic,
  hand-authored clientlibs (this skill) are simpler and need no build step.
- **Testing:** custom functions are plain functions — unit-test them with the
  `create-form-tests` skill (JUnit is for Java; use a JS test runner for these, or keep
  them pure and table-test the logic).
```
