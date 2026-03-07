# 06 - 跨通道记忆与会话压缩

## 问题背景

用户可能通过 Telegram 跟 Agent 聊天，中途切换到 Discord 继续。传统设计下每个通道（Channel）有独立会话，切换后会丢失上下文。

OpenFang 通过 **CanonicalSession** 解决了这个问题。

## CanonicalSession 设计

```mermaid
graph TB
    subgraph "Channel A (Telegram)"
        SA[Session A]
    end
    subgraph "Channel B (Discord)"
        SB[Session B]
    end
    subgraph "Channel C (Web)"
        SC[Session C]
    end

    CS[CanonicalSession<br/>每 Agent 唯一]

    SA -->|append_canonical| CS
    SB -->|append_canonical| CS
    SC -->|append_canonical| CS

    CS -->|canonical_context| SA
    CS -->|canonical_context| SB
    CS -->|canonical_context| SC
```

### 核心原理

1. **写入**：每个通道的对话消息实时追加到 CanonicalSession
2. **读取**：新通道对话开始时，从 CanonicalSession 获取摘要 + 最近消息注入 system prompt
3. **压缩**：消息数超过阈值时，旧消息被摘要化

## 数据结构

```rust
pub struct CanonicalSession {
    pub agent_id: AgentId,              // 所属 Agent
    pub messages: Vec<Message>,         // 压缩窗口内的消息
    pub compaction_cursor: usize,       // 压缩进度
    pub compacted_summary: Option<String>, // 旧消息摘要文本
    pub updated_at: String,
}
```

常量：
```rust
const DEFAULT_CANONICAL_WINDOW: usize = 50;      // 保留最近 50 条消息
const DEFAULT_COMPACTION_THRESHOLD: usize = 100;  // 超过 100 条触发压缩
```

## 压缩流程

```mermaid
sequenceDiagram
    participant CH as Channel
    participant SS as SessionStore
    participant DB as SQLite

    CH->>SS: append_canonical(agent_id, new_messages, threshold?)
    SS->>DB: load_canonical(agent_id)
    DB-->>SS: CanonicalSession

    SS->>SS: messages.extend(new_messages)

    alt messages.len() > threshold (100)
        SS->>SS: keep_count = 50 (WINDOW)
        SS->>SS: to_compact = len - keep_count

        Note over SS: 构建摘要
        SS->>SS: 对每条旧消息截取前 200 字符
        SS->>SS: 格式化为 "Role: content..."
        SS->>SS: 拼合到已有摘要
        SS->>SS: 总摘要限制 4000 字符（UTF-8 安全截断）

        SS->>SS: messages = messages[to_compact..]
        SS->>SS: compaction_cursor = 0
    end

    SS->>DB: save_canonical(session)
    SS-->>CH: CanonicalSession
```

## 摘要构建算法

```python
# 伪代码
def build_summary(existing_summary, messages_to_compact):
    parts = []
    if existing_summary:
        parts.append(existing_summary)

    for msg in messages_to_compact:
        role = msg.role  # "User" / "Assistant" / "System"
        text = msg.text_content()
        if text:
            truncated = text[:200] + "..." if len(text) > 200 else text
            parts.append(f"{role}: {truncated}")

    full = "\n".join(parts)

    # 限制总长度为 4000 字符（从尾部保留）
    if len(full) > 4000:
        full = full[-4000:]  # UTF-8 安全边界处理

    return full
```

### 关键细节

1. **尾部保留**：摘要超长时从尾部截断（保留最近的摘要，丢弃最远的）
2. **UTF-8 安全**：截断时找到合法字符边界 `is_char_boundary()`
3. **200 字符截断**：每条消息在摘要中最多保留 200 字符

## 上下文注入流程

```mermaid
sequenceDiagram
    participant AL as Agent Loop
    participant SS as SessionStore
    participant LLM as LLM Driver

    AL->>SS: canonical_context(agent_id, window_size=50)
    SS->>SS: load_canonical(agent_id)
    SS->>SS: 取最后 50 条消息
    SS-->>AL: (compacted_summary?, recent_messages)

    alt 有 compacted_summary
        AL->>AL: system_prompt += "<br/>Previous conversation context:<br/>" + summary
    end

    AL->>LLM: send_message(system_prompt, messages + recent_messages, tools)
```

## LLM 摘要压缩（高级模式）

除了基于文本截断的压缩，OpenFang 还支持 LLM 生成摘要：

```rust
pub fn store_llm_summary(
    &self,
    agent_id: AgentId,
    summary: &str,               // LLM 生成的智能摘要
    kept_messages: Vec<Message>,  // 保留的最近消息
) -> OpenFangResult<()>
```

这个方法由 `openfang-runtime` 中的 **LLM session compactor** 调用：
1. 检测到会话消息数超过上下文窗口 80%
2. 将旧消息发送给 LLM 生成摘要
3. 用 `store_llm_summary` 替换旧消息

```mermaid
graph TD
    A[检测到消息数 > 80% 上下文窗口] --> B[选取旧消息]
    B --> C[发送给 LLM 生成摘要]
    C --> D[store_llm_summary]
    D --> E[compacted_summary = LLM 摘要]
    D --> F[messages = 保留的最近消息]
    D --> G[compaction_cursor = 0]
```

## 完整的跨通道场景

```mermaid
sequenceDiagram
    participant T as Telegram
    participant D as Discord
    participant W as Web
    participant CS as CanonicalSession

    Note over T: 用户在 Telegram
    T->>CS: append: "My name is Alice"
    T->>CS: append: "I work at Acme Corp"

    Note over D: 用户切换到 Discord
    D->>CS: canonical_context()
    CS-->>D: summary=None, messages=[<br/>"My name is Alice",<br/>"I work at Acme Corp"]
    Note over D: Agent 知道用户是 Alice

    D->>CS: append: "What projects am I on?"
    D->>CS: append: "You're on Project X"

    Note over W: 用户在 Web 端
    W->>CS: canonical_context()
    CS-->>W: summary=None, messages=[4条]
    Note over W: Agent 拥有完整上下文
```

## 与 Regular Session 的区别

| 特性 | Regular Session | Canonical Session |
|------|-----------------|-------------------|
| 数量 | 每 Agent 多个 | 每 Agent 唯一 |
| 创建 | 每次新对话 | Agent 首次交互 |
| 关联 | 绑定特定通道/交互 | 跨所有通道 |
| 压缩 | 由 runtime compactor 处理 | 内建压缩逻辑 |
| 用途 | 当前对话上下文 | 持久记忆（"认识你"） |
| 存储表 | `sessions` | `canonical_sessions` |
