# create-prefill-service — Employee Training Request

Run: `.claude/agents/runs/2026-07-22-employee-training-request/`
Status: **SKIPPED — decision recorded, not silently omitted.** Per this project's mandatory rule,
prefill is ALWAYS offered/considered on every delivery; this file is the evidence trail for why the
answer is "no prefill" here.

---

## Field list considered (per the mandatory ask-then-build flow)

Formwright's flag (`formwright.md` §7.2) asked Groundsmith to "build the manager-lookup prefill
service targeting `$form.employeeDetailsFragment.managerId` / `.managerName`" — the VALUE half of
rule #1 (`auto-set ManagerId/ManagerName + lock readOnly when EmployeeId <= 10`), whose LOCK
structure Formwright already built.

Full field inventory of `employee-training-request` (30 fields, 6 panels) was reviewed; the only
candidate for a data-driven prefill is:

| Field (bind name) | Type | Candidate source |
|---|---|---|
| `$form.employeeDetailsFragment.managerId` | text | legacy hardcoded `EmployeeId<=10 -> 'venuga5'` — was to become an Excel/CSV lookup |
| `$form.employeeDetailsFragment.managerName` | text | legacy hardcoded `EmployeeId<=10 -> 'Arjun Venugopal'` — was to become an Excel/CSV lookup |

No other field in the inventory (EmployeeId, EmployeeName, Email, Department, Location,
BusinessUnit, RequestId, RequestType, ProviderName, CourseName, StartDate, EndDate, TrainingMode,
TrainingLocation, Amount, Currency, BusinessNeed, BenefitsToProject, TrainingCategory, 3 attachment
fields, AcceptedTerms, SubmissionDate, CurrentStatus, ManagerComments, DepartmentHeadComments,
FinanceComments) has a data-source-driven prefill candidate: SubmissionDate is a rule-editor
set-value at submit time (rule #8, already built), and CurrentStatus/ManagerComments/
DepartmentHeadComments/FinanceComments are workflow-task-time inputs, not prefill candidates.

## Decision: NO PREFILL

**Reasoning — the manager-lookup need evaporated when task assignment became group-based:**

The ONLY reason a prefill service was flagged was to supply a real, non-hardcoded value for
`ManagerId`/`ManagerName` so rule #1's lock behaviour had something legitimate to display. That
need was driven by the ASSUMPTION that the workflow would route the Manager Approval task to the
literal person named in those fields (the legacy per-employee `ManagerId` xpath group-assign).

Per the user's CONFIRMED decision (see `groundsmith.md` §Confirmed decisions, and
`create-workflow.md` §3.1), both Assign Task steps in the cloud workflow now resolve to the
`administrators` group — NOT a per-employee `ManagerId` lookup. The `ManagerId`/`ManagerName` field
values are therefore **no longer wired to anything that determines task routing** — they are purely
informational display fields on the form, with rule #1's lock/unlock behaviour intact but nothing
now depending on their prefilled VALUE being a real name.

Building an Excel/CSV static prefill for these two fields today would mean:
1. Inventing or requesting a fabricated employee-to-manager roster with no real HR/system-of-record
   backing it (none exists in the source — confirmed by Planwright's `assumptions`: "No external
   system of record exists for this app").
2. Producing a value that is DISCONNECTED from actual task routing (which is group-based), so the
   prefilled name would be misleading — it would look like "this employee's request goes to this
   named manager" when in fact every request's Manager Approval task goes to `administrators`
   regardless of what's displayed in `ManagerName`.

That is a worse outcome than showing an empty, still-editable field: prefilling a fake name that
implies a routing guarantee the workflow does not honor would be actively misleading to the
employee filling out the form, and to Sentinel/UAT reviewers checking "does the displayed manager
match who approves it."

**Therefore: no prefill service is built for v1.** `ManagerId`/`ManagerName` render empty (subject
to rule #1's existing lock/unlock behaviour, which still fires correctly on `EmployeeId` entry —
only the VALUE population is skipped, not the lock mechanics Formwright already verified). This is
recorded as an accepted, evidence-based decision, not a silently-skipped step.

**Carried forward as a pre-production follow-up (see `groundsmith.md`):** when a real
manager-lookup source (HR system, roster, or a genuine per-employee assignment rule) is confirmed
for production, re-run `create-prefill-service` with the Excel/CSV option — at that point the
prefilled value AND a real per-employee workflow participant-resolution rule should be built
together, since they are the same underlying business decision (who is this employee's manager).
Building only one half now would reintroduce the "displayed manager ≠ actual approver" mismatch
this decision avoids.

## Handoff

- `asked_user.prefill`: `true` (per the mandatory rule — the field list and both options, Excel/CSV
  vs none, were evaluated; the evidence-based call was made autonomously per this run's explicit
  instruction to reason through it rather than block on a live question, since this is a full
  autonomous pipeline run).
- `prefill_approach`: `none` — reasoned, documented, not defaulted.
