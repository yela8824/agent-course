# 49 - Agent-to-Agent (A2A) 协议：让不同 Agent 框架互联互通

> **难度**: ⭐⭐⭐⭐  
> **分类**: 协议/互操作性  
> **关键词**: A2A, ACP, Agent 互操作, 跨框架通信, Task Protocol

---

## 一、为什么需要 A2A？

你用 Claude Code 做代码审查，用 Gemini Agent 做数据分析，用自建 Pi Agent 处理业务逻辑——

**问题来了：这些 Agent 彼此不认识，没法协作。**

每个框架有自己的 API 格式，任务分发靠人肉拷贝粘贴，根本谈不上"多 Agent 系统"。

A2A（Agent-to-Agent Protocol）就是来解决这个问题的：

```
[Claude Code Agent] ──A2A──► [Gemini Agent]
                                    │
                               [Pi Agent]
                                    │
                            [自定义 Tool Agent]
```

**核心思路**: 所有 Agent 暴露一个标准 HTTP 接口，用统一的 JSON 格式交换任务和结果。

---

## 二、A2A 核心概念

### 2.1 Agent Card（身份证）

每个 A2A Agent 都有一张 `/.well-known/agent.json`：

```json
{
  "name": "DataAnalysis Agent",
  "version": "1.0.0",
  "description": "专注数据分析、SQL 查询和报表生成",
  "url": "https://agents.example.com/data",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true,
    "stateTransitionHistory": false
  },
  "skills": [
    {
      "id": "analyze-data",
      "name": "数据分析",
      "description": "执行 SQL 查询并生成报表",
      "inputModes": ["text"],
      "outputModes": ["text", "file"]
    }
  ],
  "authentication": {
    "schemes": ["Bearer"]
  }
}
```

### 2.2 Task（任务单）

A2A 的核心数据结构：

```typescript
interface Task {
  id: string;           // 任务唯一 ID
  sessionId?: string;   // 会话 ID（多轮对话）
  status: TaskStatus;   // submitted | working | input-required | completed | failed | canceled
  message: Message;     // 输入消息
  artifacts?: Artifact[]; // 输出产物
  history?: Message[];  // 消息历史
  metadata?: Record<string, unknown>;
}

interface Message {
  role: "user" | "agent";
  parts: Part[];        // 多模态内容
}

type Part = 
  | { type: "text"; text: string }
  | { type: "file"; file: FileContent }
  | { type: "data"; data: Record<string, unknown> };
```

### 2.3 Task 生命周期

```
submitted ──► working ──► completed
    │              │
    │         input-required ──► working (再次)
    │
    └──► failed
    └──► canceled
```

---

## 三、实战：A2A Server（Pi 风格实现）

### 3.1 最简 A2A Server

```typescript
// a2a-server.ts - 基于 Hono 的 A2A Agent 服务
import { Hono } from "hono";

const app = new Hono();

// 1. 暴露 Agent Card
app.get("/.well-known/agent.json", (c) => {
  return c.json({
    name: "MyAgent",
    version: "1.0.0",
    url: "http://localhost:3000",
    capabilities: { streaming: true },
    skills: [
      {
        id: "summarize",
        name: "文本摘要",
        description: "将长文本压缩为要点",
      },
    ],
  });
});

// 2. 任务提交端点
app.post("/tasks/send", async (c) => {
  const task: Task = await c.req.json();
  
  // 提取用户输入
  const userText = task.message.parts
    .filter((p) => p.type === "text")
    .map((p) => p.text)
    .join("\n");

  // 执行实际工作（调 LLM、工具等）
  const result = await processWithAgent(userText);

  // 返回完成的 Task
  return c.json({
    ...task,
    status: "completed",
    artifacts: [
      {
        name: "result",
        parts: [{ type: "text", text: result }],
      },
    ],
  });
});

// 3. 流式端点（SSE）
app.post("/tasks/sendSubscribe", async (c) => {
  const task: Task = await c.req.json();
  
  return c.stream(async (stream) => {
    // 发送 working 状态
    await stream.write(
      `data: ${JSON.stringify({ type: "status", status: "working", taskId: task.id })}\n\n`
    );
    
    // 流式处理
    for await (const chunk of processStreaming(task.message)) {
      await stream.write(
        `data: ${JSON.stringify({ 
          type: "artifact", 
          artifact: { parts: [{ type: "text", text: chunk }] }
        })}\n\n`
      );
    }
    
    // 完成
    await stream.write(
      `data: ${JSON.stringify({ type: "status", status: "completed", taskId: task.id })}\n\n`
    );
  });
});
```

