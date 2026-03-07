# LLM 驱动的记忆整合机制

> 这是 nanobot 记忆系统的**核心创新点**：用 LLM 自身来提取和整理记忆，而非简单截断或规则摘要。

## 1. 整合流程总览

```mermaid
flowchart TD
    A["触发整合"] --> B{"archive_all?"}
    B -->|"True (/new 命令)"| C["整合全部未整合消息"]
    B -->|"False (自动触发)"| D["整合 messages[last_consolidated : -keep_count]<br/>keep_count = memory_window // 2"]
    C --> E["格式化消息为文本行"]
    D --> E
    E --> F["读取当前 MEMORY.md"]
    F --> G["构建 consolidation prompt"]
    G --> H["调用 LLM（角色: memory consolidation agent）<br/>提供 save_memory 工具"]
    H --> I{"LLM 是否调用 save_memory?"}
    I -->|"是"| J["解析 save_memory 参数"]
    I -->|"否"| K["返回 False，整合失败"]
    J --> L["history_entry → 追加到 HISTORY.md"]
    J --> M{"memory_update != 当前内容?"}
    M -->|"是"| N["覆盖写入 MEMORY.md"]
    M -->|"否"| O["不写入（无变化）"]
    L --> P["更新 session.last_consolidated"]
    N --> P
    O --> P
    P --> Q["返回 True，整合成功"]
```

## 2. 触发机制

### 2.1 自动触发（后台异步）

**位置**：`AgentLoop._process_message()` 的消息处理末尾

```python
unconsolidated = len(session.messages) - session.last_consolidated
if unconsolidated >= self.memory_window and session.key not in self._consolidating:
    self._consolidating.add(session.key)
    lock = self._consolidation_locks.setdefault(session.key, asyncio.Lock())

    async def _consolidate_and_unlock():
        try:
            async with lock:
                await self._consolidate_memory(session)
        finally:
            self._consolidating.discard(session.key)

    task = asyncio.create_task(_consolidate_and_unlock())
    self._consolidation_tasks.add(task)  # 强引用，防止 GC
```

**条件**：未整合消息数 ≥ `memory_window`（默认 100）

**特点**：
- `asyncio.create_task()` — 后台执行，不阻塞当前响应
- `_consolidating` 集合 — 防止同一 session 并发整合
- `_consolidation_locks` — `WeakValueDictionary` 管理锁，避免内存泄漏
- `_consolidation_tasks` — 强引用集合，防止任务被 GC 回收

### 2.2 `/new` 命令触发（同步）

**位置**：`AgentLoop._process_message()` 的 `/new` 命令处理

```python
# 1. 快照未整合消息
snapshot = session.messages[session.last_consolidated:]
if snapshot:
    temp = Session(key=session.key)
    temp.messages = list(snapshot)  # 深拷贝，避免数据冲突
    # 2. 同步等待整合完成
    if not await self._consolidate_memory(temp, archive_all=True):
        return "Memory archival failed..."

# 3. 整合成功后清空 session
session.clear()
self.sessions.save(session)
self.sessions.invalidate(session.key)
```

**特点**：
- `await` 同步阻塞 — 确保整合完成后才清空
- 使用临时 Session 快照 — 避免积整合期间新消息干扰
- 失败不清空 — 容错设计

## 3. save_memory 工具定义

```json
{
  "type": "function",
  "function": {
    "name": "save_memory",
    "description": "Save the memory consolidation result to persistent storage.",
    "parameters": {
      "type": "object",
      "properties": {
        "history_entry": {
          "type": "string",
          "description": "A paragraph (2-5 sentences) summarizing key events/decisions/topics. Start with [YYYY-MM-DD HH:MM]. Include detail useful for grep search."
        },
        "memory_update": {
          "type": "string",
          "description": "Full updated long-term memory as markdown. Include all existing facts plus new ones. Return unchanged if nothing new."
        }
      },
      "required": ["history_entry", "memory_update"]
    }
  }
}
```

**关键设计**：
- 这是**内部专用工具**，不暴露给 Agent 的正常对话
- `history_entry` 要求以 `[YYYY-MM-DD HH:MM]` 开头，便于 grep 搜索
- `memory_update` 是**完整的更新后记忆**（不是增量），LLM 需要返回所有现有事实 + 新事实

