# 85. Agent 工具调用预测与预取（Tool Call Prediction & Prefetching）

> 在 Agent 还在"思考"时，就预测它接下来要调哪个工具并提前执行——类似 CPU 分支预测，命中时延迟归零。

---

## 为什么需要预取？

Agent 的延迟来源有两块：

```
总延迟 = LLM 推理时间 + 工具执行时间
```

Prompt Caching、量化模型可以压 LLM 推理；**预取**直接把工具执行时间从关键路径上搬走。

对于可预测的工作流（如：先 search → 再 fetch_page → 再 summarize），第 N 步工具执行时第 N+1 步的调用几乎是确定的。

---

## 核心思路

```
用户输入
    │
    ▼
[意图分析] ──预测下 1~2 步工具──► [后台异步执行预取]
    │                                        │
    ▼                                        ▼
[LLM 思考中...]                    [预取结果存入缓存]
    │                                        │
    ▼                                        │
[LLM 决定调 tool_A] ─────命中缓存!──────────┘
    │                    (延迟 ≈ 0)
    ▼
[继续推理]
```

---

## 实现：Python（教学版，基于 learn-claude-code 风格）

```python
import asyncio
import hashlib
import json
from typing import Any, Callable, Coroutine

# ─── 预取缓存 ─────────────────────────────────────────────
class PrefetchCache:
    def __init__(self):
        self._store: dict[str, asyncio.Future] = {}

    def _key(self, tool: str, args: dict) -> str:
        payload = json.dumps({"tool": tool, "args": args}, sort_keys=True)
        return hashlib.sha256(payload.encode()).hexdigest()[:16]

    def prefetch(self, tool: str, args: dict, executor: Callable[..., Coroutine]):
        """提交预取任务（幂等：同一 key 只跑一次）"""
        key = self._key(tool, args)
        if key not in self._store:
            future = asyncio.ensure_future(executor(tool, args))
            self._store[key] = future
            print(f"  [prefetch] 预取 {tool}({args}) → key={key}")
        return key

    async def get(self, tool: str, args: dict) -> tuple[bool, Any]:
        """获取预取结果；命中则 hit=True，否则同步执行"""
        key = self._key(tool, args)
        if key in self._store:
            result = await self._store.pop(key)
            return True, result
        return False, None


# ─── 工作流预测器 ──────────────────────────────────────────
# 基于静态规则；生产中可换成小型分类模型
WORKFLOW_PREDICTIONS: dict[str, list[tuple[str, dict]]] = {
    "web_search":    [("fetch_page", {"url": "__INFER__"})],
    "fetch_page":    [("summarize",  {"length": "short"})],
    "list_files":    [("read_file",  {"path": "__INFER__"})],
    "read_file":     [("analyze",    {})],
    "sql_query":     [("format_table", {})],
}

def predict_next_calls(current_tool: str, current_args: dict) -> list[tuple[str, dict]]:
    """给定当前工具，预测下一步可能的调用"""
    predictions = WORKFLOW_PREDICTIONS.get(current_tool, [])
    resolved = []
    for next_tool, next_args in predictions:
        # 简单的参数继承：把当前结果的关键字段透传
        if "__INFER__" in next_args.values():
            next_args = {k: current_args.get(k, v) for k, v in next_args.items()}
        resolved.append((next_tool, next_args))
    return resolved


# ─── Agent Loop（带预取）──────────────────────────────────
cache = PrefetchCache()

async def execute_tool(tool: str, args: dict) -> Any:
    """模拟工具执行（实际替换为真实调用）"""
    await asyncio.sleep(0.3)  # 模拟 300ms 网络延迟
    return f"[result of {tool}({args})]"

async def run_tool_with_prefetch(tool: str, args: dict) -> Any:
    """执行工具：优先命中预取缓存"""
    hit, result = await cache.get(tool, args)
    if hit:
        print(f"  ✅ 预取命中: {tool} → 延迟 ≈ 0ms")
        return result

    print(f"  ⏳ 预取未命中，同步执行: {tool}")
    result = await execute_tool(tool, args)

    # 执行完当前工具后，立刻预取下一步
    for next_tool, next_args in predict_next_calls(tool, args):
        cache.prefetch(next_tool, next_args, execute_tool)

    return result

async def agent_loop(steps: list[tuple[str, dict]]):
    """模拟 Agent 依次调用工具"""
    # 第一步在 LLM 推理期间就触发预取
    if steps:
        first_tool, first_args = steps[0]
        cache.prefetch(first_tool, first_args, execute_tool)

    for tool, args in steps:
        print(f"\n[Agent] 调用 {tool}...")
        result = await run_tool_with_prefetch(tool, args)
        print(f"[Agent] 结果: {result}")

# ─── 运行示例 ─────────────────────────────────────────────
asyncio.run(agent_loop([
    ("web_search",  {"query": "Claude API pricing"}),
    ("fetch_page",  {"url": "https://anthropic.com/pricing"}),
    ("summarize",   {"length": "short"}),
]))
```

