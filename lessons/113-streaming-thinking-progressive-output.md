# 113 - Agent 流式思维与渐进输出（Streaming Thinking & Progressive Output）

> 用户不想盯着光标转圈等 5 秒再看到结果——流式思维让 Agent 边想边说，渐进输出让用户实时感知进展。

---

## 为什么需要这个？

传统的 Request-Response 模式：发请求 → 等待 → 一次性收结果。

对 Agent 来说这有两个大问题：
1. **思考时间长**：复杂任务 LLM 可能要 10-30 秒才给出答案，用户以为挂了
2. **工具链串行**：Tool A → Tool B → Tool C，每步都要等上一步完成才看到结果

流式解决方案：**边生成边发送**，用户第一个 token 就在 1-2 秒内出现。

---

## 两个维度

```
流式思维 (Streaming Thinking)     渐进输出 (Progressive Output)
      │                                    │
      ▼                                    ▼
 LLM 内部推理过程                   工具结果逐步呈现
 (thinking tokens)                 (partial results)
 
 用户看到 AI "在思考"               用户看到任务进展
```

---

## Part 1：流式思维（Extended Thinking Tokens）

Claude 支持 `thinking` 块——在最终回答之前的推理过程，可以流式输出。

### OpenClaw 的实现

OpenClaw 系统提示显示：`Reasoning: off (hidden unless on/stream)` — 说明底层支持 thinking token 的开关控制。

```typescript
// pi-mono 风格：启用流式 extended thinking
const stream = await anthropic.messages.stream({
  model: "claude-opus-4-5",
  max_tokens: 16000,
  thinking: {
    type: "enabled",
    budget_tokens: 10000,   // 给思考预留的 token 预算
  },
  messages: [{ role: "user", content: userMessage }],
});

// 流式处理不同类型的事件
for await (const event of stream) {
  if (event.type === "content_block_start") {
    if (event.content_block.type === "thinking") {
      // 思考块开始
      onThinkingStart();
    } else if (event.content_block.type === "text") {
      // 正式回答开始
      onTextStart();
    }
  }
  
  if (event.type === "content_block_delta") {
    if (event.delta.type === "thinking_delta") {
      // 思考内容（可以展示给用户，也可以隐藏）
      onThinkingDelta(event.delta.thinking);
    } else if (event.delta.type === "text_delta") {
      // 正式回答内容，实时流出
      onTextDelta(event.delta.text);
    }
  }
}
```

### 思考 Token 的三种展示策略

```typescript
type ThinkingDisplay = "hidden" | "collapsed" | "stream";

class ThinkingRenderer {
  private buffer = "";
  private mode: ThinkingDisplay;
  
  constructor(mode: ThinkingDisplay) {
    this.mode = mode;
  }
  
  onThinkingDelta(chunk: string) {
    this.buffer += chunk;
    
    switch (this.mode) {
      case "hidden":
        // 不展示，但内部保留用于调试
        break;
        
      case "collapsed":
        // 只展示"正在思考..."动画，完成后可展开
        updateSpinner("🤔 思考中...");
        break;
        
      case "stream":
        // 实时流式展示思考过程（OpenClaw /reasoning 模式）
        appendToThinkingBox(chunk);
        break;
    }
  }
  
  onThinkingEnd() {
    if (this.mode === "collapsed") {
      // 思考结束后，提供折叠/展开控件
      renderCollapsible("查看推理过程", this.buffer);
    }
  }
}
```

---

## Part 2：流式工具调用

工具调用也可以流式处理——在 LLM 决定调用哪个工具时立即开始执行，不等 LLM 生成完整的 tool_use 块。

```typescript
// 流式解析 tool_use 参数
class StreamingToolParser {
  private currentTool: string | null = null;
  private inputBuffer = "";
  private toolCallId: string | null = null;
  
  async processStream(stream: AsyncIterable<MessageStreamEvent>) {
    for await (const event of stream) {
      if (event.type === "content_block_start" && 
          event.content_block.type === "tool_use") {
        // 工具调用开始——记录工具名和 ID
        this.currentTool = event.content_block.name;
        this.toolCallId = event.content_block.id;
        this.inputBuffer = "";
        
        // 立即通知：LLM 决定调用这个工具了
        this.onToolDecided(this.currentTool);
      }
      
      if (event.type === "content_block_delta" && 
          event.delta.type === "input_json_delta") {
        // 参数 JSON 在流式生成中
        this.inputBuffer += event.delta.partial_json;
      }
      
      if (event.type === "content_block_stop" && this.currentTool) {
        // 参数完整了，执行工具
        const params = JSON.parse(this.inputBuffer);
        const result = await this.executeToolWithStreaming(
          this.currentTool, 
          params,
          this.toolCallId!
        );
        this.currentTool = null;
      }
    }
  }
  
  private onToolDecided(toolName: string) {
    // 立即给用户反馈："正在搜索..."、"正在查询数据库..."
    const messages: Record<string, string> = {
      web_search: "🔍 正在搜索...",
      read_file: "📄 正在读取文件...",
      exec: "⚡ 正在执行命令...",
    };
    updateStatusBar(messages[toolName] ?? `🔧 调用 ${toolName}...`);
  }
  
  private async executeToolWithStreaming(
    name: string,
    params: Record<string, unknown>,
    id: string
  ): Promise<ToolResult> {
    // 工具执行期间持续更新进度
    return await withProgress(name, () => executeTool(name, params, id));
  }
}
```

