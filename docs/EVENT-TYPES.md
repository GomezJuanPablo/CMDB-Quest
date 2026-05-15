# Event Types

CMDB Quest ships with 10 event types: 5 active by default for v1.0, 5 inactive placeholders that signal the roadmap.

## Active in v1.0

| Tier | Name | Table | Points | Daily Cap | Detection |
|---|---|---|---|---|---|
| 1 | CI Reclassified to Child Class | `cmdb_ci` | 10 | 5 | A CI's `sys_class_name` changed to a descendant class (e.g., `cmdb_ci_server` → `cmdb_ci_db_mssql_server`) |
| 2 | Service-Layer Relationship Added | `cmdb_rel_ci` | 5 | 10 | A new `cmdb_rel_ci` where `parent` is a service-layer CI |
| 2 | Task Aligned to Service Layer | `task` | 3 | 20 | `task.cmdb_ci` changed from a non-service-layer CI (or empty) to a service-layer CI |
| 3 | Incident Resolved | `incident` | 5 | 20 | Incident `state` changed to 6 (Resolved) |
| 3 | Catalog Task Completed | `sc_task` | 3 | 20 | `sc_task.state` changed to 3 (Closed Complete) |

## Inactive (roadmap placeholders)

| Tier | Name | Table | Points | Notes |
|---|---|---|---|---|
| 1 | Application Service Operationalized | `cmdb_ci_service_auto` | 20 | Peer review required |
| 1 | Duplicate CI Resolved | `cmdb_ci` | 15 | Peer review required; needs Data Manager activation |
| 2 | Stale Owner Refreshed | `cmdb_ci` | 3 | Owner changed from inactive to active user |
| 3 | CMDB Health Remediation Closed | `cmdb_health_result` | 10 | Requires CMDB Health plugin |
| 3 | New CI Ownership Assigned | `cmdb_ci` | 2 | Owner assigned to a CI within 24h of creation |

To activate these, set `active = true` on the event type record. The detector already has handlers for some of them; the remaining handlers need to be added to the Script Include before activation.

## Service-layer CI classes

The detector's `_isServiceLayerCi()` helper considers the following classes "service-layer":

- `cmdb_ci_service`
- `cmdb_ci_service_auto`
- `cmdb_ci_service_discovered`
- `cmdb_ci_business_app`
- `cmdb_ci_service_offering`

To extend (e.g., if your CSDM model uses additional Service Offering subclasses), edit the array in `_isServiceLayerCi()` in the Script Include.

## Adding custom event types

1. **Filter Navigator → CMDB Quest → Event Types → New**
2. Fill in:
   - **Name** — display label
   - **Description** — what this event represents
   - **Target Table** — the platform table to watch
   - **Trigger Type** — insert / update / both
   - **Points** — value awarded per event
   - **Daily Cap per User** — anti-spam limit
   - **Tier** — 1 (high-value, peer-reviewed) / 2 (medium) / 3 (habit-building)
   - **Peer Review Required** — gate Tier 1 awards behind admin approval (future)
   - **Active** — set to `true` only after the corresponding handler is in place
3. Open `CmdbQuestEventDetector` Script Include
4. Add a handler function matching the event name: `_isYourEventName(current, previous)`
5. Register it in `_matches()`:

```javascript
if (name === 'Your Event Name') return this._isYourEventName(current, previous);
```

6. Add a Business Rule on the target table if one doesn't already exist for that collection

The detector intentionally uses hard-coded handler matching rather than dynamic script evaluation, for security and auditability.

## Why daily caps matter

Without daily caps, users can game the system by repeating the same action — e.g., adding and deleting the same relationship 100 times. Daily caps make repeated identical actions return diminishing rewards. Combined with reversal-on-rollback (events auto-reverse if the source record is deleted), this aligns rewards with sustained data improvement.

Default caps per event type are intentionally generous for v1.0 (most cap at 10–20/day). Tune them downward if you see gaming behavior in your environment.
