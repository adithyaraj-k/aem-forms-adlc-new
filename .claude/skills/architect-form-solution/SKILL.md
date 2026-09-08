---
name: architect-form-solution
description: >
  Solution Architecture for AEM Adaptive Forms on AEM as a Cloud Service. Consumes the Structured
  Requirements from discover-form-requirements and produces the SOLUTION ARCHITECTURE, the
  INTEGRATION & NFR STRATEGY, and an ADLC EXECUTION PLAN that maps every requirement to the
  project's EXISTING create-*/migrate-form/test-form-ui skills and their phases — deciding schema
  vs FDM, which template/theme to reuse vs build, whether a custom component is needed, the submit
  and prefill approach, workflow, tests, and the UI parity check. Use as the second half of the
  Planwright (PLAN) phase, after requirements discovery and before any build skill runs. This skill
  DESIGNS and PLANS; it does not author artifacts (the create-* skills do that during execution).
version: 1.0.0
ide:
  cursor: .cursor/skills/architect-form-solution/
  github-copilot: .github/skills/architect-form-solution/
  claude-code: .claude/skills/architect-form-solution/
---

# Skill: architect-form-solution

## Role

You are the **Solution Architect** on the Planwright (PLAN) team. You take the
`structured_requirements` from `discover-form-requirements` and the project tokens from
`.aem-forms-config.yaml`, and you turn them into a buildable plan: **which AEM artifacts are needed,
how they fit together, how external systems and NFRs are handled, and the exact ordered sequence of
existing skills/phases that delivers it.** You make the binding decisions discovery deferred
(schema vs FDM, reuse vs build), but you do **not** write code or content — the `create-*` skills do
that when the Program Agent executes your plan.

You never invent a skill or a phase that doesn't exist. The catalog you plan against is the project's
registered skills (below). If a requirement has no matching skill, you flag it as a gap rather than
inventing one.

---

## The catalog you plan against (existing skills / phases)

| Phase | Skill | Use it for |
|---|---|---|
| 1 | `generate-schema` | JSON Schema / XSD when the form is schema-bound (not FDM) |
| 2 | `create-editable-template` | a reusable form template (reuse an existing one if it fits) |
| 3 | `create-adaptive-form` | the form: fields, layout, panels, validation clientlib |
| 4 | `create-form-rules` | show/hide, validate, calculate, set-value, cascade |
| 5 | `create-form-component` | a custom field type Core Components doesn't provide |
| 6 | `create-submit-action` | custom submit: REST, email, workflow, PDF/DoR |
| 7 | `create-prefill-service` | pre-populate from API / profile / draft |
| 8 | `create-form-theme` | branding (reuse an existing project theme if one fits) |
| 9 | `create-form-clientlib` | shared client-side validation / custom functions |
| 10 | `create-form-tests` | unit + integration tests for Forms services |
| 11 | `migrate-form` | port a legacy/Foundation form to Core Components |
| 12 | `create-workflow` | approval/review/sign-off + "Invoke an AEM Workflow" submit |
| 13 | `test-form-ui` | compare rendered UI to a reference screenshot/URL; migration parity |
| (data) | `create-fdm` | a Form Data Model + data source (Swagger/REST) when FDM-bound |

Existing reusable assets in this project (prefer reuse over build):
- Themes under `/apps/fd/af/themes/{project}-{...}` (easel, fsi, healthcare, manufacturing, public, wknd).
- FDM is enabled; data sources already configured (salesforce, microsoft-dynamics-365).

---

## Step 1 — Decide the data backing (schema vs FDM vs none)

A form binds its data **at most one** way. Decide per form, using the same rule the
`create-adaptive-form` skill enforces.

### Decide by INPUT TYPE first (primary discriminator)

The kind of source the requirement was captured from is the **first** thing that decides
schema vs FDM — because it determines whether a real integration contract actually exists.

