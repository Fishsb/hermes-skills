# Desktop 覆盖重装后按“最近有效状态”恢复

## 适用场景

用户覆盖重装 Hermes Desktop 后，要求恢复“配置、工具、插件、记忆、规则”，但明确不恢复历史会话；并要求“按最近的状态”。

## 核心经验

1. **不要盲目使用时间最新的 `config.yaml.bak-*`。**
   - 最新备份可能已经是重装后的弱配置快照。
   - 应横向比较多份备份，寻找“最近有效完整状态”：工具/MCP/压缩/模型/记忆/并发/系统提示等组合最接近用户实际使用状态的版本。

2. **恢复应分层合并，而不是整文件覆盖。**
   - 从最近完整备份恢复：`compression`、`platform_toolsets.cli`、`mcp_servers`、默认 `model`。
   - 保留当前已确认更优/更新的强规则：MiMo-first system_prompt、大 memory limits、大 delegation 并发与 depth。
   - 保留当前 `custom_providers`，避免丢失新模型或新代理配置。

3. **验证依据要看实际 CLI 输出。**
   - `hermes config check`：配置版本通过。
   - `hermes tools list`：确认 browser/cronjob/memory/messaging/vision 等工具是否 enabled。
   - `hermes mcp list`：确认 MCP enabled 状态。
   - Python 读取 YAML：确认关键字段值。

4. **工具/MCP 变更需要新会话或 `/reset` 才完全进入当前 tool schema。**
   - 配置已写入不代表当前对话立即拥有新工具。

## 推荐恢复策略

```text
1. 先备份当前 config.yaml：config.yaml.pre-recent-restore-<timestamp>
2. 枚举并比较 config.yaml.bak-*：不要只按 mtime 选最新
3. 打印每份备份的摘要：model / agent / memory / delegation / compression / mcp_enabled / cli_toolsets
4. 选择最近有效完整备份作为“工具与默认状态基准”
5. 分层恢复：
   - compression.*
   - model.provider/default
   - platform_toolsets.cli
   - mcp_servers
6. 明确保留：
   - 强 MiMo-first system_prompt
   - max_turns/reasoning/gateway timeout
   - memory limits/nudge_interval
   - delegation 并发/depth/orchestrator/max_iterations/timeout
   - custom_providers 当前版本
7. 跑 config/tools/mcp/字段验证
```

## 本次会话的有效基准示例

- `config.yaml.bak-mimo-first-...` 时间更新，但含弱配置：低 max_turns、低 memory、低 delegation、MCP disabled。
- `config.yaml.bak-20260615-210509` 更接近最近有效完整状态：完整 CLI toolsets、MCP enabled、强 MiMo/记忆/并发参数。
- 最终选择后者作为工具/MCP/toolsets/压缩/模型基准，同时保留当前已恢复的强规则。

## 不要保存的内容

- 不保存具体 API key、token 或用户私密凭据。
- 不把某个临时 MCP disabled 状态写成永久规则。
- 不恢复历史会话，除非用户明确要求。
