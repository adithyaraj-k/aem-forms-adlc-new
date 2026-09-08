# AEM Adaptive Forms — Agentified Delivery Life Cycle (ADLC)

**Multi-level agent orchestration specification for AEM Adaptive Forms delivery on AEM as a Cloud Service.**

> Throughout this spec, `{project}`, `{package}`, `{group}`, `{formsContentRoot}`, `{damContentRoot}`, `{schemaContentRoot}`, `{defaultTheme}`, and `{fdmRoot}` are placeholders resolved at session start from `.aem-forms-config.yaml`. Specialists MUST read that file first and substitute these values verbatim — no agent infers project identifiers from the file system. See `AGENTS.md` → "Project identity" for the resolved current values.

This document defines the **AEM Forms Program Agent** and a catalog of **10 specialist agents** that together cover the full delivery lifecycle for AEM Adaptive Forms on AEM as a Cloud Service. Each specialist owns a stage, draws on one or more skills under `.claude/skills/`, produces a structured artifact, and surfaces validation gates before downstream specialists pick up its output.

> **History.** Early runs under `.claude/agents/runs/` (e.g. `2026-07-01-vehicle-registration-form`, whose handoff files are named `PLAN-strategist.md`, `DESI-design-forge.md`, `IMPL-blockwright.md`, `IMPL-bridgesmith.md`, `AUDITON-code-quality-report.md`, `SENTINEL-test-report.md`) were produced under a **generic, Sites/Headless-oriented specialist roster** (`strategist`, `designforge`, `blockwright`, `bridgesmith`, `auditron`, `sentinel`) shared with a companion AEMaaCS ADLC spec for Sites work. That roster had no domain knowledge of Adaptive Forms, FDM, submit actions, or prefill services. This document describes the **current, Forms-specific 11-agent roster** — `aem-forms-program-agent` plus 10 specialists — that `AGENTS.md` now mandates for every Forms delivery. Historical run folders are preserved as-is for forensic reading of those runs only; they are not dispatched under the current roster.

---

## 1. Executive Summary

The Agentified Delivery Life Cycle (ADLC) is the multi-agent analogue of a traditional SDLC, specialized for AEM Adaptive Forms. Work flows through a directed graph of specialized agents under the supervision of one orchestrator. Each agent owns a stage, draws on one or more skills, produces a structured artifact, and surfaces validation gates before downstream agents pick up its output.

The model is built around the same two invariants as the companion Sites ADLC:

- **Skills are the unit of capability, not agents.** Skills (the files under `.claude/skills/`) encode AEM Forms conventions, Core Components resource types, FDM bindings, and guardrails. Agents are thin coordinators that load the right skill at the right moment and produce artifacts that conform to that skill's output contract.
- **The Program Agent owns no domain knowledge directly.** It owns *which specialist runs next*, *which artifact must exist before that specialist runs*, and *what counts as "done"* for the current stage. Domain knowledge (schema-vs-FDM, submit-action shape, workflow steps, theme tokens) lives in the specialists.

The 10 specialists map to 8 lifecycle stages (see `AGENTS.md` → "Skill usage is mandatory" for the phrase-to-skill mapping this spec sits above):

| Stage | Specialist(s) |
|---|---|
| **Bootstrap** | `ensure-forms-agents-md` |
| **Plan** | `planwright` |
| **Design** | `draftsmith` |
| **Implement — build** | `formwright` (delegates to `migrate-form` for brownfield deliveries) |
| **Implement — integrate** | `groundsmith` |
| **Assembly** | `assembler` |
| **Deploy (pre-release)** | `forgemaster` |
| **Release (SCM)** | `pilot` (raises the PR, then the flow **halts**) |
| **⏸ Manual gate (human)** | merge the PR → run the Cloud Manager **DEV**-region pipeline → prompt Sentinel |
| **Test (post-deploy)** | `sentinel` (**LAST** — runs against **cloud DEV**, never localhost, only on an explicit human prompt) |

> **Ordering note (current flow).** Exactly like the companion Sites ADLC, the two Test-adjacent stages are split across a pause: `forgemaster` builds + deploys to the **local AEM SDK** and is the pre-release quality gate, then `pilot` raises the PR and the flow **suspends** for a human to merge + run the Cloud Manager DEV pipeline, then `sentinel` runs **last**, against that **real cloud DEV environment**. Pilot does not merge, deploy, or trigger Cloud Manager, and Sentinel never self-starts. See §5, §8.3.

---

## 2. Skill Inventory (this repository)

### Forms skills (`.claude/skills/`)

| Skill | Scope | Primary owner specialist |
|---|---|---|
| `ensure-forms-agents-md` | Bootstrap `AGENTS.md` + `CLAUDE.md` + `.aem-forms-config.yaml`. Idempotent — never overwrites. | `ensure-forms-agents-md` |
| `discover-form-requirements` | Requirements discovery from a brief/PRD/legacy form/webpage URL/Figma URL → structured requirements + user stories with acceptance criteria. | `planwright` |
| `architect-form-solution` | Solution architecture: schema-vs-FDM, template/theme reuse-vs-build, submit/prefill/workflow approach, ADLC execution plan. | `planwright` |
| `design-form-components` | Technical design: Component Inventory & Specs, Design Specifications (theme tokens), Authoring Guideline. | `draftsmith` |
| `design-form-tests` | Test design: Test Cases traced to user stories + acceptance criteria. | `draftsmith` |
| `generate-schema` | JSON Schema / XSD data schema with FDM-ready bindings. | `formwright` |
| `create-fdm` | Data-source cloud config (Swagger/REST) + Form Data Model that consumes it. | `formwright` |
| `create-editable-template` | Editable template (`af-page-v2`) + content policies for Adaptive Forms. | `formwright` |
| `create-form-component` | Custom Adaptive Forms field component (Dialog, HTL, Sling Model, Java class, JUnit). | `formwright` |
| `create-AdaptiveFormFragment` | Reusable Adaptive Form Fragment (fragment page, DAM asset, conf, filters). | `formwright` |
| `create-adaptive-form` | The form itself: fields, layout, panels, validation clientlib. | `formwright` |
| `create-form-rules` | Rule Editor rules (show/hide, validate, calculate, set-value, cascade). | `formwright` |
| `create-form-clientlib` | Shared/per-form client library (custom functions, client-side validation, styling). | `formwright` |
| `create-form-theme` | Theme (`/apps` theme.zip + DAM theme-json). | `formwright` |
| `migrate-form` | Legacy Foundation → Core Components migration, or LiveCycle/JEE re-platforming. | `formwright` (build step) / standalone `migrate-form` agent |
| `create-prefill-service` | DataProvider SPI prefill service (Excel/CSV-driven CRX/DAM JSON, REST, or JCR draft). | `groundsmith` |
| `create-submit-action` | Custom `FormSubmitActionService` + JCR submit-action node (REST/email/workflow/DoR PDF). | `groundsmith` |
| `create-workflow` | AEM Forms workflow model + variables + steps + launcher + "Invoke an AEM Workflow" wiring. | `groundsmith` |
| `create-form-tests` | JUnit 5 (AEM Mocks + Mockito) unit tests for prefill/submit/Sling Model classes. | `groundsmith` (authoring, during integration) / `sentinel` (execution, post-deploy) |
| `composer` | Embed the built form into the "Test Adaptive Form" Sites page via one AEM Form Container. | `assembler` |
| `test-form-ui` | UI parity: Cypress capture (`ui.tests/test-module`) + pixel diff + Claude Vision comparison against a reference. | `sentinel` |

