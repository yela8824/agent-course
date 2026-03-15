# 59 - Agent Middleware Patterns（Agent 中间件模式）

> 给工具调用加装"拦截器"，实现日志、鉴权、限流、重试的统一管理

---

## 核心思想

Agent 每次调用工具，都要经过一堆横切逻辑：

- 打日志
- 鉴权检查
- 参数校验
- 限流
- 失败重试
- 耗时统计

如果每个工具函数里都写一遍，代码就烂掉了。**中间件模式**的解法：把这些横切关注点抽成独立的中间件层，工具函数只管业务逻辑。

```
Request → [Logger] → [Auth] → [RateLimit] → [Retry] → Tool → Response
                                                              ↓
Response ← [Logger] ← [Auth] ← [RateLimit] ← [Retry] ←──────┘
```

---

## TypeScript 实现（参考 pi-mono 架构）

### 基础类型定义

```typescript
type ToolHandler<T = unknown> = (params: T) => Promise<unknown>;

type Middleware<T = unknown> = (
  params: T,
  next: ToolHandler<T>
) => Promise<unknown>;

// 组合中间件的 compose 函数
function compose<T>(...middlewares: Middleware<T>[]): Middleware<T> {
  return (params, finalHandler) => {
    const dispatch = (i: number): Promise<unknown> => {
      if (i === middlewares.length) return finalHandler(params);
      return middlewares[i](params, () => dispatch(i + 1));
    };
    return dispatch(0);
  };
}
```

### 日志中间件

```typescript
const loggerMiddleware: Middleware = async (params, next) => {
  const start = Date.now();
  const toolName = (params as any).__tool ?? 'unknown';
  
  console.log(`[Tool:${toolName}] →`, JSON.stringify(params));
  
  try {
    const result = await next(params);
    console.log(`[Tool:${toolName}] ✓ ${Date.now() - start}ms`);
    return result;
  } catch (err) {
    console.error(`[Tool:${toolName}] ✗ ${Date.now() - start}ms`, err);
    throw err;
  }
};
```

### 鉴权中间件

```typescript
const authMiddleware = (allowedTools: Set<string>): Middleware => {
  return async (params, next) => {
    const toolName = (params as any).__tool ?? '';
    
    if (!allowedTools.has(toolName)) {
      throw new Error(`Tool "${toolName}" not permitted for current session`);
    }
    
    return next(params);
  };
};
```

### 限流中间件（令牌桶）

```typescript
class TokenBucket {
  private tokens: number;
  private lastRefill = Date.now();
  
  constructor(
    private capacity: number,
    private refillRate: number // tokens per second
  ) {
    this.tokens = capacity;
  }
  
  consume(n = 1): boolean {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(this.capacity, this.tokens + elapsed * this.refillRate);
    this.lastRefill = now;
    
    if (this.tokens >= n) {
      this.tokens -= n;
      return true;
    }
    return false;
  }
}

const rateLimitMiddleware = (bucket: TokenBucket): Middleware => {
  return async (params, next) => {
    if (!bucket.consume()) {
      throw new Error('Rate limit exceeded, try again later');
    }
    return next(params);
  };
};
```

### 重试中间件（带指数退避）

```typescript
const retryMiddleware = (maxRetries = 3, baseDelayMs = 500): Middleware => {
  return async (params, next) => {
    let lastError: Error;
    
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        return await next(params);
      } catch (err) {
        lastError = err as Error;
        
        // 不可重试的错误直接抛
        if (err instanceof AuthError || err instanceof ValidationError) throw err;
        
        if (attempt < maxRetries) {
          const delay = baseDelayMs * Math.pow(2, attempt) * (0.5 + Math.random() * 0.5);
          console.warn(`[Retry] attempt ${attempt + 1}/${maxRetries}, delay ${delay.toFixed(0)}ms`);
          await new Promise(r => setTimeout(r, delay));
        }
      }
    }
    
    throw lastError!;
  };
};
```

---

## 组装：Tool Registry with Middleware

