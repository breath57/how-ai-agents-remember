# Nullclaw 记忆系统逆向工程文档

> 本文档通过逆向工程 nullclaw（Zig 实现的 AI Agent 框架）的 `src/memory/` 模块，
> 完整拆解其记忆能力的架构设计、数据模型、检索管线、生命周期管理等核心实现，
> 以便后续在 Python/LangGraph 技术栈上完全复刻。

## 文档结构

| 文件 | 内容 |
|------|------|
| [01-architecture.md](01-architecture.md) | 总体架构、四层分层、MemoryRuntime 组装 |
| [02-data-model.md](02-data-model.md) | 核心数据结构、Memory vtable、Category 体系 |
| [03-storage-backends.md](03-storage-backends.md) | 10 种存储后端、Registry 工厂模式 |
| [04-retrieval-pipeline.md](04-retrieval-pipeline.md) | 9 阶段检索管线、RRF、自适应策略 |
| [05-vector-plane.md](05-vector-plane.md) | Embedding 提供者、向量存储、熔断器、Outbox |
| [06-lifecycle.md](06-lifecycle.md) | 缓存、卫生清理、快照、摘要器、语义缓存 |
| [07-rollout-reliability.md](07-rollout-reliability.md) | Rollout 策略、Shadow/Canary 模式 |
| [08-replication-guide.md](08-replication-guide.md) | Python/LangGraph 复刻指南 |

## 源码映射

```
nullclaw/src/memory/
├── root.zig                    # 模块入口 + Memory/SessionStore vtable + MemoryRuntime
├── engines/                    # Layer A: 主存储层
│   ├── api.zig                 #   HTTP REST API 后端
│   ├── sqlite.zig              #   SQLite + FTS5 后端（推荐）
│   ├── markdown.zig            #   Markdown 文件后端
│   ├── postgres.zig            #   PostgreSQL 后端
│   ├── redis.zig               #   Redis 后端
│   ├── lancedb.zig             #   LanceDB 后端
│   ├── lucid.zig               #   Lucid (SQLite + 跨项目同步)
│   ├── memory_lru.zig          #   内存 LRU（测试用）
│   ├── none.zig                #   No-op 空实现
│   └── registry.zig            #   后端注册表 + 工厂
├── retrieval/                  # Layer B: 检索引擎层
│   ├── engine.zig              #   RetrievalEngine 主流程
│   ├── rrf.zig                 #   Reciprocal Rank Fusion
│   ├── query_expansion.zig     #   查询扩展（停用词/FTS5）
│   ├── adaptive.zig            #   自适应策略选择
│   ├── temporal_decay.zig      #   时间衰减
│   ├── mmr.zig                 #   MMR 多样性重排
│   ├── llm_reranker.zig        #   LLM 重排序
│   └── qmd.zig                 #   QMD 外部搜索适配器
├── vector/                     # Layer C: 向量平面层
│   ├── embeddings.zig          #   EmbeddingProvider vtable + OpenAI/Noop
│   ├── embeddings_gemini.zig   #   Gemini Embedding
│   ├── embeddings_voyage.zig   #   Voyage Embedding
│   ├── embeddings_ollama.zig   #   Ollama 本地 Embedding
│   ├── provider_router.zig     #   提供者路由 + Fallback
│   ├── store.zig               #   VectorStore vtable + SQLite 实现
│   ├── store_qdrant.zig        #   Qdrant 向量存储
│   ├── store_pgvector.zig      #   pgvector 向量存储
│   ├── math.zig                #   余弦相似度、向量序列化
│   ├── chunker.zig             #   Markdown 文档分块
│   ├── outbox.zig              #   持久化向量同步 Outbox
│   └── circuit_breaker.zig     #   熔断器状态机
└── lifecycle/                  # Layer D: 运行时编排层
    ├── cache.zig               #   LLM 响应缓存（精确匹配）
    ├── semantic_cache.zig      #   语义缓存（余弦相似度匹配）
    ├── hygiene.zig             #   定期清理（归档/清除/裁剪）
    ├── snapshot.zig            #   JSON 快照导出/导入
    ├── summarizer.zig          #   对话摘要 + 语义事实提取
    ├── rollout.zig             #   混合检索渐进发布策略
    ├── diagnostics.zig         #   运行时诊断报告
    └── migrate.zig             #   数据迁移工具
```
