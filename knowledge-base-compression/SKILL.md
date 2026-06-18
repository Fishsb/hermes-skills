---
name: knowledge-base-compression
description: Compress and deduplicate knowledge-base structures across Memory, User Profile, SOUL, skills, notes, and archives while preserving task effectiveness.
version: 2.3.0
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
    - audit
    trigger_words:
    - 上下文压缩
    - 记忆压缩
    - 知识库压缩
    - kb audit
    - knowledge audit
related_skills:
  - lk-memory-index
  - skill-cleaner
  - skill-optimization
---

# Knowledge Base Compression

> **高保真压缩** — 只删除不影响任务执行的数据，重要细节必须保留或迁移而非丢弃。

<!-- Agent: English main body is the primary read path. Chinese source is archived in `references/中文原文.md` for human lookup and rollback. -->

## Core Principle: 高保真三原则

本 skill 的一切操作受三条硬约束，优先级递减：

### ① 重要细节零丢失

任何压缩操作**不得丢失对后续任务有潜在价值的信息**。判定标准：

- 如果一条数据对**至少一个**未来任务有帮助 → 必须保留（可迁移到更合适的存储层）
- 有潜在价值但不急需的 → 归档到 Obsidian，原位置替换为短指针
- 确认无价值的历史残留 → 可删除

### ② 压缩分级执行

对不同类型数据采用不同压缩力度，不得一刀切：

| 压缩级别 | 适用类型 | 操作方式 |
|---------|---------|---------|
| **🔒 全文保留** | 环境稳态事实、网络 fallback 路径、工具配置参数、核心身份、安全规则、方法论 | 精简格式但不删内容 |
| **📎 指针化** | 长文档、历史决策记录、复杂迁移日志（>200 chars 的热存储内容） | 压缩为 ≤80 字指针 → 归档到 Obsidian |
| **🗑️ 安全删除** | 一次性清理日志、已废弃 profile 历史、已被 SOUL/PLUR/Skill 完整覆盖的重复规则 | 直接删除 |

### ③ 拆分与互联先行

