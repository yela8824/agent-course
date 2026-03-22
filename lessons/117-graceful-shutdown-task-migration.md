# 117 - Agent 优雅关机与任务迁移（Graceful Shutdown & Task Migration）

> 系统会关机，容器会被杀，但任务不能丢。

---

## 为什么需要优雅关机？

在生产环境，Agent 经常运行在 Kubernetes Pod、Docker 容器或 PM2 进程里。
当发生 **滚动更新、缩容、节点驱逐** 时，操作系统会发一个 `SIGTERM` 信号，
给你 30 秒（K8s 默认）把事情收拾好，然后强制 `SIGKILL`。

如果 Agent 不处理这个信号，后果就是：
- 🔴 正在执行的工具调用被强行截断
- 🔴 LLM 流式响应半截消失
- 🔴 用户看到莫名其妙的错误
- 🔴 任务状态不一致，重启后无法恢复

优雅关机的目标：**在被强杀前，安全地完成或迁移所有进行中的任务**。

---

## 三步优雅关机协议

```
SIGTERM 收到
    │
    ▼
1. 停止接受新任务（close intake）
    │
    ▼
2. 等待 in-flight 任务完成 / 超时后 checkpoint（drain & checkpoint）
    │
    ▼
3. 通知调度器/队列：未完成任务重新入队（migrate）
    │
    ▼
进程退出（exit 0）
```

---

## 代码实现

### Python 版（learn-claude-code 风格）

```python
import asyncio
import signal
import json
import redis.asyncio as redis
from typing import Optional

class GracefulAgentRunner:
    """支持优雅关机的 Agent 执行器"""
    
    def __init__(self, agent_id: str, shutdown_timeout: float = 25.0):
        self.agent_id = agent_id
        self.shutdown_timeout = shutdown_timeout
        self._is_shutting_down = False
        self._active_tasks: dict[str, asyncio.Task] = {}
        self._shutdown_event = asyncio.Event()
        self.redis = None
    
    async def start(self):
        self.redis = await redis.from_url("redis://localhost:6379")
        
        # 注册信号处理器
        loop = asyncio.get_event_loop()
        for sig in (signal.SIGTERM, signal.SIGINT):
            loop.add_signal_handler(sig, self._handle_shutdown_signal)
        
        print(f"[{self.agent_id}] Agent started, waiting for tasks...")
        await self._main_loop()
    
    def _handle_shutdown_signal(self):
        print(f"[{self.agent_id}] ⚠️  SIGTERM received, initiating graceful shutdown...")
        self._is_shutting_down = True
        self._shutdown_event.set()
    
    async def _main_loop(self):
        """主循环：从队列取任务执行"""
        while not self._is_shutting_down:
            try:
                # 非阻塞取任务（1秒超时，以便检查 shutdown 标志）
                task_data = await self.redis.blpop("agent:task_queue", timeout=1)
                if task_data:
                    _, payload = task_data
                    task_info = json.loads(payload)
                    asyncio.create_task(self._run_task(task_info))
            except asyncio.CancelledError:
                break
        
        # 进入关机流程
        await self._graceful_shutdown()
    
    async def _run_task(self, task_info: dict):
        task_id = task_info["id"]
        
        # 登记为活跃任务
        self._active_tasks[task_id] = asyncio.current_task()
        
        # 写入 checkpoint：任务开始，记录执行者
        await self.redis.hset(f"task:{task_id}", mapping={
            "status": "running",
            "agent_id": self.agent_id,
            "started_at": str(asyncio.get_event_loop().time()),
            "checkpoint": json.dumps({"step": 0, "context": task_info})
        })
        
        try:
            print(f"[{self.agent_id}] Running task {task_id}")
            result = await self._execute_agent_task(task_info)
            
            # 任务完成
            await self.redis.hset(f"task:{task_id}", mapping={
                "status": "completed",
                "result": json.dumps(result)
            })
        except asyncio.CancelledError:
            # 被取消时做 checkpoint
            await self._checkpoint_task(task_id, task_info)
            raise
        except Exception as e:
            await self.redis.hset(f"task:{task_id}", mapping={
                "status": "failed",
                "error": str(e)
            })
        finally:
            self._active_tasks.pop(task_id, None)
    
    async def _execute_agent_task(self, task_info: dict) -> dict:
        """实际的 Agent 执行逻辑（模拟）"""
        await asyncio.sleep(5)  # 模拟工具调用耗时
        return {"answer": f"Processed: {task_info['query']}"}
    
    async def _checkpoint_task(self, task_id: str, task_info: dict):
        """保存任务断点，以便其他实例恢复"""
        print(f"[{self.agent_id}] 💾 Checkpointing task {task_id}...")
        await self.redis.hset(f"task:{task_id}", mapping={
            "status": "checkpointed",
            "agent_id": "",  # 清空，允许其他实例认领
        })
        # 重新放回队列（优先级队列放队首）
        await self.redis.lpush("agent:task_queue", json.dumps(task_info))
        print(f"[{self.agent_id}] ↩️  Task {task_id} re-queued for migration")
    
    async def _graceful_shutdown(self):
        """执行优雅关机：等待活跃任务，超时则 checkpoint"""
        if not self._active_tasks:
            print(f"[{self.agent_id}] ✅ No active tasks, clean shutdown")
            return
        
        print(f"[{self.agent_id}] ⏳ Draining {len(self._active_tasks)} active tasks...")
        
        try:
            # 等待所有任务完成，但有超时
            await asyncio.wait_for(
                asyncio.gather(*self._active_tasks.values(), return_exceptions=True),
                timeout=self.shutdown_timeout
            )
            print(f"[{self.agent_id}] ✅ All tasks completed gracefully")
        except asyncio.TimeoutError:
            print(f"[{self.agent_id}] ⏰ Timeout! Force-checkpointing remaining tasks...")
            # 取消剩余任务（会触发 CancelledError → checkpoint）
            for task in self._active_tasks.values():
                task.cancel()
            await asyncio.gather(*self._active_tasks.values(), return_exceptions=True)
        
        await self.redis.aclose()
        print(f"[{self.agent_id}] 👋 Shutdown complete")


# 使用
async def main():
    runner = GracefulAgentRunner(agent_id="agent-001", shutdown_timeout=25.0)
    await runner.start()

asyncio.run(main())
```

