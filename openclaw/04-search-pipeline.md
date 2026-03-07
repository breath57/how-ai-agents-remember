# 04 - 搜索管线

## 搜索全流程

```mermaid
flowchart TB
    QUERY["用户查询"] --> ROUTE["getMemorySearchManager()<br/>路由到后端"]
    
    ROUTE --> |"builtin"| BUILTIN_SEARCH
    ROUTE --> |"qmd"| QMD_SEARCH
    ROUTE --> |"qmd 失败"| FALLBACK["FallbackMemoryManager<br/>自动回退 builtin"]

    subgraph BUILTIN_SEARCH["内置搜索引擎"]
        BS_START["MemoryIndexManager.search()"] --> SYNC_CHECK{"需要同步?<br/>(dirty & onSearch)"}
        SYNC_CHECK -->|"是"| DO_SYNC["同步索引"]
        SYNC_CHECK -->|"否"| SKIP_SYNC["跳过"]
        DO_SYNC & SKIP_SYNC --> EMBED_AVAILABLE{"Embedding<br/>可用?"}
        
        EMBED_AVAILABLE -->|"是"| HYBRID_SEARCH["混合搜索"]
        EMBED_AVAILABLE -->|"否"| FTS_ONLY["FTS-only 模式"]
    end

    subgraph QMD_SEARCH["QMD Sidecar 搜索"]
        QMD_START["QmdMemoryManager.search()"] --> QMD_CMD["shell: qmd search --json"]
        QMD_CMD --> QMD_PARSE["解析 JSON 结果"]
    end
    
    BUILTIN_SEARCH & QMD_SEARCH --> POST["后处理管线"]
    POST --> RESULT["MemorySearchResult[]"]
```

## 搜索路由（`getMemorySearchManager()`）

```mermaid
flowchart TB
    CALL["getMemorySearchManager(cfg, agentId)"] --> RESOLVE["resolveMemoryBackendConfig()"]
    RESOLVE --> BACKEND{"memory.backend?"}
    
    BACKEND -->|"qmd"| TRY_QMD["尝试创建 QmdMemoryManager"]
    BACKEND -->|"builtin (默认)"| BUILTIN["创建 MemoryIndexManager"]
    
    TRY_QMD --> QMD_OK{"QMD 可用?"}
    QMD_OK -->|"是"| WRAP_FALLBACK["包装为 FallbackMemoryManager<br/>(primary=QMD, fallback=builtin)"]
    QMD_OK -->|"否"| BUILTIN
    
    WRAP_FALLBACK --> CACHE["缓存到 QMD_MANAGER_CACHE"]
    BUILTIN --> IM_CACHE["缓存到 INDEX_CACHE"]
    
    CACHE & IM_CACHE --> RETURN["返回 MemorySearchManager"]
```

**FallbackMemoryManager 行为**：
- 搜索时先调用 QMD（primary）
- QMD 失败时自动降级到内置 SQLite（fallback）
- 失败后驱逐缓存，下次请求重试 QMD

## 混合搜索详解

### 搜索管线架构

```mermaid
flowchart LR
    Q["查询文本"] --> BRANCH1["向量搜索分支"]
    Q --> BRANCH2["关键词搜索分支"]
    
    BRANCH1 --> EMBED_Q["Embedding(查询)<br/>→ 查询向量"]
    EMBED_Q --> VEC_SEARCH["searchVector()<br/>cosine 距离排序"]
    VEC_SEARCH --> VEC_RESULTS["向量候选<br/>top N×multiplier"]
    
    BRANCH2 --> BUILD_FTS["buildFtsQuery()<br/>分词 → FTS5 语法"]
    BUILD_FTS --> KW_SEARCH["searchKeyword()<br/>BM25 排序"]
    KW_SEARCH --> KW_RESULTS["关键词候选<br/>top N×multiplier"]
    
    VEC_RESULTS & KW_RESULTS --> MERGE["mergeHybridResults()<br/>加权合并"]
    MERGE --> DECAY["applyTemporalDecay()<br/>时间衰减（可选）"]
    DECAY --> SORT["按 score 降序排列"]
    SORT --> MMR_CHECK{"MMR 启用?"}
    MMR_CHECK -->|"是"| MMR["applyMMR()<br/>多样性重排"]
    MMR_CHECK -->|"否"| TOPK["取 top-K"]
    MMR --> TOPK
    TOPK --> FINAL["最终结果"]
```

### 向量搜索（`searchVector()`）

两种实现路径：

