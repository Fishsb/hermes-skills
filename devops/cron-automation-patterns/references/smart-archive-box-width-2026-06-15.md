# Smart Archive Box Width Adjustment — 2026-06-15

## Session Signal

User corrected the smart-archive boxed-table display after restoring the earliest output style:

- First request: make top/bottom borders longer by “0.3”.
- Clarification: this meant **increase by 30%**, not add 3 characters.
- Concrete result: `WIDTH = round(54 * 1.3) = 70`.

## Durable Rule

For this user's smart-archive boxed-table output, keep the **earliest single-box layout** but use a wider box:

```python
WIDTH, LABEL_W = 70, 18
```

This only changes output width. It must not roll back current smart-archive behavior such as:

- `scan` deletes unnamed / empty-title sessions by default.
- `SMART_ARCHIVE_AUTO_DELETE_UNNAMED=0/false/no/off` disables unnamed cleanup.
- Old `cron_%` smart-archive report-session cleanup remains opt-in via `SMART_ARCHIVE_AUTO_CLEANUP_CRON=1`.
- Archive commands still export raw JSON → write `.meta` → queue → delete original Hermes session only after staging succeeds.

## Verification Recipe

Use AST parsing instead of only running the script after edits:

```bash
python3 - <<'PY'
import ast, pathlib, subprocess
p = pathlib.Path('C:/Users/lk/AppData/Local/hermes/scripts/smart-archive.py')
ast.parse(p.read_text(encoding='utf-8'))
print('AST_OK=True')
res = subprocess.run(['python3','-u',str(p),'scan'], capture_output=True, text=True, timeout=120)
print('SCAN_EXIT=', res.returncode)
for line in res.stdout.splitlines():
    if line.startswith('┌'):
        print('TOP_BORDER_DASHES=', line.count('─'))
        break
PY
```

Expected:

```text
AST_OK=True
SCAN_EXIT=0
TOP_BORDER_DASHES=70
```

## Pitfall From This Session

Tiny repeated `patch()` edits inside a compact function can accidentally remove adjacent lines when the old/new context is too narrow. In this session, small edits temporarily removed `top_border`, `row`, `left_label`, `sessions = sessions[:max_rows]`, `idle`, `idx_str`, and `title` lines.

Safer pattern for this class of edit:

1. Read the function boundaries.
2. Replace the whole function block programmatically or with a single sufficiently contextual patch.
3. Immediately run `ast.parse(...)` and one script smoke test.

Do not re-run the same oversized tool call if it times out; split into smaller patches or use a short deterministic script to replace only the target function block.
