---
name: lk-memory-index
description: 'Compressed memory index for user lk: Hermes paths, providers, Obsidian, WorkBuddy, smart archive, Kanban, proxy, Browser MCP. Use as pointer map instead of loading long notes.'
version: 1.4.0
author: Hermes Agent
license: private
metadata:
  hermes:
    tags:
    - memory
    - profile
    - hermes
    - environment
    - lk
---

# lk-memory-index

> Purpose: package long environment facts that would otherwise clutter persistent MEMORY. MEMORY should keep only short wake-up pointers; detailed operational context lives here.

<!-- Agent: English main body is the primary read path. Chinese source is archived in `references/中文原文.md` for human lookup and rollback. -->

## Linked References

- `references/知识库对齐流程.md` — audit and alignment procedure when skill statistics drift from the real knowledge base.
- `references/hermes-memory-audit.md` — Hermes persistent storage audit: memory + user profile deduplication and compression workflow.
- `references/中文原文.md` — full Chinese original archive before English-first rewrite.

## Hermes Three-Layer Memory Architecture

Hermes uses two persistent stores plus one expandable skill layer:

| Layer | Tool | Capacity | Content policy | Current baseline |
|---|---|---:|---|---:|
| **memory** | `memory(target='memory')` | 2,200 chars | Skill pointers only, ≤80 Chinese chars per item; no long details | baseline: 27 chars (1%) |
| **user profile** | `memory(target='user')` | 1,375 chars | Identity, interaction preferences, hard rules, skill pointers | baseline: 289 chars (21%) |
| **skill files** | `skill_manage` / `skill_view` | effectively unlimited | Long content, technical details, workflows | baseline: 31 skills |

Three constraints:
1. **No details in memory** — keep only ≤80-char skill pointers; put details in skills.
2. **Deduplicate across stores** — keep one authoritative copy of each rule. Prefer user profile over memory; prefer skill over profile for long detail.
3. **Move long content into skills** — technical details, config parameters, and workflows belong in skill files; profile keeps only pointers.

## Memory Compression Principles

- MEMORY stores only short wake-up pointers.
- User profile stores only facts that should affect every conversation: identity, preferences, hard rules.
- Details go into skills or the Obsidian knowledge base.
- Do not store short-term task progress, PR/issue/commit IDs, one-off results, or real-life information.
- When modifying long-term user rules, update the corresponding skill or knowledge-base note first, then keep only a pointer in MEMORY if needed.
- Deduplicate memory vs user profile. If duplicated, keep the more appropriate copy and remove the other.

## Knowledge Base Architecture — After LLM-Wiki Restructure

**Path**: `D:\lk\Obsidian\Hermes\`  
**Mode**: LLM-Wiki three-layer architecture, rebuilt on 2026-06-12  
**Backup**: `D:\lk\Obsidian\Hermes.backup.20260612`

### Three Layers

| Layer | Directory | Responsibility | File count baseline |
|---|---|---|---:|
| Raw source layer | `raw/` | Raw materials added by humans: screenshots, chat exports, unprocessed docs. AI is read-only here. | 2 |
| Wiki layer | `wiki/` | AI notes organized by type; AI has full maintenance authority for this layer. | 217 |
| Schema layer | `schema/` | AI rules: `AGENTS.md`, interaction rules. Modify cautiously. | 3 |

### `wiki/` Subdirectories

| Directory | Content | File count baseline | Entry |
|---|---|---:|---|
| `wiki/工具/` | AI toolchain: Hermes, Codex++, DeepSeek, plus configuration records | 20 | `📂 工具.md` |
| `wiki/环境/` | WSL2, Fcitx5 Chinese input, troubleshooting, Windows maintenance | 6 | `📂 环境.md` |
| `wiki/工作流/` | Document automation, Excel template filling | 4 | `📂 工作流.md` |
| `wiki/项目/` | GitHub project library with 10 categories, technical research, AI agent applications, 2026 Hubei employment research | 101 | `📂 项目.md` |
| `wiki/配置/` | Personal profile, AI-rule backups, Kanban, troubleshooting, hardware, maintenance reports, archive distillation | 39 | `📂 配置.md` |
| `wiki/skill/` | Self-authored skill backups: skill development and general productivity | 13 | `📂 skill.md` |
| `wiki/记忆/` | Memory index, templates, temp sessions, session archive index | 29 | `📂 记忆.md` |
| `wiki/概念/` | Abstract concepts | 2 | `📂 概念.md` |

### Key Index Pages

- `wiki/index.md` — top-level home page: category navigation, index boards, tag index, usage rules.
- `wiki/log.md` — knowledge-base change timeline.
- `wiki/配置/AI 检索索引.md` — AI behavior-to-file routing map.
- `wiki/配置/知识库结构图谱.md` — vault knowledge map from the AI perspective.
- `wiki/配置/排障总览.md` — global troubleshooting index.
- `wiki/配置/任务看板.md` — active task tracking.
- `wiki/配置/全库互链索引.md` — fallback link-integrity index.
- `schema/AGENTS.md` — new-agent startup self-configuration guide, 7 steps.
- `schema/交互规则.md` — full AI behavior rules.

### Dataview Auto-Indexes

All 23 `📂 directory-name.md` index notes were migrated to Dataview `TABLE` queries that auto-pull files in the same folder by their frontmatter `description` and `status` fields. For new notes, maintain frontmatter and the index updates automatically.

### Tag System

- `type/index`, `type/config`, `type/guide`, `type/ref`, `type/backup`, `type/note`
- `domain/ai`, `domain/railway`, `domain/wsl`, `domain/office`, `domain/search`, `domain/windows`
- `tech/python`, `tech/typescript`, `tech/rust`, `tech/shell`
- Troubleshooting: `#排障`, `#已解决`, `#未解决`, `#放弃`
- People: `#小六`, `#小F`

