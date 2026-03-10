# 第 18 课：Agent Debugging & Troubleshooting - Agent 调试与排错

> 调试 Agent 比调试普通程序更难 —— 你不仅要调试代码，还要理解 LLM 为什么做出那个决定。

## 为什么 Agent 调试特别难？

传统软件调试：代码是确定性的，同样的输入永远产生同样的输出。

Agent 调试：
1. **非确定性** - LLM 输出有随机性（temperature）
2. **黑盒决策** - 你能看到输入输出，但不知道"思考过程"
3. **长链依赖** - 一个工具调用失败可能影响后续 10 个决策
4. **上下文敏感** - 同样的指令，不同上下文下行为完全不同

## 核心调试策略

### 1. 日志分级 (Structured Logging)

不是所有日志都同等重要。Agent 系统需要精心设计的日志层级：

```typescript
// pi-mono 风格的日志设计
interface AgentLog {
  level: 'trace' | 'debug' | 'info' | 'warn' | 'error';
  category: 'llm' | 'tool' | 'context' | 'decision';
  sessionId: string;
  turnId: string;
  timestamp: number;
  data: unknown;
}

class AgentLogger {
  // LLM 交互日志 - 最重要
  logLLMRequest(messages: Message[], config: LLMConfig) {
    this.log({
      level: 'debug',
      category: 'llm',
      data: {
        messageCount: messages.length,
        totalTokens: this.estimateTokens(messages),
        model: config.model,
        temperature: config.temperature,
        // 完整 messages 存入 trace 级别
      }
    });
  }

  logLLMResponse(response: LLMResponse, latencyMs: number) {
    this.log({
      level: 'info', 
      category: 'llm',
      data: {
        hasToolCalls: response.toolCalls?.length > 0,
        toolNames: response.toolCalls?.map(t => t.name),
        tokensUsed: response.usage,
        latencyMs,
        stopReason: response.stopReason,
      }
    });
  }

  // 工具执行日志
  logToolExecution(tool: string, args: unknown, result: unknown, error?: Error) {
    this.log({
      level: error ? 'error' : 'debug',
      category: 'tool',
      data: {
        tool,
        args: this.sanitize(args), // 脱敏
        resultPreview: this.truncate(result, 500),
        error: error?.message,
        stack: error?.stack,
      }
    });
  }

  // 决策日志 - 关键调试信息
  logDecision(decision: string, reasoning: string, context: unknown) {
    this.log({
      level: 'info',
      category: 'decision',
      data: { decision, reasoning, contextSnapshot: context }
    });
  }
}
```

### 2. 会话回放 (Session Replay)

保存完整会话，支持"时间旅行"调试：

```typescript
// OpenClaw 风格的会话快照
interface SessionSnapshot {
  id: string;
  timestamp: number;
  messages: Message[];          // 完整对话历史
  toolResults: ToolResult[];    // 所有工具调用结果
  contextState: {
    tokensUsed: number;
    compactionCount: number;
    activeMemories: string[];
  };
  config: AgentConfig;          // 当时的配置
}

class SessionRecorder {
  private snapshots: SessionSnapshot[] = [];
  
  // 每次 LLM 调用前拍快照
  snapshot(session: Session): void {
    this.snapshots.push({
      id: randomId(),
      timestamp: Date.now(),
      messages: structuredClone(session.messages),
      toolResults: structuredClone(session.toolResults),
      contextState: session.getContextState(),
      config: session.config,
    });
  }

  // 回放到某个时间点
  replayTo(snapshotId: string): Session {
    const snapshot = this.snapshots.find(s => s.id === snapshotId);
    if (!snapshot) throw new Error('Snapshot not found');
    
    return Session.fromSnapshot(snapshot);
  }

  // 导出供离线分析
  export(): string {
    return JSON.stringify(this.snapshots, null, 2);
  }
}
```

### 3. 思维链可视化 (Chain of Thought Visualization)

当 Agent 使用 thinking/extended thinking 时，这些内容是调试金矿：

```typescript
interface ThinkingTrace {
  turnId: string;
  thinking: string;           // 原始思考内容
  extractedIntents: string[]; // 解析出的意图
  plannedActions: string[];   // 计划的动作
  actualActions: string[];    // 实际执行的动作
  divergence?: string;        // 计划与实际的差异
}

function analyzeThinking(response: LLMResponse): ThinkingTrace {
  const thinking = response.thinking || '';
  
  return {
    turnId: response.turnId,
    thinking,
    extractedIntents: extractIntents(thinking),
    plannedActions: extractPlannedActions(thinking),
    actualActions: response.toolCalls?.map(t => t.name) || [],
    divergence: detectDivergence(thinking, response.toolCalls),
  };
}

// 检测 "说一套做一套"
function detectDivergence(thinking: string, actions: ToolCall[]): string | undefined {
  // 如果思考中说 "我应该先读取文件"
  // 但实际调用了 write 工具
  // 这就是需要关注的 divergence
  
  const mentioned = extractMentionedTools(thinking);
  const actual = new Set(actions.map(a => a.name));
  
  const notExecuted = mentioned.filter(t => !actual.has(t));
  const unexpected = [...actual].filter(t => !mentioned.includes(t));
  
  if (notExecuted.length || unexpected.length) {
    return `Planned but not executed: ${notExecuted.join(', ')}. ` +
           `Unexpected: ${unexpected.join(', ')}`;
  }
  return undefined;
}
```

