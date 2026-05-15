# Architecture

## High-level flow

```
[CMDB change]  →  [Async Business Rule]
                          ↓
              [CmdbQuestEventDetector.detect()]
                          ↓
              [Walk table hierarchy, find matching event types]
                          ↓
              [Run event-specific _isXxx() handler]
                          ↓
              [Daily cap check via GlideAggregate]
                          ↓
              [Insert event record with evidence link]
                          ↓
              [Future: badge threshold check → award record]
```

Total typical latency: ~3 seconds from CMDB change to event record.

## Tables

| Table | Purpose | Notes |
|---|---|---|
| `x_lumis_cmdb_quest_event_type` | Admin-configurable scoring rules | Extends `sys_metadata` — ships in update sets when customized |
| `x_lumis_cmdb_quest_badge` | Badge definitions with thresholds | Extends `sys_metadata` |
| `x_lumis_cmdb_quest_event` | Audit trail of every point-earning action | Base table; high write volume |
| `x_lumis_cmdb_quest_award` | User-to-badge associations when threshold crossed | Base table |
| `x_lumis_cmdb_quest_leaderboard` | Denormalized rank rollups | Base table; refreshed by scheduled job (future) |

All tables include a `domain_path` field for MSP readiness.

## Business Rules

Three async BRs on platform tables:

| BR | Collection | When | Triggers on |
|---|---|---|---|
| CMDB Quest: Detect on cmdb_ci | `cmdb_ci` | async | Update |
| CMDB Quest: Detect on cmdb_rel_ci | `cmdb_rel_ci` | async | Insert |
| CMDB Quest: Detect on task | `task` | async | Update |

The `task` BR fires for incident, sc_task, problem, change, etc. (everything that extends `task`).

Each BR is a one-liner that hands off to the Script Include:

```javascript
(function executeRule(current, previous) {
    try {
        var det = new x_lumis_cmdb_quest.CmdbQuestEventDetector();
        det.detect(current, previous, current.operation());
    } catch (e) {
        gs.warn('[CMDB Quest] Detection error: ' + e.message);
    }
})(current, previous);
```

## Script Include: `CmdbQuestEventDetector`

Public access (callable from outside scope). API name: `x_lumis_cmdb_quest.CmdbQuestEventDetector`.

### Main entry point

```javascript
detect(current, previous, action)
```

1. Identifies the table the change occurred on
2. Walks the table's class hierarchy to find applicable event types (so a BR on `task` fires for incident updates and the detector matches via ancestry)
3. For each active event type whose `table_target` matches the hierarchy:
   - Calls the matching `_isXxx()` handler
   - If it matches AND the user is under their daily cap, calls `_awardEvent()`

### Event handlers (one per event type)

| Handler | Detects |
|---|---|
| `_isReclassToChild(current, previous)` | `sys_class_name` changed to a descendant class |
| `_isServiceLayerRel(current)` | New `cmdb_rel_ci` where parent is a service-layer CI |
| `_isTaskServiceAlignment(current, previous)` | `task.cmdb_ci` changed from infra to service-layer |
| `_isIncidentResolved(current, previous)` | `incident.state` changed to 6 (Resolved) |
| `_isCatalogTaskCompleted(current, previous)` | `sc_task.state` changed to 3 (Closed Complete) |

### Anti-gaming helpers

| Helper | Purpose |
|---|---|
| `_underDailyCap(userSysId, eventType)` | GlideAggregate count of today's events for this user + type; rejects if cap met |
| `_isSystemAccount(userName)` | Rejects `system`, `guest`, `glide.maint`, `app_creator` |
| `_getUserSysId(userName)` | Resolves `sys_updated_by` to sys_id |
| `_getTableHierarchy(tableName)` | Walks `sys_db_object` ancestors so BRs on parent tables fire for child-table changes |

## Why these architectural choices

| Decision | Why |
|---|---|
| Async BRs | Never block user transactions; detection runs in background |
| Cross-scope BRs (calling our SI from global tables) | Standard ServiceNow pattern — works without granting Restricted Caller Access |
| Class-hierarchy traversal in `_matches()` | A BR on `task` fires for incident updates; the detector matches via ancestry, not exact table name |
| GlideAggregate for daily caps | O(1) regardless of event volume; scales to millions of events |
| Public access on SI | BRs on global tables need to call into our scoped SI |
| `sys_metadata`-extended config tables | Admin customizations ship cleanly via update sets |
| Deterministic logic, no AI dependencies | FedRAMP / IL5 / IRAP / self-hosted compatibility |

## Scaling considerations

| Concern | Mitigation |
|---|---|
| High-volume CMDB tables (millions of CIs) | All queries use indexes (`user`, `event_type`, `awarded_at`); GlideAggregate avoids GlideRecord loops |
| Detection latency | Async = no impact on user transactions; latency is purely scheduler-driven (~3s typical) |
| Leaderboard recomputation | Future: scheduled job rebuilds `_leaderboard` nightly from aggregated events |
| Reversal cascade | When `event.reversed = true`, awards earned via that event get reviewed; future: automated cascade |

## Extending the engine

To add a new event type:

1. Insert a new record into `x_lumis_cmdb_quest_event_type` with `table_target`, `points`, `daily_cap_per_user`, `tier`, `active = true`
2. Open `CmdbQuestEventDetector` Script Include
3. Add a handler function: `_isYourEventName(current, previous)`
4. Register it in `_matches()` by adding a name check

See [EVENT-TYPES.md](EVENT-TYPES.md) for the current event-type roster and the patterns used.