> ⚠️ **INPUT TYPE decides schema-vs-FDM ONLY — it does NOT gate prefill, submit/workflow, or
> fragments.** The image-vs-document discriminator below governs whether to provision an **FDM data
> source** (which genuinely cannot be fabricated from a picture). It must **never** be used to skip
> the integration phase. Prefill (7), submit-action/workflow (6/12), and Adaptive Form Fragment
> extraction are **ALWAYS** planned into the Groundsmith integration phase as "ask the user, then
> build" — on EVERY delivery, regardless of input type. See the mandatory rule right after this
> section, and [[integration-phase-always-runs]].

- **Screenshot / image, or a link to an HTML page → default to JSON Schema (`generate-schema`),
  NOT FDM.** A screenshot or rendered HTML page only conveys the *fields and layout* — it carries
  no system-of-record, endpoint, auth, or process definition. Inferring an FDM data source + OAuth
  off a picture is speculative (it produces stubbed Swagger/endpoints). Back the form with a JSON
  Schema. **Do NOT plan `create-fdm`** (the external data-source/write-back) unless the client
  separately supplies the integration logic — flag an FDM source as a future enhancement, not v1.
  This does NOT mean "PDF-only submit / no workflow / no prefill": the admin-alert **workflow**,
  the **prefill** question, and **fragment** extraction are still planned into the integration
  phase and decided WITH the user (never silently dropped to `dor_pdf` + `prefill:none`).
- **Requirement document (PRD/brief/spec) with proper integration logic → FDM (`create-fdm`) and,
  where the doc defines a review/approval/routing process, `create-workflow` ARE warranted.** Only a
  document that actually specifies the system of record, endpoints/operations, auth, and the
  process justifies provisioning an FDM source + write-back and an AEM workflow. **OAuth2** is the
  recommended production auth for a new REST source.
- **Existing AF XML / OpenAPI-Swagger spec supplied** → use that contract directly (Swagger → FDM
  if a live service is wanted, else `generate-schema`; existing AF → migrate/rebind).

If the input type conflicts with the integration depth (e.g. only a screenshot but the brief asks
for CRM write-back), record it as an **open question / assumption** and default to JSON Schema for
v1 until the integration contract is confirmed — do not fabricate an FDM source from a picture.

### Integration phase is MANDATORY — never skipped by input type (ask-then-build)

Regardless of input type (image, URL, brief, or existing form), the ADLC plan **MUST** route these
into the **Groundsmith IMPL-integration phase**, which **asks the user and then builds** — it is
**never** marked `skip: true` on the grounds that the input is an image / carries no integration
contract:

- **Prefill (phase 7)** — always planned. Groundsmith shows the field list and asks: Excel/CSV
  static prefill or none. Do NOT pre-decide `prefill: none` in the architecture.
- **Submit + admin-alert workflow (phases 6 / 12)** — always planned. The **default** submit is an
  admin-alert **workflow** ("Invoke an AEM Workflow" → assign-task / send-email to admin) **in
  addition to** the DoR PDF; Groundsmith confirms workflow-vs-PDF-only with the user. Do NOT default
  the plan to a bare `dor_pdf` submit with `workflow: none`.
- **Adaptive Form Fragments (`create-AdaptiveFormFragment`)** — always planned. Groundsmith asks
  which reusable panels (e.g. participant, address, consent) to extract as fragments (or none);
  the extraction is built via `create-AdaptiveFormFragment`.

In the `adlc_execution_plan`, express these as **"ask in integration phase, then build per answer"**
— NOT `skip: true`. The only thing the input-type rule may down-scope is the **FDM data source**
(`create-fdm`), for the reason above. (Standing rule — see [[integration-phase-always-runs]].)

### Then refine the decision

- **FDM** — if the form integrates with an external/REST service (Swagger/OpenAPI) or reuses a
  configured data source, or needs server-side read/write through a data model → plan `create-fdm`
  first, then bind the form to it. **OAuth2** is the recommended production auth for a new REST source.
- **Schema** — if the data is a structured/typed payload not backed by a live service, or a schema is
  reused across forms, or a Swagger/OpenAPI spec is supplied but FDM isn't wanted, **or the input was
  only a screenshot / HTML page** → plan `generate-schema` (default JSON Schema unless XSD is requested).
- **None** — a simple standalone collect-and-submit form → no schema, no FDM.

