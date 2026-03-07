# 07 — 渐进式发布与可靠性 (Rollout & Reliability)

## 概览

Nullclaw 提供了一套完整的渐进式发布机制，允许在不停机的情况下从纯关键字检索平滑过渡到混合（关键字 + 向量）检索。同时，CircuitBreaker + Outbox + Diagnostics 三者协作确保发布过程中的可靠性。

## 1. RolloutPolicy（渐进式发布策略）

### 四种模式

```mermaid
stateDiagram-v2
    [*] --> off
    off --> shadow : 开启观测
    shadow --> canary : 灰度发布
    canary --> on : 全量上线
    on --> shadow : 回退观测
    shadow --> off : 完全关闭

    state off {
        [*]: 仅关键字检索
    }
    state shadow {
        [*]: 双路执行，只返回关键字结果
        note: 向量结果仅用于日志对比
    }
    state canary {
        [*]: 按百分比路由
        note: FNV1a_32(session_id) % 100
    }
    state on {
        [*]: 全量混合检索
    }
```

### 模式详情

| 模式 | 关键字检索 | 向量检索 | 返回结果 | 用途 |
|------|-----------|---------|---------|------|
| `off` | ✅ 执行 | ❌ 不执行 | 关键字 | 默认/保守模式 |
| `shadow` | ✅ 执行 | ✅ 执行 | 关键字 | A/B 对比观测 |
| `canary` | ✅ 执行 | 按比例 | 混合或关键字 | 灰度验证 |
| `on` | ✅ 执行 | ✅ 执行 | 混合 | 全量上线 |

### decide() 决策逻辑

```python
class RolloutPolicy:
    mode: str       # "off" | "shadow" | "canary" | "on"
    canary_percent: int  # 0-100, canary 模式下的流量百分比

    def decide(self, session_id: Optional[str]) -> str:
        if self.mode == "off":
            return "keyword_only"
        elif self.mode == "on":
            return "hybrid"
        elif self.mode == "shadow":
            return "shadow_hybrid"
        elif self.mode == "canary":
            if session_id is None:
                return "keyword_only"
            hash_val = fnv1a_32(session_id.encode())
            if hash_val % 100 < self.canary_percent:
                return "hybrid"
            else:
                return "keyword_only"
```

### Canary 路由一致性

使用 FNV-1a 32-bit 哈希确保同一 `session_id` 始终被分到同一组：

```mermaid
flowchart LR
    SID["session_id"] --> HASH["FNV1a_32(session_id)"]
    HASH --> MOD["hash % 100"]
    MOD --> CMP{"< canary_percent?"}
    CMP -->|是| HYBRID["hybrid 检索"]
    CMP -->|否| KW["keyword_only 检索"]
```

**特性**：
- 哈希分布均匀，百分比精确
- 同一 session 多次查询结果一致
- `session_id = null` → 保守回退到 keyword_only

## 2. Shadow 模式详解

### 运行流程

```mermaid
sequenceDiagram
    participant App as 应用层
    participant RT as MemoryRuntime
    participant KW as 关键字检索
    participant VEC as 向量检索
    participant LOG as 日志

    App->>RT: search(query, session_id)
    RT->>RT: rollout.decide() == "shadow_hybrid"

    par 双路并行
        RT->>KW: keyword_search(query)
        KW-->>RT: keyword_results
    and
        RT->>VEC: vector_search(query)
        VEC-->>RT: vector_results
    end

    RT->>LOG: log_comparison(keyword_results, vector_results)
    RT-->>App: keyword_results  # 只返回关键字结果
```

### Shadow 对比指标

- 结果重叠率（overlap ratio）
- 向量独有的高分结果
- 关键字独有的结果
- 延迟差异

## 3. 可靠性保障三件套

### 协作关系

```mermaid
graph TB
    subgraph "写入路径"
        WRITE["store(key, content)"] --> BACKEND["Primary Backend"]
        WRITE --> OUTBOX["VectorOutbox<br/>持久写入"]
    end

    subgraph "同步路径"
        OUTBOX --> CB{"CircuitBreaker<br/>状态?"}
        CB -->|closed/half_open| EMBED["EmbeddingProvider"]
        EMBED --> VS["VectorStore"]
        CB -->|open| WAIT["等待冷却"]
    end

    subgraph "查询路径"
        QUERY["search(query)"] --> RP{"RolloutPolicy<br/>decide()"}
        RP -->|keyword_only| KW["关键字检索"]
        RP -->|hybrid| BOTH["关键字 + 向量"]
        RP -->|shadow_hybrid| SHADOW["双路执行，返回关键字"]
    end

    subgraph "诊断"
        DIAG["Diagnostics"] -.->|监控| CB
        DIAG -.->|监控| OUTBOX
        DIAG -.->|监控| VS
        DIAG -.->|监控| BACKEND
    end
```

