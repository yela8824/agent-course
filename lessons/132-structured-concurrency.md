# 132 - Agent 结构化并发（Structured Concurrency）：任务树生命周期管理

## 为什么需要结构化并发？

Agent 经常需要并发调用多个工具，比如同时查数据库、调外部 API、读文件。

常见做法是 `Promise.all`，但裸 `Promise.all` 有三个隐患：

1. **僵尸任务**：某个任务失败后，其他任务还在跑，浪费资源
2. **取消传播缺失**：用户中断请求，正在跑的工具调用没有被取消
3. **资源泄漏**：任务抛异常，没有执行清理逻辑（关连接、释放锁）

**结构化并发**（Structured Concurrency）的核心思想：

> 所有并发任务都有一个**父作用域（Scope）**，父作用域结束时，所有子任务必须已完成或被取消。没有任务能"逃逸"出它的作用域。

这是 Kotlin 协程、Swift Async 的核心设计，在 Agent 工具调用层同样适用。

---

## 核心概念

```
TaskScope（任务作用域）
│
├── Task A（search_web）
├── Task B（query_database）
└── Task C（call_api）
     │
     └── SubTask C1（retry）
```

- **Scope** 管理所有子任务的生命周期
- **任意子任务失败** → Scope 取消其他所有任务 → 清理资源 → 向上抛出错误
- **父被取消** → AbortSignal 传播到所有子任务

---

## 代码实现

### TaskScope（核心）

```typescript
// agent/structured-concurrency.ts

interface TaskOptions {
  signal?: AbortSignal;
  timeoutMs?: number;
}

interface ScopeResult<T> {
  results: T[];
  cancelled: number;
  errors: Error[];
}

class TaskScope {
  private controller: AbortController;
  private tasks: Promise<unknown>[] = [];
  private cleanups: (() => Promise<void>)[] = [];

  constructor(parentSignal?: AbortSignal) {
    this.controller = new AbortController();

    // 父信号取消时，自动取消整个 scope
    if (parentSignal) {
      parentSignal.addEventListener("abort", () => {
        this.controller.abort(parentSignal.reason);
      });
    }
  }

  get signal(): AbortSignal {
    return this.controller.signal;
  }

  // 注册一个并发任务
  spawn<T>(
    name: string,
    fn: (signal: AbortSignal) => Promise<T>,
    cleanup?: () => Promise<void>
  ): Promise<T> {
    if (cleanup) this.cleanups.push(cleanup);

    const task = fn(this.signal).catch((err) => {
      // 任一任务失败 → 取消整个 scope（fail-fast 模式）
      if (!this.signal.aborted) {
        console.error(`[TaskScope] Task "${name}" failed:`, err.message);
        this.controller.abort(err);
      }
      throw err;
    });

    this.tasks.push(task);
    return task as Promise<T>;
  }

  // 等待所有任务，保证清理
  async join(): Promise<void> {
    try {
      await Promise.all(this.tasks);
    } finally {
      await this.runCleanups();
    }
  }

  private async runCleanups(): Promise<void> {
    await Promise.allSettled(this.cleanups.map((fn) => fn()));
  }

  cancel(reason?: string): void {
    this.controller.abort(reason ?? "scope cancelled");
  }
}
```

### 在 Agent 工具调用层使用

```typescript
// tools/parallel-tool-runner.ts

import { TaskScope } from "../agent/structured-concurrency";

interface ToolCall {
  name: string;
  params: Record<string, unknown>;
}

interface ToolResult {
  name: string;
  result: unknown;
  durationMs: number;
}

async function runToolsInScope(
  toolCalls: ToolCall[],
  executor: (call: ToolCall, signal: AbortSignal) => Promise<unknown>,
  parentSignal?: AbortSignal
): Promise<ToolResult[]> {
  const scope = new TaskScope(parentSignal);
  const results: ToolResult[] = [];

  // 所有工具调用在同一个 scope 内并发运行
  const promises = toolCalls.map((call) =>
    scope.spawn(
      call.name,
      async (signal) => {
        const start = Date.now();
        const result = await executor(call, signal);
        results.push({
          name: call.name,
          result,
          durationMs: Date.now() - start,
        });
      },
      // 清理函数：释放连接、日志等
      async () => {
        console.log(`[Cleanup] ${call.name}`);
      }
    )
  );

  try {
    await Promise.all(promises);
  } catch (err) {
    // scope 会自动取消其他任务并运行清理
    throw err;
  } finally {
    await scope.join();
  }

  return results;
}
```

### 带超时的 Scope（常用模式）