```typescript
class ToolRegistry {
  private handlers = new Map<string, ToolHandler>();
  private middleware: Middleware[] = [];

  use(mw: Middleware) {
    this.middleware.push(mw);
    return this; // 链式调用
  }

  register(name: string, handler: ToolHandler) {
    this.handlers.set(name, handler);
    return this;
  }

  async call(name: string, params: unknown): Promise<unknown> {
    const handler = this.handlers.get(name);
    if (!handler) throw new Error(`Unknown tool: ${name}`);

    const enrichedParams = { ...params as object, __tool: name };
    const pipeline = compose(...this.middleware);
    
    return pipeline(enrichedParams, handler);
  }
}

// 使用
const registry = new ToolRegistry()
  .use(loggerMiddleware)
  .use(authMiddleware(new Set(['read_file', 'web_search'])))
  .use(rateLimitMiddleware(new TokenBucket(10, 1)))
  .use(retryMiddleware(3));

registry.register('read_file', async (params) => {
  // 纯业务逻辑，不需要关心日志/鉴权/限流
  return fs.readFile((params as any).path, 'utf8');
});

registry.register('web_search', async (params) => {
  return fetchSearch((params as any).query);
});
```

---

## Python 实现（参考 learn-claude-code 风格）

```python
from typing import Callable, Any, Awaitable
from functools import wraps
import time, asyncio, logging

# 类型别名
Handler = Callable[[dict], Awaitable[Any]]
Middleware = Callable[[dict, Handler], Awaitable[Any]]

def compose(*middlewares: Middleware) -> Middleware:
    """将多个中间件组合成一个"""
    async def composed(params: dict, final: Handler) -> Any:
        async def dispatch(i: int) -> Any:
            if i == len(middlewares):
                return await final(params)
            return await middlewares[i](params, lambda p: dispatch(i + 1))
        return await dispatch(0)
    return composed

# 装饰器风格的中间件
def with_logging(handler: Handler) -> Handler:
    @wraps(handler)
    async def wrapper(params: dict) -> Any:
        tool = params.get("__tool", "unknown")
        start = time.time()
        try:
            result = await handler(params)
            logging.info(f"[{tool}] OK {(time.time()-start)*1000:.0f}ms")
            return result
        except Exception as e:
            logging.error(f"[{tool}] ERR {e}")
            raise
    return wrapper

def with_retry(max_retries=3, base_delay=0.5):
    def decorator(handler: Handler) -> Handler:
        @wraps(handler)
        async def wrapper(params: dict) -> Any:
            for attempt in range(max_retries + 1):
                try:
                    return await handler(params)
                except Exception as e:
                    if attempt == max_retries:
                        raise
                    delay = base_delay * (2 ** attempt)
                    await asyncio.sleep(delay)
        return wrapper
    return decorator
```

---

## OpenClaw 实战：工具策略管道

OpenClaw 的 `tool-policy-pipeline`（第24课）本质上就是中间件。来看它和本课的关联：

```
工具策略管道（Policy Pipeline）= 中间件在安全维度的特化版本
```

```typescript
// OpenClaw 内部实际使用类似模式
// skills/xxx/SKILL.md → 加载后注入 context middleware
// 工具调用 → [policy check] → [rate limit] → [exec] → [log]

// 你的 Agent 可以做同样的事
const agentToolPipeline = new ToolRegistry()
  .use(policyMiddleware(userContext))  // 策略检查
  .use(costTrackingMiddleware())       // 成本追踪
  .use(contextEnrichMiddleware(ctx))   // 注入上下文
  .use(retryMiddleware(2));
```

---

## 何时用中间件？

| 场景 | 推荐方式 |
|------|---------|
| 所有工具都要日志 | Logger 中间件 |
| 按 session 限流 | RateLimit 中间件 |
| 工具调用失败重试 | Retry 中间件 |
| 工具级别权限控制 | Auth 中间件 |
| 单个工具的特殊逻辑 | 直接写在 handler 里 |
| 非常复杂的业务规则 | 独立 Policy Engine |

---

## 关键原则

1. **中间件要无状态**（或状态外置）——方便测试和复用
2. **错误分类**——有些错误不应重试（鉴权失败、参数错误），有些应该（网络超时）
3. **顺序很重要**——Logger 放最外层，Auth 在 RateLimit 之前（节省配额）
4. **`next` 只调用一次**——否则工具会被执行多次

---

## 总结

中间件模式把"工具该怎么被调用"和"工具做什么"彻底分离。Agent 项目一旦工具数量超过10个，这个模式就值得引入。你的工具函数变得干净，横切逻辑变得可测试、可复用。
