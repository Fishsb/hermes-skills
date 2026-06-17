---
name: knowledge-base-compression
description: Compress and deduplicate knowledge-base structures across Memory, User Profile, SOUL, skills, notes, and archives while preserving task effectiveness.
version: 1.1.0
author: lk
platforms:
- windows
metadata:
  hermes:
    tags:
    - compression
    - cleanup
    - memory
    - skills
    - optimization
    - mainte
    trigger_words:
    - 上下文压缩
    - 记忆压缩
---

# Knowledge Base Compression

> Compress knowledge-base structure using one criterion only: **do not reduce task execution effectiveness**.

<!-- Agent: English main body is the primary read path. Chinese source is archived in `references/中文原文.md` for human lookup and rollback. -->

## Trigger

Load this skill when the user says:

- **上下文压缩**
- **记忆压缩**
- knowledge-base compression
- memory compression / memory cleanup

## Workflow

### 0️⃣ Pre-Execution Confirmation

**Always ask the user before executing compression changes.** Do not auto-clean.

Use this confirmation shape after analysis:

```text
[Compression analysis complete]
Cleanable items found: N
Estimated Memory: X% → Y%
Estimated User Profile: X% → Y%
Execute compression? (Y/n)
```

Continue only after user confirmation.

### 1️⃣ Read Current State

```python
# Use memory/terminal-style tools to obtain current capacity and entries for each storage system.
```

Objects to inspect:

| Object | Inspection tool | Judgement criteria |
|---|---|---|
| Memory | Read the current system prompt MEMORY block | entry count + used characters |
| User Profile | Read the current system prompt USER PROFILE block | entry count + used characters |
| SOUL.md | `read_file ~/AppData/Local/hermes/SOUL.md` | duplicate rules vs PLUR or skills |
| Skills directory | list `~/AppData/Local/hermes/skills/` | empty categories, platform mismatch, categories missing `DESCRIPTION.md` |
| PLUR | `mcp_plur_plur_status`, `mcp_plur_plur_doctor` | embedding status, duplicate engrams |

### 2️⃣ Judge Each Item — Deletion Criteria

**Core principle: delete only data that does not help task execution.**

#### Memory Deletion Criteria

Delete:
- ❌ Completed-session artifacts, such as “PR #XXXX submitted” or “fixed bug Y”.
- ❌ One-off cleanup records.
- ❌ Behavior rules fully duplicated by `SOUL.md` or a PLUR engram.
- ❌ Parameter details already fully documented inside a skill, such as electron-builder steps.

Keep:
- ✅ Knowledge-base pointers.
- ✅ Stable environment facts.
- ✅ Network fallback paths.
- ✅ Tool configuration parameters.

#### User Profile Deletion Criteria

Delete:
- ❌ Behavior rules fully duplicated by `SOUL.md` or PLUR.
- ❌ Historical records for deleted profiles.
- ❌ UI preference details or budget trivia.
- ❌ Procedural preferences that belong in skills.

Keep:
- ✅ Core identity.
- ✅ Interaction style preferences.
- ✅ Safety rules.
- ✅ Methodology.
- ✅ Server information.
- ✅ User decision records.

#### SOUL.md Deletion Criteria

Delete:
- ❌ Behavior rules fully duplicated by PLUR engrams.

Keep:
- ✅ Personality definition.
- ✅ Communication style.
- ✅ Tone preferences.

#### Skills Deletion / Merge Criteria

Delete:
- ❌ Platform-mismatched skills, such as Apple-only skills on Windows.
- ❌ Empty categories, such as folders containing only `DESCRIPTION.md` and no real skill.

Merge:
- 🔀 Skills with highly duplicated content.

## 3️⃣ Execute Changes

Use the `memory()` tool for Memory and User Profile. Do **not** edit those stores with `write_file`.

```python
# Memory operations
memory(action="remove", target="memory", old_text="...unique identifier...")

# User Profile operations
memory(action="remove", target="user", old_text="...unique identifier...")
memory(action="replace", target="user", old_text="...", content="...compressed content...")
```

Important details:

- Each `memory()` call operates on one entry only.
- Multiple calls are required for multi-entry cleanup.
- `old_text` must match a unique identifier from one entry. Do not include several entries inside one `old_text`.
- Rewrite `SOUL.md` with `write_file` only after reading the current content.
- Use terminal `rm -rf` for intended and safe skill directory cleanup.
- For skill merges, use `skill_manage(action='patch')`, then `skill_manage(action='delete', absorbed_into='...')`.

## 4️⃣ Verify

After execution, re-read relevant state and confirm:

- Memory / User Profile capacity decreased meaningfully.
- `SOUL.md` content is correct.
- Skills directory structure is reasonable.
- Critical task-execution information was not deleted.

## 5️⃣ Report Format

Report compression results like this:

```text
Memory: X% → Y% (removed N entries)
User Profile: X% → Y% (removed N, compressed N)
SOUL.md: optimized
Skills: merged N groups, deleted N empty/stale categories
PLUR: embedding status X
```

## Pitfalls

- **`memory()` has a character limit**: 2,200 chars for memory. If replacement content is too large, use `remove` + `add` as separate steps.
- **`SOUL.md` can be concurrently modified by subagents**: always read it before rewriting, then verify after writing.
- **`rmdir` cannot delete non-empty directories**, including directories that only contain `DESCRIPTION.md`. Use `rm -rf` when deletion is intended and safe.
- **`skill_manage()` cannot update root-level skills such as `agent-reach`** in this environment. Edit the file directly with terminal/file tools when necessary.
- **Apple-only skills are safe to remove only on Windows**. Preserve them on macOS.
- **Do not put multiple entries inside one `old_text`**: `memory()` matches one entry at a time.

## References

- `references/20260616-context-compression-recall.md` — session-derived recall map for prior context/memory compression work, including the “按最近的状态” correction and the Obsidian files to consult before proposing compression.
- `references/20260616-context-slimdown-execution.md` — execution notes for the approved 2026-06-16 context slimdown: splitting an oversized class-level skill into `references/`, archiving Memory/User Profile details to Obsidian, and verifying residue keywords.
- `references/20260616-context-slimdown-234-pattern.md` — follow-up pattern for strict numbered-scope execution (`234` means only options 2/3/4), active-runtime backups, class-level skill slimming into references, and verification after Memory/User compression.
- `references/compression-example-history.md` — full execution record from the initial creation on 2026-06-14.
- `references/中文原文.md` — full Chinese original archive.
- `SOUL.md` path: `~/AppData/Local/hermes/SOUL.md`.
- Skills root: `~/AppData/Local/hermes/skills/`.
- PLUR storage root: `~/.plur/`.