### Agent Access Path

1. New agent reads `schema/AGENTS.md`.
2. Follow the 7-step self-configuration path; first read `wiki/配置/个人画像.md`.
3. For fast lookup, use `wiki/配置/AI 检索索引.md` as the behavior-to-file routing table.
4. Search order: AI search index lookup → search `description` → search tags → search filenames → full-text search → `session_search`.

## Hermes / Desktop / Cron

- After changing anything visible in Hermes Desktop UI, such as profiles or config entries, restart or refresh the desktop app so the UI sees the change.
- If Desktop has no gateway channel, cron `deliver=origin/all` can fail. Agent-mode runs still create independent sessions visible in the list.
- Details live in skill `cron-automation-patterns`.
- Smart-archive subagent details — `parent_session_id` detection, `[child]` tags, multi-profile scanning — also live in `cron-automation-patterns`.

## Environment Backup

Full environment backup workflow lives in devops skill `agent-migration-backup`. It covers:

- Four KB documents covering: main configuration including omissions, recovery guide, and script source code.
- Auto-recovery script: `restore-hermes.sh`, 18 automatic steps.
- Key lessons: back up full script source; record `.env` variable names but not values; shell aliases may not live in `.bashrc`; do not back up huge `state.db` files.

## Smart Archive Output Format

- Single bordered box with left/right split or left/right list layout.
- Two groups: `🟡闲置` and `🟢活跃`.
- Time aligned to the right border.
- `[1]` is shifted left compared with `[2]+`; continuation lines keep indentation.
- Do not auto-prune old sessions.
- Before every push, delete old cron sessions automatically.
- CJK width may visually overflow; this is an acceptable tradeoff.
- On Windows, `smart-archive.py` must read `state.db` via `LOCALAPPDATA`; `APPDATA/Roaming` points to an empty database.
- Archive path after the 2026-06-12 LLM-Wiki restructure: `OBSIDIAN_VAULT = Path("D:/lk/Obsidian/Hermes/wiki/记忆/会话归档")`.
- `backup-session.py` has been updated to the same path.

## ACP / WorkBuddy / Vision

- ACP subagent workflow: see skill `dispatcher-workbuddy`.
- Wrapper scripts must include their own arguments and must not depend on `$@`.
- `delegation.base_url=acp://copilot` must be configured.
- WorkBuddy login/authentication is required.
- WorkBuddy ACP / vision details: see skill `workbuddy-vision-integration`.
- GLM-4.6V image analysis uses WorkBuddy; ACP `--serve` port is `18999`.
- For large images, first save them into the knowledge base, then pass the file path to WorkBuddy.
- After results return, clean original images under `composer-images`.
- Old script `~/.hermes/scripts/vision-workbuddy.py` has been removed.
- If a skill script's `curl_post` fails to parse a pure-JSON connect response, switch it to `urllib`.

## Browser MCP

- `agent-browser-mcp` is configured for operating the real Chrome browser while preserving login state.
- Path: `C:\Users\lk\agent-browser-mcp`
- Chrome extension loaded: `TMWD CDP Bridge`
- Hermes config includes `mcp_servers.agent_browser`.
- Before use, start the service, then open a fresh Hermes session.
- Tool prefix: `mcp_agent_browser_*`.

## Kanban Multi-Worker Setup

- Related skill: `kanban-workflow`.
- `default` is responsible for task decomposition and acceptance checks; workers execute tasks.
- `kanban.db` has been initialized.
- Three workers are configured with the Kanban toolset.
- Keeping Desktop tabs open avoids worker cold starts and approximates zero cold-start operation.

## Obsidian

- Obsidian configuration is archived in the knowledge base under the `#obsidian` tag.
- Current key plugins: Dataview `v0.5.70`, Metadata Menu, Templater.

## Network / Environment

- Cloud-server proxy node lives in skill `proxy-node`: VLESS+REALITY, `192.3.250.74:15727`, SNI `aws.amazon.com`.
- For external-network connectivity problems, load skill `proxy-node` first.
- WSL migration details live in skill `hermes-multi-instance`.

## Human Reference

- Chinese source archive: `references/中文原文.md`.
- Keep Chinese trigger words in the description so Chinese requests still recall this skill.
