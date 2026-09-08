---
name: design-form-tests
description: >
  Test Design for AEM Adaptive Forms on AEM as a Cloud Service. Consumes the Structured Requirements
  and acceptance criteria (from the Planwright) plus the Component Inventory & Specs and Design
  Specifications (from design-form-components), and produces the TEST CASES — a structured, traceable
  test plan covering field validation, business rules, submit/prefill, accessibility, responsive/cross
  browser, UI parity, and performance, with each case mapped to the skill that will EXECUTE it
  (create-form-tests for unit/integration, test-form-ui for visual, Sentinel for E2E/perf). Use as the
  second half of the Draftsmith (DESI) phase. This skill DESIGNS test cases (the plan) — it does NOT
  write JUnit/Selenium/Cypress code; that is create-form-tests / test-form-ui during implementation/test.
version: 1.0.0
ide:
  cursor: .cursor/skills/design-form-tests/
  github-copilot: .github/skills/design-form-tests/
  claude-code: .claude/skills/design-form-tests/
---

# Skill: design-form-tests

## Role

You are the **Test Design** agent on the Draftsmith (DESI) team. You turn the requirements,
acceptance criteria, and technical design into an unambiguous **test plan**: a set of test cases that
prove the form does what the requirements say. You design the cases; you do **not** implement them —
`create-form-tests` (JUnit/IT) and `test-form-ui` (visual) build and run them later, and Sentinel
(TEST) automates the suite.

Every test case traces to a **user story and one of its acceptance criteria** (the traceability
spine the Planwright produced). You cover the happy path, the empty/invalid path, and the error path
for each behaviour, and you do not leave a user story or any of its acceptance criteria without at
least one case.

---

## Trigger

Runs as the second step of the DESI phase, after `design-form-components`, when test cases are needed
before implementation/testing.

---

## Step 0 — Read the inputs

From the run directory (AGENTS.md → "Run output convention"):
- `.claude/agents/runs/{runId}/plan/user-stories.yaml` — requirements + the
  **`user_stories`** (each with its acceptance criteria) + the `reference_for_ui_check`
  (screenshot/legacy URL). The `user_stories` and their acceptance criteria are the traceability
  spine: every story and every acceptance criterion must end up with ≥1 case.
- `.claude/agents/runs/{runId}/design/component-design-spec.yaml` — component specs + design specs
  (so cases reference real field names, validations, and layout).

## Step 1 — Derive scenarios by category

For each form, walk the requirements and design and derive cases across these categories:
- **Field validation** — required, format/pattern, length, range, file type/size, allowed enums.
- **Business rules** — show/hide, enable/disable, make-mandatory, calculate, set-value, cascade.
- **Submit** — valid submit reaches the action; empty submit blocks with messages; error handling.
  Always include a **submit-is-gated-on-validation** case: an empty/invalid form must BLOCK submission,
  show inline errors, and produce **NO PDF/Document-of-Record**; a valid form submits AND generates the
  PDF. (User story: "submission only succeeds when validation passes.")
- **Prefill** — fields populate from the source; empty/unauthenticated path is safe.
- **Accessibility** — `aria-label` present; keyboard nav; focus; contrast (WCAG target).
- **Responsive / cross-browser** — layout at each breakpoint; supported browsers.
- **UI parity** — rendered form matches the reference design (drives `test-form-ui`).
- **Performance** — load/Core Web Vitals where volume/NFRs warrant (drives Sentinel).

## Step 2 — Write each test case and map it to its executor

Each case: a stable id, title, level/type, the field/rule it targets, preconditions, steps, expected
result, **the user story + acceptance criterion it traces to**, and **which skill executes it**.
Levels map to the catalog:
- `unit` / `integration` (services, Sling Models, submit/prefill) → `create-form-tests` (Phase 10).
- `ui-visual` (parity vs reference) → `test-form-ui` (Phase 13).
- `e2e` / `performance` / `functional` → the **`sentinel`** TEST lead (it also drives `create-form-tests`
  and `test-form-ui` and verifies story coverage).

