# 06 - 预压缩记忆刷写（Memory Flush）

## 核心问题

AI Agent 的上下文窗口有限。当对话接近 token 上限时，系统会执行 **上下文压缩（compaction）**，丢弃旧的对话内容。如果 Agent 在对话中获得了有价值的信息但没有写入磁盘，这些信息会在压缩时永久丢失。

## 解决方案

在压缩发生 **之前**，自动触发一个 **静默的 Agent 回合**，提醒 Agent 将有价值的记忆写入 Markdown 文件。

```mermaid
sequenceDiagram
    participant USER as 用户
    participant SESSION as 会话管理器
    participant FLUSH as Memory Flush
    participant AGENT as Agent
    participant FILE as 记忆文件

    USER->>SESSION: 发送消息
    SESSION->>SESSION: 检查 token 使用量
    
    alt token > threshold
        SESSION->>FLUSH: shouldRunMemoryFlush() = true
        FLUSH->>AGENT: 静默提示<br/>"会话即将压缩，请保存记忆"
        AGENT->>FILE: 写入 memory/2026-03-07.md
        AGENT-->>FLUSH: NO_REPLY（静默）
        FLUSH->>SESSION: 标记已刷写
        SESSION->>SESSION: 执行 compaction
    else token < threshold
        SESSION->>SESSION: 继续正常对话
    end
```

## 触发条件

```mermaid
flowchart TB
    CHECK["shouldRunMemoryFlush()"] --> ENTRY{"有 SessionEntry?"}
    ENTRY -->|"否"| NO["不触发"]
    
    ENTRY -->|"是"| TOKENS["获取 totalTokens"]
    TOKENS --> CALC["计算阈值<br/>threshold = contextWindow<br/>- reserveTokensFloor<br/>- softThresholdTokens"]
    
    CALC --> COMPARE{"totalTokens ≥ threshold?"}
    COMPARE -->|"否"| NO
    COMPARE -->|"是"| ALREADY{"本轮 compaction<br/>已刷写过?"}
    ALREADY -->|"是"| NO
    ALREADY -->|"否"| YES["触发 Flush"]
    
    style YES fill:#e8f5e9
    style NO fill:#ffcdd2
```

### 阈值计算

```
threshold = contextWindow - reserveTokensFloor - softThresholdTokens
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `contextWindow` | 模型决定 | 模型的上下文窗口大小 |
| `reserveTokensFloor` | 20,000 | 为压缩预留的 token 数 |
| `softThresholdTokens` | 4,000 | 提前触发的缓冲区 |

**示例**（GPT-4 128K 窗口）：
```
threshold = 128,000 - 20,000 - 4,000 = 104,000
当 totalTokens ≥ 104,000 时触发 Memory Flush
```

### 额外触发：转录文件大小

```typescript
forceFlushTranscriptBytes = 2 * 1024 * 1024  // 2MB
```

当会话转录文件达到 2MB 时也会强制触发，不依赖 token 计数。

## 刷写提示词

### 用户提示（User Prompt）

```
Pre-compaction memory flush.
Store durable memories now (use memory/YYYY-MM-DD.md; create memory/ if needed).
IMPORTANT: If the file already exists, APPEND new content only and do not overwrite existing entries.
If nothing to store, reply with NO_REPLY.
```

**动态替换**：`YYYY-MM-DD` 会替换为当前日期（按用户时区）

### 系统提示（System Prompt）

```
Pre-compaction memory flush turn.
The session is near auto-compaction; capture durable memories to disk.
You may reply, but usually NO_REPLY is correct.
```

## 单次保护

```mermaid
flowchart LR
    FLUSH["Flush 执行"] --> UPDATE["更新 SessionEntry<br/>memoryFlushCompactionCount<br/>= compactionCount"]
    
    NEXT["下次检查"] --> CHECK{"memoryFlushCompactionCount<br/>== compactionCount?"}
    CHECK -->|"是"| SKIP["跳过（本轮已刷写）"]
    CHECK -->|"否"| ALLOW["允许刷写"]
```

每个 compaction 周期只执行一次 Flush，通过 `memoryFlushCompactionCount` 追踪。

## 配置

```json5
{
    agents: {
        defaults: {
            compaction: {
                reserveTokensFloor: 20000,
                memoryFlush: {
                    enabled: true,                    // 默认开启
                    softThresholdTokens: 4000,        // 提前 4K token 触发
                    forceFlushTranscriptBytes: "2MB", // 文件大小强制触发
                    prompt: "...",                     // 自定义用户提示
                    systemPrompt: "...",               // 自定义系统提示
                }
            }
        }
    }
}
```

## 与记忆写入的关系

```mermaid
flowchart TB
    subgraph "记忆写入时机"
        USER_ASK["用户明确要求<br/>'记住这个'"]
        AGENT_DECIDE["Agent 自行决定<br/>重要信息待保存"]
        FLUSH_TRIGGER["Flush 自动提醒<br/>压缩前保存"]
        AUTO_CAPTURE["LanceDB 自动捕获<br/>规则匹配"]
    end
    
    subgraph "写入目标"
        DAILY["memory/YYYY-MM-DD.md<br/>日常笔记"]
        CURATED["MEMORY.md<br/>精选长期记忆"]
        LANCE["LanceDB<br/>向量记忆"]
    end
    
    USER_ASK --> CURATED & DAILY
    AGENT_DECIDE --> DAILY
    FLUSH_TRIGGER --> DAILY
    AUTO_CAPTURE --> LANCE
```

## 设计原则

1. **静默执行** — 用户不会看到 Flush 回合（`NO_REPLY` token）
2. **只追加** — 提示词明确要求 APPEND，不覆盖已有内容
3. **幂等** — 同一 compaction 周期只执行一次
4. **可降级** — 如果工作区只读，跳过 Flush
5. **可定制** — 提示词和阈值均可配置
