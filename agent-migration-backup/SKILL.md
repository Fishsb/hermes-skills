---
name: agent-migration-backup
description: "Hermes Agent 环境备份与迁移 — agent环境备份、agent环境迁移、全量备份。覆盖配置/技能/脚本/cron/MCP/PLUR/记忆/规则一键迁移。"
version: 3.2.0
author: Hermes Agent
triggers:
  - agent环境备份
  - agent环境迁移
  - 全量备份
paths:
  latest_backup_root: 'D:/lk/Obsidian/Hermes/wiki/配置/Hermes迁移'
  latest_restore_readme: 'D:/lk/Obsidian/Hermes/wiki/配置/Hermes迁移/README.md'
  latest_runtime_backup: 'D:/lk/Obsidian/Hermes/wiki/配置/Hermes迁移/hermes-runtime-no-sessions'
  latest_restore_notes: 'D:/lk/Obsidian/Hermes/wiki/配置/Hermes迁移/restore-notes'
  backup_main_legacy: 'D:/lk/Obsidian/Hermes/wiki/配置/Hermes完整环境备份_20260613.md'
  restore_guide_legacy: 'D:/lk/Obsidian/Hermes/wiki/配置/Hermes一键恢复指南_20260613.md'
  restore_script_legacy: 'D:/lk/Obsidian/Hermes/wiki/配置/restore-hermes.sh'
  scripts_backup_legacy: 'D:/lk/Obsidian/Hermes/wiki/配置/Hermes脚本备份_20260613.md'
related_skills: [lk-memory-index]
---

# Hermes Agent 迁移备份

> 触发词：**agent环境备份** / **agent环境迁移** / **全量备份** — 当用户说出其中任一触发词时，执行完整环境备份或引导恢复流程。

---

## 工作流程

## 最新恢复基准（2026-06-17 起）

**所有知识库中的 Hermes 备份/恢复，一律以 `wiki/配置/Hermes迁移/README.md` 与 `wiki/配置/Hermes迁移/hermes-runtime-no-sessions/` 为最新基准。**

- `Hermes完整环境备份_20260613.md`、`Hermes当前工具配置与记忆同步_20260614.md`、`Hermes一键恢复指南_20260613.md`、`restore-hermes.sh` 只作为 legacy 参考。
- 恢复目标：恢复全部配置/使用习惯/Skills/记忆/脚本/Cron/MCP/Profiles/状态，但不恢复历史会话。
- 最新备份已排除：`sessions/`、`transcripts/`、`request_dump_*.json`。
- 最新自检清单在：`wiki/配置/Hermes迁移/restore-notes/`。

---

## 📦 备份清单速查

> 以下清单合并自 `hermes-env-backup`，一目了然。

### ✅ 要备份的内容

| 组件 | 捕获方式 |
|------|---------|
| config.yaml | `cat` 全文嵌入文档 |
| .env 变量名 | `grep -v "^#" \| cut -d= -f1`（仅名字，不含 key） |
| Skills 列表 | `skills_list()` |
| 自定义 SKILL.md | `skill_view(name)` |
| 脚本源码 | `read_file()` 完整 .py/.sh |
| MCP 服务器 | `hermes mcp list` |
| Cron 任务 | `cronjob(action='list')` |
| PLUR engrams | `plur_recall_hybrid` |
| Memory 条目 | `memory()` 读取 |
| Profile 配置 | `cat profiles/<name>/config.yaml` |
| 代理节点 | `skill_view('proxy-node')` |
| WorkBuddy 配置 | `skill_view('workbuddy-vision-integration')` |
| Shell 别名 | 搜索 .bashrc 或直接记录命令 |
| 知识库结构 | 目录布局、标签、索引 |

### ❌ 不备份的内容

| 项目 | 原因 |
|------|------|
| API Key / 密钥 | 安全——只记变量名 |
| state.db (~37MB) | 运行时数据，太大且无恢复价值 |
| kanban.db (~114KB) | 可选——会话状态 |
| WorkBuddy 登录 | OAuth——需重新认证 |
| Chrome 扩展 | 手动步骤 |
| Skill reference 文件 | 从 hub 重装即可 |
| _temp_session/ 原始数据 | 临时——无恢复价值 |

