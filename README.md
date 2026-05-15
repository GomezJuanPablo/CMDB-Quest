# CMDB Quest

> Gamification engine that rewards the behaviors that actually strengthen the CMDB.

![CMDB Quest dashboard](screenshots/01-dashboard.png)

**CMDB Quest** is a scoped ServiceNow application that turns CMDB stewardship into a measurable, leaderboard-driven activity. The platform automatically detects qualifying CMDB-strengthening actions and awards points and badges — no self-reporting, no opt-in, no extra work for contributors.

## The problem it solves

Most CMDB programs stall not because of bad tooling but because nobody operational gains anything from doing the work properly. Reclassify a CI to the right subclass — nobody notices. Operationalize an Application Service — nobody notices. Fix the owner field on 50 stale CIs — nobody notices. So the work doesn't happen.

CMDB Health flags what's wrong. **CMDB Quest rewards the people who fix it.**

## What it does

Every time a user makes a CMDB-strengthening change — reclassifying a CI, adding a service-layer relationship, refreshing a stale owner, operationalizing an Application Service — the system detects the change automatically, validates it against active event-type rules, and awards points. Crossing thresholds earns badges. The Top Curators leaderboard tracks everything in near real-time.

Resolving incidents and completing catalog tasks count too, so ITSM contributors stay competitive on the leaderboard.

**Everyone is enrolled by default.** No opt-in, no role to request, no "join the program" friction.

## Quick install

1. Download [`update-set/cmdb-quest-v1.0.0.xml`](update-set/cmdb-quest-v1.0.0.xml)
2. In your ServiceNow instance: **System Update Sets → Retrieved Update Sets → Import Update Set from XML** → choose the file
3. Open the imported update set → **Preview** → resolve any conflicts → **Commit**
4. Switch scope to **CMDB Quest** in the Application Picker
5. Navigate to **CMDB Quest → Dashboard** in the Filter Navigator

For demo data (optional, populates the leaderboard with sample activity), run the script in [`docs/DEMO-DATA.md`](docs/DEMO-DATA.md).

Full installation walkthrough: [`docs/INSTALLATION.md`](docs/INSTALLATION.md).

## How points are earned

| Tier | Action | Points |
|---|---|---|
| 1 | Operationalize an Application Service | 20 |
| 1 | Resolve a duplicate CI via Data Manager | 15 |
| 1 | Reclassify a CI to a more specific subclass | 10 |
| 2 | Add a service-layer relationship in `cmdb_rel_ci` | 5 |
| 2 | Align a task to a service-layer CI instead of infrastructure | 3 |
| 2 | Refresh stale CI owner from inactive to active user | 3 |
| 3 | Resolve an incident | 5 |
| 3 | Complete a CMDB Health remediation | 10 |
| 3 | Complete a catalog task (`sc_task` closed complete) | 3 |
| 3 | Assign owner to a brand new CI within 24h | 2 |

Daily caps apply per event type. Points reverse automatically if the underlying action is rolled back.

Full breakdown including detection logic: [`docs/EVENT-TYPES.md`](docs/EVENT-TYPES.md).

## Badges

| Badge | Threshold |
|---|---|
| 🥉 Apprentice Curator | 10 total points |
| 🥈 Curator | 50 total points |
| 🥇 Senior Curator | 250 total points |
| 🏆 Master Curator | 1,000 total points |
| Relationship Builder | 10 service-layer relationships added |
| Service Aligner | 25 task service-layer alignments |

## Architecture

Async Business Rules on `cmdb_ci`, `cmdb_rel_ci`, and `task` fire on every relevant change → routes to a scoped Script Include (`CmdbQuestEventDetector`) that evaluates the change against active event types → if it qualifies and the user is under their daily cap, an event record is inserted with points, evidence link, and timestamp.

**End-to-end latency: ~3 seconds** from CMDB change to leaderboard update.

Full architecture details: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

## Modernization decisions

CMDB Quest was built in 2026 using modern ServiceNow patterns:

- **Workspace-first UX** — no Service Portal anywhere
- **Scoped application** — `x_lumis_cmdb_quest`, clean install on any instance
- **`domain_path` field on every table** — MSP-ready when downstream customers activate domain separation
- **GlideAggregate over GlideRecord loops** — runtime-safe for million-CI environments
- **Async detection** — never blocks user transactions
- **`sys_metadata`-extended config tables** — event types and badges ship cleanly in update sets when admins customize
- **Deterministic logic** — no AI, no Now Assist, no embeddings. Runs cleanly on FedRAMP, IL5, IRAP, and self-hosted instances.

## Anti-gaming

Gamification fails when the wrong things get rewarded. CMDB Quest's design rules:

- **Points are tied to material data improvements**, not to clicks or form opens
- **Reversal-on-rollback** — if the source record is deleted or rolled back, the event auto-reverses
- **Daily caps per event type per user** — prevents spam
- **Peer review flag** on Tier 1 events — high-value actions can require admin approval before awarding
- **`sys_updated_by` source check** — system accounts and import jobs don't earn points

## Credits

Inspired by [Sarah Slikk's Point Pioneer](https://sarahslikk.github.io/servicenow/servicenow.html) concept, which proved gamification can work in the ServiceNow platform. CMDB Quest reapplies the same core pattern with a different domain target (CMDB strengthening) and a modernized architecture (Workspace, scoped app, async BR → SI).

## License

MIT — see [LICENSE](LICENSE).

## Author

Built by **Juan-Pablo Gomez** as part of a portfolio focused on operational trust and CSDM adoption in ServiceNow CMDBs. Open an issue here with feedback, or connect on LinkedIn to discuss.
