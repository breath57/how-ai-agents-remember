# 逐文件实现细节

> 含关键代码片段和设计意图注释，供复刻时参考。

## 1. MemoryStore — `nanobot/agent/memory.py`

**职责**：记忆文件的 CRUD + LLM 驱动整合

### 1.1 类结构

```python
class MemoryStore:
    def __init__(self, workspace: Path):
        self.memory_dir = ensure_dir(workspace / "memory")
        self.memory_file = self.memory_dir / "MEMORY.md"
        self.history_file = self.memory_dir / "HISTORY.md"
```

### 1.2 基础读写

```python
def read_long_term(self) -> str:
    """读取 MEMORY.md 全文，文件不存在则返回空字符串"""
    if self.memory_file.exists():
        return self.memory_file.read_text(encoding="utf-8")
    return ""

def write_long_term(self, content: str) -> None:
    """覆盖写入 MEMORY.md"""
    self.memory_file.write_text(content, encoding="utf-8")

def append_history(self, entry: str) -> None:
    """追加一条记录到 HISTORY.md（两个换行分隔）"""
    with open(self.history_file, "a", encoding="utf-8") as f:
        f.write(entry.rstrip() + "\n\n")

def get_memory_context(self) -> str:
    """返回格式化的长期记忆文本，用于注入 prompt"""
    long_term = self.read_long_term()
    return f"## Long-term Memory\n{long_term}" if long_term else ""
```

### 1.3 consolidate() 核心方法

```python
async def consolidate(
    self, session, provider, model, *,
    archive_all: bool = False,
    memory_window: int = 50,
) -> bool:
    """整合旧消息到 MEMORY.md + HISTORY.md，返回 True/False"""
    
    # ===== 阶段 1：确定待整合消息范围 =====
    if archive_all:
        old_messages = session.messages
        keep_count = 0
    else:
        keep_count = memory_window // 2
        # 边界检查：消息太少则跳过
        if len(session.messages) <= keep_count:
            return True
        if len(session.messages) - session.last_consolidated <= 0:
            return True
        old_messages = session.messages[session.last_consolidated:-keep_count]
        if not old_messages:
            return True

    # ===== 阶段 2：格式化消息 =====
    lines = []
    for m in old_messages:
        if not m.get("content"):
            continue  # 跳过空消息
        tools = f" [tools: {', '.join(m['tools_used'])}]" if m.get("tools_used") else ""
        lines.append(f"[{m.get('timestamp', '?')[:16]}] {m['role'].upper()}{tools}: {m['content']}")

    # ===== 阶段 3：构建 prompt =====
    current_memory = self.read_long_term()
    prompt = f"""Process this conversation and call the save_memory tool...
## Current Long-term Memory
{current_memory or "(empty)"}
## Conversation to Process
{chr(10).join(lines)}"""

    # ===== 阶段 4：调用 LLM =====
    response = await provider.chat(
        messages=[
            {"role": "system", "content": "You are a memory consolidation agent..."},
            {"role": "user", "content": prompt},
        ],
        tools=_SAVE_MEMORY_TOOL,
        model=model,
    )

    # ===== 阶段 5：处理响应 =====
    if not response.has_tool_calls:
        return False  # LLM 未调用工具

    args = response.tool_calls[0].arguments
    if isinstance(args, str):
        args = json.loads(args)  # 兼容字符串格式

    if entry := args.get("history_entry"):
        self.append_history(entry)
    if update := args.get("memory_update"):
        if update != current_memory:
            self.write_long_term(update)

    # ===== 阶段 6：更新指针 =====
    session.last_consolidated = 0 if archive_all else len(session.messages) - keep_count
    return True
```

**设计意图**：
- `keep_count = memory_window // 2` — 保留一半窗口的消息不整合，确保上下文连续
- `archive_all` 时 `last_consolidated = 0` — 配合 `/new` 的 `session.clear()`
- `update != current_memory` 检查 — 避免无意义写入
- 异常全部 catch — 整合失败不影响主流程

---

## 2. ContextBuilder — `nanobot/agent/context.py`

**职责**：组装 system prompt + 完整消息列表