## Step 3 — Write the test cases to the run directory

Emit ONE file `.claude/agents/runs/{runId}/design/test-cases.yaml` (create the run
directory + `design/` subfolder if absent; temporary/working files go to the scratchpad dir, never
into `runs/`).

```yaml
test_cases:
  forms:
    - key: "permit-application"
      cases:
        - id: TC-001
          title: "Full name required + max 100 chars"
          category: field_validation
          level: ui-visual            # or unit | integration | e2e | performance
          target: "fullName"
          preconditions: "form loaded"
          steps: ["leave empty + submit", "enter 101 chars"]
          expected: "mandatory message on empty; length error at 101"
          executes_with: test-form-ui     # create-form-tests | test-form-ui | sentinel
          traces_to_story: US-01          # the user story this case proves
          traces_to_ac: AC-01.1           # the specific acceptance criterion
        - id: TC-014
          title: "Submit posts to CRM via FDM write-back"
          category: submit
          level: integration
          target: "guideContainer submit"
          preconditions: "valid form data"
          steps: ["fill required", "submit"]
          expected: "FDM write succeeds; thank-you shown"
          executes_with: create-form-tests
          traces_to_story: US-07
          traces_to_ac: AC-07.1
        - id: TC-020
          title: "Submit is gated on validation — invalid form makes no PDF"
          category: submit
          level: e2e
          target: "guideContainer submit + PDF/DoR generation"
          preconditions: "form loaded; required fields empty/invalid"
          steps: ["click Submit on empty/invalid form", "then fill valid data + Submit"]
          expected: "empty/invalid: submission blocked, inline errors, first invalid field focused, NO PDF; valid: submits AND PDF generated"
          executes_with: sentinel
          traces_to_story: US-08
          traces_to_ac: AC-08.1
        - id: TC-031
          title: "Rendered form matches approved design"
          category: ui_parity
          level: ui-visual
          target: "whole form"
          preconditions: "deployed; reference screenshot available"
          steps: ["capture form", "compare to reference"]
          expected: "zero Critical visual findings"
          executes_with: test-form-ui
          traces_to_story: US-09
          traces_to_ac: AC-09.1
coverage:
  user_stories_total: 0
  user_stories_with_cases: 0
  acceptance_criteria_total: 0
  acceptance_criteria_with_cases: 0
  uncovered_stories: []              # MUST be empty — every user story needs >=1 case
  uncovered_acceptance_criteria: []  # MUST be empty — every acceptance criterion needs >=1 case
executes_with_summary:
  create-form-tests: [TC-014]
  test-form-ui: [TC-001, TC-031]
  sentinel: []
```

---

## Quality checklist

- [ ] User stories + acceptance criteria + UI reference read from the run dir; component/design specs
      read so cases use real field names and validations
- [ ] Cases span all relevant categories: field validation, rules, submit, prefill, accessibility,
      responsive/cross-browser, UI parity, performance
- [ ] Each behaviour covers happy + empty/invalid + error paths where applicable
- [ ] Every case has id, level/type, target, preconditions, steps, expected result, `traces_to_story`
      + `traces_to_ac`, and `executes_with` a REAL catalog skill (create-form-tests / test-form-ui /
      sentinel)
- [ ] Full traceability — `uncovered_stories` AND `uncovered_acceptance_criteria` are empty; every
      user story and every acceptance criterion has ≥1 case
- [ ] Output written to `.claude/agents/runs/{runId}/design/test-cases.yaml`; test cases (plan)
      only, no JUnit/Selenium/Cypress code

---

## Hand off to

The **Draftsmith** lead consolidates these test cases with the component/design specs into the DESI
package; `create-form-tests` and `test-form-ui` later implement/run the cases, and Sentinel automates
the suite.
