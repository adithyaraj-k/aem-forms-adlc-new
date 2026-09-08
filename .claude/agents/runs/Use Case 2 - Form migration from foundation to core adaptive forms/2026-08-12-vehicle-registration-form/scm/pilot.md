# Pilot — SCM record

Run: `2026-08-12-vehicle-registration-form`
Entry gate: Forgemaster `gate_result: PASS`, `deploy_confirmed: true` (see
`../deployment/code-quality-report.md`) — verified before touching git.

## Commit

| Field | Value |
|---|---|
| SHA | `e570de76f0436cda6e8fa9d1c7f387762c505a87` |
| Branch | `feature/pilotAgentChanges` |
| Subject | `fix(forms): repoint Test Adaptive Form page to vehicle-registration-form, close D9 cloudDev config gap` |
| Files committed | 9 |
| `.claude/` files committed | **0** (proven below) |

**`.claude/` exclusion proof** — staged with the exclude pathspec, never by editing `.gitignore`:
```
git add -A -- . ':(exclude).claude' ':(exclude).claude/**'
git diff --cached --name-only | grep -c '^\.claude/'
0
```

## Files committed, grouped by module

| Module | Files |
|---|---|
| root (config/docs) | `.aem-forms-config.yaml`, `AGENTS.md` |
| ui.content | `ui.content/src/main/content/jcr_root/content/aem-adaptive-forms-agents/us/en/test-adaptive-form/.content.xml` |
| ui.tests (flagged working artifacts — see below) | `ui.tests/test-module/cypress/results/form-ui/author-login-final.png`, `.../devpub-event-ticket-page.png`, `.../functional-after-valid-submit.png`, `.../functional-empty-submit.png`, `.../standalone/functional-after-valid-submit.png`, `.../standalone/functional-empty-submit.png` |

### Working artifacts flagged (committed, per the "everything except `.claude/`" rule, but not a deliverable)

6 untracked Cypress capture PNGs from the 2026-08-06 DEV UI test run, ≈1.1 MB total:
- `ui.tests/test-module/cypress/results/form-ui/author-login-final.png`
- `ui.tests/test-module/cypress/results/form-ui/devpub-event-ticket-page.png`
- `ui.tests/test-module/cypress/results/form-ui/functional-after-valid-submit.png`
- `ui.tests/test-module/cypress/results/form-ui/functional-empty-submit.png`
- `ui.tests/test-module/cypress/results/form-ui/standalone/functional-after-valid-submit.png`
- `ui.tests/test-module/cypress/results/form-ui/standalone/functional-empty-submit.png`

These are test-run output, not currently covered by `.gitignore`, so `git add -A` (scoped to
exclude `.claude/`) picked them up correctly per the standing instruction. Flagging here so the
team can decide whether to add a `.gitignore` rule for `ui.tests/test-module/cypress/results/`
going forward — not changed unilaterally in this run.

## Dispatcher symlink check (run before pushing, per AGENTS.md)

```
git ls-files -s dispatcher/src/conf.d/enabled_vhosts dispatcher/src/conf.dispatcher.d/enabled_farms | grep -E '\.vhost|\.farm'
120000 0df3bb3cc38ae01132ae353ed951b53fa7e8ccec 0	dispatcher/src/conf.d/enabled_vhosts/default.vhost
120000 a3db94b59bc4f478b9264a3e7c33644760a2b7cc 0	dispatcher/src/conf.dispatcher.d/enabled_farms/default.farm
```
Both entries are mode `120000` (real git symlinks) — **no repair needed**. Safe for the Cloud
Manager "Build Image" phase / `validate-dispatcher`.

## Push

| Field | Value |
|---|---|
| Remote | `origin` (`https://github.com/arjunvenu-cts/AEM-Adaptive-Forms-Agent.git`) |
| Ref | `refs/heads/feature/pilotAgentChanges` |
| Result | `5c490f5..e570de7  feature/pilotAgentChanges -> feature/pilotAgentChanges` |
| HEAD == origin/branch | confirmed: both `e570de76f0436cda6e8fa9d1c7f387762c505a87` |

## Pull Request

| Field | Value |
|---|---|
| Number | **#5** |
| URL | https://github.com/arjunvenu-cts/AEM-Adaptive-Forms-Agent/pull/5 |
| Head → Base | `feature/pilotAgentChanges` → `main` |
| State | open |
| Created | `true` (HTTP 201 — new PR; prior PR on this branch was #4, already merged/closed) |
| Token source | resolved via `git credential fill` (Git Credential Manager) — never echoed, logged, or written to `runs/` |

## Verdict & gate

`gate_result: PASS` — commit created, push confirmed (`HEAD == origin/feature/pilotAgentChanges`),
PR #5 open against `main`, 0 files under `.claude/` committed.

## Manual next steps (human-only — the pipeline is paused here)

1. Review and **merge PR #5** into `main`.
2. In Adobe Cloud Manager, run the pipeline configured for the **DEV region** of this repository
   so the merged code deploys to cloud DEV (`p185256` / `e1945105`).
3. Wait for the Cloud Manager deployment to complete successfully.
4. Then, and only then, explicitly prompt **Sentinel** to start testing — e.g. "sentinel: DEV
   deployment is complete, start testing on cloud DEV". Sentinel will test the form on the cloud
   DEV region — not localhost.

## Handoff YAML

```yaml
agent: pilot
phase: SCM
status: PASSED
entry_gate: { forgemaster_gate: PASS, deploy_confirmed: true }
branch: "feature/pilotAgentChanges"
base_branch: main
commit:
  sha: "e570de76f0436cda6e8fa9d1c7f387762c505a87"
  subject: "fix(forms): repoint Test Adaptive Form page to vehicle-registration-form, close D9 cloudDev config gap"
  files_committed: 9
  claude_folder_files_committed: 0
  working_artifacts_flagged:
    - ui.tests/test-module/cypress/results/form-ui/author-login-final.png
    - ui.tests/test-module/cypress/results/form-ui/devpub-event-ticket-page.png
    - ui.tests/test-module/cypress/results/form-ui/functional-after-valid-submit.png
    - ui.tests/test-module/cypress/results/form-ui/functional-empty-submit.png
    - ui.tests/test-module/cypress/results/form-ui/standalone/functional-after-valid-submit.png
    - ui.tests/test-module/cypress/results/form-ui/standalone/functional-empty-submit.png
push: { remote: origin, ref: "refs/heads/feature/pilotAgentChanges", head_matches_remote: true }
pull_request:
  number: 5
  url: "https://github.com/arjunvenu-cts/AEM-Adaptive-Forms-Agent/pull/5"
  head: "feature/pilotAgentChanges"
  base: main
  state: open
  created: true
report: ".claude/agents/runs/2026-08-12-vehicle-registration-form/scm/pilot.md"
gate_result: PASS
next: MANUAL
manual_steps:
  - "Merge PR #5 into main"
  - "Run the Cloud Manager pipeline for the DEV region of this repository"
  - "Wait for the cloud DEV deployment to complete"
  - "Prompt sentinel explicitly to begin testing on cloud DEV (it will not start on its own)"
```