---

### TypeScript 版（pi-mono 风格）

```typescript
// graceful-agent-runner.ts
import { createClient } from 'redis';
import { EventEmitter } from 'events';

interface TaskInfo {
  id: string;
  query: string;
  checkpoint?: Record<string, unknown>;
}

class GracefulAgentRunner extends EventEmitter {
  private isShuttingDown = false;
  private activeTasks = new Map<string, AbortController>();
  private redis = createClient({ url: 'redis://localhost:6379' });

  constructor(
    private readonly agentId: string,
    private readonly shutdownTimeout = 25_000
  ) {
    super();
    this.setupSignalHandlers();
  }

  private setupSignalHandlers() {
    const shutdown = (signal: string) => {
      console.log(`[${this.agentId}] ⚠️  ${signal} received, graceful shutdown...`);
      this.isShuttingDown = true;
      this.drain().then(() => process.exit(0));
    };

    process.once('SIGTERM', () => shutdown('SIGTERM'));
    process.once('SIGINT',  () => shutdown('SIGINT'));
  }

  async start() {
    await this.redis.connect();
    console.log(`[${this.agentId}] Agent started`);
    await this.mainLoop();
  }

  private async mainLoop() {
    while (!this.isShuttingDown) {
      // BLPOP with timeout to allow shutdown check
      const result = await this.redis.blPop('agent:task_queue', 1);
      if (result) {
        const task: TaskInfo = JSON.parse(result.element);
        this.runTask(task); // fire-and-forget, tracked via activeTasks
      }
    }
  }

  private async runTask(task: TaskInfo) {
    const controller = new AbortController();
    this.activeTasks.set(task.id, controller);

    // Checkpoint: task starts
    await this.redis.hSet(`task:${task.id}`, {
      status: 'running',
      agentId: this.agentId,
      startedAt: Date.now().toString(),
    });

    try {
      const result = await this.executeWithAbort(task, controller.signal);
      await this.redis.hSet(`task:${task.id}`, {
        status: 'completed',
        result: JSON.stringify(result),
      });
    } catch (err: unknown) {
      if ((err as Error).name === 'AbortError') {
        // Migrate: re-queue for another instance
        await this.migrateTask(task);
      } else {
        await this.redis.hSet(`task:${task.id}`, {
          status: 'failed',
          error: String(err),
        });
      }
    } finally {
      this.activeTasks.delete(task.id);
    }
  }

  private async executeWithAbort(task: TaskInfo, signal: AbortSignal): Promise<unknown> {
    // 每个工具调用都检查 signal
    if (signal.aborted) throw new DOMException('Aborted', 'AbortError');
    
    // 模拟多步骤执行，中间检查 abort
    await this.checkpointStep(task.id, { step: 1 });
    await sleep(3000, signal);   // Tool call 1
    
    await this.checkpointStep(task.id, { step: 2 });
    await sleep(3000, signal);   // Tool call 2
    
    return { answer: `Processed: ${task.query}` };
  }

  private async checkpointStep(taskId: string, progress: Record<string, unknown>) {
    await this.redis.hSet(`task:${taskId}`, {
      checkpoint: JSON.stringify(progress),
    });
  }

  private async migrateTask(task: TaskInfo) {
    console.log(`[${this.agentId}] ↩️  Migrating task ${task.id}...`);
    // 放回队首（优先处理）
    await this.redis.lPush('agent:task_queue', JSON.stringify(task));
    await this.redis.hSet(`task:${task.id}`, { status: 'migrated', agentId: '' });
  }

  private async drain() {
    if (this.activeTasks.size === 0) {
      console.log(`[${this.agentId}] ✅ Clean shutdown`);
      await this.redis.quit();
      return;
    }

    console.log(`[${this.agentId}] ⏳ Draining ${this.activeTasks.size} tasks (${this.shutdownTimeout}ms)...`);

    const drainPromise = new Promise<void>(resolve => {
      const check = setInterval(() => {
        if (this.activeTasks.size === 0) {
          clearInterval(check);
          resolve();
        }
      }, 100);
    });

    const timeoutPromise = sleep(this.shutdownTimeout).then(() => {
      console.log(`[${this.agentId}] ⏰ Timeout! Aborting remaining tasks...`);
      for (const [, controller] of this.activeTasks) {
        controller.abort();
      }
    });

    await Promise.race([drainPromise, timeoutPromise]);
    // 等 abort 回调跑完
    await sleep(500);
    await this.redis.quit();
    console.log(`[${this.agentId}] 👋 Shutdown complete`);
  }
}

function sleep(ms: number, signal?: AbortSignal): Promise<void> {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(resolve, ms);
    signal?.addEventListener('abort', () => {
      clearTimeout(timer);
      reject(new DOMException('Aborted', 'AbortError'));
    });
  });
}
```

