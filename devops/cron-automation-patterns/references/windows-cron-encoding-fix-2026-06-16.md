# Windows Cron Encoding Fix (2026-06-16)

## Problem

On Chinese Windows (CP936/GBK system encoding), the Hermes Desktop cron scheduler's subprocess pipe reads script stdout as GBK. Box-drawing characters (`┌─┐└┘│`) and emoji (`🟡🟢`) contain byte sequences invalid in GBK, causing:

```
UnicodeDecodeError: 'gbk' codec can't decode byte 0xff in position 47: illegal multibyte sequence
```

This crashes the scheduler's `_readerthread` in `subprocess.py`, causing:
- `Script exited with code 127` in cron run history
- Potential scheduler tick disablement until Desktop app restart

The `hermes cron tick` CLI (Python 3.14+) reads the pipe as UTF-8 instead, producing a complementary error:
```
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xd6 in position 2: invalid continuation byte
```

## Root Cause Chain

1. `smart-archive.py` outputs UTF-8 box-drawing chars via `print(make_boxed_table(...))`
2. Cron scheduler spawns the script via `subprocess.Popen` with `text=True` (uses system encoding)
3. On Chinese Windows, system encoding is GBK/CP936
4. Bytes like `\xe2\x94\x80` (UTF-8 for `─`) contain `\x80` which is invalid in GBK
5. `_readerthread` in subprocess.py calls `fh.read()` with GBK decoder → crash
6. The crash can cascade — subsequent cron ticks may also fail if the scheduler thread is in a broken state

## Fix: `.py` Cron Wrapper with ASCII Replacement

The shell wrapper (`smart-archive-desktop.sh`) was replaced with a Python wrapper (`smart-archive-cron.py`) that:
1. Runs the core script via `subprocess.run(capture_output=True, text=True, errors="replace")`
2. Replaces box-drawing characters with ASCII equivalents before writing to stdout
3. Replaces emoji with text labels

### Final `smart-archive-cron.py`

Location: `C:\Users\lk\AppData\Local\hermes\scripts\smart-archive-cron.py`

```python
#!/usr/bin/env python3
"""
Cron wrapper for smart-archive.py scan — runs via Python directly (no shell dependency).
Replaces box-drawing characters with ASCII-safe equivalents for Windows GBK/UTF-8 pipe compatibility.
"""
import subprocess, os, sys

os.environ["SMART_ARCHIVE_AUTO_CLEANUP_CRON"] = "1"

script = os.path.join(os.environ["LOCALAPPDATA"], "hermes", "scripts", "smart-archive.py")

result = subprocess.run(
    [sys.executable, script, "scan"],
    capture_output=True, text=True, errors="replace",
)

if result.stdout:
    safe = result.stdout
    safe = safe.replace("\u2500", "-")   # ─ → -
    safe = safe.replace("\u2502", "|")   # │ → |
    safe = safe.replace("\u250c", "+")   # ┌ → +
    safe = safe.replace("\u2510", "+")   # ┐ → +
    safe = safe.replace("\u2514", "+")   # └ → +
    safe = safe.replace("\u2518", "+")   # ┘ → +
    safe = safe.replace("\U0001F7E2", "(G)")  # 🟢 → (G)reen
    safe = safe.replace("\U0001F7E1", "(Y)")  # 🟡 → (Y)ellow
    # Wrap in code block for consistent push formatting (no_agent delivers stdout verbatim)
    sys.stdout.write(f"```\n{safe}\n```\n")
else:
    sys.stdout.write(result.stdout or "")

sys.exit(result.returncode)
```

### Cron Config

```json
{
  "id": "db5090d04b33",
  "name": "🧠 智能归档",
  "schedule": "0 8 * * *",
  "script": "smart-archive-cron.py",
  "no_agent": true,
  "deliver": "local",
  "enabled": true
}
```

Note: `script` is a **relative** path under `C:\Users\<user>\AppData\Local\hermes\scripts\`. The cron scheduler runs `.py` files via Python directly (not shell), which avoids the PATH/env issues that affect `.sh` wrappers.

## Verification

After applying the fix:

```bash
# 1. Direct test
bash /c/Users/lk/AppData/Local/hermes/scripts/smart-archive-cron.py
# Output should have +-| instead of ┌─┐└┘│

# 2. Cron trigger
python3 /path/to/hermes cron run db5090d04b33

# 3. Force tick
python3 /path/to/hermes cron tick

# 4. Check result
python3 /path/to/hermes cron list
# Look for "Last run: ... ok" (not "error")

# 5. Verify output file
cat /c/Users/lk/AppData/Local/hermes/cron/output/db5090d04b33/$(ls -t ... | head -1)
# Should show ASCII-safe content, no "script failed"
```

## Additional Notes

- The `⚠ Gateway is not running` warning in `hermes cron list` is a **false positive** on desktop-only Hermes. The Desktop app has its own internal cron scheduler. `hermes gateway status` also reports "running" from a stale PID file — ignore both unless you're troubleshooting API-based cron.
- After repeated UnicodeDecodeError crashes, the Desktop scheduler thread may stop ticking. Restart the Desktop GUI to restore automatic tick behavior.
- When debugging encoding issues, use `hermes cron tick` (CLI) which shows the raw error in stderr, unlike the Desktop GUI which silently fails.
