---
name: cron-automation-patterns
description: Design Hermes cron workflows with no_agent scans, notifications, user-triggered execution, confirmation loops, logs, cleanup, and verification.
version: 1.9.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [cron, automation, devops, hermes, smart-archive]
    related_skills: [skill-optimization]
triggers:
- User wants to automate periodic checks (idle sessions, disk usage, memory staleness, config drift)
- User wants a cron task that notifies them and waits for their approval before acting
- User builds a cron job and needs to decide between Agent mode (LLM-driven) and no_agent script mode
- User wants batch operations on Hermes sessions (list, archive, delete) on a schedule
notes:
  script_wrapper: Cron no_agent scripts need a wrapper under scripts/ — .py files run via Python directly (preferred on Windows), .sh files via bash.
  no_direct_jobs_json: Never write jobs.json directly — always use the cronjob() tool.
---

# Cron Automation Patterns

## Overview

Reusable architecture for cron tasks needing **zero-token scanning** + **human confirmation** before execution.

```
[Phase 1] no_agent script (zero tokens) — runs on schedule
  └─ Scans state, detects changes, outputs notification
[Phase 2] Interactive agent (on demand) — user triggers
  └─ Uses clarify to present options, then executes via script
```

## When to Use Each Cron Mode

| Factor | Agent Mode (LLM) | no_Agent Script |
|--------|-----------------|-----------------|
| Token cost | 10K-100K+ per run | Zero |
| Decision-making | LLM reads context, decides | Script follows rules |
| Writing/archiving | LLM formats content | Script generates markdown |
| User interaction | One-way notification | Notification + agent handles interactive |
| Best for | Complex content decisions | Repetitive scanning, filtering, format transforms |
| **Desktop visibility** | **Creates `cron_{job_id}_...` session** — visible in Cron Jobs panel | Writes to `cron/output/<job_id>/` — NOT visible in GUI |
| **Smart-archive class** | ✅ Use with prompt-only (no `script` param) + self-cleaning | Use only when file-only history acceptable |

**Critical:** On desktop-only Hermes, `no_agent=true` + `deliver=local` saves to disk files only — NOT visible in any GUI panel. For GUI visibility, use Agent mode without `script` parameter.

## Two-Phase Architecture

### Phase 1: Scanner Script (no_agent)
- Write Python script under `scripts/` that gathers data without LLM (SQLite, `hermes sessions list`, file stats)
- Output formatted notification to stdout → becomes desktop push
- **Prefer `.py` wrapper** on Windows (not `.sh`) — avoids shell encoding issues
- Cron setup via `cronjob(action='create', script='wrapper.py', no_agent=True, deliver='local')`
- 详细 wrapper pattern、Windows 编码修复见 `references/windows-cron-encoding-fix-2026-06-16.md`

### Phase 2: Interactive Processing (Agent)
- User triggers by responding to notification → Agent loads tracker file
- Two archive modes via `smart-archive.py`:
  - **Mode A — 快速归档**: `archive <N|N+|0>` — script-only, zero token, inline `extract_knowledge_to_wiki()`
  - **Mode B — 精炼归档**: `stage <N|N+|0>` → Agent LLM refines → `delete-session <sid>`
- Session selection via `clarify` with numbered list
- 完整流程见 `references/skills-architecture.md` + `references/smart-archive-implementation.md`

## Output Format Rules

### Simple listing (cron notification inline)
```
闲置会话（N个）
  [1] 完整会话标题（和桌面对话栏完全一致）
      └ 最后一条消息的前40字符（预览）
      闲置  Xh · NN条消息

活跃会话（N个）
  [1] ...

💡 回复编号归档，如：1、1 2、2+、0=全部
```

**Golden rule:** Show exact session title from database, not truncated. Include last-message preview.

### Boxed format (for displayed output)
CJK-aware single-box layout with `WIDTH=70`, `LABEL_W=18`。详细规范见 `references/smart-archive-output-format.md`。

## Self-Cleaning Cron Sessions

Agent-mode cron jobs create visible sessions each run → accumulate fast. Solution: **delete old history before creating new session**.

Key rules:
- Match sessions by stable job-id prefix `cron_{job_id}_%`, not title
- Keep newest visible session (`rows[1:]`), delete older
- Use direct SQLite DELETE (not `hermes sessions delete`) to avoid lock contention with scheduler
- 完整实现见 `references/smart-archive-self-cleaning.md` + `references/sqlite-isolation-cron-cleanup.md`

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Match by title/prefix, not job ID | Survives cron job recreation |
| Keep latest session | User sees most recent report in session list |
| Unnamed session auto-delete on scan | User preference: `未命名` = clutter |
| Agent mode no `script` param | Avoids SQLite lock contention (120s timeout) |
| direct SQLite DELETE in cron context | `hermes sessions delete` lock conflicts with scheduler |

