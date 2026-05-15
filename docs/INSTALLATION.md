# Installation Guide

## Prerequisites

- ServiceNow instance (any release supporting scoped apps — Tokyo and later)
- Admin role on the target instance
- Empty `x_lumis_cmdb_quest` scope (no existing app with this scope ID)

## Install steps

### 1. Import the update set XML

1. Filter Navigator → **System Update Sets → Retrieved Update Sets**
2. Click **Import Update Set from XML**
3. Choose `cmdb-quest-v1.0.0.xml` from this repo's `update-set/` directory
4. Click **Upload**

### 2. Preview and commit

1. Find the imported update set in the list (named `CMDB Quest 1.0.0` or similar)
2. Click into the record
3. Click **Preview Update Set**
4. Wait for the preview to complete
5. Resolve any conflicts (typically: accept all if this is a fresh install)
6. Click **Commit Update Set**

### 3. Switch scope

1. Top-right scope picker → select **CMDB Quest**
2. Refresh your browser

### 4. Verify navigation

1. Filter Navigator → type **CMDB Quest** → you should see these modules:
   - Dashboard
   - Top Curators
   - Event Distribution
   - Activity Trend
   - Leaderboard
   - Events
   - Awards
   - Badges
   - Event Types

2. Click **Dashboard** — you should see the CMDB Quest landing page with the banner, leaderboard, and scoring legend

### 5. (Optional) Load demo data

To populate the leaderboard with sample activity for demos, run the script in [DEMO-DATA.md](DEMO-DATA.md) from a Background Script in CMDB Quest scope.

### 6. (Optional) Assign user roles

By default, anyone with a ServiceNow account earns points automatically (the engine doesn't require role assignment to fire).

To allow users to see the dashboard and reports, grant them the `x_lumis_cmdb_quest.user` role.

For admin access (managing event types, badges, ACLs), grant `x_lumis_cmdb_quest.admin`.

## Verifying the engine works end-to-end

The fastest way to prove the engine is live:

1. Open any CI in `cmdb_ci.list`
2. Either:
   - Change its `sys_class_name` to a child class (e.g., `cmdb_ci_server` → `cmdb_ci_db_mssql_server`), or
   - Open `cmdb_rel_ci.list` → New → parent: any service-layer CI, child: any infrastructure CI
3. Wait ~5 seconds (async Business Rule fires)
4. Filter Navigator → **CMDB Quest → Events**
5. Filter the list: **Evidence table** ≠ `demo_data` (to hide any demo data you loaded)
6. You should see a new event record with the appropriate points awarded and the evidence linked back to the CMDB record you just modified

## Troubleshooting

**No new event record appears after a CMDB change**
- Check that the 3 Business Rules are active: Filter Navigator → `sys_script.list` → filter by `name STARTSWITH 'CMDB Quest'`
- Check that the matching event type is active: Filter Navigator → **CMDB Quest → Event Types** → confirm `Active = true`
- Check the system log for `[CMDB Quest] Detection error` warnings

**Event awarded but to the wrong user**
- The detector uses `current.sys_updated_by` to identify who earned the points. If the change was made by a system account or an import job, the detector skips it (intentional anti-gaming behavior).

**Daily cap hit**
- Each event type has a daily cap per user. If you're testing and hit the cap, either increase the cap on the event type record or test with a different user account.
