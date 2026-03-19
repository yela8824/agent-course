# 88 - Agent OpenTelemetry 分布式追踪实战

> 日志告诉你"发生了什么"，Trace 告诉你"为什么这么慢"——把每一次 LLM 调用、工具执行都串成一条完整的调用链。

---

## 🧠 核心思想

Agent 系统一旦复杂，光靠 `console.log` 根本看不清全貌：

```
用户请求 → Agent A → Tool X → Sub-agent B → Tool Y → Tool Z → 返回

  哪段慢？哪个工具超时？Sub-agent 花了多久？
  → 日志里全是零散行，根本串不起来。
```

**OpenTelemetry（OTel）** 是 CNCF 的可观测性标准，三件套：
- **Trace**：一次请求的完整调用链（由多个 Span 组成）
- **Span**：调用链上的一个节点（有开始时间、结束时间、属性、事件）
- **Context Propagation**：把 Trace ID 跨进程 / 跨 Agent 传递

```
Trace ID: abc-123
├── Span: agent.turn          [0ms → 2341ms]
│   ├── Span: llm.call        [0ms → 890ms]   ← LLM 思考
│   ├── Span: tool.web_search [910ms → 1200ms] ← 工具执行
│   └── Span: tool.write_file [1210ms → 1250ms]
└── 完整链路一目了然
```

---

## 🏗️ 架构图

```
Agent Process
┌─────────────────────────────────────┐
│  OTel SDK                           │
│  ┌──────────────────────────────┐   │
│  │ TracerProvider               │   │
│  │  └─ Tracer("agent-core")    │   │
│  └──────────────────────────────┘   │
│           │ spans                   │
│           ▼                         │
│  ┌──────────────────────────────┐   │
│  │ BatchSpanProcessor           │   │──→ OTLP Exporter ──→ Jaeger/Tempo
│  └──────────────────────────────┘   │                      /DataDog/etc
└─────────────────────────────────────┘
```

---

## 📦 安装

```bash
# Node.js / TypeScript (pi-mono 风格)
npm install @opentelemetry/sdk-node \
            @opentelemetry/auto-instrumentations-node \
            @opentelemetry/exporter-trace-otlp-http \
            @opentelemetry/api

# Python (learn-claude-code 风格)
pip install opentelemetry-sdk \
            opentelemetry-exporter-otlp \
            opentelemetry-instrumentation-httpx
```

---

## 🔧 初始化（TypeScript）

```typescript
// src/telemetry.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-node';
import { Resource } from '@opentelemetry/resources';
import { SEMRESATTRS_SERVICE_NAME } from '@opentelemetry/semantic-conventions';

const exporter = new OTLPTraceExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://localhost:4318/v1/traces',
});

export const sdk = new NodeSDK({
  resource: new Resource({
    [SEMRESATTRS_SERVICE_NAME]: 'my-agent',
  }),
  spanProcessor: new BatchSpanProcessor(exporter),
});

// 最先执行！main() 之前
sdk.start();

process.on('SIGTERM', () => sdk.shutdown());
```

```typescript
// src/index.ts — 必须在最顶部
import './telemetry';  // ← 第一行！
import { runAgent } from './agent';
```

---

## 🎯 给 Agent Loop 埋点

```typescript
// src/agent.ts
import { trace, SpanStatusCode, context } from '@opentelemetry/api';

const tracer = trace.getTracer('agent-core', '1.0.0');

export async function runAgent(userMessage: string): Promise<string> {
  // 创建根 Span：一次完整的 Agent Turn
  return tracer.startActiveSpan('agent.turn', async (span) => {
    span.setAttributes({
      'agent.model': 'claude-sonnet-4',
      'user.message.length': userMessage.length,
      'session.id': getCurrentSessionId(),
    });

    try {
      const result = await agentLoop(userMessage);
      
      span.setAttributes({ 'agent.output.length': result.length });
      span.setStatus({ code: SpanStatusCode.OK });
      return result;

    } catch (err) {
      // 错误自动记录到 Span
      span.recordException(err as Error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: (err as Error).message });
      throw err;

    } finally {
      span.end();  // 无论成功失败都要 end！
    }
  });
}
```

---

## 🔧 给 LLM 调用埋点

