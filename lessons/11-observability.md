# 第 11 课：Observability 可观测性

> 如果你不能观测它，你就不能改进它。

## 为什么 Agent 需要可观测性？

传统软件的 bug 是确定性的——相同输入产生相同输出。但 Agent 不一样：

- LLM 输出有随机性
- 工具调用可能失败
- 上下文会动态变化
- 用户意图难以预测

没有可观测性，你就是在黑盒中调试。

## 可观测性的三大支柱

### 1. Logs（日志）- 发生了什么

```typescript
// pi-mono 风格的结构化日志
interface AgentLog {
  timestamp: number;
  level: 'debug' | 'info' | 'warn' | 'error';
  event: string;
  sessionId: string;
  turnId: string;
  data?: Record<string, unknown>;
}

// 好的日志
logger.info('tool_call_start', {
  sessionId: 'sess_123',
  turnId: 'turn_456',
  tool: 'web_search',
  params: { query: 'rust async' }
});

// 坏的日志
console.log('calling tool...');  // ❌ 没有上下文
```

### 2. Metrics（指标）- 多少、多快、多贵

```typescript
// 关键 Agent 指标
interface AgentMetrics {
  // 性能
  latency_p50_ms: number;
  latency_p99_ms: number;
  turns_per_session: number;
  
  // 成本
  tokens_in: number;
  tokens_out: number;
  cost_usd: number;
  
  // 质量
  tool_success_rate: number;
  user_satisfaction: number;  // 如果有反馈机制
  retry_rate: number;
  
  // 容量
  concurrent_sessions: number;
  queue_depth: number;
}
```

### 3. Traces（追踪）- 请求的完整旅程

```typescript
// 一个 Agent turn 的 trace
interface AgentTrace {
  traceId: string;
  spans: Span[];
}

interface Span {
  spanId: string;
  parentId?: string;
  name: string;
  startTime: number;
  endTime: number;
  attributes: Record<string, unknown>;
}

// 示例 trace
const trace = {
  traceId: 'trace_abc',
  spans: [
    { spanId: '1', name: 'agent_turn', startTime: 0, endTime: 2500, attributes: {} },
    { spanId: '2', parentId: '1', name: 'llm_call', startTime: 100, endTime: 1800, attributes: { model: 'claude-3-5-sonnet', tokens: 1500 } },
    { spanId: '3', parentId: '1', name: 'tool_exec', startTime: 1850, endTime: 2400, attributes: { tool: 'exec', exitCode: 0 } },
  ]
};
```

## 实战：OpenClaw 的可观测性实现

### Session Status 命令

OpenClaw 内置了 `session_status` 工具，这就是可观测性的体现：

```typescript
// OpenClaw session_status 返回
{
  model: 'claude-opus-4',
  tokens: { in: 15234, out: 8901 },
  cost: { usd: 0.47 },
  duration: '15m 32s',
  turns: 12
}
```

### 结构化日志设计

```typescript
// 推荐的 Agent 日志格式
class AgentLogger {
  private sessionId: string;
  
  log(event: string, data?: object) {
    const entry = {
      t: Date.now(),
      session: this.sessionId,
      event,
      ...data
    };
    
    // 结构化输出，方便后续处理
    console.log(JSON.stringify(entry));
  }
  
  // 预定义的关键事件
  turnStart(turnId: string) {
    this.log('turn_start', { turnId });
  }
  
  llmCall(model: string, tokens: { in: number; out: number }) {
    this.log('llm_call', { model, tokens });
  }
  
  toolCall(tool: string, params: object, durationMs: number, success: boolean) {
    this.log('tool_call', { tool, params, durationMs, success });
  }
  
  turnEnd(turnId: string, success: boolean) {
    this.log('turn_end', { turnId, success });
  }
}
```

### Trace 上下文传播

```typescript
// 在整个请求链路中传递 trace 上下文
interface TraceContext {
  traceId: string;
  spanId: string;
  
  // 创建子 span
  child(name: string): TraceContext;
  
  // 结束当前 span
  end(attributes?: object): void;
}

// 使用示例
async function agentTurn(ctx: TraceContext, message: string) {
  const turnCtx = ctx.child('agent_turn');
  
  try {
    // LLM 调用
    const llmCtx = turnCtx.child('llm_call');
    const response = await callLLM(message);
    llmCtx.end({ model: 'claude', tokens: response.usage });
    
    // 工具执行
    for (const tool of response.toolCalls) {
      const toolCtx = turnCtx.child('tool_exec');
      await executeTool(tool);
      toolCtx.end({ tool: tool.name });
    }
    
    turnCtx.end({ success: true });
  } catch (error) {
    turnCtx.end({ success: false, error: error.message });
    throw error;
  }
}
```

## 关键指标设计

### Token 计费追踪

```typescript
class TokenTracker {
  private usage: Map<string, { in: number; out: number; cost: number }> = new Map();
  
  track(model: string, tokensIn: number, tokensOut: number) {
    const pricing = this.getPricing(model);
    const cost = tokensIn * pricing.in + tokensOut * pricing.out;
    
    const current = this.usage.get(model) || { in: 0, out: 0, cost: 0 };
    this.usage.set(model, {
      in: current.in + tokensIn,
      out: current.out + tokensOut,
      cost: current.cost + cost
    });
  }
  
  private getPricing(model: string) {
    // 每 1M tokens 的价格
    const prices: Record<string, { in: number; out: number }> = {
      'claude-3-5-sonnet': { in: 3, out: 15 },
      'claude-opus-4': { in: 15, out: 75 },
      'gpt-4o': { in: 2.5, out: 10 },
    };
    return prices[model] || { in: 0, out: 0 };
  }
  
  report() {
    let total = 0;
    for (const [model, usage] of this.usage) {
      console.log(`${model}: ${usage.in} in, ${usage.out} out, $${usage.cost.toFixed(4)}`);
      total += usage.cost;
    }
    console.log(`Total: $${total.toFixed(4)}`);
  }
}
```

