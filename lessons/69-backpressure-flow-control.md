# 69 - Agent 背压与流量控制（Backpressure & Flow Control）

> **核心问题**：当 Agent 接收的任务速率超过处理能力时，系统会怎样？——堆积、崩溃、超时风暴。背压机制让 Agent 优雅地告诉上游「慢点，我处理不过来了」。

---

## 为什么需要背压？

```
用户/上游                Agent                  工具/LLM
   │                      │                        │
   ├──task1──────────────▶│                        │
   ├──task2──────────────▶│  处理中...              │
   ├──task3──────────────▶│  队列积压...            │
   ├──task4──────────────▶│  内存撑满...            │
   ├──task500────────────▶│  💥 OOM / 超时风暴      │
```

没有背压的 Agent 在流量峰值时：
- 任务队列无限增长 → OOM
- 所有任务都在等待 → 全部超时
- 重试风暴让情况更糟
- 一个慢工具阻塞整个 pipeline

---

## 核心模式一：信号量（Semaphore）并发控制

最简单的背压：限制同时运行的任务数。

```typescript
// pi-mono 风格：用 Semaphore 控制并发
class Semaphore {
  private permits: number;
  private queue: Array<() => void> = [];

  constructor(permits: number) {
    this.permits = permits;
  }

  async acquire(): Promise<void> {
    if (this.permits > 0) {
      this.permits--;
      return;
    }
    // 没有 permit 了，排队等待
    return new Promise(resolve => {
      this.queue.push(resolve);
    });
  }

  release(): void {
    if (this.queue.length > 0) {
      // 有等待者，直接传递 permit
      const next = this.queue.shift()!;
      next();
    } else {
      this.permits++;
    }
  }
}

// Agent 工具调用池：最多 5 个并发
const toolSemaphore = new Semaphore(5);

async function callToolWithBackpressure(
  tool: string,
  params: unknown
): Promise<unknown> {
  await toolSemaphore.acquire();
  try {
    return await callTool(tool, params);
  } finally {
    toolSemaphore.release(); // 无论成功失败都要释放
  }
}
```

**OpenClaw 中的体现**：`concurrent-tool-execution` 模块内部就用了类似机制，每个 session 最多并发 N 个工具调用。

---

## 核心模式二：令牌桶（Token Bucket）

比信号量更灵活：允许突发，但限制平均速率。

```typescript
class TokenBucket {
  private tokens: number;
  private readonly capacity: number;
  private readonly refillRate: number; // tokens per ms
  private lastRefill: number;

  constructor(capacity: number, refillPerSecond: number) {
    this.capacity = capacity;
    this.tokens = capacity;
    this.refillRate = refillPerSecond / 1000;
    this.lastRefill = Date.now();
  }

  private refill(): void {
    const now = Date.now();
    const elapsed = now - this.lastRefill;
    const newTokens = elapsed * this.refillRate;
    this.tokens = Math.min(this.capacity, this.tokens + newTokens);
    this.lastRefill = now;
  }

  tryConsume(tokens = 1): boolean {
    this.refill();
    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true;
    }
    return false; // 背压！告诉调用方现在不能处理
  }

  async consume(tokens = 1): Promise<void> {
    while (!this.tryConsume(tokens)) {
      // 等待到有足够 token
      const waitMs = (tokens - this.tokens) / this.refillRate;
      await sleep(Math.min(waitMs, 100)); // 最多等 100ms 再检查
    }
  }
}

// 实际使用：LLM API 限速 60 RPM
const llmBucket = new TokenBucket(10, 1); // 容量10，每秒补充1个

async function callLLMWithRateLimit(messages: Message[]): Promise<string> {
  await llmBucket.consume(1);
  return await llmClient.complete(messages);
}
```

---

## 核心模式三：有界队列 + 丢弃策略

不能无限等待时，选择性丢弃任务。