---

## ⚡ 简化恢复脚本（手动执行）

> 如不想走 Agent 自动恢复，可直接按以下 bash 命令手动操作。保留了 `hermes-env-backup` 的原始步骤。

```bash
# 1. 安装 Hermes Desktop
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash

# 2. 复制 config.yaml
cp backup/config.yaml ~/AppData/Local/hermes/config.yaml

# 3. 设置 .env（手动填写 API Key）
# 必需: DEEPSEEK_API_KEY, HERMEST0_API_KEY, DASHSCOPE_API_KEY
# 可选: OPENROUTER_API_KEY, GITHUB_TOKEN, WEIXIN_*

# 4. 复制 Skills
cp -r backup/skills/* ~/AppData/Local/hermes/skills/

# 5. 复制脚本
cp backup/scripts/* ~/AppData/Local/hermes/scripts/
chmod +x ~/AppData/Local/hermes/scripts/*.{sh,py}

# 6. 设置模型
hermes config set model.default deepseek-v4-flash
hermes config set model.provider deepseek

# 7. 安装 MCP 服务器
# agent-browser-mcp: git clone + pip install
# bilibili: hermes mcp add --command "npx -y @wangshunnn/bilibili-mcp-server"
# douyin: hermes mcp add --command "uvx douyin-mcp-server"
# searxng: hermes mcp add --command "uvx searxng --instance-url https://searx.party"
# plur: npm install -g @plur-ai/mcp; hermes mcp add --command "plur-mcp"

# 8. 安装 PLUR
npm install -g @plur-ai/mcp
cp -r backup/.plur ~/.plur

# 9. 安装 MimoCode
npx @mimo-ai/cli

# 10. 创建 Profiles
hermes profile create dispatcher
hermes profile create worker

# 11. 设置批准模式
hermes config set approvals.mode off

# 12. 设置时区
hermes config set timezone UTC+8

# 13. 重建 Cron 任务
hermes cron create --name "🧠 智能归档" --schedule "0 8 * * *" \
  --script "smart-archive-desktop.sh" --no_agent \
  --toolsets terminal --deliver local

# 14. 复制 Obsidian 知识库
cp -r backup/Obsidian/Hermes D:/lk/Obsidian/Hermes
```

---

### 模式 A：备份（在当前电脑）

当用户说「全量备份」且**在当前电脑**时：

0. **先运行 `hermes backup -q`** — 生成 quick snapshot 到 `state-snapshots/`，作为安全网。如果后续手动备份出错，可从 snapshot 恢复。
1. 将 `C:/Users/lk/AppData/Local/hermes/` 同步到 `wiki/配置/Hermes迁移/hermes-runtime-no-sessions/`
2. 排除历史会话：`sessions/`、`transcripts/`、`request_dump_*.json`
3. 导出 restore-notes：version/config/tools/skills/mcp/cron/profile/gateway/config-summary/env names/tree/size/exclusion/hash
4. **更新 `wiki/配置/Hermes迁移/restore-manifest.md`** — 对照当前环境重新生成恢复状态清单（config 关键字段、.env 变量名、MEMORY.md 内容、skills 列表、cron 配置等）
5. 更新 `wiki/配置/Hermes迁移/README.md` 的时间戳、自检结果和恢复说明
6. 如知识库/skill 中有旧指针，统一改为指向 `Hermes迁移/README.md`
7. 输出备份完成摘要和自检结果

### 模式 B：恢复（在新电脑）

当用户说「agent环境迁移」且**在新电脑**时：

**核心原则：不直接操作文件，而是告知 Agent 目标状态，让 Agent 自行用最可靠的方式达成。**

1. 读取 `wiki/配置/Hermes迁移/README.md` 了解恢复概览
2. 安装 Hermes Desktop 并启动一次生成默认 runtime，然后完全退出
3. 将 `hermes-runtime-no-sessions/` 复制到 `C:/Users/lk/AppData/Local/hermes/`
4. **让 Agent 读取 `wiki/配置/Hermes迁移/restore-manifest.md`，逐项执行恢复检查**

