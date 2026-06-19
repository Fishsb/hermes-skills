# SQLite Transaction Isolation — Cron Cleanup Fix

## Problem

When a cron job runs in Agent mode, `cleanup_old_sessions()` fails to delete old sessions because the current cron session isn't visible to the script's SQLite connection.

### Reproduction

1. Agent-mode cron creates session `cron_db5090d04b33_20260615_073334`
2. Scheduler opens a SQLite transaction to insert the row — **not yet committed**
3. Script runs `smart-archive.py scan` → opens its own SQLite connection
4. SQLite snapshot isolation: new connection sees database state from before the uncommitted transaction
5. `SELECT ... WHERE id LIKE 'cron_%' ORDER BY started_at DESC` returns only the OLD session
6. `len(rows) == 1` — the old logic `if len(rows) > 1` never fires
7. Result: old session accumulates each run

### Symptoms

- No `🗑️ 自动清理了 N/M 个旧智能归档 cron 会话` line in scan output
- Multiple `cron_db5090d04b33_*` sessions accumulate in `hermes sessions list`
- Running the same script manually with env var works: `SMART_ARCHIVE_AUTO_CLEANUP_CRON=1 python3 smart-archive.py scan`

### Fix (2026-06-15)

Changed `cleanup_old_sessions()` to branch on `AUTO_CLEANUP_CRON_SESSIONS_ON_SCAN`:

```python
if AUTO_CLEANUP_CRON_SESSIONS_ON_SCAN and len(rows) >= 1:
    # Cron context: current session in uncommitted tx, not visible.
    # All visible matches are old — safe to delete all.
    delete_ids = [r[0] for r in rows]
    total_candidates += len(delete_ids)
    for sid in delete_ids:
        # ... delete via hermes sessions delete ...
elif len(rows) > 1:
    # Manual context: keep latest, delete older ones.
    delete_ids = [r[0] for r in rows[1:]]
    ...
```

### Why this is safe

1. In cron context, the scheduler's new session is in an uncommitted transaction → invisible to the script's connection → won't be deleted
2. When the scheduler commits, only the new session remains
3. In manual context (user runs scan directly), there's no uncommitted transaction → the latest session IS visible → keep it, delete older ones

### Verification

After fix, running scan via cron:
```
📋 智能归档检查 (07:37)
  🗑️ 自动清理了 2/2 个旧智能归档 cron 会话
```
And `hermes sessions list` shows exactly 1 cron session (the latest).

### Also changed

`AUTO_CLEANUP_CRON_SESSIONS_ON_SCAN` default changed from `False` to `True` — cron self-cleaning is now opt-out. Use `SMART_ARCHIVE_AUTO_CLEANUP_CRON=0` to disable.