### Platform skills (Claude Code — available, not part of the Forms ADLC dispatch)

| Skill | Scope |
|---|---|
| `init` | Initialize a new `CLAUDE.md` — superseded here by `ensure-forms-agents-md`. |
| `review` | Review a pull request — diff analysis, convention checks. |
| `security-review` | Security review of pending changes on the current branch. |
| `claude-api` | Build/debug Anthropic SDK apps. Not part of the Forms delivery cycle. |

### Project metadata

Specialists resolve these values from `.aem-forms-config.yaml` at session start (see `AGENTS.md` → "Project identity" for the currently resolved values):

- `{project}` — the single project namespace, from the `project:` key — used in `sling:resourceType`, `/conf/`, AND the folder segment under `/content/forms/af/`, `/content/dam/formsanddocuments/`, `/conf/forms/`.
- `{package}` — Java root package, from the `package:` key.
- `{group}` — component group label, from the `group:` key.
- `{aemVersion}` / `{formType}` — must be `cloud` / `coreComponents` for all new work.
- `{fdmEnabled}` / `{fdmRoot}` — whether Form Data Model is configured, and its cloud-config root.
- `{defaultTheme}`, `{submitActionPackage}`, `{prefillServicePackage}`, `{formsContentRoot}`, `{damContentRoot}`, `{schemaContentRoot}`.

All specialists are expected to read `.aem-forms-config.yaml` first and respect those values verbatim. No agent infers project identifiers from the file system.

---

## 3. AEM Forms Program Agent (Primary Orchestrator)

### Purpose

Coordinate the full ADLC for an AEM Adaptive Forms delivery on the workspace's project. The Program Agent does not author forms, schemas, submit actions, or theme CSS directly. It plans stages, dispatches specialists, enforces quality gates, brokers handoffs between specialists, and reports delivery status — including the mandatory pause at the manual gate.

### Responsibilities (summary)

- Lifecycle planning: translate an intake (brief, legacy form, webpage URL, or Figma URL) into a stage-ordered execution plan (§5).
- Specialist dispatch: spawn each lead with the inputs its contract requires (§4), via the `Agent` tool; for a single atomic task, invoke the matching skill directly (Skill tool) without a lead.
- Dependency management: hold the run's artifact graph in memory (the `.claude/agents/runs/{runId}/` tree); block forward motion if an upstream artifact is missing or fails validation.
- Quality-gate enforcement (§8): reject and re-dispatch (≤2×) when a gate fails; after 2 failures, escalate to the human; never silently advance.
- Release + environment promotion (§8.3): dispatch `pilot` to raise the release PR once Forgemaster's local-SDK deploy gate passes, then **suspend the run**; resume into `sentinel` (against **cloud DEV**) only after a human explicitly prompts, having merged the PR and run the Cloud Manager DEV pipeline.
- Human-in-the-loop checkpoints: pause for explicit human confirmation at named gates (the ADLC execution plan itself, before any build phase runs) and — after Pilot raises the PR — the manual gate (merge + Cloud Manager DEV deploy + explicit Sentinel prompt). Never bypass either.

The full Program Agent contract — startup sequence, agent registry, ADLC phase dependency map, execution flowchart, and the complete phase-by-phase quality-gate table — lives at `.claude/agents/aem-forms-program-agent/AGENT.md`. Treat that file as the canonical operating procedure; this section is the reference summary.

### Decision authority

| Decision | Owner |
|---|---|
| Stage order, lead assignment, parallel vs serial within `formwright` | Program Agent (final) |
| Gate pass/fail at any stage boundary | Program Agent (final) |
| Schema vs FDM, template/theme reuse-vs-build | `planwright` proposes; Program Agent accepts |
| Custom component vs Core Component reuse | `draftsmith` flags `source: custom`; `formwright` builds only those |
| Raising the release PR | `pilot` raises it automatically once Forgemaster's gate is PASS; Program Agent then suspends the run |
| Merging the PR, triggering the Cloud Manager DEV pipeline, real deployment | **Out of ADLC scope** — a human Lead does this manually. The ADLC only resumes (into Sentinel) once a human explicitly prompts for it |
| Destructive operations (force push, `git reset --hard`, content-tree deletion) | Program Agent escalates to human — never autonomous |

### Handoff format (Program Agent → human, at session close)

The Program Agent writes `.claude/agents/runs/{runId}/handoff/program-summary.md` twice per delivery — an **interim** version at the manual gate (`TEST: PENDING`) and a **final** version after Sentinel passes on cloud DEV:

```yaml
adlc_run:
  brief: "{brief title}"
  phases:
    - phase: 0
      agent: ensure-forms-agents-md
      status: PASSED
      artifacts: [AGENTS.md, CLAUDE.md, .aem-forms-config.yaml]
    # ... one entry per executed phase (PLAN, DESI, IMPL-build, IMPL-integration, ASSEMBLY, DEPLOY, SCM, TEST)
  skipped_phases: []
  token_ledger: ".claude/agents/runs/{runId}/tokens.json"
  gate_summary: {phase_0: PASS, PLAN: PASS, DESI: PASS, IMPL-B: PASS, IMPL-I: PASS, ASSEMBLY: PASS, DEPLOY: PASS, SCM: PASS, TEST: PENDING}
  scm:
    branch: "{branch}"
    commit: "{sha}"
    pull_request: "{pr url}"
    claude_folder_committed: false   # always false — .claude/ is never committed
  manual_gate:
    merge_pr: PENDING
    cloud_manager_dev_pipeline: PENDING
    sentinel_prompted: PENDING
  deployment_ready: false            # true only after sentinel PASSES on cloud DEV
```

See §10 for the per-agent token ledger this file references.

---

## 4. Specialist Catalog

Each specialist follows the same 9-field schema (Purpose, Responsibilities, Inputs, Outputs, Tools/Skills, Decision Authority, Dependencies, Validation Criteria, Full Contract). Entries below are summaries — the full operating contracts (every non-negotiable rule, the handoff YAML schema, and the token-tracking procedure) live in the per-specialist `AGENT.md` files under `.claude/agents/<name>/`.

### 4.1 Planwright — Plan stage

- **Purpose.** Convert a business objective, brief, legacy form, public webpage URL, or Figma URL into Structured Requirements + user stories, then a Solution Architecture + Integration & NFR Strategy + ADLC execution plan.
- **Responsibilities.** Requirements discovery (`discover-form-requirements`) then solution architecture (`architect-form-solution`) — both invoked directly via the Skill tool, no sub-agents. For a URL or Figma input, fetches/extracts the source form's field inventory **and** style spec so the pipeline can produce an exact-replica form. Decides schema-vs-FDM per form; enforces template reuse-first (never forks a template per form); ensures every user story has ≥1 acceptance criterion covering every field/rule/submit behaviour.
- **Inputs.** The business intake; `.aem-forms-config.yaml`; existing project structure; a webpage/Figma URL when given.
- **Outputs.** `plan/planwright.md`, `plan/user-stories.yaml`, `plan/solution-architecture.yaml`.
- **Tools / skills.** `Read, Write, Edit, Glob, Grep, Bash, PowerShell, Skill, WebFetch` + Figma MCP tools. Skills: `discover-form-requirements`, `architect-form-solution`.
- **Decision authority.** Requirements classification; schema vs FDM; reuse vs build for template/theme; sequencing of downstream leads.
- **Dependencies.** Phase 0 (`ensure-forms-agents-md`) only.
- **Validation criteria.** Every requirement maps to a real catalog skill (no `gaps`); every user story has ≥1 acceptance criterion; no story lacks coverage; user confirms the plan before execution.
- **Full contract.** `.claude/agents/planwright/AGENT.md`.

