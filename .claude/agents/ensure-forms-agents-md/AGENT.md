---
name: ensure-forms-agents-md
# Sonnet: reads pom.xml / project files and templates out AGENTS.md, CLAUDE.md and
# .aem-forms-config.yaml. Fact extraction + templating against a fixed output shape.
# Runs once at bootstrap; every value it writes is directly checkable in the repo.
model: sonnet
effort: low
description: >
  Bootstraps an AEM Adaptive Forms project by reading pom.xml and project files,
  then generating AGENTS.md, CLAUDE.md, and .aem-forms-config.yaml with all
  project-specific tokens. Use this agent when setting up AEM Forms agent skills
  for the first time, initialising the AI assistant for an AEM Forms project,
  or when .aem-forms-config.yaml is missing. Invoked by aem-forms-program-agent
  as Phase 0 before any other agent runs.
---

# Agent: ensure-forms-agents-md

## What this agent does
Bootstraps the project configuration files that every other agent depends on.
Runs ONCE per project. Does NOT overwrite an existing `AGENTS.md`.

## How to execute
Read and follow `.claude/skills/ensure-forms-agents-md/SKILL.md` exactly.
That file contains all detection logic, file templates, and the quality checklist.

## Handoff YAML
When complete, return:
```yaml
agent: ensure-forms-agents-md
phase: 0
status: PASSED
artifacts: [AGENTS.md, CLAUDE.md, .aem-forms-config.yaml]
detected_tokens:
  project: "{project}"
  damContentRoot: "/content/dam/formsanddocuments/{project}"
  package: "{package}"
  aemVersion: cloud
  formType: "{coreComponents | foundation}"
  fdmEnabled: {true | false}
  defaultTheme: "{path}"
gate_result: PASS
```