```typescript
// 给整个 scope 设置超时 —— 超时后所有子任务一起取消
async function runWithTimeout<T>(
  fn: (scope: TaskScope) => Promise<T>,
  timeoutMs: number
): Promise<T> {
  const scope = new TaskScope();
  const timer = setTimeout(() => scope.cancel(`timeout after ${timeoutMs}ms`), timeoutMs);

  try {
    const result = await fn(scope);
    await scope.join();
    return result;
  } finally {
    clearTimeout(timer);
  }
}

// 使用示例
const results = await runWithTimeout(async (scope) => {
  const [weather, news, stocks] = await Promise.all([
    scope.spawn("weather", (sig) => fetchWeather("Sydney", sig)),
    scope.spawn("news", (sig) => fetchNews("tech", sig)),
    scope.spawn("stocks", (sig) => fetchStocks(["AAPL"], sig)),
  ]);
  return { weather, news, stocks };
}, 5000); // 5秒全局超时，任一失败立即取消其他
```

---

## OpenClaw 实战场景

OpenClaw 的工具执行层（`tool-executor.ts`）可以直接集成：

```typescript
// 伪代码，展示 OpenClaw 工具执行集成思路
class AgentToolExecutor {
  async executeParallel(
    toolCalls: ClaudeToolCall[],
    requestSignal: AbortSignal // HTTP 请求取消信号
  ): Promise<ClaudeToolResult[]> {
    const scope = new TaskScope(requestSignal);
    const results: ClaudeToolResult[] = [];

    for (const call of toolCalls) {
      scope.spawn(call.name, async (signal) => {
        const result = await this.dispatch(call, signal);
        results.push({
          type: "tool_result",
          tool_use_id: call.id,
          content: result,
        });
      });
    }

    await scope.join(); // 等所有工具完成，确保清理

    return results;
  }
}
```

**关键点**：`requestSignal` 来自 HTTP 请求（用户按 Stop 或网络断开），传入 Scope 后，所有工具调用自动取消。这在 OpenClaw 的 Telegram 消息处理中特别有用：用户发新消息打断旧请求时，旧的工具调用不会继续跑。

---

## 三种 Scope 策略对比

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| **Fail-Fast**（默认） | 任一失败 → 取消所有 | 结果互相依赖的工具调用 |
| **Collect-All** | 等所有完成，收集成功/失败 | 独立工具，部分失败可接受 |
| **Race** | 第一个完成 → 取消其他 | 多源数据，取最快响应 |

```typescript
// Collect-All 策略
async function runScopeCollectAll<T>(
  fns: Array<(signal: AbortSignal) => Promise<T>>,
  parentSignal?: AbortSignal
): Promise<Array<{ ok: true; value: T } | { ok: false; error: Error }>> {
  const scope = new TaskScope(parentSignal);

  const tasks = fns.map((fn) =>
    fn(scope.signal)
      .then((value) => ({ ok: true as const, value }))
      .catch((error) => ({ ok: false as const, error }))
  );

  const settled = await Promise.all(tasks);
  await scope.join();
  return settled;
}

// Race 策略
async function runScopeRace<T>(
  fns: Array<(signal: AbortSignal) => Promise<T>>,
  parentSignal?: AbortSignal
): Promise<T> {
  const scope = new TaskScope(parentSignal);

  const result = await Promise.race(fns.map((fn) => fn(scope.signal)));
  scope.cancel("race won"); // 取消其他竞争者
  await scope.join();
  return result;
}
```

---

## 与「并发工具执行」（Lesson 26）的区别

| 维度 | Lesson 26（简单并发） | Lesson 132（结构化并发） |
|------|--------------------|----------------------|
| 取消传播 | 手动处理 | 自动，通过 AbortSignal 树 |
| 资源清理 | 需要 try/finally | Scope 保证 cleanup 一定运行 |
| 任务逃逸 | 可能发生 | 不可能，Scope 结束前必须 join |
| 错误传播 | 第一个 reject | 取消所有，可配置策略 |
| 嵌套任务 | 不处理 | SubScope 自然形成树形结构 |

---

## 关键收益

1. **零资源泄漏**：cleanup 函数在 Scope 结束时一定执行
2. **用户中断有效**：AbortSignal 树确保取消从顶层传播到所有叶子任务
3. **调试友好**：每个 Task 有名字，错误日志清晰
4. **代码意图清晰**：看到 `scope.spawn(...)` 就知道这是受管理的并发，不是"野生"Promise

---

## 总结

结构化并发是对「裸 Promise.all」的工程化升级：

```
裸 Promise.all = 并发执行，但任务可能逃逸、泄漏、取消不彻底
TaskScope     = 并发执行 + 生命周期保证 + 取消传播 + 清理保证
```

在 Agent 系统中，每次用户请求都应该有一个对应的 `TaskScope`，所有工具调用都在这个 scope 内运行。请求结束（完成或取消）时，scope 确保一切干净收尾。

这是从「能跑」到「可靠」的关键一步。
