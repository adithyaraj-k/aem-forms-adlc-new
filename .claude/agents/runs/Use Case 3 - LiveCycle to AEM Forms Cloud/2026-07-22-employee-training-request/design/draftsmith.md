# Draftsmith — DESI Package
## Employee Training Request — LiveCycle → AEM as a Cloud Service (Path B migration)

Run: `.claude/agents/runs/2026-07-22-employee-training-request/`
Reads: `plan/planwright.md`, `plan/user-stories.yaml`, `plan/solution-architecture.yaml`
Produces: `design/component-design-spec.yaml`, `design/test-cases.yaml` (this file consolidates both)

---

## 1. Technical Design (`design-form-components`)

### Component inventory
**30 fields** across 6 panels on the primary interactive Adaptive Form `employee-training-request`
(built from `DocRoute.xdp`), all mapped to stock Core Components — no custom component needed
(confirmed: text-input, email-input, number-input, dropdown, checkbox, datepicker, file-attachment
cover every field type in the inventory). Every field carries a mandatory `aria_label`, including the
two same-visible-caption "Location" fields, disambiguated as "Employee Location" vs "Training
Location".

### Fragment decisions (project rule 3c)
Checked the repo for existing Adaptive Form Fragment content first — none exist yet, so both are NEW:
- **`employee-identity`** (fragment) — EmployeeId, EmployeeName, Email, Department, ManagerId,
  ManagerName, Location, BusinessUnit. Matches the project's explicit "contact / personal details"
  fragment-reuse pattern; reusable by future internal HR-style forms. Canonical shape `$.employee.*`,
  remapped to `$.EmployeeDetails.*` at embed time to preserve the legacy XSD node names the
  workflow/DoR depend on.
- **`declaration-consent`** (fragment) — AcceptedTerms, SubmissionDate. Matches the explicit
  "declaration / consent" pattern. Canonical shape `$.declaration.*`, remapped to `$.Declaration.*`.
- **Approval Information — kept `source: form`, explicitly NOT fragmented.** Considered and rejected:
  it's tightly coupled to this workflow's own `CurrentStatus` vocabulary (Submitted/ManagerApproved/
  DeptHeadApproved/FinanceApproved/Completed/Rejected) and two state-driven rules keyed on that exact
  vocabulary — fragmenting it would force an artificial shared status model onto unrelated future
  workflows.
- **Training Request Details / Justification / Attachments — `source: form`** (domain-specific to this
  training-request use case, no reuse pattern evidenced).
- **Both fragments are reused a second time inside the Document-of-Record template** (see below),
  doubling their reuse value — single source of truth across the interactive form and its generated PDF.

### Theme — token overrides (Option A), NEW theme
No existing project theme (`easel`/`fsi`/`healthcare`/`manufacturing`/`public`/`wknd`) matches the
exact-replica palette, so a new theme is justified and built as a **thin `--af-*` token-override layer**
over the shared base clientlib — not a full stylesheet. Tokens sourced directly from confirmed
`<font>/<fill>/<color>` values in `DocRoute.xdp` plus the 3 DAM preview JPEGs I inspected directly as
visual ground truth:

| Token | Value | Evidence |
|---|---|---|
| `--af-font` | `'Myriad Pro', Arial, sans-serif` | `<font typeface="Myriad Pro"/>` throughout |
| `--af-heading-size` / `--af-heading-weight` | `20pt` / `bold` | `<font size="20pt" weight="bold">` |
| `--af-section-heading` | `rgb(128,64,64)` | `<color value="128,64,64"/>` (maroon caption) |
| `--af-section-heading-bg` | `rgb(196,219,251)` | `<color value="196,219,251"/>` (blue header band) |
| `--af-bg` | `rgb(225,242,219)` | `<color value="225,242,219"/>` (green panel body) |
| `--af-field-border` | `1px solid #8c8c8c inset` | approximates `<edge stroke="lowered"/>` — **flagged as an extrapolation**, not an RGB-captured value |
| `--af-primary` (submit button) | `rgb(128,64,64)` | **extrapolated** — no button exists in the static XDP (Workspace chrome supplied it legacy-side), matches user-stories.yaml's own recorded assumption |
| `--af-radius` | `0px` | square-cornered panels/fields in every DAM preview |

I visually confirmed via the 3 DAM preview JPEGs that there is **no additional hairline
divider** under section headers — the header-band-colour-to-panel-body-colour transition is itself the
only visual boundary, and sections aren't numbered. I specced this explicitly as an intentional
deviation from the generic "add a border-bottom divider" default, rather than inventing a line the
source doesn't have. No logos/icons/images appear anywhere in the 3 references, so `icon_assets: []`
is evidence-based, not an oversight.

### Authoring Guideline
Template: **reuse** `blank-af-v2` (create-editable-template skipped, per Planwright). Allowed
components enumerated; CAPTCHA components explicitly disallowed (internal authenticated form). DoR
authoring approach: the 2 non-interactive XDPs (`EmployeeTrainingRequest.xdp`, `ApprovalInfo.xdp`) are
**not** built via `create-adaptive-form` — they're authored as the single merged Document-of-Record
template inside `create-workflow`'s "Generate Document of Record" step, reusing both fragments plus
form-specific panels, all rendered display-only, themed with the same new theme (no second theme
build).

