# 116 - Agent 异步工具执行与结果轮询（Async Tool Execution & Result Polling）

> 长耗时工具别让 Agent 傻等——Job ID + 轮询，才是生产级做法。

---

## 为什么需要异步工具？

同步工具调用假设工具能在几秒内返回结果。但现实中有大量"慢工具"：

| 工具类型 | 典型耗时 |
|---|---|
| 视频转码 | 30s – 10min |
| 大文件 OCR / 解析 | 10s – 2min |
| 数据库全表扫描 | 5s – 5min |
| 外部 API（支付/物流） | 2s – 60s |
| CI/CD 构建 | 1min – 20min |

如果用同步方式等待，Agent Loop 会阻塞，消耗大量 context window token，还容易超时。

**解决方案**：工具立即返回一个 `job_id`，Agent 稍后用 `poll_job(job_id)` 查询结果。

---

## 核心模式

```
Agent                    Tool Registry              Worker
  │                           │                       │
  ├──call(run_etl, params)──►│                       │
  │                           ├──enqueue(job)────────►│
  │◄──{job_id: "j-abc123"}───│                       │
  │                           │                       │ (处理中...)
  │──[其他工具调用/思考]──────│                       │
  │                           │                       │
  ├──call(poll_job, j-abc123)►│                       │
  │◄──{status: "running", progress: 40%}──────────────│
  │                           │                       │
  ├──call(poll_job, j-abc123)►│                       │
  │◄──{status: "done", result: {...}}─────────────────│
```

**关键设计原则**：
1. **立即返回 job_id**，不阻塞
2. **轮询间隔退避**：1s → 2s → 4s → 8s（避免频繁空转）
3. **超时兜底**：最大等待时间，超时后返回 partial result 或 error
4. **Agent 可在等待期间做其他事情**（并发工具调用）

---

## Python 实现（learn-claude-code 风格）

### 1. Job Registry — 任务状态存储

```python
# async_tools/job_registry.py
import asyncio
import uuid
from dataclasses import dataclass, field
from typing import Any, Optional, Callable, Awaitable
from enum import Enum
from datetime import datetime, timedelta

class JobStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    DONE = "done"
    FAILED = "failed"
    CANCELLED = "cancelled"

@dataclass
class Job:
    id: str
    tool_name: str
    params: dict
    status: JobStatus = JobStatus.PENDING
    progress: int = 0          # 0-100
    result: Any = None
    error: Optional[str] = None
    created_at: datetime = field(default_factory=datetime.utcnow)
    updated_at: datetime = field(default_factory=datetime.utcnow)
    expires_at: datetime = field(default_factory=lambda: datetime.utcnow() + timedelta(hours=1))

class JobRegistry:
    """内存 Job 存储（生产环境换成 Redis）"""
    
    def __init__(self):
        self._jobs: dict[str, Job] = {}
        self._lock = asyncio.Lock()
    
    async def create(self, tool_name: str, params: dict) -> Job:
        job = Job(
            id=f"j-{uuid.uuid4().hex[:8]}",
            tool_name=tool_name,
            params=params
        )
        async with self._lock:
            self._jobs[job.id] = job
        return job
    
    async def update(self, job_id: str, **kwargs) -> Job:
        async with self._lock:
            job = self._jobs[job_id]
            for k, v in kwargs.items():
                setattr(job, k, v)
            job.updated_at = datetime.utcnow()
            return job
    
    async def get(self, job_id: str) -> Optional[Job]:
        return self._jobs.get(job_id)

# 全局单例
registry = JobRegistry()
```

### 2. 异步工具包装器