### 3.2 A2A Client（调用其他 Agent）

```typescript
// a2a-client.ts
class A2AClient {
  private agentCard: AgentCard;

  constructor(private agentUrl: string) {}

  // 发现 Agent 能力
  async discover(): Promise<AgentCard> {
    const res = await fetch(`${this.agentUrl}/.well-known/agent.json`);
    this.agentCard = await res.json();
    return this.agentCard;
  }

  // 发送任务（非流式）
  async sendTask(message: string, sessionId?: string): Promise<Task> {
    const task: Task = {
      id: crypto.randomUUID(),
      sessionId,
      status: "submitted",
      message: {
        role: "user",
        parts: [{ type: "text", text: message }],
      },
    };

    const res = await fetch(`${this.agentUrl}/tasks/send`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(task),
    });

    return res.json();
  }

  // 发送任务（流式接收）
  async *sendTaskStreaming(message: string): AsyncGenerator<string> {
    const res = await fetch(`${this.agentUrl}/tasks/sendSubscribe`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        id: crypto.randomUUID(),
        status: "submitted",
        message: { role: "user", parts: [{ type: "text", text: message }] },
      }),
    });

    const reader = res.body!.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      
      const lines = decoder.decode(value).split("\n");
      for (const line of lines) {
        if (line.startsWith("data: ")) {
          const event = JSON.parse(line.slice(6));
          if (event.type === "artifact") {
            yield event.artifact.parts[0].text;
          }
        }
      }
    }
  }
}

// 使用示例
const client = new A2AClient("https://data-agent.example.com");
await client.discover();

const task = await client.sendTask("分析过去 7 天的开箱数据，按利润排序");
console.log(task.artifacts[0].parts[0].text);
```

---

## 四、OpenClaw 里的 ACP（A2A 的兄弟）

OpenClaw 实现的是 ACP（Agent Control Protocol），思路类似但更贴近 OpenClaw 生态：

```typescript
// OpenClaw ACP session 模式
// 创建持久会话（而不是一次性任务）
const session = await sessions_spawn({
  runtime: "acp",
  agentId: "claude-code",
  mode: "session",       // 持久会话
  thread: true,          // 绑定到当前 thread
  task: "帮我 review 这个 PR",
});

// ACP vs A2A 对比：
// A2A: 无状态 HTTP 请求/响应，适合微服务
// ACP: 有状态会话，适合 chat-thread 场景
```

**关键区别**：

| | A2A | ACP (OpenClaw) |
|---|---|---|
| **状态** | 无状态（task 为单位）| 有状态（session 为单位）|
| **传输** | HTTP REST + SSE | WebSocket / HTTP |
| **发现** | Agent Card | agents_list() |
| **适合** | 微服务间调用 | Chat 场景多轮对话 |

---

## 五、Pi-mono 的实现方式

Pi-mono 采用了更轻量的内部协议：

