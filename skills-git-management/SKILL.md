---
name: skills-git-management
description: "Manage Hermes Agent skills with Git/GitHub — version control, selective tracking, SSH auth setup, repo creation, and sync workflow. Covers local skills directory structure, .gitignore strategy, and push workflow."
version: 2.6.0
author: Hermes Agent
tags: [git, github, skills, version-control, backup, devops]
triggers:
  - 推送到github
  - 同步到github
---

# Skills Git Management

> 用 Git/GitHub 对 Hermes skills 做版本管理，实现变更追踪、增量同步、灾难恢复。

## 适用场景

- 首次将 skills 纳入 Git 管理
- 换电脑/重装后从 GitHub 恢复 skills
- 定期同步本地 skills 到远端
- 多人协作共享 skills

## 前置条件

- git 已安装（Hermes 内置在 `$HERMES_DIR/git/`）
- GitHub 账号
- 网络连通（SSH 端口 22 可达）

## 工作流

### 1. 初始化本地仓库

```bash
cd "$APPDATA/../Local/hermes/skills"
git init
git checkout -b main
```

### 2. 配置 .gitignore

必须排除 Hermes 框架系统文件和 hub 缓存：

```gitignore
# Hermes 系统文件
.hub/
.bundled_manifest
.curator_state
.usage.json
.usage.json.lock

# 社区 hub 技能（第三方管理，非自创）
community/

# 空分类目录（Hermes 初始自带但无自创skill）
apple/
mlops/
media/
email/
references/
smart-home/
social-media/
creative/
data-science/

# 操作系统
Thumbs.db
.DS_Store

# 敏感文件（按需）
*.env
*cred*
*secret*
```

### 3. 配置 Git 身份

```bash
git config user.name "YourGitHubUsername"
git config user.email "your@email.com"
```

### 4. SSH Key 认证

推荐**无密码** SSH Key，避免每次推送需要输入密码：

