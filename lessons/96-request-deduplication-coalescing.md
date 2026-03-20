# 96 - Agent 工具调用去重与请求合并（Request Deduplication & Coalescing）

> 同一时刻，10 个 Sub-agent 都想查同一个用户的余额。你会发 10 次 DB 请求吗？

---

## 问题背景

在多 Agent 并发系统里，经常出现这种场景：

```
Sub-agent A: get_user(userId=42)  ─┐
Sub-agent B: get_user(userId=42)  ─┤── 同时发出 4 个相同请求
Sub-agent C: get_user(userId=42)  ─┤
Sub-agent D: get_user(userId=42)  ─┘
```

四个请求打到 DB/API，返回完全相同的数据。这是纯粹的浪费。

**两个优化手段：**

| 技术 | 场景 | 效果 |
|------|------|------|
| **Deduplication（去重）** | 相同请求正在 inflight 时，后来者等待，共享同一结果 | 减少并发重复请求 |
| **Coalescing（合并）** | 短时间内多个单项请求 → 合并成一次批量请求 | 减少 N+1 请求 |

注意和「语义缓存」的区别：
- 语义缓存：历史请求的结果复用（跨时间）
- 去重/合并：**正在进行中**的请求共享（当前时刻）

---

## 核心模式 1：Inflight Request Deduplication

```typescript
// request-dedup.ts
class RequestDeduplicator {
  // key → Promise<result>，存的是"正在进行"的请求
  private inflight = new Map<string, Promise<any>>();

  async call<T>(
    key: string,
    fn: () => Promise<T>
  ): Promise<T> {
    // 已有相同请求在飞，直接搭便车
    if (this.inflight.has(key)) {
      console.log(`[dedup] 搭便车: ${key}`);
      return this.inflight.get(key)!;
    }

    // 第一个请求，发出去并记录
    const promise = fn().finally(() => {
      // 请求完成后清除，下次再发新请求
      this.inflight.delete(key);
    });

    this.inflight.set(key, promise);
    return promise;
  }
}

// ---- 使用 ----
const dedup = new RequestDeduplicator();

async function getUser(userId: string) {
  return dedup.call(`user:${userId}`, () =>
    db.query('SELECT * FROM users WHERE id = ?', [userId])
  );
}

// 并发 4 个相同请求 → 只发 1 次 DB 查询
const results = await Promise.all([
  getUser('42'),
  getUser('42'),
  getUser('42'),
  getUser('42'),
]);
// results[0] === results[1] === results[2] === results[3]
```

**关键点**：存的是 `Promise`，不是结果。后来者直接 `await` 同一个 Promise，天然共享。

---

## 核心模式 2：Request Coalescing（DataLoader 模式）

Facebook DataLoader 是经典实现。核心思路：把"同一个 tick"里的多个单项请求，攒成一批。

```typescript
// dataloader.ts
class DataLoader<K, V> {
  private batch: K[] = [];
  private callbacks: Map<K, { resolve: (v: V) => void; reject: (e: any) => void }[]> = new Map();
  private timer: ReturnType<typeof setTimeout> | null = null;

  constructor(
    private batchFn: (keys: K[]) => Promise<Map<K, V>>,
    private delayMs = 0  // 0 = 等下一个 microtask tick
  ) {}

  async load(key: K): Promise<V> {
    return new Promise((resolve, reject) => {
      // 加入批次
      if (!this.callbacks.has(key)) {
        this.batch.push(key);
        this.callbacks.set(key, []);
      }
      this.callbacks.get(key)!.push({ resolve, reject });

      // 调度批量执行
      if (!this.timer) {
        this.timer = setTimeout(() => this.flush(), this.delayMs);
      }
    });
  }

  private async flush() {
    const keys = [...this.batch];
    const callbacks = new Map(this.callbacks);
    
    // 清空，准备接收下一批
    this.batch = [];
    this.callbacks.clear();
    this.timer = null;

    try {
      const results = await this.batchFn(keys);
      for (const [key, cbs] of callbacks) {
        const value = results.get(key);
        if (value !== undefined) {
          cbs.forEach(cb => cb.resolve(value));
        } else {
          cbs.forEach(cb => cb.reject(new Error(`Key not found: ${key}`)));
        }
      }
    } catch (err) {
      for (const cbs of callbacks.values()) {
        cbs.forEach(cb => cb.reject(err));
      }
    }
  }
}

// ---- 使用：批量获取用户 ----
const userLoader = new DataLoader<string, User>(async (userIds) => {
  // 一次 SQL 查所有
  const users = await db.query(
    'SELECT * FROM users WHERE id IN (?)',
    [userIds]
  );
  return new Map(users.map(u => [u.id, u]));
});

// 这些调用在同一个 tick，会被合并成一次 IN 查询
const [user1, user2, user3] = await Promise.all([
  userLoader.load('1'),
  userLoader.load('2'),
  userLoader.load('3'),
]);
// 实际只发了: SELECT * FROM users WHERE id IN ('1', '2', '3')
```

