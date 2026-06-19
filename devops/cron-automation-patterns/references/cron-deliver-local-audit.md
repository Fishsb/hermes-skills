# Cron `deliver=local` Repair and Audit Pattern

## When this applies

Use this when a Hermes cron job runs successfully (`last_status: ok`) but delivery fails with errors like:

- `no delivery target resolved for deliver=all`
- `no delivery target resolved for deliver=origin`

This is common in Desktop-only Hermes where there is no resolvable gateway/home-channel target.

## Key lesson

If the user mainly needs the cron run to execute and retain its output, set:

```text
deliver = local
```

`local` avoids push delivery entirely and stores the run output under:

```text
C:\Users\<user>\AppData\Local\hermes\cron\output\<job_id>\YYYY-MM-DD_HH-MM-SS.md
```

## Critical update pitfall: preserve the full prompt

When using `cronjob(action="update", ...)`, do **not** update only `deliver` if the tool schema forces schedule/prompt fields in your calling surface. If you pass an empty `schedule`, the update fails. If you pass a truncated `prompt_preview`, you can accidentally overwrite the job's real prompt with `...`.

Safe pattern:

1. Read the real prompt from `cron/jobs.json`, a state snapshot, or another authoritative source.
2. Update with the full prompt, schedule, enabled toolsets, and new deliver value.
3. Re-read `cron/jobs.json` to confirm the stored prompt is complete.

Example known-good prompt for smart archive:

```text
执行 python3 /c/Users/lk/AppData/Local/hermes/scripts/smart-archive.py scan，然后把终端输出的原文用代码块```包围后原样返回给我，不要修改内容。
```

## Audit sequence

After changing delivery, do not stop at `cronjob(action="list")`. Run the job and verify the persisted state.

Checklist:

```text
1. cronjob(action="list") shows deliver=local.
2. Trigger cronjob(action="run", job_id=...).
3. Wait until `last_run_at` changes.
4. Confirm:
   - last_status == ok
   - last_delivery_error is null
   - schedule is unchanged
   - prompt is complete, not a preview ending in `...`
   - output file exists under cron/output/<job_id>/
5. Read the newest output file and confirm it contains the expected response.
```

## Interpretation

- `last_delivery_error` is historical until the next run. After changing `deliver`, it may still show the old error until a new run completes.
- If `deliver=origin` still reports `no delivery target resolved`, switch to `deliver=local` for save-only operation.
- `prompt_preview` in tool output is intentionally truncated; it is not proof that the stored prompt is truncated. Verify the underlying job file when in doubt.