> restore-manifest.md 包含备份时的完整状态快照。Agent 对照当前环境，发现差异时自行选择最可靠的方式修复（Python/YAML 操作、patch、write_file 等），不依赖固定的 cp/sed 命令。

5. 恢复后立即备份 .env 防止被 update 清除：`cp .env .env.post-restore`
6. 验证：`hermes config check` + `hermes skills list` + `hermes chat -q "你好"`

**恢复检查项（Agent 逐项对照 restore-manifest.md）：**

| 检查项 | Agent 做什么 | 不做什么 |
|--------|-------------|---------|
| config.yaml 关键字段 | 读 manifest 目标值 → Python yaml.safe_load 对比 → patch 修复 | 不用 sed/正则替换 |
| .env 变量名 | 读 manifest 变量列表 → 对比当前 .env → 补充缺失行 | 不用 grep 管道 |
| MEMORY.md | 读 manifest 内容 → 对比当前文件 → write_file 覆盖 | 不用 cat/echo 重定向 |
| SOUL.md | 读 manifest 关键规则 → 对比当前文件 → patch 修复 | 不用整文件替换 |
| skills/ | 读 manifest 列表 → `find skills -name SKILL.md` 对比数量 | 不用 cp -ru 批量拷贝 |
| bin/ | 读 manifest 文件列表 → `ls bin/` 对比 → 从备份复制缺失文件 | 不用 diff 命令 |
| scripts/ | 读 manifest 文件列表 → `ls scripts/` 对比 → 从备份复制 | 不用 rsync |
| cron | 读 manifest 任务配置 → `hermes cron list` 对比 | 不用 json 手动编辑 |
| profiles/ | 读 manifest profile 列表 → `ls profiles/` 对比 | 不用 mkdir 手动创建 |
| MCP | 读 manifest mcp_servers → `hermes mcp list` 对比 → 用 hermes mcp add 修复 | 不用 yaml 手动写入 |

### 模式 B2：Desktop 覆盖重装后的"最近有效状态"恢复

当用户覆盖重装 Hermes Desktop 后，要求恢复"配置/工具/插件/记忆/规则"等，但不恢复历史会话时：

**核心原则：对照 restore-manifest.md 的目标状态，逐项检查并修复差异。**

1. 读取 `wiki/配置/Hermes迁移/restore-manifest.md`，获取备份时的目标状态
2. 逐项检查以下配置，发现差异时修复：
   - `model` — manifest 中的 provider/default 值
   - `delegation` — manifest 中的 model/provider/max_iterations 等
   - `memory` — manifest 中的 enabled/char_limit/nudge 等
   - `compression` — manifest 中的 threshold/target_ratio 等
   - `approvals` — manifest 中的 mode/timeout
   - `logging` — manifest 中的 level/max_size_mb
   - `mcp_servers` — manifest 中的服务器列表和配置
   - `platform_toolsets.cli` — manifest 中的工具列表
3. 修复方式：Agent 用自己最可靠的方式（Python yaml + patch）修改 config.yaml，不要用 sed/正则
4. 验证：`hermes config check` + `hermes skills list` + `hermes mcp list`
5. 提醒用户：toolset/MCP 配置变更通常要新会话或 `/reset` 后才进入当前 tool schema

详见：`references/desktop-overwrite-restore-recent-state.md`。

### 模式 B3：Desktop 覆盖安装后的 Skills / Tools / Config 审计

当用户覆盖安装/重装 Hermes Desktop 后，询问 skills 是否完整、是否多出以前卸载的自带 skill、工具/MCP/Profile/Cron/配置是否需要恢复时：

**核心原则：对照 restore-manifest.md 的目标状态，逐项审计差异。**

