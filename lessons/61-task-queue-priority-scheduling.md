# 61 - Agent Task Queue & Priority Scheduling

> 任务队列与优先级调度：让 Agent 在多任务环境中有条不紊地干活

---

## 为什么需要任务队列？

现实中的 Agent 系统很少只有一个用户、一条消息：

- 100 个用户同时发请求
- 一个任务执行到一半，来了更紧急的事
- 后台 Cron 任务和前台交互任务争抢资源
- 某些任务需要等待依赖完成才能开始

**没有任务队列的 Agent = 收银台没有排队系统的超市**：先到的不一定先服务，贵宾也得等，收银员累死还是乱的。

---

## 核心概念

```
Task Queue 任务队列
├── Priority Queue（优先级队列）：VIP 先上
├── FIFO Queue（先进先出）：公平排队
├── Delay Queue（延迟队列）：定时任务
└── Dead Letter Queue（死信队列）：失败任务兜底
```

每个 Task 的基本结构：

```typescript
interface AgentTask {
  id: string;
  priority: number;        // 数字越小越紧急（0 = 最高）
  type: string;            // 'user_message' | 'cron_job' | 'background'
  payload: unknown;
  createdAt: number;
  deadline?: number;       // 截止时间（超时自动失败）
  retries: number;         // 已重试次数
  maxRetries: number;
  dependsOn?: string[];    // 依赖的其他 task ID
}
```

---

## 实战代码

### 1. 最简优先级队列（TypeScript）

```typescript
// 基于最小堆的优先级队列
class PriorityQueue<T extends { priority: number; createdAt: number }> {
  private heap: T[] = [];

  push(task: T) {
    this.heap.push(task);
    this._bubbleUp(this.heap.length - 1);
  }

  pop(): T | undefined {
    if (this.heap.length === 0) return undefined;
    const top = this.heap[0];
    const last = this.heap.pop()!;
    if (this.heap.length > 0) {
      this.heap[0] = last;
      this._sinkDown(0);
    }
    return top;
  }

  peek(): T | undefined {
    return this.heap[0];
  }

  get size() { return this.heap.length; }

  private _compare(a: T, b: T): boolean {
    // 优先级小的先执行；同优先级按时间先后
    if (a.priority !== b.priority) return a.priority < b.priority;
    return a.createdAt < b.createdAt;
  }

  private _bubbleUp(i: number) {
    while (i > 0) {
      const parent = Math.floor((i - 1) / 2);
      if (this._compare(this.heap[i], this.heap[parent])) {
        [this.heap[i], this.heap[parent]] = [this.heap[parent], this.heap[i]];
        i = parent;
      } else break;
    }
  }

  private _sinkDown(i: number) {
    const n = this.heap.length;
    while (true) {
      let smallest = i;
      const l = 2 * i + 1, r = 2 * i + 2;
      if (l < n && this._compare(this.heap[l], this.heap[smallest])) smallest = l;
      if (r < n && this._compare(this.heap[r], this.heap[smallest])) smallest = r;
      if (smallest === i) break;
      [this.heap[i], this.heap[smallest]] = [this.heap[smallest], this.heap[i]];
      i = smallest;
    }
  }
}
```

### 2. Agent Task Scheduler