## 常见问题诊断

### 问题 1：Agent 陷入循环

**症状**：Agent 反复调用同一个工具，或者重复相同的错误

**诊断方法**：

```typescript
class LoopDetector {
  private history: string[] = [];
  private readonly windowSize = 10;
  
  check(action: string): { isLoop: boolean; pattern?: string } {
    this.history.push(action);
    if (this.history.length > this.windowSize) {
      this.history.shift();
    }
    
    // 检测重复模式
    // A-B-A-B-A-B (周期 2)
    // A-B-C-A-B-C (周期 3)
    for (let period = 1; period <= this.windowSize / 2; period++) {
      if (this.hasRepeatingPattern(period)) {
        return {
          isLoop: true,
          pattern: this.history.slice(-period).join(' -> ')
        };
      }
    }
    
    return { isLoop: false };
  }

  private hasRepeatingPattern(period: number): boolean {
    if (this.history.length < period * 2) return false;
    
    const recent = this.history.slice(-period * 2);
    const first = recent.slice(0, period).join(',');
    const second = recent.slice(period).join(',');
    
    return first === second;
  }
}

// 使用
const detector = new LoopDetector();
for (const toolCall of session.toolCalls) {
  const result = detector.check(toolCall.name);
  if (result.isLoop) {
    logger.warn('Loop detected', { pattern: result.pattern });
    // 可以自动干预：注入 "你似乎在重复同样的操作" 提示
  }
}
```

**根因分析**：
- 工具返回了不够明确的错误信息
- System prompt 没有处理"失败后该怎么办"
- Context 中缺少之前失败的记录

### 问题 2：Agent 做出了意外决策

**症状**：用户说 "帮我发邮件"，Agent 却去读文件了

**诊断 Checklist**：

```typescript
async function diagnoseUnexpectedDecision(
  session: Session,
  expectedAction: string,
  actualAction: string
): Promise<DiagnosisReport> {
  const report: DiagnosisReport = { issues: [] };
  
  // 1. 检查最近的 messages
  const recentMessages = session.messages.slice(-5);
  const hasConflictingInstructions = recentMessages.some(m => 
    m.content.toLowerCase().includes('instead') ||
    m.content.toLowerCase().includes('actually') ||
    m.content.toLowerCase().includes('不要') ||
    m.content.toLowerCase().includes('别')
  );
  if (hasConflictingInstructions) {
    report.issues.push('Recent messages contain potentially conflicting instructions');
  }

  // 2. 检查 system prompt 优先级
  const systemPrompt = session.getSystemPrompt();
  const toolMentions = countToolMentions(systemPrompt);
  if (toolMentions[actualAction] > toolMentions[expectedAction]) {
    report.issues.push(`System prompt mentions ${actualAction} more than ${expectedAction}`);
  }

  // 3. 检查工具可用性
  const availableTools = session.getAvailableTools();
  if (!availableTools.includes(expectedAction)) {
    report.issues.push(`Expected tool "${expectedAction}" is not available`);
  }

  // 4. 检查上下文是否被截断
  if (session.wasContextCompacted) {
    report.issues.push('Context was compacted - important info may have been lost');
  }

  // 5. 检查是否有工具调用失败历史
  const failedCalls = session.toolResults.filter(r => r.error);
  if (failedCalls.some(f => f.toolName === expectedAction)) {
    report.issues.push(`Tool "${expectedAction}" failed before, Agent may be avoiding it`);
  }

  return report;
}
```

### 问题 3：工具调用参数错误

**症状**：工具被调用了，但参数不对

**诊断工具**：

```typescript
// 比较 LLM 的原始输出和解析后的参数
function debugToolCallParsing(
  rawOutput: string,
  parsedCall: ToolCall,
  schema: ToolSchema
): ParsingDebug {
  const debug: ParsingDebug = {
    raw: rawOutput,
    parsed: parsedCall,
    issues: []
  };

  // 检查必需参数
  for (const required of schema.required || []) {
    if (!(required in parsedCall.arguments)) {
      debug.issues.push(`Missing required param: ${required}`);
    }
  }

  // 检查类型
  for (const [key, value] of Object.entries(parsedCall.arguments)) {
    const expectedType = schema.properties?.[key]?.type;
    const actualType = typeof value;
    
    if (expectedType && actualType !== expectedType) {
      // 特殊处理 array
      if (expectedType === 'array' && !Array.isArray(value)) {
        debug.issues.push(`${key}: expected array, got ${actualType}`);
      } else if (expectedType !== 'array') {
        debug.issues.push(`${key}: expected ${expectedType}, got ${actualType}`);
      }
    }
  }

  // 检查是否有 schema 中没定义的参数
  const definedParams = Object.keys(schema.properties || {});
  for (const key of Object.keys(parsedCall.arguments)) {
    if (!definedParams.includes(key)) {
      debug.issues.push(`Unexpected param: ${key}`);
    }
  }

  return debug;
}
```