---

## OpenClaw 的实现方式

OpenClaw 作为 always-on Agent 平台，在关机时的处理策略：

```
SIGTERM
  ├── Cron jobs：标记为 "paused"，下次启动自动 recover
  ├── 进行中的 tool calls：等待当前 turn 完成（不强杀 LLM 请求）
  ├── Sub-agents：发送 cancel 信号，等待 checkpoint
  └── Sessions：序列化到磁盘，重启后恢复
```

关键设计：**OpenClaw 把每个 cron job 的状态写入持久化存储**，
所以重启后 job 会从上次 checkpoint 继续，而不是从头开始。

---

## Kubernetes 配置最佳实践

```yaml
# deployment.yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30  # 给 Agent 30秒

      containers:
        - name: agent
          lifecycle:
            preStop:
              exec:
                # 先让 Agent 从 LB 摘流，再等 SIGTERM
                command: ["/bin/sh", "-c", "sleep 5"]
          env:
            - name: SHUTDOWN_TIMEOUT
              value: "20"   # 比 terminationGracePeriod 少 5-10s
```

**时序：**
```
K8s 发 SIGTERM
  → preStop sleep 5s（从 LB 摘流，不接新请求）
  → Agent drain 20s
  → 总计 25s < terminationGracePeriodSeconds 30s
  → 进程正常退出
  → K8s 不需要 SIGKILL
```

---

## 任务迁移的幂等性

重新入队的任务可能被多个实例认领，**必须保证幂等**：

```python
async def claim_task(redis, task_id: str, agent_id: str) -> bool:
    """原子认领任务，防止多实例重复执行"""
    # SETNX：只有第一个 agent 能设成功
    claimed = await redis.set(
        f"task:{task_id}:lock",
        agent_id,
        nx=True,          # 不存在才设
        ex=300            # 5分钟过期（防止死锁）
    )
    return claimed is not None
```

---

## 核心要点

| 场景 | 策略 |
|------|------|
| 关机前任务完成 | drain + 等待 |
| 关机超时 | abort → checkpoint → re-queue |
| 重启后恢复 | 从 Redis checkpoint 读取状态 |
| 多实例竞争 | SETNX 原子认领 |
| K8s 滚动更新 | preStop hook + terminationGracePeriod |

**黄金法则：SIGTERM 不是错误，是计划内事件。把它当成一次正常的"任务暂停"来处理。**

---

## 延伸阅读

- [Kubernetes Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Node.js graceful shutdown patterns](https://snyk.io/blog/10-best-practices-to-containerize-nodejs-web-applications-with-docker/)
- 第 108 课：Agent 健康检查与自动重启
- 第 106 课：Agent 分布式锁与并发控制
