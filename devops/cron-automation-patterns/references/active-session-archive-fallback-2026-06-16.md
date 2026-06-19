# Active Sessions Archive Workaround (2026-06-16)

## 场景

用户对智能归档扫描回复"0"（全部归档），`smart-archive.py archive 0` 只处理了 2 个闲置会话，2 个活跃会话（归档定时触发重建、New DAO API Provider Setup）未被 tracker 捕获。

## 复现

```
📋 智能归档检查
  🟡 闲置对话（2）  [1] MiMo子Agent并行调度优化            1.7h
                    [2] MiMo Current Trigger Constraints    1.2h
  🟢 活跃对话（2）  [1] 归档定时触发重建                      1m
                    [2] New DAO API Provider Setup         30m

→ 用户回复 "0"
→ python3 smart-archive.py archive 0
  → 仅处理 2 个闲置会话 ✅
  → 2 个活跃会话未处理 ❌
```

## 手工补充步骤

从 `hermes sessions list` 获取 2 个活跃会话的 session ID：

```python
import subprocess, json, os, time

HERMES_CLI = ['python3', os.environ['LOCALAPPDATA'] + '/hermes/hermes-agent/hermes']
TEMP_DIR = 'D:/lk/Obsidian/Hermes/wiki/记忆/_temp_session'
os.makedirs(TEMP_DIR, exist_ok=True)

for sid, title in [('<SID1>', '<title1>'), ('<SID2>', '<title2>')]:
    # 1. Export
    r = subprocess.run(HERMES_CLI + ['sessions', 'export', '--session-id', sid, '-'],
        capture_output=True, text=True, errors='replace')
    if r.returncode != 0 or not r.stdout.strip():
        continue
    data = json.loads(r.stdout)
    ts = int(time.time())
    safe = title.replace(' ', '_').replace('/', '_')
    jp = f'{TEMP_DIR}/{ts}_{safe}_{sid[:8]}.json'
    with open(jp, 'w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, indent=2)
    meta = {'session_id': sid, 'profile': 'default', 'title': title, 'exported_at': ts,
            'export_status': 'ok', 'json_file': jp}
    with open(jp.replace('.json', '.meta'), 'w', encoding='utf-8') as f:
        json.dump(meta, f, ensure_ascii=False, indent=2)
    # 2. Delete
    dr = subprocess.run(HERMES_CLI + ['sessions', 'delete', sid, '--yes'],
        capture_output=True, text=True, errors='replace')
```

## Queue 更新

3. 将 2 个新条目追加到 `ingest-queue.json` 的 `pending[]` 数组，字段：
   - `id`, `title`, `profile`, `staged_file`, `meta_file`, `status: "staged"`, `created_at`, `updated_at`, `archive_reason: "manual"`, `source: "tui"`, `model`, `message_count`
4. 更新 `updated_at` 时间戳

## 验证

- `hermes sessions list` 无残留会话（除 cron 作业自身会话）
- `smart-archive.py scan` 显示 闲置=0, 活跃=0
- `ingest-queue.json` pending 数组包含 2 个新条目

## Pitfall

- `hermes` CLI 是 Python shebang 脚本（非 .exe），subprocess 必须用 `['python3', full_path, ...]`
  - Windows 下用 `os.environ['LOCALAPPDATA'] + '/hermes/hermes-agent/hermes'` 构造路径
  - 不要用 `/c/Users/...` MSYS 路径——subprocess 会错误转换为 `C:\c\Users\...`
- 导出获得的 JSON 要先 `json.loads()` 验证有效性，避免写入空文件
- 删除成功后检查 `ingest-queue.json` 的 pending 计数是否合理增加
