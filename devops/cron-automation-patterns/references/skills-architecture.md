# 智能归档 Skills 集合架构

此用户选择**多个 skill 集合管理**而非合并为一个，每个 skill 有明确的 Phase 所有权边界。

## 所有权矩阵

| Phase | 所有者 | 职责 |
|:------|:-------|:------|
| **Phase 1: 扫描** | `cron-automation-patterns` | 编排扫描流程：调用脚本、清理旧历史、分组闲置/活跃、缓存 tracker |
| **Phase 1 内联: SF 扫描** | `skill-optimization` 端① | 读取 `skill-suggestions.json` + 扫描 skills 状态 → 随推送展示 |
| **Phase 2: 用户交互** | `cron-automation-patterns` | 推送格式、用户回复解析（编号/approve） |
| **Phase 3a: 归档执行** | `smart-archive.py`（脚本） | export→stage→queue→delete 原会话 |
| **Phase 3b: 知识提取** | `skill-optimization` 端② | 读取暂存 JSON → 类别映射 → Wiki 写入 + [[WikiLink]] |
| **Phase 3c: 状态同步** | `skill-optimization` 端② | 知识提取完成后调用 `complete`/`fail` 同步 `ingest-queue.json` |
| **Phase 3d: SF 建议** | `skill-optimization` 端② → `Skill Factory` | 识别会话中已用 skills → 生成 `pending_confirmation` 建议 |
| **Phase 4: 确认执行** | `hermes-agent-skill-authoring` | 用户批准后 patch SKILL.md |

## 加载路径

```
遇到智能归档任务
  │
  ├─ 扫描/编排 → 加载 cron-automation-patterns
  ├─ 知识提取/SF 扫描 → 加载 skill-optimization
  ├─ 生成优化建议 → 加载 Skill Factory (community)
  ├─ 状态同步 → smart-archive.py complete|fail (脚本, 非 skill)
  └─ 确认后 patch → 加载 hermes-agent-skill-authoring
```

## 设计原则

- **Phase 单一所有权**: 每个 Phase 只有一个 skill 对其输出/行为负责，避免两处同时描述同一步骤导致漂移
- **集合而非单体**: 用户明确偏好"多个集合更好管理"，不要试图合并为一个 skill
- **确定性在前，LLM 在后**: 扫描和归档脚本是零 token 确定性代码；知识提取和 SF 检查是 LLM 驱动的 Agent 步骤
- **引用而非复制**: 如果 `cron-automation-patterns` 提到归档后步骤，应指向 `skill-optimization` 而非展开细节

## 相关 Skills

| Skill | 路径 |
|:------|:-----|
| `cron-automation-patterns` | (本 skill) |
| `skill-optimization` | `note-taking/knowledge-ingestion/SKILL.md` |
| `Skill Factory` | `community/hermes-skill-factory/skills/skill-factory/SKILL.md` |
| `hermes-agent-skill-authoring` | `software-development/hermes-agent-skill-authoring/SKILL.md` |