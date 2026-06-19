# Smart Archive Cron — Delivery Visibility Fix (2026-06-15)

## Problem Timeline

1. **Initial state**: Agent-mode cron created a visible session per run → user complained about session clutter
2. **Fix attempt 1**: Switched to `no_agent=true` + `deliver=local` → output went to disk files (`cron/output/db5090d04b33/`), user couldn't see results
3. **Fix attempt 2**: Switched to Agent mode + `script` data-collection + self-cleaning → sessions appeared but user explicitly wanted NO sessions in session list
4. **Final state**: `no_agent=true` + `deliver=local` — the user confirmed this is correct

## User's Explicit Requirement

> "插件推送的会话不应该在桌面端的会话区显示应该做隐藏，只在智能归档的定时任务显示"

Translation: cron-pushed sessions should NOT appear in the desktop session area. They should only be visible in the smart-archive cron job display (Cron Jobs panel → run history).

## Final Working Config

```
no_agent: true
script: smart-archive-desktop.sh
deliver: local
schedule: 0 8 * * *
```

Output goes to:
- `C:\Users\lk\AppData\Local\hermes\cron\output\db5090d04b33\<timestamp>.md` (disk)
- Cron Jobs → 🧠 智能归档 → 运行历史 (GUI panel)

## Key Lessons

1. **`no_agent=true` + `deliver=local` is the correct pattern** for scan-notification cron jobs on this user's setup. It creates zero sessions (no clutter) while still producing output in the cron run history.
2. **Agent mode creates sessions** — even with `deliver=local` and `script` data-collection. This is fundamentally incompatible with the user's requirement of no session visibility.
3. **Self-cleaning is still important** in `no_agent` mode to clean up old cron sessions that may accumulate from previous Agent-mode runs or other sources.
4. **The env var approach via `export` in `.sh` wrappers is unreliable** through the cron scheduler. Set script defaults instead and use env vars only as opt-out overrides.

## Verification Commands

```bash
# Check cron output files
ls -t "C:/Users/lk/AppData/Local/hermes/cron/output/db5090d04b33/" | head -5

# Verify no cron sessions in session list
python3 -c "import sqlite3; conn=sqlite3.connect('C:/Users/lk/AppData/Local/hermes/state.db'); print(conn.execute(\"SELECT COUNT(*) FROM sessions WHERE id LIKE 'cron_%'\").fetchone()[0]); conn.close()"

# Verify scan output
python3 "/c/Users/lk/AppData/Local/hermes/scripts/smart-archive.py" scan
```
