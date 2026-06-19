# Session Note: Adapting `skill-cleaner` into a Hermes Skill

## Context

A session adapted Peter Steinberger's `steipete/agent-scripts` `skill-cleaner` from a Codex/OpenClaw-oriented TypeScript skill into a Hermes-local read-only maintenance skill.

## Useful reusable pattern

1. Keep upstream source as attribution/reference, not as the runtime entrypoint:
   - `references/original/SKILL.md`
   - `references/original/scripts__skill-cleaner.ts`
   - `references/original/scripts__skill-cleaner.test.ts`
   - `references/original/agents__openai.yaml`
2. Create a Hermes-native `SKILL.md` that defines the local scope and safety boundary.
3. Put deterministic adapted logic in `scripts/`, not in the skill body.
4. Prefer read-only analysis first; destructive actions require user confirmation and backup.
5. Verify in layers:
   - frontmatter parse
   - script syntax/compile check
   - script runs on a small root
   - script runs on the full active skill root
   - `hermes skills list` sees the new skill if loader is not stale

## Important implementation lesson

Do not attempt to create a rich skill plus scripts using one very large `execute_code` call. The stream can time out before the tool call is delivered. Split into small calls:

1. create directories
2. download or write reference files
3. write `SKILL.md`
4. write script skeleton
5. patch in functions/sections
6. run verification

## Hermes-specific adaptation points

Original `skill-cleaner` depends on Codex/OpenClaw concepts such as `codex debug prompt-input`, `~/.codex/config.toml`, `~/.codex/models_cache.json`, Codex session logs, and OpenClaw/Clawd paths. A Hermes adaptation should replace these with Hermes-local filesystem scanning first and only add live inventory/session-log support when a stable Hermes interface exists.

Current proven landing pattern on Windows Desktop:

```text
C:/Users/lk/AppData/Local/hermes/skills/maintenance/skill-cleaner/SKILL.md
C:/Users/lk/AppData/Local/hermes/skills/maintenance/skill-cleaner/scripts/hermes_skill_cleaner.py
C:/Users/lk/AppData/Local/hermes/skills/maintenance/skill-cleaner/references/original/
```

## Safety rule

For cleanup tools, reports are not permission to mutate. Treat findings as candidates. Back up and ask before editing, merging, deleting, disabling, or moving any skill.
