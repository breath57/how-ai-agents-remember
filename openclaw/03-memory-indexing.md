# 03 - 索引构建

## 索引构建全流程

```mermaid
flowchart TB
    START["触发同步"] --> SCAN["扫描记忆文件<br/>listMemoryFiles()"]
    SCAN --> |"MEMORY.md + memory/*.md<br/>+ extraPaths"| FILES["文件列表"]
    
    FILES --> HASH["计算每个文件的<br/>SHA256 Hash"]
    HASH --> COMPARE{"对比 DB 中<br/>files.hash"}
    
    COMPARE -->|"Hash 变化"| REINDEX["需要重建索引"]
    COMPARE -->|"Hash 未变"| SKIP["跳过"]
    COMPARE -->|"新文件"| REINDEX
    COMPARE -->|"文件删除"| DELETE["清理旧记录"]
    
    REINDEX --> READ["读取文件内容"]
    READ --> CHUNK["Markdown 分块<br/>chunkMarkdown()"]
    CHUNK --> CACHE_CHECK{"检查<br/>embedding_cache"}
    
    CACHE_CHECK -->|"缓存命中"| USE_CACHE["使用缓存向量"]
    CACHE_CHECK -->|"缓存未命中"| EMBED["调用 Embedding API"]
    EMBED --> SAVE_CACHE["写入缓存"]
    
    USE_CACHE & SAVE_CACHE --> UPSERT["写入 chunks 表"]
    UPSERT --> FTS_SYNC["同步 FTS5 表"]
    UPSERT --> VEC_SYNC["同步 sqlite-vec 表"]
    UPSERT --> UPDATE_FILES["更新 files 表"]
```

## 文件扫描

### 扫描规则（`listMemoryFiles()`）

```mermaid
flowchart LR
    WD["工作区目录"] --> MEM_FILE["MEMORY.md<br/>memory.md"]
    WD --> MEM_DIR["memory/ 目录<br/>递归 *.md"]
    WD --> EXTRA["extraPaths<br/>额外路径"]
    
    MEM_FILE & MEM_DIR & EXTRA --> FILTER["过滤规则"]
    FILTER --> DEDUP["路径去重<br/>(realpath)"]
    DEDUP --> RESULT["文件列表"]
    
    FILTER -.- RULES["规则：<br/>1. 只接受 .md 文件<br/>2. 忽略符号链接<br/>3. extraPaths 支持目录递归<br/>4. extraPaths 支持绝对/相对路径"]
```

**扫描入口**：
- `MEMORY.md` 或 `memory.md`（二选一或共存）
- `memory/` 目录下所有 `.md` 文件（递归）
- `extraPaths` 配置的额外路径

**过滤规则**：
1. 只接受 `.md` 后缀
2. 忽略符号链接（文件和目录都跳过）
3. 路径去重（通过 `fs.realpath` 解析后比较）

### 路径判断（`isMemoryPath()`）

判断一个相对路径是否属于"记忆文件"：

```typescript
function isMemoryPath(relPath: string): boolean {
    const normalized = normalizeRelPath(relPath);  // 去掉前导 ./ 并统一 /
    if (normalized === "MEMORY.md" || normalized === "memory.md") return true;
    return normalized.startsWith("memory/");
}
```

## Markdown 分块

### 分块算法（`chunkMarkdown()`）

将 Markdown 内容按 token 数量分块，支持重叠以保持上下文连贯性：

```mermaid
flowchart TB
    INPUT["Markdown 内容"] --> SPLIT["按行分割"]
    SPLIT --> ITER["逐行累积"]
    
    ITER --> CHECK{"当前块<br/>≥ maxChars?"}
    CHECK -->|"否"| CONTINUE["继续累积下一行"]
    CHECK -->|"是"| FLUSH["刷出当前块"]
    
    FLUSH --> OVERLAP["计算重叠区域<br/>(overlapChars)"]
    OVERLAP --> KEEP["保留尾部<br/>作为下一块起始"]
    KEEP --> ITER
    
    CONTINUE --> END_CHECK{"文件结束?"}
    END_CHECK -->|"否"| ITER
    END_CHECK -->|"是"| FLUSH_LAST["刷出最后一块"]
    
    FLUSH & FLUSH_LAST --> HASH_CHUNK["SHA256(块文本)"]
    HASH_CHUNK --> RESULT["MemoryChunk<br/>{startLine, endLine, text, hash}"]
```

