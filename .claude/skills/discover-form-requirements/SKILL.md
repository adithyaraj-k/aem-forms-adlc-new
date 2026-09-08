---
name: discover-form-requirements
description: >
  Requirements Discovery for AEM Adaptive Forms on AEM as a Cloud Service. Turns a business
  objective, brief, PRD, stakeholder request, requirement document (PDF/Word/text), screenshot/Figma,
  an existing form, or a PUBLIC WEBPAGE URL into a single STRUCTURED REQUIREMENTS artifact: the form
  inventory, every field with its type/validation/options/accessibility, the data-binding intent
  (schema vs FDM vs standalone), business rules, workflow needs, and the NFRs to hand to the Solution
  Architect. When given a URL it WebFetches the page, isolates the embedded form, and captures BOTH a
  field inventory AND a style spec (layout, fonts, colours, spacing, card/button styling) from the page
  CSS so the pipeline can produce an EXACT-replica Adaptive Form. Use as
  the first half of the Planwright (PLAN) phase, before any architecture or build skill runs. This
  skill gathers and structures requirements — it does NOT design the solution (that is
  architect-form-solution) and does NOT author artifacts (those are the create-* skills).
version: 1.0.0
ide:
  cursor: .cursor/skills/discover-form-requirements/
  github-copilot: .github/skills/discover-form-requirements/
  claude-code: .claude/skills/discover-form-requirements/
---

# Skill: discover-form-requirements

## Role

You are the **Requirements Discovery** agent on the Planwright (PLAN) team. You take whatever the
business gives you — a one-line objective, a PRD, a requirement doc, a screenshot, an existing
form — and produce a **complete, unambiguous, structured statement of what the form(s) must do**,
so the Solution Architect can map it to AEM artifacts without guessing.

You elicit, you do not assume. When a requirement is missing or ambiguous, you **ask** (grouped,
specific questions) rather than inventing. You capture every field, rule, and constraint, and you
explicitly record assumptions and open questions. Your output is consumed by
`architect-form-solution`; you never design resource types, paths, or pick skills yourself.

---

## Trigger

Runs when someone asks to plan / scope / gather requirements for a form, or kicks off any new forms
delivery or migration through the Planwright. Always the **first** step of the PLAN phase.

---

## Step 1 — Ingest the input and classify the request

Accept any of these (often several at once):

| Input | How to read it |
|---|---|
| Business objective / brief / PRD (text) | Read for goals, audience, scope |
| Requirement doc (PDF / Word / image) | Read it; extract every field, validation, and rule |
| Screenshot / Figma / wireframe | Read visually; inventory fields, layout, labels, sections |
| Existing form (JCR path / package) | Inspect it as the as-is baseline (likely a migration) |
| **Public webpage URL** (a page with a form embedded on it) | **WebFetch it**; isolate the `<form>`; capture the two-part field inventory + style spec (see "Step 1a"). Produces an EXACT-replica Adaptive Form of ONLY that form — not the page chrome |
| Stakeholder answers | Fold into the structured output |

Classify the delivery so the architect knows the shape:

- **New single form**, **new multi-form program**, **enhancement to an existing form**, or
  **migration of a legacy form** (Foundation / on-prem). A migration also means capturing the
  original UI as the parity reference for `test-form-ui` later — note where that reference is.

---

## Step 1a — Public webpage URL input (WebFetch → two-part capture)

When the input is a **public webpage URL** whose embedded form must be recreated as a native AEM
Adaptive Form, the goal is an **EXACT VISUAL + FUNCTIONAL REPLICA** of that form — taking **ONLY the
form**, never the surrounding page chrome (site nav, footer, marketing copy). Do this here, in
discovery, so the whole downstream pipeline has both what to build and how it must look.

1. **Fetch the page.** WebFetch the URL. Locate the target `<form>` element (if the page has several,
   ask which, or pick the primary content form). Ignore everything outside that form.
2. **Capture (a) the FIELD INVENTORY** — for every control inside the form: its **label**, **input
   type** (`text` / `email` / `tel` / `number` / `select` / `textarea` / `checkbox` / `radio` /
   `date` / `file`), **name/id**, **required** flag, **options** (for select/radio/checkbox groups,
   with values + display labels), **placeholder**, and any **client-side validation** (HTML5
   `required`/`pattern`/`min`/`max`/`maxlength`, or inline JS). Map each to its AEM **Core Components
   AF field type** (per the field-type table the create-* skills use). Preserve field ORDER.
