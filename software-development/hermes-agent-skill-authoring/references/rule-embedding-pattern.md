# 核心规则嵌入模式 — 完整参考

> 从 2026-06-17 会话中提炼：为什么核心规则被忽略，以及如何确保 Agent 真正遵守。

## 会话背景

用户有一条 FIX_LOOP 核心规则写在 MEMORY.md 中：

```
FIX_LOOP — 工具/技能/插件更新: review defects → fix → re-review until zero issues remain...
```

在执行 knowledge-base-compression skill 优化时，该规则**没有被遵守**。用户质问后分析出根因：

| 问题 | 细节 |
|------|------|
| 语义归类窄化 | "技能更新" vs "技能编辑" — Agent 认为自己是在"创作"而非"更新" |
| 被动文本噪音 | MEMORY 块 6 条规则排在 system prompt 下部，不主动触发 |
| 无打断机制 | 执行→验证→输出，缺少"强制自检后再输出"的阻断 |
| 流程无锚点 | FIX_LOOP 不附着在任何一个 workflow step 上 |

## 修复方案：三通道注入

### Channel A — MEMORY 规则

旧：
```
FIX_LOOP — 工具/技能/插件更新: review defects → fix → re-review until zero issues remain...
```

新（加触发门）：
```
FIX_LOOP — 硬性执行规则，适用以下任一场景时必须强制触发：
   ① 编辑/创建任何 SKILL.md 或 skill 配置文件（含 references/ 子文件）
   ② 执行 ≥3 次 write_file/patch/memory() 的批量操作
   ③ 使用 skill_manage()/memory()/write_file 修改持久状态
   ④ 用 delegate_task 派发包含上述操作的任务
  流程：review defects → fix → re-review → 迭代到零残留 → 再输出
  报告：先输出高保真英文摘要，再追加中文说明
```

关键改动：
- 触发条件用**4 个客观编号门**（Agent 无法说"我不属于这个类别"）
- 流程 + 格式固化到规则内

### Channel B — Workflow 内嵌

在关联 skill 的 workflow step 中插入**强制锚点**：

```
### ⛔ FIX_LOOP GATE — 预输出强制门

在结束输出前，必须逐项核验：

| # | 核验项 | 判定 |
|---|--------|:----:|
| 1 | 匹配触发条件？... | 是/否 |
| 2 | 零残留？... | 是/否 |
| 3 | 进度可见？... | 是/否 |
| 4 | 输出格式？... | 是/否 |
```

GATE 必须插在 workflow 的**最后一步之前**，不只是文档末尾。

### Channel C — GATE 验证

三项/四项全部「是」才允许输出。任何一项「否」→ 停止 → 修复 → 重新 review。

## 实际执行验证

在执行此模式注入后的当前会话中，Agent 自我运行 GATE：

| Q1 匹配触发条件 | Q2 零残留 | Q3 进度可见 | Q4 输出格式 | 结果 |
|:---:|:---:|:---:|:---:|:---:|
| ✅ 是 | ✅ 是 | ✅ 是 | ✅ 是 | 通过输出 |

## 可复用规范

任何新核心规则的创建步骤：

```
1. 写入 MEMORY.md，带 4 门触发条件
2. 在相关 skill 的 workflow 中插入 GATE 或 tickpoint
3. 输出前 GATE 验证的 checklist 项
4. 重启新会话测试触发
```

---

*衍生自 2026-06-17 会话 | 适用 skill: hermes-agent-skill-authoring*
