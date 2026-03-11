# Agent Handoff：Agent 交接机制

> 在多 Agent 系统中，如何优雅地将控制权从一个 Agent 传递给另一个？

## 为什么需要 Agent Handoff？

当你构建复杂的 Agent 系统时，单个 Agent 往往无法胜任所有任务：

```
用户: "帮我分析这份财报，然后生成一份 PPT"

❌ 单 Agent 方案：一个通用 Agent 做所有事（质量差）
✅ 多 Agent 方案：分析 Agent → 交接 → 写作 Agent → 交接 → PPT Agent
```

**Handoff 的核心问题**：
1. **When** - 什么时候交接？
2. **What** - 传递什么上下文？
3. **How** - 怎么交接？
4. **Back** - 如何返回结果？

## 交接模式对比

### 模式一：直接交接（Fire and Forget）

```typescript
// pi-mono 风格
async function handleTask(task: Task) {
  // 分析阶段
  const analysis = await analyzeAgent.run(task.data);
  
  // 直接交接给写作 Agent
  const report = await writingAgent.run({
    type: 'generate_report',
    data: analysis.result,
    // 不传递对话历史，只传递结果
  });
  
  return report;
}
```

**特点**：
- ✅ 简单直接
- ✅ 上下文隔离
- ❌ 丢失对话历史
- ❌ 无法回溯

### 模式二：带上下文的交接

```typescript
// OpenClaw sessions_spawn 风格
interface HandoffContext {
  originalTask: string;           // 原始任务
  conversationHistory: Message[]; // 对话历史
  intermediateResults: any;       // 中间结果
  metadata: {
    parentSession: string;
    handoffChain: string[];       // 交接链路
  };
}

async function handoff(
  targetAgent: string,
  context: HandoffContext
) {
  return await sessions_spawn({
    agentId: targetAgent,
    task: context.originalTask,
    // 通过 attachments 传递上下文
    attachments: [{
      name: 'context.json',
      content: JSON.stringify(context),
      mimeType: 'application/json'
    }],
    mode: 'run',  // 一次性任务
    streamTo: 'parent'  // 结果流回父 Agent
  });
}
```

### 模式三：协作式交接（Handoff with Collaboration）

```typescript
// 高级模式：交接后仍保持通信
class CollaborativeHandoff {
  private sessions: Map<string, Session> = new Map();
  
  async initiateHandoff(
    from: Agent,
    to: Agent,
    task: Task
  ): Promise<HandoffHandle> {
    const sessionId = uuid();
    
    // 创建持久会话
    const session = await sessions_spawn({
      agentId: to.id,
      mode: 'session',  // 持久会话
      label: `handoff-${sessionId}`,
      task: task.description,
    });
    
    this.sessions.set(sessionId, session);
    
    return {
      sessionId,
      // 允许原 Agent 继续发送消息
      send: (msg: string) => sessions_send({
        label: `handoff-${sessionId}`,
        message: msg
      }),
      // 监听结果
      onComplete: (callback) => {
        // 订阅完成事件
      }
    };
  }
}
```

## 上下文传递策略

### 策略一：最小上下文

只传递完成任务必需的信息：

```typescript
// learn-claude-code 风格
interface MinimalContext {
  task: string;        // 当前子任务
  input: any;          // 输入数据
  constraints?: {      // 约束条件
    maxTokens?: number;
    format?: string;
  };
}

// 不传递：对话历史、父 Agent 状态、其他任务信息
```

**适用场景**：独立子任务、敏感信息隔离

### 策略二：摘要上下文

传递压缩后的历史：

```typescript
// 上下文压缩
async function summarizeForHandoff(
  history: Message[],
  maxTokens: number = 500
): Promise<string> {
  const summary = await llm.generate({
    prompt: `Summarize this conversation for a handoff:
${formatMessages(history)}

Focus on:
1. What was the original request?
2. What has been accomplished?
3. What remains to be done?
4. Any important constraints or preferences?`,
    maxTokens
  });
  
  return summary;
}

// 使用
const context = await summarizeForHandoff(conversation.history);
await handoff(nextAgent, { summary: context, task: remainingTask });
```

### 策略三：结构化上下文

定义清晰的交接协议：

```typescript
// OpenClaw 实际使用的模式
interface StructuredHandoffContext {
  // 任务定义
  task: {
    id: string;
    description: string;
    expectedOutput: string;
  };
  
  // 已完成的工作
  progress: {
    completedSteps: string[];
    artifacts: Array<{
      name: string;
      type: string;
      location: string;  // 文件路径或 URL
    }>;
  };
  
  // 环境信息
  environment: {
    workdir: string;
    availableTools: string[];
    constraints: Record<string, any>;
  };
  
  // 交接说明
  handoff: {
    reason: string;           // 为什么交接
    expectations: string[];   // 期望下一个 Agent 做什么
    returnFormat: string;     // 期望的返回格式
  };
}
```

## 实战：OpenClaw 的交接实现

### sessions_spawn 交接

