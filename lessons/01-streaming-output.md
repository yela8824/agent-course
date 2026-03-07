# 第 1 课：流式输出 (Streaming) — Agent 实时响应的秘密

> *"用户看到的不是等待，而是 AI 在思考"*

---

## 🤔 问题：为什么需要流式输出？

传统 HTTP 请求：
```
User: "写一篇文章"
     ↓
   等待 30 秒...
     ↓
Response: [完整的 5000 字文章]
```

**用户体验极差**：
- 30 秒黑屏
- 不知道 AI 在干嘛
- 可能以为卡住了

---

## 💡 解决方案：Server-Sent Events (SSE)

```
User: "写一篇文章"
     ↓
data: {"type":"text_delta","text":"今"}
data: {"type":"text_delta","text":"天"}
data: {"type":"text_delta","text":"我们"}
data: {"type":"text_delta","text":"来聊聊"}
...
```

**用户看到文字一个个蹦出来，像真人在打字。**

---

## 🔧 实现原理

### 1. LLM Provider 的流式 API

```typescript
// Anthropic
const stream = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  messages: [...],
  stream: true,  // 关键！
});

for await (const event of stream) {
  if (event.type === "content_block_delta") {
    console.log(event.delta.text);  // 逐 token 输出
  }
}
```

### 2. pi-mono 的 EventStream 抽象

```typescript
// packages/ai/src/utils/event-stream.ts
export class EventStream<T, R = T> implements AsyncIterable<T> {
  private queue: T[] = [];
  private waiting: ((value: IteratorResult<T>) => void)[] = [];
  private done = false;

  push(event: T): void {
    if (this.done) return;
    
    // 有人在等？直接给
    const waiter = this.waiting.shift();
    if (waiter) {
      waiter({ value: event, done: false });
    } else {
      // 没人等？入队
      this.queue.push(event);
    }
  }

  async *[Symbol.asyncIterator](): AsyncIterator<T> {
    while (true) {
      if (this.queue.length > 0) {
        yield this.queue.shift()!;
      } else if (this.done) {
        return;
      } else {
        // 等待新事件
        const result = await new Promise<IteratorResult<T>>(
          (resolve) => this.waiting.push(resolve)
        );
        if (result.done) return;
        yield result.value;
      }
    }
  }
}
```

**核心设计**：
- `push()`: 生产者推送事件
- `async iterator`: 消费者拉取事件
- 队列 + Promise 实现背压控制

### 3. 事件类型定义

```typescript
type AssistantMessageEvent =
  | { type: "text"; text: string }           // 文本增量
  | { type: "text_delta"; delta: string }    // 文本片段
  | { type: "thinking"; text: string }       // 思考过程
  | { type: "tool_call"; ... }               // 工具调用
  | { type: "usage"; ... }                   // Token 统计
  | { type: "done"; message: AssistantMessage }  // 完成
  | { type: "error"; error: any }            // 错误
```

---

## 📊 流式 vs 非流式

```
非流式：
[Request] ──────────────────────> [Full Response]
           30s 黑屏

流式：
[Request] ─event─event─event─event─event─> [Done]
          每 50ms 一个 token
```

| 特性 | 非流式 | 流式 |
|------|--------|------|
| 首字节时间 | 30s | 500ms |
| 用户体验 | 差 | 好 |
| 可中断 | 难 | 易 |
| 实现复杂度 | 低 | 中 |

---

## 🛠️ 实战：前端消费流式数据

### 方式 1：EventSource (SSE)

```typescript
const eventSource = new EventSource('/api/chat?prompt=hello');

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.type === 'text_delta') {
    appendToUI(data.delta);
  }
};
```

### 方式 2：fetch + ReadableStream

```typescript
const response = await fetch('/api/chat', {
  method: 'POST',
  body: JSON.stringify({ prompt: 'hello' }),
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  
  const chunk = decoder.decode(value);
  // 解析 SSE 格式
  const lines = chunk.split('\n');
  for (const line of lines) {
    if (line.startsWith('data: ')) {
      const data = JSON.parse(line.slice(6));
      handleEvent(data);
    }
  }
}
```

---

## 🎯 关键设计决策

### 1. 事件粒度

```typescript
// 太粗：一次性给整段
{ type: "text", text: "这是一整段文字..." }

// 太细：每个字符
{ type: "text_delta", delta: "这" }
{ type: "text_delta", delta: "是" }

// 合适：按 token（约 3-4 个字符）
{ type: "text_delta", delta: "这是一" }
{ type: "text_delta", delta: "整段" }
```

### 2. 错误处理

```typescript
try {
  for await (const event of stream) {
    if (event.type === 'error') {
      // 错误也是事件，不是异常
      showError(event.error);
      break;
    }
    handleEvent(event);
  }
} catch (e) {
  // 网络错误等
  showError(e);
}
```

### 3. 中断支持

```typescript
const controller = new AbortController();

const stream = fetchStream('/api/chat', {
  signal: controller.signal
});

// 用户点击取消
cancelButton.onclick = () => controller.abort();
```

---

## 💡 在 Agent 中的应用

```typescript
// Agent 循环中的流式处理
async function* agentLoop(messages) {
  const stream = llm.stream(messages, tools);
  
  for await (const event of stream) {
    // 1. 实时输出给用户
    yield event;
    
    // 2. 检测工具调用
    if (event.type === 'tool_call') {
      const result = await executeTool(event);
      messages.push(toolResult(result));
      // 继续循环...
    }
  }
}
```

**流式输出让 Agent 能够**：
- 实时显示思考过程
- 尽早展示结果
- 支持随时中断

---

## 🧠 要点总结

1. **流式 = 更好的用户体验**
2. **EventStream 模式**：push/pull + 队列 + Promise
3. **事件类型**：text_delta, tool_call, done, error
4. **前端消费**：EventSource 或 fetch + ReadableStream
5. **支持中断**：AbortController

---

下一课：**错误处理与重试策略**

🔗 参考代码：
- [pi-mono/packages/ai/src/utils/event-stream.ts](https://github.com/badlogic/pi-mono/blob/main/packages/ai/src/utils/event-stream.ts)
- [pi-mono/packages/ai/src/stream.ts](https://github.com/badlogic/pi-mono/blob/main/packages/ai/src/stream.ts)