```typescript
const PRIORITY = {
  CRITICAL: 0,    // 用户直接交互、紧急告警
  HIGH: 1,        // 普通用户请求
  NORMAL: 2,      // 后台处理任务
  LOW: 3,         // Cron 定时任务
  IDLE: 4,        // 空闲时才跑的任务
} as const;

class AgentTaskScheduler {
  private queue = new PriorityQueue<AgentTask>();
  private running = new Map<string, Promise<void>>();
  private maxConcurrent: number;

  constructor(maxConcurrent = 3) {
    this.maxConcurrent = maxConcurrent;
  }

  enqueue(task: Omit<AgentTask, 'id' | 'createdAt' | 'retries'>) {
    const fullTask: AgentTask = {
      ...task,
      id: crypto.randomUUID(),
      createdAt: Date.now(),
      retries: 0,
    };
    this.queue.push(fullTask);
    console.log(`[Queue] Enqueued ${fullTask.type} (priority=${fullTask.priority}), queue size=${this.queue.size}`);
    this._tick();
    return fullTask.id;
  }

  private async _tick() {
    // 有空闲 worker 就继续取任务
    while (this.running.size < this.maxConcurrent && this.queue.size > 0) {
      const task = this.queue.pop()!;

      // 检查依赖是否完成
      if (task.dependsOn?.some(dep => this.running.has(dep))) {
        // 依赖还在跑，放回队列（降优先级避免 busy-wait）
        this.queue.push({ ...task, priority: task.priority + 0.5 });
        break;
      }

      // 检查是否超过 deadline
      if (task.deadline && Date.now() > task.deadline) {
        console.warn(`[Queue] Task ${task.id} expired, skipping`);
        continue;
      }

      const promise = this._execute(task).finally(() => {
        this.running.delete(task.id);
        this._tick(); // 完成一个，继续取下一个
      });

      this.running.set(task.id, promise);
    }
  }

  private async _execute(task: AgentTask) {
    console.log(`[Queue] Starting ${task.type} (${task.id})`);
    try {
      await this._runTask(task);
    } catch (err) {
      if (task.retries < task.maxRetries) {
        console.warn(`[Queue] Retry ${task.retries + 1}/${task.maxRetries} for ${task.id}`);
        this.queue.push({ ...task, retries: task.retries + 1, priority: task.priority + 1 });
      } else {
        console.error(`[Queue] Task ${task.id} failed permanently`, err);
        await this._deadLetter(task, err as Error);
      }
    }
  }

  private async _runTask(task: AgentTask) {
    // 根据 type 分发给不同处理器
    const handler = taskHandlers.get(task.type);
    if (!handler) throw new Error(`Unknown task type: ${task.type}`);
    await handler(task.payload);
  }

  private async _deadLetter(task: AgentTask, error: Error) {
    // 写入死信队列（数据库/文件/告警）
    await appendFile('dead-letter.jsonl', JSON.stringify({ task, error: error.message, ts: Date.now() }) + '\n');
  }
}
```

### 3. OpenClaw 里的实际应用

OpenClaw 的 Cron 系统本质上就是一个任务调度器。它区分了两种 payload：

```typescript
// cron job 配置（来自 OpenClaw 源码逻辑）
{
  schedule: { kind: "every", everyMs: 3 * 60 * 60 * 1000 },  // 每3小时
  payload: { 
    kind: "agentTurn",    // 低优先级后台任务
    message: "Agent 开发课程时间！..." 
  },
  sessionTarget: "isolated"  // 隔离 session，不阻塞主 session
}

// vs 用户直接消息 → CRITICAL priority，立即响应
```

**关键设计**：`isolated` session 就是给低优先级任务用的独立 worker，不影响主 session 的响应速度。

### 4. Python 版（learn-claude-code 风格）

