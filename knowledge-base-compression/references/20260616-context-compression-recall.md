# 2026-06-16 Context / Memory Compression Recall

> **上下文**：[主 SKILL.md →](../SKILL.md) | **索引**：[所有参考文件](./_index.md)
> **同级文件**：[执行笔记](./20260616-context-slimdown-execution.md) | [234 模式](./20260616-context-slimdown-234-pattern.md) | [初始执行记录](./compression-example-history.md)

## Trigger

Use this reference when a user asks to inspect current context/token usage, previous compression work, memory/profile/SOUL/skill slimming, or knowledge-base compression rules.

## Durable lesson

Before recommending or executing compression, first recover the user's existing compression rules and recent-state decisions from both session history and the Obsidian knowledge base. Do not treat old config backups as current truth.

## Proven lookup sequence

1. Load the compression umbrella skill and `lk-memory-index` when available.
2. Search prior sessions for:
   - `上下文压缩`, `记忆压缩`, `知识库压缩`, `context compression`
   - `memory_char_limit`, `user_char_limit`, `SOUL.md`, `USER.md`, `MEMORY.md`
   - `config.yaml.bak`, `按最近的状态`, `token-savior`, `PLUR`
3. Search the Obsidian vault under `D:/lk/Obsidian/Hermes/` for:
   - `schema/交互规则.md`
   - `wiki/配置/Hermes 记忆压缩与规则分层.md`
   - `wiki/配置/知识库压缩与存储容量管理.md`
   - `wiki/配置/Agent记忆系统优化原则.md`
   - `wiki/配置/Agent记忆注意力机制.md`
   - `wiki/配置/Hermes热记忆上下文优化-20260609.md`
   - `wiki/配置/AI 检索索引.md`
   - `wiki/配置/个人画像.md`
   - `schema/AGENTS.md`
   - `wiki/记忆/_template-记忆源条目.md`
4. Reconcile the evidence using recency and authority:
   - current runtime state and explicit user correction first;
   - persistent memory/profile/SOUL/skills/KB next;
   - old `config.yaml.bak-*` files only as historical references;
   - ask before changing stored rules or config.

## Rules recovered in the session

- Compression criterion: **do not reduce task execution effectiveness**.
- Injection sequence: `先检索 → 再筛选 → 再压缩 → 最后注入`.
- Memory gate: `relevant+trusted → keep, else skip. execution first.`
- Memory should store only short wake-up pointers, ideally ≤80 Chinese characters per entry.
- User Profile stores stable identity, hard preferences, and global interaction rules.
- Skills store reusable procedures and tool/config workflows.
- Obsidian stores long documents, raw evidence, session archives, and research notes.
- Do not store one-off task progress, PR numbers, commit hashes, TODO states, or transient results in memory.
- Important compressed memories should retain source references and bias-risk notes.

## Important historical correction

A previous recovery flow temporarily used `config.yaml.bak-*` as if it represented the latest state. The user corrected this with “按最近的状态”. Future compression/config-recovery analysis must not blindly restore from backups. Prefer current state, explicit user memory, and verified knowledge-base records.

## Useful historical session IDs

- `20260616_023951_c00da8` — highest-priority correction: old config backups are historical only; use recent/current state.
- `20260616_023711_fb8399` — earlier broad recovery investigation; useful for context, but some conclusions were superseded.
- `cron_db5090d04b33_20260616_020947` — Skill Factory pending note about MiMo exclusion conditions for low-complexity tasks.

---

**关联参考**：[全局索引](./_index.md) | [SKILL.md →](../SKILL.md) | [执行笔记](./20260616-context-slimdown-execution.md) | [234 模式](./20260616-context-slimdown-234-pattern.md)
