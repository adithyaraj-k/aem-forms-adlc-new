# create-workflow — Fix pass 12 (D15) runtime regeneration

**Mode:** FIX/REMEDIATION on the EXISTING model `employee-training-request-approval`. No new
workflow created, no re-architecture, no submit re-wiring. REUSE path — only the two Assign Task
metaData nodes were touched, then the `/var` runtime was regenerated from the edited `/conf` design
model.

## Root cause (verified, not re-investigated)
Opening any `employee-training-request-approval` task in the AEM Inbox failed with the red box
"An error occurred while opening the task" (AEM-FD-008-013). On
`GET /aem/dashboard/formdetails.html` the log showed:
`WorkSpacePayLoadManagerImpl Not able to get input data JSON file  Invalid type for resolving
property RELATIVE_PLOAD`.

Regression from Fix pass 11 (D14), which — as an unnecessary hedge — added the colon-prefixed
combined JSON data-source props alongside the safe flat ones. The task-detail redirector reads the
payload via `PropertyResolver.getColonSeparatedPropertyValue`, which parses
`RELATIVE_PLOAD:data.xml` and throws "Invalid type for resolving property RELATIVE_PLOAD" — the
SAME class/failure Fix pass 6 (D6) already fixed for the XML case by removing `*_COMBINED_DATAXML`
and keeping the flat props.

## The fix (exactly what was done)
On BOTH `assigntask_manager` and `assigntask_finance`:

**Removed properties (per node):**
- `INPUT_COMBINED_DATAJSON` (value `RELATIVE_PLOAD:data.xml`)
- `OUTPUT_COMBINED_DATAJSON` (value `RELATIVE_PLOAD:data.xml`)

**Kept unchanged (per node):**
- `INPUT_DATAJSON = data.xml` (flat — safe)
- `OUTPUT_DATAJSON = data.xml` (flat — safe)
- `icDataSourceType = PROVIDEDATADOCUMENT` (JSON data source; submitted payload is JSON schema
  stored in a file named `data.xml`)

No `INPUT_DATAXML` / `OUTPUT_DATAXML` re-added (that reintroduces the confirmed D14 blank-form
bug where JSON is parsed as XML). Everything else on the nodes left untouched.

### 1. Design model source (committed deliverable)
`ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/employee-training-request-approval/.content.xml`
- Deleted the two combined-JSON attributes from the manager node (was lines 225 & 227) and the
  finance node (was lines 482 & 484).
- Updated the inline documentation notes on both nodes to record this as Fix pass 12 (D15),
  stating the two combined JSON props were removed to fix the RELATIVE_PLOAD crash and must not be
  re-introduced (mirroring how Fix pass 6 removed the DATAXML combined props).

### 2. Live design model + runtime regeneration (server-side, localhost:4502)
The deployed `/conf` design model still carried the combined props, so the same removal was applied
server-side (to keep the live instance consistent with the source edit) before regenerating `/var`:
- `POST .../flow/assigntask_manager/metaData` with `INPUT_COMBINED_DATAJSON@Delete` +
  `OUTPUT_COMBINED_DATAJSON@Delete` (CSRF token + Referer) → HTTP 200
- `POST .../flow/assigntask_finance/metaData` (same) → HTTP 200
- `POST .../employee-training-request-approval/jcr:content.generate.json` (ModelGenerateServlet,
  CSRF + Referer) →
  `{"msg":"Model successfully generated.","modelPath":"/var/workflow/models/employee-training-request-approval"}`

This is NOT an `mvn` build/deploy — it is the targeted design-model edit + `generate.json`
regeneration this model's prior fix passes used. The full reactor build/deploy is deferred to
Forgemaster, which will deploy the committed `.content.xml` (matching the live state).

## Readback evidence

### Deployed `/conf` design model, AFTER the delete (both nodes)
```
assigntask_manager / metaData:  INPUT_DATAJSON=data.xml, OUTPUT_DATAJSON=data.xml, icDataSourceType=PROVIDEDATADOCUMENT   (no *_COMBINED_DATAJSON)
assigntask_finance / metaData:  INPUT_DATAJSON=data.xml, OUTPUT_DATAJSON=data.xml, icDataSourceType=PROVIDEDATADOCUMENT   (no *_COMBINED_DATAJSON)
```

### `/var/workflow/models/employee-training-request-approval` runtime, AFTER generate.json
`lastSynced = 2026-07-28T10:59:04.329+05:30`, `version = 1.29`
```
runtime node id=node2  title=Manager Approval
  icDataSourceType = PROVIDEDATADOCUMENT
  INPUT_DATAJSON = data.xml
  OUTPUT_DATAJSON = data.xml

runtime node id=node4  title=Finance Manager Approval
  icDataSourceType = PROVIDEDATADOCUMENT
  INPUT_DATAJSON = data.xml
  OUTPUT_DATAJSON = data.xml
```
CONFIRMED on BOTH nodes: `INPUT_COMBINED_DATAJSON` and `OUTPUT_COMBINED_DATAJSON` are GONE; flat
`INPUT_DATAJSON`/`OUTPUT_DATAJSON="data.xml"` + `icDataSourceType="PROVIDEDATADOCUMENT"` REMAIN.

## Deploy
`mvn` build/deploy was NOT run — deployment is centralized in Forgemaster (runs after Groundsmith).
