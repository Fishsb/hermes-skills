# Hermes Desktop migration backup without historical sessions

Session-derived pattern from 2026-06-16: user wanted Hermes Desktop fully restorable after OS reinstall, but without restoring historical conversations.

## Goal

Create a knowledge-base backup that can restore configuration, habits, skills, memories, automation, MCP/profile/tool state, and local runtime state while excluding explicit conversation history.

## Recommended destination

```text
D:\lk\Obsidian\Hermes\wiki\配置\Hermes迁移\
├─ README.md
├─ backup-manifest.txt
├─ backup-timestamp.txt
├─ hermes-runtime-no-sessions\
└─ restore-notes\
```

## Source and exclude policy

Source:

```text
C:\Users\lk\AppData\Local\hermes\
```

Include almost everything, but exclude:

```text
sessions/
transcripts/
request_dump_*.json
```

Do **not** exclude `state.db`, `cache/`, `state-snapshots/`, or `backups/` for the default version of this workflow. They may preserve tool state, auth state, rollback points, or habits. Note explicitly that `state.db` may still contain some internal traces; the hard guarantee is only that the explicit `sessions/` payload is excluded.

**Include `bin/`** — this directory contains profile wrapper scripts (hermes-auto, hermes-dev, etc.), ACP launchers (mimo-acp, workbuddy-acp), and tool binaries (rtk.exe, uv.exe, workbuddy-vision). These are small but critical for profile switching and vision workflows. The restore process must verify `bin/` completeness.

## Command pattern

```bash
SRC='/c/Users/lk/AppData/Local/hermes'
DEST='/d/lk/Obsidian/Hermes/wiki/配置/Hermes迁移'
STAMP=$(date +%Y%m%d_%H%M%S)
mkdir -p "$DEST/hermes-runtime-no-sessions" "$DEST/restore-notes"
rsync -a --delete --info=stats2 \
  --exclude='sessions/' \
  --exclude='transcripts/' \
  --exclude='request_dump_*.json' \
  "$SRC/" "$DEST/hermes-runtime-no-sessions/"
printf '%s\n' "$STAMP" > "$DEST/backup-timestamp.txt"
printf 'Backup root: %s\nRuntime source: %s\nExcluded: sessions/, transcripts/, request_dump_*.json\n' "$DEST" "$SRC" > "$DEST/backup-manifest.txt"
```

## Restore notes to export

Export command outputs into `restore-notes/`:

```bash
hermes --version > "$DEST/restore-notes/hermes-version.txt" 2>&1
hermes config path > "$DEST/restore-notes/hermes-config-path.txt" 2>&1
hermes config env-path > "$DEST/restore-notes/hermes-env-path.txt" 2>&1
hermes config check > "$DEST/restore-notes/hermes-config-check.txt" 2>&1
hermes tools list > "$DEST/restore-notes/hermes-tools-list.txt" 2>&1
hermes skills list > "$DEST/restore-notes/hermes-skills-list.txt" 2>&1
hermes mcp list > "$DEST/restore-notes/hermes-mcp-list.txt" 2>&1
hermes cron list > "$DEST/restore-notes/hermes-cron-list.txt" 2>&1
hermes profile list > "$DEST/restore-notes/hermes-profile-list.txt" 2>&1
hermes gateway status > "$DEST/restore-notes/hermes-gateway-status.txt" 2>&1
```

Also generate:

- `env-variable-names.txt` — variable names only, not secret values.
- `hermes-config-summary.txt` — selected config sections: model, delegation, memory, compression, approvals, platform_toolsets, mcp_servers, custom_providers.
- `runtime-tree-top.txt` — top-level backup tree.
- `backup-size-summary.txt` — file/dir/byte counts.
- `excluded-history-check.txt` — verify excluded paths and required config directories.
- `key-file-hashes.txt` — SHA256 for `config.yaml`, `.env`, `state.db`, `memories/MEMORY.md`, `memories/USER.md`.

## Windows/MSYS pitfall

When running Python from Git Bash/MSYS on Windows, paths like `/d/lk/...` may be interpreted by Python as `\\d\\lk...` and fail. Use native Windows paths inside Python snippets:

```python
root = r'D:\lk\Obsidian\Hermes\wiki\配置\Hermes迁移\hermes-runtime-no-sessions'
```

Shell commands and rsync can use `/d/...`; Python filesystem checks should use `D:\...`.

## README content requirements

Write `README.md` in the backup root with:

1. Backup goal and exact source/destination paths.
2. Included vs excluded list.
3. Self-check result block: size/counts and exclusion checks.
4. Restore steps back to `C:\Users\lk\AppData\Local\hermes\`.
5. External dependencies to re-check: Obsidian vault, API keys, Node/npm/uv/Python, browser bridge, OAuth, proxy client, firewall/ports.
6. Agent self-configuration pointers: `schema/AGENTS.md`, memories, `SOUL.md`, `skills/`, current tool/config snapshot notes.
7. Warning that `.env` is sensitive and must not be publicly synced.

## Verification checklist

- [ ] `sessions_dir_exists: False`
- [ ] `transcripts_dir_exists: False`
- [ ] no `request_dump_*.json` files found
- [ ] `config.yaml`, `.env`, `skills/`, `memories/`, `scripts/`, `cron/`, `profiles/` exist
- [ ] `bin/` directory exists and contains expected files (workbuddy-vision, uv.exe, etc.)
- [ ] backup size/count summary is non-zero and plausible
- [ ] README exists and names the restore path
- [ ] restore-notes include current Hermes CLI state
