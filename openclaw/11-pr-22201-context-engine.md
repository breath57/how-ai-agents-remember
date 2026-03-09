# 11 - PR #22201 完整代码解剖（超详细版）：25 个 Commit、44 个文件逐层拆解 —— OpenClaw 如何用一个 PR 把上下文管理做成可插拔的

> PR：[feature(context): extend plugin system to support custom context management #22201](https://github.com/openclaw/openclaw/pull/22201)
> 作者：@jalehman（Josh Lehman）
> 合并：2026-03-06，`feature/context-engine` → `main`
> 规模：25 commits，44 files changed，+2,308 / -103
> 标签：agents, extensions: lobster, extensions: phone-control, gateway, maintainer, size: XL

---

## 这篇文档讲什么？

[10-context-engine.md](10-context-engine.md) 从"演变故事"角度讲了 ContextEngine 是什么、为什么出现。这篇文档不一样——**完全围绕 PR #22201 的实际代码改动**，逐个 commit、逐个文件组地分析：改了什么、为什么这么改、设计决策是什么。

适合想深入理解实现细节、或者想在自己项目里做类似插件化改造的读者。

---

## PR 的动机：一句话版

> **Problem**: OpenClaw's context management (compaction, assembly, etc.) is hardcoded in core, making it impossible for plugins to provide alternative context strategies.
>
> **Why it matters**: The Lossless Context Management paper from Ehrlich and Blackman demonstrated a promising alternative direction for context management. I have implemented that approach for OpenClaw (with a number of improvements), have been running it for the last week, and to say it works well would be an understatement. That work is in the lossless-claw repository.

简单说：作者先在外部项目 `lossless-claw` 里验证了"无损上下文管理"方案，效果很好，但发现没有办法以插件方式集成进 OpenClaw——因为核心代码全是硬编码的。于是他做了这个 PR：**不改任何现有逻辑，只是在外面加一层可插拔的接口**。

---

## 改动全景图

44 个文件，按职责分成 **5 个改动组**：

```
PR #22201 改动分组（44 files, +2,308 / -103）
│
├── ① 核心接口层（新增 5 文件，+561）
│   ├── src/context-engine/types.ts        — 167 行，ContextEngine 接口定义
│   ├── src/context-engine/registry.ts     —  67 行，注册表（注册/解析引擎）
│   ├── src/context-engine/legacy.ts       — 115 行，LegacyContextEngine 向后兼容包装
│   ├── src/context-engine/init.ts         —  23 行，初始化守卫
│   └── src/context-engine/index.ts        —  19 行，barrel exports
│
├── ② Agent 运行时集成（修改 3 文件 + 新增 1 文件，+349）
│   ├── src/agents/pi-embedded-runner/run.ts       — +65 行，主运行循环
│   ├── src/agents/pi-embedded-runner/run/attempt.ts — +149 行，单次尝试
│   ├── src/agents/pi-embedded-runner/compact.ts   — +54 行，/compact 命令路由
│   └── src/agents/pi-embedded-runner/run/types.ts — +9 行，类型扩展
│
├── ③ 插件系统扩展（修改/新增 10+ 文件，+500+）
│   ├── src/plugins/types.ts               — PluginKind 增加 "context-engine"
│   ├── src/plugins/slots.ts               — 新增 contextEngine slot
│   ├── src/plugins/registry.ts            — registerContextEngine 注册
│   ├── src/plugins/loader.ts              — 传递 runtimeOptions
│   ├── src/plugins/runtime/types.ts       — +55 行，subagent 运行时类型
│   ├── src/plugins/runtime/index.ts       — subagent 运行时注入
│   ├── src/plugins/runtime/gateway-request-scope.ts — +46 行，AsyncLocalStorage 请求作用域
│   ├── src/plugin-sdk/index.ts            — +29 行，SDK 导出 ContextEngine 类型
│   └── src/config/types.plugins.ts        — +2 行，配置类型
│
├── ④ 子 Agent 生命周期（修改 2 文件，+70）
│   ├── src/agents/subagent-registry.ts    — +45 行，4 种结束回调
│   └── src/agents/pi-settings.ts          — +25 行，auto-compaction 守卫
│
└── ⑤ Gateway 重构 + 子 Agent 调度（修改 3 文件，+274）
    ├── src/gateway/server-plugins.ts      — +164 行，subagent 调度运行时
    ├── src/gateway/server.impl.ts         — +111/-50 行，提取 context 为共享变量
    └── src/gateway/server-methods.ts      — +22 行，sessions.get 路由

其余：测试文件（~600+ 行）+ 配置/格式化调整
```

---

## ① 核心接口层：5 个新文件做了什么

### types.ts — 167 行定义一切

这是整个 PR 的核心。定义了 `ContextEngine` 接口的完整签名：

```typescript
interface ContextEngine {
  readonly info: ContextEngineInfo;

  // 3 个必选：管理上下文的三件核心事
  ingest(params):   Promise<IngestResult>;     // 消息进来怎么存
  assemble(params): Promise<AssembleResult>;    // 送给模型时怎么组
  compact(params):  Promise<CompactResult>;     // 太多了怎么压

  // 5 个可选：增强能力
  bootstrap?(params):            Promise<BootstrapResult>;
  ingestBatch?(params):          Promise<IngestBatchResult>;
  afterTurn?(params):            Promise<void>;
  prepareSubagentSpawn?(params): Promise<SubagentSpawnPreparation | undefined>;
  onSubagentEnded?(params):      Promise<void>;
  dispose?():                    Promise<void>;
}
```

**设计决策 #1：必选 vs 可选的分界**

只有 `ingest`、`assemble`、`compact` 是必选的——这三个是上下文管理不可回避的三件事。其余全部可选，用 `?` 标记。这意味着写一个最简引擎只需要实现 3 个方法。

**设计决策 #2：`ownsCompaction` 标志**

```typescript
type ContextEngineInfo = {
  id: string;
  name: string;
  version?: string;
  ownsCompaction?: boolean;  // ← 关键
};
```

当引擎声明 `ownsCompaction: true` 时，运行时会**禁用 SDK 内置的自动压缩**（通过 `pi-settings.ts` 的 `applyPiAutoCompactionGuard`），防止双重压缩。这是因为像 lossless-claw 这样的引擎有自己的压缩策略，不需要 SDK 再插一脚。

**设计决策 #3：`legacyParams` 透传袋**

compact 方法的参数里有一个 `legacyParams?: Record<string, unknown>`——这是给 `LegacyContextEngine` 用的"不透明参数包"。旧的压缩函数需要大量运行时参数（sessionKey、provider、agentDir 等），新接口不想把这些全塞进签名里污染接口，所以用一个 `Record<string, unknown>` 打包传递。

### registry.ts — 67 行的全局单例注册表

```typescript
const CONTEXT_ENGINE_REGISTRY_STATE = Symbol.for("openclaw.contextEngineRegistryState");

// 注册引擎工厂
registerContextEngine(id: string, factory: ContextEngineFactory): void
// 按 id 查找
getContextEngineFactory(id: string): ContextEngineFactory | undefined
// 列出所有已注册 id
listContextEngineIds(): string[]
// 按配置解析（核心函数）
resolveContextEngine(config?): Promise<ContextEngine>
```

**设计决策 #4：`Symbol.for()` 而非模块变量**

为什么不直接用模块级 `Map`？因为 OpenClaw 是 monorepo 构建，打包器可能产生多份文件拷贝，每份都有自己的模块作用域。`Symbol.for()` 挂在 `globalThis` 上，保证不管加载了几份代码，注册表只有一个。（这个 bug 后来在 v2026.3.8 #40115 中也专门修复过。）

**`resolveContextEngine` 的解析逻辑**：
1. 读 `config.plugins.slots.contextEngine`
2. 没配？默认 `"legacy"`
3. 从注册表查工厂函数，调用创建引擎实例
4. 查不到？抛错，附带所有已注册引擎列表（"Available engines: legacy, ..."），方便排查

### legacy.ts — 115 行的"什么都不做"包装

这是 **Strangler Fig 模式** 的核心：

```typescript
class LegacyContextEngine implements ContextEngine {
  info = { id: "legacy", name: "Legacy Context Engine", version: "1.0.0" };

  async ingest()   { return { ingested: false }; }           // 不管消息摄入
  async assemble() { return { messages: params.messages }; } // 原样返回
  async afterTurn() {}                                        // 空操作
  async compact()  { /* 委托给原来的函数 */ }
  async dispose()  {}                                         // 空操作
}
```

**关键细节**：compact 方法内部做了动态 import：

```typescript
async compact(params) {
  const { compactEmbeddedPiSessionDirect } =
    await import("../agents/pi-embedded-runner/compact.js");
  // 把 legacyParams 展开，调用原来的压缩函数
  const result = await compactEmbeddedPiSessionDirect({
    ...params.legacyParams,
    sessionId: params.sessionId,
    // ...
  });
}
```

动态 import 是为了**避免循环依赖**——legacy.ts 在 `context-engine/` 目录下，但要调用 `pi-embedded-runner/compact.ts` 里的函数，静态 import 会造成循环。

### init.ts — 23 行的"只跑一次"守卫

```typescript
let initialized = false;

export function ensureContextEnginesInitialized(): void {
  if (initialized) return;
  initialized = true;
  registerLegacyContextEngine(); // 确保 "legacy" 引擎始终在注册表里
}
```

为什么需要这个？因为 ContextEngine 的解析发生在多个入口点（run.ts、compact.ts、subagent-registry.ts），如果没人注册 legacy 引擎就解析不到。这个守卫确保无论哪个路径先执行，legacy 总是可用的。

---

## ② Agent 运行时集成：怎么把接口插进去的

这是改动量最大的部分——在不破坏任何现有行为的前提下，把 ContextEngine 的生命周期钩子嵌入到 Agent 运行循环的关键位置。

### run.ts：+65 行，运行循环入口

改了什么：

```typescript
// 在 try 块之前，解析一次引擎，重试时复用
ensureContextEnginesInitialized();
const contextEngine = await resolveContextEngine(params.config);

try {
  // ... 重试循环 ...
  // 把 contextEngine 传给每次 attempt
  await runEmbeddedAttempt({
    ...params,
    contextEngine,            // ← 新增
    contextTokenBudget: ctxInfo.tokens,  // ← 新增
  });
} finally {
  await contextEngine.dispose?.();  // ← 资源释放
}
```

**设计决策 #5：引擎实例跨重试复用**

引擎在 `runEmbeddedPiAgent` 开头解析一次，所有重试共享同一个实例。这避免了每次重试都重新初始化（比如连接向量数据库）的开销。

**overflow compaction 改造**：原来 overflow 时直接调 `compactEmbeddedPiSessionDirect()`，现在改为通过 `contextEngine.compact()`：

```typescript
// 旧：直接调函数
const compactResult = await compactEmbeddedPiSessionDirect({
  sessionId, sessionKey, messageChannel, ... // 20+ 个参数
});

// 新：通过引擎接口
const compactResult = await contextEngine.compact({
  sessionId,
  sessionFile,
  tokenBudget: ctxInfo.tokens,
  force: true,
  compactionTarget: "budget",
  legacyParams: { /* 原来那 20+ 个参数打包在这里 */ },
});
```

### attempt.ts：+149 行，单次尝试中的 4 个钩子

**钩子 ①：bootstrap（会话开始时）**

```typescript
if (hadSessionFile && params.contextEngine?.bootstrap) {
  try {
    await params.contextEngine.bootstrap({
      sessionId: params.sessionId,
      sessionFile: params.sessionFile,
    });
  } catch (bootstrapErr) {
    log.warn(`context engine bootstrap failed: ${String(bootstrapErr)}`);
  }
}
```

在会话文件已存在时调用。用途：引擎可以从 JSONL session 文件中加载历史消息到自己的存储（比如向量数据库）。bootstrap 失败不阻断——仅 warn 日志。

**钩子 ②：assemble（构建模型上下文时）**

```typescript
if (params.contextEngine) {
  const assembled = await params.contextEngine.assemble({
    sessionId,
    messages: activeSession.messages,
    tokenBudget: params.contextTokenBudget,
  });
  if (assembled.messages !== activeSession.messages) {
    activeSession.agent.replaceMessages(assembled.messages);
  }
  if (assembled.systemPromptAddition) {
    systemPromptText = prependSystemPromptAddition({
      systemPrompt: systemPromptText,
      systemPromptAddition: assembled.systemPromptAddition,
    });
  }
}
```

注意三个细节：
- 用**引用比较**（`!==`）判断引擎是否替换了消息——如果引擎原样返回（如 legacy），不做多余操作
- 引擎可以通过 `systemPromptAddition` 往系统提示中**注入额外内容**（比如上下文摘要）
- assemble 在旧管线（sanitize→validate→limit→repair）**之后**执行——引擎看到的是已清洗过的消息

**钩子 ③：afterTurn（一轮对话结束后）**

```typescript
if (typeof params.contextEngine.afterTurn === "function") {
  await params.contextEngine.afterTurn({
    sessionId,
    sessionFile: params.sessionFile,
    messages: messagesSnapshot,
    prePromptMessageCount,
    tokenBudget: params.contextTokenBudget,
    legacyCompactionParams: afterTurnLegacyCompactionParams,
  });
} else {
  // Fallback：逐条 ingest 新消息
  const newMessages = messagesSnapshot.slice(prePromptMessageCount);
  if (params.contextEngine.ingestBatch) {
    await params.contextEngine.ingestBatch({ sessionId, messages: newMessages });
  } else {
    for (const msg of newMessages) {
      await params.contextEngine.ingest({ sessionId, message: msg });
    }
  }
}
```

**设计决策 #6：afterTurn 优先、ingest 兜底**

如果引擎实现了 `afterTurn`，由引擎全权决定回合后做什么（压缩？入库？什么都不做？）。如果没有实现 `afterTurn`，则退化为逐条调 `ingest` / `ingestBatch`。这给了引擎最大的灵活性。

**设计决策 #7：prePromptMessageCount 切片**

通过 `prePromptMessageCount` 记录发送给模型之前的消息数量，回合结束后用 `messages.slice(prePromptMessageCount)` 算出本轮新产生的消息。这个值也传给 afterTurn，让引擎知道哪些是新的。

### compact.ts：+54 行，/compact 命令路由

`/compact` 命令的入口也改为通过 ContextEngine 路由：

```typescript
// 在 compactEmbeddedPiSession 的排队回调中：
ensureContextEnginesInitialized();
const contextEngine = await resolveContextEngine(params.config);
try {
  // 计算 tokenBudget（从模型上下文窗口推导）
  const ceCtxInfo = resolveContextWindowInfo({ ... });
  const result = await contextEngine.compact({
    sessionId: params.sessionId,
    sessionFile: params.sessionFile,
    tokenBudget: ceCtxInfo.tokens,
    force: params.trigger === "manual",  // 手动 /compact → force=true
    legacyParams: params as Record<string, unknown>,
  });
  return result;
} finally {
  await contextEngine.dispose?.();
}
```

注意这里**每次 /compact 都新建一个引擎实例**（不同于 run.ts 的复用）。因为 /compact 是独立的命令调用，不在 Agent run 生命周期内。

---

## ③ 插件系统扩展：让外部插件能注册引擎

### 新增 PluginKind 和 Slot

```typescript
// src/plugins/types.ts
type PluginKind = "memory" | "context-engine";  // memory 之外新增一种

// src/plugins/slots.ts
const DEFAULT_SLOT_BY_KEY = {
  memory: "memory-core",
  contextEngine: "legacy",   // 默认用 legacy
};
```

Slot 机制保证**同一个坑位只有一个活跃插件**。如果两个插件都声明 `kind: "context-engine"`，只有配置选中的那个加载。

### registerContextEngine API

```typescript
// src/plugins/registry.ts — 插件注册时调用
registerContextEngine: (id, factory) => registerContextEngine(id, factory),

// src/plugins/types.ts — OpenClawPluginApi 声明
registerContextEngine: (
  id: string,
  factory: ContextEngineFactory,
) => void;
```

插件在 `init()` 中调用 `api.registerContextEngine("my-engine", factoryFn)` 即可注册。

### Plugin SDK 导出

```typescript
// src/plugin-sdk/index.ts — 新增导出
export type { ContextEngine, ContextEngineInfo, AssembleResult, CompactResult, ... };
export { registerContextEngine };
```

外部插件通过 `@openclaw/plugin-sdk` 引入这些类型。

### Subagent 运行时（最大的新增块）

这不是"上下文管理"本身的改动，但 **Lossless Context Management 的一个重要特性是让子 Agent 进行深度上下文遍历**。为了让 context-engine 插件能创建和管理子 Agent，PR 增加了 `runtime.subagent` API：

```typescript
// src/plugins/runtime/types.ts
subagent: {
  run(params):              Promise<SubagentRunResult>;
  waitForRun(params):       Promise<SubagentWaitResult>;
  getSessionMessages(params): Promise<SubagentGetSessionMessagesResult>;
  getSession(params):       Promise<SubagentGetSessionResult>;    // deprecated
  deleteSession(params):    Promise<void>;
};
```

实现方式是在 `server-plugins.ts` 中构造一个**合成的 operator 客户端**，通过内部 gateway 调度。关键技术是 `AsyncLocalStorage`：

```typescript
// src/plugins/runtime/gateway-request-scope.ts
const PLUGIN_RUNTIME_GATEWAY_REQUEST_SCOPE_KEY = Symbol.for(
  "openclaw.pluginRuntimeGatewayRequestScope",
);

// 用 AsyncLocalStorage 在请求作用域中传递 gateway context
const pluginRuntimeGatewayRequestScope = (() => {
  // Symbol.for → globalThis 单例
})();
```

**设计决策 #8：AsyncLocalStorage 请求作用域桥接**

问题：插件运行时需要调 gateway 内部方法，但 gateway 方法需要一个"请求上下文"（认证、广播、节点注册等）。在 WebSocket 路径下这个上下文自然存在，但在非 WS 路径（Telegram、WhatsApp 轮询）下没有。

解决：
1. WS 路径：通过 `AsyncLocalStorage` 在每个请求处理中注入作用域
2. 非 WS 路径：启动时拍一个快照存为 `fallbackGatewayContext`

```typescript
// server.impl.ts 启动时
setFallbackGatewayContext(gatewayRequestContext);

// dispatch 时
const context = scope?.context ?? fallbackGatewayContextState.context;
```

---

## ④ 子 Agent 生命周期回调

### subagent-registry.ts：4 个回调点

子 Agent 结束有 4 种方式，每种都通知 ContextEngine：

```typescript
async function notifyContextEngineSubagentEnded(params: {
  childSessionKey: string;
  reason: SubagentEndReason;  // "deleted" | "completed" | "swept" | "released"
}) {
  try {
    ensureContextEnginesInitialized();
    const engine = await resolveContextEngine(loadConfig());
    if (!engine.onSubagentEnded) return;
    await engine.onSubagentEnded(params);
  } catch (err) {
    log.warn("context-engine onSubagentEnded failed (best-effort)", { err });
  }
}
```

4 个调用点：

| 场景 | reason | 代码位置 |
|------|--------|---------|
| 子 Agent 正常完成 | `"completed"` | `completeCleanupBookkeeping` |
| 用户主动删除 | `"deleted"` | `completeCleanupBookkeeping`（cleanup=delete） |
| 定时清扫超时子 Agent | `"swept"` | `sweepSubagentRuns` |
| 主动释放子 Agent | `"released"` | `releaseSubagentRun` |

**设计决策 #9：best-effort 策略**

所有回调都用 `void` 前缀（fire-and-forget）+ `try/catch` + `log.warn`。回调失败不阻断主流程。这是正确的——上下文引擎的回调是"有则更好"的增强，不是关键路径。

### pi-settings.ts：auto-compaction 守卫

```typescript
export function applyPiAutoCompactionGuard(params: {
  settingsManager: PiSettingsManagerLike;
  contextEngineInfo?: ContextEngineInfo;
}): { supported: boolean; disabled: boolean } {
  const disable = params.contextEngineInfo?.ownsCompaction === true;
  if (!disable || !hasMethod) return { supported: hasMethod, disabled: false };
  params.settingsManager.setCompactionEnabled!(false);
  return { supported: true, disabled: true };
}
```

当引擎声明 `ownsCompaction: true` 时，禁用 Pi SDK 的内置自动压缩。防止引擎自己的压缩策略和 SDK 的自动压缩冲突。

---

## ⑤ Gateway 重构

### server.impl.ts 的"提取变量"重构

这个 diff 看起来 +111 / -50 很大，但本质只做了一件事：把原来内联在 `attachGatewayWsHandlers` 调用中的 `context` 对象**提取为独立变量** `gatewayRequestContext`。

为什么？因为 `setFallbackGatewayContext()` 需要引用这个对象——它既要传给 WS handler，也要作为非 WS 路径的 fallback 存储。

这是一个典型的"先重构再加功能"的步骤——代码行为零变化，只改了变量作用域。

---

## 测试策略

PR 包含 **~600+ 行测试**，覆盖 6 个维度：

```
src/context-engine/context-engine.test.ts — 337 行，19 个测试
├── 1. Engine contract tests        — 接口契约验证
├── 2. Registry tests               — 注册/查找/覆盖
├── 3. Default engine selection     — 默认解析为 legacy
├── 4. Invalid engine fallback      — 错误引擎 id 的报错
├── 5. LegacyContextEngine parity   — legacy 引擎行为验证
└── 6. Initialization guard         — ensureContextEnginesInitialized 幂等性

src/plugins/runtime/gateway-request-scope.test.ts — 23 行
└── AsyncLocalStorage 跨模块重载安全性

其他 ~10+ 个测试文件的 mock 更新
└── 所有引用 PluginRuntime / OpenClawPluginApi 的测试
    都添加了 registerContextEngine / subagent mock
```

**设计决策 #10：mock 全面更新**

PR 更新了所有扩展（diffs、lobster、phone-control、bluebubbles 等）的测试 mock，给 `OpenClawPluginApi` mock 加上 `registerContextEngine`，给 `PluginRuntime` mock 加上 `subagent`。这确保现有测试不会因为接口扩展而断掉。

---

## "What did NOT change" — PR 最重要的约束

引用作者原话：

> **What did NOT change**: Zero changes to existing compaction logic. The LegacyContextEngine wraps existing behavior identically. No new dependencies. No changes to message format, API contracts, or user-facing config (beyond an optional contextEngine slot).

翻译成技术约束：

| 约束 | 保障措施 |
|------|---------|
| 现有压缩逻辑零修改 | `LegacyContextEngine.compact()` 直接委托 `compactEmbeddedPiSessionDirect()` |
| 行为 100% 向后兼容 | legacy 引擎的 ingest=no-op, assemble=pass-through, afterTurn=no-op |
| 无新依赖 | 全部用原生 Node.js API（`Symbol.for`、`AsyncLocalStorage`） |
| 消息格式不变 | `AgentMessage` 类型不改，只在外面包一层 |
| API 契约不变 | gateway 方法签名不变，只内部路由有变化 |
| 配置向后兼容 | `contextEngine` slot 是可选的，不配就用 legacy |

---

## 提交历史（25 commits 的逻辑顺序）

| # | commit 主题 | 做了什么 |
|---|------------|---------|
| 1 | `feat(context-engine): add ContextEngine interface and registry` | 核心接口 + 注册表 + legacy 包装 + 19 个测试 |
| 2 | `feat(plugins): add context-engine slot and registerContextEngine API` | 插件系统接入 |
| 3 | `feat(context-engine): wire ContextEngine into agent run lifecycle` | 嵌入运行循环（bootstrap, assemble, afterTurn, overflow compact, dispose） |
| 4 | `feat(plugins): add scoped subagent methods and gateway request scope` | subagent 运行时 + AsyncLocalStorage scope + fallback context |
| 5 | `feat(context-engine): route /compact and sessions.get through context engine` | /compact 命令路由 + sessions.get 别名 |
| 6 | `style: format with oxfmt 0.33.0` | 代码格式化 |
| 7-12 | `fix: ...` | 测试 mock 修复、重复 import 清理、子 Agent deferred delete 优化 |
| 13 | `fix: pass resolved auth profile into afterTurn compaction` | 确保 afterTurn 路径的 auth 参数与 overflow 路径一致 |
| 14 | `feat: add context-engine system prompt additions` | assemble 返回的 systemPromptAddition 支持 |
| 15-25 | `fix/test/style: ...` | rebase 修复、CI 测试修复、mock 补充、文档 changelog |

值得注意的是 commit 13 的修复——作者在调试中发现 afterTurn 路径的压缩使用了错误的 auth/profile 上下文（而 overflow 路径是正确的），单独修复并加了测试。这类"在集成调试中发现的对称性问题"很典型。

---

## 对我们项目的启示

### 1. "Strangler Fig + 不透明参数包"应对遗留代码

OpenClaw 的做法：新接口包装旧逻辑，用 `legacyParams: Record<string, unknown>` 把旧函数需要的 20+ 个参数打包传递，不污染新接口签名。

我们的 LangGraph agent 也有类似的"大参数包"函数。如果要做类似的可插拔改造，这个模式可以直接套用。

### 2. `Symbol.for()` 解决模块分裂

在 Python 中没有 `Symbol.for()`，但有类似问题——`importlib.reload()` 或多进程场景下模块变量会分裂。Python 的等价方案是用 `__builtins__` 字典或独立的 registry 进程。

### 3. AsyncLocalStorage ≈ Python 的 contextvars

Node.js 的 `AsyncLocalStorage` 用于在异步调用链隐式传递请求上下文，Python 中对应的是 `contextvars.ContextVar`。OpenClaw 的 gateway-request-scope 模式在 FastAPI middleware 中完全可以复现。

### 4. 测试策略：全面更新 mock

当你给一个被广泛引用的接口加字段时，**必须**更新所有 mock。这个 PR 更新了 6+ 个扩展的测试 mock，看起来是体力活，但跳过了就等着 CI 红到崩溃。

---

## 附：文件改动完整清单

<details>
<summary>展开 44 个文件列表</summary>

| 文件 | 改动 | 分类 |
|------|------|------|
| CHANGELOG.md | +1 | 文档 |
| src/context-engine/types.ts | +167 | ① 核心接口 |
| src/context-engine/registry.ts | +67 | ① 核心接口 |
| src/context-engine/legacy.ts | +115 | ① 核心接口 |
| src/context-engine/init.ts | +23 | ① 核心接口 |
| src/context-engine/index.ts | +19 | ① 核心接口 |
| src/context-engine/context-engine.test.ts | +337 | ① 测试 |
| src/agents/pi-embedded-runner/run.ts | +65 /- | ② 运行时 |
| src/agents/pi-embedded-runner/run/attempt.ts | +149 | ② 运行时 |
| src/agents/pi-embedded-runner/run/types.ts | +9 | ② 运行时 |
| src/agents/pi-embedded-runner/compact.ts | +54 /- | ② 运行时 |
| src/agents/pi-settings.ts | +25 | ④ 子 Agent |
| src/agents/subagent-registry.ts | +45 /- | ④ 子 Agent |
| src/gateway/server-plugins.ts | +164 /- | ⑤ Gateway |
| src/gateway/server.impl.ts | +111 /-50 | ⑤ Gateway |
| src/gateway/server-methods.ts | +22 /- | ⑤ Gateway |
| src/gateway/server-methods/sessions.ts | +23 | ⑤ Gateway |
| src/gateway/method-scopes.ts | +1 | ⑤ Gateway |
| src/gateway/config-reload.ts | +2 /-1 | ⑤ Gateway |
| src/plugins/types.ts | +7 /-1 | ③ 插件系统 |
| src/plugins/slots.ts | +2 | ③ 插件系统 |
| src/plugins/registry.ts | +2 | ③ 插件系统 |
| src/plugins/loader.ts | +5 /-1 | ③ 插件系统 |
| src/plugins/runtime/types.ts | +55 | ③ 插件系统 |
| src/plugins/runtime/index.ts | +20 /- | ③ 插件系统 |
| src/plugins/runtime/gateway-request-scope.ts | +46 | ③ 插件系统 |
| src/plugins/runtime/gateway-request-scope.test.ts | +23 | ③ 测试 |
| src/plugin-sdk/index.ts | +29 /- | ③ SDK |
| src/config/types.plugins.ts | +2 | ③ 配置 |
| src/config/zod-schema.ts | +1 | ③ 配置 |
| src/config/schema.help.ts | +2 | ③ 配置 |
| src/config/schema.labels.ts | +1 | ③ 配置 |
| src/config/config-misc.test.ts | +13 | ③ 测试 |
| extensions/diffs/index.test.ts | +2 | ⑥ Mock 更新 |
| extensions/diffs/src/tool.test.ts | +1 | ⑥ Mock 更新 |
| extensions/lobster/src/lobster-tool.test.ts | +1 | ⑥ Mock 更新 |
| extensions/phone-control/index.test.ts | +1 | ⑥ Mock 更新 |
| test/helpers/plugin-runtime-mock.ts | +7 | ⑥ Mock 更新 |
| test/helpers/gateway-server-agent.mocks.ts | +20 /- | ⑥ Mock 更新 |
| src/agents/pi-embedded-runner/run/attempt.test.ts | +54 /- | ⑥ 测试 |
| src/agents/pi-embedded-runner/compaction.overflow-compaction.test.ts | +16 | ⑥ 测试 |
| src/gateway/server-plugins.test.ts | +159 /- | ⑥ 测试 |
| src/cron/cron-agent.model-formatting.test.ts | +541 | ⑥ 测试（附带） |
| test/scripts/ios-team-id.test.ts | +2 /-1 | ⑥ 测试（附带） |

</details>

---

> 📎 **配套文章**：本文专注于代码改动的逐个解析。想了解 ContextEngine 的整体设计原理和接口演变故事？→ [10 - ContextEngine 插件接口：从硬编码到可插拔的演变](10-context-engine.md)
