# Related Community Repos — Hermes Skills

> Reference repositories discovered during public repo setup. Consult these for README structure, installation patterns, documentation standards, and naming conventions.

## Repositories

| Repo | Description | Key Takeaways |
|------|-------------|---------------|
| [veawho/via54Skills](https://github.com/veawho/via54Skills) | Personal Hermes / Claude / OpenClaw / Codex skills — bilingual README | Bilingual README with language toggle; `via54` naming prefix convention; strict frontmatter regex rules |
| [The-Aetheris/skills](https://github.com/The-Aetheris/skills) | Collection of reusable Hermes Agent skills | Minimal but clean; simple skill table; installation via clone or hub |
| [lukemcqueen/hermes-cortex](https://github.com/lukemcqueen/hermes-cortex) | Full Hermes config + skills + memory + observability | Badge headers; detailed prerequisites; multi-method install; extensive feature listing |
| [kepano/obsidian-skills](https://github.com/kepano/obsidian-skills) | Agent skills for Obsidian — 35k+ stars | Professional documentation; clear install steps; `.claude-plugin` directory structure |
| [LuoJiangYong/muse-video-skill](https://github.com/LuoJiangYong/muse-video-skill) | AI video pre-production Skill for Hermes | Single-skill repo pattern; clean README with badges |

## Documentation Pattern (from veawho/via54Skills)

Recommended structure for public Hermes skills repos:

1. **Bilingual README** — Chinese default, English secondary
   - Language toggle banner at top
   - TOC with Chinese + English headings
   - Key technical terms kept in English
2. **Skill catalog** — organized by category with descriptions
3. **Installation** — multiple methods (clone, sparse checkout, hub install)
4. **Compatibility** — list compatible agents (Hermes, Claude Code, OpenClaw, etc.)
5. **License** — MIT recommended for community sharing

## Discovery Query

```bash
curl -s "https://api.github.com/search/repositories?q=hermes+skill+OR+hermes-agent+skill&sort=updated&per_page=10"
```

Use `skills:github-project-discovery` for full GitHub project search workflow.
