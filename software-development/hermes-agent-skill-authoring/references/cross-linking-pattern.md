# Cross-Linking Pattern for Split Skills

> 通用互联规范：当技能/文件因过大（>15KB）或多主题混杂需要拆分时，使用本模式维护所有片段之间的可追溯连接。
> 加载方式：`skill_view(name='hermes-agent-skill-authoring', file_path='references/cross-linking-pattern.md')`

## 前置条件

当下任一条成立时，应拆分并应用互联：

1. **文件 >15KB** — 单文件信息密度过高，压缩或编辑可能丢失细节
2. **多主题混杂** — 一个文件中包含 2+ 个逻辑独立的主题，编辑一处会误伤另一处
3. **高价值长文档需要降级到 references/** — 重要细节多但不适合留在热存储

## 互联模板

### 主文件（SKILL.md）

自身保持为路由/执行契约（8-15KB），不包含深层细节。深层细节移入 `references/`。

```
## References

- `references/_index.md` — **全局索引**（首个必读）：所有 reference 文件的前后向链接地图。
- `references/<topic>.md` — 具体参考 1
- `references/<topic2>.md` — 具体参考 2
```

### 子文件（references/<topic>.md）

每个子文件必须包含以下双向链接：

```markdown
# 文件头部（frontmatter 之后，正文之前）
> **上下文**：[父文件](../SKILL.md) | **索引**：[所有参考文件](./_index.md)
> **同级文件**：[A](./a.md) | [B](./b.md)

# 文件尾部（结尾空行前）
---
**关联参考**：[全局索引](./_index.md) | [父文件 →](../SKILL.md) | [同级 X](./sibling-x.md)
```

### 全局索引（references/_index.md）

```markdown
# <Skill-Name> — Reference Index

> 全局互联地图：本目录所有 reference 文件的依赖关系。

## 文件全景

```
SKILL.md (主入口 — 路由契约 + 执行流程)
│
├── references/
│   ├── _index.md                              ← 本文件
│   ├── <topic-a>.md
│   └── <topic-b>.md
```

## 依赖矩阵

| 文件 | 父级 | 同级引用 | 被引用者 |
|------|------|---------|---------|
| `../SKILL.md` | — | — | `_index.md` 下所有文件 |
| `<topic-a>.md` | `../SKILL.md` | `<topic-b>.md` | `../SKILL.md` |

## 维护规则

- **新增 reference** → 立即更新 _index.md：全景图 + 依赖矩阵
- **删除 reference** → 从全景图、依赖矩阵同步移除
- **修改 reference** → 检查其父/同级/被引用关系是否仍准确
```

## 验证清单

- [ ] `references/_index.md` 列出了该 skill 下所有 reference
- [ ] 每个 reference 文件头部包含上下文链接（父文件 + 同级）
- [ ] 每个 reference 文件尾部包含关联参考（_index.md + 父文件）
- [ ] 所有链接路径可解析（文件存在、路径正确）
- [ ] 主文件 References 节指向了 `_index.md`

## Pitfalls

- **只更新主文件忘了更新 _index.md** → 形成断链。必须自上而下逐层验证。
- **指针不可达**：归档到外部存储后路径错误 → 信息彻底丢失。归档后必须验证目标存在。
- **过度拆分**：<5KB 的文件没必要拆分，否则碎片化降低效率。每个文件 5-15KB，职责单一。
- **一次性拆分过多**：每次拆分 1-3 个文件，避免大规模改动导致验证遗漏。多次迭代优于一次大改。
