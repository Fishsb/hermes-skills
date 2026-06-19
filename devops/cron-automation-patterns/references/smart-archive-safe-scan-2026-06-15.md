# Smart Archive Safe Scan Repair — 2026-06-15

> Superseded note (2026-06-15): this file records the conservative safe-scan repair. Its wrapper-script and cron `script` guidance remains useful. Any statement that unnamed sessions are hidden/not deleted by default may be superseded by the live `smart-archive.py` and later cleanup policy notes; verify with a non-destructive `scan` and the current environment flags before relying on cleanup behavior.

## Context

A smart-archive cron job was configured as `no_agent=true` with:

```json
"script": "smart-archive.py scan"
```

Hermes cron resolves `script` as a **single relative filename** under `C:/Users/<user>/AppData/Local/hermes/scripts/`; it does not split arguments. Therefore the scheduler looked for a literal file named `smart-archive.py scan` and failed with:

```text
Script not found: C:\Users\lk\AppData\Local\hermes\scripts\smart-archive.py scan
```

## Fix Pattern

Use a wrapper file and set cron `script` to the wrapper filename:

```bash
#!/usr/bin/env bash
set -euo pipefail
python3 "C:\Users\lk\AppData\Local\hermes\scripts\smart-archive.py" scan
```

Cron job:

```text
script = smart-archive-desktop.sh
no_agent = true
schedule = 0 8 * * *
deliver = local
```

Do the update through `cronjob(action='update', ...)`, not by editing `jobs.json` directly.

## Safety Repair

`cmd_scan()` had two destructive behaviors during a scheduled scan:

1. `cleanup_old_sessions()` could call `hermes sessions delete` for old `cron_%` smart-archive sessions.
2. The scan loop could delete all sessions displayed as `未命名` before showing output.

The safer pattern is:

- Scheduled `scan` is read-only by default.
- It may hide noisy unnamed sessions from output and report counts.
- It deletes only when explicit environment switches are set:

```python
SMART_ARCHIVE_AUTO_DELETE_UNNAMED=1
SMART_ARCHIVE_AUTO_CLEANUP_CRON=1
```

Use status lines such as:

```text
🔒 已隐藏 96 个未命名会话（未删除；设置 SMART_ARCHIVE_AUTO_DELETE_UNNAMED=1 才会自动清理）
🔒 检测到 N 个旧智能归档 cron 会话（未删除；设置 SMART_ARCHIVE_AUTO_CLEANUP_CRON=1 才会自动清理）
```

## Verification Checklist

Run only safe checks unless the user explicitly requested deletion:

```bash
python3 -m py_compile C:/Users/<user>/AppData/Local/hermes/scripts/smart-archive.py
bash -n C:/Users/<user>/AppData/Local/hermes/scripts/smart-archive-desktop.sh
python3 C:/Users/<user>/AppData/Local/hermes/scripts/smart-archive.py scan
```

Then compare SQLite counts before and after scan:

```sql
select count(*) from sessions;
select count(*) from sessions where title is null or title='' or title='未命名';
select count(*) from sessions where id like 'cron_%';
```

Expected safe result:

```text
scan_exit=0
unchanged=True
has_safety_line=True
```

## Pitfalls

- Git Bash `/tmp` paths may not be readable by Windows Python as `/tmp/...`; use `C:/Users/<user>/AppData/Local/Temp/...` for cross-tool temp files.
- If a combined Python validation command appears to time out, isolate script execution with shell `timeout` and redirect stdout/stderr to a Windows temp path before diagnosing the script itself.
