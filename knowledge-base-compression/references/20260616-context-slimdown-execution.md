# 2026-06-16 Context Slimdown Execution Notes

## What happened

A context-token audit found three high-impact bloat sources:

1. `hermes-agent` main `SKILL.md` was a full manual (~1029 lines / ~46.8KB).
2. Built-in hot memory files were near their configured limits:
   - `memories/MEMORY.md`: ~2065 chars before cleanup.
   - `memories/USER.md`: ~1298 chars before cleanup.
3. Long-lived but non-hot details were stored in Memory/User Profile:
   - Desktop GUI session-fragmentation bug narrative.
   - D14-D19 gaokao/career decision details.
   - A temporary DEBUG image/OCR note.

## Durable workflow learned

When the user explicitly approves compression changes, execute only the approved scope and leave everything else untouched.

Safe sequence:

1. Load relevant compression / Obsidian / skill-authoring skills.
2. Locate the real runtime memory files before editing:
   - `C:/Users/lk/AppData/Local/hermes/memories/MEMORY.md`
   - `C:/Users/lk/AppData/Local/hermes/memories/USER.md`
3. Back up every file to be edited under `C:/Users/lk/AppData/Local/hermes/backups/<descriptive-run-id>/`.
4. Move long detail to Obsidian first, then replace hot memory with a short pointer.
5. For oversized class-level skills, preserve detail in `references/` and rewrite the main `SKILL.md` as a router/index.
6. Verify by reading back the files and checking residue keywords.

## Concrete outcome from this run

- `hermes-agent` main skill: ~1029 lines / 46.8KB → 134 lines / 5.2KB.
- Details moved into `hermes-agent/references/`:
  - `cli-reference.md`
  - `slash-commands.md`
  - `config-providers-toolsets.md`
  - `security-privacy-voice.md`
  - `spawning-instances.md`
  - `durable-background-systems.md`
  - `windows-quirks.md`
  - `troubleshooting.md`
  - `where-to-find-things.md`
  - `contributor-guide.md`
- Memory cleanup:
  - Removed temporary `DEBUG` image/OCR note.
  - Archived Desktop GUI session-fragmentation bug out of hot Memory.
- User Profile cleanup:
  - Replaced D14-D19 + family-background details with a short Obsidian pointer.
- Obsidian archive:
  - `D:/lk/Obsidian/Hermes/wiki/记忆/20260616-上下文瘦身迁移归档.md`

## Verification checks used

- `skill_view("hermes-agent")` loads the compact router.
- YAML frontmatter parses.
- Markdown code fences are balanced.
- All linked `references/*.md` files exist.
- `MEMORY.md` no longer contains `DEBUG` or the Desktop GUI long bug entry.
- `USER.md` no longer contains long `D14性格` / family-background text.
- `USER.md` contains the Obsidian pointer.
- The Obsidian archive contains both D14-D19 and Desktop GUI bug details.

## Pitfall

The CLI command `hermes memory --help` only exposed provider setup/status/off/reset, not fine-grained built-in-memory replace/remove operations. In this runtime, the built-in memory source of truth was the file pair under `AppData/Local/hermes/memories/`. If a dedicated memory tool is unavailable, file editing is acceptable only after backup and read-back verification.