### 工具成功率监控

```typescript
class ToolMetrics {
  private calls: Map<string, { success: number; failure: number; totalMs: number }> = new Map();
  
  record(tool: string, success: boolean, durationMs: number) {
    const current = this.calls.get(tool) || { success: 0, failure: 0, totalMs: 0 };
    
    if (success) {
      current.success++;
    } else {
      current.failure++;
    }
    current.totalMs += durationMs;
    
    this.calls.set(tool, current);
  }
  
  getSuccessRate(tool: string): number {
    const stats = this.calls.get(tool);
    if (!stats) return 0;
    return stats.success / (stats.success + stats.failure);
  }
  
  getAvgLatency(tool: string): number {
    const stats = this.calls.get(tool);
    if (!stats) return 0;
    return stats.totalMs / (stats.success + stats.failure);
  }
}
```

## 生产环境最佳实践

### 1. 日志分级

```typescript
// 不同环境的日志级别
const LOG_LEVELS = {
  production: ['error', 'warn', 'info'],
  staging: ['error', 'warn', 'info', 'debug'],
  development: ['error', 'warn', 'info', 'debug', 'trace']
};

// 敏感信息脱敏
function sanitize(data: object): object {
  const sensitive = ['password', 'token', 'key', 'secret'];
  return JSON.parse(JSON.stringify(data, (key, value) => {
    if (sensitive.some(s => key.toLowerCase().includes(s))) {
      return '[REDACTED]';
    }
    return value;
  }));
}
```

### 2. 采样策略

```typescript
// 不是每个请求都需要完整 trace
class TraceSampler {
  sample(traceId: string): boolean {
    // 10% 采样
    const hash = this.hashCode(traceId);
    return hash % 10 === 0;
  }
  
  // 错误请求总是采样
  shouldSampleError(): boolean {
    return true;
  }
  
  // 高延迟请求总是采样
  shouldSampleSlowRequest(durationMs: number): boolean {
    return durationMs > 5000;  // 超过 5 秒
  }
  
  private hashCode(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash |= 0;
    }
    return Math.abs(hash);
  }
}
```

### 3. 告警设计

```typescript
// 定义关键告警
const ALERTS = [
  {
    name: 'high_error_rate',
    condition: (metrics: AgentMetrics) => metrics.tool_success_rate < 0.95,
    severity: 'warning',
    message: 'Tool success rate dropped below 95%'
  },
  {
    name: 'high_latency',
    condition: (metrics: AgentMetrics) => metrics.latency_p99_ms > 30000,
    severity: 'critical',
    message: 'P99 latency exceeded 30 seconds'
  },
  {
    name: 'cost_spike',
    condition: (metrics: AgentMetrics, prev: AgentMetrics) => 
      metrics.cost_usd > prev.cost_usd * 2,
    severity: 'warning',
    message: 'Cost doubled compared to previous period'
  }
];
```

## learn-claude-code 中的可观测性

```python
# Python 版本的结构化日志
import json
import time
from dataclasses import dataclass, asdict
from typing import Optional

@dataclass
class LogEntry:
    timestamp: float
    event: str
    session_id: str
    data: Optional[dict] = None
    
    def emit(self):
        print(json.dumps(asdict(self)))

class AgentLogger:
    def __init__(self, session_id: str):
        self.session_id = session_id
    
    def log(self, event: str, **data):
        entry = LogEntry(
            timestamp=time.time(),
            event=event,
            session_id=self.session_id,
            data=data if data else None
        )
        entry.emit()
    
    def turn_start(self, turn_id: str):
        self.log('turn_start', turn_id=turn_id)
    
    def llm_call(self, model: str, tokens_in: int, tokens_out: int, duration_ms: int):
        self.log('llm_call', 
                 model=model, 
                 tokens_in=tokens_in, 
                 tokens_out=tokens_out,
                 duration_ms=duration_ms)
    
    def tool_call(self, tool: str, success: bool, duration_ms: int):
        self.log('tool_call', tool=tool, success=success, duration_ms=duration_ms)
```

## 可视化工具选择

| 工具 | 用途 | 特点 |
|------|------|------|
| **Grafana** | 指标可视化 | 灵活的 dashboard |
| **Jaeger/Zipkin** | 分布式追踪 | 可视化请求链路 |
| **Loki** | 日志聚合 | 与 Grafana 集成好 |
| **OpenTelemetry** | 统一标准 | 一套 SDK 搞定三大支柱 |
| **Langfuse** | LLM 专用 | Agent 场景优化 |

## 总结

1. **日志**：结构化、带上下文、注意脱敏
2. **指标**：关注延迟、成本、成功率
3. **追踪**：理解请求的完整旅程
4. **告警**：及时发现问题

可观测性不是事后想起来才加的——从第一天就要设计进去。

---

下一课预告：**Caching Strategies 缓存策略** - 如何缓存 LLM 响应降低成本和延迟
