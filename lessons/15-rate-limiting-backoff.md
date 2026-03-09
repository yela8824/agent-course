# 第 15 课：Rate Limiting & Backoff（限流与退避策略）

> Agent 与 API 的舞蹈：如何优雅地处理"你太快了"

## 为什么这很重要？

在生产环境中，Agent 会频繁调用各种 API：
- LLM 提供商（OpenAI、Anthropic、Google）
- 外部工具（GitHub、Slack、数据库）
- 内部服务（微服务、缓存）

这些 API 都有 **Rate Limit（速率限制）**。超过限制会被拒绝，甚至被封禁。

### 真实场景

```
场景：Agent 处理 100 个用户请求
问题：每个请求都立即调用 OpenAI API
结果：429 Too Many Requests → 用户体验崩溃
```

## 核心概念

### 1. Rate Limit 类型

```typescript
// 常见的限流维度
interface RateLimits {
  // 请求数限制
  requestsPerMinute: number;      // RPM
  requestsPerDay: number;         // RPD
  
  // Token 限制（LLM 特有）
  tokensPerMinute: number;        // TPM
  tokensPerDay: number;           // TPD
  
  // 并发限制
  concurrentRequests: number;
  
  // 其他
  requestsPerUser: number;        // 按用户
  requestsPerIP: number;          // 按 IP
}
```

### 2. 退避策略类型

```typescript
// 固定延迟
const fixedDelay = () => 1000; // 永远等 1 秒

// 线性退避
const linearBackoff = (attempt: number) => attempt * 1000; // 1s, 2s, 3s...

// 指数退避（最常用）
const exponentialBackoff = (attempt: number) => Math.pow(2, attempt) * 1000; // 1s, 2s, 4s, 8s...

// 指数退避 + 抖动（推荐）
const exponentialWithJitter = (attempt: number) => {
  const base = Math.pow(2, attempt) * 1000;
  const jitter = Math.random() * 1000;
  return base + jitter;
};
```

## 实战代码

### 基础重试器

```typescript
// pi-mono 风格的重试实现
interface RetryOptions {
  maxAttempts: number;
  initialDelayMs: number;
  maxDelayMs: number;
  backoffMultiplier: number;
  retryableErrors: string[];
}

async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions
): Promise<T> {
  let lastError: Error | null = null;
  
  for (let attempt = 0; attempt < options.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      
      // 检查是否可重试
      if (!isRetryable(error, options.retryableErrors)) {
        throw error;
      }
      
      // 最后一次尝试不等待
      if (attempt === options.maxAttempts - 1) {
        break;
      }
      
      // 计算延迟
      const delay = calculateDelay(attempt, options);
      console.log(`Attempt ${attempt + 1} failed, retrying in ${delay}ms...`);
      
      await sleep(delay);
    }
  }
  
  throw lastError;
}

function calculateDelay(attempt: number, options: RetryOptions): number {
  const exponentialDelay = options.initialDelayMs * 
    Math.pow(options.backoffMultiplier, attempt);
  
  // 加入抖动防止雷群效应
  const jitter = Math.random() * 0.3 * exponentialDelay;
  
  // 限制最大延迟
  return Math.min(exponentialDelay + jitter, options.maxDelayMs);
}

function isRetryable(error: unknown, retryableErrors: string[]): boolean {
  if (error instanceof Error) {
    // HTTP 状态码
    const status = (error as any).status;
    if (status === 429 || status === 503 || status === 502) {
      return true;
    }
    
    // 错误类型
    return retryableErrors.some(e => error.message.includes(e));
  }
  return false;
}
```

### OpenClaw 中的实现

```typescript
// OpenClaw 的 API 调用包装
class RateLimitedClient {
  private requestQueue: Array<{
    fn: () => Promise<any>;
    resolve: (value: any) => void;
    reject: (error: any) => void;
  }> = [];
  
  private processing = false;
  private lastRequestTime = 0;
  private minIntervalMs: number;
  
  constructor(requestsPerMinute: number) {
    this.minIntervalMs = 60000 / requestsPerMinute;
  }
  
  async request<T>(fn: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.requestQueue.push({ fn, resolve, reject });
      this.processQueue();
    });
  }
  
  private async processQueue() {
    if (this.processing || this.requestQueue.length === 0) {
      return;
    }
    
    this.processing = true;
    
    while (this.requestQueue.length > 0) {
      const item = this.requestQueue.shift()!;
      
      // 限速
      const now = Date.now();
      const timeSinceLastRequest = now - this.lastRequestTime;
      if (timeSinceLastRequest < this.minIntervalMs) {
        await sleep(this.minIntervalMs - timeSinceLastRequest);
      }
      
      try {
        this.lastRequestTime = Date.now();
        const result = await item.fn();
        item.resolve(result);
      } catch (error) {
        item.reject(error);
      }
    }
    
    this.processing = false;
  }
}
```