## 4. Consolidation Prompt 构建

```python
prompt = f"""Process this conversation and call the save_memory tool with your consolidation.

## Current Long-term Memory
{current_memory or "(empty)"}

## Conversation to Process
{formatted_messages}
"""
```

**发送给 LLM 的消息结构**：

```mermaid
graph TD
    A["system: You are a memory consolidation agent.<br/>Call the save_memory tool with your consolidation."]
    B["user: Process this conversation...<br/>## Current Long-term Memory<br/>(MEMORY.md 内容)<br/>## Conversation to Process<br/>(格式化的旧消息)"]
    C["tools: [save_memory]"]

    A --> LLM["LLM"]
    B --> LLM
    C --> LLM
    LLM --> D["tool_call: save_memory(<br/>  history_entry: '[2026-03-05 10:00] ...',<br/>  memory_update: '# Long-term Memory\n...'<br/>)"]
```

### 4.1 消息格式化

每条消息格式化为一行文本：

```
[YYYY-MM-DD HH:MM] ROLE [tools: tool1, tool2]: content
```

示例：
```
[2026-03-05 10:00] USER: Help me debug the API
[2026-03-05 10:01] ASSISTANT [tools: read_file, exec]: I'll check the error logs first...
[2026-03-05 10:02] USER: The error is in the auth module
```

**实现细节**：
- 跳过无 content 的消息
- timestamp 截取前 16 字符 (`[:16]`)
- 附带 `tools_used` 信息

## 5. 整合结果处理

```mermaid
flowchart LR
    A["LLM response"] --> B{"has_tool_calls?"}
    B -->|"否"| C["⚠️ 返回 False"]
    B -->|"是"| D["取第一个 tool_call"]
    D --> E["解析 arguments"]
    E --> F{"arguments 类型?"}
    F -->|"str"| G["json.loads(args)"]
    F -->|"dict"| H["直接使用"]
    G --> I["提取 history_entry"]
    H --> I
    I --> J["append_history(entry)"]
    I --> K["提取 memory_update"]
    K --> L{"与当前内容不同?"}
    L -->|"是"| M["write_long_term(update)"]
    L -->|"否"| N["跳过写入"]
    J --> O["更新 last_consolidated"]
    M --> O
    N --> O
```

**容错处理**：
- LLM 未调用 save_memory → 返回 False，不修改任何文件
- arguments 可能是 str 或 dict → 兼容两种格式
- 异常捕获 → 记录日志，返回 False
- memory_update 没变化 → 不写入，避免无意义 I/O

## 6. 并发控制

```mermaid
graph TD
    subgraph 并发保护机制
        A["_consolidating: set[str]<br/>记录正在整合的 session key"]
        B["_consolidation_locks: WeakValueDict[str, Lock]<br/>每个 session 独立的异步锁"]
        C["_consolidation_tasks: set[Task]<br/>强引用防止 GC"]
        D["_processing_lock: Lock<br/>全局消息处理锁"]
    end

    E["新消息到达"] --> F{"session key in _consolidating?"}
    F -->|"是"| G["跳过整合，正常处理消息"]
    F -->|"否"| H["添加到 _consolidating"]
    H --> I["获取 session lock"]
    I --> J["执行整合"]
    J --> K["从 _consolidating 移除"]
```

**设计要点**：
1. `_consolidating` 集合：快速检查，避免重复触发
2. `_consolidation_locks`：`WeakValueDictionary`，锁用完后自动 GC
3. `_consolidation_tasks`：强引用集合，确保后台任务不被意外回收
4. `_processing_lock`：全局消息处理序列化（一次处理一条消息）

## 7. 配置参数

| 参数 | 默认值 | 影响 |
|------|--------|------|
| `memory_window` | `100` | 未整合消息达到此数触发自动整合 |
| *隐含参数* | `memory_window // 2` | 自动整合时保留的最近消息数（`keep_count`） |

配置方式（`config.yaml`）：
```yaml
agents:
  defaults:
    memoryWindow: 100
```