State the decision, the **input type it was based on**, and the one-line reason per form.

## Step 2 — Map requirements to artifacts (component coverage check)

For each form, walk the field inventory and rules and decide the artifact set:

- **Fields** → map each logical type to a Core Components field. If a required type isn't in Core
  Components (e.g. star rating, OTP, signature pad), flag **`create-form-component`** (Phase 5).
- **Layout** → single page vs `wizard`/`tabs`/`accordion` (existing layout containers).
- **Rules** → list each as a `create-form-rules` item with its rule type.
- **Template** → **reuse-first — do NOT plan a new editable template per form.** FIRST evaluate
  whether an existing template can be reused; only when none is usable do you plan
  `create-editable-template`. The repo already has ~20 templates under
  `/conf/{project}/settings/wcm/templates/` (basic-af, blank-af, blank-af-v2, and many form-specific
  ones); forms have been needlessly forking a new one each time — stop that.
  **Reuse-first decision:**
  1. **Enumerate existing templates** under `/conf/{project}/settings/wcm/templates/` (glob the repo
     AND check the running instance).
  2. A template is **REUSABLE** when: it is built on the same **template-type** (`af-page-v2` / Core
     Components); its **structure** (header / form container / footer, locked vs unlocked regions)
     fits the form; and its **content policy** allows the components the form needs. A generic base
     (`blank-af-v2` / `basic-af`) is broadly reusable. **Per-form theme/brand differences do NOT
     disqualify reuse** — the theme is a per-form theme + clientlib + conf context, not the template.
  3. A **reusable template exists** → set the form's template reference to it and plan
     `template: reuse:"<path>"` — **SKIP `create-editable-template`** (mark the phase `skip: true`).
  4. **No existing template fits** (needs a distinct locked structure, a distinct allowed-components
     policy, or a governed layout for a new form family) → plan `create-editable-template` (a new one).
  5. **Record the decision** (`reused: <name>` | `created: <name>` + reason) in the plan and the run
     implementation notes.
  The ADLC plan MUST therefore output either **"reuse template X (skip create-editable-template)"** or
  **"create new template (reason)"** for each form — never silently create a new template.
- **Theme** → reuse an existing project theme that matches branding; else `create-form-theme`.
- **Submit** → REST / email / workflow / FDM write-back / PDF-DoR → `create-submit-action` (or built-in).
- **Prefill** → `create-prefill-service` with the variant (REST / CRX-DAM / draft).
- **Workflow** → `create-workflow` (+ `create-submit-action` if PDF/DoR on approval).
- **Validation clientlib** → always part of `create-adaptive-form`; note shared functions for `create-form-clientlib`.
- **Tests** → `create-form-tests` for any Java service; **`test-form-ui`** against the acceptance
  reference (and as the migration parity check for a migration).
- **Migration** → if delivery_type is migration, the spine is `migrate-form` (Phase 11) instead of 1–3.

## Step 3 — Integration & NFR strategy

Translate the discovery NFRs into concrete plan decisions:

- **Data/integration:** data sources + auth (FDM OAuth2/Basic), external systems, write-back targets.
- **Accessibility:** WCAG target; `aria-label` on every field (enforced by `create-adaptive-form`).
- **Localization:** locales, dictionary/translation approach.
- **Performance/volume:** caching/dispatcher for static assets; submission throughput considerations.
- **Security/PII:** PII fields → masking/encryption, no PII in logs, `$[secret:...]` for credentials,
  service users via repoinit; CAPTCHA (`recaptcha`/`turnstile`/`hcaptcha`) for **public** forms.
- **Record copy:** Document of Record where a PDF receipt/retention is required.
- **Environments:** author/publish, dispatcher, secrets per environment.

## Step 4 — Produce the ADLC Execution Plan

Emit the buildable plan the Program Agent executes. Order phases by the existing dependency map,
mark parallelism, and list skipped phases with a reason. This is the artifact that drives the build.
**Write it to the run directory** (AGENTS.md → "Run output convention") as
`.claude/agents/runs/{YYYY-MM-DD}-{formName}/plan/solution-architecture.yaml` (the `plan/`
SDLC-cycle subfolder; temporary/working files go to the scratchpad dir, never into `runs/`).

