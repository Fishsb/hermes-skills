---
name: hermes-agent-skill-authoring
description: "Use when authoring or editing Hermes Agent SKILL.md files: frontmatter, structure, validation, duplicate checks, and verification."
version: 2.2.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    auto_load: true
    tags: [skills, authoring, hermes-agent, conventions, skill-md]
    related_skills: [plan, requesting-code-review]
---

# Authoring Hermes-Agent Skills

## Overview

创建或编辑 Hermes SKILL.md 的执行手册。本 skill 的 auto_load 只保留最核心的前置条件和快速参考；完整结构规范、工作流细节在引用文件中按需加载。

**使用链**：`Skill Factory`（归档时生成建议）→ 用户确认 → 本 skill（执行编辑）。

## When to Use

| ✅ 创建新 skill | ✅ 编辑已有 skill |
|----------------|------------------|
| 用户明确要求创建 | 新增工作流步骤、pitfall、触发词 |
| 无现有 skill 覆盖该流程 | 优化边界条件、回退路径 |
| 流程可复用（非一次性） | 回归保持旧行为的基础上扩展 |

## Required Frontmatter

```yaml
---
name: my-skill-name               # 小写+连字符，≤64 chars
description: Use when <trigger>.  # ≤1024 chars
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [short, tags]
    related_skills: [other-skill]
---
```

**硬要求**：`name` + `description` + 非空正文。`version`/`author`/`license`/`metadata` 非强制但每 skill 都有。

## 最小可用结构

`Overview` + `When to Use` + 可执行工作流/正文 + `Common Pitfalls` + `Verification Checklist`

## 大小限制

| 字段 | 上限 |
|------|------|
| description | 1024 chars（强制） |
| 总 SKILL.md | 100,000 chars（~36k tokens） |
| 推荐范围 | **8-15k chars**，超 20k 应拆到 references/ |

## 目录放置

```
skills/<category>/<name>/SKILL.md
```
选最接近的现有分类，不随意新增顶层分类。

## Common Pitfalls

1. **`skill_manage(action='create')` 对 in-repo 无效** — 它写入 `~/.hermes/skills/` 而非 repo 树。in-repo 创建用 `write_file`。
2. **frontmatter 前有空白** — 第 0 字节必须是 `---`。
3. **description 太泛** — 应以 "Use when ..." 描述触发类，而非单个任务。
4. **创建新 skill 前没检查重复** — 先 `ls skills/<category>/` 看 2-3 个同类 skill。优先扩展现有 skill。
5. **auto_load skill 缺排除表** — 强制规则必须引用排除表作为范围边界。
6. **大工具调用 payload 超时** — 创建带 scripts/references 的 skill 时分批写：先创建目录 → 写 SKILL.md → 写脚本骨架 → 逐块追加/补丁。
7. **漏查关联文件** — 编辑 skill 后必须检查其 references/ 目录下的文件、related_skills 中引用的其他 skill、以及任何交叉引用是否同步更新。一次编辑往往需要同步更新 2-3 个文件。
8. **修复不彻底** — 应用 FIX_LOOP: 复查缺陷 → 修复 → 重新审查，重复直到零残留。确认修改完整后再交付，不要遗留已知问题。
9. **拆分后互联断裂** — 将大文件（>15KB）拆入 references/ 后，如果只更新了主文件却没创建/更新 `references/_index.md` 或没给子文件添加双向链接（上下文 + 关联参考），会形成不可追踪的断链。必须按 `references/cross-linking-pattern.md` 的模板逐层验证。

## 核心规则嵌入规范（Rule Embedding Pattern）

> 本 session 教训：MEMORY 里的被动规则会被 Agent 忽略，必须有触发门 + 流程内嵌 + 输出前自检才能约束行为。

### 问题诊断

MEMORY 中的规则（如 FIX_LOOP）本质上是被动文本。Agent 输出时不会主动扫描 MEMORY 块逐条判断是否命中。导致规则失效的常见原因：

| 原因 | 表现 |
|------|------|
| **触发条件模糊** | "工具/技能/插件更新" → Agent 归类偏窄，认为自己做的不是「更新」 |
| **无层级区分** | 多条规则平铺 → 竞争不过更紧急的执行信号 |
| **无打断机制** | 没有强制停下来核验的步骤 → 一路执行到输出 |
| **无流程锚点** | 规则不附着在 workflow step 上 → 根本不会想起 |

