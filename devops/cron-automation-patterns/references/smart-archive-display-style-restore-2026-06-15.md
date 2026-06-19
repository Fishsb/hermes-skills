# Smart Archive Display Style Restore — 2026-06-15

## Trigger

User reported the smart-archive scan output display style looked wrong after iterative fixes and explicitly asked to restore the earliest/example style while keeping the latest optimized behavior.

## Decision

Restore only the boxed-table display function to the earliest known style. Keep all modern workflow logic intact:

- `scan` deletes `未命名` / empty-title sessions by default.
- `SMART_ARCHIVE_AUTO_DELETE_UNNAMED=0/false/no/off` disables unnamed cleanup.
- Old `cron_%` smart-archive report session cleanup remains opt-in via `SMART_ARCHIVE_AUTO_CLEANUP_CRON=1`.
- `archive <N|N+|0>` still means export raw JSON → write `.meta` → update ingest queue → delete original Hermes session after success.
- Cron still uses the `.sh` wrapper as the `script` field.

## Source of Truth for Display Style

Earliest output sample:

```text
C:/Users/lk/AppData/Local/hermes/cron/output/db5090d04b33/2026-06-12_02-54-06.md
```

Script backup containing the earliest function:

```text
D:/lk/Obsidian/Hermes/wiki/配置/Hermes脚本备份_20260613.md
```

Relevant function starts around line 221 in that note.

## Restored `make_boxed_table()`

```python
def make_boxed_table(label, sessions, icon, max_rows=15):
    WIDTH, LABEL_W = 54, 18
    def top_border(): return "┌" + "─" * WIDTH + "┐"
    def bottom_border(): return "└" + "─" * WIDTH + "┘"
    def row(content): return "│" + content.ljust(WIDTH) + "│"
    lines = [top_border()]
    left_label = f"  {icon} {label}"
    total = len(sessions)
    sessions = sessions[:max_rows]
    def make_item(idx, s, fill_extra=0):
        idle = f"{s['idle_hours']}h" if s['idle_hours'] >= 1 else f"{int(max(s['idle_hours']*60, 1))}m"
        tag = f"[{s['profile']}]" if s['profile'] not in ("default","") else ""
        idx_str = f"[{idx}] "
        fill_w = (WIDTH - LABEL_W + fill_extra) - len(idx_str) - 1 - len(idle)
        title = (tag + s['title'])[:fill_w].ljust(fill_w)
        return idx_str + title + " " + idle
    if sessions:
        line1 = left_label.ljust(LABEL_W - 5) + make_item(1, sessions[0])
        lines.append("│" + line1 + "│")
        for idx, s in enumerate(sessions[1:], 2):
            lines.append(row(" " * LABEL_W + make_item(idx, s)))
    else:
        lines.append(row(left_label))
        lines.append(row("  （无）"))
    lines.append(bottom_border())
    return "\n".join(lines)
```

## Important Pitfall

When replacing `make_boxed_table()` with targeted patching, include the boundary before `def cmd_scan():` carefully. In the 2026-06-15 session, a patch accidentally removed the `def cmd_scan():` line because the replacement range ended too broadly. Always re-read the function boundary and run an AST syntax check after patching.

## Verification Pattern

After changing only display style:

```bash
python3 - <<'PY'
import ast, pathlib, subprocess
script=pathlib.Path('C:/Users/lk/AppData/Local/hermes/scripts/smart-archive.py')
ast.parse(script.read_text(encoding='utf-8'))
print('AST_OK=True')
res=subprocess.run(['python3','-u',str(script),'scan'], capture_output=True, text=True, timeout=120)
print('SCAN_EXIT=', res.returncode)
print('\n'.join(res.stdout.splitlines()[:35]))
PY
```

Expected checks:

- `AST_OK=True`
- `SCAN_EXIT=0`
- Output keeps the single-box style from the earliest sample.
- Latest behavior remains intact: unnamed sessions are still auto-cleaned by default.
