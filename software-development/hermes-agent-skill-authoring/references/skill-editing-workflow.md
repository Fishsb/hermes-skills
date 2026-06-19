# Skill 编辑工作流

> 完整编辑流程、优化步骤、回归验证
> 加载方式：`skill_view(name='hermes-agent-skill-authoring', file_path='references/skill-editing-workflow.md')`

## 优化前架构分析

每次编辑前分析：职责边界、阶段顺序、决策门、与关联 skill 的集成、回退策略、验证流程。避免"拆东墙补西墙"的修改。

**回归保持标准**：先陈述修改前的任务目标和成功效果，再验证优化版是否能达成。旧行为是验收基线。

## 优化补丁流程

1. **加载目标 skill 和关联 skill** — 确认当前 frontmatter、章节布局、链接引用、职责边界
2. **陈述验收基线** — 写出旧 skill 已经做对了什么，哪些命令/输出/副作用必须保持兼容
3. **映射证据到结构** — 判断修改属于 Overview/When to Use/Workflow/Decision Gates/Output Contract/Common Pitfalls/Verification Checklist/References
4. **补丁最小一致单元** — 小范围用 `patch` / `skill_manage(action='patch')`，大改才用 `write_file` / `skill_manage(action='edit')`
5. **保留引用** — 长篇示例、对话记录、完整脚本放 `references/`
6. **更新交叉链接** — 修改了职责边界时同步更新 `metadata.hermes.related_skills` 和 References
7. **结构验证** — YAML frontmatter、description 长度、必需字段、必需章节、代码 fence 平衡、链接引用路径
8. **功能/回归验证** — 至少一条旧成功路径 + 一条新行为验证
9. **报告 diff 和证据** — 最终回答说明什么改了、为什么、在哪、验证了什么

### 工作流中的章节放置

| 新信息 | 放哪里 |
|--------|--------|
| Skill 职责、阶段归属、非目标 | Overview |
| 触发词、反触发 | When to Use |
| 有序步骤、状态转换 | Workflow |
| 回退/确认/删除条件 | Decision Gates |
| CLI 语法、JSON schema | Output Contract |
| 失败模式+原因+修复 | Common Pitfalls |
| 验证命令/回读检查 | Verification Checklist |
| 长示例/完整脚本 | references/ |
| 一次性任务状态/临时 ID | 不入 skill，放 session/归档 |

## 验证清单

- [ ] 文件路径正确：`skills/<category>/<name>/SKILL.md`
- [ ] frontmatter 从字节 0 以 `---` 开始，以 `\n---\n` 闭合
- [ ] `name`, `description`, `version`, `author`, `license`, `metadata.hermes.{tags, related_skills}` 都存在
- [ ] name ≤64 字符，小写+连字符
- [ ] description ≤1024 字符，以 "Use when ..." 开头
- [ ] 总文件 ≤100,000 字符（目标 8-15k）
- [ ] auto_load skill 含排除表 + 强制规则引用其范围
- [ ] related_skills 引用在 repo 内可解析（或确认为 user-local）

## 跨 Skill 引用

`related_skills` 同时搜索 `skills/`（repo）和 `~/.hermes/skills/`（本地）。优先引用 repo 内 skill。频繁引用的本地 skill 应考虑提升到 repo。
