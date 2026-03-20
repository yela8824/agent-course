# 97 - Agent 自适应超时与对冲请求（Adaptive Timeout & Hedged Requests）

> 核心问题：固定超时要么太紧导致误判，要么太松导致用户等待。怎么办？

---

## 背景

生产环境里工具调用的延迟是**抖动的**：

- 数据库查询：正常 20ms，偶尔 500ms（索引失效、锁竞争）
- LLM API：正常 800ms，有时 5s（模型负载高）
- 外部 HTTP：正常 100ms，偶尔 30s（CDN 故障）

**固定超时的问题：**
- 超时设太短 → 频繁误触，正常请求被杀死
- 超时设太长 → P99 用户等待时间爆炸

**解法：两个互补的模式**

1. **Adaptive Timeout（自适应超时）**：根据历史延迟数据动态计算合理超时值
2. **Hedged Requests（对冲请求）**：等待 P50 延迟后同时发第二个请求，取最快响应

---

## 模式一：Adaptive Timeout 自适应超时

### 核心思路

维护每个工具的历史延迟分布，动态设置超时 = `P95 × 1.5 + 固定抖动余量`。

```typescript
// adaptive-timeout.ts

interface LatencyStats {
  samples: number[];       // 最近 N 次延迟（ms）
  maxSamples: number;      // 滑动窗口大小
}

class AdaptiveTimeoutManager {
  private stats = new Map<string, LatencyStats>();
  private readonly DEFAULT_TIMEOUT = 5000;  // 冷启动默认值
  private readonly WINDOW_SIZE = 100;        // 保留最近 100 次样本

  // 记录一次工具调用延迟
  record(toolName: string, latencyMs: number): void {
    if (!this.stats.has(toolName)) {
      this.stats.set(toolName, { samples: [], maxSamples: this.WINDOW_SIZE });
    }
    const stat = this.stats.get(toolName)!;
    stat.samples.push(latencyMs);
    // 滑动窗口：超出后删头部
    if (stat.samples.length > stat.maxSamples) {
      stat.samples.shift();
    }
  }

  // 计算建议超时值
  getTimeout(toolName: string): number {
    const stat = this.stats.get(toolName);
    if (!stat || stat.samples.length < 10) {
      return this.DEFAULT_TIMEOUT;  // 样本不足，用默认值
    }

    const p95 = this.percentile(stat.samples, 95);
    const p99 = this.percentile(stat.samples, 99);
    
    // 超时 = P95 × 1.5，但不超过 P99 × 2（防止极端值）
    const timeout = Math.min(p95 * 1.5, p99 * 2);
    
    // 最小 500ms，最大 30s
    return Math.max(500, Math.min(30000, timeout));
  }

  // 获取对冲请求触发时机（P50 延迟）
  getHedgeDelay(toolName: string): number {
    const stat = this.stats.get(toolName);
    if (!stat || stat.samples.length < 10) return 1000;
    return this.percentile(stat.samples, 50);
  }

  private percentile(samples: number[], p: number): number {
    const sorted = [...samples].sort((a, b) => a - b);
    const idx = Math.floor((p / 100) * sorted.length);
    return sorted[Math.min(idx, sorted.length - 1)];
  }
}

// 包装工具调用，自动应用自适应超时
async function callToolWithAdaptiveTimeout<T>(
  manager: AdaptiveTimeoutManager,
  toolName: string,
  fn: () => Promise<T>
): Promise<T> {
  const timeout = manager.getTimeout(toolName);
  const start = Date.now();

  try {
    const result = await Promise.race([
      fn(),
      new Promise<never>((_, reject) =>
        setTimeout(() => reject(new Error(`Tool ${toolName} timed out after ${timeout}ms`)), timeout)
      )
    ]);

    // 记录成功延迟
    manager.record(toolName, Date.now() - start);
    return result;
  } catch (err) {
    // 超时也记录（用超时值作为样本）
    if ((err as Error).message.includes('timed out')) {
      manager.record(toolName, timeout);
    }
    throw err;
  }
}
```