1. 读取 `wiki/配置/Hermes迁移/restore-manifest.md`，获取备份时的目标状态
2. 逐项审计：
   - **Skills**: manifest 中 59 个 skill → `hermes skills list` 对比数量和名称 → 缺失的从备份复制
   - **Config**: manifest 中 model/delegation/memory/compression → Python yaml 读取当前 config 对比 → 差异修复
   - **MCP**: manifest 中 mcp_servers → `hermes mcp list` 对比 → 缺失的用 `hermes mcp add` 添加
   - **Cron**: manifest 中智能归档配置 → `hermes cron list` 对比
   - **Profiles**: manifest 中 4 个 profile → `ls profiles/` 对比
   - **.env**: manifest 中变量名列表 → 对比当前 .env 变量名（不输出 secret 值）
   - **bin/**: manifest 中文件列表 → `ls bin/` 对比
   - **scripts/**: manifest 中文件列表 → `ls scripts/` 对比
3. 输出审计报告：匹配/缺失/多余 三项统计

详见：`references/desktop-overwrite-skill-config-audit.md`。

### 模式 C：同机 Profile 同步

当用户说「子Agent配置同步」、「Profile同步」、「把default的工具/MCP/Skills同步到worker」时：

**核心原则：告知 Agent 要同步什么，让 Agent 用 hermes CLI 或 Python 完成，不直接 cp 文件。**

保持 default profile 为**源**，同步以下内容到目标 Profile：

| 同步项 | 目标状态 | Agent 修复方式 |
|--------|---------|---------------|
| MCP 服务器 | 目标 Profile 的 mcp_servers 与 default 一致 | `hermes --profile <name> mcp add ...` |
| 授权配置 | `approvals.mode: off` | `hermes --profile <name> config set approvals.mode off` |
| Skills | 目标 Profile 的 skills 数量与 default 一致 | 对比 `hermes skills list` 数量，缺失的从 default 复制 |
| 自定义 Provider | 目标 Profile 的 custom_providers 包含 default 的 provider | Python yaml 读取+合并 |

**同步前检查：**
```bash
hermes profile list                              # 列出所有 Profile
hermes --profile <name> mcp list                 # 各 Profile 的 MCP
hermes --profile <name> skills list | wc -l      # 各 Profile 的 skill 数量
```

**同步后验证：**
```bash
hermes --profile <name> mcp list                 # MCP 一致
hermes --profile <name> skills list | wc -l      # skill 数量一致
```

⚠️ **MCP args 格式坑：** `--args` 需每个参数独立传字符串，不要逗号拼接。

---

### 模式 D：删除自定义提供商并清理全痕迹

当需要移除一个 Hermes 自定义 provider（如 `custom:hermest1`）时：

**核心原则：告知 Agent 要清理什么，让 Agent 自行扫描并清理，不要给出具体的 sed/grep 命令。**

Agent 需要扫描并清理以下所有位置中的 TARGET_PROVIDER 和 TARGET_API_KEY 残留：

| # | 位置 | 清理什么 |
|---|------|---------|
| 1 | `config.yaml` | `custom_providers` 中该 provider 的整个区块 |
| 2 | `.env` | `TARGET_API_KEY` 行 |
| 3 | `skills/*/SKILL.md` | env 变量列表行 + 表格中的引用行 |
| 4 | `memories/MEMORY.md` | 包含旧 provider 的引用 |
| 5 | `cache/` | 含 `declare -x TARGET_VAR` 的文件 |
| 6 | `backups/` 和 `state-snapshots/` | .env + config.yaml 中的对应条目 |

**清理后验证：**
Agent 用 `search_files` 搜索 `TARGET_VAR` / `TARGET_PROVIDER` 关键词，确认无残留：
- 搜索范围：`~/AppData/Local/hermes/`
- 排除：`backups/`、`state-snapshots/`
- 预期结果：0 匹配

> **Windows 路径注意：** config.yaml/.env 不能用 patch/write_file 修改，必须走 terminal + Python。详见 `references/windows-config-editing.md`。

---

## 备份文件清单

### 最新基准（2026-06-17 起）

知识库中所有 Hermes 备份/恢复入口，**一律以最新迁移目录为准**：

| 文件/目录 | 内容 |
|------|------|
| `Hermes迁移/README.md` | 最新恢复说明、恢复顺序、外部依赖、自检结果 |
| `Hermes迁移/restore-manifest.md` | **恢复状态清单** — Agent 读取后逐项对照自动修复 |
| `Hermes迁移/hermes-runtime-no-sessions/` | 最新 Hermes runtime 原始备份；排除 `sessions/`、`transcripts/`、`request_dump_*.json`、`state.db`、`hermes-agent/` |
| `Hermes迁移/restore-notes/` | 版本、config、tools、skills、MCP、cron、profile、gateway、env变量名、目录树、大小、排除项、哈希等验证清单 |
| `Hermes迁移/backup-timestamp.txt` | 最新备份时间戳 |

### Legacy 参考文件

以下旧文件只作为历史参考；恢复时不要作为最新基准：

| 文件 | 内容 |
|------|------|
| `Hermes完整环境备份_20260613.md` | 2026-06-13 全量备份基线；legacy |
| `Hermes当前工具配置与记忆同步_20260614.md` | 2026-06-14 工具/MCP/Profile/Cron/Scripts/Skills/记忆快照；legacy |
| `记忆备份-MEMORY.md` | 旧 `AppData/Local/hermes/memories/MEMORY.md` 快照 |
| `记忆备份-USER.md` | 旧 `AppData/Local/hermes/memories/USER.md` 快照 |
| `Hermes一键恢复指南_20260613.md` | 18步旧恢复操作指南；legacy |
| `restore-hermes.sh` | 旧自动恢复脚本；legacy |
| `Hermes脚本备份_20260613.md` | 旧自定义脚本源码备份；legacy |

### 2026-06-14 Legacy 环境差异覆盖

以下差异只用于理解旧 `20260613 → 20260614` 恢复链路；**2026-06-16 起最新恢复以 `Hermes迁移/README.md` 为准**。如必须回看旧链路，才先按 `Hermes完整环境备份_20260613.md` 恢复基线，再按 `Hermes当前工具配置与记忆同步_20260614.md` 覆盖以下差异：

| 旧备份项 | 当前应改为 |
|---|---|
| 主模型 `deepseek/deepseek-v4-flash` | `custom:hermest0 / gpt-5.4` |
| 多 profile：default/dispatcher/worker/worker2/worker3 | 只保留 `default` |
| token-savior 69 tools/full profile | `TOKEN_SAVIOR_PROFILE=optimized`，实测 15 tools |
| `uumit` enabled | `uumit` disabled |
| Weixin enabled / 需 WEIXIN_TOKEN | Weixin disabled；当前生效 `.env` 不含 `WEIXIN_TOKEN` |
| Windows Headroom proxy | stopped，不接入主链路 |
| logging `max_size_mb=5` | `max_size_mb=50`, `backup_count=5` |
| WorkBuddy/dispatcher 通用调度 | WorkBuddy 只用于 vision/image understanding；通用任务 MiMo-first |
| MiMo 固定 5-7/5-8 | 1+ 按任务拆分决定，拆分尽量详细 |

---

## 恢复流程（Agent 读取 restore-manifest.md 后执行）

| # | 检查项 | Agent 做什么 | 不做什么 |
|---|--------|-------------|---------|
| 1 | config.yaml | Python yaml 读取对比 manifest 目标值 → patch 修复差异字段 | 不用 sed/正则 |
| 2 | .env 变量名 | 对比 manifest 变量列表 → Python 补充缺失行 | 不用 grep 管道 |
| 3 | MEMORY.md | 对比 manifest 内容 → write_file 覆盖 | 不用 cat/echo |
| 4 | SOUL.md | 对比 manifest 关键规则 → patch 修复 | 不用整文件替换 |
| 5 | skills/ | `find` 对比数量 → 缺失的从备份复制 | 不用 cp -ru |
| 6 | bin/ | `ls` 对比文件列表 → 缺失的从备份复制 | 不用 diff |
| 7 | scripts/ | `ls` 对比文件列表 → 缺失的从备份复制 | 不用 rsync |
| 8 | cron | `hermes cron list` 对比 manifest 任务 | 不用 json 编辑 |
| 9 | profiles/ | `ls` 对比 manifest profile 列表 | 不用 mkdir |
| 10 | MCP | `hermes mcp list` 对比 → `hermes mcp add` 补充 | 不用 yaml 写入 |
| 11 | bundled skill | `ls skills/` 对比 manifest → 删除多余目录 | — |
| 12 | .env 保护 | `cp .env .env.post-restore` | — |
| 13 | 最终验证 | `hermes config check` + `hermes chat -q "你好"` | — |

---

### Windows 路径写入：优先用正斜杠或原始读回验证

在 skill frontmatter、知识库 Markdown、YAML 表格中写 Windows 路径时，优先使用 `C:/Users/lk/...` / `D:/lk/...` 正斜杠形式；必须写反斜杠时，写后立刻读回 frontmatter/文本，确认没有把 `\r`、`\n` 等序列变成实际换行或回车。若 `patch` 的 old_string 因路径转义/换行无法唯一匹配，改用最小范围的结构化修复，并重新解析 YAML 验证。

### 🚩 Post-restore: .env 被 hermes update 静默清除

Hermes `update`/`config migrate` 会用内部模板重写 `.env`，静默删除用户自定义变量（GitHub Issue #26804）。恢复后立即备份 `.env`，每次 update 后对比差异。详见模式 B 第 8 步。

### 🚩 Post-restore: bin/ 目录恢复后丢失

Desktop 覆盖安装/更新可能清空或重置 `bin/` 目录。恢复后必须验证 `bin/` 文件列表与备份一致。详见模式 B 第 5 步。

### 🚩 Post-restore: 安装自带的 Bundled Skill 会污染技能目录

在全新的 Hermes Desktop 安装上将备份 `skills/` 恢复后，Desktop 的 `.bundled_manifest` 自行注入了 `dogfood/`、`yuanbao/`、以及根级 `SKILL.md`/`SKILL_en.md` 到 `skills/` 目录中。这些目录不在备份中，必须在恢复后清理。同时检查是否有本地 skill 目录（如 `skills/hermes/hermes-agent/`）被同名 builtin 版本覆盖 —— 此时该目录是无效的死代码，应一并删除。具体步骤见模式 B 第 5 步。

### 🚩 备份文档结构：合并而非拆分

补充说明（遗漏项、边界情况、已知问题）**必须合并到主备份文档中**，不要创建单独的遗漏清单文件。用户偏好：更少、更厚的文件优于一堆小文件。

- ❌ `Hermes完整环境备份.md` + `Hermes备份遗漏清单.md` → 用户问「为什么要单独一个文件」
- ✅ 所有信息（含遗漏项、shell 别名、参考文件映射）直接内嵌在主备份文档的 **「无法自动恢复的项目」** 章节

这一原则适用于所有文档类 task：补充说明、已知问题、边界情况都归入主文档，不要另起一篇。

## 必须手动处理的操作

| 操作 | 说明 |
|------|------|
| 填写 API Key | `.env` 中的密钥需手动填入 |
| WorkBuddy 登录 | WeChat OAuth 重新登录 |
| Chrome 扩展 | 加载 `agent-browser-mcp` 的 `TMWD CDP Bridge` 扩展 |
| 代理客户端 | 安装 v2ray/Xray，导入代理节点 |
| 知识库复制 | 物理复制 `D:/lk/Obsidian/Hermes/` 到新电脑 |

---

## API Key / `.env` 环境变量清单

当前生效 env 路径：`C:/Users/lk/AppData/Local/hermes/.env`。只备份变量名，不备份真实值。

### 当前生效 `.env` 变量名（2026-06-17）

```text
API_20188_KEY
API_SERVER_CORS_ORIGINS
API_SERVER_ENABLED
API_SERVER_HOST
API_SERVER_KEY
API_SERVER_PORT
BROWSER_INACTIVITY_TIMEOUT
BROWSER_SESSION_TIMEOUT
BROWSERBASE_ADVANCED_STEALTH
BROWSERBASE_PROXIES
DAO_API_KEY
HERMEST0_API_KEY
IMAGE_TOOLS_DEBUG
MOA_TOOLS_DEBUG
OPENCODE_GO_API_KEY
RANMENG_API_KEY
TERMINAL_LIFETIME_SECONDS
TERMINAL_MODAL_IMAGE
TERMINAL_TIMEOUT
VISION_TOOLS_DEBUG
WEB_TOOLS_DEBUG
WIKI_PATH
YU77_API_KEY
```

### 重要变量用途

| 变量 | 用途 |
|------|------|
| `OPENCODE_GO_API_KEY` | 当前主模型 provider |
| `RANMENG_API_KEY` | delegation 模型 provider |
| `DAO_API_KEY` | 备用 provider |
| `HERMEST0_API_KEY` | 备用自定义提供商 |
| `API_SERVER_*` | Desktop/Web Chat/API Server |
| `WIKI_PATH` | Obsidian/Hermes 知识库路径 |
| `TERMINAL_*` | terminal backend 超时/镜像等 |

> 2026-06-14：当前 Weixin 已 disabled，生效 `.env` 不含 `WEIXIN_TOKEN` / `WEIXIN_ACCOUNT_ID`。如将来恢复 Weixin，需要重新迁移或登录生成相关凭据。

---

## 恢复后验证

Agent 逐项检查（对照 restore-manifest.md）：

| 检查 | 命令 | 预期 |
|------|------|------|
| 配置健康 | `hermes config check` | 无 ERROR |
| Skills 数量 | `hermes skills list \| wc -l` | ≥ 50 |
| MCP 服务器 | `hermes mcp list` | agent_browser 可用 |
| 定时任务 | `hermes cron list` | 智能归档存在 |
| 模型可用 | `hermes chat -q "你好"` | 正常回复 |
| bin/ 完整 | `ls bin/` | 含 workbuddy-vision, uv.exe |
| scripts/ 完整 | `ls scripts/` | 含 smart-archive.py |
| 记忆一致 | `cat memories/MEMORY.md` | 与 manifest 一致 |
| .env 已保护 | `ls .env.post-restore` | 文件存在 |

## 参考

- `references/no-history-desktop-migration.md` — Hermes Desktop 重装系统迁移备份
- `references/hermes-backup-cli-reference.md` — 官方 `hermes backup`/`hermes import` 命令参考
- `references/desktop-overwrite-restore-recent-state.md` — 覆盖重装后配置恢复策略
- `references/desktop-overwrite-skill-config-audit.md` — 覆盖安装后审计步骤
- `references/windows-config-editing.md` — Windows 路径处理指南
- `references/hermes-config-post-cleanup-verification.md` — 恢复后自检清单（从 `hermes-env-backup` 并入）

> **v3.1.0** — 合并自 `hermes-env-backup`（2026-06-17）：新增备份清单速查表 + 简化恢复脚本 + post-cleanup 验证参考文件。

---

## 附录：Hermes 内置备份命令

Hermes 自带 `hermes backup` 和 `hermes import` 命令，可作为本 skill 的补充或替代方案。

| 命令 | 作用 | 输出 |
|------|------|------|
| `hermes backup` | 全量 zip 备份（含 sessions/state.db） | `~/hermes-backup-<ts>.zip` (~700MB) |
| `hermes backup -q` | 快照关键状态文件 | `state-snapshots/<ts>/` |
| `hermes import <zip>` | 从 zip 恢复 | 覆盖 `~/.hermes/` |
| `hermes checkpoints status` | 查看检查点存储 | 文件系统影子 git 仓库 |

**与本 skill 的差异：**
- `hermes backup` 打包**全部**文件（含 sessions/state.db/hermes-agent 源码），本 skill 选择性排除
- `hermes backup` 输出 zip，本 skill 输出到 Obsidian 知识库（可版本管理）
- `hermes backup -q` 适合操作前安全网，本 skill 适合长期归档

详见 `references/hermes-backup-cli-reference.md`。
