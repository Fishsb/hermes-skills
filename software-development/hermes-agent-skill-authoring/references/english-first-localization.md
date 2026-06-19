# English-First Localization Pattern for User Skills

Use this pattern when converting Chinese-heavy user-local skills into an Agent-primary English form while preserving Chinese usability for the user.

## Pattern

1. **Read the current skill first** with `skill_view` or `read_file` before editing.
2. **Archive the full Chinese original** into `references/中文原文.md` before rewriting `SKILL.md`.
3. **Rewrite `SKILL.md` with English as the main Agent read path**:
   - Keep frontmatter valid.
   - Keep the skill name stable.
   - Prefer bumping `version` by a minor patch level.
   - Add an Agent-facing comment such as: `English main body is the primary read path; Chinese source is archived in references/中文原文.md`.
4. **Preserve Chinese recall triggers**:
   - Keep Chinese trigger words in `description` and/or `metadata.hermes.trigger_words`.
   - Keep mandatory Chinese output literals if the workflow requires exact user-visible text, e.g. `▶ MiMo 拆分 N 个任务`.
   - Keep Chinese filenames, Obsidian paths, tags, and wiki note names unchanged.
5. **Add a pointer to the archive** in the references section of the rewritten skill.
6. **Verify after writing**:
   - `SKILL.md` starts with valid YAML frontmatter.
   - Full file size stays under Hermes limits.
   - CJK text remains only in allowed categories: trigger words, exact output literals, filenames, paths, tags, wiki note names, and the archive pointer.
   - `references/中文原文.md` exists and contains the original.

## When to Use

- User asks to make Agent configuration, tools, plugins, skills, or operational rules English-readable.
- A Chinese skill is loaded frequently by agents and should be optimized for model comprehension.
- The user still needs Chinese lookup, triggers, or rollback safety.

## Avoid

- Do not remove Chinese trigger words entirely; Chinese user prompts may stop recalling the skill.
- Do not translate path/file names or Obsidian wiki note names.
- Do not bury all Chinese in `SKILL.md`; archive the original in `references/中文原文.md` instead.
- Do not skip read-back validation after writing.
