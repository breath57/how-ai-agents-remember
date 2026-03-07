# 记忆注入 — 如何进入 LLM Prompt

## 1. System Prompt 组装顺序

```mermaid
flowchart LR
    A["1. Identity<br/>━━━━━━━<br/>运行时信息<br/>workspace 路径<br/>记忆文件位置<br/>行为准则"] --> B["2. Bootstrap Files<br/>━━━━━━━<br/>AGENTS.md<br/>SOUL.md<br/>USER.md<br/>TOOLS.md<br/>IDENTITY.md"]
    B --> C["3. ★ Memory 区块<br/>━━━━━━━<br/>MEMORY.md 全文<br/>（get_memory_context）"]
    C --> D["4. Always-on Skills<br/>━━━━━━━<br/>memory skill 等<br/>always: true 的技能"]
    D --> E["5. Skills 目录<br/>━━━━━━━<br/>所有技能的摘要<br/>XML 格式"]

    style C fill:#f9f,stroke:#333,stroke-width:2px
```

各部分之间用 `\n\n---\n\n` 分隔。

## 2. 各部分详解

### 2.1 Identity 区块

告知 Agent 记忆文件位置，使其能主动操作：

```markdown
## Workspace
Your workspace is at: /home/user/.nanobot/workspace
- Long-term memory: /home/user/.nanobot/workspace/memory/MEMORY.md (write important facts here)
- History log: /home/user/.nanobot/workspace/memory/HISTORY.md (grep-searchable). Each entry starts with [YYYY-MM-DD HH:MM].
- Custom skills: /home/user/.nanobot/workspace/skills/{skill-name}/SKILL.md
```

### 2.2 Memory 区块

```python
def get_memory_context(self) -> str:
    long_term = self.read_long_term()  # 读取 MEMORY.md
    return f"## Long-term Memory\n{long_term}" if long_term else ""
```

在 `build_system_prompt()` 中：
```python
memory = self.memory.get_memory_context()
if memory:
    parts.append(f"# Memory\n\n{memory}")
```

**注入效果示例**：
```markdown
# Memory

## Long-term Memory
# Long-term Memory

## User Information
- Name: Alice
- Role: Backend developer
- Timezone: UTC+8

## Preferences
- Prefers dark mode
- Uses YAML over JSON
- Coding style: PEP 8 strict

## Project Context
- Working on API gateway, uses OAuth2
- Database: PostgreSQL 15
```

### 2.3 Memory Skill（Always-on）

Memory skill 标记为 `always: true`，始终加载到 Agent 上下文：

```markdown
# Memory

## Structure
- `memory/MEMORY.md` — Long-term facts. Always loaded into your context.
- `memory/HISTORY.md` — Append-only event log. NOT loaded. Search with grep.

## Search Past Events
```bash
grep -i "keyword" memory/HISTORY.md
```

## When to Update MEMORY.md
Write important facts immediately using `edit_file` or `write_file`:
- User preferences ("I prefer dark mode")
- Project context ("The API uses OAuth2")
- Relationships ("Alice is the project lead")

## Auto-consolidation
Old conversations are automatically summarized. You don't need to manage this.
```

### 2.4 运行时上下文

每条用户消息前会注入一个运行时上下文块（**在 user message 中，不在 system prompt 中**）：

```
[Runtime Context — metadata only, not instructions]
Current Time: 2026-03-05 10:00 (Wednesday) (CST)
Channel: telegram
Chat ID: 12345
```

## 3. 完整消息结构

```mermaid
graph TD
    subgraph "发送给 LLM 的消息列表"
        S["system: {system_prompt}<br/>━━━━━<br/>Identity + Bootstrap + ★Memory + Skills"]
        H1["user: 历史消息 1"]
        H2["assistant: 历史回复 1"]
        H3["user: 历史消息 2"]
        H4["assistant: 历史回复 2"]
        HN["...更多历史..."]
        U["user: [Runtime Context]<br/>━━━━━<br/>用户当前消息"]
    end
    S --> H1 --> H2 --> H3 --> H4 --> HN --> U
```

```python
def build_messages(self, history, current_message, ...):
    runtime_ctx = self._build_runtime_context(channel, chat_id)
    user_content = self._build_user_content(current_message, media)
    
    # 合并运行时上下文和用户消息到一个 user message
    # 避免连续同 role 消息被某些 provider 拒绝
    merged = f"{runtime_ctx}\n\n{user_content}"
    
    return [
        {"role": "system", "content": self.build_system_prompt()},
        *history,  # 来自 session.get_history()
        {"role": "user", "content": merged},
    ]
```

## 4. Agent 主动记忆操作

除了被动注入，Agent 还可以**主动**操作记忆文件：

```mermaid
graph TD
    A["Agent 判断需要记住信息"] --> B{"什么类型的信息?"}
    B -->|"长期事实/偏好"| C["write_file / edit_file<br/>→ memory/MEMORY.md"]
    B -->|"回忆历史事件"| D["exec: grep -i 'keyword'<br/>→ memory/HISTORY.md"]
    B -->|"复杂历史搜索"| E["exec: grep -iE 'pattern1|pattern2'<br/>→ memory/HISTORY.md"]
```

**Agent 主动写入场景**：
- 用户明确表达偏好：*"I prefer dark mode"*
- 项目关键信息：*"The API uses OAuth2 with Google provider"*
- 重要关系：*"Alice is the tech lead, Bob handles DevOps"*

**Agent 搜索历史场景**：
- 用户问 *"When did we discuss the deadline?"*
- Agent 执行 `grep -i "deadline" memory/HISTORY.md`
- 返回相关历史记录

## 5. 子代理与记忆的关系

```mermaid
graph TD
    subgraph 主Agent
        CB["ContextBuilder<br/>（有 MemoryStore）"]
        MS["MemoryStore"]
        CB --> MS
    end
    
    subgraph 子代理
        SCB["无 ContextBuilder"]
        SMS["无 MemoryStore"]
        FT["有文件系统工具<br/>（理论上可读写 MEMORY.md）"]
    end
    
    主Agent -->|"spawn"| 子代理
    
    style SMS fill:#fcc,stroke:#333
    style SCB fill:#fcc,stroke:#333
```

**设计决策**：子代理**不继承**记忆上下文。
- 子代理没有 MemoryStore 或 ContextBuilder
- 子代理有文件工具，理论上可读写 MEMORY.md，但不会自动加载
- 子代理的对话不触发记忆整合
- 原因：子代理是短期任务执行者，不需要完整记忆上下文