```python
import asyncio
import heapq
from dataclasses import dataclass, field
from typing import Any, Callable, Awaitable
from uuid import uuid4

@dataclass(order=True)
class Task:
    priority: int
    created_at: float
    id: str = field(compare=False)
    type: str = field(compare=False)
    payload: Any = field(compare=False)
    retries: int = field(compare=False, default=0)
    max_retries: int = field(compare=False, default=3)

class AsyncPriorityScheduler:
    def __init__(self, max_concurrent: int = 3):
        self._heap: list[Task] = []
        self._semaphore = asyncio.Semaphore(max_concurrent)
        self._handlers: dict[str, Callable[[Any], Awaitable[None]]] = {}

    def register(self, task_type: str):
        """装饰器：注册任务处理器"""
        def decorator(fn: Callable[[Any], Awaitable[None]]):
            self._handlers[task_type] = fn
            return fn
        return decorator

    def enqueue(self, task_type: str, payload: Any, priority: int = 2):
        import time
        task = Task(
            priority=priority,
            created_at=time.time(),
            id=str(uuid4()),
            type=task_type,
            payload=payload,
        )
        heapq.heappush(self._heap, task)
        asyncio.create_task(self._tick())
        return task.id

    async def _tick(self):
        async with self._semaphore:
            if not self._heap:
                return
            task = heapq.heappop(self._heap)
            await self._execute(task)

    async def _execute(self, task: Task):
        handler = self._handlers.get(task.type)
        if not handler:
            print(f"[ERROR] No handler for {task.type}")
            return
        try:
            await handler(task.payload)
        except Exception as e:
            if task.retries < task.max_retries:
                task.retries += 1
                task.priority += 1  # 重试降优先级
                heapq.heappush(self._heap, task)
            else:
                print(f"[DEAD] Task {task.id} failed: {e}")

# 使用示例
scheduler = AsyncPriorityScheduler(max_concurrent=5)

@scheduler.register("send_message")
async def handle_message(payload: dict):
    print(f"Sending to {payload['channel']}: {payload['text']}")

@scheduler.register("cron_lesson")
async def handle_lesson(payload: dict):
    print(f"Writing lesson: {payload['topic']}")

# 紧急用户消息
scheduler.enqueue("send_message", {"channel": "user_123", "text": "Hello"}, priority=0)

# 低优先级 Cron 任务
scheduler.enqueue("cron_lesson", {"topic": "task-queue"}, priority=3)
```

---

## 实用调度策略

### Aging（防饿死）

低优先级任务等太久可能永远轮不到（被高优先级任务压死）。解决方法：

```typescript
// 每隔一段时间，提升所有等待任务的优先级
setInterval(() => {
  const now = Date.now();
  const newHeap = queue.map(task => {
    const waitMs = now - task.createdAt;
    const ageBonus = Math.floor(waitMs / 30_000); // 每等30秒，优先级+1
    return { ...task, priority: Math.max(0, task.priority - ageBonus) };
  });
  // 重建堆
  queue.rebuildFrom(newHeap);
}, 10_000);
```

### 优先级继承

避免优先级反转（低优先级任务持有高优先级任务需要的锁）：

```typescript
// 如果高优先级任务依赖低优先级任务，临时提升低优先级任务
if (highPriorityTask.dependsOn?.includes(lowPriorityTask.id)) {
  lowPriorityTask.priority = Math.min(lowPriorityTask.priority, highPriorityTask.priority);
}
```

---

## OpenClaw 中的任务优先级实践

```
Priority 0 (CRITICAL): 用户直接发消息 → 必须秒回
Priority 1 (HIGH):     Webhook 回调、告警通知
Priority 2 (NORMAL):   普通 API 请求处理
Priority 3 (LOW):      Cron 定时任务（每3小时课程）
Priority 4 (IDLE):     日志清理、内存整理、统计汇总
```

**设计原则**：
- **Cron 任务永远是 LOW**，不能影响用户体验
- **隔离 session** 给后台任务用，主 session 保持响应
- **Deadline 必须设置**，避免过期任务占用 worker

---

## 关键指标监控

```typescript
interface SchedulerMetrics {
  queueDepth: number;           // 队列深度（高了说明 worker 不够）
  avgWaitMs: number;            // 平均等待时间
  p99WaitMs: number;            // 99分位等待时间
  taskThroughput: number;       // 每秒处理任务数
  deadLetterCount: number;      // 死信数量（高了要告警）
  workerUtilization: number;    // Worker 利用率（低了浪费，高了扩容）
}
```

---

## 总结

| 场景 | 方案 |
|------|------|
| 单机多任务 | 内存优先级队列 + Semaphore |
| 多进程/多机 | Redis / BullMQ / RabbitMQ |
| 严格顺序 | FIFO + 单 worker |
| 容错要求高 | Dead Letter Queue + 告警 |
| 防饿死 | Aging 策略 |

**一句话总结**：任务队列是 Agent 系统从"玩具"到"生产"的必经之路。没有它，你的 Agent 只能单打独斗；有了它，才能扛住真实世界的并发压力。

---

*下一课预告：Agent 知识图谱集成 — 当向量搜索不够用时，用图结构建立实体关系*
