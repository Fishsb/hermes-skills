# Smart Archive Cron Audit — 2026-06-14

Session-specific audit notes for the smart archive cron workflow. Use this reference when maintaining `smart-archive.py`, the wrapper script, or cron job configuration.

## Findings

### Cron `script` field must be a wrapper filename only

A cron job was configured as:

```json
{
  "script": "smart-archive.py scan",
  "no_agent": true,
  "last_error": "Script not found: C:\\Users\\lk\\AppData\\Local\\hermes\\scripts\\smart-archive.py scan"
}
```

Hermes resolves `script` as a filename under `C:\Users\<user>\AppData\Local\hermes\scripts\`. Arguments are not parsed from this field, so `smart-archive.py scan` is treated as one literal filename.

Correct pattern:

```bash
# scripts/smart-archive-desktop.sh
#!/usr/bin/env bash
set -euo pipefail
python3 "C:\Users\lk\AppData\Local\hermes\scripts\smart-archive.py" scan
```

Cron configuration should use:

```json
{
  "script": "smart-archive-desktop.sh",
  "no_agent": true,
  "deliver": "local"
}
```

### Non-destructive verification checklist

Before changing smart archive cron wiring, run only read-only or mock tests unless the user explicitly approves deletion:

```bash
python3 -m py_compile "C:\Users\lk\AppData\Local\hermes\scripts\smart-archive.py"
bash -n "C:\Users\lk\AppData\Local\hermes\scripts\smart-archive-desktop.sh"
python3 "C:\Users\lk\AppData\Local\hermes\scripts\smart-archive.py" __path_probe__
python3 /c/Users/lk/AppData/Local/hermes/scripts/smart-archive.py __path_probe__
```

Expected path-probe output is a friendly unknown-command message; this proves the script path is reachable without running `scan`.

For destructive archive/delete paths, use monkey-patched temp directories and fake `subprocess.run` responses to verify ordering:

```text
export -> validate JSON -> write staged JSON -> write .meta -> update ingest-queue -> delete
```

Regression cases to preserve:

- export failure: do not stage, queue, or delete
- invalid JSON export: do not stage, queue, or delete
- staging/meta/queue failure: do not delete; clean orphan staging when possible
- delete failure after queue success: report `partial_delete_failed` and record status in meta/queue
- success: delete only after staged JSON, meta, and queue are durably written

### High-risk but documented behavior: unnamed session cleanup

Current `cmd_scan()` auto-deletes all sessions whose title is `未命名`, including active sessions, without export or queueing. This behavior is documented as a user preference in the skill, but it is high-impact: a read-only SQLite probe found 97 unnamed sessions in the default profile during the audit.

Future agents should not change this behavior silently. If revisiting it, explicitly ask whether to keep unconditional cleanup or narrow it to one of:

- only `cron_%` sessions
- only idle unnamed sessions older than N hours
- only an explicit cleanup command
- scan-only warning without deletion

### Stale reference warning

`references/smart-archive-implementation.md` contained outdated design notes at audit time, including Agent-mode/every-3h cron, old archive target `D:\lk\Obsidian\Hermes\会话归档\`, old markdown-save code, and `1-7` range syntax. Treat that reference as historical unless it has been refreshed; prefer the main SKILL.md and this audit note for current cron wiring.