**输出示意：**
```
  [prefetch] 预取 web_search(...) → key=a3f2...

[Agent] 调用 web_search...
  ✅ 预取命中: web_search → 延迟 ≈ 0ms
  [prefetch] 预取 fetch_page(...) → key=b7c1...

[Agent] 调用 fetch_page...
  ✅ 预取命中: fetch_page → 延迟 ≈ 0ms
  [prefetch] 预取 summarize(...) → key=d4e9...

[Agent] 调用 summarize...
  ✅ 预取命中: summarize → 延迟 ≈ 0ms
```

---

## TypeScript 实现（pi-mono 风格）

```typescript
// tool-prefetch.ts
type ToolExecutor = (tool: string, args: Record<string, unknown>) => Promise<unknown>;

class PrefetchCache {
  private store = new Map<string, Promise<unknown>>();

  private key(tool: string, args: Record<string, unknown>): string {
    return `${tool}:${JSON.stringify(args, Object.keys(args).sort())}`;
  }

  prefetch(tool: string, args: Record<string, unknown>, executor: ToolExecutor): void {
    const k = this.key(tool, args);
    if (!this.store.has(k)) {
      console.log(`[prefetch] 预取 ${tool}`);
      this.store.set(k, executor(tool, args));
    }
  }

  async get(
    tool: string,
    args: Record<string, unknown>
  ): Promise<{ hit: boolean; result?: unknown }> {
    const k = this.key(tool, args);
    const p = this.store.get(k);
    if (p) {
      this.store.delete(k);
      return { hit: true, result: await p };
    }
    return { hit: false };
  }
}

// 工作流转换图（可从配置文件/数据库加载）
const WORKFLOW_GRAPH: Record<string, Array<[string, Record<string, unknown>]>> = {
  search:    [["fetch", { depth: 1 }]],
  fetch:     [["parse", {}]],
  list_dir:  [["read_file", {}]],
};

export class PrefetchingAgent {
  private cache = new PrefetchCache();

  constructor(private executor: ToolExecutor) {}

  async call(tool: string, args: Record<string, unknown>): Promise<unknown> {
    // 1. 尝试命中预取
    const { hit, result } = await this.cache.get(tool, args);
    if (hit) {
      console.log(`✅ 命中预取: ${tool}`);
      return result;
    }

    // 2. 同步执行
    console.log(`⏳ 执行: ${tool}`);
    const res = await this.executor(tool, args);

    // 3. 触发下一步预取
    for (const [nextTool, nextArgs] of WORKFLOW_GRAPH[tool] ?? []) {
      this.cache.prefetch(nextTool, nextArgs, this.executor);
    }

    return res;
  }
}
```

---

## OpenClaw 中的真实形态

OpenClaw 的 `exec` 工具本质上是异步的——每次调用返回 `sessionId`，可以立刻再发起下一个。这天然支持预取：

```javascript
// OpenClaw 伪代码：在 LLM 流式生成时就开始预取
const prefetchPromise = exec({
  command: "git log --oneline -20",  // 预判 LLM 会要这个
  background: true,
  yieldMs: 5000
});

// LLM 确认要调这个工具时
const result = await prefetchPromise; // 已经跑了 N 秒，接近完成
```

---

## 预测精度的两种提升方式

### 1. 静态规则（零成本，适合固定流程）
```python
RULES = {
    "get_user": [("get_user_orders", {"user_id": "$result.id"})],
    "get_order": [("get_order_items", {"order_id": "$result.id"})],
}
```

### 2. 概率预测（基于历史 trace）
```python
# 统计每个工具调用后，下一个工具的频率分布
transition_matrix = {
    "search":     {"fetch": 0.8, "summarize": 0.15, "done": 0.05},
    "fetch":      {"parse": 0.7, "search": 0.2,    "done": 0.1},
}

def top_k_predictions(current_tool: str, k=2) -> list[str]:
    dist = transition_matrix.get(current_tool, {})
    return sorted(dist, key=dist.get, reverse=True)[:k]
```

---

## 什么时候不该预取？

| 场景 | 原因 |
|------|------|
| 写操作（DELETE/UPDATE） | 预取写操作会产生副作用 |
| 高度分支的工作流 | 预测精度低，浪费资源 |
| 工具执行成本极高 | 错误预取的浪费大于收益 |
| 有副作用的 API | 幂等性不保证 |

**原则：只预取读操作、幂等操作。**

---

## 性能收益估算

假设 3 步工作流，每步工具执行 300ms：

| 策略 | 总延迟 |
|------|--------|
| 串行执行 | 900ms |
| 并发执行（DAG） | 300ms（全部可并行时）|
| 预取（100% 命中） | ~0ms 工具延迟 |
| 预取（60% 命中） | 约 360ms（节省 60%）|

---

## 关键要点

1. **预取 = 把工具执行从关键路径搬走**，LLM 思考时工具已在跑
2. **只预取幂等/只读工具**，写操作绝不预取
3. **命中率决定收益**：固定流程 > 80% 命中；用历史数据训练转移矩阵可达 60-75%
4. **未命中要降级**：预取 miss 时同步执行，不影响正确性
5. **配合 Prompt Caching**（第 84 课）效果倍增：LLM 延迟降低 + 工具延迟降低

---

## 延伸阅读

- [HTTP/2 Server Push（类似思想）](https://web.dev/performance/http2)
- CPU 分支预测 → speculative execution
- 第 48 课：Caching Strategies
- 第 84 课：Prompt Caching（cache_control）
- 第 58 课：Concurrent Tool Execution
