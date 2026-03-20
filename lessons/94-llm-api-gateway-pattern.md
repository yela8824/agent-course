# 94 - Agent LLM API 网关模式（LLM API Gateway Pattern）

> **核心思想**：生产级 Agent 系统不应该让每个模块直接调 OpenAI/Anthropic API。中间加一层 **LLM Gateway**，就能得到统一的限流、成本追踪、日志、语义缓存、故障转移——而业务代码完全不感知。

---

## 1. 为什么需要 LLM Gateway？

直接调用 LLM API 的问题：

| 问题 | 后果 |
|------|------|
| 多个 Agent 各自管 API Key | 密钥泄露风险高，轮换困难 |
| 没有统一限流 | 突发流量打爆 rate limit，所有 Agent 一起挂 |
| 没有成本追踪 | 月底账单惊喜，不知道哪个功能最贵 |
| 没有统一日志 | 排查问题要翻遍每个 Agent 的日志 |
| 每个 Agent 自己实现 retry/fallback | 代码重复，行为不一致 |

**LLM Gateway = 专门处理 LLM 调用的反向代理**，类比 API Gateway 之于 REST 微服务。

---

## 2. 核心架构

```
┌─────────────────────────────────────────────────────┐
│                   Agent / Tool                      │
│  llm.chat("你好")  ──►  GatewayClient               │
└──────────────────────────┬──────────────────────────┘
                           │ HTTP / SDK
┌──────────────────────────▼──────────────────────────┐
│                   LLM Gateway                       │
│                                                     │
│  ┌─────────────┐  ┌──────────┐  ┌───────────────┐  │
│  │  Auth &     │  │  Rate    │  │   Semantic    │  │
│  │  Key Vault  │  │  Limiter │  │   Cache       │  │
│  └──────┬──────┘  └────┬─────┘  └──────┬────────┘  │
│         └──────────────┴───────────────┘           │
│                        │                           │
│  ┌─────────────────────▼─────────────────────────┐ │
│  │            Router / Load Balancer             │ │
│  │   (模型选择 · 区域路由 · 成本优先策略)          │ │
│  └──────┬──────────────────────────────┬──────────┘ │
│         │                              │            │
│  ┌──────▼──────┐                ┌──────▼──────┐     │
│  │  Anthropic  │                │   OpenAI    │     │
│  │  Claude API │                │   GPT API   │     │
│  └─────────────┘                └─────────────┘     │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  Metrics · Cost Tracker · Audit Log         │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## 3. 代码实现

### 3.1 Gateway 核心（TypeScript / pi-mono 风格）

```typescript
// llm-gateway/src/gateway.ts

import { Redis } from 'ioredis'
import { Anthropic } from '@anthropic-ai/sdk'
import { OpenAI } from 'openai'
import { createHash } from 'crypto'

interface GatewayRequest {
  model: string
  messages: Array<{ role: string; content: string }>
  maxTokens?: number
  temperature?: number
  // 扩展字段
  tenantId: string
  priority?: 'low' | 'normal' | 'high'
  budgetTokens?: number
  tags?: string[]  // 用于成本追踪
}

interface GatewayResponse {
  content: string
  model: string          // 实际使用的模型（可能被路由到不同模型）
  inputTokens: number
  outputTokens: number
  costUsd: number
  cacheHit: boolean
  latencyMs: number
  provider: 'anthropic' | 'openai'
}

// 成本表（美元/百万token）
const TOKEN_COSTS: Record<string, { input: number; output: number }> = {
  'claude-sonnet-4-5':        { input: 3.0,  output: 15.0 },
  'claude-haiku-3-5':         { input: 0.8,  output: 4.0  },
  'gpt-4o':                   { input: 2.5,  output: 10.0 },
  'gpt-4o-mini':              { input: 0.15, output: 0.6  },
}

class LLMGateway {
  private redis: Redis
  private anthropic: Anthropic
  private openai: OpenAI
  private rateLimiters = new Map<string, TokenBucket>()

  constructor(config: {
    redisUrl: string
    anthropicKey: string
    openaiKey: string
  }) {
    this.redis = new Redis(config.redisUrl)
    this.anthropic = new Anthropic({ apiKey: config.anthropicKey })
    this.openai = new OpenAI({ apiKey: config.openaiKey })
  }

