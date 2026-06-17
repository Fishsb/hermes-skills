# Hermes 内置备份命令参考

> 来源：Hermes CLI 内置命令 + GitHub PR #43058（自动备份，待合并）

## hermes backup

```bash
# 全量备份（~700MB zip，含所有文件）
hermes backup -o ~/hermes-backup-$(date +%Y%m%d).zip

# 快照关键状态文件（config/state.db/.env/auth/cron）
hermes backup -q -l "pre-restore"

# 查看快照列表
ls ~/AppData/Local/hermes/state-snapshots/
```

**包含：** config.yaml, .env, skills/, scripts/, bin/, cron/, profiles/, memories/, state.db, sessions/, hermes-agent/, 所有文件
**排除：** hermes-agent/node_modules/（如存在）

## hermes import

```bash
# 从 zip 恢复
hermes import ~/hermes-backup-20260617.zip --force
```

⚠️ `--force` 会覆盖现有文件。建议先手动备份当前状态。

## hermes checkpoints

```bash
# 查看检查点状态
hermes checkpoints status

# 清理过期检查点
hermes checkpoints prune

# 清除所有检查点
hermes checkpoints clear
```

## 与 agent-migration-backup skill 的对比

| 维度 | `hermes backup` | agent-migration-backup |
|------|----------------|----------------------|
| 输出格式 | zip 文件 | Obsidian 知识库目录 |
| 包含范围 | 全部（含 sessions/state.db） | 选择性排除 |
| 版本管理 | 无（单个 zip） | Git（Obsidian vault） |
| 恢复验证 | 无 | restore-notes 验证清单 |
| 适用场景 | 快速灾难恢复 | 长期归档 + 跨机器迁移 |

## 推荐组合使用

1. 日常：`hermes backup -q` 作为操作前安全网
2. 归档：agent-migration-backup skill 全量备份到知识库
3. 灾难恢复：优先 `hermes import`，再用知识库补充