### 2.1 类结构

```python
class ContextBuilder:
    BOOTSTRAP_FILES = ["AGENTS.md", "SOUL.md", "USER.md", "TOOLS.md", "IDENTITY.md"]
    
    def __init__(self, workspace: Path):
        self.workspace = workspace
        self.memory = MemoryStore(workspace)  # 内嵌 MemoryStore
        self.skills = SkillsLoader(workspace)  # 内嵌 SkillsLoader
```

### 2.2 build_system_prompt()

```python
def build_system_prompt(self, skill_names=None) -> str:
    parts = [self._get_identity()]            # 1. 身份 + workspace 路径
    
    bootstrap = self._load_bootstrap_files()  # 2. Bootstrap 文件
    if bootstrap:
        parts.append(bootstrap)
    
    memory = self.memory.get_memory_context()  # 3. ★ 记忆
    if memory:
        parts.append(f"# Memory\n\n{memory}")
    
    always_skills = self.skills.get_always_skills()  # 4. Always-on 技能
    if always_skills:
        parts.append(f"# Active Skills\n\n{...}")
    
    skills_summary = self.skills.build_skills_summary()  # 5. 技能目录
    if skills_summary:
        parts.append(f"# Skills\n\n{skills_summary}")
    
    return "\n\n---\n\n".join(parts)
```

### 2.3 build_messages()

```python
def build_messages(self, history, current_message, media=None, channel=None, chat_id=None):
    runtime_ctx = self._build_runtime_context(channel, chat_id)
    user_content = self._build_user_content(current_message, media)
    
    # 关键：合并 runtime_ctx 和 user_content 到一个 user message
    # 避免连续 user role 消息（某些 provider 会拒绝）
    merged = f"{runtime_ctx}\n\n{user_content}"
    
    return [
        {"role": "system", "content": self.build_system_prompt()},
        *history,
        {"role": "user", "content": merged},
    ]
```

---

## 3. Session + SessionManager — `nanobot/session/manager.py`

### 3.1 Session 数据类

```python
@dataclass
class Session:
    key: str                        # "channel:chat_id"
    messages: list[dict] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
    updated_at: datetime = field(default_factory=datetime.now)
    metadata: dict = field(default_factory=dict)
    last_consolidated: int = 0      # 已整合到记忆文件的消息数
```

### 3.2 get_history() — 滑动窗口

```python
def get_history(self, max_messages=500) -> list[dict]:
    unconsolidated = self.messages[self.last_consolidated:]  # 跳过已整合区
    sliced = unconsolidated[-max_messages:]                  # 取尾部
    
    # 对齐到 user 消息开头
    for i, m in enumerate(sliced):
        if m.get("role") == "user":
            sliced = sliced[i:]
            break
    
    # 清理：只保留标准字段
    out = []
    for m in sliced:
        entry = {"role": m["role"], "content": m.get("content", "")}
        for k in ("tool_calls", "tool_call_id", "name"):
            if k in m:
                entry[k] = m[k]
        out.append(entry)
    return out
```

### 3.3 SessionManager JSONL 持久化

```python
class SessionManager:
    def save(self, session):
        path = self._get_session_path(session.key)
        with open(path, "w") as f:
            # 第一行：metadata（含 last_consolidated）
            metadata_line = {
                "_type": "metadata",
                "key": session.key,
                "created_at": session.created_at.isoformat(),
                "last_consolidated": session.last_consolidated,
                ...
            }
            f.write(json.dumps(metadata_line) + "\n")
            # 后续行：每条消息一行
            for msg in session.messages:
                f.write(json.dumps(msg) + "\n")
    
    def _load(self, key):
        # 逐行解析，第一行为 metadata，后续为 messages
        # 支持从 legacy 路径迁移
        ...
```

---

## 4. AgentLoop — `nanobot/agent/loop.py`

### 4.1 记忆相关的属性

