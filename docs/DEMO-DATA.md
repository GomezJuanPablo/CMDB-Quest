# Demo Data Loader

Run this script in a Background Script from **CMDB Quest scope** (not Global) to populate your install with sample events, awards, and leaderboard entries — useful for demos and screenshots.

The script:
- Selects up to 8 non-system users from `sys_user`
- Generates 50 events across random users, random event types, over the last 30 days
- Computes per-user totals
- Awards badges based on thresholds
- Populates the leaderboard

Idempotent on awards and leaderboard. Each run adds 50 more events.

## How to run

1. Top-right scope picker → select **CMDB Quest**
2. Filter Navigator → **System Definition → Scripts - Background**
3. Paste the script below
4. Click **Run script**

```javascript
(function() {
    if ((gs.getCurrentScopeName() + '') !== 'x_lumis_cmdb_quest') {
        gs.info('STOP: switch scope picker to CMDB Quest first.'); return;
    }
    var report = { events:0, awards:0, leaderboard:0 };

    var users = [];
    var u = new GlideRecord('sys_user');
    u.addQuery('active', true);
    u.addQuery('user_name','NOT IN','admin,guest,system,glide.maint,app_creator');
    u.setLimit(8); u.query();
    while (u.next()) users.push(u.sys_id + '');
    users.push(gs.getUserID() + '');

    var eventTypes = [];
    var et = new GlideRecord('x_lumis_cmdb_quest_event_type');
    et.query();
    while (et.next()) eventTypes.push({ sys_id: et.sys_id+'', points: parseInt(et.points+'',10) || 1 });
    if (!users.length || !eventTypes.length) { gs.info('No users or event types'); return; }

    for (var i = 0; i < 50; i++) {
        var uid = users[Math.floor(Math.random() * users.length)];
        var ety = eventTypes[Math.floor(Math.random() * eventTypes.length)];
        var ts = new GlideDateTime();
        ts.addDaysLocalTime(-Math.floor(Math.random() * 30));
        ts.addSeconds(-Math.floor(Math.random() * 86400));
        var ev = new GlideRecord('x_lumis_cmdb_quest_event');
        ev.initialize();
        ev.user = uid; ev.event_type = ety.sys_id;
        ev.points_awarded = ety.points; ev.awarded_at = ts;
        ev.evidence_table = 'demo_data'; ev.evidence_sys_id = 'demo_' + i;
        ev.reversed = false;
        if (ev.insert()) report.events++;
    }

    // Per-user totals
    var userTotals = {};
    var ga = new GlideAggregate('x_lumis_cmdb_quest_event');
    ga.addQuery('reversed', false);
    ga.addAggregate('SUM', 'points_awarded');
    ga.addAggregate('COUNT');
    ga.groupBy('user');
    ga.query();
    while (ga.next()) userTotals[ga.user+''] = { points: parseInt(ga.getAggregate('SUM','points_awarded'),10), events: parseInt(ga.getAggregate('COUNT'),10) };

    // Award badges based on thresholds
    var badges = [];
    var b = new GlideRecord('x_lumis_cmdb_quest_badge');
    b.addQuery('active', true); b.query();
    while (b.next()) badges.push({ sys_id: b.sys_id+'', threshold_count: parseInt(b.threshold_count+'',10) || 0, threshold_total_points: parseInt(b.threshold_total_points+'',10) || 0, threshold_event_type: b.threshold_event_type+'' });
    for (var uid2 in userTotals) {
        var totals = userTotals[uid2];
        for (var bi = 0; bi < badges.length; bi++) {
            var bg = badges[bi]; var qual = false;
            if (bg.threshold_total_points > 0 && totals.points >= bg.threshold_total_points) qual = true;
            if (bg.threshold_count > 0 && bg.threshold_event_type) {
                var cga = new GlideAggregate('x_lumis_cmdb_quest_event');
                cga.addQuery('user', uid2);
                cga.addQuery('event_type', bg.threshold_event_type);
                cga.addQuery('reversed', false);
                cga.addAggregate('COUNT'); cga.query(); cga.next();
                if (parseInt(cga.getAggregate('COUNT'),10) >= bg.threshold_count) qual = true;
            }
            if (qual) {
                var ex = new GlideRecord('x_lumis_cmdb_quest_award');
                ex.addQuery('user', uid2); ex.addQuery('badge', bg.sys_id); ex.query();
                if (!ex.hasNext()) {
                    var aw = new GlideRecord('x_lumis_cmdb_quest_award');
                    aw.initialize();
                    aw.user = uid2; aw.badge = bg.sys_id; aw.awarded_at = new GlideDateTime();
                    if (aw.insert()) report.awards++;
                }
            }
        }
    }

    // Refresh leaderboard
    var badgesByUser = {};
    var bga = new GlideAggregate('x_lumis_cmdb_quest_award');
    bga.addAggregate('COUNT'); bga.groupBy('user'); bga.query();
    while (bga.next()) badgesByUser[bga.user+''] = parseInt(bga.getAggregate('COUNT'),10);
    var ranked = [];
    for (var uid3 in userTotals) ranked.push({ user: uid3, points: userTotals[uid3].points });
    ranked.sort(function(a,b) { return b.points - a.points; });
    for (var r = 0; r < ranked.length; r++) {
        var lbEx = new GlideRecord('x_lumis_cmdb_quest_leaderboard');
        lbEx.addQuery('user', ranked[r].user); lbEx.addQuery('period','all_time'); lbEx.query();
        if (lbEx.next()) {
            lbEx.total_points = ranked[r].points;
            lbEx.total_badges = badgesByUser[ranked[r].user] || 0;
            lbEx.rank = r + 1; lbEx.refreshed_at = new GlideDateTime();
            lbEx.update();
        } else {
            var lb = new GlideRecord('x_lumis_cmdb_quest_leaderboard');
            lb.initialize();
            lb.user = ranked[r].user; lb.period = 'all_time';
            lb.total_points = ranked[r].points;
            lb.total_badges = badgesByUser[ranked[r].user] || 0;
            lb.rank = r + 1; lb.refreshed_at = new GlideDateTime();
            if (lb.insert()) report.leaderboard++;
        }
    }

    gs.info('Demo data loaded: ' + JSON.stringify(report));
})();
```

## To clear demo data later

Run from CMDB Quest scope:

```javascript
(function() {
    var ev = new GlideRecord('x_lumis_cmdb_quest_event');
    ev.addQuery('evidence_table', 'demo_data');
    ev.deleteMultiple();
    gs.info('Demo events deleted.');
})();
```

This removes all events marked as `evidence_table = 'demo_data'`. Real events from BR-triggered detection (which have `evidence_table = cmdb_ci`, `cmdb_rel_ci`, `incident`, etc.) are preserved.
