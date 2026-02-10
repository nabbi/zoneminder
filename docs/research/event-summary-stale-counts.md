# Research: Stale Event Counts on Console (Event_Summaries INNER JOIN Bug)

## Problem Statement

A monitor shows "1 event in the last hour" on the console, but clicking that
number opens the events view and finds zero matching events. Other monitors
with recent events work correctly.

---

## Architecture: Why zmstats and zmaudit Both Exist

ZoneMinder splits periodic maintenance across two daemons with distinct
responsibilities. See `event-summary-architecture.drawio` in this directory
for the visual diagram.

### zmstats.pl (Lightweight Stats Collector)

- **Purpose**: Fast, frequent housekeeping and server resource telemetry
- **Loop interval**: `ZM_STATS_UPDATE_INTERVAL` (typically 5 seconds)
- **Startup delay**: 30 seconds (gives other daemons time to start)
- **What it does each tick**:
  1. Collects CPU load, memory, swap stats and writes to `Server_Stats`
  2. Updates the `Servers` table with current resource usage
  3. **Prunes time-windowed event tables** (`Events_Hour`, `Events_Day`,
     `Events_Week`, `Events_Month`) by deleting rows older than their window
  4. Prunes old `Monitor_Status` rows (stale > 1 minute)
  5. Prunes old `Server_Stats` rows (> 1 day)
  6. Enforces `ZM_LOG_DATABASE_LIMIT` on the `Logs` table
  7. Purges expired `Sessions`

### zmaudit.pl (Heavy Integrity Auditor)

- **Purpose**: Deep consistency checks between database, filesystem, and caches
- **Loop interval**: `ZM_AUDIT_CHECK_INTERVAL` (typically 900 seconds / 15 min)
- **Modes**: `--continuous` (daemon), `--report` (dry-run), `--interactive`
- **What it does each tick**:
  1. Cross-references `Events` table rows against filesystem event directories
  2. Recovers orphaned events or removes stale DB entries
  3. Validates event frame counts, disk space, and file integrity
  4. Enforces `ZM_LOG_DATABASE_LIMIT` (overlaps with zmstats)
  5. **Recalculates `Event_Summaries`** from the `Events_*` time-window tables
  6. Updates `Storage.DiskSpace` totals

### Why Two Daemons?

The split is a **performance design decision**:

- zmstats runs every ~5 seconds doing cheap deletes and inserts (O(n) where n
  is expired rows, usually small). It keeps the time-window tables trimmed.
- zmaudit runs every ~15 minutes doing expensive full-table scans and
  filesystem walks. It reconciles the cached summary counters.

The problem is in the **handoff** between them: zmstats removes rows from
`Events_Hour`, but zmaudit's recalculation query fails to account for monitors
that now have zero rows in `Events_Hour`.

---

## Root Cause: INNER JOIN Skips Zero-Count Monitors

### The Data Flow

```
Event Created (C++ zmc/zma)
  |
  |---> INSERT INTO Events_Hour (EventId, MonitorId, StartDateTime)
  |---> INSERT INTO Event_Summaries ... ON DUPLICATE KEY UPDATE HourEvents += 1
  |
  v
zmstats.pl (every ~5s)
  |
  |---> DELETE FROM Events_Hour WHERE StartDateTime < NOW() - INTERVAL 1 HOUR
  |     (removes expired rows, but does NOT touch Event_Summaries)
  |
  v
zmaudit.pl (every ~15min)
  |
  |---> UPDATE Event_Summaries INNER JOIN (
  |       SELECT MonitorId, COUNT(*) AS HourEvents FROM Events_Hour GROUP BY MonitorId
  |     ) ... SET Event_Summaries.HourEvents = ...
  |
  |     BUG: INNER JOIN produces NO row for monitors with 0 entries in Events_Hour
  |          so Event_Summaries.HourEvents retains its STALE old value
  |
  v
Console PHP (web/ajax/console.php)
  |
  |---> SELECT ... FROM Monitors LEFT JOIN Event_Summaries
  |     Reads HourEvents from cache --> displays "1"
  |
  v
User clicks "1" --> Filter: E.StartDateTime >= NOW() - INTERVAL 1 HOUR
  |
  |---> Queries Events table directly --> finds 0 matching rows
```

### The Bug in Detail

`scripts/zmaudit.pl.in` lines 902-909:

```sql
UPDATE `Event_Summaries` INNER JOIN (
  SELECT  `MonitorId`, COUNT(*) AS `HourEvents`,
          SUM(COALESCE(`DiskSpace`,0)) AS `HourEventDiskSpace`
  FROM `Events_Hour` GROUP BY `MonitorId`
) AS `E` ON `E`.`MonitorId`=`Event_Summaries`.`MonitorId` SET
  `Event_Summaries`.`HourEvents` = `E`.`HourEvents`,
  `Event_Summaries`.`HourEventDiskSpace` = `E`.`HourEventDiskSpace`
```

When `Events_Hour` has zero rows for MonitorId=5:
- The subquery `SELECT ... FROM Events_Hour GROUP BY MonitorId` returns **no
  row** for MonitorId=5
- `INNER JOIN` requires a match on both sides, so MonitorId=5 is **skipped**
- `Event_Summaries.HourEvents` for MonitorId=5 is **never set to 0**

The identical bug exists for `Events_Day` (line 914), `Events_Week` (line 925),
and `Events_Month` (line 936).

### Additional Factor: Event Deletion Never Decrements

When events are deleted (by zmfilter, zmaudit, or the web UI), **nothing
decrements** `Event_Summaries`. The C++ code only increments on event creation:

```cpp
// src/zm_event.cpp:176-184
sql = stringtf("INSERT INTO Event_Summaries "
    "(MonitorId,HourEvents,...) VALUES (%u,1,...) ON DUPLICATE KEY UPDATE"
    " HourEvents = COALESCE(HourEvents,0)+1, ...",
    monitor->Id());
```

This means the cached count can **only go up** between zmaudit recalculations.
If an event is deleted shortly after creation, `Event_Summaries.HourEvents`
stays inflated until zmaudit next recalculates (up to 15 minutes later).

### Timeline Example

| Time    | What Happens                                    | Events_Hour rows (MonId=5) | Event_Summaries.HourEvents |
|---------|-------------------------------------------------|---------------------------|---------------------------|
| T+0:00  | Event created for Monitor 5                     | 1                         | 1                         |
| T+0:05  | zmstats runs, event is fresh, no cleanup needed | 1                         | 1                         |
| T+0:15  | zmaudit runs, INNER JOIN matches, recalculates  | 1                         | 1                         |
| T+1:01  | zmstats runs, deletes expired row               | **0**                     | 1 (stale!)                |
| T+1:15  | zmaudit runs, INNER JOIN finds NO row, **skips**| 0                         | **1 (still stale!)**      |
| T+1:30  | zmaudit runs again, still no row, **skips**     | 0                         | **1 (forever stale!)**    |
| T+2:00  | User sees "1" on console, clicks, finds nothing | 0                         | **1 (bug!)**              |

The count will remain stale **indefinitely** until a new event is created for
that monitor (which resets the cycle).

---

## Fix Options

### Option A: Zero Before INNER JOIN (Simplest - 4 Added Lines)

Add a blanket zero-out before each INNER JOIN recalculation in `zmaudit.pl.in`:

```perl
# Before line 902:
ZoneMinder::Database::zmDbDo(
  'UPDATE Event_Summaries SET HourEvents=0, HourEventDiskSpace=0'
);

# Similarly before lines 914, 925, 936 for Day, Week, Month
```

**Pros**: Trivial change, zero risk of breaking existing logic, easy to review.

**Cons**: Brief window where a concurrent console read could see 0 between the
zero-out and the INNER JOIN recalculation. In practice this is sub-millisecond
and zmaudit runs every 15 minutes, so the race is negligible.

### Option B: Change INNER JOIN to LEFT JOIN (Clean SQL Fix - 4 Lines Changed)

Replace `INNER JOIN` with `LEFT JOIN` and wrap values with `COALESCE`:

```sql
UPDATE `Event_Summaries` LEFT JOIN (
  SELECT `MonitorId`, COUNT(*) AS `HourEvents`,
         SUM(COALESCE(`DiskSpace`,0)) AS `HourEventDiskSpace`
  FROM `Events_Hour` GROUP BY `MonitorId`
) AS `E` ON `E`.`MonitorId`=`Event_Summaries`.`MonitorId` SET
  `Event_Summaries`.`HourEvents` = COALESCE(`E`.`HourEvents`, 0),
  `Event_Summaries`.`HourEventDiskSpace` = COALESCE(`E`.`HourEventDiskSpace`, 0)
```

**Pros**: Single atomic statement per period. No race window. Semantically
correct -- the LEFT JOIN preserves all `Event_Summaries` rows and defaults
unmatched monitors to 0.