  async chat(req: GatewayRequest): Promise<GatewayResponse> {
    const start = Date.now()

    // 1. 权限检查
    await this.checkAuth(req.tenantId)

    // 2. 限流
    await this.rateLimit(req.tenantId, req.priority ?? 'normal')

    // 3. 语义缓存检查
    const cacheKey = this.buildCacheKey(req)
    const cached = await this.getCache(cacheKey)
    if (cached) {
      return { ...cached, cacheHit: true, latencyMs: Date.now() - start }
    }

    // 4. 路由：选择最合适的模型和 provider
    const route = this.route(req)

    // 5. 实际调用
    const raw = await this.callProvider(route, req)

    // 6. 计算成本
    const costs = TOKEN_COSTS[route.model] ?? { input: 3.0, output: 15.0 }
    const costUsd = (raw.inputTokens * costs.input + raw.outputTokens * costs.output) / 1_000_000

    const response: GatewayResponse = {
      ...raw,
      model: route.model,
      costUsd,
      cacheHit: false,
      latencyMs: Date.now() - start,
    }

    // 7. 写缓存（仅对 temperature=0 的确定性请求）
    if ((req.temperature ?? 1.0) === 0) {
      await this.setCache(cacheKey, response, 3600) // 1小时
    }

    // 8. 记录指标
    await this.recordMetrics(req, response)

    return response
  }

  // ── 路由策略 ──────────────────────────────────────

  private route(req: GatewayRequest): { provider: 'anthropic' | 'openai'; model: string } {
    // 简单路由规则：按模型名前缀决定 provider
    if (req.model.startsWith('claude')) {
      return { provider: 'anthropic', model: req.model }
    }
    if (req.model.startsWith('gpt') || req.model.startsWith('o1')) {
      return { provider: 'openai', model: req.model }
    }

    // 成本优先路由：如果 budget 紧张，降级到便宜模型
    if (req.budgetTokens && req.budgetTokens < 1000) {
      return { provider: 'anthropic', model: 'claude-haiku-3-5' }
    }

    // 默认
    return { provider: 'anthropic', model: 'claude-sonnet-4-5' }
  }

  // ── Provider 调用 ─────────────────────────────────

  private async callProvider(
    route: { provider: string; model: string },
    req: GatewayRequest
  ): Promise<Omit<GatewayResponse, 'model' | 'costUsd' | 'cacheHit' | 'latencyMs'>> {
    if (route.provider === 'anthropic') {
      const resp = await this.anthropic.messages.create({
        model: route.model,
        max_tokens: req.maxTokens ?? 4096,
        messages: req.messages as any,
      })
      return {
        content: resp.content[0].type === 'text' ? resp.content[0].text : '',
        inputTokens: resp.usage.input_tokens,
        outputTokens: resp.usage.output_tokens,
        provider: 'anthropic',
      }
    } else {
      const resp = await this.openai.chat.completions.create({
        model: route.model,
        max_tokens: req.maxTokens ?? 4096,
        messages: req.messages as any,
        temperature: req.temperature,
      })
      return {
        content: resp.choices[0].message.content ?? '',
        inputTokens: resp.usage?.prompt_tokens ?? 0,
        outputTokens: resp.usage?.completion_tokens ?? 0,
        provider: 'openai',
      }
    }
  }

  // ── 限流 (Token Bucket) ───────────────────────────

  private async rateLimit(tenantId: string, priority: string): Promise<void> {
    const key = `rl:${tenantId}`
    const limit = priority === 'high' ? 100 : priority === 'normal' ? 60 : 20 // RPM
    
    const count = await this.redis.incr(key)
    if (count === 1) await this.redis.expire(key, 60) // 1分钟窗口
    
    if (count > limit) {
      const ttl = await this.redis.ttl(key)
      throw new RateLimitError(`Rate limit exceeded. Reset in ${ttl}s`, ttl)
    }
  }

  // ── 缓存 ──────────────────────────────────────────

  private buildCacheKey(req: GatewayRequest): string {
    const payload = JSON.stringify({
      model: req.model,
      messages: req.messages,
      maxTokens: req.maxTokens,
    })
    return `llm:cache:${createHash('sha256').update(payload).digest('hex').slice(0, 16)}`
  }

  private async getCache(key: string): Promise<GatewayResponse | null> {
    const data = await this.redis.get(key)
    return data ? JSON.parse(data) : null
  }

  private async setCache(key: string, resp: GatewayResponse, ttlSec: number): Promise<void> {
    await this.redis.setex(key, ttlSec, JSON.stringify(resp))
  }

  // ── 指标 ──────────────────────────────────────────