---

## 2. Test Design (`design-form-tests`)

**34 test cases** (`TC-001`–`TC-034`) traced to all **11 user stories** and all **17 acceptance
criteria** from `plan/user-stories.yaml`. Coverage gate confirmed: `uncovered_stories: []`,
`uncovered_acceptance_criteria: []` — every story and every AC has ≥1 case.

Explicitly covered per the task's required areas:
- All **8 migrated business rules** (TC-003/004, TC-005/006, TC-007/008, TC-009/010, TC-014/015,
  TC-018/019, TC-021/022/023).
- **Submit → workflow trigger** (TC-002, TC-030).
- **Submit-gated-on-validation** (project rule 3b) — `TC-020`, the mandatory case: invalid form
  (missing required field + AcceptedTerms unchecked + EndDate≤StartDate) blocks submit with 0 workflow
  instances / 0 PDFs; valid form submits, starts the workflow immediately, and produces exactly 1 PDF
  only after both approvals succeed.
- **Both Assign Task steps → administrators/admin group** — `TC-013`, explicitly recording the
  real-manager-lookup and distinct-finance-approver items as **accepted pre-production follow-ups**,
  not failures, per the user's confirmed decisions.
- **DoR generation** (merged EmployeeTrainingRequest + ApprovalInfo content) — `TC-016`, `TC-025`,
  `TC-026`.
- **Send-Email no-op-on-localhost** — `TC-032`, explicit accepted/expected condition: steps must
  complete without throwing; actual delivery is not asserted on the unconfigured local SDK.
- **UI parity** vs all 3 DAM preview JPEGs — `TC-024` (test-form-ui, live rendered form vs
  `DocRoute.xdp`, structural not pixel-perfect) and `TC-025` (sentinel, DoR PDF vs
  `EmployeeTrainingRequest.xdp` + `ApprovalInfo.xdp`, structural — not a live-page pixel diff since
  it's a generated PDF).
- **Accessibility** — `TC-027`, spot-checks a representative field from every panel/fragment plus the
  two distinctly-labelled "Location" fields, plus keyboard operability.
- **Fragment reuse regression** — `TC-028`/`TC-029`, edit-once-reflects-both across the interactive
  form and the DoR template for each of the two new fragments.

Executor split: `create-form-tests` (2 — submit action + prefill service unit/integration),
`test-form-ui` (1 — live-page visual parity), `sentinel` (31 — functional/e2e/workflow/accessibility/
fragment-regression, consistent with Sentinel being the final quality gate that also verifies full
story coverage).

---

## 3. Gate status

**PASS.**
- Component inventory: 30/30 fields specced, 0 custom components needed, 2 new fragments specced with
  justification, theme built as token overrides with every colour/font traced to XDP evidence or
  explicitly flagged as an extrapolation.
- Test design: 34 cases, 11/11 user stories covered, 17/17 acceptance criteria covered, 0 uncovered.
- `gaps: []` in the design-skill sense — the 3 business-decision open questions carried from Planwright
  (manager-lookup source, distinct finance-approver, sender address) are recorded as authoring-guideline
  dos/don'ts and as accepted test-case follow-ups, not silently built around or silently dropped.

---

## Run metrics

- `time_taken_minutes`: ~19.0
- `tokens_consumed` (4-way split, best estimate):
  - `cli_text`: ~16,000
  - `read`: ~40,000 (3 PLAN yaml/md files, 3 DAM preview JPEGs, XDP grep scans, repo fragment-existence checks)
  - `write`: ~21,000 (component-design-spec.yaml, test-cases.yaml, this draftsmith.md)
  - `other`: ~8,000 (tool-call overhead, bash/glob/grep scan output, skill-invocation scaffolding)
  - `total`: ~85,000

---

## Handoff YAML

```yaml
agent: draftsmith
phase: DESI
status: PASSED
time_taken_minutes: 19.0
tokens_consumed:
  cli_text: 16000
  read: 40000
  write: 21000
  other: 8000
  total: 85000
produces:
  component_inventory_and_specs: present
  design_specifications: present
  authoring_guideline: present
  test_cases: present
custom_components_specced: 0
fragments_specced: 2          # employee-identity, declaration-consent (both new, both reused a 2nd time in the DoR template)
theme: build
template: reuse
test_cases: 34
uncovered_requirements: 0
gaps: 0
outputs:
  - ".claude/agents/runs/2026-07-22-employee-training-request/design/draftsmith.md"
  - ".claude/agents/runs/2026-07-22-employee-training-request/design/component-design-spec.yaml"
  - ".claude/agents/runs/2026-07-22-employee-training-request/design/test-cases.yaml"
gate_result: PASS
next: aem-forms-program-agent executes implementation/test phases against these specs & cases
```
