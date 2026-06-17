---
name: skill-optimization
description: 智能归档流程的内联优化层 — ① scan 阶段同步扫描已用 skills 生成优化建议随推送一起展示；② archive 阶段在 smart-archive.py 内联提取知识写入 Wiki 后删会话
version: 4.0.0
author: Hermes Agent
license: MIT
triggers:
  - 智能归档
  - 整理到知识库
related_skills:
  - cron-automation-patterns
  - Skill Factory
  - hermes-agent-skill-authoring
metadata:
  hermes:
    tags: [archive, skill-factory, obsidian, note-taking, optimization]
    related_skills: [cron-automation-patterns, Skill Factory, hermes-agent-skill-authoring]
---

# 智能归档优化层 (Skill Optimization)

## Overview

智能归档流程的内联优化层，分两端嵌入原有流程：

| 端 | 嵌入位置 | 职责 |
|:--|:---------|:-----|
| **① Skill Factory 扫描端** | scan 时同步运行 | 扫描可用 skills + 检查待确认建议 → 随推送一起展示 |
| **② 知识提取端** | archive 时由 `smart-archive.py` 内联执行 | 提取会话有用信息 → 类别映射 → 写入 Wiki |

提取已在 `smart-archive.py` 中内联：`export → extract_knowledge_to_wiki() → delete_session`，无需独立摄入步骤。

> **与 cron-automation-patterns 的关系**: `cron-automation-patterns` 是编排层（外视图），`skill-optimization` 是两端的内联执行文档（内视图）。

---

## ① Skill Factory 扫描端 (Scan-Phase)

### 触发条件

- `smart-archive.py scan` 执行后，Agent 在整理推送内容时调用
- 或用户主动询问"有 skill 优化建议吗"

### 执行流程

```text
smart-archive.py scan
  │
  ├─ 扫描闲置/活跃会话 (原有逻辑)
  ├─ 清理未命名会话 + 旧 cron 历史 (原有逻辑)
  │
  └─ ⏵ Skill Factory 同步扫描
        │
        ├─ 1. 读取 skill-suggestions.json 统计待确认建议
        ├─ 2. 扫描 skills 列表，标记已知待优化项
        ├─ 3. 检查最近归档会话中的已用 skills 模式
        └─ 4. 合并输出到推送内容
```

### 推送内容格式

在原有会话列表之后追加：

```
🧩 Skill Factory 优化建议

  [1] SF-20260615-001 · mimocode-cli · pending_confirmation
      MiMo skill 的触发词可增加排除条件
```

用户交互：
- `0` — 归档全部闲置 + **保持建议待确认**
- `0 + approve 1` — 归档全部 + 确认第 1 条优化建议
- `approve 1 2` — 仅确认建议，不做归档

---

## ② 知识提取端 (Archive-Phase)

已内联到 `smart-archive.py` 的 `cmd_archive()` 流程中。

### 执行流程

```text
smart-archive.py archive <N|N+|0>
  │
  ├─ export_session(sid)        ← 导出会话 JSON
  ├─ extract_knowledge_to_wiki() ← 内联提取 + 写 Wiki 笔记
  │     ├─ 按类别映射确定目标目录
  │     ├─ 提取关键片段（用户问题、工具调用、配置/命令）
  │     ├─ 写 Wiki 文件（frontmatter + 来源标记）
  │     └─ 返回文件路径
  └─ delete_session(sid)        ← 删除 Hermes 原会话
```

`extract_knowledge_to_wiki()` 函数在 `smart-archive.py` 中实现，输出格式：

```markdown
---
title: "<会话标题>"
source: "<session_id>"
model: "<model>"
archived: "<date>"
tags: ["<分类>", "archived", "auto-extracted"]
---

# <标题>

[源于会话: <标题>]

## 会话概览
- **模型**: ...
- **消息数**: ...
- **用户提问**: N 轮

## 关键内容
（第一条用户问题）

## 使用的工具
- [tool] skill_view
- [tool] terminal

## 配置/命令片段
- `npx hermes config set provider ...`
```

### 注意事项

- **不消耗 API token** — 纯本地 Python 提取，基于关键词和消息结构
- **不覆盖已有笔记** — 同名文件自动追加 `_archived_<date>.md` 后缀
- **去冗余** — 只提取可复用知识，不搬运原始对话
- **来源标记** — 每篇笔记开头 `[源于会话: {title}]`
- **失败容忍** — 提取失败不阻塞删除会话；错误信息打印到 stderr

---

## 类别映射规则

> 详细组织规范参见 `skill_view(name='skill-optimization', file_path='references/kb-organization-rules.md')`

### 目录选择矩阵

| 会话主题 | 目标目录 | 反例（不放这里） |
|---------|---------|----------------|
| Hermes 配置、工具排障、API 配置、插件、工具安装 | `wiki/工具/` | 工作流方法、环境安装 |
| WSL、系统维护、C盘清理、环境迁移 | `wiki/环境/` | 具体工具的配置细节 |
| 自动化工作流、文档处理、审计流程、工程方法论 | `wiki/工作流/` | 具体工具排障 |
| 技术调研、GitHub 项目、研究报告、竞品分析、生态推荐 | `wiki/项目/` | 归档的会话原始提取 |
| Agent 规则、Profile 配置、记忆策略、交互规范、索引、备份 | `wiki/配置/` | 工具层面的配置细节 |
| Skill 备份、开发记录 | `wiki/skill/` | 非 skill 的配置备份 |
| 通用概念、跨领域知识点 | `wiki/概念/` | 操作指南 |
| 医药健康知识 | `wiki/医学/` | 非医学内容 |
| 所有 `auto-extracted` 会话提取 | `wiki/记忆/会话归档/` | 内容目录（禁止写入） |

### 关键约束

- **`auto-extracted` 文件必须写入 `记忆/会话归档/`**，禁止进入任何内容目录
- 移入 `记忆/会话归档/` 后 tags 中的旧分类名改为 `"会话归档"`
- 非 auto-extracted 的 `superseded`/`closed` 文件按主题归入对应内容目录

---

## 归档交互约定

| 用户输入 | 动作 |
|---------|------|
| `0` | 归档全部闲置会话（提取→写 Wiki→删会话） |
| `1` / `1 2` | 归档指定编号 |
| `2+` | 从第 N 个到末尾归档 |
| `approve <suggestion_id>` | 仅确认优化建议，不做归档 |
| `0 + approve 1` | 归档全部 + 确认第 1 条优化建议 |

---

## 旧 staging 兼容

`ingest-queue.json` 中已有的旧暂存数据仍保留，`ingest-status` 命令可查看。
新归档的会话不再写入 staging 队列。

---

## Verification Checklist

- [ ] scan 输出同时包含会话列表 + Skill Factory 建议状态
- [ ] archive 后自动内联提取知识并写 Wiki，不依赖独立步骤
- [ ] 知识提取写入 Wiki 的文件有完整 frontmatter + 来源标记
- [ ] 提取失败时不阻塞删除 Hermes 原会话（但会输出错误信息）
- [ ] Skill Factory 建议只到 `pending_confirmation`，不自动 patch
- [ ] 旧 staging 队列不动，新归档不走 staging
- [ ] **归档后自检** — 按需加载 `references/kb-organization-rules.md` 验证：
  - [ ] 文件写入正确目录（auto-extracted 不进内容目录）
  - [ ] 文件名与内容/文件夹匹配
  - [ ] 移动过的文件标签已同步更新
  - [ ] 全库互链索引中的旧路径已修复
  - [ ] 所在目录的 📂 索引不需要加 `file.folder` 过滤