```yaml
solution_architecture:
  forms:
    - key: "permit-application"
      data_backing: fdm | schema | none
      data_reason: "<one line>"
      template: reuse:"<path>" | build
      theme: reuse:"/apps/fd/af/themes/{project}-public" | build
      custom_components: ["signature-pad"]   # or []
      submit: "rest -> CRM via FDM write-back"
      prefill: "REST user profile"
      workflow: "manager -> finance approval; DoR on approve"
integration_nfr_strategy:
  data_sources: "<source + auth, e.g. new REST source, OAuth2 client-credentials>"
  accessibility: "WCAG 2.1 AA; aria-label on all fields"
  localization: "en, fr"
  security_pii: "SSN masked + encrypted; secrets via $[secret:]; CAPTCHA on public form"
  record_copy: "DoR PDF via create-workflow Generate-DoR step"
  performance: "dispatcher cache theme/clientlib; ~5k/month submit"
  environments: "author+publish; per-env OAuth secret"
adlc_execution_plan:
  - phase: 0   ; skill: ensure-forms-agents-md ; note: "config present — verify only"
  - phase: data; skill: create-fdm             ; note: "REST source (OAuth2) + FDM model"
  - phase: 1   ; skill: generate-schema        ; note: "SKIP — FDM-bound"
    skip: true
  - phase: 2   ; skill: create-editable-template; note: "reuse blank-af-v2 — SKIP build"
    skip: true
  - phase: 3   ; skill: create-adaptive-form   ; note: "permit-application; wizard; public → CAPTCHA"
  - phase: 4   ; skill: create-form-rules      ; note: "show company-name; calculate total"
  - phase: 5   ; skill: create-form-component  ; note: "signature-pad"
  - phase: 6   ; skill: create-submit-action   ; note: "FDM write-back to CRM"
  - phase: 7   ; skill: create-prefill-service ; note: "REST user profile"
  - phase: 8   ; skill: create-form-theme      ; note: "reuse {project}-public — SKIP build"
    skip: true
  - phase: 12  ; skill: create-workflow        ; note: "manager→finance; DoR on approve"
  - phase: 10  ; skill: create-form-tests      ; note: "submit action + prefill unit tests"
  - phase: 13  ; skill: test-form-ui           ; note: "compare to acceptance design screenshot"
  parallelizable: ["6 with 7", "8 with 4-7"]
gaps: ["<requirement with no matching skill, if any>"]
risks: ["<integration/perf/security risk + mitigation>"]
```

---

## Quality checklist

- [ ] Data backing decided per form (schema / FDM / none) with a one-line reason; FDM sources note
      auth (OAuth2 recommended for new production REST)
- [ ] Every field mapped to a Core Components field or flagged for `create-form-component`; no
      requirement left unmapped (anything unmappable listed under `gaps`)
- [ ] **Template decided reuse-first** — an existing template was enumerated and evaluated (same
      `af-page-v2` type + fitting structure + allowed-components policy); the plan says
      `reuse:"<path>" (skip create-editable-template)` where one fits, and `create new template
      (reason)` ONLY when none does. Do NOT fork a new template per form. Theme/brand is decided
      separately (per-form theme + clientlib + conf) and never justifies a new template
- [ ] Submit, prefill, rules, workflow, clientlib, tests, and the UI check each assigned to a real
      catalog skill — no invented skills or phases
- [ ] Integration & NFR strategy covers data/auth, accessibility, i18n, performance, security/PII
      (+ CAPTCHA for public forms), record copy (DoR), and environments
- [ ] ADLC execution plan is ordered per the program-agent dependency map, marks parallelism, and
      lists skipped phases with a reason
- [ ] Migration deliveries use `migrate-form` (Phase 11) as the spine and include the `test-form-ui`
      structural-parity check against the original
- [ ] `gaps` and `risks` recorded explicitly

---

## Hand off to

The **Planwright** lead consolidates this with the Structured Requirements into the PLAN-phase
output, which the **aem-forms-program-agent** then executes phase by phase.
