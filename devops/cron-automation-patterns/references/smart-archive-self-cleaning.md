# Smart Archive: Self-Cleaning + Two-Group Output

> Historical note (2026-06-15): this file contains an early self-cleaning implementation. Current smart-archive policy keeps user ordinary sessions safe: no age-based prune; archive/delete only after JSON + `.meta` + queue staging succeeds. Cleanup of old smart-archive cron report sessions is opt-in via `SMART_ARCHIVE_AUTO_CLEANUP_CRON=1`; verify unnamed-session cleanup behavior from the live script and environment flags before assuming deletion.

Concrete implementation of the cron-automation-patterns skill with self-cleaning cron sessions and two-group output format (闲置 + 活跃).

## Architecture

```
smart-archive.py (Python script)
  ├── cmd_scan():        entry point — cleanup → find_all → 🗑️auto-del unnamed → print
  ├── cleanup_old_sessions():  delete previous cron sessions
  ├── find_all_sessions():     query state.db across all profiles
  ├── cmd_archive(N):          export + save + delete a session
  └── cmd_archive_all():       batch archive all tracked sessions
```

## Key Patterns

### 1. Self-Cleaning (before each scan)

```python
def cleanup_old_sessions():
    """Delete old cron sessions, keep only the latest."""
    dbs = get_all_state_dbs()
    for profile_name, db_path in dbs.items():
        try:
            conn = sqlite3.connect(str(db_path))
            cur = conn.cursor()
            cur.execute("""
                SELECT id FROM sessions 
                WHERE id LIKE 'cron_%'
                  AND (title LIKE '%%' OR title = '' OR title IS NULL)
                ORDER BY started_at DESC
            """)
            rows = cur.fetchall()
            conn.close()
            if len(rows) > 1:
                for sid in [r[0] for r in rows[1:]]:
                    cmd = ["hermes", "sessions", "delete", sid, "--yes"]
                    if profile_name != "default":
                        cmd = ["hermes", "--profile", profile_name, "sessions", "delete", sid, "--yes"]
                    subprocess.run(cmd, capture_output=True, timeout=10)
        except:
            pass
```

**Why title matching**: The cron job ID changes every time the job is recreated (`hermes cron remove` + `cronjob create`). Matching by `'cron_%'` + title pattern works across job recreation.

### 2. Two-Group Output Format (Boxed Left-Right Layout, [1]-Offset)

This user prefers a **single-box layout** with emoji icons (🟡🟢) and `[1]` intentionally offset from `[2]+`:

```
📋 智能归档检查 (HH:MM)

┌──────────────────────────────────────────────────────┐
│  🟡 闲置对话（N） [1] 会话标题                 2.3h     │  ← [1] title+time 5 cols left
│                  [2] 会话标题                 2.3h│  ← [2]+ at right edge
│                  [3] 会话标题                 2.2h│
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│  🟢 活跃对话（N） [1] 会话标题                 30m     │
│                  [2] 会话标题                 12m│
└──────────────────────────────────────────────────────┘

💡 回复编号归档入队并删除原会话，如：1、1 2、2+
   回复 N+ = 从第 N 个到最后全部归档入队并删除原会话
   回复 0 = 全部归档入队并删除原会话
```

**Layout rules:**

| Aspect | Rule |
|--------|------|
| Total width | 54 cols |
| Label zone | LABEL_W = 18 |
| [1] label start | 13 cols (LABEL_W - 5) — title 5 cols left of [2]+ |
| [1] time | 5 cols left of right edge (follows the title shift) |
| [2]+ time | At right edge (col 54) |
| Icons | 🟡 idle / 🟢 active |
| No column separator | Single continuous row inside │ │ |
| Filler | `·` middle dots around time filler (see output-format.md) |

**CJK note:** `len("  🟡 闲置对话（9）")` = 11 in Python but ≈17 display cells. The label overflows its 13-char zone visually — acceptable to keep `[1]` position correct.

The single `make_item()` function handles all rows:

