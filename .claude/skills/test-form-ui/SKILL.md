---
name: test-form-ui
description: >
  Verifies a deployed Adaptive Form's functional and visual parity with Playwright. Use for
  reference screenshot/URL comparison, form UI regression checks, accessibility checks, or
  post-deployment validation. Cypress is retired for this project.
version: 2.0.0
ide:
  cursor: .cursor/skills/test-form-ui/
  github-copilot: .github/skills/test-form-ui/
  claude-code: .claude/skills/test-form-ui/
---

# Skill: test-form-ui

## Role

Validate an Adaptive Form with the project's Playwright harness at `ui.tests/test-module`. This
skill replaces the retired Cypress workflow. It does not create a second test stack and it does not
modify the deployed form to make a check pass.

## Inputs and gate

Require a deployed target URL and one parity reference: a legacy URL, screenshot, or approved design.
For an ADLC Sentinel run, use the target authorized by the program agent. Verify the page returns
200 and contains the expected new form before browser checks. If either condition is not met, write
a blocked report with the observed evidence.

## Playwright execution

1. Reuse the existing `ui.tests/test-module` Playwright configuration and specs. Do not invoke or
   install Cypress.
2. Set `FORM_URL` to the deployed embedded-form page, or direct form URL when that is the specified
   test surface. Run the project Playwright command, normally `npm test` from that module.
3. Capture desktop (1440x900) and mobile (390x844) screenshots. Keep machine artifacts in
   `ui.tests/test-module/results/`; never copy screenshots into the run folder.
4. Test required fields, migrated rules, reset, submit behavior, and configured PDF/workflow/prefill
   behavior. Inspect browser console errors.
5. Run configured accessibility checks (`@axe-core/playwright` when present), classify visual
   differences by severity, and block PASS on a critical regression.

## Report

Write a text-only `test-form-ui-report.md` in the run's `testing/` folder. Include target URL,
reference, Playwright command/result, viewport results, rule/submit evidence, console and a11y
findings, visual verdict, and links to working artifacts under `ui.tests/test-module/results/`.

## Quality checklist

- [ ] Playwright was used; no Cypress command, configuration, result, or artifact was used.
- [ ] Target returns 200 and serves the intended migrated form.
- [ ] Desktop and mobile captures were taken and compared to the reference.
- [ ] Applicable functional cases were executed and recorded.
- [ ] The run report is text-only and machine artifacts remain outside `runs/`.
