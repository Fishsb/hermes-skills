# Cron SILENT Instruction Conflict

## What happens

When an Agent-mode cron job prompt tells the agent to run a command and return output, the cron scheduler injects a system-level instruction:

```
[IMPORTANT: You are running as a scheduled cron job. ... SILENT: If there is
genuinely nothing new to report, respond with exactly "[SILENT]" (nothing else)
to suppress delivery. Never combine [SILENT] with content — either report your
findings normally, or say [SILENT] and nothing more.]
```

Some models (especially reasoning-focused ones) weigh system-level instructions above the user prompt. They see the SILENT directive and, even when told to run a scan and return results, decide "nothing new to report" and respond with just `[SILENT]`.

## Detection

The cron output file shows:

```
## Response

[SILENT]
```

The job `last_status` is `ok` but no session was created and no scan output is visible.

## Fix

Always include **explicit override** in the cron prompt:

```
不要回复 SILENT
```

Or in English:

```
Do NOT respond with SILENT.
```

## Reproduction (2026-06-15)

Cron prompt:
```
执行以下步骤：
1. 运行 python3 ... smart-archive.py scan
2. 把终端输出的原文用代码块```包围后原样返回
3. 然后用 python3 -c "..." 删除当前 cron 会话
不要添加任何解释或额外内容。
```

Agent responded with `[SILENT]` — skipped all three steps because it interpreted the SILENT system instruction as overriding the user prompt.

## Working prompt

```
把下方 Script Output 中的扫描结果原文用代码块```包围后返回。不要回复 SILENT，不要添加任何解释。
```

With the `script` parameter providing pre-run data collection, the agent sees the scan output in context and just needs to format and return it. The explicit "不要回复 SILENT" prevents the system instruction from overriding.
