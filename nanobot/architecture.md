# 架构设计

## 1. 两层记忆模型

```mermaid
graph TB
    subgraph 记忆层["记忆层 (Persistent)"]
        LTM["MEMORY.md<br/>━━━━━━━━━<br/>长期记忆<br/>用户偏好 / 项目上下文 / 关键事实<br/>━━━━━━━━━<br/>全量注入 system prompt"]
        HTM["HISTORY.md<br/>━━━━━━━━━<br/>历史日志<br/>按时间排列的事件摘要<br/>━━━━━━━━━<br/>不自动加载，Agent 按需 grep"]
    end

    subgraph 会话层["会话层 (Session)"]
        SES["sessions/*.jsonl<br/>━━━━━━━━━<br/>Append-only 消息列表<br/>JSONL 持久化<br/>滑动窗口 → LLM"]
    end

    subgraph 处理层["处理层 (Runtime)"]
        CB["ContextBuilder<br/>构建 system prompt"]
        MS["MemoryStore<br/>记忆读写 + 整合"]
        SM["SessionManager<br/>会话管理"]
        AL["AgentLoop<br/>核心调度"]
    end

    LTM -->|"read_long_term()"| CB
    SES -->|"get_history()"| AL
    CB -->|"build_system_prompt()"| AL
    AL -->|"触发 consolidate()"| MS
    MS -->|"write_long_term()"| LTM
    MS -->|"append_history()"| HTM
    AL -->|"save(session)"| SM
    SM -->|"持久化"| SES
```

## 2. 核心组件

| 组件 | 源文件 | 职责 |
|------|--------|------|
| `MemoryStore` | `nanobot/agent/memory.py` | 记忆文件读写 + LLM 驱动的整合 |
| `ContextBuilder` | `nanobot/agent/context.py` | 将记忆注入 system prompt |
| `Session` | `nanobot/session/manager.py` | 会话数据结构（append-only 消息列表） |
| `SessionManager` | `nanobot/session/manager.py` | 会话生命周期管理（加载/保存/缓存） |
| `AgentLoop` | `nanobot/agent/loop.py` | 核心调度，触发自动整合 |
| `SkillsLoader` | `nanobot/agent/skills.py` | 加载 memory skill 到 Agent 上下文 |
| Memory Skill | `nanobot/skills/memory/SKILL.md` | 教 Agent 如何使用记忆系统 |

## 3. 组件交互时序

### 3.1 正常对话流

```mermaid
sequenceDiagram
    participant User as 用户
    participant AL as AgentLoop
    participant SM as SessionManager
    participant CB as ContextBuilder
    participant MS as MemoryStore
    participant LLM as LLM Provider
    participant FS as Session File (.jsonl)

    User->>AL: 发送消息
    AL->>SM: get_or_create(key)
    SM-->>AL: session
    AL->>SM: session.get_history(memory_window)
    Note right of SM: 只取 last_consolidated 之后的消息<br/>对齐到 user 消息开头
    SM-->>AL: history (滑动窗口)
    AL->>CB: build_messages(history, message)
    CB->>MS: get_memory_context()
    MS-->>CB: MEMORY.md 全文
    CB-->>AL: messages (system prompt + 记忆 + history + 当前消息)
    AL->>LLM: chat(messages, tools)
    loop 工具调用循环 (max 40 次)
        LLM-->>AL: tool_calls
        AL->>AL: execute tools
        AL->>LLM: tool results
    end
    LLM-->>AL: final response
    AL->>AL: _save_turn() 追加新消息到 session
    AL->>SM: save(session)
    SM->>FS: 写入 JSONL
    alt 未整合消息数 >= memory_window
        AL->>AL: asyncio.create_task(consolidate)
        Note right of AL: 后台异步整合，不阻塞响应
    end
    AL-->>User: 返回响应
```

### 3.2 记忆整合流

```mermaid
sequenceDiagram
    participant AL as AgentLoop
    participant MS as MemoryStore
    participant LLM as LLM Provider
    participant MF as MEMORY.md
    participant HF as HISTORY.md

    AL->>MS: consolidate(session, provider, model)
    MS->>MS: 确定待整合消息范围
    MS->>MS: 格式化消息为文本行
    MS->>MF: read_long_term()
    MF-->>MS: 当前记忆内容
    MS->>MS: 构建 consolidation prompt
    MS->>LLM: chat(system + prompt, tools=[save_memory])
    LLM-->>MS: save_memory(history_entry, memory_update)
    MS->>HF: append_history(history_entry)
    alt memory_update 与当前内容不同
        MS->>MF: write_long_term(memory_update)
    end
    MS->>MS: 更新 session.last_consolidated
```

### 3.3 `/new` 命令流

```mermaid
sequenceDiagram
    participant User as 用户
    participant AL as AgentLoop
    participant MS as MemoryStore
    participant SM as SessionManager

    User->>AL: /new
    AL->>AL: 获取 consolidation lock
    AL->>MS: consolidate(snapshot, archive_all=True)
    Note right of MS: 同步等待完成
    alt 整合成功
        AL->>SM: session.clear()
        AL->>SM: save(session)
        AL->>SM: invalidate(session.key)
        AL-->>User: "New session started."
    else 整合失败
        AL-->>User: "Memory archival failed..."
    end
```

## 4. 目录结构

```
workspace/                          # 用户工作区
├── memory/
│   ├── MEMORY.md                   # 长期记忆（LLM 维护 + Agent 主动写入）
│   └── HISTORY.md                  # 历史日志（追加式，grep 可搜索）
├── sessions/
│   ├── telegram_12345.jsonl        # 会话持久化文件
│   └── dingtalk_67890.jsonl
├── skills/
│   └── memory/
│       └── SKILL.md                # 用户可覆盖的记忆技能
└── AGENTS.md, SOUL.md, ...         # Bootstrap 文件

nanobot/                            # 源码
├── agent/
│   ├── memory.py                   # MemoryStore 核心
│   ├── context.py                  # ContextBuilder
│   ├── loop.py                     # AgentLoop（触发整合逻辑）
│   └── skills.py                   # SkillsLoader
├── session/
│   └── manager.py                  # Session + SessionManager
├── templates/
│   └── memory/
│       └── MEMORY.md               # 初始模板
├── skills/
│   └── memory/
│       └── SKILL.md                # 内置记忆技能
├── config/
│   └── schema.py                   # memory_window 等配置
└── utils/
    └── helpers.py                  # sync_workspace_templates()
```

## 5. 设计原则

| 原则 | 说明 |
|------|------|
| **LLM 驱动整合** | 不做简单截断，由 LLM 提取关键信息，确保重要内容不丢失 |
| **双层分离** | 高频事实 (MEMORY.md) 全量加载；低频历史 (HISTORY.md) 按需 grep |
| **Append-only** | 消息只追加不删除，利于 LLM prompt cache |
| **Agent 自治** | Agent 既能被动依赖自动整合，也能主动读写记忆文件 |
| **Skill 驱动** | 通过 always-on skill 教 Agent 使用记忆，而非硬编码行为 |
| **容错优先** | 整合失败不影响正常对话，不清空 session |
