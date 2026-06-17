# 会话拆分示例：聊天室项目连接故障排查

## 测试场景

| 维度 | 值 |
|------|-----|
| 会话 | "聊天室项目连接故障排查" |
| 消息数 | 246 条 |
| JSON 大小 | 448 KB |
| 主题 | Hermes web-chat-frontend 三层链路断连（Nginx → SSH隧道 → Gateway/API Server） |
| 内存占用 | 子 Agent 使用 `terminal("python3 -c ...")` 读取 + 搜索_files 扫描 wiki |
| 耗时 | ~103 秒（1 个 agent，15 API 调用） |

## 拆分决策

会话内容横跨两个独立主题，原文互不依赖，因此拆为 **2 篇独立笔记**：

### 1. 排障类 → `wiki/工具/`

**文件:** `Hermes Gateway API Server SSH隧道排障.md`（272 行）

内容：完整排障流程
- Gateway 未运行
- API Server 未配置
- SSH 反向隧道断开
- 两份 `.env` 密钥不一致（root cause）
- 全链路 `curl` 验证命令

frontmatter tags: `type/troubleshooting`, `domain/hermes`, `tech/network`, `tech/ssh`, `已解决`

### 2. 架构类 → `wiki/项目/`

**文件:** `Web Chat Frontend 聊天室架构.md`（198 行）

内容：项目架构拓扑
- VPS Nginx → chat-relay → SSH 隧道 → Windows Gateway + API Server
- 三层链路组件清单
- 配置要点汇总

frontmatter tags: `type/architecture`, `domain/hermes`, `project/chat`

## 互链

- 排障笔记 → 引用架构笔记（"参考项目架构"）
- 架构笔记 → 引用排障笔记（"排障记录"）
- 两篇笔记都回溯引用 `工具/微信Gateway修复-会话清理与模型恢复.md`

## 结论

246 条消息的大会话可以安全拆分为独立知识点笔记。拆分条件成立：
1. 内容明显分属两个类别（tools vs project）
2. 每篇内容充实但不超过 300 行红线
3. 互链保证了知识不丢失