```typescript
type OverflowStrategy = 'drop-oldest' | 'drop-newest' | 'reject';

class BoundedTaskQueue<T> {
  private queue: T[] = [];
  private readonly maxSize: number;
  private readonly strategy: OverflowStrategy;
  private droppedCount = 0;

  constructor(maxSize: number, strategy: OverflowStrategy = 'drop-oldest') {
    this.maxSize = maxSize;
    this.strategy = strategy;
  }

  enqueue(task: T): boolean {
    if (this.queue.length < this.maxSize) {
      this.queue.push(task);
      return true;
    }

    // 队列满了，执行溢出策略
    switch (this.strategy) {
      case 'drop-oldest':
        this.queue.shift(); // 丢弃最老的
        this.queue.push(task);
        this.droppedCount++;
        console.warn(`[Queue] 丢弃旧任务，已丢弃总数: ${this.droppedCount}`);
        return true;

      case 'drop-newest':
        this.droppedCount++;
        console.warn(`[Queue] 丢弃新任务，已丢弃总数: ${this.droppedCount}`);
        return false;

      case 'reject':
        throw new Error(`队列已满（${this.maxSize}），拒绝新任务`);
    }
  }

  dequeue(): T | undefined {
    return this.queue.shift();
  }

  get size(): number { return this.queue.length; }
  get pressure(): number { return this.queue.length / this.maxSize; } // 0.0~1.0
}

// Agent 任务调度器：队列满了丢旧任务（用户新指令优先）
const taskQueue = new BoundedTaskQueue<AgentTask>(100, 'drop-oldest');
```

---

## 核心模式四：自适应背压（Adaptive Backpressure）

根据系统负载动态调整处理速率——这是 OpenClaw 风格的做法。

```typescript
class AdaptiveBackpressure {
  private windowMs = 10_000; // 10s 滑动窗口
  private latencies: number[] = [];
  private errorCount = 0;
  private requestCount = 0;

  // 记录一次调用结果
  record(latencyMs: number, isError: boolean): void {
    const now = Date.now();
    this.latencies.push(latencyMs);
    if (isError) this.errorCount++;
    this.requestCount++;

    // 清理过期数据（简化版）
    if (this.latencies.length > 100) {
      this.latencies.shift();
    }
  }

  // 获取当前「压力分」0.0（轻松）~ 1.0（濒临崩溃）
  get pressureScore(): number {
    if (this.requestCount === 0) return 0;

    const p95 = this.percentile(95);
    const errorRate = this.errorCount / this.requestCount;

    // 综合延迟和错误率
    const latencyScore = Math.min(p95 / 5000, 1.0); // 5s 为满分
    const errorScore = Math.min(errorRate / 0.1, 1.0); // 10% 错误率为满分

    return Math.max(latencyScore, errorScore);
  }

  // 当前建议的并发数
  get recommendedConcurrency(): number {
    const pressure = this.pressureScore;
    if (pressure < 0.3) return 10;      // 轻松：满并发
    if (pressure < 0.6) return 5;       // 中压：减半
    if (pressure < 0.8) return 2;       // 高压：最小并发
    return 1;                            // 临界：串行处理
  }

  private percentile(p: number): number {
    if (this.latencies.length === 0) return 0;
    const sorted = [...this.latencies].sort((a, b) => a - b);
    const idx = Math.floor((p / 100) * sorted.length);
    return sorted[idx];
  }
}

// 集成到 Agent 主循环
const backpressure = new AdaptiveBackpressure();

async function agentLoopWithBackpressure(task: Task): Promise<void> {
  const start = Date.now();
  try {
    await processTask(task);
    backpressure.record(Date.now() - start, false);
  } catch (err) {
    backpressure.record(Date.now() - start, true);
    throw err;
  }
}

// 动态调整 semaphore（简化示意）
setInterval(() => {
  const recommended = backpressure.recommendedConcurrency;
  console.log(`[Backpressure] 压力=${backpressure.pressureScore.toFixed(2)}, 建议并发=${recommended}`);
  // 实际中会更新 semaphore 的 permits
}, 5000);
```

---

## OpenClaw 中的真实背压

OpenClaw 的 gateway 层对 session 处理有多层背压：

```
Telegram/Discord 消息
         │
    [速率限制层]  ← 防止消息洪水
         │
    [Session 队列]  ← 每个 session 串行处理（天然背压）
         │
    [Tool 并发池]  ← 限制同时调用的工具数
         │
    [LLM 调用层]  ← 令牌桶限速（尊重 API rate limits）
         │
      Claude API
```

**每个 session 本身就是背压**：同一个 session 的消息是串行处理的，后一条消息等前一条完成。这防止了同一用户的并发混乱。

---

## 背压信号的传播

背压的关键是**信号要向上游传播**：