---

## 在 Agent Tool 层的实现

把去重 + 合并包装成透明的工具中间件：

```typescript
// agent-tool-optimizer.ts

interface Tool {
  name: string;
  execute: (params: any) => Promise<any>;
  batchable?: boolean;  // 是否支持批量
  batchFn?: (paramsList: any[]) => Promise<any[]>;
}

class AgentToolOptimizer {
  private deduplicator = new RequestDeduplicator();
  private loaders = new Map<string, DataLoader<any, any>>();

  wrapTool(tool: Tool): Tool {
    return {
      ...tool,
      execute: async (params: any) => {
        const key = `${tool.name}:${JSON.stringify(params)}`;

        // 优先去重（相同请求合并）
        return this.deduplicator.call(key, () =>
          tool.execute(params)
        );
      }
    };
  }

  wrapBatchableTool(tool: Tool & { batchFn: (keys: string[]) => Promise<Map<string, any>> }): Tool {
    // 为可批量工具创建 DataLoader
    if (!this.loaders.has(tool.name)) {
      const loader = new DataLoader<string, any>(
        (keys) => tool.batchFn(keys)
      );
      this.loaders.set(tool.name, loader);
    }

    return {
      ...tool,
      execute: async (params: any) => {
        const loader = this.loaders.get(tool.name)!;
        // 假设 params.id 是 key
        return loader.load(params.id);
      }
    };
  }
}

// ---- 注册工具 ----
const optimizer = new AgentToolOptimizer();

const getUserTool = optimizer.wrapBatchableTool({
  name: 'get_user',
  execute: async (params) => {
    const result = await db.query('SELECT * FROM users WHERE id = ?', [params.id]);
    return result[0];
  },
  batchFn: async (ids) => {
    const users = await db.query('SELECT * FROM users WHERE id IN (?)', [ids]);
    return new Map(users.map((u: User) => [u.id, u]));
  }
});
```

---

## OpenClaw 中的实际应用

OpenClaw 的工具调用经过 `mcporter` 路由，可以在中间层加去重：

```typescript
// openclaw 工具调用链路:
// session.tool_call → tool_policy_pipeline → mcporter → external_tool

// 在 tool_policy_pipeline 中插入去重层
class DeduplicatingToolPolicy {
  private dedup = new RequestDeduplicator();

  async process(toolCall: ToolCall, next: () => Promise<ToolResult>): Promise<ToolResult> {
    // 只对幂等工具去重（GET 类操作）
    if (!toolCall.idempotent) {
      return next();
    }

    const key = `${toolCall.tool}:${stableHash(toolCall.params)}`;
    return this.dedup.call(key, next);
  }
}
```

---

## pi-mono 并发子任务场景

pi-mono 的 sub-agent fan-out 经常出现重复工具调用：

