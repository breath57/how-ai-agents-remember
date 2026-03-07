# OpenFang 记忆系统逆向工程文档

> 本文档对 [references/openfang](../../../../references/openfang) 项目的记忆（Memory）子系统进行完整逆向工程，输出可复刻方案。

## 文档索引

| 文件 | 内容 |
|------|------|
| [01-architecture-overview.md](01-architecture-overview.md) | 整体架构概览、分层设计、模块关系 |
| [02-data-model.md](02-data-model.md) | 核心类型定义、数据模型、Trait 接口 |
| [03-storage-layers.md](03-storage-layers.md) | 六层存储详解（KV/语义/知识图谱/会话/任务/用量） |
| [04-schema-migration.md](04-schema-migration.md) | SQLite Schema 设计与 7 版迁移策略 |
| [05-embedding-search.md](05-embedding-search.md) | 向量嵌入与语义检索实现 |
| [06-canonical-session.md](06-canonical-session.md) | 跨通道记忆与会话压缩机制 |
| [07-tool-integration.md](07-tool-integration.md) | Agent 工具层对接记忆系统 |
| [08-replication-guide.md](08-replication-guide.md) | 复刻方案与适配建议 |
| [09-runtime-memory-flow.md](09-runtime-memory-flow.md) | Agent Loop 运行时记忆触发机制（自动 vs 主动、recall/remember 时序、衰减必要性） |

## 关键发现

1. **单 SQLite 文件**：所有记忆存储在一个 `openfang.db` 中，通过 WAL 模式和 `Arc<Mutex<Connection>>` 实现并发安全
2. **六层统一接口**：通过 `Memory` trait 暴露 KV 存储、语义搜索、知识图谱三大核心能力
3. **渐进式向量搜索**：Phase 1 用 LIKE 匹配，Phase 2 用余弦相似度，嵌入向量存为 BLOB
4. **跨通道记忆**：`CanonicalSession` 实现跨 Telegram/Discord 等频道的持久记忆
5. **记忆衰减**：`ConsolidationEngine` 按置信度衰减 7 天未访问的记忆
6. **7 版 Schema 迁移**：`PRAGMA user_version` 驱动，从核心表到设备配对逐步演进