```mermaid
flowchart TB
    SEARCH["searchVector()"] --> VEC_CHECK{"sqlite-vec<br/>可用?"}
    
    VEC_CHECK -->|"是"| SQL_VEC["SQL 查询<br/>vec_distance_cosine()"]
    VEC_CHECK -->|"否"| JS_FALLBACK["JS 回退<br/>cosine similarity"]
    
    SQL_VEC --> |"高效"| SQL_QUERY["SELECT ... FROM chunks_vec v<br/>JOIN chunks c ON c.id = v.id<br/>WHERE c.model = ?<br/>ORDER BY vec_distance_cosine(v.embedding, ?) ASC<br/>LIMIT ?"]
    
    JS_FALLBACK --> |"全表扫描"| JS_QUERY["加载所有 chunks<br/>逐一计算 cosine similarity<br/>排序取 top-K"]
    
    SQL_QUERY & JS_QUERY --> CONVERT["score = 1 - distance"]
    CONVERT --> RESULT["SearchRowResult[]"]
```

**sqlite-vec 路径**（推荐）：
```sql
SELECT c.id, c.path, c.start_line, c.end_line, c.text, c.source,
       vec_distance_cosine(v.embedding, ?) AS dist
  FROM chunks_vec v
  JOIN chunks c ON c.id = v.id
 WHERE c.model = ?
 ORDER BY dist ASC
 LIMIT ?
```

**JS 回退路径**：
```typescript
// 加载所有 chunks 的 embedding
const candidates = listChunks(db, providerModel);
// 逐条计算余弦相似度
const scored = candidates.map(c => ({
    chunk: c,
    score: cosineSimilarity(queryVec, c.embedding)
}));
// 排序取 top-K
return scored.sort((a, b) => b.score - a.score).slice(0, limit);
```

### 关键词搜索（`searchKeyword()`）

```mermaid
flowchart LR
    QUERY["原始查询"] --> BUILD["buildFtsQuery()"]
    BUILD --> |"分词 + 引号包裹"| FTS_QUERY["'\"term1\" AND \"term2\"'"]
    FTS_QUERY --> SQL["SELECT ... FROM chunks_fts<br/>WHERE chunks_fts MATCH ?<br/>ORDER BY bm25(chunks_fts) ASC"]
    SQL --> CONVERT["textScore = 1 / (1 + max(0, rank))"]
    CONVERT --> RESULT["SearchRowResult[]"]
```

**BM25 分数转换**：
- SQLite FTS5 的 `bm25()` 返回的是"排名"（越小越好）
- 转换公式：`score = 1 / (1 + max(0, rank))`
- 结果范围约 0-1，越大越相关

### FTS 查询构建（`buildFtsQuery()`）

将原始查询转为 FTS5 语法：
```
输入: "API design for memory"
输出: "\"api\" AND \"design\" AND \"memory\""
```

过滤规则：
- 去除停用词（支持英语、中文、日语、韩语、西班牙语等）
- 去除短词（英文 < 3 字符）
- 去除纯数字和纯标点
- 中文按字符 + 双字（bigram）分词
- 韩语去除尾缀助词

### FTS-only 模式

当 Embedding 不可用时的降级路径：

```mermaid
flowchart LR
    QUERY["查询"] --> EXTRACT["extractKeywords()<br/>关键词提取"]
    EXTRACT --> KEYWORDS["有效关键词[]"]
    KEYWORDS --> FTS["searchKeyword()<br/>FTS5 BM25 搜索"]
    FTS --> RESULT["结果 (无向量分数)"]
```

`extractKeywords()` 处理流程：
1. 分词（按空格/标点分割）
2. 中文字符 → 单字 + 双字 bigram
3. 日文 → 提取汉字/片假名/ASCII 片段
4. 韩文 → 去除尾缀助词
5. 过滤停用词（7 种语言）
6. 过滤无效词（短词、纯数字等）
7. 去重

## 混合搜索合并（`mergeHybridResults()`）

```mermaid
flowchart TB
    VEC["向量结果<br/>{id, score}[]"] --> UNION["按 chunk id 合并"]
    KW["关键词结果<br/>{id, textScore}[]"] --> UNION
    
    UNION --> CALC["计算加权分数"]
    CALC --> |"finalScore = vectorWeight × vecScore<br/>+ textWeight × textScore"| SCORED["加权结果[]"]
    
    SCORED --> DECAY_CHECK{"时间衰减<br/>启用?"}
    DECAY_CHECK -->|"是"| DECAY["applyTemporalDecay()"]
    DECAY_CHECK -->|"否"| SORT["排序"]
    DECAY --> SORT
    
    SORT --> MMR_CHECK{"MMR 启用?"}
    MMR_CHECK -->|"是"| MMR["mmrRerank()"]
    MMR_CHECK -->|"否"| TOPK["取 maxResults"]
    MMR --> TOPK
    TOPK --> FINAL["最终结果"]
```

**默认权重**：
- `vectorWeight = 0.7`（语义匹配为主）
- `textWeight = 0.3`（精确匹配补充）
- 权重自动归一化到总和 1.0

**候选数量**：
- `candidateMultiplier = 4`
- 向量搜索：取 `maxResults × 4` 条
- 关键词搜索：取 `maxResults × 4` 条
- 合并后取 top `maxResults`

### 合并算法