当单个文件过大（>15KB）或信息密度过高导致压缩中无法保留重要细节时：**先拆分，后压缩，再互联**。详见下方 [拆分与互联规范](#拆分与互联规范split--cross-link)。

---

## Trigger

Load this skill when the user says:

- **上下文压缩**
- **记忆压缩**
- **知识库压缩**
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

**🎯 PROGRESS_TICK** → 读取完成后输出：`✓ 状态读取完成：Memory X/N chars, User Profile X/N chars, Skills N dirs`

Objects to inspect:

| Object | Inspection tool | Judgement criteria |
|---|---|---|
| Memory | Read the current system prompt MEMORY block | entry count + used characters |
| User Profile | Read the current system prompt USER PROFILE block | entry count + used characters |
| SOUL.md | `read_file ~/AppData/Local/hermes/SOUL.md` | duplicate rules vs PLUR or skills |
| Skills directory | list `~/AppData/Local/hermes/skills/` | empty categories, platform mismatch, categories missing `DESCRIPTION.md` |
| PLUR | `mcp_plur_plur_status`, `mcp_plur_plur_doctor` | embedding status, duplicate engrams |
| Obsidian vault | scan `D:/lk/Obsidian/Hermes/wiki/记忆/` | existing archive structure for pointer migration |

**🎯 PROGRESS_TICK** → 连续 3 次工具调用未输出时，强制插入进度

### 2️⃣ Judge Each Item — Classification

**Core principle: delete only data that does not help task execution.**

Before judging, classify each item into one of the 3 compression levels (①全文保留 / ②指针化 / ③安全删除). Apply level-appropriate handling.

#### Memory Deletion Criteria

Delete:
- ❌ Completed-session artifacts, such as "PR #XXXX submitted" or "fixed bug Y".
- ❌ One-off cleanup records.
- ❌ Behavior rules fully duplicated by `SOUL.md` or a PLUR engram.
- ❌ Parameter details already fully documented inside a skill, such as electron-builder steps.

Keep:
- ✅ Knowledge-base pointers.
- ✅ Stable environment facts.
- ✅ Network fallback paths.
- ✅ Tool configuration parameters.
- ✅ Source references for compressed items (e.g. "D14-D19 decision → see `wiki/记忆/...` in Obsidian").

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
- ✅ Decision traceability (after pointerization, retain enough context for the pointer to be actionable).

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

**🎯 PROGRESS_TICK** → 判定完成后输出：`✓ 判定完成：全文保留 N 条 / 指针化 N 条 / 安全删除 N 条`

### 拆分与互联规范（Split & Cross-Link）

当涉及"指针化"操作或发现待压缩文件 >15KB 时，执行此子流程。

#### Obsidian vault 大文件拆分模式

当遇到 `>100KB` 的 `.md` 文件（如 GitHub 项目汇总表、大量调研笔记），按 `##` 标题拆分为独立文件到子目录，而非内联拆分。参考脚本 `scripts/split-github-projects.py`。

模式：
1. 解析 frontmatter → 保留在索引文件
2. 识别 `##` 标题 → 每个标题为一个子文件
3. 创建子目录 `README.md` 作为分类索引
4. `##` 正文写入独立文件，加 `[源自: 父文件](...)` 来源标记
5. 源文件重命名为 `.md.bak` 保留

详见 `references/kb-audit-patterns.md` §大文件拆分模式。

#### 判断门

以下任一条件成立即触发拆分：

1. **文件 >15KB** — 单文件信息密度过高，压缩可能丢失细节
2. **多主题混杂** — 一个文件中包含 2+ 个逻辑独立的主题，压缩一处分会误伤另一处
3. **高价值长文档** — 重要细节多但当前不适合放在热存储中（长期决策记录、架构设计）

#### 拆分步骤

```
Step 1: 识别逻辑边界 → 按主题 MECE 切分
Step 2: 主文件保留为路由/执行契约（8-15KB），细节移入 references/ 子文件
Step 3: 创建 references/_index.md 作为全局索引
Step 4: 每个子文件添加双向链接（见下方互联规范）
Step 5: 验证所有引用路径可解析
```

#### 互联规范

每个拆分子文件必须包含以下链接信息：

```
# 文件头部（frontmatter 之后，正文之前）
> **上下文**：[父文件](relative-path.md)
> **同级文件**：[A](./a.md) | [B](./b.md)

# 文件尾部（结尾空行前）
---
**关联参考**：[全局索引](./_index.md) | [父文件 →](../SKILL.md)
```

指针化归档到 Obsidian 的规范：

- 指针格式：`[topic → see wiki/记忆/<file>.md in Obsidian]`
- 归档后验证原始文件中的指针可点击/可搜索
- 归档文件需包含 proper frontmatter 和 tags

**🎯 PROGRESS_TICK** → 进入执行前输出：`→ 开始执行：删除 N 条 memory + 指针化 N 条 + 清理 N 个 skill 目录`

### 3️⃣ Execute Changes

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
- **For split operations**: write new files with `write_file`, then update references and verify the link chain.

**🎯 PROGRESS_TICK** → 每完成一项操作后输出（如 `✓ memory 已删除 3 条`），不累积到结束才说

### 4️⃣ Verify

After execution, re-read relevant state and confirm:

- Memory / User Profile capacity decreased meaningfully.
- `SOUL.md` content is correct.
- Skills directory structure is reasonable.
- Critical task-execution information was not deleted.
- **Split integrity**: for any split performed, confirm:
  - `references/_index.md` lists the new file
  - New file contains forward/backward links
  - All linked files are reachable
  - Parent file correctly references the split
- **Pointer integrity**: for any pointerization to Obsidian, confirm the target file exists and the pointer is actionable.

### ⛔ FIX_LOOP GATE — 预输出强制门

在进入 Step 5 报告前，必须逐项核验：

| # | 核验项 | 判定 |
|---|--------|:----:|
| 1 | **匹配触发条件？** 本次压缩操作是否命中 FIX_LOOP 硬性条件（① SKILL.md 编辑 / ② ≥3 次 write/patch / ③ memory() 持久修改 / ④ delegate_task 派发）？ | 是/否 |
| 2 | **零残留？** 压缩后是否存在未修复的 link 断裂、拆分未同步 `_index.md`、指针路径错误？若有 → 立即修复 → 循环 review | 是/否 |
| 3 | **进度可见？** 执行过程中是否在每个节点输出了进度（Step 1→2→3 之间）？未输出时 >3 次调用是否强制插入？ | 是/否 |
| 4 | **输出格式？** 最终报告是否按「先高保真英文摘要 → 再中文说明」顺序？ | 是/否 |

四项全部「是」后才允许进入 Step 5 输出。

### 5️⃣ Report Format

Report compression results like this:

```text
Memory: X% → Y% (removed N entries)
User Profile: X% → Y% (removed N, compressed N)
SOUL.md: optimized
Skills: merged N groups, deleted N empty/stale categories
PLUR: X engrams (N pinned, N candidates for decay)
Split & Cross-Link:
  - Split: N files (list)
  - New references: N files
  - Cross-links verified: ✅
```

### PLUR 压缩指南

PLUR 存储的是语义 engram，压缩时注意：

| 操作 | 方式 | 说明 |
|------|------|------|
| 清理低频记忆 | 调用 `mcp_plur_plur_batch_decay` | ACT-R 衰减，不出现在报告中 |
| 自动衰减（cron） | `scripts/plur-decay.py` — 每周日 02:00 执行 `npx @plur-ai/cli batch-decay` | 纯自动，无需手动触发 |
| 检查重复 engram | 调用 `mcp_plur_plur_doctor` | 查看 engram_count / tension_count |
| 处理冲突 | 调用 `mcp_plur_plur_tensions` | 列出矛盾知识，手动确认保留哪条 |
| 标记高频保留 | 调用 `mcp_plur_plur_pin` | 对核心知识执行 pin，避免被 decay 淘汰 |
| 导出备份 | `plur sync` 到 Git 或 `~/.plur/` 目录备份 | 压缩前建议同步一次 |

PLUR 存储位置：`~/.plur/`（纯 YAML 格式，可直接编辑）
PLUR 本身不占 Memory/User Profile 容量，压缩时不需计入热存储预算。

## Pitfalls

- **`memory()` has a character limit**: 2,200 chars for memory. If replacement content is too large, use `remove` + `add` as separate steps.
- **`SOUL.md` can be concurrently modified by subagents**: always read it before rewriting, then verify after writing.
- **`rmdir` cannot delete non-empty directories** (including dirs with only `DESCRIPTION.md`). Use `rm -rf` when deletion is intended and safe.
- **`skill_manage()` cannot update root-level skills** (e.g. `agent-reach`). Edit directly with terminal/file tools.
- **Apple-only skills are safe to remove only on Windows**. Preserve them on macOS.
- **One `old_text` per call**: `memory()` matches one entry at a time. Do not bundle multiple entries.
- **拆分后互联断裂**：拆分后只更新主文件但不更新 `references/_index.md`，会形成断链。必须自上而下逐层验证。
- **指针不可达**：归档到 Obsidian 后指针路径错误 → 信息彻底丢失。归档后必须验证目标文件存在。
- **过度拆分**：<5KB 的文件没必要拆分，否则碎片化反而降低效率。每个文件 5-15KB，职责单一。
- **一次性拆分过多**：每次压缩只拆分 1-3 个文件，避免大规模改动导致验证遗漏。多次迭代优于一次大改。

## References

- `references/_index.md` — **全局索引**（首个必读）：所有 reference 文件的前后向链接地图。
- `references/kb-audit-patterns.md` — **知识库审计模式**：空文件/重复/超大文件/跨目录重复检测模式 + 清理工作流。
- `references/20260616-context-compression-recall.md` — 会话召回地图：前次压缩工作的 Obsidian 查阅序列与"按最近的状态"修正。
- `references/20260616-context-slimdown-execution.md` — 执行笔记：class-level skill 拆分为 references/、Memory/User 归档到 Obsidian、残留关键词验证。
- `references/20260616-context-slimdown-234-pattern.md` — 234 模式：严格编号范围执行、运行时备份、skill 瘦身进 references、压缩后验证。
- `references/compression-example-history.md` — 2026-06-14 首次压缩执行全记录（含删除/合并清单）。
- `references/中文原文.md` — 完整中文原始备份，供人工查阅与回滚。
- `SOUL.md` path: `~/AppData/Local/hermes/SOUL.md`
- Skills root: `~/AppData/Local/hermes/skills/`
- PLUR storage root: `~/.plur/`
- Obsidian vault root: `D:/lk/Obsidian/Hermes/`

## 变更记录

- **v2.1.0** — PROGRESS_TICK 🎯 嵌入 Step 1→2→3 各节点（读取后/判定后/执行前/每步完成）；FIX_LOOP GATE 扩展为 4 问（含进度可见性）；修复 Step 3-5 heading 层级不一致。
- **v2.0.0** — 重构：新增高保真三原则、压缩分级表、拆分互联规范；创建 references/_index.md；所有 reference 文件补充双向链接。