### Token Bucket 算法

```typescript
// 令牌桶：更灵活的限流策略
class TokenBucket {
  private tokens: number;
  private lastRefill: number;
  
  constructor(
    private capacity: number,      // 桶容量
    private refillRate: number,    // 每秒补充 token 数
  ) {
    this.tokens = capacity;
    this.lastRefill = Date.now();
  }
  
  async acquire(count: number = 1): Promise<void> {
    this.refill();
    
    while (this.tokens < count) {
      // 计算需要等待的时间
      const needed = count - this.tokens;
      const waitMs = (needed / this.refillRate) * 1000;
      
      await sleep(Math.ceil(waitMs));
      this.refill();
    }
    
    this.tokens -= count;
  }
  
  private refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    const refillAmount = elapsed * this.refillRate;
    
    this.tokens = Math.min(this.capacity, this.tokens + refillAmount);
    this.lastRefill = now;
  }
}

// 使用示例
const bucket = new TokenBucket(100, 10); // 容量100，每秒补10个

async function callAPI() {
  await bucket.acquire(1);  // 获取1个token
  return fetch('/api/...');
}
```

## Anthropic/OpenAI 的 Rate Limit 处理

### 读取 Response Headers

```typescript
// API 响应通常包含限流信息
interface RateLimitHeaders {
  'x-ratelimit-limit-requests': string;      // 请求限制
  'x-ratelimit-limit-tokens': string;        // Token 限制
  'x-ratelimit-remaining-requests': string;  // 剩余请求
  'x-ratelimit-remaining-tokens': string;    // 剩余 Token
  'x-ratelimit-reset-requests': string;      // 重置时间
  'x-ratelimit-reset-tokens': string;        // Token 重置时间
  'retry-after': string;                     // 建议等待时间
}

async function callWithRateLimitAwareness(client: any, request: any) {
  try {
    const response = await client.messages.create(request);
    
    // 检查剩余配额
    const remaining = parseInt(
      response.headers['x-ratelimit-remaining-requests'] || '999'
    );
    
    if (remaining < 10) {
      console.warn(`Low rate limit remaining: ${remaining}`);
      // 可以触发告警或降级
    }
    
    return response;
  } catch (error: any) {
    if (error.status === 429) {
      // 使用 retry-after 头
      const retryAfter = parseInt(error.headers?.['retry-after'] || '60');
      console.log(`Rate limited, waiting ${retryAfter}s`);
      await sleep(retryAfter * 1000);
      
      // 重试
      return callWithRateLimitAwareness(client, request);
    }
    throw error;
  }
}
```

### 预估 Token 消耗

```typescript
// 在发送前预估，避免超限
import { encoding_for_model } from 'tiktoken';

class TokenEstimator {
  private encoder: any;
  
  constructor(model: string = 'gpt-4') {
    this.encoder = encoding_for_model(model);
  }
  
  estimate(text: string): number {
    return this.encoder.encode(text).length;
  }
  
  estimateMessages(messages: Array<{role: string, content: string}>): number {
    let total = 0;
    for (const msg of messages) {
      total += 4; // 每条消息的格式开销
      total += this.estimate(msg.content);
    }
    total += 2; // 回复的起始
    return total;
  }
}

// 使用
const estimator = new TokenEstimator();
const estimatedTokens = estimator.estimateMessages(messages);

if (estimatedTokens > remainingTokens) {
  // 压缩上下文或等待重置
  await compactContext(messages);
}
```

## 雷群效应（Thundering Herd）

### 问题

```
场景：服务重启后，所有 Agent 同时发请求
结果：API 瞬间过载，大量 429
```

### 解决方案：随机抖动