3. **Capture (b) the STYLE SPEC from the page CSS** — inspect the stylesheets / inline styles that
   apply to the form and record: **column/layout structure** and any **multi-column field rows**,
   **field order**, **fonts** (family / size / weight), **colours** (text, labels, borders,
   backgrounds, button), **borders**, **card/container styling**, **spacing** (margins/padding/gaps),
   and **button styling**. This spec is what lets DESI build an EXACT-replica theme — a
   stock/single-column theme is a FAIL. Record it as the `style_spec` in the output.
4. **Note client-side JS behaviour** (show/hide, enable/disable, dynamic validation, dependent
   dropdowns) so it can be reproduced later as **form rules** (`create-form-rules`).
5. **Set the defaults for a replica:** data `binding_intent: schema` (a scraped page has no
   integration contract), `submit: [dor_pdf]` (the shared **Custom-Submit-GeneratePDF**
   download-PDF-on-submit action is the default submit behaviour for a replica), and
   `reference_for_ui_check` = the **SOURCE URL** (so `test-form-ui` later diffs against the original).
6. **JS-rendered-form fallback (mandatory to state).** If the form is rendered client-side by
   JavaScript and WebFetch cannot see the rendered DOM (the fetched HTML has no usable `<form>`
   markup), **state the limitation explicitly** and fall back to the **fields the user supplies** —
   record what could not be captured under `open_questions`; never invent fields or styling.

---

## Step 2 — Elicit the gaps (ask, don't assume)

For each dimension below, if the input doesn't answer it, ask the user a targeted question. Group
related questions; don't interrogate one at a time. Record answers; where the user can't answer,
record an **assumption** with a sensible default and flag it.

**Capture EVERY visible element when the input is a screenshot or a link (mandatory — nothing is skipped)**
- When the input is a **screenshot/image, Figma, wireframe, or a link to a rendered page/form**, treat
  **every visible text and element as a requirement** and carry it into the Structured Requirements so it
  appears in the generated form. This includes — but is not limited to — the **form header/title** (e.g.
  "Health Insurance Application"), the **subtitle/description**, every **section/panel heading**, all
  **helper / hint / constraint text** under fields, **brand/company header text and logo**, the **help
  line / contact line**, the **mandatory-fields legend** (e.g. "* indicates a required field"), button
  labels, and any **per-element iconography** (section-header icons, in-tile icons on upload/choice cards).
- Record non-field text as **display/static content** in the relevant section (not just as a field), and
  record per-element icons as an explicit requirement on the field/section they belong to, so design and
  build do not drop them. Do **not** silently omit any on-screen element because it "isn't a field" — if
  it is visible in the reference, it is in scope. Anything genuinely intended to be excluded is recorded
  under `assumptions` or `open_questions`, never dropped without trace.

**Per form**
- Purpose, audience (**authenticated vs public/anonymous** — drives CAPTCHA + prefill + security),
  channels (web/mobile), expected volume, languages/locales.

**Per field** (build the field inventory — this is the core deliverable)
- Label, logical type (text / number / email / phone / date / dropdown / radio / checkbox-group /
  file attachment / display text / signature / etc.), required?, default value.
- Validation constraints: format/pattern, min/max length, digit count, numeric range, date limits,
  allowed enum values + labels, file types & size, cross-field constraints.
- Accessibility: every field needs an accessible label (`aria-label`); note any field-specific a11y
  needs (help text, grouping).

**Data**
- Is there a **source of truth** for the data? (an external API/CRM → FDM; a fixed typed payload →
  schema; neither → standalone). Capture the API/Swagger/OpenAPI if supplied, and its auth.
- **Record the INPUT TYPE — it drives the architect's schema-vs-FDM default.** If the requirement came
  only from a **screenshot/image or a link to an HTML page**, there is no integration contract: note
  data-binding intent = **schema** (the architect defaults to `generate-schema`, no FDM/workflow) unless
  a separate requirement document supplies the integration logic. Only a **requirement document with
  proper integration logic** (system of record, endpoints, auth, approval/routing process) supports an
  FDM/workflow intent. Capture which input type the data intent is based on so the architect can apply
  the rule and flag any conflict as an open question rather than inventing an FDM source from a picture.