```typescript
// pi-mono: Agent 作为工具调用另一个 Agent
const tools = [
  {
    name: "delegate_to_researcher",
    description: "将研究任务委托给 Researcher Agent",
    parameters: {
      type: "object",
      properties: {
        query: { type: "string", description: "研究问题" },
        depth: { type: "string", enum: ["quick", "deep"] },
      },
    },
    // 工具背后是另一个 Agent 实例
    handler: async ({ query, depth }) => {
      const researcherAgent = new Agent({
        systemPrompt: "你是一个专业研究员...",
        tools: [webSearch, webFetch],
      });
      
      const result = await researcherAgent.run(query);
      return result.lastMessage;
    },
  },
];

// 主 Agent 透明地调用 Researcher Agent
const orchestratorAgent = new Agent({
  systemPrompt: "你是协调者，遇到需要研究的问题就调用 delegate_to_researcher",
  tools,
});
```

---

## 六、多 Agent 编排实战

```typescript
// 三个 Agent 协作完成一个复杂任务
async function runMultiAgentPipeline(userRequest: string) {
  // 1. Planner Agent：分解任务
  const planner = new A2AClient("http://planner-agent:3000");
  const plan = await planner.sendTask(
    `将以下任务分解为子任务（JSON 格式）:\n${userRequest}`
  );
  
  const subtasks = JSON.parse(
    plan.artifacts[0].parts[0].text
  );

  // 2. Worker Agents：并发执行子任务
  const workers = [
    new A2AClient("http://researcher-agent:3001"),
    new A2AClient("http://coder-agent:3002"),
    new A2AClient("http://analyst-agent:3003"),
  ];

  // 根据子任务类型分配 worker
  const results = await Promise.all(
    subtasks.map((subtask) => {
      const worker = pickWorker(workers, subtask.type);
      return worker.sendTask(subtask.description);
    })
  );

  // 3. Synthesizer Agent：整合结果
  const synthesizer = new A2AClient("http://synthesizer-agent:3004");
  const final = await synthesizer.sendTask(
    `整合以下结果，生成最终报告:\n${JSON.stringify(results)}`
  );

  return final.artifacts[0].parts[0].text;
}

function pickWorker(workers: A2AClient[], taskType: string): A2AClient {
  const mapping = {
    research: workers[0],
    code: workers[1],
    analysis: workers[2],
  };
  return mapping[taskType] ?? workers[0];
}
```

---

## 七、生产注意事项

### 7.1 认证

```typescript
// A2A 请求要带 Bearer Token
const res = await fetch(`${agentUrl}/tasks/send`, {
  headers: {
    "Authorization": `Bearer ${agentToken}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify(task),
});
```

### 7.2 任务超时与取消

```typescript
// 带超时的任务发送
async function sendTaskWithTimeout(client: A2AClient, message: string, timeoutMs = 30000) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  
  try {
    return await client.sendTask(message, { signal: controller.signal });
  } finally {
    clearTimeout(timer);
  }
}

// 取消进行中的任务
await fetch(`${agentUrl}/tasks/${taskId}/cancel`, { method: "POST" });
```

### 7.3 幂等性

```typescript
// 用固定 taskId 确保重试安全
const taskId = `${sessionId}-${subtaskIndex}`; // 不用 randomUUID
const task = await client.sendTask(message, { id: taskId });
// 如果网络中断重试，同一 taskId 不会重复执行
```

---

## 八、总结

```
单体 Agent           多 Agent (A2A)
    │                     │
  一个脑袋          专业分工协作
  干所有事          按能力路由任务
  上下文膨胀        上下文隔离清晰
  扩展难            独立部署扩展
```

**A2A 的价值不在于技术有多复杂，而在于它让 "Agent 是一等公民" 成为现实** ——就像 HTTP 让服务互联，A2A 让 Agent 互联。

OpenClaw 的 ACP、Google 的 A2A、Anthropic 的 MCP，方向相同，生态各异。理解核心协议模式，你就能在任何框架里实现 Agent 互操作。

---

**参考资源**:
- [Google A2A Protocol Spec](https://google.github.io/A2A/)
- [OpenClaw ACP 实现](https://github.com/openclaw/openclaw)
- [pi-mono Agent 示例](https://github.com/badlogic/pi-mono)