```typescript
// HTTP API 场景：用状态码传达背压
app.post('/agent/task', async (req, res) => {
  if (taskQueue.pressure > 0.8) {
    // 503 + Retry-After 头：告诉客户端等多久再试
    const retryAfter = Math.ceil(taskQueue.size * AVG_TASK_MS / 1000);
    res.status(503)
       .header('Retry-After', retryAfter.toString())
       .json({
         error: 'service_overloaded',
         message: '系统繁忙，请稍后重试',
         retryAfterSeconds: retryAfter,
         queueDepth: taskQueue.size,
       });
    return;
  }

  const taskId = taskQueue.enqueue(req.body);
  res.status(202).json({ taskId, status: 'queued' });
});

// WebSocket 场景：暂停消息接收
ws.on('message', async (data) => {
  if (taskQueue.pressure > 0.9) {
    ws.send(JSON.stringify({ type: 'backpressure', pauseMs: 2000 }));
    // 客户端收到后应暂停发送
  }
  // ...处理消息
});
```

---

## 实战：pi-mono 风格的流量控制器

```typescript
// 把上面的模式组合成一个完整的 FlowController
class AgentFlowController {
  private semaphore: Semaphore;
  private rateLimiter: TokenBucket;
  private taskQueue: BoundedTaskQueue<Task>;
  private adaptiveBackpressure: AdaptiveBackpressure;

  constructor(config: {
    maxConcurrency: number;
    ratePerSecond: number;
    queueSize: number;
  }) {
    this.semaphore = new Semaphore(config.maxConcurrency);
    this.rateLimiter = new TokenBucket(config.ratePerSecond * 2, config.ratePerSecond);
    this.taskQueue = new BoundedTaskQueue(config.queueSize, 'drop-oldest');
    this.adaptiveBackpressure = new AdaptiveBackpressure();
  }

  async submit(task: Task): Promise<TaskResult> {
    // 1. 先放入有界队列（背压点1：队列满则丢弃）
    const accepted = this.taskQueue.enqueue(task);
    if (!accepted) throw new Error('Task rejected: queue full');

    // 2. 令牌桶限速（背压点2：等待令牌）
    await this.rateLimiter.consume();

    // 3. 信号量控制并发（背压点3：等待并发槽位）
    await this.semaphore.acquire();

    const start = Date.now();
    try {
      this.taskQueue.dequeue(); // 出队，开始处理
      const result = await processTask(task);
      this.adaptiveBackpressure.record(Date.now() - start, false);
      return result;
    } catch (err) {
      this.adaptiveBackpressure.record(Date.now() - start, true);
      throw err;
    } finally {
      this.semaphore.release();
    }
  }

  get status() {
    return {
      queueDepth: this.taskQueue.size,
      pressure: this.taskQueue.pressure,
      adaptivePressure: this.adaptiveBackpressure.pressureScore,
      recommendedConcurrency: this.adaptiveBackpressure.recommendedConcurrency,
    };
  }
}

// 使用
const controller = new AgentFlowController({
  maxConcurrency: 10,
  ratePerSecond: 5,
  queueSize: 100,
});

// 暴露状态给监控
app.get('/health', (req, res) => {
  const status = controller.status;
  const httpStatus = status.pressure > 0.9 ? 503 : 200;
  res.status(httpStatus).json(status);
});
```

---

## 关键原则

| 原则 | 说明 |
|------|------|
| **失败快于失败慢** | 拒绝任务比无限等待好；给用户明确的错误比假装处理好 |
| **背压要向上游传播** | 不能只在内部消化，要让调用方知道慢下来 |
| **优先保护核心路径** | 非核心任务先丢弃，保证主流程可用 |
| **监控队列深度** | 队列长度是最直接的背压信号，必须暴露为指标 |
| **避免重试风暴** | 上游重试 + 系统过载 = 指数级恶化；重试必须有退避 |

---

## 总结

```
没有背压的 Agent：      有背压的 Agent：
  流量峰值               流量峰值
      │                      │
  任务堆积              [有界队列] ← 超出丢弃/拒绝
      │                      │
  全部超时            [令牌桶限速] ← 平滑突发
      │                      │
  系统崩溃           [信号量并发] ← 保护下游
      │                      │
  用户流失          优雅降级，用户收到明确反馈
```

背压不是「拒绝用户」，而是「诚实地告诉用户现在的处理能力」——这比假装能处理、最终全部超时要好得多。

---

**下一节预告**：Agent BDI 架构 — 用 Belief-Desire-Intention 模型让 Agent 行为更可解释
