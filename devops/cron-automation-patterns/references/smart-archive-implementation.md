# Smart Archive — Full Implementation Reference

> Historical note (2026-06-17): this file is an early implementation snapshot. Current smart-archive behavior is the v2 queue pipeline: `scan` reports candidates + Skill Factory suggestions; `archive <N|N+|0>` exports raw JSON to `D:/lk/Obsidian/Hermes/wiki/记忆/_temp_session/`, writes `.meta` + `ingest-queue.json`, deletes the Hermes source session only after staging succeeds, then knowledge extraction + Skill Factory check run inline.

## Real Technology Stack (historical snapshot)

The bullets below describe the early implementation, not the current v2 queue contract:

- **Cron**: Hermes Agent mode (LLM-driven), every 3 hours — historical; current job is `no_agent=true`, schedule `0 8 * * *`.
- **Script language**: Python 3 (direct SQLite via `sqlite3`, no Hermes SDK)
- **Multi-Profile**: Scans all profiles (default + worker/dispatcher/worker2/worker3), child sessions (delegate_task) in parent db
- **Notification**: Agent-mode cron creates an independent session visible in the desktop session list — historical workaround; current Desktop-safe mode uses `deliver=local` with script output in Cron Jobs run history.
- **Interactive**: Agent parses user reply (number selections, ranges, "0=all") — historical; current supported tracker selections are `N`, `N M`, `N+`, and `0`.
- **Archive target**: Obsidian vault (`D:\lk\Obsidian\Hermes\会话归档\`) — historical; current target is `_temp_session/` plus `ingest-queue.json`.

## Complete Script: `smart-archive.py`

Location: `C:\Users\lk\AppData\Local\hermes\scripts\smart-archive.py` (~350 lines)

### ⚠️ Critical Path Fix (2026-06-12)

**APPDATA trap:** The script originally used `os.environ["APPDATA"]` which resolves to `C:\Users\lk\AppData\Roaming` on Windows. But Hermes stores `state.db` in `C:\Users\lk\AppData\Local\hermes\`. The Roaming path had a 0-byte state.db, so the script always scanned an empty database and found zero sessions.

**Fix:** Changed to `os.environ["LOCALAPPDATA"]`:
```python
HERMES_HOME = Path(os.environ.get("LOCALAPPDATA", Path.home() / "AppData" / "Local")) / "hermes"
```

### Key Architectural Facts

1. **delegate_task children live in the PARENT profile's state.db** — NOT in sub-profile dbs. Worker/worker2/worker3 state.dbs typically have 0 sessions. Querying the default profile's `sessions` table already covers ALL sessions including children.
2. **Child sessions have `parent_session_id` set** — use this to detect and label them in output.
3. **`hermes sessions export/delete` works for child sessions** — no special handling needed when passing the session ID.

### SQL: Full Multi-Profile Query Returns Two Groups

Current function `find_all_sessions()` (replaces old `find_idle_sessions()`):

```python
def find_all_sessions():
    """Scan all profiles' sessions, return idle (>1h) and active sessions."""
    dbs = get_all_state_dbs()
    idle_sessions = []
    active_sessions = []
    cutoff = now_ts() - IDLE_HOURS * 3600
    
    for profile_name, db_path in dbs.items():
        conn = sqlite3.connect(str(db_path))
        cur = conn.cursor()
        cur.execute("""
            SELECT s.id, s.title, s.model, s.started_at, s.ended_at, s.message_count,
                   (SELECT MAX(m.timestamp) FROM messages m WHERE m.session_id = s.id) as last_active,
                   s.source, s.parent_session_id,
                   (SELECT s2.title FROM sessions s2 WHERE s2.id = s.parent_session_id) as parent_title,
                   (SELECT content FROM messages m WHERE m.session_id = s.id
                    AND m.role IN ('user','assistant')
                    ORDER BY m.timestamp DESC LIMIT 1) as last_preview
            FROM sessions s
            WHERE (s.ended_at IS NOT NULL 
               OR (SELECT MAX(m.timestamp) FROM messages m WHERE m.session_id = s.id) IS NOT NULL)
              AND (SELECT MAX(m.timestamp) FROM messages m WHERE m.session_id = s.id) IS NOT NULL
            ORDER BY last_active ASC
        """)
        
        rows = cur.fetchall()
        conn.close()
        
        for r in rows:
            last = r[6] or 0
            preview = (r[10] or "")[:60].replace('\n', ' ') if len(r) > 10 and r[10] else ""
            is_child = r[8] is not None
            parent_title = r[9] or "" if len(r) > 9 else ""
            idle_hours = round((now_ts() - last) / 3600, 1) if last else 999
            s = {
                "id": r[0], "profile": profile_name,
                "is_child": is_child, "parent_title": parent_title,
                "title": r[1] or "未命名", "model": r[2] or "unknown",
                "started": datetime.datetime.fromtimestamp(r[3]).strftime("%m-%d %H:%M") if r[3] else "未知",
                "messages": r[5] or 0, "source": r[7] or "cli",
                "preview": preview, "idle_hours": idle_hours,
            }
            if idle_hours >= IDLE_HOURS:
                idle_sessions.append(s)
            else:
                active_sessions.append(s)
    
    # Idle: longest first; Active: most recent first
    idle_sessions.sort(key=lambda x: x['idle_hours'], reverse=True)
    active_sessions.sort(key=lambda x: x['idle_hours'])
    return idle_sessions, active_sessions
```

Key changes from the old version:
- Returns **two groups** (idle, active) instead of one
- SQL no longer filters by cutoff in WHERE clause — ALL sessions are fetched, separation happens in Python
- No tuple parameters `(cutoff, cutoff)` since the WHERE clause doesn't use them
- Idle sessions sorted by idle time descending; active sessions by idle time ascending (most recently active first)
- Function renamed from `find_idle_sessions()` to `find_all_sessions()`

### Scan Output Format (Two-Group Boxed Table)

Current output uses a **boxed left-right layout** with `[1]` intentionally offset from `[2]+`. See `references/smart-archive-output-format.md` for the full format specification and `references/smart-archive-self-cleaning.md` for the implementation code.

Key parameters: `LEFT_W=13`, `RIGHT_W=41`, time-right-aligned per row, `[2]+` rows have 5-space prefix so `[1]` alone is closer to the left border.

The output is generated by `make_boxed_table()` in `cmd_scan()`. The old simple-listing format described in the code below was replaced after ~10 user format iterations.

### Archive + Delete (Profile-Aware)

```python
def cmd_archive(sid, profile="default"):
    print(f"📤 [{profile}] 正在导出会话 {sid} ...")
    cmd = ["hermes", "sessions", "export", "--session-id", sid, "-"]
    if profile != "default":
        cmd = ["hermes", "--profile", profile, "sessions", "export", "--session-id", sid, "-"]
    raw = subprocess.run(cmd, capture_output=True, text=True, timeout=30).stdout.strip()
    if not raw: return False
    md = session_to_markdown(raw)
    if not md: return False
    data = json.loads(raw)
    title = data.get("title", "未命名")
    path = save_to_vault(md, title, sid)
    print(f"✅ [{profile}] 已归档: {path}")
    del_cmd = ["hermes", "sessions", "delete", sid, "--yes"]
    if profile != "default":
        del_cmd = ["hermes", "--profile", profile, "sessions", "delete", sid, "--yes"]
    r = subprocess.run(del_cmd, capture_output=True, text=True, timeout=15)
    return True

def cmd_archive_all():
    tracker = json.loads(TRACKER_FILE.read_text())
    for s in tracker.get("sessions", []):
        cmd_archive(s["id"], s.get("profile", "default"))
```

### Wrapper Shell Script

`smart-archive-desktop.sh` — runs on cron every 3h (scan only, NO prune):

```bash
#!/usr/bin/env bash
set -euo pipefail

# Scan all profiles idle/active sessions (output notification)
python3 "C:\Users\lk\AppData\Local\hermes\scripts\smart-archive.py" scan
```

Note: this script used to also run `hermes sessions prune --older-than 90 --yes` across all profiles. The user removed this feature — they only want scan notifications, no automatic cleanup.

## Cron Configuration

**Desktop-only Hermes (no gateway channels):** Agent mode — creates a visible session in the desktop session list.

```python
cronjob(action='create',
  name='🧠 智能归档（跨 profile）',
  schedule='every 3h',
  prompt='执行智能归档扫描并报告结果。\n\n1. 运行 python3 /c/Users/lk/AppData/Local/hermes/scripts/smart-archive.py scan\n2. 读取输出并报告结果——列出闲置会话数量及详情、活跃会话数量即可',
  enabled_toolsets=['terminal'],
  deliver='origin')    # delivery fails, but agent session is created in session list
```

**With gateway channels (Telegram/Discord/etc.):** `no_agent` mode — zero tokens, delivers stdout verbatim to the channel.

```python
cronjob(action='create',
  name='🧠 智能归档（跨 profile）',
  schedule='every 180m',
  script='smart-archive-desktop.sh',
  no_agent=True,
  deliver='origin')     # works when home channels exist
```

> **Note:** On desktop-only Hermes, `deliver=origin` and `deliver=all` both fail with `"no delivery target resolved"`. The Agent-mode workaround creates a session the user can view. Test with `cronjob(action='run', job_id=...)` and check `last_delivery_error`.

## Interactive Processing Flow

Current tracker-number syntax:

| User reply | Meaning |
|-----------|---------|
| `1` | Archive + delete session #1 after JSON/meta/queue staging succeeds |
| `1 3 5` | Archive + delete sessions 1, 3, 5 |
| `2+` | Archive + delete from #2 through the end |
| `0` | Archive + delete ALL tracked sessions |
| `不处理` / `跳过` | Skip this round |

Historical note: older drafts mentioned dash ranges such as `1-7` or `1-3 5 7-9`; the current script does not treat those as the supported contract unless explicitly reimplemented and regression-tested.

The agent reads `archive-tracker.json`, parses the indices, and calls the script's archive path for each selected session. Deletion happens only after export + `.meta` + `ingest-queue.json` update succeeds.

## Key Design Decisions (Updated 2026-06-12)

| Decision | Rationale |
|----------|-----------|
| **Multi-profile scanning** | Sub-agents (worker profiles) may have their own sessions. Scan them all. |
| **Child session detection** | `parent_session_id` identifies delegate_task children. Tag them `[child]` in output so user knows they're temporary worker sessions. |
| **Profile-aware archive** | Each profile has its own sessions DB. `hermes --profile <name>` must prefix commands for non-default profiles. |
| **No LLM for interaction parsing** | The agent handles number selection parsing (single, range, mixed) — keeps the script zero-token. |
| **Desktop-only delivery** | Desktop Hermes with no gateway channels cannot use `deliver=origin` or `deliver=all`. Agent mode creates an independent session as a fallback — less intrusive than a push but visible in the session list. Always check `last_delivery_error` after first run. |
| **No automatic prune** | User explicitly removed `hermes sessions prune --older-than` from the workflow. The script only scans and reports — it does not auto-delete old sessions. |
| **Final boxed-table format** | After ~10 format iterations, user settled on: LEFT_W=13/RIGHT_W=41 boxed table with [1] offset 5 left of [2]+, time right-aligned, CJK overflow acceptable. See `smart-archive-output-format.md`. |
| **APPDATA → LOCALAPPDATA** | Windows APPDATA resolves to Roaming (empty DB); LOCALAPPDATA is where Hermes actually stores state.db. Fixed 2026-06-12. |
| **None content crash (2026-06-13)** | `session_to_markdown` assumed all messages have non-None `content`, but some sessions contain messages where `content` is `None`. Added `if content is None: content = \"\"` guard + fixed missing `f`-prefix on line 189 f-string. |
