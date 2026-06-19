# Smart Archive Delivery: no_agent vs Agent mode visibility (2026-06-15)

## Root Cause

`no_agent=true` + `deliver=local` on Desktop-only Hermes writes script stdout to disk files:

```
C:\Users\<user>\AppData\Local\hermes\cron\output\<job_id>\<timestamp>.md
```

These files are NOT surfaced in:
- The Desktop GUI session list
- The Desktop GUI Cron Jobs dropdown/run history panel
- Any push notification

The user sees nothing unless they manually navigate the filesystem.

## Working Fix (as of 2026-06-15 07:53)

**Agent mode WITHOUT script parameter:**

```
no_agent: false
prompt: 把下方 Script Output 中的扫描结果原文用代码块```包围后返回。不要回复 SILENT，不要添加任何解释。
script: smart-archive-desktop.sh   # data collection — script runs BEFORE agent
deliver: local
enabled_toolsets: ["terminal"]
```

Why `script` parameter (not prompt-based `terminal`):
- Prompt-based approach works but requires the agent to call `terminal` tool, which adds latency and risks the SILENT instruction overriding
- `script` parameter runs the scan BEFORE the agent starts — the output is already in context
- Both approaches work; `script` parameter is preferred for minimal latency and reliability

Requirements:
1. `AUTO_CLEANUP_CRON_SESSIONS_ON_SCAN = True` (default) — deletes old cron sessions
2. Direct SQLite DELETE (not `hermes sessions delete` subprocess) — avoids lock timeout
3. Prompt must include "不要回复 SILENT" — overrides cron system instruction
4. Prompt must require code block wrapping with ```

Result:
- 1 session visible in session list (the current run)
- Output visible in Cron Jobs → run history panel
- Self-cleaning deletes old sessions on each run
- Box-drawing characters preserved in code block

## Failed Approaches

### no_agent=true + deliver=local
Script runs, output in files — invisible in GUI. User sees nothing.

### Agent mode with script parameter + hermes sessions delete subprocess
`hermes sessions delete` subprocess acquires write lock → conflicts with scheduler's uncommitted transaction → 120s timeout.

### Agent mode without script, prompt tells agent to self-delete
Agent returns `[SILENT]` instead of running scan — SILENT system instruction overrides.

### Agent mode without script, explicit "不要回复 SILENT"
Works but slower (agent must make 2-3 tool calls vs. 0 for script parameter).
