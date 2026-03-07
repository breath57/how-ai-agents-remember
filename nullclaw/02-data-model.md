# 02 — 核心数据模型

## MemoryEntry（记忆条目）

系统的基本存储单元，所有后端都存储和返回此结构：

```mermaid
classDiagram
    class MemoryEntry {
        +string id          // 唯一标识（UUID 或自增 ID）
        +string key         // 用户可读的键名（如 "user_preference_language"）
        +string content     // 记忆内容文本
        +MemoryCategory category  // 分类
        +string timestamp   // Unix 时间戳字符串
        +string? session_id // 会话隔离 ID
        +float? score       // 搜索评分（recall 时填充）
        +deinit(allocator)  // 释放所有字段
    }

    class MemoryCategory {
        <<union>>
        core         // 长期核心记忆（永不衰减）
        daily        // 每日记忆（可归档/清除）
        conversation // 对话级记忆（会话结束可清理）
        custom(name) // 自定义分类
        +toString() string
        +fromString(s) MemoryCategory
        +eql(other) bool
    }

    MemoryEntry --> MemoryCategory
```

### 分类语义

| Category | 含义 | 生命周期 | 时间衰减 |
|----------|------|---------|---------|
| `core` | 长期知识、用户偏好、身份信息 | 永久 | 不衰减（evergreen） |
| `daily` | 每日工作记录、临时事实 | archive_after_days → purge_after_days | 正常衰减 |
| `conversation` | 对话上下文、临时提取的事实 | conversation_retention_days | 正常衰减 |
| `custom(name)` | 用户自定义分类 | 取决于应用逻辑 | 正常衰减 |

### 内部保留键

系统使用前缀约定标识内部记忆键，这些键在 UI 列表中被过滤：

```
autosave_user_*          # 自动保存的用户消息
autosave_assistant_*     # 自动保存的助手消息
last_hygiene_at          # 上次卫生清理时间戳
__bootstrap.prompt.*     # 引导文档（AGENTS.md, SOUL.md 等）
```

### Bootstrap 文档映射

```python
PROMPT_BOOTSTRAP_DOCS = {
    "AGENTS.md":    "__bootstrap.prompt.AGENTS.md",
    "SOUL.md":      "__bootstrap.prompt.SOUL.md",
    "TOOLS.md":     "__bootstrap.prompt.TOOLS.md",
    "IDENTITY.md":  "__bootstrap.prompt.IDENTITY.md",
    "USER.md":      "__bootstrap.prompt.USER.md",
    "HEARTBEAT.md": "__bootstrap.prompt.HEARTBEAT.md",
    "BOOTSTRAP.md": "__bootstrap.prompt.BOOTSTRAP.md",
    "MEMORY.md":    "__bootstrap.prompt.MEMORY.md",
}
```

## Memory 接口（vtable）

所有存储后端实现此统一接口：

```mermaid
classDiagram
    class Memory {
        <<interface>>
        +name() string
        +store(key, content, category, session_id?)
        +recall(allocator, query, limit, session_id?) MemoryEntry[]
        +get(allocator, key) MemoryEntry?
        +list(allocator, category?, session_id?) MemoryEntry[]
        +forget(key) bool
        +count() usize
        +healthCheck() bool
        +search(allocator, query, limit) MemoryEntry[]
        +deinit()
    }

    class SqliteMemory {
        // FTS5 BM25 搜索
        // 事务支持
        // Session Store
        // KV 存储
    }
    class MarkdownMemory {
        // MEMORY.md 核心文件
        // memory/YYYY-MM-DD.md 日志
        // 只追加，forget 是 no-op
    }
    class PostgresMemory { }
    class RedisMemory { }
    class ApiMemory { }
    class LanceDbMemory { }
    class InMemoryLruMemory { }
    class NoneMemory { }

    Memory <|.. SqliteMemory
    Memory <|.. MarkdownMemory
    Memory <|.. PostgresMemory
    Memory <|.. RedisMemory
    Memory <|.. ApiMemory
    Memory <|.. LanceDbMemory
    Memory <|.. InMemoryLruMemory
    Memory <|.. NoneMemory
```

### 方法语义

