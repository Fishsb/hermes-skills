# Desktop 覆盖安装后的 Skills / Tools / Config 审计

适用场景：用户覆盖安装或重装 Hermes Desktop 后，询问 skills 是否完整、是否恢复了已卸载的自带 skill、工具/MCP/Profile/Cron/配置是否需要恢复。

## 可信来源优先级

1. 最近一次专门迁移备份：`D:/lk/Obsidian/Hermes/wiki/配置/Hermes迁移/restore-notes/`
2. 备份 runtime：`D:/lk/Obsidian/Hermes/wiki/配置/Hermes迁移/hermes-runtime-no-sessions/`
3. 当前 live CLI：`hermes skills list`, `hermes tools list`, `hermes mcp list`, `hermes profile list`, `hermes cron list`, `hermes gateway status`
4. 当前 runtime 文件扫描：`C:/Users/lk/AppData/Local/hermes/skills/**/SKILL.md`
5. 卸载备份：`C:/Users/lk/.hermes/skills-backup/*uninstalled-skills*.md`
6. `.bundled_manifest`：判断是否为 bundled/self-shipped skill

## 标准检查步骤

### 1. 先比较 Hermes 实际加载数量，不要只看文件数量

`SKILL.md` 文件扫描会把 references/original、备份副本、社区仓库中的样例也算进去。真正判断是否缺失，应以：

```bash
hermes skills list
```

为准。

典型输出口径：

```text
0 hub-installed, 5 builtin, 48 local — 53 enabled, 0 disabled
```

如果当前 CLI 数量与 `restore-notes/hermes-skills-list.txt` 一致，一般不需要恢复安装。

### 2. 只把文件扫描作为辅助

文件扫描用于发现：

- 重复 `SKILL.md`
- references 里的原始备份误放为 `SKILL.md`
- 社区仓库 examples 里自带的 `SKILL.md`
- bundled manifest 中恢复出来的官方 skill

但不要把文件扫描总数直接当成 Hermes 生效 skill 数。

### 3. 判断“覆盖安装恢复了以前卸载的自带 skill”

强候选需要同时满足：

- 当前 `hermes skills list` 或当前 skills 目录中存在；
- 出现在 `~/.hermes/skills-backup/*uninstalled-skills*.md`；
- 出现在 `skills/.bundled_manifest`；
- 路径/mtime 与 bundled 批量安装特征一致。

本次案例中强候选：

```text
segment-anything-model
```

但注意：如果覆盖前 `restore-notes/hermes-skills-list.txt` 已经包含它，则它不是“本次覆盖后才新增”，而是覆盖前最新状态中已经存在。

### 4. 工具/MCP/Profile/Cron/Gateway/config 对比

至少比较这些快照文件：

```text
restore-notes/hermes-tools-list.txt
restore-notes/hermes-mcp-list.txt
restore-notes/hermes-cron-list.txt
restore-notes/hermes-profile-list.txt
restore-notes/hermes-gateway-status.txt
restore-notes/hermes-version.txt
restore-notes/hermes-config-check.txt
hermes-runtime-no-sessions/config.yaml
hermes-runtime-no-sessions/.env
```

Live 对应命令：

```bash
hermes tools list
hermes mcp list
hermes cron list
hermes profile list
hermes gateway status
hermes --version
```

对 `config.yaml` 重点比较：

- `model`
- `delegation`
- `memory`
- `compression`
- `approvals`
- `platform_toolsets.cli`
- `mcp_servers` names + enabled flags
- `custom_providers` names
- `agent.system_prompt` 如用户关心习惯/路由

对 `.env` 只比较变量名和数量，不要泄露值。

## 输出判断模板

```text
结论：覆盖前最近快照是 X enabled；当前是 X enabled；无需恢复安装。

| 类型 | 覆盖前 | 当前 | 是否需要恢复 |
|---|---|---|---|
| Skills | ... | ... | 不需要/需要 |
| Tools | ... | ... | ... |
| MCP | ... | ... | ... |
| Profiles | ... | ... | ... |
| Cron | ... | ... | ... |
| Gateway | ... | ... | ... |
| config.yaml | 核心项一致/差异 | ... | ... |
| .env | 变量名一致/差异 | ... | ... |
```

## Pitfalls

- 不要把旧的 84 / 36 / 22 skill 记录误当成当前目标；它们可能分别代表历史全量、旧恢复指南、极简核心清理目标。
- 用户说“知识库中的所有备份恢复都以现在最新的为准”时，优先使用 `Hermes迁移/` 最新备份，而不是更早的 `Hermes完整环境备份_20260613.md`。
- `hermes mcp list` 显示 disabled 不一定是异常；若覆盖前就是 disabled，当前一致则无需恢复。
- Gateway stopped 若覆盖前也 stopped，不要当成覆盖安装导致的问题。
