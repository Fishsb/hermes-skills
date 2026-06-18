# 2026-06-16 Context Slimdown 234 Pattern

> **上下文**：[主 SKILL.md →](../SKILL.md) | **索引**：[所有参考文件](./_index.md)
> **同级文件**：[执行笔记](./20260616-context-slimdown-execution.md) | [会话召回地图](./20260616-context-compression-recall.md)
> **派生自**：[初始执行记录](./compression-example-history.md)

Session-derived pattern

## Durable lessons

1. **Treat numbered selection as strict scope.** If the user replies with numbers such as `234`, execute only those options. Explicitly state which high-impact option was not changed.
2. **Backup before active runtime edits.** For skills/memory/profile edits, create timestamped backups under `~/AppData/Local/hermes/backups/` before writing.
3. **For oversized class-level skills, split, don't delete semantics.** Keep SKILL.md as a routing/execution contract and move bulky runtime detail to `references/<topic>.md`.
4. **Validate exact required terms after slimming.** For orchestration skills, verify frontmatter, code fences, linked references, and critical trigger/contract literals.
5. **Separate mutable execution from read-only audit.** Main agent can perform serial file writes; subagents are useful for read-only config/token audit cross-checks.
6. **Report unselected top finding separately.** If the biggest token win was not in the selected scope, preserve it as a next-step recommendation rather than silently applying it.

## Verification checklist used

- Compare before/after chars and lines.
- Parse YAML frontmatter and check description length.
- Confirm code fences are balanced.
- Confirm all linked `references/` files exist.
- Re-read Memory/User after compression and check critical pointers remain.
- Run active Hermes path/tool/MCP commands for token audit evidence.

## Example acceptance result

- `mimo/SKILL.md`: 20072 chars / 377 lines → 7269 chars / 143 lines.
- `mimo/references/dag-wave-routing.md`: added with detailed runtime contracts.
- Memory: 1842 chars → 1273 chars.
- User Profile: 1168 chars → 610 chars.
- Explicitly not changed: forced MiMo-first system prompt, because it was recommendation #1 and user selected only `234`.

---

**关联参考**：[全局索引](./_index.md) | [SKILL.md →](../SKILL.md) | [执行笔记](./20260616-context-slimdown-execution.md) | [会话召回地图](./20260616-context-compression-recall.md)