**参数**：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `tokens` | 400 | 每块目标 token 数 |
| `overlap` | 80 | 块之间重叠 token 数 |

**token 估算**：`maxChars = tokens × 4`（简单按 4 字符/token 估算）

**重叠策略**：
- 从当前块尾部取 `overlapChars` 字符的行作为下一块起始
- 确保跨块边界的信息不丢失
- 每个块记录 `startLine` 和 `endLine`（1-based）

### 分块示例

假设 `tokens=400, overlap=80`（即 maxChars=1600, overlapChars=320）：

```
原始文件 (50 行):
  行 1-20  → 块 1 (约 1500 字符)
  行 17-38 → 块 2 (重叠 3 行 ≈ 320 字符)
  行 35-50 → 块 3 (剩余内容)
```

## Embedding 处理

### Embedding 缓存机制

```mermaid
flowchart TB
    CHUNK["待 Embedding 的块"] --> HASH_TEXT["SHA256(块文本)"]
    HASH_TEXT --> LOOKUP{"查询 embedding_cache<br/>WHERE provider=? AND model=?<br/>AND provider_key=? AND hash=?"}
    
    LOOKUP -->|"命中"| USE_CACHED["使用缓存向量"]
    LOOKUP -->|"未命中"| CALL_API["调用 Embedding API"]
    CALL_API --> NORMALIZE["归一化向量<br/>L2 Normalize"]
    NORMALIZE --> WRITE_CACHE["写入 embedding_cache"]
    WRITE_CACHE --> USE_NEW["使用新向量"]
    
    USE_CACHED & USE_NEW --> STORE["写入 chunks 表"]
```

**缓存键**：`(provider, model, provider_key, hash)`
- `provider_key`：API Key 的指纹/前缀，区分不同密钥
- `hash`：文本内容的 SHA256
- 同一文本、同一提供商+模型，只需 Embed 一次

### 批量 Embedding（`MemoryManagerEmbeddingOps`）

支持三种批量模式：

1. **同步批量** — 逐批调用 `embedBatch()`
2. **OpenAI Batch API** — 异步提交大批量任务
3. **Gemini Batch API** — 类似 OpenAI 的异步批量

```mermaid
sequenceDiagram
    participant SYNC as 同步流程
    participant EMBED as EmbeddingOps
    participant CACHE as embedding_cache
    participant API as Embedding API
    
    SYNC->>EMBED: embedChunks(chunks[])
    
    loop 对每个 chunk
        EMBED->>CACHE: 查缓存(hash)
        alt 缓存命中
            CACHE-->>EMBED: 返回缓存向量
        else 缓存未命中
            EMBED->>EMBED: 加入待 Embed 队列
        end
    end
    
    EMBED->>API: embedBatch(待embed文本[])
    API-->>EMBED: 返回向量[]
    EMBED->>CACHE: 批量写入缓存
    EMBED->>EMBED: 组装最终结果
```

### 向量归一化

所有 Embedding 向量在存储前统一做 **L2 归一化**：

```typescript
function sanitizeAndNormalizeEmbedding(vec: number[]): number[] {
    // 1. 替换 NaN/Infinity 为 0
    const sanitized = vec.map(v => Number.isFinite(v) ? v : 0);
    // 2. 计算 L2 范数
    const magnitude = Math.sqrt(sanitized.reduce((s, v) => s + v * v, 0));
    if (magnitude < 1e-10) return sanitized;
    // 3. 归一化
    return sanitized.map(v => v / magnitude);
}
```

## 同步触发机制