```bash
# 生成（无密码，推荐首次使用）
ssh-keygen -t ed25519 -C "your@email.com" -N ""

# 或者生成（有密码，更安全）
ssh-keygen -t ed25519 -C "your@email.com"  # 会提示输入密码

# 查看公钥并复制
cat ~/.ssh/id_ed25519.pub

# 添加到 GitHub: https://github.com/settings/ssh/new
# 类型选 Authentication Key

# 启动 ssh-agent 并加载 key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# 验证连接
ssh -T git@github.com -o StrictHostKeyChecking=accept-new
# 应输出: Hi YourUsername! You've successfully authenticated...

### 5. 创建 GitHub 仓库

**方式 A — 手动（推荐首次）：**
1. 打开 https://github.com/new
2. 仓库名：自定（如 `hermes-skills`）
3. 设为 **Private**
4. 不要初始化 README/license/.gitignore
5. 创建

**方式 B — gh CLI：**
```bash
# 安装 gh
# 登录
gh auth login --git-protocol ssh
# 创建
gh repo create hermes-skills --private --source=. --remote=origin --push
```

### 6. 确定跟踪范围（选择性发布必做）

不是所有 skills 都适合上传。Hermes skills 目录包含平台自带、Hub 安装、Agent 代建、用户自创四类。上传前先筛选：

**方式 A — 从知识库查分类（推荐）：**

如果 Hermes 有 Obsidian 知识库且维护了 Skill 分类笔记，优先查阅：

```bash
# 搜索分类笔记（路径可能为 wiki/skill/Skill开发/ 或类似）
grep -rl "平台自带\|自建\|代建\|网络安装" <vault_path> --include="*.md" | head -5
```

**补充 — 用 session_search 追溯创建记录：**

分类笔记可能过时。对新创建或分类不明的 skill，用会话搜索追溯其来源：

```bash
# Hermes Agent 中直接查询
session_search(query="创建 skill <name> 或 skill_manage create <name>")
session_search(query="<skill名> 或 技能描述关键词")
```

> 如果在知识库分类笔记和会话记录间有冲突，**以会话记录为准**——分类笔记是静态快照，会话是实际交互证据。

分类笔记通常按以下结构组织：
- 🏠 **平台自带**（Hermes 附送，不上传）
- 🌐 **网络安装**（社区 Hub 下载，不上传）
- 👤 **用户/Agent 自建**（上传候选）
  - 用户自行编写 → 优先上传
  - Agent 代为创建 → 除非用户明确指示，否则不上传

**方式 B — 检查 author 字段：**

```bash
for cat in <categories>; do
  for skill in "$cat"/*/SKILL.md; do
    echo "$(basename $(dirname $skill)) → $(grep '^author:' $skill 2>/dev/null)"
  done
done
```

- `author: Hermes Agent` → 平台自带或自动生成，除非确定是自创
- `author: YourName` → 用户自创
- `author: <org>` → 可能来自社区

**方式 C — 检查 .hub 记录：**

```bash
cat .hub/audit.log .hub/taps.json 2>/dev/null  # Hub 安装记录
cat .bundled_manifest  # Hermes 自带清单
```

> 🔑 **铁律**：如果有知识库分类笔记，**以知识库为准**。不要仅凭 author 字段或文件名判断——author: "Hermes Agent" 也可能是用户通过 agent 创建的实用技能。
>
> ⏰ **分类笔记时效性**：知识库分类笔记是静态快照，可能滞后于实际创建的技能。如果用户最近新增了技能，分类笔记可能未收录。遇到分类笔记与实际情况不符时，用 `session_search` 追溯创建记录做交叉验证，以实际会话为准。

**方式 D — 用户明确确认：**

不确定时，列出所有候选 skills 给用户选择，不要替用户做决定。

推送前，先调研类似仓库的做法，确保文档质量与社区对齐：

```bash
curl -s "https://api.github.com/search/repositories?q=hermes+skill+OR+hermes-agent+skill&sort=updated&per_page=8" \
  | python -c "import json,sys; d=json.load(sys.stdin); [print(f'{r[\"full_name\"]:45s} ⭐{r[\"stargazers_count\"]:5d}  {r[\"description\"] or \"-\"}') for r in d.get('items',[])]"
```

重点关注以下仓库的 README 结构和安装方式：
- `veawho/via54Skills` — 双语 README、命名规范、前注格式
- `The-Aetheris/skills` — 简洁技能列表、安装方式
- `lukemcqueen/hermes-cortex` — 徽章头、多方法安装
- `kepano/obsidian-skills` — 专业文档、社区标准

> 参考结果见 `references/related-community-repos.md`

### 7. 创建 README 与文档（公开仓库必做）

公开仓库需要完整的文档，标准结构：

```markdown
# 仓库名 — 一句话说明

徽章（GitHub Stars、Hermes 版本、许可证等）

## 目录 / TOC

## 简介
- 仓库用途、适用人群
- 技能总览表格（分类 | 数量 | 领域）

## 技能总览 / Skill Catalog

### 模式 A — 功能分组（推荐，用户偏好）

按**协作流水线**分组展示，每个 skill 配角色详情表。适合 5-15 个技能的中等仓库。

```markdown
### 📚 知识管理流水线

> 一句话说明这组技能共同实现的目标。

```
skill-A       ← 做什么
    ↕
skill-B       ← 做什么
```

| Skill | 在此流水线中的角色 |
|-------|-------------------|
| [`skill-a`](skill-a/) | 🔄 入口——做什么 & 配合谁 |
| [`skill-b`](skill-b/) | 🧹 后处理——做什么 & 配合谁 |

#### `skill-a` — 中文名（角色说明）

| 项目 | 说明 |
|------|------|
| **做什么** | 一句话核心功能 |
| **什么时候用** | 触发场景 |
| **上下游** | <kbd>上游</kbd> 依赖什么 → <kbd>下游</kbd> 产出什么 |
| **触发词** | `触发词1`、`触发词2` |
| **不做什么** | 边界说明，避免预期偏差 |
```

每个角色表字段说明：

| 字段 | 内容要求 |
|------|---------|
| **做什么** | 一句话核心功能，不含触发词 |
| **什么时候用** | 具体的触发场景描述 |
| **上下游** | 该 skill 依赖什么、产出什么 |
| **触发词** | 2-3 个简短中文词，用 ` · ` 分隔 |
| **不做什么** | 边界说明，让用户知道这个 skill 不该用于什么 |

### 模式 B — 按分类平铺（传统）

适合大量 skill（20+）的仓库，保持简洁：

```markdown
| Skill | 描述 |
|-------|------|
| [`skill-a`](skill-a/) | 一句话描述 |
| [`skill-b`](skill-b/) | 一句话描述 |
```

## 安装指南 / Installation
- 完整克隆
- 稀疏检出（选择性安装）
- 前提条件

## 更新同步 / Sync & Update

## 技能管理 / Skill Management
- 查看、创建、清理 skills

## 相关资源 / Related Resources
- 链接到社区参考仓库

## 许可证 / License
```

> ✨ 公开仓库的 README 建议用中文为主 + 英文术语/标题双语，方便国内用户也服务国际读者。
>
> ✨ 触发词最佳实践：每个 skill 的触发词用 2-3 个简短中文词（如 `智能归档`、`agent环境备份`），不同 skill 之间不重叠。触发词写在 SKILL.md 的 frontmatter `triggers:` 字段和 README 角色表中的「触发词」行，两端保持一致。避免长句(>`用git/github做项目管理`)或模糊单字(>`归档`)。
>
> 📄 功能分组模板见 `templates/README-grouped.md`，复制后按需修改即可。

### 8. 首次推送

```bash
git add .
git commit -m "🎉 init: initial skills snapshot"
git remote add origin git@github.com:YourUsername/hermes-skills.git
git push -u origin main
```

### 9. 日常更新

```bash
cd "$APPDATA/../Local/hermes/skills"

# 查看变更
git status

# 暂存并提交
git add -A
git commit -m "update: [简要描述变更]"

# 推送
git push
```

### 10. 推送到公开仓库前的隐私审查 🔒

> 仓库设为 Public 前，必须做一次隐私扫描。一旦推送，历史无法撤回（除非 force push 重写）。

**扫描清单：**

```bash
cd "$APPDATA/../Local/hermes/skills"

# 1. 检查 tracked 文件中是否含路径/用户名
grep -rniE "$(whoami)|$USERNAME|C:\\\\Users\\\\[^\\\\]+" $(git ls-files) --include="*.md" 2>/dev/null | grep -v ".gitignore\|README.md"

# 2. 检查 API Key / Token / 密码字面值（非 ${VAR} 占位符）
grep -rniE "sk-[a-zA-Z0-9]{20,}|api.?key[=:].{8,}|secret[=:].{8,}|token[=:].{8,}|password[=:].{8,}" $(git ls-files) 2>/dev/null

# 3. 检查个人画像类信息（真名、邮箱、手机、地址等）
grep -rniE "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" $(git ls-files) --include="*.md" 2>/dev/null
grep -rniE "1[3-9][0-9]{9}" $(git ls-files) --include="*.md" 2>/dev/null
```

**分类评估：**

| 发现类型 | 风险 | 处理方式 |
|---------|------|---------|
| 路径中的用户名（`C:\Users\lk\`、`D:\lk\`） | 低 — 技术仓库中常见 | 可留，或替换为 `$HOME` |
| API Key / Token 字面值 | 🔴 极高 — 立即撤销该 key | force push 清除历史后替换为 `${VAR}` |
| 邮箱/手机/地址 | 🟡 中 — 建议清理 | 替换占位符后重新提交 |
| 个人偏好/习惯/履历 | 🟡 中 — 视用户意愿 | 评估是否暴露过多隐私 |

> 如果发现已推送的内容含敏感信息，参考「历史清理需要 force push」pitfall 处理。

### 11. 从远端恢复

```bash
cd "$APPDATA/../Local/hermes/skills"
git init
git remote add origin git@github.com:YourUsername/hermes-skills.git
git fetch origin
git reset --hard origin/main
```

## 目录结构参考

有两种主流布局，按需选择：

### 🅰 分类结构（Hermes 原生）

按 Hermes 自动创建的分类目录组织，适合**个人私有备份**：

```
skills/
├── .git/
├── .gitignore
├── README.md
├── autonomous-ai-agents/   # 分类目录
│   ├── DESCRIPTION.md
│   └── my-first-skill/SKILL.md
├── devops/
│   └── another-skill/SKILL.md
└── ...
```

`.gitignore` 需要逐类排除非自创 skill。

### 🅱 Flat 平铺结构（社区标准）

所有 skill 直接放在根目录，适合**公开分享**。社区主流仓库（The-Aetheris/skills、kepano/obsidian-skills）均采用此方式：

```
skills/
├── .gitignore
├── README.md
├── my-first-skill/
│   └── SKILL.md
├── another-skill/
│   └── SKILL.md
└── ...
```

`.gitignore` 仅需排除系统文件：

```gitignore
.hub/
.bundled_manifest
.curator_state
.usage.json
.usage.json.lock
__pycache__/
*.pyc
.DS_Store
```

> 💡 **推荐**：公开仓库用 flat 结构。安装更直观、仓库更干净、与社区兼容。

## 分类→Flat 迁移（改造已有仓库）

如果初期用了分类结构，后期想改成 flat，流程如下：

```bash
cd "$APPDATA/../Local/hermes/skills"

# 1. 定位每个 skill 并移动到根目录
mv autonomous-ai-agents/my-skill/ ./
mv devops/another-skill/ ./

# 2. 删除空分类目录
rmdir autonomous-ai-agents devops note-taking  # 逐个检查是否为空

# 3. 简化 .gitignore（仅系统文件，无分类排除）
cat > .gitignore << 'EOF'
.hub/
.bundled_manifest
.curator_state
.usage.json
.usage.json.lock
__pycache__/
*.pyc
.DS_Store
Thumbs.db
EOF

# 4. 清历史建新库（推荐）
rm -rf .git
git init && git checkout -b main
git remote add origin git@github.com:User/repo.git
git add . && git commit -m "🎉 init: skills repo (flat)"
git push -u origin main --force   # ⚠️ 覆盖远端，单人仓库适用
```

⚠️ 多人协作的仓库不要用 `--force`，改用正常 PR 流程。

## Skill 合并（消除重复）

当发现两个 skill 功能重叠时，合并流程：

1. **比较 SKILL.md** — 读两个 skill 的完整内容，识别各自独特部分
2. **合并到目标 skill** — 将较新的 skill 的独特内容（清单、pitfalls、参考文件）添加到主 skill 中
3. **迁移参考文件** — 将被删除 skill 的 `references/` 文件复制到主 skill 目录
4. **更新索引** — 在被保留的 skill 中加版本变更说明，标记合并来源
5. **删除冗余 skill** — 从磁盘和 git 同时移除

```bash
# 示例：合并 hermes-env-backup → agent-migration-backup
cp hermes-env-backup/references/*.md agent-migration-backup/references/
rm -rf hermes-env-backup
git add -A && git commit -m "🔀 合并 <被删skill> → <目标skill>，已迁移全部内容"
git push
```

6. **更新 README** — 删除被合并的 skill 条目，更新目标 skill 的描述行

## 选择策略

| 场景 | 推荐策略 |
|------|---------|
| 首次设置（私有备份） | 分类结构 + 手动创建仓库 + 本地 init + push |
| 首次设置（公开分享） | **Flat 结构** + 手动创建仓库 + 调研社区仓库 + 写完整 README + 本地 init + push |
| 日常同步 | `git add . && git commit -m "msg" && git push` |
| 选择性跟踪 | 用知识库分类笔记筛选，只跟踪用户自创 skill |
| 换电脑恢复 | git init + remote + reset 方式 |
| 从分类改 flat | 见上方「分类→Flat 迁移」 |

## Pitfalls

- **SSH 超时**：首次 `ssh -T` 可能慢，设 `-o ConnectTimeout=10` 重试即可
- **社区技能体积**：`community/loci/` 可达 90MB+，建议排除或作为子模块
- **系统文件污染**：`.hub/`、`.usage.json` 等频繁变动，务必加入 `.gitignore`
- **Windows 路径**：Hermes skills 在 `%LOCALAPPDATA%\hermes\skills`，非 `%APPDATA%`
- **gh CLI 未安装**：建议首次用手动方式创建仓库，跳过 gh 认证步骤
- **远程已有初始提交**：如果创建 GitHub 仓库时勾选了 README，首次推送需先 pull 合并：
  ```bash
  git pull origin main --allow-unrelated-histories --no-rebase
  # 解决冲突后
  git push -u origin main
  ```
- **ssh-agent 会话过期**：每次新终端窗口需要 `eval "$(ssh-agent -s)" && ssh-add` 重新加载 key
- **公钥/私有选择**：如果想分享给社区，建议设为 **Public**（公开）；仅个人备份则用 Private
- **推送被拒**：检查是否用了正确的远端 URL：`git remote -v`，确认是 SSH 格式 `git@github.com:User/repo.git`
- **公开仓库文档不足**：公开 repo 必须有完整 README（技能总览+安装指南+相关资源），否则别人拿到也无法使用。推送前调研社区类似仓库的文档模式，参考 `references/related-community-repos.md`
- **公开仓库隐私泄露**：文件路径中的用户名（`C:\Users\lk\`）、API Key 字面值、个人信息（邮箱/手机/地址）容易被忽视。推送到 Public 仓库前必须执行隐私扫描（见步骤 10）。历史已有泄露时，参考「历史清理需要 force push」处理。
- **`.gitignore` 路径匹配陷阱**：`.gitignore` 中不带前导 `/` 的路径模式会匹配**任意层级**的目录。例如 `references/` 会排除 skills 所有子目录中名为 `references` 的文件夹（如 `devops/agent-migration-backup/references/`）。**必须用 `/references/`（前导斜杠）才只匹配根目录**，或用完整路径如 `path/to/specific/references/` 做精确排除
- **`git add -A` 会重添已删除文件**：使用 `git rm --cached` 从索引移除文件后，**不要用 `git add -A` 或 `git add --all`**，否则会重新添加所有仍在磁盘上的文件。应改为：显式指定要添加的路径 `git add specific/path/`，或用 `git add .` 配合完整的 `.gitignore` 过滤
- **历史清理需要 force push**：如果错误的文件已经被推送，用以下流程重写历史：
  ```bash
  git rm --cached -r unwanted/dir/
  git commit --amend --no-edit   # 追加到最近一次 commit
  git push -u origin main --force  # 强制覆盖远端
  ```
  ⚠️ 多人协作的仓库慎用 force push
- **完全清空历史重新开始**：如果推送了错误内容且不想保留任何历史记录，彻底删除 `.git` 重新初始化是最干净的方式：
  ```bash
  rm -rf .git
  git init -b main
  git remote add origin git@github.com:User/repo.git
  git add . && git commit -m "🎉 clean slate"
  git push -u origin main --force
  ```
  适用场景：单人仓库清理历史、分类→flat 迁移后重建、删除了大量错误文件后。**多人协作仓库不要用**。

## 验证

推送后确认仓库状态健康（公开仓库额外加 [x]）：

- [ ] 推送前已执行隐私审查（步骤 10），无敏感信息泄露
- [ ] 远端与本地一致 — `git status` → `nothing to commit, working tree clean`
- [ ] 远端分支可达 — `git branch -r` 显示 `origin/main`
```bash
# 检查远端与本地一致
git status
# 应输出: nothing to commit, working tree clean

# 检查远端分支
git branch -r
# 应显示 origin/main
```
