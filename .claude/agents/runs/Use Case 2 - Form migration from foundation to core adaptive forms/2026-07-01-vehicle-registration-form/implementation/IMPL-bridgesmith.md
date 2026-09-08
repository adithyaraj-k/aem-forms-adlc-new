# IMPL — Bridgesmith (INTEGRATION) · Consolidated Integration Summary

- **Form:** `vehicle-registration-form`
- **runId:** `2026-07-01-vehicle-registration-form`
- **Lead:** bridgesmith (IMPL integration) — second of `blockwright → bridgesmith → auditon → sentinel`
- **Scope:** prefill + submit + workflow integration only (form/schema build owned by Blockwright; build/deploy by Auditon; test by Sentinel)
- **Author-only:** YES — no `mvn` build/deploy run; deployment deferred to Auditon (centralized). The delegated skill was instructed author-only.
- **Overall gate:** PASS

---

## Integration phases (per confirmed plan)

| Phase | Skill | Status | Deliverable file |
|-------|-------|--------|------------------|
| 6 | create-submit-action (SHARED DoR) | **PASS** | `phase06-create-submit-action.md` |

### Phases skipped (confirmed by PLAN — do NOT run)

| Phase | Skill | Reason |
|-------|-------|--------|
| 7 | create-prefill-service | Anonymous public form, no source of truth — nothing to prefill (PLAN `prefill.needed: false`). |
| 12 | create-workflow | No approval/review/routing/e-signature — plain PDF-on-submit only (PLAN `workflow.needed: false`). |

---

## Phase 6 — Shared Document-of-Record submit action (US-10)

**Outcome: EXISTED — fully present and correctly REUSED.** The one shared, generic
`Custom-Submit-GeneratePDF` action (single OSGi service + single GeneratePDFServlet + single
clientlib) was already in the repo from a prior run. No shared artifact was duplicated and **no
per-form PDF action was scaffolded**. Blockwright had pre-wired the form; Bridgesmith verified the
wiring end-to-end against the real guideContainer through the `create-submit-action` skill's gate.

### Shared artifacts (created once, reused by every form)

| Artifact | Path | Notes |
|----------|------|-------|
| OSGi service | `core/src/main/java/com/aem/forms/agents/forms/submit/CustomSubmitGeneratePDFAction.java` | `implements FormSubmitActionService`; `getServiceName()=="Custom-Submit-GeneratePDF"`; `submit()` returns `GuideConstants.FORM_SUBMISSION_COMPLETE=TRUE`; never throws |
| JCR node | `/apps/{project}/Custom-Submit-GeneratePDF` (`ui.apps/.../apps/{project}/Custom-Submit-GeneratePDF/.content.xml`) | `sling:Folder`; `guideComponentType="fd/af/components/guidesubmittype"`; `guideDataModel="basic,xfa,xsd"`; `submitService="Custom-Submit-GeneratePDF"` |
| Shared servlet | `core/src/main/java/com/aem/forms/agents/core/servlets/GeneratePDFServlet.java` | POST-bound `/bin/{project}/generatePDF`; renders PDF, uploads to DAM, streams download; DAM write via service-user resolver in **try-with-resources**; no secrets; zero new Maven dependency |
| Shared clientlib | `/apps/{project}/clientlibs/clientlib-generate-pdf` | category `{project}.forms.generate-pdf`; `js.txt` (`#base=js` + `generate-pdf.js`); collects fields generically and POSTs to the servlet |
| Unit test (authored this phase) | `core/src/test/java/com/aem/forms/agents/forms/submit/CustomSubmitGeneratePDFActionTest.java` | the ONLY new file — completes the shared action's quality checklist |

### JCR node path — deliberate, skill-correct convention

The AGENTS.md generic table lists per-project submit actions under
`/apps/{project}/fd/af/submitactions/{action}`. The **shared DoR action deliberately uses the
`create-submit-action` skill's own canonical shared-action path** `/apps/{project}/Custom-Submit-GeneratePDF`
(SKILL.md lines 452, 466-468). The existing node is at exactly that skill-correct location and is
covered by the `ui.apps` filter.xml. **This is not a deviation to fix** — reusing it as-is is the
mandated behaviour ("REUSE it, do NOT duplicate").

### Both-halves rule (non-negotiable) — SATISFIED

Submit action has **BOTH** the OSGi service **AND** the JCR node, and
`getServiceName()` (Java) === node `submitService` === `"Custom-Submit-GeneratePDF"` (the identity
the runtime uses to bind the action at submit time).

---

## Wiring verified on the REAL guideContainer

Form: `ui.content/src/main/content/jcr_root/content/forms/af/{appFolder}/vehicle-registration-form/.content.xml`