### 4.2 Draftsmith — Design stage

- **Purpose.** Convert Planwright's plan (plus UX/brand inputs, or a captured URL/Figma style spec) into the design assets Formwright and Sentinel build/test against. **Design-only — never authors code.**
- **Responsibilities.** Technical Design (`design-form-components`) then Test Design (`design-form-tests`). Specs every visual in a reference (separators, images, logos, icons — as authored DAM assets + AF Image components, never CSS backgrounds); maps theme to `--af-*` token overrides (never a new full stylesheet); flags every generic reusable section (address, contact, declaration, signature) as an Adaptive Form Fragment by default; designs a submit-gated-on-validation test case.
- **Inputs.** `plan/user-stories.yaml` + `plan/solution-architecture.yaml`; UX/brand inputs (or the PLAN-captured style spec for a URL/Figma replica).
- **Outputs.** `design/draftsmith.md`, `design/component-design-spec.yaml`, `design/test-cases.yaml`.
- **Tools / skills.** `Read, Write, Edit, Glob, Grep, Bash, PowerShell, Skill, WebFetch` + Figma MCP tools. Skills: `design-form-components`, `design-form-tests`.
- **Decision authority.** Component contract shape; theme token values; template/theme reuse decision; test-case derivation.
- **Dependencies.** `planwright` (required).
- **Validation criteria.** Every component traces to a requirement; every test case traces to a user story + acceptance criterion (`coverage.uncovered_stories` empty); custom components flagged for Phase 5; no code artifacts produced.
- **Full contract.** `.claude/agents/draftsmith/AGENT.md`.

### 4.3 Formwright — Implement stage (build branch)

