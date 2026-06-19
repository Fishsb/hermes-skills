# Skill 结构与设计参考

> 完整 Skill 结构规范、清单、Auto-Load 设计原则
> 加载方式：`skill_view(name='hermes-agent-skill-authoring', file_path='references/skill-structure.md')`

## 完整结构模板

```
# <Title>

## Overview
责任边界、目的、非目标。声明本 skill 拥有哪个阶段，哪个关联 skill 拥有相邻阶段。

## When to Use
- 正向触发：用户措辞、任务类别、文件/路径/脚本信号
- 反向触发：何时不使用
- 交接触发：何时加载关联 skill

## Preconditions / Inputs / Environment（可选但推荐）
所需文件、命令、凭据、路径、队列状态或用户确认边界。

## Workflow / Phases
带确定步骤、决策门、回退分支的编号执行路径。

## Output Contract / State Contract（适用时）
CLI 语法、JSON schema、状态枚举、队列文件、sidecar 文件或用户可见响应格式。

## Common Pitfalls
已知失败模式编号列表、原因和修复方法。

## Verification Checklist
- [ ] 结构检查：frontmatter、必需章节、链接
- [ ] 功能检查：命令/脚本仍正常
- [ ] 回归检查：旧成功路径和失败路径仍行为一致

## References
指向大的实现笔记、示例、对话证据、模板或脚本。

## One-Shot Recipes（可选）
命名场景 → 具体命令序列。
```

## 章节放置规则

| 新信息类型 | 放哪里 |
|-----------|--------|
| Skill 职责、阶段归属、非目标 | `## Overview` |
| 用户措辞、触发词、反触发 | `## When to Use` |
| 有序步骤、状态转换、交接顺序 | Workflow / Phases |
| fallback/确认/删除/重试/停止条件 | Decision Gates / Workflow |
| CLI 语法、JSON schema、队列状态 | Output Contract |
| 失败模式 + 原因 + 修复 | `## Common Pitfalls` |
| 验证命令/搜索/回读检查 | `## Verification Checklist` |
| 长示例、完整脚本、对话证据 | `references/*.md` |

## Auto-Load 设计原则

Skill 设置 `metadata.hermes.auto_load: true` 后，完整 SKILL.md 内容会在会话启动时注入 system prompt。

**代价：** per-session，非 per-turn（注入后缓存）
**仍然要精简：** 每会话都消耗 prompt budget

### 必须包含排除表

```yaml
metadata:
  hermes:
    auto_load: true
```

排除表放在强制规则之后：

| 排除场景 | 原因 |
|---------|------|
| Skill/工具自修改 | 框架操作，非领域任务 |
| 纯读操作 | 无需编排 |
| 单步编辑 | 无依赖链 |
| 一次查询 | 一步解决 |
| 纯对话 | 无可执行任务 |
| 维护操作 | Agent 内务 |
| Agent 专用工具 | 工作流桥无法提供 |

### 强制规则-排除表耦合

- "适用范围内（即未命中排除表的任务）必须先走本工作流"
- "排除条件优先 — 命中排除表的任务由主 Agent 直接执行"

### 回归验证标准

优化 auto_load skill 时：
1. 确认排除表覆盖所有历史误触发
2. 确认强制规则引用排除表作为范围边界
3. 确认排除的任务仍能通过主 Agent 绕过成功执行
4. 搜索强制规则中的无条件语言（"所有任务"、"无例外"）— 这些必须有范围约束

### 🛡 三重保障层（可靠性）

auto_load 注入机制可能因框架 bug、索引失败、缓存问题而失效。**纯依赖 auto_load 层 = 单点故障。**

创建 auto_load skill 时，必须将**核心强制规则的紧凑版本**部署到另外两层：

| 层 | 位置 | 特点 |
|----|------|------|
| 🥇 config.yaml `system_prompt` | `config.yaml agent.system_prompt` | 最高优先级，需重启 Desktop GUI |
| 🥈 SOUL.md | `%LOCALAPPDATA%/hermes/SOUL.md` | 每消息加载，修改后立即生效 |
| 🥉 Skill auto_load | 本 SKILL.md | 框架扫描注入 |

**何时需要**：仅含强制规则（排除表+触发条件+执行命令）的 auto_load skill 需要三重保障。纯参考/指南性 auto_load skill（如 tool-router 的路由分类表）不需要。

**同步维护**：修改 SKILL.md 中的核心规则时，同步更新 SOUL.md 和 config.yaml 的紧凑副本。
