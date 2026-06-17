# Hermes config post-cleanup verification

Use this after a Hermes config cleanup, migration, profile removal, or MCP/tool pruning session. It captures the durable verification pattern, not the one-off session state.

## Goal

Confirm the active Desktop/Gateway setup is using the intended config, no legacy profile/config residue is influencing runtime, MCP tool counts match the token budget strategy, and Windows log rollover is not spamming the terminal.

## Read-only verification commands

```bash
# Active config and env paths
hermes config path
hermes config env-path

# Critical config fields: MCP profiles, disabled servers/platforms, logging
python - <<'PY'
from pathlib import Path
import yaml
p = Path.home() / 'AppData/Local/hermes/config.yaml'
data = yaml.safe_load(p.read_text(encoding='utf-8-sig'))
ms = data.get('mcp_servers', {}) or {}
print('token_savior_profile=', (ms.get('token-savior') or {}).get('env', {}).get('TOKEN_SAVIOR_PROFILE'))
print('uumit_enabled=', (ms.get('uumit') or {}).get('enabled'))
print('weixin_enabled=', ((data.get('platforms') or {}).get('weixin') or {}).get('enabled'))
print('logging=', data.get('logging'))
PY

# MCP/tool verification
hermes mcp list
hermes mcp test token-savior

# Profile and gateway state
hermes profile list
hermes gateway status

# Headroom/proxy residue check on Windows bash
ps aux | grep -i '[h]eadroom' || true
netstat -ano 2>/dev/null | grep ':8787' || true

# Legacy marker check
sed -n '1,5p' "$HOME/.hermes/README-LEGACY.txt" 2>/dev/null || true

# Config health
hermes config check
```

## Expected healthy patterns

- `hermes config path` points to `C:\Users\<user>\AppData\Local\hermes\config.yaml` for the Desktop GUI setup.
- Token Savior optimized strategy: `TOKEN_SAVIOR_PROFILE=optimized` and `hermes mcp test token-savior` discovers the optimized/hot tool set, not the full schema.
- Known-bad or unused MCP/platforms from cleanup are disabled in config, not merely ignored at runtime.
- `hermes profile list` matches the intended profile topology; do not rely on memory or old docs.
- Legacy `~/.hermes` should have an explicit `README-LEGACY.txt` if it remains, so future edits do not target the wrong config tree.
- Headroom/proxy ports should be absent unless intentionally routed into the current model provider chain.

## Windows log rollover pitfall

When Desktop/Gateway and CLI commands run concurrently, `agent.log` can be held open by another process during rotation, producing repeated:

```text
PermissionError: [WinError 32]
agent.log -> agent.log.1
另一个程序正在使用此文件
```

Prefer changing config through Hermes rather than editing the file directly:

```bash
hermes config set logging.max_size_mb 50
hermes config set logging.backup_count 5
```

This reduces rollover frequency and terminal noise. It is not a substitute for a code fix if lock-safe rotation is later implemented upstream.

## Reporting format

Report a compact table of: active config path, token-savior profile/tool count, disabled MCP/platforms, profile list, gateway status, legacy marker, proxy residue, and any warnings from `hermes config check` / `hermes doctor`.