```typescript
// 真实使用场景：代码审查任务交接
const reviewResult = await sessions_spawn({
  runtime: 'subagent',
  agentId: 'code-reviewer',
  task: `Review the following PR:
Repository: ${repo}
PR #${prNumber}

Previous analysis found these issues:
${JSON.stringify(staticAnalysis.issues)}

Please provide:
1. Security assessment
2. Performance concerns
3. Code quality suggestions`,
  
  attachments: [
    {
      name: 'diff.patch',
      content: prDiff,
      mimeType: 'text/x-patch'
    },
    {
      name: 'context.md',
      content: projectContext,
      mimeType: 'text/markdown'
    }
  ],
  
  mode: 'run',
  runTimeoutSeconds: 300,
});
```

### 链式交接

```typescript
// 多 Agent 流水线
async function processDocument(doc: Document) {
  // Step 1: 提取 Agent
  const extraction = await sessions_spawn({
    agentId: 'extractor',
    task: `Extract key information from: ${doc.content}`,
    mode: 'run'
  });
  
  // Step 2: 分析 Agent（基于提取结果）
  const analysis = await sessions_spawn({
    agentId: 'analyzer',
    task: `Analyze this extracted data`,
    attachments: [{
      name: 'extracted.json',
      content: extraction.result,
      mimeType: 'application/json'
    }],
    mode: 'run'
  });
  
  // Step 3: 生成 Agent（基于分析结果）
  const report = await sessions_spawn({
    agentId: 'report-generator',
    task: `Generate a report from this analysis`,
    attachments: [{
      name: 'analysis.json',
      content: analysis.result,
      mimeType: 'application/json'
    }],
    mode: 'run'
  });
  
  return report;
}
```

## 交接的错误处理

```typescript
// 健壮的交接实现
async function robustHandoff(
  targetAgent: string,
  context: HandoffContext,
  options: { retries?: number; fallbackAgent?: string } = {}
): Promise<HandoffResult> {
  const { retries = 2, fallbackAgent } = options;
  
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      const result = await sessions_spawn({
        agentId: targetAgent,
        task: context.task,
        attachments: context.attachments,
        mode: 'run',
        runTimeoutSeconds: 300,
      });
      
      // 验证结果
      if (isValidResult(result)) {
        return { success: true, result, agent: targetAgent };
      }
      
      // 结果无效，记录并重试
      console.warn(`Invalid result from ${targetAgent}, attempt ${attempt + 1}`);
      
    } catch (error) {
      if (attempt === retries && fallbackAgent) {
        // 切换到备用 Agent
        return robustHandoff(fallbackAgent, context, { retries: 1 });
      }
      
      if (attempt === retries) {
        // 交接失败，返回控制权给父 Agent
        return {
          success: false,
          error: `Handoff to ${targetAgent} failed after ${retries + 1} attempts`,
          context: context,  // 返回上下文以便人工处理
        };
      }
    }
  }
}
```

## 交接 vs 其他模式

| 模式 | 适用场景 | 上下文传递 | 复杂度 |
|------|----------|------------|--------|
| **Handoff** | 专业任务分工 | 结构化传递 | 中等 |
| **Subagent** | 子任务分解 | 自动继承 | 低 |
| **Team** | 协作讨论 | 共享上下文 | 高 |
| **Pipeline** | 顺序处理 | 流式传递 | 低 |

## Best Practices

### 1. 明确交接边界

```typescript
// ❌ 模糊的交接
await handoff(agent, "继续处理这个任务");

// ✅ 清晰的交接
await handoff(agent, {
  task: "将分析结果转换为 Markdown 报告",
  input: analysisData,
  expectedOutput: "markdown_file",
  constraints: {
    maxLength: 2000,
    includeCharts: false
  }
});
```

### 2. 保持交接链可追溯

```typescript
interface HandoffChain {
  id: string;
  steps: Array<{
    agent: string;
    startTime: Date;
    endTime?: Date;
    input: string;  // 摘要
    output: string; // 摘要
    status: 'running' | 'completed' | 'failed';
  }>;
}

// 每次交接记录到链中
function recordHandoff(chain: HandoffChain, step: HandoffStep) {
  chain.steps.push(step);
  // 持久化到文件或数据库
}
```

### 3. 设计清晰的返回路径

```typescript
// 定义返回格式
interface HandoffReturn {
  status: 'completed' | 'partial' | 'failed';
  result?: any;
  artifacts?: string[];  // 产出的文件路径
  continuation?: {
    // 如果需要继续处理
    remainingTask: string;
    suggestedAgent: string;
  };
  error?: string;
}
```

## 总结

**Agent Handoff 三要素**：

1. **上下文** - 传递什么、传递多少
2. **协议** - 输入输出格式、错误处理
3. **追溯** - 记录交接链路，便于调试

**核心原则**：

- 最小必要上下文
- 结构化传递格式
- 清晰的期望和约束
- 健壮的错误处理
- 可追溯的交接链

下一课我们将探讨 **Agent Orchestration Patterns** - 更高层次的多 Agent 编排模式。