### 三通道注入（3-Channel Injection）

确保核心规则真正约束行为的推荐模式：

```
Channel A: MEMORY 规则（触发门 + 流程 + 格式）
  └─ 明确的 4 个触发条件门（① 编辑 SKILL.md / ② ≥3 次批量写 / …）
  └─ 执行流程（review → fix → re-review → 零残留）
  └─ 输出格式（英文先中文后）

Channel B: Skill workflow 内嵌
  └─ 在相关 skill 的 workflow step 中插入强制锚点
  └─ 典型锚点：Step 间的 check（🎯 标记）或输出前 GATE
  └─ 输出前：嵌入 FIX_LOOP GATE（问核验表）

Channel C: GATE 门 + 验证清单
  └─ 输出前逐项核验，任意一项「否」→ 停止 → 修复 → 重新 review
```

### 检查清单

注入规则时确认：
- [ ] 触发条件用编号门列出，无模糊表述
- [ ] 每个条件客观可判定（不是"复杂时"而是"≥3 次"）
- [ ] 规则附着在至少一个 workflow step 上
- [ ] 输出前有 GATE 验证
- [ ] 违反了知道怎么修复

> 🚨 **输出前必须逐一核验以下三项，任意一项为「否」则停止输出，修复后再 review**

| # | 核验项 | 判定 |
|---|--------|:----:|
| 1 | **匹配触发条件？** 本次操作是否命中 FIX_LOOP 硬性条件（① SKILL.md 编辑 / ② ≥3 次批量写 / ③ 持久状态修改 / ④ delegat_task 含上述）？ | 是/否 |
| 2 | **零残留？** 是否存在任何已知缺陷、链接断裂、前后不一致？若有 → 立即修复 → 循环 review | 是/否 |
| 3 | **进度可见？** 执行过程中是否在以下关键节点输出了进度：每完成一个子步骤、≥3 次工具调用无输出时强制插入、遇到异常即时说明？ | 是/否 |
| 4 | **输出格式？** 最终报告是否按「先高保真英文摘要 → 再中文说明」顺序？ | 是/否 |

四项全部「是」后才允许结束输出。

## Verification Checklist

- [ ] 编辑完成后的 skill 可正常加载（无 frontmatter 解析错误）
- [ ] 编辑后的所有 references/ 文件路径正确，内容同步更新
- [ ] related_skills 中的关联 skill 引用可解析
- [ ] 如果修改了职责边界，同步更新了交叉引用的 skill
- [ ] 若文件 >15KB 或有拆分操作，按 `references/cross-linking-pattern.md` 验证所有跨文件链接完整性
- [ ] 应用 FIX_LOOP 确认零残留后再结束输出

---

## 渐进式参考层

> 核心规则已全部在 SKILL.md 中。深层细节在引用文件中按需加载：

| 场景 | 加载方式 |
|------|---------|
| 需要**完整结构模板、章节放置规则、Auto-Load 设计原则** | `skill_view(name='hermes-agent-skill-authoring', file_path='references/skill-structure.md')` |
| 需要**完整编辑工作流、优化补丁流程、验证清单** | `skill_view(name='hermes-agent-skill-authoring', file_path='references/skill-editing-workflow.md')` |
| 需要**拆分与互联规范、跨文件链接模板**（文件 >15KB 时必读） | `skill_view(name='hermes-agent-skill-authoring', file_path='references/cross-linking-pattern.md')` |

## 变更记录

- **v2.2.0** — 新增「核心规则嵌入规范（Rule Embedding Pattern）」节 + 三通道注入方法；FIX_LOOP GATE 扩展为 4 问（含进度可见性 Q3）；PROGRESS_TICK 嵌入；新增 references/rule-embedding-pattern.md 参考文件。
- **v2.1.0** — 新增 `references/cross-linking-pattern.md`：拆分与互联规范模板；新增 Pitfall #9（拆分后互联断裂）；验证清单增加跨文件链接完整性检查。
- **v2.0.0** — 渐进式拆分：SKILL.md ~4KB（auto_load），深层工作流/结构规范移到 `references/` 按需加载。原 22KB → 4KB，↓82% 常驻上下文。
- **v1.1.0** — Auto-load 设计指南、回归验证标准
- **v1.0.0** — 初始版本
