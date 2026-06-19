# Smart Archive — Unnamed Cleanup Default (2026-06-15)

## Session learning

The user explicitly corrected the smart-archive cleanup policy: `未命名` / empty-title Hermes sessions should be deleted by default during `smart-archive.py scan`, not merely hidden.

This overrides the earlier conservative default for this specific cleanup class. The scan path should still avoid broad destructive behavior such as age-based pruning or old cron-report deletion unless separately enabled.

## Implementation shape

Use a reversible env flag helper:

```python
def env_flag(name, default=False):
    value = os.environ.get(name)
    if value is None or value == "":
        return default
    value = value.lower()
    if value in {"0", "false", "no", "off"}:
        return False
    return value in {"1", "true", "yes", "on"}

AUTO_DELETE_UNNAMED_ON_SCAN = env_flag("SMART_ARCHIVE_AUTO_DELETE_UNNAMED", True)
AUTO_CLEANUP_CRON_SESSIONS_ON_SCAN = env_flag("SMART_ARCHIVE_AUTO_CLEANUP_CRON", False)
```

Behavior:

- Default `scan`: delete unnamed / empty-title sessions via `hermes sessions delete`.
- Temporary disable: `SMART_ARCHIVE_AUTO_DELETE_UNNAMED=0`.
- Old smart-archive cron report cleanup remains opt-in: `SMART_ARCHIVE_AUTO_CLEANUP_CRON=1`.

## Verification pattern

Before changing cleanup defaults, back up at least:

```text
C:/Users/<user>/AppData/Local/hermes/state.db
C:/Users/<user>/AppData/Local/hermes/scripts/smart-archive.py
```

Then run a direct scan and compare counts:

```python
before = {
  "total": count_all_sessions(),
  "null_or_empty_title": count_null_empty_or_unnamed_titles(),
}
run("python3 smart-archive.py scan")
after = {...}
assert after["null_or_empty_title"] == 0
assert before["total"] - after["total"] == before["null_or_empty_title"]
```

Observed verification in the source session:

```text
BEFORE= {'total': 117, 'null_or_empty_title': 100, 'cron': 1}
AFTER= {'total': 17, 'null_or_empty_title': 0, 'cron': 1}
DELETED_UNNAMED_COUNT=100
TOTAL_DELTA=100
STDERR_LEN=0
🗑️ 自动清理了 100/100 个未命名会话
```

## Pitfall

When patching this block, keep `remaining_idle = []` and the `if s['title'] == '未命名':` guard intact. A small patch that only changes comments/defaults can accidentally corrupt indentation or delete the guard. Always syntax-check and run a real scan against a backed-up database after editing.
