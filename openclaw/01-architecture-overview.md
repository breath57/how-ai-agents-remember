# 01 - 整体架构总览

## 分层架构

OpenClaw 的记忆系统采用 **六层分层架构**，从上到下依次为：插件层 → Agent 工具层 → 搜索管理器路由层 → 记忆引擎层 → Embedding 提供商层 → 存储与文件层。

```mermaid
graph TB
    subgraph "Plugin Layer（插件层）"
        MC["memory-core 插件<br/>memory_search + memory_get"]
        ML["memory-lancedb 插件<br/>memory_recall + memory_store + memory_forget"]
    end

    subgraph "Agent Tool Layer（工具层）"
        MST["createMemorySearchTool()"]
        MGT["createMemoryGetTool()"]
        MRT["memory_recall (LanceDB)"]
        MSS["memory_store (LanceDB)"]
        MFG["memory_forget (LanceDB)"]
    end

    subgraph "Search Manager Router（路由层）"
        GSM["getMemorySearchManager()"]
        FBM["FallbackMemoryManager<br/>（QMD 优先 + 内置回退）"]
    end

    subgraph "Memory Engine（引擎层）"
        MIM["MemoryIndexManager<br/>（内置 SQLite 引擎）"]
        QMD["QmdMemoryManager<br/>（QMD sidecar 引擎）"]
        MDB["MemoryDB<br/>（LanceDB 引擎）"]
    end

    subgraph "Embedding Provider（Embedding 层）"
        CEP["createEmbeddingProvider()"]
        OAI["OpenAI"]
        GEM["Gemini"]
        VOY["Voyage"]
        MIS["Mistral"]
        OLL["Ollama"]
        LOC["node-llama-cpp (本地)"]
    end

    subgraph "Storage & Files（存储层）"
        SQL["SQLite DB<br/>chunks + FTS5 + sqlite-vec"]
        LDB["LanceDB<br/>memories 表"]
        MDF["Markdown 文件<br/>MEMORY.md + memory/*.md"]
        SES["Session 日志<br/>sessions/*.jsonl"]
    end

    MC --> MST & MGT
    ML --> MRT & MSS & MFG
    MST & MGT --> GSM
    MRT & MSS & MFG --> MDB
    GSM --> FBM
    FBM --> QMD & MIM
    MIM --> CEP
    MIM --> SQL
    QMD --> |"shell out"|QMD_CLI["qmd CLI"]
    MDB --> LDB
    CEP --> OAI & GEM & VOY & MIS & OLL & LOC
    SQL --> |"索引自"|MDF & SES
    LDB --> |"独立存储"|LDB_PATH["~/.openclaw/memory/lancedb"]
    MDF --> |"真相之源"|MDF_PATH["~/.openclaw/workspace/"]
```

## 两条独立的记忆管线

OpenClaw 有 **两个完全独立的记忆子系统**，它们互不依赖：

### 管线 A：内置 SQLite 记忆（`memory-core`）

```mermaid
flowchart LR
    MD["Markdown 文件"] -->|"扫描 + 分块"| CHUNK["文本块"]
    CHUNK -->|"Embedding"| VEC["向量 + 文本"]
    VEC -->|"存入"| SQLITE["SQLite<br/>chunks + FTS5 + vec"]
    QUERY["用户查询"] -->|"Embedding"| QVEC["查询向量"]
    QUERY -->|"关键词提取"| FTS["FTS 查询"]
    QVEC & FTS -->|"混合搜索"| MERGE["加权合并"]
    MERGE -->|"后处理"| RESULT["排序结果"]

    style MD fill:#e1f5fe
    style SQLITE fill:#fff3e0
    style RESULT fill:#e8f5e9
```

**特点**：
- Markdown 文件是源，SQLite 只是索引
- 支持混合搜索（向量 + BM25）
- 支持 MMR 多样性重排
- 支持时间衰减
- 通过 `memory_search` 和 `memory_get` 工具暴露

### 管线 B：LanceDB 长期记忆（`memory-lancedb`）

```mermaid
flowchart LR
    USER["用户消息"] -->|"规则判断<br/>shouldCapture()"| CAP["自动捕获"]
    AGENT["Agent 主动调用"] -->|"memory_store"| STORE["存储"]
    CAP & STORE -->|"Embedding + 去重"| LANCE["LanceDB<br/>memories 表"]

    REC_AUTO["before_agent_start"] -->|"自动注入"| RECALL["向量搜索"]
    REC_TOOL["memory_recall 工具"] --> RECALL
    RECALL --> LANCE
    LANCE --> CTX["注入 Agent 上下文"]

    style USER fill:#e1f5fe
    style LANCE fill:#fff3e0
    style CTX fill:#e8f5e9
```

**特点**：
- 独立的向量数据库（LanceDB），不依赖 Markdown 文件
- 自动捕获 + 自动召回（生命周期 Hook）
- 支持 GDPR 遗忘（`memory_forget`）
- 分类存储（preference / fact / decision / entity / other）

## 模块文件映射