  private async recordMetrics(req: GatewayRequest, resp: GatewayResponse): Promise<void> {
    const pipe = this.redis.pipeline()
    const day = new Date().toISOString().slice(0, 10)

    // 按租户统计今日成本
    pipe.incrbyfloat(`cost:${req.tenantId}:${day}`, resp.costUsd)
    // 按模型统计 token 用量
    pipe.incrby(`tokens:${resp.model}:input`,  resp.inputTokens)
    pipe.incrby(`tokens:${resp.model}:output`, resp.outputTokens)
    // 按 tag 统计（用于功能级成本分析）
    for (const tag of req.tags ?? []) {
      pipe.incrbyfloat(`cost:tag:${tag}:${day}`, resp.costUsd)
    }
    await pipe.exec()
  }

  private async checkAuth(tenantId: string): Promise<void> {
    const valid = await this.redis.sismember('tenants:active', tenantId)
    if (!valid) throw new AuthError(`Unknown tenant: ${tenantId}`)
  }
}

class RateLimitError extends Error {
  constructor(message: string, public retryAfter: number) { super(message) }
}
class AuthError extends Error {}

export { LLMGateway, GatewayRequest, GatewayResponse }
```

### 3.2 Agent 侧的客户端封装

```typescript
// llm-gateway/src/client.ts
// Agent 不直接用 Anthropic SDK，改用这个

import axios from 'axios'
import { GatewayRequest, GatewayResponse } from './gateway'

class GatewayClient {
  private http = axios.create({
    baseURL: process.env.LLM_GATEWAY_URL ?? 'http://localhost:4000',
    headers: { 'X-Tenant-ID': process.env.TENANT_ID ?? 'default' },
    timeout: 60_000,
  })

  async chat(messages: Array<{ role: string; content: string }>, opts: {
    model?: string
    temperature?: number
    tags?: string[]
  } = {}): Promise<string> {
    const req: Partial<GatewayRequest> = {
      model: opts.model ?? 'claude-sonnet-4-5',
      messages,
      temperature: opts.temperature,
      tags: opts.tags,
      tenantId: process.env.TENANT_ID ?? 'default',
    }

    const { data } = await this.http.post<GatewayResponse>('/chat', req)

    // 打印成本（开发调试用）
    if (process.env.DEBUG_COST) {
      console.log(`[Gateway] $${data.costUsd.toFixed(6)} | ${data.latencyMs}ms | cache=${data.cacheHit}`)
    }

    return data.content
  }
}

// 全局单例，所有 Agent 工具共享
export const llm = new GatewayClient()
```

### 3.3 在 OpenClaw（pi-mono）风格中使用

```typescript
// tools/summarize.ts - Tool 不需要知道底层模型

import { llm } from '../llm-gateway/client'

export async function summarizeTool(text: string): Promise<string> {
  return llm.chat([
    { role: 'user', content: `请用3句话总结：\n\n${text}` }
  ], {
    temperature: 0,  // 确定性输出 → 会被缓存
    tags: ['summarize'],  // 成本追踪 tag
  })
}
```

---

## 4. Python 版本（learn-claude-code 风格）

```python
# llm_gateway/gateway.py

import hashlib, json, time
from dataclasses import dataclass
from typing import Optional
import redis
import anthropic
import openai

@dataclass
class GatewayRequest:
    messages: list[dict]
    model: str = "claude-sonnet-4-5"
    max_tokens: int = 4096
    temperature: float = 1.0
    tenant_id: str = "default"
    tags: list[str] = None

@dataclass
class GatewayResponse:
    content: str
    model: str
    input_tokens: int
    output_tokens: int
    cost_usd: float
    cache_hit: bool
    latency_ms: int

TOKEN_COSTS = {
    "claude-sonnet-4-5": (3.0, 15.0),
    "claude-haiku-3-5":  (0.8,  4.0),
    "gpt-4o":            (2.5, 10.0),
    "gpt-4o-mini":       (0.15, 0.6),
}