---

## 模式二：Hedged Requests 对冲请求

### 核心思路

"对冲"来自金融：**同时押注两边，赢一个就行。**

- 发出第一个请求
- 等待 P50 时间（中位延迟）
- 如果第一个还没返回 → 发出第二个**相同**请求
- 谁先返回用谁，另一个取消

```typescript
// hedged-requests.ts

class HedgedRequestExecutor {
  constructor(private timeout: AdaptiveTimeoutManager) {}

  async execute<T>(
    toolName: string,
    fn: () => Promise<T>,
    options: { maxHedges?: number } = {}
  ): Promise<T> {
    const { maxHedges = 1 } = options;
    const hedgeDelay = this.timeout.getHedgeDelay(toolName);
    const totalTimeout = this.timeout.getTimeout(toolName);

    // AbortController 用于取消慢的那个请求
    const controllers: AbortController[] = [];
    const start = Date.now();

    const makeAttempt = (signal: AbortSignal): Promise<T> => {
      return new Promise<T>((resolve, reject) => {
        signal.addEventListener('abort', () =>
          reject(new DOMException('Hedged request cancelled', 'AbortError'))
        );
        fn().then(resolve, reject);
      });
    };

    try {
      const result = await new Promise<T>((resolve, reject) => {
        let settled = false;
        let hedgeCount = 0;
        const attempts: Promise<void>[] = [];

        const settle = (val: T | Error, isError = false) => {
          if (settled) return;
          settled = true;
          // 取消所有其他请求
          controllers.forEach(c => c.abort());
          if (isError) reject(val as Error);
          else resolve(val as T);
        };

        // 第一个请求
        const ctrl0 = new AbortController();
        controllers.push(ctrl0);
        const attempt0 = makeAttempt(ctrl0.signal)
          .then(v => settle(v))
          .catch(e => { if (!settled && e.name !== 'AbortError') settle(e, true); });
        attempts.push(attempt0);

        // 对冲计时器：等 P50 后发第二个
        const scheduleHedge = () => {
          if (hedgeCount >= maxHedges || settled) return;
          hedgeCount++;
          const ctrl = new AbortController();
          controllers.push(ctrl);
          console.log(`[Hedge] ${toolName}: launching hedge #${hedgeCount} after ${hedgeDelay}ms`);
          const attemptN = makeAttempt(ctrl.signal)
            .then(v => settle(v))
            .catch(e => { if (!settled && e.name !== 'AbortError') settle(e, true); });
          attempts.push(attemptN);
        };

        setTimeout(scheduleHedge, hedgeDelay);

        // 总超时
        setTimeout(() => {
          if (!settled) settle(new Error(`${toolName} exceeded total timeout ${totalTimeout}ms`), true);
        }, totalTimeout);
      });

      this.timeout.record(toolName, Date.now() - start);
      return result;
    } catch (err) {
      if ((err as Error).message?.includes('timeout')) {
        this.timeout.record(toolName, totalTimeout);
      }
      throw err;
    }
  }
}
```

### 何时适合对冲？

| 场景 | 适合对冲 | 原因 |
|------|----------|------|
| 读操作（查询、搜索） | ✅ | 幂等，发两次没问题 |
| LLM API 调用 | ✅ | 每次独立，取消未完成的 |
| 数据库写入 | ❌ | 非幂等，会造成重复写 |
| 文件上传 | ❌ | 非幂等，浪费带宽 |
| 支付接口 | ❌ | 绝对不行！ |

---

## 在 Agent 工具调用中集成

```typescript
// agent-tool-executor.ts（集成两个模式）

const latencyManager = new AdaptiveTimeoutManager();
const hedgedExecutor = new HedgedRequestExecutor(latencyManager);

