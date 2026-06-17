# Hermes Agent 记忆系统逆向工程文档

> 本文档对 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) 的记忆、会话回忆、上下文压缩、记忆 Provider 插件和技能型程序记忆进行源码级逆向工程。

## 文档索引

| 文件 | 内容 |
|------|------|
| [01-architecture-overview.md](01-architecture-overview.md) | 整体架构：五条记忆平面、模块关系、数据流 |
| [02-built-in-memory.md](02-built-in-memory.md) | 内置 `MEMORY.md` / `USER.md`：冻结快照、写入工具、容量控制 |
| [03-memory-provider-plugins.md](03-memory-provider-plugins.md) | 外部记忆 Provider：ABC、MemoryManager、Honcho 等插件 |
| [04-session-storage-search.md](04-session-storage-search.md) | `state.db` 会话存储、FTS5 检索、跨 profile 会话读取 |
| [05-context-compression.md](05-context-compression.md) | ContextEngine、上下文压缩、session split、记忆边界钩子 |
| [06-runtime-flow.md](06-runtime-flow.md) | 从启动到每一轮对话的完整运行时记忆流程 |
| [07-security-reliability.md](07-security-reliability.md) | 注入防护、写入审批、漂移检测、异步隔离、DB 自愈 |
| [08-skills-procedural-memory.md](08-skills-procedural-memory.md) | Skills 作为程序性记忆：渐进披露、slash command、curator |
| [09-replication-guide.md](09-replication-guide.md) | 在 Python/LangGraph 技术栈复刻 Hermes 记忆系统的实施方案 |

## 关键发现

1. **冻结快照是核心约束**：`MEMORY.md` / `USER.md` 在 session 启动时读入 system prompt，随后不在普通 turn 中变更，保护 prompt cache。
2. **外部记忆是插件，不是内核**：`MemoryProvider` ABC + `MemoryManager` 支持 Honcho、Mem0、Hindsight 等后端，但一次只允许一个外部 Provider。
3. **检索分两条路**：关键事实走小而稳定的 prompt 记忆，历史对话走 `state.db` + FTS5 的 `session_search` 按需检索。
4. **外部 recall 不污染缓存前缀**：Provider 的召回结果用 `<memory-context>` 包进当前 user message 的 API copy，不写回 session，也不改 system prompt。
5. **上下文压缩就是 session 边界**：压缩前触发记忆提取，旧 session 结束，新 session 以 `parent_session_id` 串起来，Provider 收到 session switch 钩子。
6. **技能是程序性记忆**：Hermes 不把复杂流程塞进事实记忆，而是把可复用流程沉淀成 Skill，通过渐进披露按需加载。
7. **可靠性设计很厚**：内置注入扫描、写入审批、外部漂移备份、streaming scrubber、异步 Provider、WAL fallback、schema repair 都围绕“记住但不污染”展开。

## 源码主入口

| 模块 | 源文件 | 作用 |
|------|--------|------|
| 内置记忆 | `tools/memory_tool.py` | `MemoryStore`、`memory` tool、文件写入与扫描 |
| Provider ABC | `agent/memory_provider.py` | 外部记忆插件接口与生命周期钩子 |
| Provider 编排 | `agent/memory_manager.py` | 注册、prefetch、sync、tool routing、context fencing |
| Agent 初始化 | `agent/agent_init.py` | 加载内置记忆、外部 Provider、ContextEngine |
| Prompt 构造 | `agent/system_prompt.py` | system prompt 分层与冻结快照注入 |
| API 注入 | `agent/conversation_loop.py` | 将外部 recall 注入当前 user message copy |
| 会话 DB | `hermes_state.py` | SQLite `state.db`、FTS5、压缩 lineage、锁和迁移 |
| 会话检索 | `tools/session_search_tool.py` | discovery / scroll / read / browse 四种回忆形态 |
| 上下文引擎 | `agent/context_engine.py` | 可插拔 ContextEngine 抽象 |
| 压缩器 | `agent/context_compressor.py` | 内置摘要压缩、消息保护、工具对修复 |
| 压缩编排 | `agent/conversation_compression.py` | 压缩锁、session split、记忆钩子 |