```python
# async_tools/async_wrapper.py
import asyncio
from typing import Callable, Awaitable
from .job_registry import registry, JobStatus

class AsyncToolWrapper:
    """把任意同步/异步函数包装成 submit + poll 两个工具"""
    
    def __init__(self, name: str, fn: Callable):
        self.name = name
        self.fn = fn
    
    async def submit(self, **params) -> dict:
        """立即返回 job_id，后台执行"""
        job = await registry.create(self.name, params)
        
        # 后台启动任务，不等待
        asyncio.create_task(self._run(job.id, params))
        
        return {
            "job_id": job.id,
            "status": "pending",
            "message": f"任务已提交，使用 poll_job('{job.id}') 查询进度"
        }
    
    async def _run(self, job_id: str, params: dict):
        await registry.update(job_id, status=JobStatus.RUNNING)
        try:
            if asyncio.iscoroutinefunction(self.fn):
                result = await self.fn(**params)
            else:
                # 同步函数跑在线程池，不阻塞事件循环
                loop = asyncio.get_event_loop()
                result = await loop.run_in_executor(None, lambda: self.fn(**params))
            
            await registry.update(job_id, status=JobStatus.DONE, result=result, progress=100)
        except Exception as e:
            await registry.update(job_id, status=JobStatus.FAILED, error=str(e))


async def poll_job(job_id: str) -> dict:
    """查询任务状态 — 这是暴露给 LLM 的工具"""
    job = await registry.get(job_id)
    if not job:
        return {"error": f"Job {job_id} 不存在"}
    
    response = {
        "job_id": job.id,
        "tool": job.tool_name,
        "status": job.status.value,
        "progress": job.progress,
        "elapsed_seconds": (datetime.utcnow() - job.created_at).seconds
    }
    
    if job.status == JobStatus.DONE:
        response["result"] = job.result
    elif job.status == JobStatus.FAILED:
        response["error"] = job.error
    elif job.status == JobStatus.RUNNING:
        response["hint"] = "任务进行中，请稍后再查询"
    
    return response
```

### 3. 暴露给 LLM 的工具定义

```python
# tools/definitions.py
ASYNC_TOOLS = [
    {
        "name": "run_etl_job",
        "description": "提交 ETL 数据处理任务（耗时较长），立即返回 job_id",
        "input_schema": {
            "type": "object",
            "properties": {
                "source": {"type": "string", "description": "数据源 URL"},
                "target_table": {"type": "string"},
                "filters": {"type": "object"}
            },
            "required": ["source", "target_table"]
        }
    },
    {
        "name": "poll_job",
        "description": "查询异步任务的执行状态和结果。当任务 status=running 时，等待几秒后再次调用。",
        "input_schema": {
            "type": "object",
            "properties": {
                "job_id": {
                    "type": "string",
                    "description": "由 run_* 工具返回的 job_id，格式如 j-abc123"
                }
            },
            "required": ["job_id"]
        }
    }
]
```

---

## TypeScript 实现（pi-mono 风格）

```typescript
// tools/async-executor.ts
import { randomBytes } from 'crypto'

type JobStatus = 'pending' | 'running' | 'done' | 'failed'

interface Job<T = unknown> {
  id: string
  status: JobStatus
  progress: number
  result?: T
  error?: string
  createdAt: number
}

class AsyncJobRegistry {
  private jobs = new Map<string, Job>()

  create(toolName: string): Job {
    const job: Job = {
      id: `j-${randomBytes(4).toString('hex')}`,
      status: 'pending',
      progress: 0,
      createdAt: Date.now()
    }
    this.jobs.set(job.id, job)
    return job
  }

  update(id: string, patch: Partial<Job>): void {
    const job = this.jobs.get(id)
    if (job) Object.assign(job, patch)
  }

  get(id: string): Job | undefined {
    return this.jobs.get(id)
  }
}

export const jobRegistry = new AsyncJobRegistry()

// ── 工具包装工厂 ────────────────────────────────────────────
export function makeAsyncTool<P extends object, R>(
  name: string,
  handler: (params: P, onProgress: (pct: number) => void) => Promise<R>
) {
  return {
    name,
    async submit(params: P) {
      const job = jobRegistry.create(name)
      jobRegistry.update(job.id, { status: 'running' })

      // 不 await，让任务在后台跑
      handler(params, (pct) => jobRegistry.update(job.id, { progress: pct }))
        .then((result) => jobRegistry.update(job.id, { status: 'done', result, progress: 100 }))
        .catch((err) => jobRegistry.update(job.id, { status: 'failed', error: String(err) }))

      return {
        job_id: job.id,
        message: `已提交，调用 poll_job("${job.id}") 查询进度`
      }
    }
  }
}

// ── poll_job 工具实现 ──────────────────────────────────────
export function pollJob(jobId: string) {
  const job = jobRegistry.get(jobId)
  if (!job) return { error: `Job ${jobId} 不存在` }

  const elapsed = Math.floor((Date.now() - job.createdAt) / 1000)

  if (job.status === 'done') {
    return { job_id: jobId, status: 'done', result: job.result, elapsed_seconds: elapsed }
  }
  if (job.status === 'failed') {
    return { job_id: jobId, status: 'failed', error: job.error }
  }
  return {
    job_id: jobId,
    status: job.status,
    progress: job.progress,
    elapsed_seconds: elapsed,
    hint: '任务进行中，建议 3-5 秒后再次 poll'
  }
}
```