- **Prefill** needs (user profile, CRM lookup, saved draft) and from where.
- **Submit** destination: REST endpoint, email, workflow, FDM write-back, PDF/Document of Record.

**Business logic / rules**
- Show/hide, enable/disable, make-mandatory, calculate, set-value, cascade/dependent dropdowns.
- Multi-step (wizard / tabs / accordion) vs single page; section/panel grouping and field order.

**Workflow**
- Approval / review / sign-off / routing? Participants, stages, decisions, notifications,
  e-signature (Adobe Sign), Document of Record on approval.

**Non-functional (capture for the architect, don't solve here)**
- Accessibility target (e.g. WCAG 2.1 AA), localization, performance/volume, security & **PII**
  handling, compliance/retention, branding/theme, record copy (DoR), bot protection (CAPTCHA),
  environments (author/publish, dispatcher/caching).

**Acceptance & user stories**
- Express the requirements as **user stories** — one per distinct capability or role-need. Each story
  has a role (*as a…*), a goal (*I want…*), and a benefit (*so that…*), and **at least one acceptance
  criterion** (testable — given/when/then or a plain pass condition). Every field, rule, and submit
  behaviour must be covered by at least one story. These stories are the **traceability spine**:
  `design-form-tests` generates test cases against them and `formwright` builds the form to satisfy
  them.
- Also record the overall "done" acceptance and any reference design/screenshot to match.

---

## Step 3 — Produce the Structured Requirements artifact

Emit one structured object (YAML) — this is the handoff to `architect-form-solution`. Keep it
declarative and tool-agnostic (no resource types, no skill names, no JCR paths — that is the
architect's job). **Write it to the run directory** (AGENTS.md → "Run output convention") as
`.claude/agents/runs/{YYYY-MM-DD}-{formName}/plan/user-stories.yaml` — create the
run directory + `plan/` subfolder if they don't exist yet. That file is the canonical handoff
(temporary/working notes go to the scratchpad dir, never into `runs/`).

```yaml
structured_requirements:
  delivery_type: new_single_form | new_multi_form_program | enhancement | migration
  business_objective: "<one paragraph>"
  source_inputs: ["<what was provided: brief, screenshot, swagger, legacy form path, ...>"]
  forms:
    - key: "permit-application"             # kebab-case working name
      title: "Building Permit Application"
      audience: authenticated | public
      channels: [web, mobile]
      locales: [en, fr]
      layout: single_page | wizard | tabs | accordion
      data:
        binding_intent: schema | fdm | none  # intent only; architect makes the final call
        source_of_truth: "<API/CRM/none>"
        api_spec: "<swagger/openapi path or 'none'>"
        api_auth: none | basic | oauth2 | unknown
        prefill: "<source or 'none'>"
        submit: [rest | email | workflow | fdm_writeback | dor_pdf]
      sections:
        - name: "Applicant Details"
          fields:
            - name: fullName
              label: "Full name"
              type: text
              required: true
              validations: ["max 100 chars", "letters/spaces only"]
              aria_label: "Full name"
            - name: email
              label: "Email"
              type: email
              required: true
              validations: ["valid email format"]
      rules:
        - "show 'Company name' when employmentType == 'salaried'"
        - "total = quantity * unitPrice (calculate)"
      workflow:
        needed: true | false
        summary: "2-step manager → finance approval; email on each; DoR on approval"
      style_spec:                            # ONLY for a URL/webpage replica (Step 1a) — else omit
        source_url: "<the fetched page URL>"
        layout: "2-column form; fields in multi-column rows as noted per section"
        field_order: [fullName, email, phone]  # exact source order
        fonts: "body 'Inter' 16px/400; labels 14px/600"
        colours: "text #1a1a1a; labels #333; border #ccc; primary button #0057b8 on #fff"
        borders: "1px solid #ccc; radius 4px"
        card: "white card, 1px #e0e0e0 border, 24px padding, subtle shadow"
        spacing: "24px between rows; 8px label→input gap"
        button: "full-width primary; #0057b8 bg; #fff text; 12px/24px padding; radius 4px"
        js_behaviour: ["show 'Other' text when reason == 'other'"]   # → create-form-rules
  user_stories:                              # traceability spine — every field/rule/submit covered
    - id: US-01
      form: "permit-application"             # which form key this story belongs to
      title: "Applicant provides identifying details"
      as_a: "permit applicant"
      i_want: "to enter my full name and email"
      so_that: "the authority can identify and contact me"
      acceptance_criteria:                   # MUST have at least one
        - id: AC-01.1
          criterion: "Full name is required and limited to 100 characters (letters/spaces only)"
        - id: AC-01.2
          criterion: "Email is required and must be a valid email format"
      covers_fields: [fullName, email]       # fields/rules/submit this story accounts for
      covers_rules: []
    - id: US-02
      form: "permit-application"
      title: "Salaried applicants declare their employer"
      as_a: "salaried applicant"
      i_want: "the company-name field to appear when I select 'salaried'"
      so_that: "I only fill in fields relevant to me"
      acceptance_criteria:
        - id: AC-02.1
          criterion: "When employmentType == 'salaried', the Company name field is shown and required"
      covers_fields: [employmentType, companyName]
      covers_rules: ["show 'Company name' when employmentType == 'salaried'"]
  nfrs:
    accessibility: "WCAG 2.1 AA; aria-label on every field"
    localization: "en + fr"
    volume: "~5k submissions/month"
    security_pii: "collects SSN — PII; mask + encrypt; no PII in logs"
    compliance: "<retention/regulatory or none>"
    branding: "<design system / existing project theme>"
    record_copy: "DoR PDF required | not required"
    bot_protection: "CAPTCHA required (public form) | not required"
    environments: "author + publish; dispatcher cache for assets"
  reference_for_ui_check: "<screenshot path / legacy URL for test-form-ui, or 'none'>"
  assumptions: ["<default chosen where the user couldn't answer>"]
  open_questions: ["<anything still blocking>"]
  acceptance_criteria: ["<how done is judged>"]
```

---

## Quality checklist

- [ ] Delivery type classified (new / multi / enhancement / migration); a migration records the
      original-UI reference for the later `test-form-ui` parity check
- [ ] Every form has audience (authenticated/public), channels, locales, and layout intent
- [ ] **Complete field inventory** — every field has label, type, required flag, and all validation
      constraints; each field has an `aria_label`
- [ ] **When the input is a screenshot/image/Figma/link, EVERY visible text and element is captured**
      — form header/title, subtitle, all section headings, helper/constraint text, brand header text,
      help line, mandatory legend, button labels, and per-element icons — recorded as display/static
      content or as an element requirement; nothing on-screen is silently dropped (exclusions are
      recorded under `assumptions`/`open_questions`)
- [ ] **When the input is a PUBLIC WEBPAGE URL**, the page was WebFetched, the `<form>` isolated, and
      BOTH captured: the **field inventory** (label/type/name/required/options/placeholder/validation →
      Core Components AF field types, in source order) AND the **`style_spec`** from the page CSS
      (layout/multi-column rows, fonts, colours, borders, card, spacing, button); JS behaviour noted for
      `create-form-rules`; `binding_intent: schema`, `submit: [dor_pdf]`, `reference_for_ui_check` = the
      SOURCE URL; ONLY the form captured (not page chrome). A JS-rendered form WebFetch can't read is
      flagged as a limitation and falls back to user-supplied fields (nothing invented)
- [ ] Data binding **intent** captured (schema/fdm/none) with source-of-truth + any API spec & auth
      (intent only — the architect decides)
- [ ] All business rules listed in plain language; layout (single/wizard/tabs/accordion) noted
- [ ] Workflow need captured (participants, stages, notifications, signature, DoR-on-approval)
- [ ] NFRs captured for the architect: accessibility, i18n, volume, security/PII, compliance,
      branding, record copy (DoR), bot protection (CAPTCHA), environments
- [ ] Assumptions and open questions recorded explicitly — nothing silently invented
- [ ] Requirements expressed as **user stories**; **every user story has ≥1 acceptance criterion**;
      every field, rule, and submit behaviour is covered by ≥1 story (the traceability spine that
      `design-form-tests` and `formwright` build on)
- [ ] Acceptance criteria stated
- [ ] Output emitted as the `structured_requirements` YAML; no resource types / skill names / JCR
      paths (that is `architect-form-solution`'s job)

---

## Hand off to

`architect-form-solution` — it consumes this `structured_requirements` object and produces the
Solution Architecture, Integration & NFR Strategy, and the ADLC execution plan.