```typescript
// src/llm.ts
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('agent-core');

export async function callLLM(messages: Message[], tools: Tool[]): Promise<LLMResponse> {
  return tracer.startActiveSpan('llm.call', async (span) => {
    span.setAttributes({
      'llm.model': 'claude-sonnet-4',
      'llm.input.messages': messages.length,
      'llm.input.tools': tools.length,
      // 注意：不要把完整 prompt 放进 span，太大且可能含敏感信息
      'llm.input.tokens.estimate': estimateTokens(messages),
    });

    const start = Date.now();
    const response = await anthropic.messages.create({
      model: 'claude-sonnet-4',
      messages,
      tools,
    });
    const latency = Date.now() - start;

    span.setAttributes({
      'llm.output.stop_reason': response.stop_reason,
      'llm.usage.input_tokens': response.usage.input_tokens,
      'llm.usage.output_tokens': response.usage.output_tokens,
      'llm.latency_ms': latency,
    });

    // 记录关键事件（非结构化的"里程碑"）
    if (response.stop_reason === 'tool_use') {
      span.addEvent('tool_use_requested', {
        tool_names: response.content
          .filter(b => b.type === 'tool_use')
          .map(b => b.name)
          .join(','),
      });
    }

    span.end();
    return response;
  });
}
```

---

## 🔨 给工具执行埋点

```typescript
// src/tool-executor.ts
import { trace, SpanStatusCode, SpanKind } from '@opentelemetry/api';

const tracer = trace.getTracer('agent-core');

export async function executeTool(
  toolName: string,
  toolInput: Record<string, unknown>
): Promise<ToolResult> {
  return tracer.startActiveSpan(
    `tool.${toolName}`,
    { kind: SpanKind.INTERNAL },
    async (span) => {
      span.setAttributes({
        'tool.name': toolName,
        // 记录关键参数（避免记录大文本）
        'tool.input.keys': Object.keys(toolInput).join(','),
      });

      const start = Date.now();

      try {
        const result = await TOOL_MAP[toolName](toolInput);
        
        span.setAttributes({
          'tool.success': true,
          'tool.latency_ms': Date.now() - start,
          'tool.output.size': JSON.stringify(result).length,
        });
        span.end();
        return result;

      } catch (err) {
        span.setAttributes({
          'tool.success': false,
          'tool.error': (err as Error).message,
          'tool.latency_ms': Date.now() - start,
        });
        span.recordException(err as Error);
        span.setStatus({ code: SpanStatusCode.ERROR });
        span.end();
        throw err;
      }
    }
  );
}
```

---

## 🌐 跨 Sub-agent 传递 Trace Context（重点！）

这是分布式追踪的精髓：让 Sub-agent 的 spans 挂在父 Agent 的 Trace 下。

```typescript
// src/subagent-launcher.ts
import { context, propagation, trace } from '@opentelemetry/api';

// ① 父 Agent：把当前 context 注入 HTTP Header
async function spawnSubAgent(task: string): Promise<void> {
  const headers: Record<string, string> = {};
  
  // 把 W3C traceparent header 注入（包含 traceId + spanId）
  propagation.inject(context.active(), headers);
  // headers 现在有: { traceparent: "00-abc123-def456-01" }

  await fetch('http://sub-agent-service/run', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      ...headers,  // ← 关键：把 trace context 带过去
    },
    body: JSON.stringify({ task }),
  });
}

// ② Sub-agent 服务端：从 Header 提取并恢复 context
import express from 'express';
const app = express();

app.post('/run', async (req, res) => {
  // 从请求头恢复 trace context
  const parentContext = propagation.extract(context.active(), req.headers);
  
  // 在父 context 下创建 span → 自动成为父 Agent span 的子节点！
  await context.with(parentContext, async () => {
    await tracer.startActiveSpan('subagent.run', async (span) => {
      span.setAttributes({ 'subagent.task': req.body.task });
      
      const result = await runSubAgentLogic(req.body.task);
      
      span.end();
      res.json({ result });
    });
  });
});
```

结果：在 Jaeger 看到的是一棵完整的树：
```
Trace: user-request [3.2s]
├── agent.turn [3.2s]
│   ├── llm.call [0.9s]
│   ├── tool.web_search [0.4s]
│   └── subagent.run [1.8s]    ← Sub-agent 的 spans 自动归入！
│       ├── llm.call [0.7s]
│       └── tool.write_file [0.1s]
```

---

## 🐍 Python 版本（learn-claude-code 风格）