---

## Part 3：渐进输出模式

### 3.1 多工具并行流式

多个工具同时执行，结果按完成顺序流出（不是等全部完成再一起给）：

```typescript
class ProgressiveResultAggregator {
  private results = new Map<string, unknown>();
  private pending = new Set<string>();
  
  async executeParallel(
    tools: Array<{ name: string; params: unknown }>,
    onPartialResult: (name: string, result: unknown) => void
  ) {
    const promises = tools.map(async ({ name, params }) => {
      this.pending.add(name);
      
      try {
        const result = await executeTool(name, params);
        this.results.set(name, result);
        this.pending.delete(name);
        
        // 🔑 每个工具完成就立即回调，不等其他工具
        onPartialResult(name, result);
        
        return { name, result };
      } catch (err) {
        this.pending.delete(name);
        onPartialResult(name, { error: (err as Error).message });
      }
    });
    
    await Promise.allSettled(promises);
  }
}

// 使用示例
const aggregator = new ProgressiveResultAggregator();

await aggregator.executeParallel(
  [
    { name: "web_search", params: { query: "Claude 3.5 benchmark" } },
    { name: "read_file", params: { path: "notes.md" } },
    { name: "exec", params: { command: "git log --oneline -5" } },
  ],
  (name, result) => {
    // 每个工具完成时立即更新 UI
    console.log(`✅ ${name} 完成`);
    updateUI(name, result);
  }
);
```

### 3.2 流式 Markdown 渲染

文字流式输出时，边流边渲染 Markdown，不等完整内容：

```typescript
class StreamingMarkdownRenderer {
  private buffer = "";
  private lastRenderedLength = 0;
  
  // 积累 token，达到安全断点才渲染
  appendToken(token: string) {
    this.buffer += token;
    
    // 找安全断点：换行符、句号、段落
    const safeBreak = this.findSafeBreak();
    
    if (safeBreak > this.lastRenderedLength) {
      const toRender = this.buffer.slice(0, safeBreak);
      this.renderMarkdown(toRender);
      this.lastRenderedLength = safeBreak;
    }
  }
  
  private findSafeBreak(): number {
    // 从后往前找换行符（不在代码块内）
    let inCodeBlock = false;
    for (let i = this.buffer.length - 1; i >= this.lastRenderedLength; i--) {
      if (this.buffer[i] === "\n") {
        // 检查是否在代码块内（简化版）
        const before = this.buffer.slice(0, i);
        const codeBlockCount = (before.match(/```/g) || []).length;
        if (codeBlockCount % 2 === 0) {
          return i;
        }
      }
    }
    return this.lastRenderedLength;
  }
  
  flush() {
    // 流结束时，强制渲染剩余内容
    this.renderMarkdown(this.buffer);
  }
  
  private renderMarkdown(content: string) {
    // 调用 markdown 渲染器（如 marked.js）
    const html = marked(content);
    document.getElementById("output")!.innerHTML = html;
  }
}
```

---

## Part 4：OpenClaw 中的流式 Agent Loop

OpenClaw 实际的流式处理架构（简化版）：

```
用户消息
    │
    ▼
