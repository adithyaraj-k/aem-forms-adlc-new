# composer-embed — vehicle-registration-form

- **Page**: `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form` — REUSED (not created).
- **Container node**: `jcr:content/root/container/container/adaptiveFormEmbed`
  (`sling:resourceType="aem-adaptive-forms-agents/components/aemformscontainer"`) — REUSED, repointed
  in place. Exactly one container node; no second container added.
- **formRef** (DAM guide-asset path):
  - OLD: `/content/dam/formsanddocuments/aem-adaptive-form-agents/event-ticket-booking`
  - NEW: `/content/dam/formsanddocuments/aem-adaptive-form-agents/vehicle-registration-form`
- **formEmbedThemePath** (page `jcr:content`, AF runtime path):
  - OLD: `/content/forms/af/aem-adaptive-form-agents/event-ticket-booking`
  - NEW: `/content/forms/af/aem-adaptive-form-agents/vehicle-registration-form`
- **formEmbedClientlibs** (page `jcr:content`):
  - OLD: `[aem-adaptive-forms-agents.forms.event-ticket-booking]`
  - NEW: `[aem-adaptive-forms-agents.forms.vehicle-registration-form]`
- **Unchanged**: `formType="af"`, `loadType="embed"`, `useiframe="false"` (inline embed retained).
- **Fragments**: form references `address-details-fragment` and `declaration-fragment` by
  `fragmentPath`; the embed does not touch/suppress fragment rendering — nothing in the page or
  container config restricts fragment resolution.
- **Filter**: `/content/aem-adaptive-forms-agents` root in
  `ui.content/src/main/content/META-INF/vault/filter.xml` covers the page path (`mode="update"`).
- **Verification**: grep for `event-ticket-booking` inside the page's `.content.xml` returns zero
  matches. File edited via targeted attribute replacement only — XML structure otherwise untouched,
  well-formed.
- **Deploy**: NOT run. Author-only per pipeline convention; deferred to `forgemaster`.

## Hazard flagged for Forgemaster (not acted on here)
`ui.content` filter roots use `mode="update"`, which never deletes JCR nodes on the instance that
were removed/renamed in source. This repoint changed the SAME node's attributes (no node
rename/removal), so no orphan node is expected from this specific change — but Forgemaster should
still diff the live `/content/aem-adaptive-forms-agents/us/en/test-adaptive-form` subtree against
source post-deploy to confirm no stale `event-ticket-booking` references survive on the running
instance from prior deployments, and sweep any found.