```typescript
// 启动时添加随机延迟
async function startWithJitter(maxJitterMs: number = 5000) {
  const jitter = Math.random() * maxJitterMs;
  await sleep(jitter);
  
  // 开始处理
  startProcessing();
}

// 重试时添加抖动
function getRetryDelay(attempt: number): number {
  const baseDelay = Math.pow(2, attempt) * 1000;
  const jitter = Math.random() * baseDelay * 0.5; // 0-50% 抖动
  return baseDelay + jitter;
}
```

## 生产环境最佳实践

### 1. 分层限流

```typescript
// 多层限流策略
class LayeredRateLimiter {
  private globalBucket: TokenBucket;    // 全局限制
  private userBuckets: Map<string, TokenBucket>;  // 用户级
  private modelBuckets: Map<string, TokenBucket>; // 模型级
  
  async acquire(userId: string, model: string): Promise<void> {
    // 按优先级检查
    await this.globalBucket.acquire(1);
    
    if (!this.userBuckets.has(userId)) {
      this.userBuckets.set(userId, new TokenBucket(10, 1));
    }
    await this.userBuckets.get(userId)!.acquire(1);
    
    if (!this.modelBuckets.has(model)) {
      this.modelBuckets.set(model, this.getModelBucket(model));
    }
    await this.modelBuckets.get(model)!.acquire(1);
  }
  
  private getModelBucket(model: string): TokenBucket {
    // 不同模型不同限制
    const limits: Record<string, [number, number]> = {
      'claude-3-opus': [20, 1],     // 容量20，每秒1个
      'claude-3-sonnet': [60, 5],   // 容量60，每秒5个
      'claude-3-haiku': [100, 10],  // 容量100，每秒10个
    };
    const [cap, rate] = limits[model] || [50, 5];
    return new TokenBucket(cap, rate);
  }
}
```

### 2. 优雅降级

```typescript
// 超限时的降级策略
async function callWithFallback(request: AgentRequest): Promise<Response> {
  try {
    // 首选：Opus
    return await callModel('claude-3-opus', request);
  } catch (error: any) {
    if (error.status === 429) {
      console.log('Opus rate limited, falling back to Sonnet');
      
      try {
        // 降级：Sonnet
        return await callModel('claude-3-sonnet', request);
      } catch (error2: any) {
        if (error2.status === 429) {
          console.log('Sonnet rate limited, falling back to Haiku');
          
          // 再降级：Haiku
          return await callModel('claude-3-haiku', request);
        }
        throw error2;
      }
    }
    throw error;
  }
}
```

### 3. 监控与告警

```typescript
// 限流监控
class RateLimitMonitor {
  private metrics = {
    totalRequests: 0,
    rateLimitedRequests: 0,
    totalRetries: 0,
    avgRetryDelay: 0,
  };
  
  recordRequest() {
    this.metrics.totalRequests++;
  }
  
  recordRateLimited() {
    this.metrics.rateLimitedRequests++;
    
    const ratio = this.metrics.rateLimitedRequests / this.metrics.totalRequests;
    if (ratio > 0.1) {
      this.alert('High rate limit ratio: ' + (ratio * 100).toFixed(1) + '%');
    }
  }
  
  recordRetry(delayMs: number) {
    this.metrics.totalRetries++;
    this.metrics.avgRetryDelay = 
      (this.metrics.avgRetryDelay * (this.metrics.totalRetries - 1) + delayMs) /
      this.metrics.totalRetries;
  }
  
  private alert(message: string) {
    // 发送告警
    console.error(`[RATE LIMIT ALERT] ${message}`);
  }
  
  getMetrics() {
    return { ...this.metrics };
  }
}
```

## 小结

| 策略 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| 固定延迟 | 简单场景 | 实现简单 | 不灵活 |
| 指数退避 | 通用场景 | 平衡效率和保护 | 恢复可能慢 |
| 指数+抖动 | 高并发 | 避免雷群 | 稍复杂 |
| 令牌桶 | 需要突发 | 允许短时突发 | 内存开销 |
| 滑动窗口 | 精确控制 | 平滑限流 | 实现复杂 |

## 核心要点

1. **永远用指数退避 + 抖动**：简单有效
2. **读取 API 的 Rate Limit Headers**：根据实际情况调整
3. **预估 Token 消耗**：避免超限才重试
4. **分层限流**：全局、用户、模型多维度
5. **优雅降级**：高端模型超限时降级到低端
6. **监控告警**：及时发现问题

---

下节预告：**Caching Strategies（缓存策略）** - 如何减少重复 API 调用，提升性能降低成本。