┌─────────────────────────────────────────┐
│         Streaming Agent Loop             │
│                                          │
│  LLM Stream ──────────────────────────► │
│     │                                    │
│     ├─ thinking_delta → ThinkingBox      │
│     ├─ text_delta → MessageBubble        │
│     └─ tool_use → ToolStatusBar          │
│                                          │
│  Tool Execution (parallel)               │
│     │                                    │
│     ├─ progress → ProgressIndicator      │
│     └─ result → ToolResultCard           │
│                                          │
│  Next LLM Turn (with all results)        │
└─────────────────────────────────────────┘
```

实际代码结构参考（pi-mono 风格）：

```typescript
async function* streamingAgentLoop(
  messages: Message[],
  tools: Tool[]
): AsyncGenerator<AgentEvent> {
  while (true) {
    const stream = anthropic.messages.stream({
      model: "claude-opus-4-5",
      max_tokens: 8192,
      thinking: { type: "enabled", budget_tokens: 5000 },
      tools,
      messages,
    });
    
    const toolUses: ToolUse[] = [];
    
    for await (const event of stream) {
      // 实时 yield 给调用方
      if (event.type === "content_block_delta") {
        if (event.delta.type === "thinking_delta") {
          yield { type: "thinking", chunk: event.delta.thinking };
        } else if (event.delta.type === "text_delta") {
          yield { type: "text", chunk: event.delta.text };
        } else if (event.delta.type === "input_json_delta") {
          yield { type: "tool_input", chunk: event.delta.partial_json };
        }
      }
      
      if (event.type === "content_block_start" &&
          event.content_block.type === "tool_use") {
        yield { type: "tool_start", name: event.content_block.name };
      }
    }
    
    const finalMessage = await stream.finalMessage();
    
    // 检查是否需要调用工具
    const toolUseCalls = finalMessage.content.filter(
      (b) => b.type === "tool_use"
    ) as ToolUse[];
    
    if (toolUseCalls.length === 0) break;  // 没有工具调用，结束
    
    // 并行执行所有工具，每个完成就 yield
    const toolResults = await Promise.all(
      toolUseCalls.map(async (tu) => {
        yield { type: "tool_executing", name: tu.name };
        const result = await executeTool(tu);
        yield { type: "tool_done", name: tu.name, result };
        return { tool_use_id: tu.id, content: JSON.stringify(result) };
      })
    );
    
    // 把工具结果追加到消息历史，继续下一轮
    messages = [
      ...messages,
      { role: "assistant", content: finalMessage.content },
      { role: "user", content: toolResults.map(r => ({ type: "tool_result", ...r })) },
    ];
  }
}

// 消费这个 Generator
for await (const event of streamingAgentLoop(messages, tools)) {
  switch (event.type) {
    case "thinking": process.stdout.write(`💭 ${event.chunk}`); break;
    case "text": process.stdout.write(event.chunk); break;
    case "tool_start": console.log(`\n🔧 调用工具: ${event.name}`); break;
    case "tool_done": console.log(`✅ ${event.name} 完成`); break;
  }
}
```

---

## Part 5：实战技巧

### 5.1 首 Token 时间（TTFT）优化

```typescript
// ❌ 慢：先做一堆处理，再开始流
const data = await fetchContext();     // 500ms
const enriched = await enrich(data);  // 300ms  
const stream = await llm.stream();    // 才开始流

// ✅ 快：立即开始流，数据在流的过程中准备好
const [stream, contextPromise] = await Promise.all([
  llm.stream({ messages: partialMessages }),  // 立即开始
  fetchAndEnrichContext(),                     // 同步准备
]);

// 如果 LLM 需要 context 才能继续（比如 RAG），
// 用 tool call 模式让它"暂停"等数据
```

### 5.2 流式输出的背压（Backpressure）

```typescript
// 如果下游处理（渲染、发送）比 LLM 产出慢，需要背压
class BackpressuredStream {
  private queue: string[] = [];
  private processing = false;
  
  async push(chunk: string) {
    this.queue.push(chunk);
    
    if (!this.processing) {
      await this.drain();
    }
  }
  
  private async drain() {
    this.processing = true;
    
    while (this.queue.length > 0) {
      const chunk = this.queue.shift()!;
      await this.sendToClient(chunk);  // 等待发送完成
    }
    
    this.processing = false;
  }
  
  private async sendToClient(chunk: string) {
    // WebSocket / SSE 发送
    await ws.send(JSON.stringify({ type: "chunk", data: chunk }));
  }
}
```

### 5.3 流式中断与取消

```typescript
const controller = new AbortController();

// 用户点击"停止"
stopButton.onclick = () => controller.abort();

try {
  const stream = await anthropic.messages.stream(
    { model: "claude-opus-4-5", ... },
    { signal: controller.signal }  // 传入 AbortSignal
  );
  
  for await (const event of stream) {
    // 如果被取消，这里会抛出 AbortError
    processEvent(event);
  }
} catch (err) {
  if ((err as Error).name === "AbortError") {
    console.log("用户取消了生成");
    // 保留已生成的部分内容
    displayPartialResult(partialContent);
  }
}
```

---

## 总结

| 技术 | 解决问题 | 用户体验提升 |
|------|----------|-------------|
| Streaming Thinking | 长思考时白屏 | 看到 AI 在"思考" |
| 流式 Tool Input | 工具决策延迟 | 立即知道 AI 要做什么 |
| 并行工具渐进输出 | 多工具串行等待 | 最快的结果先出现 |
| 流式 Markdown 渲染 | 等完整内容才渲染 | 边读边看，自然流畅 |
| 首 Token 时间优化 | 请求到显示延迟高 | < 2s 看到第一个字 |

**核心原则**：不要让用户等待你做完再说——**边做边说**。

---

## 参考

- [Anthropic Streaming Docs](https://docs.anthropic.com/en/api/messages-streaming)
- [Extended Thinking](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking)
- OpenClaw Runtime: `model=anthropic/claude-sonnet-4-6 | thinking=adaptive`
- pi-mono: `packages/claude/src/streaming/`