```mermaid
graph TB
    subgraph "src/memory/ — 核心引擎"
        types["types.ts<br/>核心接口定义"]
        schema["memory-schema.ts<br/>SQLite DDL"]
        internal["internal.ts<br/>文件扫描 + 分块 + Hash"]
        manager["manager.ts<br/>MemoryIndexManager 主类"]
        sync["manager-sync-ops.ts<br/>文件监听 + 会话同步"]
        embed["manager-embedding-ops.ts<br/>批量 Embedding"]
        search["manager-search.ts<br/>向量/关键词搜索实现"]
        sm["search-manager.ts<br/>路由 + FallbackManager"]
        hybrid["hybrid.ts<br/>混合搜索合并"]
        mmr["mmr.ts<br/>MMR 多样性重排"]
        td["temporal-decay.ts<br/>时间衰减"]
        qe["query-expansion.ts<br/>查询扩展"]
        embeddings["embeddings.ts<br/>Embedding Provider 工厂"]
        qmd["qmd-manager.ts<br/>QMD sidecar"]
        bc["backend-config.ts<br/>后端配置解析"]
        sf["session-files.ts<br/>会话文件解析"]
    end

    subgraph "src/agents/ — Agent 集成"
        ms["memory-search.ts<br/>配置解析"]
        mt["tools/memory-tool.ts<br/>Agent 工具定义"]
    end

    subgraph "src/auto-reply/ — 自动回复"
        mf["reply/memory-flush.ts<br/>预压缩刷写"]
    end

    subgraph "extensions/ — 插件"
        mcore["memory-core/index.ts<br/>注册搜索工具"]
        mlance["memory-lancedb/index.ts<br/>LanceDB 全功能插件"]
        mconfig["memory-lancedb/config.ts<br/>LanceDB 配置"]
    end

    subgraph "src/config/ — 配置"
        tmem["types.memory.ts<br/>内存配置类型"]
    end

    manager --> sync --> embed
    manager --> search
    manager --> hybrid --> mmr & td
    sm --> manager & qmd
    mt --> sm
    mcore --> mt
    mlance --> mconfig
```

## 类继承关系

```mermaid
classDiagram
    class MemorySearchManager {
        <<interface>>
        +search(query, opts) MemorySearchResult[]
        +readFile(params) text, path
        +status() MemoryProviderStatus
        +sync(params) void
        +probeEmbeddingAvailability() MemoryEmbeddingProbeResult
        +probeVectorAvailability() boolean
        +close() void
    }

    class MemoryManagerSyncOps {
        #db: DatabaseSync
        #settings: ResolvedMemorySearchConfig
        #watcher: FSWatcher
        +startWatcher()
        +stopWatcher()
        +syncFiles(reason, force)
        +migrateSchema()
    }

    class MemoryManagerEmbeddingOps {
        #embeddingProvider: EmbeddingProvider
        +embedChunks(chunks)
        +pruneEmbeddingCache()
        +ensureVectorTable(dims)
    }

    class MemoryIndexManager {
        -INDEX_CACHE: Map
        +get(cfg, agentId) MemoryIndexManager
        +search(query, opts) MemorySearchResult[]
        +readFile(params) text, path
        +status() MemoryProviderStatus
        +sync(params) void
    }

    class FallbackMemoryManager {
        -primary: MemorySearchManager
        -fallback: MemorySearchManager
        +search(query, opts)
        +status()
    }

    class QmdMemoryManager {
        -qmdBinary: string
        -agentId: string
        +search(query, opts)
        +spawn()
        +update()
    }

    class MemoryDB {
        -db: LanceDB.Connection
        -table: LanceDB.Table
        +store(entry)
        +search(vector, limit, minScore)
        +delete(id)
        +count()
    }

    MemorySearchManager <|.. MemoryIndexManager
    MemorySearchManager <|.. FallbackMemoryManager
    MemorySearchManager <|.. QmdMemoryManager
    MemoryManagerSyncOps <|-- MemoryManagerEmbeddingOps
    MemoryManagerEmbeddingOps <|-- MemoryIndexManager
    FallbackMemoryManager --> QmdMemoryManager : primary
    FallbackMemoryManager --> MemoryIndexManager : fallback
```

## 关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 真相之源 | Markdown 文件 | 可读、可用 Git 版控、Agent 可直接读写 |
| 内置搜索 | SQLite + FTS5 + sqlite-vec | 零依赖、嵌入式、性能好 |
| 向量搜索 | sqlite-vec 扩展（可选） | 避免加载全部 Embedding 到内存 |
| 混合权重 | 向量 0.7 + BM25 0.3（默认） | 语义匹配为主，精确匹配补充 |
| 多 Embedding | 6 种 provider + auto 选择 | 最大兼容性 |
| 增量更新 | chokidar 文件监听 + 内容 Hash | 只重建变化的块 |
| 单例缓存 | `INDEX_CACHE` Map | 多次 get() 同一 agent 共享实例 |
| 降级策略 | Embedding → FTS-only；QMD → 内置 | 任何组件失败不影响基本功能 |