**Cons**: Marginally more complex SQL change, but still trivial.

### Option C: Move Recalculation into zmstats (Architectural Refactor)

Since zmstats already prunes `Events_Hour`, it could also recalculate
`Event_Summaries` immediately after pruning. This eliminates the timing gap
entirely.

```perl
# In zmstats.pl.in, after the DELETE FROM Events_Hour block:
zmDbDo('UPDATE Event_Summaries ES LEFT JOIN (
  SELECT MonitorId, COUNT(*) AS HourEvents,
         SUM(COALESCE(DiskSpace,0)) AS HourEventDiskSpace
  FROM Events_Hour GROUP BY MonitorId
) AS E ON E.MonitorId=ES.MonitorId SET
  ES.HourEvents = COALESCE(E.HourEvents, 0),
  ES.HourEventDiskSpace = COALESCE(E.HourEventDiskSpace, 0)');
```

**Pros**: Counts update within seconds of pruning (zmstats interval is ~5s).
Eliminates the up-to-15-minute staleness window completely. Logically groups
the prune and recalculate operations together.

**Cons**: Larger change. Adds load to the frequent zmstats loop (though the
query is cheap since `Events_Hour` is small). Need to decide whether to also
remove the duplicate logic from zmaudit or keep it as a safety net. Also,
zmaudit still needs the Total/Archived recalculation which queries the full
`Events` table, so it can't be fully eliminated.

### Option D: Decrement on Event Deletion (Most Correct, Most Invasive)

Add decrement logic wherever events are deleted: zmfilter.pl, zmaudit.pl,
the web UI delete action, and the API delete endpoint. Also modify
`zm_event.cpp` to decrement when an `Event` destructor runs for a deleted
event.

**Pros**: `Event_Summaries` would always be accurate in real-time, no
periodic recalculation needed for time-windowed counts.

**Cons**: Very invasive. Events can be deleted from 5+ code paths across 3
languages (C++, Perl, PHP). High risk of missing a path and re-introducing
drift. The periodic recalculation in zmaudit would still be needed as a
consistency safety net anyway.

### Recommendation

**Option B (LEFT JOIN)** is the best fix:

- It is a **4-line change** (one word + one COALESCE wrapper per query)
- It is **atomic** -- no race window
- It is **semantically correct** -- LEFT JOIN is what this query should have
  been from the start
- It fixes all four time periods (Hour, Day, Week, Month)
- It requires no architectural changes
- Risk of regression: **near zero**

Option C is worth considering as a follow-up improvement to reduce the
staleness window from ~15 minutes to ~5 seconds, but Option B alone fully
fixes the bug where counts get stuck permanently.

---

## Files Involved

| File | Role | Lines |
|------|------|-------|
| `src/zm_event.cpp` | Increments Event_Summaries on event creation | 154-185 |
| `scripts/zmstats.pl.in` | Prunes Events_Hour (deletes expired rows) | 90-100 |
| `scripts/zmaudit.pl.in` | Recalculates Event_Summaries (has the bug) | 891-945 |
| `web/ajax/console.php` | Reads Event_Summaries for display | 144-147, 359 |
| `web/includes/FilterTerm.php` | Converts `-1 hour` to live SQL timestamp | 135-142 |
| `web/includes/Monitor.php` | Reads Event_Summaries via `__call` magic | 519-530 |
| `db/zm_create.sql.in` | Schema for Events_Hour, Event_Summaries | 229-237, 686-701 |

---

## How to Reproduce

1. Have a monitor that generates infrequent events (e.g., 1 per hour or less)
2. Wait for the event to age past 1 hour
3. Wait for zmstats to prune the Events_Hour entry (~5s cycle)
4. Wait for zmaudit to run its recalculation (~15 min cycle)
5. Observe the console still shows "1" for the Hour column
6. Click it -- events view shows no results

Can also verify directly in the database:

```sql
-- Check the cached count
SELECT MonitorId, HourEvents FROM Event_Summaries WHERE MonitorId = <id>;

-- Check what's actually in Events_Hour
SELECT COUNT(*) FROM Events_Hour WHERE MonitorId = <id>;

-- Check the live query (what the click-through runs)
SELECT COUNT(*) FROM Events WHERE MonitorId = <id>
  AND StartDateTime >= DATE_SUB(NOW(), INTERVAL 1 HOUR);
```

If the first returns a non-zero value but the other two return 0, you've hit
this bug.