```mermaid
flowchart TB
    subgraph "触发源"
        W["文件监听器<br/>(chokidar)"]
        S["会话开始"]
        Q["搜索请求"]
        I["定时间隔"]
        M["手动 CLI"]
    end
    
    subgraph "同步调度"
        DEB["Debounce<br/>(默认 1500ms)"]
        FLAG["dirty 标记"]
        SYNC["sync()"]
    end
    
    W -->|"文件变化"| DEB
    DEB --> FLAG
    S -->|"onSessionStart=true"| SYNC
    Q -->|"onSearch=true 且 dirty"| SYNC
    I -->|"intervalMinutes > 0"| SYNC
    M -->|"openclaw memory sync"| SYNC
    FLAG --> Q
```

### 文件监听器（chokidar）

```typescript
// 监听路径
const watchPaths = [
    path.join(workspaceDir, "MEMORY.md"),
    path.join(workspaceDir, "memory"),
    ...normalizedExtraPaths
];

// 配置
chokidar.watch(watchPaths, {
    ignoreInitial: true,
    awaitWriteFinish: { stabilityThreshold: 300 }
});

// 事件处理（debounced）
watcher.on("change/add/unlink", debounce(() => {
    this.markDirty();
}, watchDebounceMs));  // 默认 1500ms
```

### 增量同步逻辑

```mermaid
flowchart TB
    SYNC_START["sync()"] --> LIST["listMemoryFiles()"]
    LIST --> |"+ listSessionFiles() 如果启用"| ALL_FILES["所有源文件"]
    
    ALL_FILES --> DIFF["对比 DB 中的 files 表"]
    
    DIFF --> NEW["新增文件"]
    DIFF --> CHANGED["变化文件<br/>(hash 不同)"]
    DIFF --> DELETED["删除的文件"]
    DIFF --> UNCHANGED["未变文件"]
    
    NEW & CHANGED --> BUILD["buildFileEntry()<br/>读取 + Hash"]
    BUILD --> CHUNK_MD["chunkMarkdown()"]
    CHUNK_MD --> DIFF_CHUNKS["对比现有 chunks"]
    
    DIFF_CHUNKS --> NEW_CHUNKS["新块 → Embed + Insert"]
    DIFF_CHUNKS --> STALE_CHUNKS["过期块 → Delete"]
    DIFF_CHUNKS --> SAME_CHUNKS["未变块 → Skip"]
    
    NEW_CHUNKS --> EMBED_BATCH["批量 Embedding"]
    EMBED_BATCH --> WRITE_DB["写入 chunks + FTS + vec"]
    
    DELETED --> CLEAN["删除相关 chunks + files 记录"]
    
    WRITE_DB & CLEAN --> DONE["同步完成"]
```

### 会话文件索引

当 `sources` 包含 `"sessions"` 时，还会索引会话日志：

```mermaid
flowchart LR
    JSONL["sessions/*.jsonl"] -->|"解析"| PARSE["buildSessionEntry()"]
    PARSE --> EXTRACT["提取 user/assistant 消息"]
    EXTRACT --> REDACT["脱敏处理<br/>redactSensitiveText()"]
    REDACT --> FORMAT["格式化为<br/>User: ... / Assistant: ..."]
    FORMAT --> INDEX["按 memory 管线索引"]
```

**JSONL 格式**：
```json
{"type":"message","message":{"role":"user","content":"设置 VLAN 10"}}
{"type":"message","message":{"role":"assistant","content":"好的，我来配置..."}}
```

**解析规则**：
1. 只提取 `type: "message"` 的行
2. 只保留 `role: "user"` 和 `role: "assistant"` 的消息
3. 内容脱敏（隐藏 API Key 等敏感信息）
4. 格式化为 `User: {text}` / `Assistant: {text}`
5. 计算 lineMap 用于引用定位

### 配置变化检测

当 Embedding 配置（provider/model/endpoint）变化时，自动全量重建：

```mermaid
flowchart LR
    CONFIG["当前配置指纹<br/>provider + model + endpoint"] --> COMPARE{"对比 DB<br/>meta.config_fingerprint"}
    COMPARE -->|"一致"| INCREMENTAL["增量同步"]
    COMPARE -->|"不一致"| RESET["全量重建<br/>清空 chunks + files"]
    RESET --> FULL_SYNC["重新扫描所有文件"]
```