| Property | Value | Meaning |
|----------|-------|---------|
| `submitService` | `Custom-Submit-GeneratePDF` | binds to the shared OSGi action (`getServiceName()` match) — **proves US-10** |
| `actionType` | `fd/af/components/guidesubmittype/submitservice` | Core Components submit-service marker — **same proven pattern as the deployed health-insurance-form** (not churned to the alternate resourceType form) |
| `clientLibRef` | `{project}.forms.vehicle-registration-form,{project}.forms.generate-pdf` | loads BOTH the per-form validation clientlib AND the shared PDF clientlib on this form |
| `thankYouOption` / `thankYouMessage` | `message` / "Thank you. Your vehicle registration details have been submitted." | acknowledgement shown after successful submit |
| `dorType` | `none` | DoR is produced by the shared servlet path (clientlib → GeneratePDFServlet → DAM + download), not the container DoR renderer |

No dangling reference: `submitService` points at an action that fully exists (service + node + servlet + clientlib all present).

---

## Integration user-story satisfaction

| Story | AC | Satisfied by | Status |
|-------|----|--------------|--------|
| **US-10** — Owner submits and receives a PDF record | AC-10.1 Submit blocked until required fields valid | Required flags + `fd:validate` AST rules (built by Blockwright) enforce validity before submit | PASS |
| | AC-10.2 On success, PDF DoR generated via shared `Custom-Submit-GeneratePDF` and made available to owner | Shared OSGi action + `GeneratePDFServlet` (PDF → DAM + browser download) + shared PDF clientlib, wired onto this form's guideContainer | PASS |

**integration_stories_satisfied: 1 (US-10)**
**integration_stories_unsatisfied: 0**

(Non-integration stories US-01..US-09, US-11, US-12, US-13 were satisfied build-side by Blockwright and are out of Bridgesmith's lane.)

---

## Compliance with critical rules

- Reused existing agent (`create-submit-action`); nothing hand-authored by the lead.
- Submit action = BOTH OSGi service AND JCR node.
- Core Components only; no Foundation types authored (the `fd/af/components/guidesubmittype/submitservice` marker is the expected CC submit-service pattern).
- try-with-resources for ResourceResolver in `GeneratePDFServlet`; no hardcoded secrets (none needed — no external integration); OSGi via annotations (no per-config `.cfg.json` required for these components).
- `{project}` (`{project}`) vs `{appFolder}` (`{appFolder}`) kept distinct.
- Author-only — no deploy run; deferred to Auditon.

---

## For Auditon (build/deploy)

1. Run the single authoritative `mvn clean install -PautoInstallSinglePackage` (no `-pl`).
2. Confirm the new unit test `CustomSubmitGeneratePDFActionTest` compiles and passes.
3. Confirm OSGi components register on deploy: `CustomSubmitGeneratePDFAction` (FormSubmitActionService) and `GeneratePDFServlet` (Servlet, POST).
4. Confirm repoinit creates the `pdf-writer-service` system user and the mapping `{project}.core:pdf-writer=pdf-writer-service`.
5. Confirm filter.xml installs `/apps/{project}/Custom-Submit-GeneratePDF` and the `clientlib-generate-pdf` clientlib.

## For Sentinel (test)

1. On Submit of a valid form, the browser auto-downloads the PDF.
2. A PDF copy lands under `/content/dam/{project}/`.
3. The POST to `/bin/{project}/generatePDF` returns 200.
4. The configured thank-you message displays after submit.
5. Submit is blocked while required fields are empty/invalid (AC-10.1).

---

## Handoff YAML

```yaml
agent: bridgesmith
phase: IMPL-integration
status: PASSED
phases_executed: [6]
phases_skipped:
  - "7 create-prefill-service — anonymous public form, no source of truth (nothing to prefill)"
  - "12 create-workflow — no approval/review/routing/e-signature (plain PDF-on-submit only)"
integration_stories_satisfied: 1      # US-10
integration_stories_unsatisfied: 0
shared_action: EXISTED_AND_REUSED     # no duplication, no per-form action
artifacts:
  prefill: none
  submit_action: "/apps/{project}/Custom-Submit-GeneratePDF (JCR node) + com.aem.forms.agents.forms.submit.CustomSubmitGeneratePDFAction (OSGi service) + com.aem.forms.agents.core.servlets.GeneratePDFServlet + clientlib {project}.forms.generate-pdf"
  workflow: none
integration_summary: ".claude/agents/runs/2026-07-01-vehicle-registration-form/implementation/IMPL-bridgesmith.md"
gate_result: PASS
next: aem-forms-program-agent runs auditon (build/deploy) -> sentinel (test)
```