```typescript
// pi-mono/src/agent/parallel-executor.ts

// 场景：3 个 sub-agent 并发分析，都需要读同一份配置
const tasks = [
  analyzeModule('auth'),      // 内部调用 read_config('base')
  analyzeModule('billing'),   // 内部调用 read_config('base')
  analyzeModule('api'),       // 内部调用 read_config('base')
];

// 不加去重：read_config 被调 3 次
// 加去重后：read_config 只调 1 次，结果共享给 3 个 sub-agent
```

实现方式：在 pi-mono 的 `ToolRegistry` 注册工具时，包一层去重 wrapper：

```typescript
class ToolRegistry {
  private dedup = new RequestDeduplicator();

  register(tool: Tool) {
    const wrapped = tool.readonly
      ? {
          ...tool,
          call: (params: any) => {
            const key = `${tool.name}:${JSON.stringify(params)}`;
            return this.dedup.call(key, () => tool.call(params));
          }
        }
      : tool;  // 写操作不去重

    this.tools.set(tool.name, wrapped);
  }
}
```

---

## 去重 Key 的设计

去重效果完全依赖 key 的设计：

```typescript
// ❌ 太宽松：不同参数命中同一 key
const badKey = tool.name;

// ❌ 太严格：JSON 序列化顺序不稳定
const unstableKey = `${tool.name}:${JSON.stringify(params)}`;
// { a: 1, b: 2 } vs { b: 2, a: 1 } → 不同的 key！

// ✅ 稳定 hash
import { createHash } from 'crypto';

function stableHash(obj: any): string {
  // 递归排序 key 后序列化
  const sorted = sortKeys(obj);
  return createHash('sha256')
    .update(JSON.stringify(sorted))
    .digest('hex')
    .slice(0, 16);
}

function sortKeys(obj: any): any {
  if (typeof obj !== 'object' || obj === null) return obj;
  if (Array.isArray(obj)) return obj.map(sortKeys);
  return Object.keys(obj).sort().reduce((acc, k) => {
    acc[k] = sortKeys(obj[k]);
    return acc;
  }, {} as any);
}

const goodKey = `${tool.name}:${stableHash(params)}`;
```

---

## 适用场景 vs 禁用场景

| 场景 | 去重？ | 原因 |
|------|--------|------|
| GET /user/{id} | ✅ | 幂等，只读 |
| 读配置/文件 | ✅ | 幂等，只读 |
| LLM 推理调用（相同 prompt）| ✅ | 成本高，结果一样 |
| POST 创建订单 | ❌ | 非幂等，每次都要执行 |
| 发送消息/邮件 | ❌ | 副作用，重复不可接受 |
| 带时间戳的查询 | ⚠️ | key 里要包含时间粒度 |
| 随机性操作（骰子/UUID）| ❌ | 预期每次不同 |

---

## 监控指标

```typescript
class MonitoredDeduplicator extends RequestDeduplicator {
  private stats = { hits: 0, misses: 0 };

  async call<T>(key: string, fn: () => Promise<T>): Promise<T> {
    const isHit = this.inflight.has(key);
    if (isHit) this.stats.hits++;
    else this.stats.misses++;

    return super.call(key, fn);
  }

  getHitRate() {
    const total = this.stats.hits + this.stats.misses;
    return total === 0 ? 0 : this.stats.hits / total;
  }
}

// 通过 OpenTelemetry 上报
meter.createObservableGauge('tool.dedup.hit_rate', {
  callback: (obs) => obs.observe(deduplicator.getHitRate())
});
```

---

## 总结

```
用户请求
    ↓
[Tool Policy Pipeline]
    ↓
[Deduplication Layer] ← 相同请求？搭便车，等结果
    ↓
[Coalescing Layer]   ← 同批次请求？合并成批量
    ↓
[Actual Tool/API]    ← 实际只发最少的请求
```

**一句话记忆**：
- **Deduplication** = "你已经在排队了，我跟你等" （相同 key，共享 Promise）
- **Coalescing** = "等大家都到了，一起进" （不同 key，合并批量）

两者结合，可以在多 Agent 并发场景下大幅降低下游压力，不需要改任何业务逻辑。