---

## OpenClaw 中的真实案例

OpenClaw 里的 `exec` 工具天然支持异步模式：

```typescript
// 用 yieldMs 提交后台任务，拿到 sessionId 轮询
const session = await exec({
  command: 'ffmpeg -i input.mp4 -c:v h264 output.mp4',
  background: true,
  yieldMs: 500  // 500ms 后后台化，立即返回 sessionId
})

// 稍后用 process.poll 查询结果
const result = await process({
  action: 'poll',
  sessionId: session.sessionId,
  timeout: 5000
})
```

这和我们自己实现的 job_id + poll 是同一个思路：**提交 → 拿 ID → 后续查询**。

---

## Agent 的轮询策略：指数退避

```python
# agent_loop.py 中的轮询逻辑示例
async def wait_for_job(agent, job_id: str, max_wait: int = 120) -> dict:
    """带退避的轮询 helper，在 Agent Loop 外调用"""
    delay = 1.0
    total = 0
    
    while total < max_wait:
        result = await poll_job(job_id)
        
        if result["status"] in ("done", "failed"):
            return result
        
        # 打印进度（可选）
        if "progress" in result:
            print(f"  ⏳ {job_id}: {result['progress']}% ({total}s elapsed)")
        
        await asyncio.sleep(delay)
        total += delay
        delay = min(delay * 1.5, 10)  # 最大 10s 间隔
    
    return {"job_id": job_id, "status": "timeout", "error": "超过最大等待时间"}
```

**让 LLM 自己决定何时 poll**（更灵活）：

在工具描述中写清楚：

```
poll_job: 查询异步任务状态。
- status=pending/running 时：等待 3-10 秒后再次调用
- status=done：任务完成，result 字段包含结果  
- status=failed：任务失败，error 字段说明原因
- 建议：提交任务后先做其他工作，再来 poll，避免空转
```

LLM 会自然地在 tool_result 中看到"running"，然后决定先做别的工具调用，再回来 poll。

---

## 生产级增强：Redis 持久化

```python
# 替换 JobRegistry 为 Redis 版本
import redis.asyncio as redis
import json

class RedisJobRegistry:
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
        self.ttl = 3600  # 1小时过期
    
    async def create(self, tool_name: str, params: dict) -> Job:
        job = Job(id=f"j-{uuid.uuid4().hex[:8]}", tool_name=tool_name, params=params)
        await self.redis.setex(
            f"job:{job.id}",
            self.ttl,
            json.dumps(job.__dict__, default=str)
        )
        return job
    
    async def update(self, job_id: str, **kwargs) -> Job:
        key = f"job:{job_id}"
        raw = await self.redis.get(key)
        data = json.loads(raw)
        data.update(kwargs)
        data["updated_at"] = datetime.utcnow().isoformat()
        await self.redis.setex(key, self.ttl, json.dumps(data, default=str))
    
    async def get(self, job_id: str) -> Optional[dict]:
        raw = await self.redis.get(f"job:{job_id}")
        return json.loads(raw) if raw else None
```

**优势**：
- 多进程/多实例共享任务状态
- 天然 TTL 过期清理
- 崩溃后任务状态不丢失

---

## 反模式与陷阱

| 反模式 | 问题 | 正确做法 |
|---|---|---|
| 轮询间隔固定 1s | 浪费 token，频繁空转 | 指数退避，最大 10s |
| job_id 不透传给 LLM | Agent 无法追踪多个并发任务 | 在 tool_result 中始终返回 job_id |
| 不设超时 | 任务卡死，Agent 无限等待 | max_wait + timeout 状态 |
| 任务结果不清理 | 内存/Redis 无限增长 | TTL 自动过期 |
| 同步包装异步 | `asyncio.run()` 嵌套报错 | 统一用 async/await 或 `run_in_executor` |

---

## 小结

| 场景 | 用同步工具 | 用异步工具 |
|---|---|---|
| 响应 < 3s | ✅ | 不必要 |
| 响应 3-30s | 可能阻塞 | ✅ 推荐 |
| 响应 > 30s | ❌ 必然超时 | ✅ 必须 |
| 多任务并发 | 串行 | ✅ 并发提交 |

**核心口诀**：慢工具不等结果，先拿 job_id，Agent 继续跑，空了再来 poll。

---

*下一节预告：Agent 动态限流与自适应并发控制（Dynamic Rate Limiting & Adaptive Concurrency）*