## 调试工具集成

### VS Code 扩展思路

```typescript
// 概念：Agent Debug Adapter
interface AgentDebugSession {
  // 断点：在特定工具调用前暂停
  setToolBreakpoint(toolName: string): void;
  
  // 断点：在特定条件下暂停
  setConditionalBreakpoint(condition: (ctx: AgentContext) => boolean): void;
  
  // 单步执行
  stepOver(): Promise<void>;  // 执行一次 LLM 调用
  stepInto(): Promise<void>;  // 进入工具执行
  
  // 变量检查
  inspectContext(): ContextInspection;
  inspectMemory(): MemoryInspection;
  
  // 修改运行时状态
  injectMessage(message: Message): void;
  modifyContext(patch: Partial<AgentContext>): void;
}
```

### CLI 调试工具

```bash
# 假设的 agent-debug CLI

# 回放会话
agent-debug replay session-123.json --step

# 分析循环
agent-debug analyze loops session-123.json

# 比较两次运行
agent-debug diff session-123.json session-456.json

# 提取所有工具调用
agent-debug extract-tools session-123.json --format table
```

## 生产环境调试

生产环境不能随意打断点，需要依赖：

### 1. 采样日志

```typescript
class SampledLogger {
  private sampleRate: number;
  
  constructor(sampleRate = 0.01) { // 1% 采样
    this.sampleRate = sampleRate;
  }

  logVerbose(data: unknown): void {
    if (Math.random() < this.sampleRate) {
      // 完整记录这个会话的所有信息
      this.fullLog(data);
    }
  }

  // 错误始终完整记录
  logError(error: Error, context: unknown): void {
    this.fullLog({ error, context, stack: error.stack });
  }
}
```

### 2. 异常会话标记

```typescript
// 标记"有问题"的会话供后续分析
function markForReview(session: Session, reason: string): void {
  session.metadata.needsReview = true;
  session.metadata.reviewReason = reason;
  session.metadata.markedAt = Date.now();
  
  // 异步上传完整会话数据到分析系统
  uploadToAnalytics(session);
}

// 自动标记条件
const autoMarkConditions = [
  (s: Session) => s.turnCount > 50,                    // 对话太长
  (s: Session) => s.errorCount > 3,                    // 错误太多
  (s: Session) => s.hasNegativeFeedback,               // 用户不满意
  (s: Session) => s.loopDetected,                      // 检测到循环
  (s: Session) => s.unexpectedTermination,             // 意外终止
];
```

### 3. A/B 测试对比

```typescript
// 当怀疑某个改动导致问题时
// 部署两个版本，对比指标

interface ABTestMetrics {
  version: 'A' | 'B';
  successRate: number;
  avgTurns: number;
  avgLatency: number;
  loopRate: number;
  errorRate: number;
  userSatisfaction: number;
}

function analyzeABTest(
  metricsA: ABTestMetrics[],
  metricsB: ABTestMetrics[]
): ABTestReport {
  // 统计显著性检验
  // 找出哪个版本更好
  // 定位具体的差异点
}
```

## 实战案例：调试一个"不听话"的 Agent

场景：用户说"帮我订明天的机票"，Agent 却开始搜索酒店。

调试过程：

```typescript
// Step 1: 导出会话
const session = await getSession('session-problematic');
const snapshot = sessionRecorder.export();

// Step 2: 找到出问题的 turn
const problematicTurn = session.turns.find(t => 
  t.toolCalls.some(tc => tc.name === 'search_hotels')
);

// Step 3: 检查那个 turn 的输入
console.log('Messages before this turn:');
console.log(problematicTurn.inputMessages);

// Step 4: 发现问题
// 原来用户之前说过 "顺便也看看酒店"
// Agent 记住了这个，但优先级判断错误

// Step 5: 修复
// 在 system prompt 中添加：
// "当用户在一条消息中提到多个任务时，按提及顺序依次完成"
```

## 调试心法

1. **先复现，再修复** - 确保能稳定复现问题再动手
2. **最小化上下文** - 用最少的 messages 复现问题
3. **检查假设** - "我以为 Agent 知道"往往是 bug 根源
4. **日志要丰富** - 调试时恨日志少，永远不会恨日志多
5. **保存现场** - 问题会话要完整保存，别急着清理

## 总结

| 问题类型 | 诊断方法 | 常见原因 |
|---------|---------|---------|
| 循环 | LoopDetector | 错误信息不明确、缺少失败处理 |
| 意外决策 | 上下文分析 | 指令冲突、工具不可用、上下文丢失 |
| 参数错误 | Schema 比对 | Schema 描述不清、类型不匹配 |
| 性能问题 | 延迟追踪 | 工具超时、模型响应慢 |
| 不一致 | 版本对比 | 配置变更、prompt 修改 |

记住：**Agent 调试 = 代码调试 + 心理分析**。你要理解 LLM 的"想法"，才能找到问题所在。

## 下一步

- 第 19 课预告：Cost Optimization - 成本优化策略
