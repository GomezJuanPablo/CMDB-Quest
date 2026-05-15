# Changelog

All notable changes to CMDB Quest will be documented in this file.

## [1.0.0] — 2026-05-14

Initial release.

### Added

- 5 scoped tables: `event`, `event_type`, `badge`, `award`, `leaderboard` — all with `domain_path` MSP-readiness
- 10 event types (5 active for v1.0, 5 roadmap-ready placeholders)
- 6 badges (4 progression tiers + 2 specialty)
- `CmdbQuestEventDetector` Script Include with anti-gaming guards
- 3 async Business Rules on `cmdb_ci`, `cmdb_rel_ci`, `task`
- ITSM event types: Incident Resolved, Catalog Task Completed
- CMDB Quest application menu with 10 modules (Dashboard, charts, lists)
- 3 reports (Top Curators bar chart, Event Distribution donut, Activity Trend line)
- UI page dashboard with banner, leaderboard, achievement stats, scoring legend
- 2 roles: `x_lumis_cmdb_quest.admin`, `x_lumis_cmdb_quest.user`
- 20+ ACLs covering all 5 tables
- Optional demo data loader script (in `docs/DEMO-DATA.md`)

### Architecture decisions

- Deterministic logic only — no AI dependencies for FedRAMP / IL5 / IRAP compatibility
- Async BRs for zero impact on user transactions
- `sys_metadata`-extended config tables so admin customizations ship in update sets
- Cross-scope BRs on global tables (`cmdb_ci`, `cmdb_rel_ci`, `task`) calling into our scoped SI

## Roadmap (post-v1.0)

- Activate the 5 roadmap-placeholder event types as v1.1 features (Application Service Operationalized, Duplicate CI Resolved, Stale Owner Refreshed, CMDB Health Remediation Closed, New CI Ownership Assigned)
- Scheduled job for nightly leaderboard recompute
- Flow Designer-driven badge awarding (replace inline insert logic with declarative flow)
- ATF test coverage for each event handler
- HAM / SAM event-type packs as optional add-ons
