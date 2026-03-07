# OpenClaw 记忆系统逆向工程文档

> 基于 `references/openclaw` 源码分析，目标：完整复刻其记忆能力

## 文档索引

| 文档 | 内容 |
|------|------|
| [01-architecture-overview.md](01-architecture-overview.md) | 整体架构总览、分层设计、模块关系 |
| [02-data-model.md](02-data-model.md) | 数据模型、SQLite Schema、LanceDB Schema |
| [03-memory-indexing.md](03-memory-indexing.md) | 索引构建：文件扫描、分块、Embedding、同步 |
| [04-search-pipeline.md](04-search-pipeline.md) | 搜索管线：混合搜索、MMR、时间衰减 |
| [05-embedding-providers.md](05-embedding-providers.md) | Embedding 提供商：多后端、自动选择、降级策略 |
| [06-memory-flush.md](06-memory-flush.md) | 预压缩记忆刷写：自动持久化机制 |
| [07-plugin-system.md](07-plugin-system.md) | 插件体系：memory-core + memory-lancedb |
| [08-config-reference.md](08-config-reference.md) | 完整配置参考 |
| [09-replication-guide.md](09-replication-guide.md) | 复刻实施方案（适配本项目） |

## 核心设计哲学

1. **Markdown 是真相之源** — 数据库只是索引，记忆文件本身才是持久化存储
2. **插件化** — 记忆能力通过插件注册，可替换后端
3. **混合搜索** — 向量语义 + BM25 关键词，兼顾语义理解和精确匹配
4. **渐进降级** — Embedding 不可用时降级为 FTS-only；QMD 失败时回退到内置 SQLite
5. **增量同步** — 文件监听 + 会话监听，实时更新索引
6. **零拷贝设计** — Agent 只能通过 tool 读取记忆内容片段，不会加载全文到上下文

## 技术栈

| 组件 | 技术选型 |
|------|---------|
| 语言 | TypeScript (Node.js) |
| 内置存储 | SQLite (node:sqlite) + FTS5 + sqlite-vec |
| 向量数据库 | LanceDB（可选插件） |
| Embedding | OpenAI / Gemini / Voyage / Mistral / Ollama / node-llama-cpp |
| 文件监听 | chokidar |
| 搜索后端 | QMD sidecar（可选）/ 内置 SQLite |