// 工具元数据：标记是否支持对冲
const TOOL_CONFIG: Record<string, { hedgeable: boolean }> = {
  'web_search':    { hedgeable: true },   // 读操作，可对冲
  'web_fetch':     { hedgeable: true },   // 读操作，可对冲
  'memory_search': { hedgeable: true },   // 读操作，可对冲
  'exec':          { hedgeable: false },  // 副作用，不可对冲
  'Write':         { hedgeable: false },  // 写文件，不可对冲
  'message_send':  { hedgeable: false },  // 发消息，不可对冲
};

async function dispatchTool(name: string, params: unknown): Promise<unknown> {
  const config = TOOL_CONFIG[name] ?? { hedgeable: false };

  if (config.hedgeable) {
    // 可对冲工具：用对冲执行器（内含自适应超时）
    return hedgedExecutor.execute(name, () => rawToolCall(name, params));
  } else {
    // 不可对冲工具：只用自适应超时
    return callToolWithAdaptiveTimeout(latencyManager, name, () => rawToolCall(name, params));
  }
}
```

---

## OpenClaw 视角：实际效果

OpenClaw 中每个工具调用都经过调度层。假设 `web_search` 历史 P50=800ms，P95=3s：

```
用户: 搜索 X

→ 发出 web_search 请求 #1
→ 等待 800ms（P50）
→ 请求 #1 还没返回 → 发出请求 #2（对冲）
→ 请求 #2 在 600ms 后返回
→ 取消请求 #1，使用请求 #2 的结果
→ 总耗时：800 + 600 = 1400ms（比等请求 #1 超时快多了）
```

**实测收益（Google 工程博客数据）：**
- P99 延迟降低 **40-60%**
- 额外请求量增加 **< 5%**（因为只有慢的那次才触发对冲）

---

## pi-mono 中的类似模式

```python
# pi-mono 中使用 asyncio 实现对冲
import asyncio

async def hedged_call(coro_factory, hedge_delay: float, timeout: float):
    """对冲请求：等 hedge_delay 秒后如未完成则发第二个"""
    
    async def attempt():
        return await coro_factory()
    
    task1 = asyncio.create_task(attempt())
    
    # 等待 hedge_delay，如果 task1 完成则直接返回
    done, pending = await asyncio.wait(
        [task1], timeout=hedge_delay
    )
    
    if task1 in done:
        return task1.result()  # 快，无需对冲
    
    # 发对冲请求
    task2 = asyncio.create_task(attempt())
    
    try:
        done, pending = await asyncio.wait(
            [task1, task2],
            timeout=timeout - hedge_delay,
            return_when=asyncio.FIRST_COMPLETED
        )
        for task in pending:
            task.cancel()
        
        if not done:
            raise TimeoutError(f"Both hedged requests timed out")
        
        return next(iter(done)).result()
    except Exception:
        task1.cancel()
        task2.cancel()
        raise
```

---

## 关键指标监控

```typescript
// 记录对冲触发率，判断系统健康
interface HedgeMetrics {
  total: number;
  hedged: number;      // 触发过对冲的请求数
  hedgeWon: number;    // 对冲请求比原始请求更快的次数
}

// 健康的系统：hedgeRate < 5%
// 如果 hedgeRate > 20%，说明 P50 本身就太慢了，要排查根因
const hedgeRate = metrics.hedged / metrics.total;
```

---

## 总结

| 模式 | 解决问题 | 代价 |
|------|----------|------|
| Adaptive Timeout | 超时值跟上实际延迟，减少误判 | 需维护延迟统计状态 |
| Hedged Requests | 消除 P99 长尾，用户感知更快 | 偶尔 2× 请求量（< 5% 频率） |

**黄金组合：** Hedged Requests 处理偶发慢请求（尾延迟），Adaptive Timeout 处理系统性变慢（整体退化）。两者互补，缺一不可。

---

**下一课预告：** Agent 知识库热更新与增量索引（Knowledge Base Hot Update）