| 方法 | 语义 | 说明 |
|------|------|------|
| `store(key, content, category, session_id?)` | Upsert | 相同 key 覆盖更新，不同 key 插入 |
| `recall(query, limit, session_id?)` | 关键词搜索 | SQLite 用 FTS5 BM25，其他后端用简单匹配 |
| `get(key)` | 精确获取 | 按 key 精确查找单条 |
| `list(category?, session_id?)` | 列举 | 可按 category 和 session_id 过滤 |
| `forget(key)` | 删除 | Markdown 后端的 forget 是 no-op |
| `count()` | 计数 | 返回总条目数 |
| `search(query, limit)` | 混合搜索 | 目前委托给 recall()，未来升级为 hybrid |

## SessionStore 接口（vtable）

会话消息存储，部分后端支持：

```mermaid
classDiagram
    class SessionStore {
        <<interface>>
        +saveMessage(session_id, role, content)
        +loadMessages(allocator, session_id) MessageEntry[]
        +clearMessages(session_id)
        +clearAutoSaved(session_id?)
    }

    class MessageEntry {
        +string role      // "user" | "assistant" | "system"
        +string content   // 消息内容
    }

    SessionStore --> MessageEntry
```

支持 SessionStore 的后端：SQLite、Lucid、Postgres、API。

## RetrievalCandidate（检索候选）

检索引擎的输出单元，比 MemoryEntry 更丰富：

```mermaid
classDiagram
    class RetrievalCandidate {
        +string id
        +string key
        +string content
        +string snippet       // 截断的摘要片段
        +MemoryCategory category
        +u32? keyword_rank    // FTS5 排名
        +f32? vector_score    // 向量相似度 [0,1]
        +f64 final_score      // RRF 合并后的最终分数
        +string source        // 来源名称："primary" | "qmd" | ...
        +string source_path   // 文件路径（QMD 用）
        +u32 start_line       // 起始行号
        +u32 end_line         // 结束行号
        +i64 created_at       // 创建时间戳
    }
```

### MemoryEntry → RetrievalCandidate 转换

```python
def entry_to_candidate(entry: MemoryEntry, rank: int) -> RetrievalCandidate:
    return RetrievalCandidate(
        id=entry.id,
        key=entry.key,
        content=entry.content,
        snippet=entry.content,  # 直接复制
        category=entry.category,
        keyword_rank=rank,       # 1-based
        vector_score=None,
        final_score=0.0,         # RRF 阶段填充
        source="primary",
        source_path="",
        start_line=0,
        end_line=0,
        created_at=int(entry.timestamp) or 0,
    )
```

## MemoryRuntime（运行时聚合体）

最终暴露给上层的组合结构：

```mermaid
classDiagram
    class MemoryRuntime {
        +Memory memory
        +SessionStore? session_store
        +ResponseCache? response_cache
        +BackendCapabilities capabilities
        +ResolvedConfig resolved
        -RetrievalEngine? _engine
        -RolloutPolicy _rollout_policy
        -SummarizerConfig _summarizer_cfg
        -SemanticCache? _semantic_cache
        -EmbeddingProvider? _embedding_provider
        -VectorStore? _vector_store
        -CircuitBreaker? _circuit_breaker
        -VectorOutbox? _outbox
        +search(allocator, query, limit, session_id?) RetrievalCandidate[]
        +syncVectorAfterStore(allocator, key, content)
        +deleteFromVectorStore(key)
        +drainOutbox(allocator) u32
        +reindex(allocator) u32
        +diagnose() DiagnosticReport
        +deinit()
    }

    class BackendCapabilities {
        +bool supports_keyword_rank  // FTS5 BM25
        +bool supports_session_store
        +bool supports_transactions
        +bool supports_outbox        // durable vector-sync
    }

    MemoryRuntime --> BackendCapabilities
```

### search() 决策逻辑

```mermaid
flowchart TD
    A[search 请求] --> B{Rollout 决策}
    B -->|keyword_only| C[memory.recall]
    B -->|hybrid| D{Engine 存在?}
    B -->|shadow_hybrid| E[同时执行 keyword + hybrid]

    D -->|是| F[engine.search]
    D -->|否| C

    C --> G[entries → candidates]
    F --> H[trimToLimit]
    E --> I[返回 keyword 结果<br/>记录 hybrid 结果]

    G --> J[返回]
    H --> J
    I --> J
```
