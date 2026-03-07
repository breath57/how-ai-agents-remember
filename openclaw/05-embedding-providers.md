# 05 - Embedding 提供商

## 多提供商架构

```mermaid
flowchart TB
    FACTORY["createEmbeddingProvider(options)"]
    
    FACTORY --> MODE{"provider 参数?"}
    
    MODE -->|"auto"| AUTO["自动选择"]
    MODE -->|"openai"| P_OAI["OpenAI"]
    MODE -->|"gemini"| P_GEM["Gemini"]
    MODE -->|"voyage"| P_VOY["Voyage"]
    MODE -->|"mistral"| P_MIS["Mistral"]
    MODE -->|"ollama"| P_OLL["Ollama"]
    MODE -->|"local"| P_LOC["node-llama-cpp"]
    
    AUTO --> AUTO_SEQ["按序尝试"]
    AUTO_SEQ --> |"1"| TRY_LOCAL{"本地模型<br/>文件存在?"}
    TRY_LOCAL -->|"是"| P_LOC
    TRY_LOCAL -->|"否"| TRY_OAI{"OpenAI<br/>API Key?"}
    TRY_OAI -->|"是"| P_OAI
    TRY_OAI -->|"否"| TRY_GEM{"Gemini<br/>API Key?"}
    TRY_GEM -->|"是"| P_GEM
    TRY_GEM -->|"否"| TRY_VOY{"Voyage<br/>API Key?"}
    TRY_VOY -->|"是"| P_VOY
    TRY_VOY -->|"否"| TRY_MIS{"Mistral<br/>API Key?"}
    TRY_MIS -->|"是"| P_MIS
    TRY_MIS -->|"否"| FTS_ONLY["无 Embedding<br/>FTS-only 模式"]

    P_OAI & P_GEM & P_VOY & P_MIS & P_OLL & P_LOC --> RESULT["EmbeddingProvider"]
    
    style FTS_ONLY fill:#fff3e0
    style RESULT fill:#e8f5e9
```

## 提供商对比

| 提供商 | 默认模型 | 维度 | 类型 | 自动选择 | 备注 |
|--------|---------|------|------|---------|------|
| `openai` | text-embedding-3-small | 1536 | 远程 | ✅ 第2优先 | 推荐，支持 Batch API |
| `gemini` | gemini-embedding-001 | — | 远程 | ✅ 第3优先 | 支持 Batch API |
| `voyage` | voyage-4-large | — | 远程 | ✅ 第4优先 | 高质量 |
| `mistral` | mistral-embed | — | 远程 | ✅ 第5优先 | |
| `ollama` | nomic-embed-text | — | 本地/自托管 | ❌ 不自动 | 需要 Ollama 守护进程 |
| `local` | embeddinggemma-300m | — | 本地 | ✅ 第1优先* | 需 node-llama-cpp |

> *Local 仅在指定的模型文件存在时自动选择，不会自动下载

## `EmbeddingProvider` 接口

```typescript
type EmbeddingProvider = {
    id: string;              // "openai" | "local" | "gemini" | ...
    model: string;           // 模型名称
    maxInputTokens?: number; // 最大输入 token 数
    embedQuery(text: string): Promise<number[]>;      // 单条 Embedding
    embedBatch(texts: string[]): Promise<number[][]>;  // 批量 Embedding
};
```

## 降级策略（Fallback）

```mermaid
flowchart TB
    PRIMARY["主提供商"] --> TRY_PRIMARY["尝试创建"]
    TRY_PRIMARY --> PRIMARY_OK{"成功?"}
    
    PRIMARY_OK -->|"是"| USE_PRIMARY["使用主提供商"]
    PRIMARY_OK -->|"否"| CHECK_FALLBACK{"有 fallback<br/>配置?"}
    
    CHECK_FALLBACK -->|"是"| TRY_FALLBACK["尝试 fallback 提供商"]
    CHECK_FALLBACK -->|"否"| CHECK_AUTH{"API Key<br/>缺失错误?"}
    
    TRY_FALLBACK --> FALLBACK_OK{"成功?"}
    FALLBACK_OK -->|"是"| USE_FALLBACK["使用 fallback<br/>记录降级原因"]
    FALLBACK_OK -->|"否"| BOTH_AUTH{"两者都是<br/>Key 缺失?"}
    
    BOTH_AUTH -->|"是"| FTS_ONLY["FTS-only 模式"]
    BOTH_AUTH -->|"否"| THROW["抛出错误"]
    
    CHECK_AUTH -->|"是"| FTS_ONLY
    CHECK_AUTH -->|"否"| THROW
    
    style FTS_ONLY fill:#fff3e0
    style THROW fill:#ffcdd2
```