```python
# telemetry.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

def setup_tracing(service_name: str = "agent"):
    resource = Resource({"service.name": service_name})
    provider = TracerProvider(resource=resource)
    exporter = OTLPSpanExporter(
        endpoint=os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT", "http://localhost:4318/v1/traces")
    )
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)
    return trace.get_tracer(service_name)

tracer = setup_tracing("my-agent")

# 装饰器写法（更 Pythonic）
from opentelemetry import trace
from functools import wraps

def traced(span_name: str = None):
    """给任意函数加 span 的装饰器"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            name = span_name or f"{func.__module__}.{func.__name__}"
            with tracer.start_as_current_span(name) as span:
                try:
                    result = await func(*args, **kwargs)
                    return result
                except Exception as e:
                    span.record_exception(e)
                    span.set_status(trace.StatusCode.ERROR, str(e))
                    raise
        return wrapper
    return decorator

# 使用
@traced("agent.turn")
async def run_agent(message: str) -> str:
    ...

@traced("tool.search")
async def web_search(query: str) -> list:
    ...
```

---

## 🐳 本地快速搭建 Jaeger

```yaml
# docker-compose.yml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"   # Jaeger UI
      - "4318:4318"     # OTLP HTTP receiver
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

```bash
docker-compose up -d

# 跑你的 Agent
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318/v1/traces node src/index.js

# 打开 Jaeger UI
open http://localhost:16686
```

---

## 📊 OpenClaw 中的实际埋点策略

OpenClaw 本身是一个 always-on Agent 平台，以下是合理的埋点位置：

```
用户消息到来
    ↓
[span: session.inbound_message]
    ↓
[span: skill.select]        ← 技能选择耗时
    ↓
[span: agent.turn]          ← 完整 Agent 循环
    ├── [span: memory.search]    ← 记忆检索
    ├── [span: llm.call] × N    ← 每次 LLM 调用
    ├── [span: tool.xxx] × M    ← 每次工具执行
    └── [span: message.send]    ← 回复发送
```

关键指标（通过 span attributes 收集）：
- `llm.latency_ms` 分布 → P50/P95/P99
- `tool.latency_ms` by tool name → 找最慢的工具
- `agent.turn.total_ms` → 用户感知延迟
- `llm.usage.total_tokens` → 成本监控

---

## ⚡ 实用技巧

### 1. 采样策略：不是每条都需要追踪
```typescript
import { ParentBasedSampler, TraceIdRatioBased } from '@opentelemetry/sdk-trace-node';

// 只采样 10% 的 trace（生产环境节省开销）
const sampler = new ParentBasedSampler({
  root: new TraceIdRatioBased(0.1),
});
```

### 2. 给 Span 加 Baggage（跨服务透传业务字段）
```typescript
import { propagation, context } from '@opentelemetry/api';

// 设置 baggage（会跟随 trace context 传播）
const bag = propagation.createBaggage({
  'user.id': { value: userId },
  'session.id': { value: sessionId },
});
const ctx = propagation.setBaggage(context.active(), bag);

// 后续所有子 span 都能读到
const userIdFromBaggage = propagation.getBaggage(context.active())
  ?.getEntry('user.id')?.value;
```

### 3. 避免 span 信息泄漏
```typescript
// ❌ 不要把用户消息原文放进 span attributes
span.setAttribute('user.message', userMessage);  // 可能含敏感信息

// ✅ 只记录元数据
span.setAttribute('user.message.length', userMessage.length);
span.setAttribute('user.message.language', detectLanguage(userMessage));
```

---

## 🎯 总结

| 维度 | 日志（Logs） | 指标（Metrics） | 链路追踪（Traces） |
|------|-------------|----------------|-------------------|
| 问题 | 发生了什么 | 有多少 | 为什么慢 |
| 粒度 | 行级别 | 聚合数值 | 请求级别 |
| 关联 | 难 | 无 | 天然关联 |
| Agent 价值 | 调试 | 监控告警 | 性能分析 |

三者结合才是完整可观测性。OpenTelemetry 的优势是**一套 SDK，接任何后端**（Jaeger/Grafana Tempo/DataDog/New Relic），不锁厂商。

**推荐起步顺序：**
1. 先给 `agent.turn` + `llm.call` 埋点
2. 本地跑 Jaeger 验证链路
3. 加工具 span
4. 实现跨 Sub-agent 传播
5. 生产环境接 Grafana Tempo 或 DataDog

---

*下一课预告：Agent API Schema 版本管理与向后兼容*