### CircuitBreaker 状态机

```mermaid
stateDiagram-v2
    [*] --> Closed

    Closed --> Open : 连续失败 >= threshold
    Open --> HalfOpen : cooldown 超时
    HalfOpen --> Closed : 单次探测成功
    HalfOpen --> Open : 探测失败

    state Closed {
        [*]: 正常放行所有请求
        note: failure_count++/reset
    }
    state Open {
        [*]: 拒绝所有请求
        note: 等待 cooldown_ns
    }
    state HalfOpen {
        [*]: 放行 1 个探测请求
        note: 成功→Closed，失败→Open
    }
```

**配置**：
```python
@dataclass
class CircuitBreakerConfig:
    failure_threshold: int = 5       # 触发 Open 的连续失败次数
    cooldown_secs: int = 60          # Open → HalfOpen 等待秒数
    half_open_max_probes: int = 1    # HalfOpen 放行探测数
```

### VectorOutbox 持久化保障

```mermaid
flowchart TD
    STORE["store(key, content)"] --> ENQ["enqueue(key, 'upsert')"]
    FORGET["forget(key)"] --> ENQ2["enqueue(key, 'delete')"]

    ENQ --> TABLE["vector_outbox 表<br/>id, memory_key, operation,<br/>attempts, last_error, status"]
    ENQ2 --> TABLE

    DRAIN["drain()"] --> SEL["SELECT pending<br/>LIMIT 50"]
    SEL --> EACH{"逐条处理"}
    EACH -->|upsert| EMB["embed → upsert to VectorStore"]
    EACH -->|delete| DEL["delete from VectorStore"]
    EMB --> OK{"成功?"}
    DEL --> OK
    OK -->|是| RM["DELETE from outbox"]
    OK -->|否| RETRY["attempts++ & last_error 更新"]
```

**特性**：
- 最大批次 50 条
- 失败不删除，下次 drain 重试
- 与 CircuitBreaker 配合：breaker open 时 drain 跳过

## 4. 推荐发布流程

### 从零到全量的完整路径

```mermaid
flowchart TD
    START[1. 部署向量基础设施] --> CONFIG[2. 配置 EmbeddingProvider<br/>+ VectorStore]
    CONFIG --> ROLLOUT_OFF[3. rollout=off<br/>纯关键字运行]
    ROLLOUT_OFF --> BACKFILL[4. Outbox drain 回填历史向量]
    BACKFILL --> SHADOW[5. rollout=shadow<br/>观测向量质量]
    SHADOW --> METRICS{6. 对比指标达标?}
    METRICS -->|否| TUNE[调优 Embedding/参数]
    TUNE --> SHADOW
    METRICS -->|是| CANARY[7. rollout=canary<br/>canary_percent=10]
    CANARY --> RAMP[8. 逐步提升百分比<br/>10→25→50→100]
    RAMP --> ON[9. rollout=on<br/>全量混合检索]
```

### 各阶段检查清单

| 阶段 | 检查项 | Diagnostics 字段 |
|------|--------|-----------------|
| shadow | 向量结果不为空 | `vector_entry_count > 0` |
| shadow | Outbox 清空 | `outbox_pending == 0` |
| shadow | CircuitBreaker 健康 | `circuit_breaker.state == closed` |
| canary | 10% 流量无异常 | 日志无错误 |
| canary | 向量延迟可接受 | 自定义监控 |
| on | 全量无降级 | `backend_healthy == true` |

## 5. 故障降级策略

```mermaid
flowchart TD
    SEARCH["search(query)"] --> RP["RolloutPolicy.decide()"]
    RP --> MODE{返回模式}
    MODE -->|keyword_only| KW["关键字检索"]
    MODE -->|hybrid| HYBRID_PATH

    subgraph HYBRID_PATH["混合检索路径"]
        direction TB
        CB_CHECK{"CircuitBreaker<br/>允许?"}
        CB_CHECK -->|closed| VEC["向量检索"]
        CB_CHECK -->|open| FALLBACK["降级: 仅关键字"]
        VEC --> VEC_OK{"成功?"}
        VEC_OK -->|是| MERGE["RRF 合并"]
        VEC_OK -->|否| CB_FAIL["CircuitBreaker.recordFailure()"]
        CB_FAIL --> FALLBACK
    end

    KW --> RESULT["返回结果"]
    MERGE --> RESULT
    FALLBACK --> RESULT
```

**核心原则**：向量平面的任何故障都不会导致整体检索失败，始终降级到关键字检索。