**降级结果报告**：

```typescript
type EmbeddingProviderResult = {
    provider: EmbeddingProvider | null;  // null = FTS-only
    requestedProvider: "auto" | "openai" | ...;
    fallbackFrom?: string;        // 从哪个提供商降级来的
    fallbackReason?: string;      // 降级原因
    providerUnavailableReason?: string;  // 不可用原因
};
```

## 各提供商实现细节

### OpenAI

```mermaid
sequenceDiagram
    participant APP as 应用
    participant PROV as OpenAI Provider
    participant API as OpenAI API
    
    APP->>PROV: embedQuery("查询文本")
    PROV->>API: POST /v1/embeddings<br/>{model, input}
    API-->>PROV: {data: [{embedding: [...]}]}
    PROV->>PROV: L2 归一化
    PROV-->>APP: number[]
```

**配置**：
- 支持自定义 `baseUrl`（兼容 OpenRouter、vLLM 等）
- 支持自定义 `headers`
- 支持 Batch API（大规模索引优化）

### Gemini

```mermaid
sequenceDiagram
    participant APP as 应用
    participant PROV as Gemini Provider
    participant API as Gemini API
    
    APP->>PROV: embedBatch(texts[])
    PROV->>API: POST embedContent<br/>{model, content}
    API-->>PROV: {embedding: {values: [...]}}
    PROV->>PROV: L2 归一化
    PROV-->>APP: number[][]
```

**API Key 来源**：`GEMINI_API_KEY` 或 `models.providers.google.apiKey`

### Ollama（本地/自托管）

```mermaid
sequenceDiagram
    participant APP as 应用
    participant PROV as Ollama Provider
    participant DAEMON as Ollama 守护进程
    
    APP->>PROV: embedQuery("查询文本")
    PROV->>DAEMON: POST /api/embeddings<br/>{model, prompt}
    DAEMON-->>PROV: {embedding: [...]}
    PROV->>PROV: L2 归一化
    PROV-->>APP: number[]
```

**特点**：
- 不在自动选择列表中（需显式指定 `provider: "ollama"`）
- 通常不需要真实 API Key（占位符即可）
- 默认模型：`nomic-embed-text`

### Local（node-llama-cpp）

```mermaid
flowchart TB
    INIT["初始化"] --> LAZY["lazy-load node-llama-cpp"]
    LAZY --> RESOLVE["resolveModelFile(modelPath)"]
    RESOLVE --> DOWNLOAD{"本地文件<br/>存在?"}
    DOWNLOAD -->|"否"| AUTO_DL["自动下载 GGUF"]
    DOWNLOAD -->|"是"| LOAD["加载模型"]
    AUTO_DL --> LOAD
    LOAD --> CTX["创建 EmbeddingContext"]
    CTX --> READY["就绪"]
    
    QUERY["embedQuery(text)"] --> READY
    READY --> GGUF["GGUF 推理"]
    GGUF --> NORMALIZE["L2 归一化"]
    NORMALIZE --> RESULT["向量"]
```

**默认模型**：`hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF` (~0.6GB)
**前提**：需要安装 `node-llama-cpp`（`pnpm approve-builds` 后构建原生模块）

## Batch API 支持

对于大规模索引，支持 OpenAI / Gemini / Voyage 的 Batch API：

```mermaid
sequenceDiagram
    participant SYNC as 索引同步
    participant BATCH as Batch Runner
    participant API as Embedding API (Batch)
    
    SYNC->>BATCH: 提交大批量文本
    BATCH->>API: 上传 batch 文件
    API-->>BATCH: batch_id
    
    loop 轮询 (每 pollIntervalMs)
        BATCH->>API: 查询 batch 状态
        API-->>BATCH: status: processing/completed
    end
    
    API-->>BATCH: 完成，返回结果文件
    BATCH->>BATCH: 解析结果
    BATCH-->>SYNC: 向量数组
```

**配置**：
```json5
{
    remote: {
        batch: {
            enabled: true,       // 默认 false
            wait: true,          // 等待完成
            concurrency: 2,      // 并行 batch 数
            pollIntervalMs: 2000,
            timeoutMinutes: 60
        }
    }
}
```

## 向量归一化

所有提供商返回的向量统一做 L2 归一化：

```
向量 v = [v1, v2, ..., vn]
范数 ||v|| = sqrt(v1² + v2² + ... + vn²)
归一化 v' = v / ||v||
```

这确保：
1. 余弦相似度计算正确性
2. 不同提供商的向量可比
3. NaN/Infinity 替换为 0
