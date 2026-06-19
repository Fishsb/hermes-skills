# Desktop Cron History vs no_agent Output Files

Session-derived reference for maintaining Desktop-visible cron notifications without modifying Hermes upstream source.

## Root Cause

Hermes Desktop's Cron Jobs history UI calls:

```text
GET /api/cron/jobs/{job_id}/runs
  -> SessionDB.list_cron_job_runs(job_id)
  -> state.db sessions where id LIKE 'cron_{job_id}_%'
```

It does **not** read files under:

```text
C:/Users/<user>/AppData/Local/hermes/cron/output/<job_id>/*.md
```

Therefore:

| Mode | Desktop Cron Jobs history | Desktop session list | output files |
|---|---|---|---|
| `no_agent=true` script-only | not shown in current Desktop UI | no cron session | yes |
| Agent mode with `script` data collection | shown | creates `cron_{job_id}_...` session | yes |

If the user wants history visible in the Desktop Cron Jobs dropdown while keeping upstream Hermes source unchanged, use Agent mode with `script`, not `no_agent`.

## Recommended Official-Source-Compatible Pattern

```text
no_agent=false
script=<wrapper>.sh
prompt="把下方 Script Output 中的扫描结果原文用代码块```包围后返回。不要回复 SILENT，不要添加任何解释。"
deliver=local
enabled_toolsets=["terminal"]
```

This produces a cron session that Desktop can list as run history, while the script remains deterministic and reusable as a plugin/workflow without patching Hermes source.

## Cleanup Pitfall: Do Not Delete the Latest Visible Cron Session During Pre-Run

The script runs before the new cron session is fully committed to `state.db`. At script time, the newest visible `cron_{job_id}_...` session is the **previous** run — the one currently backing the Desktop history display.

Bad pattern:

```python
# Deletes every visible old cron row during pre-run.
# This can make Desktop history disappear until the new run finishes,
# or stay empty if the run fails later.
delete_ids = [row.id for row in rows]
```

Safe pattern:

```python
# Keep the newest visible run; delete only older rows.
rows = query("SELECT id FROM sessions WHERE id LIKE ? ORDER BY started_at DESC", [f"cron_{job_id}_%"])
delete_ids = [row.id for row in rows[1:]]
```

This means the steady state is usually **two** rows immediately after a successful run: previous visible run + newly-created current run. On the next run, older rows are cleaned. This is safer than deleting the only visible history.

## Cleanup Pitfall: Output Files Are Separate From Desktop History

Agent-mode cron also writes markdown files to `cron/output/<job_id>/`. These are not what Desktop reads for run history, but they can accumulate on disk.

Safe pre-run output cleanup:

```python
# Current run's output file does not exist yet.
# Keep the newest existing file and delete older ones.
files = sorted(output_dir.glob("*.md"), key=lambda p: p.stat().st_mtime, reverse=True)
for old in files[1:]:
    old.unlink()
```

Do not use `keep=0` pre-run unless the user explicitly accepts that there may be no file-backed history if the current run fails before writing output.

## Prefer Job-ID Prefix Over Title Matching

Cron session titles can be model-generated or changed by the agent. Match sessions by stable ID prefix:

```text
cron_{job_id}_%
```

Do not rely on title substrings such as `智能归档` or emoji prefixes.

## Verification Checklist

After changing a Desktop-visible cron job:

1. `cronjob(action="list")` shows `no_agent=false`, `script=<wrapper>`, `deliver=local`.
2. Trigger with `cronjob(action="run", job_id=...)` or `hermes cron run <id>`.
3. Wait for the scheduler tick and model response to finish.
4. Verify `cron/output/<job_id>/` has a recent `.md` output.
5. Verify `state.db` has at least one `cron_{job_id}_...` session.
6. Verify the newest cron session assistant response contains the code-fenced scan output.
7. Verify cleanup did not delete the newest visible run before the new one completed.