## Verification Checklist

- [ ] Choose `no_agent` for zero-token scans, Agent mode only when LLM reasoning required
- [ ] Cron `script` is relative path under `scripts/` directory
- [ ] Desktop-only delivery verified via `cronjob(action='run')` + `cronjob(action='list')`
- [ ] Windows paths use `LOCALAPPDATA`, not `APPDATA` (Roaming)
- [ ] Smart archive: `archive N` (fast), `stage N` (agent refine), `delete-session <sid>` (cleanup)
- [ ] Scan output shows exact session titles + last-message preview
- [ ] `cronjob(action='update')` — omit unchanged fields; never send `schedule=""` placeholder
- [ ] After cron config repair, verify via `cronjob list` + direct script run (not `last_error` alone)

## 🚩 Critical Pitfalls

| Pitfall | Fix |
|---------|-----|
| `deliver=origin` silently fails on desktop-only | Switch to Agent mode, no `script` param |
| Agent mode + `script` parameter → 120s timeout | Use prompt-only, agent runs via `terminal` tool |
| `sys.executable` resolves to MS Store `python3` stub | `shutil.which("python") or shutil.which("python3") or sys.executable` |
| Windows GBK/UTF-8 pipe crash on box-drawing/emoji | `.py` wrapper with `errors="replace"` + ASCII-safe replace |
| Surrogate pair escapes (`\ud83d\udce4`) crash `print()` | Replace with `\U0001f4e4` or literal emoji |
| `cronjob(action='update')` rejects empty `schedule=""` | Omit unchanged fields; preserve existing valid values |
| `archive 0` only covers idle sessions in tracker | Active sessions not in tracker; clarify with user before manual archive |
| APPDATA 陷阱: `APPDATA` = Roaming, state.db in Local | Use `LOCALAPPDATA` for DB paths |
| SQLite WAL checkpoint → empty query results | `PRAGMA wal_checkpoint(TRUNCATE)` before query |
| `cron` scheduler stops ticking after repeated encoding crashes | Restart Desktop GUI or `hermes cron tick` |

Windows 专属排查、hermes CLI 路径解析、编码修复详见 `references/windows-cron-encoding-fix-2026-06-16.md`。

## Multi-Profile Scanning

Use `get_all_state_dbs()` to discover all profiles' `state.db`, including the default. `delegate_task` child sessions stored in **parent profile's `state.db`** — NOT in sub-profile databases. Full query pattern in `references/skills-architecture.md` and `references/smart-archive-implementation.md`。

## References

| File | Content |
|------|---------|
| `references/skills-architecture.md` | Smart-archive skills ownership matrix |
| `references/smart-archive-implementation.md` | Full smart-archive.py script, cron config, backup |
| `references/smart-archive-self-cleaning.md` | Self-cleaning cron sessions + two-group output |
| `references/smart-archive-output-format.md` | CJK-aware boxed-table layout, width, alignment |
| `references/windows-cron-encoding-fix-2026-06-16.md` | GBK/UTF-8 crash root cause, `.py` wrapper fix, verification |
| `references/desktop-cron-history-and-cleanup.md` | Desktop Cron Jobs history: reads `cron_*` sessions, not output files |
| `references/cron-deliver-local-audit.md` | `deliver=all/origin` failure on desktop-only; full-prompt preservation |
| `references/smart-archive-cron-audit-2026-06-14.md` | Session audit: `script` argument pitfall, safe probes |
| `references/smart-archive-safe-scan-2026-06-15.md` | Wrapper-based cron, env-gated cleanup (superseded) |
| `references/smart-archive-unnamed-cleanup-default-2026-06-15.md` | User-corrected unnamed session delete policy |
| `references/smart-archive-display-style-restore-2026-06-15.md` | Output format restoration from backup |
| `references/smart-archive-box-width-2026-06-15.md` | Width adjustment +30% → WIDTH=70 |
| `references/desktop-cron-history-vs-output-files.md` | Source audit: UI reads sessions, not files |
| `references/sqlite-isolation-cron-cleanup.md` | SQLite tx isolation in `cleanup_old_sessions()` |
| `references/smart-archive-no-agent-deliver-final.md` | Delivery visibility fix timeline |
| `references/cron-silent-instruction-conflict.md` | SILENT instruction overrides prompt |
| `references/surrogate-pair-escape-fix.md` | `\ud83d\udce4` surrogate pair fix |
| `references/active-session-archive-fallback-2026-06-16.md` | Active session export/delete workflow |

## Changelog

- **v1.9.0** — 拆分：核心骨架 ~8KB，详情下沉到 references/（原 65KB → 精简）
