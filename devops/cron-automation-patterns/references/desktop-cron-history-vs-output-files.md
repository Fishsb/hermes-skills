# Desktop Cron history vs no_agent output files

## Session-derived lesson

Hermes Desktop currently shows Cron Jobs run history by calling:

```text
apps/desktop/src/hermes.ts:getCronJobRuns(jobId)
  → GET /api/cron/jobs/{job_id}/runs
hermes_cli/web_server.py:/api/cron/jobs/{job_id}/runs
  → SessionDB.list_cron_job_runs(...)
  → state.db sessions whose id starts with cron_{job_id}_
```

It does **not** read files under:

```text
C:/Users/<user>/AppData/Local/hermes/cron/output/<job_id>/*.md
```

Therefore:

| Cron mode | Creates cron session | Desktop sidebar/history shows run | Writes cron/output file |
|---|---:|---:|---:|
| `no_agent=true` + script | No | No (in current Desktop implementation) | Yes |
| Agent mode (`no_agent=false`) | Yes | Yes | Yes |

The user's desired behavior — “do not show a session in the normal session area, but show the output in Cron Jobs history” — requires a Desktop/API feature change. The current upstream `main` implementation (checked via GitHub API against `apps/desktop/src/hermes.ts`, `apps/desktop/src/app/chat/sidebar/cron-jobs-section.tsx`, `hermes_cli/web_server.py`, and `hermes_state.py`) matches local behavior and does not support output-file backed run history.

## Practical workflow for this user

1. If the user prioritizes **no session-area pollution**, use:

```text
no_agent=true
script=<wrapper>.sh
deliver=local
```

Then inspect output files directly or via CLI tooling; Desktop run history may appear empty.

2. If the user prioritizes **visible Desktop Cron Jobs history**, use Agent mode:

```text
no_agent=false
prompt='run the script/command and return output in a code block'
enabled_toolsets=["terminal"]
deliver=local
```

This creates `cron_<job_id>_...` sessions. If session clutter is unacceptable, either keep only the latest cron session or implement a proper Desktop/API patch.

3. For long-running or lock-sensitive scripts, prefer Agent mode where the prompt calls `terminal(...)` directly rather than using the cron `script` data-collection field. The data-collection stage can hold scheduler context while the script also touches `state.db`, causing timeouts or stale error files in this user’s workflow.

## Output-file cleanup pattern

If using no-agent/script mode and output files should not accumulate, add deterministic cleanup to the script:

```python
CRON_JOB_ID = os.environ.get("SMART_ARCHIVE_CRON_JOB_ID", "<job_id>")
CRON_OUTPUT_DIR = HERMES_HOME / "cron" / "output" / CRON_JOB_ID

def cleanup_cron_output_files(keep=1):
    if not CRON_OUTPUT_DIR.exists():
        return {"candidates": 0, "deleted": 0}
    files = sorted(CRON_OUTPUT_DIR.glob("*.md"), key=lambda p: p.stat().st_mtime, reverse=True)
    old_files = files[keep:]
    deleted = 0
    for f in old_files:
        try:
            f.unlink()
            deleted += 1
        except Exception:
            pass
    return {"candidates": len(old_files), "deleted": deleted}
```

Call it at scan start, before printing the report, and optionally print a short line such as:

```text
🧹 自动清理了 X/Y 个旧 Cron Jobs 历史文件
```

## Verification checklist

- `cronjob(action="list")` confirms `no_agent`, `script`, `deliver`, `last_status`.
- Check output files:

```bash
ls -t "C:/Users/<user>/AppData/Local/hermes/cron/output/<job_id>/" | head
```

- Check whether Desktop history can see runs by counting cron sessions:

```python
SELECT id,title,started_at FROM sessions WHERE id LIKE 'cron_<job_id>_%' ORDER BY started_at DESC;
```

- If there are output files but zero cron sessions, current Desktop history will not show them.

## Proper product fix

Patch the Desktop/API layer so `/api/cron/jobs/{job_id}/runs` returns both:

1. real cron sessions from `state.db`; and
2. synthetic run records from `cron/output/<job_id>/*.md` for no-agent jobs.

The frontend then needs a way to open/render these synthetic output-file records without treating them as normal chat sessions.