class LLMGateway:
    def __init__(self, redis_url: str):
        self.r = redis.from_url(redis_url)
        self.anthropic = anthropic.Anthropic()
        self.openai_client = openai.OpenAI()

    def chat(self, req: GatewayRequest) -> GatewayResponse:
        start = time.time()

        # 限流
        self._rate_limit(req.tenant_id)

        # 缓存
        key = self._cache_key(req)
        if req.temperature == 0 and (hit := self.r.get(key)):
            cached = GatewayResponse(**json.loads(hit))
            cached.cache_hit = True
            cached.latency_ms = int((time.time() - start) * 1000)
            return cached

        # 调用
        if req.model.startswith("claude"):
            resp = self.anthropic.messages.create(
                model=req.model,
                max_tokens=req.max_tokens,
                messages=req.messages,
            )
            content = resp.content[0].text
            in_tok, out_tok = resp.usage.input_tokens, resp.usage.output_tokens
        else:
            resp = self.openai_client.chat.completions.create(
                model=req.model, messages=req.messages,
                max_tokens=req.max_tokens, temperature=req.temperature,
            )
            content = resp.choices[0].message.content
            in_tok  = resp.usage.prompt_tokens
            out_tok = resp.usage.completion_tokens

        in_cost, out_cost = TOKEN_COSTS.get(req.model, (3.0, 15.0))
        cost = (in_tok * in_cost + out_tok * out_cost) / 1_000_000

        result = GatewayResponse(
            content=content, model=req.model,
            input_tokens=in_tok, output_tokens=out_tok,
            cost_usd=cost, cache_hit=False,
            latency_ms=int((time.time() - start) * 1000),
        )

        # 写缓存
        if req.temperature == 0:
            self.r.setex(key, 3600, json.dumps(result.__dict__))

        # 记录成本
        day = time.strftime("%Y-%m-%d")
        self.r.incrbyfloat(f"cost:{req.tenant_id}:{day}", cost)
        for tag in (req.tags or []):
            self.r.incrbyfloat(f"cost:tag:{tag}:{day}", cost)

        return result

    def _rate_limit(self, tenant_id: str):
        key = f"rl:{tenant_id}"
        count = self.r.incr(key)
        if count == 1:
            self.r.expire(key, 60)
        if count > 60:
            ttl = self.r.ttl(key)
            raise Exception(f"Rate limited. Retry in {ttl}s")

    def _cache_key(self, req: GatewayRequest) -> str:
        payload = json.dumps({"model": req.model, "messages": req.messages}, sort_keys=True)
        return f"llm:cache:{hashlib.sha256(payload.encode()).hexdigest()[:16]}"
```

---

## 5. 进阶功能：成本仪表板查询

```typescript
// 查询今日各 tag 的成本分布
async function getDailyCostByTag(date: string): Promise<Record<string, number>> {
  const keys = await redis.keys(`cost:tag:*:${date}`)
  const result: Record<string, number> = {}

  for (const key of keys) {
    const tag = key.split(':')[2]
    result[tag] = parseFloat(await redis.get(key) ?? '0')
  }

  return result
}

// 输出示例：
// {
//   "summarize": 0.0042,
//   "code-review": 0.0218,
//   "rag-query": 0.0089
// }
```

---

## 6. 与已学知识的关联

| 本节 | 依赖/扩展 |
|------|-----------|
| Rate Limiter | Lesson 15 - 限流与退避 |
| Semantic Cache | Lesson 54 - 语义缓存 |
| Multi-Model Fallback | Lesson 91 - 多模型回退链（在 Gateway 内实现更合适） |
| Cost Optimization | Lesson 22 - 成本优化策略 |
| Tenant Auth | Lesson 48 - 多租户架构 |
| Metrics/Audit | Lesson 11 - 可观测性 |

---

## 7. 关键设计原则

1. **业务代码零感知**：Agent/Tool 只调用 `llm.chat()`，不知道背后是 Claude 还是 GPT
2. **密钥集中管理**：所有 API Key 只在 Gateway 里，业务服务用 Tenant ID 鉴权
3. **成本可追踪**：通过 `tags` 字段，精确到功能级别的成本分析
4. **缓存在网关层**：`temperature=0` 的请求自动缓存，业务代码不需要手动管
5. **限流保护上游**：突发流量在网关被消化，不会打爆 Provider 的 rate limit

---

## 8. 什么时候用 Gateway？

✅ 适合用 Gateway：
- 多个 Agent / 服务共享 LLM 访问
- 需要统一成本追踪
- 需要跨 Provider 路由
- 生产环境，API Key 需要集中管理

❌ 不适合：
- 单个简单脚本
- 原型阶段，还在快速迭代
- 延迟极度敏感（Gateway 会增加 1-5ms 网络跳转）

---

## 小结

LLM Gateway 是生产级 Agent 架构的「基础设施层」：它把所有 LLM 调用收口，统一处理横切关注点（限流、缓存、成本、日志），让业务 Agent 专注于业务逻辑。规模越大，Gateway 的价值越明显。