- **Purpose.** Engineer the form's data foundation and reusable UI artifacts — schema/FDM, editable template, custom components, Adaptive Form Fragments, the form + its rules, theme, and clientlib — maximizing Core Component reuse. Also runs the URL/Figma-replica build path and delegates brownfield deliveries to `migrate-form`.
- **Responsibilities.** Data foundation first (`generate-schema` or `create-fdm`), then (only the phases DESI marked needed) `create-editable-template`, `create-form-component`, `create-AdaptiveFormFragment`, `create-adaptive-form`, `create-form-rules`, `create-form-clientlib` / `create-form-theme`. Every form auto-wires the shared `assign-task-to-admin` workflow submit by default; every generic reusable section becomes a fragment by default; clientlibs split base (shared) vs form-specific; theme is a thin `:root` token override over the base clientlib's standard styling.
- **Inputs.** `design/component-design-spec.yaml`, `design/test-cases.yaml`, `plan/solution-architecture.yaml`, `plan/user-stories.yaml`.
- **Outputs.** Schema/FDM, template, components, fragments, the form itself, rules, clientlib, theme — plus `implementation/formwright.md` (and one `phaseNN-<skill>.md` per delegated skill).
- **Tools / skills.** All tools. Skills: `generate-schema`, `create-fdm`, `create-editable-template`, `create-form-component`, `create-AdaptiveFormFragment`, `create-adaptive-form`, `create-form-rules`, `create-form-clientlib`, `create-form-theme`, `migrate-form`.
- **Decision authority.** Core-Component-reuse-vs-custom (bounded by DESI's `source: custom` flag); template reuse-vs-new (with recorded justification); fragment reuse-vs-new.
- **Dependencies.** `draftsmith` (required).
- **Validation criteria.** Every phase the plan marked needed ran and passed its own skill gate; every build-side user story satisfied (`user_stories_unsatisfied: 0`); zero-defect pre-handoff checklist confirmed (title component, required asterisks, submit gated on validation, multi-column layout, brand colours in the embedded page, date pickers, footer buttons); **author only — never runs `mvn`** (deployment centralized in `forgemaster`).
- **Full contract.** `.claude/agents/formwright/AGENT.md`.

### 4.4 Groundsmith — Implement stage (integration branch)

- **Purpose.** Wire the form to the outside world — prefill (DataProvider SPI), submit action (REST/email/workflow/DoR PDF), and AEM Forms workflow (approval/review/routing/Adobe Sign).
- **Responsibilities.** Always offers the user a prefill choice (Excel/CSV file, or no prefill — no manual value-entry option) before running `create-prefill-service`; runs `create-submit-action` (6) then `create-workflow` (12, if a process is triggered); confirms or wires the shared `assign-task-to-admin` model so every new form ends with a workflow-backed submit; runs `create-form-tests` (13) LAST — one JUnit test per Java class it authored — and confirms `mvn test -pl core` passes before handoff.
- **Inputs.** `plan/solution-architecture.yaml`, `plan/user-stories.yaml`, `design/component-design-spec.yaml`, `implementation/formwright.md`.
- **Outputs.** Prefill service, submit action, workflow model — plus `implementation/groundsmith.md`.
- **Tools / skills.** All tools. Skills: `create-prefill-service`, `create-submit-action`, `create-workflow`, `create-form-tests`.
- **Decision authority.** Prefill source (bounded by the user's Excel/CSV-or-none choice, or the plan's explicit override); submit-action variant (OOTB REST vs custom vs email vs DoR PDF); workflow step shape.
- **Dependencies.** `formwright` (the form + its data foundation must already exist).
- **Validation criteria.** Every integration-facing user story satisfied (`integration_stories_unsatisfied: 0`); submit actions always have BOTH an OSGi service AND a JCR node; one JUnit test file per Java class authored here (`test_files_created == java_classes_authored`); `mvn test -pl core` PASS; **author only — never runs the deploy `mvn`**.
- **Full contract.** `.claude/agents/groundsmith/AGENT.md`.

### 4.5 Assembler — Assembly stage

- **Purpose.** Embed the finished Adaptive Form into the project's fixed AEM Sites showcase page, "Test Adaptive Form", via the AEM Form Container — replacing whatever form was previously embedded there.
- **Responsibilities.** Delegates entirely to the existing `composer` skill (Skill tool, no sub-agent). Ensures exactly one AEM Form Container, authored INLINE (`useiframe="false"`, no `height`), `formRef` bound to the form's DAM guide-asset path, and the host-page `formEmbedThemePath` + `formEmbedClientlibs` props repointed together with `formRef`.
- **Inputs.** `implementation/formwright.md` (the built form's path), `implementation/groundsmith.md` (integration is complete), `.aem-forms-config.yaml`.
- **Outputs.** The "Test Adaptive Form" Sites page (created once, reused thereafter) + `assembly/assembler.md` + `assembly/composer-embed.md`.
- **Tools / skills.** All tools. Skill: `composer`.
- **Decision authority.** None beyond the mechanical embed — this is a narrow, deterministic edit by design.
- **Dependencies.** `groundsmith` (the form must be fully built + wired first).
- **Validation criteria.** `container_count: 1`; the old `formRef` is gone (replaced, not stacked); the page path is covered by a `ui.content` filter; **author only — never runs `mvn`**.
- **Full contract.** `.claude/agents/assembler/AGENT.md`.

### 4.6 Forgemaster — Deploy stage (pre-release quality gate)

- **Purpose.** Perform the single authoritative Maven build + deploy of the whole reactor to the **local AEM SDK** (including the updated "Test Adaptive Form" page), then produce the Code Quality report. This is the deployment gate — nothing proceeds to Pilot until BUILD SUCCESS and a confirmed, verified deploy.
- **Responsibilities.** `mvn clean install -PautoInstallSinglePackage -q` (the ONE build of record — never a per-module profile for the full build); confirms deploy reached AEM (HTTP 200 on the form + the embedded page); deploy-integrity orphan sweep on `mode="update"` filter roots; generates every workflow model's `/var` runtime (`.../jcr:content.generate.json`) — a step that is easy to skip and leaves a model invisible in Tools → Workflow → Models; enumerates every deployment artifact by name + version.
- **Inputs.** `implementation/formwright.md`, `implementation/groundsmith.md`, `assembly/assembler.md`.
- **Outputs.** `deployment/code-quality-report.md`.
- **Tools / skills.** All tools. No skill — raw `mvn` + `curl` against the local AEM instance.
- **Decision authority.** Build-verdict interpretation (never soft-passes a failed build or an unresolved orphan/duplicate node).
- **Dependencies.** `formwright`, `groundsmith`, `assembler` (all three IMPL/ASSEMBLY summaries must exist).
- **Validation criteria.** `BUILD SUCCESS`; deploy confirmed (form + embedded page both HTTP 200); no failed unit tests; instance matches source (no unresolved orphans); every workflow model's `/var` runtime generated; the report names every artifact (content packages + bundles, by name + version) — a report without this manifest does not pass the gate.
- **Full contract.** `.claude/agents/forgemaster/AGENT.md`.

### 4.7 Pilot — Release stage (raise PR → halt)

- **Purpose.** Take Forgemaster's verified, deployed work to source control: commit everything except `.claude/`, push the current feature branch, and raise a Pull Request against `main` — then **halt the pipeline** at the manual gate.
- **Responsibilities.** Stages with a git exclude pathspec (`':(exclude).claude'`) and *proves* `.claude/` is absent from the commit; one commit per delivery; pushes (never `--force`); raises the PR via the GitHub REST API (`gh` is not installed on this machine) with token resolution order `$GH_TOKEN`/`$GITHUB_TOKEN` → Git Credential Manager → ask the human; reuses an existing open PR rather than duplicating one; prints the manual runbook and stops.
- **Inputs.** `deployment/code-quality-report.md` (`gate_result: PASS` is the entry gate), `implementation/formwright.md` + `groundsmith.md`, `assembly/assembler.md`, `plan/user-stories.yaml`.
- **Outputs.** `scm/pilot.md`.
- **Tools / skills.** All tools. No skill — raw `git` + the GitHub REST API via `curl.exe`.
- **Decision authority.** PR title/body content; refusing to commit against a red Forgemaster gate or from `main`/detached HEAD.
- **Decisions escalated / out of scope.** Merging the PR, triggering the Cloud Manager pipeline, starting Sentinel, deploying anywhere real — all three are human actions.
- **Dependencies.** `forgemaster` (its gate must be PASS — the sole precondition).
- **Validation criteria.** A commit exists with **zero** files under `.claude/` (`claude_folder_files_committed: 0`, proven via `git diff --cached --name-only | grep -c '^\.claude/'`); branch pushed and `HEAD == origin/{branch}`; a PR is open with `main` as base (created or reused, never duplicated); `next: MANUAL`.
- **Full contract.** `.claude/agents/pilot/AGENT.md`.

### 4.8 Sentinel — Test stage (post-deploy, NFR + functional enforcement) — **LAST stage**

- **Purpose.** Enforce final acceptance against the **real cloud DEV environment** — unit/integration tests, UI parity, and functional validation of every rule/validation/submit/prefill/workflow, **as the form renders inside the "Test Adaptive Form" page**. **Never self-starts** — begins only on an explicit human prompt after the PR is merged and the Cloud Manager DEV pipeline has deployed.
- **Responsibilities.** Confirms the entry gate (merge landed on `origin/main`; DEV URLs return 200 with the NEW form) before doing anything; generates every workflow model's cloud-DEV `/var` runtime (the CM pipeline deploys only the `/conf` design copy — this is a second, independent generate step from Forgemaster's local one); runs `create-form-tests` (unit/integration, locally against merged source — AEM Mocks need no instance) then `test-form-ui` (UI parity, captured from the cloud-DEV embedded page, never localhost) then drives functional validation directly against the deployed form; closes the traceability loop (every DESI test case executed + every user story covered).
- **Inputs.** `design/test-cases.yaml`, `plan/user-stories.yaml`, `deployment/code-quality-report.md`, `scm/pilot.md`, `implementation/formwright.md` + `groundsmith.md`, `assembly/assembler.md`.
- **Outputs.** `testing/test-report.md` (+ the delegated skills' `integration-test-report.md` / `test-form-ui-report.md`, text-only — no images in `runs/`).
- **Tools / skills.** All tools. Skills: `create-form-tests`, `test-form-ui`; functional/E2E validation is driven directly (no skill).
- **Decision authority.** Whether a UI-parity finding is Critical (blocking) vs cosmetic; pass/fail per user story.
- **Dependencies.** The ⏸ manual gate (PR merged + Cloud Manager DEV deploy + an explicit human prompt) — no agent hands off to Sentinel automatically.
- **Validation criteria.** `unexecuted_cases: 0`; `uncovered_stories: 0`; UI parity ≥90% pixel match AND zero Critical findings; the embedded page shows exactly the new form (one container, no stale/duplicate); every workflow model's cloud-DEV `/var` runtime generated; `started_on: human-prompt` and `environment.target: cloud-dev` recorded (never `localhost`).
- **Full contract.** `.claude/agents/sentinel/AGENT.md`.

### 4.9 Migrate-form — Implement stage (brownfield build, standalone or delegated)

- **Purpose.** Migrate an existing form to AEM as a Cloud Service Core Components via two paths: **Path A** (an AEM Adaptive Form content package/JCR tree — Foundation → Core Components rewrite, migrating every field/panel/rule PLUS all form-related assets) or **Path B** (Adobe LiveCycle / AEM Forms on JEE — an `.lca` + custom-DSC `.jar`, re-platformed onto native AEMaaCS services).
- **Responsibilities.** Preserves ALL functionality (never drops a field/rule/prefill/submit/binding); migrates the WHOLE package, not just the form page (DAM images/logos/fonts, DoR templates, schema/binding files, fragments, icons/SVGs); extracts the theme from source truth (never invents a palette); on Path B, re-platforms each construct onto its cloud-native equivalent (XDP → Core Components AF with the XDP retained as DoR; Workbench orchestration → AEM Workflow; custom DSC → OSGi service) and flags anything with no cloud equivalent rather than dropping it.
- **Inputs.** The legacy source (package/tree/path, or `.lca` + DSC `.jar`); when run standalone, hands off to `aem-forms-program-agent` to orchestrate the full pipeline below rather than migrating + deploying inline.
- **Outputs.** The 4 cloud artifacts (form page, DAM guide asset, conf context, filter entries) + every migrated asset + (Path B) workflows/OSGi services — written into `implementation/` when run as the IMPL-build migrate step.
- **Tools / skills.** All tools. Skill: `migrate-form` (this agent IS effectively that skill's operating persona).
- **Decision authority.** Foundation-attribute → Core-Components-property renames; OOTB-vs-custom re-implementation choice on Path B.
- **Dependencies.** A migration ALWAYS runs as a full ADLC delivery (PLAN → DESI → IMPL-build → IMPL-integration → ASSEMBLY → DEPLOY → TEST → HANDOFF) — never a one-shot standalone run.
- **Validation criteria.** Zero Foundation resource types survive; every packaged asset migrated or explicitly flagged (never silently dropped); no `com.adobe.livecycle.*` / `com.adobe.idp.*` import survives in the deployed bundle (Path B); **author only — deploy is Forgemaster's, UI parity is Sentinel's**.
- **Full contract.** `.claude/agents/migrate-form/AGENT.md`.

### 4.10 Ensure-forms-agents-md — Bootstrap stage

- **Purpose.** Bootstrap the project configuration files every other agent depends on — `AGENTS.md`, `CLAUDE.md`, `.aem-forms-config.yaml` — by reading `pom.xml` and project files. Runs once per project; never overwrites an existing `AGENTS.md`.
- **Responsibilities.** Detects Foundation vs Core Components, FDM configuration, and the Java package; templates the three output files with all project-specific tokens.
- **Inputs.** `pom.xml`, existing project structure.
- **Outputs.** `AGENTS.md`, `CLAUDE.md`, `.aem-forms-config.yaml`.
- **Tools / skills.** All tools. Skill: `ensure-forms-agents-md` (this agent's `AGENT.md` simply defers to that `SKILL.md` in full).
- **Decision authority.** None beyond fact extraction — this is deterministic templating against a fixed output shape.
- **Dependencies.** None — Phase 0, runs before any other agent.
- **Validation criteria.** All three files present; `.aem-forms-config.yaml` has every required key (`configured: true`, `project`, `package`, `aemVersion`, `formType`, `fdmEnabled`, `defaultTheme`).
- **Full contract.** `.claude/agents/ensure-forms-agents-md/AGENT.md`.

---

## 5. Orchestration Model

### 5.1 Stage graph — compact vertical view (current flow)

```
                    ┌──────────────────────────┐
                    │  Program Agent (intake)  │
                    └────────────┬─────────────┘
                                 ▼
   ══════════════════════════ BOOTSTRAP ══════════════════════════════
                    ┌──────────────────────────┐
                    │  ensure-forms-agents-md  │   (skipped if
                    │  (AGENTS.md, CLAUDE.md,  │    .aem-forms-config.yaml
                    │  .aem-forms-config.yaml) │    already exists)
                    └────────────┬─────────────┘
                                 ▼
   ═══════════════════════════ PLAN ══════════════════════════════════
                    ┌──────────────────────────┐
                    │       planwright         │   ◀── human checkpoint
                    │  (requirements + user    │       (confirm the ADLC
                    │   stories + solution     │        execution plan)
                    │   architecture)          │
                    └────────────┬─────────────┘
                                 ▼
   ══════════════════════════ DESIGN ═════════════════════════════════
                    ┌──────────────────────────┐
                    │        draftsmith        │   ◀── gate: every component
                    │  (component/design specs │       traces to a requirement;
                    │   + authoring guideline  │       every test case traces to
                    │   + test cases — NO code)│       a user story + AC
                    └────────────┬─────────────┘
                                 ▼
   ═════════════════════ IMPLEMENT + INTEGRATE ═══════════════════════
                    ┌──────────────────────────┐
                    │        formwright        │   ◀── gate: zero-defect
                    │  (schema/FDM + template +│       pre-handoff checklist;
                    │   component + fragment + │       every build-side user
                    │   form + rules + theme + │       story satisfied
                    │   clientlib; delegates to │
                    │   migrate-form for        │
                    │   brownfield)            │
                    └────────────┬─────────────┘
                                 ▼
                    ┌──────────────────────────┐
                    │       groundsmith         │   ◀── gate: submit action has
                    │  (prefill + submit +     │       BOTH OSGi service AND
                    │   workflow + JUnit tests  │       JCR node; every JUnit
                    │   for every Java class)   │       test authored; mvn test
                    │                          │       PASS
                    └────────────┬─────────────┘
                                 ▼
   ═══════════════════════════ ASSEMBLY ══════════════════════════════
                    ┌──────────────────────────┐
                    │         assembler         │   ◀── gate: exactly ONE
                    │  (embed the form in the  │       Form Container; old
                    │   "Test Adaptive Form"   │       formRef replaced,
                    │   Sites page)            │       never stacked
                    └────────────┬─────────────┘
                                 ▼
   ═════════════════════════ DEPLOY (local) ══════════════════════════
                    ┌──────────────────────────┐
                    │        forgemaster       │   ◀── gate: BUILD SUCCESS +
                    │  mvn clean install       │       confirmed deploy (form +
                    │  -PautoInstallSingle     │       page both 200) + no
                    │  Package (LOCAL SDK)     │       failed unit tests + no
                    │                          │       unresolved orphans
                    └────────────┬─────────────┘
                                 ▼
   ═══════════════════════════ RELEASE ════════════════════════════════
                    ┌──────────────────────────┐
                    │          pilot           │   ◀── auto-runs once
                    │  commit (excl. .claude/) │       Forgemaster is green.
                    │  push feature branch     │       No human approval
                    │  raise PR → main         │       needed to open the PR.
                    └────────────┬─────────────┘
                                 ▼
                          pr opened, awaiting_lead_approval
                                 ▼
   ═══════════ ADLC FLOW PAUSES — awaiting Lead (manual) ═════════════
     Lead (human, OUTSIDE the agent flow): review PR → merge into
     main → run the Cloud Manager pipeline for the DEV region →
     wait for the cloud DEV deployment to complete.
   ═══════════════════════════════════════════════════════════════════
                                 ▼
   ═════ RESUME — explicit human prompt to Sentinel ══════════════════
                    ┌──────────────────────────┐
                    │  Human prompts Sentinel   │   ◀── Program Agent never
                    │  ("DEV deployment is      │       auto-resumes and never
                    │   complete, start         │       fabricates a DEV URL;
                    │   testing")                │       Sentinel confirms the
                    │                          │       merge + DEV 200s itself
                    └────────────┬─────────────┘
                                 ▼
   ══════════════ TEST (post-deploy) — LAST STAGE ═════════════════════
                    ┌──────────────────────────┐
                    │         sentinel          │   ◀── gate: every test case
                    │  (unit/integration tests +│      executed + every user
                    │   UI parity + functional  │      story covered; UI parity
                    │   validation, against     │      ≥90% match + 0 Critical;
                    │   CLOUD DEV, embedded in  │      workflow /var runtime
                    │   the "Test Adaptive Form"│      generated on cloud DEV
                    │   page)                  │
                    └────────────┬─────────────┘
                                 ▼
                         test-report.md (PASS)
   ═════════════════ END OF ADLC AGENT FLOW ══════════════════════════
   The real merge + Cloud Manager DEV pipeline trigger are the human
   Lead's manual, out-of-ADLC step. Stage/Prod promotion and post-deploy
   operations (incident triage, rollback, postmortems) are likewise
   OUTSIDE the ADLC agent set.
```

### 5.1.a Capability coverage — what each specialist actually does today

| Specialist | Currently supported capabilities | Full contract |
|---|---|---|
| `planwright` | Requirements discovery + user stories with acceptance criteria, solution architecture, schema-vs-FDM decision, template/theme reuse-first enumeration, URL/Figma field-inventory + style-spec capture. | §4.1 |
| `draftsmith` | Component/dialog/test specs, `--af-*` token mapping, fragment-by-default flagging for reusable sections, submit-gated-on-validation test case design. **Markdown only.** | §4.2 |
| `formwright` | Schema/FDM, editable templates, custom components, Adaptive Form Fragments, the form + rules + clientlib + theme, brownfield migration (via `migrate-form`), URL/Figma exact-replica build. | §4.3 |
| `groundsmith` | Prefill (DataProvider SPI), submit actions (REST/email/workflow/DoR PDF, always both OSGi service + JCR node), AEM Forms workflows, JUnit 5 tests for every Java class it authors. | §4.4 |
| `assembler` | One narrow, deterministic edit: repoint the "Test Adaptive Form" page's single AEM Form Container at the new form (inline embed, host-page clientlib wiring). | §4.5 |
| `forgemaster` | Single authoritative `mvn` build + deploy to the local SDK, deploy-integrity orphan sweep, workflow `/var` runtime generation, deployment-artifact manifest, Code Quality report. | §4.6 |
| `pilot` | Commit (excl. `.claude/`), push, raise the PR via the GitHub REST API (`gh` not installed), then halt at the manual gate. **Never merges, deploys, or starts Sentinel.** | §4.7 |
| `sentinel` | Unit/integration tests, UI parity (Cypress + pixel diff + Claude Vision), functional validation against **cloud DEV**, traceability-loop closure. **Never self-starts.** | §4.8 |
| `migrate-form` | Foundation→Core-Components migration (Path A) or LiveCycle/JEE re-platforming (Path B), always as a full ADLC delivery. | §4.9 |
| `ensure-forms-agents-md` | Bootstrap templating of `AGENTS.md` / `CLAUDE.md` / `.aem-forms-config.yaml`. Runs once. | §4.10 |

**Not currently supported by any ADLC specialist (gaps):**

- Security/permissions/dispatcher configuration for AEM Forms — `AGENTS.md`'s ADLC phase-dependency map notes this belongs to a future **Configsmith**-equivalent lead; until it exists the Program Agent runs those configs directly.
- Cloud Manager Dev/Stage/Prod pipeline triggering + human-approval enforcement — external process, entirely the human Lead's.
- Post-deploy operations — incident triage, rollback, postmortems, recurring-incident escalation — no ADLC agent owns these.

### 5.2 Parallelism rules

- **`draftsmith` runs as a single serial stage between `planwright` and the implementation fan-out.** `formwright` depends on its design pack as authoritative input.
- **Within `formwright`, several build phases run in parallel** where the ADLC phase-dependency map (`aem-forms-program-agent/AGENT.md`) allows it: Phase 5 (custom component) is parallel with Phase 4 (rules); Phase 8 (theme) is parallel with Phases 4–7; Phase 6 (submit) and Phase 7 (prefill) are parallel with each other (both depend only on Phase 3, the form itself).
- **`groundsmith` runs serially after `formwright`** — its prefill/submit/workflow wiring attaches to the real `guideContainer` and real field names Formwright produced.
- **`assembler` runs serially after `groundsmith`**, so the page it embeds points at a fully-integrated form.
- **`forgemaster` runs serially after `assembler`** — its one authoritative build+deploy must pick up the updated page along with the form.
- **`pilot` runs AFTER `forgemaster` and BEFORE `sentinel`.** Once Forgemaster's gate is PASS, Pilot raises the PR **automatically — no human approval needed to open it** — and the flow suspends.
- **The flow PAUSES after the PR.** The Program Agent suspends the run while a human Lead manually merges, then runs the Cloud Manager DEV pipeline. It resumes only on an explicit human prompt to Sentinel — never auto-resume, never fabricate a DEV URL.
- **`sentinel` runs LAST**, on that prompt, against the **cloud DEV** URL(s) — never the local SDK Forgemaster deployed to. Its report is the terminal acceptance verdict.

### 5.3 Iteration loops

- A failed gate **never** advances. The Program Agent re-dispatches the failing lead with the gate evaluator's notes.
- Re-dispatch is bounded — after **2** failed iterations on the same stage, the Program Agent escalates to the human (tighter than the companion Sites ADLC's 3-iteration cap).
- **A `sentinel` failure does NOT auto-re-dispatch to a local fix-and-retest.** Because remediation re-enters a full new cycle (fix → `forgemaster` rebuild → `pilot` PR → Lead merge + re-deploy → a new prompt to `sentinel`), the defect is routed back to the owning lead (`formwright` for build defects, `groundsmith` for integration, `assembler` for embed defects, `forgemaster` for build/deploy) and the human decides when to re-run the full route. Sentinel never patches the DEV environment directly to make a test pass.
- **`pilot` and `sentinel` are non-deferrable.** Neither may be skipped or marked out of scope because no environment is available — Pilot's only precondition is Forgemaster's local-SDK PASS (a PR needs a git remote, not a deployed cloud instance), and Sentinel is at most *pending* at the manual gate.

---

## 6. Specialist Operating Modes

Every specialist operates in one of two modes.

### 6.1 Independent mode

- **Trigger.** A human invokes a single skill directly, or asks for one atomic task (e.g. "add one show/hide rule to form X").
- **Input source.** The human prompt + any explicit references attached.
- **Output destination.** The repository directly, plus (when run inside a delivery) the matching run-record subfolder.
- **Gates.** Internal skill-level gates fire (e.g. the dialog-spec confirmation inside `create-adaptive-form`). External program-level gates do not fire — there is no Program Agent orchestrating.

### 6.2 Orchestrated mode

- **Trigger.** The Program Agent dispatches the lead (via the `Agent` tool) with a structured input packet drawn from prior-phase run-record files.
- **Input source.** The previous stage's artifacts under `.claude/agents/runs/{runId}/` + the plan/design specs.
- **Output destination.** Repository (deployable artifacts, under `core/`, `ui.apps/`, `ui.content/`, etc.) **and** the structured handoff (each lead's `AGENT.md`-defined YAML, returned to the Program Agent) **and** the run-record file under the matching SDLC-cycle subfolder.
- **Gates.** Internal skill-level gates fire **and** the Program Agent applies the external stage-boundary gate (§8) before advancing.

### 6.3 Mode invariants

- A specialist's authored artifacts are the same shape between modes for the same input — orchestration changes *who reads the handoff*, not *what gets produced*.
- A specialist must never silently skip its skill's gate checks because the Program Agent is calling it (`AGENTS.md` → "Skill usage is mandatory" applies uniformly).
- The Program Agent **never** writes to a lead's output paths directly. Every artifact in a lead's territory is produced by that lead.

---

## 7. Skill ↔ Specialist Mapping Matrix

See `AGENTS.md` → "Agent registry" and "Skill → owning lead" for the authoritative version of this table (kept in one place to avoid drift). Summary:

| Skill | Owning lead | Mode notes |
|---|---|---|
| `discover-form-requirements`, `architect-form-solution` | `planwright` | Both run in-conversation; no sub-agent |
| `design-form-components`, `design-form-tests` | `draftsmith` | Both run in-conversation; no sub-agent |
| `generate-schema`, `create-fdm`, `create-editable-template`, `create-form-component`, `create-AdaptiveFormFragment`, `create-adaptive-form`, `create-form-rules`, `create-form-clientlib`, `create-form-theme` | `formwright` | Ordered per §4.3; skip a phase DESI marked "reuse" |
| `create-prefill-service`, `create-submit-action`, `create-workflow` | `groundsmith` | `create-form-tests` also runs here for unit tests, LAST |
| `composer` | `assembler` | Sole skill; no code decisions |
| — (raw `mvn` + `curl`) | `forgemaster` | No skill — the deployment of record |
| — (raw `git` + GitHub REST API) | `pilot` | No skill — `gh` is not installed on this machine |
| `create-form-tests` (execution), `test-form-ui` | `sentinel` | Functional/E2E validation is driven directly, no skill |
| `migrate-form` | `formwright` (delegated) / standalone `migrate-form` agent | Always inside a full ADLC delivery, never one-shot |
| `ensure-forms-agents-md` | `ensure-forms-agents-md` | Phase 0, runs once |

---

## 8. Validation, Quality Gates, and Promotion

### 8.1 Per-stage gates (summary)

The complete, phase-by-phase gate table (every one of the 14 build/test phases, plus the 8 lifecycle-stage gates) already lives in `.claude/agents/aem-forms-program-agent/AGENT.md` → "Step 4 — Quality gates". This section is the condensed, stage-level view; treat the AGENT.md table as authoritative on any conflict.

| Stage | Gate (must all be true) |
|---|---|
| Bootstrap | `.aem-forms-config.yaml` exists with all required keys |
| PLAN | Structured Requirements + user stories (each ≥1 acceptance criterion) + Solution Architecture + ADLC plan present; every requirement maps to a real skill; user confirmed the plan |
| DESI | Component Inventory + Design Specs + Authoring Guideline + Test Cases present; every component traces to a requirement; every test case traces to a story + AC; no code artifacts produced |
| IMPL — build (`formwright`) | Every needed build phase ran and passed its own skill gate; every build-side user story satisfied; zero-defect pre-handoff checklist confirmed; author-only (no `mvn`) |
| IMPL — integrate (`groundsmith`) | Every needed integration phase ran and passed; submit action has both OSGi service AND JCR node; every JUnit test authored and `mvn test -pl core` PASS; author-only |
| ASSEMBLY (`assembler`) | Exactly one Form Container, old `formRef` replaced (not stacked); page covered by a `ui.content` filter; author-only |
| DEPLOY (`forgemaster`) | `BUILD SUCCESS`; confirmed deploy (form + page both 200); no failed unit tests; no unresolved orphans; every workflow model's local `/var` runtime generated; deployment-artifact manifest present |
| SCM (`pilot`) | A commit on the feature branch with **zero** `.claude/` files; branch pushed; PR open against `main` (created or reused, never duplicated) |
| ⏸ MANUAL (human) | PR merged into `main`; Cloud Manager DEV pipeline succeeded; a human explicitly prompted Sentinel |
| TEST (`sentinel`) | Started on a human prompt against confirmed cloud DEV; every test case executed; every user story covered; UI parity ≥90% match + 0 Critical; workflow models' cloud-DEV `/var` runtime generated |

If a gate fails → re-assign to the same lead with the failure reason. After 2 failures → escalate to the human with the specific error + a recommended fix.

### 8.2 Build/deploy budget (token + deployment-of-record policy)

Unlike the companion Sites ADLC's numeric 2-`mvn`-call budget, this Forms ADLC enforces a **single-build-of-record** rule instead: `AGENTS.md` → "Deployment is centralized in Forgemaster" — every build/integration/assembly skill invoked *inside* a pipeline delivery (`create-adaptive-form`, `create-fdm`, `create-form-clientlib`, `create-form-rules`, `create-workflow`, `migrate-form`, `composer`, …) **authors artifacts only and skips its own `mvn` build/deploy step**, even one marked "MANDATORY" in its own `SKILL.md`. `forgemaster` runs the ONE authoritative `mvn clean install -PautoInstallSinglePackage` after every artifact (including the Assembler page embed) is authored. `groundsmith` is the sole exception: it MUST run `mvn test -pl core` itself (unit tests only, not the full install) before handoff, so a broken Java class is caught before Forgemaster's build.

A skill invoked **standalone / directly** (not via a lead inside a pipeline) still runs its own deploy step as written — the centralization rule applies only inside an orchestrated delivery.

### 8.3 Environment promotion

The ADLC agent flow ends at a **PR** and pauses; the real merge + Cloud Manager DEV deployment is the human Lead's manual, out-of-ADLC step; then **Sentinel validates the real cloud DEV environment last**.

```
Local (mvn -PautoInstallSinglePackage — owned by Forgemaster)
   │  installed to http://localhost:4502 — the pre-release build-validation gate
   ▼
Forgemaster PASS (BUILD SUCCESS + confirmed deploy + no orphans + workflow /var generated)
   ▼
Pilot — commit (excl. .claude/) + push + raise PR (feature branch → main)  ── auto once Forgemaster is green
   │  → status: awaiting_lead_approval
   ▼
═══════════════ ADLC FLOW PAUSES — Lead (human, manual) ════════════
  OUTSIDE the agent flow, the Lead:
    • reviews + merges the PR into main
    • runs the Cloud Manager pipeline for the DEV region
    • waits for the cloud DEV deployment to complete
════════════════════════════════════════════════════════════════════
   │
   ▼
═══════════ RESUME — explicit human prompt to Sentinel ══════════════
  No config file records this approval in this project (unlike the
  companion Sites ADLC's DECISIONS.md row) — the human's prompt text
  itself ("DEV deployment is complete, start testing") is the record,
  and Sentinel independently re-confirms the merge + DEV 200s before
  proceeding. Never auto-resume; never fabricate a URL.
════════════════════════════════════════════════════════════════════
   │
   ▼
Sentinel — unit/integration tests + UI parity + functional  ── LAST STAGE, against cloud DEV
   │  MUST all pass → terminal acceptance verdict
   ▼
═══════════════════ END OF ADLC AGENT FLOW ═════════════════════════

The real merge + Cloud Manager DEV trigger, Stage/Prod promotion, and
post-deploy operations are handled OUTSIDE the ADLC agent set — by the
human Lead / an external pipeline / SRE process.
```

Rollback / re-validation rules within ADLC scope:

- **Sentinel FAIL against cloud DEV:** route findings to the owning lead; remediation re-enters as a **new cycle** (fix → Forgemaster rebuild → Pilot PR → Lead merge + re-deploy → a new prompt to Sentinel). There is no in-place agent rollback of the real environment — that is the Lead's call.
- **Anything on the real cloud DEV environment** (rollback, hotfix-in-place): out of ADLC scope.

---

## 9. Implementation Notes & Bridges

### 9.1 UI test framework — Cypress (not Playwright)

This Forms project's mandated UI-test / visual-comparison framework is **Cypress**, via the existing `ui.tests/test-module` module — the opposite choice from the companion Sites ADLC, which mandates Playwright. `test-form-ui` owns the whole track: it captures the live form (cropped to the embedded form region, never the whole Sites page), runs a pixel diff for a hard pass/fail gate, then a Claude Vision pass that explains what actually differs (missing/extra fields, wrong labels, layout, colour, spacing) with severity. `create-form-tests` separately owns JUnit 5 (AEM Mocks + Mockito) unit tests and Selenium/WebDriver integration tests for rule-editor behaviour — a different tool from the UI-parity Cypress track.

Cypress spec inputs are read via `Cypress.env(...)`, which only picks up `CYPRESS_`-prefixed environment variables (`CYPRESS_FORM_URL`, `CYPRESS_REFERENCE_IMAGE`, `CYPRESS_VIEWPORT`, `CYPRESS_MAX_MISMATCH_PCT`) — a plain `FORM_URL` env var is silently ignored and produces a confusing "Cannot read properties of undefined" failure (see `sentinel/AGENT.md` critical rule 4c).

### 9.2 Skill execution surface

All skills in this repo are invoked via the Claude Code `Skill` tool. The Program Agent and every lead issue `Skill` tool calls with the exact skill name — never a shell command — exactly as in the companion Sites ADLC.

### 9.3 Repository layout for agent state

```
.claude/
├── agents/
│   ├── ADLC-SPEC.md                    # this file
│   ├── aem-forms-program-agent/AGENT.md
│   ├── planwright/AGENT.md
│   ├── draftsmith/AGENT.md
│   ├── formwright/AGENT.md
│   ├── groundsmith/AGENT.md
│   ├── assembler/AGENT.md
│   ├── forgemaster/AGENT.md
│   ├── pilot/AGENT.md
│   ├── sentinel/AGENT.md
│   ├── migrate-form/AGENT.md
│   ├── ensure-forms-agents-md/AGENT.md
│   └── runs/                           # every delivery's run record (never committed — see AGENTS.md)
│       └── {YYYY-MM-DD}-{formName}/
│           ├── plan/                   # planwright.md, user-stories.yaml, solution-architecture.yaml
│           ├── design/                 # draftsmith.md, component-design-spec.yaml, test-cases.yaml
│           ├── implementation/         # formwright.md, groundsmith.md, phaseNN-<skill>.md
│           ├── assembly/               # assembler.md, composer-embed.md
│           ├── deployment/             # code-quality-report.md
│           ├── scm/                    # pilot.md
│           ├── testing/                # test-report.md, integration-test-report.md, test-form-ui-report.md
│           ├── handoff/                # program-summary.md
│           └── tokens.json             # shared per-run token ledger (§10)
├── skills/                             # skill definitions (SKILL.md + references/)
└── settings.json / settings.local.json
```

`{formName}` is the exact kebab-case form node name the run is for (or the brief/program name for a multi-form program). Temporary/internal-processing files (scratch notes, raw tool dumps, draft scripts) go to the session scratchpad directory, **never** into `runs/`.

> **Known documentation gap.** `AGENTS.md`'s "Organize each run by SDLC cycle" table currently lists 7 subfolders (`plan/ design/ implementation/ assembly/ deployment/ testing/ handoff/`) and omits `scm/` — even though `pilot/AGENT.md` and the Program Agent's own startup step both create and write to `scm/pilot.md`. Treat `scm/` as real and required; this spec's layout above reflects actual practice, not the incomplete table.

### 9.4 Independent-mode / cross-CLI discovery

Every specialist under `.claude/agents/<name>/AGENT.md` is available as a Claude Code sub-agent type by virtue of that folder existing. `scripts/sync-agent-routing.mjs` reads this same folder-based roster (via `scripts/.agent-routing.json` for the role-tier-to-model mapping) and generates the equivalent per-agent shims for other CLIs — `.github/agents/<name>.agent.md` for Copilot CLI, `.codex/agents/<name>.toml` for Codex CLI — so the same 10 specialists (plus the Program Agent) are nameable from any of the three tools. Those generated files are pointers back to this roster; they never fork the prompt logic that lives here.

---

## 10. Token & Reporting Conventions

This project does not maintain a dollar-denominated pricing/report system like the companion Sites ADLC's §10. Instead, every specialist's own `AGENT.md` defines an identical, simpler **per-run token ledger** convention:

- **File.** `.claude/agents/runs/{runId}/tokens.json` — one shared file, read-modify-written by every agent that runs in the delivery (never overwritten wholesale — each agent preserves the others' entries).
- **Schema** (one entry per agent, appended to on every pass/fix-pass):

```json
{
  "agents": {
    "<agent-name>": {
      "phase": "PLAN | DESI | IMPL-build | IMPL-integration | ASSEMBLY | DEPLOY | SCM | TEST",
      "passes": [
        { "pass": 0, "label": "initial", "cli_text": 0, "read": 0, "write": 0, "other": 0, "total": 0 }
      ],
      "agent_total": 0
    }
  }
}
```

  - `cli_text` — system/user prompt tokens (role instructions, pasted context).
  - `read` / `write` — tokens consumed reading/writing files via tool calls.
  - `other` — tool-call overhead, shell output, scaffolding noise.
  - `total` per pass = the sum of the four; `agent_total` = the sum across all passes.
- **Never write a secret token here** — GitHub PATs (`pilot`) and AEM Bearer tokens (`sentinel`) are explicitly excluded from this file by their own `AGENT.md` rules; only LLM context-token counts belong in `tokens.json`.
- **Where model tiers are recorded.** Which Claude model backs each agent (sonnet/opus/haiku) is set in that agent's own `AGENT.md` frontmatter — the authoritative source. `scripts/.agent-routing.json` mirrors that as a role-tier-to-model mapping (also covering Copilot CLI and Codex CLI's equivalent models) for `scripts/sync-agent-routing.mjs` to generate the cross-CLI shims from; it does not itself carry $/token pricing.
- **The program-level report** (§3's handoff format) references `token_ledger: ".claude/agents/runs/{runId}/tokens.json"` rather than inlining a breakdown — no agent duplicates the ledger's numbers into its own `.md` report; a one-line `token_usage: see tokens.json` note is sufficient there.

---

## 11. Reference

For per-specialist operational detail — every non-negotiable rule, the full handoff YAML schema, and the exact token-tracking procedure — read the individual `AGENT.md` files under `.claude/agents/<name>/`. For the skill contracts themselves (field-level authoring rules, code templates, troubleshooting tables), see `.claude/skills/<skill>/SKILL.md` and its `references/` subfolder where present.

**End of ADLC-SPEC.md.**