```python
class AgentLoop:
    def __init__(self, ...):
        self.memory_window = memory_window  # 默认 100
        self.context = ContextBuilder(workspace)
        self.sessions = SessionManager(workspace)
        
        # 并发控制
        self._consolidating: set[str] = set()
        self._consolidation_tasks: set[asyncio.Task] = set()
        self._consolidation_locks: weakref.WeakValueDictionary = weakref.WeakValueDictionary()
```

### 4.2 _save_turn() — 消息持久化

```python
def _save_turn(self, session, messages, skip):
    """保存新一轮对话的消息到 session"""
    for m in messages[skip:]:
        entry = dict(m)
        role, content = entry.get("role"), entry.get("content")
        
        # 跳过空的 assistant 消息（避免污染上下文）
        if role == "assistant" and not content and not entry.get("tool_calls"):
            continue
        
        # 截断过长的 tool result（最大 500 字符）
        if role == "tool" and isinstance(content, str) and len(content) > 500:
            entry["content"] = content[:500] + "\n... (truncated)"
        
        # 剥离 runtime context 前缀
        if role == "user" and isinstance(content, str):
            if content.startswith(ContextBuilder._RUNTIME_CONTEXT_TAG):
                parts = content.split("\n\n", 1)
                entry["content"] = parts[1] if len(parts) > 1 else continue
        
        # 替换 base64 图片为 [image] 标记
        if role == "user" and isinstance(content, list):
            # ... 图片内容替换逻辑
        
        entry.setdefault("timestamp", datetime.now().isoformat())
        session.messages.append(entry)
```

**设计意图**：
- 截断 tool result — 避免 session 文件膨胀
- 剥离 runtime context — 不保存运行时上下文（每次重新生成）
- base64 图片替换 — 不在 session 中存储大体积图片

### 4.3 _consolidate_memory() — 委托方法

```python
async def _consolidate_memory(self, session, archive_all=False):
    """创建新的 MemoryStore 实例执行整合"""
    return await MemoryStore(self.workspace).consolidate(
        session, self.provider, self.model,
        archive_all=archive_all, memory_window=self.memory_window,
    )
```

**注意**：每次整合都新建 `MemoryStore` 实例，避免状态污染。

---

## 5. SkillsLoader — `nanobot/agent/skills.py`

### 5.1 记忆相关的方法

```python
def get_always_skills(self) -> list[str]:
    """获取 always=true 的技能列表（memory skill 在此列表中）"""
    result = []
    for s in self.list_skills():
        meta = self.get_skill_metadata(s["name"])
        skill_meta = self._parse_nanobot_metadata(meta.get("metadata", ""))
        if skill_meta.get("always") or meta.get("always"):
            result.append(s["name"])
    return result
```

### 5.2 技能优先级

```
workspace/skills/ > builtin skills (nanobot/skills/)
```

用户可在 workspace 中覆盖内置 memory skill。

---

## 6. Memory Skill — `nanobot/skills/memory/SKILL.md`

```yaml
---
name: memory
description: Two-layer memory system with grep-based recall.
always: true    # ← 关键：始终加载
---
```

内容教 Agent：
1. MEMORY.md 是什么、在哪里、如何更新
2. HISTORY.md 是什么、如何用 grep 搜索
3. 什么时候应该主动更新 MEMORY.md
4. 不需要手动管理自动整合

---

## 7. 模板同步 — `nanobot/utils/helpers.py`

```python
def sync_workspace_templates(workspace: Path):
    """首次启动时同步模板文件到 workspace"""
    tpl = pkg_files("nanobot") / "templates"
    
    # 创建 memory/MEMORY.md（从模板）
    _write(tpl / "memory" / "MEMORY.md", workspace / "memory" / "MEMORY.md")
    # 创建 memory/HISTORY.md（空文件）
    _write(None, workspace / "memory" / "HISTORY.md")
    # 创建 skills/ 目录
    (workspace / "skills").mkdir(exist_ok=True)
    
    # ★ 只创建不存在的文件，不覆盖已有内容
```

---

## 8. 配置 — `nanobot/config/schema.py`

```python
class AgentDefaults(Base):
    memory_window: int = 100  # 唯一的记忆相关配置项
```

对应 YAML：
```yaml
agents:
  defaults:
    memoryWindow: 100
```
