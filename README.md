# 🧠 Hermes Skills — Fishsb

> 个人自创的 [Hermes Agent](https://hermes-agent.nousresearch.com) 技能集
> *Personal Hermes Agent skills by Fishsb*

[![GitHub](https://img.shields.io/badge/GitHub-Fishsb/hermes--skills-181717?logo=github)](https://github.com/Fishsb/hermes-skills)

---

## 依赖总览 / Dependency Map

本仓库只包含**自建 skill**。部分 skill 依赖其他 Hermes skill 或外部工具，这些**不在本仓库内**，需要单独获取。

```
┌─ 本仓库内 ───────────────────────────┐
│                                       │
│  knowledge-ingestion                  │
│    ↓ 依赖 ↓                          │
│  knowledge-base-compression           │
│    ↓ 依赖 ↓                          │
│  agent-migration-backup               │
│                                       │
│  skills-git-management (无依赖)       │
└───────────────────────────────────────┘

┌─ 外部依赖（需用户自行获取） ──────────┐
│                                       │
│  lk-memory-index          ← 已删除   │
│  cron-automation-patterns ← 社区 skill│
│  Skill Factory            ← 社区 skill│
│  skill-cleaner            ← 未实现   │
│  hermes-agent-skill-authoring ← 系统  │
│  @mimo-ai/cli             ← npm 包   │
└───────────────────────────────────────┘
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
| **📦 来源** | 自建（Agent 代建） |
| **🔗 依赖** | 需以下 skill 配合（均**不在本仓库**，见下方指引）：<br>`cron-automation-patterns` — 定时触发归档<br>`Skill Factory` — skill 自动生成框架<br>`hermes-agent-skill-authoring` — Hermes 系统自带 skill |
| **⚠️ 外部指引** | `cron-automation-patterns` 与 `Skill Factory` 需从社区或自行创建。<br>`hermes-agent-skill-authoring` 随 Hermes 安装自带。 |

#### `knowledge-base-compression` — 知识库压缩（后处理）

| 项目 | 说明 |
|------|------|
| **做什么** | 扫描 Obsidian 知识库中的冗余/过长/过时的笔记，自动去重合并、压缩精简 |
| **什么时候用** | 知识库膨胀后手动触发，或每周定时维护 |
| **上下游** | <kbd>上游</kbd> 已归档的知识库笔记 → <kbd>下游</kbd> 精简后的知识库 |
| **触发词** | `压缩知识库`、`知识库去重`、`知识库维护` |
| **不做什么** | 不负责归档新内容——那是 `knowledge-ingestion` 的事 |
| **📦 来源** | 自建（作者 `lk`） |
| **🔗 依赖** | `skill-optimization` ✅ 本仓库（即 `knowledge-ingestion`）<br>`lk-memory-index` ❌ 已删除，从本仓库移除<br>`skill-cleaner` ❌ 未实现/未找到 |
| **⚠️ 外部指引** | `skill-cleaner` 暂无可用实现，不影响主体功能。<br>`lk-memory-index` 原为环境索引 skill，已归档不再维护。如缺失该依赖，压缩流程会尝试跳过相关步骤。 |

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
| **🔗 依赖** | `lk-memory-index` ❌ 已删除，从本仓库移除 |
| **⚠️ 外部指引** | 恢复流程中会调用以下外部工具（各工具独立，按需安装）：<br>`npx @mimo-ai/cli` — MimoCode CLI（需 `npm`）<br>`npx @wangshunnn/bilibili-mcp-server` — B站 MCP（可选）<br>`uvx douyin-mcp-server` — 抖音 MCP（可选）<br>`npm install -g @plur-ai/mcp` — PLUR MCP（可选）<br>参考文档中列出的恢复步骤会按需提示安装，非强制。 |

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

### 🔧 外部依赖获取指引

本仓库的部分 skill 依赖以下外部资源，需**单独获取**：

| 依赖项 | 类型 | 获取方式 |
|--------|------|---------|
| `cron-automation-patterns` | Hermes Skill | 社区创建或自行编写（暂无稳定来源） |
| `Skill Factory` | Hermes Skill | 社区创建或自行编写（暂无稳定来源） |
| `hermes-agent-skill-authoring` | Hermes Skill | Hermes 安装自带，无需单独获取 |
| `skill-cleaner` | Hermes Skill | 未实现，暂无可用版本 |
| `lk-memory-index` | Hermes Skill | 已从本仓库移除，不再维护 |
| `@mimo-ai/cli` | npm 包 | `npm install -g @mimo-ai/cli` |
| Obsidian vault | 外部系统 | 自建 Obsidian 知识库，路径可在 skill 内配置 |

---

## 相关资源

- [Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs)
- [The-Aetheris/skills](https://github.com/The-Aetheris/skills)
- [veawho/via54Skills](https://github.com/veawho/via54Skills)

---

> ⭐ 有用的话给个 Star！
