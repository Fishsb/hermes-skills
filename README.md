# 🧠 Hermes Skills — Fishsb

> 个人自创的 [Hermes Agent](https://hermes-agent.nousresearch.com) 技能集
> *Personal Hermes Agent skills by Fishsb*

[![GitHub](https://img.shields.io/badge/GitHub-Fishsb/hermes--skills-181717?logo=github)](https://github.com/Fishsb/hermes-skills)

---

## 依赖总览 / Dependency Map

本仓库包含**自建 skill** 及其主要依赖。部分依赖项仍不在本仓库内，详见下方标注。

```
┌─ 本仓库内 ─────────────────────────────────┐
│                                             │
│  cron-automation-patterns                   │
│    ↓ 定时调度                               │
│  knowledge-ingestion (skill-optimization)   │
│    ↓ 输出到知识库                            │
│  knowledge-base-compression                 │
│    ↓ 引用了部分索引                          │
│  agent-migration-backup                     │
│                                             │
│  hermes-agent-skill-authoring (skill编写规范)│
│  skills-git-management (Git 管理)           │
└─────────────────────────────────────────────┘
```

---

## Skills

### 📚 知识管理流水线

> **目标**：会话结束后自动归档 → 提炼关键信息写入知识库 → 定期压缩去重，形成知识管理闭环。
> **协作方式**：`knowledge-ingestion` 是每日自动运行的入口，`knowledge-base-compression` 是定期跑的后处理维护。

```
 daily/hourly cron                weekly/monthly
     │                               │
     ▼                               ▼
knowledge-ingestion ──→ 知识库 ──→ knowledge-base-compression
     │                               │
     ├ 扫描已结束会话                  ├ 去重合并同类笔记
     ├ 提炼关键信息                    ├ 压缩过长条目
     └ 写入 Obsidian 知识库            └ 保持检索效率
```

#### `knowledge-ingestion` — 智能归档（入口）

| 项目 | 说明 |
|------|------|
| **做什么** | 自动扫描已结束的 Hermes 会话，从中提炼可复用的知识/模板/规则，写入 Obsidian 知识库 |
| **什么时候用** | 每天/每小时自动运行（Cron 定时），也可手动触发「智能归档」 |
| **上下游** | <kbd>上游</kbd> 任何已结束的 Hermes 会话 → <kbd>下游</kbd> 知识库存档笔记 |
| **触发词** | `智能归档`、`整理到知识库` |
| **不做什么** | 不做去重和压缩——那是 `knowledge-base-compression` 的事 |
| **📦 来源** | 自建（Agent 代建，内部 name: `skill-optimization`） |
| **🔗 依赖** | 见下方⬇️ |

**依赖详情：**

| 依赖项 | 仓库状态 | 说明 |
|--------|---------|------|
| `cron-automation-patterns` | ✅ 本仓库内 | `devops/cron-automation-patterns/` — Hermes Cron 调度编排层，定义定时触发模式（每日/每小时智能归档）、任务生命周期管理（创建→暂停→清理）。<br>**来源**：本地自建（author: Hermes Agent），无外部社区仓库。 |
| `Skill Factory` | ❌ 不存在社区仓库<br>✅ 方法论已合并到本仓库 | **实现原理**：归档会话 Skill 优化扫描框架。在 scan 阶段同步检查已使用的 skills 与实际工作流的匹配度，发现不匹配项时生成待确认的补丁建议随推送展示给用户。核心方法论已合并入本 skill 的 `references/skill-factory-methodology.md`。<br>**效果**：不安装不影响归档主体功能，只是 scan 阶段不再输出技能优化建议。 |
| `hermes-agent-skill-authoring` | ✅ 本仓库内 | `software-development/hermes-agent-skill-authoring/` — Skill 编写规范指南，提供 SKILL.md frontmatter 格式标准、命名规范、最佳实践。<br>**来源**：本地自建（author: Hermes Agent），无外部社区仓库。 |

#### `knowledge-base-compression` — 知识库压缩（后处理）

| 项目 | 说明 |
|------|------|
| **做什么** | 扫描 Obsidian 知识库中的冗余/过长/过时的笔记，自动去重合并、压缩精简 |
| **什么时候用** | 知识库膨胀后手动触发，或每周定时维护 |
| **上下游** | <kbd>上游</kbd> 已归档的知识库笔记 → <kbd>下游</kbd> 精简后的知识库 |
| **触发词** | `压缩知识库`、`知识库去重`、`知识库维护` |
| **不做什么** | 不负责归档新内容——那是 `knowledge-ingestion` 的事 |
| **📦 来源** | 自建（作者 `lk`） |
| **🔗 依赖** | 见下方⬇️ |

**依赖详情：**

| 依赖项 | 仓库状态 | 说明 |
|--------|---------|------|
| `skill-optimization` | ✅ 本仓库内 | 即本仓库的 `knowledge-ingestion`。两者配合形成"写入 → 压缩"流水线。 |
| `lk-memory-index` | ❌ 已从本仓库删除 | **实现原理**：极简（~2KB）环境指针索引 skill，记录 Hermes 关键路径、Provider 配置、工具集位置、知识库挂载点。压缩流程加载它以快速定位环境资源，避免硬编码。<br>**效果**：不安装时，压缩流程会尝试跳过相关索引步骤，改用路径猜测或硬编码替代，不影响核心压缩功能。 |
| `skill-cleaner` | ❌ 不存在社区仓库 | **实现原理**：预期功能——扫描 skills 目录中的废弃/重复/长时间未使用的 skill，生成清理建议或自动归档到备份目录。<br>**效果**：暂无可用实现。不安装不影响 knowledge-base-compression 的主体压缩功能。如后续社区出现，推荐配合使用。 |

> 💡 **两者配合**：`knowledge-ingestion` 不断写入新内容 → 知识库逐渐膨胀 → `knowledge-base-compression` 定期瘦身。缺一不可。

---

### 🔄 环境运维

> **目标**：一键备份当前 Hermes 环境，换电脑/重装后快速恢复。
> **协作方式**：单 skill 完成全流程，从备份到恢复闭环。

```
 ┌─备份─────────────────┐     ┌─恢复──────────────────┐
 │ agent-migration-backup│────→│ agent-migration-backup│
 │   → 配置/skills/脚本   │     │   → 对照 manifest 逐项修复│
 │   → Cron/MCP/Profile  │     │   → 自检验证           │
 │   → PLUR/记忆/规则    │     │   → 输出恢复报告        │
 │   → 输出到知识库归档   │     │                        │
 └──────────────────────┘     └──────────────────────┘
```

#### `agent-migration-backup` — 环境迁移 + 备份恢复

| 项目 | 说明 |
|------|------|
| **做什么** | Hermes 环境备份与迁移：备份配置、Skills、脚本、Cron、MCP、Profiles、PLUR 记忆、交互规则等；换电脑后对照 manifest 逐项自动恢复并自检 |
| **什么时候用** | agent环境备份 / agent环境迁移 / 全量备份 |
| **工作模式** | <kbd>模式A</kbd> 本机备份 → <kbd>模式B</kbd> 新机恢复 → <kbd>模式C</kbd> Profile 同步 → <kbd>模式D</kbd> 清理残留 Provider |
| **触发词** | `agent环境备份`、`agent环境迁移`、`全量备份` |
| **依赖** | Obsidian 知识库中 `wiki/配置/Hermes迁移/` 目录存放备份基准 |
| **注意事项** | 已合并原 `hermes-env-backup` 功能，含备份清单速查表和简化恢复脚本 |
| **📦 来源** | 自建（Agent 代建） |
| **🔗 依赖** | `lk-memory-index` ❌ 已从本仓库删除（见上方说明） |
| **⚠️ 外部工具** | 恢复流程中会调用以下外部工具（各工具独立，按需安装）：<br>`npx @mimo-ai/cli` — MimoCode CLI（需 `npm`）<br>`npx @wangshunnn/bilibili-mcp-server` — B站 MCP（可选）<br>`uvx douyin-mcp-server` — 抖音 MCP（可选）<br>`npm install -g @plur-ai/mcp` — PLUR MCP（可选）<br>参考文档中列出的恢复步骤会按需提示安装，非强制。 |

---

### ⚙️ 工具链

> **目标**：管理技能仓库本身 + 快速定位 Hermes 环境配置。
> **协作方式**：`skills-git-management` 管整个技能目录的版本控制。

#### `skills-git-management` — 技能版本管理

| 项目 | 说明 |
|------|------|
| **做什么** | 用 Git/GitHub 对 Hermes Skills 做版本管理：初始化仓库、配置 SSH、选择性跟踪、增量同步、灾难恢复 |
| **什么时候用** | 首次把 skills 纳入 Git 管理 / 换电脑恢复 / 日常同步 / 分享技能到社区 |
| **触发词** | `推送到github`、`同步到github` |
| **不做什么** | 不备份 API Key 等敏感信息（那是 `agent-migration-backup` 的事） |
| **📦 来源** | 自建（Agent 代建） |
| **🔗 依赖** | 无（纯 Git 操作，仅需本机已安装 git） |

---

## 📥 安装

```bash
cd ~/AppData/Local/hermes/skills   # Windows
git clone git@github.com:Fishsb/hermes-skills.git
# 或按流水线只克隆需要的 skill:
git clone --depth 1 --filter=blob:none --sparse git@github.com:Fishsb/hermes-skills.git
cd hermes-skills
git sparse-checkout set knowledge-ingestion knowledge-base-compression
```

每个 skill 是独立目录，内含 `SKILL.md` + 可选 `references/`。复制到 Hermes skills 路径下即可自动加载。

### 🔧 外部依赖详解

本仓库以下依赖项**不在本仓库内**，经查询 GitHub **目前社区无公开仓库**。以下是每个依赖的**实现原理和影响说明**，方便你自行决定是否需要或自行创建替代：

#### `Skill Factory`
- **原理**：归档会话 Skill 优化扫描框架。在智能归档的 scan 阶段同步执行：扫描本地 skills 列表 → 检查各 skill 的 `SKILL.md` 与实际工作流的匹配度 → 对不匹配项生成待确认的补丁建议 → 随归档推送展示给用户审批。
- **在本仓库中的用途**：`knowledge-ingestion` 的 scan 阶段会调用 Skill Factory 方法生成技能优化建议。
- **缺失影响**：归档主体功能不受影响，只是 scan 阶段不会输出技能优化建议。核心方法论已合并到本仓库 `knowledge-ingestion/references/skill-factory-methodology.md`，可参考实现。
- **自行实现要点**：在 scan 阶段添加 skills 目录扫描 → 对比 frontmatter 的 `related_skills` 与技能实际内容 → 输出差异报告。

#### `skill-cleaner`
- **原理**：（待实现）预期功能为扫描 skills 目录，检测以下冗余：从未被加载的 skill、功能重叠的 skill、超过 N 天未更新的 skill、不再满足前置条件的 skill。生成清理建议列表或自动归档到备份目录。
- **在本仓库中的用途**：`knowledge-base-compression` 将其列为关联 skill，期望在压缩知识库的同时也清理冗余技能。
- **缺失影响**：不影响 knowledge-base-compression 的主体压缩功能。如需此功能，可手动删除不需要的 skill 目录。
- **自行实现要点**：遍历 skills 目录 → 检查 .usage.json 中的加载记录 → 比对 SKILL.md 的 triggers 关键词是否在当前配置中被调用 → 输出建议列表。

#### `lk-memory-index`
- **原理**：极简（~2KB）环境指针索引。记录了：Hermes 关键路径（config.yaml / skills / memory 位置）、Provider 列表及模型映射、工具集挂载点、知识库路径等。Agent 在运行时加载它即可获得环境全貌，无需长篇幅内存笔记。
- **在本仓库中的用途**：`agent-migration-backup` 用它定位备份目标路径；`knowledge-base-compression` 用它快速找到知识库和 memory 文件。
- **缺失影响**：备份和压缩流程会尝试跳过索引环节，使用硬编码路径或路径猜测替代。核心功能完整但可能遗漏非标准路径的自定义配置。
- **自行实现要点**：创建一个 SKILL.md，内容为当前环境的路径索引清单（JSON/YAML 格式），在 `references/` 中维护关键路径列表。首次运行备份或压缩时自动生成。

#### `@mimo-ai/cli`（外部 npm 包）
- **获取方式**：`npm install -g @mimo-ai/cli`
- **用途**：MimoCode CLI 执行引擎，`agent-migration-backup` 的恢复流程中可选调用。

---

## 相关资源

- [Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs)
- [The-Aetheris/skills](https://github.com/The-Aetheris/skills)
- [veawho/via54Skills](https://github.com/veawho/via54Skills)

---

> ⭐ 有用的话给个 Star！