```typescript
// 两个分支可能返回相同 chunk
// 按 chunk id 建立映射
const map = new Map<string, MergedEntry>();

for (const vec of vectorResults) {
    map.set(vec.id, { ...vec, vecScore: vec.score, textScore: 0 });
}

for (const kw of keywordResults) {
    const existing = map.get(kw.id);
    if (existing) {
        existing.textScore = kw.textScore;       // 补充关键词分数
    } else {
        map.set(kw.id, { ...kw, vecScore: 0, textScore: kw.textScore });
    }
}

// 计算最终分数
for (const entry of map.values()) {
    entry.score = vectorWeight * entry.vecScore + textWeight * entry.textScore;
}
```

## 时间衰减（`applyTemporalDecayToHybridResults()`）

```mermaid
graph LR
    subgraph "衰减公式"
        FORMULA["decayedScore = score × e^(-λ × ageInDays)<br/>λ = ln(2) / halfLifeDays"]
    end
    
    subgraph "衰减效果（halfLife=30天）"
        D0["今天: 100%"]
        D7["7天前: 84%"]
        D30["30天前: 50%"]
        D90["90天前: 12.5%"]
        D180["180天前: 1.6%"]
    end
```

**时间来源优先级**：

```mermaid
flowchart TB
    FILE["文件路径"] --> DATE_CHECK{"路径包含日期?<br/>memory/YYYY-MM-DD.md"}
    DATE_CHECK -->|"是"| USE_DATE["使用文件名日期"]
    DATE_CHECK -->|"否"| EVERGREEN_CHECK{"是常青文件?<br/>MEMORY.md 或<br/>memory/topic.md"}
    EVERGREEN_CHECK -->|"是"| NO_DECAY["不衰减 (返回 null)"]
    EVERGREEN_CHECK -->|"否"| MTIME["使用文件 mtime"]
```

**常青文件规则**（不衰减）：
- `MEMORY.md` / `memory.md`（根记忆文件）
- `memory/` 下非日期命名的文件（如 `memory/projects.md`）

**日期文件**：
- 匹配 `memory/YYYY-MM-DD.md` 格式
- 从文件名提取日期，按天计算年龄

## MMR 多样性重排（`applyMMRToHybridResults()`）

```mermaid
flowchart TB
    INPUT["排序后的候选结果"] --> INIT["选空结果集 S = {}, 候选集 C = 全部"]
    INIT --> LOOP["迭代选择"]
    
    LOOP --> CALC["对 C 中每个 d 计算:<br/>MMR(d) = λ × relevance(d)<br/>- (1-λ) × max{sim(d, s) | s ∈ S}"]
    CALC --> SELECT["选 MMR 最高的 d*"]
    SELECT --> ADD["S += {d*}, C -= {d*}"]
    ADD --> CHECK{"|S| ≥ maxResults?"}
    CHECK -->|"否"| LOOP
    CHECK -->|"是"| DONE["返回 S"]
```

**参数**：
- `lambda = 0.7`（默认）
  - `1.0` = 纯相关性（无多样性）
  - `0.0` = 纯多样性（忽略相关性）
  - `0.7` = 偏向相关性的平衡

**相似度计算**：
- 基于 Jaccard 相似度（token 集合交集/并集）
- 分词后比较，不需要 Embedding

**效果示例**：
```
无 MMR:                            有 MMR (λ=0.7):
1. 路由器配置 (0.92)              1. 路由器配置 (0.92)
2. 路由器配置-重复 (0.89) ←重复    2. 网络拓扑文档 (0.85) ←多样
3. 网络拓扑文档 (0.85)            3. DNS 配置 (0.78) ←多样
```

## 搜索配置汇总

```mermaid
graph TB
    subgraph "查询参数"
        MR["maxResults = 6"]
        MS["minScore = 0.35"]
    end
    
    subgraph "混合搜索"
        HE["hybrid.enabled = true"]
        VW["vectorWeight = 0.7"]
        TW["textWeight = 0.3"]
        CM["candidateMultiplier = 4"]
    end
    
    subgraph "后处理"
        MMR_E["mmr.enabled = false"]
        MMR_L["mmr.lambda = 0.7"]
        TD_E["temporalDecay.enabled = false"]
        TD_H["temporalDecay.halfLifeDays = 30"]
    end
    
    subgraph "结果格式"
        PATH["path: 文件路径"]
        LINES["startLine, endLine"]
        SCORE["score: 0-1"]
        SNIPPET["snippet: ≤700 字符"]
        SOURCE["source: memory|sessions"]
        CITE["citation?: path#L1-L5"]
    end
```

## 引用模式（Citations）

```mermaid
flowchart LR
    MODE{"memory.citations?"}
    MODE -->|"auto (默认)"| AUTO["DM → 显示引用<br/>群组/频道 → 隐藏引用"]
    MODE -->|"on"| ON["总是显示引用"]
    MODE -->|"off"| OFF["总是隐藏引用"]
    
    AUTO & ON --> FORMAT["Source: path#L1-L5<br/>追加到 snippet 末尾"]
```