```python
def make_item(idx, s, fill_extra=0):
    idle = f"{s['idle_hours']}h" if s['idle_hours'] >= 1 else f"{int(max(s['idle_hours']*60, 1))}m"
    tag = f"[{s['profile']}]" if s['profile'] not in ("default","") else ""
    idx_str = f"[{idx}] "
    fill_w = (WIDTH - LABEL_W + fill_extra) - len(idx_str) - 1 - len(idle)
    title = (tag + s['title'])[:fill_w].ljust(fill_w)
    return idx_str + title + " " + idle
```

**[1] row** — label occupies 13 cols, item `fill_extra=0` (36 cols). Total 13+36=49, `row().ljust(54)` pads the remaining 5 at the end, putting time at col 49:
```python
line1 = left_label.ljust(LABEL_W - 5) + make_item(1, sessions[0])
```

**[2]+ rows** — 18 blanks + 36-char item = 54:
```python
lines.append(row(" " * LABEL_W + make_item(idx, s)))
```

**Time alignment:** The 5 trailing spaces on [1] (from `row().ljust(54)`) mean the time ends at col 49 — 5 cols left of the [2]+ time at col 54. This is intentional: the [1] title is shifted left by 5, and the time follows. The time-to-title gap on [1] is the same as on [2]+.

### 3. find_all_sessions() — No SQL Cutoff

Modern query returns ALL sessions (no WHERE cutoff in SQL), then separates into idle (>1h) and active groups in Python:

```python
idle_sessions = [s for s in all_sessions if s['idle_hours'] >= 1]
active_sessions = [s for s in all_sessions if s['idle_hours'] < 1]
```

### 4. Unnamed Session Auto-Delete (during scan)

Before displaying results, `cmd_scan()` automatically deletes idle unnamed sessions:

```python
for s in idle_sessions:
    if s['title'] == '未命名':
        ok, _ = delete_session(s['id'], s.get('profile', 'default'))
        if ok:
            unnamed_deleted += 1
    else:
        remaining_idle.append(s)
idle_sessions = remaining_idle  # unnamed removed from display + tracker
```

**Key behavior:**
- Only affects **idle** unnamed sessions (active ones kept as-is)
- Deletes directly — no archive, no Obsidian note
- Removed from display output and tracker cache
- Shows a `🗑️ 自动清理了 N 个未命名会话` line in the output
- Does NOT affect the `cmd_archive()` path (which still has the legacy fallback check for unnames)

### 5. Cron Config (Agent Mode)

```python
cronjob(action='create',
  name='🧠 智能归档（跨 profile）',
  schedule='every 3h',
  enabled_toolsets=['terminal'],
  prompt='''Run the smart archive scan and report results.
    1. Run: python3 /c/Users/<user>/AppData/Local/hermes/scripts/smart-archive.py scan
       (script auto-cleans old sessions, then scans)
    2. Read output and report idle/active session counts.''',
  deliver='origin')  # fails on desktop but creates visible session
```

## Windows Path Quirk

| Env var | Resolves to | Correct for state.db? |
|---------|------------|----------------------|
| `APPDATA` | `C:\Users\<user>\AppData\Roaming` | ❌ Empty dummy DB |
| `LOCALAPPDATA` | `C:\Users\<user>\AppData\Local` | ✅ Real state.db |

Always use `LOCALAPPDATA`:

```python
HERMES_HOME = Path(os.environ.get("LOCALAPPDATA", Path.home() / "AppData" / "Local")) / "hermes"
```

## Version History

- 2026-06-12: Initial self-cleaning + two-group format
- 2026-06-12: Fixed APPDATA → LOCALAPPDATA (was reading empty Roaming DB)
- 2026-06-12: Removed `hermes sessions prune --older-than` (user request)
- 2026-06-12: Added auto-delete of old cron sessions (keep latest only)
- 2026-06-12: Boxed left-right layout with 🟡🟢 emoji icons (user preference)
- 2026-06-12: Added unnamed session auto-delete during scan (before display, no archive)
- 2026-06-12: Unified make_item() — [1] uses same 36-char item as [2]+, time left-shifted by 5 cols alongside title
